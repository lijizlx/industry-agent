# 8. 异常事件与根因分析模块

## 8.1 做什么 / 为什么

这是平台的核心业务：监控 Agent 发现异常事件（如刻蚀功率漂移、温控超标）→ 工程师触发根因分析 → 平台调用 Python 智能体服务返回 Top-3 根因嫌疑。本章涉及企业级后端最经典的场景：**调用外部服务时的容错**（超时、重试、熔断降级）——在 FastAPI 里对应 `httpx.AsyncClient` + 一个自研的三态熔断器。

涉及的三个角色与端口再强调一次（V2.0 契约冻结）：

| 进程 | 端口 | 说明 |
| --- | --- | --- |
| industryagent-server（本章被调方→调用方） | 7001 | 业务后端 |
| Python 智能体服务（真实/Mock） | **8300** | `.env` 的 `IA_AGENT_BASE_URL=http://127.0.0.1:8300`（V1.x 的 8000 口径已废弃） |

## 8.2 数据模型（完整内容）

`app/models/anomaly_event.py`（完整内容）：

```python
from datetime import datetime, timezone

from sqlalchemy import CheckConstraint, DateTime, String, Text
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base


def _utcnow() -> datetime:
    # asyncpg 写 timestamptz 列拒绝 naive datetime，全库统一"带时区的当前时间"
    return datetime.now(timezone.utc)


class AnomalyEvent(Base):
    """
    异常事件表 anomaly_events
    由监控 Agent / FDC 系统推送写入

    契约冻结：列名沿用原教程 camelCase（eventCode/occurredAt/...），
    种子数据与前端/智能体教程的 JSON 字段零变化。
    """
    __tablename__ = "anomaly_events"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True, comment="主键")
    eventCode: Mapped[str] = mapped_column(String(64), nullable=False, unique=True, comment="事件编号")
    batchNo: Mapped[str | None] = mapped_column(String(32), nullable=True, comment="关联批次号")
    equipment: Mapped[str] = mapped_column(String(64), nullable=False, comment="设备，如 ETCH-03")
    processStage: Mapped[str] = mapped_column(String(32), nullable=False, comment="工序：etch/deposition/litho...")
    severity: Mapped[str] = mapped_column(
        String(16), nullable=False, default="minor", server_default="'minor'", comment="严重度",
    )
    title: Mapped[str] = mapped_column(String(255), nullable=False, comment="事件标题")
    description: Mapped[str | None] = mapped_column(Text, nullable=True, comment="事件描述")
    status: Mapped[str] = mapped_column(
        String(16), nullable=False, default="open", server_default="'open'", comment="处理状态",
    )
    occurredAt: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, comment="发生时间")
    # createdAt / updatedAt 自动维护（对标 sequelize timestamps: true）
    createdAt: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=_utcnow, server_default=text_now(), comment="创建时间",
    )
    updatedAt: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=_utcnow, onupdate=_utcnow,
        server_default=text_now(), comment="更新时间",
    )

    __table_args__ = (
        # 用 CHECK 约束表达枚举：比 PG 原生 ENUM 好删改（同 wafers 的做法）
        CheckConstraint("severity IN ('critical', 'major', 'minor')", name="ck_anomaly_events_severity"),
        CheckConstraint("status IN ('open', 'processing', 'closed')", name="ck_anomaly_events_status"),
    )
```

上面用了一个小助手 `text_now()`——它与 5.2 的 `text("now()")` 等价，只是为了避免同一行写两遍。把它加到 `app/models/base.py` 里，全库共用：

`app/models/base.py`（完整内容，本章版本）：

```python
from sqlalchemy import text
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    """所有模型的公共基类。

    为什么单独一个文件：init_db.py 依赖 Base.metadata 建表，
    各模型文件又依赖 Base 做继承——放到独立模块可避免循环 import。
    """


def text_now():
    """server_default 用的 SQL now() 表达式：插入时由数据库生成时间，绕开 Python 时区问题。"""
    return text("now()")
```

`app/models/agent_task.py`（完整内容）：

```python
from datetime import datetime, timezone

from sqlalchemy import CheckConstraint, DateTime, Integer, String, Text
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base, text_now


def _utcnow() -> datetime:
    return datetime.now(timezone.utc)


class AgentTask(Base):
    """
    根因分析任务表 agent_tasks
    记录每次"人/智能体"协作的分析任务全生命周期
    """
    __tablename__ = "agent_tasks"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True, comment="主键")
    taskNo: Mapped[str] = mapped_column(String(64), nullable=False, unique=True, comment="任务编号 RC+时间戳")
    batchNo: Mapped[str] = mapped_column(String(32), nullable=False, comment="批次号")
    eventId: Mapped[int | None] = mapped_column(Integer, nullable=True, comment="关联异常事件")
    status: Mapped[str] = mapped_column(
        String(16), nullable=False, default="pending", server_default="'pending'", comment="任务状态",
    )
    result: Mapped[dict | None] = mapped_column(JSONB, nullable=True, comment="智能体返回的根因分析结果")
    error: Mapped[str | None] = mapped_column(Text, nullable=True, comment="失败原因")
    createdBy: Mapped[int | None] = mapped_column(Integer, nullable=True, comment="触发人")
    startedAt: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True, comment="开始时间")
    finishedAt: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True, comment="结束时间")
    createdAt: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=_utcnow, server_default=text_now(), comment="创建时间",
    )
    updatedAt: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=_utcnow, onupdate=_utcnow,
        server_default=text_now(), comment="更新时间",
    )

    __table_args__ = (
        CheckConstraint("status IN ('pending', 'running', 'succeeded', 'failed')", name="ck_agent_tasks_status"),
    )
```

`app/models/__init__.py` 更新为（完整内容，"模型注册表"每新增一个模型文件都登记）：

```python
from app.models.agent_task import AgentTask
from app.models.anomaly_event import AnomalyEvent
from app.models.audit_log import AuditLog
from app.models.base import Base
from app.models.role import Role
from app.models.user import User
from app.models.user_role import UserRole
from app.models.wafer import Wafer

__all__ = ["AgentTask", "AnomalyEvent", "AuditLog", "Base", "Role", "User", "UserRole", "Wafer"]
```

## 8.3 Python 智能体 Mock 服务

**做什么/为什么**：真正的智能体教程见《03-智能体开发教程》。为了让后端教程可以独立跑通闭环，这里提供一个 30 行的 FastAPI Mock：睡眠 2 秒模拟推理耗时，返回结构化的 Top-3 根因 + 建议。**响应体逐字节沿用契约冻结版本**（root_causes 三条 confidence 0.87/0.64/0.41、suggestions 两条 param_adjust/hold_wafer），第 13 章 MCP 与第 18 章演示数据都依赖它。

`agent-mock/main.py`（完整内容）：

```python
import asyncio
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="IndustryAgent 智能体服务 (Mock)")


class AnalyzeRequest(BaseModel):
    batch_no: str
    yield_rate: float | None = None
    top_k: int = 3


@app.get("/healthz")
async def health():
    return {"status": "ok"}


@app.post("/agent/analyze")
async def analyze(req: AnalyzeRequest):
    # 模拟多智能体编排（图谱检索 + 因果推断 + RAG）的耗时
    await asyncio.sleep(2)
    return {
        "batch_no": req.batch_no,
        "summary": f"批次 {req.batch_no} 良率异常，疑似刻蚀工序参数漂移（结合知识图谱与近 7 天 FDC 数据）",
        "root_causes": [
            {"rank": 1, "factor": "etch_power_drift", "confidence": 0.87,
             "description": "ETCH-03 刻蚀功率较基准漂移 +3.2%，时间窗与缺陷分布吻合"},
            {"rank": 2, "factor": "chamber_pressure", "confidence": 0.64,
             "description": "腔体压力波动方差增大 1.8 倍"},
            {"rank": 3, "factor": "photoresist_expiry", "confidence": 0.41,
             "description": "光刻胶接近有效期，显影对比度下降"},
        ][: req.top_k],
        "suggestions": [
            {"type": "param_adjust", "content": "将 ETCH-03 射频功率回调至基准值 -2%",
             "risk_level": "low"},
            {"type": "hold_wafer", "content": "隔离该批次剩余在制晶圆，复测 WAT",
             "risk_level": "medium"},
        ],
        "generated_at": asyncio.get_event_loop().time(),
    }
```

安装并启动（与 `app/` 并列同仓；也可以偷懒复用 server 的 `.venv`——依赖完全相同）：

```bash
cd ~/code/industryagent/server/agent-mock
python -m venv .venv
source .venv/bin/activate          # Windows Git Bash: source .venv/Scripts/activate
pip install fastapi uvicorn        # pip 镜像已在第 2 章配好
uvicorn main:app --host 127.0.0.1 --port 8300
```

> **端口口径**：8300 是 V2.0 统一口径（`.env` 已配 `IA_AGENT_BASE_URL=http://127.0.0.1:8300`），与《03-智能体开发教程》《04-部署上线教程》一致。V1.x 教程里的 8000 已废弃，跑错端口时第 8.9 节会全部"无法连接智能体服务"。

**验证**：

```bash
curl -s http://127.0.0.1:8300/healthz
# 预期输出：{"status":"ok"}

curl -s -X POST http://127.0.0.1:8300/agent/analyze \
  -H 'Content-Type: application/json' -d '{"batch_no":"B20261015001"}'
# 预期输出：完整 JSON（root_causes + suggestions），约 2 秒后返回
```

## 8.4 熔断器：app/core/circuit_breaker.py（完整内容）

**做什么/为什么**：当 Python 服务整宕机时，每次请求都等满超时再重试，会把上游的连接池/协程拖死（雪崩）。熔断器像家里的电闸：连续失败 N 次 → "跳闸"（open），直接快速失败；冷却期后放一个请求"试探"（half-open），成功才"合闸"（closed）。

**参数语义（与原 8.4 一致）**：`failure_threshold=3` 连续 3 次任务失败跳闸；`open_duration_ms=30000` 跳闸后冷却 30 秒；重试退避 500ms 起步线性放大（0.5s、1s）。这两个参数已冻结在第 4 章 `Settings`（`breaker_failure_threshold` / `breaker_open_seconds`）。

**注意单例问题**：Egg 的 Service 是每请求新建实例的，熔断状态必须跨请求共享——所以用模块级注册表。FastAPI 的 `XxxService` 是无状态的静态方法类、模块只 import 一次，天然没有这个问题；但注册表写法依然保留——按"被保护依赖的名字"隔离实例，多个外部依赖互不干扰，也不会随请求重建。

`app/core/circuit_breaker.py`（完整内容）：

