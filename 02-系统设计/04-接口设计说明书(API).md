# IndustryAgent 工业智能体平台 —— 接口设计说明书（API）

## 文档控制块

| 项目 | 内容 |
|---|---|
| 文档编号 | IA-SD-005 |
| 文档名称 | 接口设计说明书（REST / WebSocket / MCP） |
| 版本 | V1.0 |
| 密级 | 内部 |
| 编制 | 李吉 |
| 审核 | 　 |
| 批准 | 　 |
| 日期 | 2026-10-15 |

## 修订记录

| 版本 | 日期 | 修订人 | 修订说明 |
|---|---|---|---|
| V0.1 | 2026-10-02 | 李吉 | 初稿：总则 + 认证/用户模块接口 |
| V0.5 | 2026-10-13 | 李吉 | 补全 30+ 接口、WS 事件、MCP 工具 |
| V1.0 | 2026-10-15 | 李吉 | 定稿：限流幂等设计、错误码表复核 |

---

## 1. API 设计总则

1. **风格**：RESTful，资源名复数名词（`/anomalies`），动作用 HTTP 方法表达；
2. **版本**：URL 版本前缀 `/api/v1`，破坏性变更升 `/api/v2` 并保持 v1 兼容 6 个月；
3. **命名**：路径 kebab/snake 小写；查询参数 camelCase；时间 ISO 8601 带时区；
4. **分页**：统一 `?pageNum=1&pageSize=20`（pageSize 上限 100），列表响应携带 `total`；
5. **过滤/排序**：`?status=WAIT_APPROVAL&equipmentCode=ETCH-03&startTime=…&endTime=…&orderBy=createdAt:desc`；
6. **统一响应结构**：

```json
// 成功
{ "code": 0, "message": "success", "data": { } }
// 失败
{ "code": "IA-11001", "message": "资源不存在", "data": null }
// 分页 data 结构
{ "list": [ ... ], "total": 128, "pageNum": 1, "pageSize": 20 }
```

7. **幂等**：POST 创建类接口支持 `Idempotency-Key` 头（UUID，Redis 保存 24h，重复提交返回首次结果）；
8. **追踪**：请求头 `X-Request-Id`（缺省服务端生成），响应头回传，贯穿日志与 audit_logs.trace_id。

## 2. 认证鉴权与错误码

### 2.1 认证

- 除 `/auth/login`、健康检查外全部接口要求 `Authorization: Bearer <JWT>`（Keycloak 签发，RS256）；
- JWT claims：`sub`（用户ID）、`preferred_username`、`realm_access.roles`、`exp`；
- 401 响应触发前端 refresh_token 流程（见 IA-SD-003 第 7.2 节）。

### 2.2 错误码表（IA-1xxxx 分类段）

| 错误码 | HTTP | 说明 |
|---|---|---|
| IA-10401 | 401 | 未认证 / 令牌无效或过期 |
| IA-10403 | 403 | 无权限（OPA 拒绝） |
| IA-10429 | 429 | 触发限流 |
| IA-11001 | 404 | 资源不存在 |
| IA-11002 | 400 | 参数校验失败（message 含字段明细） |
| IA-11003 | 409 | 资源冲突（重复创建/并发修改） |
| IA-12001 | 409 | 任务状态非法流转 |
| IA-12002 | 409 | 建议已被审批（重复审批） |
| IA-12003 | 400 | 驳回理由不足（<10 字） |
| IA-13001 | 502 | ai-service 不可用 |
| IA-13002 | 504 | MCP 工具调用超时 |
| IA-13003 | 422 | 智能体结果证据链不完整 |
| IA-15001 | 500 | 数据库错误 |
| IA-15002 | 500 | 消息队列异常 |
| IA-15003 | 500 | 对象存储异常 |

---

## 3. 接口清单总览（34 个 REST 接口）

