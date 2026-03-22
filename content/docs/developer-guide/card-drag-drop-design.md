---
title: "卡片拖拽移动设计"
weight: 6
---

# 卡片拖拽移动设计

## 概述

本文档描述卡片在看板列表（Space）之间拖拽移动的设计方案。用户可以通过拖拽操作将卡片从一个列表移动到另一个列表，实现灵活的任务管理。

## 核心需求

### 用户故事

作为用户，我希望能够：
- 通过拖拽操作将卡片从一个列表移动到另一个列表
- 在拖拽过程中看到清晰的视觉反馈
- 拖拽完成后卡片立即出现在目标列表中
- 拖拽操作能够跨设备同步

### 功能需求

1. 拖拽启动：用户按住卡片并移动鼠标/手指
2. 拖拽过程：显示拖拽预览，高亮可放置区域
3. 拖拽放置：释放鼠标/手指时将卡片移动到目标位置
4. 拖拽取消：拖拽到无效区域时取消操作

## 技术设计

### 拖拽状态管理

在 `KanbanAppState` 中添加拖拽状态：

```rust
pub struct KanbanAppState {
    // ... 现有字段
    
    // 拖拽状态
    pub drag_state: Option<DragState>,
}

pub struct DragState {
    // 被拖拽的卡片 ID
    pub card_id: OwnedRoomId,
    // 卡片原始所属的 Space ID
    pub source_space_id: OwnedRoomId,
    // 卡片在原列表中的位置
    pub source_position: f64,
    // 当前鼠标/手指位置
    pub current_pos: DVec2,
    // 拖拽开始时间
    pub start_time: u64,
}
```

### 拖拽事件流程

#### 1. 拖拽开始（Drag Start）

触发条件：
- 用户在卡片上按下鼠标左键/触摸
- 移动距离超过阈值（5px）

处理逻辑：
```rust
KanbanActions::StartDragCard {
    card_id: OwnedRoomId,
    space_id: OwnedRoomId,
    position: f64,
    start_pos: DVec2,
}
```

状态更新：
- 创建 `DragState` 并存储到 `KanbanAppState`
- 设置卡片为"拖拽中"状态（视觉效果：半透明）
- 创建拖拽预览层（跟随鼠标移动的卡片副本）

#### 2. 拖拽移动（Drag Move）

触发条件：
- 鼠标/手指移动

处理逻辑：
```rust
KanbanActions::MoveDragCard {
    current_pos: DVec2,
}
```

状态更新：
- 更新 `DragState.current_pos`
- 检测鼠标下方的目标 Space
- 高亮目标 Space（视觉效果：边框高亮）
- 计算插入位置（在哪两张卡片之间）

#### 3. 拖拽放置（Drag Drop）

触发条件：
- 释放鼠标左键/结束触摸

处理逻辑：
```rust
KanbanActions::DropCard {
    card_id: OwnedRoomId,
    target_space_id: OwnedRoomId,
    target_position: f64,
}
```

状态更新：
- 清除 `DragState`
- 如果目标 Space 与源 Space 不同：
  - 从源 Space 的 `card_ids` 中移除卡片
  - 添加到目标 Space 的 `card_ids` 中
  - 更新卡片的 `space_id` 和 `position`
- 如果目标 Space 与源 Space 相同：
  - 仅更新卡片的 `position`（重新排序）
- 发送 Matrix 请求更新服务器状态

#### 4. 拖拽取消（Drag Cancel）

触发条件：
- 按下 ESC 键
- 拖拽到窗口外部
- 拖拽到无效区域

处理逻辑：
```rust
KanbanActions::CancelDragCard
```

状态更新：
- 清除 `DragState`
- 恢复卡片原始状态

### Position 计算逻辑

使用浮点数 `position` 字段实现灵活的排序：

#### 插入到列表开头
```rust
new_position = first_card_position / 2.0
```

#### 插入到列表末尾
```rust
new_position = last_card_position + 1000.0
```

