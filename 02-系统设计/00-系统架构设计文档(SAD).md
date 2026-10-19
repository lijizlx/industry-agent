# IndustryAgent 工业智能体平台 —— 系统架构设计文档（SAD）

## 文档控制块

| 项目 | 内容 |
|---|---|
| 文档编号 | IA-SD-001 |
| 文档名称 | 系统架构设计文档（System Architecture Document） |
| 版本 | V1.0 |
| 密级 | 内部 |
| 编制 | 李吉 |
| 审核 | 　 |
| 批准 | 　 |
| 日期 | 2026-10-15 |

## 修订记录

| 版本 | 日期 | 修订人 | 修订说明 |
|---|---|---|---|
| V0.1 | 2026-09-20 | 李吉 | 初稿，建立文档骨架与总体架构草案 |
| V0.5 | 2026-10-08 | 李吉 | 补充六层复杂度映射、部署视图、横切关注点 |
| V1.0 | 2026-10-15 | 李吉 | 定稿：新增架构风险评估、K8s 演进路线，全文评审修订 |

---

## 1. 引言

### 1.1 编制目的

本文档描述 IndustryAgent（工业智能体平台）v1.0 的总体系统架构，包括分层架构、子系统划分、关键技术选型、部署视图、数据流全景、横切关注点与架构风险评估，作为后续详细设计（IA-SD-002 ~ IA-SD-006）的统一架构基线。

### 1.2 项目背景

IndustryAgent 服务于芯联半导体制造有限公司（虚拟企业沙盒）：

- 12 英寸晶圆厂，3 条产线，月产能 5 万片，产品为 MCU、电源管理芯片；
- 已有系统：MES、ERP、SCADA、FDC、LIMS；
- 业务 KPI：良率目标 95%（当前 93%）、OEE 目标 85%（当前 78%）；
- 核心痛点：根因定位平均 8 小时，目标 4 小时内给出 Top-3 嫌疑；
- 用户角色：工艺工程师、设备工程师、生产主管、质量工程师、系统管理员。

### 1.3 读者对象

后端/前端/AI 算法工程师、DBA、运维工程师、项目经理、架构评审委员会。

### 1.4 参考文档

| 编号 | 名称 |
|---|---|
| [1] | 《企业级工业半导体智能体项目模拟实战手册》v1.0 |
| [2] | 《IndustryAgent 需求规格说明书》IA-RS-001 |
| [3] | 《芯联半导体虚拟企业白皮书》IA-BP-001 |

### 1.5 术语与缩略语

| 术语 | 说明 |
|---|---|
| FDC | Fault Detection & Classification，故障检测与分类 |
| SPC | Statistical Process Control，统计过程控制 |
| RCA | Root Cause Analysis，根因分析 |
| HITL | Human-In-The-Loop，人在回路 |
| 8D | Eight Disciplines，八步法质量问题报告 |
| MCP | Model Context Protocol，模型上下文协议 |
| OEE | Overall Equipment Effectiveness，设备综合效率 |
| MTTR | Mean Time To Repair，平均修复时间 |
| RBAC | Role-Based Access Control，基于角色的访问控制 |
| OPA | Open Policy Agent，开放策略代理 |

---

## 2. 架构目标与约束

### 2.1 架构目标

| 编号 | 目标 | 度量 |
|---|---|---|
| G1 | 提升良率分析效率 | 根因定位从 8h 降至 4h 内，Top-3 命中率 > 80% |
| G2 | 降低 MTTR | MTTR 降低 > 50% |
| G3 | 控制误报 | 误报率 < 10% |
| G4 | 降低人工负担 | 人工接管率 < 30% |
| G5 | 端到端实时性 | 异常事件→Top-3 根因输出延迟 < 5 分钟 |
| G6 | 高可用 | 系统可用性 99.9% |
| G7 | 可审计 | 全链路操作留痕，审批记录完整可回溯 |

### 2.2 硬约束

| 编号 | 约束 | 架构应对 |
|---|---|---|
| C1 | 数据不出域 | 全组件私有化部署（Docker Compose / K8s），LLM 采用本地推理网关（vLLM），禁用外部 SaaS API |
| C2 | 关键建议必须人工审批 | 建议 Agent 输出进入 HITL 审批流，未审批建议不触达执行系统 |
| C3 | RBAC + 全链路审计 | Keycloak 认证 + OPA 授权 + 审计中间件落库 audit_logs |

