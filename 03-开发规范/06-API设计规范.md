# API 设计规范

| 文档控制 | 内容 |
|---|---|
| 文档编号 | IA-DEV-007 |
| 文档名称 | API 设计规范 |
| 版本 | V1.0 |
| 密级 | 内部 |
| 编制 | 李吉 |
| 审核 | （待填） |
| 批准 | （待填） |
| 日期 | 2026-10-15 |

## 修订记录

| 版本 | 日期 | 修订内容 | 编制/修订人 | 审核人 |
|---|---|---|---|---|
| V1.0 | 2026-10-15 | 初始版本发布 | 李吉 | — |

---

## 1. 适用范围

本规范约束 IndustryAgent 平台对外的全部 HTTP API：

- `industryagent-server`（Egg.js）暴露的 `/api/v1/*` 业务接口
- `industryagent-agent`（FastAPI）暴露的 `/api/v1/agent/*` 智能体接口（服务间调用亦遵守本规范）

前端与后端、后端与智能体服务的接口契约以本规范 + OpenAPI 文档为准。

## 2. URL 设计

### 2.1 资源命名与层级

1. **资源用名词复数、kebab-case**，禁止 URL 中出现动词（动词由 HTTP 方法表达）。
2. 层级不超过 3 层，子资源挂父资源 ID 下；超过 3 层说明资源应独立。
3. 服务前缀统一 `/api/v1`，业务域为第一段。

```text
# ✅ 正例
GET    /api/v1/yield/lots                    # 批次列表
GET    /api/v1/yield/lots/:lotId             # 批次详情
POST   /api/v1/yield/lots/:lotId/analysis-tasks   # 对批次发起根因分析（创建子资源）
GET    /api/v1/agent/tasks/:taskId           # 智能体任务状态
GET    /api/v1/equipments/:equipmentId/readings   # 设备传感器数据

# ❌ 反例
GET    /api/getLotList                       # 动词 + 驼峰
POST   /api/v1/yield/lots/create             # 动词 create（POST 本身即创建）
DELETE /api/v1/deleteLot?id=123              # 动词 + 删除却用 GET 语义
GET    /api/v1/yield/lots/:lotId/wafers/:waferId/defects/maps/cells   # 层级过深
```

### 2.2 查询参数规范

| 用途 | 参数 | 约定 |
|---|---|---|
| 分页 | `page`（从 1 起）、`pageSize`（默认 20，上限 100） | 响应配 `total` |
| 过滤 | 语义化字段名 `productId`、`status`、`lotNo` | 多值用逗号：`status=running,completed` |
| 排序 | `sort=field`（升序）、`sort=-field`（降序，`-` 前缀） | 多字段逗号分隔 |
| 时间范围 | `startTime` / `endTime`（ISO 8601，UTC） | 闭开区间 [start, end) |
| 字段裁剪 | `fields=id,lotNo,yieldRate` | 白名单机制，禁止裁剪未暴露字段 |
| 关键字 | `keyword` | 服务端限定匹配字段 |

```text
# ✅ 正例
GET /api/v1/yield/lots?page=1&pageSize=20&productId=88&status=running&sort=-createdAt&fields=id,lotNo,yieldRate

# ❌ 反例
GET /api/v1/yield/lots?currentPage=1&size=20&sort=createdAt_desc&like=Y02&select=*
```

## 3. HTTP 方法语义与状态码

### 3.1 方法语义

| 方法 | 语义 | 幂等 | 示例 |
|---|---|---|---|
| GET | 读取资源/列表 | 是 | 获取批次详情 |
| POST | 创建资源、触发任务 | 否（可用幂等键，见第 7 章） | 发起根因分析 |
| PUT | 全量替换资源 | 是 | 更新告警规则全量字段 |
| PATCH | 局部更新资源 | 是 | 更新批次状态 |
| DELETE | 删除资源（软删除） | 是 | 删除看板配置 |

### 3.2 状态码使用表

| 状态码 | 使用场景 | 禁止用法 |
|---|---|---|
| 200 OK | GET/PUT/PATCH 成功 | — |
| 201 Created | POST 创建资源成功（响应含 Location 或新资源） | — |
| 202 Accepted | 异步任务受理（根因分析、报告生成），响应体含任务查询地址 | — |
| 204 No Content | DELETE 成功、无响应体 | — |
| 400 Bad Request | 请求体格式错误（非法 JSON 等） | 用作通用业务错误筐 |
| 401 Unauthorized | 未登录 / token 无效或过期 | 与 403 混用 |
| 403 Forbidden | 已登录但无权限 | — |
| 404 Not Found | 资源不存在 / 未授权访问他人资源（防枚举，不暴露"存在但无权"） | — |
| 409 Conflict | 资源状态冲突（如批次非 running 状态发起分析） | — |
| 410 Gone | 已废弃接口（版本下线期） | — |
| 422 Unprocessable Entity | 参数校验失败（IA-20001） | — |
| 429 Too Many Requests | 触发限流（响应含 Retry-After） | — |
| 500 Internal Server Error | 未预期系统错误 | 暴露堆栈给客户端 |
| 502/504 | 下游智能体服务失败/超时 | — |

