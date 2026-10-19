# 1. 教程导读

## 1.1 学完你将获得什么

本教程带你从零搭建 **industryagent-server**——芯联半导体良率根因分析平台的业务后端。教程结束时，你将拥有一个具备以下能力的、可运行、可部署、可写进简历的企业级后端：

- 用户登录鉴权（JWT 双 Token + bcrypt 密码加密）
- 晶圆批次 / 缺陷数据完整 CRUD（Pydantic 参数校验 + 事务）
- 异常事件查询
- 根因分析任务触发（httpx 异步调用 Python 智能体服务，含超时 / 重试 / 熔断降级）
- 建议审批接口（事务 + 行锁 + 完整"分析 → 建议 → 审批"业务闭环）
- 审计日志中间件（谁 / 何时 / 做了什么 / 耗时）
- RBAC 权限模型（admin / 工艺工程师 / 审批员三种角色）
- Redis 应用（看板缓存 cache-aside + 接口幂等）
- 定时任务（APScheduler：传感器数据断流告警 + 良率日报）
- 单元测试与集成测试（pytest + pytest-asyncio + httpx）
- MCP Server（把查询异常事件 / 查询晶圆批次 / 触发根因分析注册为 MCP 工具，供智能体调用）
- 可观测基础（结构化日志 + traceId + Prometheus 指标）
- Docker 多阶段构建与 docker-compose 本地编排

## 1.2 为什么从 Egg.js 换成 FastAPI

如果你看过本教程的 V1.x 版（Egg.js 实现），第一个问题大概是：好好的为什么要重写？这不是"哪个框架更强"的问题，而是本平台走到 V2.0 后，三个工程收益变得非常明确：

**理由一：与智能体同栈，统一依赖、部署与人才技能。**
本平台的智能体服务（见《03-智能体开发教程》）从第一版起就是 Python + FastAPI——AI 应用生态（模型 SDK、MCP、结构化输出）的重心在 Python 是现实。后端换到同一技术栈后：

- 一套依赖体系：统一用 requirements.txt + pip 镜像，不再维护 npm 与 pip 两份"物料清单"；
- 一套部署模板：同一镜像基座（python:3.12-slim）、同一套 Dockerfile / compose 写法（第 15 章）；
- 一支队伍的技能：后端工程师能看懂并接手智能体服务，AI 工程师也能 review 后端代码；联调根因分析接口时不再跨语言切换心智。

**理由二：async 原生，对症 IO 密集场景。**
本后端是典型的 IO 密集服务——等 PostgreSQL、等 Redis、等智能体的 HTTP 响应。FastAPI 基于 ASGI + asyncio，从协议层就是异步；配合 SQLAlchemy 2.0（async）、asyncpg、redis.asyncio、httpx.AsyncClient，全链路 `async/await`：一个等待外部 IO 的请求不占用其他请求的处理能力，单进程即可从容承载教学与演示规模的并发（第 18 章 WebSocket 实时推送同样受益）。

**理由三：Pydantic 契约即文档。**
请求/响应模型（Pydantic）声明一次，自动获得两样东西：参数校验（错参直接 422，带精确到字段的报错）和交互式 API 文档（`/docs` Swagger UI、`/redoc`）。Egg 版需要 egg-validate 校验 + 手写接口文档两份并行维护，FastAPI 版天然一致——文档永远不会和代码打架。

顺带回答"为什么不是 Django / Flask"：Django 自带 admin、模板、同步 ORM 一整套全家桶，对本教程的纯 API 服务偏重；Flask 够轻但异步能力要靠扩展拼装。FastAPI 恰好命中"轻量 + 异步原生 + 自动文档"这个组合。

**兼容承诺（重要）**：V2.0 只换"引擎"，不动"契约"——路由与路径、统一响应包、错误码、种子数据、演示剧本与 V1.x **逐字节兼容**。前端教程、数据库教程、部署教程均不因换栈而修改。唯一的口径修正：智能体服务地址统一为 **8300**（V1.x 教程正文写的 8000 是历史分裂口径，现与《03-智能体开发教程》《04-部署上线教程》V1.1 对齐，详见 1.3 端口表）。

## 1.3 技术栈总览与端口口径

```text
                         ┌────────────────────────────────────────────┐
                         │        前端（React，见前端教程）            │
                         │  REST 7001（JSON）/ WS 7002（第 18 章起）   │
                         └───────────────────┬────────────────────────┘
                                             │ REST API (JSON)
                                             ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                    industryagent-server（本教程交付物）                     │
│                                                                            │
│  Python 3.12 + FastAPI + uvicorn                                           │
│  ├─ Router / Service / Model（SQLAlchemy 2.0 async + asyncpg + PG 16）     │
│  ├─ Pydantic v2 请求/响应契约（参数校验 + /docs 自动文档）                  │
│  ├─ 中间件：错误处理 / 审计日志 / 请求日志(traceId) / 鉴权 / 幂等 / 指标  │
│  ├─ JWT 鉴权 + RBAC（roles / user_roles，PyJWT + bcrypt）                  │
│  ├─ Redis（redis-py asyncio）：缓存 / 幂等 / refresh token                 │
│  ├─ 定时任务 APScheduler：数据健康检查 / 良率日报                           │
│  └─ MCP Server（mcp 官方 Python SDK，stdio）：暴露工具给智能体             │
└──────────────┬──────────────────────────────────────┬─────────────────────┘
               │ HTTP (httpx.AsyncClient)             │ MCP (stdio)
               ▼                                      ▼
┌──────────────────────────────┐       ┌──────────────────────────────────┐
│  Python 智能体服务 (FastAPI)  │       │  MCP 客户端（Cursor / Claude /    │
│  127.0.0.1:8300              │       │  Claude Code / 自研智能体）        │
│  /agent/analyze 根因分析      │       └──────────────────────────────────┘
└──────────────────────────────┘
```

全平台端口口径（本教程与前端、智能体、部署教程共同遵守，遇到端口问题先回这张表对）：

| 端口 | 用途 | 说明 |
| --- | --- | --- |
| 7001 | 后端 REST API | 本教程主服务（uvicorn #1） |
| 7002 | 后端 WS 推送 | 第 18 章通知/异常实时推送；与 7001 同进程、双 Server 启动（run.py） |
| 8300 | Python 智能体服务 | `/agent/analyze` 根因分析；第 8 章先用 agent-mock 占位 |
| 5432 | PostgreSQL 16 | 容器 ia-pg |
| 6379 | Redis 7 | 容器 ia-redis |

> MCP 走 stdio 子进程通信，无端口。MinIO（9000/9001）与 Keycloak（8080）分别在第 18、19 章引入，见《05-后端环境与软件安装教程》。

