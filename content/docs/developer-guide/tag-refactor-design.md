---
title: "标签功能重构设计"
weight: 13
---

# 标签功能重构设计

## 需求概述

重构标签功能，实现以下目标：
1. 标签以下拉多选框的形式呈现
2. 标签数据在 Space 级别共享，所有卡片使用同一个标签库
3. 支持创建新标签并添加到标签库
4. 支持为卡片添加/移除标签
5. 标签显示为带颜色的标签卡片

## 架构设计

### 数据模型

#### Space 标签库

每个 Space 维护一个共享的标签库：

```rust
// Space 级别的标签定义
pub struct SpaceTag {
    pub id: String,           // 标签 ID（格式：tag_时间戳_哈希）
    pub name: String,         // 标签名称（用户可见）
    pub color: String,        // 标签颜色（十六进制，如 "#0079BF"）
    pub created_at: u64,      // 创建时间戳
}

// KanbanList (Space) 结构扩展
pub struct KanbanList {
    pub id: OwnedRoomId,
    pub name: String,
    pub cards: Vec<KanbanCard>,
    pub tags: Vec<SpaceTag>,  // 新增：Space 标签库
    // ... 其他字段
}
```

#### Card 标签引用

卡片只存储标签 ID 的引用：

```rust
pub struct KanbanCard {
    pub id: OwnedRoomId,
    pub title: String,
    pub tags: Vec<String>,    // 存储标签 ID，不是标签名称
    // ... 其他字段
}
```

### 存储方案

#### Matrix 协议存储

1. **Space 标签库存储**
   - 存储位置：Space Room 的 State Event
   - Event Type: `m.space.tags`
   - Content 格式：
     ```json
     {
       "tags": [
         {
           "id": "tag_1234567890_abc123",
           "name": "紧急",
           "color": "#FF6B6B",
           "created_at": 1234567890
         },
         {
           "id": "tag_1234567891_def456",
           "name": "设计",
           "color": "#4ECDC4",
           "created_at": 1234567891
         }
       ]
     }
     ```

2. **Card 标签引用存储**
   - 存储位置：Card Room 的 Timeline Message
   - 在现有的 `m.room.message` 中添加 `tags` 字段
   - Content 格式：
     ```json
     {
       "msgtype": "m.text",
       "body": "卡片标题",
       "toona.card.metadata": {
         "title": "卡片标题",
         "status": "Active",
         "tags": ["tag_1234567890_abc123", "tag_1234567891_def456"],
         "end_time": 1234567890
       }
     }
     ```

### UI 设计

#### 标签选择器组件

```
┌─────────────────────────────────────┐
│ 标签                          [▼]   │  ← 下拉按钮
├─────────────────────────────────────┤
│ 当前标签:                           │
│ ┌────────┐ ┌────────┐              │
│ │ 紧急 ×  │ │ 设计 ×  │              │  ← 已选标签（可点击 × 删除）
│ └────────┘ └────────┘              │
└─────────────────────────────────────┘

点击下拉按钮后：
┌─────────────────────────────────────┐
│ 选择标签                      [×]   │  ← 关闭按钮
├─────────────────────────────────────┤
│ ☐ 紧急                              │  ← 未选中的标签
│ ☑ 设计                              │  ← 已选中的标签
│ ☐ 开发                              │
│ ☐ 测试                              │
├─────────────────────────────────────┤
│ [+ 创建新标签]                      │  ← 创建按钮
└─────────────────────────────────────┘

点击创建新标签后：
┌─────────────────────────────────────┐
│ 创建新标签                          │
├─────────────────────────────────────┤
│ 标签名称: [____________]            │
│                                     │
│ 颜色:                               │
│ ⬤ 红色  ⬤ 蓝色  ⬤ 绿色             │
│ ⬤ 黄色  ⬤ 紫色  ⬤ 橙色             │
│                                     │
│ [取消]  [创建]                      │
└─────────────────────────────────────┘
```

#### 标签显示样式

已选标签显示为带颜色的卡片：

```
┌──────────────┐
│  紧急    ×   │  ← 红色背景，白色文字
└──────────────┘

┌──────────────┐
│  设计    ×   │  ← 蓝色背景，白色文字
└──────────────┘
```

### 组件结构

```
TagSection (标签管理组件)
├── TagDisplay (标签显示区域)
│   ├── TagChip (单个标签卡片) × N
│   │   ├── Label (标签名称)
│   │   └── Button (删除按钮 ×)
│   └── Button (下拉按钮 ▼)
│
└── TagDropdown (标签下拉选择器)
    ├── TagCheckboxList (标签复选框列表)
    │   └── TagCheckboxItem × N
    │       ├── Checkbox (选择框)
    │       └── Label (标签名称)
    │
    ├── Button (创建新标签按钮)
    │
    └── CreateTagModal (创建标签模态框)
        ├── TextInput (标签名称输入)
        ├── ColorPicker (颜色选择器)
        └── Buttons (取消/创建按钮)
```

## 功能流程

### 1. 加载标签库

```
用户打开卡片详情
    ↓
从 AppState 获取当前 Space ID
    ↓
从 Space 的 tags 字段获取标签库
    ↓
渲染标签选择器
```

### 2. 显示卡片标签

```
从 Card 的 tags 字段获取标签 ID 列表
    ↓
根据 ID 从 Space 标签库中查找标签信息
    ↓
渲染标签卡片（显示名称和颜色）
```

### 3. 添加标签到卡片

```
用户点击下拉按钮
    ↓
显示标签复选框列表
    ↓
用户勾选标签
    ↓
触发 AddTagToCard 动作
    ↓
更新 Card 的 tags 字段
    ↓
保存到 Matrix
    ↓
刷新 UI
```

