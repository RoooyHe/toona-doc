---
title: "标签删除功能（简化方案）"
weight: 12
---

# 标签删除功能（简化方案）

## 背景

由于 Makepad 框架的限制，PortalList 方案在实现标签删除时遇到了布局计算错误（NaN 错误导致程序崩溃）。因此采用了更简单实用的方案。

## 实现方案

### UI 设计

1. 标签显示区域：使用 Label 组件显示所有标签，格式为 "标签: tag1, tag2, tag3"
2. 删除区域：
   - 一个文本输入框，用户输入要删除的标签名称
   - 一个删除按钮
   - 一个提示文本："提示: 输入标签名称后点击删除按钮"

### 代码实现

#### 1. UI 结构 (tag_section.rs)

```rust
// 标签列表容器
tags_container = <View> {
    width: Fill,
    height: Fit,
    flow: Down,
    spacing: 8,
    
    // 标签显示（使用简单的 Label）
    tags_display_label = <Label> {
        width: Fill,
        height: Fit,
        text: "",
        visible: false,
        draw_text: {
            color: #0079BF,
            text_style: <THEME_FONT_REGULAR>{font_size: 14}
            wrap: Word
        }
    }

    // 空状态提示
    empty_label = <Label> {
        width: Fill,
        height: Fit,
        padding: {top: 10, bottom: 10},
        text: "暂无标签",
        visible: true,
        draw_text: {
            color: #x95A5A6,
            text_style: <THEME_FONT_REGULAR>{font_size: 13}
        }
    }
    
    // 删除标签区域（选择要删除的标签）
    remove_tag_container = <View> {
        width: Fill,
        height: Fit,
        flow: Down,
        spacing: 8,
        margin: {top: 8},
        visible: false,
        
        <Label> {
            text: "删除标签:",
            draw_text: {
                color: #x5E6C84,
                text_style: <THEME_FONT_REGULAR>{font_size: 12}
            }
        }
        
        <View> {
            width: Fill,
            height: Fit,
            flow: Right,
            spacing: 10,
            align: {y: 0.5},
            
            remove_tag_input = <TextInput> {
                width: Fill,
                height: 32,
                text: "",
                // ... 样式配置
            }
            
            remove_tag_button = <Button> {
                width: 60,
                height: 32,
                text: "删除",
                draw_bg: {
                    color: #FF6B6B,
                    radius: 3.0,
                }
                // ... 样式配置
            }
        }
        
        <Label> {
            text: "提示: 输入标签名称后点击删除按钮",
            draw_text: {
                color: #x95A5A6,
                text_style: <THEME_FONT_REGULAR>{font_size: 11}
            }
        }
    }
}
```

#### 2. 事件处理

```rust
impl Widget for TagSection {
    fn handle_event(&mut self, cx: &mut Cx, event: &Event, scope: &mut Scope) {
        self.view.handle_event(cx, event, scope);
        
        if let Event::Actions(actions) = event {
            // 处理删除标签按钮
            if self.view.button(ids!(remove_tag_button)).clicked(actions) {
                let tag_name = self.view.text_input(ids!(remove_tag_input)).text();
                
                if !tag_name.trim().is_empty() {
                    if let Some(card_id) = &self.card_id {
                        log!("TagSection: 删除标签 '{}' 从卡片 {}", tag_name.trim(), card_id);
                        
                        cx.action(crate::kanban::KanbanActions::RemoveTag {
                            card_id: card_id.clone(),
                            tag: tag_name.trim().to_string(),
                        });
                        
                        // 清空输入框
                        self.view.text_input(ids!(remove_tag_input)).set_text(cx, "");
                        self.view.redraw(cx);
                    }
                }
            }
        }
    }
}
```

#### 3. 渲染逻辑