| # | 模块 | 方法 | 路径 | 说明 | 权限 |
|---|---|---|---|---|---|
| 1 | 认证 | POST | /auth/login | 登录交换令牌 | 公开 |
| 2 | 认证 | POST | /auth/refresh | 刷新令牌 | 公开(refresh_token) |
| 3 | 认证 | POST | /auth/logout | 登出吊销 | 登录用户 |
| 4 | 认证 | GET | /auth/profile | 当前用户信息+权限码 | 登录用户 |
| 5 | 用户 | GET | /users | 用户分页列表 | admin |
| 6 | 用户 | POST | /users | 创建用户 | admin |
| 7 | 用户 | PUT | /users/:id | 更新用户 | admin |
| 8 | 用户 | PUT | /users/:id/roles | 分配角色 | admin |
| 9 | 晶圆 | GET | /wafers | 晶圆分页查询 | 登录用户 |
| 10 | 晶圆 | GET | /wafers/:barcode | 晶圆详情 | 登录用户 |
| 11 | 晶圆 | GET | /lots | 批次分页查询 | 登录用户 |
| 12 | 缺陷 | GET | /defects | 缺陷记录分页 | 登录用户 |
| 13 | 缺陷 | GET | /defects/:id/image-url | 缺陷图预签名URL | 登录用户 |
| 14 | 缺陷 | GET | /defects/stat | 缺陷类型统计 | 登录用户 |
| 15 | 异常 | GET | /anomalies | 异常事件分页 | 登录用户 |
| 16 | 异常 | GET | /anomalies/:id | 异常详情（含任务链） | 登录用户 |
| 17 | 根因 | POST | /agent-tasks | 手动创建根因任务 | 工艺/质量工程师 |
| 18 | 根因 | GET | /agent-tasks | 任务分页 | 登录用户 |
| 19 | 根因 | GET | /agent-tasks/:id | 任务详情 | 登录用户 |
| 20 | 根因 | POST | /agent-tasks/:id/retry | 失败重试 | 工艺工程师 |
| 21 | 根因 | POST | /agent-tasks/:id/takeover | 人工接管 | 工艺/设备工程师 |
| 22 | 根因 | GET | /rootcause-results/:taskId | Top-3 结果+证据链 | 登录用户 |
| 23 | 建议 | GET | /suggestions | 建议分页 | 登录用户 |
| 24 | 建议 | GET | /suggestions/:id | 建议详情 | 登录用户 |
| 25 | 审批 | GET | /approvals | 审批分页（待审/已审） | 工艺工程师/生产主管 |
| 26 | 审批 | POST | /approvals | 提交审批决定 | 工艺工程师/生产主管 |
| 27 | 审批 | GET | /approvals/stat | 审批SLA统计 | 生产主管 |
| 28 | 报告 | GET | /reports | 8D 报告分页 | 质量/生产/工艺 |
| 29 | 报告 | GET | /reports/:id | 报告详情(D1-D8) | 同上 |
| 30 | 报告 | POST | /reports/:id/release | 签发报告 | 质量工程师 |
| 31 | 看板 | GET | /dashboard/overview | 看板总览 | 登录用户 |
| 32 | 看板 | GET | /dashboard/yield-trend | 良率趋势 | 登录用户 |
| 33 | 审计 | GET | /audits | 审计日志分页 | admin |
| 34 | 通知 | GET | /notifications | 我的通知（断线补偿） | 登录用户 |

以下按模块给出代表性接口的字段级设计（其余同构接口遵循总则）。

---

## 4. 认证模块

### 4.1 POST /api/v1/auth/login

| 参数 | 位置 | 类型 | 必填 | 说明 |
|---|---|---|---|---|
| username | body | string | 是 | 用户名 |
| password | body | string | 是 | 密码（Keycloak ROGP 校验） |

请求示例：

```json
{ "username": "li_ji", "password": "********" }
```

响应示例：

```json
{ "code": 0, "message": "success", "data": {
  "accessToken": "eyJhbGciOiJSUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJSUzI1NiIs...",
  "expiresIn": 900,
  "user": { "id": "u-0001", "username": "li_ji", "fullName": "李吉",
            "roles": ["process_engineer"],
            "permissions": ["approval:approve","rootcause:view","suggestion:notify"] }
} }
```

错误码：IA-10401（凭据错误，message 不区分用户不存在/密码错误）。

### 4.2 GET /api/v1/auth/profile

响应示例：

```json
{ "code": 0, "message": "success", "data": {
  "id": "u-0001", "username": "li_ji", "fullName": "李吉",
  "email": "li***@xinlian.com", "department": "工艺部",
  "roles": ["process_engineer"], "permissions": ["approval:approve", "..."],
  "lastLoginAt": "2026-10-15T08:30:00+08:00"
} }
```

---

## 5. 用户管理模块

### 5.1 GET /api/v1/users

查询参数：`pageNum`、`pageSize`、`keyword`（姓名/用户名模糊）、`status`、`department`。

响应示例：

