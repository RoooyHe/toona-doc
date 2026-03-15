---
title: "Card 状态管理设计"
weight: 11
---

# Card 状态管理设计

## 概述

本文档描述如何为 Card 实现三种状态：未完成、完成和归档。这些状态用于组织和管理任务的生命周期。

## 状态定义

### 三种状态

1. **未完成 (Pending)** - 默认状态，表示任务正在进行中或待处理
2. **完成 (Completed)** - 表示任务已完成
3. **归档 (Archived)** - 表示任务已归档，不再显示在主列表中

## 数据模型

### CardStatus 枚举

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum CardStatus {
    /// 未完成状态
    Pending,
    
    /// 已完成状态
    Completed,
    
    /// 已归档状态
    Archived,
}

impl CardStatus {
    /// 获取状态的显示名称
    pub fn display_name(&self) -> &'static str {
        match self {
            CardStatus::Pending => "未完成",
            CardStatus::Completed => "已完成",
            CardStatus::Archived => "已归档",
        }
    }
    
    /// 获取状态的颜色
    pub fn color(&self) -> &'static str {
        match self {
            CardStatus::Pending => "#FFA500",    // 橙色
            CardStatus::Completed => "#61BD4F",  // 绿色
            CardStatus::Archived => "#95A5A6",   // 灰色
        }
    }
}
```

### KanbanCard 扩展

在 `KanbanCard` 中添加 `status` 字段：

```rust
pub struct KanbanCard {
    // ... 现有字段
    
    /// 卡片状态
    pub status: CardStatus,
    
    // ... 其他字段
}
```

## 实现方案

### 1. 数据结构更新

在 `src/kanban/state/kanban_state.rs` 中：

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum CardStatus {
    Pending,
    Completed,
    Archived,
}

impl CardStatus {
    pub fn display_name(&self) -> &'static str {
        match self {
            CardStatus::Pending => "未完成",
            CardStatus::Completed => "已完成",
            CardStatus::Archived => "已归档",
        }
    }
    
    pub fn color(&self) -> &'static str {
        match self {
            CardStatus::Pending => "#FFA500",
            CardStatus::Completed => "#61BD4F",
            CardStatus::Archived => "#95A5A6",
        }
    }
}

impl Default for CardStatus {
    fn default() -> Self {
        CardStatus::Pending
    }
}
```

### 2. Action 定义

在 `src/kanban/state/kanban_actions.rs` 中添加：

```rust
pub enum KanbanActions {
    // ... 现有 Actions
    
    /// 更新卡片状态
    UpdateCardStatus {
        card_id: OwnedRoomId,
        status: CardStatus,
    },
}
```

### 3. Matrix 适配器

在 `src/kanban/matrix_adapter.rs` 中，`CardMetadataRaw` 需要包含 status 字段：

```rust
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
struct CardMetadataRaw {
    pub title: String,
    #[serde(default)]
    pub description: Option<String>,
    #[serde(default = "default_position")]
    pub position: f64,
    #[serde(default)]
    pub end_time: Option<u64>,
    #[serde(default)]
    pub tags: Vec<String>,
    #[serde(default)]
    pub status: CardStatus,  // 新增
    #[serde(default = "default_timestamp")]
    pub created_at: u64,
    #[serde(default = "default_timestamp")]
    pub updated_at: u64,
}
```

### 4. UI 组件

在 `src/kanban/components/card_info_section.rs` 中添加状态显示和切换：

- 显示当前状态
- 提供状态切换按钮或下拉菜单
- 根据状态显示不同的颜色

### 5. 应用状态处理

在 `src/app.rs` 的 `handle_kanban_action` 中处理 `UpdateCardStatus`：

```rust
KanbanActions::UpdateCardStatus { card_id, status } => {
    log!("UpdateCardStatus: card_id='{}', status={:?}", card_id, status);
    
    if let Some(card) = state.cards.get_mut(&card_id) {
        card.status = status;
        card.touch();
        
        // 保存到 Matrix
        if get_client().is_some() {
            submit_async_request(MatrixRequest::SaveCardMetadata {
                card: card.clone(),
            });
        }
        
        // 触发 UI 重绘
        self.ui.redraw(cx);
    }
}
```

## 显示逻辑

### 卡片列表中的状态显示

- 未完成：显示为橙色标签
- 已完成：显示为绿色标签，可选择淡化显示
- 已归档：显示为灰色标签，可选择隐藏或单独显示

### 卡片详情中的状态管理

- 显示当前状态
- 提供状态切换按钮
- 允许用户在三种状态之间切换

## 迁移策略

### 向后兼容

对于现有的卡片（没有 status 字段），默认使用 `CardStatus::Pending`。

```rust
#[serde(default)]
pub status: CardStatus,
```

## 实现优先级

### Phase 1: 基础功能

1. 添加 CardStatus 枚举
2. 在 KanbanCard 中添加 status 字段
3. 在 CardMetadataRaw 中添加 status 字段
4. 实现 UpdateCardStatus Action
5. 在 card_info_section 中显示状态
6. 实现状态切换功能

### Phase 2: UI 增强

1. 在卡片列表中显示状态指示器
2. 根据状态过滤卡片显示
3. 添加状态统计信息
4. 实现状态转换动画

### Phase 3: 高级功能

1. 状态转换规则（例如：只能从 Pending 转到 Completed）
2. 状态转换历史记录
3. 自动状态转换（例如：超过截止时间自动标记为逾期）
4. 状态转换通知

## 测试计划

1. 单元测试：CardStatus 枚举和方法
2. 集成测试：状态更新和持久化
3. UI 测试：状态显示和切换
4. 迁移测试：旧数据的默认状态处理
