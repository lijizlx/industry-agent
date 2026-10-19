# 18. 补全模块：8D 报告 / 通知 / 缺陷 / 看板统计 / 审计查询 / 用户管理

## 18.0 做什么 / 为什么

前文（第 1~16 章）已经实现了系统的主干：工程骨架、配置体系、wafer 模块模板、中间件、JWT+RBAC 鉴权、异常事件与根因分析、建议审批、Redis 缓存与幂等、定时任务、测试、MCP Server 与部署。**本章补全最后一批业务模块，补完后系统与《接口设计说明书（IA-SD-005）》完全对齐**，形成可交付的完整后端：

| 小节 | 模块 | 解决的问题 |
|---|---|---|
| 18.1 | 8D 报告 | 质量闭环的"归档"环节：审批通过后自动生成报告骨架，工程师填写 D1~D8，质量签发 |
| 18.2 | 通知 + WebSocket | 审批通过 / 异常发生时实时推送，前端不用轮询 |
| 18.3 | 缺陷 + MinIO | 缺陷图对象存储与预签名 URL（数据不出域的图片访问方案） |
| 18.4 | 看板统计 | 大屏四接口：总览 / 良率趋势 / 缺陷分类 / 设备 Top |
| 18.5 | 审计查询 | 管理员检索第 6 章审计中间件落库的操作记录 |
| 18.6 | 用户管理 | 管理员创建用户、启停、改绑角色、重置密码 |
| 18.7 | 收官 | 全量路由总表 + 与 IA-SD-005 的 34 个接口对照 + 一键冒烟脚本 |

本章延续全书纪律：**每个模块仍按「Model → Service → Router → 注册 → curl 验证」五步走**，错误统一走 BizError + ErrorCode（沿用第 5 章错误码段，不新开号段），响应用 `success()` / `page()` 统一助手（`app/core/response.py`，第 3 章定义），权限走 `Depends(get_current_user)` 与 `Depends(require_perm('...'))`（第 7 章定义）。

### 18.0.1 安装本章新依赖

```bash
pip install minio python-multipart email-validator -i https://pypi.tuna.tsinghua.edu.cn/simple
# 预期：Successfully installed minio-... python-multipart-... email-validator-...，无 error
```

> `minio`：MinIO 官方 Python SDK（S3 兼容协议）；`python-multipart`：FastAPI 解析 `multipart/form-data` 的前置依赖（18.3 的 `UploadFile` 缺了它启动即报错）；`email-validator`：支撑 pydantic 的 `EmailStr`（18.6 用户邮箱校验）。**WebSocket 无需新增包**——第 2 章装的 `uvicorn[standard]` 已内置 `websockets` 协议库。

### 18.0.2 .env 追加配置

`.env` 末尾追加（**此文件绝不提交 git**）：

```bash
# MinIO 对象存储（18.3）
IA_MINIO_ENDPOINT=127.0.0.1
IA_MINIO_PORT=9000
IA_MINIO_ACCESS_KEY=industry
IA_MINIO_SECRET_KEY=industry123

# WebSocket 推送端口（18.2）
IA_WS_PORT=7002
```

### 18.0.3 Settings 追加配置

`app/core/config.py` 的 `Settings` 类中追加字段（pydantic-settings 按**同名字段**绑定 .env 变量，无需额外配置）：

```python
class Settings(BaseSettings):
    # ...（第 2~16 章已有字段，此处省略）...

    # ---- 对象存储（18.3 缺陷图） ----
    IA_MINIO_ENDPOINT: str = '127.0.0.1'
    IA_MINIO_PORT: int = 9000
    IA_MINIO_ACCESS_KEY: str = 'industry'
    IA_MINIO_SECRET_KEY: str = 'industry123'
    IA_MINIO_BUCKET: str = 'defect-images'

    # ---- WebSocket 推送端口（18.2） ----
    IA_WS_PORT: int = 7002
```

> 原教程在这里还给 Egg 配过 `config.multipart`（stream 模式 + 10mb 上限 + 图片白名单）。FastAPI 没有全局 multipart 配置项——**白名单与大小上限改在 18.3.5 的 service 里显式实现**，语义与原配置一致（扩展名白名单 `.png/.jpg/.jpeg/.bmp/.webp`、单文件 10MB 上限）。

### 18.0.4 角色权限点扩充

`app/core/role_permissions.py` 中 `ROLE_PERMISSIONS` 更新为（给工艺工程师加报告编辑与缺陷上传，给审批员加报告签发；用户管理 / 审计查询等新权限点**不分配给任何具体角色**，只由 admin 的 `*` 覆盖——这就是"管理员专用"的实现方式）：

```python
ROLE_PERMISSIONS: dict[str, list[str]] = {
    'admin': ['*'],
    'process_engineer': [
        'wafer:create', 'wafer:update', 'rootcause:trigger',
        'report:edit',      # 18.1：编辑 8D 报告草稿
        'defect:upload',    # 18.3：上传缺陷图
    ],
    'reviewer': [
        'rootcause:trigger', 'suggestion:approve',
        'report:release',   # 18.1：签发 8D 报告
    ],
    'viewer': [],
}
```

> 本章新增权限点一览：`report:read`（登录即可查报告，路由上只挂登录依赖不挂 perm）、`report:edit`、`report:release`、`defect:upload`、`audit:read`、`user:read`、`user:create`、`user:update`、`user:reset_password`。除 `report:edit` / `report:release` / `defect:upload` 外均仅 admin（`*`）可用。

同时给角色表补上 `viewer` 角色：在 `scripts/init_db.py` 的角色种子里追加一行（与 admin/process_engineer/reviewer 并列）：

```python
    ('viewer', '访客（只读大屏）'),
```

重跑 `python scripts/init_db.py` 生效——18.6 节创建 viewer 用户时要用到这个角色码。

---

## 18.1 8D 报告模块

### 18.1.1 做什么 / 为什么

8D（8 Disciplines）是半导体行业质量问题的标准归档方法论：D1 成立小组 → D2 问题描述 → D3 临时围堵 → D4 根本原因 → D5 纠正措施 → D6 实施验证 → D7 预防再发 → D8 团队认可。平台在**建议审批通过后自动生成 8D 报告骨架**（D4 直接带入智能体的根因结论），质量工程师补全内容后由质量主管签发——这就是 IA-SD-005 第 11 节的 `/reports` 三个接口 + 我们补的编辑接口。

状态机：`draft`（草稿）→ `pending_review`（待复核）→ `released`（已签发，终态）。签发后不可再编辑（质量档案不可篡改）。

### 18.1.2 数据模型：app/models/report_8d.py（完整内容）

```python
from datetime import datetime

import sqlalchemy as sa
from sqlalchemy import func
from sqlalchemy.orm import Mapped, mapped_column

from app.core.database import Base


class Report8D(Base):
    """
    8D 报告表 reports_8d
    审批通过后由 ReportService.consume_approval 自动生成骨架（status=draft），
    质量工程师补全 D1~D8 后由质量主管签发（released 为终态）
    """
    __tablename__ = 'reports_8d'

    id: Mapped[int] = mapped_column(sa.Integer, primary_key=True, autoincrement=True)
    reportNo: Mapped[str] = mapped_column(sa.String(64), nullable=False, unique=True, comment='报告编号 8D-日期-序号')
    eventId: Mapped[int | None] = mapped_column(sa.Integer, comment='关联异常事件')
    taskId: Mapped[int | None] = mapped_column(sa.Integer, comment='关联根因分析任务')
    suggestionId: Mapped[int | None] = mapped_column(sa.Integer, comment='触发本报告的建议')
    title: Mapped[str] = mapped_column(sa.String(255), nullable=False, server_default='', comment='报告标题')

    # ---- D1~D8 八个 discipline，全部 TEXT（草稿期允许空串） ----
    d1Team: Mapped[str | None] = mapped_column(sa.Text, comment='D1 成立小组（JSON：成员/职责）')
    d2Problem: Mapped[str | None] = mapped_column(sa.Text, comment='D2 问题描述')
    d3Containment: Mapped[str | None] = mapped_column(sa.Text, comment='D3 临时围堵措施')
    d4Rootcause: Mapped[str | None] = mapped_column(sa.Text, comment='D4 根本原因（审批通过时自动带入）')
    d4EvidenceRefs: Mapped[str | None] = mapped_column(sa.Text, comment='D4 证据链引用（JSON 数组）')
    d5Corrective: Mapped[str | None] = mapped_column(sa.Text, comment='D5 纠正措施')
    d6Implementation: Mapped[str | None] = mapped_column(sa.Text, comment='D6 措施实施与验证')
    d7Preventive: Mapped[str | None] = mapped_column(sa.Text, comment='D7 预防再发')
    d8Recognition: Mapped[str | None] = mapped_column(sa.Text, comment='D8 团队认可')

    status: Mapped[str] = mapped_column(
        sa.Enum('draft', 'pending_review', 'released', name='report_status'),
        nullable=False, server_default='draft', comment='报告状态机',
    )

    createdBy: Mapped[int | None] = mapped_column(sa.Integer, comment='骨架生成时的建议发起人')
    reviewedBy: Mapped[int | None] = mapped_column(sa.Integer, comment='复核人')
    releasedBy: Mapped[int | None] = mapped_column(sa.Integer, comment='签发人')
    releasedAt: Mapped[datetime | None] = mapped_column(sa.DateTime(timezone=True), comment='签发时间')

    # 软删除：不物理删行，而是写 deletedAt（质量档案留痕）——查询处显式过滤 is_(None)
    deletedAt: Mapped[datetime | None] = mapped_column(sa.DateTime(timezone=True), comment='软删除时间')
    createdAt: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), server_default=func.now(), comment='创建时间')
    updatedAt: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), comment='更新时间')

    def as_dict(self, fields: list[str] | None = None) -> dict:
        """序列化：datetime 转 isoformat，保证响应体键名与前端契约一致"""
        keys = fields if fields else [c.name for c in self.__table__.columns]
        out: dict = {}
        for k in keys:
            v = getattr(self, k)
            out[k] = v.isoformat() if isinstance(v, datetime) else v
        return out


# 列表只带摘要字段，D1~D8 全文留给详情接口
LIST_FIELDS = [
    'id', 'reportNo', 'eventId', 'taskId', 'suggestionId', 'title',
    'status', 'createdBy', 'releasedBy', 'releasedAt', 'createdAt', 'updatedAt',
]
```

> **建表说明**：本教程主线走 `Base.metadata.create_all`（`scripts/init_db.py` 统一执行），重跑 `python scripts/init_db.py` 即会建出 `reports_8d` 表。生产环境的版本化迁移属《02-数据库设计与迁移教程》的 Alembic 主线，此处不展开。
>
> **软删除（paranoid）语义**：SQLAlchemy 没有内置软删开关，本教程用 `deletedAt` 列 + 查询处显式 `deletedAt IS NULL` 过滤实现同语义（质量档案永不物理删除）。

### 18.1.3 Service：app/services/report.py（完整内容）

**做什么/为什么**：三个核心业务——① `consume_approval`：审批通过后生成骨架（供第 9 章挂钩调用）；② `update`：草稿期编辑（released 后拒绝）；③ `release`：签发状态机校验 + 显式写一条业务级审计。

```python
import json
import logging
import time
from datetime import datetime, timezone

from sqlalchemy import func, or_, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.biz_error import BizError
from app.core.error_codes import ErrorCode
from app.models.agent_task import AgentTask
from app.models.anomaly_event import AnomalyEvent
from app.models.report_8d import LIST_FIELDS, Report8D
from app.services.audit import AuditService

logger = logging.getLogger(__name__)

# 8D 报告编辑接口允许提交的字段（白名单，防止越权改 createdBy/releasedBy 等内部字段）
EDITABLE_FIELDS = [
    'title', 'd1Team', 'd2Problem', 'd3Containment', 'd4Rootcause', 'd4EvidenceRefs',
    'd5Corrective', 'd6Implementation', 'd7Preventive', 'd8Recognition', 'status',
]


class ReportService:

    def __init__(self, db: AsyncSession):
        self.db = db

    async def list(self, param: dict) -> dict:
        """分页查询（支持状态过滤 + 标题/编号关键字模糊）"""
        stmt = select(Report8D).where(Report8D.deletedAt.is_(None))
        if param.get('status'):
            stmt = stmt.where(Report8D.status == param['status'])
        if param.get('keyword'):
            kw = f"%{param['keyword']}%"
            stmt = stmt.where(or_(Report8D.title.like(kw), Report8D.reportNo.like(kw)))

        total = await self.db.scalar(select(func.count()).select_from(stmt.subquery()))
        rows = (await self.db.execute(
            stmt.order_by(Report8D.id.desc())
            .offset((param['page'] - 1) * param['pageSize'])
            .limit(param['pageSize'])
        )).scalars().all()
        return {'rows': [r.as_dict(LIST_FIELDS) for r in rows], 'total': total or 0}

    async def detail(self, report_id: int) -> dict:
        """详情（含 D1~D8 全文）"""
        report = await self._get_or_404(report_id)
        plain = report.as_dict()
        # D4 证据引用是 JSON 字符串，详情里还原成结构（前端免解析）
        if plain.get('d4EvidenceRefs'):
            try:
                plain['d4EvidenceRefs'] = json.loads(plain['d4EvidenceRefs'])
            except (json.JSONDecodeError, TypeError):
                pass  # 非法 JSON 原样返回
        return plain

    async def consume_approval(self, suggestion, approval) -> Report8D:
        """
        由审批通过事件自动生成报告骨架（第 9 章审批通过后调用）
        - 同一任务只生成一份（幂等：taskId 已有报告则直接返回已有报告）
        - D4 直接带入智能体根因结论与证据链引用
        - 生成后给建议发起人推一条通知（18.2 实现后生效）
        """
        task_id = suggestion.taskId

        # 幂等：同一任务已生成过报告就不再生成
        stmt = select(Report8D).where(Report8D.deletedAt.is_(None))
        if task_id:
            stmt = stmt.where(Report8D.taskId == task_id)
        else:
            stmt = stmt.where(Report8D.suggestionId == suggestion.id)
        exists = (await self.db.execute(stmt)).scalar_one_or_none()
        if exists:
            return exists

        # 从任务结果中提取根因结论与证据链
        rootcause_text = ''
        evidence_refs: list[str] = []
        event = None
        if task_id:
            task = await self.db.get(AgentTask, task_id)
            if task and task.result:
                top_list = task.result.get('root_causes') or []
                top = top_list[0] if isinstance(top_list, list) and top_list else None
                if top:
                    rootcause_text = (
                        f"#{top['rank']} {top['factor']}"
                        f"（置信度 {top['confidence']}）：{top.get('description') or ''}"
                    )
                    evidence_refs = [
                        f"rootcause:{task.taskNo}#rank{top['rank']}",
                        'evidence:GRAPH', 'evidence:CAUSAL', 'evidence:RAG',
                    ]
                if task.eventId:
                    event = await self.db.get(AnomalyEvent, task.eventId)

        now = datetime.now(timezone.utc)
        report = Report8D(
            reportNo=f"8D-{now.strftime('%Y%m%d')}-{int(time.time() * 1000) % 100000}",
            eventId=event.id if event else None,
            taskId=task_id,
            suggestionId=suggestion.id,
            title=f"批次 {suggestion.batchNo} 建议审批通过 8D 报告（审批人：{approval.approverName}）",
            # D2 问题描述骨架：带入建议内容
            d2Problem=f"审批通过的建议：{suggestion.content}",
            d4Rootcause=rootcause_text,
            d4EvidenceRefs=json.dumps(evidence_refs, ensure_ascii=False),
            status='draft',
            createdBy=suggestion.createdBy,
        )
        self.db.add(report)
        await self.db.commit()
        await self.db.refresh(report)

        # 18.2 通知模块实现后，此处自动生效（通知失败不影响骨架生成）
        try:
            from app.services.notification import NotificationService
            await NotificationService(self.db).create_notification({
                'userId': suggestion.createdBy,
                'type': 'approval',
                'title': '建议已批准，8D 报告骨架已生成',
                'content': (f"建议「{str(suggestion.content)[:60]}」已由 {approval.approverName} 批准，"
                            f"请补全 8D 报告（编号 {report.reportNo}）后提交签发。"),
                'sourceId': report.id,
            })
        except Exception as e:  # noqa: BLE001
            logger.warning('[report] 通知发送失败: %s', e)

        return report

    async def update(self, report_id: int, data: dict, operator_id: int) -> dict:
        """编辑草稿（draft / pending_review 可编辑，released 拒绝）"""
        report = await self._get_or_404(report_id)
        if report.status == 'released':
            raise BizError(ErrorCode.STATE_CONFLICT, '报告已签发，质量档案不可再修改')

        # 只允许编辑白名单字段，且 status 只能在 draft/pending_review 之间流转
        patch = {k: v for k, v in data.items() if k in EDITABLE_FIELDS and v is not None}
        if patch.get('status') and patch['status'] not in ('draft', 'pending_review'):
            raise BizError(ErrorCode.PARAM_INVALID, '编辑时 status 只能为 draft 或 pending_review', http_status=422)
        if patch.get('status') == 'pending_review':
            patch['reviewedBy'] = operator_id

        for k, v in patch.items():
            setattr(report, k, v)
        await self.db.commit()
        await self.db.refresh(report)
        return report.as_dict()

    async def release(self, report_id: int, operator: dict, request=None) -> dict:
        """
        签发（状态机：draft / pending_review -> released）
        签发是质量档案定稿动作，除了 HTTP 层审计中间件自动记录外，
        这里再显式写一条业务级审计（含报告编号），并通知骨架发起人。
        """
        report = await self._get_or_404(report_id)
        if report.status == 'released':
            raise BizError(ErrorCode.STATE_CONFLICT, '报告已签发，不能重复签发')
        # 签发前最低完备性校验：D2 与 D4 必须已填写
        if not report.d2Problem or not report.d4Rootcause:
            raise BizError(ErrorCode.PARAM_INVALID, '签发前必须填写 D2 问题描述与 D4 根本原因', http_status=422)

        report.status = 'released'
        report.releasedBy = operator['userId']
        report.releasedAt = datetime.now(timezone.utc)
        await self.db.commit()
        await self.db.refresh(report)

        # 显式业务级审计（区别于中间件按 HTTP 维度的自动记录）
        await AuditService(self.db).record({
            'userId': operator['userId'],
            'username': operator['username'],
            'method': 'POST',
            'path': f'/api/v1/reports/{report_id}/release',
            'status': 200,
            'durationMs': 0,
            'ip': request.client.host if (request and request.client) else None,
            'requestSummary': json.dumps(
                {'reportNo': report.reportNo, 'action': 'release_8d_report'}, ensure_ascii=False),
            'errorMessage': None,
            'traceId': getattr(request.state, 'trace_id', '') if request else '',
        })

        try:
            from app.services.notification import NotificationService
            await NotificationService(self.db).create_notification({
                'userId': report.createdBy,
                'type': 'system',
                'title': '8D 报告已签发',
                'content': f"报告 {report.reportNo} 已由 {operator['username']} 签发。",
                'sourceId': report.id,
            })
        except Exception as e:  # noqa: BLE001
            logger.warning('[report] 通知发送失败: %s', e)

        return report.as_dict()

    async def _get_or_404(self, report_id: int) -> Report8D:
        report = await self.db.get(Report8D, report_id)
        if not report or report.deletedAt is not None:
            raise BizError(ErrorCode.NOT_FOUND, f'报告不存在（id={report_id}）', http_status=404)
        return report
```

### 18.1.4 请求 Schema：app/schemas/report.py（完整内容）

```python
from typing import Literal

from pydantic import BaseModel, Field

# 与原 Egg 版 updateRule 逐字段对齐：全部可选、允许空串、长度上限、status 枚举
class ReportUpdateIn(BaseModel):
    title: str | None = Field(None, max_length=255)
    d1Team: str | None = Field(None, max_length=20000)
    d2Problem: str | None = Field(None, max_length=20000)
    d3Containment: str | None = Field(None, max_length=20000)
    d4Rootcause: str | None = Field(None, max_length=20000)
    d4EvidenceRefs: str | None = Field(None, max_length=20000)
    d5Corrective: str | None = Field(None, max_length=20000)
    d6Implementation: str | None = Field(None, max_length=20000)
    d7Preventive: str | None = Field(None, max_length=20000)
    d8Recognition: str | None = Field(None, max_length=20000)
    status: Literal['draft', 'pending_review'] | None = None
```

