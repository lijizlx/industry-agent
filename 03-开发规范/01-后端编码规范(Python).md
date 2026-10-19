# 后端编码规范（Python FastAPI）

| 文档控制 | 内容 |
|---|---|
| 文档编号 | IA-DEV-002 |
| 文档名称 | 后端编码规范（Python） |
| 版本 | V2.0 |
| 密级 | 内部 |
| 编制 | 李吉 |
| 审核 | （待填） |
| 批准 | （待填） |
| 日期 | 2026-10-18 |

## 修订记录

| 版本 | 日期 | 修订内容 | 编制/修订人 | 审核人 |
|---|---|---|---|---|
| V1.0 | 2026-10-15 | 初始版本发布（Egg.js 技术栈） | 李吉 | — |
| V2.0 | 2026-10-18 | 技术栈 Egg.js→Python FastAPI 全面改写 | 李吉 | — |

---

本规范适用于 `industryagent-server`：Python 3.12 + FastAPI + uvicorn + SQLAlchemy 2.0（async, asyncpg）+ pydantic v2 + PyJWT/bcrypt + httpx + redis-py + APScheduler + pytest。技术选型以《07-开发教程》FastAPI 重写契约为唯一事实源，不得更换。与 IA-DEV-001《代码规范总则》冲突时以本规范为准。

## 1. 目录与分层约定

### 1.1 分层架构

```text
Router（薄） → Service（厚） → Model（SQLAlchemy 2.0 async）+ Schema（Pydantic v2）
     ↓               ↓                      ↓
 依赖注入/中间件    httpx → 智能体服务(8300)     PostgreSQL / Redis / MinIO
（get_db/审计/traceId）  redis-py / MinIO SDK
```

| 层 | 职责 | 禁止事项 |
|---|---|---|
| Router | 参数提取与校验声明、权限标注、调 Service、组装响应 | 禁止写业务逻辑、禁止直接操作数据库、禁止 import 其他 Router |
| Service | 业务逻辑、事务与行锁、外部服务调用（httpx/Redis）、数据聚合 | 禁止读写 Request/Response、禁止手拼响应 JSON、禁止 import fastapi 的 Depends/Request |
| Model | 数据表定义（Declarative）、单表数据访问 | 禁止跨表 JOIN 业务语义、禁止调用 Service |
| Schema | 请求/响应结构与字段校验（camelCase 契约字段） | 禁止写业务逻辑、禁止把 ORM 对象直接透传给响应 |

**禁止跨层调用**：Router 不得绕过 Service 直接查 Model；Service 不得复用另一个 Service 的私有方法（公共逻辑下沉为独立 Service 或放入 `app/core/`）。

### 1.2 目录结构（与《07-开发教程》FastAPI 重写骨架一致，不得另创结构）

```text
industryagent-server/
├── app/
│   ├── main.py                # create_app()：FastAPI 实例、中间件、异常处理、路由挂载、lifespan
│   ├── run.py                 # 双 uvicorn Server 启动器（7001 REST + 7002 WS）
│   ├── core/                  # config / database / error_codes / biz_error / security / deps /
│   │                          # role_permissions / circuit_breaker / metrics（.py）
│   ├── models/                # SQLAlchemy Declarative 模型（wafer.py / user.py / ...）
│   ├── schemas/               # Pydantic 请求/响应模型（字段名即 camelCase 契约，无需 alias）
│   ├── services/              # XxxService 类（业务逻辑、事务、行锁）
│   ├── routers/               # APIRouter 按模块（home/auth/wafer/anomaly/rootcause/suggestion/
│   │                          # stats/report/notification/defect/dashboard/audit/user）
│   ├── middleware/            # audit_log.py / request_log.py(traceId) / idempotency.py
│   ├── ws/                    # manager.py（ConnectionManager）+ app.py（7002 WS 子应用）
│   ├── tasks/                 # APScheduler 任务（data_health_check.py / yield_daily_report.py）
│   └── mcp/                   # server.py（FastMCP stdio）
├── scripts/                   # init_db.py（建表+种子）/ mcp_client.py / smoke-test.sh
├── tests/                     # unit/ + integration/（测试库 industryagent_test）
├── agent-mock/                # 智能体 Mock（端口 8300，教程第 8 章）
├── requirements.txt / requirements-dev.txt
├── .env / .env.example / .gitignore / Dockerfile / docker-compose.yml
```

