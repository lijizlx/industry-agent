# 12. 单元测试与集成测试

## 12.1 做什么 / 为什么

手册的工程化要求包含 CI/CD 自动测试。测试分两层：

- **单元测试**：只测 service 的业务逻辑，用 `unittest.mock` 把 `AsyncSession` mock 掉——快、稳定、不依赖环境；
- **集成测试**：不起真实端口，用 `httpx.AsyncClient` 挂 `ASGITransport` **在进程内直连 FastAPI 应用**，走完整中间件链（traceId / 审计 / 鉴权），数据库连**测试库** `industryagent_test`。

技术选型对标（原 Egg 版 → Python 版）：

| 原版 | Python 版 | 职责 |
|---|---|---|
| egg-bin（内置 mocha） | pytest | 测试框架与发现 |
| chai | pytest 原生 `assert` | 断言 |
| sinon | unittest.mock（MagicMock / AsyncMock） | mock 数据库层 |
| egg-mock 的 `app.httpRequest()`（内置 supertest） | httpx.AsyncClient + ASGITransport | 进程内 HTTP 集成测试 |
| egg-bin cov | coverage.py | 覆盖率 |

## 12.2 安装测试依赖：requirements-dev.txt

项目根新建 `requirements-dev.txt`（完整内容）：

```text
# 测试与覆盖率工具：只在开发期安装，不进生产镜像（第 15 章只 COPY requirements.txt）
-r requirements.txt
pytest==8.3.4
pytest-asyncio==0.25.2
httpx==0.28.1
coverage==7.6.10
```

> `httpx` 在 `requirements.txt` 里已有（第 8 章调智能体用），这里重复声明无害——dev 文件的价值是「一条命令装齐开发环境」。

安装：

```bash
pip install -r requirements-dev.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
# 预期：Successfully installed pytest-... pytest-asyncio-... coverage-... 无 error
```

顺手在 `.gitignore` 追加三行（测试产物）：

```text
.pytest_cache/
.coverage
htmlcov/
```

## 12.3 测试库准备与 pytest 配置

建测试库（命令与第 2 章建业务库同款，只是库名不同）：

```bash
docker exec -it ia-pg psql -U industry -d postgres \
  -c "CREATE DATABASE industryagent_test OWNER industry;"
# 预期输出：CREATE DATABASE
```

`pytest.ini`（完整内容，项目根）：

```ini
[pytest]
# 异步用例自动接管：不用逐个函数加 @pytest.mark.asyncio
asyncio_mode = auto
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
filterwarnings =
    ignore::DeprecationWarning
```

> 若你习惯把配置集中在 `pyproject.toml`，等价写法是 `[tool.pytest.ini_options]` 段里 `asyncio_mode = "auto"`、`testpaths = ["tests"]`。本项目统一用 `pytest.ini`，少一个文件参与者。

`tests/conftest.py`（完整内容）——**这是本章配置的核心**：

```python
"""
测试全局夹具。

关键顺序：先设环境变量，再导入 app ——
Settings 与 async engine 都在模块导入期完成初始化，之后再生效的环境变量改不了它们。
这是 Python 版「配置指向测试库」的方式，对标原 Egg 版的 config.test.ts。
"""
import os

os.environ["IA_ENV"] = "test"
os.environ.setdefault("IA_DB_NAME", "industryagent_test")
# Redis 用独立 db（可选隔离：测试的 refresh 白名单/缓存不冲掉开发数据）
os.environ.setdefault("IA_REDIS_DB", "15")

import httpx      # noqa: E402
import pytest     # noqa: E402

from app.main import create_app          # noqa: E402


@pytest.fixture()
def app():
    """每个用例新建应用实例（create_app 很轻，无连接副作用）"""
    return create_app()


@pytest.fixture()
async def client(app):
    """进程内 HTTP 客户端：不占端口、不走网络栈，但走完整 ASGI 中间件链"""
    transport = httpx.ASGITransport(app=app)
    async with httpx.AsyncClient(transport=transport, base_url="http://testserver") as c:
        yield c
```

> **为什么用环境变量而不是 `app.dependency_overrides` 覆盖 `get_settings`**：`get_settings` 的返回值在 `app.core.database` 被 import 时就已用来创建 engine（模块级单例），依赖覆盖发生在应用组装期，太晚了。环境变量先于导入生效，才是对「import 期配置」可靠的测试注入方式。`dependency_overrides` 留给运行期依赖（如覆盖 `get_current_user` 造匿名场景）——那才是它的主场。

## 12.4 单元测试：tests/unit/test_wafer_service.py（完整内容）

**做什么/为什么**：mock 掉 `AsyncSession`，让 `execute` 返回我们构造的 `FakeResult`，验证 `list` 的 where 拼装与分页结构——完全不碰数据库，毫秒级完成。

两个前置约定（第 5 章已立的契约，用例锚定它们）：
- `WaferService.list(page, page_size, batch_no=None, ...)` 返回 `{"rows": [...], "total": N}`，内部执行**两次**查询（行 + count）；
- `WaferService.detail(id)` 查不到时抛 `BizError(ErrorCode.NOT_FOUND, http_status=404)`。

```python
"""
WaferService 单元测试：mock AsyncSession，验证 list 的 where 拼装与分页结构。
对标原 12.3 的 sinon mock findAndCountAll 用例语义。
"""
from types import SimpleNamespace
from unittest.mock import AsyncMock, MagicMock

import pytest

from app.core.biz_error import BizError
from app.core.error_codes import ErrorCode
from app.services.wafer import WaferService


class FakeResult:
    """模拟 SQLAlchemy 的 Result：scalars().all() 给行数据，scalar() 给 count"""

    def __init__(self, rows=None, scalar_value=None):
        self._rows = rows or []
        self._scalar_value = scalar_value

    def scalars(self):
        return self

    def all(self):
        return self._rows

    def scalar(self):
        return self._scalar_value


def make_row(row_id: int, batch_no: str, yield_rate: str):
    return SimpleNamespace(id=row_id, batchNo=batch_no, yieldRate=yield_rate)


def make_session(side_effects):
    """构造 mock 的 AsyncSession：execute 按调用顺序依次返回 FakeResult"""
    session = MagicMock()
    session.execute = AsyncMock(side_effect=side_effects)
    return session


def sql_of(stmt) -> str:
    """把 select 表达式编译成 SQL 文本。

    SQLAlchemy 的 select 是不可变表达式对象，compile 后能直接检查
    where/limit/offset 拼装——这是 Python 版对标「sinon 检查 stub 入参」的方式。
    """
    return str(stmt.compile(compile_kwargs={"literal_binds": True}))


class TestWaferServiceList:

    async def test_无过滤条件时返回全部分页数据(self):
        rows = [
            make_row(1, "B20261015001", "91.20"),
            make_row(2, "B20261015002", "96.80"),
        ]
        session = make_session([
            FakeResult(rows=rows),        # 第 1 次 execute：查行
            FakeResult(scalar_value=2),   # 第 2 次 execute：查 count
        ])

        result = await WaferService(session).list(page=1, page_size=10)

        assert result["total"] == 2
        assert len(result["rows"]) == 2
        assert result["rows"][0].batchNo == "B20261015001"
        # 行查询 + count 查询，共两次
        assert session.execute.await_count == 2

        # 校验传给 ORM 的分页参数（对标原版断言 stub.firstCall.args[0].offset/limit）
        rows_stmt = session.execute.await_args_list[0].args[0]
        sql = sql_of(rows_stmt)
        assert "LIMIT 10" in sql
        assert "WHERE" not in sql       # 无过滤条件时不应该有 WHERE 子句

    async def test_带批次号时拼接模糊查询条件(self):
        session = make_session([
            FakeResult(rows=[]),
            FakeResult(scalar_value=0),
        ])

        await WaferService(session).list(page=2, page_size=5, batch_no="15001")

        sql = sql_of(session.execute.await_args_list[0].args[0])
        # page=2, pageSize=5 -> offset 5
        assert "OFFSET 5" in sql
        assert "LIMIT 5" in sql
        # 模糊查询应包含 %15001%（对标原版 where.batchNo 的 %xxx% 断言）
        assert "%15001%" in sql


class TestWaferServiceDetail:

    async def test_批次不存在时抛出_NOT_FOUND_业务错误(self):
        session = MagicMock()
        session.get = AsyncMock(return_value=None)   # db.get 是 findByPk 的对应物

        with pytest.raises(BizError) as exc_info:
            await WaferService(session).detail(999)

        assert exc_info.value.code == ErrorCode.NOT_FOUND.value   # 10002
        assert exc_info.value.http_status == 404
```