### 18.1.5 Router：app/routers/report.py（完整内容）

```python
from fastapi import APIRouter, Depends, Query, Request
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.database import get_db
from app.core.deps import CurrentUser, require_perm
from app.core.response import page, success
from app.schemas.report import ReportUpdateIn
from app.services.report import ReportService

router = APIRouter()


@router.get('/api/v1/reports')
async def list_reports(
    page_no: int = Query(1, alias='page', ge=1),
    page_size: int = Query(10, alias='pageSize', ge=1, le=500),
    status: str | None = Query(None, pattern='^(draft|pending_review|released)$'),
    keyword: str | None = Query(None, max_length=64),
    db: AsyncSession = Depends(get_db),
    _: CurrentUser,
):
    """GET /api/v1/reports?page=1&pageSize=10&status=draft&keyword=（登录即可）"""
    result = await ReportService(db).list({
        'page': page_no, 'pageSize': page_size,
        'status': status, 'keyword': keyword,
    })
    return page(result['rows'], result['total'], page_no, page_size)


@router.get('/api/v1/reports/{report_id}')
async def get_report(
    report_id: int,
    db: AsyncSession = Depends(get_db),
    _: CurrentUser,
):
    """GET /api/v1/reports/:id（含 D1~D8 全文，登录即可）"""
    return success(await ReportService(db).detail(report_id))


@router.put('/api/v1/reports/{report_id}')
async def update_report(
    report_id: int,
    body: ReportUpdateIn,
    request: Request,
    db: AsyncSession = Depends(get_db),
    user: dict = Depends(require_perm('report:edit')),
):
    """PUT /api/v1/reports/:id（质量工程师编辑草稿）"""
    report = await ReportService(db).update(
        report_id, body.model_dump(exclude_unset=True), user['userId'])
    return success(report, '更新成功')


@router.post('/api/v1/reports/{report_id}/release')
async def release_report(
    report_id: int,
    request: Request,
    db: AsyncSession = Depends(get_db),
    user: dict = Depends(require_perm('report:release')),
):
    """POST /api/v1/reports/:id/release（签发）"""
    report = await ReportService(db).release(
        report_id, {'userId': user['userId'], 'username': user['username']}, request)
    return success(report, '报告已签发')
```

### 18.1.6 路由注册与审批挂钩

`app/main.py` 的 `create_app()` 中追加路由挂载（与既有 `include_router` 并列）：

```python
    from app.routers import report
    app.include_router(report.router)
```

**与前文挂钩（关键的一处修改）**：让第 9 章的审批通过自动触发报告骨架。在 `app/services/suggestion.py` 的 `submit_approval` 方法中，把最后的 `return await ...`（事务块）改为先接住事务结果、挂钩、再返回：

```python
    # 原：async with ... 事务体执行完直接 return {'suggestion': ..., 'approval': ...}
    result = await self._run_approval_tx(param)   # /* 原事务体一字不改，仅包成内部方法 */

    # ★ 第 18 章挂钩：审批通过后自动生成 8D 报告骨架。
    #   注意必须在事务提交之后调用（放事务内会拉长锁持有时间，且失败会连坐回滚审批本身）
    if param['action'] == 'approve':
        try:
            await ReportService(self.db).consume_approval(result['suggestion'], result['approval'])
        except Exception as e:  # noqa: BLE001
            logger.warning('[report] 报告骨架生成失败: %s', e)

    return result
```

> 挂钩失败只打警告日志、不影响审批结果——报告骨架是"锦上添花"动作，审批是"法定"动作，二者解耦。第 9 章若已用 `async with self.db.begin():` 的写法，保持事务体不动、把 `return` 挪到事务块外接住结果即可。

### 18.1.7 curl 验证

前置：重新 `python scripts/init_db.py`（此时建议/审批数据重建），重启 `python -m app.run`，跑一遍第 9.6 节的审批剧本（engineer 触发分析 → reviewer 批准建议 1）。批准动作会自动生成报告骨架。

**① 查看审批通过自动生成的报告（无过滤 + 状态过滤）**

```bash
ADMIN_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"Admin@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

curl -s 'http://127.0.0.1:7001/api/v1/reports?status=draft' -H "Authorization: Bearer $ADMIN_TOKEN"
```

预期输出（已格式化）：

```json
{
  "code": 0, "message": "ok",
  "data": {
    "rows": [
      { "id": 1, "reportNo": "8D-20261015-xxxxx", "taskId": 1, "suggestionId": 1,
        "title": "批次 B20261015003 建议审批通过 8D 报告（审批人：reviewer）",
        "status": "draft", "createdBy": 2, "releasedBy": null, "releasedAt": null,
        "createdAt": "2026-10-15T...", "updatedAt": "2026-10-15T..." }
    ],
    "total": 1, "page": 1, "pageSize": 10
  },
  "traceId": "..."
}
```

**② 报告详情（含 D4 自动带入的根因结论）**

```bash
curl -s http://127.0.0.1:7001/api/v1/reports/1 -H "Authorization: Bearer $ADMIN_TOKEN"
```

预期输出（节选）：

```json
{
  "code": 0, "message": "ok",
  "data": {
    "id": 1, "reportNo": "8D-20261015-xxxxx", "status": "draft",
    "d2Problem": "审批通过的建议：将 ETCH-03 射频功率回调至基准值 -2%",
    "d4Rootcause": "#1 etch_power_drift（置信度 0.87）：ETCH-03 刻蚀功率较基准漂移 +3.2%，时间窗与缺陷分布吻合",
    "d4EvidenceRefs": [ "rootcause:RC-...#rank1", "evidence:GRAPH", "evidence:CAUSAL", "evidence:RAG" ],
    "d1Team": null, "d3Containment": null, "...": "..."
  },
  "traceId": "..."
}
```

**③ 工程师编辑草稿（补全 D1/D3）**

```bash
ENGINEER_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"engineer","password":"Engineer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

curl -s -X PUT http://127.0.0.1:7001/api/v1/reports/1 \
  -H "Authorization: Bearer $ENGINEER_TOKEN" -H 'Content-Type: application/json' \
  -d '{
    "d1Team": "{\"members\":[\"李工(工艺)\",\"王工(设备)\",\"王主管(质量)\"]}",
    "d3Containment": "隔离批次 B20261015003 剩余在制晶圆，全检 WAT",
    "status": "pending_review"
  }'
# 预期：code 0，message "更新成功"，data.status = "pending_review"
```

**④ 签发（reviewer 有 report:release 权限）**

```bash
REVIEWER_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"reviewer","password":"Reviewer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

curl -s -X POST http://127.0.0.1:7001/api/v1/reports/1/release \
  -H "Authorization: Bearer $REVIEWER_TOKEN"
# 预期：code 0，message "报告已签发"，data.status = "released"，data.releasedAt 非空
```

**⑤ 状态机校验：重复签发 → 400**

```bash
curl -si -X POST http://127.0.0.1:7001/api/v1/reports/1/release \
  -H "Authorization: Bearer $REVIEWER_TOKEN" | head -n 1
# 预期：HTTP/1.1 400 Bad Request
# 响应体：{"code":10004,"message":"报告已签发，不能重复签发",...}

# 已签发的报告再编辑 -> 同样 400
curl -si -X PUT http://127.0.0.1:7001/api/v1/reports/1 \
  -H "Authorization: Bearer $ENGINEER_TOKEN" -H 'Content-Type: application/json' \
  -d '{"d8Recognition":"团队嘉奖"}' | head -n 1
# 预期：HTTP/1.1 400 Bad Request，message "报告已签发，质量档案不可再修改"
```

**⑥ 签发审计落库验证**

```bash
docker exec -it ia-pg psql -U industry -d industryagent \
  -c "SELECT username, path, request_summary FROM audit_logs WHERE request_summary LIKE '%release_8d_report%' ORDER BY id DESC LIMIT 1;"
# 预期：reviewer | /api/v1/reports/1/release | {"reportNo":"8D-...","action":"release_8d_report"}
```

---

## 18.2 通知模块 + WebSocket 实时推送

### 18.2.1 做什么 / 为什么

审批通过、异常发生这类事件，前端如果靠轮询感知会有几十秒延迟且浪费请求。方案是经典的"**站内信 + 实时推送**"组合：

- **站内信（notifications 表）**：落库的"信箱"，前端断线重连后按 `unread-count` / 列表接口补偿拉取（IA-SD-005 第 12.3 节的"断线补偿"设计）；
- **WebSocket 实时推送**：事件发生瞬间推到浏览器，帧格式为 `{"event","data","ts"}`（与 IA-SD-005 第 13 节及规格冻结的 WS 帧契约一致）。

**为什么是"单进程双 Server"**：推送服务与业务服务不在一个端口（WS 走 7002、REST 走 7001），但 FastAPI 是单进程异步模型——在**同一个进程**里用 `asyncio` 跑两个 uvicorn Server 即可，两个 Server 直接 `import` 同一个 `ConnectionManager` 单例，推送零进程间通信。

> （对照原 Egg.js 版的实现差异）原版框架是"多 worker 进程 + 唯一 agent 进程"的模型，WS server 必须挂在 agent 进程、worker 经 messenger 进程间消息转发广播，否则多 worker 各起端口必然冲突。FastAPI 单进程双 Server 是**等价简化**：连接表天然只有一份（无需分布式锁），且更贴合部署拓扑——第 15 章的一个容器 = 一个进程 = 两个端口。集群化（多副本广播）时再引入 Redis pub/sub，见 19.6.4 的进阶方向。

### 18.2.2 数据模型：app/models/notification.py（完整内容）

```python
from datetime import datetime

import sqlalchemy as sa
from sqlalchemy import func
from sqlalchemy.orm import Mapped, mapped_column

from app.core.database import Base


class Notification(Base):
    """
    通知表 notifications：用户的站内信箱
    type: anomaly（异常相关）/ approval（审批相关）/ system（系统公告）
    """
    __tablename__ = 'notifications'
    __table_args__ = (
        sa.Index('idx_notifications_user_read', 'userId', 'readFlag'),
    )

    id: Mapped[int] = mapped_column(sa.Integer, primary_key=True, autoincrement=True)
    userId: Mapped[int] = mapped_column(sa.Integer, nullable=False, comment='收件人')
    type: Mapped[str] = mapped_column(
        sa.Enum('anomaly', 'approval', 'system', name='notification_type'),
        nullable=False, server_default='system')
    title: Mapped[str] = mapped_column(sa.String(255), nullable=False)
    content: Mapped[str | None] = mapped_column(sa.Text)
    readFlag: Mapped[bool] = mapped_column(sa.Boolean, nullable=False, server_default=sa.false(), comment='已读标记')
    readAt: Mapped[datetime | None] = mapped_column(sa.DateTime(timezone=True), comment='已读时间')
    sourceId: Mapped[int | None] = mapped_column(sa.Integer, comment='来源业务对象 id（报告/事件等）')
    createdAt: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), server_default=func.now(), comment='创建时间')
    updatedAt: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), comment='更新时间')

    def as_dict(self) -> dict:
        return {
            'id': self.id, 'userId': self.userId, 'type': self.type, 'title': self.title,
            'content': self.content, 'readFlag': self.readFlag,
            'readAt': self.readAt.isoformat() if self.readAt else None,
            'sourceId': self.sourceId,
            'createdAt': self.createdAt.isoformat() if self.createdAt else None,
        }
```

> **建表说明**：同 18.1.2，重跑 `python scripts/init_db.py` 建表；`(userId, readFlag)` 复合索引支撑"未读数 / 未读列表"两个高频查询。版本化迁移属 Alembic 主线（`02-数据库设计与迁移教程`），此处不展开。

### 18.2.3 连接管理器：app/ws/manager.py（完整内容）

**做什么/为什么**：`ConnectionManager` 负责"谁在线"（按 userId 分房间）、"怎么推"（定向 / 广播）、"怎么踢"（心跳巡检）。它是**模块级单例**——REST Server 的通知 service 与 WS Server 的连接生命周期管理都 import 它，这是"单进程双 Server 共享 ConnectionManager"的关键。

```python
import asyncio
import json
import logging
import time

from fastapi import WebSocket

logger = logging.getLogger('app.ws')

# 心跳参数：30s 巡检一轮，连续两轮（60s）没有任何上行消息即视为死连接踢线
HEARTBEAT_INTERVAL_SECONDS = 30
HEARTBEAT_TOLERANCE_ROUNDS = 2


def user_room(user_id: int) -> str:
    """房间名约定：用户房间 user:<userId>"""
    return f'user:{user_id}'


def make_frame(event: str, data) -> str:
    """推送帧格式（IA-SD-005 第 13 节 / 重写规格第五节）：{"event","data","ts"}，ts 为 unix 秒"""
    return json.dumps({'event': event, 'data': data, 'ts': int(time.time())},
                      ensure_ascii=False, separators=(',', ':'))


class ConnectionManager:
    """WS 连接管理器：单进程内唯一，REST 与 WS 两个 Server 共享"""

    def __init__(self) -> None:
        # 房间表：room -> 连接集合（单进程唯一，天然无需分布式锁）
        self._rooms: dict[str, set[WebSocket]] = {}
        # 反查表：连接 -> 所在房间（断开时 O(1) 清理）
        self._room_of: dict[WebSocket, str] = {}
        # 活跃表：连接 -> 最近一次上行时间（心跳巡检依据）
        self._last_seen: dict[WebSocket, float] = {}
        self._heartbeat_task: asyncio.Task | None = None

    # ---------- 连接生命周期 ----------

    def connect(self, ws: WebSocket, user_id: int) -> None:
        room = user_room(user_id)
        self._rooms.setdefault(room, set()).add(ws)
        self._room_of[ws] = room
        self._last_seen[ws] = time.time()

    def disconnect(self, ws: WebSocket) -> None:
        room = self._room_of.pop(ws, None)
        self._last_seen.pop(ws, None)
        if room:
            conns = self._rooms.get(room)
            if conns:
                conns.discard(ws)
                if not conns:
                    self._rooms.pop(room, None)

    def mark_seen(self, ws: WebSocket) -> None:
        """任何上行消息（含 ping）都视为存活证据"""
        if ws in self._last_seen:
            self._last_seen[ws] = time.time()

    def online_count(self) -> int:
        return len(self._room_of)

    # ---------- 推送 ----------

    async def send_to_user(self, user_id: int, event: str, data) -> None:
        """定向推送到房间（notification.push 用，房间隔离：A 用户收不到 B 用户的通知）"""
        conns = self._rooms.get(user_room(user_id))
        if not conns:
            return  # 目标用户不在线：只落库不推送，前端上线后拉列表补偿
        message = make_frame(event, data)
        for ws in list(conns):
            await self._safe_send(ws, message)

    async def broadcast(self, event: str, data) -> None:
        """广播到全体已认证连接（anomaly.push 用，大屏关注）"""
        message = make_frame(event, data)
        for room in list(self._rooms.values()):
            for ws in list(room):
                await self._safe_send(ws, message)

    async def _safe_send(self, ws: WebSocket, message: str) -> None:
        # 推送瞬间对端可能刚好断开：吞掉异常，清理交给巡检/断开回调
        try:
            await ws.send_text(message)
        except Exception:  # noqa: BLE001
            self.disconnect(ws)

    # ---------- 心跳巡检 ----------

    def start_heartbeat(self) -> None:
        if self._heartbeat_task is None or self._heartbeat_task.done():
            self._heartbeat_task = asyncio.create_task(self._heartbeat_loop())

    async def shutdown(self) -> None:
        if self._heartbeat_task:
            self._heartbeat_task.cancel()
            try:
                await self._heartbeat_task
            except asyncio.CancelledError:
                pass

    async def _heartbeat_loop(self) -> None:
        """服务端心跳：30s 轮询，两次无应答即踢（客户端每 20s 上行 ping 即可保活）"""
        while True:
            await asyncio.sleep(HEARTBEAT_INTERVAL_SECONDS)
            deadline = time.time() - HEARTBEAT_INTERVAL_SECONDS * HEARTBEAT_TOLERANCE_ROUNDS
            dead = [ws for ws, seen in self._last_seen.items() if seen < deadline]
            for ws in dead:
                logger.info('[ws] 心跳超时，踢线：%s', self._room_of.get(ws))
                try:
                    await ws.close(code=4000, reason='heartbeat timeout')
                except Exception:  # noqa: BLE001
                    pass
                self.disconnect(ws)


# 进程级单例：REST（通知 service）与 WS（连接管理）两个 Server 共享同一份连接表
manager = ConnectionManager()
```

> 说明：浏览器 WebSocket API 不暴露协议层 ping/pong 帧，所以教程用**应用层心跳**——客户端上行 `{"event":"ping"}`，服务端回 `{"event":"pong","ts":...}`；服务端以"最近上行时间"判活，60 秒（两轮巡检）没有任何上行即踢线，与规格第五节的心跳契约一致。

### 18.2.4 WS 子应用：app/ws/app.py（完整内容）

**做什么/为什么**：一个极简的 FastAPI 子应用，只挂一条 WS 路由。**故意不连数据库**——它只做 token 验签与连接管理，依赖越少，单独重启/扩缩容越容易。

```python
import json
import logging
import time
from contextlib import asynccontextmanager

from fastapi import FastAPI, WebSocket, WebSocketDisconnect

from app.core.biz_error import BizError
from app.core.security import decode_token
from app.ws.manager import manager

logger = logging.getLogger('app.ws')


@asynccontextmanager
async def lifespan(app: FastAPI):
    manager.start_heartbeat()
    logger.info('[ws] 心跳巡检已启动（%ss 轮询，两次无应答踢线）', 30)
    yield
    await manager.shutdown()
    logger.info('[ws] 心跳巡检已停止')


# 子应用不暴露文档路由（WS 服务没有 HTTP 语义）
app = FastAPI(title='industryagent-ws', lifespan=lifespan,
              docs_url=None, redoc_url=None, openapi_url=None)


@app.websocket('/')
async def ws_endpoint(ws: WebSocket):
    # ---------- 1. 连接鉴权：URL query 携带 access token（WS 无法自定义 Header，通行做法） ----------
    token = ws.query_params.get('token', '')
    try:
        payload = decode_token(token)   # 第 7 章的验签函数：签名/过期校验
    except BizError:
        # 先完成握手再以 4001 关闭：WebSocket 握手阶段无法回 401，
        # 客户端要看到"关闭码 4001"就必须 accept 后再 close
        await ws.accept()
        await ws.close(code=4001, reason='unauthorized: invalid or expired token')
        return

    await ws.accept()
    user_id = payload['userId']
    username = payload.get('username', '')
    manager.connect(ws, user_id)
    logger.info('[ws] 用户上线：%s（userId=%s），当前连接数 %s',
                username, user_id, manager.online_count())

    try:
        # ---------- 2. 业务消息循环（客户端上行，本教程仅要求心跳） ----------
        while True:
            raw = await ws.receive_text()
            manager.mark_seen(ws)
            try:
                msg = json.loads(raw)
            except (json.JSONDecodeError, TypeError):
                continue
            if msg.get('event') == 'ping':
                await ws.send_text(json.dumps(
                    {'event': 'pong', 'ts': int(time.time())}, separators=(',', ':')))
    except WebSocketDisconnect:
        pass
    finally:
        manager.disconnect(ws)
        logger.info('[ws] 用户离线：%s，当前连接数 %s', username, manager.online_count())
```

### 18.2.5 run.py：双 uvicorn Server 启动器（完整内容）

**做什么/为什么**：从本章起，`python -m app.run` 一次拉起 **7001 REST + 7002 WS** 两个 Server（同一进程、同一事件循环）。三个工程要点：