---

## 3. 总体架构

### 3.1 分层架构图（ASCII）

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                        前端展示层（React 18 + TS + Vite + AntD 5）              │
│  良率看板 │ 异常事件列表 │ 根因分析详情(证据链) │ 建议审批工作台 │ 8D报告 │ 系统管理   │
└──────────────────────────────┬──────────────────────────────────────────────┘
                    HTTPS/REST + WebSocket（JWT Bearer）
┌──────────────────────────────┴──────────────────────────────────────────────┐
│                     业务编排层（Egg.js 4.0 业务后端 + MCP Server）               │
│  auth/user │ wafer │ defect │ sensor │ agent-task │ approval │ report │ audit │
│  ├─ Keycloak OIDC JWT 校验中间件 + OPA 策略中间件                                │
│  ├─ 审计中间件（写 audit_logs）                                                 │
│  └─ MCP Server（工具注册：query_sensors/search_docs/query_graph/run_causality）│
└──────┬───────────────────────────────┬───────────────────────────────────────┘
       │ REST（内部服务令牌）             │ MCP / REST
┌──────┴───────────────────────┐ ┌──────┴──────────────────────────────────────┐
│   智能体服务层                │ │        基础数据服务（既有系统适配）              │
│   Python FastAPI + LangGraph │ │   MES/ERP/SCADA/FDC/LIMS 适配器（REST/DB直连） │
│   ├─ 监控Agent(FDC+SPC+iForest)│ └─────────────────────────────────────────────┘
│   ├─ 根因Agent(图谱+DoWhy+RAG) │
│   ├─ 建议Agent(调参/停线+风险)  │ ┌─────────────────────────────────────────────┐
│   ├─ 审批Agent(HITL中断)       │ │        缓存与消息（Redis / Kafka）             │
│   ├─ 报告Agent(8D生成)         │ │   Redis：会话/令牌/热点缓存/限流计数             │
│   └─ 评估Agent(离线指标)       │ │   Kafka：sensor.raw / fdc.alarm / agent.task  │
└──────┬───────────────────────┘ └─────────────────────────────────────────────┘
       │
┌──────┴──────────────────────────────────────────────────────────────────────┐
│                                存储层                                        │
│  PostgreSQL 16（业务库）+ TimescaleDB（sensor_readings 超表）+ pgvector        │
│  Neo4j 5（知识图谱：设备/工序/参数/缺陷关联）                                    │
│  MinIO（缺陷图 / 工艺文档 / 8D报告PDF 对象存储）                                │
└──────┬──────────────────────────────────────────────────────────────────────┘
       │
┌──────┴──────────────────────────────────────────────────────────────────────┐
│                          数据接入层 / 流处理层                                  │
│  MQTT（设备传感器接入 EMQX）→ Kafka ingest → 流处理（规则引擎+特征计算 Flink/自研）│
│  → TimescaleDB 落库 + 实时推送 WebSocket                                       │
└──────────────────────────────────────────────────────────────────────────────┘

────────────────────────── 横切：可观测性 ─────────────────────────────────────
  OpenTelemetry SDK（全服务埋点）→ Prometheus（指标）+ Grafana（看板/告警）
  + Loki（日志聚合）+ Jaeger（分布式追踪）
────────────────────────── 横切：安全 ─────────────────────────────────────────
  Keycloak（OIDC 认证）+ OPA（RBAC/ABAC 策略）+ TLS + 数据脱敏 + 网络隔离