```json
{ "code": 0, "message": "success", "data": {
  "list": [
    { "id": "u-0002", "username": "wang_gong", "fullName": "王工",
      "email": "wa***@xinlian.com", "department": "设备部",
      "status": "ACTIVE", "roles": ["equipment_engineer"], "createdAt": "2026-08-01T10:00:00+08:00" }
  ], "total": 48, "pageNum": 1, "pageSize": 20 } }
```

### 5.2 PUT /api/v1/users/:id/roles

请求示例：

```json
{ "roleCodes": ["quality_engineer"] }
```

响应：`{ "code": 0, "message": "success", "data": null }`。错误码：IA-11001（用户不存在）、IA-11002（角色码非法）。写审计。

---

## 6. 晶圆 / 批次模块

### 6.1 GET /api/v1/wafers

查询参数：`pageNum`、`pageSize`、`lotNo`、`grade`、`product`、`yieldRateGte`、`startTime`、`endTime`、`orderBy`。

响应示例：

```json
{ "code": 0, "message": "success", "data": {
  "list": [
    { "barcode": "W-20261014-000123", "lotNo": "LOT-20261014-007",
      "product": "MCU", "currentStep": 34, "grade": "B",
      "yieldRate": 88.50, "currentEquipment": "ETCH-03",
      "startedAt": "2026-10-12T06:00:00+08:00" }
  ], "total": 50213, "pageNum": 1, "pageSize": 20 } }
```

### 6.2 GET /api/v1/lots

查询参数：`lotNo`、`product`、`lineNo`、`status`、分页同上。响应 `list[]` 字段：`lotNo/product/quantity/lineNo/currentStep/status/recipeCode/planEndAt`。

---

## 7. 缺陷模块

### 7.1 GET /api/v1/defects

查询参数：`waferBarcode`、`equipmentCode`、`defectType`、`severity`、`startTime`、`endTime`、分页。

响应示例：

```json
{ "code": 0, "message": "success", "data": {
  "list": [
    { "id": "d-9f21", "waferBarcode": "W-20261014-000123", "equipmentCode": "ETCH-03",
      "processStep": "ETCH_MAIN", "defectType": "PATTERN", "defectCount": 6,
      "xPos": 42.10, "yPos": 88.30, "severity": "HIGH",
      "detectedAt": "2026-10-14T23:41:12+08:00" }
  ], "total": 356, "pageNum": 1, "pageSize": 20 } }
```

### 7.2 GET /api/v1/defects/:id/image-url

响应示例：

```json
{ "code": 0, "message": "success", "data": {
  "url": "https://minio.local/defect-images/2026/10/xxx.png?X-Amz-Expires=300&...",
  "expiresIn": 300 } }
```

错误码：IA-11001、IA-15003（对象存储异常）。

---

## 8. 异常事件模块

### 8.1 GET /api/v1/anomalies

查询参数：`equipmentCode`、`paramName`、`taskStatus`、`startTime`、`endTime`、分页。

响应示例：

```json
{ "code": 0, "message": "success", "data": {
  "list": [
    { "anomalyId": "AN-20261015-0031", "equipmentCode": "ETCH-03",
      "equipType": "ETCH", "lineNo": "L2", "paramName": "RF_POWER",
      "score": 0.87, "detectedBy": ["FDC_RULE_R12","SPC_EWMA","IFOREST"],
      "taskStatus": "WAIT_APPROVAL", "occurredAt": "2026-10-15T09:12:33+08:00" }
  ], "total": 42, "pageNum": 1, "pageSize": 20 } }
```

### 8.2 GET /api/v1/anomalies/:id

响应示例（含任务链）：

```json
{ "code": 0, "message": "success", "data": {
  "anomalyId": "AN-20261015-0031", "equipmentCode": "ETCH-03", "paramName": "RF_POWER",
  "score": 0.87, "occurredAt": "2026-10-15T09:12:33+08:00",
  "detail": { "fdcRule": "R12", "window": "10min", "meanShiftPct": 3.2,
              "uclBreach": true, "iforestScore": -0.41 },
  "taskChain": [
    { "taskId": "t-77a1", "status": "WAIT_APPROVAL", "startedAt": "2026-10-15T09:12:40+08:00" },
    { "suggestionId": "s-33c2", "riskLevel": "MID", "status": "PENDING" }
  ] } }
```

---

## 9. 根因分析模块（agent-task）

### 9.1 POST /api/v1/agent-tasks（手动触发，支持幂等）

请求头：`Idempotency-Key: 5f8b3c2e-…`。