## 1.4 前置准备清单

| 项 | 要求 |
| ---------- | ---------------------------------------------------- |
| Python     | ≥ 3.12（`python --version` 验证；本教程在 3.12.8 上编写验证） |
| pip        | 会 `pip install` 即可，第 2 章配置国内镜像            |
| Python 语法 | 熟悉 `async/await`、类型标注（`typing` 基础即可，教程循序渐进） |
| SQL        | 会基本 SELECT / INSERT 即可                           |
| HTTP       | 理解请求/响应、Header、JSON（验证全靠 curl）           |
| Docker     | 用于一键拉起 PostgreSQL 16 与 Redis 7（详见《05-后端环境与软件安装教程》） |
| 操作系统    | macOS / Linux 命令为主，Windows 差异处单独注明（推荐用 Git Bash 执行本文命令） |

## 1.5 常见疑问与避坑

- **我跟过 V1.x（Egg.js 版），还要重学吗？** 业务与契约一模一样（错误码、响应包、种子数据逐字节兼容），想复习业务线可以快速翻阅；想真正掌握 FastAPI 实现，建议从头跟一遍——两版对照本身是极好的学习材料。
- **Python 零基础能跟吗？** 需要 `async/await` 与类型标注的基本概念（见 1.4）。依赖注入、Pydantic、lifespan 等新概念在首次出现的章节都会展开讲。
- **前端 / 数据库 / 部署教程会跟着变吗？** 不会。契约冻结；数据库表名、列名、种子数据原样沿用（第 5 章起建表时逐字段对齐）。
- **教程命令在哪里执行？** 所有命令默认：已激活虚拟环境、当前目录是项目根（monorepo 的 `server/`，见 2.4）。Windows 用户全程推荐 Git Bash。
- **为什么到处都找不到 8000 端口？** 已废弃。智能体统一 8300，见 1.3 端口表；若你机器上有 V1.x 遗留的 agent mock 还挂在 8000，直接停掉。

---

# 2. 环境准备

## 2.1 做什么 / 为什么

**做什么/为什么**：工欲善其事必先利其器。本教程需要四样东西：Python 3.12 运行时、隔离的虚拟环境（venv）、可用的 pip 国内镜像、以及 PostgreSQL（业务数据）与 Redis（缓存 / 幂等 / Token 白名单）两个数据库。

**不推荐**在宿主机直接安装 PostgreSQL 与 Redis——版本不一致、卸载残留、多项目冲突都是坑。用 Docker 一分钟拉起、一键销毁，是最干净的方式。Docker Desktop 的完整安装（WSL2、镜像加速、资源分配）与端口占用排查已在《05-后端环境与软件安装教程》讲透，本章只做"拉起 + 验证"，不重复。

## 2.2 安装并验证 Python 3.12

```bash
# 验证 Python 版本（需 >= 3.12）
python --version
# 预期输出类似：Python 3.12.8

# 查看当前用的是哪个 Python（后续排查"装错环境"全靠它）
which python
# 预期输出：Python 安装路径（Git Bash 下形如 /c/Users/<你>/AppData/Local/Programs/Python/Python312/python）

# 验证 pip
pip --version
# 预期输出类似：pip 25.x from ...Python312\lib\site-packages\pip (python 3.12)
```