```python
"""自研熔断器：三态 closed / open / half_open。

- closed  ：正常放行，失败计数达到阈值则跳闸；
- open    ：快速失败（不发起网络调用），冷却期满转入 half_open；
- half_open：放行一次探测请求，成功合闸（closed），失败继续跳闸（open）。
"""
import asyncio
import math
import time
from dataclasses import dataclass
from typing import Awaitable, Callable

from app.core.biz_error import BizError
from app.core.error_codes import ErrorCode


class _BreakerOpenError(BizError):
    """熔断器"自己"抛出的跳闸错误。

    为什么单独一个子类：_call_agent 会把网络异常翻译成 BizError(30001)，
    如果像原 TS 版那样"见到 30001 就免重试"，这些被翻译过的错误会在
    重试循环外被直接抛出、且不计入失败数——熔断器将永远不跳闸，
    8.9 的"第 4 次秒回熔断中"就不成立了。所以只对"熔断器自己抛的"
    错误免重试。
    """


@dataclass(frozen=True)
class BreakerOptions:
    """连续失败多少次跳闸；跳闸后冷却多久（毫秒）"""
    failure_threshold: int
    open_duration_ms: int


class CircuitBreaker:
    def __init__(self, name: str, opts: BreakerOptions) -> None:
        self.name = name
        self.opts = opts
        self.state: str = "closed"        # closed / open / half_open
        self._failure_count = 0
        self._last_open_at: float = 0.0   # time.monotonic() 毫秒

    async def execute(
        self,
        fn: Callable[[int], Awaitable],
        *,
        retries: int,
        retry_delay_ms: int,
    ):
        """执行受保护调用：业务失败自动重试（线性退避），全部重试失败记一次熔断失败。"""
        if self.state == "open":
            elapsed_ms = (time.monotonic() - self._last_open_at) * 1000
            remain_ms = self.opts.open_duration_ms - elapsed_ms
            if remain_ms > 0:
                # 跳闸中：直接降级，不再发起调用
                raise _BreakerOpenError(
                    ErrorCode.AGENT_UNAVAILABLE,
                    f"依赖服务 {self.name} 熔断中（{math.ceil(remain_ms / 1000)}s 后重试）",
                    503,
                )
            # 冷却期满：half-open，放行本次请求做探测
            self.state = "half_open"

        last_error: Exception | None = None
        for attempt in range(retries + 1):
            try:
                result = await fn(attempt)
                self._on_success()  # 探测成功：合闸
                return result
            except _BreakerOpenError:
                raise  # 熔断器自己的错误不重试（避免套娃）
            except Exception as e:
                last_error = e
                if attempt < retries:
                    # 线性退避：500ms、1000ms...（对标原 retryDelayMs * (attempt+1)）
                    await asyncio.sleep(retry_delay_ms / 1000 * (attempt + 1))
        self._on_failure()  # 全部重试失败：跳闸
        assert last_error is not None
        raise last_error

    def _on_success(self) -> None:
        self.state = "closed"
        self._failure_count = 0

    def _on_failure(self) -> None:
        self._failure_count += 1
        if self.state == "half_open" or self._failure_count >= self.opts.failure_threshold:
            self.state = "open"
            self._last_open_at = time.monotonic()


# 模块级注册表：按依赖名共享熔断状态（多个被保护依赖各自一个实例）
_registry: dict[str, CircuitBreaker] = {}


def get_circuit_breaker(name: str, opts: BreakerOptions) -> CircuitBreaker:
    breaker = _registry.get(name)
    if breaker is None:
        breaker = CircuitBreaker(name, opts)
        _registry[name] = breaker
    return breaker
```

## 8.5 Service：app/services/anomaly.py（完整内容）

```python
from datetime import datetime, timezone

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.biz_error import BizError
from app.core.error_codes import ErrorCode
from app.models.anomaly_event import AnomalyEvent


def _ensure_aware(dt: datetime) -> datetime:
    """查询参数可能是 naive datetime（如 2026-10-15T00:00:00）。
    asyncpg 给 timestamptz 列绑参数要求带时区——统一按 UTC 解释。"""
    return dt if dt.tzinfo is not None else dt.replace(tzinfo=timezone.utc)


class AnomalyService:
    """异常事件业务逻辑：分页查询与详情。"""

    @staticmethod
    async def list(
        db: AsyncSession,
        *,
        page: int,
        page_size: int,
        severity: str | None = None,
        status: str | None = None,
        equipment: str | None = None,
        batch_no: str | None = None,
        start_time: datetime | None = None,
        end_time: datetime | None = None,
    ) -> tuple[list[AnomalyEvent], int]:
        """异常事件分页查询（支持严重度/状态/设备/时间范围过滤）"""
        conditions = []
        if severity:
            conditions.append(AnomalyEvent.severity == severity)
        if status:
            conditions.append(AnomalyEvent.status == status)
        if equipment:
            conditions.append(AnomalyEvent.equipment.ilike(f"%{equipment}%"))
        if batch_no:
            conditions.append(AnomalyEvent.batchNo.ilike(f"%{batch_no}%"))
        if start_time:
            conditions.append(AnomalyEvent.occurredAt >= _ensure_aware(start_time))
        if end_time:
            conditions.append(AnomalyEvent.occurredAt <= _ensure_aware(end_time))

        rows = (
            await db.execute(
                select(AnomalyEvent)
                .where(*conditions)
                .order_by(AnomalyEvent.occurredAt.desc())
                .offset((page - 1) * page_size)
                .limit(page_size)
            )
        ).scalars().all()

        total = (
            await db.execute(select(func.count()).select_from(AnomalyEvent).where(*conditions))
        ).scalar_one()
        return list(rows), total

    @staticmethod
    async def detail(db: AsyncSession, event_id: int) -> AnomalyEvent:
        """事件详情"""
        event = await db.get(AnomalyEvent, event_id)
        if event is None:
            raise BizError(ErrorCode.NOT_FOUND, f"异常事件不存在（id={event_id}）", 404)
        return event
```

## 8.6 Service：app/services/rootcause.py（完整内容，含 httpx 容错）

**做什么/为什么**：触发分析的主流程（校验批次 → 建任务 → 调智能体 → 落结果 → 落建议）。与 Egg 版 `ctx.curl` 对应的是 `httpx.AsyncClient`，两个工程要点：

1. **客户端复用**：模块级共享一个 `AsyncClient`（连接池复用 TCP），在 `main.py` 的 lifespan 关闭——每请求新建/关闭客户端是常见的性能事故；
2. **错误翻译**：`httpx` 的超时/连接异常是"服务不可用"（30001，503），HTTP 非 200 或缺 `root_causes` 是"返回异常"（30002，502）——只有翻译成 `BizError`，熔断器才能记账、前端才能拿到统一错误包。

超时语义沿用第 4 章配置：连接 3s / 响应 20s（对标原数组 `[connect, response]`），重试 2 次、退避 500ms 起步。

`app/services/rootcause.py`（完整内容）：

```python
import logging
import random
import time
from datetime import datetime, timezone

import httpx
from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.biz_error import BizError
from app.core.circuit_breaker import BreakerOptions, get_circuit_breaker
from app.core.config import get_settings
from app.core.error_codes import ErrorCode
from app.models.agent_task import AgentTask
from app.models.wafer import Wafer
# from app.models.suggestion import Suggestion  # 第 9 章建表后放开

logger = logging.getLogger("industryagent.rootcause")

# ---------------- 共享 httpx 客户端（连接池复用，lifespan 关闭） ----------------

_agent_client: httpx.AsyncClient | None = None


def get_agent_client() -> httpx.AsyncClient:
    """懒创建共享客户端：复用 TCP 连接，避免每请求握手。
    超时映射：connect=3s（连接）/ read=20s（响应），对标原 [connect, response]。"""
    global _agent_client
    if _agent_client is None:
        settings = get_settings()
        _agent_client = httpx.AsyncClient(
            base_url=settings.agent_base_url,
            timeout=httpx.Timeout(
                connect=settings.rootcause_connect_timeout_ms / 1000,
                read=settings.rootcause_response_timeout_ms / 1000,
                write=5.0,
                pool=5.0,
            ),
        )
    return _agent_client


async def close_agent_client() -> None:
    """lifespan 关闭阶段调用：释放连接池"""
    global _agent_client
    if _agent_client is not None:
        await _agent_client.aclose()
        _agent_client = None


class RootcauseService:

    @staticmethod
    async def analyze(
        db: AsyncSession,
        *,
        batch_no: str,
        event_id: int | None = None,
        created_by: int | None = None,
    ) -> AgentTask:
        """
        触发根因分析：
        1. 校验批次存在
        2. 创建任务记录（pending -> running -> succeeded/failed）
        3. 通过 httpx 调用 Python 智能体（超时/重试/熔断）
        4. 把智能体生成的建议落库（第 9 章的 suggestions 表）
        """
        wafer = (
            await db.execute(select(Wafer).where(Wafer.batchNo == batch_no))
        ).scalar_one_or_none()
        if wafer is None:
            raise BizError(ErrorCode.NOT_FOUND, f"批次 {batch_no} 不存在", 404)

        task = AgentTask(
            taskNo=f"RC-{int(time.time() * 1000)}-{random.randint(0, 999)}",
            batchNo=batch_no,
            eventId=event_id,
            status="pending",
            createdBy=created_by,
        )
        db.add(task)
        await db.commit()      # 先落 pending：哪怕后续崩溃，也留下"发起过"的痕迹
        await db.refresh(task)

        try:
            task.status = "running"
            task.startedAt = datetime.now(timezone.utc)
            await db.commit()

            agent_result = await RootcauseService._call_agent(
                batch_no=batch_no,
                yield_rate=float(wafer.yieldRate),
            )

            task.status = "succeeded"
            task.result = agent_result
            task.finishedAt = datetime.now(timezone.utc)
            await db.commit()

            # 智能体输出的建议落库（第 9 章的表，先建表再放开这段注释）
            # if isinstance(agent_result.get("suggestions"), list) and agent_result["suggestions"]:
            #     db.add_all([
            #         Suggestion(
            #             taskId=task.id,
            #             batchNo=batch_no,
            #             type=s["type"],
            #             content=s["content"],
            #             riskLevel=s["risk_level"],
            #             status="pending",
            #             createdBy=created_by,
            #         )
            #         for s in agent_result["suggestions"]
            #     ])
            #     await db.commit()

            await db.refresh(task)
            return task
        except Exception as e:
            # 失败也要留痕：status=failed + 截断到 1000 字的 error
            task.status = "failed"
            task.error = str(getattr(e, "message", None) or e)[:1000]
            task.finishedAt = datetime.now(timezone.utc)
            await db.commit()
            raise

    @staticmethod
    async def task_detail(db: AsyncSession, task_id: int) -> AgentTask:
        """任务详情（轮询用）"""
        task = await db.get(AgentTask, task_id)
        if task is None:
            raise BizError(ErrorCode.NOT_FOUND, f"任务不存在（id={task_id}）", 404)
        return task

    @staticmethod
    async def task_list(
        db: AsyncSession,
        *,
        page: int,
        page_size: int,
        batch_no: str | None = None,
        status: str | None = None,
    ) -> tuple[list[AgentTask], int]:
        """任务列表"""
        conditions = []
        if batch_no:
            conditions.append(AgentTask.batchNo == batch_no)
        if status:
            conditions.append(AgentTask.status == status)

        rows = (
            await db.execute(
                select(AgentTask)
                .where(*conditions)
                .order_by(AgentTask.id.desc())
                .offset((page - 1) * page_size)
                .limit(page_size)
            )
        ).scalars().all()
        total = (
            await db.execute(select(func.count()).select_from(AgentTask).where(*conditions))
        ).scalar_one()
        return list(rows), total

    @staticmethod
    async def _call_agent(*, batch_no: str, yield_rate: float) -> dict:
        """
        调用 Python 智能体服务（httpx + 熔断 + 重试）
        - 超时：连接 3s / 响应 20s（Settings 的 rootcause_* 配置）
        - 重试：最多 2 次，线性退避
        - 熔断：连续 3 次任务失败跳闸 30s，期间快速失败
        """
        settings = get_settings()
        breaker = get_circuit_breaker(
            "python-agent",
            BreakerOptions(
                failure_threshold=settings.breaker_failure_threshold,
                open_duration_ms=settings.breaker_open_seconds * 1000,
            ),
        )
        client = get_agent_client()

        async def attempt(n: int) -> dict:
            logger.info("[rootcause] 调用智能体（第 %s 次）batchNo=%s", n + 1, batch_no)
            try:
                res = await client.post(
                    "/agent/analyze",
                    json={"batch_no": batch_no, "yield_rate": yield_rate, "top_k": 3},
                )
            except httpx.TimeoutException as e:
                # 超时 = 服务不可用（30001/503），交给熔断器记账
                raise BizError(ErrorCode.AGENT_UNAVAILABLE, f"智能体服务响应超时（{type(e).__name__}）", 503) from e
            except httpx.TransportError as e:
                # 连不上/连接被断 = 服务不可用（30001/503）
                raise BizError(ErrorCode.AGENT_UNAVAILABLE, f"无法连接智能体服务：{e}", 503) from e
            if res.status_code != 200:
                raise BizError(ErrorCode.AGENT_BAD_RESPONSE, f"智能体服务返回 HTTP {res.status_code}", 502)
            data = res.json()
            if not isinstance(data, dict) or not isinstance(data.get("root_causes"), list):
                raise BizError(ErrorCode.AGENT_BAD_RESPONSE, "智能体返回格式异常（缺少 root_causes）", 502)
            return data

        return await breaker.execute(
            attempt,
            retries=settings.rootcause_retries,
            retry_delay_ms=500,
        )
```