```typescript
// ❌ 反例：一切皆 200 + body 里自造 error 字段
ctx.status = 200;
ctx.body = { error: 'lot not found', success: false };

// ✅ 正例：语义化状态码 + 统一响应包
ctx.status = 404;
ctx.body = { code: 'IA-30011', message: '批次不存在', data: null, requestId };
```

## 4. 统一响应包

### 4.1 成功响应

```json
{
  "code": "IA-00000",
  "message": "ok",
  "data": { "id": "9007199254740993", "lotNo": "Y02-A123", "yieldRate": 0.9312 },
  "requestId": "req-a1b2c3d4"
}
```

### 4.2 错误响应

```json
{
  "code": "IA-30021",
  "message": "批次当前状态不允许发起根因分析",
  "data": null,
  "requestId": "req-a1b2c3d4"
}
```

### 4.3 分页结构

```json
{
  "code": "IA-00000",
  "message": "ok",
  "data": {
    "list": [ { "id": "9007199254740993", "lotNo": "Y02-A123" } ],
    "page": 1,
    "pageSize": 20,
    "total": 358
  },
  "requestId": "req-a1b2c3d4"
}
```

规则：`requestId` 由网关中间件生成并透传（响应头 `x-request-id` 同步返回），用于全链路日志追踪；`code` 为字符串；ID 大于 JS 安全整数的统一以字符串返回。

## 5. 错误码分配表

| 错误码 | HTTP | 含义 |
|---|---|---|
| **IA-00xxx 成功** | | |
| IA-00000 | 200 | 成功 |
| **IA-10xxx 认证与授权** | | |
| IA-10001 | 401 | 未登录或 token 缺失 |
| IA-10002 | 401 | token 无效或已过期 |
| IA-10003 | 403 | 无接口访问权限 |
| IA-10004 | 403 | 无资源访问权限（越权拦截） |
| **IA-20xxx 参数与校验** | | |
| IA-20001 | 422 | 参数校验失败（message 附明细） |
| IA-20002 | 400 | 请求体格式错误（非法 JSON） |
| IA-20003 | 400 | 不支持的查询参数组合 |
| **IA-30xxx 业务规则** | | |
| IA-30010 | 409 | 批次号已存在 |
| IA-30011 | 404 | 批次不存在 |
| IA-30021 | 409 | 批次状态不允许该操作 |
| IA-30030 | 409 | 重复提交（无幂等键的重复创建） |
| **IA-40xxx 智能体服务** | | |
| IA-40001 | 504 | 智能体任务执行超时 |
| IA-40002 | 429 | 智能体并发达到上限 |
| IA-40010 | 502 | LLM 调用失败 |
| IA-40011 | 502 | 工具调用失败（DB/图谱/RAG） |
| IA-40020 | 409 | 根因分析任务已在执行中 |
| **IA-50xxx 系统内部** | | |
| IA-50001 | 500 | 数据库错误 |
| IA-50002 | 500 | 缓存服务异常 |
| IA-50000 | 500 | 未分类系统错误 |

新增错误码先登记 `error-codes.ts` 与本表，再使用；禁止私有造码。

## 6. 版本策略

1. 版本入 URL：`/api/v1/...`，不使用 Header 版本（便于网关路由与调试）。
2. 大版本升级（v1→v2）仅在破坏性变更时发生；同版本内**只加不改不删**（见第 10 章）。
3. **废弃流程**：新路径上线 → 旧路径响应加 `Deprecation: true` 与 `Sunset: <HTTP-date>` 响应头 → 客户端观察期 ≥ 2 个 release（约 8 周）→ 旧路径返回 `410 Gone`；废弃计划须发技术公告并同步前端排期。

## 7. 幂等性

- GET/PUT/PATCH/DELETE 天然幂等。
- **POST 幂等键**：创建类、任务触发类 POST 接口支持 `Idempotency-Key` 请求头（UUID），服务端在 Redis 记录 `key → 响应体`（TTL 24h）：

```text
POST /api/v1/yield/lots/123/analysis-tasks
Idempotency-Key: 7f9c24e5-1b3a-4d8e-9c2f-6a1e8d0b7c33
```