若版本低于 3.12：到 [python.org](https://www.python.org/downloads/) 下载 3.12.x 安装包，Windows 安装时**务必勾选 "Add python.exe to PATH"**，装完重开终端再验证。

机器上有多个 Python 时，Windows 可用启动器精确指定：

```bash
py -3.12 --version
# 预期输出类似：Python 3.12.8
# 后续创建虚拟环境也可以用：py -3.12 -m venv .venv
```

## 2.3 配置 pip 国内镜像（清华源）

**做什么/为什么**：pip 默认源在国外，国内网络下安装又慢又容易超时。清华 TUNA 镜像每 5 分钟同步一次 PyPI，配置一次全局生效（写入用户级 pip 配置，与项目无关）。

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
# 预期输出：Writing to C:\Users\<你>\AppData\Roaming\pip\pip.ini

# 验证配置已生效
pip config list
# 预期输出：global.index-url='https://pypi.tuna.tsinghua.edu.cn/simple'
```

配置后再执行任何 `pip install`，输出第一行都会是 `Looking in indexes: https://pypi.tuna.tsinghua.edu.cn/simple`，说明走的是镜像。

## 2.4 目录规划

**做什么/为什么**：本系列教程的项目统一放在一个仓库目录下（monorepo 风格，但各自独立可运行）。

```bash
mkdir -p ~/code/industryagent && cd ~/code/industryagent
mkdir -p server agent docs web

ls ~/code/industryagent
# 预期输出：agent docs server web
```

```text
~/code/industryagent/
├── server/    # 本教程：Python FastAPI 业务后端（industryagent-server）
├── agent/     # 智能体教程：Python FastAPI 根因分析服务（8300 端口）
├── docs/      # 项目文档（本教程属于其中一部分）
└── web/       # 前端教程：React 管理台
```

> **Windows 用户**：路径形如 `D:\code\industryagent`，在 Git Bash 中为 `/d/code/industryagent`。下文所有 `~/code/industryagent` 请自行替换。

两点说明：

1. **`server/` 目录就是本教程的项目根**，项目名叫 industryagent-server。后文所有相对路径（`app/`、`scripts/`、`.env`…）都以 `server/` 为根（V1.x 教程"2.5 节规划 server/、3.2 节又新建 industryagent-server/"的自相矛盾在 V2.0 里修正掉了）。
2. 第 8 章会在 `server/` 里再建一个 `agent-mock/`（智能体 Mock 服务，跑 8300 端口），与 `app/` 并列放在同一仓库里维护——不需要动 monorepo 顶层的 `agent/`。

## 2.5 创建并激活虚拟环境（venv）

**做什么/为什么**：Python 的包是装给"某个解释器"的。不用虚拟环境，所有项目的依赖全挤在系统 Python 里，版本互相踩踏。venv 给每个项目一个独立的 `site-packages`，删掉 `.venv/` 目录即可整环境销毁——这是 Python 生态的标准做法，也是后面 Docker 化（容器内自带干净环境）的宿主机对应物。

```bash
cd ~/code/industryagent/server

# 在项目根创建虚拟环境（目录名约定为 .venv）
python -m venv .venv
# 预期输出：无输出即成功

ls -a
# 预期输出：.  ..  .venv
```

激活方式按终端选择（**注意 Windows 与 macOS/Linux 的目录差异：Windows 是 `Scripts/`，macOS/Linux 是 `bin/`**）：

```bash
# Git Bash（Windows）——本教程默认终端
source .venv/Scripts/activate
# 预期输出：命令提示符前出现 (.venv) 前缀，如 (.venv) <你>@<机器> MINGW64 ~/code/industryagent/server
```

```bash
# PowerShell（Windows）
.venv\Scripts\Activate.ps1
# 预期输出：提示符前出现 (.venv) 前缀

# CMD（Windows）
.venv\Scripts\activate.bat

# macOS / Linux（对照：路径是 bin 不是 Scripts）
# source .venv/bin/activate
```

激活后验证解释器确实来自虚拟环境：

```bash
python -c "import sys; print(sys.executable)"
# 预期输出：.../server/.venv/Scripts/python.exe（Windows）
#          .../server/.venv/bin/python（macOS/Linux）
```

退出虚拟环境用 `deactivate`（提示符前缀消失）。每开一个新终端都要重新激活——忘了激活就装包是最常见的坑，见 2.8。

## 2.6 用 Docker 拉起 PostgreSQL 16 与 Redis 7

**做什么/为什么**：后端第 5 章起要连 PostgreSQL，第 10 章起要连 Redis。Docker Desktop 的安装（WSL2、CPU 虚拟化、镜像加速、资源分配）与端口占用排查见《05-后端环境与软件安装教程》第 1、2 章，此处不重复；下面的启动命令与 05 教程完全一致（含数据卷与自动重启两项增强）。

```bash
# PostgreSQL 16：容器名 ia-pg，端口 5432，用户 industry / 密码 industry123，库 industryagent
docker run -d \
  --name ia-pg \
  --restart unless-stopped \
  -e POSTGRES_USER=industry \
  -e POSTGRES_PASSWORD=industry123 \
  -e POSTGRES_DB=industryagent \
  -p 5432:5432 \
  -v ia-pg-data:/var/lib/postgresql/data \
  postgres:16-alpine
# 预期输出：一长串容器 ID（64 位十六进制字符串）

# Redis 7：容器名 ia-redis，端口 6379
docker run -d \
  --name ia-redis \
  --restart unless-stopped \
  -p 6379:6379 \
  -v ia-redis-data:/data \
  redis:7-alpine
# 预期输出：一长串容器 ID
```

> 容器名（ia-pg / ia-redis）、端口、凭证必须与本教程一致：后续所有 `docker exec` 验证、`.env` 连接配置（第 4 章）都建立在这组约定上。

两个容器都在跑的样例：

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
# 预期输出：
# NAMES     STATUS         PORTS
# ia-pg     Up 10 seconds   0.0.0.0:5432->5432/tcp
# ia-redis  Up 8 seconds    0.0.0.0:6379->6379/tcp
```

## 2.7 验证数据库就绪

```bash
# 验证 PostgreSQL（进入容器执行 psql 查版本；首次启动等 3~5 秒初始化）
docker exec -it ia-pg psql -U industry -d industryagent -c "SELECT version();"

# 预期输出（节选）：
# PostgreSQL 16.x on x86_64-pc-linux-musl ...

# 验证 Redis
docker exec -it ia-redis redis-cli ping
# 预期输出：PONG
```

> **Windows 用户**：以上命令在 PowerShell 或 Git Bash 中执行均可。若提示 `docker: command not found`，先安装 Docker Desktop 并确保其在运行（见《05-后端环境与软件安装教程》2.3）。

## 2.8 常见坑

1. **`python` 命令打开 Microsoft Store 或版本不对**：Windows 的"应用执行别名"劫持了 python。到"设置 → 应用 → 高级应用设置 → 应用执行别名"关掉两个 python 别名；或统一用 `py -3.12`。
2. **venv 激活路径搞混**：Windows 下是 `.venv/Scripts/`（Git Bash 用 `source .venv/Scripts/activate`），macOS/Linux 是 `.venv/bin/`。在 Windows 上照抄 Linux 博客的 `source .venv/bin/activate` 必然报 `No such file or directory`。
3. **PowerShell 激活报"在此系统上禁止运行脚本"**：执行 `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser` 后重试；或干脆用 Git Bash。
4. **没激活 venv 就 `pip install`**：包全装进了系统 Python。判断方法：`python -c "import sys; print(sys.executable)"` 的输出里没有 `.venv` 就是装错了。补救：`pip uninstall` 对应包，激活 venv 后重装。
5. **端口被占**：宿主机本来就装过 PostgreSQL / Redis 时，5432 / 6379 会冲突。先查占用再处理（Windows：`netstat -ano | findstr 5432`），完整排查表见《05-后端环境与软件安装教程》1.2。
6. **容器名冲突**：`docker: The container name "/ia-pg" is already in use`——之前跑过。要么 `docker start ia-pg` 复用，要么 `docker rm -f ia-pg` 删除重建（未挂数据卷的旧容器数据会丢，05 教程 3.1 有说明）。

---

# 3. 项目初始化与第一个路由

## 3.1 做什么 / 为什么

**做什么/为什么**：Egg.js 靠"约定优于配置"（目录名即职责）组织工程；FastAPI 没有强制目录约定，因此本教程**显式地**定下一套目录分层并全程贯彻——效果等价：团队代码风格天然统一，这也是它适合企业级项目的原因。

本章做四件事：写 `requirements.txt`（依赖清单）、看一遍最终目录结构（先浏览后面逐章创建）、创建本章所需的两个文件（`app/core/config.py` 最小版 + `app/main.py`）、启动 uvicorn 并用 curl 验证"HTTP 请求 → 路由 → JSON 响应"最小闭环。

> **为什么不用 poetry / uv / pyproject.toml**：教程读者面优先——pip + requirements.txt 是所有 Python 环境的公约数，零额外概念。企业里若已用 poetry / uv，把同一份依赖清单导入即可，代码零改动。

## 3.2 requirements.txt（完整内容）

**做什么/为什么**：声明依赖，这是整个项目的"物料清单"。本章实际用到的只有前四个包（fastapi、uvicorn、pydantic 系）；其余是后面章节的"提前备货"——一次装齐，避免每章停下来装一次依赖。版本策略用 `>=` 下限而非精确锁死（避免镜像缺某个精确版本装不上；企业生产建议 `pip freeze` 锁死全集）。

在项目根新建 `requirements.txt`（完整内容）：

```text
# ---------- Web 框架与 ASGI 服务器 ----------
fastapi>=0.115            # Web 框架：路由 / 校验 / 自动文档
uvicorn[standard]>=0.30   # ASGI 服务器；[standard] 附带 watchfiles 等（--reload 更快更稳）
pydantic>=2.7             # fastapi 会随带安装，显式声明便于锁定 v2 大版本
pydantic-settings>=2.3    # .env / 环境变量 -> 强类型配置（第 4 章主力）
python-dotenv>=1.0        # 可省略：pydantic-settings 解析 .env 依赖它并会自动装上，
                          # 显式声明确保存在且版本可控

# ---------- 数据库（第 5 章启用） ----------
sqlalchemy[asyncio]>=2.0.30   # ORM；[asyncio] 附带 greenlet（异步必需）
asyncpg>=0.29                 # PostgreSQL 异步驱动

# ---------- 缓存 / 外部调用（第 8、10 章启用） ----------
redis>=5.0                # redis-py（含 redis.asyncio）
httpx>=0.27               # 异步 HTTP 客户端：调智能体服务（8300）

# ---------- 认证（第 7 章启用） ----------
PyJWT>=2.8                # JWT 签发/验签（HS256；第 19 章升级 RS256）
bcrypt>=4.1               # 密码哈希；提供预编译 wheel，Windows 免编译器，
                          # 且与种子数据的 $2b$ 哈希直接兼容（V1.x 用 bcryptjs 的顾虑不复存在）

# ---------- 可观测（第 14 章启用） ----------
prometheus-client>=0.20   # /metrics 指标暴露
```

> 后续章节新增依赖时会明确说明：APScheduler（第 11 章）、minio（第 18 章）、mcp（第 13 章）；pytest 系进 `requirements-dev.txt`（第 12 章）。

安装（确认 `.venv` 已激活，且在 `server/` 目录）：

```bash
pip install -r requirements.txt
# 预期输出（末尾节选）：Successfully installed fastapi-x.x.x pydantic-settings-x.x.x uvicorn-x.x.x ...
#（版本号以安装当时为准；首行应出现 Looking in indexes: https://pypi.tuna.tsinghua.edu.cn/simple）

# 快速验证关键包可导入
python -c "import fastapi, uvicorn, pydantic_settings; print(fastapi.__version__)"
# 预期输出类似：0.115.x
```

## 3.3 目录结构总览与分层约定

**做什么/为什么**：先看全景再动工。下面是项目**最终**目录结构（本章只创建其中两个文件，其余到对应章节再建，避免一堆空目录）：

```text
industryagent-server/            # 项目根 = monorepo 的 server/ 目录
├─ app/
│  ├─ main.py                    # 本章：create_app()、lifespan、GET /
│  ├─ run.py                     # 第 18 章：双 uvicorn Server 启动器（7001 REST + 7002 WS）
│  ├─ core/
│  │  ├─ config.py               # 本章：最小版 Settings（第 4 章替换为全量版）
│  │  ├─ database.py             # 第 5 章：async engine + AsyncSessionLocal + get_db
│  │  ├─ error_codes.py          # 第 5 章：ErrorCode 枚举 + DEFAULT_MESSAGES
│  │  ├─ biz_error.py            # 第 5 章：BizError 业务异常
│  │  ├─ security.py             # 第 7 章：JWT 签发/验签、bcrypt
│  │  ├─ deps.py                 # 第 7 章：get_current_user / require_perm("...")
│  │  ├─ role_permissions.py     # 第 7 章：角色-权限映射
│  │  ├─ circuit_breaker.py      # 第 8 章：自研熔断器
│  │  └─ metrics.py              # 第 14 章：prometheus 指标注册
│  ├─ models/                    # 第 5 章起：SQLAlchemy 表模型（wafer.py / user.py / ...）
│  ├─ schemas/                   # 第 5 章起：Pydantic 请求/响应模型
│  ├─ services/                  # 第 5 章起：业务逻辑层（XxxService 类，方法 async）
│  ├─ routers/                   # 第 5 章起：APIRouter 按模块拆分（home/wafer/auth/...）
│  ├─ middleware/                # 第 6/7/10 章：审计日志 / 请求日志(traceId) / 幂等
│  ├─ ws/                        # 第 18 章：ConnectionManager + WS 子应用（7002）
│  ├─ tasks/                     # 第 11 章：APScheduler 定时任务
│  └─ mcp/                       # 第 13 章：MCP Server（stdio）
├─ scripts/                      # 第 5 章起：init_db.py（建表+种子）等
├─ tests/                        # 第 12 章：unit/ 与 integration/
├─ agent-mock/                   # 第 8 章：智能体 Mock（30 行 FastAPI，8300，与 app/ 并列同仓）
├─ requirements.txt              # 本章创建
├─ requirements-dev.txt          # 第 12 章创建
├─ .env / .env.example / .gitignore  # 第 4 章创建
├─ Dockerfile / docker-compose.yml   # 第 15 章创建
└─ keycloak/                     # 第 19 章：ia-realm-export.json
```

**分层约定（务必理解，与 V1.x 一一对应）**：

- **Router（app/routers/）**：对标原 Controller。只做"HTTP 世界与业务世界的翻译"——声明路径、参数与响应模型，调用 service，返回数据。**不写业务逻辑**。
- **Service（app/services/）**：业务逻辑层。一个方法对应一个业务动作（如"创建批次并查重"）。**不直接碰 HTTP**。全库统一用类（`XxxService`，方法 async）。
- **Model（app/models/）**：SQLAlchemy 表定义 + 表级操作。
- **Schema（app/schemas/）**：Pydantic 请求/响应模型——Egg 版没有的独立层，参数校验与 OpenAPI 文档都由它驱动。
- **Middleware（app/middleware/）**：横切关注点（鉴权、日志、审计、幂等），像洋葱一样包裹请求。

给跟过 V1.x 的读者一张对照表：

| V1.x（Egg.js） | V2.0（FastAPI） | 说明 |
| --- | --- | --- |
| app/controller/ | app/routers/ | 只做 HTTP 翻译，不写业务逻辑 |
| app/service/ | app/services/ | 职责不变 |
| app/model/（sequelize） | app/models/（SQLAlchemy 2.0 async） | 表定义 |
| （无独立层） | app/schemas/（Pydantic） | 校验 + 文档的单一事实源 |
| app/extend/context.ts（ctx.success） | 统一响应包约定 | `{code,message,data,traceId}`，本章 3.5 立规矩 |
| config/config.*.ts 按环境分层 | app/core/config.py + .env（IA_*） | 环境分层改为 IA_ENV 开关（第 4 章） |
| app/schedule/ | app/tasks/（APScheduler） | 定时任务 |
| egg-jwt / egg-redis / egg-sequelize | PyJWT / redis-py / SQLAlchemy | 插件换库，语义对应 |
| typings/index.d.ts | 无需 | 类型信息由注解与库自带 |

## 3.4 app/core/config.py（最小版，完整内容）

**做什么/为什么**：`GET /` 要返回应用名与版本，先建一个最小配置类占位。它同时提前引入 pydantic-settings——第 4 章会用全量 Settings 完整替换本文件。

创建目录与文件：

```bash
cd ~/code/industryagent/server
mkdir -p app/core
```

`app/core/config.py`（完整内容）：

```python
"""全局配置（最小版）。

第 3 章只需要应用名与版本供 GET / 使用；
第 4 章会用全量 Settings（数据库 / Redis / JWT / 智能体地址等）完整替换本文件。
"""
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    """应用级配置项：字段即契约，默认值即兜底。"""

    # 应用标识：展示在 GET / 与 /docs 标题上
    app_name: str = "industryagent-server"
    app_version: str = "1.0.0"
```

## 3.5 app/main.py（完整内容）：create_app + lifespan + GET /

**做什么/为什么**：一个文件讲清三件事——应用工厂、生命周期钩子、第一个路由。它对标原 Egg 版的 `app/router.ts` + `app/controller/home.ts` + `app/extend/context.ts` 之和，并从第一行代码起冻结**统一响应包**契约：

- 成功：`{ code: 0, message: 'ok', data: ..., traceId: '...' }`
- 分页：`{ code: 0, message: 'ok', data: { rows, total, page, pageSize }, traceId }`
- 失败：`{ code: 业务错误码, message: '...', traceId }`（由第 5 章的全局异常处理器产出）

`app/main.py`（完整内容）：

```python
"""应用入口：create_app() 组装 FastAPI 实例。

所有装配（中间件 / 异常处理 / 路由挂载 / lifespan）都集中在工厂函数里，
测试可以独立创建应用实例而不互相污染——这是"工厂模式"在测试上的直接回报。
"""
from contextlib import asynccontextmanager
from datetime import datetime, timezone
from uuid import uuid4

from fastapi import FastAPI, Request

from app.core.config import Settings

# 第 4 章会改为 get_settings() 单例，这里先直接实例化最小版
settings = Settings()


@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期：yield 前是启动阶段，后是关闭阶段。

    为什么用 lifespan 而不是 @app.on_event("startup")：
    1. on_event 已被 FastAPI 官方标记为废弃（deprecated）；
    2. yield 前后天然成对，连接池的"创建-关闭"写在一处不易漏；
    3. 第 5 / 10 章将在这里初始化数据库引擎与 Redis 连接池，关闭时统一释放。
    """
    # ---- startup：数据库 / Redis 连接池将在这里初始化 ----
    yield
    # ---- shutdown：连接池将在这里关闭 ----


def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.app_name,
        version=settings.app_version,
        description="芯联半导体良率根因分析平台 - 业务后端",
        lifespan=lifespan,
    )

    @app.get("/")
    async def platform_info(request: Request) -> dict:
        """平台信息（欢迎页）——对外契约与 V1.x Egg 版逐字节一致。"""
        # traceId：统一响应包必带，用于跨日志 / 跨服务串联一次请求。
        # 本章先用随机 uuid 兜底；第 6 章请求日志中间件上线后，会优先透传
        # 上游 X-Trace-Id 并写入 request.state.trace_id，这里自然取到注入值。
        trace_id = getattr(request.state, "trace_id", None) or str(uuid4())
        return {
            "code": 0,
            "message": "ok",
            "data": {
                "name": settings.app_name,
                "version": settings.app_version,
                "project": "芯联半导体良率根因分析平台 - 业务后端",
                "doc": "IA-TU-001 后端开发教程",
                # 对齐原 JS new Date().toISOString() 的"毫秒 + Z"格式
                "time": datetime.now(timezone.utc)
                .isoformat(timespec="milliseconds")
                .replace("+00:00", "Z"),
            },
            "traceId": trace_id,
        }

    return app


# uvicorn app.main:app 引用的模块级实例
app = create_app()
```

三个要点：

1. **`request: Request`**：路由函数声明 Request 参数即可拿到请求上下文（读头、写 state），不需要它时可以完全不声明——FastAPI 按签名按需注入，这也是后面 `Depends` 依赖注入的同族机制。
2. **返回 dict 而不是手动 JSON 化**：FastAPI 负责序列化；第 5 章起会配合 response_model 做更严格的输出契约。
3. **traceId 的演进路径**：本章兜底生成，第 6 章中间件接管——对外包结构从头到尾不变。

## 3.6 启动并验证：uvicorn 与第一个里程碑

```bash
# 在项目根（server/）、venv 已激活的状态下执行
uvicorn app.main:app --reload --port 7001
```

预期输出（节选）：

```text
INFO:     Will watch for changes in these directories: ['.../server']
INFO:     Uvicorn running on http://127.0.0.1:7001 (Press CTRL+C to quit)
INFO:     Started reloader process [12345] (WatchFiles)
INFO:     Application startup complete.
```

- `app.main:app`：`模块路径:实例名`，即 `app/main.py` 里的 `app`；
- `--reload`：开发模式，保存 `.py` 文件自动重启（由 uvicorn[standard] 附带的 watchfiles 驱动）；
- `--port 7001`：本教程 REST 服务固定端口（1.3 端口表）；**看到 `Application startup complete` 才算就绪**。停止用 `CTRL+C`。

新开一个终端验证（**所有 curl 建议在 Git Bash 中执行**，Windows cmd 的引号转义是重灾区）：

```bash
curl -s http://127.0.0.1:7001/
```

预期输出：

```json
{"code":0,"message":"ok","data":{"name":"industryagent-server","version":"1.0.0","project":"芯联半导体良率根因分析平台 - 业务后端","doc":"IA-TU-001 后端开发教程","time":"2026-10-17T08:40:30.123Z"},"traceId":"5f9c2a1e-8b7d-4c3a-9e2f-1a2b3c4d5e6f"}
```

看到 `code:0` 和 `traceId`，说明应用工厂、配置读取、路由、统一响应包全部生效。**第一个里程碑达成。**

## 3.7 顺手的惊喜：/docs 自动交互文档

保持服务运行，浏览器打开：

```text
http://127.0.0.1:7001/docs
```

预期看到：

- 页面标题为 `industryagent-server - Swagger UI`，顶部有应用描述与版本 `1.0.0`；
- `GET /` 已列在接口列表中，点开 → `Try it out` → `Execute`，Response body 与上面 curl 的输出完全一致（200）。

同款数据还有一份阅读友好的文档版：`http://127.0.0.1:7001/redoc`。

这就是 1.2 节"契约即文档"的直观体现：Egg 版要自建或引入第三方方案维护接口文档，FastAPI 从路由与 Pydantic 模型**自动生成** OpenAPI 规范与两套 UI，文档与代码永远同步。部署上如需按环境关闭文档（`docs_url=None`），第 15 章再谈。

## 3.8 常见坑

1. **`ERROR: No module named uvicorn`（或 ModuleNotFoundError: fastapi）**：venv 没激活，或包装到了另一个 Python。`python -c "import sys; print(sys.executable)"` 确认解释器路径含 `.venv` 后重装。
2. **`ERROR: Error loading ASGI app. Could not import module "app.main".`**：不在项目根执行——uvicorn 按当前目录解析 `app.main`。`cd ~/code/industryagent/server` 后再启动。
3. **端口占用**：`[Errno 10048] error while attempting to bind on address ('127.0.0.1', 7001)`。Windows 查占用：`netstat -ano | findstr 7001`，任务管理器结束对应 PID；或临时换 `--port 7005` 先验证服务本身没问题。
4. **改了代码但没重启**：`--reload` 只监听 Python 文件变更；改 `.env`（第 4 章）不会触发重启，需手动 `CTRL+C` 后重新启动。个别机器上中文/空格路径会让 reload 不稳定，遇到就先去掉 `--reload` 手动重启验证。
5. **`/docs` 404**：多半是地址打错（`/docs`、`/redoc` 都是复数路径），或自行传了 `docs_url=None`。本章代码默认开启，不应出现。
6. **服务起不来但报 `TypeError: 'NoneType' object is not callable` 之类**：检查 `app = create_app()` 是否忘了调用工厂（写成 `app = create_app`）。uvicorn 需要的是实例不是函数。

---

# 4. 配置管理

## 4.1 做什么 / 为什么

**做什么/为什么**：企业级项目的第一原则：**代码与环境隔离**。数据库地址、密码、JWT 密钥绝不能写死在代码里。V1.x 靠 Egg 的 config 按环境分层（config.default.ts → config.local/prod/test.ts）；V2.0 换成更直白的模型：

- **一份** `Settings` 类（`app/core/config.py`）：字段 + 类型 + 默认值，就是配置的全部真相；
- **一个** `.env` 文件 + `IA_` 前缀环境变量：本地/生产/测试的差异全部显式注入，而不是散在多个配置文件里；
- **一个** `IA_ENV` 开关（`local|prod|test`）：代码里按它做行为分支（日志级别、防裸奔守卫、部署差异）。

这样"某个值从哪来"永远只有一个答案：环境变量 → `.env` → 字段默认值（优先级从高到低，由 pydantic-settings 保证）。

## 4.2 .env / .env.example / .gitignore（完整内容）

**做什么/为什么**：`.env` 放本地真实值（**绝不提交 git**）；`.env.example` 是给队友的"配置说明书"（进 git）；`.gitignore` 从机制上保证前者进不了仓库。

项目根新建 `.env`（完整内容）：

```bash
# ===== 运行环境：local=本地开发 / prod=生产 / test=集成测试（第 12 章） =====
IA_ENV=local

# ===== 数据库（对应第 2 章的 ia-pg 容器） =====
IA_DB_HOST=127.0.0.1
IA_DB_PORT=5432
IA_DB_NAME=industryagent
IA_DB_USER=industry
IA_DB_PASSWORD=industry123

# ===== Redis（对应第 2 章的 ia-redis 容器） =====
IA_REDIS_HOST=127.0.0.1
IA_REDIS_PORT=6379

# ===== 安全（生产务必换成随机 32+ 位字符串，见 4.6 的防裸奔守卫） =====
IA_KEYS=dev-keys-please-change-me
IA_JWT_SECRET=dev-jwt-secret-please-change-me

# ===== Python 智能体服务地址 =====
# V2.0 口径：智能体（含第 8 章 agent-mock）统一跑 8300，
# 与《03-智能体开发教程》《04-部署上线教程》对齐（V1.x 教程的 8000 已废弃）
IA_AGENT_BASE_URL=http://127.0.0.1:8300

# ===== 认证模式（第 19 章启用：local=自签 JWT / keycloak=对接 Keycloak） =====
AUTH_MODE=local
```

再新建 `.env.example`（完整内容，与 `.env` 同结构，作为可提交的模板）：

```bash
# 复制本文件为 .env 后按环境修改；.env 不进 git，本文件进 git。
# 值为本教程本地开发用的演示值。

IA_ENV=local

IA_DB_HOST=127.0.0.1
IA_DB_PORT=5432
IA_DB_NAME=industryagent
IA_DB_USER=industry
IA_DB_PASSWORD=industry123

IA_REDIS_HOST=127.0.0.1
IA_REDIS_PORT=6379

IA_KEYS=change-me-32-chars-min
IA_JWT_SECRET=change-me-32-chars-min

IA_AGENT_BASE_URL=http://127.0.0.1:8300

AUTH_MODE=local
```

同时新建 `.gitignore`（完整内容）：

```text
# Python
__pycache__/
*.py[cod]
.venv/
venv/

# 环境变量（绝不提交）
.env

# 日志
logs/
*.log

# 测试与覆盖率（第 12 章）
.pytest_cache/
.coverage
htmlcov/

# 构建产物
dist/
build/
```

> **编码提醒**：`.env` 含中文注释，务必以 UTF-8 保存（VS Code 右下角确认），否则 pydantic-settings 读取时可能整段乱码。

## 4.3 app/core/config.py（完整内容）：Settings 全量版

**做什么/为什么**：用全量 Settings **完整替换** 3.4 的最小版。字段分四组：应用与环境、基础设施（DB/Redis）、安全、外部服务；再加上从 V1.x `config.biz` 平移过来的业务参数（分页、缓存、幂等、根因容错、Token 有效期），第 5~10 章直接引用，不再散落各处。

`app/core/config.py`（完整内容）：

```python
"""全局配置：全库唯一入口是 get_settings()。

为什么用 pydantic-settings：
1. .env / 环境变量 -> 强类型字段（IA_DB_PORT 自动转 int，写错启动即报错，不带病运行）；
2. 配置集中一处，全库统一 from app.core.config import get_settings；
3. 与 FastAPI 依赖注入体系天然配合，测试可整体替换（见 4.4）。
"""
from functools import lru_cache

from pydantic import Field, model_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """IA_ 前缀的环境变量（或 .env 同名变量）自动装载到字段。"""

    model_config = SettingsConfigDict(
        env_file=".env",          # 项目根的 .env（相对启动时的工作目录）
        env_file_encoding="utf-8",
        env_prefix="IA_",         # 字段 db_host -> 环境变量 IA_DB_HOST
        extra="ignore",           # .env 里出现未声明变量时不报错
        case_sensitive=False,
    )

    # ---------- 应用与环境 ----------
    app_name: str = "industryagent-server"
    app_version: str = "1.0.0"
    # 运行环境开关：local（本地开发）/ prod（生产）/ test（集成测试，第 12 章）
    env: str = "local"
    # 认证模式：local（自签 JWT，第 7 章）/ keycloak（第 19 章）。
    # 注意：契约规定它不带 IA_ 前缀，用显式别名接住 AUTH_MODE
    auth_mode: str = Field(default="local", validation_alias="AUTH_MODE")
    # 日志级别（第 14 章结构化日志读取；生产可用环境变量调成 WARNING）
    log_level: str = "INFO"

    # ---------- 数据库（PostgreSQL 16，对应容器 ia-pg） ----------
    db_host: str = "127.0.0.1"
    db_port: int = 5432
    db_name: str = "industryagent"
    db_user: str = "industry"
    db_password: str = "industry123"

    # ---------- Redis（对应容器 ia-redis） ----------
    redis_host: str = "127.0.0.1"
    redis_port: int = 6379

    # ---------- 安全 ----------
    # 通用签名密钥（生产必须换成随机 32+ 位字符串）
    keys: str = "dev-keys-please-change-me"
    # JWT 签名密钥（HS256；第 19 章 keycloak 模式下作为回退开关保留）
    jwt_secret: str = "dev-jwt-secret-please-change-me"

    # ---------- 外部服务 ----------
    # Python 智能体服务基地址（第 8 章根因分析调用；V2.0 起统一 8300）
    agent_base_url: str = "http://127.0.0.1:8300"

    # ---------- 业务参数（原 config.biz 平移，环境无关的默认值） ----------
    # 分页（第 5 章起所有列表接口共用）
    pagination_default_page: int = 1
    pagination_default_page_size: int = 10
    pagination_max_page_size: int = 100
    # 看板缓存 TTL 秒（第 10 章 cache-aside）
    dashboard_cache_ttl_seconds: int = 60
    # 幂等键保留时长秒（第 10 章）
    idempotency_ttl_seconds: int = 86400
    # 根因分析调用智能体的容错参数（第 8 章：httpx 超时 + 自研熔断器）
    rootcause_connect_timeout_ms: int = 3000
    rootcause_response_timeout_ms: int = 20000
    rootcause_retries: int = 2
    breaker_failure_threshold: int = 3
    breaker_open_seconds: int = 30
    # 访问令牌 2h / 刷新令牌 7d（第 7 章；原 '2h' / 7*24*3600 换算为秒）
    access_token_expires_seconds: int = 2 * 3600
    refresh_token_expires_seconds: int = 7 * 24 * 3600

    @property
    def is_prod(self) -> bool:
        return self.env == "prod"

    @model_validator(mode="after")
    def _guard_prod_secrets(self) -> "Settings":
        """生产防裸奔：IA_ENV=prod 时安全密钥必须显式提供，不给兜底。

        对标原 config.prod.ts 的"未设置则启动即失败"——把配置错误拦在
        启动阶段，而不是等默认密钥 leaked 之后才发现。
        """
        if self.is_prod and (
            self.jwt_secret == "dev-jwt-secret-please-change-me"
            or self.keys == "dev-keys-please-change-me"
        ):
            raise ValueError(
                "IA_ENV=prod 但 IA_KEYS / IA_JWT_SECRET 仍是开发默认值，"
                "请通过环境变量配置随机 32+ 位字符串后重启"
            )
        return self


@lru_cache
def get_settings() -> Settings:
    """配置单例：全库统一从这里取 Settings（见 4.4）。"""
    return Settings()
```

命名映射规则（排查"配置没读到"时按这张对）：

| 字段 | 环境变量 / .env 键 |
| --- | --- |
| db_host / db_port / db_name / db_user / db_password | IA_DB_HOST / IA_DB_PORT / IA_DB_NAME / IA_DB_USER / IA_DB_PASSWORD |
| redis_host / redis_port | IA_REDIS_HOST / IA_REDIS_PORT |
| keys / jwt_secret | IA_KEYS / IA_JWT_SECRET |
| agent_base_url | IA_AGENT_BASE_URL |
| env | IA_ENV |
| auth_mode | **AUTH_MODE**（显式别名，唯一不带 IA_ 前缀者） |

## 4.4 get_settings()：进程内单例（lru_cache）

**做什么/为什么**：`Settings()` 每次实例化都要读 `.env` 并跑一遍校验，请求级高频调用没必要重复。加 `@lru_cache` 后进程内只构造一次，这是 FastAPI 社区的标准姿势，好处有三：

1. **省**：一次读取，处处复用；
2. **稳**：全库拿到同一个实例，配置不会"半路变卦"；
3. **可换**：作为依赖使用（`Depends(get_settings)`）时，测试可用 `app.dependency_overrides[get_settings] = lambda: fake_settings` 整体替换，或 `get_settings.cache_clear()` 清缓存后重取——第 12 章会用。

使用方式（第 5 章起的标准写法，先混个眼熟）：

```python
from fastapi import Depends
from app.core.config import Settings, get_settings

@app.get("/demo")
async def demo(settings: Settings = Depends(get_settings)):
    return {"db": settings.db_name}
```

## 4.5 app/main.py 改用 get_settings（完整内容）

**做什么/为什么**：3.5 在模块级直接 `Settings()`，现在配置全量了，切换到单例入口——变化只有两行（import 与实例获取），为保持"跟着敲就能跑"，给出完整内容。

`app/main.py`（完整内容）：

```python
"""应用入口：create_app() 组装 FastAPI 实例。

所有装配（中间件 / 异常处理 / 路由挂载 / lifespan）都集中在工厂函数里，
测试可以独立创建应用实例而不互相污染——这是"工厂模式"在测试上的直接回报。
"""
from contextlib import asynccontextmanager
from datetime import datetime, timezone
from uuid import uuid4

from fastapi import FastAPI, Request

from app.core.config import get_settings

# 单例入口：进程内只构造一次 Settings（读一次 .env、跑一次校验）
settings = get_settings()


@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期：yield 前是启动阶段，后是关闭阶段。

    为什么用 lifespan 而不是 @app.on_event("startup")：
    1. on_event 已被 FastAPI 官方标记为废弃（deprecated）；
    2. yield 前后天然成对，连接池的"创建-关闭"写在一处不易漏；
    3. 第 5 / 10 章将在这里初始化数据库引擎与 Redis 连接池，关闭时统一释放。
    """
    # ---- startup：数据库 / Redis 连接池将在这里初始化 ----
    yield
    # ---- shutdown：连接池将在这里关闭 ----


def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.app_name,
        version=settings.app_version,
        description="芯联半导体良率根因分析平台 - 业务后端",
        lifespan=lifespan,
    )

    @app.get("/")
    async def platform_info(request: Request) -> dict:
        """平台信息（欢迎页）——对外契约与 V1.x Egg 版逐字节一致。"""
        # traceId：统一响应包必带，用于跨日志 / 跨服务串联一次请求。
        # 本章先用随机 uuid 兜底；第 6 章请求日志中间件上线后，会优先透传
        # 上游 X-Trace-Id 并写入 request.state.trace_id，这里自然取到注入值。
        trace_id = getattr(request.state, "trace_id", None) or str(uuid4())
        return {
            "code": 0,
            "message": "ok",
            "data": {
                "name": settings.app_name,
                "version": settings.app_version,
                "project": "芯联半导体良率根因分析平台 - 业务后端",
                "doc": "IA-TU-001 后端开发教程",
                # 对齐原 JS new Date().toISOString() 的"毫秒 + Z"格式
                "time": datetime.now(timezone.utc)
                .isoformat(timespec="milliseconds")
                .replace("+00:00", "Z"),
            },
            "traceId": trace_id,
        }

    return app


