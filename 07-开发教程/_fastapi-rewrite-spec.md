# 后端教程 FastAPI 重写：契约冻结与写作规格（工作文件，随 V2.0 发布后移除）

> 本文档是《00-后端开发教程》从 Egg.js 重写为 Python FastAPI 的**唯一事实源**。所有章节写手必须严格遵守本契约；对外行为（路由/错误码/响应包/种子数据/演示剧本）**逐字节兼容**原 Egg.js 版，前端/智能体/数据库/部署教程与验收文档都建立在这些契约之上。

---

## 一、技术选型（定稿，不得更换）

| 层 | 选型 | 说明 |
|---|---|---|
| 语言/运行时 | Python 3.12 | 用户机器已有 3.12.8 |
| Web 框架 | FastAPI + uvicorn[standard] | 与智能体服务同栈 |
| ORM | SQLAlchemy 2.0（async, asyncpg 驱动） | 对标 egg-sequelize |
| 校验 | pydantic v2 + pydantic-settings | .env 与多环境配置 |
| 认证 | PyJWT（HS256→第19章 RS256）+ bcrypt | 兼容种子密码哈希 $2b$ |
| HTTP 客户端 | httpx.AsyncClient | 调智能体，含熔断器自研 |
| 缓存/幂等 | redis-py ≥5（redis.asyncio） | 对标 egg-redis |
| 定时任务 | APScheduler（AsyncIOScheduler） | 对标 egg schedule |
| WS | uvicorn 双 Server 单进程（7001 REST + 7002 WS） | 教学清晰且贴合部署拓扑 |
| MCP | mcp 官方 Python SDK（FastMCP, stdio） | 与智能体教程一致 |
| 指标 | prometheus-client | /metrics + /healthz |
| 对象存储 | minio（Python SDK） | 第 18 章 |
| 测试 | pytest + pytest-asyncio + httpx.AsyncClient(ASGITransport) | 测试库 industryagent_test |
| 建表 | metadata.create_all + scripts/init_db.py 种子 | 对标原 sequelize.sync 方案；Alembic 属数据库教程线 |

依赖管理：requirements.txt（分 requirements.txt / requirements-dev.txt），pip 清华镜像。**不用** poetry/uv（教程读者面）。

## 二、端口与部署口径（定稿，修正历史分裂）

- 后端 REST：**7001**（uvicorn #1）
- 后端 WS 推送：**7002**（uvicorn #2，同进程，run.py 双 Server 启动）
- Python 智能体服务（含第 8 章 Mock 与真实智能体）：**8300**（`.env` 的 `IA_AGENT_BASE_URL=http://127.0.0.1:8300`）——**修正原 Egg 教程的 8000 旧口径**，与《03-智能体开发教程》《04-部署上线教程》V1.1 对齐
- MCP：stdio（无端口）
- PG 5432 / Redis 6379 / MinIO 9000+9001 / Keycloak 8080（沿用 05 安装教程）

## 三、统一响应包与错误码（逐字节兼容）

成功：`{"code":0,"message":"ok","data":...,"traceId":"<uuid>"}`；分页 data：`{"rows":[...],"total":N,"page":1,"pageSize":10}`。

错误码（app/core/error_codes.py 的 ErrorCode 枚举 + DEFAULT_MESSAGES）：OK=0；10xxx 通用（10001 参数/10002 不存在/10003 重复/10004 状态冲突/10005 幂等重复）；20xxx 认证权限（20001 未认证/20002 过期/20003 无权限/20004 密码错/20005 禁用）；30xxx 智能体（30001 不可用/30002 返回异常）；50000 内部错误。BizError(code, message, http_status=400) 对应 Python `BizError(Exception)` + FastAPI 全局 exception_handler。HTTP 映射：10001→422、10002→404、10003/10004/10005→400、20xxx→401/403、50000→500。

## 四、43 端点路由表（18.7.2 原样冻结，逐条实现）