### 1.3 "代码放哪"判定表

| 要写的代码 | 放哪 | 判定要点 |
|---|---|---|
| 路由声明、参数提取、权限标注 | `app/routers/<模块>.py` | 函数体以"取参 → 调 Service → return"为限 |
| 业务规则、事务、行锁、数据聚合 | `app/services/xxx_service.py` | 抛 BizError 的唯一层；事务边界唯一合法位置 |
| 表结构、列映射、索引 | `app/models/*.py` | 只描述数据，不含行为 |
| 请求/响应结构、字段校验规则 | `app/schemas/*.py` | 字段名即前端契约（camelCase），禁止改名 |
| JWT/bcrypt、当前用户、权限点 | `app/core/security.py` / `deps.py` / `role_permissions.py` | 认证授权横切逻辑 |
| 错误码枚举、BizError | `app/core/error_codes.py` / `biz_error.py` | 错误码登记制唯一入口 |
| 配置项（IA_* 环境变量） | `app/core/config.py` | pydantic-settings |
| 审计 / traceId / 幂等中间件 | `app/middleware/` | HTTP 横切关注点 |
| WS 推送 / 定时任务 / MCP 工具 | `app/ws/` / `app/tasks/` / `app/mcp/` | 各自独立子系统 |
| 建表/种子/冒烟脚本 | `scripts/` | 不进运行时包 |
| 单元/集成测试 | `tests/unit/`、`tests/integration/` | 与被测模块同构命名 |
| 跨模块通用工具 | `app/core/` | 就近原则：仅单模块使用的留在该模块内 |

### 1.4 复杂度与行数阈值（CI 以 ruff 强制检查）

| 指标 | 阈值 | 超出处理 |
|---|---|---|
| 函数行数（不含空行/注释） | ≤ 50（后端从严，专项规范优先于总则第 6 章） | 必须拆分 |
| 圈复杂度 | ≤ 15 | 拆分为多个函数或引入策略表 |
| 函数参数个数 | ≤ 5 | 超出时收拢为 Pydantic 模型或 dataclass |
| 嵌套深度 | ≤ 4 | 提前返回（guard clause）拍平 |
| 单文件行数 | ≤ 500（含注释） | 拆分模块 |

```toml
# pyproject.toml 的 ruff 关键配置（配合 IA-DEV-001 第 8 章 lint-staged 的 ruff check --fix）
[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "S", "ASYNC", "T20", "G", "BLE"]  # T20 禁 print，BLE 禁裸 except，G 禁日志 f-string
[tool.ruff.lint.mccabe]
max-complexity = 15
```

## 2. 代码模板

### 2.1 Router 模板（薄）

```python
# app/routers/wafer.py
from fastapi import APIRouter, Depends, Query

from app.core.database import get_db
from app.core.deps import DbSession, CurrentUser, get_current_user, require_perm
from app.schemas.wafer import WaferCreateRequest, WaferItemResponse, WaferListResponse
from app.services.wafer_service import WaferService

router = APIRouter(prefix="/api/v1/wafers", tags=["wafers"])


@router.get("", response_model=WaferListResponse)
async def list_wafers(
    page: int = Query(default=1, ge=1),
    pageSize: int = Query(default=10, ge=1, le=100),
    batchNo: str | None = Query(default=None),
    db: DbSession = Depends(get_db),
    user: CurrentUser = Depends(get_current_user),   # 登录即可访问
) -> WaferListResponse:
    """分页查询批次列表（GET /api/v1/wafers）。仅做参数提取与调 Service。"""
    return await WaferService(db).list(page, pageSize, batchNo)


@router.post("", response_model=WaferItemResponse, status_code=201,
             dependencies=[require_perm("wafer:create")])   # 权限点标注
async def create_wafer(payload: WaferCreateRequest, db: DbSession = Depends(get_db)) -> WaferItemResponse:
    """创建批次（POST /api/v1/wafers）。校验失败由 Pydantic 自动 422，无需 try/except。"""
    return await WaferService(db).create(payload)
```