1. **信号处理器只装一份**：uvicorn 的 `Server.serve()` 默认各自安装 SIGINT/SIGTERM 处理器，两个 Server 会互相覆盖——所以两边都用子类禁用自带安装，由 `run.py` 统一接收信号、统一分发退出事件；
2. **优雅关闭有顺序**：先让 REST 停收新请求、等存量请求完成，再关 WS（连接逐个断开），避免"WS 先断、前端还在重试 REST"的空洞窗口；
3. **任一 Server 启动失败（端口被占）立即整体退出**，不留半瘫进程。

`app/run.py`（完整内容）：

```python
"""
双 uvicorn Server 启动器（第 18 章起）：
  - 7001 REST（app.main:app）
  - 7002 WS（app.ws.app:app）
用法：python -m app.run
"""
import asyncio
import logging
import os
import signal

import uvicorn

from app.core.config import get_settings

logging.basicConfig(level=logging.INFO, format='%(asctime)s %(levelname)s %(message)s')
logger = logging.getLogger('app.run')


class QuietServer(uvicorn.Server):
    """不安装自己的信号处理器：退出信号由 run.py 统一分发给两个 Server"""

    def install_signal_handlers(self) -> None:
        pass


async def main() -> None:
    settings = get_settings()
    rest_port = int(os.getenv('IA_PORT', '7001'))
    ws_port = settings.IA_WS_PORT

    rest = QuietServer(uvicorn.Config('app.main:app', host='0.0.0.0', port=rest_port, log_level='info'))
    ws = QuietServer(uvicorn.Config('app.ws.app:app', host='0.0.0.0', port=ws_port, log_level='info'))

    # 统一的停止事件：Ctrl+C / SIGTERM / 任一 Server 意外退出都会触发
    loop = asyncio.get_running_loop()
    stop = asyncio.Event()
    for sig in (signal.SIGINT, signal.SIGTERM):
        try:
            loop.add_signal_handler(sig, stop.set)
        except NotImplementedError:
            # Windows ProactorEventLoop 不支持 add_signal_handler，退回传统注册（仅在主线程可用）
            signal.signal(sig, lambda *_: stop.set())

    serve_tasks = [
        asyncio.create_task(rest.serve(), name='rest-server'),
        asyncio.create_task(ws.serve(), name='ws-server'),
    ]
    stop_task = asyncio.create_task(stop.wait(), name='stop-wait')
    logger.info('启动 REST Server :%s / WS Server :%s', rest_port, ws_port)

    done, _ = await asyncio.wait({stop_task, *serve_tasks}, return_when=asyncio.FIRST_COMPLETED)

    # 优雅关闭：REST 先停（停收新连接、等存量请求），1 秒后再关 WS（连接逐个断开）
    rest.should_exit = True
    await asyncio.sleep(1)
    ws.should_exit = True
    results = await asyncio.gather(*serve_tasks, return_exceptions=True)

    # 任一 Server 带异常退出（典型：端口被占用）→ 非零退出码，交给 supervisor/compose 重启
    failed = [r for r in results if isinstance(r, Exception)]
    for err in failed:
        logger.error('Server 异常退出：%s', err)
    if failed or any(t in done for t in serve_tasks):
        raise SystemExit(1)


if __name__ == '__main__':
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        pass
```

> **Windows/Git Bash 提示**：开发期调试 REST 单模块时，仍可用第 2 章的 `uvicorn app.main:app --reload --port 7001`；只要开始验证 18.2 的推送，就换 `python -m app.run`（`--reload` 与双 Server 的信号编排冲突，run.py 不做热重载）。

### 18.2.6 Service：app/services/notification.py（完整内容）

```python
import logging

from sqlalchemy import func, select, update
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.biz_error import BizError
from app.core.error_codes import ErrorCode
from app.models.notification import Notification
from app.ws.manager import manager

logger = logging.getLogger(__name__)


class NotificationService:
    """站内信 + 实时推送：落库（信箱）与推帧（在线直达）双通道"""

    def __init__(self, db: AsyncSession):
        self.db = db

    async def create_notification(self, param: dict) -> Notification:
        """
        创建通知（内部 service 方法，供异常事件/审批/报告等业务埋点调用）
        落库 + 经共享 ConnectionManager 向该用户的 WS 房间实时推送
        """
        notification = Notification(
            userId=param['userId'],
            type=param.get('type', 'system'),
            title=param['title'],
            content=param.get('content'),
            sourceId=param.get('sourceId'),
        )
        self.db.add(notification)
        await self.db.commit()
        await self.db.refresh(notification)

        # 实时推送：同进程直接调 ConnectionManager（用户不在线则只落库，前端上线后补偿拉取）
        try:
            await manager.send_to_user(param['userId'], 'notification.push', {
                'id': notification.id,
                'type': notification.type,
                'title': notification.title,
                'content': notification.content,
                'sourceId': notification.sourceId,
                'createdAt': notification.createdAt.isoformat() if notification.createdAt else None,
            })
        except Exception as e:  # noqa: BLE001
            logger.warning('[notify] 实时推送失败: %s', e)
        return notification

    async def push_anomaly(self, data: dict) -> None:
        """
        异常事件广播（供异常事件创建/根因任务完成等处调用）
        与 create_notification 不同：anomaly.push 面向全体在线用户广播（大屏关注），不落库
        """
        await manager.broadcast('anomaly.push', {
            'anomalyId': data['anomalyId'],
            'equipmentCode': data['equipment'],
            'title': data['title'],
            'severity': data['severity'],
            'occurredAt': data['occurredAt'].isoformat() if hasattr(data['occurredAt'], 'isoformat') else data['occurredAt'],
        })

    async def unread_count(self, user_id: int) -> dict:
        count = await self.db.scalar(
            select(func.count()).select_from(Notification)
            .where(Notification.userId == user_id, Notification.readFlag.is_(False)))
        return {'unreadCount': count or 0}

    async def list(self, param: dict) -> dict:
        """我的通知列表（分页 + 已读过滤）"""
        stmt = select(Notification).where(Notification.userId == param['userId'])
        if param.get('readFlag') is not None:
            stmt = stmt.where(Notification.readFlag == param['readFlag'])
        total = await self.db.scalar(select(func.count()).select_from(stmt.subquery()))
        rows = (await self.db.execute(
            stmt.order_by(Notification.id.desc())
            .offset((param['page'] - 1) * param['pageSize'])
            .limit(param['pageSize'])
        )).scalars().all()
        return {'rows': [r.as_dict() for r in rows], 'total': total or 0}

    async def mark_read(self, user_id: int, notification_id: int) -> dict:
        """标记单条已读（只能操作自己的通知：where 同时带 userId）"""
        notification = (await self.db.execute(
            select(Notification).where(Notification.id == notification_id,
                                        Notification.userId == user_id)
        )).scalar_one_or_none()
        if not notification:
            raise BizError(ErrorCode.NOT_FOUND, f'通知不存在（id={notification_id}）', http_status=404)
        if not notification.readFlag:
            notification.readFlag = True
            notification.readAt = func.now()
            await self.db.commit()
            await self.db.refresh(notification)
        return notification.as_dict()

    async def mark_all_read(self, user_id: int) -> dict:
        result = await self.db.execute(
            update(Notification)
            .where(Notification.userId == user_id, Notification.readFlag.is_(False))
            .values(readFlag=True, readAt=func.now())
        )
        await self.db.commit()
        return {'updated': result.rowcount}
```

### 18.2.7 Router：app/routers/notification.py（完整内容）

```python
from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.database import get_db
from app.core.deps import CurrentUser
from app.core.response import page, success
from app.services.notification import NotificationService

router = APIRouter()


@router.get('/api/v1/notifications/unread-count')
async def unread_count(
    db: AsyncSession = Depends(get_db),
    user: CurrentUser,
):
    """GET /api/v1/notifications/unread-count"""
    return success(await NotificationService(db).unread_count(user['userId']))


@router.get('/api/v1/notifications')
async def list_notifications(
    page_no: int = Query(1, alias='page', ge=1),
    page_size: int = Query(10, alias='pageSize', ge=1, le=500),
    readFlag: bool | None = Query(None),
    db: AsyncSession = Depends(get_db),
    user: CurrentUser,
):
    """GET /api/v1/notifications?page=1&pageSize=10&readFlag=false"""
    result = await NotificationService(db).list({
        'userId': user['userId'], 'page': page_no, 'pageSize': page_size, 'readFlag': readFlag,
    })
    return page(result['rows'], result['total'], page_no, page_size)


@router.put('/api/v1/notifications/read-all')
async def mark_all_read(
    db: AsyncSession = Depends(get_db),
    user: CurrentUser,
):
    """PUT /api/v1/notifications/read-all"""
    return success(await NotificationService(db).mark_all_read(user['userId']), '全部已读')


@router.put('/api/v1/notifications/{notification_id}/read')
async def mark_read(
    notification_id: int,
    db: AsyncSession = Depends(get_db),
    user: CurrentUser,
):
    """PUT /api/v1/notifications/:id/read"""
    return success(await NotificationService(db).mark_read(user['userId'], notification_id), '已标记已读')
```

> **注册顺序说明**：FastAPI 按注册顺序匹配路由。`read-all`（单段）与 `/{notification_id}/read`（两段）路径结构不同，不会互相吞噬；但沿用原教程的注册顺序（静态段在前、参数段在后），是可读性更好的习惯——哪天加了 `GET /api/v1/notifications/{id}` 就能避免 `unread-count` 被当成 `{id}` 解析。

`app/main.py` 追加挂载：

```python
    from app.routers import notification
    app.include_router(notification.router)
```

### 18.2.8 业务埋点：在哪里调用通知

三个埋点位置（都在 **service 层**，不在 router——通知是业务副作用，与 HTTP 无关）：

**埋点 1：异常事件发生时广播**。前文异常事件由监控 Agent / FDC 推送写入（教程主线暂无创建接口）。若你按 8 章扩展了 `POST /api/v1/anomaly-events`，在事件 create 之后加：

```python
    # 广播异常事件 + 通知相关批次责任人（演示固定通知 engineer）
    await NotificationService(self.db).push_anomaly({
        'anomalyId': event.id, 'equipment': event.equipment,
        'title': event.title, 'severity': event.severity, 'occurredAt': event.occurredAt,
    })
```

**埋点 2：根因分析完成时通知触发人**。在 `app/services/rootcause.py` 的 `analyze` 方法中，`task` 更新为 `status='succeeded'` 之后、`return task` 之前加：

```python
      # 通知任务触发人：分析已完成
      if param.get('createdBy'):
          try:
              await NotificationService(self.db).create_notification({
                  'userId': param['createdBy'],
                  'type': 'anomaly',
                  'title': f"批次 {param['batchNo']} 根因分析完成",
                  'content': f"Top-1 根因：{(agent_result.get('root_causes') or [{}])[0].get('description') or '见任务详情'}",
                  'sourceId': task.id,
              })
          except Exception as e:  # noqa: BLE001
              logger.warning('[notify] %s', e)
```

**埋点 3：审批完成时通知建议发起人**。在 `app/services/suggestion.py` 的 `submit_approval` 中、第 18.1.6 节挂钩代码的 `if param['action'] == 'approve'` 块**之后**追加（approve / reject 都要通知）：

```python
    # 通知建议发起人审批结果
    if result['suggestion'].createdBy:
        try:
            await NotificationService(self.db).create_notification({
                'userId': result['suggestion'].createdBy,
                'type': 'approval',
                'title': '你的建议已批准' if param['action'] == 'approve' else '你的建议被驳回',
                'content': f"审批人 {param['approverName']}，理由：{param['reason']}",
                'sourceId': result['suggestion'].id,
            })
        except Exception as e:  # noqa: BLE001
            logger.warning('[notify] %s', e)
```

（18.1.3 `consume_approval` 与 `release` 中的通知调用已经在各自 service 内部埋好。）

### 18.2.9 前端 WS 客户端如何对接（呼应前端教程）

前端（React）侧的对接要点（完整实现见《01-前端开发教程-React从0到1》相应章节）：

```typescript
// 1. 建连：token 走 query（WS 无法自定义 Header，这是通行做法）
const ws = new WebSocket(`ws://127.0.0.1:7002/?token=${accessToken}`);

// 2. 收到推送：按 event 分发
ws.onmessage = e => {
  const { event, data } = JSON.parse(e.data);
  switch (event) {
    case 'anomaly.push':      // 大屏弹新异常红点
      store.appendAnomaly(data); break;
    case 'notification.push': // 铃铛角标 +1
      store.increaseUnread(data); break;
    case 'pong': break;
  }
};

// 3. 心跳（服务端 30s 踢无应答连接，客户端每 20s 主动 ping）
setInterval(() => ws.readyState === WebSocket.OPEN && ws.send('{"event":"ping"}'), 20000);

// 4. 断线重连 + 补偿：重连成功后先拉 GET /api/v1/notifications?readFlag=false，
//    补上断线期间漏掉的通知（IA-SD-005 12.3 的"断线补偿"）。
```

### 18.2.10 curl 验证

重启 `python -m app.run`（日志应同时出现 REST 与 WS 两个 Server 的启动行，以及 `[ws] 心跳巡检已启动`）。跑一遍审批剧本让通知落库后：

**① 未读数**

```bash
ENGINEER_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"engineer","password":"Engineer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

curl -s http://127.0.0.1:7001/api/v1/notifications/unread-count \
  -H "Authorization: Bearer $ENGINEER_TOKEN"
```

预期输出（审批批准 + 报告骨架 + 签发各一条，具体数量取决于你跑了几遍剧本）：

```json
{ "code": 0, "message": "ok", "data": { "unreadCount": 3 }, "traceId": "..." }
```

**② 通知列表（只看未读）**

```bash
curl -s 'http://127.0.0.1:7001/api/v1/notifications?readFlag=false&pageSize=5' \
  -H "Authorization: Bearer $ENGINEER_TOKEN"
```

预期输出（节选）：

```json
{
  "code": 0, "message": "ok",
  "data": {
    "rows": [
      { "id": 3, "userId": 2, "type": "system", "title": "8D 报告已签发",
        "content": "报告 8D-20261015-xxxxx 已由 reviewer 签发。", "readFlag": false,
        "readAt": null, "sourceId": 1, "createdAt": "2026-10-15T..." },
      { "id": 2, "userId": 2, "type": "approval", "title": "你的建议已批准", "...": "..." }
    ],
    "total": 3, "page": 1, "pageSize": 5
  },
  "traceId": "..."
}
```

**③ 标记已读 + 全部已读**

```bash
curl -s -X PUT http://127.0.0.1:7001/api/v1/notifications/1/read \
  -H "Authorization: Bearer $ENGINEER_TOKEN"
# 预期：code 0，message "已标记已读"，data.readFlag = true，data.readAt 非空

curl -s -X PUT http://127.0.0.1:7001/api/v1/notifications/read-all \
  -H "Authorization: Bearer $ENGINEER_TOKEN"
# 预期：{"code":0,"message":"全部已读","data":{"updated":2},...}

curl -s http://127.0.0.1:7001/api/v1/notifications/unread-count \
  -H "Authorization: Bearer $ENGINEER_TOKEN"
# 预期：{"unreadCount":0}
```

**④ WebSocket 推送验证**（需要 Python 一行脚本做客户端；`websockets` 库已随 `uvicorn[standard]` 装好）：

```bash
python -c "
import asyncio, sys
import websockets

async def main():
    token = sys.argv[1]
    async with websockets.connect(f'ws://127.0.0.1:7002/?token={token}') as ws:
        print('[client] 已连接')
        try:
            while True:
                msg = await asyncio.wait_for(ws.recv(), timeout=15)
                print('[client] 收到推送:', msg)
        except asyncio.TimeoutError:
            pass  # 15 秒没收到推送就退出

asyncio.run(main())
" "$ENGINEER_TOKEN"
```

保持这个客户端挂着，另开终端触发一次审批（对 pending 状态的建议 approve），客户端应立即打印：

```text
[client] 已连接
[client] 收到推送: {"event":"notification.push","data":{"id":4,"type":"approval","title":"你的建议已批准","content":"审批人 reviewer，理由：...","sourceId":2,"createdAt":"..."},"ts":1760493305}
```

再验证鉴权（不带 token 连接应被 4001 关闭）：

```bash
python -c "
import asyncio
import websockets

async def main():
    try:
        async with websockets.connect('ws://127.0.0.1:7002/') as ws:
            await ws.recv()
    except websockets.exceptions.ConnectionClosed as e:
        print('[client] 关闭码:', e.rcvd.code, '原因:', e.rcvd.reason)

asyncio.run(main())
"
# 预期输出：[client] 关闭码: 4001 原因: unauthorized: invalid or expired token
```

> 习惯 Node 工具链的读者也可用 `npx wscat --connect "ws://127.0.0.1:7002/?token=<JWT>"` 做同样的收帧验证；心跳自测可手动发 `{"event":"ping"}`，应立即收到 `{"event":"pong","ts":...}`。

---

## 18.3 缺陷模块（MinIO 对象存储 + 预签名 URL）

### 18.3.1 做什么 / 为什么

缺陷图（AOI 显微照片）单张几 MB 到几十 MB，存 PostgreSQL 会把库拖垮；正确姿势是**对象存储**（MinIO，S3 兼容），数据库只存 `imageObjectKey`。前端看图走**预签名 URL**：后端用凭证替用户签一个限时 GET URL，图片流量直连 MinIO，不经过业务后端，凭证也不下发到前端——这就是"数据不出域"的图片访问方案（对应 IA-SD-004 1.1 节 MinIO `defect-images` 桶与 IA-SD-005 7.2 节接口）。

### 18.3.2 拉起 MinIO 并建桶

```bash
# MinIO：容器名 ia-minio，API 9000 / 控制台 9001，与 .env 中凭证一致
docker run -d --name ia-minio \
  -e MINIO_ROOT_USER=industry \
  -e MINIO_ROOT_PASSWORD=industry123 \
  -p 9000:9000 -p 9001:9001 \
  minio/minio server /data --console-address ":9001"

# 预期输出：一长串容器 ID
```

浏览器打开 `http://127.0.0.1:9001`（industry / industry123）可看到控制台。桶由下方 `app/core/minio_client.py` 的 `ensure_bucket` 在首次访问时自动创建，无需手工操作。

### 18.3.3 数据模型：app/models/defect_record.py（完整内容）

字段语义对齐《数据库设计说明书（IA-SD-004）》的 `defect_records`（wafer/批次、设备、缺陷类型、坐标、尺寸、图片键、严重度、检出时间）；按教程惯例用整型主键与 camelCase 字段（差异说明见 18.7.3）。

```python
from datetime import datetime
from decimal import Decimal

import sqlalchemy as sa
from sqlalchemy import func
from sqlalchemy.orm import Mapped, mapped_column

from app.core.database import Base


class DefectRecord(Base):
    """
    缺陷记录表 defect_records
    由 AOI / 缺陷检测设备检出写入；imageObjectKey 指向 MinIO 的缺陷图
    """
    __tablename__ = 'defect_records'
    __table_args__ = (
        sa.Index('idx_defect_records_defectType', 'defectType'),
        sa.Index('idx_defect_records_detectedAt', 'detectedAt'),
    )

    id: Mapped[int] = mapped_column(sa.Integer, primary_key=True, autoincrement=True)
    waferId: Mapped[int | None] = mapped_column(sa.Integer, comment='关联晶圆（教程简化：批次维度）')
    batchNo: Mapped[str | None] = mapped_column(sa.String(32), comment='批次号')
    equipmentCode: Mapped[str] = mapped_column(sa.String(64), nullable=False, comment='检出设备，如 ETCH-03')
    processStep: Mapped[str | None] = mapped_column(sa.String(32), comment='工序步，如 ETCH_MAIN')
    defectType: Mapped[str] = mapped_column(sa.String(32), nullable=False, comment='缺陷类型：PATTERN/PARTICLE/SCRATCH/RESIDUE...')
    defectCount: Mapped[int] = mapped_column(sa.Integer, nullable=False, server_default='1', comment='该类型缺陷个数')
    xPos: Mapped[Decimal | None] = mapped_column(sa.Numeric(8, 2), comment='缺陷位置 X（mm）')
    yPos: Mapped[Decimal | None] = mapped_column(sa.Numeric(8, 2), comment='缺陷位置 Y（mm）')
    sizeUm: Mapped[Decimal | None] = mapped_column(sa.Numeric(8, 2), comment='缺陷尺寸（微米）')
    imageObjectKey: Mapped[str | None] = mapped_column(sa.String(255), comment='MinIO 对象键，如 defects/2026/10/xxx.png')
    severity: Mapped[str] = mapped_column(
        sa.Enum('low', 'medium', 'high', name='defect_severity'),
        nullable=False, server_default='low', comment='严重度')
    detectedAt: Mapped[datetime] = mapped_column(sa.DateTime(timezone=True), nullable=False, comment='检出时间')
    createdAt: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), server_default=func.now(), comment='创建时间')
    updatedAt: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), comment='更新时间')

    def as_dict(self) -> dict:
        """Decimal 保持字符串序列化（与原教程 DECIMAL 的 JSON 输出一致，避免浮点精度争议）"""
        out: dict = {}
        for c in self.__table__.columns:
            v = getattr(self, c.name)
            if isinstance(v, datetime):
                v = v.isoformat()
            elif isinstance(v, Decimal):
                v = str(v)
            out[c.name] = v
        return out
```