## 8.7 Router：app/routers/anomaly.py 与 rootcause.py（完整内容）

**做什么/为什么**：路由函数只做三件事——取参、调 service、统一响应。请求/响应模型（对标原 Controller 里的 egg-validate 规则）先给出，再给两个路由文件。

`app/schemas/anomaly_event.py`（完整内容）：

```python
from datetime import datetime

from pydantic import BaseModel, ConfigDict


class AnomalyEventOut(BaseModel):
    """异常事件响应模型（ORM -> JSON）"""
    model_config = ConfigDict(from_attributes=True)

    id: int
    eventCode: str
    batchNo: str | None
    equipment: str
    processStage: str
    severity: str
    title: str
    description: str | None
    status: str
    occurredAt: datetime
    createdAt: datetime
    updatedAt: datetime
```

`app/schemas/agent_task.py`（完整内容）：

```python
from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class RootcauseAnalyzeIn(BaseModel):
    """触发根因分析（对标原 analyzeRule：batchNo 6~32 位，eventId 可选）"""
    batchNo: str = Field(min_length=6, max_length=32)
    eventId: int | None = Field(default=None, ge=1)


class AgentTaskOut(BaseModel):
    """分析任务响应模型：result 是智能体返回的原始 JSON（dict 原样透传）"""
    model_config = ConfigDict(from_attributes=True)

    id: int
    taskNo: str
    batchNo: str
    eventId: int | None
    status: str
    result: dict | None
    error: str | None
    createdBy: int | None
    startedAt: datetime | None
    finishedAt: datetime | None
    createdAt: datetime
    updatedAt: datetime
```

`app/routers/anomaly.py`（完整内容）：

```python
from datetime import datetime
from typing import Literal

from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.core import response
from app.core.config import get_settings
from app.core.database import get_db
from app.core.deps import get_current_user
from app.schemas.anomaly_event import AnomalyEventOut
from app.services.anomaly import AnomalyService

router = APIRouter(prefix="/api/v1/anomaly-events", tags=["anomaly-events"])

SeverityQuery = Literal["critical", "major", "minor"]
EventStatusQuery = Literal["open", "processing", "closed"]


def _dump(event) -> dict:
    # ORM -> dict：统一走 AnomalyEventOut，序列化只写一次
    return AnomalyEventOut.model_validate(event).model_dump(mode="json")


@router.get("", dependencies=[Depends(get_current_user)])
async def list_anomaly_events(
    page: int | None = Query(default=None, ge=1, description="页码"),
    pageSize: int | None = Query(default=None, ge=1, le=500, description="每页条数"),
    severity: SeverityQuery | None = Query(default=None, description="严重度"),
    status: EventStatusQuery | None = Query(default=None, description="处理状态"),
    equipment: str | None = Query(default=None, description="设备模糊匹配"),
    batchNo: str | None = Query(default=None, description="批次号模糊匹配"),
    startTime: datetime | None = Query(default=None, description="发生时间起（ISO 8601）"),
    endTime: datetime | None = Query(default=None, description="发生时间止（ISO 8601）"),
    db: AsyncSession = Depends(get_db),
):
    """GET /api/v1/anomaly-events（需登录）"""
    settings = get_settings()
    page_num = page or settings.pagination_default_page
    page_size = min(pageSize or settings.pagination_default_page_size, settings.pagination_max_page_size)

    rows, total = await AnomalyService.list(
        db,
        page=page_num,
        page_size=page_size,
        severity=severity,
        status=status,
        equipment=equipment or None,
        batch_no=batchNo or None,
        start_time=startTime,
        end_time=endTime,
    )
    return response.page([_dump(r) for r in rows], total, page_num, page_size)


@router.get("/{event_id}", dependencies=[Depends(get_current_user)])
async def get_anomaly_event(event_id: int, db: AsyncSession = Depends(get_db)):
    """GET /api/v1/anomaly-events/:id（需登录）"""
    return response.ok(_dump(await AnomalyService.detail(db, event_id)))
```

`app/routers/rootcause.py`（完整内容）：

```python
from typing import Literal

from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.core import response
from app.core.config import get_settings
from app.core.database import get_db
from app.core.deps import CurrentUser, get_current_user, require_perm
from app.schemas.agent_task import AgentTaskOut, RootcauseAnalyzeIn
from app.services.rootcause import RootcauseService

router = APIRouter(prefix="/api/v1/rootcause", tags=["rootcause"])

TaskStatusQuery = Literal["pending", "running", "succeeded", "failed"]


def _dump(task) -> dict:
    return AgentTaskOut.model_validate(task).model_dump(mode="json")


@router.post("/analyze", status_code=202)
async def analyze(
    payload: RootcauseAnalyzeIn,
    user: CurrentUser = Depends(require_perm("rootcause:trigger")),
    db: AsyncSession = Depends(get_db),
):
    """POST /api/v1/rootcause/analyze（需 rootcause:trigger 权限）

    202 Accepted：分析任务已受理并完成调度。
    """
    task = await RootcauseService.analyze(
        db,
        batch_no=payload.batchNo,
        event_id=payload.eventId,
        created_by=user.userId,
    )
    return response.ok(_dump(task), "根因分析任务已完成")


@router.get("/tasks", dependencies=[Depends(get_current_user)])
async def list_tasks(
    page: int | None = Query(default=None, ge=1, description="页码"),
    pageSize: int | None = Query(default=None, ge=1, le=500, description="每页条数"),
    batchNo: str | None = Query(default=None, description="批次号精确匹配"),
    status: TaskStatusQuery | None = Query(default=None, description="任务状态"),
    db: AsyncSession = Depends(get_db),
):
    """GET /api/v1/rootcause/tasks（需登录）"""
    settings = get_settings()
    page_num = page or settings.pagination_default_page
    page_size = min(pageSize or settings.pagination_default_page_size, settings.pagination_max_page_size)

    rows, total = await RootcauseService.task_list(
        db,
        page=page_num,
        page_size=page_size,
        batch_no=batchNo or None,
        status=status,
    )
    return response.page([_dump(t) for t in rows], total, page_num, page_size)


@router.get("/tasks/{task_id}", dependencies=[Depends(get_current_user)])
async def get_task(task_id: int, db: AsyncSession = Depends(get_db)):
    """GET /api/v1/rootcause/tasks/:id（轮询任务状态，需登录）"""
    return response.ok(_dump(await RootcauseService.task_detail(db, task_id)))
```

## 8.8 种子数据追加异常事件 & 路由

`scripts/init_db.py` 的 `main()` 中，在"用户与角色"种子数据之后追加：

```python
    # ---------- 种子数据：异常事件 ----------
    from app.models.anomaly_event import AnomalyEvent

    async with AsyncSessionLocal() as db:
        db.add_all([
            AnomalyEvent(eventCode="AE-20261015-001", batchNo="B20261015001", equipment="ETCH-03",
                         processStage="etch", severity="critical", title="刻蚀功率漂移超标",
                         description="射频功率较基准漂移 +3.2%，持续 42 分钟", status="open", occurredAt=hours_ago(28)),
            AnomalyEvent(eventCode="AE-20261015-002", batchNo="B20261015003", equipment="CVD-11",
                         processStage="deposition", severity="major", title="腔体压力波动",
                         description="压力方差增大 1.8 倍", status="processing", occurredAt=hours_ago(6)),
            AnomalyEvent(eventCode="AE-20261014-007", batchNo="B20261015003", equipment="LITH-02",
                         processStage="litho", severity="minor", title="光刻胶接近有效期",
                         description="显影对比度轻微下降", status="closed", occurredAt=hours_ago(30)),
        ])
        await db.commit()
    print("[seed] 异常事件 3 条写入完成")
```

`app/main.py` 本章改三处（其余与第 7 章相同）：

```python
# ① 文件顶部：routers 加入 anomaly、rootcause；引入 httpx 客户端关闭函数
from app.routers import anomaly, auth, home, rootcause, wafer  # ★ 第 8 章新增 anomaly、rootcause
from app.services.rootcause import close_agent_client            # ★ 第 8 章新增
```

```python
# ② lifespan 关闭阶段：释放 httpx 连接池（在 Redis / engine 之前）
    yield
    await close_agent_client()   # ★
    await redis_client.close()
    await engine.dispose()
    logger.info("[shutdown] 数据库连接池与 Redis 已释放")
```

```python
# ③ create_app()：挂载路由（auth 之后）
    app.include_router(anomaly.router)    # ★
    app.include_router(rootcause.router)  # ★
```

重新执行 `python scripts/init_db.py`（会清库重建）。预期输出：

```text
[seed] 同步表结构（drop+create 会清空重建）...
[seed] 表结构同步完成
[seed] 晶圆批次 5 条写入完成
[seed] 用户与角色写入完成：admin/engineer/reviewer
[seed] 异常事件 3 条写入完成
[seed] 全部完成 ✔
```

## 8.9 验证

先 `python scripts/init_db.py` 重建数据，重启 uvicorn（`python -m uvicorn app.main:app --host 127.0.0.1 --port 7001 --reload --no-access-log`），确保 Python Mock 在跑（8300 端口），然后（engineer 拥有 `rootcause:trigger` 权限，先登录拿 TOKEN）：

```bash
TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"engineer","password":"Engineer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)
echo "engineer token: ${TOKEN:0:30}..."
```

**① 查询异常事件（按严重度过滤）**

