# IndustryAgent CI/CD 流水线配置

| 项目 | 内容 |
|------|------|
| 文档编号 | IA-OPS-002 |
| 文档名称 | CI/CD 流水线配置 |
| 版本 | V1.0 |
| 密级 | 内部 |
| 编制 | 李吉 |
| 审核 | （空） |
| 批准 | （空） |
| 日期 | 2026-10-15 |

## 修订记录

| 版本 | 日期 | 修订人 | 修订说明 |
|------|------|--------|----------|
| V1.0 | 2026-10-15 | 李吉 | 初版创建：流水线全景、GitHub Actions 完整 workflow、镜像构建规范、质量门禁、部署流水线、密钥与制品管理 |

---

## 1. 概述

本文档定义 IndustryAgent 的 CI/CD 流水线架构、GitHub Actions 完整配置、镜像构建规范、质量门禁策略、部署流水线与密钥管理方案。目标：**每次提交可自动验证、每个版本可追溯、每次发布可回滚**，支撑 99.9% 可用性与数据不出域约束。

- 代码仓库：`platform/industryagent`（monorepo：`services/server`、`services/web`、`services/agent`）
- 镜像仓库：`ghcr.io/industryagent`（生产可切内网 Harbor：`harbor.industryagent.internal/industryagent`）
- 部署演进：Docker Compose（I2~I5）→ Kubernetes（生产形态，k3s 模拟）

---

## 2. 流水线全景图

```
 开发者                GitHub                    ghcr.io/Harbor            Staging(Compose)        生产(k3s)
   │  push/PR             │                            │                         │                     │
   ├────────────────────> │  ┌─── PR 流水线 ────┐      │                         │                     │
   │                      │  │ lint → unit-test │      │                         │                     │
   │                      │  │ → build(不推镜像) │      │                         │                     │
   │                      │  │ → trivy 扫描      │      │                         │                     │
   │                      │  └────────┬─────────┘      │                         │                     │
   │   review approve     │           v 门禁通过方可合并 │                         │                     │
   ├────────────────────> │  ┌─── 主干流水线 ─────────────────┐                   │                     │
   │                      │  │ lint → test → build 镜像      │                   │                     │
   │                      │  │ → 推送 ghcr(sha+semver) ──────┼──> 镜像仓库         │                     │
   │                      │  │ → trivy 镜像扫描(阻断策略)      │                   │                     │
   │                      │  └────────┬─────────────────────┘                   │                     │
   │                      │           v tag v1.x.0 触发                         │                     │
   │                      │  ┌─── 发布流水线 ─────────────────────────────────────┼──> 自动部署 ─┐        │
   │                      │  │ ① 部署 Staging（compose ssh 部署）                 │            │        │
   │                      │  │ ② Staging 冒烟+集成验证                           │            │        │
   │                      │  │ ③ 人工审批 gate（manual approval）───────────────────┼────────────┼──> 手动确认
   │                      │  │ ④ kubectl apply 到生产 k3s                        │            │    ┌───┴───┐
   │                      │  │ ⑤ 生产健康检查 + 回滚判定                          │<───────────┼────┤ 成功✓ │
   │                      │  └───────────────────────────────────────────────────┘            │    │ 失败→回滚
   │                      │                                                                    │    └───────┘
```

三条流水线的职责边界：

| 流水线 | 触发条件 | 产出 | 是否部署 |
|--------|----------|------|----------|
| PR 流水线 | 任意 Pull Request | lint/测试/构建结果、扫描报告（不推镜像） | 否 |
| 主干流水线 | push 到 main | 三服务镜像（tag=git sha）推送至 ghcr | 否（仅出制品） |
| 发布流水线 | 推送 tag `v*.*.*` 或手动 dispatch | Staging 自动部署 + 审批后生产部署 | 是 |

---

## 3. GitHub Actions 完整 Workflow

### 3.1 PR 流水线（.github/workflows/pr-ci.yml）