请求示例：

```json
{ "equipmentCode": "ETCH-03", "paramName": "RF_POWER",
  "defectType": "PATTERN", "timeWindowHours": 24, "priority": 1,
  "note": "工艺例会要求复核" }
```

响应示例：

```json
{ "code": 0, "message": "success", "data": {
  "taskId": "t-77a1", "anomalyId": "AN-20261015-0031",
  "status": "PENDING", "estimatedMinutes": 5 } }
```

错误码：IA-11003（同设备同参数存在未终态任务）、IA-13001（ai-service 不可用，任务仍创建为 PENDING 由补偿调度重派）。

### 9.2 POST /api/v1/agent-tasks/:id/takeover

请求示例：

```json
{ "reason": "智能体Top-3与现场观察不符，转人工FMEA排查" }
```

响应：`{ "code": 0, "message": "success", "data": { "taskId": "t-77a1", "status": "TAKEOVER" } }`。写审计，计入人工接管率指标。

### 9.3 GET /api/v1/rootcause-results/:taskId

响应示例（证据链完整结构，与 IA-SD-004 3.10 JSON 契约一致）：

```json
{ "code": 0, "message": "success", "data": {
  "taskId": "t-77a1", "confidence": 87.0,
  "suspectsTop3": [
    { "rank": 1, "equipmentCode": "ETCH-03", "processStep": "ETCH_MAIN",
      "paramName": "RF_POWER", "confidence": 0.87,
      "evidences": [
        { "type": "GRAPH", "desc": "三跳关联 weight=0.82",
          "detail": { "path": ["Equipment(ETCH-03)","HAS_STEP(ETCH_MAIN)","USES_PARAM(RF_POWER)","CORRELATES(PATTERN)"], "weight": 0.82 } },
        { "type": "CAUSAL", "desc": "DoWhy ATT=0.41 [0.33,0.49]",
          "detail": { "estimand": "att", "effect": 0.41, "pvalue": 0.003 } },
        { "type": "RAG", "desc": "历史CAPA-2025-118 同源案例",
          "detail": { "docId": "doc-a12f", "chunkIndex": 42, "score": 0.89, "quote": "RF_POWER 腔体漂移导致 PATTERN 缺陷率上升…" } },
        { "type": "SENSOR", "desc": "24h 均值 +3.2% 超UCL",
          "detail": { "window": "24h", "meanShiftPct": 3.2, "uclBreach": true } } ] },
    { "rank": 2, "equipmentCode": "CVD-07", "paramName": "TEMP_2", "confidence": 0.54, "evidences": [ "..." ] },
    { "rank": 3, "equipmentCode": "CMP-02", "paramName": "SLURRY_FLOW", "confidence": 0.31, "evidences": [ "..." ] } ],
  "finalRootcause": null, "modelMeta": { "graphRunId": "run-9c01", "tokensIn": 8210, "tokensOut": 1840 } } }
```

---

## 10. 建议 / 审批模块

### 10.1 GET /api/v1/suggestions/:id

响应示例：

```json
{ "code": 0, "message": "success", "data": {
  "id": "s-33c2", "taskId": "t-77a1", "suspectRank": 1,
  "suggestionType": "PARAM_ADJUST", "riskLevel": "MID",
  "content": "建议将 ETCH-03 腔体 RF_POWER 由 1290W 回调至 1250W（配方基准），并执行 3 片验证批。",
  "paramsDelta": { "RF_POWER": { "from": 1290, "to": 1250, "unit": "W" } },
  "riskAssessment": "回调幅度在规格带内；风险为当批 2 批次 WIP 重排队（约 4h 产能缓冲）。",
  "affectedWaferCount": 50, "estimatedBenefit": "预计 PATTERN 缺陷率回落 60%，良率恢复 +1.8pp",
  "status": "PENDING", "createdAt": "2026-10-15T09:15:02+08:00" } }
```

### 10.2 POST /api/v1/approvals（核心接口：审批决定）

请求头：`Idempotency-Key` 必填（防重复点击）。

| 参数 | 位置 | 类型 | 必填 | 说明 |
|---|---|---|---|---|
| suggestionId | body | string(uuid) | 是 | 建议ID |
| decision | body | string | 是 | APPROVED / REJECTED |
| reason | body | string | 是 | 驳回必填≥10字；批准建议填写执行备注 |

请求示例：

```json
{ "suggestionId": "s-33c2", "decision": "APPROVED",
  "reason": "同意回调，验证批安排在 LOT-20261016-002。" }
```