```bash
curl -s 'http://127.0.0.1:7001/api/v1/anomaly-events?severity=critical' -H "Authorization: Bearer $TOKEN"
# 预期：total 为 1，rows[0].equipment = "ETCH-03"
```

**② 触发根因分析（engineer 有 rootcause:trigger 权限）**

```bash
curl -s -X POST http://127.0.0.1:7001/api/v1/rootcause/analyze \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"batchNo":"B20261015001","eventId":1}'
```

预期输出（约 2 秒后，HTTP 202）：

```json
{
  "code": 0, "message": "根因分析任务已完成",
  "data": {
    "id": 1, "taskNo": "RC-1760...", "batchNo": "B20261015001",
    "status": "succeeded",
    "result": {
      "summary": "批次 B20261015001 良率异常，疑似刻蚀工序参数漂移...",
      "root_causes": [ { "rank": 1, "factor": "etch_power_drift", "confidence": 0.87, "...": "..." }, "..." ]
    },
    "...": "..."
  },
  "traceId": "..."
}
```

**③ 轮询任务状态**

```bash
curl -s http://127.0.0.1:7001/api/v1/rootcause/tasks/1 -H "Authorization: Bearer $TOKEN"
# 预期：data.status = "succeeded"（前端可 2s 间隔轮询，直到非 pending/running）
```

**④ 熔断验证（关闭 Python 服务再连打 3 次）**

```bash
# 在 uvicorn 终端 Ctrl+C 停掉 Python 服务（agent-mock，8300），然后连续触发 3 次分析
for i in 1 2 3; do
  curl -s -X POST http://127.0.0.1:7001/api/v1/rootcause/analyze \
    -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
    -d '{"batchNo":"B20261015002"}' | head -c 120; echo; done
# 预期：三次都失败（code 30001/30002），第 4 次开始秒回：
# {"code":30001,"message":"依赖服务 python-agent 熔断中（xxs 后重试）",...}
# 重新启动 uvicorn，30 秒冷却期后自动恢复（half-open 探测成功 -> closed）
```

> **生产提示**：本实现中 `analyze` 是同步等待智能体返回的（约 2 秒，可接受）。若单次分析耗时分钟级，应改为"立即返回 taskId + MQ 异步执行 + 前端轮询"——任务表结构已为此设计好（status/startedAt/finishedAt）。

## 8.10 常见坑

1. **httpx 客户端必须复用**：每次调用都 `async with httpx.AsyncClient()` 会反复建/拆 TCP 连接，高峰期端口耗尽。本项目的共享客户端在 `get_agent_client()` 懒创建、`close_agent_client()` 在 lifespan 释放——忘了关闭，进程退出时连接自然消亡，但热重载（--reload）下会积累，所以还是老老实实关。
2. **"见到 30001 就免重试"会让熔断器失灵**：`_call_agent` 把网络异常翻译成 `BizError(30001)` 后，如果熔断器对一切 30001 都直接抛出（原 TS 版的行为），这些错误既不重试也不计数——熔断器永远不跳闸。所以 Python 版只对"熔断器自己抛的" `_BreakerOpenError` 免重试（见 8.4 的注释）。
3. **重试会放大最坏耗时**：retries=2 意味着一次 analyze 最多 3 次 HTTP、每次读超时 20s——最坏要 60 秒+ 才返回 503。前端/网关的超时要按这个上限设置，否则上游先超时，用户看到的是另一种错误。
4. **熔断状态在进程内存里**：`--workers 4` 多进程或多副本部署时，每个进程各有一份熔断器（各自数各自的失败）。教学规模没问题；生产要么把失败计数挪到 Redis，要么用服务网格级的熔断。
5. **JSONB 读写不对称**：`result` 列读出来已经是 `dict`（SQLAlchemy 自动反序列化），直接透传即可，别再 `json.loads` 一次；反之往 JSONB 写的对象必须可 JSON 序列化（`Decimal` 不行，见第 11 章日报任务的转换）。
6. **查询参数里的 naive datetime**：`startTime=2026-10-15T00:00:00` 解析出来不带时区，asyncpg 给 timestamptz 列绑参数会拒绝。`AnomalyService._ensure_aware` 统一按 UTC 补时区；传 `2026-10-15T00:00:00Z` 这类带时区的字符串则原样使用。

---

# 9. 建议与审批模块

## 9.1 做什么 / 为什么

手册的组织约束："关键建议必须人工审批，记录审批人、时间、理由"。这是"人机协同"闭环的最后一环：智能体产出建议（suggestions）→ 审批员批准/驳回（approvals）→ 全程留痕可审计。

## 9.2 数据模型（完整内容）

`app/models/suggestion.py`（完整内容）：

```python
from datetime import datetime, timezone

from sqlalchemy import CheckConstraint, DateTime, Integer, String, Text
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base, text_now


def _utcnow() -> datetime:
    return datetime.now(timezone.utc)


class Suggestion(Base):
    """
    建议表 suggestions
    由根因分析任务的智能体输出自动生成，等待人工审批
    """
    __tablename__ = "suggestions"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True, comment="主键")
    taskId: Mapped[int | None] = mapped_column(Integer, nullable=True, comment="来源分析任务")
    batchNo: Mapped[str | None] = mapped_column(String(32), nullable=True, comment="批次号")
    type: Mapped[str] = mapped_column(
        String(32), nullable=False, default="other", server_default="'other'",
        comment="建议类型：调参/隔离批次/停线/加检/其他",
    )
    content: Mapped[str] = mapped_column(Text, nullable=False, comment="建议内容")
    riskLevel: Mapped[str] = mapped_column(
        String(16), nullable=False, default="low", server_default="'low'", comment="风险等级",
    )
    status: Mapped[str] = mapped_column(
        String(16), nullable=False, default="pending", server_default="'pending'", comment="审批状态",
    )
    createdBy: Mapped[int | None] = mapped_column(Integer, nullable=True, comment="生成该建议的任务触发人")
    createdAt: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=_utcnow, server_default=text_now(), comment="创建时间",
    )
    updatedAt: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=_utcnow, onupdate=_utcnow,
        server_default=text_now(), comment="更新时间",
    )

    __table_args__ = (
        CheckConstraint(
            "type IN ('param_adjust', 'hold_wafer', 'stop_line', 'inspection', 'other')",
            name="ck_suggestions_type",
        ),
        CheckConstraint("riskLevel IN ('low', 'medium', 'high')", name="ck_suggestions_risk_level"),
        CheckConstraint("status IN ('pending', 'approved', 'rejected')", name="ck_suggestions_status"),
    )
```

`app/models/approval.py`（完整内容）：

```python
from datetime import datetime, timezone

from sqlalchemy import CheckConstraint, DateTime, String
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base, text_now


def _utcnow() -> datetime:
    return datetime.now(timezone.utc)


class ApprovalRecord(Base):
    """
    审批记录表 approval_records：谁、何时、批/驳、理由（只插入、不可改）

    表名按 V2.0 契约冻结为 approval_records（V1.x 为 approvals），
    字段与对外 JSON 完全一致，仅表名变化。
    """
    __tablename__ = "approval_records"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True, comment="主键")
    suggestionId: Mapped[int] = mapped_column(nullable=False, comment="被审批的建议")
    action: Mapped[str] = mapped_column(nullable=False, comment="动作：approve / reject")
    reason: Mapped[str] = mapped_column(String(500), nullable=False, comment="审批理由（必填）")
    approverId: Mapped[int] = mapped_column(nullable=False, comment="审批人用户 id")
    approverName: Mapped[str] = mapped_column(String(64), nullable=False, comment="审批人用户名")
    approvedAt: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, comment="审批时间")
    createdAt: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=_utcnow, server_default=text_now(), comment="创建时间",
    )
    updatedAt: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=_utcnow, onupdate=_utcnow,
        server_default=text_now(), comment="更新时间",
    )

    __table_args__ = (
        CheckConstraint("action IN ('approve', 'reject')", name="ck_approval_records_action"),
    )
```

`app/models/__init__.py` 更新为（完整内容）：

```python
from app.models.agent_task import AgentTask
from app.models.anomaly_event import AnomalyEvent
from app.models.approval import ApprovalRecord
from app.models.audit_log import AuditLog
from app.models.base import Base
from app.models.role import Role
from app.models.suggestion import Suggestion
from app.models.user import User
from app.models.user_role import UserRole
from app.models.wafer import Wafer

__all__ = [
    "AgentTask", "AnomalyEvent", "ApprovalRecord", "AuditLog", "Base",
    "Role", "Suggestion", "User", "UserRole", "Wafer",
]
```

## 9.3 Service：app/services/suggestion.py（完整内容，含事务 + 行锁）

**做什么/为什么**：审批是典型的一致性敏感操作——"更新建议状态"与"写审批记录"必须同生共死（事务），且两个审批员同时点击时只能有一人成功（`SELECT ... FOR UPDATE` 行锁 + 状态检查）。async SQLAlchemy 的写法是 `async with db.begin()` 包住整个事务，查询语句加 `.with_for_update()`。

`app/services/suggestion.py`（完整内容）：

```python
from datetime import datetime, timezone

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.biz_error import BizError
from app.core.error_codes import ErrorCode
from app.models.approval import ApprovalRecord
from app.models.suggestion import Suggestion


class SuggestionService:

    @staticmethod
    async def list(
        db: AsyncSession,
        *,
        page: int,
        page_size: int,
        batch_no: str | None = None,
        status: str | None = None,
        risk_level: str | None = None,
    ) -> tuple[list[Suggestion], int]:
        """建议列表"""
        conditions = []
        if batch_no:
            conditions.append(Suggestion.batchNo.ilike(f"%{batch_no}%"))
        if status:
            conditions.append(Suggestion.status == status)
        if risk_level:
            conditions.append(Suggestion.riskLevel == risk_level)

        rows = (
            await db.execute(
                select(Suggestion)
                .where(*conditions)
                .order_by(Suggestion.id.desc())
                .offset((page - 1) * page_size)
                .limit(page_size)
            )
        ).scalars().all()
        total = (
            await db.execute(select(func.count()).select_from(Suggestion).where(*conditions))
        ).scalar_one()
        return list(rows), total

    @staticmethod
    async def submit_approval(
        db: AsyncSession,
        *,
        suggestion_id: int,
        action: str,          # 'approve' | 'reject'
        reason: str,
        approver_id: int,
        approver_name: str,
    ) -> tuple[Suggestion, ApprovalRecord]:
        """
        提交审批（approve / reject）
        事务 + 行锁：防止并发双审/漏审
        """
        async with db.begin():  # 事务：状态更新与审批记录同生共死
            # 行锁：并发审批时，后到者在此等待；先到者提交后，
            # 后到者读到的新状态会触发下面的冲突检查
            suggestion = (
                await db.execute(
                    select(Suggestion).where(Suggestion.id == suggestion_id).with_for_update()
                )
            ).scalar_one_or_none()
            if suggestion is None:
                raise BizError(ErrorCode.NOT_FOUND, f"建议不存在（id={suggestion_id}）", 404)
            if suggestion.status != "pending":
                raise BizError(
                    ErrorCode.STATE_CONFLICT,
                    f"该建议已{'批准' if suggestion.status == 'approved' else '驳回'}，不能重复审批",
                )

            # 1. 更新建议状态
            suggestion.status = "approved" if action == "approve" else "rejected"

            # 2. 写审批记录（与状态更新同事务）
            approval = ApprovalRecord(
                suggestionId=suggestion.id,
                action=action,
                reason=reason,
                approverId=approver_id,
                approverName=approver_name,
                approvedAt=datetime.now(timezone.utc),
            )
            db.add(approval)
            await db.flush()  # 拿到自增 id，仍在事务内

        # 事务已提交；对象属性可能已过期，重新加载后再交给路由序列化
        await db.refresh(suggestion)
        await db.refresh(approval)
        return suggestion, approval

    @staticmethod
    async def approval_history(db: AsyncSession, suggestion_id: int) -> list[ApprovalRecord]:
        """审批历史：某建议的全部审批流水"""
        suggestion = await db.get(Suggestion, suggestion_id)
        if suggestion is None:
            raise BizError(ErrorCode.NOT_FOUND, f"建议不存在（id={suggestion_id}）", 404)
        rows = (
            await db.execute(
                select(ApprovalRecord)
                .where(ApprovalRecord.suggestionId == suggestion_id)
                .order_by(ApprovalRecord.id.asc())
            )
        ).scalars().all()
        return list(rows)
```

