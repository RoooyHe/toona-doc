---
title: 清除应用缓存
weight: 110
---

## 概述

Toona 使用本地文件系统存储应用数据、Matrix 会话、媒体缓存等。清除缓存可以解决登录问题、数据不一致等情况。

## 缓存位置

Toona 使用 `robius_directories` 库确定应用数据目录，位置因操作系统而异：

### Windows

```
C:\Users\<用户名>\AppData\Roaming\robius\robrix\
```

或使用环境变量：
```
%APPDATA%\robius\robrix\
```

### macOS

```
~/Library/Application Support/org.robius.robrix/
```

### Linux

```
~/.local/share/robrix/
```

## 缓存内容

应用数据目录包含以下内容：

### 1. Matrix 会话数据

每个登录用户都有独立的目录：
```
<用户ID>/
  persistent_state/
    matrix_sdk_state.json    # Matrix SDK 状态
    matrix_sdk_crypto.sqlite # 加密密钥数据库
  db_<时间戳>/               # Matrix 客户端数据库
```

### 2. 应用状态

```
latest_user_id.txt          # 最近登录的用户 ID
latest_app_state.json       # 应用状态（选中的房间等）
window_geom_state.json      # 窗口位置和大小
```

### 3. TSP 钱包（如果启用）

```
tsp_state.json              # TSP 状态
wallets/                    # TSP 钱包文件
```

### 4. Kanban 缓存

```
kanban_cache.json           # Kanban 数据缓存
```

## 清除缓存方法

### 方法 1：完全清除（推荐）

删除整个应用数据目录，清除所有数据和会话：

Windows PowerShell：
```powershell
Remove-Item -Recurse -Force "$env:APPDATA\robius\robrix"
```

Windows CMD：
```cmd
rmdir /s /q "%APPDATA%\robius\robrix"
```

macOS/Linux：
```bash
rm -rf ~/Library/Application\ Support/org.robius.robrix/  # macOS
rm -rf ~/.local/share/robrix/                              # Linux
```

### 方法 2：仅清除特定用户会话

如果只想清除某个用户的登录会话：

1. 找到用户 ID 对应的目录（例如 `@username_server.com`）
2. 删除该目录

Windows PowerShell：
```powershell
Remove-Item -Recurse -Force "$env:APPDATA\robius\robrix\@username_server.com"
```

### 方法 3：仅清除 Kanban 缓存

如果只想清除 Kanban 数据缓存：

Windows PowerShell：
```powershell
Remove-Item "$env:APPDATA\robius\robrix\kanban_cache.json"
```

### 方法 4：通过应用内退出登录

在 Toona 中退出登录会自动清除当前用户的会话数据（但不会删除其他用户的数据）。

## 清除后的影响

清除缓存后：

1. 需要重新登录所有账户
2. 本地消息历史会被清除（但服务器上的消息不受影响）
3. 加密密钥会被清除（可能需要重新验证设备）
4. 应用设置恢复默认值
5. Kanban 数据需要从服务器重新加载

## 注意事项

1. 清除缓存前建议先退出 Toona 应用
2. 如果使用端到端加密，清除缓存后可能无法解密旧消息
3. 清除缓存不会影响服务器上的数据
4. 如果只是想切换服务器，不需要清除缓存，直接退出登录即可

## 故障排查

### 找不到缓存目录

在 Toona 启动日志中查找 `app_data_dir` 输出，会显示实际使用的目录路径。

### 删除失败（权限问题）

Windows：以管理员身份运行 PowerShell

macOS/Linux：使用 `sudo` 命令（通常不需要）

### 删除后仍有问题

1. 确认应用已完全关闭
2. 检查是否有多个 Toona 实例在运行
3. 重启计算机后再试

## 开发调试

开发时如果需要频繁清除缓存，可以创建清理脚本：

Windows PowerShell 脚本（`clear-toona-cache.ps1`）：
```powershell
$cachePath = "$env:APPDATA\robius\robrix"
if (Test-Path $cachePath) {
    Remove-Item -Recurse -Force $cachePath
    Write-Host "Cache cleared: $cachePath" -ForegroundColor Green
} else {
    Write-Host "Cache directory not found: $cachePath" -ForegroundColor Yellow
}
```

使用：
```powershell
.\clear-toona-cache.ps1
```