### 4. 从卡片移除标签

```
用户点击标签卡片上的 × 按钮
    ↓
触发 RemoveTagFromCard 动作
    ↓
从 Card 的 tags 字段移除标签 ID
    ↓
保存到 Matrix
    ↓
刷新 UI
```

### 5. 创建新标签

```
用户点击"创建新标签"按钮
    ↓
显示创建标签模态框
    ↓
用户输入标签名称和选择颜色
    ↓
触发 CreateSpaceTag 动作
    ↓
生成标签 ID（格式：tag_时间戳_哈希）
    ↓
添加到 Space 的 tags 字段
    ↓
保存到 Matrix
    ↓
刷新标签库
```

## 动作定义

### KanbanActions 扩展

```rust
pub enum KanbanActions {
    // ... 现有动作
    
    // Space 标签管理
    LoadSpaceTags {
        space_id: OwnedRoomId,
    },
    
    SpaceTagsLoaded {
        space_id: OwnedRoomId,
        tags: Vec<SpaceTag>,
    },
    
    CreateSpaceTag {
        space_id: OwnedRoomId,
        name: String,
        color: String,
    },
    
    UpdateSpaceTag {
        space_id: OwnedRoomId,
        tag: SpaceTag,
    },
    
    DeleteSpaceTag {
        space_id: OwnedRoomId,
        tag_id: String,
    },
    
    // Card 标签操作
    AddTagToCard {
        card_id: OwnedRoomId,
        tag_id: String,
    },
    
    RemoveTagFromCard {
        card_id: OwnedRoomId,
        tag_id: String,
    },
}
```

## 实现步骤

### Phase 1: 数据模型和存储

1. 定义 `SpaceTag` 结构体
2. 扩展 `KanbanList` 添加 `tags` 字段
3. 实现 Space 标签库的 Matrix 存储和加载
4. 修改 Card 标签存储为 ID 引用

### Phase 2: 标签库管理

1. 实现 `LoadSpaceTags` 动作处理
2. 实现 `CreateSpaceTag` 动作处理
3. 实现 `UpdateSpaceTag` 动作处理
4. 实现 `DeleteSpaceTag` 动作处理

### Phase 3: UI 组件

1. 创建 `TagChip` 组件（标签卡片）
2. 创建 `TagDropdown` 组件（下拉选择器）
3. 创建 `TagCheckboxList` 组件（复选框列表）
4. 创建 `CreateTagModal` 组件（创建标签模态框）
5. 重构 `TagSection` 组件集成以上组件

### Phase 4: 卡片标签操作

1. 实现 `AddTagToCard` 动作处理
2. 实现 `RemoveTagFromCard` 动作处理
3. 实现标签显示逻辑（ID → 标签信息）
4. 实现标签选择器交互

### Phase 5: 数据迁移

1. 实现旧数据迁移逻辑
2. 将现有的标签名称转换为标签 ID
3. 为每个 Space 创建初始标签库

## 预设颜色方案

提供 8 种预设颜色供用户选择：

```rust
pub const TAG_COLORS: &[(&str, &str)] = &[
    ("红色", "#FF6B6B"),
    ("橙色", "#FFA500"),
    ("黄色", "#FFD93D"),
    ("绿色", "#6BCF7F"),
    ("蓝色", "#0079BF"),
    ("紫色", "#9B59B6"),
    ("粉色", "#FF69B4"),
    ("灰色", "#95A5A6"),
];
```

## 优势分析

### 相比当前方案的改进

1. **统一管理**: Space 级别的标签库，避免标签重复和不一致
2. **更好的 UX**: 下拉多选框比手动输入更直观
3. **视觉识别**: 彩色标签卡片提供更好的视觉区分
4. **可扩展性**: 支持标签的编辑、删除、重命名等高级功能
5. **数据一致性**: 使用 ID 引用，修改标签名称时自动同步到所有卡片

### 技术优势

1. **性能优化**: 标签库在 Space 级别缓存，减少重复加载
2. **数据完整性**: ID 引用确保标签数据的一致性
3. **易于维护**: 清晰的数据模型和组件结构
4. **可测试性**: 独立的组件便于单元测试

## 注意事项

### Makepad 框架限制

1. **下拉框实现**: Makepad 可能没有原生的下拉多选框组件，需要使用 Modal + PortalList 实现
2. **颜色选择器**: 可能需要自定义实现颜色选择器组件
3. **复选框**: 使用 Makepad 的 CheckBox 组件或自定义实现

### 数据迁移

1. **向后兼容**: 需要处理旧格式的标签数据
2. **渐进式迁移**: 在加载时自动迁移，不影响现有功能
3. **数据备份**: 迁移前建议备份数据

### 性能考虑

1. **标签库大小**: 建议限制每个 Space 的标签数量（如 50 个）
2. **卡片标签数量**: 建议限制每个卡片的标签数量（如 10 个）
3. **缓存策略**: 在 AppState 中缓存标签库，减少重复加载

## 后续扩展

### 可能的功能扩展

1. **标签搜索**: 在标签选择器中添加搜索功能
2. **标签排序**: 支持按名称、使用频率排序
3. **标签统计**: 显示每个标签被使用的次数
4. **标签过滤**: 在看板视图中按标签过滤卡片
5. **标签模板**: 提供常用标签模板（如：优先级、类型等）
6. **自定义颜色**: 支持用户输入自定义颜色值
7. **标签图标**: 为标签添加图标支持

## 参考资料

- Trello 标签系统设计
- Makepad PortalList 文档
- Matrix State Event 规范
- 现有的 TodoSection 实现（复选框参考）