#### 插入到两张卡片之间
```rust
new_position = (prev_card_position + next_card_position) / 2.0
```

#### Position 重整（可选优化）
当 position 值过小（< 1.0）或过大（> 1000000.0）时，重新分配所有卡片的 position：
```rust
for (index, card) in cards.iter_mut().enumerate() {
    card.position = (index + 1) as f64 * 1000.0;
}
```

### Matrix 协议实现

#### 更新卡片所属 Space

使用 Matrix Space 的父子关系：

1. 从旧 Space 移除：
   - 删除旧 Space 的 `m.space.child` 事件（state_key = card_room_id）
   - 删除 Card Room 的 `m.space.parent` 事件（state_key = old_space_id）

2. 添加到新 Space：
   - 在新 Space 创建 `m.space.child` 事件（state_key = card_room_id）
   - 在 Card Room 创建 `m.space.parent` 事件（state_key = new_space_id）

3. 更新卡片元数据：
   - 更新卡片的 `space_id` 和 `position` 字段
   - 通过 Timeline Message 保存新的元数据

#### 备用方案（自定义事件）

如果服务器不完全支持 Space 关系，使用自定义 State Event：

```json
// 在 Space Room 中
{
  "type": "m.kanban.cards",
  "state_key": "",
  "content": {
    "card_ids": ["!card1:server", "!card2:server", "!card3:server"]
  }
}
```

### UI 实现

#### 卡片组件（CardItem）

添加拖拽事件处理：

```rust
impl Widget for CardItem {
    fn handle_event(&mut self, cx: &mut Cx, event: &Event, scope: &mut Scope) {
        match event {
            Event::FingerDown(e) => {
                // 记录按下位置
                self.drag_start_pos = Some(e.abs);
            }
            Event::FingerMove(e) => {
                if let Some(start_pos) = self.drag_start_pos {
                    let distance = (e.abs - start_pos).length();
                    if distance > 5.0 && !self.is_dragging {
                        // 开始拖拽
                        cx.action(KanbanActions::StartDragCard {
                            card_id: self.card_id.clone(),
                            space_id: self.space_id.clone(),
                            position: self.position,
                            start_pos,
                        });
                        self.is_dragging = true;
                    }
                }
                
                if self.is_dragging {
                    // 更新拖拽位置
                    cx.action(KanbanActions::MoveDragCard {
                        current_pos: e.abs,
                    });
                }
            }
            Event::FingerUp(_) => {
                if self.is_dragging {
                    // 结束拖拽
                    cx.action(KanbanActions::DropCard {
                        card_id: self.card_id.clone(),
                        target_space_id: self.target_space_id.clone(),
                        target_position: self.target_position,
                    });
                    self.is_dragging = false;
                }
                self.drag_start_pos = None;
            }
        }
    }
}
```

#### 拖拽预览层

在 `Space` 组件中渲染拖拽预览：

```rust
// 在 draw_walk 中
if let Some(drag_state) = &app_state.kanban_state.drag_state {
    // 绘制半透明的拖拽预览
    self.draw_drag_preview(cx, drag_state);
}
```

#### 放置目标高亮

检测鼠标位置并高亮目标 Space：

```rust
fn detect_drop_target(&self, pos: DVec2, spaces: &[KanbanList]) -> Option<OwnedRoomId> {
    for space in spaces {
        if space.rect.contains(pos) {
            return Some(space.id.clone());
        }
    }
    None
}
```

### 性能优化

#### 1. 节流（Throttling）

拖拽移动事件频率很高，需要节流：

```rust
// 最多每 16ms 处理一次移动事件（60 FPS）
if now - last_move_time < 16 {
    return;
}
```

#### 2. 延迟同步

拖拽过程中不立即同步到 Matrix，只在放置时同步：

```rust
// 拖拽移动：仅更新本地状态
KanbanActions::MoveDragCard { .. } => {
    // 不发送 Matrix 请求
}

// 拖拽放置：同步到 Matrix
KanbanActions::DropCard { .. } => {
    submit_async_request(MatrixRequest::MoveCard { .. });
}
```