─────────────────────────────────────────────────────────────────────────────
```

### 3.2 六层复杂度映射表

依据参考手册第 2/3 章的企业级复杂度六层模型，逐层映射本方案实现：

| 复杂度层级 | 真实企业表现 | IndustryAgent 对应实现 |
|---|---|---|
| 业务领域复杂度 | 工艺规范、良率、OEE、排产规则 | 芯联白皮书定义 KPI（良率 95%、OEE 85%）；领域模型覆盖晶圆/批次/Recipe/缺陷/工序；工艺文档知识库 RAG 化 |
| 系统集成复杂度 | MES/ERP/SCADA/PLC/LIMS 多系统 | 适配器模式对接 5 个既有系统；MQTT+Kafka 承载设备数据流；REST 网关承载业务查询；OPC UA 预留扩展点 |
| 数据复杂度 | 时序、图像、日志、图谱、文档；缺失、漂移、噪声 | TimescaleDB 时序 + MinIO 图像/文档 + Neo4j 图谱 + pgvector 向量四模态存储；数据健康检查定时任务注入缺失/漂移/延迟脏数据探针 |
| 算法/智能体复杂度 | 多目标、不确定性、因果、可解释、人机协同 | LangGraph 六智能体编排；Top-3 嫌疑 + 置信度 + 证据链输出；DoWhy 因果推断；HITL 审批中断节点 |
| 工程/运维复杂度 | SLA、高可用、延迟、吞吐、可观测、审计 | 99.9% 可用性目标；Prometheus+Grafana+OTel 全链路可观测；GitHub Actions CI/CD；Docker Compose→K8s 演进 |
| 组织/合规复杂度 | 权限、审批、责任链、数据不出域 | 五角色 RBAC；Keycloak+OPA；审批记录含审批人/时间/理由；全组件私有化部署，数据不出域 |

### 3.3 架构风格与原则

1. **分层解耦**：六层单向依赖，层间通过契约（REST/MCP/Kafka Topic）通信，禁止跨层直连。
2. **混合技术栈分工**：Egg.js 负责业务编排、权限、审计、CRUD；Python FastAPI 负责模型推理与智能体编排；两者经 MCP/REST 解耦。
3. **智能体工具化**：数据库查询、图谱查询、RAG 检索、模型推理一律封装为 MCP 工具，智能体不直连存储。
4. **人在回路**：所有影响产线的建议（调参/停线）必须经审批节点，未审批不生效。
5. **可观测优先**：每个智能体暴露延迟、成功率、人工接管率、Token 消耗四类指标。
6. **演进式部署**：v1.0 Docker Compose 单机编排，v1.5+ 平滑迁移 K8s。

---

## 4. 子系统划分

| 编号 | 子系统 | 职责 | 承载组件 | 对外接口 |
|---|---|---|---|---|
| SS1 | 数据接入子系统 | 设备传感器采集、协议转换、Kafka 入队、流处理与特征计算 | EMQX(MQTT)、Kafka、流处理 Worker | MQTT Topic、Kafka Topic |
| SS2 | 数据存储子系统 | 业务数据、时序数据、向量、图谱、对象的持久化 | PostgreSQL 16+TimescaleDB+pgvector、Neo4j 5、MinIO、Redis | SQL/Cypher/S3 API |
| SS3 | 智能体子系统 | 六智能体编排、HITL 中断、上下文管理、Token 控制 | Python FastAPI + LangGraph | REST /api/v1/agent/*、MCP |
| SS4 | 业务编排子系统 | 用户/晶圆/缺陷/任务/审批/报告/审计/看板八大业务域 | Egg.js 4.0 | REST /api/v1/*、WebSocket、MCP |
| SS5 | 前端展示子系统 | 看板、异常、根因、审批、报告、管理六大页面域 | React 18 + AntD 5 | HTTPS 页面 |
| SS6 | 认证授权子系统 | OIDC 认证、JWT 签发校验、RBAC/ABAC 策略决策 | Keycloak、OPA | OIDC 端点、REST /v1/data/authz |
| SS7 | 可观测子系统 | 指标采集、日志聚合、分布式追踪、告警 | Prometheus、Grafana、OTel Collector、Loki、Jaeger | Prometheus /metrics |

子系统依赖关系：

```text
SS5(前端) ──→ SS4(业务编排) ──→ SS3(智能体) ──→ SS2(存储)
   │              │    │            │              ▲
   │              │    └──→ SS6(认证授权)          │
   │              └──→ SS2(存储)                   │