## 9.4 Router：app/routers/suggestion.py（完整内容）

`app/schemas/suggestion.py`（完整内容）：

```python
from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class ApprovalIn(BaseModel):
    """提交审批（对标原 approvalRule：reason 必填 2~500 字）"""
    reason: str = Field(min_length=2, max_length=500)


class SuggestionOut(BaseModel):
    """建议响应模型"""
    model_config = ConfigDict(from_attributes=True)

    id: int
    taskId: int | None
    batchNo: str | None
    type: str
    content: str
    riskLevel: str
    status: str
    createdBy: int | None
    createdAt: datetime
    updatedAt: datetime


class ApprovalOut(BaseModel):
    """审批记录响应模型"""
    model_config = ConfigDict(from_attributes=True)

    id: int
    suggestionId: int
    action: str
    reason: str
    approverId: int
    approverName: str
    approvedAt: datetime
    createdAt: datetime
```

`app/routers/suggestion.py`（完整内容）：

```python
from typing import Literal

from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.core import response
from app.core.config import get_settings
from app.core.database import get_db
from app.core.deps import CurrentUser, get_current_user, require_perm
from app.schemas.suggestion import ApprovalIn, ApprovalOut, SuggestionOut
from app.services.suggestion import SuggestionService

router = APIRouter(prefix="/api/v1/suggestions", tags=["suggestions"])

SuggestionStatusQuery = Literal["pending", "approved", "rejected"]
RiskLevelQuery = Literal["low", "medium", "high"]


def _dump_suggestion(s) -> dict:
    return SuggestionOut.model_validate(s).model_dump(mode="json")


def _dump_approval(a) -> dict:
    return ApprovalOut.model_validate(a).model_dump(mode="json")


@router.get("", dependencies=[Depends(get_current_user)])
async def list_suggestions(
    page: int | None = Query(default=None, ge=1, description="页码"),
    pageSize: int | None = Query(default=None, ge=1, le=500, description="每页条数"),
    batchNo: str | None = Query(default=None, description="批次号模糊匹配"),
    status: SuggestionStatusQuery | None = Query(default=None, description="审批状态"),
    riskLevel: RiskLevelQuery | None = Query(default=None, description="风险等级"),
    db: AsyncSession = Depends(get_db),
):
    """GET /api/v1/suggestions（需登录）"""
    settings = get_settings()
    page_num = page or settings.pagination_default_page
    page_size = min(pageSize or settings.pagination_default_page_size, settings.pagination_max_page_size)

    rows, total = await SuggestionService.list(
        db,
        page=page_num,
        page_size=page_size,
        batch_no=batchNo or None,
        status=status,
        risk_level=riskLevel,
    )
    return response.page([_dump_suggestion(s) for s in rows], total, page_num, page_size)


@router.post("/{suggestion_id}/approve")
async def approve(
    suggestion_id: int,
    payload: ApprovalIn,
    user: CurrentUser = Depends(require_perm("suggestion:approve")),
    db: AsyncSession = Depends(get_db),
):
    """POST /api/v1/suggestions/:id/approve（需 suggestion:approve 权限）"""
    suggestion, approval = await SuggestionService.submit_approval(
        db,
        suggestion_id=suggestion_id,
        action="approve",
        reason=payload.reason,
        approver_id=user.userId,
        approver_name=user.username,
    )
    return response.ok(
        {"suggestion": _dump_suggestion(suggestion), "approval": _dump_approval(approval)},
        "已批准",
    )


@router.post("/{suggestion_id}/reject")
async def reject(
    suggestion_id: int,
    payload: ApprovalIn,
    user: CurrentUser = Depends(require_perm("suggestion:approve")),
    db: AsyncSession = Depends(get_db),
):
    """POST /api/v1/suggestions/:id/reject（需 suggestion:approve 权限）"""
    suggestion, approval = await SuggestionService.submit_approval(
        db,
        suggestion_id=suggestion_id,
        action="reject",
        reason=payload.reason,
        approver_id=user.userId,
        approver_name=user.username,
    )
    return response.ok(
        {"suggestion": _dump_suggestion(suggestion), "approval": _dump_approval(approval)},
        "已驳回",
    )


@router.get("/{suggestion_id}/approvals", dependencies=[Depends(get_current_user)])
async def approval_history(suggestion_id: int, db: AsyncSession = Depends(get_db)):
    """GET /api/v1/suggestions/:id/approvals（需登录）"""
    rows = await SuggestionService.approval_history(db, suggestion_id)
    return response.ok([_dump_approval(a) for a in rows])
```

## 9.5 路由与"建议落库"开关

`app/main.py` 本章改一处：

```python
# ① 文件顶部 routers 一行加入 suggestion；挂载处同样追加
from app.routers import anomaly, auth, home, rootcause, suggestion, wafer  # ★ 第 9 章新增 suggestion
...
    app.include_router(suggestion.router)  # ★
```

同时把 8.6 节 `app/services/rootcause.py` 里"先建表再放开注释"的那段建议落库代码确认为已放开（建议表本已建好）——放开后的片段：

```python
            # 智能体输出的建议落库（第 9 章的 suggestions 表）
            if isinstance(agent_result.get("suggestions"), list) and agent_result["suggestions"]:
                db.add_all([
                    Suggestion(
                        taskId=task.id,
                        batchNo=batch_no,
                        type=s["type"],
                        content=s["content"],
                        riskLevel=s["risk_level"],
                        status="pending",
                        createdBy=created_by,
                    )
                    for s in agent_result["suggestions"]
                ])
                await db.commit()
```

并放开文件顶部的模型 import：

```python
from app.models.suggestion import Suggestion  # 第 9 章起启用
```

## 9.6 业务闭环完整演示（curl 调用序列）

重启 uvicorn（确保 PG、Redis、Python Mock（8300）、server 四者都在跑；建议先 `python scripts/init_db.py` 清库，保证建议 id 从 1 开始）。**剧本**：工程师 engineer 触发分析 → 智能体产出建议 → 审批员 reviewer 审批。

```bash
# ① 工程师登录
ENGINEER_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"engineer","password":"Engineer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)
echo "engineer token: ${ENGINEER_TOKEN:0:30}..."

# ② 工程师触发根因分析（约 2 秒返回）
curl -s -X POST http://127.0.0.1:7001/api/v1/rootcause/analyze \
  -H "Authorization: Bearer $ENGINEER_TOKEN" -H 'Content-Type: application/json' \
  -d '{"batchNo":"B20261015003"}'
# 预期：status "succeeded"，result.root_causes 长度 3

# ③ 查看智能体生成的待审建议
curl -s 'http://127.0.0.1:7001/api/v1/suggestions?status=pending' \
  -H "Authorization: Bearer $ENGINEER_TOKEN"
# 预期：total 2（param_adjust + hold_wafer 两条）

# ④ 审批员登录
REVIEWER_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"reviewer","password":"Reviewer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

# ⑤ 审批员批准第 1 条建议（低风险调参）
curl -s -X POST http://127.0.0.1:7001/api/v1/suggestions/1/approve \
  -H "Authorization: Bearer $REVIEWER_TOKEN" -H 'Content-Type: application/json' \
  -d '{"reason":"同意回调功率，请安排夜班执行并复测"}'
# 预期：code 0，message "已批准"，data.suggestion.status = "approved"

# ⑥ 审批员驳回第 2 条建议
curl -s -X POST http://127.0.0.1:7001/api/v1/suggestions/2/reject \
  -H "Authorization: Bearer $REVIEWER_TOKEN" -H 'Content-Type: application/json' \
  -d '{"reason":"该批次已出货，隔离无意义"}'
# 预期：code 0，message "已驳回"

# ⑦ 重复审批 -> 业务冲突（HTTP 400）
curl -si -X POST http://127.0.0.1:7001/api/v1/suggestions/1/approve \
  -H "Authorization: Bearer $REVIEWER_TOKEN" -H 'Content-Type: application/json' \
  -d '{"reason":"再批一次试试"}' | head -n 1
# 预期：HTTP/1.1 400 Bad Request
# 响应体：{"code":10004,"message":"该建议已批准，不能重复审批",...}

# ⑧ 工程师尝试审批 -> 403（无 suggestion:approve 权限）
curl -si -X POST http://127.0.0.1:7001/api/v1/suggestions/3/approve \
  -H "Authorization: Bearer $ENGINEER_TOKEN" -H 'Content-Type: application/json' \
  -d '{"reason":"我自己批"}' | head -n 1
# 预期：HTTP/1.1 403 Forbidden

# ⑨ 查看审批历史（含审批人/时间/理由）
curl -s http://127.0.0.1:7001/api/v1/suggestions/1/approvals \
  -H "Authorization: Bearer $ENGINEER_TOKEN"
# 预期：
# "data": [{ "id": 1, "suggestionId": 1, "action": "approve",
#            "reason": "同意回调功率，请安排夜班执行并复测",
#            "approverId": 3, "approverName": "reviewer", "approvedAt": "2026-10-15T..." }]

# ⑩ 数据库层面验证审计闭环
docker exec -it ia-pg psql -U industry -d industryagent \
  -c "SELECT username, method, path, status FROM audit_logs WHERE path LIKE '%suggestions%' ORDER BY id;"
# 预期：reviewer 的两次审批、engineer 的一次 403 尝试，全部在案
```

**至此，"分析 → 建议 → 审批 → 审计"业务闭环全部打通。** 这就是手册说的"人机协同 + 责任链"在后端的落地形态。

## 9.7 常见坑

