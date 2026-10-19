# 后端开发教程：FastAPI 从 0 到 1

# 芯联半导体良率根因分析平台（industryagent-server）

---

## 文档控制块

| 项目 | 内容 |
|---|---|
| 文档编号 | IA-TU-001 |
| 文档名称 | 后端开发教程：FastAPI 从 0 到 1 |
| 版本 | V2.0 |
| 编制 | 李吉 |
| 日期 | 2026-10-18 |
| 适用项目 | IndustryAgent（芯联半导体良率根因分析智能体平台） |
| 适用读者 | 有 Python 基础、未使用过 FastAPI 的后端初学者 |
| 交付物 | 可运行的 industryagent-server 后端服务（FastAPI + SQLAlchemy 2.0 async + PG + Redis + JWT + MCP Server） |

### 修订记录

| 版本 | 日期 | 修订人 | 修订说明 |
|---|---|---|---|
| V1.0 | 2026-10-15 | 李吉 | 初版发布（Egg.js）：从环境搭建到 MCP Server 暴露的全链路后端教程 |
| V1.1 | 2026-10-15 | 李吉 | 补全第 18 章（8D 报告 / 通知 / 缺陷 / 看板统计 / 审计查询 / 用户管理）与第 19 章（Keycloak 真实对接）；原第 17 章小结并入 19.6 全书完结小结 |
| V2.0 | 2026-10-18 | 李吉 | **技术栈由 Egg.js 迁移为 Python FastAPI**（决策见 IA-ADR-001 V2.0）；对外契约（43 端点/错误码/响应包/种子数据/演示剧本）逐字节兼容；agent 端口统一为 8300、WebSocket 帧格式定稿为 {event, data, ts} |

> **V2.0 阅读须知**：本教程面向"跟着一行行敲"的读者——每个新文件**全量收录**（标注"完整内容"），文件随章节演进时**每次修改都给出当时的完整代码或锚定改动说明**；每章末尾的验证步骤（curl + 预期输出）不通过不要进入下一章。与智能体服务（《03-智能体开发教程》）同为 FastAPI，环境、依赖、调试方式完全复用。

---

## 目录

1. 教程导读
2. 环境准备
3. 项目初始化与第一个路由
4. 配置管理
5. 第一个完整模块——晶圆批次 wafer
6. 中间件体系：审计日志与请求链路
7. 用户与鉴权：JWT + RBAC
8. 异常事件与根因分析模块
9. 建议与审批模块
10. Redis 应用：缓存与幂等
11. 定时任务（APScheduler）
12. 单元测试与集成测试
13. FastAPI MCP Server：把后端能力暴露给智能体
14. 日志与可观测基础
15. 构建与部署
16. 常见坑与 FAQ
18. 补全模块：8D 报告 / 通知 / 缺陷 / 看板统计 / 审计查询 / 用户管理
19. 进阶：Keycloak 真实对接（含全书完结小结）

> 编号说明：第 17 章「本章小结与下一步」自 V1.1 起并入 19.6「本章小结：全书完结」，编号 17 空缺以保持 18/19 章编号稳定。

---
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