```python
# ❌ 反例：胖 Router——手写 if 校验 + 业务规则 + text() 拼接 SQL + 手拼响应包
if not payload.get("batchNo") or len(payload["batchNo"]) < 6:        # 手写校验散落路由
    return {"code": 10001, "message": "批次号不合法"}
await db.execute(text(f'INSERT INTO wafers ("batchNo") VALUES (\'{payload["batchNo"]}\')'))  # SQL 拼接
return {"code": 0, "message": "ok", "data": None, "traceId": "-"}   # 手拼响应包破坏契约
```

### 2.2 Service 模板（厚）

```python
# app/services/wafer_service.py（imports 略：logging / select / AsyncSession / BizError / ErrorCode / Model / Schema）
logger = logging.getLogger(__name__)


class WaferService:
    """批次业务服务：批次生命周期、良率统计、根因分析入口。"""

    def __init__(self, db: AsyncSession) -> None:
        self.db = db

    async def create(self, payload: WaferCreateRequest) -> Wafer:
        """创建批次。Raises BizError: 10003 批次号已存在。"""
        stmt = select(Wafer).where(Wafer.batch_no == payload.batchNo)
        if await self.db.scalar(stmt) is not None:
            raise BizError(ErrorCode.DUPLICATED, f"批次号 {payload.batchNo} 已存在")
        wafer = Wafer(**payload.model_dump())
        self.db.add(wafer)
        await self.db.flush()  # 只拿主键，不提交；提交归事务块（见第 5 章）
        logger.info("[WaferService] wafer created, id=%s batchNo=%s", wafer.id, wafer.batch_no)
        return wafer
```

### 2.3 Model 模板（SQLAlchemy 2.0 async）

```python
# app/models/wafer.py（Declarative Base 全项目仅定义一处，随教程第 5 章落位 models 包）
from datetime import datetime

from sqlalchemy import BigInteger, DateTime, Numeric, String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from decimal import Decimal


class Base(DeclarativeBase):
    """全项目唯一 Base：metadata.create_all 建表依据。"""


class Wafer(Base):
    """晶圆批次表 wafers。表名/列名沿用既有契约（列名 camelCase，见 IA-DEV-006）。"""

    __tablename__ = "wafers"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)
    # Python 属性 snake_case，契约列名 camelCase 经位置参数映射写死，禁止随手改名
    batch_no: Mapped[str] = mapped_column("batchNo", String(32), nullable=False, unique=True,
                                          comment="批次号，厂内唯一")
    yield_rate: Mapped[Decimal | None] = mapped_column("yieldRate", Numeric(6, 4),
                                                       comment="良率，0-1，4 位小数精度")
    status: Mapped[str] = mapped_column(String(16), nullable=False, default="pending",
                                        comment="状态：pending/running/completed/scrapped")
    created_at: Mapped[datetime] = mapped_column("createdAt", DateTime, server_default=func.now())
```

## 3. 参数校验与 Schema（Pydantic）

统一交给 Pydantic：Query/Path 参数用 `Query(ge=..., le=...)`，Body 用模型 + `Field` 约束；Schema 只放 `app/schemas/`，字段名即前端契约（camelCase，无需 alias）。校验失败由全局异常处理器统一转 `422 + 10001`（第 4 章）。

```python
# app/schemas/wafer.py —— ✅ 正例：声明式校验，路由零 if；响应模型直接吃 ORM 对象
class WaferCreateRequest(BaseModel):
    """创建批次请求体：字段名即契约，禁止改名或私自增删。"""
    batchNo: str = Field(min_length=6, max_length=32, pattern=r"^[A-Z0-9-]+$")
    plannedQty: int = Field(ge=1, le=25)


class WaferItemResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    batchNo: str
    yieldRate: float | None = None
    status: str
    createdAt: datetime


# ❌ 反例：手写 if 校验散落各处，错误码/文案随手拼
if not batchNo or not isinstance(batchNo, str) or len(batchNo) < 6:
    return JSONResponse(status_code=400, content={"code": 10001, "message": "批次号不合法"})
```

分页响应 data 结构契约冻结：`{"rows":[...],"total":N,"page":1,"pageSize":10}`。禁止在 Service 重复校验请求字段；跨字段/依赖库表的业务校验（如状态机流转）放 Service，失败抛 `BizError` 而非 422。