> **建表说明**：同 18.1.2，重跑 `python scripts/init_db.py` 建表（18.3.7 一起把种子数据补上）。`defectType` / `detectedAt` 两个索引支撑 18.4 看板的分组聚合与时间过滤。版本化迁移属 Alembic 主线。

### 18.3.4 MinIO 客户端：app/core/minio_client.py（完整内容）

```python
"""
MinIO 客户端（18.3）：进程内单例 + 首次访问自动建桶
注意：minio 官方 Python SDK 是同步实现，所有调用经 asyncio.to_thread 丢进线程池，
避免阻塞事件循环（对象存储操作动辄几十毫秒，直接 await 会拖慢全服吞吐）。
"""
import asyncio
import logging

from minio import Minio

from app.core.config import get_settings

logger = logging.getLogger('app.minio')

settings = get_settings()

client = Minio(
    f'{settings.IA_MINIO_ENDPOINT}:{settings.IA_MINIO_PORT}',
    access_key=settings.IA_MINIO_ACCESS_KEY,
    secret_key=settings.IA_MINIO_SECRET_KEY,
    secure=False,   # 内网 HTTP；生产走 TLS 置 True 并配证书
)

# 教程用的桶名：与 IA-SD-004 1.1 节一致
BUCKET = settings.IA_MINIO_BUCKET

_bucket_ready = False


async def ensure_bucket() -> str:
    """确保桶存在（幂等，首次调用时创建；进程内只查一次）"""
    global _bucket_ready
    if _bucket_ready:
        return BUCKET
    exists = await asyncio.to_thread(client.bucket_exists, BUCKET)
    if not exists:
        # 第二个参数是 region（S3 兼容语义），预签名 URL 的 Credential 路径会带上它
        await asyncio.to_thread(client.make_bucket, BUCKET, 'cn-north-1')
        logger.info('[minio] 桶 %s 不存在，已自动创建', BUCKET)
    _bucket_ready = True
    return BUCKET
```

### 18.3.5 Service：app/services/defect.py（完整内容）

```python
import asyncio
import io
import logging
import uuid
from datetime import datetime

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.biz_error import BizError
from app.core.error_codes import ErrorCode
from app.core.minio_client import BUCKET, client, ensure_bucket
from app.models.defect_record import DefectRecord

logger = logging.getLogger(__name__)

# 预签名有效期：7 天（秒）——质检复盘窗口
PRESIGN_EXPIRES_SECONDS = 7 * 24 * 3600

# 上传约束（语义同原 egg-multipart 配置：白名单 + 10mb 上限）
ALLOWED_EXTS = ('.png', '.jpg', '.jpeg', '.bmp', '.webp')
MAX_UPLOAD_BYTES = 10 * 1024 * 1024


class DefectService:

    def __init__(self, db: AsyncSession):
        self.db = db

    async def list(self, param: dict) -> dict:
        """分页查询（多条件过滤：批次/设备/类型/严重度/时间范围）"""
        stmt = select(DefectRecord)
        if param.get('batchNo'):
            stmt = stmt.where(DefectRecord.batchNo.like(f"%{param['batchNo']}%"))
        if param.get('equipmentCode'):
            stmt = stmt.where(DefectRecord.equipmentCode.like(f"%{param['equipmentCode']}%"))
        if param.get('defectType'):
            stmt = stmt.where(DefectRecord.defectType == param['defectType'])
        if param.get('severity'):
            stmt = stmt.where(DefectRecord.severity == param['severity'])
        if param.get('startTime') or param.get('endTime'):
            cond = []
            if param.get('startTime'):
                cond.append(DefectRecord.detectedAt >= datetime.fromisoformat(param['startTime']))
            if param.get('endTime'):
                cond.append(DefectRecord.detectedAt <= datetime.fromisoformat(param['endTime']))
            stmt = stmt.where(*cond)

        total = await self.db.scalar(select(func.count()).select_from(stmt.subquery()))
        rows = (await self.db.execute(
            stmt.order_by(DefectRecord.detectedAt.desc())
            .offset((param['page'] - 1) * param['pageSize'])
            .limit(param['pageSize'])
        )).scalars().all()
        return {'rows': [r.as_dict() for r in rows], 'total': total or 0}

    async def detail(self, defect_id: int) -> DefectRecord:
        defect = await self.db.get(DefectRecord, defect_id)
        if not defect:
            raise BizError(ErrorCode.NOT_FOUND, f'缺陷记录不存在（id={defect_id}）', http_status=404)
        return defect

    async def get_image_url(self, defect_id: int) -> dict:
        """
        缺陷图预签名 GET URL（7 天有效）
        凭证只在本服务端使用，URL 本身限时、可分享给内网浏览器直接访问
        """
        defect = await self.detail(defect_id)
        if not defect.imageObjectKey:
            raise BizError(ErrorCode.NOT_FOUND, f'该缺陷记录没有关联图片（id={defect_id}）', http_status=404)
        bucket = await ensure_bucket()
        url = await asyncio.to_thread(
            client.presigned_get_object, bucket, defect.imageObjectKey, PRESIGN_EXPIRES_SECONDS)
        return {
            'url': url,
            'objectKey': defect.imageObjectKey,
            'expiresIn': PRESIGN_EXPIRES_SECONDS,
        }

    async def upload_image(self, defect_id: int, filename: str, content_type: str | None, data: bytes) -> dict:
        """
        上传缺陷图（UploadFile 已被 starlette 落到磁盘临时区，这里读入内存后直写 MinIO，不落业务盘）
        返回新的 objectKey 并回写到缺陷记录
        """
        defect = await self.detail(defect_id)

        # 扩展名白名单校验（第一道在 Router 的 UploadFile 文件名上，此处兜底）
        ext = '.' + str(filename or '').split('.')[-1].lower()
        if ext not in ALLOWED_EXTS:
            raise BizError(ErrorCode.PARAM_INVALID, f'不支持的图片类型：{ext}', http_status=422)
        if len(data) > MAX_UPLOAD_BYTES:
            raise BizError(ErrorCode.PARAM_INVALID, '图片不能超过 10MB', http_status=422)

        bucket = await ensure_bucket()
        now = datetime.now()
        object_key = f'defects/{now.year}/{now.month:02d}/{uuid.uuid4().hex}{ext}'

        try:
            await asyncio.to_thread(
                client.put_object, bucket, object_key,
                io.BytesIO(data), len(data),
                content_type=content_type or 'application/octet-stream',
            )
        except Exception as e:  # noqa: BLE001
            logger.error('[defect] MinIO 上传失败: %s', e)
            raise BizError(50000, '对象存储上传失败，请稍后重试', http_status=502) from e

        defect.imageObjectKey = object_key
        await self.db.commit()
        return {'id': defect.id, 'imageObjectKey': object_key}
```

### 18.3.6 Router：app/routers/defect.py（完整内容）

```python
from fastapi import APIRouter, Depends, File, Query, UploadFile
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.database import get_db
from app.core.deps import CurrentUser, require_perm
from app.core.response import page, success
from app.services.defect import DefectService

router = APIRouter()


@router.get('/api/v1/defects')
async def list_defects(
    page_no: int = Query(1, alias='page', ge=1),
    page_size: int = Query(10, alias='pageSize', ge=1, le=500),
    batchNo: str | None = Query(None),
    equipmentCode: str | None = Query(None),
    defectType: str | None = Query(None),
    severity: str | None = Query(None, pattern='^(low|medium|high)$'),
    startTime: str | None = Query(None),
    endTime: str | None = Query(None),
    db: AsyncSession = Depends(get_db),
    _: CurrentUser,
):
    """GET /api/v1/defects?batchNo=&equipmentCode=&defectType=&severity=&startTime=&endTime="""
    result = await DefectService(db).list({
        'page': page_no, 'pageSize': page_size,
        'batchNo': batchNo, 'equipmentCode': equipmentCode,
        'defectType': defectType, 'severity': severity,
        'startTime': startTime, 'endTime': endTime,
    })
    return page(result['rows'], result['total'], page_no, page_size)


@router.get('/api/v1/defects/{defect_id}/image-url')
async def get_image_url(
    defect_id: int,
    db: AsyncSession = Depends(get_db),
    _: CurrentUser,
):
    """GET /api/v1/defects/:id/image-url（MinIO 预签名）"""
    return success(await DefectService(db).get_image_url(defect_id))


@router.post('/api/v1/defects/{defect_id}/image')
async def upload_image(
    defect_id: int,
    file: UploadFile = File(...),
    db: AsyncSession = Depends(get_db),
    user: dict = Depends(require_perm('defect:upload')),
):
    """POST /api/v1/defects/:id/image（multipart 上传，直写 MinIO）

    原教程在 Controller 手写的"非 multipart 返回 422"检查，FastAPI 声明了
    UploadFile = File(...) 后自动完成（缺 file 字段即 422），无需重复实现。
    """
    data = await file.read()
    result = await DefectService(db).upload_image(
        defect_id, file.filename, file.content_type, data)
    return success(result, '上传成功', status_code=201)
```

### 18.3.7 路由注册与种子数据

`app/main.py` 追加挂载：

```python
    from app.routers import defect
    app.include_router(defect.router)
```

`scripts/init_db.py` 追加种子数据（放在传感器读数之后）：

```python
  # ---------- 种子数据：缺陷记录（18.3） ----------
  from app.models.defect_record import DefectRecord

  def hours_ago(n: int):
      """种子相对时间：n 小时前（沿用第 5 章种子脚本的同名辅助）"""
      return datetime.now(timezone.utc) - timedelta(hours=n)

  db.add_all([
      DefectRecord(batchNo='B20261015001', equipmentCode='ETCH-03', processStep='ETCH_MAIN', defectType='PATTERN', defectCount=6, xPos=Decimal('42.10'), yPos=Decimal('88.30'), sizeUm=Decimal('1.80'), severity='high', detectedAt=hours_ago(28)),
      DefectRecord(batchNo='B20261015001', equipmentCode='ETCH-03', processStep='ETCH_MAIN', defectType='PARTICLE', defectCount=2, xPos=Decimal('12.50'), yPos=Decimal('30.00'), sizeUm=Decimal('0.90'), severity='medium', detectedAt=hours_ago(27)),
      DefectRecord(batchNo='B20261015003', equipmentCode='CVD-11', processStep='CVD_MAIN', defectType='SCRATCH', defectCount=3, xPos=Decimal('66.20'), yPos=Decimal('15.80'), sizeUm=Decimal('5.20'), severity='medium', detectedAt=hours_ago(6)),
      DefectRecord(batchNo='B20261015003', equipmentCode='LITH-02', processStep='LITHO_DEV', defectType='RESIDUE', defectCount=1, xPos=Decimal('50.00'), yPos=Decimal('50.00'), sizeUm=Decimal('2.40'), severity='low', detectedAt=hours_ago(30)),
      DefectRecord(batchNo='B20261015002', equipmentCode='CVD-11', processStep='CVD_MAIN', defectType='PATTERN', defectCount=1, xPos=Decimal('20.10'), yPos=Decimal('71.60'), sizeUm=Decimal('1.10'), severity='low', detectedAt=hours_ago(25)),
      DefectRecord(batchNo='B20261015001', equipmentCode='CMP-02', processStep='CMP_MAIN', defectType='PARTICLE', defectCount=4, xPos=Decimal('88.00'), yPos=Decimal('9.90'), sizeUm=Decimal('0.60'), severity='medium', detectedAt=hours_ago(20)),
  ])
  await db.commit()
  print('[seed] 缺陷记录 6 条写入完成')
```

再准备一个造图小工具 `scripts/make_test_png.py`（完整内容；原教程用 shell `printf` 拼 PNG 字节，Python 版改为脚本，字节内容与原版逐字节一致）：

```python
"""造一张 1x1 像素 PNG 测试图（18.3 上传演练用）
用法：python scripts/make_test_png.py [输出路径]   # 默认 /tmp/defect.png
"""
import sys

# 标准 1x1 RGBA PNG（67 字节）：文件签名 + IHDR + IDAT + IEND
PNG_1X1 = (
    b'\x89PNG\r\n\x1a\n'
    b'\x00\x00\x00\rIHDR\x00\x00\x00\x01\x00\x00\x00\x01\x08\x06\x00\x00\x00\x1f\x15\xc4\x89'
    b'\x00\x00\x00\nIDATx\x9cc\x00\x01\x00\x00\x05\x00\x01\r\n-\xb4'
    b'\x00\x00\x00\x00IEND\xaeB`\x82'
)

if __name__ == '__main__':
    out = sys.argv[1] if len(sys.argv) > 1 else '/tmp/defect.png'
    with open(out, 'wb') as f:
        f.write(PNG_1X1)
    print(f'[make_test_png] 已生成 {out}（{len(PNG_1X1)} 字节）')
```

重新执行 `python scripts/init_db.py` 并重启 `python -m app.run`（确认 ia-minio 容器在跑）。

### 18.3.8 curl 验证

**① 分页 + 多条件过滤（缺陷类型 PATTERN + 严重度 high）**

```bash
ADMIN_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"Admin@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

curl -s 'http://127.0.0.1:7001/api/v1/defects?defectType=PATTERN&severity=high' \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

预期输出：

```json
{
  "code": 0, "message": "ok",
  "data": {
    "rows": [
      { "id": 1, "batchNo": "B20261015001", "equipmentCode": "ETCH-03",
        "processStep": "ETCH_MAIN", "defectType": "PATTERN", "defectCount": 6,
        "xPos": "42.10", "yPos": "88.30", "sizeUm": "1.80",
        "severity": "high", "detectedAt": "2026-10-14T..." }
    ],
    "total": 1, "page": 1, "pageSize": 10
  },
  "traceId": "..."
}
```

**② 无图记录申请预签名 → 404**

```bash
curl -si http://127.0.0.1:7001/api/v1/defects/1/image-url \
  -H "Authorization: Bearer $ADMIN_TOKEN" | head -n 1
# 预期：HTTP/1.1 404 Not Found
# 响应体：{"code":10002,"message":"该缺陷记录没有关联图片（id=1）",...}
```

**③ 工程师上传缺陷图（multipart）**

```bash
ENGINEER_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"engineer","password":"Engineer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

# 造一张 1x1 像素 PNG 测试图
python scripts/make_test_png.py /tmp/defect.png
# 预期输出：[make_test_png] 已生成 /tmp/defect.png（67 字节）

curl -s -X POST http://127.0.0.1:7001/api/v1/defects/1/image \
  -H "Authorization: Bearer $ENGINEER_TOKEN" \
  -F 'file=@/tmp/defect.png;type=image/png'
```

预期输出（HTTP 201）：

```json
{
  "code": 0, "message": "上传成功",
  "data": { "id": 1, "imageObjectKey": "defects/2026/10/3f2c1a4e-....png" },
  "traceId": "..."
}
```

**④ 再次申请预签名 URL 并直接访问图片**

```bash
curl -s http://127.0.0.1:7001/api/v1/defects/1/image-url \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

预期输出：

```json
{
  "code": 0, "message": "ok",
  "data": {
    "url": "http://127.0.0.1:9000/defect-images/defects/2026/10/3f2c1a4e-....png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=industry%2F20261015%2Fcn-north-1%2Fs3%2Faws4_request&X-Amz-Date=...&X-Amz-Expires=604800&X-Amz-SignedHeaders=host&X-Amz-Signature=...",
    "objectKey": "defects/2026/10/3f2c1a4e-....png",
    "expiresIn": 604800
  },
  "traceId": "..."
}
```

```bash
# 用返回的 URL 直连 MinIO 取图（无需任何业务 token）
curl -si '<上面返回的 url>' | head -n 3
# 预期：HTTP/1.1 200 OK + Content-Type: image/png
```

**⑤ 越权验证：viewer 角色上传 → 403**（seed 里没有 viewer 用户，用 reviewer 试）：

```bash
REVIEWER_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"reviewer","password":"Reviewer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

curl -si -X POST http://127.0.0.1:7001/api/v1/defects/2/image \
  -H "Authorization: Bearer $REVIEWER_TOKEN" -F 'file=@/tmp/defect.png' | head -n 1
# 预期：HTTP/1.1 403 Forbidden（reviewer 没有 defect:upload 权限）
```

---

## 18.4 看板统计模块

### 18.4.1 做什么 / 为什么

车间大屏需要四个聚合接口（IA-SD-005 第 12 节）：总览（良率 / OEE / 今日异常 / 待审批）、良率趋势、缺陷分类、设备 Top。这类多表聚合查询 + 前端 5 秒轮询是数据库杀手，必须套**第 10 章的 cache-aside 封装**（读缓存 → 未命中查库 → 回填带 TTL；写路径失效）。

**OEE 口径说明（教程简化）**：OEE = 时间稼动率 × 性能稼动率 × 良品率。教程数据只有 `wafers` 与 `sensor_readings`，演示口径为：时间稼动率 = 近 24h 有传感器数据的设备小时数占计划运行的比例；性能稼动率固定按演示值 0.92；良品率取近 7 天完工批次平均良率。接入真实 MES/MCS 后按产线实际口径替换 SQL 即可（口径集中在 service 一处，改动不扩散）。

**SQL 书写提醒**：本教程建表沿用原教程的 camelCase 列名（`"yieldRate"`、`"producedAt"` 等），PG 对未加引号的标识符自动折叠为小写——**原生 SQL 里所有 camelCase 列必须双引号**，否则报 `column "yieldrate" does not exist`。

### 18.4.2 Service：app/services/dashboard.py（完整内容）

```python
import json
import logging
from datetime import datetime, timezone

from sqlalchemy import func, select, text
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.config import get_settings
from app.core.redis import get_redis
from app.models.suggestion import Suggestion

logger = logging.getLogger(__name__)

# 缓存键（IA-SD-005 12.1 规定 dash:overview 前缀，教程沿用 ia: 命名空间）
OVERVIEW_KEY = 'ia:stats:dashboard:overview'

# 总览缓存 TTL：与第 10 章看板缓存配置同值（60 秒）
OVERVIEW_TTL_SECONDS = getattr(get_settings(), 'IA_CACHE_DASHBOARD_TTL_SECONDS', 60)


