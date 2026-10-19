# 5. 第一个完整模块——晶圆批次 wafer

## 5.1 做什么 / 为什么

晶圆批次（wafer）是平台最核心的主数据：每一片 12 英寸晶圆按批次投产，批次携带产品型号、工艺节点、产线、良率、缺陷数等字段。根因分析的一切都围绕"某个批次良率为什么掉了"展开。

本章按 **Model（SQLAlchemy 模型）→ 错误体系（错误码 + BizError）→ 全局异常处理 → Service → Schema/Router（Pydantic 校验 + APIRouter）→ init_db.py 建表种子 → curl 测试** 的顺序，把一个生产级 CRUD 模块的所有零件装配起来——这是后续所有模块的模板。

对照原 Egg.js 版：egg-sequelize 自动加载 `app/model/`、egg-validate 在 Controller 里校验、errorHandler 中间件兜底，这三件事在 FastAPI 里分别变成"显式 import 模型并注册到 `Base.metadata`"、"Pydantic 请求模型 + `RequestValidationError` 异常处理器"、"app/main.py 里注册的全局异常处理器"。**对外行为（路由、错误码、响应包）与原版逐字节一致**。

> 本章会用到第 4 章 `Settings` 里的分页参数。若你的字段名尚未定义，在 `app/core/config.py` 的 `Settings` 中补上（环境变量名同 `.env`，无则用默认值）：
>
> ```python
>     # ---------- 分页（对标原 config.biz.pagination） ----------
>     default_page: int = 1
>     default_page_size: int = 10
>     max_page_size: int = 100
> ```

## 5.2 数据模型：app/models/base.py 与 app/models/wafer.py（完整内容）

**做什么/为什么**：SQLAlchemy 2.0 用一个 `DeclarativeBase` 子类承载所有表的元数据，`init_db.py` 建表、`drop_all` 清库都靠它。与 egg-sequelize "文件名决定挂载名"不同，FastAPI 项目里模型靠**显式 import** 生效——我们在 `app/models/__init__.py` 统一注册，建表脚本只要 `import app.models` 就能看到全部表。

先建包骨架（`__init__.py` 先留空，5.2 末尾会填内容）：`app/models/`、`app/schemas/`、`app/services/`、`app/routers/`、`app/middleware/`。

`app/models/base.py`（完整内容）：

```python
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    """所有模型的公共基类。

    为什么单独一个文件：init_db.py 依赖 Base.metadata 建表，
    各模型文件又依赖 Base 做继承——放到独立模块可避免循环 import。
    """
```

`app/models/wafer.py`（完整内容）：

```python
from datetime import datetime, timezone

from sqlalchemy import CheckConstraint, DateTime, Numeric, String, Text, text
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base


def _utcnow() -> datetime:
    # 统一用"带时区的当前时间"：asyncpg 写 timestamptz 列时拒绝 naive datetime
    return datetime.now(timezone.utc)


class Wafer(Base):
    """
    晶圆批次表 wafers
    一个批次 = 一次投片的 25 片晶圆（1 个 lot）

    契约冻结：列名沿用原教程的 camelCase（batchNo/yieldRate/...），
    保证种子数据、前端与智能体教程的 JSON 字段零变化。
    Python 属性名与列名同名，Pydantic schema 无需 alias。
    """
    __tablename__ = "wafers"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True, comment="主键")
    batchNo: Mapped[str] = mapped_column(String(32), nullable=False, unique=True, comment="批次号，如 B20261015001")
    productModel: Mapped[str] = mapped_column(String(64), nullable=False, comment="产品型号，如 MCU-XS32")
    processNode: Mapped[str] = mapped_column(
        String(16), nullable=False, default="28nm", server_default=text("'28nm'"), comment="工艺节点",
    )
    productionLine: Mapped[str] = mapped_column(String(32), nullable=False, comment="产线，如 LINE-1 / LINE-2 / LINE-3")
    quantity: Mapped[int] = mapped_column(nullable=False, default=25, server_default=text("25"), comment="批次晶圆数量")
    defectCount: Mapped[int] = mapped_column(nullable=False, default=0, server_default=text("0"), comment="缺陷晶圆数")
    yieldRate: Mapped[float] = mapped_column(Numeric(5, 2), nullable=False, comment="批次良率（%），如 92.50")
    status: Mapped[str] = mapped_column(
        String(16), nullable=False, default="producing", server_default=text("'producing'"),
        comment="批次状态：在制/完工/出货/隔离",
    )
    remark: Mapped[str | None] = mapped_column(Text, nullable=True, comment="备注")
    producedAt: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True, comment="投产时间")
    # createdAt / updatedAt 自动维护（对标 sequelize timestamps: true）
    createdAt: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=_utcnow, server_default=text("now()"), comment="创建时间",
    )
    updatedAt: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=_utcnow, onupdate=_utcnow,
        server_default=text("now()"), comment="更新时间",
    )

    __table_args__ = (
        # 用 CHECK 约束表达枚举：比 PG 原生 ENUM 类型好删改（ALTER 枚举值痛苦）
        CheckConstraint(
            "status IN ('producing', 'completed', 'shipped', 'quarantined')",
            name="ck_wafers_status",
        ),
    )
```

`app/models/__init__.py`（完整内容，"模型注册表"——每新增一个模型文件都在这里登记）：

```python
from app.models.base import Base
from app.models.wafer import Wafer

__all__ = ["Base", "Wafer"]
```

> **建表机制说明（企业实践）**：`Base.metadata.create_all + scripts/init_db.py` 适合开发与演示；生产环境的表结构演进要用迁移脚本（Alembic）管理，见《02-数据库设计与迁移教程》。本教程主线采用 `create_all`（见 5.8 建表脚本），两套机制了解即可。

## 5.3 错误体系：app/core/error_codes.py 与 app/core/biz_error.py（完整内容）

**做什么/为什么**：业务错误（"批次已存在"）与系统错误（数据库挂了）必须区分开。前者要精确告知前端原因（业务码 + HTTP 400），后者只能笼统提示（HTTP 500）并详细记日志——**绝不能把堆栈泄露给外部**。

`app/core/error_codes.py`（完整内容）：

```python
"""
全局业务错误码约定：
0        成功
10xxx    通用业务错误
20xxx    认证与权限
30xxx    根因分析/智能体
50000    服务器内部错误
"""
from enum import IntEnum


class ErrorCode(IntEnum):
    OK = 0

    PARAM_INVALID = 10001      # 参数校验失败
    NOT_FOUND = 10002          # 资源不存在
    DUPLICATE = 10003          # 重复资源
    STATE_CONFLICT = 10004     # 状态不允许该操作
    DUPLICATE_REQUEST = 10005  # 幂等命中：重复提交

    UNAUTHORIZED = 20001       # 未认证
    TOKEN_EXPIRED = 20002      # 令牌过期
    FORBIDDEN = 20003          # 无权限
    LOGIN_FAILED = 20004       # 用户名或密码错误
    USER_DISABLED = 20005      # 用户被禁用

    AGENT_UNAVAILABLE = 30001  # 智能体服务不可用（熔断/超时）
    AGENT_BAD_RESPONSE = 30002 # 智能体返回异常

    INTERNAL_ERROR = 50000     # 服务器内部错误


# 错误码 -> 默认提示文案（便于统一维护；BizError 未传 message 时回落到这里）
DEFAULT_MESSAGES: dict[ErrorCode, str] = {
    ErrorCode.PARAM_INVALID: "参数校验失败",
    ErrorCode.NOT_FOUND: "资源不存在",
    ErrorCode.DUPLICATE: "资源已存在",
    ErrorCode.STATE_CONFLICT: "当前状态不允许该操作",
    ErrorCode.DUPLICATE_REQUEST: "重复请求，请勿重复提交",
    ErrorCode.UNAUTHORIZED: "请先登录",
    ErrorCode.TOKEN_EXPIRED: "登录已过期，请重新登录",
    ErrorCode.FORBIDDEN: "没有执行该操作的权限",
    ErrorCode.LOGIN_FAILED: "用户名或密码错误",
    ErrorCode.USER_DISABLED: "账号已被禁用，请联系管理员",
    ErrorCode.AGENT_UNAVAILABLE: "智能体服务暂不可用，请稍后重试",
    ErrorCode.AGENT_BAD_RESPONSE: "智能体服务返回异常",
    ErrorCode.INTERNAL_ERROR: "服务器内部错误",
}
```

`app/core/biz_error.py`（完整内容）：

```python
from app.core.error_codes import DEFAULT_MESSAGES, ErrorCode


class BizError(Exception):
    """
    业务错误：所有可预期的、需要告知用户原因的错误。
    app/main.py 里注册的全局异常处理器会把它转成
    { code, message, traceId } 响应（HTTP 状态取 http_status）。
    """

    def __init__(self, code: int, message: str | None = None, http_status: int = 400) -> None:
        # 只传错误码也能拿到统一文案：BizError(ErrorCode.NOT_FOUND)
        if message is None and isinstance(code, ErrorCode):
            message = DEFAULT_MESSAGES[code]
        super().__init__(message)
        self.name = "BizError"
        self.code = int(code)          # 业务错误码（见 ErrorCode）
        self.message = str(message)    # 给用户看的提示
        self.http_status = http_status  # HTTP 状态码
```

## 5.4 统一响应与全局异常处理：app/core/response.py 与 app/main.py（完整内容）

**做什么/为什么**：Egg 版有 `ctx.success() / ctx.page()` 和 error_handler 洋葱中间件；FastAPI 没有框架级"响应扩展"，我们建一个响应包助手模块收敛格式，再用**全局异常处理器**（对标"最外层洋葱"）统一错误出口。traceId 本章先以"未接中间件时的兜底 uuid"出现，第 6 章中间件接入后自动变成"每请求唯一、贯穿日志"。

`app/core/response.py`（完整内容）：