三个用例分别对标原 12.3 的三个：无过滤分页 / 模糊条件与 offset / detail 抛 `NOT_FOUND`。

## 12.5 集成测试：tests/integration/test_auth_flow.py（完整内容）

**做什么/为什么**：`httpx.ASGITransport(app=...)` 把 FastAPI 应用当传输层直连——请求真的会穿过 traceId、审计、鉴权中间件和路由层，只是不监听网卡。数据库连 `industryagent_test`（conftest 已把 `IA_ENV` 置为 `test`）。

```python
"""
auth 认证接口（集成测试）：
login → profile → 权限不足 403 的全链路用例，对标原 12.4 语义。
前提：PG 与 Redis 容器在运行（login 会写 refresh 白名单到 Redis）。
"""
import bcrypt
import pytest

import app.models                     # noqa: F401  确保 create_all 时全部模型已注册
from app.core.database import Base, AsyncSessionLocal, engine
from app.models.user import Role, User, UserRole


def hash_pw(plain: str) -> str:
    return bcrypt.hashpw(plain.encode(), bcrypt.gensalt(rounds=10)).decode()


@pytest.fixture(autouse=True)
async def _database(client):
    """每个用例：重建全部表 + 灌入已知用户；用例结束释放连接池。

    为什么每个用例都重建（而不是整个文件一次）：
    pytest-asyncio 默认每个用例一个新事件循环，而 asyncpg 连接绑定
    「创建它的那个循环」——测试之间必须 engine.dispose() 释放连接池，
    否则第二个用例就报 RuntimeError: Event loop is closed（FAQ 第 2 条）。
    """
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
        await conn.run_sync(Base.metadata.create_all)

    async with AsyncSessionLocal() as db:
        db.add_all([
            Role(code="admin", name="管理员"),
            Role(code="process_engineer", name="工艺工程师"),
            User(username="it_user", passwordHash=hash_pw("Passw0rd"),
                 realName="集成测试用户", email="it_user@xinlian.demo", status="active"),
            User(username="it_engineer", passwordHash=hash_pw("Passw0rd"),
                 realName="集成测试工程师", email="it_engineer@xinlian.demo", status="active"),
            # 角色绑定（id 从 1 自增，与建表顺序一致）
            UserRole(userId=1, roleId=1),
            UserRole(userId=2, roleId=2),
        ])
        await db.commit()

    yield

    await engine.dispose()


async def login(client, username="it_user", password="Passw0rd") -> str:
    res = await client.post("/api/v1/auth/login", json={"username": username, "password": password})
    return res.json()["data"]["accessToken"]


class TestAuthFlow:

    async def test_login_正确密码返回双token(self, client):
        res = await client.post(
            "/api/v1/auth/login",
            json={"username": "it_user", "password": "Passw0rd"},
        )
        assert res.status_code == 200
        body = res.json()
        assert body["code"] == 0
        assert body["data"]["accessToken"]
        assert body["data"]["refreshToken"]
        assert body["data"]["user"]["roles"] == ["admin"]

    async def test_login_错误密码返回_20004(self, client):
        res = await client.post(
            "/api/v1/auth/login",
            json={"username": "it_user", "password": "wrong-password"},
        )
        assert res.status_code == 400
        assert res.json()["code"] == 20004   # ErrorCode.LOGIN_FAILED

    async def test_未带token访问受保护接口返回401(self, client):
        res = await client.get("/api/v1/wafers")
        assert res.status_code == 401
        assert res.json()["code"] == 20001   # ErrorCode.UNAUTHORIZED

    async def test_登录后带token可访问列表接口(self, client):
        token = await login(client)
        res = await client.get("/api/v1/wafers", headers={"Authorization": f"Bearer {token}"})
        assert res.status_code == 200
        assert res.json()["code"] == 0

    async def test_profile_返回当前用户信息(self, client):
        token = await login(client)
        res = await client.get("/api/v1/auth/profile", headers={"Authorization": f"Bearer {token}"})
        assert res.status_code == 200
        data = res.json()["data"]
        assert data["username"] == "it_user"
        assert data["realName"] == "集成测试用户"
        assert data["roles"] == ["admin"]

    async def test_工程师删除批次返回403(self, client):
        # process_engineer 没有 wafer:delete 权限点（第 7 章角色映射）
        token = await login(client, username="it_engineer")
        res = await client.delete("/api/v1/wafers/1", headers={"Authorization": f"Bearer {token}"})
        assert res.status_code == 403
        body = res.json()
        assert body["code"] == 20003       # ErrorCode.FORBIDDEN
        assert "wafer:delete" in body["message"]
```

> **注意**：`ASGITransport` 不会触发应用的 `lifespan`——所以第 11 章的 APScheduler 任务不会在测试进程里启动，天然隔离；同理也意味着本章不测「启动期逻辑」（那属于第 15 章部署冒烟的职责）。

## 12.6 运行测试与覆盖率

在**项目根**执行（Git Bash 与 PowerShell 命令一致）：

```bash
pytest -v
```

预期输出：

```text
tests/unit/test_wafer_service.py::TestWaferServiceList::test_无过滤条件时返回全部分页数据 PASSED [ 10%]
tests/unit/test_wafer_service.py::TestWaferServiceList::test_带批次号时拼接模糊查询条件 PASSED [ 16%]
tests/unit/test_wafer_service.py::TestWaferServiceDetail::test_批次不存在时抛出_NOT_FOUND_业务错误 PASSED [ 22%]
tests/integration/test_auth_flow.py::TestAuthFlow::test_login_正确密码返回双token PASSED [ 33%]
tests/integration/test_auth_flow.py::TestAuthFlow::test_login_错误密码返回_20004 PASSED [ 40%]
tests/integration/test_auth_flow.py::TestAuthFlow::test_未带token访问受保护接口返回401 PASSED [ 48%]
tests/integration/test_auth_flow.py::TestAuthFlow::test_登录后带token可访问列表接口 PASSED [ 56%]
tests/integration/test_auth_flow.py::TestAuthFlow::test_profile_返回当前用户信息 PASSED [ 64%]
tests/integration/test_auth_flow.py::TestAuthFlow::test_工程师删除批次返回403 PASSED [ 72%]

================================== 9 passed in 5.41s ==================================
```

覆盖率报告。项目根新建 `.coveragerc`（完整内容）：

```ini
[run]
source = app
omit =
    app/mcp/*
    app/run.py

[report]
show_missing = False
```

> `app/mcp/server.py` 是独立进程入口，由第 13 章的客户端脚本验收，不进 Web 覆盖统计；`app/run.py` 是第 18 章的双 Server 启动器。

```bash
coverage run -m pytest && coverage report
```

预期输出（节选，数字因代码而异）：

```text
Name                              Stmts   Miss  Cover
-----------------------------------------------------
app\services\wafer.py                38      4    89%
app\services\auth.py                 52      8    85%
app\routers\auth.py                  25      6    76%
app\core\security.py                 31      7    77%
...
-----------------------------------------------------
TOTAL                               412     95    77%
```

**覆盖率门禁（团队约定，写进提交规范）**：