class DashboardService:

    def __init__(self, db: AsyncSession):
        self.db = db

    async def overview(self) -> dict:
        """
        总览（cache-aside，同第 10 章 StatsService 的模式）
        聚合：当前良率 / OEE / 今日异常统计 / 待审批数 + 扩展指标
        """
        redis = get_redis()

        cached = await redis.get(OVERVIEW_KEY)
        if cached:
            return {'source': 'cache', 'data': json.loads(cached)}

        # ---- 1. 近 7 天完工/出货批次平均良率 ----
        yield_row = (await self.db.execute(text(
            """SELECT ROUND(AVG("yieldRate"), 2) AS avg_yield
                 FROM wafers
                WHERE status IN ('completed', 'shipped')
                  AND "producedAt" >= CURRENT_DATE - INTERVAL '7 days'"""
        ))).mappings().first()
        yield_rate = float(yield_row['avg_yield']) if yield_row and yield_row['avg_yield'] is not None else None

        # ---- 2. OEE（教程简化口径，见 18.4.1 说明） ----
        # 2a. 时间稼动率：近 24h 出现过读数的 (设备, 小时) 组合数 / (设备数 × 24)
        avail_row = (await self.db.execute(text(
            """SELECT COUNT(DISTINCT ("equipmentId", date_trunc('hour', "recordedAt"))) AS active_slots,
                      COUNT(DISTINCT "equipmentId") AS equip_count
                 FROM sensor_readings
                WHERE "recordedAt" >= NOW() - INTERVAL '24 hours'"""
        ))).mappings().first()
        equip_count = int(avail_row['equip_count']) if avail_row else 0
        active_slots = int(avail_row['active_slots']) if avail_row else 0
        availability = min(active_slots / (equip_count * 24), 1) if equip_count > 0 else 0
        # 2b. 性能稼动率：演示固定值（接入 MES 后按实际节拍/理论节拍计算）
        performance = 0.92
        # 2c. 良品率
        quality = yield_rate / 100 if yield_rate is not None else 0
        oee = round(availability * performance * quality * 10000) / 100

        # ---- 3. 今日异常统计（按严重度分组；预置三档为 0，保证输出形状稳定） ----
        anomaly_rows = (await self.db.execute(text(
            """SELECT severity, COUNT(*) AS cnt
                 FROM anomaly_events
                WHERE "occurredAt" >= CURRENT_DATE
                GROUP BY severity"""
        ))).mappings().all()
        anomaly_by_severity = {'critical': 0, 'major': 0, 'minor': 0}
        for r in anomaly_rows:
            anomaly_by_severity[r['severity']] = int(r['cnt'])
        anomaly_today = sum(anomaly_by_severity.values())

        # ---- 4. 待审批数 ----
        pending_approvals = await self.db.scalar(
            select(func.count()).select_from(Suggestion).where(Suggestion.status == 'pending'))

        # ---- 5. Top 根因因子（近 30 天任务结果的 JSONB 展开统计） ----
        top_rows = (await self.db.execute(text(
            """SELECT rc->>'factor' AS name, COUNT(*) AS cnt
                 FROM agent_tasks t,
                      jsonb_array_elements(t.result->'root_causes') AS rc
                WHERE t.status = 'succeeded'
                  AND t."createdAt" >= CURRENT_DATE - INTERVAL '30 days'
                GROUP BY rc->>'factor'
                ORDER BY cnt DESC
                LIMIT 5"""
        ))).mappings().all()

        data = {
            'yieldRate': yield_rate,
            'yieldTarget': 95.0,
            'oee': oee,
            'oeeTarget': 85.0,
            'availability': round(availability * 10000) / 100,
            'performance': round(performance * 10000) / 100,
            'anomalyToday': anomaly_today,
            'anomalyBySeverity': anomaly_by_severity,
            'pendingApprovals': pending_approvals or 0,
            'topRootcauses': [{'name': r['name'], 'count': int(r['cnt'])} for r in top_rows],
            'generatedAt': datetime.now(timezone.utc).isoformat(),
        }

        # 回填缓存（带 TTL，即使忘了失效最多脏 60 秒）
        await redis.set(OVERVIEW_KEY, json.dumps(data, ensure_ascii=False), ex=OVERVIEW_TTL_SECONDS)
        return {'source': 'db', 'data': data}

    async def invalidate_overview(self) -> None:
        """总览缓存失效（建议在审批/异常写入路径调用，同第 10 章模式）"""
        await get_redis().delete(OVERVIEW_KEY)

    async def yield_trend(self, days: int) -> list[dict]:
        """良率趋势（近 N 天，按完工批次日聚合；generate_series 补零保证连续日期）"""
        rows = (await self.db.execute(text(
            """SELECT to_char(d.day, 'YYYY-MM-DD') AS date,
                      COALESCE(ROUND(AVG(w."yieldRate"), 2), 0) AS avg_yield,
                      COUNT(w.id) AS batches
                 FROM generate_series(
                        CURRENT_DATE - make_interval(days => :days - 1),
                        CURRENT_DATE,
                        INTERVAL '1 day'
                      ) AS d(day)
                 LEFT JOIN wafers w
                   ON date_trunc('day', w."producedAt") = d.day
                  AND w.status IN ('completed', 'shipped')
                GROUP BY d.day
                ORDER BY d.day"""
        ), {'days': days})).mappings().all()
        return [{
            'date': r['date'],
            'avgYield': float(r['avg_yield']),
            'batches': int(r['batches']),
        } for r in rows]

    async def defect_category(self) -> list[dict]:
        """缺陷分类统计（饼图数据源）"""
        rows = (await self.db.execute(text(
            """SELECT "defectType", COUNT(*) AS records, SUM("defectCount") AS total_defects
                 FROM defect_records
                WHERE "detectedAt" >= CURRENT_DATE - INTERVAL '30 days'
                GROUP BY "defectType"
                ORDER BY total_defects DESC"""
        ))).mappings().all()
        return [{
            'defectType': r['defectType'],
            'records': int(r['records']),
            'totalDefects': int(r['total_defects']),
        } for r in rows]

    async def equipment_top(self, limit: int) -> list[dict]:
        """异常高发设备 Top N（柱图数据源）"""
        rows = (await self.db.execute(text(
            """SELECT equipment,
                      COUNT(*) AS anomaly_count,
                      COUNT(*) FILTER (WHERE severity = 'critical') AS critical_count
                 FROM anomaly_events
                WHERE "occurredAt" >= CURRENT_DATE - INTERVAL '30 days'
                GROUP BY equipment
                ORDER BY anomaly_count DESC, equipment
                LIMIT :limit"""
        ), {'limit': limit})).mappings().all()
        return [{
            'equipment': r['equipment'],
            'anomalyCount': int(r['anomaly_count']),
            'criticalCount': int(r['critical_count']),
        } for r in rows]
```

> 聚合结果里 asyncpg 会把 `numeric/count` 以 `Decimal` 返回，代码里统一 `float()/int()` 转换——这与第 10 章 StatsService 处理 `PG numeric -> string` 的注意事项同源。

### 18.4.3 Router：app/routers/dashboard.py（完整内容）

```python
from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.database import get_db
from app.core.deps import CurrentUser
from app.core.response import success
from app.services.dashboard import DashboardService

router = APIRouter()


@router.get('/api/v1/dashboard/overview')
async def overview(
    db: AsyncSession = Depends(get_db),
    _: CurrentUser,
):
    """GET /api/v1/dashboard/overview"""
    return success(await DashboardService(db).overview())


@router.get('/api/v1/dashboard/yield-trend')
async def yield_trend(
    days: int = Query(14, ge=1, le=90),
    db: AsyncSession = Depends(get_db),
    _: CurrentUser,
):
    """GET /api/v1/dashboard/yield-trend?days=14"""
    return success(await DashboardService(db).yield_trend(days))


@router.get('/api/v1/dashboard/defect-category')
async def defect_category(
    db: AsyncSession = Depends(get_db),
    _: CurrentUser,
):
    """GET /api/v1/dashboard/defect-category"""
    return success(await DashboardService(db).defect_category())


@router.get('/api/v1/dashboard/equipment-top')
async def equipment_top(
    limit: int = Query(5, ge=1, le=20),
    db: AsyncSession = Depends(get_db),
    _: CurrentUser,
):
    """GET /api/v1/dashboard/equipment-top?limit=5"""
    return success(await DashboardService(db).equipment_top(limit))
```

### 18.4.4 路由注册与缓存失效挂钩

`app/main.py` 追加挂载：

```python
    from app.routers import dashboard
    app.include_router(dashboard.router)
```

缓存失效挂钩（同第 10 章做法）：在 `app/services/suggestion.py` 的 `submit_approval` 末尾、以及未来异常事件的写路径中追加：

```python
    # 看板总览缓存失效（18.4）
    await DashboardService(self.db).invalidate_overview()
```

> 与第 10 章 `StatsService.invalidate_dashboard()`（旧 stats 端点）是两个不同的缓存键，互不影响；建议在写路径上两个都失效（各调用一行）。

### 18.4.5 curl 验证

**① 总览（两次调用验证缓存）**

```bash
ADMIN_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"Admin@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

curl -s http://127.0.0.1:7001/api/v1/dashboard/overview -H "Authorization: Bearer $ADMIN_TOKEN"
# 第 1 次预期："source":"db"

curl -s http://127.0.0.1:7001/api/v1/dashboard/overview -H "Authorization: Bearer $ADMIN_TOKEN"
# 第 2 次预期："source":"cache"（60 秒内）
```

预期输出（第 1 次，已格式化；数值取决于种子数据与当天时间）：

```json
{
  "code": 0, "message": "ok",
  "data": {
    "source": "db",
    "data": {
      "yieldRate": 93.85, "yieldTarget": 95.0,
      "oee": 41.66, "oeeTarget": 85.0,
      "availability": 45.24, "performance": 92.0,
      "anomalyToday": 1,
      "anomalyBySeverity": { "critical": 0, "major": 1, "minor": 0 },
      "pendingApprovals": 2,
      "topRootcauses": [ { "name": "etch_power_drift", "count": 1 } ],
      "generatedAt": "2026-10-15T09:00:00.000Z"
    }
  },
  "traceId": "..."
}
```

**② 良率趋势（近 14 天，无数据的日期补零）**

```bash
curl -s 'http://127.0.0.1:7001/api/v1/dashboard/yield-trend?days=14' \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

预期输出（节选，末尾 3 天有种子批次数据）：

```json
{
  "code": 0, "message": "ok",
  "data": [
    { "date": "2026-10-01", "avgYield": 0, "batches": 0 },
    { "date": "2026-10-02", "avgYield": 0, "batches": 0 },
    "...": "...",
    { "date": "2026-10-14", "avgYield": 91.20, "batches": 4 },
    { "date": "2026-10-15", "avgYield": 0, "batches": 0 }
  ],
  "traceId": "..."
}
```

**③ 缺陷分类**

```bash
curl -s http://127.0.0.1:7001/api/v1/dashboard/defect-category \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

预期输出：

```json
{
  "code": 0, "message": "ok",
  "data": [
    { "defectType": "PATTERN", "records": 2, "totalDefects": 7 },
    { "defectType": "PARTICLE", "records": 2, "totalDefects": 6 },
    { "defectType": "SCRATCH", "records": 1, "totalDefects": 3 },
    { "defectType": "RESIDUE", "records": 1, "totalDefects": 1 }
  ],
  "traceId": "..."
}
```

**④ 异常高发设备 Top**

```bash
curl -s 'http://127.0.0.1:7001/api/v1/dashboard/equipment-top?limit=5' \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

预期输出：

```json
{
  "code": 0, "message": "ok",
  "data": [
    { "equipment": "LITH-02", "anomalyCount": 1, "criticalCount": 0 },
    { "equipment": "ETCH-03", "anomalyCount": 1, "criticalCount": 1 },
    { "equipment": "CVD-11", "anomalyCount": 1, "criticalCount": 0 }
  ],
  "traceId": "..."
}
```

---

## 18.5 审计查询模块

### 18.5.1 做什么 / 为什么

第 6 章的审计中间件已经把所有写操作落进 `audit_logs` 表（只插入、不修改、不删除）。现在给管理员一个检索接口：按操作人 / 请求路径 / 时间范围过滤 + 分页——对应 IA-SD-005 第 12.2 节 `GET /audits`（仅 admin）。查询是只读的，不建新表。

### 18.5.2 Service：给 app/services/audit.py 追加 list 方法

在 `AuditService` 类中追加（与 6.2 节的 `record` 并列；同时文件头部补 `from datetime import datetime` 与 `from sqlalchemy import func, select`）：

```python
  _ROW_FIELDS = ('id', 'userId', 'username', 'method', 'path', 'status', 'durationMs',
                 'ip', 'requestSummary', 'errorMessage', 'traceId')

  async def list(self, param: dict) -> dict:
    """审计日志检索（管理员专用）：操作人 / 路径前缀 / 时间范围"""
    conditions = []
    if param.get('username'):
        conditions.append(AuditLog.username.like(f"%{param['username']}%"))
    if param.get('path'):
        conditions.append(AuditLog.path.like(f"{param['path']}%"))   # 前缀匹配：按接口模块过滤
    if param.get('startTime'):
        conditions.append(AuditLog.created_at >= datetime.fromisoformat(param['startTime']))
    if param.get('endTime'):
        conditions.append(AuditLog.created_at <= datetime.fromisoformat(param['endTime']))

    stmt = select(AuditLog).where(*conditions)
    total = await self.db.scalar(select(func.count()).select_from(stmt.subquery()))
    rows = (await self.db.execute(
        stmt.order_by(AuditLog.id.desc())
        .offset((param['page'] - 1) * param['pageSize'])
        .limit(param['pageSize'])
    )).scalars().all()
    return {'rows': [self._to_row(r) for r in rows], 'total': total or 0}

  @staticmethod
  def _to_row(row) -> dict:
    """键名对齐原教程输出：业务字段 camelCase，时间列 created_at"""
    out = {f: getattr(row, f) for f in AuditService._ROW_FIELDS}
    out['created_at'] = row.created_at.isoformat() if row.created_at else None
    return out
```

### 18.5.3 Router：app/routers/audit.py（完整内容）

```python
from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.database import get_db
from app.core.deps import require_perm
from app.core.response import page
from app.services.audit import AuditService

router = APIRouter()


@router.get('/api/v1/audits')
async def list_audits(
    page_no: int = Query(1, alias='page', ge=1),
    page_size: int = Query(10, alias='pageSize', ge=1, le=500),
    username: str | None = Query(None, max_length=64),
    path: str | None = Query(None, max_length=255),
    startTime: str | None = Query(None),
    endTime: str | None = Query(None),
    db: AsyncSession = Depends(get_db),
    user: dict = Depends(require_perm('audit:read')),
):
    """GET /api/v1/audits?username=&path=&startTime=&endTime=（仅 admin）"""
    result = await AuditService(db).list({
        'page': page_no, 'pageSize': page_size,
        'username': username, 'path': path,
        'startTime': startTime, 'endTime': endTime,
    })
    return page(result['rows'], result['total'], page_no, page_size)
```

### 18.5.4 路由注册

`app/main.py` 追加挂载：

```python
    from app.routers import audit
    app.include_router(audit.router)
```

### 18.5.5 curl 验证

**① 管理员按操作人过滤**

```bash
ADMIN_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"Admin@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

curl -s 'http://127.0.0.1:7001/api/v1/audits?username=reviewer&pageSize=3' \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

预期输出（节选）：

```json
{
  "code": 0, "message": "ok",
  "data": {
    "rows": [
      { "id": 42, "userId": 3, "username": "reviewer", "method": "POST",
        "path": "/api/v1/suggestions/1/approve", "status": 200,
        "durationMs": 45, "ip": "::1",
        "requestSummary": "{\"reason\":\"同意回调功率...\"}",
        "errorMessage": null, "traceId": "...", "created_at": "2026-10-15T..." }
    ],
    "total": 6, "page": 1, "pageSize": 3
  },
  "traceId": "..."
}
```

**② 按接口前缀 + 时间范围过滤**

```bash
curl -s 'http://127.0.0.1:7001/api/v1/audits?path=/api/v1/reports&startTime=2026-10-15T00:00:00%2B08:00&endTime=2026-10-16T00:00:00%2B08:00' \
  -H "Authorization: Bearer $ADMIN_TOKEN"
# 预期：只含 /api/v1/reports 开头的记录（编辑 / 签发 / 显式业务审计）
```

**③ 非 admin 访问 → 403**

```bash
ENGINEER_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"engineer","password":"Engineer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

curl -si 'http://127.0.0.1:7001/api/v1/audits' -H "Authorization: Bearer $ENGINEER_TOKEN" | head -n 1
# 预期：HTTP/1.1 403 Forbidden（audit:read 仅 admin 的 * 覆盖）
```

---

## 18.6 用户管理模块

### 18.6.1 做什么 / 为什么

第 7 章的用户只能靠 seed 脚本造。真实系统里管理员需要：分页查用户、创建用户（bcrypt 加密）、启用/禁用（禁用即踢出登录）、改绑角色（RBAC 的运营入口）、重置密码。对应 IA-SD-005 第 5 节的用户四接口 + 教程补充的密码重置。**全部走 admin 权限**（`user:*` 权限点不分配给任何角色，仅 admin 的 `*` 覆盖）。

### 18.6.2 Service：app/services/user.py（完整内容）

```python
import logging

from sqlalchemy import delete, func, or_, select
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.biz_error import BizError
from app.core.error_codes import ErrorCode
from app.core.security import hash_password
from app.models.role import Role
from app.models.user import User
from app.models.user_role import UserRole

logger = logging.getLogger(__name__)

# 用户表敏感列：任何查询都不允许返回
SAFE_FIELDS = ('id', 'username', 'realName', 'email', 'status', 'lastLoginAt', 'createdAt', 'updatedAt')


class UserService:

    def __init__(self, db: AsyncSession):
        self.db = db

    async def list(self, param: dict) -> dict:
        """用户分页列表（含角色；keyword 模糊匹配用户名/姓名）"""
        stmt = select(User)
        if param.get('keyword'):
            kw = f"%{param['keyword']}%"
            stmt = stmt.where(or_(User.username.like(kw), User.realName.like(kw)))
        if param.get('status'):
            stmt = stmt.where(User.status == param['status'])

        total = await self.db.scalar(select(func.count()).select_from(stmt.subquery()))
        users = (await self.db.execute(
            stmt.order_by(User.id.asc())
            .offset((param['page'] - 1) * param['pageSize'])
            .limit(param['pageSize'])
        )).scalars().all()

        # 批量补角色（两条查询，避免 N+1）
        user_ids = [u.id for u in users]
        roles_map = await self._get_roles_map(user_ids)
        rows = [self._safe_dict(u, roles_map.get(u.id, [])) for u in users]
        return {'rows': rows, 'total': total or 0}

    async def create(self, param: dict) -> dict:
        """创建用户（建用户 + 绑角色；密码 bcrypt 哈希，永不明文）"""
        exists = (await self.db.execute(
            select(User).where(User.username == param['username']))).scalar_one_or_none()
        if exists:
            raise BizError(ErrorCode.DUPLICATE, f"用户名 {param['username']} 已存在")

        # 先校验角色码再落库：非法角色码不会留下半截用户
        roles = await self._validate_roles(param['roleCodes'])

        user = User(
            username=param['username'],
            passwordHash=hash_password(param['password']),   # bcrypt cost 10（第 7 章同款）
            realName=param.get('realName'),
            email=param.get('email'),
            status='active',
        )
        self.db.add(user)
        await self.db.flush()   # 拿自增 id，稍后统一 commit
        self._bind_roles(user.id, roles)
        await self.db.commit()
        await self.db.refresh(user)
        return self._safe_dict(user, param['roleCodes'])

    async def update(self, user_id: int, param: dict, operator_id: int) -> dict:
        """更新用户（资料与启用/禁用；不许改用户名）"""
        user = await self._get_or_404(user_id)

        if param.get('status') == 'disabled' and user_id == operator_id:
            raise BizError(ErrorCode.STATE_CONFLICT, '不能禁用当前登录的自己')

        if param.get('realName') is not None:
            user.realName = param['realName']
        if param.get('email') is not None:
            user.email = param['email']
        if param.get('status') is not None:
            user.status = param['status']
        await self.db.commit()
        await self.db.refresh(user)
        return self._safe_dict(user)

    async def update_roles(self, user_id: int, role_codes: list[str]) -> dict:
        """改绑角色（删旧全量重插；校验角色码全部合法）"""
        user = await self._get_or_404(user_id)

        # 校验角色码全部合法（IA-SD-005 5.2：IA-11002 角色码非法）
        roles = await self._validate_roles(role_codes)

        await self.db.execute(delete(UserRole).where(UserRole.userId == user_id))
        self._bind_roles(user_id, roles)
        await self.db.commit()
        return {'userId': user_id, 'roles': role_codes}

    async def reset_password(self, user_id: int, new_password: str) -> dict:
        """重置密码（bcrypt 新哈希）"""
        user = await self._get_or_404(user_id)
        user.passwordHash = hash_password(new_password)
        await self.db.commit()
        return {'id': user_id, 'passwordReset': True}

    # ---------- 内部方法 ----------

    def _bind_roles(self, user_id: int, roles) -> None:
        for role in roles:
            self.db.add(UserRole(userId=user_id, roleId=role.id))

    async def _validate_roles(self, role_codes: list[str]) -> list[Role]:
        """角色码全量校验：一个不在 roles 表就整体 422（10001）"""
        role_codes = role_codes or []
        roles = (await self.db.execute(
            select(Role).where(Role.code.in_(role_codes)))).scalars().all() if role_codes else []
        if len(roles) != len(set(role_codes)):
            found = {r.code for r in roles}
            missing = [c for c in dict.fromkeys(role_codes) if c not in found]
            raise BizError(ErrorCode.PARAM_INVALID, f"角色码非法：{','.join(missing)}", http_status=422)
        return list(roles)

    async def _get_roles_map(self, user_ids: list[int]) -> dict[int, list[str]]:
        """userId -> 角色码数组 的批量映射（两条查询）"""
        if not user_ids:
            return {}
        user_roles = (await self.db.execute(
            select(UserRole).where(UserRole.userId.in_(user_ids)))).scalars().all()
        if not user_roles:
            return {}
        role_ids = [ur.roleId for ur in user_roles]
        roles = (await self.db.execute(select(Role).where(Role.id.in_(role_ids)))).scalars().all()
        role_id_to_code = {r.id: r.code for r in roles}
        mapping: dict[int, list[str]] = {}
        for ur in user_roles:
            code = role_id_to_code.get(ur.roleId)
            if code:
                mapping.setdefault(ur.userId, []).append(code)
        return mapping

    async def _get_or_404(self, user_id: int) -> User:
        user = await self.db.get(User, user_id)
        if not user:
            raise BizError(ErrorCode.NOT_FOUND, f'用户不存在（id={user_id}）', http_status=404)
        return user

    @staticmethod
    def _safe_dict(user: User, roles: list[str] | None = None) -> dict:
        out: dict = {}
        for f in SAFE_FIELDS:
            v = getattr(user, f)
            out[f] = v.isoformat() if hasattr(v, 'isoformat') else v
        if roles is not None:
            out['roles'] = roles
        return out