SS1(数据接入) ─────────────────────────────────────┘
SS7(可观测) ←── 全部子系统（横切）
```

---

## 5. 关键技术选型汇总表

| 领域 | 选型 | 版本 | 选型理由 |
|---|---|---|---|
| 业务后端 | Egg.js | 4.0 | 团队 Node.js 技能栈；企业级工程化（约定式目录/插件/Loader）成熟；4.0 原生支持 MCP Server，工具注册即业务复用；内置 schedule 定时任务。详见 ADR-001 |
| AI 微服务 | Python + FastAPI | 3.11 / 0.110 | LangGraph/LangChain/DoWhy/scikit-learn 等 AI 生态仅 Python 一等公民；FastAPI 异步高吞吐 + Pydantic Schema 即 API 文档 |
| 智能体编排 | LangGraph | 0.2+ | 状态机显式可控、原生 HITL interrupt、checkpointer 持久化、时间旅行调试。详见 ADR-003 |
| 前端 | React + TypeScript + Vite + Ant Design | 18 / 5.x / 5.x | AntD 5 企业中后台组件完备（ProTable/表单/流程）；TS 类型安全；Vite 秒级热更 |
| 关系/时序/向量库 | PostgreSQL + TimescaleDB + pgvector | 16 / 2.x / 0.7 | 一栈多能：业务事务 + 时序压缩 + 向量检索共用一套运维与备份体系。详见 ADR-002 |
| 知识图谱 | Neo4j | 5.x | Cypher 图查询表达设备-工序-参数-缺陷多跳关联；GDS 库支持图算法 |
| 对象存储 | MinIO | RELEASE.2025-x | S3 兼容、私有化部署满足数据不出域；存缺陷图/工艺文档/8D PDF |
| 消息流 | Kafka | 3.7 | 高吞吐背压解耦传感器洪峰（3 产线 × 2k 测点 × 1Hz）；分区保证设备级顺序 |
| 设备接入 | EMQX (MQTT) | 5.x | 工业设备标准协议接入，规则引擎直接桥接 Kafka |
| 缓存 | Redis | 7.x | 热点看板缓存、JWT 吊销名单、分布式锁、接口限流计数 |
| 认证 | Keycloak | 24.x | OIDC/OAuth2 开箱即用，五角色 realm 管理，JWT 签发 |
| 授权 | OPA | 0.64 | 策略即代码（Rego），RBAC+ABAC 统一决策点，策略可单测 |
| 可观测 | Prometheus + Grafana + OpenTelemetry | 最新稳定 | 指标+日志+追踪三支柱；OTel 统一埋点标准，供应商中立 |
| CI/CD | GitHub Actions | — | 仓库内建 CI；lint→test→build→docker push→compose 部署 |
| 部署 | Docker Compose → K8s | v2 / 1.29+ | v1.0 Compose 单机快速闭环；v1.5 迁 K8s 获得自愈与弹性 |

---

## 6. 部署视图

### 6.1 v1.0 Docker Compose 拓扑

```text
┌────────────────────── docker network: ia-backend（内部）──────────────────────┐
│                                                                              │
│  ┌──────────┐   ┌──────────┐   ┌────────────┐   ┌─────────────┐             │
│  │ postgres │   │  neo4j   │   │   minio    │   │    redis    │             │
│  │ :5432    │   │ :7474/  │   │ :9000/     │   │ :6379       │             │
│  │ TS+vec   │   │  7687    │   │ :9001      │   │             │             │
│  └────┬─────┘   └────┬─────┘   └─────┬──────┘   └──────┬──────┘             │
│       └──────────────┴───────┬───────┴─────────────────┘                    │
│                             │                                                │
│  ┌──────────┐        ┌──────┴──────┐        ┌──────────────┐                │
│  │  kafka   │◄───────┤ ai-service  │───────►│  egg-server  │                │
│  │ :9092    │ (ZK内置)│ FastAPI     │  MCP/  │ :7001        │                │
│  └────┬─────┘        │ LangGraph   │  REST  │ + MCP :7002  │                │
│       │              │ :8000       │        └──────┬───────┘                │
│  ┌────┴─────┐        └──────┬──────┘               │                        │
│  │  emqx    │               │                      │                        │
│  │ :1883    │        ┌──────┴──────┐               │                        │
│  │ :8083    │        │  vllm-llm   │ 本地大模型推理（数据不出域）               │
│  └──────────┘        │ :11434      │               │                        │
│                      └─────────────┘               │                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────┴───────┐                │
│  │prometheus│  │ grafana   │  │ keycloak │  │   web (前端)  │                │
│  │ :9090    │  │ :3000     │  │ :8080    │  │ nginx :80/443│──► 用户浏览器     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                                   │
│  │ otel-col │  │ loki     │  │ jaeger   │     （可观测三支柱）                 │
│  └──────────┘  └──────────┘  └──────────┘                                   │
└──────────────────────────────────────────────────────────────────────────────┘
外部数据源：MES/ERP/SCADA/FDC/LIMS 通过内网 REST / DB 视图 / MQTT 桥接接入
```

关键部署约束：

- 仅 `web(nginx)`、`keycloak`、`grafana`、`emqx` 端口暴露于 `ia-edge` 网络，其余全部内网隔离；
- Postgres 挂载 `pgdata` 卷，Neo4j 挂载 `neo4j-data` 卷，MinIO 挂载 `minio-data` 卷，每日 `pg_dump` + `minio mc mirror` 备份至备份卷；
- Kafka 单机 3 分区起步，`sensor.raw` Topic 预留 12 分区以支撑产线扩容。

### 6.2 K8s 演进视图（v1.5+）

| 维度 | Compose（v1.0） | K8s（v1.5+） |
|---|---|---|
| 无状态服务 | 单副本容器 | Deployment 多副本 + HPA（CPU/队列深度自动扩缩） |
| 数据库 | 单实例卷挂载 | StatefulSet + 云盘 PV，或托管实例；只读副本分离报表负载 |
| 智能体服务 | 单副本 | 按 Agent 类型独立 Deployment，Kafka 消费组再平衡横向扩展 |
| 发布 | compose pull + up | GitOps（ArgoCD）蓝绿/金丝雀发布 |
| 故障恢复 | restart: always | liveness/readiness 探针自愈 + PodDisruptionBudget |
| 配置 | .env 文件 | ConfigMap + Secret（Vault 加密） |

---

## 7. 数据流全景图

```text
【主链路：异常检测→根因→建议→审批→报告】
设备传感器 ──MQTT──► EMQX ──桥接──► Kafka(sensor.raw)
    Kafka(sensor.raw) ──► 流处理Worker ──► TimescaleDB(sensor_readings)
                             │
                             ├─► 监控Agent(FDC规则+SPC+iForest) ──► Kafka(fdc.alarm)
                             │        │
