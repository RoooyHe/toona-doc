---
title: "分页请求去重修复"
weight: 15
---

# 分页请求去重修复

## 问题描述

在 `room_screen.rs` 的 `draw_walk` 方法中，每一帧都会检查时间线是否填满视口。如果没有填满，就会提交一个向后分页请求。但这导致了一个问题：

1. 分页请求被提交到后台任务
2. 在请求完成前，下一帧又会检查 `is_filling_viewport()`
3. 由于列表仍然没有填满视口，又会提交新的分页请求
4. 这导致大量的分页请求堆积，产生警告：`Another pagination request is already happening, returning early`

## 根本原因

缺少一个标志来跟踪分页请求是否正在进行中。`TimelineUiState` 有 `fully_paginated` 字段来表示是否已经完全分页，但没有字段来表示是否有分页请求正在进行中。

## 解决方案

在 `TimelineUiState` 中添加两个字段来跟踪分页状态：

### 1. 添加状态字段

在 `src/home/room_screen.rs` 的 `TimelineUiState` 结构体中添加：

```rust
/// Whether a pagination request is currently in progress.
/// This prevents multiple pagination requests from being submitted in rapid succession.
pagination_in_progress: bool,

/// Whether the last pagination request failed with an error.
/// This prevents automatic pagination from retrying immediately after an error.
pagination_failed: bool,
```

### 2. 初始化字段

在创建新的 `TimelineUiState` 时初始化为 `false`：

```rust
let tl_state = TimelineUiState {
    // ... 其他字段
    pagination_in_progress: false,
    pagination_failed: false,
    // ... 其他字段
};
```

### 3. 更新状态

在处理分页相关的 `TimelineUpdate` 时更新这些字段：

- `PaginationRunning`: 设置 `pagination_in_progress = true`, `pagination_failed = false`
- `PaginationIdle`: 设置 `pagination_in_progress = false`
- `PaginationError`: 设置 `pagination_in_progress = false`, `pagination_failed = true`

### 4. 检查状态并立即设置标志

在 `draw_walk` 中提交分页请求前检查这些标志，并在提交请求时立即设置 `pagination_in_progress` 标志：

```rust
if !tl_state.fully_paginated 
    && !tl_state.pagination_in_progress 
    && !tl_state.pagination_failed 
    && !list.is_filling_viewport() 
{
    // 立即设置标志，防止在同一帧内提交多个请求
    tl_state.pagination_in_progress = true;
    // 提交分页请求
    submit_async_request(MatrixRequest::PaginateRoomTimeline { ... });
}
```

这种立即设置标志的方式确保了即使在 `PaginationRunning` 更新到达之前，也不会提交重复的请求。

## 修改的文件

- `src/home/room_screen.rs`
  - 添加 `pagination_in_progress` 字段到 `TimelineUiState`
  - 初始化该字段
  - 在 `PaginationRunning` 处理中设置为 `true`
  - 在 `PaginationIdle` 处理中设置为 `false`
  - 在 `PaginationError` 处理中设置为 `false`
  - 在 `draw_walk` 中检查该字段

## 效果

修复后，分页请求将被正确去重和错误处理：

1. 第一个分页请求被提交，`pagination_in_progress` 设置为 `true`，`pagination_failed` 设置为 `false`
2. 在请求完成前，`draw_walk` 会检查 `pagination_in_progress` 并跳过提交新请求
3. 请求成功完成后，`pagination_in_progress` 设置为 `false`
4. 如果请求失败，`pagination_in_progress` 设置为 `false`，`pagination_failed` 设置为 `true`
5. 当 `pagination_failed` 为 `true` 时，自动分页将被禁用，避免无限重试循环

这样可以：
- 消除大量的重复分页请求警告
- 避免在分页错误后无限重试
- 提高应用性能和稳定性

## 分页失败处理

当分页请求失败时（例如服务器返回 400 错误），`pagination_failed` 标志会阻止自动重试。用户可以通过以下方式手动触发分页：

- 滚动到时间线顶部
- 刷新房间
- 重新进入房间

这种设计避免了在服务器端问题（如无效的批次令牌）导致的无限重试循环。

## 常见问题

### "invalid batch token" 错误

如果看到类似以下的错误：

```
Error sending backwards pagination request: 
EventCacheError(BackpaginationError(Http(Api(Server(ClientApi(Error { 
    status_code: 400, 
    body: Standard(StandardErrorBody { 
        kind: InvalidParam, 
        message: "invalid batch token: must start with 's' or 't'" 
    }) 
}))))))
```

这是 Matrix 服务器端的问题，不是客户端代码的问题。可能的原因：

1. 服务器实现有问题（特别是测试服务器）
2. 服务器数据库中的批次令牌已损坏
3. 服务器版本与客户端 SDK 不兼容

解决方法：

1. **清除本地缓存**：删除应用的本地数据目录
2. **等待服务器修复**：如果是服务器问题，需要服务器管理员修复
3. **使用其他服务器**：尝试连接到官方 Matrix 服务器（matrix.org）

我们的修复确保了即使遇到这种错误，应用也不会陷入无限重试循环。

## 相关代码位置

- 状态字段定义: `src/home/room_screen.rs` 第 2935 行左右
- 初始化: `src/home/room_screen.rs` 第 2425 行左右
- 状态更新: `src/home/room_screen.rs` 第 1540-1580 行
- 检查: `src/home/room_screen.rs` 第 1280 行左右