```

> `create` 里"先校验角色码、再落库"的顺序让非法角色码不会留下半截用户；`_validate_roles` 对重复角色码先去重再比对，`"viewer","viewer"` 不会误报非法。

### 18.6.3 请求 Schema：app/schemas/user.py（完整内容）

```python
from typing import Literal

from pydantic import BaseModel, EmailStr, Field


class UserCreateIn(BaseModel):
    username: str = Field(min_length=3, max_length=64)
    password: str = Field(min_length=8, max_length=64)
    realName: str | None = Field(None, max_length=64)
    email: EmailStr | None = None
    roleCodes: list[str] = Field(min_length=1, max_length=5)


class UserUpdateIn(BaseModel):
    realName: str | None = Field(None, max_length=64)
    email: EmailStr | None = None
    status: Literal['active', 'disabled'] | None = None


class UserRolesIn(BaseModel):
    roleCodes: list[str] = Field(max_length=5)   # 允许空数组（清空全部角色）


class PasswordResetIn(BaseModel):
    password: str = Field(min_length=8, max_length=64)
```

### 18.6.4 Router：app/routers/user.py（完整内容）

```python
from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.database import get_db
from app.core.deps import require_perm
from app.core.response import page, success
from app.schemas.user import PasswordResetIn, UserCreateIn, UserRolesIn, UserUpdateIn
from app.services.user import UserService

router = APIRouter()


@router.get('/api/v1/users')
async def list_users(
    page_no: int = Query(1, alias='page', ge=1),
    page_size: int = Query(10, alias='pageSize', ge=1, le=500),
    keyword: str | None = Query(None, max_length=64),
    status: str | None = Query(None, pattern='^(active|disabled)$'),
    db: AsyncSession = Depends(get_db),
    user: dict = Depends(require_perm('user:read')),
):
    """GET /api/v1/users?keyword=&status="""
    result = await UserService(db).list({
        'page': page_no, 'pageSize': page_size, 'keyword': keyword, 'status': status,
    })
    return page(result['rows'], result['total'], page_no, page_size)


@router.post('/api/v1/users')
async def create_user(
    body: UserCreateIn,
    db: AsyncSession = Depends(get_db),
    user: dict = Depends(require_perm('user:create')),
):
    """POST /api/v1/users（管理员创建）"""
    created = await UserService(db).create(body.model_dump())
    return success(created, '创建成功', status_code=201)


@router.put('/api/v1/users/{user_id}')
async def update_user(
    user_id: int,
    body: UserUpdateIn,
    db: AsyncSession = Depends(get_db),
    user: dict = Depends(require_perm('user:update')),
):
    """PUT /api/v1/users/:id（资料 + 启用禁用）"""
    result = await UserService(db).update(user_id, body.model_dump(exclude_unset=True), user['userId'])
    return success(result, '更新成功')


@router.put('/api/v1/users/{user_id}/roles')
async def update_roles(
    user_id: int,
    body: UserRolesIn,
    db: AsyncSession = Depends(get_db),
    user: dict = Depends(require_perm('user:update')),
):
    """PUT /api/v1/users/:id/roles（改绑角色）"""
    result = await UserService(db).update_roles(user_id, body.roleCodes)
    return success(result, '角色已更新')


@router.put('/api/v1/users/{user_id}/password')
async def reset_password(
    user_id: int,
    body: PasswordResetIn,
    db: AsyncSession = Depends(get_db),
    user: dict = Depends(require_perm('user:reset_password')),
):
    """PUT /api/v1/users/:id/password（重置密码）"""
    result = await UserService(db).reset_password(user_id, body.password)
    return success(result, '密码已重置')
```

### 18.6.5 路由注册

`app/main.py` 追加挂载：

```python
    from app.routers import user
    app.include_router(user.router)
```

### 18.6.6 curl 验证

**① 管理员创建用户（带角色）**

```bash
ADMIN_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"Admin@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

curl -s -X POST http://127.0.0.1:7001/api/v1/users \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  -d '{"username":"viewer01","password":"Viewer@123","realName":"赵参观","roleCodes":["viewer"]}'
```

预期输出（HTTP 201；注意返回中没有 passwordHash）：

```json
{
  "code": 0, "message": "创建成功",
  "data": { "id": 4, "username": "viewer01", "realName": "赵参观", "email": null,
             "status": "active", "roles": [ "viewer" ], "...": "..." },
  "traceId": "..."
}
```

**② 分页 + 关键字过滤（结果带角色，无密码哈希）**

```bash
curl -s 'http://127.0.0.1:7001/api/v1/users?keyword=李' -H "Authorization: Bearer $ADMIN_TOKEN"
# 预期：total 1，rows[0].username = "engineer"，rows[0].roles = ["process_engineer"]，
#       且任何字段都不含 passwordHash
```

**③ 改绑角色（给 viewer01 加 process_engineer）**

```bash
curl -s -X PUT http://127.0.0.1:7001/api/v1/users/4/roles \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  -d '{"roleCodes":["viewer","process_engineer"]}'
# 预期：{"code":0,"message":"角色已更新","data":{"userId":4,"roles":["viewer","process_engineer"]},...}

# 非法角色码 -> 422
curl -s -X PUT http://127.0.0.1:7001/api/v1/users/4/roles \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  -d '{"roleCodes":["hacker"]}'
# 预期：HTTP 422，code 10001，message 含 "角色码非法：hacker"
```

**④ 禁用用户并验证登录被拒**

```bash
curl -s -X PUT http://127.0.0.1:7001/api/v1/users/4 \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  -d '{"status":"disabled"}'
# 预期：code 0，message "更新成功"，data.status = "disabled"

curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"viewer01","password":"Viewer@123"}'
# 预期：{"code":20005,"message":"账号已被禁用，请联系管理员",...}（HTTP 403）

# 管理员不能禁用自己 -> 400
curl -s -X PUT http://127.0.0.1:7001/api/v1/users/1 \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  -d '{"status":"disabled"}'
# 预期：{"code":10004,"message":"不能禁用当前登录的自己",...}
```

**⑤ 重置密码**

```bash
curl -s -X PUT http://127.0.0.1:7001/api/v1/users/4/password \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  -d '{"password":"NewPass@123"}'
# 预期：{"code":0,"message":"密码已重置","data":{"id":4,"passwordReset":true},...}

# 先解禁再用新密码登录验证
curl -s -X PUT http://127.0.0.1:7001/api/v1/users/4 \
  -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' \
  -d '{"status":"active"}' > /dev/null
curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"viewer01","password":"NewPass@123"}' | head -c 80
# 预期：{"code":0,"message":"ok","data":{"accessToken":"...
```

**⑥ 非 admin 访问 → 403**

```bash
ENGINEER_TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"engineer","password":"Engineer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

curl -si 'http://127.0.0.1:7001/api/v1/users' -H "Authorization: Bearer $ENGINEER_TOKEN" | head -n 1
# 预期：HTTP/1.1 403 Forbidden（user:read 仅 admin）
```

---

## 18.7 模块补全收官：全量路由总表与冒烟验证

### 18.7.1 建表脚本补全

`scripts/init_db.py` 中追加两个模型的 import（无需种子数据，仅为 `Base.metadata.create_all` 能建表；缺陷模型已在 18.3.7 加入）：

```python
  # ---------- 18 章新表（仅建表，无种子） ----------
  from app.models.report_8d import Report8D        # noqa: F401
  from app.models.notification import Notification  # noqa: F401
  print('[seed] reports_8d / notifications 表结构同步完成')
```

> 保险起见，也可把这两个 import 一并写进 `app/models/__init__.py`（教程主线的模型汇总处），这样无论从哪个入口触发 `create_all` 都不会漏表。

重新执行 `python scripts/init_db.py`（会清库重建，重跑一遍 9.6 审批剧本让 18 章数据就位），重启 `python -m app.run`。

### 18.7.2 全量路由总表（截至本章，43 个端点）

| # | 方法 | 路径 | 权限 | 所属模块 | 实现章节 |
|---|---|---|---|---|---|
| 1 | GET | / | 公开 | 平台信息 | 3 |
| 2 | GET | /healthz | 公开 | 运维探活 | 14 |
| 3 | GET | /metrics | 公开 | Prometheus 指标 | 14 |
| 4 | POST | /api/v1/auth/login | 公开 | 认证 | 7 |
| 5 | POST | /api/v1/auth/refresh | 公开 | 认证 | 7 |
| 6 | POST | /api/v1/auth/logout | 登录 | 认证 | 7 |
| 7 | GET | /api/v1/auth/profile | 登录 | 认证 | 7 |
| 8 | GET | /api/v1/wafers | 登录 | 晶圆批次 | 5 |
| 9 | GET | /api/v1/wafers/:id | 登录 | 晶圆批次 | 5 |
| 10 | POST | /api/v1/wafers | wafer:create | 晶圆批次 | 5 |
| 11 | PUT | /api/v1/wafers/:id | wafer:update | 晶圆批次 | 5 |
| 12 | DELETE | /api/v1/wafers/:id | wafer:delete | 晶圆批次 | 5 |
| 13 | GET | /api/v1/anomaly-events | 登录 | 异常事件 | 8 |
| 14 | GET | /api/v1/anomaly-events/:id | 登录 | 异常事件 | 8 |
| 15 | POST | /api/v1/rootcause/analyze | rootcause:trigger | 根因分析 | 8 |
| 16 | GET | /api/v1/rootcause/tasks | 登录 | 根因分析 | 8 |
| 17 | GET | /api/v1/rootcause/tasks/:id | 登录 | 根因分析 | 8 |
| 18 | GET | /api/v1/suggestions | 登录 | 建议审批 | 9 |
| 19 | POST | /api/v1/suggestions/:id/approve | suggestion:approve | 建议审批 | 9 |
| 20 | POST | /api/v1/suggestions/:id/reject | suggestion:approve | 建议审批 | 9 |
| 21 | GET | /api/v1/suggestions/:id/approvals | 登录 | 建议审批 | 9 |
| 22 | GET | /api/v1/stats/dashboard | 登录 | 看板统计(旧) | 10 |
| 23 | GET | /api/v1/reports | 登录 | 8D 报告 | 18.1 |
| 24 | GET | /api/v1/reports/:id | 登录 | 8D 报告 | 18.1 |
| 25 | PUT | /api/v1/reports/:id | report:edit | 8D 报告 | 18.1 |
| 26 | POST | /api/v1/reports/:id/release | report:release | 8D 报告 | 18.1 |
| 27 | GET | /api/v1/notifications/unread-count | 登录 | 通知 | 18.2 |
| 28 | GET | /api/v1/notifications | 登录 | 通知 | 18.2 |
| 29 | PUT | /api/v1/notifications/read-all | 登录 | 通知 | 18.2 |
| 30 | PUT | /api/v1/notifications/:id/read | 登录 | 通知 | 18.2 |
| 31 | GET | /api/v1/defects | 登录 | 缺陷 | 18.3 |
| 32 | GET | /api/v1/defects/:id/image-url | 登录 | 缺陷 | 18.3 |
| 33 | POST | /api/v1/defects/:id/image | defect:upload | 缺陷 | 18.3 |
| 34 | GET | /api/v1/dashboard/overview | 登录 | 看板统计 | 18.4 |
| 35 | GET | /api/v1/dashboard/yield-trend | 登录 | 看板统计 | 18.4 |
| 36 | GET | /api/v1/dashboard/defect-category | 登录 | 看板统计 | 18.4 |
| 37 | GET | /api/v1/dashboard/equipment-top | 登录 | 看板统计 | 18.4 |
| 38 | GET | /api/v1/audits | audit:read | 审计查询 | 18.5 |
| 39 | GET | /api/v1/users | user:read | 用户管理 | 18.6 |
| 40 | POST | /api/v1/users | user:create | 用户管理 | 18.6 |
| 41 | PUT | /api/v1/users/:id | user:update | 用户管理 | 18.6 |
| 42 | PUT | /api/v1/users/:id/roles | user:update | 用户管理 | 18.6 |
| 43 | PUT | /api/v1/users/:id/password | user:reset_password | 用户管理 | 18.6 |

另有 WS 推送端点 `ws://<host>:7002/?token=<JWT>`（事件：`anomaly.push` / `notification.push`，见 18.2）。

### 18.7.3 与 IA-SD-005（34 个 REST 接口）的对照与差异说明

| IA-SD-005 接口 | 教程实现 | 差异说明 |
|---|---|---|
| #1~4 认证（login/refresh/logout/profile） | #4~7 | 一致；教程 JWT 为 HS256 自签（第 19 章升级 Keycloak RS256） |
| #5~8 用户（list/create/update/roles） | #39~42 | 一致；教程补充 #43 密码重置（IA-SD-005 未列） |
| #9 GET /wafers、#10 /wafers/:barcode | #8~9 | 教程批次模型合并 wafer/lot（`batchNo` 即批次维度）；路径参数用整型 `:id` 而非 barcode |
| #11 GET /lots | #8 | 同上，由 /wafers 的批次字段覆盖 |
| #12~13 缺陷（list/image-url） | #31~32 | 一致；预签名有效期 7 天（教程质检复盘口径）vs IA-SD-005 示例 300s |
| #14 GET /defects/stat | #36 | 由 /dashboard/defect-category 覆盖同能力（聚合维度更全） |
| #15~16 异常（/anomalies） | #13~14 | 资源命名 `anomaly-events` vs `anomalies`；详情暂不含任务链展开 |
| #17 POST /agent-tasks | #15 | 教程为 `POST /rootcause/analyze`，同步等待返回（约 2s）；IA-SD-005 为异步受理 |
| #18~19 agent-tasks 列表/详情 | #16~17 | 一致（路径命名不同） |
| #20 retry / #21 takeover | — | **未实现**，属智能体教程联动特性（依赖异步任务队列），列为后续迭代 |
| #22 GET /rootcause-results/:taskId | #17 | 任务详情内嵌 `result` 字段（Top-3 + 证据链） |
| #23~24 建议 list/详情 | #18 | 一致；详情由列表行展开（教程建议字段较少） |
| #25 GET /approvals | #21 | 由 /suggestions/:id/approvals 覆盖（按建议查审批历史） |
| #26 POST /approvals | #19~20 | 教程拆为 approve/reject 两个语义化端点 |
| #27 approvals/stat | — | **未实现**（审批 SLA 统计），可基于 approvals 表扩展 |
| #28~30 报告（list/详情/release） | #23~26 | 一致；教程补充 PUT 编辑接口（IA-SD-005 未列但业务必需） |
| #31~32 看板（overview/yield-trend） | #34~35 | 一致；教程额外提供 defect-category / equipment-top（#36~37） |
| #33 GET /audits | #38 | 一致 |
| #34 GET /notifications | #27~30 | 教程拆为列表/未读数/已读标记四个接口（IA-SD-005 的 sinceEventId 断线补偿由前端结合通知列表补偿拉取实现） |

**约定级差异（以本教程为准）**：

1. **分页参数**：教程统一 `page` / `pageSize`（IA-SD-005 写作 `pageNum`）；分页响应为 `data: { rows, total, page, pageSize }`（IA-SD-005 为 `{ list, total, pageNum, pageSize }`）。若需严格对齐 IA-SD-005，只需改 `page()` 响应助手一处 + Router 取参处。
2. **错误码**：教程用整型段（10xxx/20xxx/30xxx/50000，见 5.3 节）；IA-SD-005 用 `IA-1xxxx` 字符串段。映射关系一一对应（如 10002↔IA-11001、20001↔IA-10401），网关层可无损转换。
3. **统一响应**：教程成功 message 为 `ok`（IA-SD-005 为 `success`），且响应体带 `traceId`（教程增强项）。
4. **JWT 签名**：HS256 自签（教学简化）→ 第 19 章给出 Keycloak RS256 完整升级方案。

结论：**IA-SD-005 的 34 个接口中 32 个已实现（含等价替代），2 个（retry/takeover、approvals/stat）列为后续迭代**；教程另补充 9 个实用端点（编辑报告、通知四件套、密码重置、看板两扩展、旧 stats 端点）。

### 18.7.4 一键冒烟脚本

`scripts/smoke-test.sh`（完整内容；Git Bash 执行 `bash scripts/smoke-test.sh`）。脚本顺序调用 18 个接口，登录提取 token，逐个断言 `code:0`：