## 4. 错误处理

### 4.1 统一业务错误类 BizError

```python
# app/core/biz_error.py
from app.core.error_codes import DEFAULT_MESSAGES


class BizError(Exception):
    """业务异常：所有可预期业务错误必须抛 BizError，由全局异常处理器统一转为响应包。"""

    def __init__(self, code: int, message: str | None = None, http_status: int = 400) -> None:
        self.code = code                                   # 业务错误码，如 10003
        self.message = message or DEFAULT_MESSAGES.get(code, "业务处理失败")
        self.http_status = http_status
        super().__init__(f"[{code}] {self.message}")
```

### 4.2 错误码段分配（与重写契约逐字一致）

| 段 | 含义 | 冻结错误码 | HTTP 映射 |
|---|---|---|---|
| 0 | 成功 | 0 ok | 200/201 |
| 10xxx | 通用 | 10001 参数校验失败 / 10002 资源不存在 / 10003 数据重复 / 10004 状态冲突 / 10005 幂等重复 | 10001→422、10002→404、10003/10004/10005→400 |
| 20xxx | 认证与权限 | 20001 未认证 / 20002 令牌过期 / 20003 无权限 / 20004 密码错误 / 20005 账号禁用 | 401/403 |
| 30xxx | 智能体服务 | 30001 智能体不可用 / 30002 智能体返回异常 | 默认 400，可按语义指定 |
| 50000 | 系统内部 | 50000 内部错误 | 500 |

统一响应包（逐字节兼容）：成功 `{"code":0,"message":"ok","data":...,"traceId":"<uuid>"}`；错误 `{"code":...,"message":...,"data":null,"traceId":...}`。

**错误码登记制**：新增错误码必须先登记 `app/core/error_codes.py`（`ErrorCode` 枚举 + `DEFAULT_MESSAGES` 默认文案）并同步 IA-DEV-007 错误码分配表，禁止在调用处裸写数字、禁止私有造码。

### 4.3 全局异常处理器（完整代码）

```python
# app/main.py 节选：register_exception_handlers(app)，在 create_app() 内注册
import logging

from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

from app.core.biz_error import BizError
from app.core.error_codes import ErrorCode

logger = logging.getLogger(__name__)


def register_exception_handlers(app: FastAPI) -> None:
    @app.exception_handler(BizError)
    async def biz_error_handler(request: Request, exc: BizError) -> JSONResponse:
        logger.warning("[biz-error] traceId=%s code=%s msg=%s",   # 可预期错误记 warn
                       request.state.trace_id, exc.code, exc.message)
        return JSONResponse(status_code=exc.http_status, content={
            "code": exc.code, "message": exc.message, "data": None, "traceId": request.state.trace_id})

    @app.exception_handler(RequestValidationError)
    async def validation_handler(request: Request, exc: RequestValidationError) -> JSONResponse:
        detail = "; ".join(f"{'.'.join(map(str, e['loc']))}: {e['msg']}" for e in exc.errors())
        return JSONResponse(status_code=422, content={"code": ErrorCode.PARAM_INVALID,
            "message": f"参数校验失败：{detail}", "data": None, "traceId": request.state.trace_id})

    @app.exception_handler(Exception)
    async def unhandled_handler(request: Request, exc: Exception) -> JSONResponse:
        logger.exception("[unhandled] traceId=%s", request.state.trace_id)  # 堆栈只进日志
        return JSONResponse(status_code=500, content={
            "code": ErrorCode.INTERNAL_ERROR, "message": "系统内部错误，请联系管理员",
            "data": None, "traceId": request.state.trace_id})
```

### 4.4 异常处理禁止事项

```python
# ❌ 反例 1：裸 except 吞掉所有异常（ruff BLE001/E722 拦截）——不打日志、不上抛、破坏响应包契约
try:
    await WaferService(db).create(payload)
except Exception:
    pass          # 或 return {"ok": False}：错误无处排查，前端无从下手

# ❌ 反例 2：把堆栈/内部信息返回前端（泄漏实现细节）
except Exception as e:
    return {"code": 50000, "message": str(e), "stack": traceback.format_exc()}

# ✅ 正例：可预期错误抛 BizError；不可预期错误不要捕获，交给全局处理器记日志并兜底 50000
```