1. **`with_for_update()` 必须在事务内**：`SELECT ... FOR UPDATE` 的锁在事务结束时释放。如果查询跑在自动提交里（没有 `async with db.begin()`），语句一结束锁就没了——"后到者等待"的效果完全依赖事务边界。本节把"取锁 → 校验状态 → 更新 → 写记录"整段包在 `db.begin()` 里，缺一环都不成立。
2. **锁 + 状态机缺一不可**：只有行锁没有状态检查，两个审批员会先后都"成功"；只有状态检查没有行锁，两个并发请求可能都读到 `pending` 一起通过。先拿锁、再读最新状态、再改——顺序不能反。
3. **事务里抛 BizError 会自动回滚**：`async with db.begin()` 退出时若上层异常会 rollback——冲突的那次审批不留任何半成品（状态没改、记录没写），这正是我们要的语义。
4. **死锁预防**：单条审批一次只锁一行，安全；如果未来做"批量审批"，务必按 id 升序逐条加锁，否则两个并发批量请求以不同顺序锁行会互相等待。
5. **审批表只插不改**：`approval_records` 是审计凭据，任何"改审批理由"的需求都应新插一条补记，而不是 UPDATE 原记录——这正是它没有 update 路由的原因。
6. **commit 后对象属性过期**：async session 默认 `expire_on_commit`，事务提交后访问属性会触发懒加载，在异步上下文里可能报 `MissingGreenlet`。稳妥做法就是本节的 `await db.refresh(obj)` 后再交给响应序列化。

---

# 10. Redis 应用：缓存与幂等

## 10.1 做什么 / 为什么

Redis 在本平台承担三个职责（第三个已在第 7 章实现）：

1. **看板统计缓存**（cache-aside 模式）：看板统计是多表聚合查询，前端 5 秒自动刷新会打爆数据库；
2. **接口幂等**（Idempotency-Key）：前端双击提交 / 网络重试时，防止创建两份一样的资源；
3. refresh token 白名单（已实现）。

动手前先把第 7 章的"裸 Redis 客户端"升级成**连接池 + 依赖注入**形态——本章的两个功能都要高频碰 Redis，连接池避免高峰期反复建 TCP 连接。

`app/core/redis.py`（完整内容，本章版本）：

```python
"""全局 Redis 客户端：refresh 白名单 / 登出黑名单 / 缓存 / 幂等键共用。

第 7 章的"裸 Redis 实例"升级为"连接池"：
- ConnectionPool 进程内共享，连接复用，避免高峰期反复握手；
- get_redis() 提供依赖注入形态（路由里 redis: Redis = Depends(get_redis)，测试可整体替换）；
- get_redis_client() 是共享客户端访问器（第 14 章 /healthz 依赖检查用）。
"""
from collections.abc import AsyncIterator

from redis.asyncio import ConnectionPool, Redis

from app.core.config import get_settings

_settings = get_settings()

# decode_responses=True：读回来直接是 str（缓存 JSON / 比对占位符 '1' 都依赖这一点）
pool = ConnectionPool(
    host=_settings.redis_host,
    port=_settings.redis_port,
    password=_settings.redis_password or None,
    db=_settings.redis_db,
    decode_responses=True,
    max_connections=50,
)

# 全局共享客户端（第 7 章的 deps / auth service 继续用它）
redis_client: Redis = Redis(connection_pool=pool)


def get_redis_client() -> Redis:
    """共享客户端访问器（第 14 章 /healthz 用）"""
    return redis_client


async def get_redis() -> AsyncIterator[Redis]:
    """FastAPI 依赖注入形态。

    客户端归连接池管理，请求结束时**不做** aclose——
    池在 lifespan 关闭阶段统一释放（见 close_redis）。
    """
    yield redis_client


async def close_redis() -> None:
    """lifespan 关闭阶段调用：释放整个连接池（redis-py >= 5.0.1；旧版用 await pool.disconnect()）"""
    await pool.aclose()
```

> 第 7 章 `deps.py` / `auth.py` 里的 `from app.core.redis import redis_client` 无需改动——模块级 `redis_client` 仍然存在，只是底层换成了连接池。

配套改 `app/main.py` 的 lifespan（关闭阶段从"关客户端"换成"关连接池"）：

```python
# ① import 行：close_redis 替换原来的直接引用
from app.core.redis import close_redis, redis_client   # ★ 第 10 章调整
...
# ② lifespan 关闭阶段
    yield
    await close_agent_client()
    await close_redis()   # ★ 释放整个连接池（原先只关单个客户端连接）
    await engine.dispose()
```

## 10.2 看板统计 Service：app/services/stats.py（完整内容）

**做什么/为什么**：cache-aside（旁路缓存）是最常用的缓存模式——**读**：先查缓存，命中直接返回；未命中查库、回填缓存、设置 TTL。**写**：业务数据变化时**删除**缓存（而非更新），下次读取自然回填最新值。删除而非更新，避免了"并发写缓存与写库顺序不一致"的脏数据问题。

`app/services/stats.py`（完整内容）：

```python
import json
from datetime import datetime, timezone

from redis.asyncio import Redis
from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.config import get_settings
from app.core.redis import get_redis_client
from app.models.agent_task import AgentTask
from app.models.anomaly_event import AnomalyEvent
from app.models.suggestion import Suggestion
from app.models.wafer import Wafer

DASHBOARD_KEY = "ia:stats:dashboard"


class StatsService:

    @staticmethod
    async def dashboard(db: AsyncSession, redis: Redis) -> dict:
        """看板统计（cache-aside 读）"""
        settings = get_settings()
        ttl = settings.dashboard_cache_ttl_seconds

        # 1. 先查缓存
        cached = await redis.get(DASHBOARD_KEY)
        if cached:
            return {"source": "cache", "data": json.loads(cached)}

        # 2. 未命中：查库聚合
        avg_yield = (
            await db.execute(
                select(func.round(func.avg(Wafer.yieldRate), 2)).where(
                    Wafer.status.in_(["completed", "shipped"])
                )
            )
        ).scalar()
        production_lines = (
            await db.execute(select(func.count(func.distinct(Wafer.productionLine))))
        ).scalar()

        data = {
            "totalBatches": (
                await db.execute(select(func.count()).select_from(Wafer))
            ).scalar_one(),
            "avgYield": float(avg_yield) if avg_yield is not None else None,  # PG numeric -> Decimal，需转换
            "productionLines": production_lines or 0,
            "openAnomalyEvents": (
                await db.execute(
                    select(func.count()).select_from(AnomalyEvent).where(AnomalyEvent.status == "open")
                )
            ).scalar_one(),
            "pendingSuggestions": (
                await db.execute(
                    select(func.count()).select_from(Suggestion).where(Suggestion.status == "pending")
                )
            ).scalar_one(),
            "agentTasks": {
                "total": (await db.execute(select(func.count()).select_from(AgentTask))).scalar_one(),
                "succeeded": (
                    await db.execute(
                        select(func.count()).select_from(AgentTask).where(AgentTask.status == "succeeded")
                    )
                ).scalar_one(),
                "failed": (
                    await db.execute(
                        select(func.count()).select_from(AgentTask).where(AgentTask.status == "failed")
                    )
                ).scalar_one(),
            },
            "computedAt": datetime.now(timezone.utc).isoformat(),
        }

        # 3. 回填缓存（带 TTL，即使忘了失效最多脏 60 秒）
        await redis.set(DASHBOARD_KEY, json.dumps(data, ensure_ascii=False), ex=ttl)
        return {"source": "db", "data": data}

    @staticmethod
    async def invalidate_dashboard() -> None:
        """缓存失效：任何业务写操作后调用（删缓存，下次读取回填）"""
        await get_redis_client().delete(DASHBOARD_KEY)
```

`app/routers/stats.py`（完整内容）：

```python
from fastapi import APIRouter, Depends
from redis.asyncio import Redis
from sqlalchemy.ext.asyncio import AsyncSession

from app.core import response
from app.core.database import get_db
from app.core.deps import get_current_user
from app.core.redis import get_redis
from app.services.stats import StatsService

router = APIRouter(prefix="/api/v1/stats", tags=["stats"])


@router.get("/dashboard", dependencies=[Depends(get_current_user)])
async def dashboard(
    db: AsyncSession = Depends(get_db),
    redis: Redis = Depends(get_redis),
):
    """GET /api/v1/stats/dashboard（需登录；cache-aside 缓存）"""
    return response.ok(await StatsService.dashboard(db, redis))
```

在写路径上挂失效。`app/services/wafer.py` 的 `create/update/remove` 末尾各追加一行（文件顶部加 `from app.services.stats import StatsService`）：

```python
# create()：return 前追加
            await db.refresh(wafer)
            # 缓存失效：看板统计数据已过期
            await StatsService.invalidate_dashboard()
            return wafer
```

```python
# update()：return 前追加
        await db.commit()
        await db.refresh(wafer)
        # 缓存失效：看板统计数据已过期
        await StatsService.invalidate_dashboard()
        return wafer
```

```python
# remove()：await db.delete(wafer) 提交后、return 前追加
        await db.delete(wafer)
        await db.commit()
        # 缓存失效：看板统计数据已过期
        await StatsService.invalidate_dashboard()
        return {"id": wafer_id}
```

`app/services/suggestion.py` 的 `submit_approval` 事务提交后同样追加一行：

```python
        await db.refresh(suggestion)
        await db.refresh(approval)
        # 缓存失效：看板的 pendingSuggestions 已变化
        await StatsService.invalidate_dashboard()
        return suggestion, approval
```

`app/main.py` 挂载路由：

```python
from app.routers import anomaly, auth, home, rootcause, stats, suggestion, wafer  # ★ 第 10 章新增 stats
...
    app.include_router(stats.router)  # ★
```

**验证**（用第 7 章任一账号登录取得 TOKEN，下同）：

```bash
TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"engineer","password":"Engineer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

curl -s http://127.0.0.1:7001/api/v1/stats/dashboard -H "Authorization: Bearer $TOKEN"
# 第 1 次预期："source":"db"
curl -s http://127.0.0.1:7001/api/v1/stats/dashboard -H "Authorization: Bearer $TOKEN"
# 第 2 次预期："source":"cache"（60 秒内）

# 再创建一个批次，缓存应立即失效
curl -s -X POST http://127.0.0.1:7001/api/v1/wafers \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"batchNo":"B20261015099","productModel":"MCU-XS32","processNode":"28nm","productionLine":"LINE-1","quantity":25,"yieldRate":95.00}' > /dev/null

curl -s http://127.0.0.1:7001/api/v1/stats/dashboard -H "Authorization: Bearer $TOKEN"
# 预期："source":"db" 且 totalBatches 已 +1（写路径触发了失效）
```

## 10.3 接口幂等中间件：app/middleware/idempotency.py（完整内容）

**做什么/为什么**：网络抖动时前端/网关会重试 POST。约定：客户端为每个"创建类"请求生成唯一 `Idempotency-Key` 头。服务端用 Redis `SETNX` 抢占：

- 抢占成功 → 正常执行，成功后把响应缓存 24 小时；
- 抢占失败且已有缓存响应 → 直接重放缓存的响应（头 `Idempotent-Replay: true`）；
- 抢占失败且占位值还在处理中 → 409 提示稍后重试。

写法沿用第 6 章的**纯 ASGI 中间件**（原因同 6.6 常见坑 1）：幂等中间件不读请求体，但要**截获响应体**缓存下来——纯 ASGI 包一层 `send` 最干净。

`app/middleware/idempotency.py`（完整内容）：