（路径、方法、权限与原教程 18.7.2 完全一致：GET / 、/healthz、/metrics、auth 4 个、wafers 5 个、anomaly-events 2、rootcause 3、suggestions 4、stats/dashboard 1、reports 4、notifications 4、defects 3、dashboard 4、audits 1、users 5，共 43。权限点：wafer:create/update/delete、rootcause:trigger、suggestion:approve、report:edit/release、defect:upload、audit:read、user:read/create/update/reset_password；其余"登录"即可。）

角色-权限映射（第 7 章冻结）：admin=['*']；process_engineer=['wafer:create','wafer:update','rootcause:trigger','report:edit','defect:upload']；reviewer=['rootcause:trigger','suggestion:approve','report:release']；viewer=[]（第 18 章加入）。

## 五、WS 帧契约（定稿，修正前后端分裂）

- 连接：`ws://<host>:7002/?token=<JWT>`；未认证/过期 → 关闭码 4001 reason "unauthorized: invalid or expired token"
- 下行帧：`{"event":"anomaly.push","data":{...},"ts":<unix秒>}`；`{"event":"notification.push","data":{id,type,title,content,sourceId,createdAt},"ts":...}`
- anomaly.push data：`{anomalyId, equipmentCode, title, severity, occurredAt}`
- 心跳：客户端上行 `{"event":"ping"}` → 服务端回 `{"event":"pong","ts":...}`；服务端 30s 轮询两次无应答踢线
- 房间隔离：notification.push 按 userId 定向；anomaly.push 全体广播
- （前端教程后续同步统一到此帧格式——在《01-前端开发教程》9.1/19 章改 useWebSocket 解析，属于另个工作项）

## 六、种子数据（冻结）

- 角色：admin/process_engineer/reviewer（第 18 章加 viewer）；用户：admin/Admin@123（系统管理员）、engineer/Engineer@123（李工（工艺））、reviewer/Reviewer@123（王主管（质量）），邮箱 *@xinlian.demo，bcrypt cost 10
- 批次：B20261015001（91.20，ETCH-03，2026-10-15 08:30）；异常事件：ETCH-03 温度漂移 severity=high
- agent-mock /agent/analyze 响应（8300 端口）：summary 批次良率异常…刻蚀工序参数漂移；root_causes：etch_power_drift 0.87「ETCH-03 刻蚀功率较基准漂移 +3.2%…」/chamber_pressure 0.64/photoresist_expiry 0.41；suggestions：param_adjust「将 ETCH-03 射频功率回调至基准值 -2%」low + hold_wafer「隔离该批次剩余在制晶圆，复测 WAT」medium
- 9.6 十步审批剧本、18 章演示数据的请求/响应逐字沿用原教程各"预期输出"

## 七、认证契约

- POST /api/v1/auth/login {username,password} → data:{accessToken,refreshToken,user:{userId,username,realName,roles:[code]}}
- HS256，access 2h / refresh 7d，IA_JWT_SECRET；refresh token 存 Redis 白名单；logout 拉黑； Authorization: Bearer
- 401 响应体 {code:20001,...}（令牌过期 20002）；权限不足 403 {code:20003,"message":"没有权限：需要 wafer:delete（当前角色：process_engineer）"}
- 第 19 章双模式：AUTH_MODE=local|keycloak（.env），keycloak 模式 PyJWT 验 RS256（JWKS 拉公钥缓存），代理 token 端点方案，回退开关保留

## 八、新教程章节结构（编号与原版一一对应，17 依旧空缺）