## 5. 事务使用规范

1. 事务边界必须在 **Service 层** 闭合，统一 `async with db.begin():`——进入开事务、正常退出自动提交、异常自动回滚（该上下文即本项目的 UoW）。**Service 禁止手工 `commit()`/`rollback()`**，提交与回滚由事务块或最外层调用方控制；Router 永不碰事务。
2. 跨多个 Service 的编排由最外层调用方包一个事务块，内层复用同一 session 不再嵌套 `begin()`；确需部分提交用 `begin_nested()`（SAVEPOINT）。
3. 一个事务内 SQL 条目数 ≤ 10；事务内禁止调智能体 HTTP、发消息等慢 IO（httpx 调用放事务块外）。
4. `with_for_update()` 行锁时机：审批、状态流转、幂等更新等**读-改-写竞争路径必加**；纯只读查询与无并发竞争的后台写入**禁加**（避免锁放大拖垮吞吐）。

```python
# ✅ 正例：教程第 9 章审批流做法——事务块闭合 + 行锁防并发重复审批
async def approve(self, suggestion_id: int, action: str) -> None:
    async with self.db.begin():  # UoW：退出即提交，异常即回滚
        stmt = select(Suggestion).where(Suggestion.id == suggestion_id).with_for_update()
        suggestion = (await self.db.execute(stmt)).scalar_one_or_none()
        if suggestion is None:
            raise BizError(ErrorCode.NOT_FOUND, f"建议不存在: {suggestion_id}", http_status=404)
        if suggestion.status != "pending":
            raise BizError(ErrorCode.STATUS_CONFLICT, f"建议当前状态不允许审批: {suggestion.status}")
        suggestion.status = "approved" if action == "approve" else "rejected"
        self.db.add(ApprovalRecord(suggestionId=suggestion_id, action=action))

# ❌ 反例：Service 手工提交、失败不回滚、事务内调智能体（慢 IO 拉长持锁时间）
wafer.status = "scrapped"
await self.db.commit()                     # Service 越权提交；后续抛异常则数据不一致
await self.agent_client.analyze(event_id)  # 应移出事务块
```

## 6. 依赖注入规范

```python
# app/core/database.py —— 会话依赖（async generator，只发会话不管事务）
async def get_db() -> AsyncIterator[AsyncSession]:
    async with AsyncSessionLocal() as session:
        yield session


# app/core/deps.py —— 当前用户与权限点依赖（imports 略）
DbSession = Annotated[AsyncSession, Depends(get_db)]
CurrentUser = Annotated[User, Depends(get_current_user)]


async def get_current_user(token: Annotated[str, Depends(bearer_scheme)], db: DbSession) -> User:
    payload = decode_token(token)  # 过期抛 20002、无效抛 20001（见 10.1）
    user = await UserService(db).get_by_id(payload["sub"])
    if user is None or user.disabled:
        raise BizError(ErrorCode.USER_DISABLED, "账号已禁用", http_status=401)
    return user


def require_perm(perm: str):
    """权限点依赖工厂：403 文案格式与认证契约逐字一致。"""
    async def _checker(user: CurrentUser) -> User:
        if not ROLE_PERMISSIONS.has_perm(user.role_codes, perm):
            raise BizError(ErrorCode.FORBIDDEN,
                           f"没有权限：需要 {perm}（当前角色：{'/'.join(user.role_codes)}）",
                           http_status=403)
        return user
    return Depends(_checker)
```

用法约定：

1. 需登录：`user: CurrentUser = Depends(get_current_user)`；需权限点：`dependencies=[require_perm("wafer:delete")]`。权限点与角色映射集中维护在 `app/core/role_permissions.py` 的 `ROLE_PERMISSIONS`（admin=`['*']`，process_engineer/reviewer/viewer 按契约映射），禁止散落硬编码。
2. Service 不感知 FastAPI：只接受 db 等显式参数，禁止在 Service/Model 里 import `Depends`/`Request`。
3. 测试覆盖依赖统一走 `app.dependency_overrides`（见第 13 章），禁止用环境变量分支切换依赖。

## 7. 异步规范

