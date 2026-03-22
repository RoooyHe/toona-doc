---
title: "Space 共用标签设计"
weight: 10
---

# Space 共用标签设计

## 概述

本文档描述如何在一个 Space 内实现共用标签系统，使得同一 Space 下的所有 Card 可以使用统一的标签集合。

## 背景

当前实现中，每个 Card 的 tags 字段是一个独立的字符串数组，存储在 Card 的 Matrix Room 状态中。这种设计存在以下问题：

1. 标签定义分散在各个 Card 中，无法统一管理
2. 相同名称的标签可能在不同 Card 中重复定义
3. 无法为标签设置统一的颜色、描述等元数据
4. 标签重命名或删除时需要遍历所有 Card 进行更新

## 设计目标

1. 在 Space 级别维护一个共用的标签库
2. Card 通过标签 ID 引用 Space 中的标签
3. 支持标签的增删改查操作
4. 保持与 Matrix 协议的兼容性
5. 最小化对现有代码的改动

## 数据模型设计

### Space 标签库

在 Space 的 Room State 中存储标签定义：

```rust
/// Space 标签定义（存储在 Space Room State）
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SpaceTag {
    /// 标签唯一 ID
    pub id: String,
    
    /// 标签名称
    pub name: String,
    
    /// 标签颜色（十六进制）
    pub color: String,
    
    /// 标签描述（可选）
    pub description: Option<String>,
    
    /// 创建时间
    pub created_at: u64,
    
    /// 最后更新时间
    pub updated_at: u64,
}

/// Space 标签库（存储在 Space Room State）
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SpaceTagLibrary {
    /// 标签列表
    pub tags: Vec<SpaceTag>,
    
    /// 版本号（用于同步冲突检测）
    pub version: u64,
}
```

### Matrix 存储方案

使用 Matrix Room State Event 存储标签库：

```json
{
  "type": "m.space.tag_library",
  "state_key": "",
  "content": {
    "tags": [
      {
        "id": "tag_1234567890_abc",
        "name": "紧急",
        "color": "#EB5A46",
        "description": "需要立即处理的任务",
        "created_at": 1234567890,
        "updated_at": 1234567890
      },
      {
        "id": "tag_1234567891_def",
        "name": "设计",
        "color": "#0079BF",
        "description": "设计相关任务",
        "created_at": 1234567891,
        "updated_at": 1234567891
      }
    ],
    "version": 1
  }
}
```

### Card 标签引用

Card 继续使用 `tags` 字段，但存储的是标签 ID 而非标签名称：

```rust
/// KanbanCard 中的 tags 字段
pub struct KanbanCard {
    // ... 其他字段
    
    /// 标签 ID 列表（引用 Space 标签库中的标签）
    pub tags: Vec<String>,  // 存储标签 ID，如 ["tag_1234567890_abc", "tag_1234567891_def"]
}
```

## 实现方案

### 1. 数据结构扩展

在 `src/kanban/state/kanban_state.rs` 中添加：

```rust
/// Space 标签定义
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SpaceTag {
    pub id: String,
    pub name: String,
    pub color: String,
    pub description: Option<String>,
    pub created_at: u64,
    pub updated_at: u64,
}

impl SpaceTag {
    pub fn new(name: String, color: String) -> Self {
        let now = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_secs();
        
        let random = uuid::Uuid::new_v4().to_string();
        let id = format!("tag_{}_{}", now, &random[..8]);
        
        Self {
            id,
            name,
            color,
            description: None,
            created_at: now,
            updated_at: now,
        }
    }
}

/// KanbanAppState 扩展
impl KanbanAppState {
    /// Space 标签库（Space ID -> Tags）
    pub space_tags: HashMap<OwnedRoomId, Vec<SpaceTag>>,
}
```

### 2. Matrix 适配器扩展

在 `src/kanban/matrix_adapter.rs` 中添加标签库操作方法：