```rust
fn draw_walk(&mut self, cx: &mut Cx2d, scope: &mut Scope, walk: Walk) -> DrawStep {
    // 从 AppState 获取标签信息
    let tag_names: Vec<String> = if let Some(app_state) = scope.data.get::<crate::app::AppState>() {
        if let Some(selected_card_id) = &app_state.kanban_state.selected_card_id {
            self.card_id = Some(selected_card_id.clone());
            
            if let Some(card) = app_state.kanban_state.cards.get(selected_card_id) {
                // 过滤掉旧格式的标签 ID（格式：tag_数字_哈希）
                card.tags.iter()
                    .filter(|tag| {
                        let is_old_format = tag.starts_with("tag_") && 
                            tag.matches('_').count() >= 2;
                        !is_old_format
                    })
                    .cloned()
                    .collect()
            } else {
                Vec::new()
            }
        } else {
            Vec::new()
        }
    } else {
        Vec::new()
    };
    
    // 设置可见性和内容
    if tag_names.is_empty() {
        self.view.label(ids!(tags_display_label)).set_visible(cx, false);
        self.view.label(ids!(empty_label)).set_visible(cx, true);
        self.view.view(ids!(remove_tag_container)).set_visible(cx, false);
    } else {
        // 显示标签名称
        let tags_text = format!("标签: {}", tag_names.join(", "));
        self.view.label(ids!(tags_display_label)).set_text(cx, &tags_text);
        self.view.label(ids!(tags_display_label)).set_visible(cx, true);
        self.view.label(ids!(empty_label)).set_visible(cx, false);
        self.view.view(ids!(remove_tag_container)).set_visible(cx, true);
    }

    self.view.draw_walk(cx, scope, walk)
}
```

#### 4. 后端处理 (app.rs)

```rust
KanbanActions::RemoveTag { card_id, tag } => {
    log!("RemoveTag: card_id='{}', tag='{}'", card_id, tag);
    
    // 立即更新内存中的 state
    if let Some(card) = state.cards.get_mut(&card_id) {
        card.tags.retain(|t| t != &tag);
        card.touch();
        log!("Removed tag '{}' in memory immediately", tag);
        
        // 如果模态框打开的是这张卡片，立即重绘
        if state.selected_card_id.as_ref() == Some(&card_id) {
            log!("Forcing immediate modal redraw");
            self.ui.view(ids!(card_detail_modal.modal.content)).redraw(cx);
        }
        self.ui.redraw(cx);
        
        // 异步保存到 Matrix（传递完整的卡片数据）
        if get_client().is_some() {
            let card_clone = card.clone();
            submit_async_request(MatrixRequest::SaveCardMetadata { card: card_clone });
        }
    }
}
```

## 使用方法

1. 打开卡片详情模态框
2. 查看标签区域，可以看到所有标签以逗号分隔的形式显示
3. 在"删除标签"输入框中输入要删除的标签名称（必须完全匹配）
4. 点击"删除"按钮
5. 标签会立即从界面上消失，并同步到 Matrix 服务器

## 注意事项

1. 标签名称必须完全匹配（区分大小写）
2. 输入框会在删除后自动清空
3. 删除操作会立即更新 UI，然后异步保存到服务器
4. 旧格式的标签 ID（格式：tag_数字_哈希）会被自动过滤，不会显示
5. 只有当卡片有标签时，删除区域才会显示

## 优缺点分析

### 优点

1. 实现简单，不依赖复杂的 UI 组件
2. 稳定可靠，不会出现布局计算错误
3. 代码易于维护和理解
4. 性能开销小

### 缺点

1. 用户体验不如每个标签独立删除按钮直观
2. 需要用户手动输入标签名称
3. 容易输入错误（大小写、拼写）

## 未来改进方向

如果 Makepad 框架支持更复杂的动态 UI，可以考虑：

1. 为每个标签添加独立的删除按钮（类似 Trello）
2. 使用下拉选择框选择要删除的标签
3. 支持批量删除标签
4. 添加删除确认对话框
5. 支持标签的拖拽排序

## 相关文件

- `src/kanban/components/tag_section.rs` - 标签 UI 组件
- `src/app.rs` - RemoveTag 动作处理
- `src/kanban/state/kanban_actions.rs` - KanbanActions 定义
- `src/kanban/matrix_adapter.rs` - Matrix 协议适配器