```python
"""
统一响应包：
成功：{ code: 0, message: 'ok', data: ..., traceId: '...' }
分页：{ code: 0, message: 'ok', data: { rows, total, page, pageSize }, traceId }
失败：{ code: 业务错误码, message: '...', traceId }（由全局异常处理器产出）

traceId 存在 contextvars 里：中间件设置一次，全请求（含后台任务）都能读到，
这与 Egg 的 ctx.tracer.traceId 语义一致。
"""
import uuid
from contextvars import ContextVar

# 每个请求一份的 traceId（第 6 章的中间件负责 set）
_trace_id_var: ContextVar[str] = ContextVar("trace_id", default="")


def set_trace_id(trace_id: str) -> None:
    _trace_id_var.set(trace_id)


def get_trace_id() -> str:
    # 兜底：中间件尚未接入（第 5 章阶段）时也保证响应包里有合法 traceId
    return _trace_id_var.get() or str(uuid.uuid4())


def ok(data=None, message: str = "ok") -> dict:
    """成功响应包（对标 ctx.success）"""
    return {"code": 0, "message": message, "data": data, "traceId": get_trace_id()}


def page(rows: list, total: int, page: int, page_size: int, message: str = "ok") -> dict:
    """分页响应包（对标 ctx.page）"""
    return ok({"rows": rows, "total": total, "page": page, "pageSize": page_size}, message)


def error_body(code: int, message: str, detail: list | None = None) -> dict:
    """错误响应包（对标 error_handler 产出的 body；结构需逐字节兼容前端契约）"""
    body = {"code": int(code), "message": message, "traceId": get_trace_id()}
    if detail is not None:
        body["detail"] = detail
    return body
```

接着改造 `app/main.py`。Egg 版的 `error_handler` 中间件在 FastAPI 里对应四个异常处理器（`BizError` / 参数校验 / HTTP 404 / 兜底 `Exception`），全部注册在 `create_app()` 里。

`app/main.py`（完整内容，本章版本）：

```python
"""应用入口：FastAPI 实例、全局异常处理、路由挂载（中间件从第 6 章加入）。"""
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

from app.core.biz_error import BizError
from app.core.config import get_settings
from app.core.database import engine
from app.core.error_codes import ErrorCode
from app.core.response import error_body
from app.routers import home, wafer

logger = logging.getLogger("industryagent")


@asynccontextmanager
async def lifespan(app: FastAPI):
    # lifespan 对标 Egg 的 didReady/beforeClose：统一管理启停资源
    settings = get_settings()
    logger.info("[startup] industryagent-server 就绪（env=%s）", settings.env)
    yield
    await engine.dispose()  # 优雅关闭连接池，避免 asyncpg 报 "event loop is closed"
    logger.info("[shutdown] 数据库连接池已释放")


# ---------------- 参数校验错误的文案翻译 ----------------

def _fmt_number(value) -> str:
    """100.0 -> '100'：让提示与原教程 egg-validate 的文案风格一致。"""
    if isinstance(value, float) and value.is_integer():
        return str(int(value))
    return str(value)


def _translate_validation_errors(exc: RequestValidationError) -> list[dict]:
    """
    把 pydantic v2 的错误翻成 { field, message }。
    为什么不用 pydantic 原生文案：前端/教程契约是 egg 时代的
    "yieldRate must be less than or equal to 100"，逐字节兼容优先。
    """
    translated = []
    for err in exc.errors():
        loc = [str(x) for x in err.get("loc", []) if x not in ("body", "query", "path")]
        field = ".".join(loc) or "(root)"
        err_type = err.get("type", "")
        ctx = err.get("ctx") or {}
        if err_type == "missing":
            msg = f"{field} is required"
        elif err_type == "string_type":
            msg = f"{field} must be a string"
        elif err_type in ("int_type", "int_parsing"):
            msg = f"{field} must be an integer"
        elif err_type in ("float_type", "float_parsing"):
            msg = f"{field} must be a number"
        elif err_type == "greater_than_equal":
            msg = f"{field} must be greater than or equal to {_fmt_number(ctx.get('ge'))}"
        elif err_type == "less_than_equal":
            msg = f"{field} must be less than or equal to {_fmt_number(ctx.get('le'))}"
        elif err_type == "string_too_short":
            msg = f"{field} must be longer than {ctx.get('min_length')} characters"
        elif err_type == "string_too_long":
            msg = f"{field} must be shorter than or equal to {ctx.get('max_length')} characters"
        else:
            msg = f"{field} {err.get('msg', '参数不合法')}"
        translated.append({"field": field, "message": msg})
    return translated


# ---------------- 全局异常处理器（对标 Egg 的 error_handler 洋葱） ----------------

async def biz_error_handler(request: Request, exc: BizError) -> JSONResponse:
    # 业务错误：精确告知前端原因（HTTP 状态按错误映射：10002->404、10003->400、20xxx->401/403...）
    request.state.error_summary = exc.message  # 留给第 6 章审计中间件读"失败原因"
    return JSONResponse(status_code=exc.http_status, content=error_body(exc.code, exc.message))


async def validation_error_handler(request: Request, exc: RequestValidationError) -> JSONResponse:
    # 参数校验失败：egg-validate（parameter）风格的 422 + detail 数组
    return JSONResponse(
        status_code=422,
        content=error_body(ErrorCode.PARAM_INVALID, "参数校验失败", detail=_translate_validation_errors(exc)),
    )


async def http_exception_handler(request: Request, exc: StarletteHTTPException) -> JSONResponse:
    # 路由未命中（404）也统一成 JSON（对标原 errorHandler 的 404 分支）
    if exc.status_code == 404:
        return JSONResponse(status_code=404, content=error_body(40400, "接口不存在"))
    return JSONResponse(status_code=exc.status_code, content=error_body(exc.status_code * 100, str(exc.detail)))


async def unhandled_exception_handler(request: Request, exc: Exception) -> JSONResponse:
    # 未知错误：详细记录（服务端可见），对外只报笼统信息——绝不泄露堆栈
    logger.exception("[unhandled] %s %s ->", request.method, request.url.path)
    return JSONResponse(status_code=500, content=error_body(50000, "服务器内部错误"))


def create_app() -> FastAPI:
    settings = get_settings()
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s %(levelname)s %(name)s - %(message)s",
    )
    app = FastAPI(title="industryagent-server", version="1.0.0", lifespan=lifespan)

    # ---------------- 路由 ----------------
    app.include_router(home.router)
    app.include_router(wafer.router)

    # ---------------- 全局异常处理 ----------------
    app.add_exception_handler(BizError, biz_error_handler)
    app.add_exception_handler(RequestValidationError, validation_error_handler)
    app.add_exception_handler(StarletteHTTPException, http_exception_handler)
    app.add_exception_handler(Exception, unhandled_exception_handler)

    return app


app = create_app()
```

顺带把第 3 章的欢迎页换成统一助手（`app/routers/home.py` 完整内容，字段与原版一致）：

```python
from datetime import datetime, timezone

from fastapi import APIRouter

from app.core import response

router = APIRouter()


@router.get("/")
async def index():
    # 平台信息（欢迎页）
    return response.ok({
        "name": "industryagent-server",
        "version": "1.0.0",
        "project": "芯联半导体良率根因分析平台 - 业务后端",
        "doc": "IA-TU-001 后端开发教程",
        "time": datetime.now(timezone.utc).isoformat(),
    })
```

## 5.5 Service 层：app/services/wafer.py（完整内容）

**做什么/为什么**：业务规则（批次号唯一、分页上限、事务）集中在 service。**创建操作演示 SQLAlchemy 事务**：先查重再插入的两步必须在同一事务里，否则并发下有竞态。与 Egg 版"查完再 create"不同，async SQLAlchemy 的节奏是"查重 → add → commit"，失败统一 rollback。

`app/services/wafer.py`（完整内容）：

```python
from datetime import datetime, timezone

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.biz_error import BizError
from app.core.error_codes import ErrorCode
from app.models.wafer import Wafer
from app.schemas.wafer import WaferCreate, WaferUpdate


class WaferService:
    """晶圆批次业务逻辑：路由层只管翻译 HTTP，规则都压在这里。"""

    @staticmethod
    async def list(
        db: AsyncSession,
        *,
        page: int,
        page_size: int,
        batch_no: str | None = None,
        status: str | None = None,
        production_line: str | None = None,
    ) -> tuple[list[Wafer], int]:
        """分页查询批次列表（支持批次号模糊/状态/产线过滤）"""
        conditions = []
        if batch_no:
            # ilike 不区分大小写，比原版的 LIKE 对人工输入更友好
            conditions.append(Wafer.batchNo.ilike(f"%{batch_no}%"))
        if status:
            conditions.append(Wafer.status == status)
        if production_line:
            conditions.append(Wafer.productionLine == production_line)

        rows = (
            await db.execute(
                select(Wafer)
                .where(*conditions)
                .order_by(Wafer.id.desc())
                .offset((page - 1) * page_size)
                .limit(page_size)
            )
        ).scalars().all()

        total = (
            await db.execute(select(func.count()).select_from(Wafer).where(*conditions))
        ).scalar_one()

        return list(rows), total

    @staticmethod
    async def detail(db: AsyncSession, wafer_id: int) -> Wafer:
        """批次详情（不存在时抛业务错误，HTTP 404 / 业务码 10002）"""
        wafer = await db.get(Wafer, wafer_id)
        if wafer is None:
            raise BizError(ErrorCode.NOT_FOUND, f"批次不存在（id={wafer_id}）", 404)
        return wafer

    @staticmethod
    async def create(db: AsyncSession, data: WaferCreate) -> Wafer:
        """创建批次（事务示例：查重 + 插入在同一事务内，避免并发竞态）"""
        try:
            exists = (
                await db.execute(select(Wafer).where(Wafer.batchNo == data.batchNo))
            ).scalar_one_or_none()
            if exists:
                raise BizError(ErrorCode.DUPLICATE, f"批次号 {data.batchNo} 已存在")

            values = data.model_dump()
            # 投产时间缺省为当前时间（对标原版 new Date()）
            values["producedAt"] = values.get("producedAt") or datetime.now(timezone.utc)

            wafer = Wafer(**values)
            db.add(wafer)
            await db.flush()   # 拿到自增 id，仍在同一事务内
            await db.commit()  # 两步都成功才落库
            await db.refresh(wafer)
            return wafer
        except Exception:
            await db.rollback()  # 查重与插入同生共死
            raise

    @staticmethod
    async def update(db: AsyncSession, wafer_id: int, data: WaferUpdate) -> Wafer:
        """更新批次（只更新请求里出现的字段）"""
        wafer = await WaferService.detail(db, wafer_id)
        for field, value in data.model_dump(exclude_unset=True).items():
            setattr(wafer, field, value)
        await db.commit()
        await db.refresh(wafer)
        return wafer

    @staticmethod
    async def remove(db: AsyncSession, wafer_id: int) -> dict:
        """删除批次（软删除思路可扩展 deletedAt 字段，这里演示物理删除）"""
        wafer = await WaferService.detail(db, wafer_id)
        await db.delete(wafer)
        await db.commit()
        return {"id": wafer_id}
```

## 5.6 请求/响应模型：app/schemas/wafer.py（完整内容）

