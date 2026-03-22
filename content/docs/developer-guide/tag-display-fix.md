---
title: "标签显示和删除功能修复"
weight: 11
---

# 标签显示和删除功能修复

## 问题描述

标签功能存在以下问题：

1. 标签显示为 ID 格式（`tag_1774186137_688c8d24`）而不是用户输入的名称
2. 使用简化的 Label 显示标签，没有删除按钮
3. 尝试使用 PortalList 实现删除功能时，标签不显示在 UI 上

## 修复方案

### 1. 自动过滤旧格式标签 ID

在 `tag_section.rs` 的 `draw_walk` 方法中添加过滤逻辑：

```rust
// 过滤掉旧格式的标签 ID（格式：tag_数字_哈希）
card.tags.iter()
    .filter(|tag| {
        let is_old_format = tag.starts_with("tag_") && 
            tag.matches('_').count() >= 2;
        
        if is_old_format {
            log!("TagSection: 过滤掉旧格式标签 ID: {}", tag);
        }
        
        !is_old_format
    })
    .cloned()
    .collect()
```

检测规则：
- 以 `tag_` 开头
- 包含至少 2 个下划线（格式：`tag_时间戳_哈希`）

### 2. 使用 PortalList 实现删除功能

参考 `todo_section.rs` 的实现模式，使用 PortalList 为每个标签创建独立的 `TagItem` 组件。

#### TagItem 组件定义

```rust
TagItem = {{TagItem}} {
    width: Fit,
    height: Fit,
    
    <View> {
        width: Fit,
        height: Fit,
        flow: Right,
        spacing: 5,
        align: {y: 0.5},
        padding: {top: 8, bottom: 8, left: 16, right: 16},
        margin: {right: 8, bottom: 8},
        draw_bg: {
            color: #0079BF  // 蓝色背景
            border_radius: 3.0
        }

        tag_text = <Label> {
            width: Fit,
            height: Fit,
            text: "标签",
            draw_text: {
                color: #FFFFFF,
                text_style: <THEME_FONT_REGULAR>{font_size: 14}
            }
        }

        remove_btn = <Button> {
            width: 22,
            height: 22,
            margin: {left: 6},
            text: "×",
            draw_bg: {
                color: #00000000  // 透明背景
            }
            draw_text: {
                color: #FFFFFF,
                text_style: <THEME_FONT_BOLD>{font_size: 20}
            }
        }
    }
}
```

关键点：
- 使用 `<View>` 包装内容，设置背景色和圆角
- `flow: Right` 横向排列标签文本和删除按钮
- `border_radius: 3.0` 设置圆角

#### PortalList 配置

```rust
tag_list = <PortalList> {
    width: Fill,
    height: Fit,
    flow: Right,  // 横向排列标签
    spacing: 5,
    wrap: Wrap,   // 自动换行

    TagItem = <TagItem> {}
}
```

关键点：
- `flow: Right` 横向排列标签（而不是 `Down` 纵向）
- `wrap: Wrap` 当一行放不下时自动换行
- `height: Fit` 根据内容自动调整高度

#### 渲染逻辑

```rust
fn draw_walk(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) -> DrawStep {
    // 获取并过滤标签
    let tag_names: Vec<String> = /* ... 过滤逻辑 ... */;
    
    // 设置空状态提示
    if tag_names.is_empty() {
        self.view.label(ids!(empty_label)).set_visible(cx, true);
    } else {
        self.view.label(ids!(empty_label)).set_visible(cx, false);
    }

    // 使用 PortalList 渲染标签
    while let Some(item) = self.view.draw_walk(cx, scope, walk).step() {
        if let Some(mut list) = item.as_portal_list().borrow_mut() {
            list.set_item_range(cx, 0, tag_names.len());

            while let Some(tag_idx) = list.next_visible_item(cx) {
                if tag_idx >= tag_names.len() {
                    continue;
                }

                let tag_item_widget = list.item(cx, tag_idx, live_id!(TagItem));
                let tag_name = &tag_names[tag_idx];
                
                // 设置标签文本
                tag_item_widget.label(ids!(tag_text)).set_text(cx, tag_name);
                
                // 传递数据给 TagItem
                let tag_item_ref = tag_item_widget.as_tag_item();
                if let Some(mut tag_item) = tag_item_ref.borrow_mut() {
                    tag_item.tag_text = tag_name.clone();
                    tag_item.card_id = self.card_id.clone();
                }

                tag_item_widget.draw_all(cx, scope);
            }
        }
    }

    DrawStep::done()
}
```

关键点：
- 使用 `while let Some(item) = self.view.draw_walk(cx, scope, walk).step()` 循环
- 调用 `list.set_item_range(cx, 0, tag_names.len())` 设置项目数量
- 使用 `list.next_visible_item(cx)` 遍历每个可见项目
- 调用 `tag_item_widget.draw_all(cx, scope)` 绘制每个标签
- 返回 `DrawStep::done()` 而不是 `self.view.draw_walk(cx, scope, walk)`

### 3. 删除按钮事件处理

在 `TagItem` 的 `handle_event` 方法中：

```rust
fn handle_event(&mut self, cx: &mut Cx, event: &Event, scope: &mut Scope) {
    self.view.handle_event(cx, event, scope);
    
    if let Event::Actions(actions) = event {
        if self.view.button(ids!(remove_btn)).clicked(actions) {
            if let Some(card_id) = &self.card_id {
                log!("TagItem: 删除标签 '{}'", self.tag_text);
                cx.action(crate::kanban::KanbanActions::RemoveTag {
                    card_id: card_id.clone(),
                    tag: self.tag_text.clone(),
                });
            }
        }
    }
}
```

## 常见问题

### 问题：PortalList 不显示标签

可能原因：
1. `TagItem` 没有使用 `<View>` 包装，导致背景色不显示
2. PortalList 的 `flow` 设置为 `Down` 而不是 `Right`
3. 没有调用 `tag_item_widget.draw_all(cx, scope)`
4. `draw_walk` 方法返回了 `self.view.draw_walk(cx, scope, walk)` 而不是 `DrawStep::done()`

解决方法：
- 使用 `<View>` 包装 TagItem 的内容
- 设置 `flow: Right, wrap: Wrap` 让标签横向排列并自动换行
- 在渲染循环中调用 `draw_all`
- 返回 `DrawStep::done()`

### 问题：标签显示为 ID 格式

原因：旧数据中标签被保存为生成的 ID。

解决方法：在 `draw_walk` 中自动过滤掉旧格式的标签 ID。

### 问题：删除按钮不响应

可能原因：
1. 没有在 `TagItem` 中保存 `card_id` 和 `tag_text`
2. 没有在渲染时传递这些数据

解决方法：
- 在 `TagItem` 结构体中添加 `card_id` 和 `tag_text` 字段
- 在渲染循环中通过 `borrow_mut()` 设置这些字段

## 测试验证

1. 打开卡片详情页面
2. 验证旧格式标签（`tag_数字_哈希`）被自动隐藏
3. 验证新格式标签正常显示为蓝色卡片
4. 点击标签右侧的 × 按钮，验证删除功能
5. 添加新标签，验证显示正确

## 参考文件

- `src/kanban/components/tag_section.rs` - 标签显示和删除实现
- `src/kanban/components/todo_section.rs` - PortalList 参考实现
- `src/kanban/state/kanban_actions.rs` - RemoveTag action 定义
- `src/app.rs` - RemoveTag action 处理逻辑
