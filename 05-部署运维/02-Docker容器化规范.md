# IndustryAgent Docker 容器化规范

| 项目 | 内容 |
|------|------|
| 文档编号 | IA-OPS-003 |
| 文档名称 | Docker 容器化规范 |
| 版本 | V1.0 |
| 密级 | 内部 |
| 编制 | 李吉 |
| 审核 | （空） |
| 批准 | （空） |
| 日期 | 2026-10-15 |

## 修订记录

| 版本 | 日期 | 修订人 | 修订说明 |
|------|------|--------|----------|
| V1.0 | 2026-10-15 | 李吉 | 初版创建：镜像规范、Dockerfile×3、完整 docker-compose.yml、运行时安全、仓库管理、override、故障排查手册 |

---

## 1. 概述

本文档规定 IndustryAgent 全部服务与中间件的容器镜像构建规范、Dockerfile 标准、docker-compose.yml 生产级配置、容器运行时安全基线、镜像仓库管理与本地开发覆盖机制。适用于开发、Staging、生产（k3s）全部环境；K8s 环境镜像规范与本规范一致，仅编排方式不同。

---

## 2. 镜像规范

### 2.1 基础镜像选择表

| 用途 | 基础镜像 | 构建阶段镜像 | 说明 |
|------|----------|--------------|------|
| industryagent-server | node:20-alpine | node:20-alpine | Egg.js 4.0 运行时，alpine 体积小、攻击面小 |
| industryagent-agent | python:3.11-slim | python:3.11-slim | FastAPI + LangGraph，slim 平衡体积与 wheel 兼容性 |
| industryagent-web | nginx:alpine | node:20-alpine | React 18 构建产物由 nginx 托管 |
| PostgreSQL | timescale/timescaledb:2.17.2-pg16 | — | 内置 TimescaleDB，装 pgvector 需执行 CREATE EXTENSION |
| Redis | redis:7.4 | — | 官方镜像 |
| Kafka | bitnami/kafka:3.7 | — | 含 KRaft 兼容工具 |
| Neo4j | neo4j:5.20-community | — | 社区版满足沙盒 |
| MinIO | minio/minio:RELEASE.2025-01-20T14-49-04Z | — | 固定 RELEASE tag，禁用 latest |
| Keycloak | quay.io/keycloak/keycloak:25.0 | — | — |
| OPA | openpolicyagent/opa:0.70.0-static-debug 或 0.70.0 | — | 静态二进制镜像 |
| Prometheus / Grafana | prom/prometheus:v2.54.1 / grafana/grafana:11.2.0 | — | 固定版本 tag |

**通用规则**：所有镜像必须固定精确版本 tag，禁止 `latest`；基础镜像每季度评审升级一次，配合 IA-OPS-002 的 trivy 扫描门禁。

### 2.2 镜像构建四原则

1. **多阶段构建**：构建工具（npm ci、pip、vite）不出现在运行镜像；
2. **非 root 运行**：运行阶段必须有专用低权限用户；
3. **层缓存优化**：依赖清单（package*.json / requirements.txt）与源码分层 COPY，依赖层优先复用；
4. **HEALTHCHECK 必备**：每个运行镜像内嵌健康检查，供 compose depends_on condition 与 K8s 探针对齐。

### 2.3 .dockerignore（三服务统一模板）

```
.git
.gitignore
node_modules
__pycache__
*.pyc
dist
build
coverage
tests
__tests__
.env
.env.*
*.md
.vscode
.idea
```

---

## 3. 完整 Dockerfile 示例

### 3.1 industryagent-server（Egg.js 4.0，services/server/Dockerfile）