Kafka(fdc.alarm) ────────────┘        ▼
  Egg.js 业务编排：创建 agent_tasks(状态=PENDING) ──REST──► ai-service(LangGraph)
        LangGraph 编排：
        监控复核 → 根因Agent ──MCP工具──► query_graph(Neo4j) / search_docs(pgvector)
        │                              / run_causality(DoWhy) / query_sensors(Timescale)
        ├──► rootcause_results(Top-3 + 置信度 + 证据链JSON)
        ├──► 建议Agent ──► suggestions(调参/停线 + 风险等级 + 影响晶圆数)
        ├──► 审批Agent(HITL interrupt) ──WebSocket──► 审批工作台(人工确认)
        │        ├── 批准 → 执行通知(工艺工程师现场调参) + audit_logs
        │        └── 驳回 → 反馈回流评估Agent
        └──► 报告Agent ──► reports_8d(D1-D8, 引用证据) ──► MinIO(PDF归档)

【辅链路1：知识沉淀】
工艺文档(Word/PDF) ──上传──► MinIO ──解析分块+embedding──► doc_chunks(pgvector)
设备/工序/参数/缺陷主数据 ──同步──► Neo4j 知识图谱

【辅链路2：看板】
Egg.js 定时任务(良率日报/数据健康检查) ──► 聚合结果写Redis缓存 ──► 前端看板轮询/WS推送

【辅链路3：反馈闭环】
人工接管/驳回/修正 ──► 评估Agent(离线) ──► 指标: Top-3准确率/误报率/人工接管率 ──► Grafana
```

---

## 8. 横切关注点

### 8.1 鉴权与授权

```text
浏览器 ──1.登录──► Keycloak(OIDC Authorization Code + PKCE)
       ◄──2.JWT(access_token: sub/roles/exp, refresh_token)
浏览器 ──3.API请求(Bearer JWT)──► Egg.js 鉴权中间件
   ├─4.本地校验签名/过期/aud（JWKS公钥）
   ├─5.查Redis黑名单(吊销检查)
   └─6.OPA策略决策(input={user,role,resource,action})
        ├─ allow → 进入Controller + 审计中间件记录
        └─ deny  → 403 + 审计记录(拒绝也留痕)