| 场景 | 正例 | 反例 |
|---|---|---|
| 延时等待 | `await asyncio.sleep(2)` | `time.sleep(2)`（阻塞整个事件循环，全部并发请求卡死） |
| HTTP 调用 | 进程级 `httpx.AsyncClient` 复用 | `requests.get()`（同步库禁入请求路径） |
| 数据库 / Redis | asyncpg / AsyncSession、`redis.asyncio` | psycopg2 等同步驱动、请求路径用同步 `redis.Redis` |
| CPU 密集/重计算 | BackgroundTasks / `run_in_threadpool` / 任务队列 | 路由内直接跑推理、pandas 全表计算 |
| 文件读写 | `aiofiles` | 请求路径 `open()` 读写大文件 |

```python
# app/main.py —— lifespan 内创建进程级客户端并随应用关闭（教程第 8 章：调智能体 8300，超时 10s）
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = httpx.AsyncClient(base_url=settings.AGENT_BASE_URL, timeout=10.0)
    app.state.redis = Redis.from_url(settings.REDIS_URL, decode_responses=True)
    yield
    await app.state.http.aclose()
    await app.state.redis.aclose()

# ❌ 反例：每请求新建客户端（连接池作废，高并发下句柄耗尽）
async with httpx.AsyncClient() as client:   # ✅ 应复用 app.state.http
    resp = await client.post("/agent/analyze", json={...})
```

## 8. 定时任务规范

统一 APScheduler（`AsyncIOScheduler`，在 lifespan 注册），任务实现放 `app/tasks/`；禁止请求路径或模块导入时 `asyncio.create_task` 自建轮询、`while True + sleep` 循环、`threading.Timer`。

```python
# app/tasks/data_health_check.py —— 要点示意（多实例用 Redis 锁保证唯一执行者）
async def run_data_health_check(db, redis, metrics) -> None:
    """数据健康检查：每 30 分钟扫描缺失批次读数。"""
    acquired = await redis.set("schedule:data-health-check", "1", ex=600, nx=True)  # SET NX EX，TTL 略大于执行时长
    if not acquired:
        return
    try:
        missing = await DataHealthService(db).scan_missing_readings()
        logger.info("[schedule] data health checked, missing=%d", len(missing))
    except Exception:
        logger.exception("[schedule] data health check failed")   # 定时任务是唯一允许兜底捕获 Exception 的场景
        metrics.schedule_failure_total.inc({"job": "data-health-check"})  # 失败告警接 Prometheus
    finally:
        await redis.delete("schedule:data-health-check")

# 注册（lifespan 内）：scheduler.add_job(run_data_health_check_job, "cron", minute="*/30")
```

## 9. 日志规范

| 规则 | 说明 |
|---|---|
| 禁止 print | 一律 `logger = logging.getLogger(__name__)`；ruff T201 拦截 |
| 分级 | `debug` 开发诊断；`info` 关键业务动作；`warning` 可预期异常（BizError）；`error`/`exception` 系统错误（后者自动带堆栈） |
| 结构化 + 惰性格式化 | `%s`/`%d` 占位参数，统一 JSON 格式器输出供采集；禁止 f-string/拼接（ruff G004） |
| 必带上下文 | traceId、lotId 等关键标识必须出现；traceId 由中间件经 X-Trace-Id 头写入 `contextvars`，日志侧统一读取 |
| 禁止敏感信息 | password、token、apiKey、客户敏感字段禁止入日志 |

```python
# ✅ 正例
logger.info("[WaferService] wafer created, traceId=%s id=%s batchNo=%s",
            get_trace_id(), wafer.id, wafer.batch_no)
logger.exception("[AgentClient] rootcause failed, traceId=%s", get_trace_id())

# ❌ 反例
print("created!", wafer)                                   # print + 整对象输出
logger.info(f"created: {json.dumps(wafer, default=str)}")  # 拼接 + 全量序列化（可能带 passwordHash）
logger.info("user login ok, password=%s", password)        # 敏感字段入日志
```

## 10. 安全规范

### 10.1 密码与令牌（bcrypt / PyJWT）