```dockerfile
# syntax=docker/dockerfile:1.7
# ============ 构建阶段 ============
FROM node:20-alpine AS builder
WORKDIR /app
ENV NODE_ENV=production
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run clean && npm run build

# ============ 运行阶段 ============
FROM node:20-alpine AS runner
LABEL org.opencontainers.image.title="industryagent-server" \
      org.opencontainers.image.vendor="IndustryAgent" \
      org.opencontainers.image.source="https://git.industryagent.internal/platform/industryagent"
WORKDIR /app
RUN apk add --no-cache curl tzdata \
 && addgroup -g 1001 -S nodeapp \
 && adduser -u 1001 -S nodeapp -G nodeapp \
 && mkdir -p /app/logs && chown -R nodeapp:nodeapp /app
ENV NODE_ENV=production \
    TZ=Asia/Shanghai \
    EGGIN_SERVER_PORT=7001
COPY --from=builder --chown=nodeapp:nodeapp /app/package.json /app/package-lock.json ./
COPY --from=builder --chown=nodeapp:nodeapp /app/node_modules ./node_modules
COPY --from=builder --chown=nodeapp:nodeapp /app/app ./app
COPY --from=builder --chown=nodeapp:nodeapp /app/config ./config
COPY --from=builder --chown=nodeapp:nodeapp /app/app.js ./app.js
COPY --from=builder --chown=nodeapp:nodeapp /app/migrations ./migrations
USER nodeapp
EXPOSE 7001
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD curl -sf http://127.0.0.1:7001/healthz || exit 1
CMD ["npm", "run", "start"]
```

### 3.2 industryagent-agent（Python FastAPI + LangGraph，services/agent/Dockerfile）

```dockerfile
# syntax=docker/dockerfile:1.7
# ============ 构建阶段：编译依赖 wheel ============
FROM python:3.11-slim AS builder
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends gcc g++ \
 && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

# ============ 运行阶段 ============
FROM python:3.11-slim AS runner
WORKDIR /app
LABEL org.opencontainers.image.title="industryagent-agent" \
      org.opencontainers.image.vendor="IndustryAgent"
RUN apt-get update && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/* \
 && groupadd -g 1001 agentapp && useradd -u 1001 -g agentapp -m -s /bin/bash agentapp \
 && mkdir -p /app/.cache && chown -R agentapp:agentapp /app
COPY --from=builder /wheels /wheels
COPY requirements.txt .
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
 && rm -rf /wheels
COPY --chown=agentapp:agentapp . .
USER agentapp
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=3 \
  CMD curl -sf http://127.0.0.1:8000/healthz || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

### 3.3 industryagent-web（React 18 + Nginx，services/web/Dockerfile）

```dockerfile
# syntax=docker/dockerfile:1.7
# ============ 构建阶段 ============
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# ============ 运行阶段 ============
FROM nginx:alpine AS runner
LABEL org.opencontainers.image.title="industryagent-web" \
      org.opencontainers.image.vendor="IndustryAgent"
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget -q --spider http://127.0.0.1:80/ || exit 1
CMD ["nginx", "-g", "daemon off;"]
```

---

## 4. docker-compose.yml 完整配置（deploy/docker-compose.yml）

> 生产要点：全部镜像固定 tag（`IMAGE_TAG` 由 .env 注入）、健康检查 depends_on condition、资源限制、日志轮转、frontend/backend/monitoring 三网络划分。数据卷规划见 §4.2。

```yaml
name: ${COMPOSE_PROJECT_NAME:-industryagent}

x-logging: &default-logging
  driver: json-file
  options:
    max-size: "50m"
    max-file: "5"

x-restart: &default-restart
  restart: unless-stopped

