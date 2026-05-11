# 分布式集群部署指南

本文档说明如何以主从集群模式部署 Sub2API。

## 架构概览

```
              ┌─────────────────────────────────────┐
              │            负载均衡器                │
              │  - HTTP: 轮询到从节点                │
              │  - WebSocket: 会话保持              │
              └─────────────────────────────────────┘
                    │                    │
         ┌──────────┴───┐          ┌─────┴──────────┐
         │   从节点 1    │          │    从节点 2    │
         │   (流量)      │          │    (流量)      │
         └──────────────┘          └────────────────┘
         
              ┌─────────────────────────────────────┐
              │              主节点                  │
              │    (后台任务，不接收流量)             │
              └─────────────────────────────────────┘
              
              ┌─────────────────────────────────────┐
              │         Redis + PostgreSQL          │
              │            (共享存储)                │
              └─────────────────────────────────────┘
```

### 节点角色

| 角色 | 后台任务 | API 流量 | 说明 |
|------|---------|----------|------|
| **主节点 (Master)** | 运行 | 不接收 | 执行定时任务、Token 刷新、监控等 |
| **从节点 (Slave)** | 不运行 | 接收 | 处理 API 请求、WebSocket 连接 |
| **单机 (Standalone)** | 运行 | 接收 | 单实例模式（默认） |

## 配置说明

在 `config.yaml` 中添加以下配置：

### 主节点配置

```yaml
cluster:
  enabled: true
  role: master
  instance_id: "master-01"  # 可选，留空自动生成
  heartbeat_interval_seconds: 5
  heartbeat_ttl_seconds: 15
  instance_registry_key: "cluster:instances"
```

### 从节点配置

```yaml
cluster:
  enabled: true
  role: slave
  instance_id: "slave-01"  # 可选，留空自动生成
  heartbeat_interval_seconds: 5
  heartbeat_ttl_seconds: 15
  instance_registry_key: "cluster:instances"
```

### 单机模式（默认）

```yaml
cluster:
  enabled: false
```

## Docker Compose 快速启动

```bash
cd deploy
docker-compose -f docker-compose.dev.distributed.yml up -d
```

启动内容：
- 1 个主节点（端口 8081）
- 1 个从节点（端口 8082）
- Nginx 负载均衡（端口 8080）
- Redis
- PostgreSQL

访问地址：`http://localhost:8080`

## 手动部署

### 前置条件

- Redis（所有节点共享）
- PostgreSQL（所有节点共享）
- Nginx 或其他负载均衡器

### 步骤 1：配置主节点

```yaml
# 主节点 config.yaml
cluster:
  enabled: true
  role: master
  instance_id: "master-01"

redis:
  addr: "redis:6379"

database:
  host: "postgres"
  # ... 其他数据库配置
```

### 步骤 2：配置从节点

```yaml
# 从节点 config.yaml
cluster:
  enabled: true
  role: slave
  instance_id: "slave-01"

redis:
  addr: "redis:6379"

database:
  host: "postgres"
  # ... 其他数据库配置
```

### 步骤 3：配置负载均衡器

参考 `nginx/nginx.conf` 完整示例。关键配置：

```nginx
# 只将流量转发到从节点
upstream backend_api {
    server slave-1:8080;
    server slave-2:8080;
    # 不要包含主节点
}

# WebSocket 需要会话保持
upstream backend_ws {
    ip_hash;  # 按客户端 IP 保持会话
    server slave-1:8080;
    server slave-2:8080;
}
```

## 扩展

### 添加更多从节点

1. 创建新的配置文件，使用唯一的 `instance_id`
2. 启动新的从节点实例
3. 在负载均衡器的 upstream 中添加新节点

### 高可用性说明

当前实现使用**静态主从模式**：
- 主从角色由配置文件决定
- 如果主节点故障，后台任务停止，需要重启主节点恢复
- 从节点可独立继续处理流量

**后续升级**：可升级为自动选举模式以实现更高可用性。

## 监控

### 集群状态 API

```bash
# 获取集群状态
curl http://localhost:8080/api/admin/cluster/status

# 响应示例：
{
  "current_instance": {
    "id": "slave-01",
    "hostname": "slave-1",
    "role": "slave",
    "status": "healthy"
  },
  "instances": [...],
  "master_id": "master-01",
  "cluster_mode": "master-slave",
  "cluster_healthy": true,
  "total_instances": 2,
  "master_count": 1,
  "slave_count": 1
}
```

### 健康检查端点

- 主节点：`http://master:8080/health`
- 从节点：`http://slave-n:8080/health`

## 故障排查

### 后台任务未运行

- 检查主节点是否正在运行
- 确认 `cluster.role` 设置为 `master`
- 查看主节点日志排查错误

### WebSocket 连接断开

- 确保负载均衡器为 WebSocket 启用了会话保持
- 检查 Nginx 的 `ip_hash` 或基于 cookie 的会话保持配置

### 实例未出现在集群状态中

- 确认 Redis 连接正常
- 检查所有节点的 `instance_registry_key` 配置一致
- 确认心跳配置正确

## 常见问题

**Q: 可以配置多个主节点吗？**  
A: 不可以，只能配置一个主节点。多个主节点会导致任务重复执行。

**Q: 主节点故障会怎样？**  
A: 后台任务停止，但从节点继续处理 API 流量。重启主节点即可恢复任务。

**Q: 主节点变更时需要重启从节点吗？**  
A: 不需要，从节点是独立的，不需要感知主节点的存在。