```rust
impl MatrixKanbanAdapter {
    /// 加载 Space 标签库
    pub async fn load_space_tags(&self, space_id: &RoomId) -> Result<Vec<SpaceTag>> {
        let space = self.client.get_room(space_id)
            .ok_or_else(|| anyhow!("Space not found"))?;
        
        // 从 Room State 读取标签库
        let event = space.get_state_event_static::<RoomTagLibraryEventContent>().await?;
        
        if let Some(content) = event {
            Ok(content.tags)
        } else {
            Ok(Vec::new())
        }
    }
    
    /// 保存 Space 标签库
    pub async fn save_space_tags(&self, space_id: &RoomId, tags: Vec<SpaceTag>) -> Result<()> {
        let space = self.client.get_room(space_id)
            .ok_or_else(|| anyhow!("Space not found"))?;
        
        let content = RoomTagLibraryEventContent { tags };
        space.send_state_event(content).await?;
        
        Ok(())
    }
    
    /// 添加标签到 Space
    pub async fn add_space_tag(&self, space_id: &RoomId, tag: SpaceTag) -> Result<()> {
        let mut tags = self.load_space_tags(space_id).await?;
        tags.push(tag);
        self.save_space_tags(space_id, tags).await
    }
    
    /// 更新 Space 标签
    pub async fn update_space_tag(&self, space_id: &RoomId, tag: SpaceTag) -> Result<()> {
        let mut tags = self.load_space_tags(space_id).await?;
        
        if let Some(existing) = tags.iter_mut().find(|t| t.id == tag.id) {
            *existing = tag;
            self.save_space_tags(space_id, tags).await
        } else {
            Err(anyhow!("Tag not found"))
        }
    }
    
    /// 删除 Space 标签
    pub async fn delete_space_tag(&self, space_id: &RoomId, tag_id: &str) -> Result<()> {
        let mut tags = self.load_space_tags(space_id).await?;
        tags.retain(|t| t.id != tag_id);
        self.save_space_tags(space_id, tags).await
    }
}
```

### 3. Action 扩展

在 `src/kanban/mod.rs` 中添加标签管理 Action：

```rust
#[derive(Clone, Debug, DefaultNone)]
pub enum KanbanActions {
    // ... 现有 Actions
    
    // Space 标签管理
    LoadSpaceTags { space_id: OwnedRoomId },
    CreateSpaceTag { space_id: OwnedRoomId, name: String, color: String },
    UpdateSpaceTag { space_id: OwnedRoomId, tag: SpaceTag },
    DeleteSpaceTag { space_id: OwnedRoomId, tag_id: String },
    
    // Card 标签操作（修改为使用标签 ID）
    AddTagToCard { card_id: OwnedRoomId, tag_id: String },
    RemoveTagFromCard { card_id: OwnedRoomId, tag_id: String },
}
```

### 4. UI 组件更新

修改 `src/kanban/components/tag_section.rs`：

```rust
impl Widget for TagSection {
    fn draw_walk(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) -> DrawStep {
        // 获取当前 Card 和所属 Space
        let (card_tags, space_tags) = if let Some(app_state) = scope.data.get::<AppState>() {
            if let Some(card_id) = &app_state.kanban_state.selected_card_id {
                let card = app_state.kanban_state.cards.get(card_id);
                let space_id = card.map(|c| &c.space_id);
                
                let card_tag_ids = card.map(|c| c.tags.clone()).unwrap_or_default();
                let space_tags = space_id
                    .and_then(|sid| app_state.kanban_state.space_tags.get(sid))
                    .cloned()
                    .unwrap_or_default();
                
                (card_tag_ids, space_tags)
            } else {
                (Vec::new(), Vec::new())
            }
        } else {
            (Vec::new(), Vec::new())
        };
        
        // 根据 tag_id 查找 tag 定义并显示
        let tag_names: Vec<String> = card_tags
            .iter()
            .filter_map(|tag_id| {
                space_tags.iter()
                    .find(|t| &t.id == tag_id)
                    .map(|t| format!("{} ({})", t.name, t.color))
            })
            .collect();
        
        // 显示标签
        if tag_names.is_empty() {
            self.view.label(ids!(empty_label)).set_visible(cx, true);
        } else {
            let tags_text = tag_names.join(", ");
            self.view.label(ids!(tags_display_label)).set_text(cx, &tags_text);
            self.view.label(ids!(tags_display_label)).set_visible(cx, true);
        }
        
        self.view.draw_walk(cx, scope, walk)
    }
}
```

## 迁移策略

### 向后兼容

为了保持向后兼容，需要支持旧数据格式的迁移：