- 全局行覆盖 **≥ 70%**：CI 里用 `coverage report --fail-under=70` 硬卡，低于即失败；
- 核心业务 service（`app/services/` 下的 wafer / auth / rootcause / suggestion）**≥ 85%**：这几处承载事务、行锁与熔断，漏测一行就是漏测一个生产事故剧本。教学项目在 code review 里人工把关；企业化做法是 `coverage json` 后用脚本按目录断言。

> **注意**：集成测试需要 PG 与 Redis 在运行（docker 容器别停，Redis 用 db 15 已隔离）。测试对开发库零污染——表只建在 `industryagent_test` 里。

---

# 13. MCP Server：把后端能力暴露给智能体

## 13.1 做什么 / 为什么

**MCP（Model Context Protocol）** 是 Anthropic 开源的智能体工具调用协议，现已成为事实标准。它解决的问题是：让 LLM/智能体以**标准化、可发现**的方式调用外部工具（查数据库、触发分析、调用 API）。

架构定位（见 1.2 节架构图）：industryagent-server 除了给前端供 REST API，还作为 **MCP Server** 向智能体暴露三个工具：

| 工具名 | 能力 |
|---|---|
| `query_wafers` | 查询晶圆批次（批次号模糊 + 分页） |
| `query_anomalies` | 查询异常事件（按时间倒序，limit 条） |
| `trigger_rootcause` | 对指定异常事件触发根因分析（调智能体服务返回 Top-3 根因） |

这样工程师在 Cursor / Claude Desktop / Claude Code 里直接问："帮我查下最近的异常事件，并对涉及的批次做根因分析"——智能体自动调用这些工具完成闭环。

**集成方式选择**：MCP 有 stdio（标准输入输出，客户端拉起服务进程）与 Streamable HTTP 两种传输。stdio 最通用（Cursor、Claude Desktop、Claude Code 全都支持），且与本教程"数据库能力复用"的技巧完美配合。我们用官方 `mcp` Python SDK 的 **FastMCP** 实现 stdio Server。

**进程拓扑（与 web server 的关系）**：MCP Server 是一个**独立进程**，不跑在 uvicorn 里、不占 7001/7002 端口；但它 **import 复用** `app/models/` 的 SQLAlchemy 模型与 `app/core/config.py` 的 Settings——一份模型两处使用，避免双维护。原 Egg 版需要构造"迷你 app"才能在进程外复用模型，Python 的 import 机制让这件事天然成立：**模型即唯一事实源，import 即复用**。

## 13.2 安装依赖

```bash
pip install mcp -i https://pypi.tuna.tsinghua.edu.cn/simple
# 预期：Successfully installed mcp-... 无 error
```

并在 `requirements.txt` 追加一行：

```text
mcp>=1.2
```

## 13.3 MCP Server：app/mcp/server.py（完整内容）

**做什么/为什么**：

- 每个工具 = `@mcp.tool()` 装饰一个 async 函数，**函数签名（类型注解）+ docstring 就是给 LLM 看的 schema**——描述写得越清楚，智能体调用得越准；
- **所有日志必须走 stderr**：stdio 通道承载协议报文，一行 `print`（stdout）就会污染协议导致客户端断连（这是最高频的 MCP 踩坑，FAQ 第 15 条）；
- `trigger_rootcause` 与 `app/services/rootcause.py` 的 `analyze()` 是同一套校验与调用语义（校验批次存在 → POST `/agent/analyze` 拿 Top-3 根因），但走**只读视角的轻量通道**：不落 `agent_tasks`/`suggestions` 表，避免智能体一次误调用就污染业务库——要完整闭环（任务留痕、建议待审）请走 REST 的 `/api/v1/rootcause/analyze`。

```python
"""
industryagent MCP Server（stdio 传输）
暴露工具：查询晶圆批次 / 查询异常事件 / 触发根因分析

与 web server（uvicorn :7001）是两个独立进程：
- 共享 app/models 的 SQLAlchemy 模型与 app/core/config 的 Settings（import 即复用）
- 不占用 HTTP 端口；客户端（Cursor 等）通过 stdin/stdout 与它对话
"""
import logging
import sys
from datetime import datetime
from decimal import Decimal

import httpx
from sqlalchemy import func, select

from app.core.config import get_settings
from app.core.database import AsyncSessionLocal
from app.models.anomaly_event import AnomalyEvent
from app.models.wafer import Wafer
from mcp.server.fastmcp import FastMCP

# ---------- 日志：必须走 stderr！stdio 通道承载协议报文，一行 print 就会打爆协议 ----------
logging.basicConfig(
    stream=sys.stderr,
    level=logging.INFO,
    format="%(asctime)s | %(levelname)-7s | %(name)s | %(message)s",
)
logger = logging.getLogger("ia.mcp")

settings = get_settings()
mcp = FastMCP("industryagent-server")


def _row_to_dict(row) -> dict:
    """ORM 行 -> 普通 dict。

    - Decimal 转 str：保留 "91.20" 原样（numeric 契约与 REST 接口一致，FAQ 第 14 条）
    - datetime 转 ISO 字符串：LLM 与客户端都能直接读
    """
    data = {}
    for key, value in row._mapping.items():
        if isinstance(value, Decimal):
            data[key] = str(value)
        elif isinstance(value, datetime):
            data[key] = value.isoformat()
        else:
            data[key] = value
    return data


# ---------- 工具 1：查询晶圆批次 ----------
@mcp.tool()
async def query_wafers(batch_no: str = "", page: int = 1, page_size: int = 10) -> dict:
    """根据批次号（模糊匹配）分页查询晶圆批次主数据。

    返回 {rows, total, page, pageSize}，rows 内字段：
    batchNo(批次号)、productModel(产品型号)、processNode(工艺节点)、
    productionLine(产线)、quantity(数量)、defectCount(缺陷数)、
    yieldRate(良率%，字符串如 "91.20")、status(状态)。
    状态枚举：producing(在制)/completed(完工)/shipped(出货)/quarantined(隔离)。
    """
    page = max(page, 1)
    page_size = min(max(page_size, 1), 50)

    stmt = select(Wafer).order_by(Wafer.id.desc())
    count_stmt = select(func.count()).select_from(Wafer)
    if batch_no:
        stmt = stmt.where(Wafer.batchNo.ilike(f"%{batch_no}%"))
        count_stmt = count_stmt.where(Wafer.batchNo.ilike(f"%{batch_no}%"))
    stmt = stmt.offset((page - 1) * page_size).limit(page_size)

    async with AsyncSessionLocal() as db:
        total = (await db.execute(count_stmt)).scalar() or 0
        rows = (await db.execute(stmt)).scalars().all()

    return {
        "rows": [_row_to_dict(r) for r in rows],
        "total": total,
        "page": page,
        "pageSize": page_size,
    }


# ---------- 工具 2：查询异常事件 ----------
@mcp.tool()
async def query_anomalies(limit: int = 10) -> list:
    """查询产线异常事件（FDC 报警），按发生时间倒序。

    返回字段：eventCode(事件编号)、batchNo(关联批次)、equipment(设备)、
    processStage(工序)、severity(严重度)、title(标题)、status(状态)。
    严重度枚举：critical/major/minor；状态枚举：open(待处理)/processing(处理中)/closed(已关闭)。
    """
    limit = min(max(limit, 1), 50)
    async with AsyncSessionLocal() as db:
        rows = (await db.execute(
            select(AnomalyEvent).order_by(AnomalyEvent.occurredAt.desc()).limit(limit)
        )).scalars().all()
    return [_row_to_dict(r) for r in rows]


# ---------- 工具 3：触发根因分析 ----------
@mcp.tool()
async def trigger_rootcause(anomaly_id: int) -> dict:
    """对指定异常事件触发良率根因分析：先校验事件与批次存在，
    再调用智能体服务，返回 Top-3 根因嫌疑（factor/置信度/描述）与整改建议。

    参数 anomaly_id 是异常事件 id（可先用 query_anomalies 查到）。分析耗时约 2~5 秒。
    """
    async with AsyncSessionLocal() as db:
        event = await db.get(AnomalyEvent, anomaly_id)
        if not event:
            return {"isError": True,
                    "message": f"异常事件不存在（id={anomaly_id}），请先用 query_anomalies 确认"}
        if not event.batchNo:
            return {"isError": True,
                    "message": f"事件 {event.eventCode} 未关联批次，无法分析"}

        wafer = (await db.execute(
            select(Wafer).where(Wafer.batchNo == event.batchNo)
        )).scalars().first()
        if not wafer:
            return {"isError": True,
                    "message": f"批次 {event.batchNo} 不存在，请用 query_wafers 确认批次号"}

        # 会话关闭前把值取出来（ detached 实例的已加载属性虽可读，显式取出更稳 ）
        batch_no = event.batchNo
        yield_rate = float(wafer.yieldRate)

    # 调用智能体：与 app/services/rootcause.py 的 _call_agent 同一套语义
    try:
        async with httpx.AsyncClient(timeout=30) as client:
            resp = await client.post(
                f"{settings.agent_base_url}/agent/analyze",
                json={"batch_no": batch_no, "yield_rate": yield_rate, "top_k": 3},
            )
        if resp.status_code != 200:
            return {"isError": True, "message": f"智能体服务返回 HTTP {resp.status_code}"}
        return resp.json()
    except httpx.HTTPError as e:
        return {"isError": True, "message": f"智能体服务调用失败：{e}"}


if __name__ == "__main__":
    logger.info("[mcp] industryagent MCP Server 已启动（stdio），工具数：3")
    mcp.run(transport="stdio")
```