```yaml
name: PR CI

on:
  pull_request:
    branches: [main]

concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      server: ${{ steps.filter.outputs.server }}
      web: ${{ steps.filter.outputs.web }}
      agent: ${{ steps.filter.outputs.agent }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            server:
              - 'services/server/**'
            web:
              - 'services/web/**'
            agent:
              - 'services/agent/**'

  server-ci:
    needs: changes
    if: needs.changes.outputs.server == 'true'
    runs-on: ubuntu-latest
    services:
      postgres:
        image: timescale/timescaledb:2.17.2-pg16
        env:
          POSTGRES_USER: industryagent
          POSTGRES_PASSWORD: citest
          POSTGRES_DB: industryagent
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready -U industryagent"
          --health-interval 10s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7.4
        ports: ['6379:6379']
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s --health-timeout 5s --health-retries 5
    env:
      DATABASE_URL: postgresql://industryagent:citest@localhost:5432/industryagent
      REDIS_URL: redis://localhost:6379/0
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: services/server/package-lock.json
      - name: Install
        working-directory: services/server
        run: npm ci
      - name: Lint
        working-directory: services/server
        run: npm run lint
      - name: Migrate DB
        working-directory: services/server
        run: npm run migrate:up
      - name: Unit & Integration Test
        working-directory: services/server
        run: npm run test:ci
      - name: Coverage gate (>=80%)
        working-directory: services/server
        run: npm run test:coverage -- --check-coverage lines=80 functions=80 branches=75 statements=80

  web-ci:
    needs: changes
    if: needs.changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: services/web/package-lock.json
      - name: Install
        working-directory: services/web
        run: npm ci
      - name: Lint
        working-directory: services/web
        run: npm run lint
      - name: Build
        working-directory: services/web
        run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: web-dist-${{ github.sha }}
          path: services/web/dist/
          retention-days: 14

  agent-ci:
    needs: changes
    if: needs.changes.outputs.agent == 'true'
    runs-on: ubuntu-latest
    services:
      postgres:
        image: timescale/timescaledb:2.17.2-pg16
        env:
          POSTGRES_USER: industryagent
          POSTGRES_PASSWORD: citest
          POSTGRES_DB: industryagent
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready -U industryagent"
          --health-interval 10s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7.4
        ports: ['6379:6379']
    env:
      DATABASE_URL: postgresql://industryagent:citest@localhost:5432/industryagent
      REDIS_URL: redis://localhost:6379/0
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: pip
          cache-dependency-path: services/agent/requirements*.txt
      - name: Install
        working-directory: services/agent
        run: |
          pip install -r requirements.txt -r requirements-dev.txt
      - name: Lint (ruff)
        working-directory: services/agent
        run: ruff check . && ruff format --check .
      - name: Pytest
        working-directory: services/agent
        run: pytest --cov=app --cov-fail-under=80 --cov-report=xml -v

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Trivy filesystem scan (blocking)
        uses: aquasecurity/trivy-action@0.28.0
        with:
          scan-type: fs
          scan-ref: .
          severity: HIGH,CRITICAL
          exit-code: '1'
          ignore-unfixed: true
          format: table
```

### 3.2 主干流水线（.github/workflows/main-ci.yml，构建并推送镜像）

```yaml
name: Main CI

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  REGISTRY_PREFIX: ghcr.io/${{ github.repository_owner }}
  TAG: ${{ github.sha }}

jobs:
  server-image:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: services/server
          push: true
          tags: |
            ${{ env.REGISTRY_PREFIX }}/industryagent-server:${{ env.TAG }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - name: Trivy image scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: ${{ env.REGISTRY_PREFIX }}/industryagent-server:${{ env.TAG }}
          severity: HIGH,CRITICAL
          exit-code: '1'
          ignore-unfixed: true

  web-image:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: services/web
          push: true
          tags: ${{ env.REGISTRY_PREFIX }}/industryagent-web:${{ env.TAG }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - name: Trivy image scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: ${{ env.REGISTRY_PREFIX }}/industryagent-web:${{ env.TAG }}
          severity: HIGH,CRITICAL
          exit-code: '1'
          ignore-unfixed: true

  agent-image:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: services/agent
          push: true
          tags: ${{ env.REGISTRY_PREFIX }}/industryagent-agent:${{ env.TAG }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - name: Trivy image scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: ${{ env.REGISTRY_PREFIX }}/industryagent-agent:${{ env.TAG }}
          severity: HIGH,CRITICAL
          exit-code: '1'
          ignore-unfixed: true
```

### 3.3 发布流水线（.github/workflows/release.yml）