```bash
#!/usr/bin/env bash
# industryagent-server 冒烟测试：跑通前先确保 PG/Redis/MinIO/Python Mock/server 都在跑，
# 且已执行过 python scripts/init_db.py + 9.6 节审批剧本（保证有 pending 建议与生成的报告）。
set -u
BASE=http://127.0.0.1:7001
PASS=0; FAIL=0

# 断言函数：响应体中含 "code":0 即通过
check() {
  local name="$1"; local body="$2"
  if echo "$body" | grep -q '"code":0'; then
    PASS=$((PASS+1)); echo "  ✔ $name"
  else
    FAIL=$((FAIL+1)); echo "  ✘ $name -> $body"
  fi
}

echo "== 0. 平台与探活 =="
check "GET /"                    "$(curl -s $BASE/)"
check "GET /healthz"             "$(curl -s $BASE/healthz)"

echo "== 1. 登录取 token =="
ADMIN_TOKEN=$(curl -s -X POST $BASE/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"Admin@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)
ENGINEER_TOKEN=$(curl -s -X POST $BASE/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"engineer","password":"Engineer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)
REVIEWER_TOKEN=$(curl -s -X POST $BASE/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"reviewer","password":"Reviewer@123"}' | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)
[ -n "$ADMIN_TOKEN" ] && { PASS=$((PASS+1)); echo "  ✔ 三账号登录"; } || { FAIL=$((FAIL+1)); echo "  ✘ 登录失败"; }

AH="Authorization: Bearer $ADMIN_TOKEN"
EH="Authorization: Bearer $ENGINEER_TOKEN"
RH="Authorization: Bearer $REVIEWER_TOKEN"

echo "== 2. 认证与批次 =="
check "GET /auth/profile"        "$(curl -s $BASE/api/v1/auth/profile -H "$AH")"
check "GET /wafers"              "$(curl -s "$BASE/api/v1/wafers?page=1&pageSize=2" -H "$AH")"
check "GET /wafers/1"            "$(curl -s $BASE/api/v1/wafers/1 -H "$AH")"

echo "== 3. 异常与根因 =="
check "GET /anomaly-events"      "$(curl -s "$BASE/api/v1/anomaly-events?pageSize=2" -H "$AH")"
check "GET /rootcause/tasks"     "$(curl -s $BASE/api/v1/rootcause/tasks -H "$AH")"

echo "== 4. 建议 =="
check "GET /suggestions"         "$(curl -s "$BASE/api/v1/suggestions?pageSize=2" -H "$AH")"

echo "== 5. 8D 报告 =="
check "GET /reports"             "$(curl -s "$BASE/api/v1/reports?status=draft" -H "$AH")"
check "GET /reports/1"           "$(curl -s $BASE/api/v1/reports/1 -H "$AH")"

echo "== 6. 通知 =="
check "GET /notifications/unread-count" "$(curl -s $BASE/api/v1/notifications/unread-count -H "$EH")"
check "GET /notifications"       "$(curl -s "$BASE/api/v1/notifications?pageSize=3" -H "$EH")"

echo "== 7. 缺陷 =="
check "GET /defects"             "$(curl -s "$BASE/api/v1/defects?pageSize=2" -H "$AH")"

echo "== 8. 看板 =="
check "GET /dashboard/overview"  "$(curl -s $BASE/api/v1/dashboard/overview -H "$AH")"
check "GET /dashboard/yield-trend" "$(curl -s "$BASE/api/v1/dashboard/yield-trend?days=7" -H "$AH")"
check "GET /dashboard/defect-category" "$(curl -s $BASE/api/v1/dashboard/defect-category -H "$AH")"
check "GET /dashboard/equipment-top"  "$(curl -s "$BASE/api/v1/dashboard/equipment-top?limit=5" -H "$AH")"

echo "== 9. 审计与用户（admin） =="
check "GET /audits"              "$(curl -s "$BASE/api/v1/audits?pageSize=2" -H "$AH")"
check "GET /users"               "$(curl -s "$BASE/api/v1/users?pageSize=2" -H "$AH")"

echo ""
echo "冒烟结果：通过 $PASS，失败 $FAIL"
[ "$FAIL" -eq 0 ] && echo "全部通过 ✔" || exit 1
```

预期输出：

```text
== 0. 平台与探活 ==
  ✔ GET /
  ✔ GET /healthz
== 1. 登录取 token ==
  ✔ 三账号登录
== 2. 认证与批次 ==
  ✔ GET /auth/profile
  ...
== 9. 审计与用户（admin） ==
  ✔ GET /audits
  ✔ GET /users

冒烟结果：通过 21，失败 0
全部通过 ✔
```

> **Postman 用户**：按 18.7.2 总表逐条建请求，Collection 变量存 `{{token}}`，登录请求的 Tests 脚本里 `pm.collectionVariables.set('token', pm.response.json().data.accessToken)` 即可实现同样的链式冒烟。

---

## 18.8 常见坑

**坑 1：WS 端口 7002 在 Windows 上连不上（防火墙拦截）**

- 现象：`websockets.connect` 报 `ConnectionRefusedError` / 长时间无响应，而 `curl 127.0.0.1:7001` 正常。
- 排查：先 `netstat -ano | findstr 7002` 确认监听存在（应有 `0.0.0.0:7002`）；再查 Windows Defender 防火墙——首次启动 `python.exe` 时弹的"允许访问"若点了取消，入站就被拦。
- 修复：管理员 PowerShell 执行 `New-NetFirewallRule -DisplayName "ia-ws-7002" -Direction Inbound -LocalPort 7002 -Protocol TCP -Action Allow`；本机回环（127.0.0.1）通常不受影响，**局域网另一台机器连不上时几乎都是这个原因**。Linux 服务器上对应 `firewall-cmd --add-port=7002/tcp --permanent` 或安全组放行。

**坑 2：预签名 URL 报 `RequestTimeTooSkewed` / `AccessDenied`（时钟偏移）**

- 原因：S3 V4 签名带时间戳，MinIO 校验请求时间与签名时间的偏差（默认容忍约 15 分钟）。后端机器或访问浏览器所在机器时钟漂移过大，签名即失效。
- 排查：`docker exec ia-minio mc admin info local` 看 MinIO 时间；宿主机 `w32tm /query /status`（Windows）或 `timedatectl`（Linux）对时。
- 修复：开启 NTP 自动对时。**容器尤其注意**：宿主机时间对，容器才对；`docker restart ia-minio` 让容器重新同步宿主机时钟。另外确认 URL 是"原样复制"——curl 里含 `&` 的 URL 不加单引号会被 shell 拆断，拿到 403/SignatureDoesNotMatch。

**坑 3：MinIO 桶策略——预签名 GET 仍 403**

- `presigned_get_object` 签的 URL 无需桶公共读权限即可访问，**但对象必须存在**：对不存在的 key 访问会得到 403/404（MinIO 对私有桶不泄露对象存在性）。
- 若把桶改成了公共读（`mc anonymous set download local/defect-images`），预签名反而成为多余但无害的双保险；**不要**反向把 AK/SK 写进前端换取"方便直传"——那是把对象存储的钥匙交了出去，违背"数据不出域"设计。
- 桶名含大写字母或下划线会导致 S3 兼容路径签名问题：教程固定小写 `defect-images`，自定义桶名时保持 DNS 合法（小写、数字、连字符）。

**坑 4：uvicorn 双 Server 的优雅关闭顺序**

- 信号处理器必须只装一份（run.py 的 `QuietServer` 子类）：两个 `Server.serve()` 都默认 `install_signal_handlers()`，后者覆盖前者，导致 Ctrl+C 只关一个 Server、另一个成为孤儿进程占着端口。
- 关闭顺序先 REST 后 WS：REST `should_exit=True` 后 uvicorn 会停收新连接并等待存量请求完成（默认宽限期内），再关 WS 才不会出现"前端还在发请求、推送通道已断"的窗口。教程里 `await asyncio.sleep(1)` 是给 REST 留的排水时间，生产可换成轮询 `rest.servers` 直到空连接再关 WS。
- Windows 上 `Ctrl+C` 触发的是 `KeyboardInterrupt` 而非 `add_signal_handler`（ProactorEventLoop 不支持），run.py 已做双路径兼容；若你用 `--reload` 或多 worker，请回到单 Server 模式分别部署 WS 与 REST——双 Server 启动器与 reload/多进程互斥。

# 19. 进阶：Keycloak 真实对接

## 19.1 为什么教程先用自签 JWT、何时切换 Keycloak

**为什么第 7 章先用自签 JWT（HS256）**：

1. **教学聚焦**：第 7 章的核心是"JWT 双 token + Redis 白名单 + RBAC"这套**机制**，HS256 一个密钥字符串就能跑通全流程，不引入任何外部系统；
2. **零外部依赖**：跟着教程敲到第 7 章时，Keycloak 还不存在于你的环境里——先跑通再演进，是工程学习正道；
3. **HS256 的局限恰好是学习点**：对称密钥意味着"谁能验证 token，谁就能伪造 token"。单后端没问题；一旦有多个服务（前端 BFF、MCP 网关、Python 智能体服务）都要验 token，就得把密钥分发到每个服务——爆炸半径不可控。

**什么时候切换 Keycloak（IA-SD-005 2.1 节的规定形态）**：

| 触发条件 | 说明 |
|---|---|
| 多系统需要统一身份 | 前端管理台、移动端、智能体服务都要验同一套 token |
| RS256 非对称验签需求 | 私钥只在 Keycloak，各服务只持公钥——只能验、不能造 |
| 标准化用户生命周期 | 注册 / 改密 / 强制登出 / 密码策略 / 账号锁定，自己写一遍至少三周 |
| 单点登录（SSO） | 多个内部系统一次登录全通行 |
| 合规审计 | 登录事件、token 签发记录集中留痕 |

切换后**架构收益**：认证（Authentication）外包给 Keycloak，后端只做授权（Authorization）——权限仍走第 7 章的本地 RBAC 表，两者职责分明（这是本方案的关键设计）。

## 19.2 部署 Keycloak 24 并完成配置

### 19.2.1 Docker 启动

```bash
docker run -d --name ia-keycloak \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin123 \
  quay.io/keycloak/keycloak:24.0 start-dev
```

> `start-dev` 是开发模式（HTTPS 强制、启动加速等生产检查全部关闭）；生产用 `start`（需配置 TLS）。预期输出一长串容器 ID。

验证：

```bash
curl -s http://127.0.0.1:8080/realms/master/.well-known/openid-configuration | head -c 120
# 预期输出：{"issuer":"http://127.0.0.1:8080/realms/master",...}
```

### 19.2.2 建 realm / client / 测试用户（控制台操作步骤）

浏览器打开 `http://127.0.0.1:8080`，右上角 **Administration Console** 登录（admin / admin123）：

1. **建 realm**：左上角下拉 → *Create Realm* → Realm name 填 `ia` → Create；
2. **建 client**：`ia` realm → Clients → *Create client*：
   - Client ID：`industryagent-server`
   - Client authentication：**On**（confidential 类型，后端代理登录需要 client secret）
   - 直接点 Next → 勾选 **Direct access grants**（允许密码直换 token，即 ROPC 流程）→ Save；
   - 创建后进入 *Credentials* 页签，复制 **Client secret**（后面写入 .env）；
3. **建测试用户**：Users → *Add user*：
   - Username `engineer`，First name 随意 → Create → *Credentials* 页签 → Set password：`Engineer@123`，**Temporary 关掉**（否则首次登录强制改密）；
   - 同样方式再建 `reviewer` / `Reviewer@123`（用户名与本地 users 表种子数据一致，这是 19.3 节"claims → 本地用户同步"能对上的前提）。

验证密码直换 token（ROPC）：

```bash
KC_CLIENT_SECRET='第 2 步复制的 secret'
curl -s -X POST 'http://127.0.0.1:8080/realms/ia/protocol/openid-connect/token' \
  -d "grant_type=password" \
  -d "client_id=industryagent-server" \
  -d "client_secret=$KC_CLIENT_SECRET" \
  -d "username=engineer" \
  -d "password=Engineer@123" | head -c 200
# 预期输出：{"access_token":"eyJhbGciOiJSUzI1NiIs...",...（RS256 签发的 access_token）}
```

### 19.2.3 realm-json 导出：配置即代码

控制台手点只适合第一次探索。**生产与团队协作的正确姿势**是把 realm 配置导出为 JSON 纳入 git（配置即代码，同第 4 章"代码与环境隔离"的精神），新环境 `--import-realm` 一键重建。

`ia-realm-export.json`（精简版完整内容，与 19.2.2 等价，存放在项目根 `keycloak/` 目录）：

```json
{
  "realm": "ia",
  "enabled": true,
  "registrationAllowed": false,
  "clients": [
    {
      "clientId": "industryagent-server",
      "enabled": true,
      "protocol": "openid-connect",
      "publicClient": false,
      "secret": "industryagent-server-secret-change-me",
      "directAccessGrantsEnabled": true,
      "redirectUris": [ "http://127.0.0.1:3000/*" ],
      "webOrigins": [ "+" ]
    }
  ],
  "users": [
    {
      "username": "engineer",
      "enabled": true,
      "credentials": [ { "type": "password", "value": "Engineer@123", "temporary": false } ]
    },
    {
      "username": "reviewer",
      "enabled": true,
      "credentials": [ { "type": "password", "value": "Reviewer@123", "temporary": false } ]
    }
  ]
}
```

用挂载导入方式启动（替代 19.2.1 的裸启动）：

```bash
docker rm -f ia-keycloak
docker run -d --name ia-keycloak \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin123 \
  -v "$(pwd)/keycloak:/opt/keycloak/data/import" \
  quay.io/keycloak/keycloak:24.0 start-dev --import-realm
# 预期：日志出现 Realm 'ia' imported，控制台可见 ia realm
```

> 生产提示：导出完整 realm（含角色/密码策略）用 `kc.sh export --realm ia --users realm_file`；JSON 中的明文密码仅演示用，正式环境用户走 LDAP/AD federation 或管理接口开号。

## 19.3 后端改造：PyJWKClient 验签 RS256

**改造目标**：`get_current_user` 依赖在 `keycloak` 模式下改验 Keycloak 签发的 RS256 token；当前用户从 claims 映射并**同步到本地 users 表（upsert）**；权限判定**不动**（仍走本地 RBAC 表）。用配置开关切换两种模式，随时可回退。

### 19.3.1 获取验签公钥：JWKS 端点（公钥的"活水源头"）

与原方案把公钥静态抄进 .env 不同，Python 版用 **PyJWT 的 `PyJWKClient`** 直接对接 Keycloak 的 JWKS 端点——公钥集合（JSON Web Key Set）按 `kid` 索引，**密钥轮换时自动刷新**，无需改配置重启：

```bash
curl -s http://127.0.0.1:8080/realms/ia/protocol/openid-connect/certs | head -c 200
# 预期输出：{"keys":[{"kty":"RSA","kid":"...","use":"sig","alg":"RS256","n":"MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...","e":"AQAB",...}
```

> 工作机制：每个 RS256 token 的 JWT 头里带 `kid`（key id）；`PyJWKClient` 首次请求时拉取 JWKS 并**进程内缓存**，token 的 `kid` 在缓存里找不到时自动重新拉取——轮换发生后的第一个请求最多多一次 HTTP 往返，业务无感。缓存与轮换的细节见 19.7 常见坑。

### 19.3.2 配置与 .env 变更

`.env` 追加：

```bash
# ---- 认证模式（19.3）：local=自签JWT（默认） / keycloak ----
IA_AUTH_MODE=keycloak
IA_KEYCLOAK_BASE_URL=http://127.0.0.1:8080
IA_KEYCLOAK_REALM=ia
IA_KEYCLOAK_CLIENT_ID=industryagent-server
IA_KEYCLOAK_CLIENT_SECRET=industryagent-server-secret-change-me
```

`app/core/config.py` 的 `Settings` 追加字段：

```python
    # ---- 认证模式与 Keycloak（19.3） ----
    IA_AUTH_MODE: str = 'local'                       # 'local' | 'keycloak'
    IA_KEYCLOAK_BASE_URL: str = 'http://127.0.0.1:8080'
    IA_KEYCLOAK_REALM: str = 'ia'
    IA_KEYCLOAK_CLIENT_ID: str = 'industryagent-server'
    IA_KEYCLOAK_CLIENT_SECRET: str = ''
```

> 注意公钥**不进 .env**——JWKS 端点在线获取并缓存，这是比静态公钥更标准的做法（少一份会过期的配置）。

### 19.3.3 用户模型与 get_current_user 改造

**① users 表加 Keycloak 关联字段**：`app/models/user.py` 的字段定义中追加一行（IA-SD-004 3.1 的 `keycloak_id`）：

```python
    # Keycloak sub（OIDC 关联，19.3）。列名按 IA-SD-004 用 snake_case：keycloak_id
    keycloakId: Mapped[str | None] = mapped_column(
        'keycloak_id', sa.String(64), unique=True, comment='Keycloak sub（OIDC 关联，19.3）')
```

**② deps 改造**：`app/core/deps.py` 中 `get_current_user` 完整替换为（local 分支 = 第 7 章原逻辑一字不差，keycloak 分支 = 新增）：

```python
import asyncio
import logging

import jwt
from jwt import PyJWKClient

from app.core.biz_error import BizError
from app.core.config import get_settings
from app.core.error_codes import ErrorCode
from app.core.security import decode_token

logger = logging.getLogger(__name__)

# 进程级 JWKS 客户端：PyJWKClient 自带缓存与密钥轮换刷新（首次/轮换后各拉一次 JWKS）
_jwk_client: PyJWKClient | None = None


def _get_jwk_client() -> PyJWKClient:
    global _jwk_client
    if _jwk_client is None:
        s = get_settings()
        _jwk_client = PyJWKClient(
            f'{s.IA_KEYCLOAK_BASE_URL}/realms/{s.IA_KEYCLOAK_REALM}'
            f'/protocol/openid-connect/certs',
            cache_keys=True,
        )
    return _jwk_client


async def decode_keycloak_token(token: str) -> dict:
    """Keycloak RS256 验签：issuer 校验 + azp（audience 语义）校验"""
    s = get_settings()
    issuer = f'{s.IA_KEYCLOAK_BASE_URL}/realms/{s.IA_KEYCLOAK_REALM}'
    try:
        # 按JWT 头的 kid 取公钥；首次（及轮换后）会同步拉一次 JWKS，
        # to_thread 丢进线程池避免阻塞事件循环
        signing_key = await asyncio.to_thread(_get_jwk_client().get_signing_key_from_jwt, token)
        payload = jwt.decode(
            token,
            signing_key.key,
            algorithms=['RS256'],        # 必须显式锁定算法，防 alg 混淆攻击
            issuer=issuer,               # 签发方必须是本 realm
            options={'verify_aud': False},
            # Keycloak 默认把 aud 签成 "account"，真正标识"发给哪个客户端"的是 azp；
            # 要严格校验 aud，给 client 配 audience mapper 后把 options 改为
            # {'verify_aud': True} 并加 audience=s.IA_KEYCLOAK_CLIENT_ID（见 19.7）
        )
    except jwt.ExpiredSignatureError:
        raise BizError(ErrorCode.TOKEN_EXPIRED, '登录已过期，请刷新令牌或重新登录', http_status=401)
    except jwt.PyJWTError as e:
        raise BizError(ErrorCode.UNAUTHORIZED, f'无效的访问令牌：{e}', http_status=401)
    if payload.get('azp') != s.IA_KEYCLOAK_CLIENT_ID:
        raise BizError(ErrorCode.UNAUTHORIZED, '令牌不是签发给本客户端的', http_status=401)
    return payload


async def get_current_user(request, db) -> dict:
    """JWT 鉴权依赖（路由级按需挂载）
    - local 模式：验自签 HS256（第 7 章，完全保留）
    - keycloak 模式：验 Keycloak RS256 + claims -> 本地用户 upsert
    用法不变：user: CurrentUser 或 Depends(require_perm('...'))
    """
    header = request.headers.get('authorization', '')
    token = header[7:] if header.lower().startswith('bearer ') else ''
    if not token:
        raise BizError(ErrorCode.UNAUTHORIZED, '缺少访问令牌（Authorization: Bearer <token>）', http_status=401)

    s = get_settings()
    if s.IA_AUTH_MODE == 'keycloak':
        # ---------- Keycloak RS256 验签 ----------
        claims = await decode_keycloak_token(token)
        # claims -> 本地用户 upsert（sub 优先，其次 preferred_username）
        from app.services.auth import AuthService
        return await AuthService(db).sync_keycloak_user(claims)

    # ---------- 自签 HS256（第 7 章原逻辑） ----------
    payload = decode_token(token)
    return {
        'userId': payload['userId'],
        'username': payload['username'],
        'roles': payload.get('roles', []),
    }
```

> `CurrentUser` 类型别名（`Annotated[dict, Depends(get_current_user)]`）与 `require_perm()` 保持第 7 章的定义不变——所有路由零改动，这就是把鉴权收敛到一个依赖函数的红利。

**③ 用户同步 Service**：`app/services/auth.py` 中 `AuthService` 追加方法（放在 `_get_user_roles` 前面）：