## 13.4 用 MCP 客户端验证：scripts/mcp_client.py（完整内容）

**做什么/为什么**：`stdio_client` 会把 server 当**子进程**拉起，通过 stdin/stdout 完成 JSON-RPC 握手（`initialize`）→ `tools/list` → `tools/call`——与 Cursor 等客户端的行为完全一致。这个脚本是我们的"验收测试"。

```python
"""
MCP 客户端验证脚本：
1. 拉起 server 子进程并完成 JSON-RPC 握手（initialize）
2. 列出 server 暴露的全部工具（tools/list）
3. 依次调用 query_wafers / query_anomalies / trigger_rootcause（tools/call）
"""
import asyncio
import os
import sys

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

PROJECT_ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))


async def main():
    # 与 Cursor 等客户端一致：把 server 当子进程拉起，走 stdin/stdout 对话
    params = StdioServerParameters(
        command=sys.executable,                              # 当前 venv 的 python.exe
        args=[os.path.join("app", "mcp", "server.py")],
        cwd=PROJECT_ROOT,                                    # .env / 相对导入依赖项目根
    )

    async with stdio_client(params) as (read_stream, write_stream):
        async with ClientSession(read_stream, write_stream) as session:
            # 1. JSON-RPC 握手
            await session.initialize()
            print("已连接 MCP Server ✔")

            # 2. 列出工具
            tools = (await session.list_tools()).tools
            print("\n=== 工具列表 ===")
            for t in tools:
                desc = (t.description or "").splitlines()[0]
                print(f"- {t.name}: {desc[:50]}...")

            # 3. 查询晶圆批次
            print("\n=== 调用 query_wafers ===")
            result = await session.call_tool("query_wafers", {"batch_no": "B20261015001"})
            print(result.content[0].text)

            # 4. 查询异常事件
            print("\n=== 调用 query_anomalies ===")
            result = await session.call_tool("query_anomalies", {"limit": 5})
            print(result.content[0].text)

            # 5. 触发根因分析（需要智能体 Mock 在 8300 端口跑着）
            print("\n=== 调用 trigger_rootcause（约 2 秒） ===")
            result = await session.call_tool("trigger_rootcause", {"anomaly_id": 1})
            print(result.content[0].text)

    print("\n验证完成 ✔")


if __name__ == "__main__":
    asyncio.run(main())
```

## 13.5 验证

前提：PG 可连（MCP 直连 PG，不依赖 Redis 和 uvicorn）；`trigger_rootcause` 需要智能体 Mock 在跑（第 8 章，端口 **8300**）。

```bash
cd ~/code/industryagent/industryagent-server   # 项目根
python scripts/mcp_client.py
```

预期输出（节选；FastMCP 返回紧凑单行 JSON，字段与原版一致）：

```text
已连接 MCP Server ✔

=== 工具列表 ===
- query_wafers: 根据批次号（模糊匹配）分页查询晶圆批次主数据。...
- query_anomalies: 查询产线异常事件（FDC 报警），按发生时间倒序。...
- trigger_rootcause: 对指定异常事件触发良率根因分析：先校验事件与批次存在，...

=== 调用 query_wafers ===
{"rows": [{"id": 1, "batchNo": "B20261015001", "productModel": "MCU-XS32", "processNode": "28nm", "productionLine": "LINE-1", "quantity": 25, "defectCount": 4, "yieldRate": "91.20", "status": "completed", ...}], "total": 1, "page": 1, "pageSize": 10}

=== 调用 query_anomalies ===
[{"id": 1, "eventCode": "AE-20261015-001", "batchNo": "B20261015001", "equipment": "ETCH-03", "processStage": "etch", "severity": "critical", "title": "刻蚀功率漂移超标", "status": "open", ...}, ...]

=== 调用 trigger_rootcause（约 2 秒） ===
{"batch_no": "B20261015001", "summary": "批次 B20261015001 良率异常，疑似刻蚀工序参数漂移（结合知识图谱与近 7 天 FDC 数据）", "root_causes": [{"rank": 1, "factor": "etch_power_drift", "confidence": 0.87, ...}], "suggestions": [...]}

验证完成 ✔
```

## 13.6 接入智能体客户端（Cursor / Claude Desktop / Claude Code）

Cursor：项目根 `.cursor/mcp.json`：

```json
{
  "mcpServers": {
    "industryagent": {
      "command": "python",
      "args": ["app/mcp/server.py"],
      "env": {
        "IA_DB_HOST": "127.0.0.1",
        "IA_DB_PORT": "5432",
        "IA_DB_NAME": "industryagent",
        "IA_DB_USER": "industry",
        "IA_DB_PASSWORD": "industry123",
        "IA_AGENT_BASE_URL": "http://127.0.0.1:8300"
      }
    }
  }
}
```

Claude Desktop（配置文件 `claude_desktop_config.json`，路径见其设置页）：

```json
{
  "mcpServers": {
    "industryagent": {
      "command": "python",
      "args": ["app/mcp/server.py"],
      "env": {
        "IA_DB_HOST": "127.0.0.1",
        "IA_DB_PORT": "5432",
        "IA_DB_NAME": "industryagent",
        "IA_DB_USER": "industry",
        "IA_DB_PASSWORD": "industry123",
        "IA_AGENT_BASE_URL": "http://127.0.0.1:8300"
      }
    }
  }
}
```

Claude Code（项目根执行一次即可）：

```bash
claude mcp add industryagent -- python app/mcp/server.py
```

> 三个客户端的注意点：
>
> 1. **客户端不会帮你激活 venv**——`command: "python"` 解析到的是系统 PATH 里的解释器，若 `mcp` 装在项目 venv 里就会起不来。最稳的写法是把 `command` 换成 venv 绝对路径：`"F:\\code\\industryagent\\industryagent-server\\.venv\\Scripts\\python.exe"`（Windows JSON 双反斜杠）。
> 2. **`cwd` 不保证是项目根**——Cursor/Claude Desktop 拉起子进程的工作目录不可控，所以配置里用 `env` 显式带上数据库变量（.env 读不到时不至于连错库）；能设 `cwd` 的客户端就设成项目根。
> 3. 配置后**重启客户端**，工具列表里出现三个工具即接入成功。

