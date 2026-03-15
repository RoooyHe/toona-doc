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

### Phase 1: 基础功能 (已完成)

已在项目中实现以下内容:

1. **数据结构** (`src/kanban/state/kanban_state.rs`)
   - `SpaceTag` 结构体: 定义标签的 ID、名称、颜色、描述等
   - `PREDEFINED_TAG_COLORS` 常量: 预定义的 10 种标签颜色
   - `KanbanAppState` 扩展: 添加 `space_tags` 字段存储 Space 标签库

2. **Matrix 适配器** (`src/kanban/matrix_adapter.rs`)
   - `load_space_tags()`: 从 Space 的 Room State 加载标签库
   - `save_space_tags()`: 保存标签库到 Space 的 Room State
   - `add_space_tag()`: 添加新标签到 Space
   - `update_space_tag()`: 更新 Space 中的标签
   - `delete_space_tag()`: 删除 Space 中的标签
   - `migrate_card_tags()`: 自动迁移旧格式标签(标签名称 -> 标签 ID)

3. **Action 系统** (`src/kanban/state/kanban_actions.rs`)
   - `LoadSpaceTags`: 加载 Space 标签库
   - `SpaceTagsLoaded`: 标签库加载完成
   - `CreateSpaceTag`: 创建新标签
   - `UpdateSpaceTag`: 更新标签
   - `DeleteSpaceTag`: 删除标签
   - `AddTagToCard`: 添加标签到 Card
   - `RemoveTagFromCard`: 从 Card 移除标签

4. **Matrix 请求** (`src/sliding_sync.rs`)
   - `LoadSpaceTags`: 异步加载标签库
   - `CreateSpaceTag`: 异步创建标签
   - `UpdateSpaceTag`: 异步更新标签
   - `DeleteSpaceTag`: 异步删除标签

5. **应用状态处理** (`src/app.rs`)
   - 处理所有标签相关的 Actions
   - 在内存中更新 Card 的标签
   - 自动保存到 Matrix

6. **自动迁移**
   - 加载 Space 时自动加载标签库
   - 加载 Card 时自动检测并迁移旧格式标签

### Phase 2 & 3: 待实现

- UI 组件: 标签选择器、标签管理界面、颜色选择器
- 标签使用统计
- 标签分组和搜索

## 实现细节

### 自动标签添加流程

当用户在 Card 中创建新标签时，需要自动将该标签添加到 Card。实现流程如下:

1. **用户输入标签名称** (`tag_section.rs` - `handle_event`)
   - 用户点击"保存"按钮
   - 保存待添加的标签名称到 `pending_tag_name`
   - 保存 Space ID 到 `pending_space_id`
   - 检查标签库中是否已存在同名标签
   - 如果存在，直接调用 `AddTagToCard` Action
   - 如果不存在，调用 `CreateSpaceTag` Action

2. **创建标签** (`sliding_sync.rs` - `matrix_worker_task`)
   - `CreateSpaceTag` 请求被处理
   - 创建新的 `SpaceTag` 对象
   - 通过 Matrix 适配器保存到 Space 的 Room State
   - 重新加载 Space 标签库
   - 发送 `SpaceTagsLoaded` Action

3. **自动添加标签到 Card** (`tag_section.rs` - `draw_walk`)
   - 在 `draw_walk` 中检查是否有待添加的标签
   - 当 `SpaceTagsLoaded` 被处理后，标签库已更新
   - 在新加载的标签库中查找待添加的标签
   - 如果找到，自动调用 `AddTagToCard` Action
   - 清除 `pending_tag_name` 和 `pending_space_id`

4. **保存标签到 Card** (`app.rs` - `handle_kanban_action`)
   - `AddTagToCard` Action 被处理
   - 在内存中更新 Card 的 tags 字段
   - 调用 `SaveCardMetadata` 请求保存到 Matrix

### 关键实现点

**TagSection 组件状态**:
```rust
pub struct TagSection {
    view: View,
    card_id: Option<OwnedRoomId>,
    is_adding: bool,
    pending_tag_name: Option<String>,      // 待添加的标签名称
    pending_space_id: Option<OwnedRoomId>, // 待添加标签的 Space ID
}
```

**自动添加逻辑** (`draw_walk` 方法):
- 每次重绘时检查 `pending_tag_name` 和 `pending_space_id`
- 如果存在待添加的标签，在当前 Space 的标签库中查找
- 找到后自动触发 `AddTagToCard` Action
- 清除待添加状态，防止重复添加

**流程图**:
```
用户输入标签名称
    |
    v
检查标签库中是否存在
    |
    +---> 存在 ---> 直接 AddTagToCard
    |
    +---> 不存在 ---> CreateSpaceTag
                        |
                        v
                    Matrix 保存
                        |
                        v
                    SpaceTagsLoaded
                        |
                        v
                    draw_walk 检测
                        |
                        v
                    自动 AddTagToCard
                        |
                        v
                    SaveCardMetadata
```

## 参考资料

- Matrix Room State Events: https://spec.matrix.org/v1.11/client-server-api/#room-state
- Trello Labels: https://trello.com/guide/labels