1. 读取 Card 时，检查 tags 字段内容
2. 如果是标签名称（不符合 `tag_*` 格式），自动转换为标签 ID
3. 在 Space 标签库中创建对应的标签定义
4. 更新 Card 的 tags 字段为标签 ID

```rust
impl MatrixKanbanAdapter {
    /// 迁移旧格式标签
    async fn migrate_card_tags(&self, card: &mut KanbanCard, space_id: &RoomId) -> Result<()> {
        let mut space_tags = self.load_space_tags(space_id).await?;
        let mut updated = false;
        
        for tag in &mut card.tags {
            // 检查是否为旧格式（标签名称而非 ID）
            if !tag.starts_with("tag_") {
                // 在 Space 标签库中查找或创建
                let space_tag = space_tags.iter()
                    .find(|t| t.name == *tag)
                    .cloned()
                    .unwrap_or_else(|| {
                        let new_tag = SpaceTag::new(tag.clone(), "#0079BF".to_string());
                        space_tags.push(new_tag.clone());
                        new_tag
                    });
                
                // 替换为标签 ID
                *tag = space_tag.id;
                updated = true;
            }
        }
        
        if updated {
            self.save_space_tags(space_id, space_tags).await?;
            self.save_card_metadata(card).await?;
        }
        
        Ok(())
    }
}
```

## 预定义标签颜色

提供一组预定义的标签颜色供用户选择：

```rust
pub const PREDEFINED_TAG_COLORS: &[(&str, &str)] = &[
    ("红色", "#EB5A46"),
    ("橙色", "#FF9F1A"),
    ("黄色", "#F2D600"),
    ("绿色", "#61BD4F"),
    ("青色", "#00C2E0"),
    ("蓝色", "#0079BF"),
    ("紫色", "#9775FA"),
    ("粉色", "#FF78CB"),
    ("灰色", "#95A5A6"),
    ("黑色", "#343434"),
];
```

## 同步策略

### 加载流程

1. 加载 Space 列表时，同时加载每个 Space 的标签库
2. 加载 Card 时，通过标签 ID 引用 Space 标签库中的标签
3. 在 UI 中显示标签时，从标签库中查找标签定义

### 更新流程

1. 用户创建/修改/删除标签时，更新 Space 标签库
2. 标签库更新后，触发 UI 重绘
3. 删除标签时，需要从所有引用该标签的 Card 中移除

### 冲突处理

使用版本号机制处理并发修改：

1. 每次更新标签库时递增版本号
2. 更新前检查版本号是否匹配
3. 如果版本号不匹配，提示用户刷新后重试

## 实现优先级

### Phase 1: 基础功能

1. 添加 SpaceTag 数据结构
2. 实现 Space 标签库的加载和保存
3. 修改 Card 标签引用机制
4. 实现旧数据迁移

### Phase 2: UI 增强

1. 标签选择器组件（从 Space 标签库选择）
2. 标签管理界面（创建、编辑、删除标签）
3. 标签颜色选择器
4. 标签使用统计

### Phase 3: 高级功能

1. 标签分组
2. 标签搜索和过滤
3. 标签模板（跨 Space 复制标签库）
4. 标签使用分析

## 测试计划

1. 单元测试：标签 CRUD 操作
2. 集成测试：标签库与 Card 的关联
3. 迁移测试：旧格式数据的自动迁移
4. 并发测试：多用户同时修改标签库
5. UI 测试：标签选择和显示

## 实现状态

### 当前实现: 简化方案 (已完成)

为了快速实现标签功能，采用了简化方案：**直接存储标签名称而不是标签 ID**。

这种方案的优点：
- 实现简单，无需维护 Space 标签库
- 用户可以自由创建任意标签名称
- 标签数据直接存储在 Card 中，无需额外的 Matrix Room State

实现内容：

1. **数据结构** (`src/kanban/state/kanban_state.rs`)
   - `KanbanCard.tags`: 直接存储标签名称字符串列表
   - `CardStatus` 枚举: 卡片状态（未完成、已完成、已归档）

2. **UI 组件** (`src/kanban/components/tag_section.rs`)
   - 标签显示: 直接显示 `card.tags` 中的标签名称
   - 标签添加: 用户输入标签名称，直接保存到 Card
   - 标签删除: 从 `card.tags` 中移除指定标签