**做什么/为什么**：原教程里 Controller 的 egg-validate 规则（`createRule`/`updateRule`）在 FastAPI 里就是 Pydantic 模型——声明即校验，校验失败的响应由 5.4 的 `RequestValidationError` 处理器统一产出（422 / 10001 / `{field, message}` 数组）。**字段名直接用 camelCase**（契约冻结，无需 alias）。

`app/schemas/wafer.py`（完整内容）：

```python
from datetime import datetime
from typing import Literal

from pydantic import BaseModel, ConfigDict, Field

# 批次状态枚举（与 wafers 表的 CheckConstraint 一一对应）
WaferStatus = Literal["producing", "completed", "shipped", "quarantined"]


class WaferCreate(BaseModel):
    """创建批次（对标原 createRule）"""
    batchNo: str = Field(min_length=6, max_length=32)
    productModel: str = Field(min_length=1, max_length=64)
    processNode: str = Field(default="28nm", min_length=1, max_length=16)
    productionLine: str = Field(min_length=1, max_length=32)
    quantity: int = Field(ge=1, le=1000)
    yieldRate: float = Field(ge=0, le=100)
    defectCount: int = Field(default=0, ge=0)
    status: WaferStatus = "producing"
    remark: str | None = Field(default=None, max_length=2000)
    producedAt: datetime | None = None


class WaferUpdate(BaseModel):
    """更新批次（全部可选，对标原 updateRule）"""
    productModel: str | None = Field(default=None, min_length=1, max_length=64)
    processNode: str | None = Field(default=None, min_length=1, max_length=16)
    productionLine: str | None = Field(default=None, min_length=1, max_length=32)
    quantity: int | None = Field(default=None, ge=1, le=1000)
    yieldRate: float | None = Field(default=None, ge=0, le=100)
    defectCount: int | None = Field(default=None, ge=0)
    status: WaferStatus | None = None
    remark: str | None = Field(default=None, max_length=2000)


class WaferOut(BaseModel):
    """批次响应模型：ORM 对象 -> JSON（yieldRate 声明为 float，避免 Decimal 无法序列化）"""
    model_config = ConfigDict(from_attributes=True)

    id: int
    batchNo: str
    productModel: str
    processNode: str
    productionLine: str
    quantity: int
    defectCount: int
    yieldRate: float
    status: str
    remark: str | None
    producedAt: datetime | None
    createdAt: datetime
    updatedAt: datetime
```

## 5.7 路由层：app/routers/wafer.py（完整内容）

**做什么/为什么**：Egg 的 Controller + router.ts 两层，在 FastAPI 里合并成"APIRouter 里的路由函数"——函数只干三件事：取参（Query/路径/Pydantic 模型）、调 service、统一响应。数据库会话通过 `Depends(get_db)` 注入（第 4 章 `app/core/database.py` 提供的 async generator 依赖）。

`app/routers/wafer.py`（完整内容，鉴权依赖第 7 章再加）：

```python
from typing import Literal

from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.core import response
from app.core.config import get_settings
from app.core.database import get_db
from app.schemas.wafer import WaferCreate, WaferOut, WaferUpdate
from app.services.wafer import WaferService

# URL 统一带 /api/v1 版本前缀——上线后若有破坏性改动，可以平行开 /api/v2
router = APIRouter(prefix="/api/v1/wafers", tags=["wafers"])

WaferStatusQuery = Literal["producing", "completed", "shipped", "quarantined"]


def _dump(wafer) -> dict:
    # ORM -> dict：统一走 WaferOut，字段与 Decimal/DateTime 的序列化只写一次
    return WaferOut.model_validate(wafer).model_dump(mode="json")


@router.get("")
async def list_wafers(
    page: int | None = Query(default=None, ge=1, description="页码"),
    pageSize: int | None = Query(default=None, ge=1, le=500, description="每页条数"),
    batchNo: str | None = Query(default=None, description="批次号模糊匹配"),
    status: WaferStatusQuery | None = Query(default=None, description="批次状态"),
    productionLine: str | None = Query(default=None, description="产线"),
    db: AsyncSession = Depends(get_db),
):
    """GET /api/v1/wafers?page=1&pageSize=10&batchNo=&status="""
    settings = get_settings()
    # 复刻原版行为：缺省用配置默认值；pageSize 超业务上限时静默截断到 maxPageSize
    page_num = page or settings.default_page
    page_size = min(pageSize or settings.default_page_size, settings.max_page_size)

    rows, total = await WaferService.list(
        db,
        page=page_num,
        page_size=page_size,
        batch_no=batchNo or None,
        status=status,
        production_line=productionLine or None,
    )
    return response.page([_dump(w) for w in rows], total, page_num, page_size)


@router.get("/{wafer_id}")
async def get_wafer(wafer_id: int, db: AsyncSession = Depends(get_db)):
    """GET /api/v1/wafers/:id"""
    return response.ok(_dump(await WaferService.detail(db, wafer_id)))


@router.post("", status_code=201)
async def create_wafer(payload: WaferCreate, db: AsyncSession = Depends(get_db)):
    """POST /api/v1/wafers"""
    wafer = await WaferService.create(db, payload)
    return response.ok(_dump(wafer), "创建成功")


@router.put("/{wafer_id}")
async def update_wafer(wafer_id: int, payload: WaferUpdate, db: AsyncSession = Depends(get_db)):
    """PUT /api/v1/wafers/:id"""
    wafer = await WaferService.update(db, wafer_id, payload)
    return response.ok(_dump(wafer), "更新成功")


@router.delete("/{wafer_id}")
async def delete_wafer(wafer_id: int, db: AsyncSession = Depends(get_db)):
    """DELETE /api/v1/wafers/:id"""
    result = await WaferService.remove(db, wafer_id)
    return response.ok(result, "删除成功")
```

`app/routers/__init__.py` 留空即可（`app/main.py` 已 `from app.routers import home, wafer`）。

> URL 统一带 `/api/v1` 版本前缀——上线后若有破坏性改动，可以平行开 `/api/v2`，这是企业 API 的基本素养。

## 5.8 建表与种子数据：scripts/init_db.py（完整内容）

**做什么/为什么**：`Base.metadata.create_all` 按模型自动建表，`drop_all` 先清场——对标 sequelize 的 `sync({ force: true })`。种子脚本同时灌入演示数据，让后续每一章的 curl 都有数据可查。**本脚本会清空并重建所有表，仅用于开发环境。**

`scripts/init_db.py`（完整内容，第一版）：

```python
"""建表 + 种子数据（开发环境专用：drop_all + create_all 会清空重建所有表）。"""
import asyncio
import sys
from datetime import datetime, timedelta, timezone
from pathlib import Path

# 让 `python scripts/init_db.py` 能找到项目根的 app 包
# （直接运行脚本时 sys.path[0] 是 scripts/ 而不是项目根，Windows 下同样适用）
sys.path.insert(0, str(Path(__file__).resolve().parents[1]))

import app.models  # noqa: F401,E402  # 导入即注册：所有模型登记到 Base.metadata
from app.core.database import AsyncSessionLocal, engine  # noqa: E402
from app.models.base import Base  # noqa: E402
from app.models.wafer import Wafer  # noqa: E402


async def main() -> None:
    print("[seed] 同步表结构（drop+create 会清空重建）...")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
        await conn.run_sync(Base.metadata.create_all)
    print("[seed] 表结构同步完成")

    # ---------- 种子数据：晶圆批次 ----------
    now = datetime.now(timezone.utc)

    def hours_ago(hours: int) -> datetime:
        return now - timedelta(hours=hours)

    async with AsyncSessionLocal() as db:
        db.add_all([
            Wafer(batchNo="B20261015001", productModel="MCU-XS32", processNode="28nm", productionLine="LINE-1",
                  quantity=25, defectCount=4, yieldRate=91.20, status="completed", producedAt=hours_ago(30),
                  remark="良率低于目标 95%，已触发根因分析"),
            Wafer(batchNo="B20261015002", productModel="PMIC-65W", processNode="40nm", productionLine="LINE-2",
                  quantity=25, defectCount=1, yieldRate=96.80, status="completed", producedAt=hours_ago(26)),
            Wafer(batchNo="B20261015003", productModel="MCU-XS32", processNode="28nm", productionLine="LINE-1",
                  quantity=25, defectCount=6, yieldRate=89.60, status="completed", producedAt=hours_ago(8),
                  remark="连续两批良率下滑"),
            Wafer(batchNo="B20261015004", productModel="RF-SOC", processNode="22nm", productionLine="LINE-3",
                  quantity=25, defectCount=0, yieldRate=97.50, status="shipped", producedAt=hours_ago(72)),
            Wafer(batchNo="B20261015005", productModel="PMIC-65W", processNode="40nm", productionLine="LINE-2",
                  quantity=25, defectCount=0, yieldRate=0, status="producing", producedAt=hours_ago(2),
                  remark="在制批次"),
        ])
        await db.commit()
    print("[seed] 晶圆批次 5 条写入完成")

    # 用户/角色等种子数据在第 7 章追加

    await engine.dispose()
    print("[seed] 全部完成 ✔")


if __name__ == "__main__":
    asyncio.run(main())
```

执行（在项目根目录）：

```bash
python scripts/init_db.py
```

预期输出：

```text
[seed] 同步表结构（drop+create 会清空重建）...
[seed] 表结构同步完成
[seed] 晶圆批次 5 条写入完成
[seed] 全部完成 ✔
```

## 5.9 curl 全接口测试演示

确保服务在跑（第 3 章的启动命令：`python -m uvicorn app.main:app --host 127.0.0.1 --port 7001 --reload`），然后依次执行（Git Bash）：

**① 分页查询列表**

```bash
curl -s 'http://127.0.0.1:7001/api/v1/wafers?page=1&pageSize=2'
```

预期输出（已格式化）：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "rows": [
      { "id": 5, "batchNo": "B20261015005", "productModel": "PMIC-65W", "...": "..." },
      { "id": 4, "batchNo": "B20261015004", "...": "..." }
    ],
    "total": 5,
    "page": 1,
    "pageSize": 2
  },
  "traceId": "..."
}
```

**② 条件查询（批次号模糊 + 状态过滤）**

```bash
curl -s 'http://127.0.0.1:7001/api/v1/wafers?batchNo=15001&status=completed'
# 预期：data.total 为 1，rows[0].batchNo = "B20261015001"
```

**③ 详情**

```bash
curl -s http://127.0.0.1:7001/api/v1/wafers/1
# 预期：code 0，data.batchNo = "B20261015001"
```

**④ 创建（成功）**

```bash
curl -s -X POST http://127.0.0.1:7001/api/v1/wafers \
  -H 'Content-Type: application/json' \
  -d '{
    "batchNo": "B20261015006",
    "productModel": "MCU-XS32",
    "processNode": "28nm",
    "productionLine": "LINE-1",
    "quantity": 25,
    "yieldRate": 94.10,
    "defectCount": 2,
    "producedAt": "2026-10-15T08:00:00+08:00"
  }'
