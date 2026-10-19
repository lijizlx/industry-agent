# ADR-001：业务后端选择 Python FastAPI

## 文档控制块

| 项目 | 内容 |
|---|---|
| 文档编号 | IA-ADR-001 |
| 文档名称 | 架构决策记录：业务后端框架选型 |
| 版本 | V2.0 |
| 密级 | 内部 |
| 编制 | 李吉 |
| 审核 | 　 |
| 批准 | 　 |
| 日期 | 2026-10-18 |

## 修订记录

| 版本 | 日期 | 修订人 | 修订说明 |
|---|---|---|---|
| V1.0 | 2026-10-15 | 李吉 | 首次定稿：选择 Egg.js 4.0，状态：已接受 |
| V2.0 | 2026-10-18 | 李吉 | **决策变更**：改选 Python FastAPI（Egg.js 4.x 发布线为 ESM-only 重构版，与既定工程体系不兼容；详见"背景"）。V1.0 决策废止 |

---

## 状态

已接受（2026-10-18，架构变更评审通过；替代 V1.0 的 Egg.js 决策）

## 背景

IndustryAgent v1.0 需要一个**业务编排层**服务，承担以下职责：

1. 八大业务域（auth/user、wafer、defect、sensor、agent-task、approval、report、audit、dashboard）的 REST API 与 CRUD；
2. 与 Keycloak（OIDC JWT 校验）的鉴权集成 + 本地 RBAC 判权；
3. 全链路审计中间件（写 audit_logs，仅追加）；
4. **同时作为 MCP Server**，将数据库查询等能力封装为工具暴露给 Python 智能体（LangGraph）；
5. WebSocket 实时推送（anomaly.push / notification.push，端口 7002）；
6. 定时任务（良率日报、数据健康检查）；
7. 与 Python FastAPI + LangGraph AI 微服务通过 REST + MCP 双通道协作。

**V1.0 决策（Egg.js 4.0）在实施验证中暴露两个无法绕过的问题（2026-10-17~18）：**

1. **egg 4.x 发布线已重构为 ESM-only**（`"type": "module"`，无 CJS 入口），与既定的 egg-bin/ts-node/egg-sequelize 插件链等经典 CJS 工程体系不兼容——教材、插件生态与运行时三者无法同时成立；退守 egg 3.x 则与全库"Egg.js 4.0"技术口径冲突且仍维持双栈；
2. **双栈成本被低估**：Node + Python 双语言意味着构建/镜像/依赖/CI ×2、工具链与人才技能割裂，而智能体侧本就是 FastAPI。

## 备选方案对比

| 维度 | FastAPI (Python) | Egg.js 3.x（回退） | NestJS 10 | Spring Boot 3 |
|---|---|---|---|---|
| 与智能体同栈 | ★★★★★ 与 AI 服务同语言同框架，模型/工具/业务共享依赖与部署经验 | ★★ 双栈维持 | ★★ 双栈维持 | ★★ 双栈维持 |
| 工程化成熟度 | ★★★★★ Pydantic 契约即文档（自动 OpenAPI）、依赖注入、async 原生 | ★★★★ 约定式工程化成熟但为上一代 | ★★★★★ 模块化与 DI 完备 | ★★★★★ 企业级标杆 |
| MCP Server 支持 | ★★★★★ Python MCP SDK（FastMCP）为官方一线支持，与 LangGraph 生态同源 | ★★★ 需社区封装 | ★★★ 需自行封装 | ★★ 社区 SDK 成熟度低 |
| 生态（DB/缓存/WS） | ★★★★★ SQLAlchemy 2.0 async / redis-py / APScheduler / uvicorn | ★★★★ egg 插件链齐备 | ★★★★★ 生态最大 | ★★★★★ 最全 |
| 团队技能匹配 | ★★★★ 算法同学主力栈；后端同学 Python 上手成本低 | ★★★★ 主力栈零学习 | ★★★ 需适应 DI 体系 | ★★ 无 Java 背景 |
| 学习成本 | 低（类型注解 + 自动文档） | 低 | 中 | 高 |
| 运维资源占用 | 中（单进程双 Server：REST+WS） | 中 | 中 | 高（JVM） |

## 决策

**采用 Python 3.12 + FastAPI 作为 IndustryAgent 的业务编排层（服务名 `industryagent-server`，REST 端口 7001，WebSocket 推送端口 7002，MCP 以 stdio 方式同机暴露），智能体服务（FastAPI + LangGraph，端口 8300）与之通过 REST 协作；两服务共享 Python 技术栈。**

端口口径随本决策一次性定稿（修正历史文档分歧）：**7001=REST，7002=WebSocket，8300=智能体服务，MCP=stdio（无端口）**。

## 理由

1. **消除双栈**：业务后端与智能体同为 Python/FastAPI，构建、镜像、CI、可观测埋点、依赖治理一套体系；MCP 工具层与智能体同源 SDK；
2. **实施验证的直接结论**：Egg.js 4.x 的 ESM-only 转向使原决策前提（4.0 原生 MCP + 成熟 CJS 插件生态）不成立，回退 3.x 属于为保留框架而妥协；
3. **契约不变，切换风险受控**：43 个 REST 端点、错误码、响应包、种子数据、WS 帧格式全部逐字节兼容（见 IA-TU-001 V2.0 教程），前端/智能体/部署教程接口层零改动；
4. **Pydantic + 自动 OpenAPI 文档**降低了前后端联调与契约漂移成本（原需自建 Swagger）。

## 后果

### 正面

- 单技术栈：一份 Dockerfile 规范、一条 CI 流水线模板、一套依赖安全扫描策略；
- MCP（FastMCP）与智能体侧 SDK 同源，工具注册与调用形态一致；
- 团队技能聚焦，算法与后端同学可互相补位。

### 负面

- Python 运行时性能上限低于 JVM/Go（GIL 下的 CPU 密集场景）——本项目业务层为 IO 密集，风险有限；
- 失去 Egg.js 约定式多进程模型，需以 uvicorn/gunicorn 多 worker 方式等效；
- 原 Egg.js 体系下的存量文档（后端详设/编码规范/教程 V1.x）需同步改写，一次性文档成本约数天。

### 缓解

- CPU 密集计算（孤立森林、因果推断、嵌入）全部留在智能体/独立 worker 进程，业务层仅做 IO 编排；
- 部署采用 uvicorn 多 worker（Compose 单机 1~2 worker，K8s 阶段以副本扩容）；
- 文档同步清单化管理：本 ADR + IA-SD-002 + IA-DEV-002 + IA-TU-001 V2.0 一次变更、交叉引用同步更新。

## 关联

- IA-SD-001 第 3/5 章（总体架构、技术选型汇总）
- IA-SD-002（后端详细设计，V2.0 同步改写）
- ADR-003（LangGraph 编排框架）
- IA-TU-001《后端开发教程》V2.0（FastAPI 版，契约冻结清单）