```python
# bcrypt（cost=10，兼容种子数据 $2b$ 哈希）；禁止 MD5/SHA 自造方案、禁止明文或可逆加密存储
password_hash = bcrypt.hashpw(password.encode("utf-8"), bcrypt.gensalt(rounds=10))
ok = bcrypt.checkpw(password.encode("utf-8"), password_hash.encode())

# PyJWT 验签：exp 过期校验是 decode 默认行为，禁止 options={"verify_exp": False}
def decode_token(token: str) -> dict:
    try:
        return jwt.decode(token, settings.JWT_SECRET, algorithms=["HS256"])
    except jwt.ExpiredSignatureError:
        raise BizError(ErrorCode.TOKEN_EXPIRED, "令牌已过期，请刷新", http_status=401) from None
    except jwt.InvalidTokenError:
        raise BizError(ErrorCode.UNAUTHENTICATED, "未认证或令牌无效", http_status=401) from None
```

令牌契约：HS256（第 19 章起支持 RS256/Keycloak），access 2h / refresh 7d；refresh 白名单存 Redis，logout 拉黑；`Authorization: Bearer <token>`。

### 10.2 SQL 注入与参数化查询

```python
# ✅ 正例：ORM 查询（SQLAlchemy 自动参数化，天然免疫注入）
wafer = (await db.execute(
    select(Wafer.id, Wafer.batch_no).where(Wafer.batch_no == batch_no))).scalar_one_or_none()

# ✅ 正例：确需原生 SQL 时，text() 只配绑定参数
rows = (await db.execute(
    text('SELECT id, "batchNo" FROM wafers WHERE "batchNo" = :batch_no'),
    {"batch_no": batch_no})).all()

# ❌ 反例：f-string 拼进 text()——唯一能击穿参数化的写法，一票否决合并
await db.execute(text(f'SELECT * FROM wafers WHERE "batchNo" = \'{batch_no}\''))
```

### 10.3 敏感信息脱敏

统一使用集中实现的脱敏工具（全局导入复用），禁止各模块自行实现；响应中的手机号、邮箱、令牌字段必须脱敏后才可返回前端：

```python
def mask_phone(v: str) -> str:
    return re.sub(r"(\d{3})\d{4}(\d{4})", r"\1****\2", v)

def mask_email(v: str) -> str:
    return re.sub(r"^(.).*(@.*)$", r"\1***\2", v)

def mask_token(v: str) -> str:
    return f"{v[:4]}****" if len(v) > 8 else "****"
```

### 10.4 CORS 白名单

```python
# ✅ 正例：白名单来自配置（IA_CORS_ORIGINS），随环境收敛
app.add_middleware(CORSMiddleware, allow_origins=settings.CORS_ORIGINS,
                   allow_credentials=True, allow_methods=["*"], allow_headers=["*"])

# ❌ 反例：全放开 + 带凭证——规范禁止且直接暴露跨域凭证
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_credentials=True)
```

## 11. 配置管理

1. 配置统一收敛到 `app/core/config.py` 的 `Settings`（pydantic-settings），环境变量一律 `IA_` 前缀，多环境用 `IA_ENV=local|prod|test` 切换。
2. **密钥（数据库口令、IA_JWT_SECRET、MinIO secretKey、LLM API Key）一律经环境变量注入**；`.env` 不入库（.gitignore），仓库只提交 `.env.example`。

```python
# app/core/config.py
from typing import Literal
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="IA_")

    ENV: Literal["local", "prod", "test"] = "local"
    DATABASE_URL: str = "postgresql+asyncpg://ia:ia@127.0.0.1:5432/industryagent"
    JWT_SECRET: str                      # 无默认值：缺失即启动失败，逼着环境注入
    AGENT_BASE_URL: str = "http://127.0.0.1:8300"
    CORS_ORIGINS: list[str] = ["http://localhost:5173"]
```

```python
# ❌ 反例：密钥硬编码进仓库（属于安全事故，一票否决合并）
JWT_SECRET = "xinlian-super-secret-2026"
settings = Settings(DATABASE_URL="postgresql+asyncpg://ia:P@ssw0rd2026!@db:5432/industryagent")
```

## 12. 命名规范（Python 仓库以本节为准，PEP 8）

