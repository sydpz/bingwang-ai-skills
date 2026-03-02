# Docker Migration Skill

将 OpenClaw agent 从本地迁移到 Docker 容器的 skill。

## 用途

将现有的 OpenClaw agent 配置迁移到 Docker 容器中运行。

## 前置条件

1. Docker 已安装并运行
2. 已创建配置目录
3. 已配置消息应用 (app_id, app_secret)

## 迁移步骤

### 1. 创建容器配置目录

```bash
mkdir -p ~/openclaw-config/openclaw-{container_name}
```

### 2. 复制配置到容器目录

```bash
# 复制现有配置
cp -r ~/.openclaw/* ~/openclaw-config/openclaw-{container_name}/

# 设置权限（容器内用户是 openclaw）
chmod -R 777 ~/openclaw-config/openclaw-{container_name}/
```

### 3. 更新配置

更新配置文件 `~/openclaw-config/openclaw-{container_name}/openclaw.json`：

```json
{
  "models": {
    "providers": {
      "{provider_name}": {
        "baseUrl": "https://api.xxx.com/anthropic",
        "api": "anthropic-messages",
        "models": [
          {
            "id": "{model_id}",
            "name": "{model_name}"
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "{provider_name}/{model_id}"
      },
      "workspace": "/home/openclaw/.openclaw/workspace"
    }
  },
  "bindings": [
    {
      "agentId": "defaults",
      "match": {
        "channel": "{channel_name}",
        "accountId": "{account_id}"
      }
    }
  ],
  "channels": {
    "{channel_name}": {
      "enabled": true,
      "accounts": {
        "{account_id}": {
          "appId": "{app_id}",
          "appSecret": "{app_secret}"
        }
      }
    }
  },
  "auth": {
    "profiles": {
      "{provider_name}:default": {
        "provider": "{provider_name}",
        "mode": "api_key"
      }
    }
  }
}
```

> **注意**: 请将 `{xxx}` 替换为实际配置值

### 4. 更新 docker-compose.yml

添加服务配置：

```yaml
  {container_name}:
    image: openclaw-agent:latest
    container_name: openclaw-{container_name}
    volumes:
      - ~/openclaw-config/openclaw-{container_name}:/home/openclaw/.openclaw
      - ./team-config:/openclaw/team-config:ro
    environment:
      - AGENT_ROLE=member
      - MEMBER_NAME={member_name}
      - OPENCLAW_ALLOW_UNCONFIGURED=true
      - {PROVIDER}_API_KEY=${PROVIDER}_API_KEY
    restart: unless-stopped
    networks:
      - openclaw-network
```

### 5. 重新构建并启动

```bash
cd ~/openclaw-team

# 重新构建镜像
docker build -f Dockerfile.openclaw-agent -t openclaw-agent:latest .

# 启动容器
docker compose up -d {container_name}

# 启动 Gateway
docker exec -d openclaw-{container_name} openclaw gateway --allow-unconfigured
```

### 6. 添加用户到白名单

如果需要特定用户才能对话：

```bash
# 在容器内执行
docker exec openclaw-{container_name} openclaw config set channels.{channel_name}.accounts.{account_id}.groupAllowFrom '["ou_user1", "ou_user2"]'
docker exec openclaw-{container_name} openclaw config set channels.{channel_name}.accounts.{account_id}.dmPolicy "open"

# 重启 Gateway
docker restart openclaw-{container_name}
```

## 验证

检查容器运行状态：
```bash
docker ps | grep {container_name}
```

检查 Gateway 日志：
```bash
docker logs openclaw-{container_name}
```

测试消息发送。

## 常见问题

### 权限问题

如果遇到权限错误：
```bash
chmod -R 777 ~/openclaw-config/openclaw-{container_name}/
```

### systemctl 不可用

Docker 容器内不需要 systemctl，确保 Dockerfile 有：
```dockerfile
ENV OPENCLAW_SKIP_SYSTEMD_CHECK=true
CMD ["tail", "-f", "/dev/null"]
```

### 端口映射

如需外部访问，添加端口映射：
```yaml
ports:
  - "18790:18789"
```

## 相关文件

- 部署目录：`~/openclaw-team/`
- 统一镜像：`openclaw-agent:latest`
- 配置文件：`~/openclaw-config/{容器名}/openclaw.json`