响应示例：

```json
{ "code": 0, "message": "success", "data": {
  "approvalId": "ap-88d3", "suggestionId": "s-33c2", "decision": "APPROVED",
  "approver": { "id": "u-0001", "name": "李吉", "role": "process_engineer" },
  "decidedAt": "2026-10-15T09:41:18+08:00", "slaMinutes": 26 } }
```

错误码：IA-12002（建议已被审批）、IA-12003（驳回理由不足）、IA-11001。写审计；决策后触发 WS `task.status` 推送与 Kafka `agent.feedback`。

### 10.3 GET /api/v1/approvals/stat

查询参数：`startTime`、`endTime`。响应 data：`{ total, approved, rejected, avgSlaMinutes, escalationCount, takeoverRate }`。

---

## 11. 8D 报告模块

### 11.1 GET /api/v1/reports/:id

响应示例（content_d1_d8 摘要）：

```json
{ "code": 0, "message": "success", "data": {
  "id": "r-5e01", "reportNo": "8D-20261015-007", "taskId": "t-77a1",
  "title": "ETCH-03 RF_POWER 漂移致 PATTERN 缺陷率异常 8D 报告",
  "status": "RELEASED", "author": { "id": "u-0004", "name": "陈质检" },
  "content": {
    "d1_team": { "members": ["李吉(工艺)", "王工(设备)", "陈质检(质量)"] },
    "d2_problem": "2026-10-15 PATTERN 缺陷率 6.2%（基线 2.1%）…",
    "d3_containment": "HOLD 影响批次 2 个/50 片，全检隔离…",
    "d4_rootcause": "RF_POWER 由 1250W 漂移至 1290W（置信度 87%）",
    "d4_evidence_refs": ["rootcause:t-77a1#rank1", "evidence:CAUSAL", "evidence:RAG"],
    "d5_corrective": "参数回调 + 匹配器校准 SOP 修订…",
    "d6_preventive": "FMEA 更新 RF_POWER 漂移探测规则 R12 灵敏度…",
    "d7_verify": "验证批 3 片 PATTERN 缺陷率 2.0%（目标 ≤2.5%）",
    "d8_recognition": "…"
  },
  "pdfKey": "reports-8d/2026/10/8D-20261015-007.pdf" } }
```

### 11.2 POST /api/v1/reports/:id/release

仅 `quality_engineer`。请求：`{ "comment": "验证数据齐备，同意签发" }`。错误码：IA-11002（验证数据未齐）。

---

## 12. 看板 / 审计 / 通知模块

### 12.1 GET /api/v1/dashboard/overview

查询参数：`range=7d|30d|90d`（默认 7d）。响应 data：

```json
{ "yieldRate": 93.8, "yieldTarget": 95.0, "oee": 81.2, "oeeTarget": 85.0,
  "anomalyToday": 12, "pendingApprovals": 3, "mttrHours": 3.6,
  "topRootcauses": [ { "name": "参数漂移(ETCH)", "count": 18 },
                      { "name": "颗粒污染(CMP)", "count": 9 } ],
  "agentMetrics": { "top3Accuracy": 0.83, "falseAlarmRate": 0.08, "takeoverRate": 0.24 },
  "generatedAt": "2026-10-15T10:00:00+08:00" }
```

数据来源 Redis 缓存（TTL 60s），缓存键 `dash:overview:{range}`。

### 12.2 GET /api/v1/audits

查询参数：`userId`、`username`、`action`、`resource`（前缀匹配）、`traceId`、`startTime`、`endTime`、分页。仅 admin。响应 `list[]` 字段：`id/username/action/resource/resourceId/statusCode/durationMs/ip/traceId/createdAt`，`requestDigest` 脱敏后返回。

### 12.3 GET /api/v1/notifications

查询参数：`sinceEventId`（断线补偿游标）、`isRead`。响应 data：`{ list: [...], latestEventId }`。

---

## 13. WebSocket 事件接口

连接：`wss://host/ws?token=<JWT>`；心跳 30s ping/pong；断线由前端按 eventId 补偿拉取。

| 事件 | 方向 | 触发时机 | data 结构 |
|---|---|---|---|
| anomaly.push | 服务端→客户端 | 新异常事件落库 | `{ anomalyId, equipmentCode, paramName, score, occurredAt }` |
| task.status | 服务端→客户端 | 任务状态流转 | `{ taskId, anomalyId, status, updatedAt }` |
| approval.escalated | 服务端→客户端 | 审批超时升级 | `{ approvalId, escalatedTo, level }` |
| report.ready | 服务端→客户端 | 8D 生成完成 | `{ reportId, reportNo, taskId }` |
| ping/pong | 双向 | 保活 | — |