```

预期输出：HTTP 201，`code:0`，`data.id` 为 6。

**⑤ 创建（参数校验失败——良率超出范围）**

```bash
curl -s -X POST http://127.0.0.1:7001/api/v1/wafers \
  -H 'Content-Type: application/json' \
  -d '{"batchNo":"B20261015007","productModel":"X","processNode":"28nm","productionLine":"LINE-1","quantity":25,"yieldRate":120}'
```

预期输出（HTTP 422）：

```json
{
  "code": 10001,
  "message": "参数校验失败",
  "detail": [ { "field": "yieldRate", "message": "yieldRate must be less than or equal to 100" } ],
  "traceId": "..."
}
```

**⑥ 创建（业务错误——批次号重复）**

```bash
curl -s -X POST http://127.0.0.1:7001/api/v1/wafers \
  -H 'Content-Type: application/json' \
  -d '{"batchNo":"B20261015006","productModel":"X","processNode":"28nm","productionLine":"LINE-1","quantity":25,"yieldRate":90}'
```

预期输出（HTTP 400）：

```json
{ "code": 10003, "message": "批次号 B20261015006 已存在", "traceId": "..." }
```

**⑦ 更新 / 删除**

```bash
curl -s -X PUT http://127.0.0.1:7001/api/v1/wafers/6 \
  -H 'Content-Type: application/json' -d '{"yieldRate": 95.30}'
# 预期：code 0，message "更新成功"

curl -s -X DELETE http://127.0.0.1:7001/api/v1/wafers/6
# 预期：code 0，message "删除成功"
```

> **Windows 提示**：cmd.exe 中多行 JSON 的引号几乎必然出错，请使用 Git Bash；或用 PowerShell 的 `Invoke-RestMethod`，或直接使用 Postman/Apifox。

## 5.10 常见坑

1. **async session 生命周期**：`get_db` 是请求级依赖，响应返回时会话即关闭。Service 里必须显式 `commit`（async session 不会自动提交），异常时 `rollback`。第 6 章的审计后台任务**绝不能复用请求会话**——那时它已关闭，必须自开短会话。
2. **忘记 commit**：查能查到、写却"不生效"（重启后消失），九成是少了 `await db.commit()`。
3. **camelCase 列名在 psql 里要加双引号**：`SELECT "batchNo", "yieldRate" FROM wafers;`——不加引号 PG 会把标识符折成小写而报 `column "batchno" does not exist`。Git Bash 中外层用单引号包整条 SQL 即可避免转义问题。
4. **naive datetime 事故**：asyncpg 对 `timestamptz` 列拒绝 naive `datetime`（`can't subtract offset-naive and offset-aware datetimes`）。全项目统一 `datetime.now(timezone.utc)`。
5. **Decimal 序列化**：`Numeric` 列读出来是 `Decimal`，`jsonable_encoder` 默认不认——响应侧统一声明 `float`（见 `WaferOut.yieldRate`）。
6. **直接运行脚本 import 不到 app**：`python scripts/init_db.py` 的 `sys.path[0]` 是 `scripts/`，所以 init_db.py 开头必须 `sys.path.insert(0, 项目根)`。

---

# 6. 中间件体系：审计日志与请求链路

## 6.1 做什么 / 为什么

第 5 章已经有了一层"中间件"（全局异常处理器）。本章补齐企业级后端的两大横切关注点：

1. **审计日志**：手册明确要求"审批人、时间、理由"可追溯，数据不出域。任何"写操作"都要落 `audit_logs` 表：**谁、何时、对什么路径、做了什么、结果如何、耗时多少**。
2. **请求日志 + traceId**：每个请求分配一个 traceId 贯穿日志，出问题时能按 traceId 捞出完整链路——这是排障的生命线。

FastAPI/Starlette 的中间件是 **ASGI 可调用**（`async def __call__(self, scope, receive, send)`），洋葱语义与 Egg 中间件一致；但注册用 `app.add_middleware(...)`，且**后添加的在最外层**。本章三个中间件都写成纯 ASGI 类——原因见 6.6 常见坑第 1 条（`BaseHTTPMiddleware` 读请求体的回注问题）。

先建审计表模型 `app/models/audit_log.py`（完整内容。注意：与 wafers 的 camelCase 列不同，这张表沿用原教程全局 `underscored` 约定产出的小写列名，`duration_ms`/`trace_id` 等，6.5 的 psql 验证依赖这些列名）：

```python
from datetime import datetime, timezone

from sqlalchemy import DateTime, Integer, String, Text, text
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base


class AuditLog(Base):
    """
    审计日志表 audit_logs：只插入、不修改、不删除（合规要求）。
    属性名保持原文的 camelCase，列名显式映射为小写下划线（与原库一致）。
    """
    __tablename__ = "audit_logs"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    userId: Mapped[int | None] = mapped_column("user_id", Integer, nullable=True, comment="操作人 id（未登录为 null）")
    username: Mapped[str | None] = mapped_column(String(64), nullable=True, comment="操作人用户名")
    method: Mapped[str] = mapped_column(String(8), nullable=False, comment="HTTP 方法")
    path: Mapped[str] = mapped_column(String(255), nullable=False, comment="请求路径")
    status: Mapped[int] = mapped_column(Integer, nullable=False, comment="响应状态码")
    durationMs: Mapped[int] = mapped_column("duration_ms", Integer, nullable=False, comment="耗时（毫秒）")
    ip: Mapped[str | None] = mapped_column(String(64), nullable=True)
    requestSummary: Mapped[str | None] = mapped_column("request_summary", Text, nullable=True, comment="请求体摘要（脱敏后）")
    errorMessage: Mapped[str | None] = mapped_column("error_message", Text, nullable=True, comment="失败原因")
    traceId: Mapped[str | None] = mapped_column("trace_id", String(64), nullable=True)
    # 只写 created_at、不维护 updated_at（timestamps: false + createdAt 映射的等价物）
    createdAt: Mapped[datetime] = mapped_column(
        "created_at", DateTime(timezone=True), nullable=False,
        default=lambda: datetime.now(timezone.utc), server_default=text("now()"),
    )
```

`app/models/__init__.py` 更新为：

```python
from app.models.audit_log import AuditLog
from app.models.base import Base
from app.models.wafer import Wafer

__all__ = ["AuditLog", "Base", "Wafer"]
```

`app/middleware/__init__.py` 留空。

## 6.2 审计 Service：app/services/audit.py（完整内容）

**做什么/为什么**：审计是"后台任务"——响应不等人，写库失败也不能影响业务。Egg 版用 `ctx.service.audit.record().catch(...)` fire-and-forget；FastAPI 对应 `asyncio.create_task(...)`。关键差异：**请求的 AsyncSession 随响应关闭，审计必须自己开一个短会话**（这是 async 生命周期与 Egg 连接池最大的不同，记住这条能少踩一半异步坑）。

`app/services/audit.py`（完整内容）：

```python
import json
import logging

from app.core.database import AsyncSessionLocal
from app.models.audit_log import AuditLog

# 独立命名的审计 logger：第 14 章会把它路由到单独文件（对标原 customLogger.auditLogger）
audit_logger = logging.getLogger("industryagent.audit")


class AuditService:
    """审计落库：调用方不关心成败，失败只打警告日志。"""

    @staticmethod
    async def write(record: dict) -> None:
        """
        落一条审计记录 + 输出一行审计文件日志（双保险：表被误清时文件还在）。

        为什么自开 AsyncSessionLocal：本方法运行在后台任务里，
        请求会话早已随响应关闭——复用必然报 "session is closed"。
        """
        try:
            async with AsyncSessionLocal() as db:
                db.add(AuditLog(**record))
                await db.commit()
            audit_logger.info(json.dumps(record, ensure_ascii=False, default=str))
        except Exception as exc:  # noqa: BLE001 审计失败绝不影响业务
            audit_logger.warning("[audit] 审计写入失败: %s", exc)
```

> `default=str` 让 datetime 等类型在 JSON 序列化时自动转字符串，避免审计日志本身把任务炸掉。

## 6.3 审计中间件：app/middleware/audit_log.py（完整内容）

**做什么/为什么**：记录"谁/何时/对什么/做了什么/耗时/结果"。FastAPI 的鉴权在第 7 章是**依赖注入**而不是中间件，所以审计中间件从 `scope["state"]` 里读当前用户（`get_current_user` 依赖会写入 `request.state.user`，同一个 state 字典贯穿整条 ASGI 链）。请求体摘要用"包一层 receive"的方式旁路留存——消息原样下传，内层照常读体，互不干扰。

`app/middleware/audit_log.py`（完整内容）：