# uvicorn app.main:app 引用的模块级实例
app = create_app()
```

## 4.6 分环境覆盖：IA_ENV=local|prod|test

**做什么/为什么**：V1.x 靠 Egg 自动加载 `config.<env>.ts` 分层覆盖；V2.0 的模型是"**配置只有一份，环境差异全部显式**"，三条规则讲清楚：

1. **优先级**：真实环境变量 > `.env` > 字段默认值（pydantic-settings 的固定行为）。生产容器里用 compose 的 `environment:` 注入真实值，天然盖过 `.env`。
2. **`IA_ENV` 是开关不是魔法**：代码里按 `settings.env` / `settings.is_prod` 做行为分支——本章的防裸奔守卫（4.3）、第 14 章的日志级别、第 15 章部署时的文档开关，都是这个模式。
3. **test 环境**：第 12 章集成测试以 `IA_ENV=test IA_DB_NAME=industryagent_test` 启动（测试库已在《05-后端环境与软件安装教程》第 5 章提前建好），避免污染开发数据。

三个环境的启动示意：

```bash
# 本地开发（默认）：读 .env，IA_ENV=local
uvicorn app.main:app --reload --port 7001

# 集成测试（第 12 章）：环境变量临时覆盖库名
IA_ENV=test IA_DB_NAME=industryagent_test pytest tests/integration -v