1 导读（交付物清单改 Python 栈，强调与智能体同栈的收益）/ 2 环境准备（引 05 安装教程；venv+pip 镜像；目录规划）/ 3 项目初始化与第一个路由（工程结构、pyproject 无关用 requirements.txt、main.py、lifespan、GET / 平台信息 JSON 同原字段 name/version/project/doc/time）/ 4 配置管理（pydantic-settings、.env 同名变量 IA_*、多环境 EGG_SERVER_ENV→IA_ENV=local|prod|test）/ 5 第一个完整模块 wafer（五步套路：Model→错误体系→全局异常处理器→Service→Router→init_db.py→curl 全接口演示，分页/过滤字段与响应逐字同原 5.9）/ 6 中间件：审计日志（BackgroundTasks 异步入库）+traceId 中间件（X-Trace-Id 头）+请求日志 / 7 用户与鉴权（模型/role-permissions.py/依赖注入 Depends(get_current_user)/require_perm("...")/auth service/router/种子/全链路 401·403·refresh·审计验证）/ 8 异常事件与根因分析（模型、agent-mock FastAPI 30 行、自研熔断器类（三态 5s/30s 参数同原）、anomaly+rootcause service（httpx 超时 10s）、任务落库 status 轮询语义同原、熔断演示剧本同原 8.9）/ 9 建议与审批（事务+with_for_update 行锁、十步 curl 剧本逐字、审计落库验证）/ 10 Redis 缓存与幂等（cache-aside 60s、写路径失效、Idempotency-Key 中间件 201/200+Idempotent-Replay 头）/ 11 定时任务（APScheduler 数据健康检查+良率日报，种子与验证同原）/ 12 测试（pytest 单测 mock service；集成测试 ASGITransport+测试库；覆盖率）/ 13 MCP Server（mcp SDK FastMCP stdio：query_anomalies/query_wafers/trigger_rootcause 三工具+scripts/mcp_client.py 验证+接入 Cursor 配置）/ 14 可观测（logging 结构化+traceId 注入、prometheus-client Counter/Histogram、/metrics、/healthz 依赖检查 503 演示）/ 15 构建与部署（tsc 编译验证→无：pytest+uvicorn 冒烟；Dockerfile python:3.12-slim 多阶段非 root+HEALTHCHECK；compose 同原三服务改 Python；幂等重跑验证）/ 16 FAQ（改写为 Python 常见坑：asyncpg 连接池、事件循环、bcrypt 版本、Windows venv、中文字符集等 12+ 条）/ 18 补全模块（18.0 依赖+权限点扩充+multipart；18.1 8D 报告；18.2 通知+WS 双 Server run.py；18.3 缺陷+MinIO ensure_bucket+预签名；18.4 看板四接口+缓存；18.5 审计查询；18.6 用户管理五接口；18.7 全量路由表 43 端点+冒烟脚本 smoke-test.sh 逐字沿用（npm run seed→python scripts/init_db.py））/ 19 Keycloak（19.1 双模式动机；19.2 部署配置同原；19.3 PyJWT RS256 验签改造+jwks；19.4 代理 token 端点；19.5 验证与回退剧本逐字沿用；19.6 小结）

每章骨架四件套：**做什么/为什么 → 完整代码（每个新文件全量收录，不省略）→ 验证（curl/pytest 带预期输出）→ 常见坑**。代码注释解释"为什么"而非"是什么"。Windows/Git Bash 差异单独注明。中文注释。文件路径相对项目根。

## 九、写作风格样例（对齐原教程）

- 章节标题格式：`# N. 标题`、`## N.M 小节`
- 代码块前置一句话说明用途 + 文件路径加粗或反引号
- curl 演示后必须跟 `# 预期输出：...`
- 种子/演示数据延续：B20261015001、ETCH-03、EVT 编号、9.6 剧本
- 教程自身代码风格：模块级 `router = APIRouter()`，service 类静态感弱化用函数或类均可但全库统一为**类**（XxxService，方法 async），依赖注入 `get_db`（async generator）/`CurrentUser = Depends(get_current_user)`

## 十·零、项目目录骨架（全教程统一，写手不得另创结构）