接入后即可在对话中直接提问验证：

> "帮我查一下 LINE-1 有哪些未关闭的严重异常，然后对涉及的批次做根因分析。"

智能体会自动调用 `query_anomalies` → `query_wafers` → `trigger_rootcause` 完成任务。**这就是手册架构图中"FastAPI 业务后端 → MCP Server → 智能体工具链"的实机落地。**

再强调一次进程边界：MCP server 与 web server **分进程**、互不影响——uvicorn 重启不影响 Cursor 里的工具会话，MCP server 崩了也不影响 REST 服务；两者共享的只有 `app/models/` 与 `app/core/config.py` 这两份"事实源"。

---

# 14. 日志与可观测基础

## 14.1 做什么 / 为什么

手册工程化要求：可观测（日志、指标、追踪）。第 6 章已经实现了结构化 JSON 单行请求日志 + traceId（contextvars + `X-Trace-Id` 响应头），但那是**请求层**的日志。本章补齐两块：

1. **分环境 logging 体系**：Python 标准库 `logging` 功能完备但需要自己装配——dev 全量进控制台，prod 落文件并轮转。所有模块的 logger（含第 6 章的请求日志、审计日志）统一到同一套格式：**时间 | 级别 | traceId | 模块 | 内容**；
2. **Prometheus 指标与探活端点**：QPS / 延迟分布 / 状态码分布（`prometheus-client`），外加带依赖检查的 `/healthz`——为接 Grafana 与容器健康检查打地基。

## 14.2 分环境 logging：setup_logging()（完整内容）

**做什么/为什么**：追加到 `app/core/config.py` 末尾（配置与日志都按环境分层，放同一个文件最直观，不再新开模块）。dev（`IA_ENV=local`）只有控制台；prod（`IA_ENV=prod`）追加三个 `RotatingFileHandler`（10MB × 5 份轮转，防止日志吃满磁盘）。

```python
# ---------- 追加到 app/core/config.py 末尾 ----------
import logging
import logging.handlers
import sys
from pathlib import Path

# 结构化单行格式：时间 | 级别 | traceId | 模块 | 内容
LOG_FORMAT = "%(asctime)s | %(levelname)-7s | %(trace_id)s | %(name)s | %(message)s"


class TraceIdFilter(logging.Filter):
    """把 contextvars 里的 traceId 注入每条日志记录（第 6 章中间件负责 set）"""

    def filter(self, record: logging.LogRecord) -> bool:
        # 局部导入：避免 core.config 与 middleware 在模块期互相 import
        from app.middleware.request_log import trace_id_var
        record.trace_id = trace_id_var.get() or "-"
        return True


def setup_logging(settings: "Settings") -> None:
    """装配全局日志：local=控制台 DEBUG；prod=控制台 WARNING + 文件轮转"""
    root = logging.getLogger()
    if root.handlers:
        # 已配置过（测试进程反复 create_app）则跳过，防止日志重复打 N 遍
        return

    root.setLevel(logging.DEBUG if settings.env == "local" else logging.INFO)

    # uvicorn 自带 handler，关掉向 root 的传播——否则我们的 root handler 会把它再打一遍
    for name in ("uvicorn", "uvicorn.error", "uvicorn.access"):
        logging.getLogger(name).propagate = False

    formatter = logging.Formatter(LOG_FORMAT)
    trace_filter = TraceIdFilter()

    console = logging.StreamHandler(sys.stdout)
    console.setLevel(logging.DEBUG if settings.env == "local" else logging.WARNING)
    console.setFormatter(formatter)
    console.addFilter(trace_filter)
    root.addHandler(console)

    if settings.env == "prod":
        log_dir = Path("logs")
        log_dir.mkdir(exist_ok=True)

        # 应用日志：结构化请求日志（第 6 章）+ 业务日志
        app_handler = logging.handlers.RotatingFileHandler(
            log_dir / "app.log", maxBytes=10 * 1024 * 1024, backupCount=5, encoding="utf-8")
        app_handler.setLevel(logging.INFO)
        app_handler.setFormatter(formatter)
        app_handler.addFilter(trace_filter)
        root.addHandler(app_handler)

        # 错误日志：堆栈单独收口，排障第一入口
        error_handler = logging.handlers.RotatingFileHandler(
            log_dir / "error.log", maxBytes=10 * 1024 * 1024, backupCount=5, encoding="utf-8")
        error_handler.setLevel(logging.ERROR)
        error_handler.setFormatter(formatter)
        error_handler.addFilter(trace_filter)
        root.addHandler(error_handler)

        # 审计专用文件：第 6 章 audit logger 的"双保险"落点（表被误清文件还在）
        audit = logging.getLogger("audit")
        audit.propagate = False
        audit_handler = logging.handlers.RotatingFileHandler(
            log_dir / "audit.log", maxBytes=10 * 1024 * 1024, backupCount=5, encoding="utf-8")
        audit_handler.setFormatter(logging.Formatter("%(asctime)s | %(message)s"))
        audit.addHandler(audit_handler)
```

在 `app/main.py` 的 `create_app()` **开头**调用一行：

```python
    settings = get_settings()
    setup_logging(settings)     # 越早越好：后续所有组件的日志都走这套配置
```

日志分级速览：

| 通道 | 位置 | 级别（local / prod） | 内容 |
|---|---|---|---|
| 控制台 | stdout | DEBUG / WARNING | 开发终端实时看；生产容器由 Docker 收集 |
| 应用文件 | `logs/app.log` | — / INFO | 第 6 章结构化请求日志 + 业务日志，10MB×5 轮转 |
| 错误文件 | `logs/error.log` | — / ERROR | 错误堆栈单独收口 |
| 审计文件 | `logs/audit.log` | — / INFO | 第 6 章审计双保险（只追加，合规导出用） |
| uvicorn 访问日志 | 控制台 | INFO | 框架自带（access log），不参与轮转 |

**验证（local）**：重启服务后发一个请求，终端应出现带 traceId 的行。

```bash
uvicorn app.main:app --port 7001
# 新终端：
curl -s http://127.0.0.1:7001/api/v1/wafers -H "Authorization: Bearer $TOKEN" > /dev/null
```

终端预期输出（第二列后面的十六进制串就是该请求的 traceId）：

```text
2026-10-17 10:00:00,123 | INFO    | 6f9c1b2a4d8e...e2 | app.request | {"ts":"2026-10-17T02:00:00.123Z","kind":"http_request","traceId":"6f9c1b2a4d8e...e2","method":"GET","path":"/api/v1/wafers","status":200,"durationMs":18,...}
```

**验证（prod 文件轮转）**：

```bash
IA_ENV=prod uvicorn app.main:app --port 7001
curl -s http://127.0.0.1:7001/ > /dev/null
ls logs
# 预期：app.log  audit.log  error.log
```

## 14.3 Prometheus 指标：app/core/metrics.py 与中间件（完整内容）

`app/core/metrics.py`（完整内容）——指标对象全进程**只注册一次**，中间件与 `/metrics` 端点共用，所以独立成模块：

```python
"""Prometheus 指标注册（全进程唯一份，中间件与 /metrics 端点共用）"""
from prometheus_client import Counter, Histogram

# Counter：请求总量（配合 rate() 求 QPS）
HTTP_REQUESTS_TOTAL = Counter(
    "http_requests_total",
    "HTTP 请求总数",
    labelnames=["method", "route", "status"],
)

# Histogram：延迟分布（P50/P90/P95/P99 由 Prometheus 服务端计算）
HTTP_REQUEST_DURATION_SECONDS = Histogram(
    "http_request_duration_seconds",
    "HTTP 请求耗时（秒）",
    labelnames=["method", "route", "status"],
    buckets=(0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5),
)
```

安装依赖并追加到 `requirements.txt`：

```bash
pip install prometheus-client -i https://pypi.tuna.tsinghua.edu.cn/simple
```

```text
prometheus-client>=0.21
```