# 生产（第 15 章容器内）：全部真实值走环境变量，.env 不进镜像
IA_ENV=prod IA_KEYS=<随机串> IA_JWT_SECRET=<随机串> uvicorn app.main:app --port 7001
```

## 4.7 验证

```bash
# 1. 基本读取：.env 中的值应被正确装载
python -c "from app.core.config import get_settings; s = get_settings(); print(s.app_name, s.env, s.db_host, s.db_port, s.agent_base_url, s.auth_mode)"
# 预期输出：industryagent-server local 127.0.0.1 5432 http://127.0.0.1:8300 local

# 2. 单例生效：两次调用拿到同一对象
python -c "from app.core.config import get_settings; print(get_settings() is get_settings())"
# 预期输出：True
```

验证优先级（真实环境变量 > `.env`）：

```bash
# 3. 临时用环境变量盖过 .env 的库名（Git Bash 写法）
IA_DB_NAME=wrongdb python -c "from app.core.config import Settings; print(Settings().db_name)"
# 预期输出：wrongdb

# 4. 不加环境变量时，值来自 .env
python -c "from app.core.config import Settings; print(Settings().db_name)"
# 预期输出：industryagent
```

验证防裸奔守卫（对标 V1.x "prod 未配 keys 启动即失败"）：

```bash
# 5. prod 环境但仍用开发默认密钥 -> 启动即被拦截
IA_ENV=prod python -c "from app.core.config import Settings; Settings()"
# 预期输出（节选）：
# pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings
#   Value error, IA_ENV=prod 但 IA_KEYS / IA_JWT_SECRET 仍是开发默认值，
#   请通过环境变量配置随机 32+ 位字符串后重启 ...

