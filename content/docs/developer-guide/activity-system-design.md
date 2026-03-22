---
title: "Room 活动记录系统设计"
weight: 15
---

# Room 活动记录系统设计

## 概述

活动记录（Activity）系统用于追踪和显示卡片（Card）的所有变更历史，包括评论、状态变更、标签操作、待办事项变更等。本文档描述活动记录系统的完整设计方案。

## 设计目标

1. 完整记录卡片的所有操作历史
2. 支持多种活动类型（评论、状态变更、标签操作等）
3. 实时同步活动记录到所有客户端
4. 提供清晰的时间线视图
5. 支持富文本评论和 Markdown 格式
6. 性能优化：分页加载、缓存机制

## 数据模型

### ActivityType（活动类型）

```rust
pub enum ActivityType {
    Comment,              // 评论
    StatusChange,         // 状态变更
    TagAdded,            // 添加标签
    TagRemoved,          // 移除标签
    TodoAdded,           // 添加待办事项
    TodoCompleted,       // 完成待办事项
    TodoUncompleted,     // 取消完成待办事项
    EndTimeSet,          // 设置截止时间
    EndTimeRemoved,      // 移除截止时间
    DescriptionChanged,  // 修改描述
    TitleChanged,        // 修改标题
}
```

### CardActivity（卡片活动）

```rust
pub struct CardActivity {
    /// 活动ID（Timeline Event ID）
    pub id: String,
    
    /// 活动类型
    pub activity_type: ActivityType,
    
    /// 活动文本内容
    pub text: String,
    
    /// 活动元数据（JSON格式，存储额外信息）
    pub metadata: Option<serde_json::Value>,
    
    /// 创建时间（Unix timestamp 秒）
    pub created_at: u64,
    
    /// 创建者用户ID
    pub user_id: String,
}
```

### 元数据格式

不同活动类型的元数据格式：

```json
// StatusChange
{
  "old_status": "pending",
  "new_status": "completed"
}

// TagAdded / TagRemoved
{
  "tag_id": "tag_xxx",
  "tag_name": "紧急"
}

// TodoAdded / TodoCompleted / TodoUncompleted
{
  "todo_id": "todo_xxx",
  "todo_text": "完成设计文档"
}

// EndTimeSet
{
  "end_time": 1774282320
}

// TitleChanged / DescriptionChanged
{
  "old_value": "旧内容",
  "new_value": "新内容"
}
```

## Matrix 存储方案

### 方案选择

活动记录使用 Matrix Timeline Messages 存储，而不是 State Events。原因：

1. Timeline 天然支持时间顺序
2. 支持分页加载历史记录
3. 不会因为状态更新而丢失历史
4. 符合 Matrix 协议的消息语义

### 消息类型

使用自定义消息类型：`m.kanban.card.activity`

消息格式：
```json
{
  "msgtype": "m.kanban.card.activity",
  "body": "用户评论内容 或 系统生成的活动描述",
  "activity_type": "comment",
  "metadata": {
    // 活动类型特定的元数据
  }
}
```

### 评论消息

评论使用标准的 `m.room.message` 类型，`msgtype` 为 `m.text`：

```json
{
  "msgtype": "m.text",
  "body": "这是一条评论",
  "format": "org.matrix.custom.html",
  "formatted_body": "<p>这是一条评论</p>"
}
```

### 系统活动消息

系统自动生成的活动（状态变更、标签操作等）使用 `m.kanban.card.activity`：

```json
{
  "msgtype": "m.kanban.card.activity",
  "body": "将状态从 '未完成' 改为 '已完成'",
  "activity_type": "status_change",
  "metadata": {
    "old_status": "pending",
    "new_status": "completed"
  }
}
```

## 功能实现

### 1. 加载活动记录

使用 Matrix SDK 的 Timeline API 加载消息：

```rust
pub async fn load_activities(
    &self,
    card_id: &RoomId,
    limit: Option<usize>,
) -> Result<Vec<CardActivity>> {
    let room = self.client.get_room(card_id)
        .context("Card room not found")?;
    
    // 使用 Timeline API 加载消息
    let timeline = room.timeline().await?;
    let items = timeline.items().await;
    
    let mut activities = Vec::new();
    
    for item in items.iter().take(limit.unwrap_or(50)) {
        if let Some(activity) = self.parse_activity_from_timeline_item(item).await {
            activities.push(activity);
        }
    }
    
    // 按时间倒序排列（最新的在前）
    activities.sort_by(|a, b| b.created_at.cmp(&a.created_at));
    
    Ok(activities)
}
```

### 2. 解析 Timeline Item

