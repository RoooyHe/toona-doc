---
title: 本地 Matrix 服务器部署
weight: 100
---

## 概述

使用 Docker Compose 在本地部署 Synapse Matrix 服务器，用于开发和测试。

## 前置要求

- Docker Desktop（Windows/macOS）或 Docker Engine（Linux）
- Docker Compose

## 部署步骤

### 1. 创建配置文件

在项目根目录创建 `docker-compose.yml` 和相关配置文件。

### 2. 生成服务器配置

首次运行时需要生成 Synapse 配置文件：

```bash
# 创建数据目录
mkdir synapse-data

# 生成配置
docker run -it --rm \
  -v ./synapse-data:/data \
  -e SYNAPSE_SERVER_NAME=localhost \
  -e SYNAPSE_REPORT_STATS=no \
  matrixdotorg/synapse:latest generate
```

Windows PowerShell 使用：
```powershell
# 创建数据目录
New-Item -ItemType Directory -Force -Path synapse-data

# 生成配置
docker run -it --rm -v ${PWD}/synapse-data:/data -e SYNAPSE_SERVER_NAME=localhost -e SYNAPSE_REPORT_STATS=no matrixdotorg/synapse:latest generate
```

生成后可以编辑 `synapse-data/homeserver.yaml` 启用用户注册：
```yaml
enable_registration: true  # 允许注册新用户
```

### 3. 启动服务

```bash
docker-compose up -d
```

### 4. 创建管理员账户

```bash
docker exec -it synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml -a
```

按提示输入用户名和密码。

### 5. 访问服务器

- Matrix 服务器地址：`http://localhost:8008`
- 在 Toona 登录时使用：
  - 服务器：`http://localhost:8008`
  - 用户名：`@username:localhost`
  - 密码：创建时设置的密码

## 配置说明

### 端口映射

- `8008`：Matrix 客户端 API（HTTP）
- `8448`：Matrix 联邦 API（HTTPS，可选）

### 数据持久化

所有数据存储在 `./synapse-data` 目录中，包括：
- 数据库文件
- 媒体文件
- 配置文件

### 性能优化

默认配置使用 SQLite 数据库，适合开发测试。生产环境建议使用 PostgreSQL。

## 常用命令

### 查看日志

```bash
docker-compose logs -f synapse
```

### 停止服务

```bash
docker-compose down
```

### 重启服务

```bash
docker-compose restart
```

### 清理数据（重新开始）

```bash
docker-compose down -v
rm -rf synapse-data
```

Windows PowerShell：
```powershell
docker-compose down -v
Remove-Item -Recurse -Force synapse-data
```

## 故障排查

### 端口冲突

如果 8008 端口被占用，修改 `docker-compose.yml` 中的端口映射：
```yaml
ports:
  - "8009:8008"  # 使用 8009 端口
```

### 权限问题

Linux 系统可能需要调整 synapse-data 目录权限：
```bash
sudo chown -R 991:991 synapse-data
```

### 连接问题

确保防火墙允许 Docker 容器访问网络。

## 高级配置

### 启用注册

编辑 `synapse-data/homeserver.yaml`：
```yaml
enable_registration: true
enable_registration_without_verification: true
```

重启服务使配置生效。

### 配置 Sliding Sync

Toona 需要 Sliding Sync 支持。Synapse 1.114+ 内置支持，无需额外配置。

### 使用 PostgreSQL

修改 `docker-compose.yml` 添加 PostgreSQL 服务，并更新 Synapse 配置。

## 注意事项

1. 本地服务器仅用于开发测试，不要用于生产环境
2. 默认配置不启用 HTTPS，不要暴露到公网
3. 定期备份 synapse-data 目录
4. localhost 服务器无法与其他 Matrix 服务器联邦