# 6. prod 环境 + 显式密钥 -> 通过
IA_ENV=prod IA_KEYS=prod-random-keys-000000000000 IA_JWT_SECRET=prod-random-jwt-0000000000000 \
  python -c "from app.core.config import Settings; print(Settings().env, 'ok')"
# 预期输出：prod ok
```

最后回归主服务（重启让新配置生效）：

```bash
uvicorn app.main:app --reload --port 7001
# 预期输出（节选）：INFO: Application startup complete.

curl -s http://127.0.0.1:7001/
# 预期输出：{"code":0,"message":"ok","data":{"name":"industryagent-server",...},"traceId":"..."}
#（与 3.6 完全一致——配置体系升级不改变对外契约）
```

## 4.8 常见坑

1. **`.env` 不生效**：按顺序查三件事——文件在项目根（与 `app/` 同级，不是 `app/` 里面）；文件名就是 `.env`（有人存成了 `.env.txt`）；编码是 UTF-8（含中文注释时尤其注意）。另外 `env_file=".env"` 相对的是**启动时的工作目录**，所以必须在 `server/` 根启动 uvicorn（与 3.8 坑 2 同源）。
2. **某个 IA_* 变量读不到**：对照 4.3 的映射表查拼写——前缀必须是 `IA_`（`DB_HOST` 不等于 `IA_DB_HOST`）；唯一例外是 `AUTH_MODE`。改完 `.env` 记得重启进程（reload 只盯 `.py` 文件）。
3. **环境变量盖住了 `.env` 却浑然不觉**：当前 shell 里 `export` 过的变量优先级最高，排查时用 `echo $IA_DB_NAME`（Git Bash）看一眼；PowerShell 用 `echo $env:IA_DB_NAME`。
4. **`ValidationError: N validation error(s) for Settings`**：看报错里的字段名——多半是 `IA_DB_PORT=5432 ` 带了空格/引号，或某个 int 字段填了非数字。pydantic 的报错会精确到字段与原因，比"运行到一半才炸"友好得多。
5. **改了 `.env` 但输出没变**：`get_settings()` 被 `@lru_cache` 缓存，且 `--reload` 不监听 `.env`——重启进程；测试里需要刷新时用 `get_settings.cache_clear()`。
6. **`IA_ENV=prod` 起不来（报 4.7 第 5 条的错）**：这正是防裸奔守卫在干活。生产必须显式提供 `IA_KEYS` / `IA_JWT_SECRET`，本地开发保持 `IA_ENV=local` 即可。