`app/middleware/metrics.py`（完整内容）：

```python
"""
Prometheus 指标中间件：
- route 标签用路由模板（/api/v1/wafers/{id}）而非真实路径，
  避免每个 id 都成一个新标签导致标签基数爆炸（Prometheus 最佳实践红线）
"""
import time

from starlette.middleware.base import BaseHTTPMiddleware

from app.core.metrics import HTTP_REQUEST_DURATION_SECONDS, HTTP_REQUESTS_TOTAL

EXCLUDED_PATHS = {"/metrics", "/healthz"}


class MetricsMiddleware(BaseHTTPMiddleware):

    async def dispatch(self, request, call_next):
        # /metrics、/healthz 自身不统计：前者避免自激励，后者避免探活刷量
        if request.url.path in EXCLUDED_PATHS:
            return await call_next(request)

        start = time.perf_counter()
        status = 500
        try:
            response = await call_next(request)
            status = response.status_code
            return response
        finally:
            # 路由匹配后 FastAPI 会把路由对象放进 scope["route"]，
            # route.path 即模板（含 {id} 占位符）；拿不到（如 404）时退回原始路径
            route = request.scope.get("route")
            path_template = route.path if route else request.url.path
            labels = {
                "method": request.method,
                "route": path_template,
                "status": str(status),
            }
            HTTP_REQUESTS_TOTAL.labels(**labels).inc()
            HTTP_REQUEST_DURATION_SECONDS.labels(**labels).observe(time.perf_counter() - start)
```

## 14.4 /metrics 与 /healthz 端点（完整内容）

`app/main.py` 的 `create_app()` 中，在路由挂载之后追加（中间件注册 + 两个运维端点）：

```python
from datetime import datetime, timezone

import httpx
from prometheus_client import CONTENT_TYPE_LATEST, generate_latest
from sqlalchemy import text
from starlette.responses import JSONResponse, Response

from app.core.database import AsyncSessionLocal
from app.core.redis import get_redis_client      # 第 7 章创建的 redis.asyncio 客户端
from app.middleware.metrics import MetricsMiddleware


def create_app() -> FastAPI:
    settings = get_settings()
    setup_logging(settings)
    app = FastAPI(...)
    # ...（第 3~13 章的中间件、异常处理、路由挂载）...

    # ---------------- 运维端点 ----------------
    app.add_middleware(MetricsMiddleware)

    @app.get("/metrics")
    def metrics_endpoint() -> Response:
        """GET /metrics：Prometheus 抓取端点"""
        return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)

    @app.get("/healthz")
    async def healthz() -> JSONResponse:
        """GET /healthz：探活端点（PG SELECT 1 + Redis ping + 可选智能体探活）"""
        checks: dict = {}

        try:
            async with AsyncSessionLocal() as db:
                await db.execute(text("SELECT 1"))
            checks["database"] = "ok"
        except Exception:
            checks["database"] = "error"

        try:
            await get_redis_client().ping()
            checks["redis"] = "ok"
        except Exception:
            checks["redis"] = "error"

        # 可选：连带探活智能体服务（默认关——它宕了不该拖垮业务后端的探活）
        if settings.health_check_agent:
            try:
                async with httpx.AsyncClient(timeout=2) as client:
                    resp = await client.get(f"{settings.agent_base_url}/healthz")
                checks["agent"] = "ok" if resp.status_code == 200 else "error"
            except Exception:
                checks["agent"] = "error"

        healthy = all(v == "ok" for v in checks.values())
        return JSONResponse(
            status_code=200 if healthy else 503,
            content={
                "status": "ok" if healthy else "error",
                "checks": checks,
                "ts": datetime.now(timezone.utc).isoformat(),
            },
        )

    return app
```

`Settings`（`app/core/config.py`）追加一个开关字段：

```python
    # /healthz 是否连带探活智能体服务（IA_HEALTH_CHECK_AGENT=true 开启）
    health_check_agent: bool = False
```

> 两个设计取舍：
> 1. **任一依赖失败 → 503**：编排器（compose/K8s）据此摘除流量；`checks` 里逐项给出原因，人看得懂；
> 2. **agent 检查默认关闭**：智能体是"增强能力"不是"存活前提"，根因分析还有熔断器兜底（第 8 章）——是否把它纳入探活由部署者决定。

## 14.5 验证

先登录拿 token，并制造一些流量：

```bash
TOKEN=$(curl -s -X POST http://127.0.0.1:7001/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"Admin@123"}' \
  | python -c "import sys,json;print(json.load(sys.stdin)['data']['accessToken'])")

for i in $(seq 1 5); do
  curl -s http://127.0.0.1:7001/api/v1/wafers -H "Authorization: Bearer $TOKEN" > /dev/null; done

curl -s http://127.0.0.1:7001/metrics | grep -E 'http_requests_total|http_request_duration_seconds_bucket' | head -n 6
```

预期输出：

```text
# HELP http_requests_total HTTP 请求总数
# TYPE http_requests_total counter
http_requests_total{method="GET",route="/api/v1/wafers",status="200"} 5
# HELP http_request_duration_seconds HTTP 请求耗时（秒）
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",route="/api/v1/wafers",status="200",le="0.01"} 5
...
```

注意 `route="/api/v1/wafers"` 是模板而非带 id 的真实路径——查详情 100 次也只会累积到 `/api/v1/wafers/{id}` 这一个标签上。

**验证健康检查**（临时停掉 Redis 容器看降级效果）：

```bash
curl -s http://127.0.0.1:7001/healthz
# 正常预期：{"status":"ok","checks":{"database":"ok","redis":"ok"},"ts":"..."}

docker stop ia-redis
curl -si http://127.0.0.1:7001/healthz | head -n 1
# 预期：HTTP/1.1 503 Service Unavailable

curl -s http://127.0.0.1:7001/healthz
# 预期：{"status":"error","checks":{"database":"ok","redis":"error"},"ts":"..."}

docker start ia-redis   # 恢复
```

> 接入 Prometheus 只需在其 `prometheus.yml` 中加抓取任务 `targets: ['<server-host>:7001']`，Grafana 即可出 QPS/P95 面板——属于运维教程范畴，这里打牢数据地基即可。

---

# 15. 构建与部署

## 15.1 做什么 / 为什么

`uvicorn app.main:app --reload` 是开发模式直跑源码，性能与安全都不适合生产。生产路径：**测试全绿 + 语法编译检查 → 多阶段 Docker 镜像 → docker-compose 编排**（PG + Redis + server 一条命令起全套）。

Python 没有 TS 那样的「编译到 dist」步骤，原版的 `npm run build` 对应物是两道检查：`pytest`（行为正确）+ `python -m compileall`（语法可编译，能提前暴露拼接错文件、中文引号混入这类低级错误）。真正的"构建"发生在 Docker 多阶段镜像里：builder 阶段把依赖装进独立 venv，runtime 阶段只拷贝 venv 与业务代码。

## 15.2 构建前检查：测试全绿 + 语法编译 + 冒烟

```bash
pytest
# 预期：9 passed（一行绿；详细的 -v 输出见 12.6）

python -m compileall -q app
# 预期：无任何输出、退出码 0（-q 静默模式；出现 SyntaxError 会打印文件行号并返回非 0）
```

再做一次 30 秒冒烟（确认本地环境完好，后面的运行交给容器）：

```bash
uvicorn app.main:app --port 7001
# 新终端：
curl -s http://127.0.0.1:7001/healthz
# 预期：{"status":"ok","checks":{"database":"ok","redis":"ok"},"ts":"..."}
# 确认后 Ctrl+C 停掉
```

## 15.3 Dockerfile 与 .dockerignore（完整内容，多阶段构建）

`Dockerfile`（完整内容）：