```python
import asyncio
import json
import time

from starlette.types import ASGIApp, Message, Receive, Scope, Send

from app.core.response import get_trace_id
from app.services.audit import AuditService

# 需要审计的方法：一切"写"操作。
# GET /（欢迎页）、/healthz、/metrics 天然不在其中——审计只关心写。
AUDITED_METHODS = {"POST", "PUT", "DELETE", "PATCH"}
# 请求体摘要最大长度
MAX_SUMMARY = 512

# 裸 create_task 的任务可能被垃圾回收（asyncio 只持弱引用），必须留强引用
_background_tasks: set[asyncio.Task] = set()


def summarize(raw: bytes) -> str | None:
    """摘要：JSON 序列化 + 脱敏（密码类字段直接抹掉）"""
    if not raw:
        return None
    try:
        body = json.loads(raw)
    except (json.JSONDecodeError, UnicodeDecodeError):
        # 非 JSON 体（文件上传等）不记内容，只记长度
        return f"<binary {len(raw)} bytes>"
    if not isinstance(body, dict):
        return None
    clone = dict(body)
    for key in list(clone):
        # 密码类字段直接抹掉（对标原 /password|secret|token/i）
        if any(s in key.lower() for s in ("password", "secret", "token")):
            clone[key] = "***"
    s = json.dumps(clone, ensure_ascii=False)
    return s[:MAX_SUMMARY] + "..." if len(s) > MAX_SUMMARY else s


class AuditLogMiddleware:
    """
    审计中间件：记录"谁/何时/对什么/做了什么/耗时/结果"。
    - BizError 等业务失败：全局异常处理器已把响应转成 JSON（含状态码），
      并在 request.state.error_summary 留了失败原因，这里照单全收；
    - 未处理异常：接住记一笔（状态记 500）再原样上抛，交给最外层兜底；
    - fire-and-forget：审计写入失败不阻塞业务响应。
    """

    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] != "http" or scope.get("method") not in AUDITED_METHODS:
            await self.app(scope, receive, send)
            return

        start = time.monotonic()
        body_chunks: list[bytes] = []
        holder = {"status": 500}  # 出错时外层尚未设置状态码，用 500 近似
        caught_error: list[str] = []

        async def receive_tee(message: Message) -> Message:
            # 旁路留存请求体：消息原样下传，内层（路由）照常能读
            if message["type"] == "http.request":
                body_chunks.append(message.get("body", b""))
            return message

        async def send_wrap(message: Message) -> None:
            if message["type"] == "http.response.start":
                holder["status"] = message["status"]
            await send(message)

        try:
            await self.app(scope, receive_tee, send_wrap)
        except Exception as exc:
            caught_error.append(str(exc)[:512])  # 先接住，记录完再抛给外层兜底
            raise
        finally:
            duration_ms = int((time.monotonic() - start) * 1000)
            state = scope.get("state") or {}
            user = state.get("user")  # 第 7 章 get_current_user 写入；未登录为 None
            record = {
                "userId": user.userId if user else None,
                "username": user.username if user else "anonymous",
                "method": scope["method"],
                "path": scope["path"],
                "status": holder["status"],
                "durationMs": duration_ms,
                "ip": (scope.get("client") or ["?"])[0],
                "requestSummary": summarize(b"".join(body_chunks)),
                # 有异常用异常信息；否则 4xx/5xx 用业务处理器留下的摘要
                "errorMessage": (
                    caught_error[0] if caught_error
                    else (state.get("error_summary") if holder["status"] >= 400 else None)
                ),
                "traceId": get_trace_id(),
            }
            # fire-and-forget：不 await，失败只记警告（见 AuditService.write）
            task = asyncio.create_task(AuditService.write(record))
            _background_tasks.add(task)
            task.add_done_callback(_background_tasks.discard)
```

## 6.4 请求日志与 traceId：app/middleware/request_log.py（完整内容）

**做什么/为什么**：Egg 自带 `ctx.tracer.traceId`，FastAPI 没有——我们自己造一个同样语义的最小实现：**每请求生成 uuid → 写入 contextvars（第 5 章 `app/core/response.py` 已经留好接口）→ 回写响应头 `X-Trace-Id`**。前端报障时只需提供这个 ID。同时把每个请求输出为**结构化 JSON 单行日志**（第 14 章会讲为什么必须是单行 JSON：方便 ELK/Loki 采集解析）。

`app/middleware/request_log.py`（完整内容）：

```python
import json
import logging
import time
import uuid
from datetime import datetime, timezone

from starlette.datastructures import MutableHeaders
from starlette.types import ASGIApp, Message, Receive, Scope, Send

from app.core.response import get_trace_id, set_trace_id

logger = logging.getLogger("industryagent.request")


class TraceIdMiddleware:
    """
    traceId 中间件（必须处于最外层）：
    - 每请求生成唯一 uuid，注入 contextvars——响应包、日志、后台任务全都能读；
    - 响应头回写 X-Trace-Id，便于用户报障定位；
    - 同时写入 scope["state"]，异常处理器/依赖里也能通过 request.state 兜底读取。
    """

    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        trace_id = str(uuid.uuid4())
        set_trace_id(trace_id)
        scope.setdefault("state", {})["traceId"] = trace_id

        async def send_with_header(message: Message) -> None:
            if message["type"] == "http.response.start":
                # 在响应头里追加 X-Trace-Id（无论成败，含 4xx/5xx/500）
                MutableHeaders(scope=message).append("X-Trace-Id", trace_id)
            await send(message)

        await self.app(scope, receive, send_with_header)


class RequestLogMiddleware:
    """结构化请求日志中间件：每请求输出一行 JSON（含 traceId）。"""

    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        start = time.monotonic()
        holder = {"status": 500}

        async def send_wrap(message: Message) -> None:
            if message["type"] == "http.response.start":
                holder["status"] = message["status"]
            await send(message)

        try:
            await self.app(scope, receive, send_wrap)
        finally:
            # finally 保证：出错上抛前也留下一行请求日志
            state = scope.get("state") or {}
            user = state.get("user")
            logger.info(json.dumps({
                "ts": datetime.now(timezone.utc).isoformat(),
                "kind": "http_request",
                "traceId": get_trace_id(),
                "method": scope["method"],
                "path": scope["path"],
                "status": holder["status"],
                "durationMs": int((time.monotonic() - start) * 1000),
                "userId": user.userId if user else None,
                "ip": (scope.get("client") or ["?"])[0],
            }, ensure_ascii=False))
```

## 6.5 启用与验证

`app/main.py` 本章只改两处（其余与 5.4 相同）。

第一处：文件顶部新增 import：

```python
from app.middleware.audit_log import AuditLogMiddleware
from app.middleware.request_log import RequestLogMiddleware, TraceIdMiddleware
```

第二处：`create_app()` 里、路由挂载之后追加中间件注册——**注意 `add_middleware` 后加者在最外层**，注册顺序与洋葱顺序相反：

```python
    # ---------------- 中间件（洋葱：TraceId -> RequestLog -> Audit -> 业务） ----------------
    # add_middleware 是"后加者在最外层"：先注册最贴近业务的，最后注册最外层的
    app.add_middleware(AuditLogMiddleware)    # 第 3 层：贴近业务，记录写操作（要拿到最终状态码与用户）
    app.add_middleware(RequestLogMiddleware)  # 第 2 层：结构化请求日志
    app.add_middleware(TraceIdMiddleware)     # 第 1 层：最外层，最先设置 traceId
```

先重跑种子脚本建出 `audit_logs` 表（注意会清库重建），再重启服务（建议加 `--no-access-log`，避免 uvicorn 自带的访问日志淹没我们的 JSON 行）：

```bash
python scripts/init_db.py
python -m uvicorn app.main:app --host 127.0.0.1 --port 7001 --reload --no-access-log
```

然后执行一次创建操作：

```bash
curl -s -X POST http://127.0.0.1:7001/api/v1/wafers \
  -H 'Content-Type: application/json' \
  -d '{"batchNo":"B20261015008","productModel":"MCU-XS32","processNode":"28nm","productionLine":"LINE-1","quantity":25,"yieldRate":93.00}'
```

**验证 1（traceId 响应头）**：

```bash
curl -si -X POST http://127.0.0.1:7001/api/v1/wafers \
  -H 'Content-Type: application/json' \
  -d '{"batchNo":"B20261015009","productModel":"MCU-XS32","processNode":"28nm","productionLine":"LINE-1","quantity":25,"yieldRate":93.00}' \
  | grep -i x-trace-id
# 预期输出：X-Trace-Id: xxxxxxxx-xxxx-...
```

**验证 2（结构化日志）**：查看终端 dev 日志，应出现一行 JSON：

```text
{"ts":"2026-10-15T09:00:00.000Z","kind":"http_request","traceId":"...","method":"POST","path":"/api/v1/wafers","status":201,"durationMs":23,"userId":null,"ip":"::1"}
```

**验证 3（审计表）**：

```bash
docker exec -it ia-pg psql -U industry -d industryagent \
  -c "SELECT id, username, method, path, status, duration_ms, trace_id FROM audit_logs ORDER BY id DESC LIMIT 3;"
```

预期输出（`psql` 表格）：

```text
 id | username  | method |      path       | status | duration_ms |             trace_id
----+-----------+--------+-----------------+--------+-------------+----------------------------------
  2 | anonymous | POST   | /api/v1/wafers  |    201 |          23 | ...
  1 | anonymous | POST   | /api/v1/wafers  |    201 |          18 | ...
```

> `username` 目前是 `anonymous`，因为鉴权还没做——第 7 章完成后，这里就会变成真实用户名。

## 6.6 常见坑

1. **为什么不用 `@app.middleware("http")` / `BaseHTTPMiddleware`**：它在中间件里 `await request.body()` 后，内层路由再读体会挂起（body 已被消费且不会自动回注）。审计需要"留存请求体又不惊动内层"，纯 ASGI 包一层 `receive` 最干净。请求日志/traceId 不读体，但保持同一种写法降低心智负担。
2. **`add_middleware` 的顺序是反的**：后注册的在最外层。想要 `TraceId -> RequestLog -> Audit` 的洋葱，就得按 Audit、RequestLog、TraceId 的顺序注册。
3. **裸 `asyncio.create_task` 会被垃圾回收**：asyncio 对任务只持弱引用，没有强引用的任务可能执行到一半被 GC。标准解法是模块级 `set` 收纳 + `add_done_callback(discard)`（见 `audit_log.py`）。
4. **后台任务里用请求会话**：响应一返回 `get_db` 的会话就关了，后台任务必须 `async with AsyncSessionLocal()` 自开短会话（见 `AuditService.write`）。
5. **contextvars 的传播边界**：`set` 必须发生在最外层中间件（进入内层之前）。跨任务（`create_task`）只能"复制"不能"回传"——所以先 set 再进内层，处处可见。
6. **审计写库失败不能影响业务**：`AuditService.write` 整体 try/except + 独立会话 + 警告日志，三层保险。

---

# 7. 用户与鉴权：JWT + RBAC

## 7.1 做什么 / 为什么

手册的组织约束明确："关键建议需人工审批""RBAC、审计日志"。本章实现：

- **users / roles / user_roles 三张表**（经典 RBAC 多对多）
- **登录**：查库 + bcrypt 密码比对 + 签发 access token（2h）
- **刷新 token**：长效 refresh token（7 天）存 Redis 白名单，可主动失效；刷新时**轮换**（旧 refresh 作废、发新的一对）
- **token 校验依赖**：`Depends(get_current_user)` 解析 payload 挂到当前用户，并检查 Redis 黑名单（登出即拉黑）
- **权限校验依赖**：路由级声明所需权限，按角色映射表判定

Egg 版用 `auth()` / `permission()` 两个中间件；FastAPI 的对应物是**依赖注入**——`Depends(require_perm("wafer:delete"))` 写在路由上，粒度更细，且天然先于业务代码执行（审计中间件仍能看到 `request.state.user`）。