```python
from starlette.datastructures import Headers
from starlette.responses import JSONResponse, Response
from starlette.types import ASGIApp, Message, Receive, Scope, Send

from app.core.config import get_settings
from app.core.error_codes import ErrorCode
from app.core.redis import get_redis_client
from app.core.response import error_body


class IdempotencyMiddleware:
    """
    接口幂等中间件（对 POST 生效，未携带 Idempotency-Key 的请求直接放行）。
    - 占位成功：执行业务；成功（<400）后把"原始响应字节"缓存，供后续重放；
    - 占位失败且已有缓存：200 + Idempotent-Replay: true 重放（响应体与首次完全一致）；
    - 占位失败且仍在处理中：409。
    """

    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] != "http" or scope.get("method") != "POST":
            await self.app(scope, receive, send)
            return

        idem_key = Headers(scope=scope).get("Idempotency-Key")
        if not idem_key:
            await self.app(scope, receive, send)
            return  # 不强制：未带 key 的请求不享受幂等保护

        settings = get_settings()
        ttl = settings.idempotency_ttl_seconds
        # key 绑定"路径"，避免同一 key 被用在别的接口上
        redis_key = f"ia:idem:{scope['path']}:{idem_key}"
        redis = get_redis_client()

        locked = await redis.setnx(redis_key, "1")
        if locked:
            # 首次见到这个 key：占位并设置兜底 TTL（防止请求崩溃后占位永不释放）
            await redis.expire(redis_key, ttl)

            holder = {"status": 500}
            body_chunks: list[bytes] = []

            async def send_wrap(message: Message) -> None:
                if message["type"] == "http.response.start":
                    holder["status"] = message["status"]
                elif message["type"] == "http.response.body":
                    body_chunks.append(message.get("body", b""))
                await send(message)

            await self.app(scope, receive, send_wrap)
            # 业务成功：缓存"原始响应字节"——重放时逐字节一致（含 traceId），
            # 不做 parse -> dumps（序列化差异会破坏"与首次完全一致"）
            if holder["status"] < 400 and body_chunks:
                await redis.set(redis_key, b"".join(body_chunks), ex=ttl)
            return

        # 已存在：要么是处理中（值还是 '1'），要么已有缓存响应可重放
        cached = await redis.get(redis_key)
        if cached and cached != "1":
            replay = Response(content=cached, status_code=200, media_type="application/json")
            replay.headers["Idempotent-Replay"] = "true"
            await replay(scope, receive, send)
            return

        # 处理中：并发重复提交 -> 409。
        # 注意：这里不能 raise BizError——用户中间件位于全局异常处理器洋葱之外，
        # 抛出去会被最外层当 500 处理；直接构造 409 响应发回去（error_body 自带 traceId）
        conflict = JSONResponse(
            status_code=409,
            content=error_body(
                ErrorCode.DUPLICATE_REQUEST,
                "重复请求正在处理中，请稍后查询结果或更换 Idempotency-Key",
            ),
        )
        await conflict(scope, receive, send)
```

在 `app/main.py` 注册（对标原 Egg 的 `config.middleware = ['errorHandler','requestLog','auditLog','idempotency','metrics']`——幂等层最贴近业务；`metrics` 第 14 章再加）。`add_middleware` 后加者在最外层，所以幂等要**最先**注册：

```python
    # ---------------- 中间件（洋葱：TraceId -> RequestLog -> Audit -> Idempotency -> 业务） ----------------
    app.add_middleware(IdempotencyMiddleware)  # 第 4 层：最贴近业务，重放/拦截发生在路由之前
    app.add_middleware(AuditLogMiddleware)     # 第 3 层：贴近业务，记录写操作（重放请求也会被审计）
    app.add_middleware(RequestLogMiddleware)   # 第 2 层：结构化请求日志
    app.add_middleware(TraceIdMiddleware)      # 第 1 层：最外层，最先设置 traceId
```

（import 行相应加上 `from app.middleware.idempotency import IdempotencyMiddleware`。）

**验证**：

```bash
# 用同一个 Idempotency-Key 连续创建两次
curl -si -X POST http://127.0.0.1:7001/api/v1/wafers \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: order-frontend-uuid-0001' \
  -d '{"batchNo":"B20261015100","productModel":"MCU-XS32","processNode":"28nm","productionLine":"LINE-1","quantity":25,"yieldRate":95.00}' \
  | grep -E 'HTTP|Idempotent'
# 第 1 次预期：HTTP/1.1 201 Created（无 Idempotent-Replay 头）

curl -si -X POST http://127.0.0.1:7001/api/v1/wafers \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: order-frontend-uuid-0001' \
  -d '{"batchNo":"B20261015100","productModel":"MCU-XS32","processNode":"28nm","productionLine":"LINE-1","quantity":25,"yieldRate":95.00}' \
  | grep -E 'HTTP|Idempotent'
# 第 2 次预期：HTTP/1.1 200 OK + Idempotent-Replay: true，响应体与第 1 次完全一致

# 数据库中只会有一个 B20261015100 批次
curl -s 'http://127.0.0.1:7001/api/v1/wafers?batchNo=15100' -H "Authorization: Bearer $TOKEN"
# 预期：total 为 1
```

## 10.4 常见坑

1. **Redis 忘了 `decode_responses=True`**：读回来是 `bytes`，`cached != "1"` 恒真、`json.loads` 还得先 decode——缓存永远"未命中"、幂等永远"处理中"。连接池参数里这一个开关救掉本章两个功能。
2. **`setnx` + `expire` 两步不原子**：进程在两步之间崩溃，占位键永不过期，这个 key 就"废"了。教程沿用原版两步写法（语义一致、便于讲解）；生产请用一条 `SET key value EX ttl NX`。
3. **重放响应必须逐字节一致**：缓存"原始响应字节"而不是 `json.loads` 再 `json.dumps`——键序、空格、`ensure_ascii` 的差异都会让 diff 工具告诉你"响应体变了"。前端/网关做幂等校验时是按字节比对哈希的。
4. **中间件里不能抛 `BizError`**：全局异常处理器在最内层（包着路由），用户中间件在它外面——这里抛出去会被最外层兜底当 500。需要主动回错误响应时，直接构造 `JSONResponse` 发送（见 409 分支）。
5. **cache-aside 删除而非更新**：写库成功后"更新缓存"，并发下会出现"旧值覆盖新值"；改为**删除**，让下次读取回填，最多短暂多查一次库。
6. **缓存值必须可 JSON 序列化**：`Decimal`、`datetime` 直接进 `json.dumps` 会炸——`avgYield` 转 `float`、`computedAt` 转 ISO 字符串，都在构造 `data` 时完成，回填/读取两侧就都不用再操心。

---

# 11. 定时任务：APScheduler

## 11.1 做什么 / 为什么

Egg 的 `app/schedule/` 目录专放定时任务（框架自动加载，不需要自己写循环）；FastAPI 生态里对应 **APScheduler 的 `AsyncIOScheduler`**——在 `lifespan` 里启动/关闭（同样不需要自己写 `while True + asyncio.sleep`）。平台需要两个后台任务：

1. **数据健康检查**：传感器数据若断流超过阈值（30 分钟），说明 FDC/采集链路故障——"没有数据"本身就是最严重的异常；
2. **良率日报**：每天早上 7 点统计前一日各产线良率，写入日报表。

先装依赖（第 3 章 requirements.txt 预告过）。`requirements.txt` 追加：

```text
# ---------- 定时任务（第 11 章启用） ----------
APScheduler>=3.10            # AsyncIOScheduler：对标 egg schedule（3.x 自带 pytz，Windows 免装 tzdata）
```

```bash
pip install -r requirements.txt
# 预期输出（末尾节选）：Successfully installed APScheduler-3.x.x ...
```

再建任务包：`app/tasks/`，`__init__.py` 先留空（一个 docstring 即可）：

```python
"""APScheduler 定时任务包（第 11 章）：数据健康检查 + 良率日报。"""
```

调度器装配放在 `app/main.py` 的 lifespan 里——任务文件先建好再回来改 main.py。

## 11.2 数据模型（完整内容）

`app/models/sensor_reading.py`（完整内容）：

```python
from datetime import datetime

from sqlalchemy import DateTime, Index, Numeric, String
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base


class SensorReading(Base):
    """传感器读数表 sensor_readings（演示用简化模型；生产用 TimescaleDB）

    timestamps=False：时序数据只认采集时间 recordedAt，
    不需要 createdAt/updatedAt 两列冗余时间戳。
    """
    __tablename__ = "sensor_readings"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True, comment="主键")
    equipmentId: Mapped[str] = mapped_column(String(64), nullable=False, comment="设备编号，如 ETCH-03")
    metric: Mapped[str] = mapped_column(String(64), nullable=False, comment="指标名，如 rf_power")
    value: Mapped[float] = mapped_column(Numeric(10, 3), nullable=False, comment="读数")
    recordedAt: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False, comment="采集时间")

    __table_args__ = (
        # 断流检查按时间扫表，给 recordedAt 建索引（对标原 indexes: [recordedAt]）
        Index("ix_sensor_readings_recorded_at", "recordedAt"),
    )
```

`app/models/daily_yield_report.py`（完整内容）：

```python
from datetime import date, datetime, timezone

from sqlalchemy import Date, DateTime, Integer, Numeric, text
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from app.models.base import Base, text_now


def _utcnow() -> datetime:
    return datetime.now(timezone.utc)


class DailyYieldReport(Base):
    """良率日报表 daily_yield_reports"""
    __tablename__ = "daily_yield_reports"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True, comment="主键")
    reportDate: Mapped[date] = mapped_column(Date, nullable=False, unique=True, comment="报告日期（YYYY-MM-DD）")
    avgYield: Mapped[float | None] = mapped_column(Numeric(5, 2), nullable=True, comment="当日完工批次平均良率")
    totalBatches: Mapped[int] = mapped_column(
        nullable=False, default=0, server_default=text("0"), comment="当日完工批次数",
    )
    detail: Mapped[list | None] = mapped_column(JSONB, nullable=True, comment="分产线明细")
    createdAt: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=_utcnow, server_default=text_now(), comment="创建时间",
    )
    updatedAt: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, default=_utcnow, onupdate=_utcnow,
        server_default=text_now(), comment="更新时间",
    )
```

> `Date` 类型：纯日期无时间，正好用于"报告日期"（对标原 `DATEONLY`）。

`app/models/__init__.py` 更新为（完整内容）：

```python
from app.models.agent_task import AgentTask
from app.models.anomaly_event import AnomalyEvent
from app.models.approval import ApprovalRecord
from app.models.audit_log import AuditLog
from app.models.base import Base
from app.models.daily_yield_report import DailyYieldReport
from app.models.role import Role
from app.models.sensor_reading import SensorReading
from app.models.suggestion import Suggestion
from app.models.user import User
from app.models.user_role import UserRole
from app.models.wafer import Wafer

__all__ = [
    "AgentTask", "AnomalyEvent", "ApprovalRecord", "AuditLog", "Base",
    "DailyYieldReport", "Role", "SensorReading", "Suggestion", "User", "UserRole", "Wafer",
]
```

## 11.3 数据健康检查任务：app/tasks/data_health_check.py（完整内容）