services:
  # ================= 应用层 =================
  industryagent-web:
    image: ghcr.io/industryagent/industryagent-web:${IMAGE_TAG:-v1.0.0}
    container_name: ia-web
    <<: *default-restart
    logging: *default-logging
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./certs:/etc/nginx/certs:ro
    networks: [frontend]
    depends_on:
      industryagent-server:
        condition: service_healthy
      keycloak:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 512M

  industryagent-server:
    image: ghcr.io/industryagent/industryagent-server:${IMAGE_TAG:-v1.0.0}
    container_name: ia-server
    <<: *default-restart
    logging: *default-logging
    environment:
      NODE_ENV: ${NODE_ENV:-production}
      SERVER_PORT: "7001"
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      KAFKA_BROKERS: ${KAFKA_BROKERS}
      NEO4J_URI: ${NEO4J_URI}
      MINIO_ENDPOINT: ${MINIO_ENDPOINT}
      KEYCLOAK_URL: ${KEYCLOAK_URL}
      KEYCLOAK_REALM: ${KEYCLOAK_REALM}
      KEYCLOAK_CLIENT_ID: ${KEYCLOAK_CLIENT_ID}
      OPA_URL: ${OPA_URL}
      LOG_LEVEL: ${LOG_LEVEL:-info}
      AUDIT_LOG_RETENTION_DAYS: ${AUDIT_LOG_RETENTION_DAYS:-400}
      TZ: ${TZ:-Asia/Shanghai}
    networks: [frontend, backend]
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      kafka:
        condition: service_healthy
      neo4j:
        condition: service_healthy
      minio:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 2G
        reservations:
          memory: 1G

  industryagent-agent:
    image: ghcr.io/industryagent/industryagent-agent:${IMAGE_TAG:-v1.0.0}
    container_name: ia-agent
    <<: *default-restart
    logging: *default-logging
    environment:
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      KAFKA_BROKERS: ${KAFKA_BROKERS}
      KAFKA_TOPIC_SENSOR: ${KAFKA_TOPIC_SENSOR}
      KAFKA_TOPIC_ALERT: ${KAFKA_TOPIC_ALERT}
      NEO4J_URI: ${NEO4J_URI}
      NEO4J_AUTH: ${NEO4J_AUTH}
      MINIO_ENDPOINT: ${MINIO_ENDPOINT}
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
      MINIO_BUCKET_WAFER: ${MINIO_BUCKET_WAFER}
      MINIO_BUCKET_DOCS: ${MINIO_BUCKET_DOCS}
      LLM_BASE_URL: ${LLM_BASE_URL}
      LLM_MODEL: ${LLM_MODEL}
      LOG_LEVEL: ${LOG_LEVEL:-info}
      TZ: ${TZ:-Asia/Shanghai}
    networks: [frontend, backend]
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      kafka:
        condition: service_healthy
      neo4j:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: "4.0"
          memory: 6G
        reservations:
          memory: 2G

  # ================= 数据层 =================
  postgres:
    image: timescale/timescaledb:2.17.2-pg16
    container_name: ia-postgres
    <<: *default-restart
    logging: *default-logging
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      PGDATA: ${PGDATA:-/var/lib/postgresql/data}
      TIMESCALEDB_TELEMETRY: "off"
    command:
      - postgres
      - -c
      - shared_buffers=2GB
      - -c
      - max_connections=200
      - -c
      - work_mem=16MB
      - -c
      - log_min_duration_statement=500
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
    deploy:
      resources:
        limits:
          cpus: "4.0"
          memory: 8G

  redis:
    image: redis:7.4
    container_name: ia-redis
    <<: *default-restart
    logging: *default-logging
    command:
      - redis-server
      - --requirepass
      - ${REDIS_PASSWORD}
      - --maxmemory
      - 1.5gb
      - --maxmemory-policy
      - allkeys-lru
      - --appendonly
      - "yes"
    volumes:
      - redis_data:/data
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "redis-cli -a ${REDIS_PASSWORD} ping | grep PONG"]
      interval: 10s
      timeout: 5s
      retries: 10
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 2G

  zookeeper:
    image: bitnami/zookeeper:3.9
    container_name: ia-zookeeper
    <<: *default-restart
    logging: *default-logging
    environment:
      ALLOW_ANONYMOUS_LOGIN: "yes"
    volumes:
      - zk_data:/bitnami/zookeeper
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "echo ruok | nc localhost 2181 | grep imok"]
      interval: 10s
      timeout: 5s
      retries: 10
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 1G

  kafka:
    image: bitnami/kafka:3.7
    container_name: ia-kafka
    <<: *default-restart
    logging: *default-logging
    environment:
      KAFKA_CFG_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092
      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CFG_NUM_PARTITIONS: "6"
      KAFKA_CFG_LOG_RETENTION_HOURS: "168"
      KAFKA_CFG_LOG_SEGMENT_BYTES: "536870912"
      KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE: "false"
    volumes:
      - kafka_data:/bitnami/kafka
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "kafka-topics.sh --bootstrap-server localhost:9092 --list >/dev/null 2>&1"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 30s
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 4G

  neo4j:
    image: neo4j:5.20-community
    container_name: ia-neo4j
    <<: *default-restart
    logging: *default-logging
    environment:
      NEO4J_AUTH: ${NEO4J_AUTH}
      NEO4J_server_memory_heap_max__size: 2G
      NEO4J_server_memory_pagecache_size: 1G
    volumes:
      - neo4j_data:/data
      - neo4j_logs:/logs
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "wget -q --spider http://localhost:7474 || exit 1"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 60s
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 4G

  minio:
    image: minio/minio:RELEASE.2025-01-20T14-49-04Z
    container_name: ia-minio
    <<: *default-restart
    logging: *default-logging
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    volumes:
      - minio_data:/data
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:9000/minio/health/live || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 10
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 2G

  # ================= 安全层 =================
  keycloak:
    image: quay.io/keycloak/keycloak:25.0
    container_name: ia-keycloak
    <<: *default-restart
    logging: *default-logging
    command: start-dev --http-port 8080
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: ${KEYCLOAK_ADMIN}
      KC_BOOTSTRAP_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/${POSTGRES_DB}
      KC_DB_USERNAME: ${POSTGRES_USER}
      KC_DB_PASSWORD: ${POSTGRES_PASSWORD}
      KC_HEALTH_ENABLED: "true"
    volumes:
      - ./keycloak/import:/opt/keycloak/data/import:ro
    networks: [frontend, backend]
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:8080/health/ready || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 20
      start_period: 90s
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1536M

  opa:
    image: openpolicyagent/opa:0.70.0-static
    container_name: ia-opa
    <<: *default-restart
    logging: *default-logging
    command: run --server --addr :8181 --log-level warn /policies
    volumes:
      - ./opa/policies:/policies:ro
    networks: [backend]
    healthcheck:
      test: ["CMD-SHELL", "wget -q --spider http://localhost:8181/health || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M

  # ================= 观测层 =================
  prometheus:
    image: prom/prometheus:v2.54.1
    container_name: ia-prometheus
    <<: *default-restart
    logging: *default-logging
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=30d
      - --web.enable-lifecycle
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml:ro
      - prometheus_data:/prometheus
    networks: [monitoring, backend]
    healthcheck:
      test: ["CMD-SHELL", "wget -q --spider http://localhost:9090/-/healthy || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 4G

  grafana:
    image: grafana/grafana:11.2.0
    container_name: ia-grafana
    <<: *default-restart
    logging: *default-logging
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD:-Ch@ngeMe_Grafana_2026}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_SERVER_ROOT_URL: https://staging.industryagent.internal/grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    networks: [monitoring, frontend]
    healthcheck:
      test: ["CMD-SHELL", "wget -q --spider http://localhost:3000/api/health || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M

  node-exporter:
    image: prom/node-exporter:v1.8.2
    container_name: ia-node-exporter
    <<: *default-restart
    logging: *default-logging
    command:
      - --path.rootfs=/host
      - --collector.systemd
    pid: host
    volumes:
      - /:/host:ro,rslave
    networks: [monitoring]
    deploy:
      resources:
        limits:
          cpus: "0.2"
          memory: 256M

networks:
  frontend:
    name: ia-frontend
    driver: bridge
  backend:
    name: ia-backend
    internal: false   # 沙盒需访问内网 LLM 推理服务；生产建议 internal: true + 白名单出口
  monitoring:
    name: ia-monitoring
    driver: bridge

volumes:
  pg_data:
    name: ia-pg_data
  redis_data:
    name: ia-redis_data
  zk_data:
    name: ia-zk_data
  kafka_data:
    name: ia-kafka_data
  neo4j_data:
    name: ia-neo4j_data
  neo4j_logs:
    name: ia-neo4j_logs
  minio_data:
    name: ia-minio_data
  prometheus_data:
    name: ia-prometheus_data
  grafana_data:
    name: ia-grafana_data
```

### 4.2 数据卷规划表

| 卷名 | 挂载点 | 预估容量 | 备份策略（见 IA-OPS-006） |
|------|--------|----------|---------------------------|
| ia-pg_data | /var/lib/postgresql/data | 200G | 每日 pg_dump 全量 + 每小时 WAL 归档 |
| ia-redis_data | /data | 5G | 仅 AOF 快照（可重建，无跨日备份） |
| ia-kafka_data | /bitnami/kafka | 100G | 7 天保留，不备份（数据源可重放） |
| ia-zk_data | /bitnami/zookeeper | 1G | 元数据随主题重建 |
| ia-neo4j_data | /data | 50G | 每周 neo4j-admin dump |
| ia-minio_data | /data | 100G | 缺陷图/文档 rsync 每日增量；审计桶版本化 |
| ia-prometheus_data | /prometheus | 50G | 不备份（30 天滚动） |
| ia-grafana_data | /var/lib/grafana | 1G | 每月导出 dashboard JSON 入库 |

---

## 5. 容器运行时安全

### 5.1 基线规则

| 规则 | 要求 |
|------|------|
| 非 root 运行 | 应用容器以 1001 专用用户运行；K8s 中补充 `runAsNonRoot: true` |
| 只读文件系统 | 应用容器加 `read_only: true`，仅可写目录显式挂载 tmpfs（如 /tmp） |
| cap_drop | 所有应用容器 `cap_drop: [ALL]`，按需个别加回（web 需 NET_BIND_SERVICE） |
| 禁止 privileged | 任何容器不得 privileged（node-exporter 使用 host pid + ro 挂载替代） |
| 资源限制 | 全部容器必须配置 cpus/memory limits，防止单容器耗尽宿主机 |
| 日志轮转 | 全局 json-file，max-size 50m × max-file 5 |
| 密钥注入 | 通过 .env/Secrets 注入，禁止写入镜像层；镜像内不得存在 .env |

### 5.2 加固 compose 片段（应用层示例）

```yaml
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    # web 如需绑定 80/443：
    # cap_add: [NET_BIND_SERVICE]
```

### 5.3 违规自检命令

```bash
# 检查是否有 privileged 容器（输出应为空）
docker ps -q | xargs docker inspect --format '{{.Name}} Privileged={{.HostConfig.Privileged}}' | grep true

# 检查 root 运行的应用容器（server/agent/web 应输出 nodeapp/agentapp/nginx）
docker ps --format '{{.Names}}' | grep '^ia-' | while read c; do
  docker inspect --format "$c User={{.Config.User}}" "$c"
done
```

---

## 6. 镜像仓库管理

### 6.1 命名空间与标签约定

| 项 | 约定 |
|----|------|
| 命名空间 | ghcr.io/industryagent（或 Harbor 项目 industryagent） |
| 仓库名 | industryagent-server / industryagent-web / industryagent-agent |
| 标签 | git sha 与 semver（v主.次.修），详见 IA-OPS-002 §4.2 |
| latest | 禁止用于部署；仓库策略上不推送 latest |

### 6.2 保留与清理策略

- semver 标签：永久保留；
- git sha 标签：保留 90 天（支持任意历史版本回滚窗口）；
- 未打标镜像（dangling）：7 天后清理。

### 6.3 清理脚本（deploy/scripts/registry-cleanup.sh，cron 每日 03:30）

```bash
#!/usr/bin/env bash
# 清理 ghcr.io/industryagent 下超过 90 天且非 semver 标签的镜像（Harbor 场景用其 API 等价实现）
set -euo pipefail
OWNER="industryagent"
KEEP_DAYS=90
CUTOFF=$(date -d "-${KEEP_DAYS} days" +%s)

for repo in industryagent-server industryagent-web industryagent-agent; do
  # ghcr 清理依赖 ghcr-io 页面/REST，沙盒使用 skopeo 列举：
  skopeo list-tags "docker://ghcr.io/${OWNER}/${repo}" | tail -n +3 | while read -r tag; do
    # 跳过 semver 标签
    if echo "$tag" | grep -qE '^v[0-9]+\.[0-9]+\.[0-9]+.*$'; then continue; fi
    # sha 标签的创建时间由 CI 侧台账（releases.tsv）判定，此处按台账清理：
    CREATED=$(grep -P "^${repo}\t${tag}\t" /data/app/industryagent/deploy/releases.tsv | cut -f3)
    [ -z "$CREATED" ] && continue
    if [ "$(date -d "$CREATED" +%s)" -lt "$CUTOFF" ]; then
      echo "cleanup: ${repo}:${tag}"
      # ghcr 删除需管理员 PAT：gh api -X DELETE /user/packages/container/${repo}/versions/<id>
    fi
  done
done

# 清理本机 dangling 镜像
docker image prune -f --filter "until=168h"
```

---

## 7. 本地开发 compose override 说明

本地开发使用 `docker-compose.override.yml`（compose 默认自动叠加，无需 `-f` 参数），差异点：

| 差异项 | 基础 compose | override（开发） |
|--------|--------------|-------------------|
| 镜像 | ghcr 固定 tag | 本地 build（`build:` 指向服务目录） |
| 资源限制 | 生产限额 | 去除（本地放开） |
| 端口 | 仅 80/443 入口 | 暴露 7001/8000/5432/7474/9001/9090 便于调试 |
| 磁盘 | 完整保留策略 | Kafka 保留 24h、Prometheus 3d |
| Loki | 可选启动 | 默认不启动 |
| 数据卷 | named volume | 支持绑定 ./local-data 便于清库重灌 |

override 关键片段：

```yaml
services:
  industryagent-server:
    build:
      context: ../services/server
    ports:
      - "7001:7001"
    deploy:
      resources:
        limits: {}
  industryagent-agent:
    build:
      context: ../services/agent
    ports:
      - "8000:8000"
    volumes:
      - ../services/agent:/app:rw
    environment:
      LOG_LEVEL: debug
  kafka:
    environment:
      KAFKA_CFG_LOG_RETENTION_HOURS: "24"
  loki:
    profiles: ["full"]
    image: grafana/loki:3.1.1
```

使用方式：

```bash
docker compose up -d                # 自动叠加 override
docker compose --profile full up -d # 需要 Loki 时
docker compose restart industryagent-server
```

---

## 8. 容器故障排查手册

### 8.1 常见问题速查表

| 现象 | 高概率原因 | 排查命令 |
|------|------------|----------|
| 容器反复重启（Restarting） | 进程崩溃 / 健康检查失败 / 依赖未就绪 | `docker logs --tail 200 ia-server`；`docker inspect --format '{{json .State}}' ia-server \| python -m json.tool` |
| 容器起不来直接 Exited | 配置缺失、端口冲突、权限问题 | `docker compose ps -a`；`docker logs ia-server`；`docker inspect ia-server \| grep -A5 ExitCode` |
| 健康检查一直 failing | /healthz 未实现或依赖不可达 | `docker inspect --format '{{json .State.Health}}' ia-agent \| python -m json.tool`；进容器手工执行 HEALTHCHECK 命令 |
| 服务 OOM 被杀（Exit 137） | 内存限制过小 / 内存泄漏 | `docker inspect ia-agent \| grep OOMKilled`；`docker stats --no-stream`；Node: `--max-old-space-size`，Python: 检查模型常驻内存 |
| PG 连接数打满 | max_connections 不够 / 连接泄漏 | `docker exec ia-postgres psql -U industryagent -c "SELECT count(*),state FROM pg_stat_activity GROUP BY state;"` |
| Kafka 消费 Lag 飙升 | 消费者挂了 / 吞吐不足 | `docker exec ia-kafka kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --all-groups` |
| Redis 命中率低 | maxmemory 策略不当 / key 失效 | `docker exec ia-redis redis-cli -a "$REDIS_PASSWORD" info stats \| grep keyspace` |
| Neo4j 拒绝连接 | heap 超内存限制被 OOM | `docker logs ia-neo4j \| grep -i oom`；核对 heap_max 与容器 memory limits |
| 磁盘写满 | 日志/卷未轮转 | `df -h /data`；`du -sh /var/lib/docker/volumes/ia-* \| sort -h` |
| 网络不通（容器间） | 跨网络访问 / DNS 解析失败 | `docker exec ia-server curl -sf http://postgres:5432`（或 `getent hosts postgres`） |

### 8.2 标准排查三板斧

```bash
# 1) 状态总览：看哪些容器非 healthy
docker compose -f deploy/docker-compose.yml ps

# 2) 日志定位：时间范围 + 关键字
docker logs --since 30m ia-server 2>&1 | grep -iE 'error|fatal|unhandled'

# 3) 进入现场（只读文件系统容器加 --no-tty 无 shell 时用 docker cp）
docker exec -it ia-server sh
```

### 8.3 OOM 专项判定流程

1. `docker inspect --format '{{.State.OOMKilled}}' ia-agent` → true 即触发内核 OOM；
2. 对照 compose limits 与 3.1 资源表（agent 6G 峰值来自模型推理，需保留）；
3. Node 侧：确认设置了 `--max-old-space-size=1536`（低于容器 limit）；
4. 临时缓解：`docker compose up -d --force-recreate industryagent-agent`；
5. 若持续复发：抓 heap dump（Node `--inspect` + `node --heapsnapshot-signal`），开事故单按 IA-OPS-005 复盘。

---

## 9. 参考文档

- IA-OPS-001《部署方案》：环境规划与部署步骤
- IA-OPS-002《CI/CD 流水线配置》：镜像 tag 与构建门禁
- IA-OPS-004《监控告警方案》：健康检查与指标采集的衔接
- IA-OPS-006《回滚方案》：数据卷备份与恢复