本章用到第 2 章 requirements.txt 里的 `bcrypt`、`PyJWT`、`redis`（若尚未安装：`pip install bcrypt pyjwt redis`）。

## 7.2 数据模型（完整内容）

`app/models/user.py`：

```python
from datetime import datetime, timezone

from sqlalchemy import DateTime, String
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base


def _utcnow() -> datetime:
    return datetime.now(timezone.utc)


class User(Base):
    """用户表 users（列名沿用原库的小写下划线：password_hash/real_name/...）"""
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    username: Mapped[str] = mapped_column(String(64), nullable=False, unique=True)
    passwordHash: Mapped[str] = mapped_column("password_hash", String(128), nullable=False,
                                              comment="bcrypt 哈希，永不明文存密码")
    realName: Mapped[str | None] = mapped_column("real_name", String(64), nullable=True)
    email: Mapped[str | None] = mapped_column(String(128), nullable=True)
    status: Mapped[str] = mapped_column(String(16), nullable=False, default="active",
                                        comment="active / disabled")
    lastLoginAt: Mapped[datetime | None] = mapped_column("last_login_at", DateTime(timezone=True), nullable=True)
    createdAt: Mapped[datetime] = mapped_column("created_at", DateTime(timezone=True), nullable=False, default=_utcnow)
    updatedAt: Mapped[datetime] = mapped_column("updated_at", DateTime(timezone=True), nullable=False,
                                                default=_utcnow, onupdate=_utcnow)
```

`app/models/role.py`：

```python
from datetime import datetime, timezone

from sqlalchemy import DateTime, String, Text
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base


def _utcnow() -> datetime:
    return datetime.now(timezone.utc)


class Role(Base):
    """角色表 roles"""
    __tablename__ = "roles"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    code: Mapped[str] = mapped_column(String(32), nullable=False, unique=True,
                                      comment="角色编码：admin/process_engineer/reviewer")
    name: Mapped[str] = mapped_column(String(64), nullable=False)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    createdAt: Mapped[datetime] = mapped_column("created_at", DateTime(timezone=True), nullable=False, default=_utcnow)
    updatedAt: Mapped[datetime] = mapped_column("updated_at", DateTime(timezone=True), nullable=False,
                                                default=_utcnow, onupdate=_utcnow)
```

`app/models/user_role.py`：

```python
from datetime import datetime, timezone

from sqlalchemy import DateTime, UniqueConstraint
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base


def _utcnow() -> datetime:
    return datetime.now(timezone.utc)


class UserRole(Base):
    """用户-角色关联表 user_roles"""
    __tablename__ = "user_roles"
    __table_args__ = (
        # 同一用户同一角色只绑一次
        UniqueConstraint("user_id", "role_id", name="uq_user_roles_user_id_role_id"),
    )

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    userId: Mapped[int] = mapped_column("user_id", nullable=False)
    roleId: Mapped[int] = mapped_column("role_id", nullable=False)
    createdAt: Mapped[datetime] = mapped_column("created_at", DateTime(timezone=True), nullable=False, default=_utcnow)
    updatedAt: Mapped[datetime] = mapped_column("updated_at", DateTime(timezone=True), nullable=False,
                                                default=_utcnow, onupdate=_utcnow)
```

`app/models/__init__.py` 更新为：

```python
from app.models.audit_log import AuditLog
from app.models.base import Base
from app.models.role import Role
from app.models.user import User
from app.models.user_role import UserRole
from app.models.wafer import Wafer

__all__ = ["AuditLog", "Base", "Role", "User", "UserRole", "Wafer"]
```

## 7.3 角色-权限映射：app/core/role_permissions.py（完整内容）

**做什么/为什么**：把"角色拥有哪些权限"做成一张静态映射表（权限点编码与路由声明一一对应）。小型系统这样最直观；更复杂的场景可升级为数据库存储 + OPA 策略引擎。

`app/core/role_permissions.py`（完整内容，映射按重写规格第四节冻结；`report:edit` 等权限点对应第 9/18 章的接口，此处一并冻结，避免后续再改这张表）：

```python
"""
角色 -> 权限点映射
权限点命名：<资源>:<动作>，如 wafer:create
"""

ROLE_PERMISSIONS: dict[str, list[str]] = {
    "admin": ["*"],  # 管理员：全部权限
    # 工艺工程师
    "process_engineer": ["wafer:create", "wafer:update", "rootcause:trigger", "report:edit", "defect:upload"],
    # 审批员（质量/主管）
    "reviewer": ["rootcause:trigger", "suggestion:approve", "report:release"],
    # 只读（第 18 章加入该角色）
    "viewer": [],
}


def has_permission(roles: list[str], permission: str) -> bool:
    """判断一组角色是否拥有某权限"""
    return any(
        "*" in ROLE_PERMISSIONS.get(role, []) or permission in ROLE_PERMISSIONS.get(role, [])
        for role in roles
    )
```

## 7.4 令牌与密码工具：app/core/security.py 与 app/core/redis.py（完整内容）

**做什么/为什么**：Egg 版直接用 `app.jwt` / `app.redis` 插件；FastAPI 版自己装"电池"。JWT 用 **PyJWT（HS256，第 19 章升级 RS256）**，密码用 **bcrypt**（种子哈希是 `$2b$` 前缀、cost 10，与 Python 的 bcrypt 包完全兼容）。

`app/core/redis.py`（完整内容，全局唯一的 Redis 客户端）：

```python
"""全局 Redis 客户端：refresh 白名单 / 登出黑名单 / 缓存 / 幂等键共用。"""
from redis.asyncio import Redis

from app.core.config import get_settings

_settings = get_settings()

redis_client: Redis = Redis(
    host=_settings.redis_host,
    port=_settings.redis_port,
    password=_settings.redis_password or None,
    db=_settings.redis_db,
    decode_responses=True,  # 读回来直接是 str，省去到处 decode
)
```

> 若第 4 章 `Settings` 还没有 redis 相关字段，补上（默认值对应 `.env` 的 `IA_REDIS_HOST/PORT`）：
>
> ```python
>     redis_host: str = "127.0.0.1"
>     redis_port: int = 6379
>     redis_password: str = ""
>     redis_db: int = 0
> ```

`app/core/security.py`（完整内容）：

```python
"""密码哈希与 JWT 签发/验签（对标 Egg 的 egg-jwt + bcryptjs）。"""
import uuid
from datetime import datetime, timedelta, timezone

import bcrypt
import jwt

from app.core.config import get_settings

ALGORITHM = "HS256"  # 第 19 章 Keycloak 改造时升级为 RS256


# ---------------- bcrypt ----------------

def hash_password(password: str) -> str:
    """cost 10：约几十毫秒，兼顾安全与登录体验（与种子数据一致）"""
    return bcrypt.hashpw(password.encode("utf-8"), bcrypt.gensalt(rounds=10)).decode("ascii")


def verify_password(password: str, password_hash: str) -> bool:
    try:
        return bcrypt.checkpw(password.encode("utf-8"), password_hash.encode("ascii"))
    except ValueError:
        # 哈希格式非法（历史脏数据/被截断）一律视为不匹配，绝不让它变成 500
        return False


# ---------------- JWT ----------------

def create_access_token(*, user_id: int, username: str, roles: list[str]) -> str:
    """签发访问令牌（2h）。带 jti：登出时按它拉黑。"""
    settings = get_settings()
    now = datetime.now(timezone.utc)  # 必须带时区：见 7.10 常见坑第 2 条
    payload = {
        "userId": user_id,
        "username": username,
        "roles": roles,
        "type": "access",
        "jti": str(uuid.uuid4()),
        "iat": now,
        "exp": now + timedelta(seconds=settings.access_token_expire_seconds),
    }
    return jwt.encode(payload, settings.jwt_secret, algorithm=ALGORITHM)


def create_refresh_token(*, user_id: int, username: str) -> tuple[str, str]:
    """签发刷新令牌（7d）。返回 (token, jti)：jti 是 Redis 白名单的键。"""
    settings = get_settings()
    now = datetime.now(timezone.utc)
    jti = str(uuid.uuid4())
    payload = {
        "userId": user_id,
        "username": username,
        "jti": jti,
        "type": "refresh",
        "iat": now,
        "exp": now + timedelta(seconds=settings.refresh_token_expire_seconds),
    }
    return jwt.encode(payload, settings.jwt_secret, algorithm=ALGORITHM), jti


def decode_token(token: str) -> dict:
    """验签 + 过期检查；失败抛 jwt.InvalidTokenError / jwt.ExpiredSignatureError，
    由调用方（deps / auth service）翻译成 BizError。"""
    return jwt.decode(token, get_settings().jwt_secret, algorithms=[ALGORITHM])
```

> 若第 4 章 `Settings` 还没有令牌有效期字段，补上（对标原 `config.biz.auth`）：
>
> ```python
>     # ---------- 访问令牌 / 刷新令牌有效期 ----------
>     access_token_expire_seconds: int = 2 * 3600        # 2h
>     refresh_token_expire_seconds: int = 7 * 24 * 3600  # 7d
> ```

## 7.5 鉴权依赖：app/core/deps.py（完整内容）

**做什么/为什么**：对标原 7.4 鉴权中间件 + 7.5 权限中间件。FastAPI 的做法是依赖注入：`HTTPBearer` 从 `Authorization: Bearer <token>` 取令牌，`get_current_user` 完成"解析 → 黑名单检查 → 查用户 → 构造当前用户"，`require_perm(...)` 是权限工厂。选 `HTTPBearer` 而不是 `OAuth2PasswordBearer` 的原因见 7.10 常见坑第 3 条。

`app/core/deps.py`（完整内容）：