```python
  async def sync_keycloak_user(self, claims: dict) -> dict:
      """
      Keycloak claims -> 本地用户 upsert（19.3）
      按 sub（keycloakId）查，查不到按 username 兜底，仍没有则建档。
      返回的当前用户角色来自本地 RBAC 表——权限永远在本地。
      """
      sub = claims.get('sub')
      username = claims.get('preferred_username')
      user = None
      if sub:
          user = (await self.db.execute(
              select(User).where(User.keycloakId == sub))).scalar_one_or_none()
      if user is None and username:
          user = (await self.db.execute(
              select(User).where(User.username == username))).scalar_one_or_none()
          # 老用户首次用 Keycloak 登录：回填 keycloakId 建立绑定
          if user is not None and user.keycloakId is None:
              user.keycloakId = sub
              await self.db.commit()
      if user is None:
          import uuid
          from app.core.security import hash_password
          user = User(
              username=username or f'kc_{str(sub)[:12]}',
              passwordHash=hash_password(str(uuid.uuid4())),  # 占位哈希：密码认证已外包给 Keycloak
              realName=claims.get('name') or username,
              email=claims.get('email'),
              keycloakId=sub,
              status='active',
          )
          self.db.add(user)
          await self.db.commit()
          await self.db.refresh(user)
      if user.status != 'active':
          raise BizError(ErrorCode.USER_DISABLED, '账号已被禁用，请联系管理员', http_status=403)
      roles = await self._get_user_roles(user.id)
      return {'userId': user.id, 'username': user.username, 'roles': roles}
```

> **权限映射完全不动**：`require_perm()`、`ROLE_PERMISSIONS`、`user_roles` 表原样生效——Keycloak 负责"你是谁"，本地 RBAC 负责"你能干什么"。文件头部按需补 `from sqlalchemy import select` 与 `from app.models.user import User`（第 7 章已有则忽略）。

**④ users 表结构变更生效**：模型加了字段，开发环境直接重跑 `python scripts/init_db.py` 重建（清库），或手工执行：

```bash
docker exec -it ia-pg psql -U industry -d industryagent \
  -c "ALTER TABLE users ADD COLUMN IF NOT EXISTS keycloak_id VARCHAR(64) UNIQUE;"
```

> **WS 通道说明**：教程的 WS 子应用（`app/ws/app.py`）沿用 local HS256 验签——前端建 WS 连接用的 token 由后端代理登录返回，双模式下都可先经 19.4 的代理端点拿 token。若组织要求 WS 也直验 Keycloak RS256，把 `app/ws/app.py` 里的 `decode_token(token)` 换成 `await decode_keycloak_token(token)` 即可（同一进程，直接 import）。

## 19.4 登录流程变化：后端代理 token 端点（推荐方案）

**方案选择**：授权码流程（前端跳 Keycloak 登录页）是 OAuth2 最标准玩法，但需要前端处理回调、PKCE、跨域会话，改造成本高且体验割裂（内外网两套登录页）。**教程推荐"后端代理"**：前端表单照旧 POST 用户名密码到后端，后端持 client_secret 走 ROPC 找 Keycloak 换 token——前端零改造、密钥不出服务器、响应结构与第 7 章 `login` 完全一致。

`app/services/keycloak.py`（完整内容，新增文件）：

```python
"""Keycloak 对接（19.4）：后端代理登录/刷新/登出
前端只见本服务返回的统一结构，client_secret 永不下发浏览器
"""
import logging

import httpx
import jwt as pyjwt
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.biz_error import BizError
from app.core.config import get_settings
from app.core.error_codes import ErrorCode
from app.services.auth import AuthService

logger = logging.getLogger(__name__)


class KeycloakService:

    def __init__(self, db: AsyncSession):
        self.db = db

    @property
    def _cfg(self):
        s = get_settings()
        return {
            'base_url': s.IA_KEYCLOAK_BASE_URL,
            'realm': s.IA_KEYCLOAK_REALM,
            'client_id': s.IA_KEYCLOAK_CLIENT_ID,
            'client_secret': s.IA_KEYCLOAK_CLIENT_SECRET,
        }

    @property
    def _token_url(self) -> str:
        return (f"{self._cfg['base_url']}/realms/{self._cfg['realm']}"
                f"/protocol/openid-connect/token")

    async def login(self, username: str, password: str) -> dict:
        """密码直换 token（ROPC）"""
        async with httpx.AsyncClient(timeout=5.0) as client:
            resp = await client.post(self._token_url, data={
                'grant_type': 'password',
                'client_id': self._cfg['client_id'],
                'client_secret': self._cfg['client_secret'],
                'username': username,
                'password': password,
            })
        body = resp.json() if resp.content else {}
        if resp.status_code != 200 or 'access_token' not in body:
            # Keycloak 对"用户不存在"与"密码错误"同样返回 400 invalid_grant，天然防用户名枚举
            raise BizError(ErrorCode.LOGIN_FAILED, '用户名或密码错误')
        return await self._normalize(body)

    async def refresh(self, refresh_token: str) -> dict:
        """刷新 token"""
        async with httpx.AsyncClient(timeout=5.0) as client:
            resp = await client.post(self._token_url, data={
                'grant_type': 'refresh_token',
                'client_id': self._cfg['client_id'],
                'client_secret': self._cfg['client_secret'],
                'refresh_token': refresh_token,
            })
        body = resp.json() if resp.content else {}
        if resp.status_code != 200 or 'access_token' not in body:
            raise BizError(ErrorCode.TOKEN_EXPIRED, '刷新令牌无效或已过期，请重新登录', http_status=401)
        return await self._normalize(body)

    async def logout(self, refresh_token: str | None) -> dict:
        """登出：吊销 refresh token（access token 短时效自然过期）"""
        if not refresh_token:
            return {'revoked': False}
        async with httpx.AsyncClient(timeout=5.0) as client:
            await client.post(
                f"{self._cfg['base_url']}/realms/{self._cfg['realm']}"
                f"/protocol/openid-connect/logout",
                data={
                    'client_id': self._cfg['client_id'],
                    'client_secret': self._cfg['client_secret'],
                    'refresh_token': refresh_token,
                },
            )
        return {'revoked': True}

    async def _normalize(self, data: dict) -> dict:
        """Keycloak 响应 -> 第 7 章 login 接口的返回结构（前端零改造的关键）"""
        # 从 access_token 解出用户信息（验签在 get_current_user 依赖，这里只 decode 取 claims）
        payload = pyjwt.decode(data['access_token'], options={'verify_signature': False})
        user = await AuthService(self.db).sync_keycloak_user(payload)
        return {
            'accessToken': data['access_token'],
            'refreshToken': data['refresh_token'],
            'tokenType': 'Bearer',
            'expiresInSeconds': data['expires_in'],
            'user': {
                'userId': user['userId'],
                'username': user['username'],
                'roles': user['roles'],
            },
        }
```

`app/routers/auth.py` 中追加三个方法（与原有 login/refresh/logout 并列，**原有方法保留**作为 local 模式入口；复用第 7 章的 `LoginIn` / `RefreshIn` schema）：

```python
@router.post('/api/v1/auth/keycloak/login')
async def kc_login(
    body: LoginIn,
    db: AsyncSession = Depends(get_db),
):
    """POST /api/v1/auth/keycloak/login（19.4：后端代理 Keycloak 登录）"""
    return success(await KeycloakService(db).login(body.username, body.password))


@router.post('/api/v1/auth/keycloak/refresh')
async def kc_refresh(
    body: RefreshIn,
    db: AsyncSession = Depends(get_db),
):
    """POST /api/v1/auth/keycloak/refresh"""
    return success(await KeycloakService(db).refresh(body.refreshToken))


@router.post('/api/v1/auth/keycloak/logout')
async def kc_logout(
    body: dict | None = None,
    db: AsyncSession = Depends(get_db),
):
    """POST /api/v1/auth/keycloak/logout（body 可选携带 refreshToken）"""
    return success(await KeycloakService(db).logout((body or {}).get('refreshToken')))
```

路由无需再挂载（auth router 已在 `create_app()` 注册，新增的三条随模块生效；文件头部补 `from app.services.keycloak import KeycloakService`）。

> 前端改造：把登录/刷新的请求地址从 `/auth/login` 换成 `/auth/keycloak/login`（或由网关按 `IA_AUTH_MODE` 转发），**请求体与响应结构完全不变**——这就是代理方案的回报。

## 19.5 验证与回退

### 19.5.1 全链路验证

确保 ia-keycloak 在跑、`.env` 已按 19.3.2 配置（`IA_AUTH_MODE=keycloak`），重启 `python -m app.run`：

```bash
# ① 通过后端代理登录（密码其实由 Keycloak 校验）
KC_LOGIN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/keycloak/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"engineer","password":"Engineer@123"}')
echo "$KC_LOGIN" | head -c 200
# 预期：{"code":0,"message":"ok","data":{"accessToken":"eyJhbGciOiJSUzI1NiIs...","refreshToken":"...",...
#       "user":{"userId":2,"username":"engineer","roles":["process_engineer"]},...}

KC_TOKEN=$(echo "$KC_LOGIN" | grep -o '"accessToken":"[^"]*"' | cut -d'"' -f4)

# ② Keycloak 签发的 token 访问业务接口（RS256 公钥验签 + 本地 RBAC 判权）
curl -s 'http://127.0.0.1:7001/api/v1/wafers?pageSize=1' -H "Authorization: Bearer $KC_TOKEN" | head -c 120
# 预期：{"code":0,"message":"ok","data":{"rows":[...

# ③ 错误密码 -> Keycloak 拒绝
curl -s -X POST http://127.0.0.1:7001/api/v1/auth/keycloak/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"engineer","password":"wrong"}'
# 预期：{"code":20004,"message":"用户名或密码错误",...}

# ④ 越权验证（engineer 访问 admin 接口，权限仍由本地 RBAC 判定）
curl -si 'http://127.0.0.1:7001/api/v1/users' -H "Authorization: Bearer $KC_TOKEN" | head -n 1
# 预期：HTTP/1.1 403 Forbidden（user:read 仅 admin——证明"认证外包、授权本地"）

# ⑤ 用户同步验证：users 表应出现 keycloak_id
docker exec -it ia-pg psql -U industry -d industryagent \
  -c "SELECT id, username, keycloak_id FROM users ORDER BY id;"
# 预期：engineer 行的 keycloak_id 已回填为 Keycloak sub（UUID 形态）
```

### 19.5.2 回退方案

认证是系统的命门，升级必须可回退——本方案的回退成本是**一行配置**：

```bash
# .env 改回本地自签 JWT
IA_AUTH_MODE=local
```

重启 `python -m app.run` 即恢复第 7 章全流程（`/auth/login` 端点从未被删除，双 token + Redis 白名单照常工作）。建议生产切换时灰度观察一周：两套端点并存，前端按环境变量决定调哪套，确认无回归后再下线 local 模式。

## 19.6 本章小结：全书完结

### 19.6.1 全书模块清单（1~19 章交付物总览）

| 能力 | 关键实现 | 章节 |
|---|---|---|
| 工程骨架 | FastAPI + uvicorn 分层工程（routers/services/models/middleware/core） | 3 |
| 配置体系 | pydantic-settings + .env（IA_* 同名变量）+ IA_ENV 多环境 | 4 |
| CRUD 模板 | wafer 模块：pydantic 校验、事务查重、统一响应、curl 全流程验证 | 5 |
| 横切关注点 | BizError 错误体系、审计日志（BackgroundTasks 异步入库）、结构化请求日志 + traceId | 5~6 |
| 鉴权（本地） | bcrypt + PyJWT 双 token + Redis 白名单 + RBAC 多角色 | 7 |
| 智能体协作 | httpx 超时/重试/自研熔断调用智能体服务，任务表全生命周期 | 8 |
| 业务闭环 | 分析 → 建议落库 → 审批（事务 + 行锁）→ 审计留痕 | 9 |
| Redis 进阶 | cache-aside 看板缓存 + 写路径失效、Idempotency-Key 幂等中间件 | 10 |
| 后台任务 | APScheduler 数据健康检查 + 良率日报（interval/cron 触发器） | 11 |
| 质量保障 | pytest 单测（mock service）+ httpx ASGITransport 集成测试 + 覆盖率 | 12 |
| MCP | FastMCP stdio Server 三工具 + 客户端脚本验收 | 13 |
| 可观测 | prometheus-client QPS/延迟指标、/healthz 探活（依赖故障 503 演示） | 14 |
| 部署 | pytest + uvicorn 冒烟、多阶段 Dockerfile（python:3.12-slim、非 root、HEALTHCHECK）+ compose 编排 | 15 |
| 8D 报告 | reports_8d 状态机（draft→pending_review→released）、审批通过自动生成骨架、签发写审计 | 18.1 |
| 通知 + WebSocket | notifications 站内信四接口、双 uvicorn Server 共享 ConnectionManager（鉴权/房间隔离/心跳）、IA-SD-005 帧格式 | 18.2 |
| 缺陷 + 对象存储 | defect_records 多条件检索、MinIO 上传（to_thread 不阻塞事件循环）、7 天预签名 GET URL | 18.3 |
| 看板统计 | overview（良率/OEE/今日异常/待审批，cache-aside 60s）、yield-trend、defect-category、equipment-top | 18.4 |
| 审计查询 | audit_logs 管理员检索（操作人/路径/时间范围） | 18.5 |
| 用户管理 | 创建（bcrypt）/启停/改绑角色/重置密码，全部 admin 权限 | 18.6 |
| Keycloak | PyJWKClient 拉 JWKS 验 RS256（issuer/azp 校验）、claims→本地用户 upsert、后端代理登录、一键回退 | 19 |

**接口规模**：43 个 REST 端点（18.7.2 总表）+ 1 个 WS 推送通道（2 类事件）+ 3 个 MCP 工具。与 IA-SD-005 的对照结论见 18.7.3：34 个接口中 32 个已实现（含等价替代），2 个（任务 retry/takeover、审批 SLA 统计）列为后续迭代。

**Python 版技术收获清单**（对照 Egg.js 版的迁移红利）：

1. **与智能体服务同栈**：后端（FastAPI）与多智能体服务共享 Python 生态，LangGraph/知识图谱/RAG 的集成不再跨语言，共享同一套虚拟环境管理经验；
2. **原生 async 全链路**：SQLAlchemy 2.0 async + asyncpg + httpx + redis.asyncio，IO 密集型业务（调智能体、查库、推 WS）单进程高并发，无需多 worker 心智负担；
3. **类型即文档**：pydantic v2 的请求/响应模型自动生成 OpenAPI 文档，`/docs` 页面天然成为联调工具；
4. **依赖注入即中间件**：`Depends` 把鉴权/权限/DB 会话收敛为可组合的函数，第 19 章换认证模式只动一个函数；
5. **双 Server 单进程**：uvicorn 的 `Server` 对象可编程编排，REST 与 WS 共享连接管理器，比"后台进程 + 进程间消息转发"的架构少一整层复杂度。

### 19.6.2 与文档群的映射

| 文档 | 编号 | 与本教程的对应关系 |
|---|---|---|
| 用户需求说明书 / SRS | IA-URS-001 / IA-SRS-001 | 业务能力来源（异常监控、根因分析、人机协同审批） |
| 系统架构设计文档 | IA-SD-001 | 1.2 节架构图即其 FastAPI 后端部分的落地 |
| 后端详细设计说明书 | IA-SD-002 | 模块/分层设计的实现依据 |
| 数据库设计说明书 | IA-SD-004 | 表结构语义依据（教程简化：整型主键、camelCase 字段，差异见 18.7.3） |
| 接口设计说明书 | IA-SD-005 | 34 接口 + WS 事件 + MCP 工具的对照基准（18.7.3 差异表） |
| 前端开发教程 | IA-TU-002 | 消费本教程全部 REST API 与 WS 通道（18.2.9 对接要点） |
| 智能体开发教程 | IA-TU-003 | 替换 agent-mock，对接 POST /api/v1/rootcause/analyze 协议与 MCP 工具 |
| 部署上线教程 | IA-TU-004 | 第 15 章 compose 的生产化延伸 |

### 19.6.3 三条心法（贯穿全书）

1. **分层纪律**：Router 薄、Service 厚、Model 纯——18 章的六个新模块全部按 wafer 模板的五步走（Model → Service → Router → 注册 → curl 验证），没有一处例外；
2. **约束前置**：错误码、审计、traceId 在第 5~6 章就立好规矩，18 章补模块时**零成本**继承——18.7 的差异表里所有"教程增强项"（traceId、幂等、审计）都是早期约束的红利；
3. **复用技巧**："同一份 SQLAlchemy 模型 + Declarative Base"让表定义在 API 进程、`scripts/init_db.py` 种子、MCP Server 三处复用；"认证外包、授权本地"让 Keycloak 迁移只动了 `get_current_user` 一个依赖函数——**好的架构让变更局部化**。

### 19.6.4 下一步

- **《03-智能体开发教程》**：把 `agent-mock/main.py` 替换为真正的多智能体服务（LangGraph 编排监控/根因/建议 Agent，接入知识图谱与 RAG），并对接本教程的 `POST /api/v1/rootcause/analyze` 协议与 13 章的 MCP 工具；
- **《01-前端开发教程-React从0到1》**：基于 43 个 REST 端点与 WS 通道构建管理台（登录、批次管理、看板大屏、审批工作流、通知铃铛），落地 token 静默刷新与断线补偿；
- **进阶方向（按优先级）**：补齐 retry/takeover 与审批 SLA 统计两接口 → OpenTelemetry 分布式追踪打通 traceId → GitHub Actions CI（lint + pytest + build 镜像）→ Alembic migration 替代 create_all（见《02-数据库设计与迁移教程》）→ WebSocket 集群化（Redis pub/sub 替代单进程 ConnectionManager）→ K8s 编排与灰度发布。

**全教程完。** 从 `pip install fastapi uvicorn` 到 43 个端点、从自签 HS256 到 Keycloak RS256、从单表 CRUD 到"分析→建议→审批→报告→通知"的完整人机协同闭环——这套代码与文档群，就是你的第一份企业级全栈交付物。

---

## 19.7 常见坑

**坑 1：Keycloak JWKS 缓存与轮换——`Unable to find a signing key`**

- 现象：Keycloak 侧轮换 realm 密钥（或重建容器）后，个别请求报 401 `无效的访问令牌：Unable to find a signing key that matches 'kid'`，之后又自愈。
- 原因：`PyJWKClient` 进程内缓存了旧 JWKS；token 带新 `kid` 时它会**自动重新拉取**（教程 `cache_keys=True` 开启的就是这个行为），自愈的原因即在此——中间那一两个请求的 401 是刷新窗口的固有代价。
- 缓解：客户端在收到 401（20001）时静默重登一次再重试，是 OAuth2 客户端的标准做法；服务端不要为"消灭这一次 401"把 JWKS 拉取改成每请求一次——JWKS 端点也会被 Keycloak 限流。
- 相关坑：`PyJWKClient` 的拉取是**同步 urllib**，教程用 `asyncio.to_thread` 包了一层；如果你的进程里出现 `_RunningEventLoop` 警告或卡顿，检查是不是有人把同步调用直接放进了 async 函数。另外 realm 重建（`--import-realm` 重导）会生成全新密钥，旧 token 全部作废属预期行为——测试环境先 `docker rm -f ia-keycloak` 重来时，记得前端也重新登录。

**坑 2：`verify_aud` 严格校验一开就 401（aud vs azp）**

- 原因：Keycloak 默认把 ROPC access token 的 `aud` 签成 `["account"]`，而不是你的 client id；真正标识"发给哪个客户端"的 claims 是 `azp`。教程因此默认 `verify_aud: False` + 手工校验 `azp == IA_KEYCLOAK_CLIENT_ID`。
- 要严格 `aud` 校验：Keycloak 控制台 → Clients → industryagent-server → Client scopes → dedicated scope → Add mapper → Audience，把本 client 加入 audience，然后 `jwt.decode(..., audience=IA_KEYCLOAK_CLIENT_ID)` 并移除手工 azp 检查。

**坑 3：代理登录 401 但密码明明正确**

- 依次检查：`.env` 的 `IA_KEYCLOAK_CLIENT_SECRET` 是否与 Credentials 页一致（realm-json 导入时就是 `industryagent-server-secret-change-me`）；client 的 **Direct access grants** 是否开启（19.2.2 第 2 步勾选项，关着则 ROPC 一律 400 `unauthorized_client`）；测试用户密码是否勾了 **Temporary**（临时密码会在首次登录时返回 `account is not fully set up`，不是标准 invalid_grant，容易误判）。