帧格式：

```json
{ "event": "task.status", "eventId": "evt-20261015-0102",
  "data": { "taskId": "t-77a1", "anomalyId": "AN-20261015-0031",
            "status": "WAIT_APPROVAL", "updatedAt": "2026-10-15T09:15:05+08:00" },
  "ts": 1760493305000 }
```

---

## 14. MCP 工具接口（egg-server:7002 → ai-service）

MCP 会话认证：ai-service 持服务身份令牌，按工具 `permission` 二次鉴权；每次调用写审计。

### 14.1 query_sensors

| 项 | 内容 |
|---|---|
| 功能 | 查询传感器时序（TimescaleDB） |
| 输入 schema | `{ equipment_code: string, param_name: string, start_time: ISO8601, end_time: ISO8601, agg: enum["raw","1m","1h"]="raw", limit: int<=10000 }` |
| 输出 schema | `{ rows: [{ ts: string, value: number, quality: string }], stats: { avg, min, max, std, missing_rate } }` |
| 权限 | sensor:read |

### 14.2 search_docs

| 项 | 内容 |
|---|---|
| 功能 | RAG 检索工艺文档（pgvector HNSW 余弦） |
| 输入 schema | `{ query: string, top_k: int=8, filter: { category?: string[], equipment_code?: string } }` |
| 输出 schema | `{ hits: [{ doc_id, doc_title, chunk_index, content, score }] }` |
| 权限 | doc:read |

### 14.3 query_graph

| 项 | 内容 |
|---|---|
| 功能 | 知识图谱多跳查询（Cypher 模板白名单，参数化） |
| 输入 schema | `{ template: enum["rootcause_trace","sibling_check","param_defect_corr","recipe_steps"], params: object, limit: int=20 }` |
| 输出 schema | `{ nodes: [...], edges: [...], meta: { template, latency_ms } }` |
| 权限 | graph:read |

### 14.4 run_causality

| 项 | 内容 |
|---|---|
| 功能 | DoWhy 因果效应估计（代理 ai-service 内部端点） |
| 输入 schema | `{ treatment: string(param), outcome: string(defect_rate), equipment_code: string, window: { start, end }, confounders: string[] }` |
| 输出 schema | `{ estimand: string, effect: number, ci: [number, number], pvalue: number, refute_tests: [{ name, passed }] }` |
| 权限 | causality:run |

### 14.5 submit_result

| 项 | 内容 |
|---|---|
| 功能 | 智能体回写根因/建议结果（submit 契约校验，证据链缺失返回 IA-13003） |
| 输入 schema | `{ task_id: string, rootcause: SuspectsTop3(同9.3), suggestions: Suggestion[] }` |
| 输出 schema | `{ accepted: boolean, errors?: string[] }` |
| 权限 | agent:submit |

---

## 15. 限流与幂等设计

### 15.1 限流（Redis 令牌桶，rateLimit 中间件）

| 范围 | 策略 | 超限响应 |
|---|---|---|
| 全局（per user） | 600 req / 5min | 429 + IA-10429 + Retry-After |
| /sensors、/dashboard | 60 req / min | 同上 |
| /auth/login | 10 次 / 5min / IP（防爆破） | 同上 |
| MCP 工具 | 120 次 / min / 会话 | MCP 错误帧 + code=IA-10429 |

### 15.2 幂等

| 接口 | 机制 |
|---|---|
| POST /agent-tasks、POST /approvals | `Idempotency-Key` 头；Redis `SETNX idem:{key}` 24h；重复请求返回首次响应快照 |
| POST /agent-tasks（业务级） | 同设备+同参数存在非终态任务 → IA-11003 |
| POST /approvals（业务级） | suggestion 状态非 PENDING → IA-12002 |

---

## 16. 契约管理与联调

- OpenAPI 3.1 由 Egg.js 注释自动生成（`/swagger.json`，仅内网可访问）；
- `src/types` 前端类型与 MCP JSON Schema 由 `@ia/contracts` npm 包统一发布，CI 中做契约漂移检查；
- 接口变更流程：契约 PR 评审 → 契约包发版 → 前后端同步升级 → 集成测试（GitHub Actions）。

（全文完）