```typescript
// 幂等中间件核心逻辑
const cached = await app.redis.get(`idem:${key}`);
if (cached) {
  ctx.status = 200;                 // 首次为 201，重放统一 200 + 幂等标记
  ctx.set('x-idempotent-replay', 'true');
  ctx.body = JSON.parse(cached);
  return;
}
await next();
if (ctx.status === 201 || ctx.status === 202) {
  await app.redis.set(`idem:${key}`, JSON.stringify(ctx.body), 'EX', 86400);
}
```

未带幂等键的重复提交由业务层防重（IA-30030）。

## 8. 限流

| 维度 | 阈值（默认） | 实现 |
|---|---|---|
| 按用户（登录态） | 120 次/分钟 | Redis 令牌桶，key=`rl:u:{userId}` |
| 按 IP（未登录） | 30 次/分钟 | key=`rl:ip:{ip}` |
| 智能体触发接口 | 5 次/分钟/用户（LLM 成本保护） | 单独收紧 |

超限响应：

```text
HTTP/1.1 429 Too Many Requests
Retry-After: 30
Content-Type: application/json

{ "code": "IA-40002", "message": "请求过于频繁，请稍后重试", "data": null, "requestId": "req-..." }
```

## 9. 安全要求

1. **全站 HTTPS**（TLS 1.2+），内网服务间调用同样走 mTLS 或内网 TLS。
2. **Bearer 认证**：`Authorization: Bearer <jwt>`；JWT 有效期 ≤ 2h，刷新走独立 refresh token。
3. **最小权限**：接口按角色声明所需权限（RBAC），资源级权限（IDOR）在 Service 校验归属。
4. **敏感字段响应过滤**：手机号/邮箱/token 响应前必须经 `mask` 工具脱敏；密码、密钥类字段永不返回（即便内部接口）。
5. 响应头安全基线：`X-Content-Type-Options: nosniff`、CSP（前端域）、禁止在响应中回显服务端堆栈。

## 10. 变更管理（兼容性原则）

**同版本内只加不改不删：**

| 操作 | 是否允许 | 规则 |
|---|---|---|
| 新增接口/字段 | ✅ | 新字段可为空或有默认值，旧客户端无感 |
| 新增可选请求参数 | ✅ | 默认值保证旧行为不变 |
| 修改字段语义/类型 | ❌ | 须新版本（v2）或新增字段替代 |
| 删除字段/接口 | ❌ | 走第 6 章废弃流程，≥ 2 个 release 观察期 |
| 收紧参数校验 | ⚠️ | 可能破坏旧客户端，须评估并公告 |

任何接口变更 PR 必须同步更新 OpenAPI 文档并跑契约测试（CI 对比 openapi diff，破坏性变更直接红灯）。

## 11. 文档要求

1. **每个接口必须有 OpenAPI 注解与示例**（后端使用 `egg-swagger`/`swagger-jsdoc` 注释生成；agent 服务用 FastAPI 自带 OpenAPI）。
2. PR 中新增/修改接口未更新文档 → CR 打回（IA-DEV-005 清单 2.2）。
3. 示例必须真实可执行（可直接 curl 复制运行）。

```typescript
/**
 * @openapi
 * /api/v1/yield/lots:
 *   get:
 *     tags: [yield]
 *     summary: 分页查询批次列表
 *     parameters:
 *       - { in: query, name: page, schema: { type: integer, default: 1 } }
 *       - { in: query, name: pageSize, schema: { type: integer, default: 20, maximum: 100 } }
 *       - { in: query, name: status, schema: { type: string, enum: [pending, running, completed, scrapped] } }
 *     responses:
 *       200:
 *         description: 批次分页结果
 *         content:
 *           application/json:
 *             schema: { $ref: '#/components/schemas/LotPage' }
 *             example:
 *               code: IA-00000
 *               message: ok
 *               data:
 *                 list: [{ id: "9007199254740993", lotNo: "Y02-A123", yieldRate: 0.9312 }]
 *                 page: 1
 *                 pageSize: 20
 *                 total: 358
 *               requestId: req-a1b2c3d4
 *       401: { description: 未登录（IA-10001） }
 */
```

## 12. 检查清单速览

- [ ] URL 名词复数、kebab-case、无动词、层级 ≤ 3
- [ ] 查询参数符合 page/pageSize/sort/fields 约定
- [ ] 状态码语义准确（401/403/404/409/422 不混用）
- [ ] 响应为统一包结构，含 requestId；分页含 total
- [ ] 错误码已登记且在分配表内
- [ ] POST 创建/任务接口支持 Idempotency-Key
- [ ] 限流生效，429 带 Retry-After
- [ ] 响应敏感字段已脱敏，无堆栈泄露
- [ ] OpenAPI 注解 + 可执行示例已更新，契约测试通过
- [ ] 变更符合"只加不改不删"，废弃流程已公告