#### 3. 乐观更新

先更新本地 UI，再同步到服务器：

```rust
// 1. 立即更新本地状态
state.cards.get_mut(&card_id).unwrap().space_id = target_space_id;

// 2. 异步同步到服务器
submit_async_request(MatrixRequest::MoveCard { .. });

// 3. 如果失败，回滚本地状态
KanbanActions::MoveCardFailed { card_id, original_space_id } => {
    state.cards.get_mut(&card_id).unwrap().space_id = original_space_id;
}
```

## 边界情况处理

### 1. 拖拽到同一位置

如果目标位置与源位置相同，取消操作：

```rust
if source_space_id == target_space_id && 
   (source_position - target_position).abs() < 0.1 {
    // 取消操作，不发送请求
    return;
}
```

### 2. 拖拽到空列表

空列表的第一张卡片 position 设为 1000.0：

```rust
if target_space_cards.is_empty() {
    new_position = 1000.0;
}
```

### 3. 网络失败处理

如果 Matrix 请求失败，回滚本地状态并提示用户：

```rust
KanbanActions::MoveCardFailed { card_id, error } => {
    // 回滚状态
    self.rollback_card_move(card_id);
    
    // 显示错误提示
    enqueue_popup_notification(PopupItem {
        message: format!("移动卡片失败: {}", error),
        kind: PopupKind::Error,
        auto_dismissal_duration: Some(3000),
    });
}
```

### 4. 并发冲突

如果多个用户同时移动同一张卡片：

- 使用 Matrix 的事件时间戳作为冲突解决依据
- 后发生的操作覆盖先发生的操作
- 通过 Sliding Sync 自动同步最新状态

## 用户体验优化

### 视觉反馈

1. 拖拽开始：
   - 原卡片变为半透明（opacity: 0.5）
   - 显示拖拽预览（跟随鼠标）
   - 鼠标光标变为"抓取"状态

2. 拖拽移动：
   - 目标 Space 边框高亮（蓝色，2px）
   - 显示插入位置指示线（虚线）
   - 其他卡片自动让出空间（动画过渡）

3. 拖拽放置：
   - 卡片平滑移动到目标位置（动画）
   - 目标 Space 取消高亮
   - 恢复鼠标光标

4. 拖拽取消：
   - 卡片弹回原位置（动画）
   - 恢复所有视觉状态

### 动画效果

使用 CSS 过渡实现平滑动画：

```rust
// 卡片移动动画
draw_bg: {
    transition: {
        position: { duration: 0.3, ease: EaseInOut }
    }
}
```

### 触摸设备支持

- 长按 500ms 后启动拖拽（避免与滚动冲突）
- 拖拽时禁用页面滚动
- 提供触觉反馈（震动）

## 实现阶段

### Phase 1: 基础拖拽（MVP）

- 实现拖拽状态管理
- 实现卡片在同一 Space 内重新排序
- 实现基本的视觉反馈

### Phase 2: 跨 Space 移动

- 实现卡片在不同 Space 之间移动
- 更新 Matrix Space 关系
- 实现乐观更新和错误回滚

### Phase 3: 体验优化

- 添加动画效果
- 优化拖拽预览
- 添加触摸设备支持

### Phase 4: 高级功能

- 批量拖拽（选中多张卡片）
- 拖拽到归档区域
- 拖拽历史记录

## 测试要点

### 功能测试

1. 单张卡片拖拽
2. 跨 Space 移动
3. 拖拽到空列表
4. 拖拽取消
5. 网络失败恢复

### 性能测试

1. 大量卡片时的拖拽性能
2. 拖拽动画流畅度
3. 内存占用

### 兼容性测试

1. 桌面端（鼠标）
2. 移动端（触摸）
3. 不同浏览器/平台

## 参考资料

- Makepad 拖拽事件文档
- Matrix Spaces 规范
- Trello 拖拽交互设计