```rust
async fn parse_activity_from_timeline_item(
    &self,
    item: &TimelineItem,
) -> Option<CardActivity> {
    match item {
        TimelineItem::Event(event) => {
            match event.content() {
                // 解析评论消息
                TimelineItemContent::Message(msg) => {
                    if let MessageType::Text(text_msg) = msg.msgtype() {
                        return Some(CardActivity {
                            id: event.event_id()?.to_string(),
                            activity_type: ActivityType::Comment,
                            text: text_msg.body.clone(),
                            metadata: None,
                            created_at: event.timestamp().as_secs(),
                            user_id: event.sender().to_string(),
                        });
                    }
                }
                
                // 解析系统活动消息
                TimelineItemContent::UnableToDecrypt(_) => {
                    // 处理加密消息
                }
                
                _ => {}
            }
        }
        _ => {}
    }
    
    None
}
```

### 3. 添加评论

```rust
pub async fn add_comment(
    &self,
    card_id: &RoomId,
    text: String,
) -> Result<()> {
    let room = self.client.get_room(card_id)
        .context("Card room not found")?;
    
    // 发送文本消息
    let content = RoomMessageEventContent::text_plain(text);
    room.send(content).await?;
    
    Ok(())
}
```

### 4. 记录系统活动

在执行操作时自动记录活动：

```rust
pub async fn save_card_metadata(&self, card: &KanbanCard) -> Result<()> {
    // 保存卡片元数据
    // ...
    
    // 记录状态变更活动
    if card.status != old_status {
        self.add_system_activity(
            card.id.as_ref(),
            ActivityType::StatusChange,
            format!("将状态从 '{}' 改为 '{}'", 
                old_status.display_name(), 
                card.status.display_name()
            ),
            Some(serde_json::json!({
                "old_status": old_status,
                "new_status": card.status,
            })),
        ).await?;
    }
    
    Ok(())
}

async fn add_system_activity(
    &self,
    card_id: &RoomId,
    activity_type: ActivityType,
    text: String,
    metadata: Option<serde_json::Value>,
) -> Result<()> {
    let room = self.client.get_room(card_id)
        .context("Card room not found")?;
    
    // 构建系统活动消息
    let content = serde_json::json!({
        "msgtype": "m.kanban.card.activity",
        "body": text,
        "activity_type": activity_type,
        "metadata": metadata,
    });
    
    let raw_content = serde_json::value::to_raw_value(&content)?;
    room.send_raw(raw_content).await?;
    
    Ok(())
}
```

## UI 组件设计

### ActiveSection 组件

当前实现：
- 使用 PortalList 显示活动列表
- 支持滚动查看历史记录
- 底部有评论输入框

改进方案：

1. 活动项分组显示
   - 按日期分组（今天、昨天、本周、更早）
   - 每组显示日期标题

2. 活动项样式优化
   - 评论：显示用户头像、用户名、时间、内容
   - 系统活动：使用图标 + 简洁描述
   - 不同活动类型使用不同颜色标识

3. 交互功能
   - 评论支持编辑和删除（仅自己的评论）
   - 评论支持 @ 提及用户
   - 评论支持 Markdown 格式
   - 点击活动项可以展开详情

4. 性能优化
   - 分页加载（初始加载 50 条）
   - 滚动到顶部时自动加载更多
   - 虚拟滚动优化长列表

### UI 布局

```
┌─────────────────────────────────────┐
│ 活动记录                      [刷新] │
├─────────────────────────────────────┤
│                                     │
│ 今天                                │
│ ┌─────────────────────────────────┐ │
│ │ 👤 张三  5分钟前                │ │
│ │ 这个功能需要优先处理            │ │
│ └─────────────────────────────────┘ │
│                                     │
│ ┌─────────────────────────────────┐ │
│ │ 🔄 李四 将状态改为 "进行中"     │ │
│ │    10分钟前                     │ │
│ └─────────────────────────────────┘ │
│                                     │
│ 昨天                                │
│ ┌─────────────────────────────────┐ │
│ │ 🏷️ 王五 添加标签 "紧急"         │ │
│ │    昨天 15:30                   │ │
│ └─────────────────────────────────┘ │
│                                     │
│ [加载更多...]                       │
│                                     │
├─────────────────────────────────────┤
│ 添加评论                            │
│ ┌─────────────────────────────────┐ │
│ │                                 │ │
│ │ 输入评论内容...                 │ │
│ │                                 │ │
│ └─────────────────────────────────┘ │
│                          [发送]     │
└─────────────────────────────────────┘
```

## 实现步骤

### Phase 1: 完善数据加载

1. 实现 `load_activities` 方法
   - 使用 Matrix SDK Timeline API
   - 解析 Timeline Items
   - 过滤和转换为 CardActivity

2. 实现 `parse_activity_from_timeline_item` 方法
   - 识别评论消息（m.room.message）
   - 识别系统活动消息（m.kanban.card.activity）
   - 提取用户信息和时间戳

3. 添加分页支持
   - 支持 limit 参数
   - 实现"加载更多"功能

### Phase 2: 系统活动自动记录

在以下操作时自动记录活动：

1. 状态变更（SetCardStatus）
   - 记录旧状态和新状态
   - 生成描述文本

2. 标签操作（AddTagToCard, RemoveTagFromCard）
   - 记录标签名称
   - 生成描述文本