```dockerfile
# ========== 阶段 1：builder（把全部依赖装进独立 venv） ==========
FROM python:3.12-slim AS builder
WORKDIR /app
ENV PIP_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple \
    PIP_NO_CACHE_DIR=1

# 先只拷贝依赖清单，充分利用 Docker 层缓存（改业务代码不重装依赖）
COPY requirements.txt ./
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --upgrade pip && pip install -r requirements.txt

# ========== 阶段 2：runtime（venv 整体拷贝 + 非 root 运行） ==========
FROM python:3.12-slim
ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1
WORKDIR /app

# 非 root 运行（企业级安全基线：容器内进程拿不到 root 身份）
RUN useradd --create-home appuser

COPY --from=builder /opt/venv /opt/venv
COPY app ./app

USER appuser

EXPOSE 7001

# 容器级健康检查：/healthz 非 200（503 会抛 HTTPError）即判定不健康
# 方案 A（推荐）：slim 镜像不装 curl，用标准库 urllib 探活，零额外体积
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:7001/healthz', timeout=3)"]

# 生产启动：单进程 uvicorn（多 worker 压测场景加 --workers N）
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "7001"]
```

> **HEALTHCHECK 方案 B（装 curl）**：如果团队习惯用 curl，在 runtime 阶段加
> `RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*`
> （镜像增大约 30MB），健康检查改 `CMD curl -sf http://127.0.0.1:7001/healthz || exit 1`。本教程用方案 A——Python 镜像里自带"探活工具"。
>
> **第 18 章预告**：WS 上线后服务由 `app/run.py` 双 Server（7001 REST + 7002 WS）接管，届时把 CMD 换成 `["python", "-m", "app.run"]`、EXPOSE 追加 7002 即可，镜像结构不变。

`.dockerignore`（完整内容）：

```text
.git
.venv
__pycache__/
*.pyc
.pytest_cache/
.coverage
logs/
.env
tests/
scripts/
agent-mock/
*.md
```

> **注意**：`.env` 被 ignore，容器内的配置全部走 docker-compose 的 `environment` 注入——这正是第 4 章"代码与环境隔离"设计的回报。`scripts/` 不进镜像：init_db.py 在宿主机执行（15.5 节），mcp_client.py 属于客户端侧工具。

## 15.4 docker-compose.yml（完整内容）

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: ia-compose-pg
    environment:
      POSTGRES_USER: industry
      POSTGRES_PASSWORD: industry123
      POSTGRES_DB: industryagent
    ports:
      - "15432:5432"          # 宿主机错峰端口：避免与第 2 章单容器 ia-pg 的 5432 冲突
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U industry -d industryagent"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7-alpine
    container_name: ia-compose-redis
    ports:
      - "16379:6379"          # 同理错峰：两套环境（单容器 vs compose）可同时共存
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

  server:
    build: .
    container_name: ia-compose-server
    depends_on:
      postgres:
        condition: service_healthy   # 等 PG 通过健康检查才起 server（避免启动期连库失败）
      redis:
        condition: service_healthy
    environment:
      IA_ENV: prod
      IA_JWT_SECRET: prod-jwt-secret-change-me-please
      IA_DB_HOST: postgres          # 服务名即容器网络内的主机名
      IA_DB_PORT: "5432"            # 注意是容器内端口，不是宿主机错峰端口
      IA_DB_NAME: industryagent
      IA_DB_USER: industry
      IA_DB_PASSWORD: industry123
      IA_REDIS_HOST: redis
      IA_REDIS_PORT: "6379"
      IA_AGENT_BASE_URL: http://host.docker.internal:8300   # 智能体 Mock 跑在宿主机
    ports:
      - "7001:7001"

volumes:
  pgdata:
```

> - 新版 docker compose 已废弃 `version:` 字段（写了会告警），直接以 `services:` 开头。
> - **错峰端口设计说明**：宿主机 `15432/16379` 映射容器 `5432/6379`，与第 2 章起的单容器环境（`ia-pg`:5432、`ia-redis`:6379）互不抢占——开发库与部署验证可以同时在线；server 容器**网络内**访问的是 `postgres:5432`（同 compose 网络不走宿主机映射）。
> - **Windows / macOS 说明**：容器内访问宿主机服务用 `host.docker.internal`；Linux 上需在 server 服务加 `extra_hosts: ["host.docker.internal:host-gateway"]`。

## 15.5 一键起全套并验证

```bash
docker compose up --build -d
docker compose ps
```

预期输出（三个容器 healthy/up）：

```text
NAME                 STATUS                   PORTS
ia-compose-pg        Up 2 minutes (healthy)   0.0.0.0:15432->5432/tcp
ia-compose-redis     Up 2 minutes (healthy)   0.0.0.0:16379->6379/tcp
ia-compose-server    Up 1 minute (healthy)    0.0.0.0:7001->7001/tcp
```

初始化表结构（第 5 章脚本指向 compose 的 PG——宿主机侧走错峰端口 15432）：

```bash
IA_DB_PORT=15432 python scripts/init_db.py
# 预期：[seed] 全部完成 ✔（数据落在 compose 的 PG 卷里）
```

> 环境变量优先级：**shell 里显式设置的 `IA_DB_PORT` 高于 `.env` 文件**（pydantic-settings 的规则），所以不需要临时改 `.env`——这是原 Egg 版 dotenv 同款行为。

验证服务：

```bash
curl -s http://127.0.0.1:7001/healthz
# 预期：{"status":"ok","checks":{"database":"ok","redis":"ok"},"ts":"..."}

curl -s http://127.0.0.1:7001/metrics | head -n 2
# 预期：# HELP ... 指标正常输出
```

**幂等重跑验证**：

```bash
docker compose up --build -d     # 再执行一次
# 预期：容器已存在则原地复用/重建变化项，不报错（幂等）