```yaml
name: Release

on:
  push:
    tags: ['v*.*.*']
  workflow_dispatch:
    inputs:
      version:
        description: '发布版本（如 v1.2.0）'
        required: true
        type: string

permissions:
  contents: write
  packages: write

jobs:
  resolve-version:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.v.outputs.version }}
      image_tag: ${{ steps.v.outputs.image_tag }}
    steps:
      - id: v
        run: |
          if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            echo "version=${{ inputs.version }}" >> $GITHUB_OUTPUT
            echo "image_tag=${{ inputs.version }}" >> $GITHUB_OUTPUT
          else
            echo "version=${GITHUB_REF_NAME}" >> $GITHUB_OUTPUT
            echo "image_tag=${GITHUB_REF_NAME}" >> $GITHUB_OUTPUT
          fi

  deploy-staging:
    needs: resolve-version
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Tag semver release images
        run: |
          echo "${{ secrets.GHCR_TOKEN }}" | docker login ghcr.io -u ${{ secrets.GHCR_USER }} --password-stdin
          for svc in server web agent; do
            docker pull ghcr.io/industryagent/industryagent-${svc}:${{ github.sha }} || true
            docker tag ghcr.io/industryagent/industryagent-${svc}:${{ github.sha }} \
                       ghcr.io/industryagent/industryagent-${svc}:${{ needs.resolve-version.outputs.image_tag }}
            docker push ghcr.io/industryagent/industryagent-${svc}:${{ needs.resolve-version.outputs.image_tag }}
          done
      - name: Deploy to Staging via SSH
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /data/app/industryagent
            git fetch --tags && git checkout ${{ needs.resolve-version.outputs.version }}
            export IMAGE_TAG=${{ needs.resolve-version.outputs.image_tag }}
            docker compose -f deploy/docker-compose.yml pull server web agent
            docker compose -f deploy/docker-compose.yml up -d server web agent
      - name: Staging smoke test
        run: |
          for i in $(seq 1 30); do
            if curl -sfk https://staging.industryagent.internal/api/healthz; then exit 0; fi
            sleep 10
          done
          echo "staging smoke test failed" && exit 1

  production-approval:
    needs: [resolve-version, deploy-staging]
    runs-on: ubuntu-latest
    environment: production      # GitHub Environment 配置 Required Reviewers = 发布审批人
    steps:
      - name: Manual approval gate
        run: echo "生产发布已获批准：${{ needs.resolve-version.outputs.version }}"

  deploy-production:
    needs: [resolve-version, production-approval]
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Setup kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.K3S_KUBECONFIG }}" | base64 -d > ~/.kube/config
          chmod 600 ~/.kube/config
      - name: Deploy to k3s
        run: |
          kubectl -n industryagent-app set image deployment/industryagent-server \
            server=ghcr.io/industryagent/industryagent-server:${{ needs.resolve-version.outputs.image_tag }}
          kubectl -n industryagent-app set image deployment/industryagent-agent \
            agent=ghcr.io/industryagent/industryagent-agent:${{ needs.resolve-version.outputs.image_tag }}
          kubectl -n industryagent-app set image deployment/industryagent-web \
            web=ghcr.io/industryagent/industryagent-web:${{ needs.resolve-version.outputs.image_tag }}
          kubectl -n industryagent-app rollout status deployment/industryagent-server --timeout=300s
          kubectl -n industryagent-app rollout status deployment/industryagent-agent --timeout=300s
          kubectl -n industryagent-app rollout status deployment/industryagent-web --timeout=300s
      - name: Production health check
        run: |
          kubectl -n industryagent-app get pods -o wide
          for i in $(seq 1 30); do
            STATUS=$(curl -sk -o /dev/null -w "%{http_code}" https://industryagent.internal/api/healthz)
            if [ "$STATUS" = "200" ]; then echo "production healthy"; exit 0; fi
            sleep 10
          done
          echo "production health check failed, rollback needed (见 IA-OPS-006)" && exit 1
      - name: Rollback on failure
        if: failure()
        run: |
          kubectl -n industryagent-app rollout undo deployment/industryagent-server
          kubectl -n industryagent-app rollout undo deployment/industryagent-agent
          kubectl -n industryagent-app rollout undo deployment/industryagent-web
```

---

## 4. 镜像构建规范

### 4.1 基础规范