| 对象 | 规则 | 正例 | 反例 |
|---|---|---|---|
| 文件/模块 | snake_case | `wafer_service.py` | `WaferService.py`、`waferService.py` |
| 包/目录 | snake_case 短名 | `routers/`、`tasks/` | `YieldAnalysis/` |
| 类（含 Service/Schema/Exception） | PascalCase，不加 `I` 前缀 | `WaferService`、`BizError` | `IWaferService` |
| 函数/方法/变量 | snake_case，动词开头见名知义 | `get_defect_rate()` | `getData()`、`dr` |
| 路由处理函数 | 动词开头 | `list_wafers`、`create_wafer`、`approve_suggestion` | `wafer_list`、`do_stuff` |
| 常量 | UPPER_SNAKE_CASE，集中定义 | `MAX_RETRY_COUNT = 5` | `maxRetry` |
| 布尔 | is/has/can/should 前缀 | `is_anomaly_detected` | `anomaly_flag` |
| 枚举成员 | UPPER_SNAKE_CASE | `ErrorCode.TOKEN_EXPIRED` | `CODE_1` |
| 对外 JSON 字段 | camelCase 契约字段，Schema 同名直写（无需 alias） | `batchNo`、`yieldRate` | `batch_no` |
| 数据库表/列 | 表名沿用契约表（wafers/users/...）；列名沿用 camelCase 契约列，Python 属性 snake_case 经 `mapped_column("batchNo")` 映射 | `__tablename__ = "wafers"` | 重命名契约列/表 |

与 IA-DEV-001《代码规范总则》的衔接：总则第 3 章命名表面向 TS/JS（文件 kebab-case 等），Python 仓库以本节为准；命名长度控制（2~4 个单词、禁无意义缩写）、魔法值禁止、注释规范（docstring 解释为什么，公开类/函数必写 docstring）等通用条款原样沿用总则。

## 13. 测试要求

- 框架：pytest + pytest-asyncio；单元测试 mock 数据库/外部服务，集成测试用 `httpx.AsyncClient(ASGITransport)` 打真应用、连测试库 `industryagent_test`。
- Service 层必须有单元测试，覆盖率 ≥ 70%（核心业务 ≥ 85%），CI 阻断不达标 PR。

```python
# tests/unit/test_wafer_service.py
@pytest.mark.asyncio
async def test_create_duplicated_batch_raises_10003(db_session):
    payload = WaferCreateRequest(batchNo="B20261015001", plannedQty=25)
    await WaferService(db_session).create(payload)
    with pytest.raises(BizError) as exc_info:                # 再建同批次号 → 10003
        await WaferService(db_session).create(payload)
    assert exc_info.value.code == ErrorCode.DUPLICATED
    assert exc_info.value.http_status == 400
```

```python
# tests/integration/conftest.py —— 依赖覆盖：get_db 指向测试库，绝无打穿生产库的可能（imports 略）
async def override_get_db():
    async with TestingSessionLocal() as session:   # 绑定 industryagent_test
        yield session


app.dependency_overrides[get_db] = override_get_db


@pytest.fixture
async def client():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c


# tests/integration/test_auth_flow.py —— 响应包契约校验
@pytest.mark.asyncio
async def test_login_ok(client):
    resp = await client.post("/api/v1/auth/login",
                             json={"username": "admin", "password": "Admin@123"})
    assert resp.status_code == 200 and resp.json()["code"] == 0
```

## 14. 检查清单速览

- [ ] Router 无业务逻辑、无 SQL，路由函数动词开头
- [ ] 参数校验交给 Pydantic Schema，无手写 if 校验
- [ ] 可预期业务错误抛 `BizError`，错误码已在 `error_codes.py` 登记并同步 IA-DEV-007
- [ ] 事务以 `async with db.begin():` 在 Service 闭合，事务内无慢 IO；竞争路径 `with_for_update`
- [ ] 无裸 except、无吞异常；堆栈只进日志，绝不返回前端
- [ ] 日志全部走 `logging`，含 traceId，无敏感字段，无 f-string 拼接
- [ ] SQL 走 ORM 或 `text()` + 绑定参数，无字符串拼接
- [ ] 密码 bcrypt、JWT 校验 exp、CORS 白名单
- [ ] 请求路径无阻塞调用；`httpx.AsyncClient` 进程级复用
- [ ] 密钥零硬编码，全部 `IA_*` 环境变量，`.env` 不入库
- [ ] 函数 ≤ 50 行、圈复杂度 ≤ 15、单文件 ≤ 500 行