3. **Action 系统** (`src/kanban/state/kanban_actions.rs`)
   - `AddTagToCard`: 添加标签名称到 Card
   - `RemoveTag`: 从 Card 移除标签

4. **应用状态处理** (`src/app.rs`)
   - 处理 `AddTagToCard` 和 `RemoveTag` Actions
   - 在内存中更新 Card 的标签
   - 通过 `SaveCardMetadata` 请求保存到 Matrix

5. **卡片状态管理** (`src/kanban/components/card_info_section.rs`)
   - 状态按钮: 未完成、已完成、已归档
   - 状态持久化: 通过 `SaveCardMetadata` 保存到 Matrix

### Phase 1 完成的功能

已在项目中实现以下内容:

1. **数据结构** (`src/kanban/state/kanban_state.rs`)
   - `SpaceTag` 结构体: 定义标签的 ID、名称、颜色、描述等（预留用于未来扩展）
   - `PREDEFINED_TAG_COLORS` 常量: 预定义的 10 种标签颜色
   - `KanbanAppState` 扩展: 添加 `space_tags` 字段存储 Space 标签库（预留用于未来扩展）

2. **Matrix 适配器** (`src/kanban/matrix_adapter.rs`)
   - `load_space_tags()`: 从 Space 的 Room State 加载标签库（预留）
   - `save_space_tags()`: 保存标签库到 Space 的 Room State（预留）
   - `add_space_tag()`: 添加新标签到 Space（预留）
   - `update_space_tag()`: 更新 Space 中的标签（预留）
   - `delete_space_tag()`: 删除 Space 中的标签（预留）
   - `migrate_card_tags()`: 自动迁移旧格式标签(标签名称 -> 标签 ID)（预留）

3. **Action 系统** (`src/kanban/state/kanban_actions.rs`)
   - `AddTagToCard`: 添加标签到 Card
   - `RemoveTag`: 从 Card 移除标签
   - 其他标签库相关 Actions（预留用于未来扩展）

4. **应用状态处理** (`src/app.rs`)
   - 处理 `AddTagToCard` 和 `RemoveTag` Actions
   - 在内存中更新 Card 的标签
   - 自动保存到 Matrix

### 未来扩展方向

当需要更复杂的标签管理功能时，可以升级到完整的 Space 标签库方案：

1. **标签库管理**: 在 Space 级别维护统一的标签定义
2. **标签颜色**: 为每个标签分配颜色
3. **标签描述**: 为标签添加描述信息
4. **标签权限**: 控制谁可以创建/修改/删除标签
5. **标签统计**: 显示标签的使用情况

## 参考资料

- Matrix Room State Events: https://spec.matrix.org/v1.11/client-server-api/#room-state
- Trello Labels: https://trello.com/guide/labels


## 已知问题修复

### 标签显示为 ID 格式

问题：标签被保存为 `tag_1774186137_688c8d24` 这样的 ID 格式，而不是用户输入的名称。

原因：`matrix_adapter.rs` 中的 `migrate_card_tags` 方法会将标签名称转换为生成的 ID。

修复方案：

1. 移除标签迁移逻辑，直接使用标签名称

```rust
pub async fn migrate_card_tags(&self, _card: &mut KanbanCard, _space_id: &RoomId) -> Result<()> {
    // 简化方案：直接使用标签名称，不需要迁移
    Ok(())
}
```

2. 在 UI 显示时自动过滤旧格式的标签 ID

在 `tag_section.rs` 的 `draw_walk` 方法中添加过滤逻辑：

```rust
// 过滤掉旧格式的标签 ID（格式：tag_数字_哈希）
card.tags.iter()
    .filter(|tag| {
        let is_old_format = tag.starts_with("tag_") && 
            tag.matches('_').count() >= 2;
        !is_old_format
    })
    .cloned()
    .collect()
```

修复效果：
- 新添加的标签直接使用用户输入的名称
- 旧格式的标签 ID 会被自动隐藏，不显示在 UI 上
- 用户可以重新添加正确格式的标签

数据清理：
- 旧格式标签仍存储在 Matrix 服务器中，但不影响使用
- 如需彻底清理，可以在卡片详情页面重新添加标签（旧标签会被自动过滤）