```python
from dataclasses import dataclass, field

import jwt
from fastapi import Depends, Request
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.biz_error import BizError
from app.core.database import get_db
from app.core.error_codes import ErrorCode
from app.core.redis import redis_client
from app.core.role_permissions import has_permission
from app.core.security import decode_token
from app.models.user import User
from app.services.auth import AuthService

# auto_error=False：缺头时不让框架抛它自己的 403（响应格式会脱离统一响应包），
# 由我们抛 BizError(20001/401)
bearer_scheme = HTTPBearer(auto_error=False)


@dataclass
class CurrentUser:
    """当前登录用户（对标原 ctx.user，多了两个内部字段）"""
    userId: int
    username: str
    realName: str | None
    roles: list[str] = field(default_factory=list)
    jti: str = ""      # 内部：登出时按它拉黑 access token
    expiresAt: int = 0  # 内部：计算黑名单剩余 TTL


async def get_current_user(
    request: Request,
    credentials: HTTPAuthorizationCredentials | None = Depends(bearer_scheme),
    db: AsyncSession = Depends(get_db),
) -> CurrentUser:
    """JWT 鉴权依赖：解析 token -> Redis 黑名单检查 -> 查用户 -> 构造 CurrentUser"""
    if credentials is None:
        raise BizError(ErrorCode.UNAUTHORIZED, "缺少访问令牌（Authorization: Bearer <token>）", 401)

    token = credentials.credentials
    try:
        payload = decode_token(token)
    except jwt.ExpiredSignatureError:
        # 区分"过期"与"非法"，前端据此决定是否静默刷新
        raise BizError(ErrorCode.TOKEN_EXPIRED, "登录已过期，请刷新令牌或重新登录", 401)
    except jwt.InvalidTokenError:
        raise BizError(ErrorCode.UNAUTHORIZED, "无效的访问令牌", 401)

    if payload.get("type") != "access":
        # 拿 refresh token 当 access 用：直接拒绝
        raise BizError(ErrorCode.UNAUTHORIZED, "无效的访问令牌", 401)

    # Redis 黑名单（已登出的 token 在剩余有效期内一律拒绝）
    jti = payload.get("jti", "")
    if jti and await redis_client.exists(f"ia:blacklist:{jti}"):
        raise BizError(ErrorCode.UNAUTHORIZED, "无效的访问令牌", 401)

    # 以库内现状为准：用户被删/被禁后，旧 token 立即失效
    user = await db.get(User, payload["userId"])
    if user is None or user.status != "active":
        raise BizError(ErrorCode.UNAUTHORIZED, "无效的访问令牌", 401)

    roles = await AuthService.get_user_role_codes(db, user.id)
    current = CurrentUser(
        userId=user.id,
        username=user.username,
        realName=user.realName,
        roles=roles,
        jti=jti,
        expiresAt=int(payload.get("exp", 0)),
    )
    # 挂到 request.state：审计/请求日志中间件据此记录"谁"在操作
    request.state.user = current
    return current


def require_perm(permission: str):
    """
    权限校验依赖工厂。
    用法：router.post("/api/v1/wafers", dependencies=[Depends(require_perm("wafer:create"))])
    """
    async def checker(user: CurrentUser = Depends(get_current_user)) -> CurrentUser:
        if not has_permission(user.roles, permission):
            raise BizError(
                ErrorCode.FORBIDDEN,
                f"没有权限：需要 {permission}（当前角色：{','.join(user.roles) or '无'}）",
                403,
            )
        return user
    return checker
```

## 7.6 认证 Service：app/services/auth.py（完整内容）

**做什么/为什么**：登录/刷新/登出的全部规则集中在此。refresh token 的白名单与轮换、access token 的黑名单都在 Redis——**Redis 成为"可主动踢人下线"的那块状态**，这是纯 JWT 做不到的。

`app/services/auth.py`（完整内容）：

```python
import time
from datetime import datetime, timezone

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.biz_error import BizError
from app.core.config import get_settings
from app.core.error_codes import ErrorCode
from app.core.redis import redis_client
from app.core.security import (
    create_access_token,
    create_refresh_token,
    decode_token,
    verify_password,
)
from app.models.role import Role
from app.models.user import User
from app.models.user_role import UserRole

REFRESH_PREFIX = "ia:refresh:"    # refresh 白名单键前缀
BLACKLIST_PREFIX = "ia:blacklist:"  # access 黑名单键前缀


class AuthService:

    @staticmethod
    async def get_user_role_codes(db: AsyncSession, user_id: int) -> list[str]:
        """查某用户的角色编码列表（deps 与本服务共用）"""
        result = await db.execute(
            select(Role.code)
            .join(UserRole, UserRole.roleId == Role.id)
            .where(UserRole.userId == user_id)
        )
        return [row[0] for row in result.all()]

    @staticmethod
    async def login(db: AsyncSession, username: str, password: str) -> dict:
        """登录：查库 + bcrypt 比对 + 签发双 token"""
        user = (
            await db.execute(select(User).where(User.username == username))
        ).scalar_one_or_none()
        # 注意：用户不存在与密码错误返回同一话术，避免用户名枚举攻击
        if user is None:
            raise BizError(ErrorCode.LOGIN_FAILED, "用户名或密码错误")

        if not verify_password(password, user.passwordHash):
            raise BizError(ErrorCode.LOGIN_FAILED, "用户名或密码错误")

        if user.status != "active":
            raise BizError(ErrorCode.USER_DISABLED, "账号已被禁用，请联系管理员", 403)

        roles = await AuthService.get_user_role_codes(db, user.id)

        # 更新最后登录时间（失败不影响主流程：最坏情况少记一次）
        user.lastLoginAt = datetime.now(timezone.utc)
        await db.commit()

        return await AuthService._issue_tokens(
            user_id=user.id, username=user.username, real_name=user.realName, roles=roles,
        )

    @staticmethod
    async def refresh(db: AsyncSession, refresh_token: str) -> dict:
        """刷新 access token：refresh token 必须仍在 Redis 白名单内；旧票作废（轮换）"""
        try:
            payload = decode_token(refresh_token)
        except Exception:
            # 过期/非法统一话术，引导用户重新登录
            raise BizError(ErrorCode.TOKEN_EXPIRED, "刷新令牌无效或已过期，请重新登录", 401)

        if payload.get("type") != "refresh":
            raise BizError(ErrorCode.UNAUTHORIZED, "请使用刷新令牌调用本接口", 401)

        user_id_in_redis = await redis_client.get(REFRESH_PREFIX + payload.get("jti", ""))
        if not user_id_in_redis or user_id_in_redis != str(payload["userId"]):
            # 已登出或被管理员吊销
            raise BizError(ErrorCode.TOKEN_EXPIRED, "刷新令牌已失效，请重新登录", 401)

        user = await db.get(User, payload["userId"])
        if user is None or user.status != "active":
            raise BizError(ErrorCode.TOKEN_EXPIRED, "刷新令牌已失效，请重新登录", 401)

        roles = await AuthService.get_user_role_codes(db, user.id)

        # 轮换：旧的 refresh 立即作废，再用新的一对
        await redis_client.delete(REFRESH_PREFIX + payload.get("jti", ""))

        return await AuthService._issue_tokens(
            user_id=user.id, username=user.username, real_name=user.realName, roles=roles,
        )

    @staticmethod
    async def logout(jti: str, expires_at: int, refresh_token: str | None) -> dict:
        """登出：refresh 出白名单 + access 进黑名单（剩余存活期内拒绝）"""
        if refresh_token:
            try:
                payload = decode_token(refresh_token)
                if payload.get("jti"):
                    await redis_client.delete(REFRESH_PREFIX + payload["jti"])
            except Exception:
                pass  # 无效 token 直接视为已吊销

        if jti:
            remaining = max(1, expires_at - int(time.time()))
            await redis_client.setex(BLACKLIST_PREFIX + jti, remaining, "1")
        return {"revoked": True}

    @staticmethod
    async def _issue_tokens(*, user_id: int, username: str, real_name: str | None, roles: list[str]) -> dict:
        """签发 access + refresh 双 token（refresh 白名单写 Redis：可主动吊销）"""
        settings = get_settings()

        access_token = create_access_token(user_id=user_id, username=username, roles=roles)
        refresh_token, jti = create_refresh_token(user_id=user_id, username=username)

        await redis_client.setex(
            REFRESH_PREFIX + jti,
            settings.refresh_token_expire_seconds,
            str(user_id),
        )

        return {
            "accessToken": access_token,
            "refreshToken": refresh_token,
            "tokenType": "Bearer",
            "expiresInSeconds": settings.access_token_expire_seconds,
            "user": {
                "userId": user_id,
                "username": username,
                "realName": real_name,
                "roles": roles,
            },
        }
```

## 7.7 认证路由：app/schemas/auth.py 与 app/routers/auth.py（完整内容）

**做什么/为什么**：对标原 auth controller 的四个接口（login/refresh/logout/profile）。校验规则同样用 Pydantic 承接：`username` 3~64、`password` 6~64。

`app/schemas/auth.py`（完整内容）：

```python
from pydantic import BaseModel, Field


class LoginRequest(BaseModel):
    """POST /api/v1/auth/login 请求体（对标原 loginRule）"""
    username: str = Field(min_length=3, max_length=64)
    password: str = Field(min_length=6, max_length=64)


class RefreshRequest(BaseModel):
    """POST /api/v1/auth/refresh 请求体"""
    refreshToken: str = Field(min_length=1)


class LogoutRequest(BaseModel):
    """POST /api/v1/auth/logout 请求体（refreshToken 可选携带）"""
    refreshToken: str | None = None
```

`app/routers/auth.py`（完整内容）：

```python
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.core import response
from app.core.database import get_db
from app.core.deps import CurrentUser, get_current_user
from app.models.user import User
from app.schemas.auth import LoginRequest, LogoutRequest, RefreshRequest
from app.services.auth import AuthService

router = APIRouter(prefix="/api/v1/auth", tags=["auth"])


@router.post("/login")
async def login(payload: LoginRequest, db: AsyncSession = Depends(get_db)):
    """POST /api/v1/auth/login"""
    return response.ok(await AuthService.login(db, payload.username, payload.password))


@router.post("/refresh")
async def refresh(payload: RefreshRequest, db: AsyncSession = Depends(get_db)):
    """POST /api/v1/auth/refresh"""
    return response.ok(await AuthService.refresh(db, payload.refreshToken))


@router.post("/logout")
async def logout(
    payload: LogoutRequest | None = None,  # body 可选携带 refreshToken
    user: CurrentUser = Depends(get_current_user),
):
    """POST /api/v1/auth/logout"""
    result = await AuthService.logout(user.jti, user.expiresAt, payload.refreshToken if payload else None)
    return response.ok(result)


@router.get("/profile")
async def profile(user: CurrentUser = Depends(get_current_user), db: AsyncSession = Depends(get_db)):
    """GET /api/v1/auth/profile：当前登录用户信息"""
    db_user = await db.get(User, user.userId)
    return response.ok({
        "userId": user.userId,
        "username": user.username,
        "realName": db_user.realName if db_user else None,
        "email": db_user.email if db_user else None,
        "roles": user.roles,
    })
```

`app/main.py` 本章改三处（其余与第 6 章相同）：

```python
# ① 文件顶部：新增 redis 客户端 import，routers 一行加入 auth
from app.core.redis import redis_client  # ★ 第 7 章新增
from app.routers import auth, home, wafer  # ★ 第 7 章新增 auth
```