| 规范项 | 要求 |
|--------|------|
| 基础镜像 | server/agent/web 构建镜像统一见 IA-OPS-003（node:20-alpine、python:3.11-slim、nginx:alpine） |
| 多阶段构建 | 构建/运行分离，运行镜像不含编译器与源码 |
| 非 root 运行 | 运行阶段创建专用用户（node / appuser / nginx） |
| 层缓存优化 | 先 COPY 依赖清单再 COPY 源码，依赖层可复用 |
| HEALTHCHECK | 运行镜像必须含 HEALTHCHECK 指令 |
| .dockerignore | 必须排除 node_modules、.git、__pycache__、dist、tests、.env |

### 4.2 镜像 Tag 规则

| Tag 格式 | 用途 | 示例 |
|-----------|------|------|
| `<git-sha>` | 主干每次构建的唯一不可变制品 | `a1b2c3d` |
| `v<semver>` | 正式发布版本（主干流水线产物重新打 tag） | `v1.2.0` |
| `v<semver>-rc.N` | 发布候选（Staging 验证用） | `v1.2.0-rc.1` |

**强制规则：`latest` 标签禁止上生产**。生产部署清单中只允许出现 git-sha 或 semver tag；流水线在部署前校验：

```bash
if echo "$IMAGE_TAG" | grep -qE '^(latest|main)$'; then
  echo "禁止部署 latest/main 标签到任何环境" && exit 1
fi
```

---

## 5. 流水线质量门禁

| 门禁 | 工具/命令 | 通过标准 | 阻断级别 |
|------|-----------|----------|----------|
| Lint | eslint（web/server）、ruff（agent） | 0 error（warning ≤ 10） | 硬阻断 |
| 单元/集成测试 | jest / pytest | 全部通过 | 硬阻断 |
| 覆盖率 | jest --check-coverage / pytest --cov-fail-under | 行覆盖 ≥ 80%（分支 ≥ 75%） | 硬阻断 |
| 镜像漏洞扫描 | trivy（fs 与 image 两种模式） | 无 HIGH/CRITICAL 且可修复漏洞 | 硬阻断（`--ignore-unfixed`） |
| 镜像大小 | docker images 校验 | server ≤ 400MB、agent ≤ 1.2GB、web ≤ 60MB | 软告警（超限评审） |
| 冒烟测试 | curl /healthz 全链路 | 30 次重试内 200 | 硬阻断（Staging） |
| 生产审批 | GitHub Environment Required Reviewers | 审批人 ≥ 1（release manager） | 硬阻断 |

### 5.1 漏洞阻断策略细节

- PR 阶段：源码文件系统扫描，HIGH/CRITICAL 可修复漏洞存在即 fail；
- 主干阶段：镜像扫描，同上；扫描报告以 SARIF 上传至 GitHub Security tab；
- 误报白名单：`trivy-ignore.yaml` 收录，须注明 CVE、理由、复审日期，超过 90 天复审一次。

---

## 6. 部署流水线

### 6.1 Compose 环境（Staging）SSH 部署脚本要点

见 3.3 中 `Deploy to Staging via SSH` 步骤，落地要点：

1. 部署机仅 Staging 跳板机；SSH 使用专用 key（Secrets 注入），禁止口令登录；
2. 部署顺序：**先数据层无变更 → server → agent → web**（入口最后切）；
3. 部署后 30 分钟内观察 `docker compose ps` 健康状态与 Grafana 错误率面板；
4. 失败处置：执行 `docker compose rollback`（详见 IA-OPS-006 L1 应用回滚）。

### 6.2 K8s 生产环境（k3s）

1. 通过 `kubectl set image + rollout status` 滚动更新，`maxUnavailable=0` 保证零中断；
2. `readinessProbe` 未就绪的 Pod 不进 Service，自动保活旧版本；
3. 生产部署必须携带发布工单号，rollout 历史记录版本追溯命令：

```bash
kubectl -n industryagent-app rollout history deployment/industryagent-server
kubectl -n industryagent-app rollout history deployment/industryagent-server --revision=3
```

### 6.3 发布审批（Manual Gate）

- 审批人：release manager + 业务负责人（双人确认，体现在 GitHub Environment `production` 的 Required Reviewers）；
- 审批依据：Staging 冒烟报告、变更清单、回滚预案（IA-OPS-006 链接）；
- P1 事故期间的变更冻结：事故未关闭时，审批人一律拒绝合并与发布。