docker compose down              # 停止并删除容器
docker compose up -d             # 再起
curl -s http://127.0.0.1:7001/api/v1/wafers?page=1\&pageSize=1 -H "Authorization: Bearer $TOKEN"
# 预期：之前 seed 的批次还在（pgdata 数据卷保留，down 不删卷）
```

常用运维命令：

```bash
docker compose logs -f server      # 跟踪服务日志（prod 模式同时落容器内 logs/ 卷外不可见，看这里）
docker compose restart server      # 重启服务
docker compose down                # 停止并删除容器（数据卷保留）
docker compose down -v             # 连数据卷一起删（慎用：清库）
```

---

# 16. 常见坑与 FAQ

**1. asyncpg 连接池耗尽 / TooManyConnections**
现象：`sqlalchemy.exc.TimeoutError: QueuePool limit ... connection timed out`，或 PG 侧 `FATAL: sorry, too many clients already`。
原因：连接没归还（session 没 `async with`）、在事务里 `await` 外部慢服务（占着连接睡觉）、或多进程/多测试进程各建一套池叠加。
解法：一律 `async with AsyncSessionLocal() as db`（get_db 依赖已保证）；事务内不调外部 HTTP（智能体调用挪到事务外，第 8 章即如此）；池上限在 `app/core/database.py` 的 `create_async_engine(..., pool_size=, max_overflow=)` 调整；compose 的 PG 默认 `max_connections=100`，本文单 server 进程默认池完全够用。

**2. RuntimeError: Event loop is closed（Windows 高发）**
现象：第二个测试用例、或 uvicorn `--reload` 重启后报此错，有时伴随 `Task got Future attached to a different loop`。
原因：asyncpg/redis 的连接绑定"创建它的事件循环"，模块级 engine/客户端被跨循环复用。
解法：测试里每个用例后 `await engine.dispose()`（见 12.5 的 `_database` 夹具注释）；进程退出前主动 dispose（lifespan 的 shutdown 钩子）；绝不在请求处理中 `asyncio.run()` 新建循环。

**3. bcrypt 报 `AttributeError: module 'bcrypt' has no attribute '__about__'`**
原因：`passlib 1.7.4` 读取 `bcrypt.__about__.__version__`，而 bcrypt 4.x 删了这个属性——两者的老兼容坑。
解法：**本教程直接用 `bcrypt` 官方库**（第 7 章 `security.py` 的 `hashpw/checkpw`），不需要 passlib，别画蛇添足去装它。已中招的：`pip uninstall passlib` 即愈。

**4. JWT 过期后前端一片报错（ExpiredSignatureError）**
现象：access token 2 小时过期后，所有请求 401。
解法：这是设计使然。后端 `decode_token` 必须**单独捕获** `jwt.ExpiredSignatureError` 映射为 20002（TOKEN_EXPIRED），不要用宽泛的 `except Exception` 一把搂——会把 20002 吞成 20001，前端就无法区分"该刷新"与"该跳登录"。前端拦截 401 且业务码为 20002 时，静默调 `/api/v1/auth/refresh` 换新 token 并重放原请求；refresh 也失败才跳登录页。

**5. uvicorn --reload 在 Windows 监听不到子目录/新文件**
现象：新增文件或新建子目录（如 `app/mcp/`）后改动不触发重载。
原因：Windows 的文件系统事件对"新建目录内文件"通知不稳定；路径含中文/空格（如 `F:\全栈项目\...`）时更明显。
解法：重启 uvicorn（新增**文件**本来就该重启）；项目放纯 ASCII 路径最稳；确认装的是 `uvicorn[standard]`（带 watchfiles，比轮询可靠）。

**6. 中文乱码 / 字符集**
现象：接口返回或日志里中文变问号、`\uXXXX`。
解法：分三处排查——① PG：`docker exec -it ia-pg psql ... -c "SHOW server_encoding;"` 应为 `UTF8`（官方镜像默认就是）；连接串或 `create_async_engine` 需要强制时加 `connect_args={"server_settings": {"client_encoding": "utf8"}}`；② 终端：cmd/PowerShell 执行 `chcp 65001`，或直接用 Git Bash（本教程口径）；③ 文件：日志 handler 都要 `encoding="utf-8"`（14.2 的 RotatingFileHandler 已设），手动 `open()` 也别省这个参数。

**7. 跨域 CORS 报错**
现象：浏览器控制台 `blocked by CORS policy`。
解法：第 7 章若未配置，在 `create_app()` 里加中间件（纯 token 认证的后端，跨域只放前端来源）：

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],   # 前端 dev 地址
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**注意**：`allow_origins=["*"]` 不能与 `allow_credentials=True` 同用（浏览器规范禁止），需要 cookie/凭证时必须写精确域名。

**8. 端口占用（uvicorn 起不来）**
现象：`[Errno 98] Address already in use`（Linux）或 `[WinError 10048]`（Windows）。
解法：上次 dev 进程没退干净。Windows：`netstat -ano | findstr 7001` 找到 PID（最后一列），`taskkill /PID <pid> /F`；Git Bash 也可 `npx killport 7001`；Linux/macOS：`lsof -ti:7001 | xargs kill -9`。

**9. Docker 容器内连不上宿主 Redis / 智能体**
现象：compose 里的 server 报 Redis 或 agent 连接拒绝。
解法：① 容器里访问宿主机服务用 `host.docker.internal`（Windows/macOS 自带；Linux 加 `extra_hosts: ["host.docker.internal:host-gateway"]`）；② 宿主机服务必须监听 `0.0.0.0`——智能体 Mock 用 `uvicorn main:app --host 0.0.0.0 --port 8300`，只听 127.0.0.1 时容器进不来；③ 别把 `IA_REDIS_PORT` 写成宿主机错峰端口 16379——server 在 compose 网络内直达 `redis:6379`。

**10. 时间差 8 小时**
现象：库里 `2026-10-15 10:00:00`，接口返回/查询条件差 8 小时。
解法：确认三处一致——PG 容器默认 UTC（要东八区可在 compose 的 postgres 服务加 `TZ: Asia/Shanghai`）；代码里生成时间统一 `datetime.now(timezone.utc)` 或带时区对象，别用裸 `datetime.now()`（asyncpg 对 naive datetime 按 UTC 处理，歧义源头）；前端传参统一带时区（`2026-10-15T08:00:00+08:00`）。

**11. traceId 不透传到 service / 日志里是 `-`**
现象：`X-Trace-Id` 响应头有值，但 service 层日志的 trace_id 列是 `-`。
原因：第 6 章 traceId 用 `contextvars` 存取、14.2 的 `TraceIdFilter` 读同一个 `trace_id_var`——断点通常在两处：logger 走的 handler 没挂 `TraceIdFilter`（自建的 logger 体系要补）；或者跨线程（`run_in_executor`）/自建事件循环里 context 不会自动复制。
解法：统一用 `logging.getLogger(...)` 挂到 root（自动继承 handler 与 filter）；跨线程时显式传 traceId 参数，或 `contextvars.copy_context()` 携带。

**12. pytest 异步用例没跑 / 警告 coroutine never awaited**
现象：用例显示 passed 但断言没执行，或告警 `coroutine 'test_xxx' was never awaited`。
原因：pytest 原生不认识 `async def` 用例——没配 `asyncio_mode` 时它们被当成普通函数"调用即通过"。
解法：确认 `pytest.ini` 里 `asyncio_mode = auto`（12.3 已配），且装了 `pytest-asyncio`。逐个标注 `@pytest.mark.asyncio` 是另一种流派，本项目统一 auto。

**13. Docker Desktop / WSL 未启动**
现象：`error during connect: ... docker daemon is not running`，或 Redis 报 `ConnectionError: Error 111 connecting to 127.0.0.1:6379`。
解法：先起 Docker Desktop（Windows）；`docker ps` 确认 `ia-pg`/`ia-redis` 在运行；compose 环境注意宿主机端口是错峰的 15432/16379，宿主机侧直连时 `IA_DB_PORT=15432`。PG/Redis 容器没起是"一切连接问题的第一嫌疑"。

**14. numeric/Decimal 字段与 JSON 序列化**
现象：`yieldRate` 手动 `json.dumps` 时报 `TypeError: Object of type Decimal is not JSON serializable`；或纠结返回 `91.2` 还是 `"91.20"`。
原因：PG 的 `numeric` 经 asyncpg 返回 `Decimal`（防精度丢失），属正常行为。
解法：三种口径按需选——schema 字段声明 `Decimal` 时 pydantic 序列化为字符串 `"91.20"`（与原版契约一致）；手动 `json.dumps(..., default=str)`；计算层 `float(row.yieldRate)`（第 10 章看板已示范）。MCP Server 的 `_row_to_dict` 用 `str()` 保持 `"91.20"`（13.3 节）。

**15. MCP Server 连上但工具调用全部失败 / 客户端直接断连**
原因：stdio 传输下，任何写到 stdout 的内容（`print`、某些库的调试输出）都会与 MCP 协议报文混流，一行就能打爆协议解析。
解法：MCP Server 进程内**所有日志走 stderr**（13.3 的 `logging.basicConfig(stream=sys.stderr)`），永远不要 `print`；第三方库若抢写 stdout，在其初始化时关掉 verbose 选项。

**16. Windows 下 curl 传 JSON 全是引号报错**
解法：本文所有 curl 命令请在 **Git Bash** 执行；PowerShell 用 `Invoke-RestMethod`；或用 Postman/Apifox。cmd.exe 的转义规则不适合 JSON。

**17. ModuleNotFoundError：明明 pip install 过**
原因：装与用不是同一个解释器——pip 落到了全局 Python，uvicorn/pytest 用的是 venv（或反之）。
解法：统一从 venv 内执行：Git Bash 先 `source .venv/Scripts/activate`（Windows 路径是 Scripts 不是 bin）；或始终 `python -m pip install ...`、`python -m uvicorn ...`、`python -m pytest`——`-m` 形式保证与当前 `python` 同源。CI/客户端拉起子进程时（如 Cursor 的 MCP 配置）用解释器**绝对路径**。
