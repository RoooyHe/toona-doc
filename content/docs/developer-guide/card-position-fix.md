---
title: "卡片排序位置修复"
weight: 16
---

# 卡片排序位置修复

## 问题描述

在创建新卡片时，所有卡片的 `position` 字段都被设置为固定值 `1000.0`。这导致：

1. 多个卡片具有相同的 position 值
2. 卡片的显示顺序不确定
3. 新创建的卡片可能不会显示在列表末尾

## 排序字段

卡片使用 `position: f64` 字段进行排序：

```rust
pub struct KanbanCard {
    // ... 其他字段
    
    /// 排序位置（用于拖拽排序）
    pub position: f64,
    
    // ... 其他字段
}
```

- 较小的 position 值排在前面
- 较大的 position 值排在后面
- 使用浮点数允许在两个卡片之间插入新卡片

## 解决方案

修改 `create_card` 方法，在创建新卡片时计算正确的 position 值：

### 1. 获取现有卡片的最大 position

在创建新卡片之前，查询 Space 中所有现有卡片的 position 值，找到最大值。

### 2. 计算新卡片的 position

新卡片的 position = max_position + 1000.0

这确保：
- 新卡片总是排在现有卡片之后
- 保留足够的间隔（1000.0）用于未来的拖拽排序

### 3. 实现代码

在 `src/kanban/matrix_adapter.rs` 的 `create_card` 方法中：

```rust
// 计算新卡片的 position
let position = match self.client.get_room(space_id) {
    Some(space_room) => {
        match self.get_card_list_from_state(&space_room).await {
            Ok(card_ids) => {
                let mut max_position = 0.0;
                for card_id in card_ids {
                    if let Ok(card) = self.load_card(&card_id, space_id.to_owned()).await {
                        if card.position > max_position {
                            max_position = card.position;
                        }
                    }
                }
                max_position + 1000.0
            }
            Err(e) => {
                log!("⚠️ Failed to get existing cards, using default position: {:?}", e);
                1000.0
            }
        }
    }
    None => {
        log!("⚠️ Space not found, using default position");
        1000.0
    }
};

// 创建卡片并设置 position
let mut card = KanbanCard::new(card_room_id.clone(), title.to_string(), space_id.to_owned());
card.position = position;
```

## 降级处理

如果无法获取现有卡片列表（例如网络错误），使用默认值 `1000.0`：

- 第一张卡片：position = 1000.0
- 如果获取失败：position = 1000.0（可能与其他卡片重复）

这确保即使在错误情况下，卡片仍然可以被创建。

## 效果

修复后：

1. 新创建的卡片总是显示在列表末尾
2. 每张卡片都有唯一的 position 值（除非发生错误）
3. 卡片的显示顺序是确定的和可预测的
4. 为未来的拖拽排序功能预留了足够的间隔

## 未来改进

1. **批量创建优化**：如果同时创建多张卡片，可以批量计算 position
2. **position 重新分配**：当 position 值过大或间隔过小时，重新分配所有卡片的 position
3. **拖拽排序**：实现拖拽功能时，计算两张卡片之间的中间 position 值

## 相关代码位置

- 卡片结构定义: `src/kanban/state/kanban_state.rs` 第 139-230 行
- 创建卡片方法: `src/kanban/matrix_adapter.rs` 第 159-240 行
- 卡片元数据保存: `src/kanban/matrix_adapter.rs` 第 265-302 行