---

## 7. 密钥管理（GitHub Secrets 清单）

| Secret 名称 | 用途 | 示例/说明 | 轮换周期 |
|--------------|------|-----------|----------|
| GITHUB_TOKEN | 内置 token，推 ghcr | 自动生成 | 每次 job |
| GHCR_USER / GHCR_TOKEN | 跨仓库推/拉镜像 | PAT（write:packages） | 90 天 |
| STAGING_HOST | Staging 服务器 IP | 10.10.9.11 | — |
| STAGING_USER | SSH 用户 | deploy | — |
| STAGING_SSH_KEY | SSH 私钥（ed25519） | -----BEGIN OPENSSH... | 180 天 |
| K3S_KUBECONFIG | 生产 k3s kubeconfig（base64） | base64(/etc/rancher/k3s/k3s.yaml) | 180 天 |
| SLACK_WEBHOOK_URL | 流水线失败通知（内网机器人） | https://hooks... | 90 天 |
| CODECOV_TOKEN 或内网覆盖率网关 token | 覆盖率上报（沙盒可本地存档） | — | 180 天 |

**管理原则**：Secrets 仅在 GitHub Settings → Secrets of Environments 配置（staging/production 分环境隔离）；任何 Secret 不得写入代码、workflow 明文、日志；打印日志前使用 `${{ secrets.X }}` 的 job 自动脱敏。

---

## 8. 制品管理与版本追溯

### 8.1 制品清单

| 制品 | 位置 | 保留策略 |
|------|------|----------|
| 三服务容器镜像 | ghcr.io/industryagent/* | semver 永久保留；git-sha 保留 90 天 |
| web dist 产物 | GitHub Artifacts | 14 天 |
| 测试与覆盖率报告 | GitHub Artifacts / PR 评论摘要 | 14 天 |
| Trivy SARIF | GitHub Security tab | 随仓库 |
| Release Notes | GitHub Releases（tag 关联） | 永久 |

### 8.2 版本追溯链路

```
代码 commit (sha) → 构建日志 (Actions run id) → 镜像 (sha tag / semver tag)
→ 部署记录 (rollout history / compose ps 镜像摘要) → 变更工单 (发布单编号)
```

追溯验证命令：

```bash
# 1) 生产当前运行版本
kubectl -n industryagent-app get deployment industryagent-server \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# 2) 该镜像的构建来源（ghcr 页面 labels：org.opencontainers.image.revision）
docker inspect ghcr.io/industryagent/industryagent-server:v1.2.0 \
  --format '{{ index .Config.Labels "org.opencontainers.image.revision" }}'
# 3) 由 sha 反查 commit
git log --oneline | grep a1b2c3d
```

### 8.3 Release Note 模板

```markdown
## v1.2.0 (2026-10-15)
- 镜像：server@v1.2.0 / agent@v1.2.0 / web@v1.2.0（基础 commit a1b2c3d）
- 变更：根因 Agent 新增因果推断工具；审计导出支持按 traceId 检索
- 数据库迁移：0042_add_audit_trace_idx（向前兼容，无需停机）
- 已知风险：Kafka 消费者扩容后 Lag 短暂抖动（观察 30 分钟）
- 回滚方案：IA-OPS-006 §5.2 L1 应用回滚，5 分钟内完成
```

---

## 9. 附：流水线故障排查

| 现象 | 排查方向 |
|------|----------|
| services 容器起不来导致测试卡住 | 检查 options 健康命令是否可执行（pg_isready 路径）、端口映射冲突 |
| ghcr 推送 403 | GITHUB_TOKEN 权限缺 `packages: write`；或 PAT 过期 |
| docker layer 缓存失效 | 检查是否破坏"先依赖后源码"的 COPY 顺序 |
| Staging SSH 部署超时 | 跳板机防火墙、STAGING_SSH_KEY 与用户匹配 |
| rollout status 超时 | 镜像拉取失败（imagePullPolicy/secret）或 readinessProbe 未通过 |

---

## 10. 参考文档

- IA-OPS-001《部署方案》：环境与 Staging/生产拓扑
- IA-OPS-003《Docker 容器化规范》：三个服务的完整 Dockerfile
- IA-OPS-006《回滚方案》：发布失败自动回滚与手工回滚 SOP