```python
# ② lifespan：接管 Redis 生命周期（★ 两处新增）
@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    # Redis 强依赖（refresh 白名单/登出黑名单）：连不上就启动失败，避免带病上线
    await redis_client.ping()
    logger.info("[startup] industryagent-server 就绪（env=%s，redis=ok）", settings.env)
    yield
    await redis_client.close()  # ★
    await engine.dispose()
    logger.info("[shutdown] 数据库连接池与 Redis 已释放")
```

```python
# ③ create_app()：挂载认证路由（在 home 之后）
    app.include_router(auth.router)
```

## 7.8 种子数据追加用户与角色

在 `scripts/init_db.py` 的 `main()` 中，替换注释行 `# 用户/角色等种子数据在第 7 章追加` 为：

```python
    # ---------- 种子数据：用户与角色 ----------
    from app.core.security import hash_password
    from app.models.role import Role
    from app.models.user import User
    from app.models.user_role import UserRole

    async with AsyncSessionLocal() as db:
        admin_role = Role(code="admin", name="管理员", description="全部权限")
        engineer_role = Role(code="process_engineer", name="工艺工程师", description="批次维护与根因分析")
        reviewer_role = Role(code="reviewer", name="审批员", description="建议审批")
        db.add_all([admin_role, engineer_role, reviewer_role])
        await db.flush()  # 先拿到自增 id，再建关联

        admin_user = User(username="admin", passwordHash=hash_password("Admin@123"),
                          realName="系统管理员", email="admin@xinlian.demo")
        engineer_user = User(username="engineer", passwordHash=hash_password("Engineer@123"),
                             realName="李工（工艺）", email="engineer@xinlian.demo")
        reviewer_user = User(username="reviewer", passwordHash=hash_password("Reviewer@123"),
                             realName="王主管（质量）", email="reviewer@xinlian.demo")
        db.add_all([admin_user, engineer_user, reviewer_user])
        await db.flush()

        db.add_all([
            UserRole(userId=admin_user.id, roleId=admin_role.id),
            UserRole(userId=engineer_user.id, roleId=engineer_role.id),
            UserRole(userId=reviewer_user.id, roleId=reviewer_role.id),
        ])
        await db.commit()
    print("[seed] 用户与角色写入完成：admin/engineer/reviewer")
```

重新执行 `python scripts/init_db.py`（会清库重建，之前的测试数据无需心疼）。预期输出：

```text
[seed] 同步表结构（drop+create 会清空重建）...
[seed] 表结构同步完成
[seed] 晶圆批次 5 条写入完成
[seed] 用户与角色写入完成：admin/engineer/reviewer
[seed] 全部完成 ✔
```

## 7.9 路由更新与全链路验证

`app/routers/wafer.py` 更新为（完整内容，鉴权时代来临）：

```python
from typing import Literal

from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.core import response
from app.core.config import get_settings
from app.core.database import get_db
from app.core.deps import CurrentUser, get_current_user, require_perm
from app.schemas.wafer import WaferCreate, WaferOut, WaferUpdate
from app.services.wafer import WaferService

router = APIRouter(prefix="/api/v1/wafers", tags=["wafers"])

WaferStatusQuery = Literal["producing", "completed", "shipped", "quarantined"]


def _dump(wafer) -> dict:
    return WaferOut.model_validate(wafer).model_dump(mode="json")


@router.get("", dependencies=[Depends(get_current_user)])
async def list_wafers(
    page: int | None = Query(default=None, ge=1, description="页码"),
    pageSize: int | None = Query(default=None, ge=1, le=500, description="每页条数"),
    batchNo: str | None = Query(default=None, description="批次号模糊匹配"),
    status: WaferStatusQuery | None = Query(default=None, description="批次状态"),
    productionLine: str | None = Query(default=None, description="产线"),
    db: AsyncSession = Depends(get_db),
):
    """GET /api/v1/wafers（需登录）"""
    settings = get_settings()
    page_num = page or settings.default_page
    page_size = min(pageSize or settings.default_page_size, settings.max_page_size)

    rows, total = await WaferService.list(
        db,
        page=page_num,
        page_size=page_size,
        batch_no=batchNo or None,
        status=status,
        production_line=productionLine or None,
    )
    return response.page([_dump(w) for w in rows], total, page_num, page_size)


@router.get("/{wafer_id}", dependencies=[Depends(get_current_user)])
async def get_wafer(wafer_id: int, db: AsyncSession = Depends(get_db)):
    """GET /api/v1/wafers/:id（需登录）"""
    return response.ok(_dump(await WaferService.detail(db, wafer_id)))


@router.post("", status_code=201)
async def create_wafer(
    payload: WaferCreate,
    _user: CurrentUser = Depends(require_perm("wafer:create")),
    db: AsyncSession = Depends(get_db),
):
    """POST /api/v1/wafers（需 wafer:create 权限）"""
    wafer = await WaferService.create(db, payload)
    return response.ok(_dump(wafer), "创建成功")


@router.put("/{wafer_id}")
async def update_wafer(
    wafer_id: int,
    payload: WaferUpdate,
    _user: CurrentUser = Depends(require_perm("wafer:update")),
    db: AsyncSession = Depends(get_db),
):
    """PUT /api/v1/wafers/:id（需 wafer:update 权限）"""
    wafer = await WaferService.update(db, wafer_id, payload)
    return response.ok(_dump(wafer), "更新成功")


@router.delete("/{wafer_id}")
async def delete_wafer(
    wafer_id: int,
    _user: CurrentUser = Depends(require_perm("wafer:delete")),
    db: AsyncSession = Depends(get_db),
):
    """DELETE /api/v1/wafers/:id（需 wafer:delete 权限）"""
    result = await WaferService.remove(db, wafer_id)
    return response.ok(result, "删除成功")
```

> `wafer:delete` 故意**没有**分配给任何角色（连 admin 也没有？不——admin 有 `*`）。观察：engineer 删除会 403，admin 删除会成功，这就是 RBAC 生效的证据。

重启服务（`python -m uvicorn app.main:app --host 127.0.0.1 --port 7001 --reload --no-access-log`），依次验证：

**① 未带 token 访问 → 401**

```bash
curl -si http://127.0.0.1:7001/api/v1/wafers | head -n 1
# 预期输出：HTTP/1.1 401 Unauthorized
```

**② 登录**

```bash
curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"engineer","password":"Engineer@123"}'
```

预期输出：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "tokenType": "Bearer",
    "expiresInSeconds": 7200,
    "user": { "userId": 2, "username": "engineer", "realName": "李工（工艺）", "roles": ["process_engineer"] }
  },
  "traceId": "..."
}
```

**③ 携带 token 访问 → 200**

```bash
TOKEN='上一步返回的 accessToken'
curl -s http://127.0.0.1:7001/api/v1/wafers -H "Authorization: Bearer $TOKEN" | head -c 200
# 预期输出：{"code":0,"message":"ok","data":{"rows":[...
```

**④ 权限验证：engineer 触发删除 → 403**

```bash
curl -si -X DELETE http://127.0.0.1:7001/api/v1/wafers/1 \
  -H "Authorization: Bearer $TOKEN" | head -n 1
# 预期输出：HTTP/1.1 403 Forbidden
# 响应体：{"code":20003,"message":"没有权限：需要 wafer:delete（当前角色：process_engineer）",...}
```

**⑤ 刷新 token**

```bash
REFRESH='第②步返回的 refreshToken'
curl -s -X POST http://127.0.0.1:7001/api/v1/auth/refresh \
  -H 'Content-Type: application/json' \
  -d "{\"refreshToken\":\"$REFRESH\"}"
# 预期输出：新的 accessToken / refreshToken
```

**⑥ 审计中间件联动验证**：再看审计表，`username` 已变成真实用户名：

```bash
docker exec -it ia-pg psql -U industry -d industryagent \
  -c "SELECT username, method, path, status FROM audit_logs ORDER BY id DESC LIMIT 3;"
# 预期：engineer | DELETE | /api/v1/wafers/1 | 403
```

> 附赠一个 Swagger 侧的验证：打开 `http://127.0.0.1:7001/docs`，点右上角 **Authorize**，把 ② 返回的 accessToken 粘进 Bearer 框，即可在文档页里直接调受保护接口——`HTTPBearer` 方案与我们的 JSON 登录契约天然兼容（见 7.10 第 3 条）。

## 7.10 常见坑

1. **bcrypt 的 72 字节上限**：`bcrypt` 只取密码前 72 字节，超出部分静默截断（`Admin@123` 这类短密码无碍，但别用它保护长口令）。同时注意中文密码按 UTF-8 计字节，一个汉字 3 字节。要么在 schema 层限制长度，要么先 SHA-256 再 bcrypt。
2. **PyJWT 的 exp 必须带时区**：用 `datetime.now(timezone.utc)`（aware）。`datetime.utcnow()` 在 Python 3.12 已弃用，且产出的 naive 时间一旦被按本地时区解释，就是经典的"token 提前/延后 8 小时过期"事故。
3. **OAuth2PasswordBearer 与 Swagger Authorize 按钮**：`OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")` 会让 Authorize 按钮以**表单**提交账号密码，而我们的登录契约是 JSON——按钮一发就是 422。所以选 `HTTPBearer(auto_error=False)`：Authorize 只收一个 Bearer 令牌，粘 accessToken 即可。
4. **`HTTPBearer` 记得 `auto_error=False`**：默认缺头时框架直接抛 403 `{"detail": "Not authenticated"}`，状态码与响应体都脱离统一契约（我们要 401 + `{"code":20001,...}`）。
5. **`bcrypt.checkpw` 会因非法哈希抛 `ValueError`**（长度不对、非 bcrypt 格式的历史脏数据）——必须兜住返回 False，否则登录变 500。
6. **登录/刷新报 500，先查 Redis**：白名单/黑名单都压在 Redis 上，Redis 没起就是一串 `ConnectionError`。本章 lifespan 里的 `ping()` 就是让它在启动期就炸出来，而不是等到用户登录时。
7. **时钟偏差**：多机部署时签发方与校验方时钟不同步会导致刚签的 token 判过期，必要时 `jwt.decode(..., leeway=10)` 容忍几秒。
8. **JWT payload 不是加密的**：Base64 可解，只放 userId/username/roles 这类标识信息，绝不放密码、手机号。
9. **bcrypt 是 CPU 密集操作**：cost 10 一次约几十毫秒，会阻塞事件循环。教程规模可接受；高并发场景把 `verify_password` 丢进线程池（`await anyio.to_thread.run_sync(verify_password, ...)`）。