```python
"""数据健康检查：传感器数据断流告警（每 5 分钟）。"""
import json
import logging
from datetime import datetime, timezone

from sqlalchemy import text

from app.core.database import AsyncSessionLocal

logger = logging.getLogger("industryagent.task")


async def data_health_check() -> None:
    """
    为什么任务里自开 AsyncSessionLocal：定时任务不在请求上下文里，
    没有 get_db 依赖可注入；复用任何"别人的会话"都可能踩到已关闭的连接。
    开短会话、查完即关，是任务层访问数据库的标准姿势。

    说明：列名是 camelCase（recordedAt），原生 SQL 里必须加双引号——
    不加引号 PG 会把标识符折成小写而报 column not exist（同 5.10 常见坑 3）。
    """
    async with AsyncSessionLocal() as db:
        row = (
            await db.execute(text('SELECT MAX("recordedAt") AS latest FROM sensor_readings'))
        ).mappings().first()

    latest = row.get("latest") if row else None
    latest_ms = latest.timestamp() * 1000 if latest is not None else 0
    gap_minutes = int((datetime.now(timezone.utc).timestamp() * 1000 - latest_ms) // 60000)

    if gap_minutes > 30:
        # 结构化告警日志（生产可扩展为 webhook / 邮件 / 短信）
        logger.warning(json.dumps({
            "kind": "alert",
            "level": "warn",
            "type": "sensor_data_stall",
            "msg": "传感器数据断流，请检查 FDC 采集链路",
            "gapMinutes": gap_minutes,
            "lastRecordedAt": latest.isoformat() if latest is not None else None,
            "ts": datetime.now(timezone.utc).isoformat(),
        }, ensure_ascii=False))
    else:
        logger.info("[health-check] 传感器数据正常，最近数据 %s 分钟前", gap_minutes)
```

任务函数本身不声明周期——周期在装配处集中可见。`app/main.py` 的 lifespan 更新为（完整内容，本章最终版）：

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    await redis_client.ping()
    logger.info("[startup] industryagent-server 就绪（env=%s，redis=ok）", settings.env)

    # ---------------- 定时任务（第 11 章）：对标 egg schedule ----------------
    scheduler = AsyncIOScheduler(timezone="Asia/Shanghai")
    scheduler.add_job(
        data_health_check,                 # 数据健康检查：每 5 分钟
        IntervalTrigger(minutes=5),
        id="data_health_check",
        max_instances=1,                   # 上一轮没跑完就跳过本轮，避免慢任务堆积
        coalesce=True,                     # 错过的触发合并成一次，防刷屏
    )
    scheduler.add_job(
        yield_daily_report,                # 良率日报：每天 07:00
        CronTrigger(hour=7, minute=0),     # 对标 egg 六段式 cron '0 0 7 * * *'（秒 分 时 日 月 周）
        id="yield_daily_report",
        max_instances=1,
        coalesce=True,
    )
    scheduler.start()
    logger.info("[startup] 定时任务已启动：data_health_check(每 5 分钟) / yield_daily_report(每日 07:00)")

    yield

    scheduler.shutdown(wait=False)         # 不等在跑的任务，快速退出（要"跑完再关"改 wait=True）
    await close_agent_client()             # 第 8 章：httpx 连接池
    await close_redis()                    # 第 10 章：Redis 连接池
    await engine.dispose()
    logger.info("[shutdown] 定时任务、Redis 与数据库连接池已释放")
```

import 区相应补四行：

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
from apscheduler.triggers.interval import IntervalTrigger

from app.tasks.data_health_check import data_health_check
from app.tasks.yield_daily_report import yield_daily_report
```

> 原 Egg 版注释里的"type: 'worker' 表示整个集群只由一个执行方运行"——FastAPI 单进程天然满足；但 `uvicorn --workers 4` 或多副本部署时**每个进程都会跑一份调度器**，告警会重复。教学规模没问题，生产见 11.6 常见坑 4。

## 11.4 良率日报任务：app/tasks/yield_daily_report.py（完整内容）

```python
"""良率日报：每天 07:00 统计前一日完工批次良率。"""
import logging
from datetime import date, datetime, timedelta, timezone

from sqlalchemy import text
from sqlalchemy.dialects.postgresql import insert as pg_insert

from app.core.database import AsyncSessionLocal
from app.models.daily_yield_report import DailyYieldReport

logger = logging.getLogger("industryagent.task")

# 昨日 00:00 ~ 今日 00:00（数据库时区）。
# 列名 camelCase，原生 SQL 里加双引号（同 11.3 的说明）。
REPORT_SQL = text("""
    SELECT "productionLine" AS production_line,
           ROUND(AVG("yieldRate"), 2) AS avg_yield,
           COUNT(*) AS batches
      FROM wafers
     WHERE status IN ('completed', 'shipped')
       AND "producedAt" >= CURRENT_DATE - INTERVAL '1 day'
       AND "producedAt" <  CURRENT_DATE
     GROUP BY "productionLine"
     ORDER BY "productionLine"
""")


async def yield_daily_report() -> None:
    async with AsyncSessionLocal() as db:
        rows = (await db.execute(REPORT_SQL)).mappings().all()

    # PG numeric -> Decimal：进 JSONB 前统一转 str（asyncpg 的 JSON 编码不认 Decimal）
    detail = [
        {
            "production_line": r["production_line"],
            "avg_yield": str(r["avg_yield"]),
            "batches": int(r["batches"]),
        }
        for r in rows
    ]

    total_batches = sum(item["batches"] for item in detail)
    avg_yield = (
        round(sum(float(item["avg_yield"]) * item["batches"] for item in detail) / total_batches, 2)
        if total_batches > 0
        else None
    )

    report_date: date = (datetime.now(timezone.utc) - timedelta(hours=24)).date()

    # upsert：同一天重复执行会覆盖而非报错（对标 sequelize upsert）
    async with AsyncSessionLocal() as db:
        stmt = pg_insert(DailyYieldReport).values(
            reportDate=report_date,
            avgYield=avg_yield,
            totalBatches=total_batches,
            detail=detail,
        )
        stmt = stmt.on_conflict_do_update(
            index_elements=[DailyYieldReport.reportDate],
            set_={
                "avgYield": stmt.excluded.avgYield,
                "totalBatches": stmt.excluded.totalBatches,
                "detail": stmt.excluded.detail,
            },
        )
        await db.execute(stmt)
        await db.commit()

    logger.info(
        "[daily-report] 良率日报已生成：%s，平均良率 %s，批次 %s",
        report_date, "-" if avg_yield is None else f"{avg_yield:.2f}", total_batches,
    )
```

## 11.5 种子数据与验证

`scripts/init_db.py` 追加传感器数据（放在异常事件种子之后）：

```python
    # ---------- 种子数据：传感器读数 ----------
    import math

    from app.models.sensor_reading import SensorReading

    def minutes_ago(m: int) -> datetime:
        return now - timedelta(minutes=m)

    readings = [
        SensorReading(
            equipmentId="ETCH-03", metric="rf_power",
            value=1500 + math.sin(i) * 20, recordedAt=minutes_ago(i * 5),
        )
        for i in range(1, 11)
    ]
    async with AsyncSessionLocal() as db:
        db.add_all(readings)
        await db.commit()
    print("[seed] 传感器读数 10 条写入完成")
```

重新执行 `python scripts/init_db.py`。预期输出：

```text
[seed] 同步表结构（drop+create 会清空重建）...
[seed] 表结构同步完成
[seed] 晶圆批次 5 条写入完成
[seed] 用户与角色写入完成：admin/engineer/reviewer
[seed] 异常事件 3 条写入完成
[seed] 传感器读数 10 条写入完成
[seed] 全部完成 ✔
```

**验证（把检查周期临时调快）**：把 `app/main.py` 里 `data_health_check` 的触发器临时改为 `IntervalTrigger(seconds=10)`，并给这条 `add_job` 加参数 `next_run_time=datetime.now(timezone.utc)`（对标原 `immediate: true`：启动立即执行一次；`main.py` 顶部补 `from datetime import datetime, timezone`），重启 uvicorn。

- 种子数据最新读数是 5 分钟前 → 预期日志每 10 秒输出：`[health-check] 传感器数据正常，最近数据 5 分钟前`；
- 手动清空读数模拟断流：

```bash
docker exec -it ia-pg psql -U industry -d industryagent -c "TRUNCATE sensor_readings;"
```

- 预期 10 秒内出现告警日志：

```text
{"kind":"alert","level":"warn","type":"sensor_data_stall","msg":"传感器数据断流，请检查 FDC 采集链路","gapMinutes":xxx,...}
```

**验证良率日报**：临时把 `yield_daily_report` 的触发器改成 `IntervalTrigger(seconds=30)`（每 30 秒，对标原 cron `'*/30 * * * * *'`）再重启，然后查表：

```bash
docker exec -it ia-pg psql -U industry -d industryagent \
  -c "SELECT report_date, avg_yield, total_batches, detail FROM daily_yield_reports;"
```

预期输出（种子数据的 producedAt 在 24 小时内的批次会进入统计）：

```text
 report_date | avg_yield | total_batches |                      detail
-------------+-----------+---------------+--------------------------------------------------
 2026-10-14  |     92.10 |             2 | [{"production_line":"LINE-1","avg_yield":"91.20",...}]
```

验证完成后把两个任务的时间改回 `IntervalTrigger(minutes=5)` 与 `CronTrigger(hour=7, minute=0)`。

## 11.6 常见坑

1. **任务内必须自建 session**：定时任务不在请求上下文，`Depends(get_db)` 无从谈起；也绝不能把某个请求的会话"存下来"给任务用（响应一返回它就关了）。标准姿势就是任务函数里 `async with AsyncSessionLocal() as db:` 开短会话、用完即关（两个任务文件都是这么写的）。
2. **`AsyncIOScheduler.start()` 必须在事件循环里**：它把任务挂到**当前运行的**循环上。放在模块 import 期调用会 `RuntimeError: no running event loop`——所以装配放 lifespan（uvicorn 已进入循环）。也正因如此，测试里用 `ASGITransport` 不触发 lifespan，调度器天然不会跟着测试跑起来（第 12 章会用到这个特性）。
3. **时区要显式给**：`AsyncIOScheduler(timezone="Asia/Shanghai")`。APScheduler 3.x 用 pytz 解析，Windows 上直接可用；不指定则取系统本地时区，容器/服务器时区一变，"每日 07:00"就悄悄漂移。另外日报 SQL 里的 `CURRENT_DATE` 走的是**数据库**时区，与调度时区是两回事——教学环境可忽略，生产建议两侧统一 UTC。
4. **多进程/多副本会重复执行**：`uvicorn --workers 4` 起四个进程就有四个调度器，同一时刻跑四份健康检查、写四份日报（upsert 掩盖了问题但告警会重复四条）。生产解法：单实例跑调度（独立 cron 容器）、或用 Redis 分布式锁选主。
5. **`max_instances=1` 防叠加**：任务耗时超过触发间隔时，APScheduler 默认会告警并跳过，但显式声明意图更稳；配合 `coalesce=True` 把错过的触发合并，重启补跑时不会连环轰炸。
6. **interval 默认不立即执行**：首次触发要等满一个间隔。调试想"启动就跑一次"，用 `add_job(..., next_run_time=datetime.now(timezone.utc))`（11.5 的验证就靠它）。
7. **`Date` 列与 `CURRENT_DATE` 的口径**：`reportDate` 存 UTC 计算的昨日日期，`CURRENT_DATE` 按数据库时区算——两者在时区边界（UTC 16:00~24:00 之间）可能差一天。教学数据无碍；追求严谨时把 SQL 边界也改成显式 UTC 参数。