```
industryagent-server/
├─ app/
│  ├─ main.py                  # create_app()：FastAPI 实例、中间件、异常处理、路由挂载、lifespan
│  ├─ run.py                   # 双 uvicorn Server 启动器（7001 REST + 7002 WS，第 18 章起）
│  ├─ core/
│  │  ├─ config.py             # pydantic-settings Settings（IA_* 环境变量）
│  │  ├─ database.py           # async engine + AsyncSessionLocal + get_db 依赖
│  │  ├─ error_codes.py        # ErrorCode 枚举 + DEFAULT_MESSAGES
│  │  ├─ biz_error.py          # BizError 异常类
│  │  ├─ security.py           # JWT 签发/验签、bcrypt、decode_token
│  │  ├─ deps.py               # get_current_user / require_perm("wafer:delete")
│  │  ├─ role_permissions.py   # ROLE_PERMISSIONS 映射
│  │  ├─ circuit_breaker.py    # 熔断器（第 8 章）
│  │  └─ metrics.py            # prometheus 指标注册（第 14 章）
│  ├─ models/                  # SQLAlchemy Declarative 模型（wafer.py/user.py/...）
│  ├─ schemas/                 # Pydantic 请求/响应模型（与教程字段 camelCase 对齐：alias + populate_by_name）
│  ├─ services/                # XxxService 类（业务逻辑、事务、行锁）
│  ├─ routers/                 # APIRouter 按模块（home/auth/wafer/anomaly/rootcause/suggestion/stats/report/notification/defect/dashboard/audit/user）
│  ├─ middleware/              # audit_log.py / request_log.py(traceId) / idempotency.py
│  ├─ ws/                      # manager.py（ConnectionManager，房间+广播+心跳）+ app.py（WS FastAPI 子应用）
│  ├─ tasks/                   # APScheduler 任务（data_health_check.py / yield_daily_report.py）
│  └─ mcp/                     # server.py（FastMCP stdio）
├─ scripts/                    # init_db.py（建表+种子）/ mcp_client.py / smoke-test.sh / make_test_png.py
├─ tests/                      # unit/test_wafer_service.py、integration/test_auth_flow.py
├─ agent-mock/                 # main.py（第 8 章，30 行 FastAPI Mock，端口 8300）
├─ requirements.txt / requirements-dev.txt
├─ .env / .env.example / .gitignore
├─ Dockerfile / docker-compose.yml
└─ keycloak/                   # ia-realm-export.json（第 19 章）
```

**JSON 字段命名规则（重要）**：数据库列 snake_case（同原 sequelize 教程的 camelCase 列名直接映射为同名列——即建表列名沿用原教程 camelCase，如 batchNo/yieldRate/passwordHash，保证种子与前端契约零变化）；Pydantic schema 字段同名 camelCase（无需 alias）。表名同原：wafer_lots? —— 统一用原教程建表名（wafers/users/roles/user_roles/anomaly_events/agent_tasks/suggestions/approval_records/reports_8d/notifications/defect_records/audit_logs/sensor_readings），列名逐字段沿用原教程 5.2/7.2/8.2/9.2/11.2/18.x 的字段定义。

## 十、全局修正清单（写手须知）

1. 原文所有 `npm run`→`python`/`uvicorn`/`pytest` 对应命令；`npm run seed`→`python scripts/init_db.py`
2. 原文 8000（agent mock）→ **8300**
3. 原文响应包/错误码/路径/字段名一律不变（本契约三四节）
4. Egg 术语（agent/worker/messenger/plugin/loader）不得出现在新教程；WS 广播机制说明改为"单进程双 Server 共享 ConnectionManager"
5. 第 15 章 compose 的 server 环境变量改 IA_* + Python 镜像，健康检查 `/healthz`
6. 文档控制块：IA-TU-001，版本 **V2.0**，修订记录新增「V2.0 | 2026-10-18 | 李吉 | 技术栈由 Egg.js 迁移为 Python FastAPI（契约不变）」并保留 V1.0/V1.1 两行
