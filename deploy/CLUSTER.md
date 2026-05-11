# Distributed Cluster Deployment Guide

This guide explains how to deploy Sub2API in a distributed master-slave cluster mode.

## Architecture Overview

```
              ┌─────────────────────────────────────┐
              │          Load Balancer              │
              │  - HTTP: round-robin to slaves      │
              │  - WebSocket: sticky session        │
              └─────────────────────────────────────┘
                    │                    │
         ┌──────────┴───┐          ┌─────┴──────────┐
         │   Slave 1    │          │    Slave 2     │
         │  (traffic)   │          │   (traffic)    │
         └──────────────┘          └────────────────┘
         
              ┌─────────────────────────────────────┐
              │             Master                  │
              │  (background tasks, no traffic)     │
              └─────────────────────────────────────┘
              
              ┌─────────────────────────────────────┐
              │         Redis + PostgreSQL          │
              │         (shared storage)            │
              └─────────────────────────────────────┘
```

### Node Roles

| Role | Background Tasks | API Traffic | Description |
|------|-----------------|-------------|-------------|
| **Master** | Yes | No | Runs scheduled tasks, token refresh, monitoring, etc. |
| **Slave** | No | Yes | Handles API requests, WebSocket connections |
| **Standalone** | Yes | Yes | Single instance mode (default) |

## Configuration

Add the following to your `config.yaml`:

### Master Node

```yaml
cluster:
  enabled: true
  role: master
  instance_id: "master-01"  # Optional, auto-generated if empty
  heartbeat_interval_seconds: 5
  heartbeat_ttl_seconds: 15
  instance_registry_key: "cluster:instances"
```

### Slave Node

```yaml
cluster:
  enabled: true
  role: slave
  instance_id: "slave-01"  # Optional, auto-generated if empty
  heartbeat_interval_seconds: 5
  heartbeat_ttl_seconds: 15
  instance_registry_key: "cluster:instances"
```

### Standalone Mode (Default)

```yaml
cluster:
  enabled: false
```

## Quick Start with Docker Compose

```bash
cd deploy
docker-compose -f docker-compose.dev.distributed.yml up -d
```

This starts:
- 1 Master node (port 8081)
- 1 Slave node (port 8082)
- Nginx load balancer (port 8080)
- Redis
- PostgreSQL

Access the application at `http://localhost:8080`

## Manual Deployment

### Prerequisites

- Redis (shared between all nodes)
- PostgreSQL (shared between all nodes)
- Nginx or other load balancer

### Step 1: Configure Master Node

```yaml
# config.yaml for master
cluster:
  enabled: true
  role: master
  instance_id: "master-01"

redis:
  addr: "redis:6379"

database:
  host: "postgres"
  # ... other database settings
```

### Step 2: Configure Slave Nodes

```yaml
# config.yaml for slave
cluster:
  enabled: true
  role: slave
  instance_id: "slave-01"

redis:
  addr: "redis:6379"

database:
  host: "postgres"
  # ... other database settings
```

### Step 3: Configure Load Balancer

See `nginx/nginx.conf` for a complete example. Key points:

```nginx
# Only route traffic to slave nodes
upstream backend_api {
    server slave-1:8080;
    server slave-2:8080;
    # DO NOT include master here
}

# WebSocket requires sticky session
upstream backend_ws {
    ip_hash;  # Sticky session by client IP
    server slave-1:8080;
    server slave-2:8080;
}
```

## Scaling

### Adding More Slave Nodes

1. Create a new config file with unique `instance_id`
2. Start the new slave instance
3. Add the slave to the load balancer upstream

### High Availability Considerations

This implementation uses **static master-slave mode**:
- Master role is determined by configuration
- If master fails, background tasks stop until master is restarted
- Slaves continue to handle traffic independently

**Future Enhancement**: Can be upgraded to automatic leader election for higher availability.

## Monitoring

### Cluster Status API

```bash
# Get cluster status
curl http://localhost:8080/api/admin/cluster/status

# Response example:
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

### Health Check Endpoints

- Master: `http://master:8080/health`
- Slaves: `http://slave-n:8080/health`

## Troubleshooting

### Background tasks not running

- Check if master node is running
- Verify `cluster.role` is set to `master`
- Check master logs for errors

### WebSocket connections dropping

- Ensure load balancer has sticky session enabled for WebSocket
- Check `ip_hash` or cookie-based sticky session in Nginx

### Instances not appearing in cluster status

- Verify Redis connectivity
- Check `instance_registry_key` is the same across all nodes
- Verify heartbeat settings

## FAQ

**Q: Can I have multiple masters?**  
A: No, only one master should be configured. Multiple masters will cause duplicate task execution.

**Q: What happens if master fails?**  
A: Background tasks stop, but slaves continue handling API traffic. Restart master to resume tasks.

**Q: Do I need to restart slaves when master changes?**  
A: No, slaves are independent and don't need to know about master.