```

### 8.2 审计

- 审计中间件拦截全部写操作（POST/PUT/DELETE）及敏感读（根因证据、8D 报告）；
- 审计记录字段：操作人、时间、IP、方法、路径、请求摘要（脱敏）、响应码、耗时；
- `audit_logs` 仅追加（append-only），按月分区，保留 3 年。

### 8.3 可观测性

| 支柱 | 实现 | 关键指标 |
|---|---|---|
| 指标 | Prometheus + Grafana | API P95 延迟、Agent 各节点耗时/成功率、Kafka lag、Top-3 准确率（离线回填）、人工接管率、Token 成本 |
| 日志 | pino(JSON) → Loki | trace_id 贯穿全链路日志检索 |
| 追踪 | OpenTelemetry → Jaeger | 异常事件从 MQTT 入站到 8D 生成的完整 span 链 |

### 8.4 安全

- 传输：全链路 TLS；内网服务间 mTLS（K8s 阶段启用 service mesh 可选）；
- 数据：静态加密（卷加密）、敏感字段脱敏（手机号/邮箱掩码）、MinIO 预签名 URL 限时访问；
- 数据不出域：LLM 本地 vLLM 推理；禁用外部 DNS 白名单外请求（egress 策略）；
- 应用：SQL 参数化（Sequelize/psycopg）、Cypher 参数化、上传文件类型白名单 + 病毒扫描预留。

---

## 9. 架构风险评估

| 编号 | 风险 | 等级 | 影响 | 缓解措施 |
|---|---|---|---|---|
| R1 | 传感器数据洪峰导致 Kafka 积压，根因分析超时（>5min） | 高 | 违反 G5 延迟目标 | 分区扩容预案；消费者 Lag 告警（>10万条触发）；流处理降级为抽样模式 |
| R2 | LLM 本地模型推理能力有限，根因解释质量不足 | 高 | Top-3 准确率 < 80% | 根因 Agent 以确定性工具（图谱+因果）为主、LLM 仅做综合表述；评估 Agent 每周回归 |
| R3 | 双技术栈（Node+Python）运维复杂度上升 | 中 | 发布/排障成本增加 | 统一 Docker 镜像规范与 OTel 埋点标准；MCP 契约驱动联调 |
| R4 | 知识图谱冷启动数据不足，多跳推理召回低 | 中 | 证据链薄弱 | 上线前批量导入设备/工序主数据；SECOM/历史 8D 报告回灌图谱 |
| R5 | 工程师不信任智能体建议，人工接管率超标 | 中 | 违反 G4 | 强制证据链展示；上线初期影子模式（只提示不执行）建立信任 |
| R6 | Neo4j 单点故障 | 中 | 图谱查询不可用 | 每日全量导出 GraphML 备份；降级策略：根因 Agent 跳过图谱证据并降置信度 |
| R7 | 审批超时阻塞流程 | 低 | MTTR 恶化 | 审批 Agent 30min 未响应升级通知（主管→厂长），2h 自动转人工工单 |
| R8 | 误报风暴导致审批疲劳 | 低 | 用户体验差 | 监控 Agent 三模型投票（FDC+SPC+iForest 至少 2/3 触发才告警）；误报率>10% 自动收紧阈值 |

---

## 10. 架构决策记录索引

| ADR 编号 | 决策主题 | 状态 |
|---|---|---|
| ADR-001 | 业务后端选择 Egg.js 4.0 | 已接受 |
| ADR-002 | 数据库选择 PostgreSQL+TimescaleDB+pgvector 一栈式 | 已接受 |
| ADR-003 | 智能体编排选择 LangGraph + Egg.js MCP 工具层 | 已接受 |

（详见 `06-架构决策记录(ADR)/` 目录。）

---

## 11. 附录

### 11.1 智能体链路总览（与 IA-SD-006 口径一致）

监控Agent(FDC异常检测) → 根因Agent(Neo4j图谱 + DoWhy因果 + RAG检索 → Top-3嫌疑) → 建议Agent(调参/停线建议 + 风险评估) → 审批Agent(人工确认 + 审计记录) → 报告Agent(自动生成 8D 报告)，另有评估 Agent 离线闭环。

### 11.2 评估指标基线（与需求口径一致）

| 指标 | 目标值 |
|---|---|
| 根因 Top-3 准确率 | > 80% |
| MTTR 降低 | > 50% |
| 误报率 | < 10% |
| 人工接管率 | < 30% |
| 端到端延迟 | < 5 分钟 |
| 可用性 | 99.9% |

（全文完）