3. 待办事项操作（AddTodo, ToggleTodo）
   - 记录待办事项内容
   - 生成描述文本

4. 截止时间操作（SetEndTime）
   - 记录时间值
   - 生成描述文本

5. 标题和描述变更（SaveCardMetadata）
   - 记录旧值和新值
   - 生成描述文本

### Phase 3: UI 组件优化

1. 活动项组件改进
   - 添加用户头像显示
   - 区分评论和系统活动的样式
   - 添加活动类型图标

2. 时间显示优化
   - 相对时间（刚刚、5分钟前、昨天）
   - 绝对时间（鼠标悬停显示完整时间）

3. 分组显示
   - 按日期分组（今天、昨天、本周、更早）
   - 每组显示日期标题

4. 交互功能
   - 评论编辑和删除
   - 点击展开/折叠详情
   - 复制评论内容

### Phase 4: 性能优化

1. 分页加载
   - 初始加载 50 条
   - 滚动到顶部加载更多
   - 显示加载状态

2. 缓存机制
   - 缓存已加载的活动记录
   - 增量更新新活动

3. 虚拟滚动
   - 只渲染可见的活动项
   - 优化长列表性能

## Matrix 协议集成

### Timeline API 使用

```rust
// 获取 Timeline
let timeline = room.timeline().await?;

// 订阅 Timeline 更新
let (items, stream) = timeline.subscribe().await;

// 处理新消息
while let Some(diff) = stream.next().await {
    match diff {
        VectorDiff::Append { values } => {
            // 处理新消息
        }
        VectorDiff::Insert { index, value } => {
            // 处理插入的消息
        }
        _ => {}
    }
}
```

### 消息过滤

只处理以下类型的消息：
1. `m.room.message` 且 `msgtype` 为 `m.text`（评论）
2. `m.kanban.card.activity`（系统活动）

忽略其他消息类型（如卡片元数据消息）。

### 加密支持

活动记录支持端到端加密：
- 评论消息自动加密（如果房间启用加密）
- 系统活动消息也支持加密
- 解密失败时显示占位符

## 用户体验设计

### 活动类型显示

不同活动类型使用不同的图标和颜色：

| 活动类型 | 图标 | 颜色 | 示例文本 |
|---------|------|------|---------|
| Comment | 💬 | #2C3E50 | 用户评论内容 |
| StatusChange | 🔄 | #3498DB | 将状态改为 "已完成" |
| TagAdded | 🏷️ | #27AE60 | 添加标签 "紧急" |
| TagRemoved | 🏷️ | #E74C3C | 移除标签 "紧急" |
| TodoAdded | ✅ | #27AE60 | 添加待办 "完成设计" |
| TodoCompleted | ✅ | #27AE60 | 完成待办 "完成设计" |
| EndTimeSet | ⏰ | #F39C12 | 设置截止时间为 2024-03-25 |
| DescriptionChanged | 📝 | #95A5A6 | 修改了描述 |
| TitleChanged | 📝 | #95A5A6 | 修改了标题 |

注意：实际实现中使用 SVG 图标，不使用 emoji。

### 时间显示规则

- 1分钟内：刚刚
- 1小时内：X分钟前
- 24小时内：X小时前
- 7天内：X天前
- 更早：显示具体日期（YYYY-MM-DD HH:MM）

### 空状态

当没有活动记录时显示：
```
暂无活动记录
成为第一个评论的人吧
```

## 技术要点

### Timeline API 注意事项

1. Timeline 需要异步初始化
2. Timeline Items 是响应式的（使用 eyeball-im）
3. 需要处理加密消息的解密
4. 需要处理消息编辑和删除

### 性能考虑

1. 避免一次性加载所有历史记录
2. 使用虚拟滚动优化长列表
3. 缓存已加载的活动记录
4. 增量更新新活动

### 错误处理

1. Timeline 加载失败：显示错误提示，提供重试按钮
2. 消息解析失败：跳过该消息，记录日志
3. 加密消息解密失败：显示占位符
4. 网络错误：显示离线状态，支持本地缓存

## 测试计划

### 单元测试

1. 活动类型解析测试
2. 时间格式化测试
3. 元数据序列化/反序列化测试

### 集成测试

1. 评论发送和接收测试
2. 系统活动自动记录测试
3. 分页加载测试
4. 加密消息测试

### UI 测试

1. 活动列表显示测试
2. 分组显示测试
3. 滚动加载测试
4. 评论输入和发送测试

## 后续优化方向

1. 支持富文本评论（Markdown、链接、图片）
2. 支持 @ 提及用户
3. 支持评论回复（线程）
4. 支持评论点赞
5. 支持活动筛选（只看评论、只看系统活动）
6. 支持活动搜索
7. 支持导出活动记录

## 参考资料

- Matrix SDK Timeline API 文档
- matrix-sdk-ui crate 文档
- Makepad PortalList 使用指南
- 现有实现：src/kanban/components/active_section.rs
