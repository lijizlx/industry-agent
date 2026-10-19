# 后端开发教程：Egg.js 从 0 到 1

# 芯联半导体良率根因分析平台（industryagent-server）

---

## 文档控制块

| 项目 | 内容 |
|---|---|
| 文档编号 | IA-TU-001 |
| 文档名称 | 后端开发教程：Egg.js 从 0 到 1 |
| 版本 | V1.1 |
| 编制 | 李吉 |
| 日期 | 2026-10-15 |
| 适用项目 | IndustryAgent（芯联半导体良率根因分析智能体平台） |
| 适用读者 | 有 JavaScript 基础、未使用过 Egg.js 的后端初学者 |
| 交付物 | 可运行的 industryagent-server 后端服务（Egg.js + TS + PG + Redis + JWT + MCP Server） |

### 修订记录

| 版本 | 日期 | 修订人 | 修订说明 |
|---|---|---|---|
| V1.0 | 2026-10-15 | 李吉 | 初版发布：从环境搭建到 MCP Server 暴露的全链路后端教程 |
| V1.1 | 2026-10-15 | 李吉 | 补全第 18 章（8D 报告 / 通知 / 缺陷 / 看板统计 / 审计查询 / 用户管理）与第 19 章（Keycloak 真实对接）；原第 17 章小结并入 19.6 全书完结小结 |

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
11. 定时任务 schedule
12. 单元测试与集成测试
13. Egg.js MCP Server：把后端能力暴露给智能体
14. 日志与可观测基础
15. 构建与部署
16. 常见坑与 FAQ
18. 补全模块：8D 报告 / 通知 / 缺陷 / 看板统计 / 审计查询 / 用户管理
19. 进阶：Keycloak 真实对接（含全书完结小结）

> 编号说明：原第 17 章「本章小结与下一步」在 V1.1 修订中并入 19.6「本章小结：全书完结」，编号 17 空缺以保持 18/19 章编号稳定。

---

# 1. 教程导读

## 1.1 学完你将获得什么

本教程带你从零搭建 **industryagent-server**——芯联半导体良率根因分析平台的业务后端。教程结束时，你将拥有一个具备以下能力的、可运行、可部署、可写进简历的企业级后端：

- 用户登录鉴权（JWT 双 Token + bcrypt 密码加密）
- 晶圆批次 / 缺陷数据完整 CRUD（参数校验 + 事务）
- 异常事件查询
- 根因分析任务触发（HTTP 调用 Python 智能体服务，含超时 / 重试 / 熔断降级）
- 建议审批接口（事务 + 行锁 + 完整"分析 → 建议 → 审批"业务闭环）
- 审计日志中间件（谁 / 何时 / 做了什么 / 耗时）
- RBAC 权限模型（admin / 工艺工程师 / 审批员三种角色）
- Redis 应用（看板缓存 cache-aside + 接口幂等）
- 定时任务（传感器数据断流告警 + 良率日报）
- 单元测试与集成测试（mocha + chai + sinon + supertest）
- MCP Server（把查询异常事件 / 查询晶圆批次 / 触发根因分析注册为 MCP 工具，供智能体调用）
- 可观测基础（结构化日志 + traceId + Prometheus 指标）
- Docker 多阶段构建与 docker-compose 本地编排

## 1.2 技术栈总览

```text
                         ┌────────────────────────────────────────────┐
                         │        前端（React，见前端教程）            │
                         └───────────────────┬────────────────────────┘
                                             │ REST API (JSON)
                                             ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                    industryagent-server（本教程交付物）                     │
│                                                                            │
│  Egg.js 4.0 + TypeScript                                                   │
│  ├─ Router / Controller / Service / Model（egg-sequelize + PostgreSQL 16）│
│  ├─ 中间件：错误处理 / 审计日志 / 请求日志(traceId) / 鉴权 / 幂等 / 指标 │
│  ├─ JWT 鉴权 + RBAC（roles / user_roles）                                  │
│  ├─ Redis（ioredis via egg-redis）：缓存 / 幂等 / refresh token           │
│  ├─ 定时任务 schedule：数据健康检查 / 良率日报                             │
│  └─ MCP Server（@modelcontextprotocol/sdk）：暴露工具给智能体             │
└──────────────┬──────────────────────────────────────┬─────────────────────┘
               │ HTTP (httpclient)                    │ MCP (stdio)
               ▼                                      ▼
┌──────────────────────────────┐       ┌──────────────────────────────────┐
│  Python 智能体服务 (FastAPI)  │       │  MCP 客户端（Cursor / Claude /    │
│  /agent/analyze 根因分析      │       │  Claude Code / 自研智能体）        │
└──────────────────────────────┘       └──────────────────────────────────┘
```

## 1.3 前置准备清单

| 项          | 要求                                                      |
| ---------- | ------------------------------------------------------- |
| Node.js    | ≥ 20.x（`node -v` 验证）                                    |
| TypeScript | 会基本类型标注即可，教程循序渐进                                        |
| JavaScript | 熟悉 async/await、Promise、模块化                              |
| SQL        | 会基本 SELECT / INSERT 即可                                  |
| Docker     | 用于一键拉起 PostgreSQL 16 与 Redis 7                          |
| 包管理器       | 本教程主用 npm（兼容性最好）；pnpm 用户见 2.2 节说明                       |
| 操作系统       | macOS / Linux 命令为主，Windows 差异处单独注明（推荐用 Git Bash 执行本文命令） |

---

# 2. 环境准备

## 2.1 做什么 / 为什么

工欲善其事必先利其器。后端依赖 PostgreSQL（业务数据）与 Redis（缓存 / 幂等 / Token 白名单）。**不推荐**在宿主机直接安装这两个数据库——版本不一致、卸载残留、多项目冲突都是坑。用 Docker 一分钟拉起、一键销毁，是最干净的方式。

## 2.2 安装 Node 20+ 与包管理器

```bash
# 验证 Node 版本（需 >= 20）
node -v
# 预期输出类似：v20.15.0

# 验证 npm
npm -v
# 预期输出类似：10.7.0
```

若 Node 版本过低，推荐使用 [nvm](https://github.com/nvm-sh/nvm)（Windows 用 [nvm-windows](https://github.com/coreybutler/nvm-windows)）：

```bash
nvm install 20
nvm use 20
```

> **pnpm 用户注意**：Egg.js 在 pnpm 的严格 `node_modules` 结构下通常可以正常运行，但个别插件（含动态 require 的）可能解析失败。若遇到 `Cannot find module 'egg-xxx'` 类报错，在项目根新建 `.npmrc` 写入 `node-linker=hoisted` 后重新 `pnpm install`。本教程命令统一用 `npm`，pnpm 用户把 `npm install` 换成 `pnpm install` 即可，其余无差别。

## 2.3 用 Docker 拉起 PostgreSQL 16 与 Redis 7

```bash
# PostgreSQL 16：容器名 ia-pg，端口 5432，用户 industry / 密码 industry123，库 industryagent
docker run -d \
  --name ia-pg \
  -e POSTGRES_USER=industry \
  -e POSTGRES_PASSWORD=industry123 \
  -e POSTGRES_DB=industryagent \
  -p 5432:5432 \
  postgres:16-alpine

# 预期输出：一长串容器 ID（64 位十六进制字符串）

# Redis 7：容器名 ia-redis，端口 6379
docker run -d --name ia-redis -p 6379:6379 redis:7-alpine
# 预期输出：一长串容器 ID
```

> **Windows 用户**：在 PowerShell 或 Git Bash 中执行均可。若提示 `docker: command not found`，先安装 Docker Desktop 并确保其在运行。

## 2.4 验证数据库就绪

```bash
# 验证 PostgreSQL（进入容器执行 psql 查版本）
docker exec -it ia-pg psql -U industry -d industryagent -c "SELECT version();"

# 预期输出（节选）：
# PostgreSQL 16.x on x86_64-pc-linux-musl ...

# 验证 Redis
docker exec -it ia-redis redis-cli ping
# 预期输出：PONG
```

## 2.5 目录规划

本系列教程的项目统一放在一个仓库目录下（monorepo 风格，但各自独立可运行）：

```bash
mkdir -p ~/code/industryagent && cd ~/code/industryagent
mkdir -p server agent docs web
```

```text
~/code/industryagent/
├── server/    # 本教程：Egg.js 业务后端（industryagent-server）
├── agent/     # 智能体教程：Python FastAPI 根因分析服务
├── docs/      # 项目文档（本教程属于其中一部分）
└── web/       # 前端教程：React 管理台
```

> **Windows 用户**：路径形如 `D:\code\industryagent`，在 Git Bash 中为 `/d/code/industryagent`。下文所有 `~/code/industryagent` 请自行替换。

**验证**：`ls ~/code/industryagent` 应看到 `agent docs server web` 四个目录。

---

# 3. 项目初始化与第一个路由

## 3.1 做什么 / 为什么

Egg.js 是阿里开源的企业级 Node.js 框架，核心理念是"**约定优于配置**"：目录名即职责（`app/router.ts` 管路由、`app/controller/` 管控制器、`app/service/` 管业务逻辑、`app/middleware/` 管中间件）。这套约定让团队代码风格天然统一——这正是它适合企业级项目的原因。

我们先用官方脚手架生成 TS 项目骨架，再逐文件讲解并替换为最终内容（本教程给出每个文件的完整内容，保证跟着敲就能跑）。

## 3.2 脚手架初始化

```bash
cd ~/code/industryagent
mkdir industryagent-server && cd industryagent-server

# 官方 TS 脚手架（在空目录中执行）
npm init egg --type=ts
# 交互提示全部回车使用默认值即可
```

脚手架会生成 `package.json`、`tsconfig.json`、`app/`、`config/`、`typings/` 等。生成的 `app/controller/home.ts`、`config/config.default.ts` 等文件，后文都会用完整内容替换。

> 说明：若 `npm init egg` 拉取慢，可 `npm init egg --type=ts --registry=https://registry.npmmirror.com`。若脚手架生成的 `package.json` 中 egg 版本不是 4.x，直接手动改成本教程 3.3 节的内容即可。

## 3.3 package.json（完整内容）

**做什么/为什么**：声明依赖与脚本。这是整个项目的"物料清单"。Egg 4.0 于 2025 年发布；若你的镜像源尚未同步或安装失败，可把 egg 版本临时改为 `^3.30.0`，本教程代码在 3.x / 4.x 下均可运行。

用以下内容**完整替换** `package.json`：

```json
{
  "name": "industryagent-server",
  "version": "1.0.0",
  "description": "芯联半导体良率根因分析平台 - 业务后端（Egg.js 4.0 + TypeScript）",
  "private": true,
  "egg": {
    "typescript": true,
    "declarations": false
  },
  "scripts": {
    "dev": "egg-bin dev",
    "debug": "egg-bin debug",
    "test": "egg-bin test",
    "cov": "egg-bin cov",
    "seed": "ts-node scripts/init-db.ts",
    "mcp": "ts-node mcp/server.ts",
    "mcp:test": "ts-node scripts/mcp-client.ts",
    "build": "tsc -p tsconfig.build.json",
    "start": "egg-scripts start dist --daemon=false --workers=1",
    "stop": "egg-scripts stop"
  },
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "dotenv": "^16.4.5",
    "egg": "^4.0.0",
    "egg-cors": "^2.2.3",
    "egg-jwt": "^3.1.7",
    "egg-redis": "^2.4.0",
    "egg-scripts": "^2.17.0",
    "egg-sequelize": "^6.0.0",
    "egg-validate": "^2.0.2",
    "prom-client": "^15.1.3",
    "sequelize": "^6.37.3"
  },
  "devDependencies": {
    "@modelcontextprotocol/sdk": "^1.12.0",
    "@types/bcryptjs": "^2.4.6",
    "@types/node": "^20.14.0",
    "@types/sinon": "^17.0.3",
    "@types/supertest": "^6.0.2",
    "chai": "^4.4.1",
    "egg-bin": "^6.2.0",
    "egg-mock": "^5.10.4",
    "sinon": "^17.0.2",
    "supertest": "^7.0.0",
    "ts-node": "^10.9.2",
    "typescript": "^5.4.5",
    "zod": "^3.23.8"
  },
  "engines": {
    "node": ">=20.0.0"
  }
}
```

**npm scripts 详解**：

| 脚本               | 作用                                                |
| ---------------- | ------------------------------------------------- |
| `dev`            | 以 ts-node 方式热启动开发服务器（egg-bin dev），改代码自动重启，端口 7001 |
| `debug`          | 带断点调试模式，可配合 VSCode                                |
| `test` / `cov`   | 跑测试 / 跑测试并输出覆盖率报告                                 |
| `seed`           | 初始化数据库表结构 + 写入演示数据（第 5 章创建）                       |
| `mcp`            | 启动 MCP Server 独立进程（第 13 章）                        |
| `mcp:test`       | 用 MCP 客户端脚本验证 MCP Server（第 13 章）                  |
| `build`          | TypeScript 编译产出 `dist/`（第 15 章）                   |
| `start` / `stop` | 生产环境启停（egg-scripts，第 15 章）                        |

> **依赖选型说明**：
> - `bcryptjs` 而非 `bcrypt`：后者是原生 C++ 模块，Windows 下需要 Visual Studio 编译链，初学者极易踩坑；bcryptjs 纯 JS 实现，API 完全一致，性能差异在登录场景可忽略。
> - `chai` 锁定 v4：v5 起仅支持 ESM，与 egg-bin（CommonJS mocha）不兼容。
> - `zod` 与 `@modelcontextprotocol/sdk` 供第 13 章 MCP 使用。

安装依赖：

```bash
npm install --registry=https://registry.npmmirror.com
# 预期：added xxx packages，无 error（个别 deprecated 警告可忽略）
```

## 3.4 tsconfig.json（完整内容）

替换 `tsconfig.json`：

```json
{
  "compileOnSave": true,
  "compilerOptions": {
    "target": "ES2021",
    "module": "commonjs",
    "strict": true,
    "strictNullChecks": false,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "skipLibCheck": true,
    "suppressImplicitAnyIndexErrors": true,
    "forceConsistentCasingInFileNames": true,
    "noImplicitThis": false,
    "sourceMap": true,
    "outDir": "./dist"
  },
  "include": ["app", "config", "typings", "mcp", "scripts"]
}
```

再新建 `tsconfig.build.json`（生产构建排除测试脚本）：

```json
{
  "extends": "./tsconfig.json",
  "include": ["app", "config", "typings", "mcp"],
  "exclude": ["test", "scripts", "node_modules", "dist"]
}
```

> `experimentalDecorators` / `emitDecoratorMetadata`：本教程主体代码不用装饰器，但打开后可兼容 sequelize 装饰器风格与后续扩展（FAQ 第 1 条会讲相关报错）。

## 3.5 目录结构逐个讲解

当前项目最终目录结构如下（先浏览，后面章节逐个创建）：

```text
industryagent-server/
├── package.json              # 依赖与脚本
├── tsconfig.json             # TS 编译配置（开发）
├── tsconfig.build.json       # TS 编译配置（生产构建）
├── .env                      # 本地环境变量（不提交 git）
├── app/                      # ★ 应用代码（约定目录）
│   ├── router.ts             #   全部路由注册（唯一入口）
│   ├── controller/           #   控制器：解析请求、调用 service、组装响应
│   │   ├── home.ts
│   │   ├── health.ts
│   │   ├── auth.ts
│   │   ├── wafer.ts
│   │   ├── anomaly.ts
│   │   ├── rootcause.ts
│   │   ├── suggestion.ts
│   │   ├── stats.ts
│   │   └── metrics.ts
│   ├── service/              #   业务逻辑层：事务、外部调用等复杂逻辑都在这
│   │   ├── audit.ts
│   │   ├── auth.ts
│   │   ├── wafer.ts
│   │   ├── rootcause.ts
│   │   ├── suggestion.ts
│   │   └── stats.ts
│   ├── model/                #   Sequelize 数据模型（egg-sequelize 自动加载）
│   │   ├── user.ts / role.ts / user_role.ts
│   │   ├── wafer.ts
│   │   ├── audit_log.ts
│   │   ├── anomaly_event.ts
│   │   ├── agent_task.ts
│   │   ├── suggestion.ts
│   │   ├── approval.ts
│   │   ├── sensor_reading.ts
│   │   └── daily_yield_report.ts
│   ├── middleware/           #   中间件
│   │   ├── error_handler.ts
│   │   ├── request_log.ts
│   │   ├── audit_log.ts
│   │   ├── auth.ts
│   │   ├── permission.ts
│   │   ├── idempotency.ts
│   │   └── metrics.ts
│   ├── extend/context.ts     #   扩展 ctx：统一响应 helper
│   ├── core/                 #   框架无关的纯代码：错误类/错误码/熔断器/权限表
│   │   ├── biz-error.ts
│   │   ├── error-codes.ts
│   │   ├── circuit-breaker.ts
│   │   └── role-permissions.ts
│   └── schedule/             #   定时任务
│       ├── data-health-check.ts
│       └── yield-daily-report.ts
├── config/                   # ★ 配置（按环境覆盖）
│   ├── config.default.ts     #   所有环境共享
│   ├── config.local.ts       #   本地开发覆盖
│   ├── config.prod.ts        #   生产覆盖
│   ├── config.test.ts        #   测试覆盖
│   └── plugin.ts             #   插件开关
├── typings/
│   └── index.d.ts            #   TS 类型补全（给 egg 插件补类型）
├── scripts/
│   ├── init-db.ts            #   建表 + 种子数据
│   └── mcp-client.ts         #   MCP 客户端验证脚本
├── mcp/
│   └── server.ts             #   MCP Server 独立入口
├── test/                     #   测试
├── database/migrations/      #   （可选）正式迁移脚本
├── agent-mock/               #   Python 智能体 Mock 服务
│   └── main.py
├── Dockerfile
├── docker-compose.yml
└── .dockerignore
```

**Egg 分层约定（务必理解）**：

- **Controller**：只做" HTTP 世界与业务世界的翻译"——校验参数、调用 service、返回响应。**不写业务逻辑**。
- **Service**：业务逻辑层。一个方法对应一个业务动作（如"创建批次并查重"）。**不直接碰 HTTP**。
- **Model**：数据库表定义 + 表级操作。
- **Middleware**：横切关注点（鉴权、日志、审计），像洋葱一样包裹请求。

## 3.6 typings/index.d.ts（完整内容）

**做什么/为什么**：egg-jwt、egg-redis 等插件没有官方 TS 类型，需要手动声明，否则 TS 编译报错。用以下内容替换 `typings/index.d.ts`：

```typescript
import { Sequelize } from 'sequelize';

declare module 'egg' {
  interface Application {
    /** sequelize 库本身（用于取 Op、数据类型等） */
    Sequelize: typeof import('sequelize');
    /** egg-sequelize 注入的 sequelize 实例，兼模型注册表 */
    model: Sequelize & Record<string, any>;
    /** egg-jwt */
    jwt: {
      sign(payload: object, secret: string, options?: any): string;
      verify<T = any>(token: string, secret: string, options?: any): T;
      decode(token: string, options?: any): any;
    };
    /** egg-redis（ioredis 实例，这里只声明教程用到的子集） */
    redis: {
      get(key: string): Promise<string | null>;
      set(key: string, value: string, mode?: string, ttl?: number): Promise<any>;
      setnx(key: string, value: string): Promise<number>;
      expire(key: string, seconds: number): Promise<number>;
      del(...keys: string[]): Promise<number>;
      keys(pattern: string): Promise<string[]>;
      ping(): Promise<string>;
    };
  }

  interface Context {
    /** 由 auth 中间件挂载的当前登录用户（未登录为 null） */
    user: { userId: number; username: string; roles: string[] } | null;
    /** 统一响应 helper（app/extend/context.ts 提供） */
    success(data?: any, message?: string): void;
    page(rows: any[], total: number, page: number, pageSize: number, message?: string): void;
  }
}

declare module 'egg-sequelize';
declare module 'egg-validate';
declare module 'egg-jwt';
declare module 'egg-redis';
declare module 'egg-cors';
declare module 'egg-mock/bootstrap' {
  export const app: import('egg').Application;
}
```

## 3.7 第一个路由：router.ts + home controller

**做什么/为什么**：跑通"HTTP 请求 → 路由 → 控制器 → JSON 响应"最小闭环，确认骨架无问题。

`app/router.ts`（完整内容）：

```typescript
import { Application } from 'egg';

export default (app: Application) => {
  const { router, controller } = app;

  // 平台信息（欢迎页）
  router.get('/', controller.home.index);
};
```

`app/controller/home.ts`（完整内容）：

```typescript
import { Controller } from 'egg';

export default class HomeController extends Controller {
  public async index() {
    const { app } = this;
    this.ctx.success({
      name: app.name,
      version: '1.0.0',
      project: '芯联半导体良率根因分析平台 - 业务后端',
      doc: 'IA-TU-001 后端开发教程',
      time: new Date().toISOString(),
    });
  }
}
```

`app/extend/context.ts`（完整内容，统一响应从第 3 章就立好规矩）：

```typescript
import { Context } from 'egg';

/**
 * 统一响应结构：
 * 成功：{ code: 0, message: 'ok', data: ..., traceId: '...' }
 * 失败：{ code: 业务错误码, message: '...', traceId: '...' }（由错误中间件产出）
 * 分页：{ code: 0, message: 'ok', data: { rows, total, page, pageSize }, traceId }
 */
export default {
  success(this: Context, data: any = null, message = 'ok') {
    this.status = 200;
    this.body = {
      code: 0,
      message,
      data,
      traceId: this.tracer.traceId,
    };
  },

  page(this: Context, rows: any[], total: number, page: number, pageSize: number, message = 'ok') {
    this.status = 200;
    this.body = {
      code: 0,
      message,
      data: { rows, total, page, pageSize },
      traceId: this.tracer.traceId,
    };
  },
};
```

`config/plugin.ts` 暂时替换为（后续章节逐步增加）：

```typescript
export default {
  // 第 4 章起逐个开启，这里先全部留空
};
```

`config/config.default.ts` 暂时替换为最小版本（第 4 章补全）：

```typescript
import { EggAppConfig, PowerPartial } from 'egg';

export default (appInfo: EggAppConfig) => {
  const config = {} as PowerPartial<EggAppConfig>;

  // 安全密钥（用于 cookie 签名等，生产必须换成随机长串）
  config.keys = appInfo.name + '_1700000000000_1234';

  return config;
};
```

## 3.8 启动并验证

```bash
npm run dev
```

预期输出（节选）：

```text
2026-10-15 16:40:01,xxx INFO 38240 [master] node version v20.15.0
2026-10-15 16:40:02,xxx INFO 38240 [master] agent_worker#1:1 started (7480)
```

新开一个终端验证（**所有 curl 建议在 Git Bash 中执行**，Windows cmd 的引号转义是重灾区）：

```bash
curl -s http://127.0.0.1:7001/
```

预期输出：

```json
{"code":0,"message":"ok","data":{"name":"industryagent-server","version":"1.0.0","project":"芯联半导体良率根因分析平台 - 业务后端","doc":"IA-TU-001 后端开发教程","time":"2026-10-15T08:40:30.000Z"},"traceId":"...一串哈希..."}
```

看到 `code:0` 和 `traceId`，说明路由、控制器、context 扩展全部生效。**第一个里程碑达成。**

---

# 4. 配置管理

## 4.1 做什么 / 为什么

企业级项目的第一原则：**代码与环境隔离**。数据库地址、密码、JWT 密钥绝不能写死在代码里。Egg 的配置体系天然按环境分层：

- `config.default.ts`：所有环境共享的默认值
- `config.local.ts`：本地开发覆盖（仅 `EGG_SERVER_ENV=local` 生效，`npm run dev` 默认就是 local）
- `config.prod.ts`：生产覆盖
- `config.test.ts`：测试覆盖

读取顺序：default → 环境配置（后者覆盖前者）。敏感值通过 **环境变量 + dotenv** 注入。

## 4.2 .env 文件与 dotenv

项目根新建 `.env`（**此文件绝不提交 git**）：

```bash
# 数据库
IA_DB_HOST=127.0.0.1
IA_DB_PORT=5432
IA_DB_NAME=industryagent
IA_DB_USER=industry
IA_DB_PASSWORD=industry123

# Redis
IA_REDIS_HOST=127.0.0.1
IA_REDIS_PORT=6379

# 安全（生产务必换成随机 32+ 位字符串）
IA_KEYS=dev-keys-please-change-me
IA_JWT_SECRET=dev-jwt-secret-please-change-me

# Python 智能体服务地址
IA_AGENT_BASE_URL=http://127.0.0.1:8000
```

同时新建 `.gitignore`：

```text
node_modules/
dist/
logs/
run/
.env
*.log
```

## 4.3 config/plugin.ts（完整内容）

**做什么/为什么**：Egg 插件 = 官方维护的"可插拔中间件包"。开启后 `app` 上自动多出对应能力（如 `app.redis`）。

```typescript
export default {
  sequelize: { enable: true, package: 'egg-sequelize' },
  validate: { enable: true, package: 'egg-validate' },
  jwt: { enable: true, package: 'egg-jwt' },
  redis: { enable: true, package: 'egg-redis' },
  cors: { enable: true, package: 'egg-cors' },
};
```

## 4.4 config/config.default.ts（完整内容）

```typescript
import { EggAppConfig, PowerPartial } from 'egg';
import * as path from 'path';
import * as dotenv from 'dotenv';

// 把 .env 中定义的变量挂到 process.env（已存在的不覆盖）
dotenv.config();

/** 读环境变量：没有则用兜底值 */
const env = (key: string, fallback: string): string => process.env[key] || fallback;

export default (appInfo: EggAppConfig) => {
  const config = {} as PowerPartial<EggAppConfig>;

  // ---------- 安全 ----------
  config.keys = env('IA_KEYS', 'industryagent-dev-keys-please-change');

  config.security = {
    // 纯 API 服务，无页面表单，关闭 CSRF（跨域由 cors 插件接管）
    csrf: { enable: false },
  };

  config.cors = {
    origin: '*',
    allowMethods: 'GET,HEAD,PUT,POST,DELETE,PATCH',
  };

  // ---------- 全局中间件（按洋葱顺序：errorHandler 最外层） ----------
  config.middleware = ['errorHandler', 'requestLog', 'auditLog', 'metrics'];

  // ---------- 参数校验（egg-validate） ----------
  config.validate = {
    // 自动做类型转换，'10' 可通过 int 校验
    convert: true,
  };

  // ---------- 数据库（egg-sequelize） ----------
  config.sequelize = {
    dialect: 'postgres',
    database: env('IA_DB_NAME', 'industryagent'),
    host: env('IA_DB_HOST', '127.0.0.1'),
    port: Number(env('IA_DB_PORT', '5432')),
    username: env('IA_DB_USER', 'industry'),
    password: env('IA_DB_PASSWORD', 'industry123'),
    // 时区：PG 里 timestamptz 与东八区对齐，避免"差 8 小时"经典坑
    timezone: '+08:00',
    define: {
      // 字段名自动转下划线（createdAt -> created_at）
      underscored: true,
      freezeTableName: true,
    },
    // 关闭 SQL 打印（调试时可改为 msg => console.log(msg)）
    logging: false,
  };

  // ---------- Redis（egg-redis） ----------
  config.redis = {
    client: {
      port: Number(env('IA_REDIS_PORT', '6379')),
      host: env('IA_REDIS_HOST', '127.0.0.1'),
      password: env('IA_REDIS_PASSWORD', ''),
      db: Number(env('IA_REDIS_DB', '0')),
    },
  };

  // ---------- JWT（egg-jwt） ----------
  config.jwt = {
    secret: env('IA_JWT_SECRET', 'industryagent-jwt-secret-dev'),
    enable: true,
  };

  // ---------- 日志 ----------
  config.logger = {
    level: 'INFO',
    consoleLevel: 'INFO',
  };
  // 业务审计单独落一个文件，方便合规导出
  config.customLogger = {
    auditLogger: {
      file: path.join(appInfo.root, 'logs', appInfo.name, 'audit.log'),
    },
  };

  // ---------- 自定义业务配置 ----------
  (config as any).biz = {
    // Python 智能体服务基地址
    agentBaseUrl: env('IA_AGENT_BASE_URL', 'http://127.0.0.1:8000'),
    // 分页默认值
    pagination: { defaultPage: 1, defaultPageSize: 10, maxPageSize: 100 },
    // 看板缓存 TTL（秒）
    cache: { dashboardTtlSeconds: 60 },
    // 幂等键保留时长（秒）
    idempotency: { ttlSeconds: 86400 },
    // 根因分析调用智能体的容错参数
    rootcause: {
      connectTimeoutMs: 3000,
      responseTimeoutMs: 20000,
      retries: 2,
      breaker: { failureThreshold: 3, openSeconds: 30 },
    },
    // 访问令牌 / 刷新令牌有效期
    auth: {
      accessExpiresIn: '2h',
      refreshExpiresInSeconds: 7 * 24 * 3600,
    },
  };

  return config;
};
```

> 上面 `config.middleware` 引用的 `auditLog` 等中间件将在第 6、7、10、14 章创建。**在中间件文件创建之前，请先把数组里尚未存在的名字注释掉**，每章创建后逐个放开——否则启动会报 `middleware ... not found`。

## 4.5 config.local.ts / config.prod.ts / config.test.ts

`config/config.local.ts`（完整内容）：

```typescript
import { PowerPartial } from 'egg';

export default (): PowerPartial<EggAppConfig> => ({
  // 本地开发：尽可能多的日志
  logger: {
    level: 'DEBUG',
    consoleLevel: 'DEBUG',
  },
  // 本地关闭静态资源缓存等
  static: { maxAge: 0 },
});
```

`config/config.prod.ts`（完整内容）：

```typescript
import { PowerPartial } from 'egg';

export default (): PowerPartial<EggAppConfig> => ({
  // 生产：安全值必须来自环境变量，不给兜底（未设置则启动即失败，避免裸奔上线）
  keys: process.env.IA_KEYS || '',
  logger: {
    level: 'INFO',
    consoleLevel: 'WARN',
  },
  // 生产开启 SQL 慢查询日志（仅打印超过 500ms 的语句）
  sequelize: {
    logging: (sql: string, cost?: number) => {
      if (cost && cost > 500) console.warn('[slow-sql]', cost + 'ms', sql);
    },
  } as any,
});
```

`config/config.test.ts`（完整内容，第 12 章使用）：

```typescript
import { PowerPartial } from 'egg';

export default (): PowerPartial<EggAppConfig> => ({
  // 集成测试连独立的测试库，避免污染开发数据
  sequelize: {
    database: process.env.IA_TEST_DB_NAME || 'industryagent_test',
  } as any,
});
```

## 4.6 验证

```bash
npm run dev
```

若能正常启动（连不上数据库没关系，此时还没用到 sequelize 的连接重试逻辑会在首次查询时报错），且 `curl -s http://127.0.0.1:7001/` 仍返回 `code:0`，说明配置体系生效。

再做一个环境变量生效性验证——临时改 `.env` 里 `IA_DB_NAME=wrongdb` 后重启 `npm run dev`，日志中不应崩溃（连接惰性建立），但后续查询会报 `database "wrongdb" does not exist`，证明配置确实从 `.env` 读到了。验证完改回来。

---

# 5. 第一个完整模块——晶圆批次 wafer

## 5.1 做什么 / 为什么

晶圆批次（wafer）是平台最核心的主数据：每一片 12 英寸晶圆按批次投产，批次携带产品型号、工艺节点、产线、良率、缺陷数等字段。根因分析的一切都围绕"某个批次良率为什么掉了"展开。

本章按 **Model → Service → Controller → Router → curl 测试** 的顺序，把一个生产级 CRUD 模块的所有零件装配起来，并顺带建立错误体系（BizError / 错误码 / 统一错误中间件）——这是后续所有模块的模板。

## 5.2 数据模型：app/model/wafer.ts（完整内容）

**做什么/为什么**：`app/model/` 下的文件会被 egg-sequelize 自动加载，文件名决定模型挂载名（`wafer.ts` → `app.model.Wafer`）。模型即建表依据（我们用 `sync()` 自动建表）。

```typescript
import { Application } from 'egg';

/**
 * 晶圆批次表 wafers
 * 一个批次 = 一次投片的 25 片晶圆（1 个 lot）
 */
export default (app: Application) => {
  const { STRING, INTEGER, DATE, DECIMAL, ENUM, TEXT } = app.Sequelize;

  const Wafer = app.model.define('wafer', {
    id: {
      type: INTEGER,
      primaryKey: true,
      autoIncrement: true,
      comment: '主键',
    },
    batchNo: {
      type: STRING(32),
      allowNull: false,
      unique: true,
      comment: '批次号，如 B20261015001',
    },
    productModel: {
      type: STRING(64),
      allowNull: false,
      comment: '产品型号，如 MCU-XS32',
    },
    processNode: {
      type: STRING(16),
      allowNull: false,
      defaultValue: '28nm',
      comment: '工艺节点',
    },
    productionLine: {
      type: STRING(32),
      allowNull: false,
      comment: '产线，如 LINE-1 / LINE-2 / LINE-3',
    },
    quantity: {
      type: INTEGER,
      allowNull: false,
      defaultValue: 25,
      comment: '批次晶圆数量',
    },
    defectCount: {
      type: INTEGER,
      allowNull: false,
      defaultValue: 0,
      comment: '缺陷晶圆数',
    },
    yieldRate: {
      type: DECIMAL(5, 2),
      allowNull: false,
      comment: '批次良率（%），如 92.50',
    },
    status: {
      type: ENUM('producing', 'completed', 'shipped', 'quarantined'),
      allowNull: false,
      defaultValue: 'producing',
      comment: '批次状态：在制/完工/出货/隔离',
    },
    remark: {
      type: TEXT,
      allowNull: true,
      comment: '备注',
    },
    producedAt: {
      type: DATE,
      allowNull: true,
      comment: '投产时间',
    },
  }, {
    // 表名固定为 wafers（禁止单数化）
    tableName: 'wafers',
    // 启用 createdAt / updatedAt 自动维护
    timestamps: true,
    // 已在字段上显式声明，这里不再额外蛇形转换
    underscored: false,
  });

  return Wafer;
};
```

> **Migration 说明（企业实践）**：`sequelize.sync()` 适合开发与演示；生产环境推荐用迁移脚本管理表结构变更，在 `database/migrations/` 下维护。示例 `database/migrations/20261015000001-create-wafers.ts`：
>
> ```typescript
> import { QueryInterface, DataTypes } from 'sequelize';
>
> module.exports = {
>   async up(queryInterface: QueryInterface) {
>     await queryInterface.createTable('wafers', {
>       id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
>       batch_no: { type: DataTypes.STRING(32), allowNull: false, unique: true },
>       product_model: { type: DataTypes.STRING(64), allowNull: false },
>       process_node: { type: DataTypes.STRING(16), allowNull: false, defaultValue: '28nm' },
>       production_line: { type: DataTypes.STRING(32), allowNull: false },
>       quantity: { type: DataTypes.INTEGER, allowNull: false, defaultValue: 25 },
>       defect_count: { type: DataTypes.INTEGER, allowNull: false, defaultValue: 0 },
>       yield_rate: { type: DataTypes.DECIMAL(5, 2), allowNull: false },
>       status: { type: DataTypes.ENUM('producing', 'completed', 'shipped', 'quarantined'), allowNull: false, defaultValue: 'producing' },
>       remark: { type: DataTypes.TEXT, allowNull: true },
>       produced_at: { type: DataTypes.DATE, allowNull: true },
>       created_at: { type: DataTypes.DATE, allowNull: false },
>       updated_at: { type: DataTypes.DATE, allowNull: false },
>     });
>   },
>   async down(queryInterface: QueryInterface) {
>     await queryInterface.dropTable('wafers');
>   },
> };
> ```
>
> 本教程主线采用 `sync()`（见 5.8 建表脚本），两套机制了解即可。

## 5.3 错误体系：BizError 与错误码

**做什么/为什么**：业务错误（"批次已存在"）与系统错误（数据库挂了）必须区分开。前者要精确告知前端原因（业务码 + HTTP 400），后者只能笼统提示（HTTP 500）并详细记日志——**绝不能把堆栈泄露给外部**。

`app/core/error-codes.ts`（完整内容）：

```typescript
/**
 * 全局业务错误码约定：
 * 0        成功
 * 10xxx    通用业务错误
 * 20xxx    认证与权限
 * 30xxx    根因分析/智能体
 * 50000    服务器内部错误
 */
export enum ErrorCode {
  OK = 0,

  PARAM_INVALID = 10001,      // 参数校验失败
  NOT_FOUND = 10002,          // 资源不存在
  DUPLICATE = 10003,          // 重复资源
  STATE_CONFLICT = 10004,     // 状态不允许该操作
  DUPLICATE_REQUEST = 10005,  // 幂等命中：重复提交

  UNAUTHORIZED = 20001,       // 未认证
  TOKEN_EXPIRED = 20002,      // 令牌过期
  FORBIDDEN = 20003,          // 无权限
  LOGIN_FAILED = 20004,       // 用户名或密码错误
  USER_DISABLED = 20005,      // 用户被禁用

  AGENT_UNAVAILABLE = 30001,  // 智能体服务不可用（熔断/超时）
  AGENT_BAD_RESPONSE = 30002, // 智能体返回异常

  INTERNAL_ERROR = 50000,     // 服务器内部错误
}

/** 错误码 -> 默认提示文案（便于统一维护） */
export const ERROR_MESSAGES: Record<number, string> = {
  [ErrorCode.PARAM_INVALID]: '参数校验失败',
  [ErrorCode.NOT_FOUND]: '资源不存在',
  [ErrorCode.DUPLICATE]: '资源已存在',
  [ErrorCode.STATE_CONFLICT]: '当前状态不允许该操作',
  [ErrorCode.DUPLICATE_REQUEST]: '重复请求，请勿重复提交',
  [ErrorCode.UNAUTHORIZED]: '请先登录',
  [ErrorCode.TOKEN_EXPIRED]: '登录已过期，请重新登录',
  [ErrorCode.FORBIDDEN]: '没有执行该操作的权限',
  [ErrorCode.LOGIN_FAILED]: '用户名或密码错误',
  [ErrorCode.USER_DISABLED]: '账号已被禁用，请联系管理员',
  [ErrorCode.AGENT_UNAVAILABLE]: '智能体服务暂不可用，请稍后重试',
  [ErrorCode.AGENT_BAD_RESPONSE]: '智能体服务返回异常',
  [ErrorCode.INTERNAL_ERROR]: '服务器内部错误',
};
```

`app/core/biz-error.ts`（完整内容）：

```typescript
/**
 * 业务错误：所有可预期的、需要告知用户原因的错误。
 * 中间件 error_handler 会把它转成 { code, message } 响应。
 */
export class BizError extends Error {
  /** 业务错误码（见 ErrorCode） */
  public readonly code: number;
  /** HTTP 状态码 */
  public readonly httpStatus: number;

  constructor(code: number, message?: string, httpStatus = 400) {
    super(message);
    this.name = 'BizError';
    this.code = code;
    this.httpStatus = httpStatus;
  }
}
```

## 5.4 全局错误处理中间件：app/middleware/error_handler.ts（完整内容）

```typescript
import { Context, Next } from 'egg';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';

/**
 * 统一错误出口（最外层洋葱）。
 * - BizError          -> 对应 HTTP 状态 + 业务码
 * - ValidationError   -> 422（egg-validate 抛出）
 * - 其它未知错误      -> 500 + 详细日志（不泄露给客户端）
 */
export default () => {
  return async function errorHandler(ctx: Context, next: Next) {
    try {
      await next();

      // 路由未命中（404）也统一成 JSON
      if (ctx.status === 404 && !ctx.body) {
        ctx.status = 404;
        ctx.body = { code: 40400, message: '接口不存在', traceId: ctx.tracer.traceId };
      }
    } catch (err: any) {
      // 让 egg 框架自身也记录一份（logs/<app>/common-error.log）
      ctx.app.emit('error', err, ctx);

      if (err instanceof BizError) {
        ctx.status = err.httpStatus;
        ctx.body = { code: err.code, message: err.message, traceId: ctx.tracer.traceId };
      } else if (err.name === 'ValidationError' || Array.isArray(err.errors)) {
        // egg-validate（parameter）参数校验失败
        ctx.status = 422;
        ctx.body = {
          code: ErrorCode.PARAM_INVALID,
          message: '参数校验失败',
          detail: err.errors,
          traceId: ctx.tracer.traceId,
        };
      } else {
        // 未知错误：详细记录，对外只报笼统信息
        ctx.logger.error('[unhandled] %s %s ->', ctx.method, ctx.path, err);
        ctx.status = 500;
        ctx.body = {
          code: ErrorCode.INTERNAL_ERROR,
          message: '服务器内部错误',
          traceId: ctx.tracer.traceId,
        };
      }
    }
  };
};
```

放开 `config.default.ts` 中 `config.middleware` 的 `'errorHandler'`。

## 5.5 Service 层：app/service/wafer.ts（完整内容）

**做什么/为什么**：业务规则（批次号唯一、分页上限、事务）集中在 service。**创建操作演示 Sequelize 事务**：先查重再插入的两步必须放进同一事务，否则并发下有竞态。

```typescript
import { Service } from 'egg';
import { Op } from 'sequelize';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';

export interface WaferListParam {
  page: number;
  pageSize: number;
  batchNo?: string;
  status?: string;
  productionLine?: string;
}

export interface WaferCreateParam {
  batchNo: string;
  productModel: string;
  processNode: string;
  productionLine: string;
  quantity: number;
  yieldRate: number;
  defectCount?: number;
  status?: string;
  remark?: string;
  producedAt?: string;
}

export default class WaferService extends Service {

  /** 分页查询批次列表（支持批次号模糊/状态/产线过滤） */
  public async list(param: WaferListParam) {
    const where: any = {};
    if (param.batchNo) {
      where.batchNo = { [Op.like]: `%${param.batchNo}%` };
    }
    if (param.status) where.status = param.status;
    if (param.productionLine) where.productionLine = param.productionLine;

    const result = await this.app.model.Wafer.findAndCountAll({
      where,
      offset: (param.page - 1) * param.pageSize,
      limit: param.pageSize,
      order: [[ 'id', 'DESC' ]],
    });

    return { rows: result.rows, total: result.count };
  }

  /** 批次详情 */
  public async detail(id: number) {
    const wafer = await this.app.model.Wafer.findByPk(id);
    if (!wafer) {
      throw new BizError(ErrorCode.NOT_FOUND, `批次不存在（id=${id}）`, 404);
    }
    return wafer;
  }

  /** 创建批次（事务示例：查重 + 插入在同一事务内，避免并发竞态） */
  public async create(data: WaferCreateParam) {
    return await this.app.model.transaction(async t => {
      const exists = await this.app.model.Wafer.findOne({
        where: { batchNo: data.batchNo },
        transaction: t,
      });
      if (exists) {
        throw new BizError(ErrorCode.DUPLICATE, `批次号 ${data.batchNo} 已存在`);
      }
      const wafer = await this.app.model.Wafer.create({
        ...data,
        // 字符串日期转 Date
        producedAt: data.producedAt ? new Date(data.producedAt) : new Date(),
      }, { transaction: t });
      return wafer;
    });
  }

  /** 更新批次 */
  public async update(id: number, data: Partial<WaferCreateParam>) {
    const wafer = await this.detail(id);
    return await wafer.update(data);
  }

  /** 删除批次（软删除思路可扩展 deletedAt 字段，这里演示物理删除） */
  public async remove(id: number) {
    const wafer = await this.detail(id);
    await wafer.destroy();
    return { id };
  }
}
```

## 5.6 Controller 层：app/controller/wafer.ts（完整内容）

**做什么/为什么**：Controller 只干三件事——取参、校验（egg-validate 规则）、调 service、统一响应。校验规则使用 [parameter](https://github.com/node-modules/parameter) 语法。

```typescript
import { Controller } from 'egg';

/** 创建批次校验规则 */
const createRule = {
  batchNo: { type: 'string', required: true, min: 6, max: 32 },
  productModel: { type: 'string', required: true, min: 1, max: 64 },
  processNode: { type: 'string', required: true, min: 1, max: 16 },
  productionLine: { type: 'string', required: true, min: 1, max: 32 },
  quantity: { type: 'int', required: true, min: 1, max: 1000 },
  yieldRate: { type: 'number', required: true, min: 0, max: 100 },
  defectCount: { type: 'int', required: false, min: 0 },
  status: { type: 'enum', values: [ 'producing', 'completed', 'shipped', 'quarantined' ], required: false },
  remark: { type: 'string', required: false, allowEmpty: true, max: 2000 },
  producedAt: { type: 'datetime', required: false },
};

/** 更新批次校验规则（全部可选） */
const updateRule = {
  productModel: { type: 'string', required: false, min: 1, max: 64 },
  processNode: { type: 'string', required: false, min: 1, max: 16 },
  productionLine: { type: 'string', required: false, min: 1, max: 32 },
  quantity: { type: 'int', required: false, min: 1, max: 1000 },
  yieldRate: { type: 'number', required: false, min: 0, max: 100 },
  defectCount: { type: 'int', required: false, min: 0 },
  status: { type: 'enum', values: [ 'producing', 'completed', 'shipped', 'quarantined' ], required: false },
  remark: { type: 'string', required: false, allowEmpty: true, max: 2000 },
};

export default class WaferController extends Controller {

  /** GET /api/v1/wafers?page=1&pageSize=10&batchNo=&status= */
  public async list() {
    const { ctx } = this;
    const { pagination } = (ctx.app.config as any).biz;

    // GET 的参数在 query 上，validate 第二个参数必须显式传
    ctx.validate({
      page: { type: 'int', required: false, min: 1 },
      pageSize: { type: 'int', required: false, min: 1, max: 500 },
      batchNo: { type: 'string', required: false, allowEmpty: true },
      status: { type: 'enum', values: [ 'producing', 'completed', 'shipped', 'quarantined' ], required: false },
      productionLine: { type: 'string', required: false, allowEmpty: true },
    }, ctx.query as any);

    const page = Math.max(parseInt(String(ctx.query.page ?? ''), 10) || pagination.defaultPage, 1);
    const pageSize = Math.min(
      parseInt(String(ctx.query.pageSize ?? ''), 10) || pagination.defaultPageSize,
      pagination.maxPageSize,
    );

    const result = await ctx.service.wafer.list({
      page,
      pageSize,
      batchNo: ctx.query.batchNo || undefined,
      status: ctx.query.status || undefined,
      productionLine: ctx.query.productionLine || undefined,
    });
    ctx.page(result.rows, result.total, page, pageSize);
  }

  /** GET /api/v1/wafers/:id */
  public async detail() {
    const id = parseInt(this.ctx.params.id, 10);
    this.ctx.success(await this.ctx.service.wafer.detail(id));
  }

  /** POST /api/v1/wafers */
  public async create() {
    const { ctx } = this;
    ctx.validate(createRule);
    const wafer = await ctx.service.wafer.create(ctx.request.body as any);
    ctx.status = 201;
    ctx.success(wafer, '创建成功');
  }

  /** PUT /api/v1/wafers/:id */
  public async update() {
    const { ctx } = this;
    ctx.validate(updateRule);
    const id = parseInt(ctx.params.id, 10);
    const wafer = await ctx.service.wafer.update(id, ctx.request.body as any);
    ctx.success(wafer, '更新成功');
  }

  /** DELETE /api/v1/wafers/:id */
  public async remove() {
    const id = parseInt(this.ctx.params.id, 10);
    const result = await this.ctx.service.wafer.remove(id);
    ctx.success(result, '删除成功');
  }
}
```

## 5.7 注册路由：app/router.ts（完整内容）

```typescript
import { Application } from 'egg';

export default (app: Application) => {
  const { router, controller } = app;

  // 平台信息
  router.get('/', controller.home.index);

  // ---------------- 晶圆批次 ----------------
  router.get('/api/v1/wafers', controller.wafer.list);
  router.get('/api/v1/wafers/:id', controller.wafer.detail);
  router.post('/api/v1/wafers', controller.wafer.create);
  router.put('/api/v1/wafers/:id', controller.wafer.update);
  router.delete('/api/v1/wafers/:id', controller.wafer.remove);
};
```

> URL 统一带 `/api/v1` 版本前缀——上线后若有破坏性改动，可以平行开 `/api/v2`，这是企业 API 的基本素养。

## 5.8 建表与种子数据：scripts/init-db.ts（完整内容）

**做什么/为什么**：Sequelize 的 `sync()` 会按模型自动建表。种子脚本同时灌入演示数据，让后续每一章的 curl 都有数据可查。**本脚本会清空并重建所有表，仅用于开发环境。**

这里演示一个重要技巧：**在 Egg 进程外复用 app/model 下的模型定义**——构造一个只含 `Sequelize`（数据类型）与 `model`（sequelize 实例）两个属性的"迷你 app"传给模型工厂函数。第 13 章 MCP Server 也会用同样的技巧。

```typescript
import { Sequelize, DataTypes } from 'sequelize';
import * as bcrypt from 'bcryptjs';
import * as dotenv from 'dotenv';

dotenv.config();

async function main() {
  const sequelize = new Sequelize(
    process.env.IA_DB_NAME || 'industryagent',
    process.env.IA_DB_USER || 'industry',
    process.env.IA_DB_PASSWORD || 'industry123',
    {
      host: process.env.IA_DB_HOST || '127.0.0.1',
      port: Number(process.env.IA_DB_PORT || 5432),
      dialect: 'postgres',
      logging: false,
      timezone: '+08:00',
    },
  );

  // 构造 egg-sequelize 兼容的"迷你 app"，复用 app/model 下的模型定义
  const miniApp: any = { Sequelize: DataTypes, model: sequelize };

  const Wafer = (await import('../app/model/wafer')).default(miniApp);

  console.log('[seed] 同步表结构（force 会清空重建）...');
  await sequelize.sync({ force: true });
  console.log('[seed] 表结构同步完成');

  // ---------- 种子数据：晶圆批次 ----------
  const now = Date.now();
  const hoursAgo = (h: number) => new Date(now - h * 3600 * 1000);
  await Wafer.bulkCreate([
    { batchNo: 'B20261015001', productModel: 'MCU-XS32', processNode: '28nm', productionLine: 'LINE-1', quantity: 25, defectCount: 4, yieldRate: 91.20, status: 'completed', producedAt: hoursAgo(30), remark: '良率低于目标 95%，已触发根因分析' },
    { batchNo: 'B20261015002', productModel: 'PMIC-65W', processNode: '40nm', productionLine: 'LINE-2', quantity: 25, defectCount: 1, yieldRate: 96.80, status: 'completed', producedAt: hoursAgo(26) },
    { batchNo: 'B20261015003', productModel: 'MCU-XS32', processNode: '28nm', productionLine: 'LINE-1', quantity: 25, defectCount: 6, yieldRate: 89.60, status: 'completed', producedAt: hoursAgo(8), remark: '连续两批良率下滑' },
    { batchNo: 'B20261015004', productModel: 'RF-SOC', processNode: '22nm', productionLine: 'LINE-3', quantity: 25, defectCount: 0, yieldRate: 97.50, status: 'shipped', producedAt: hoursAgo(72) },
    { batchNo: 'B20261015005', productModel: 'PMIC-65W', processNode: '40nm', productionLine: 'LINE-2', quantity: 25, yieldRate: 0, status: 'producing', producedAt: hoursAgo(2), remark: '在制批次' },
  ]);
  console.log('[seed] 晶圆批次 5 条写入完成');

  // 用户/角色等种子数据在第 7 章追加

  await sequelize.close();
  console.log('[seed] 全部完成 ✔');
}

main().catch(e => {
  console.error('[seed] 失败：', e);
  process.exit(1);
});
```

执行：

```bash
npm run seed
```

预期输出：

```text
[seed] 同步表结构（force 会清空重建）...
[seed] 表结构同步完成
[seed] 晶圆批次 5 条写入完成
[seed] 全部完成 ✔
```

## 5.9 curl 全接口测试演示

确保 `npm run dev` 在跑，然后依次执行（Git Bash）：

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

---

# 6. 中间件体系：审计日志与请求链路

## 6.1 做什么 / 为什么

第 5 章已经有了一个中间件（errorHandler）。本章补齐企业级后端的两大横切关注点：

1. **审计日志**：手册明确要求"审批人、时间、理由"可追溯，数据不出域。任何"写操作"都要落 `audit_logs` 表：**谁、何时、对什么路径、做了什么、结果如何、耗时多少**。
2. **请求日志 + traceId**：每个请求分配一个 traceId 贯穿日志，出问题时能按 traceId 捞出完整链路——这是排障的生命线。

先建审计表模型 `app/model/audit_log.ts`（完整内容）：

```typescript
import { Application } from 'egg';

/**
 * 审计日志表 audit_logs：只插入、不修改、不删除（合规要求）
 */
export default (app: Application) => {
  const { INTEGER, STRING, TEXT, DATE } = app.Sequelize;

  const AuditLog = app.model.define('audit_log', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    userId: { type: INTEGER, allowNull: true, comment: '操作人 id（未登录为 null）' },
    username: { type: STRING(64), allowNull: true, comment: '操作人用户名' },
    method: { type: STRING(8), allowNull: false, comment: 'HTTP 方法' },
    path: { type: STRING(255), allowNull: false, comment: '请求路径' },
    status: { type: INTEGER, allowNull: false, comment: '响应状态码' },
    durationMs: { type: INTEGER, allowNull: false, comment: '耗时（毫秒）' },
    ip: { type: STRING(64), allowNull: true },
    requestSummary: { type: TEXT, allowNull: true, comment: '请求体摘要（脱敏后）' },
    errorMessage: { type: TEXT, allowNull: true, comment: '失败原因' },
    traceId: { type: STRING(64), allowNull: true },
  }, {
    tableName: 'audit_logs',
    timestamps: false,
    createdAt: 'created_at',
    updatedAt: false,
  });

  return AuditLog;
};
```

## 6.2 审计 Service：app/service/audit.ts（完整内容）

```typescript
import { Service } from 'egg';

export interface AuditRecord {
  userId: number | null;
  username: string | null;
  method: string;
  path: string;
  status: number;
  durationMs: number;
  ip: string;
  requestSummary: string | null;
  errorMessage: string | null;
  traceId: string;
}

export default class AuditService extends Service {

  /** 落库一条审计记录（调用方不关心成败，失败只打警告日志） */
  public async record(data: AuditRecord) {
    await this.app.model.AuditLog.create(data as any);
    // 同时输出到独立的审计文件日志（双保险：即使表被误清，文件还在）
    (this.app as any).customLogger.auditLogger.info(JSON.stringify(data));
  }
}
```

> 注意：`timestamps: false` + `createdAt: 'created_at'` 的组合让 sequelize 自动写 created_at 但不维护 updated_at。

## 6.3 审计中间件：app/middleware/audit_log.ts（完整内容）

```typescript
import { Context, Next } from 'egg';

/** 需要审计的方法：一切"写"操作 */
const AUDITED_METHODS = new Set([ 'POST', 'PUT', 'DELETE', 'PATCH' ]);
/** 请求体摘要最大长度 */
const MAX_SUMMARY = 512;

/** 摘要：JSON 序列化 + 脱敏（密码类字段直接抹掉） */
function summarize(body: any): string | null {
  if (!body || typeof body !== 'object') return null;
  const clone: any = { ...body };
  for (const key of Object.keys(clone)) {
    if (/password|secret|token/i.test(key)) clone[key] = '***';
  }
  const s = JSON.stringify(clone);
  return s.length > MAX_SUMMARY ? s.slice(0, MAX_SUMMARY) + '...' : s;
}

/**
 * 审计中间件：记录"谁/何时/对什么/做了什么/耗时/结果"
 * - 放在鉴权中间件外层：await next() 之后 ctx.user 已被内层中间件挂载
 * - fire-and-forget：审计写入失败不阻塞业务响应
 */
export default () => {
  return async function auditLog(ctx: Context, next: Next) {
    const start = Date.now();
    let caughtError: any = null;

    try {
      await next();
    } catch (e) {
      caughtError = e; // 先接住，记录完再抛给外层 error_handler
    }

    const duration = Date.now() - start;

    if (AUDITED_METHODS.has(ctx.method)) {
      const record = {
        userId: ctx.user ? ctx.user.userId : null,
        username: ctx.user ? ctx.user.username : 'anonymous',
        method: ctx.method,
        path: ctx.path,
        // 出错时外层尚未设置状态码，用 500 近似
        status: caughtError ? 500 : ctx.status,
        durationMs: duration,
        ip: ctx.ip,
        requestSummary: summarize(ctx.request.body),
        errorMessage: caughtError ? String(caughtError.message).slice(0, 512) : null,
        traceId: ctx.tracer.traceId,
      };
      // fire-and-forget：不 await，失败只记警告
      ctx.service.audit.record(record).catch((e: Error) => {
        ctx.logger.warn('[audit] 审计写入失败: %s', e.message);
      });
    }

    if (caughtError) throw caughtError;
  };
};
```

## 6.4 请求日志与 traceId：app/middleware/request_log.ts（完整内容）

**做什么/为什么**：Egg 自带 `ctx.tracer.traceId`，但默认不打到业务日志里。这里把每个请求输出为**结构化 JSON 单行日志**（第 14 章会讲为什么必须是单行 JSON：方便 ELK/Loki 采集解析），并把 traceId 回写到响应头，前端报障时只需提供这个 ID。

```typescript
import { Context, Next } from 'egg';

/**
 * 结构化请求日志中间件：
 * - 每请求输出一行 JSON（含 traceId）
 * - 响应头回写 X-Trace-Id，便于用户报障定位
 */
export default () => {
  return async function requestLog(ctx: Context, next: Next) {
    const start = Date.now();
    await next();
    const durationMs = Date.now() - start;

    ctx.set('X-Trace-Id', ctx.tracer.traceId);

    ctx.logger.info(JSON.stringify({
      ts: new Date().toISOString(),
      kind: 'http_request',
      traceId: ctx.tracer.traceId,
      method: ctx.method,
      path: ctx.path,
      status: ctx.status,
      durationMs,
      userId: ctx.user ? ctx.user.userId : null,
      ip: ctx.ip,
    }));
  };
};
```

## 6.5 启用与验证

确认 `config.default.ts` 的 `config.middleware` 为：

```typescript
config.middleware = ['errorHandler', 'requestLog', 'auditLog'];
```

重启 `npm run dev`，然后执行一次创建操作：

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

---

# 7. 用户与鉴权：JWT + RBAC

## 7.1 做什么 / 为什么

手册的组织约束明确："关键建议需人工审批""RBAC、审计日志"。本章实现：

- **users / roles / user_roles 三张表**（经典 RBAC 多对多）
- **登录**：查库 + bcrypt 密码比对 + 签发 access token（2h）
- **刷新 token**：长效 refresh token（7 天）存 Redis 白名单，可主动失效
- **token 校验中间件**：解析 payload 挂到 `ctx.user`
- **权限校验中间件**：路由级声明所需权限，按角色映射表判定

## 7.2 数据模型（完整内容）

`app/model/user.ts`：

```typescript
import { Application } from 'egg';

/** 用户表 users */
export default (app: Application) => {
  const { INTEGER, STRING, DATE } = app.Sequelize;

  const User = app.model.define('user', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    username: { type: STRING(64), allowNull: false, unique: true },
    passwordHash: { type: STRING(128), allowNull: false, comment: 'bcrypt 哈希，永不明文存密码' },
    realName: { type: STRING(64), allowNull: true },
    email: { type: STRING(128), allowNull: true },
    status: { type: STRING(16), allowNull: false, defaultValue: 'active', comment: 'active / disabled' },
    lastLoginAt: { type: DATE, allowNull: true },
  }, {
    tableName: 'users',
    timestamps: true,
  });

  return User;
};
```

`app/model/role.ts`：

```typescript
import { Application } from 'egg';

/** 角色表 roles */
export default (app: Application) => {
  const { INTEGER, STRING, TEXT } = app.Sequelize;

  const Role = app.model.define('role', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    code: { type: STRING(32), allowNull: false, unique: true, comment: '角色编码：admin/process_engineer/reviewer' },
    name: { type: STRING(64), allowNull: false },
    description: { type: TEXT, allowNull: true },
  }, {
    tableName: 'roles',
    timestamps: true,
  });

  return Role;
};
```

`app/model/user_role.ts`：

```typescript
import { Application } from 'egg';

/** 用户-角色关联表 user_roles */
export default (app: Application) => {
  const { INTEGER } = app.Sequelize;

  const UserRole = app.model.define('user_role', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    userId: { type: INTEGER, allowNull: false },
    roleId: { type: INTEGER, allowNull: false },
  }, {
    tableName: 'user_roles',
    timestamps: true,
    indexes: [
      { unique: true, fields: [ 'userId', 'roleId' ] },
    ],
  });

  return UserRole;
};
```

## 7.3 角色-权限映射：app/core/role-permissions.ts（完整内容）

**做什么/为什么**：把"角色拥有哪些权限"做成一张静态映射表（权限点编码与路由声明一一对应）。小型系统这样最直观；更复杂的场景可升级为数据库存储 + OPA 策略引擎。

```typescript
/**
 * 角色 -> 权限点映射
 * 权限点命名：<资源>:<动作>，如 wafer:create
 */
export const ROLE_PERMISSIONS: Record<string, string[]> = {
  admin: [ '*' ],                                                          // 管理员：全部权限
  process_engineer: [ 'wafer:create', 'wafer:update', 'rootcause:trigger' ], // 工艺工程师
  reviewer: [ 'rootcause:trigger', 'suggestion:approve' ],                 // 审批员（质量/主管）
  viewer: [],                                                             // 只读
};

/** 判断一组角色是否拥有某权限 */
export function hasPermission(roles: string[], permission: string): boolean {
  return roles.some(role => {
    const perms = ROLE_PERMISSIONS[role] || [];
    return perms.includes('*') || perms.includes(permission);
  });
}
```

## 7.4 鉴权中间件：app/middleware/auth.ts（完整内容）

**做什么/为什么**：从 `Authorization: Bearer <token>` 解析 JWT，把 payload 挂到 `ctx.user`，供后续所有业务代码（以及审计中间件）使用。

```typescript
import { Context, Next } from 'egg';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';

/**
 * JWT 鉴权中间件（路由级按需挂载，而非全局）
 * 用法：router.get('/api/v1/xxx', auth(), controller.xxx)
 */
export default () => {
  return async function jwtAuth(ctx: Context, next: Next) {
    const header = ctx.get('authorization');
    const token = header ? header.replace(/^Bearer\s+/i, '') : '';
    if (!token) {
      throw new BizError(ErrorCode.UNAUTHORIZED, '缺少访问令牌（Authorization: Bearer <token>）', 401);
    }
    try {
      const payload = ctx.app.jwt.verify(token, ctx.app.config.jwt.secret);
      ctx.user = {
        userId: payload.userId,
        username: payload.username,
        roles: payload.roles || [],
      };
    } catch (e: any) {
      // 区分"过期"与"非法"，前端据此决定是否静默刷新
      if (e && e.name === 'TokenExpiredError') {
        throw new BizError(ErrorCode.TOKEN_EXPIRED, '登录已过期，请刷新令牌或重新登录', 401);
      }
      throw new BizError(ErrorCode.UNAUTHORIZED, '无效的访问令牌', 401);
    }
    await next();
  };
};
```

## 7.5 权限中间件：app/middleware/permission.ts（完整内容）

**做什么/为什么**：路由级声明"这个接口需要什么权限"。Egg 支持在 `router.get(path, ...middlewares, handler)` 里叠加中间件，我们把权限作为中间件工厂的入参（这是"路由 meta 标注权限"风格；若偏好装饰器风格，可基于 `emitDecoratorMetadata` 反射方法所属做，思路相同，工程上路由式更直观可控）。

```typescript
import { Context, Next } from 'egg';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';
import { hasPermission } from '../core/role-permissions';

/**
 * 权限校验中间件工厂
 * 用法：router.post('/api/v1/wafers', auth(), permission({ permission: 'wafer:create' }), controller.wafer.create)
 */
export default (options: { permission: string }, _app: any) => {
  return async function requirePermission(ctx: Context, next: Next) {
    const roles = ctx.user ? ctx.user.roles : [];
    if (!hasPermission(roles, options.permission)) {
      throw new BizError(
        ErrorCode.FORBIDDEN,
        `没有权限：需要 ${options.permission}（当前角色：${roles.join(',') || '无'}）`,
        403,
      );
    }
    await next();
  };
};
```

## 7.6 认证 Service：app/service/auth.ts（完整内容）

```typescript
import { Service } from 'egg';
import * as bcrypt from 'bcryptjs';
import * as crypto from 'crypto';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';

const REFRESH_PREFIX = 'ia:refresh:';

export default class AuthService extends Service {

  /** 登录：查库 + bcrypt 比对 + 签发双 token */
  public async login(username: string, password: string) {
    const { app } = this;
    const user = await app.model.User.findOne({ where: { username } });
    // 注意：用户不存在与密码错误返回同一话术，避免用户名枚举攻击
    if (!user) throw new BizError(ErrorCode.LOGIN_FAILED, '用户名或密码错误');

    const ok = await bcrypt.compare(password, user.passwordHash);
    if (!ok) throw new BizError(ErrorCode.LOGIN_FAILED, '用户名或密码错误');

    if (user.status !== 'active') {
      throw new BizError(ErrorCode.USER_DISABLED, '账号已被禁用，请联系管理员', 403);
    }

    const roles = await this._getUserRoles(user.id);

    // 更新最后登录时间（失败不影响主流程）
    user.update({ lastLoginAt: new Date() }).catch(() => {});

    return this._issueTokens({ userId: user.id, username: user.username, roles });
  }

  /** 刷新 access token：refresh token 必须仍在 Redis 白名单内 */
  public async refresh(refreshToken: string) {
    const { app } = this;
    let payload: any;
    try {
      payload = app.jwt.verify(refreshToken, app.config.jwt.secret);
    } catch (e) {
      throw new BizError(ErrorCode.TOKEN_EXPIRED, '刷新令牌无效或已过期，请重新登录', 401);
    }
    if (payload.type !== 'refresh') {
      throw new BizError(ErrorCode.UNAUTHORIZED, '请使用刷新令牌调用本接口', 401);
    }
    const userIdInRedis = await app.redis.get(REFRESH_PREFIX + payload.jti);
    if (!userIdInRedis || userIdInRedis !== String(payload.userId)) {
      // 已登出或被管理员吊销
      throw new BizError(ErrorCode.TOKEN_EXPIRED, '刷新令牌已失效，请重新登录', 401);
    }
    const roles = await this._getUserRoles(payload.userId);
    return this._issueTokens({ userId: payload.userId, username: payload.username, roles });
  }

  /** 登出：吊销 refresh token（access token 短时效，自然过期即可） */
  public async logout(refreshToken?: string) {
    if (!refreshToken) return { revoked: false };
    let payload: any = null;
    try {
      payload = this.app.jwt.verify(refreshToken, this.app.config.jwt.secret);
    } catch { /* 无效 token 直接视为已吊销 */ }
    if (payload && payload.jti) {
      await this.app.redis.del(REFRESH_PREFIX + payload.jti);
    }
    return { revoked: true };
  }

  /** 签发 access + refresh 双 token */
  private async _issueTokens(payload: { userId: number; username: string; roles: string[] }) {
    const { app } = this;
    const authCfg = (app.config as any).biz.auth;

    const accessToken = app.jwt.sign(
      { userId: payload.userId, username: payload.username, roles: payload.roles, type: 'access' },
      app.config.jwt.secret,
      { expiresIn: authCfg.accessExpiresIn },
    );

    const jti = crypto.randomUUID();
    const refreshToken = app.jwt.sign(
      { userId: payload.userId, username: payload.username, jti, type: 'refresh' },
      app.config.jwt.secret,
      { expiresIn: authCfg.refreshExpiresInSeconds },
    );
    // refresh token 白名单写 Redis：可主动吊销
    await app.redis.set(
      REFRESH_PREFIX + jti,
      String(payload.userId),
      'EX',
      authCfg.refreshExpiresInSeconds,
    );

    return {
      accessToken,
      refreshToken,
      tokenType: 'Bearer',
      expiresInSeconds: 2 * 3600,
      user: {
        userId: payload.userId,
        username: payload.username,
        realName: await this._getRealName(payload.userId),
        roles: payload.roles,
      },
    };
  }

  private async _getUserRoles(userId: number): Promise<string[]> {
    const userRoles = await this.app.model.UserRole.findAll({ where: { userId } });
    if (userRoles.length === 0) return [];
    const roleIds = userRoles.map(ur => ur.roleId);
    const roles = await this.app.model.Role.findAll({ where: { id: roleIds } });
    return roles.map(r => r.code);
  }

  private async _getRealName(userId: number): Promise<string | null> {
    const user = await this.app.model.User.findByPk(userId);
    return user ? user.realName : null;
  }
}
```

## 7.7 认证 Controller：app/controller/auth.ts（完整内容）

```typescript
import { Controller } from 'egg';

const loginRule = {
  username: { type: 'string', required: true, min: 3, max: 64 },
  password: { type: 'string', required: true, min: 6, max: 64 },
};

const refreshRule = {
  refreshToken: { type: 'string', required: true },
};

export default class AuthController extends Controller {

  /** POST /api/v1/auth/login */
  public async login() {
    const { ctx } = this;
    ctx.validate(loginRule);
    const { username, password } = ctx.request.body as any;
    ctx.success(await ctx.service.auth.login(username, password));
  }

  /** POST /api/v1/auth/refresh */
  public async refresh() {
    const { ctx } = this;
    ctx.validate(refreshRule);
    const { refreshToken } = ctx.request.body as any;
    ctx.success(await ctx.service.auth.refresh(refreshToken));
  }

  /** POST /api/v1/auth/logout（body 可选携带 refreshToken） */
  public async logout() {
    const { refreshToken } = (this.ctx.request.body || {}) as any;
    this.ctx.success(await this.ctx.service.auth.logout(refreshToken));
  }

  /** GET /api/v1/auth/profile：当前登录用户信息 */
  public async profile() {
    const user = this.ctx.user!;
    const dbUser = await this.app.model.User.findByPk(user.userId);
    this.ctx.success({
      userId: user.userId,
      username: user.username,
      realName: dbUser ? dbUser.realName : null,
      email: dbUser ? dbUser.email : null,
      roles: user.roles,
    });
  }
}
```

## 7.8 种子数据追加用户与角色

在 `scripts/init-db.ts` 的 `main()` 中，替换注释行 `// 用户/角色等种子数据在第 7 章追加` 为：

```typescript
  // ---------- 种子数据：用户与角色 ----------
  const User = (await import('../app/model/user')).default(miniApp);
  const Role = (await import('../app/model/role')).default(miniApp);
  const UserRole = (await import('../app/model/user_role')).default(miniApp);

  const [adminRole, engineerRole, reviewerRole] = await Role.bulkCreate([
    { code: 'admin', name: '管理员', description: '全部权限' },
    { code: 'process_engineer', name: '工艺工程师', description: '批次维护与根因分析' },
    { code: 'reviewer', name: '审批员', description: '建议审批' },
  ], { returning: true });

  const hash = (pwd: string) => bcrypt.hashSync(pwd, 10);
  const [adminUser, engineerUser, reviewerUser] = await User.bulkCreate([
    { username: 'admin', passwordHash: hash('Admin@123'), realName: '系统管理员', email: 'admin@xinlian.demo' },
    { username: 'engineer', passwordHash: hash('Engineer@123'), realName: '李工（工艺）', email: 'engineer@xinlian.demo' },
    { username: 'reviewer', passwordHash: hash('Reviewer@123'), realName: '王主管（质量）', email: 'reviewer@xinlian.demo' },
  ], { returning: true });

  await UserRole.bulkCreate([
    { userId: adminUser.id, roleId: adminRole.id },
    { userId: engineerUser.id, roleId: engineerRole.id },
    { userId: reviewerUser.id, roleId: reviewerRole.id },
  ]);
  console.log('[seed] 用户与角色写入完成：admin/engineer/reviewer');
```

重新执行 `npm run seed`（会清库重建，之前的测试数据无需心疼）。

## 7.9 路由更新与全链路验证

`app/router.ts` 更新为（完整内容，鉴权时代来临）：

```typescript
import { Application } from 'egg';

export default (app: Application) => {
  const { router, controller, middleware } = app;
  const auth = middleware.auth();
  const perm = middleware.permission;

  // 平台信息
  router.get('/', controller.home.index);

  // ---------------- 认证 ----------------
  router.post('/api/v1/auth/login', controller.auth.login);
  router.post('/api/v1/auth/refresh', controller.auth.refresh);
  router.post('/api/v1/auth/logout', auth, controller.auth.logout);
  router.get('/api/v1/auth/profile', auth, controller.auth.profile);

  // ---------------- 晶圆批次（需登录） ----------------
  router.get('/api/v1/wafers', auth, controller.wafer.list);
  router.get('/api/v1/wafers/:id', auth, controller.wafer.detail);
  router.post('/api/v1/wafers', auth, perm({ permission: 'wafer:create' }), controller.wafer.create);
  router.put('/api/v1/wafers/:id', auth, perm({ permission: 'wafer:update' }), controller.wafer.update);
  router.delete('/api/v1/wafers/:id', auth, perm({ permission: 'wafer:delete' }), controller.wafer.remove);
};
```

> `wafer:delete` 故意**没有**分配给任何角色（连 admin 也没有？不——admin 有 `*`）。观察：engineer 删除会 403，admin 删除会成功，这就是 RBAC 生效的证据。

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

---

# 8. 异常事件与根因分析模块

## 8.1 做什么 / 为什么

这是平台的核心业务：监控 Agent 发现异常事件（如刻蚀功率漂移、温控超标）→ 工程师触发根因分析 → 平台调用 Python 智能体服务返回 Top-3 根因嫌疑。本章涉及企业级后端最经典的场景：**调用外部服务时的容错**（超时、重试、熔断降级）。

## 8.2 数据模型（完整内容）

`app/model/anomaly_event.ts`：

```typescript
import { Application } from 'egg';

/**
 * 异常事件表 anomaly_events
 * 由监控 Agent / FDC 系统推送写入
 */
export default (app: Application) => {
  const { INTEGER, STRING, TEXT, DATE, ENUM } = app.Sequelize;

  const AnomalyEvent = app.model.define('anomaly_event', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    eventCode: { type: STRING(64), allowNull: false, unique: true, comment: '事件编号' },
    batchNo: { type: STRING(32), allowNull: true, comment: '关联批次号' },
    equipment: { type: STRING(64), allowNull: false, comment: '设备，如 ETCH-03' },
    processStage: { type: STRING(32), allowNull: false, comment: '工序：etch/deposition/litho...' },
    severity: {
      type: ENUM('critical', 'major', 'minor'),
      allowNull: false,
      defaultValue: 'minor',
    },
    title: { type: STRING(255), allowNull: false },
    description: { type: TEXT, allowNull: true },
    status: {
      type: ENUM('open', 'processing', 'closed'),
      allowNull: false,
      defaultValue: 'open',
    },
    occurredAt: { type: DATE, allowNull: false, comment: '发生时间' },
  }, {
    tableName: 'anomaly_events',
    timestamps: true,
  });

  return AnomalyEvent;
};
```

`app/model/agent_task.ts`：

```typescript
import { Application } from 'egg';

/**
 * 根因分析任务表 agent_tasks
 * 记录每次"人/智能体"协作的分析任务全生命周期
 */
export default (app: Application) => {
  const { INTEGER, STRING, TEXT, DATE, ENUM, JSONB } = app.Sequelize;

  const AgentTask = app.model.define('agent_task', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    taskNo: { type: STRING(64), allowNull: false, unique: true, comment: '任务编号 RC+时间戳' },
    batchNo: { type: STRING(32), allowNull: false },
    eventId: { type: INTEGER, allowNull: true, comment: '关联异常事件' },
    status: {
      type: ENUM('pending', 'running', 'succeeded', 'failed'),
      allowNull: false,
      defaultValue: 'pending',
    },
    result: { type: JSONB, allowNull: true, comment: '智能体返回的根因分析结果' },
    error: { type: TEXT, allowNull: true },
    createdBy: { type: INTEGER, allowNull: true, comment: '触发人' },
    startedAt: { type: DATE, allowNull: true },
    finishedAt: { type: DATE, allowNull: true },
  }, {
    tableName: 'agent_tasks',
    timestamps: true,
  });

  return AgentTask;
};
```

## 8.3 Python 智能体 Mock 服务

**做什么/为什么**：真正的智能体教程见《03-智能体开发教程》。为了让后端教程可以独立跑通闭环，这里提供一个 30 行的 FastAPI Mock：睡眠 2 秒模拟推理耗时，返回结构化的 Top-3 根因 + 建议。

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

安装并启动（Python 3.10+）：

```bash
cd ~/code/industryagent/agent-mock
python -m venv .venv
source .venv/bin/activate          # Windows Git Bash: source .venv/Scripts/activate
pip install fastapi uvicorn
uvicorn main:app --host 127.0.0.1 --port 8000
```

**验证**：

```bash
curl -s http://127.0.0.1:8000/healthz
# 预期输出：{"status":"ok"}

curl -s -X POST http://127.0.0.1:8000/agent/analyze \
  -H 'Content-Type: application/json' -d '{"batch_no":"B20261015001"}'
# 预期输出：完整 JSON（root_causes + suggestions），约 2 秒后返回
```

## 8.4 熔断器：app/core/circuit-breaker.ts（完整内容）

**做什么/为什么**：当 Python 服务整宕机时，每次请求都等满超时再重试，会把上游线程/连接拖死（雪崩）。熔断器像家里的电闸：连续失败 N 次 → "跳闸"（open），直接快速失败；冷却期后放一个请求"试探"（half-open），成功才"合闸"（closed）。

**注意单例问题**：Egg 的 Service 是**每次请求新建实例**的，熔断状态必须跨请求共享——所以用模块级 `Map` 保存熔断器实例。

```typescript
import { BizError } from './biz-error';
import { ErrorCode } from './error-codes';

type BreakerState = 'closed' | 'open' | 'half_open';

export interface BreakerOptions {
  /** 连续失败多少次跳闸 */
  failureThreshold: number;
  /** 跳闸后冷却多久（毫秒） */
  openDurationMs: number;
}

export class CircuitBreaker {
  public state: BreakerState = 'closed';
  private failureCount = 0;
  private lastOpenAt = 0;

  constructor(public readonly name: string, private readonly opts: BreakerOptions) {}

  /**
   * 执行受保护调用：
   * - open 状态下快速失败（冷却期满转入 half-open 放行一次探测）
   * - 业务失败自动重试（指数退避）
   * - 全部重试失败记一次熔断失败
   */
  public async execute<T>(
    fn: (attempt: number) => Promise<T>,
    retryOpts: { retries: number; retryDelayMs: number },
  ): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastOpenAt < this.opts.openDurationMs) {
        // 跳闸中：直接降级，不再发起调用
        throw new BizError(
          ErrorCode.AGENT_UNAVAILABLE,
          `依赖服务 ${this.name} 熔断中（${Math.ceil((this.opts.openDurationMs - (Date.now() - this.lastOpenAt)) / 1000)}s 后重试）`,
          503,
        );
      }
      // 冷却期满：half-open，放行本次请求做探测
      this.state = 'half_open';
    }

    let lastError: any = null;
    for (let attempt = 0; attempt <= retryOpts.retries; attempt++) {
      try {
        const result = await fn(attempt);
        this._onSuccess(); // 探测成功：合闸
        return result;
      } catch (e) {
        // 熔断类错误不重试（避免套娃）
        if (e instanceof BizError && e.code === ErrorCode.AGENT_UNAVAILABLE) throw e;
        lastError = e;
        if (attempt < retryOpts.retries) {
          await new Promise(r => setTimeout(r, retryOpts.retryDelayMs * (attempt + 1)));
        }
      }
    }
    this._onFailure(); // 全部重试失败：跳闸
    throw lastError;
  }

  private _onSuccess() {
    this.state = 'closed';
    this.failureCount = 0;
  }

  private _onFailure() {
    this.failureCount++;
    if (this.state === 'half_open' || this.failureCount >= this.opts.failureThreshold) {
      this.state = 'open';
      this.lastOpenAt = Date.now();
    }
  }
}

/** 模块级注册表：跨请求共享熔断状态（service 每请求实例化，不能存 service 里） */
const registry = new Map<string, CircuitBreaker>();

export function getCircuitBreaker(name: string, opts: BreakerOptions): CircuitBreaker {
  let breaker = registry.get(name);
  if (!breaker) {
    breaker = new CircuitBreaker(name, opts);
    registry.set(name, breaker);
  }
  return breaker;
}
```

## 8.5 Service：app/service/anomaly.ts（完整内容）

```typescript
import { Service } from 'egg';
import { Op } from 'sequelize';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';

export default class AnomalyService extends Service {

  /** 异常事件分页查询（支持严重度/状态/设备/时间范围过滤） */
  public async list(param: {
    page: number; pageSize: number;
    severity?: string; status?: string; equipment?: string;
    batchNo?: string;
    startTime?: string; endTime?: string;
  }) {
    const where: any = {};
    if (param.severity) where.severity = param.severity;
    if (param.status) where.status = param.status;
    if (param.equipment) where.equipment = { [Op.like]: `%${param.equipment}%` };
    if (param.batchNo) where.batchNo = { [Op.like]: `%${param.batchNo}%` };
    if (param.startTime || param.endTime) {
      where.occurredAt = {};
      if (param.startTime) where.occurredAt[Op.gte] = new Date(param.startTime);
      if (param.endTime) where.occurredAt[Op.lte] = new Date(param.endTime);
    }
    const result = await this.app.model.AnomalyEvent.findAndCountAll({
      where,
      offset: (param.page - 1) * param.pageSize,
      limit: param.pageSize,
      order: [[ 'occurredAt', 'DESC' ]],
    });
    return { rows: result.rows, total: result.count };
  }

  /** 事件详情 */
  public async detail(id: number) {
    const event = await this.app.model.AnomalyEvent.findByPk(id);
    if (!event) throw new BizError(ErrorCode.NOT_FOUND, `异常事件不存在（id=${id}）`, 404);
    return event;
  }
}
```

## 8.6 Service：app/service/rootcause.ts（完整内容，含 httpclient 容错）

```typescript
import { Service } from 'egg';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';
import { getCircuitBreaker } from '../core/circuit-breaker';

export default class RootcauseService extends Service {

  /**
   * 触发根因分析：
   * 1. 校验批次存在
   * 2. 创建任务记录（pending -> running -> succeeded/failed）
   * 3. 通过 httpclient 调用 Python 智能体（超时/重试/熔断）
   * 4. 把智能体生成的建议落库（第 9 章的 suggestions 表）
   */
  public async analyze(param: { batchNo: string; eventId?: number; createdBy?: number }) {
    const { app } = this;

    const wafer = await app.model.Wafer.findOne({ where: { batchNo: param.batchNo } });
    if (!wafer) {
      throw new BizError(ErrorCode.NOT_FOUND, `批次 ${param.batchNo} 不存在`, 404);
    }

    const task = await app.model.AgentTask.create({
      taskNo: `RC-${Date.now()}-${Math.floor(Math.random() * 1000)}`,
      batchNo: param.batchNo,
      eventId: param.eventId || null,
      status: 'pending',
      createdBy: param.createdBy || null,
    });

    try {
      await task.update({ status: 'running', startedAt: new Date() });

      const agentResult = await this._callAgent({
        batchNo: param.batchNo,
        yieldRate: Number(wafer.yieldRate),
      });

      await task.update({
        status: 'succeeded',
        result: agentResult,
        finishedAt: new Date(),
      });

      // 智能体输出的建议落库（第 9 章的表，先建表再放开注释）
      if (Array.isArray(agentResult.suggestions) && agentResult.suggestions.length > 0) {
        await app.model.Suggestion.bulkCreate(
          agentResult.suggestions.map((s: any) => ({
            taskId: task.id,
            batchNo: param.batchNo,
            type: s.type,
            content: s.content,
            riskLevel: s.risk_level,
            status: 'pending',
            createdBy: param.createdBy || null,
          })),
        );
      }

      return task;
    } catch (e: any) {
      await task.update({
        status: 'failed',
        error: String(e.message || e).slice(0, 1000),
        finishedAt: new Date(),
      });
      throw e;
    }
  }

  /** 任务详情（轮询用） */
  public async taskDetail(id: number) {
    const task = await this.app.model.AgentTask.findByPk(id);
    if (!task) throw new BizError(ErrorCode.NOT_FOUND, `任务不存在（id=${id}）`, 404);
    return task;
  }

  /** 任务列表 */
  public async taskList(param: { page: number; pageSize: number; batchNo?: string; status?: string }) {
    const where: any = {};
    if (param.batchNo) where.batchNo = param.batchNo;
    if (param.status) where.status = param.status;
    const result = await this.app.model.AgentTask.findAndCountAll({
      where,
      offset: (param.page - 1) * param.pageSize,
      limit: param.pageSize,
      order: [[ 'id', 'DESC' ]],
    });
    return { rows: result.rows, total: result.count };
  }

  /**
   * 调用 Python 智能体服务（egg 内置 httpclient = ctx.curl）
   * - 超时：连接 3s / 响应 20s（数组形式 [connect, response]）
   * - 重试：最多 2 次，指数退避
   * - 熔断：连续 3 次任务失败跳闸 30s，期间快速失败
   */
  private async _callAgent(payload: { batchNo: string; yieldRate: number }) {
    const { app, ctx } = this;
    const rcCfg = (app.config as any).biz.rootcause;
    const breaker = getCircuitBreaker('python-agent', {
      failureThreshold: rcCfg.breaker.failureThreshold,
      openDurationMs: rcCfg.breaker.openSeconds * 1000,
    });

    return breaker.execute(async attempt => {
      ctx.logger.info('[rootcause] 调用智能体（第 %s 次）batchNo=%s', attempt + 1, payload.batchNo);
      const res = await ctx.curl(`${(app.config as any).biz.agentBaseUrl}/agent/analyze`, {
        method: 'POST',
        contentType: 'json',
        dataType: 'json',
        data: {
          batch_no: payload.batchNo,
          yield_rate: payload.yieldRate,
          top_k: 3,
        },
        timeout: [ rcCfg.connectTimeoutMs, rcCfg.responseTimeoutMs ],
      });
      if (res.status !== 200) {
        throw new BizError(ErrorCode.AGENT_BAD_RESPONSE, `智能体服务返回 HTTP ${res.status}`, 502);
      }
      const data = res.data as any;
      if (!data || !Array.isArray(data.root_causes)) {
        throw new BizError(ErrorCode.AGENT_BAD_RESPONSE, '智能体返回格式异常（缺少 root_causes）', 502);
      }
      return data;
    }, { retries: rcCfg.retries, retryDelayMs: 500 });
  }
}
```

## 8.7 Controller：app/controller/anomaly.ts 与 rootcause.ts（完整内容）

`app/controller/anomaly.ts`：

```typescript
import { Controller } from 'egg';

export default class AnomalyController extends Controller {

  /** GET /api/v1/anomaly-events */
  public async list() {
    const { ctx } = this;
    const { pagination } = (ctx.app.config as any).biz;
    ctx.validate({
      page: { type: 'int', required: false, min: 1 },
      pageSize: { type: 'int', required: false, min: 1, max: 500 },
      severity: { type: 'enum', values: [ 'critical', 'major', 'minor' ], required: false },
      status: { type: 'enum', values: [ 'open', 'processing', 'closed' ], required: false },
      equipment: { type: 'string', required: false, allowEmpty: true },
      batchNo: { type: 'string', required: false, allowEmpty: true },
      startTime: { type: 'datetime', required: false },
      endTime: { type: 'datetime', required: false },
    }, ctx.query as any);

    const page = Math.max(parseInt(String(ctx.query.page ?? ''), 10) || pagination.defaultPage, 1);
    const pageSize = Math.min(
      parseInt(String(ctx.query.pageSize ?? ''), 10) || pagination.defaultPageSize,
      pagination.maxPageSize,
    );

    const result = await ctx.service.anomaly.list({
      page, pageSize,
      severity: ctx.query.severity || undefined,
      status: ctx.query.status || undefined,
      equipment: ctx.query.equipment || undefined,
      batchNo: ctx.query.batchNo || undefined,
      startTime: ctx.query.startTime || undefined,
      endTime: ctx.query.endTime || undefined,
    });
    ctx.page(result.rows, result.total, page, pageSize);
  }

  /** GET /api/v1/anomaly-events/:id */
  public async detail() {
    this.ctx.success(await this.ctx.service.anomaly.detail(parseInt(this.ctx.params.id, 10)));
  }
}
```

`app/controller/rootcause.ts`：

```typescript
import { Controller } from 'egg';

const analyzeRule = {
  batchNo: { type: 'string', required: true, min: 6, max: 32 },
  eventId: { type: 'int', required: false, min: 1 },
};

export default class RootcauseController extends Controller {

  /** POST /api/v1/rootcause/analyze */
  public async analyze() {
    const { ctx } = this;
    ctx.validate(analyzeRule);
    const { batchNo, eventId } = ctx.request.body as any;
    const task = await ctx.service.rootcause.analyze({
      batchNo,
      eventId,
      createdBy: ctx.user ? ctx.user.userId : null,
    });
    ctx.status = 202; // Accepted：分析任务已受理并完成调度
    ctx.success(task, '根因分析任务已完成');
  }

  /** GET /api/v1/rootcause/tasks/:id（轮询任务状态） */
  public async taskDetail() {
    this.ctx.success(await this.ctx.service.rootcause.taskDetail(parseInt(this.ctx.params.id, 10)));
  }

  /** GET /api/v1/rootcause/tasks */
  public async taskList() {
    const { ctx } = this;
    const { pagination } = (ctx.app.config as any).biz;
    const page = Math.max(parseInt(String(ctx.query.page ?? ''), 10) || pagination.defaultPage, 1);
    const pageSize = Math.min(
      parseInt(String(ctx.query.pageSize ?? ''), 10) || pagination.defaultPageSize,
      pagination.maxPageSize,
    );
    const result = await ctx.service.rootcause.taskList({
      page, pageSize,
      batchNo: ctx.query.batchNo || undefined,
      status: ctx.query.status || undefined,
    });
    ctx.page(result.rows, result.total, page, pageSize);
  }
}
```

## 8.8 种子数据追加异常事件 & 路由

`scripts/init-db.ts` 中追加（放在用户种子数据之后）：

```typescript
  // ---------- 种子数据：异常事件 ----------
  const AnomalyEvent = (await import('../app/model/anomaly_event')).default(miniApp);
  await AnomalyEvent.bulkCreate([
    { eventCode: 'AE-20261015-001', batchNo: 'B20261015001', equipment: 'ETCH-03', processStage: 'etch', severity: 'critical', title: '刻蚀功率漂移超标', description: '射频功率较基准漂移 +3.2%，持续 42 分钟', status: 'open', occurredAt: hoursAgo(28) },
    { eventCode: 'AE-20261015-002', batchNo: 'B20261015003', equipment: 'CVD-11', processStage: 'deposition', severity: 'major', title: '腔体压力波动', description: '压力方差增大 1.8 倍', status: 'processing', occurredAt: hoursAgo(6) },
    { eventCode: 'AE-20261014-007', batchNo: 'B20261015003', equipment: 'LITH-02', processStage: 'litho', severity: 'minor', title: '光刻胶接近有效期', description: '显影对比度轻微下降', status: 'closed', occurredAt: hoursAgo(30) },
  ]);
  console.log('[seed] 异常事件 3 条写入完成');
```

`app/router.ts` 追加路由：

```typescript
  // ---------------- 异常事件 ----------------
  router.get('/api/v1/anomaly-events', auth, controller.anomaly.list);
  router.get('/api/v1/anomaly-events/:id', auth, controller.anomaly.detail);

  // ---------------- 根因分析 ----------------
  router.post('/api/v1/rootcause/analyze',
    auth, perm({ permission: 'rootcause:trigger' }), controller.rootcause.analyze);
  router.get('/api/v1/rootcause/tasks', auth, controller.rootcause.taskList);
  router.get('/api/v1/rootcause/tasks/:id', auth, controller.rootcause.taskDetail);
```

## 8.9 验证

先 `npm run seed` 重建数据，重启 `npm run dev`，确保 Python Mock 在跑，然后：

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
# 在 uvicorn 终端 Ctrl+C 停掉 Python 服务，然后连续触发 3 次分析
for i in 1 2 3; do
  curl -s -X POST http://127.0.0.1:7001/api/v1/rootcause/analyze \
    -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
    -d '{"batchNo":"B20261015002"}' | head -c 120; echo; done
# 预期：三次都失败（code 30001/30002），第 4 次开始秒回：
# {"code":30001,"message":"依赖服务 python-agent 熔断中（xxs 后重试）",...}
# 重新启动 uvicorn，30 秒冷却期后自动恢复（half-open 探测成功 -> closed）
```

> **生产提示**：本实现中 `analyze` 是同步等待智能体返回的（约 2 秒，可接受）。若单次分析耗时分钟级，应改为"立即返回 taskId + MQ 异步执行 + 前端轮询"——任务表结构已为此设计好（status/started_at/finished_at）。

---

# 9. 建议与审批模块

## 9.1 做什么 / 为什么

手册的组织约束："关键建议必须人工审批，记录审批人、时间、理由"。这是"人机协同"闭环的最后一环：智能体产出建议（suggestions）→ 审批员批准/驳回（approvals）→ 全程留痕可审计。

## 9.2 数据模型（完整内容）

`app/model/suggestion.ts`：

```typescript
import { Application } from 'egg';

/**
 * 建议表 suggestions
 * 由根因分析任务的智能体输出自动生成，等待人工审批
 */
export default (app: Application) => {
  const { INTEGER, STRING, TEXT, ENUM } = app.Sequelize;

  const Suggestion = app.model.define('suggestion', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    taskId: { type: INTEGER, allowNull: true, comment: '来源分析任务' },
    batchNo: { type: STRING(32), allowNull: true },
    type: {
      type: ENUM('param_adjust', 'hold_wafer', 'stop_line', 'inspection', 'other'),
      allowNull: false,
      defaultValue: 'other',
      comment: '建议类型：调参/隔离批次/停线/加检/其他',
    },
    content: { type: TEXT, allowNull: false, comment: '建议内容' },
    riskLevel: {
      type: ENUM('low', 'medium', 'high'),
      allowNull: false,
      defaultValue: 'low',
    },
    status: {
      type: ENUM('pending', 'approved', 'rejected'),
      allowNull: false,
      defaultValue: 'pending',
    },
    createdBy: { type: INTEGER, allowNull: true, comment: '生成该建议的任务触发人' },
  }, {
    tableName: 'suggestions',
    timestamps: true,
  });

  return Suggestion;
};
```

`app/model/approval.ts`：

```typescript
import { Application } from 'egg';

/**
 * 审批记录表 approvals：谁、何时、批/驳、理由（只插入、不可改）
 */
export default (app: Application) => {
  const { INTEGER, STRING, TEXT, DATE, ENUM } = app.Sequelize;

  const Approval = app.model.define('approval', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    suggestionId: { type: INTEGER, allowNull: false },
    action: { type: ENUM('approve', 'reject'), allowNull: false },
    reason: { type: STRING(500), allowNull: false, comment: '审批理由（必填）' },
    approverId: { type: INTEGER, allowNull: false },
    approverName: { type: STRING(64), allowNull: false },
    approvedAt: { type: DATE, allowNull: false },
  }, {
    tableName: 'approvals',
    timestamps: true,
  });

  return Approval;
};
```

## 9.3 Service：app/service/suggestion.ts（完整内容，含事务 + 行锁）

**做什么/为什么**：审批是典型的一致性敏感操作——"更新建议状态"与"写审批记录"必须同生共死（事务），且两个审批员同时点击时只能有一人成功（`SELECT ... FOR UPDATE` 行锁 + 状态检查）。

```typescript
import { Service } from 'egg';
import { Op } from 'sequelize';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';

export default class SuggestionService extends Service {

  /** 建议列表 */
  public async list(param: {
    page: number; pageSize: number;
    batchNo?: string; status?: string; riskLevel?: string;
  }) {
    const where: any = {};
    if (param.batchNo) where.batchNo = { [Op.like]: `%${param.batchNo}%` };
    if (param.status) where.status = param.status;
    if (param.riskLevel) where.riskLevel = param.riskLevel;
    const result = await this.app.model.Suggestion.findAndCountAll({
      where,
      offset: (param.page - 1) * param.pageSize,
      limit: param.pageSize,
      order: [[ 'id', 'DESC' ]],
    });
    return { rows: result.rows, total: result.count };
  }

  /**
   * 提交审批（approve / reject）
   * 事务 + 行锁：防止并发双审/漏审
   */
  public async submitApproval(param: {
    suggestionId: number;
    action: 'approve' | 'reject';
    reason: string;
    approverId: number;
    approverName: string;
  }) {
    return await this.app.model.transaction(async t => {
      // 行锁：并发审批时，后到者在此等待；先到者提交后，后到者读到的新状态会触发下面的冲突检查
      const suggestion = await this.app.model.Suggestion.findByPk(param.suggestionId, {
        transaction: t,
        lock: t.LOCK.UPDATE,
      });
      if (!suggestion) {
        throw new BizError(ErrorCode.NOT_FOUND, `建议不存在（id=${param.suggestionId}）`, 404);
      }
      if (suggestion.status !== 'pending') {
        throw new BizError(
          ErrorCode.STATE_CONFLICT,
          `该建议已${suggestion.status === 'approved' ? '批准' : '驳回'}，不能重复审批`,
        );
      }

      // 1. 更新建议状态
      await suggestion.update({
        status: param.action === 'approve' ? 'approved' : 'rejected',
      }, { transaction: t });

      // 2. 写审批记录（与状态更新同事务）
      const approval = await this.app.model.Approval.create({
        suggestionId: suggestion.id,
        action: param.action,
        reason: param.reason,
        approverId: param.approverId,
        approverName: param.approverName,
        approvedAt: new Date(),
      }, { transaction: t });

      return { suggestion, approval };
    });
  }

  /** 审批历史：某建议的全部审批流水 */
  public async approvalHistory(suggestionId: number) {
    const suggestion = await this.app.model.Suggestion.findByPk(suggestionId);
    if (!suggestion) throw new BizError(ErrorCode.NOT_FOUND, `建议不存在（id=${suggestionId}）`, 404);
    return this.app.model.Approval.findAll({
      where: { suggestionId },
      order: [[ 'id', 'ASC' ]],
    });
  }
}
```

## 9.4 Controller：app/controller/suggestion.ts（完整内容）

```typescript
import { Controller } from 'egg';

const approvalRule = {
  reason: { type: 'string', required: true, min: 2, max: 500 },
};

export default class SuggestionController extends Controller {

  /** GET /api/v1/suggestions */
  public async list() {
    const { ctx } = this;
    const { pagination } = (ctx.app.config as any).biz;
    const page = Math.max(parseInt(String(ctx.query.page ?? ''), 10) || pagination.defaultPage, 1);
    const pageSize = Math.min(
      parseInt(String(ctx.query.pageSize ?? ''), 10) || pagination.defaultPageSize,
      pagination.maxPageSize,
    );
    const result = await ctx.service.suggestion.list({
      page, pageSize,
      batchNo: ctx.query.batchNo || undefined,
      status: ctx.query.status || undefined,
      riskLevel: ctx.query.riskLevel || undefined,
    });
    ctx.page(result.rows, result.total, page, pageSize);
  }

  /** POST /api/v1/suggestions/:id/approve */
  public async approve() {
    const { ctx } = this;
    ctx.validate(approvalRule);
    const result = await ctx.service.suggestion.submitApproval({
      suggestionId: parseInt(ctx.params.id, 10),
      action: 'approve',
      reason: (ctx.request.body as any).reason,
      approverId: ctx.user!.userId,
      approverName: ctx.user!.username,
    });
    ctx.success(result, '已批准');
  }

  /** POST /api/v1/suggestions/:id/reject */
  public async reject() {
    const { ctx } = this;
    ctx.validate(approvalRule);
    const result = await ctx.service.suggestion.submitApproval({
      suggestionId: parseInt(ctx.params.id, 10),
      action: 'reject',
      reason: (ctx.request.body as any).reason,
      approverId: ctx.user!.userId,
      approverName: ctx.user!.username,
    });
    ctx.success(result, '已驳回');
  }

  /** GET /api/v1/suggestions/:id/approvals */
  public async approvalHistory() {
    this.ctx.success(
      await this.ctx.service.suggestion.approvalHistory(parseInt(this.ctx.params.id, 10)),
    );
  }
}
```

## 9.5 路由

`app/router.ts` 追加：

```typescript
  // ---------------- 建议与审批 ----------------
  router.get('/api/v1/suggestions', auth, controller.suggestion.list);
  router.post('/api/v1/suggestions/:id/approve',
    auth, perm({ permission: 'suggestion:approve' }), controller.suggestion.approve);
  router.post('/api/v1/suggestions/:id/reject',
    auth, perm({ permission: 'suggestion:approve' }), controller.suggestion.reject);
  router.get('/api/v1/suggestions/:id/approvals', auth, controller.suggestion.approvalHistory);
```

同时把 8.6 节 rootcause.ts 里"先建表再放开注释"的那段建议落库代码确认为已放开（建议表本已建好）。

## 9.6 业务闭环完整演示（curl 调用序列）

重启 `npm run dev`（确保 PG、Redis、Python Mock、server 四者都在跑）。**剧本**：工程师 engineer 触发分析 → 智能体产出建议 → 审批员 reviewer 审批。

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

---

# 10. Redis 应用：缓存与幂等

## 10.1 做什么 / 为什么

Redis 在本平台承担三个职责（第三个已在第 7 章实现）：

1. **看板统计缓存**（cache-aside 模式）：看板统计是多表聚合查询，前端 5 秒自动刷新会打爆数据库；
2. **接口幂等**（Idempotency-Key）：前端双击提交 / 网络重试时，防止创建两份一样的资源；
3. refresh token 白名单（已实现）。

## 10.2 看板统计 Service：app/service/stats.ts（完整内容）

**做什么/为什么**：cache-aside（旁路缓存）是最常用的缓存模式——**读**：先查缓存，命中直接返回；未命中查库、回填缓存、设置 TTL。**写**：业务数据变化时**删除**缓存（而非更新），下次读取自然回填最新值。删除而非更新，避免了"并发写缓存与写库顺序不一致"的脏数据问题。

```typescript
import { Service } from 'egg';

const DASHBOARD_KEY = 'ia:stats:dashboard';

export default class StatsService extends Service {

  /** 看板统计（cache-aside 读） */
  public async dashboard() {
    const ttl = (this.app.config as any).biz.cache.dashboardTtlSeconds;

    // 1. 先查缓存
    const cached = await this.app.redis.get(DASHBOARD_KEY);
    if (cached) {
      return { source: 'cache', data: JSON.parse(cached) };
    }

    // 2. 未命中：查库聚合
    const QueryTypes = this.app.Sequelize.QueryTypes;
    const [ yieldRow ] = await this.app.model.query(
      `SELECT ROUND(AVG(yield_rate), 2) AS avg_yield
         FROM wafers
        WHERE status IN ('completed', 'shipped')`,
      { type: QueryTypes.SELECT },
    );
    const [ lineRow ] = await this.app.model.query(
      `SELECT COUNT(DISTINCT production_line) AS lines FROM wafers`,
      { type: QueryTypes.SELECT },
    );

    const data = {
      totalBatches: await this.app.model.Wafer.count(),
      avgYield: yieldRow ? Number(yieldRow.avg_yield) : null,  // PG numeric -> string，需转换
      productionLines: lineRow ? Number(lineRow.lines) : 0,
      openAnomalyEvents: await this.app.model.AnomalyEvent.count({ where: { status: 'open' } }),
      pendingSuggestions: await this.app.model.Suggestion.count({ where: { status: 'pending' } }),
      agentTasks: {
        total: await this.app.model.AgentTask.count(),
        succeeded: await this.app.model.AgentTask.count({ where: { status: 'succeeded' } }),
        failed: await this.app.model.AgentTask.count({ where: { status: 'failed' } }),
      },
      computedAt: new Date().toISOString(),
    };

    // 3. 回填缓存（带 TTL，即使忘了失效最多脏 60 秒）
    await this.app.redis.set(DASHBOARD_KEY, JSON.stringify(data), 'EX', ttl);
    return { source: 'db', data };
  }

  /** 缓存失效：任何业务写操作后调用（删缓存，下次读取回填） */
  public async invalidateDashboard() {
    await this.app.redis.del(DASHBOARD_KEY);
  }
}
```

`app/controller/stats.ts`（完整内容）：

```typescript
import { Controller } from 'egg';

export default class StatsController extends Controller {
  /** GET /api/v1/stats/dashboard */
  public async dashboard() {
    this.ctx.success(await this.ctx.service.stats.dashboard());
  }
}
```

在写路径上挂失效。`app/service/wafer.ts` 的 `create/update/remove` 末尾各追加一行：

```typescript
    // 缓存失效：看板统计数据已过期
    await this.service.stats.invalidateDashboard();
```

> 注意 `remove` 方法里 `await wafer.destroy()` 之后追加；`create` 的 return 前追加；`update` 的 return 前追加。`suggestion.ts` 的 `submitApproval` 事务提交后同样追加一行。

路由追加：

```typescript
  // ---------------- 看板统计 ----------------
  router.get('/api/v1/stats/dashboard', auth, controller.stats.dashboard);
```

**验证**：

```bash
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

## 10.3 接口幂等中间件：app/middleware/idempotency.ts（完整内容）

**做什么/为什么**：网络抖动时前端/网关会重试 POST。约定：客户端为每个"创建类"请求生成唯一 `Idempotency-Key` 头。服务端用 Redis `SETNX` 抢占：

- 抢占成功 → 正常执行，成功后把响应缓存 24 小时；
- 抢占失败且已有缓存响应 → 直接重放缓存的响应（头 `Idempotent-Replay: true`）；
- 抢占失败且占位值还在处理中 → 409 提示稍后重试。

```typescript
import { Context, Next } from 'egg';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';

/**
 * 接口幂等中间件（对 POST 生效，未携带 Idempotency-Key 的请求直接放行）
 */
export default () => {
  return async function idempotency(ctx: Context, next: Next) {
    if (ctx.method !== 'POST') return await next();

    const idemKey = ctx.get('Idempotency-Key');
    if (!idemKey) return await next(); // 不强制：未带 key 的请求不享受幂等保护

    const ttl = (ctx.app.config as any).biz.idempotency.ttlSeconds;
    // key 绑定"路径 + 参数"，避免同一 key 被用在别的接口上
    const redisKey = `ia:idem:${ctx.path}:${idemKey}`;

    const locked = await ctx.app.redis.setnx(redisKey, '1');
    if (locked === 1) {
      // 首次见到这个 key：占位并设置兜底 TTL（防止请求崩溃后占位永不释放）
      await ctx.app.redis.expire(redisKey, ttl);
      await next();
      // 业务成功：把响应体缓存下来，供后续重放
      if (ctx.status < 400 && ctx.body) {
        await ctx.app.redis.set(redisKey, JSON.stringify(ctx.body), 'EX', ttl);
      }
      return;
    }

    // 已存在：要么是处理中（值还是 '1'），要么已有缓存响应可重放
    const cached = await ctx.app.redis.get(redisKey);
    if (cached && cached !== '1') {
      ctx.set('Idempotent-Replay', 'true');
      ctx.status = 200;
      ctx.body = JSON.parse(cached);
      return;
    }
    throw new BizError(ErrorCode.DUPLICATE_REQUEST, '重复请求正在处理中，请稍后查询结果或更换 Idempotency-Key', 409);
  };
};
```

放开 `config.default.ts` 中 `config.middleware` 的 `'idempotency'`，最终全局中间件顺序：

```typescript
config.middleware = ['errorHandler', 'requestLog', 'auditLog', 'idempotency', 'metrics'];
```

（`metrics` 第 14 章创建，届时再放开。）

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

---

# 11. 定时任务 schedule

## 11.1 做什么 / 为什么

Egg 的 `app/schedule/` 目录专放定时任务（框架自动加载、自动分配 worker，**不需要自己写 setInterval**）。平台需要两个后台任务：

1. **数据健康检查**：传感器数据若断流超过阈值，说明 FDC/采集链路故障——"没有数据"本身就是最严重的异常；
2. **良率日报**：每天早上 7 点统计前一日各产线良率，写入日报表。

## 11.2 数据模型（完整内容）

`app/model/sensor_reading.ts`：

```typescript
import { Application } from 'egg';

/** 传感器读数表 sensor_readings（演示用简化模型；生产用 TimescaleDB） */
export default (app: Application) => {
  const { INTEGER, STRING, DECIMAL, DATE } = app.Sequelize;

  const SensorReading = app.model.define('sensor_reading', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    equipmentId: { type: STRING(64), allowNull: false, comment: '设备编号，如 ETCH-03' },
    metric: { type: STRING(64), allowNull: false, comment: '指标名，如 rf_power' },
    value: { type: DECIMAL(10, 3), allowNull: false },
    recordedAt: { type: DATE, allowNull: false, comment: '采集时间' },
  }, {
    tableName: 'sensor_readings',
    timestamps: false,
    indexes: [
      { fields: [ 'recordedAt' ] },
    ],
  });

  return SensorReading;
};
```

`app/model/daily_yield_report.ts`：

```typescript
import { Application } from 'egg';

/** 良率日报表 daily_yield_reports */
export default (app: Application) => {
  const { INTEGER, DATE, DECIMAL, JSONB } = app.Sequelize;

  const DailyYieldReport = app.model.define('daily_yield_report', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    reportDate: { type: DATEONLY, allowNull: false, unique: true, comment: '报告日期（YYYY-MM-DD）' },
    avgYield: { type: DECIMAL(5, 2), allowNull: true, comment: '当日完工批次平均良率' },
    totalBatches: { type: INTEGER, allowNull: false, defaultValue: 0 },
    detail: { type: JSONB, allowNull: true, comment: '分产线明细' },
  }, {
    tableName: 'daily_yield_reports',
    timestamps: true,
  });

  return DailyYieldReport;
};
```

> `DATEONLY` 类型：纯日期无时间，正好用于"报告日期"。

## 11.3 数据健康检查任务：app/schedule/data-health-check.ts（完整内容）

```typescript
import { Subscription } from 'egg';

/**
 * 数据健康检查：传感器数据断流告警
 * - type: 'worker' 表示整个集群只由一个 worker 执行（避免多 worker 重复告警）
 * - interval 支持 '5m' / '10s' 等人类可读写法
 */
export default class DataHealthCheck extends Subscription {
  static get schedule() {
    return {
      interval: '5m',
      type: 'worker',
      // immediate: true,  // 启动立即执行一次（调试用）
    };
  }

  async subscribe() {
    const { ctx, app } = this;
    const QueryTypes = app.Sequelize.QueryTypes;

    const [ row ] = await app.model.query(
      `SELECT MAX(recorded_at) AS latest FROM sensor_readings`,
      { type: QueryTypes.SELECT },
    );

    const latestMs = row && row.latest ? new Date(row.latest).getTime() : 0;
    const gapMinutes = Math.floor((Date.now() - latestMs) / 60000);

    if (gapMinutes > 30) {
      // 结构化告警日志（生产可扩展为 webhook / 邮件 / 短信）
      ctx.logger.warn(JSON.stringify({
        kind: 'alert',
        level: 'warn',
        type: 'sensor_data_stall',
        msg: '传感器数据断流，请检查 FDC 采集链路',
        gapMinutes,
        lastRecordedAt: row ? row.latest : null,
        ts: new Date().toISOString(),
      }));
    } else {
      ctx.logger.info('[health-check] 传感器数据正常，最近数据 %s 分钟前', gapMinutes);
    }
  }
}
```

## 11.4 良率日报任务：app/schedule/yield-daily-report.ts（完整内容）

```typescript
import { Subscription } from 'egg';

/**
 * 良率日报：每天 07:00 统计前一日完工批次良率
 * egg 的 cron 是 6 段式：秒 分 时 日 月 周
 */
export default class YieldDailyReport extends Subscription {
  static get schedule() {
    return {
      cron: '0 0 7 * * *',
      type: 'worker',
    };
  }

  async subscribe() {
    const { ctx, app } = this;
    const QueryTypes = app.Sequelize.QueryTypes;

    // 昨日 00:00 ~ 今日 00:00（数据库时区）
    const rows = await app.model.query(
      `SELECT production_line,
              ROUND(AVG(yield_rate), 2) AS avg_yield,
              COUNT(*) AS batches
         FROM wafers
        WHERE status IN ('completed', 'shipped')
          AND produced_at >= CURRENT_DATE - INTERVAL '1 day'
          AND produced_at <  CURRENT_DATE
        GROUP BY production_line
        ORDER BY production_line`,
      { type: QueryTypes.SELECT },
    );

    const totalBatches = rows.reduce((s: number, r: any) => s + Number(r.batches), 0);
    const avgYield = totalBatches > 0
      ? rows.reduce((s: number, r: any) => s + Number(r.avg_yield) * Number(r.batches), 0) / totalBatches
      : null;

    const yesterday = new Date(Date.now() - 24 * 3600 * 1000);
    const reportDate = yesterday.toISOString().slice(0, 10);

    // upsert：同一天重复执行会覆盖而非报错
    await app.model.DailyYieldReport.upsert({
      reportDate,
      avgYield: avgYield === null ? null : Math.round(avgYield * 100) / 100,
      totalBatches,
      detail: rows,
    } as any);

    ctx.logger.info('[daily-report] 良率日报已生成：%s，平均良率 %s，批次 %s',
      reportDate, avgYield === null ? '-' : avgYield.toFixed(2), totalBatches);
  }
}
```

## 11.5 种子数据与验证

`scripts/init-db.ts` 追加传感器数据：

```typescript
  // ---------- 种子数据：传感器读数 ----------
  const SensorReading = (await import('../app/model/sensor_reading')).default(miniApp);
  const minutesAgo = (m: number) => new Date(now - m * 60 * 1000);
  const readings: any[] = [];
  for (let i = 1; i <= 10; i++) {
    readings.push({
      equipmentId: 'ETCH-03', metric: 'rf_power',
      value: 1500 + Math.sin(i) * 20, recordedAt: minutesAgo(i * 5),
    });
  }
  await SensorReading.bulkCreate(readings);
  console.log('[seed] 传感器读数 10 条写入完成');
```

**验证（把检查周期临时调快）**：把 `data-health-check.ts` 的 `interval` 临时改为 `'10s'` 并放开 `immediate: true`，重启 `npm run dev`。

- 种子数据最新读数是 5 分钟前 → 预期日志每 10 秒输出：`[health-check] 传感器数据正常，最近数据 5 分钟前`；
- 手动清空读数模拟断流：

```bash
docker exec -it ia-pg psql -U industry -d industryagent -c "TRUNCATE sensor_readings;"
```

- 预期 10 秒内出现告警日志：

```text
{"kind":"alert","level":"warn","type":"sensor_data_stall","msg":"传感器数据断流，请检查 FDC 采集链路","gapMinutes":xxx,...}
```

**验证良率日报**：临时把 cron 改成 `'*/30 * * * * *'`（每 30 秒）再重启，然后查表：

```bash
docker exec -it ia-pg psql -U industry -d industryagent \
  -c "SELECT report_date, avg_yield, total_batches, detail FROM daily_yield_reports;"
```

预期输出（种子数据的 produced_at 在 24 小时内的批次会进入统计）：

```text
 report_date | avg_yield | total_batches |                      detail
-------------+-----------+---------------+--------------------------------------------------
 2026-10-14  |     92.10 |             2 | [{"production_line":"LINE-1","avg_yield":"91.20",...}]
```

验证完成后把两个任务的时间改回 `5m` 与 `0 0 7 * * *`。

---

# 12. 单元测试与集成测试

## 12.1 做什么 / 为什么

手册的工程化要求包含 CI/CD 自动测试。测试分两层：

- **单元测试**：只测 service 的业务逻辑，用 sinon 把数据库模型 mock 掉——快、稳定、不依赖环境；
- **集成测试**：真实起一个 Egg 应用连**测试数据库**，走完整 HTTP 链路（egg-mock 内置 supertest）。

测试框架 egg-bin 已内置 mocha；我们补充 chai（断言）与 sinon（mock）。

## 12.2 测试库准备

```bash
docker exec -it ia-pg psql -U industry -d postgres \
  -c "CREATE DATABASE industryagent_test OWNER industry;"
# 预期输出：CREATE DATABASE
```

## 12.3 单元测试：test/service/wafer.test.ts（完整内容）

**做什么/为什么**：mock 掉 `app.model.Wafer.findAndCountAll`，验证 list 的 where 拼装与返回结构——完全不碰数据库，毫秒级完成。

```typescript
import { app } from 'egg-mock/bootstrap';
import { expect } from 'chai';
import * as sinon from 'sinon';

describe('service/wafer -> list()', () => {

  afterEach(() => {
    // 恢复所有 stub，避免影响其它用例
    sinon.restore();
  });

  it('无过滤条件时返回全部分页数据', async () => {
    const fakeRows = [
      { id: 1, batchNo: 'B20261015001', yieldRate: 91.20 },
      { id: 2, batchNo: 'B20261015002', yieldRate: 96.80 },
    ];
    const stub = sinon.stub(app.model.Wafer, 'findAndCountAll').resolves({
      rows: fakeRows as any[],
      count: 2,
    } as any);

    const ctx = app.mockContext();
    const result = await ctx.service.wafer.list({ page: 1, pageSize: 10 });

    expect(result.total).to.equal(2);
    expect(result.rows).to.have.lengthOf(2);
    expect(result.rows[0].batchNo).to.equal('B20261015001');
    expect(stub.calledOnce).to.be.true;

    // 校验传给模型的分页参数
    const arg = stub.firstCall.args[0];
    expect(arg.offset).to.equal(0);
    expect(arg.limit).to.equal(10);
  });

  it('带批次号时拼接模糊查询条件', async () => {
    const stub = sinon.stub(app.model.Wafer, 'findAndCountAll').resolves({
      rows: [], count: 0,
    } as any);

    const ctx = app.mockContext();
    await ctx.service.wafer.list({ page: 2, pageSize: 5, batchNo: '15001' });

    const arg = stub.firstCall.args[0];
    // page=2, pageSize=5 -> offset 5
    expect(arg.offset).to.equal(5);
    // 模糊查询应包含 %15001%
    expect(JSON.stringify(arg.where.batchNo)).to.contain('%15001%');
  });
});
```

再补一个错误路径单测 `test/service/wafer.detail.test.ts`：

```typescript
import { app } from 'egg-mock/bootstrap';
import { expect } from 'chai';
import * as sinon from 'sinon';

describe('service/wafer -> detail()', () => {
  afterEach(() => sinon.restore());

  it('批次不存在时抛出 NOT_FOUND 业务错误', async () => {
    sinon.stub(app.model.Wafer, 'findByPk').resolves(null);

    const ctx = app.mockContext();
    let caught: any = null;
    try {
      await ctx.service.wafer.detail(999);
    } catch (e) {
      caught = e;
    }
    expect(caught).to.not.be.null;
    expect(caught.code).to.equal(10002);   // ErrorCode.NOT_FOUND
    expect(caught.httpStatus).to.equal(404);
  });
});
```

## 12.4 集成测试：test/app/controller/auth.test.ts（完整内容）

**做什么/为什么**：`egg-mock/bootstrap` 提供的 `app` 会真实加载框架与插件、连接测试库（`config.test.ts` 已指向 industryagent_test）。`app.httpRequest()` 是内置的 supertest 封装。

```typescript
import { app } from 'egg-mock/bootstrap';
import { expect } from 'chai';
import * as bcrypt from 'bcryptjs';

describe('auth 认证接口（集成）', () => {

  before(async () => {
    // 测试库建表 + 准备一个已知密码的用户
    await app.model.sync({ force: true });
    const User = app.model.User;
    const Role = app.model.Role;
    const UserRole = app.model.UserRole;
    await Role.create({ code: 'admin', name: '管理员' });
    await User.create({
      username: 'it_user',
      passwordHash: bcrypt.hashSync('Passw0rd', 10),
      realName: '集成测试用户',
      status: 'active',
    });
    await UserRole.create({ userId: 1, roleId: 1 });
  });

  after(async () => {
    await app.model.drop();
  });

  it('POST /api/v1/auth/login 正确密码返回双 token', async () => {
    const res = await app.httpRequest()
      .post('/api/v1/auth/login')
      .send({ username: 'it_user', password: 'Passw0rd' })
      .expect(200);
    expect(res.body.code).to.equal(0);
    expect(res.body.data.accessToken).to.be.a('string').and.not.empty;
    expect(res.body.data.refreshToken).to.be.a('string').and.not.empty;
    expect(res.body.data.user.roles).to.deep.equal([ 'admin' ]);
  });

  it('POST /api/v1/auth/login 错误密码返回 LOGIN_FAILED', async () => {
    const res = await app.httpRequest()
      .post('/api/v1/auth/login')
      .send({ username: 'it_user', password: 'wrong-password' })
      .expect(400);
    expect(res.body.code).to.equal(20004); // ErrorCode.LOGIN_FAILED
  });

  it('GET /api/v1/wafers 未带 token 返回 401', async () => {
    const res = await app.httpRequest().get('/api/v1/wafers').expect(401);
    expect(res.body.code).to.equal(20001); // ErrorCode.UNAUTHORIZED
  });

  it('登录后带 token 可访问受保护接口', async () => {
    const login = await app.httpRequest()
      .post('/api/v1/auth/login')
      .send({ username: 'it_user', password: 'Passw0rd' });
    const token = login.body.data.accessToken;

    const res = await app.httpRequest()
      .get('/api/v1/wafers')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);
    expect(res.body.code).to.equal(0);
  });
});
```

## 12.5 运行测试与覆盖率

```bash
npm test
```

预期输出（节选）：

```text
  service/wafer -> list()
    ✓ 无过滤条件时返回全部分页数据
    ✓ 带批次号时拼接模糊查询条件
  service/wafer -> detail()
    ✓ 批次不存在时抛出 NOT_FOUND 业务错误
  auth 认证接口（集成）
    ✓ POST /api/v1/auth/login 正确密码返回双 token
    ✓ POST /api/v1/auth/login 错误密码返回 LOGIN_FAILED
    ✓ GET /api/v1/wafers 未带 token 返回 401
    ✓ 登录后带 token 可访问受保护接口

  7 passing (3s)
```

覆盖率报告：

```bash
npx egg-bin cov
```

预期输出（节选，数字因代码而异）：

```text
File                         | % Stmts | % Branch | % Funcs | % Lines |
-----------------------------|---------|----------|---------|---------+
 app/service/wafer.ts        |   85.71 |    66.67 |     100 |   85.71 |
 app/controller/auth.ts      |   78.95 |       50 |     100 |   78.95 |
-----------------------------|---------|----------|---------|---------+
```

> **注意**：集成测试需要 PG 与 Redis 在运行（docker 容器别停）。schedule 任务在测试进程里也会加载，但 5 分钟间隔的它们不会干扰短时测试。

---

# 13. Egg.js MCP Server：把后端能力暴露给智能体

## 13.1 做什么 / 为什么

**MCP（Model Context Protocol）** 是 Anthropic 开源的智能体工具调用协议，现已成为事实标准。它解决的问题是：让 LLM/智能体以**标准化、可发现**的方式调用外部工具（查数据库、触发分析、调用 API）。

架构定位（见 1.2 节架构图）：industryagent-server 除了给前端供 REST API，还作为 **MCP Server** 向智能体暴露三个工具：

| 工具名 | 能力 |
|---|---|
| `query_wafer_batches` | 查询晶圆批次（批次号/状态过滤） |
| `query_anomaly_events` | 查询异常事件（严重度/状态过滤） |
| `trigger_rootcause_analysis` | 触发根因分析（调用 Python 智能体并返回 Top-3 根因） |

这样工程师在 Cursor / Claude Desktop / 自研智能体里直接问："帮我查下 LINE-1 最近的异常事件，并对 B20261015001 做根因分析"——智能体自动调用这些工具完成闭环。

**集成方式选择**：MCP 有 stdio（标准输入输出，客户端拉起服务进程）与 Streamable HTTP 两种传输。stdio 最通用（Cursor、Claude Desktop、Claude Code 全都支持），且与本教程"数据库能力复用"的技巧完美配合。我们用官方 `@modelcontextprotocol/sdk` 实现 stdio Server。

## 13.2 安装依赖

依赖已在 package.json 中（`@modelcontextprotocol/sdk`、`zod`）。确认已 `npm install`。

## 13.3 MCP Server：mcp/server.ts（完整内容）

**做什么/为什么**：

- 独立进程入口（不跑在 Egg worker 里），但**复用** `app/model/` 下的模型定义（用第 5.8 节的"迷你 app"技巧，一份模型两处使用，避免双维护）；
- 每个工具 = `registerTool(名称, { 描述 + zod schema }, handler)`。**schema 是给 LLM 看的**——描述写得越清楚，智能体调用得越准；
- **所有日志必须走 `console.error`（stderr）**：stdio 通道承载协议报文，一行 `console.log` 就会污染协议导致客户端断连（这是最高频的 MCP 踩坑，FAQ 第 13 条）。

```typescript
/**
 * industryagent MCP Server（stdio 传输）
 * 暴露工具：查询晶圆批次 / 查询异常事件 / 触发根因分析
 */
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { z } from 'zod';
import { Sequelize, DataTypes, Op } from 'sequelize';
import * as dotenv from 'dotenv';
import * as path from 'path';

dotenv.config({ path: path.join(__dirname, '..', '.env') });

async function main() {
  // ---------- 数据库连接（独立于 Egg 进程） ----------
  const sequelize = new Sequelize(
    process.env.IA_DB_NAME || 'industryagent',
    process.env.IA_DB_USER || 'industry',
    process.env.IA_DB_PASSWORD || 'industry123',
    {
      host: process.env.IA_DB_HOST || '127.0.0.1',
      port: Number(process.env.IA_DB_PORT || 5432),
      dialect: 'postgres',
      logging: false,
    },
  );

  // 复用 egg 项目的模型定义（迷你 app 技巧，与 scripts/init-db.ts 一致）
  const miniApp: any = { Sequelize: DataTypes, model: sequelize };
  const Wafer = (await import('../app/model/wafer')).default(miniApp);
  const AnomalyEvent = (await import('../app/model/anomaly_event')).default(miniApp);

  const AGENT_BASE_URL = process.env.IA_AGENT_BASE_URL || 'http://127.0.0.1:8000';

  // ---------- MCP Server 实例 ----------
  const server = new McpServer({ name: 'industryagent-server', version: '1.0.0' });

  // ---------- 工具 1：查询晶圆批次 ----------
  server.registerTool('query_wafer_batches', {
    title: '查询晶圆批次',
    description:
      '根据批次号（模糊匹配）、状态、产线查询晶圆批次主数据。' +
      '返回字段：batchNo(批次号)、productModel(产品型号)、processNode(工艺节点)、' +
      'productionLine(产线)、quantity(数量)、defectCount(缺陷数)、yieldRate(良率%)、status(状态)。' +
      '状态枚举：producing(在制)/completed(完工)/shipped(出货)/quarantined(隔离)。',
    inputSchema: {
      batchNo: z.string().optional().describe('批次号，模糊匹配，如 B20261015'),
      status: z.enum([ 'producing', 'completed', 'shipped', 'quarantined' ])
        .optional().describe('批次状态过滤'),
      limit: z.number().int().min(1).max(50).default(10).describe('返回条数，默认 10'),
    },
  }, async ({ batchNo, status, limit }) => {
    const where: any = {};
    if (batchNo) where.batchNo = { [Op.like]: `%${batchNo}%` };
    if (status) where.status = status;
    const rows = await Wafer.findAll({
      where, limit, order: [[ 'id', 'DESC' ]], raw: true,
    });
    return {
      content: [ { type: 'text', text: JSON.stringify(rows, null, 2) } ],
    };
  });

  // ---------- 工具 2：查询异常事件 ----------
  server.registerTool('query_anomaly_events', {
    title: '查询异常事件',
    description:
      '查询产线异常事件（FDC 报警）。' +
      '返回字段：eventCode(事件编号)、batchNo(关联批次)、equipment(设备)、' +
      'processStage(工序)、severity(严重度)、title(标题)、status(状态)。' +
      '严重度枚举：critical/major/minor；状态枚举：open(待处理)/processing(处理中)/closed(已关闭)。',
    inputSchema: {
      severity: z.enum([ 'critical', 'major', 'minor' ]).optional().describe('严重度过滤'),
      status: z.enum([ 'open', 'processing', 'closed' ]).optional().describe('状态过滤'),
      equipment: z.string().optional().describe('设备编号，模糊匹配，如 ETCH'),
      limit: z.number().int().min(1).max(50).default(10).describe('返回条数，默认 10'),
    },
  }, async ({ severity, status, equipment, limit }) => {
    const where: any = {};
    if (severity) where.severity = severity;
    if (status) where.status = status;
    if (equipment) where.equipment = { [Op.like]: `%${equipment}%` };
    const rows = await AnomalyEvent.findAll({
      where, limit, order: [[ 'occurredAt', 'DESC' ]], raw: true,
    });
    return {
      content: [ { type: 'text', text: JSON.stringify(rows, null, 2) } ],
    };
  });

  // ---------- 工具 3：触发根因分析 ----------
  server.registerTool('trigger_rootcause_analysis', {
    title: '触发根因分析',
    description:
      '对指定晶圆批次触发良率根因分析：先校验批次存在，再调用 Python 智能体服务，' +
      '返回 Top-3 根因嫌疑（factor/置信度/描述）与整改建议。分析耗时约 2~5 秒。',
    inputSchema: {
      batchNo: z.string().min(6).max(32).describe('晶圆批次号，如 B20261015001'),
    },
  }, async ({ batchNo }) => {
    // 校验批次存在
    const wafer = await Wafer.findOne({ where: { batchNo }, raw: true });
    if (!wafer) {
      return {
        content: [ { type: 'text', text: `批次 ${batchNo} 不存在，请用 query_wafer_batches 确认批次号` } ],
        isError: true,
      };
    }
    try {
      const resp = await fetch(`${AGENT_BASE_URL}/agent/analyze`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ batch_no: batchNo, yield_rate: Number(wafer.yieldRate), top_k: 3 }),
        signal: AbortSignal.timeout(30000),
      });
      if (!resp.ok) {
        return { content: [ { type: 'text', text: `智能体服务返回 HTTP ${resp.status}` } ], isError: true };
      }
      const data = await resp.json();
      return { content: [ { type: 'text', text: JSON.stringify(data, null, 2) } ] };
    } catch (e: any) {
      return { content: [ { type: 'text', text: `智能体服务调用失败：${e.message}` } ], isError: true };
    }
  });

  // ---------- 启动：stdio 传输 ----------
  const transport = new StdioServerTransport();
  await server.connect(transport);
  // 关键：日志只能写 stderr，写 stdout 会破坏 MCP 协议！
  console.error('[mcp] industryagent MCP Server 已启动（stdio），工具数：3');
}

main().catch(e => {
  console.error('[mcp] 启动失败：', e);
  process.exit(1);
});
```

## 13.4 用 MCP 客户端验证：scripts/mcp-client.ts（完整内容）

**做什么/为什么**：`StdioClientTransport` 会把 server 当子进程拉起，通过 stdin/stdout 对话——与 Cursor 等客户端的行为完全一致。这个脚本是我们的"验收测试"。

```typescript
/**
 * MCP 客户端验证脚本：
 * 1. 列出 server 暴露的全部工具
 * 2. 调用 query_wafer_batches 查询批次
 * 3. 调用 trigger_rootcause_analysis 触发一次分析
 */
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';

async function main() {
  const transport = new StdioClientTransport({
    // 用当前 node 进程以 ts-node 方式拉起 mcp/server.ts
    command: process.execPath,
    args: [ '-r', 'ts-node/register', 'mcp/server.ts' ],
    cwd: process.cwd(),
  });

  const client = new Client({ name: 'ia-mcp-test-client', version: '1.0.0' });
  await client.connect(transport);
  console.log('已连接 MCP Server ✔');

  // 1. 列出工具
  const { tools } = await client.listTools();
  console.log('\n=== 工具列表 ===');
  tools.forEach(t => console.log(`- ${t.name}: ${t.description?.slice(0, 50)}...`));

  // 2. 查询晶圆批次
  console.log('\n=== 调用 query_wafer_batches ===');
  const waferResult = await client.callTool({
    name: 'query_wafer_batches',
    arguments: { batchNo: 'B20261015001' },
  });
  console.log((waferResult.content as any[])[0].text);

  // 3. 查询异常事件
  console.log('\n=== 调用 query_anomaly_events（critical） ===');
  const eventResult = await client.callTool({
    name: 'query_anomaly_events',
    arguments: { severity: 'critical' },
  });
  console.log((eventResult.content as any[])[0].text);

  // 4. 触发根因分析（需要 Python Mock 在跑）
  console.log('\n=== 调用 trigger_rootcause_analysis（约 2 秒） ===');
  const analyzeResult = await client.callTool({
    name: 'trigger_rootcause_analysis',
    arguments: { batchNo: 'B20261015001' },
  });
  console.log((analyzeResult.content as any[])[0].text);

  await client.close();
  console.log('\n验证完成 ✔');
}

main().catch(e => {
  console.error('验证失败：', e);
  process.exit(1);
});
```

## 13.5 验证

前提：PG、Redis 可连（MCP 直连 PG，不依赖 Redis/Python；分析工具需要 Python Mock 在跑）。

```bash
npm run mcp:test
```

预期输出（节选）：

```text
已连接 MCP Server ✔

=== 工具列表 ===
- query_wafer_batches: 根据批次号（模糊匹配）、状态、产线查询晶圆批次主数据。...
- query_anomaly_events: 查询产线异常事件（FDC 报警）。...
- trigger_rootcause_analysis: 对指定晶圆批次触发良率根因分析：先校验批次存在，...

=== 调用 query_wafer_batches ===
[
  {
    "id": 1,
    "batchNo": "B20261015001",
    "productModel": "MCU-XS32",
    "yieldRate": "91.20",
    "status": "completed",
    ...
  }
]

=== 调用 query_anomaly_events（critical） ===
[
  {
    "eventCode": "AE-20261015-001",
    "equipment": "ETCH-03",
    "severity": "critical",
    ...
  }
]

=== 调用 trigger_rootcause_analysis（约 2 秒） ===
{
  "batch_no": "B20261015001",
  "summary": "批次 B20261015001 良率异常，疑似刻蚀工序参数漂移...",
  "root_causes": [ { "rank": 1, "factor": "etch_power_drift", "confidence": 0.87, ... } ],
  "suggestions": [ ... ]
}

验证完成 ✔
```

## 13.6 接入智能体客户端（Cursor / Claude Desktop / Claude Code）

Cursor：项目根 `.cursor/mcp.json`：

```json
{
  "mcpServers": {
    "industryagent": {
      "command": "node",
      "args": [ "-r", "ts-node/register", "mcp/server.ts" ],
      "cwd": "/绝对路径/industryagent-server"
    }
  }
}
```

Claude Desktop（配置文件 `claude_desktop_config.json`）：

```json
{
  "mcpServers": {
    "industryagent": {
      "command": "node",
      "args": [ "-r", "ts-node/register", "mcp/server.ts" ],
      "cwd": "/绝对路径/industryagent-server"
    }
  }
}
```

> Windows 用户把 `cwd` 写成 `D:\\code\\industryagent\\industryagent-server`（JSON 双反斜杠）。

配置后重启客户端，即可在对话中直接提问验证：

> "帮我查一下 LINE-1 有哪些未关闭的严重异常，然后对涉及的批次做根因分析。"

智能体会自动调用 `query_anomaly_events` → `trigger_rootcause_analysis` 完成任务。**这就是手册架构图中"Egg.js 业务后端 → MCP Server → 智能体工具链"的实机落地。**

---

# 14. 日志与可观测基础

## 14.1 做什么 / 为什么

手册工程化要求：可观测（日志、指标、追踪）。日志部分第 6 章已实现结构化 JSON 单行 + traceId（自带的 `ctx.logger` 底层按天分文件，无需 pino 即满足入门要求；若追求极致性能可自行替换 egg-logger 为 pino，思路一致）。本章补齐另一半：**Prometheus 指标**（QPS / 延迟分布 / 状态码分布），为后续接 Grafana 打地基。

## 14.2 指标中间件：app/middleware/metrics.ts（完整内容）

```typescript
import { Context, Next } from 'egg';
import client from 'prom-client';

// Histogram：延迟分布（P50/P90/P95/P99 由 Prometheus 服务端计算）
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP 请求耗时（秒）',
  labelNames: [ 'method', 'route', 'status' ] as const,
  buckets: [ 0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5 ],
});

// Counter：请求总量（配合 rate() 求 QPS）
const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'HTTP 请求总数',
  labelNames: [ 'method', 'route', 'status' ] as const,
});

/**
 * Prometheus 指标中间件：
 * - route 标签用路由模板（/api/v1/wafers/:id）而非真实路径，避免标签基数爆炸
 */
export default () => {
  return async function metrics(ctx: Context, next: Next) {
    // /metrics 自身不统计，避免自激励
    if (ctx.path === '/metrics' || ctx.path === '/healthz') return await next();

    const start = Date.now();
    await next();

    const route = (ctx as any).routerPath || ctx.path;
    const labels = { method: ctx.method, route, status: String(ctx.status) };
    httpRequestsTotal.inc(labels);
    httpRequestDuration.observe(labels, (Date.now() - start) / 1000);
  };
};
```

`app/controller/metrics.ts`（完整内容）：

```typescript
import { Controller } from 'egg';
import client from 'prom-client';

export default class MetricsController extends Controller {
  /** GET /metrics：Prometheus 抓取端点 */
  public async index() {
    this.ctx.set('Content-Type', client.register.contentType);
    this.ctx.body = await client.register.metrics();
  }
}
```

健康检查也顺手补上——`app/controller/health.ts`（完整内容）：

```typescript
import { Controller } from 'egg';

export default class HealthController extends Controller {
  /** GET /healthz：探活端点（检查 DB 与 Redis 连通性） */
  public async check() {
    const checks: Record<string, string> = {};

    try {
      await this.app.model.authenticate();
      checks.database = 'ok';
    } catch {
      checks.database = 'error';
    }
    try {
      await this.app.redis.ping();
      checks.redis = 'ok';
    } catch {
      checks.redis = 'error';
    }

    const healthy = Object.values(checks).every(v => v === 'ok');
    this.ctx.status = healthy ? 200 : 503;
    this.ctx.body = {
      status: healthy ? 'ok' : 'degraded',
      checks,
      ts: new Date().toISOString(),
    };
  }
}
```

路由追加（`app/router.ts`）：

```typescript
  // ---------------- 运维端点 ----------------
  router.get('/healthz', controller.health.check);
  router.get('/metrics', controller.metrics.index);
```

放开 `config.default.ts` 中 `config.middleware` 的 `'metrics'`（至此五个全局中间件全部就位）。

## 14.3 日志分级速览与验证

Egg 自带 logger 分级（`DEBUG < INFO < WARN < ERROR`），按环境在 `config.logger.level` 控制（local=DEBUG / prod=INFO，已在第 4 章配置）。文件输出位置：`logs/industryagent-server/`：

| 文件 | 内容 |
|---|---|
| `${app}.log` | 应用日志（含我们第 6 章的结构化请求日志） |
| `common-error.log` | 框架捕获的错误堆栈 |
| `audit.log` | 审计专用（第 6 章配置） |
| `egg-web.log` / `egg-schedule.log` | 框架自身日志 |

**验证指标端点**：

```bash
# 先制造一些流量
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

**验证健康检查**（临时停掉 Redis 容器看降级效果）：

```bash
curl -s http://127.0.0.1:7001/healthz
# 正常预期：{"status":"ok","checks":{"database":"ok","redis":"ok"},"ts":"..."}

docker stop ia-redis
curl -si http://127.0.0.1:7001/healthz | head -n 1
# 预期：HTTP/1.1 503 Service Unavailable
docker start ia-redis   # 恢复
```

> 接入 Prometheus 只需在其 `prometheus.yml` 中加抓取任务 `targets: ['<server-host>:7001']`，Grafana 即可出 QPS/P95 面板——属于运维教程范畴，这里打牢数据地基即可。

---

# 15. 构建与部署

## 15.1 做什么 / 为什么

`npm run dev` 是 ts-node 直跑，性能与安全都不适合生产。生产路径：**tsc 编译产出 dist → 多阶段 Docker 镜像 → docker-compose 编排**（PG + Redis + server 一条命令起全套）。

## 15.2 TS 编译验证

```bash
npm run build
ls dist
# 预期：dist/ 下出现 app/ config/ typings/ mcp/ 目录与 .js / .d.ts 产物
```

> 若报 TS 类型错误，逐个修复后再继续（大多数是拼写或导入路径问题）。

## 15.3 Dockerfile（完整内容，多阶段构建）

```dockerfile
# ========== 阶段 1：构建（装全部依赖 + 编译 TS） ==========
FROM node:20-alpine AS builder
WORKDIR /app

# 先只拷贝依赖清单，充分利用 Docker 层缓存
COPY package.json package-lock.json ./
RUN npm ci --registry=https://registry.npmmirror.com

# 再拷贝源码与编译配置
COPY tsconfig.json tsconfig.build.json ./
COPY app ./app
COPY config ./config
COPY typings ./typings
COPY mcp ./mcp

RUN npm run build

# ========== 阶段 2：运行（只装生产依赖 + 编译产物） ==========
FROM node:20-alpine
ENV NODE_ENV=production
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev --registry=https://registry.npmmirror.com && npm cache clean --force

COPY --from=builder /app/dist ./dist

EXPOSE 7001

# 容器级健康检查：/healthz 返回非 200 则判定不健康
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD wget -qO- http://127.0.0.1:7001/healthz || exit 1

# egg-scripts 前台运行（容器内不能 daemon）；单 worker 足够演示，压测场景改 --workers
CMD ["egg-scripts", "start", "dist", "--daemon=false", "--workers=1", "--port=7001"]
```

`.dockerignore`（完整内容）：

```text
node_modules
dist
logs
run
.env
test
scripts
agent-mock
.git
*.md
```

> **注意**：`.env` 被 ignore，容器内的配置全部走 docker-compose 的 `environment` 注入——这正是第 4 章"代码与环境隔离"设计的回报。

## 15.4 docker-compose.yml（完整内容）

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:16-alpine
    container_name: ia-compose-pg
    environment:
      POSTGRES_USER: industry
      POSTGRES_PASSWORD: industry123
      POSTGRES_DB: industryagent
    ports:
      - "15432:5432"          # 宿主机用 15432，避免与本地已有的 5432 冲突
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
      - "16379:6379"
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
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      NODE_ENV: production
      EGG_SERVER_ENV: prod
      IA_KEYS: prod-keys-change-me-please-32chars
      IA_JWT_SECRET: prod-jwt-secret-change-me-please
      IA_DB_HOST: postgres
      IA_DB_PORT: "5432"
      IA_DB_NAME: industryagent
      IA_DB_USER: industry
      IA_DB_PASSWORD: industry123
      IA_REDIS_HOST: redis
      IA_REDIS_PORT: "6379"
      IA_AGENT_BASE_URL: http://host.docker.internal:8000   # Python Mock 跑在宿主机
    ports:
      - "7001:7001"

volumes:
  pgdata:
```

> **Windows / macOS 说明**：容器内访问宿主机服务用 `host.docker.internal`；Linux 上 docker compose 需在 server 服务加 `extra_hosts: ["host.docker.internal:host-gateway"]`。

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

初始化表结构（用第 5 章脚本指向 compose 的 PG）：

```bash
IA_DB_PORT=15432 npm run seed
# 预期：[seed] 全部完成 ✔（数据落在 compose 的 PG 卷里）
```

验证服务：

```bash
curl -s http://127.0.0.1:7001/healthz
# 预期：{"status":"ok","checks":{"database":"ok","redis":"ok"},"ts":"..."}

curl -s http://127.0.0.1:7001/metrics | head -n 2
# 预期：# HELP ... 指标正常输出
```

常用运维命令：

```bash
docker compose logs -f server      # 跟踪服务日志
docker compose restart server      # 重启服务
docker compose down                # 停止并删除容器（数据卷保留）
docker compose down -v             # 连数据卷一起删（慎用：清库）
```

---

# 16. 常见坑与 FAQ

**1. TS 报"装饰器"相关错误（experimental decorators）**
现象：使用类装饰器/属性装饰器时报 `Experimental support for decorators is a feature that is subject to change`。
解法：确认 `tsconfig.json` 中 `"experimentalDecorators": true` 且 `"target": "ES2021"` 以上；修改后**重启 egg-bin**（TS 配置不热加载）。若用 sequelize 装饰器风格（`@Table`），还需 `"emitDecoratorMetadata": true`。

**2. egg-validate 对 GET 请求不生效**
现象：`ctx.validate(rules)` 校验 POST 正常，GET 时永远"参数校验通过"。
原因：`ctx.validate` 默认校验 `ctx.request.body`，GET 没有 body。
解法：显式传第二参数 `ctx.validate(rules, ctx.query as any)`（见 5.6 节）。

**3. Sequelize 时间差 8 小时**
现象：库里 `2026-10-15 10:00:00`，接口返回/查询条件差 8 小时。
解法：确认三处一致：连接配置 `timezone: '+08:00'`（第 4 章）、PG 服务端时区（docker 默认 UTC，可在 compose 加 `TZ: Asia/Shanghai` 环境变量）、前端传参统一带时区（`2026-10-15T08:00:00+08:00`）。查询条件传 ISO 字符串而非 `new Date()` 裸值也能减少歧义。

**4. 事务嵌套 / 事务不生效**
现象：service 方法里开了事务，但里面调用的另一个方法内部又自己开了新事务，导致锁不住或回滚不完整。
解法：事务**从最外层业务方法开启并向内传递**（`transaction: t` 参数一路传下去），内层方法**优先复用**传入的事务；绝不在内层重复 `app.model.transaction`。Sequelize 的 CLS（continuation local storage）可以自动传递，但 egg-sequelize 默认未开启，建议显式传参最直观。

**5. JWT 过期后前端一片报错**
现象：access token 2 小时过期后，所有请求 401。
解法：这是设计使然。前端拦截 401 且业务码为 `20002`（TOKEN_EXPIRED）时，静默调 `/api/v1/auth/refresh` 换新 token 并重放原请求（第 7 章的 refresh 机制就是为此准备的）；refresh 也失败才跳登录页。

**6. 跨域 CORS 报错**
现象：浏览器控制台 `blocked by CORS policy`。
解法：确认 egg-cors 已在 `config/plugin.ts` 开启、`config.cors.origin` 覆盖前端域名；**注意**：`origin: '*'` 不能与 `credentials: 'include'`（携带 cookie）同时使用，需要 cookie 时必须写精确域名。

**7. POST 请求 403 且提示 missing csrf token**
现象：明明装了 egg-cors 还是 403。
原因：脚手架默认开启了 CSRF 防护。
解法：`config.security.csrf.enable = false`（本教程第 4 章已关，纯 token 认证的 API 不需要 CSRF）。

**8. Windows 下 curl 传 JSON 全是引号报错**
解法：本文所有 curl 命令请在 **Git Bash** 执行；PowerShell 用 `Invoke-RestMethod`；或用 Postman/Apifox。cmd.exe 的转义规则不适合 JSON。

**9. 启动报 EADDRINUSE（端口占用）**
现象：`Error: listen EADDRINUSE :::7001`。
解法：上次 dev 进程没退干净。`npx killport 7001`（或 `lsof -ti:7001 | xargs kill -9`，Windows 下 `netstat -ano | findstr 7001` 后 `taskkill /PID <pid> /F`）。

**10. 模型不生效：app.model.Xxx is not a function**
原因：egg-sequelize 只加载 `app/model/` 下的文件，文件名即模型名（`user_role.ts` → `app.model.UserRole`）。
解法：确认文件在正确目录、命名无特殊字符、默认导出是 `(app) => Model` 工厂函数；新增模型后**必须重启**（模型不热加载）。

**11. bcrypt 安装失败（gyp/编译错误）**
解法：本教程用的就是纯 JS 的 `bcryptjs`。若你执意用原生 `bcrypt`，Windows 需要 Visual Studio Build Tools、Python 3，mac 需要 Xcode Command Line Tools。

**12. Redis 连接拒绝（ECONNREFUSED 127.0.0.1:6379）**
解法：容器没起或端口不对。`docker ps` 确认 ia-redis 在运行；compose 环境注意宿主机端口映射是 `16379`，配置里要写 `IA_REDIS_PORT=16379`。

**13. MCP Server 连上但工具调用全部失败 / 客户端直接断连**
原因：stdio 传输下，`console.log`（stdout）会与 MCP 协议报文混流，一行日志就能打爆协议解析。
解法：MCP Server 进程内**所有日志走 `console.error`（stderr）**（见 13.3 节），永远不要 `console.log`。

**14. DECIMAL 字段返回的是字符串**
现象：`yieldRate: "91.20"` 而不是 `91.2`。
原因：PG 的 `numeric/decimal` 精度类型驱动返回字符串（防精度丢失），属正常行为。
解法：展示层无所谓；计算层 `Number(row.yieldRate)` 转换（第 10 章 dashboard 已示范）。

**15. ts-node 跑脚本报 "Cannot find module"**
原因：`ts-node` 对 `import()` 动态导入的解析基于 `tsconfig` 的 `module` 设置。
解法：保持 `module: "commonjs"`（本文配置）；脚本放 `scripts/` 且已在 tsconfig `include` 中；从项目根目录运行 `npm run seed` / `npm run mcp`（相对路径依赖 cwd）。

---

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

本章延续全书纪律：**每个模块仍按「Model → migration → Service → Controller → Router → curl 验证」五步走**，错误统一走 BizError + ErrorCode（沿用第 5 章错误码段，不新开号段），响应用 `ctx.success` / `ctx.page`，权限走 `auth()` + `perm({ permission })`。

### 18.0.1 安装本章新依赖

```bash
npm install minio ws --registry=https://registry.npmmirror.com
npm install -D @types/ws --registry=https://registry.npmmirror.com
# 预期：added xxx packages，无 error
```

> `minio`：MinIO 官方 JS SDK（S3 兼容协议）；`ws`：原生 WebSocket 服务端（第 19 章还会装 `openid-client`）。

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

### 18.0.3 config.default.ts 追加配置

`config.default.ts` 的 `(config as any).biz` 对象中追加两个配置块：

```typescript
    // 对象存储（18.3 缺陷图）
    minio: {
      endPoint: env('IA_MINIO_ENDPOINT', '127.0.0.1'),
      port: Number(env('IA_MINIO_PORT', '9000')),
      useSSL: false,
      accessKey: env('IA_MINIO_ACCESS_KEY', 'industry'),
      secretKey: env('IA_MINIO_SECRET_KEY', 'industry123'),
      bucket: 'defect-images',
    },
    // WebSocket（18.2）
    websocket: {
      port: Number(env('IA_WS_PORT', '7002')),
    },
```

同时在文件中新增 multipart 文件上传配置（18.3 使用，放在 `config.security` 之后即可）：

```typescript
  // 文件上传（egg-multipart）：18.3 缺陷图上传，流式处理 + 图片白名单
  config.multipart = {
    mode: 'stream',
    fileSize: '10mb',
    whitelist: [ '.png', '.jpg', '.jpeg', '.bmp', '.webp' ],
  };
```

### 18.0.4 角色权限点扩充

`app/core/role-permissions.ts` 中 `ROLE_PERMISSIONS` 更新为（给工艺工程师加报告编辑与缺陷上传，给审批员加报告签发；用户管理 / 审计查询等新权限点**不分配给任何具体角色**，只由 admin 的 `*` 覆盖——这就是"管理员专用"的实现方式）：

```typescript
export const ROLE_PERMISSIONS: Record<string, string[]> = {
  admin: [ '*' ],
  process_engineer: [
    'wafer:create', 'wafer:update', 'rootcause:trigger',
    'report:edit',      // 18.1：编辑 8D 报告草稿
    'defect:upload',    // 18.3：上传缺陷图
  ],
  reviewer: [
    'rootcause:trigger', 'suggestion:approve',
    'report:release',    // 18.1：签发 8D 报告
  ],
  viewer: [],
};
```

> 本章新增权限点一览：`report:read`（登录即可查报告，路由上只挂 auth 不挂 perm）、`report:edit`、`report:release`、`defect:upload`、`audit:read`、`user:read`、`user:create`、`user:update`、`user:reset_password`。除 `report:edit` / `report:release` / `defect:upload` 外均仅 admin（`*`）可用。

---

## 18.1 8D 报告模块

### 18.1.1 做什么 / 为什么

8D（8 Disciplines）是半导体行业质量问题的标准归档方法论：D1 成立小组 → D2 问题描述 → D3 临时围堵 → D4 根本原因 → D5 纠正措施 → D6 实施验证 → D7 预防再发 → D8 团队认可。平台在**建议审批通过后自动生成 8D 报告骨架**（D4 直接带入智能体的根因结论），质量工程师补全内容后由质量主管签发——这就是 IA-SD-005 第 11 节的 `/reports` 三个接口 + 我们补的编辑接口。

状态机：`draft`（草稿）→ `pending_review`（待复核）→ `released`（已签发，终态）。签发后不可再编辑（质量档案不可篡改）。

### 18.1.2 数据模型：app/model/report_8d.ts（完整内容）

```typescript
import { Application } from 'egg';

/**
 * 8D 报告表 reports_8d
 * 审批通过后由 report.consumeApproval 自动生成骨架（status=draft），
 * 质量工程师补全 D1~D8 后由质量主管签发（released 为终态）
 */
export default (app: Application) => {
  const { INTEGER, STRING, TEXT, DATE, ENUM } = app.Sequelize;

  const Report8D = app.model.define('report_8d', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    reportNo: { type: STRING(64), allowNull: false, unique: true, comment: '报告编号 8D-日期-序号' },
    eventId: { type: INTEGER, allowNull: true, comment: '关联异常事件' },
    taskId: { type: INTEGER, allowNull: true, comment: '关联根因分析任务' },
    suggestionId: { type: INTEGER, allowNull: true, comment: '触发本报告的建议' },
    title: { type: STRING(255), allowNull: false, defaultValue: '', comment: '报告标题' },
    // ---- D1~D8 八个 discipline，全部 TEXT（草稿期允许空串） ----
    d1Team: { type: TEXT, allowNull: true, comment: 'D1 成立小组（JSON：成员/职责）' },
    d2Problem: { type: TEXT, allowNull: true, comment: 'D2 问题描述' },
    d3Containment: { type: TEXT, allowNull: true, comment: 'D3 临时围堵措施' },
    d4Rootcause: { type: TEXT, allowNull: true, comment: 'D4 根本原因（审批通过时自动带入）' },
    d4EvidenceRefs: { type: TEXT, allowNull: true, comment: 'D4 证据链引用（JSON 数组）' },
    d5Corrective: { type: TEXT, allowNull: true, comment: 'D5 纠正措施' },
    d6Implementation: { type: TEXT, allowNull: true, comment: 'D6 措施实施与验证' },
    d7Preventive: { type: TEXT, allowNull: true, comment: 'D7 预防再发' },
    d8Recognition: { type: TEXT, allowNull: true, comment: 'D8 团队认可' },
    status: {
      type: ENUM('draft', 'pending_review', 'released'),
      allowNull: false,
      defaultValue: 'draft',
      comment: '报告状态机',
    },
    createdBy: { type: INTEGER, allowNull: true, comment: '骨架生成时的建议发起人' },
    reviewedBy: { type: INTEGER, allowNull: true, comment: '复核人' },
    releasedBy: { type: INTEGER, allowNull: true, comment: '签发人' },
    releasedAt: { type: DATE, allowNull: true, comment: '签发时间' },
  }, {
    tableName: 'reports_8d',
    timestamps: true,
    // 软删除：destroy() 不物理删行，而是写 deleted_at（质量档案留痕）
    paranoid: true,
  });

  return Report8D;
};
```

> **Migration 说明（企业实践，同 5.2 节两套机制）**：示例 `database/migrations/20261015000002-create-reports-8d.ts`：
>
> ```typescript
> import { QueryInterface, DataTypes } from 'sequelize';
>
> module.exports = {
>   async up(queryInterface: QueryInterface) {
>     await queryInterface.createTable('reports_8d', {
>       id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
>       report_no: { type: DataTypes.STRING(64), allowNull: false, unique: true },
>       event_id: { type: DataTypes.INTEGER, allowNull: true },
>       task_id: { type: DataTypes.INTEGER, allowNull: true },
>       suggestion_id: { type: DataTypes.INTEGER, allowNull: true },
>       title: { type: DataTypes.STRING(255), allowNull: false, defaultValue: '' },
>       d1_team: { type: DataTypes.TEXT, allowNull: true },
>       d2_problem: { type: DataTypes.TEXT, allowNull: true },
>       d3_containment: { type: DataTypes.TEXT, allowNull: true },
>       d4_rootcause: { type: DataTypes.TEXT, allowNull: true },
>       d4_evidence_refs: { type: DataTypes.TEXT, allowNull: true },
>       d5_corrective: { type: DataTypes.TEXT, allowNull: true },
>       d6_implementation: { type: DataTypes.TEXT, allowNull: true },
>       d7_preventive: { type: DataTypes.TEXT, allowNull: true },
>       d8_recognition: { type: DataTypes.TEXT, allowNull: true },
>       status: {
>         type: DataTypes.ENUM('draft', 'pending_review', 'released'),
>         allowNull: false, defaultValue: 'draft',
>       },
>       created_by: { type: DataTypes.INTEGER, allowNull: true },
>       reviewed_by: { type: DataTypes.INTEGER, allowNull: true },
>       released_by: { type: DataTypes.INTEGER, allowNull: true },
>       released_at: { type: DataTypes.DATE, allowNull: true },
>       created_at: { type: DataTypes.DATE, allowNull: false },
>       updated_at: { type: DataTypes.DATE, allowNull: false },
>       deleted_at: { type: DataTypes.DATE, allowNull: true },
>     });
>   },
>   async down(queryInterface: QueryInterface) {
>     await queryInterface.dropTable('reports_8d');
>   },
> };
> ```
>
> 主线仍走 `sequelize.sync()`：重新执行 `npm run seed` 即会建出新表（第 12 行之后追加的模型 import 见 18.7.4）。

### 18.1.3 Service：app/service/report.ts（完整内容）

**做什么/为什么**：三个核心业务——① `consumeApproval`：审批通过后生成骨架（供第 9 章挂钩调用）；② `update`：草稿期编辑（released 后拒绝）；③ `release`：签发状态机校验 + 显式写一条业务级审计。

```typescript
import { Service } from 'egg';
import { Op } from 'sequelize';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';

/** 8D 报告编辑接口允许提交的字段 */
const EDITABLE_FIELDS = [
  'title', 'd1Team', 'd2Problem', 'd3Containment', 'd4Rootcause', 'd4EvidenceRefs',
  'd5Corrective', 'd6Implementation', 'd7Preventive', 'd8Recognition', 'status',
] as const;

export default class ReportService extends Service {

  /** 分页查询（支持状态过滤） */
  public async list(param: { page: number; pageSize: number; status?: string; keyword?: string }) {
    const where: any = {};
    if (param.status) where.status = param.status;
    if (param.keyword) {
      where[Op.or] = [
        { title: { [Op.like]: `%${param.keyword}%` } },
        { reportNo: { [Op.like]: `%${param.keyword}%` } },
      ];
    }
    const result = await this.app.model.Report8D.findAndCountAll({
      where,
      offset: (param.page - 1) * param.pageSize,
      limit: param.pageSize,
      order: [[ 'id', 'DESC' ]],
      // 列表只带摘要字段，D1~D8 全文留给详情接口
      attributes: [ 'id', 'reportNo', 'eventId', 'taskId', 'suggestionId', 'title', 'status', 'createdBy', 'releasedBy', 'releasedAt', 'createdAt', 'updatedAt' ],
    });
    return { rows: result.rows, total: result.count };
  }

  /** 详情（含报告全文） */
  public async detail(id: number) {
    const report = await this.app.model.Report8D.findByPk(id);
    if (!report) {
      throw new BizError(ErrorCode.NOT_FOUND, `报告不存在（id=${id}）`, 404);
    }
    // D4 证据引用是 JSON 字符串，详情里还原成结构（前端免解析）
    const plain = report.toJSON();
    if (plain.d4EvidenceRefs) {
      try { plain.d4EvidenceRefs = JSON.parse(plain.d4EvidenceRefs); } catch { /* 非法 JSON 原样返回 */ }
    }
    return plain;
  }

  /**
   * 由审批通过事件自动生成报告骨架（第 9 章审批通过后调用）
   * - 同一任务只生成一份（幂等：task_id 已有报告则直接返回已有报告）
   * - D4 直接带入智能体根因结论与证据链引用
   * - 生成后给建议发起人推一条通知（18.2 实现后生效）
   */
  public async consumeApproval(param: { suggestion: any; approval: any }) {
    const { app } = this;
    const { suggestion, approval } = param;
    const taskId = suggestion.taskId;

    // 幂等：同一任务已生成过报告就不再生成
    const exists = taskId
      ? await app.model.Report8D.findOne({ where: { taskId } })
      : await app.model.Report8D.findOne({ where: { suggestionId: suggestion.id } });
    if (exists) return exists;

    // 从任务结果中提取根因结论与证据链
    let rootcauseText = '';
    let evidenceRefs: string[] = [];
    let event: any = null;
    if (taskId) {
      const task = await app.model.AgentTask.findByPk(taskId);
      if (task && task.result) {
        const top = Array.isArray(task.result.root_causes) ? task.result.root_causes[0] : null;
        if (top) {
          rootcauseText = `#${top.rank} ${top.factor}（置信度 ${top.confidence}）：${top.description || ''}`;
          evidenceRefs = [ `rootcause:${task.taskNo}#rank${top.rank}`, 'evidence:GRAPH', 'evidence:CAUSAL', 'evidence:RAG' ];
        }
        if (task.eventId) {
          event = await app.model.AnomalyEvent.findByPk(task.eventId);
        }
      }
    }

    const report = await app.model.Report8D.create({
      reportNo: `8D-${new Date().toISOString().slice(0, 10).replace(/-/g, '')}-${Date.now() % 100000}`,
      eventId: event ? event.id : null,
      taskId,
      suggestionId: suggestion.id,
      title: `批次 ${suggestion.batchNo} 建议审批通过 8D 报告（审批人：${approval.approverName}）`,
      // D2 问题描述骨架：带入建议内容
      d2Problem: `审批通过的建议：${suggestion.content}`,
      d4Rootcause: rootcauseText,
      d4EvidenceRefs: JSON.stringify(evidenceRefs),
      status: 'draft',
      createdBy: suggestion.createdBy,
    });

    // 18.2 通知模块实现后，此处自动生效（方法不存在时静默跳过）
    if ((this.service as any).notification) {
      await (this.service as any).notification.createNotification({
        userId: suggestion.createdBy,
        type: 'approval',
        title: '建议已批准，8D 报告骨架已生成',
        content: `建议「${String(suggestion.content).slice(0, 60)}」已由 ${approval.approverName} 批准，请补全 8D 报告（编号 ${report.reportNo}）后提交签发。`,
        sourceId: report.id,
      }).catch((e: Error) => this.ctx.logger.warn('[report] 通知发送失败: %s', e.message));
    }

    return report;
  }

  /** 编辑草稿（draft / pending_review 可编辑，released 拒绝） */
  public async update(id: number, data: Partial<Record<typeof EDITABLE_FIELDS[number], any>>, operatorId: number) {
    const report = await this.app.model.Report8D.findByPk(id);
    if (!report) {
      throw new BizError(ErrorCode.NOT_FOUND, `报告不存在（id=${id}）`, 404);
    }
    if (report.status === 'released') {
      throw new BizError(ErrorCode.STATE_CONFLICT, '报告已签发，质量档案不可再修改');
    }
    // 只允许编辑白名单字段，且 status 只能在 draft/pending_review 之间流转
    const patch: any = {};
    for (const key of EDITABLE_FIELDS) {
      if (data[key] !== undefined) patch[key] = data[key];
    }
    if (patch.status && ![ 'draft', 'pending_review' ].includes(patch.status)) {
      throw new BizError(ErrorCode.PARAM_INVALID, '编辑时 status 只能为 draft 或 pending_review');
    }
    if (patch.status === 'pending_review') patch.reviewedBy = operatorId;
    return await report.update(patch);
  }

  /**
   * 签发（状态机：draft / pending_review -> released）
   * 签发是质量档案定稿动作，除了 HTTP 层审计中间件自动记录外，
   * 这里再显式写一条业务级审计（含报告编号），并通知骨架发起人。
   */
  public async release(id: number, operator: { userId: number; username: string }) {
    const report = await this.app.model.Report8D.findByPk(id);
    if (!report) {
      throw new BizError(ErrorCode.NOT_FOUND, `报告不存在（id=${id}）`, 404);
    }
    if (report.status === 'released') {
      throw new BizError(ErrorCode.STATE_CONFLICT, '报告已签发，不能重复签发');
    }
    // 签发前最低完备性校验：D2 与 D4 必须已填写
    if (!report.d2Problem || !report.d4Rootcause) {
      throw new BizError(ErrorCode.PARAM_INVALID, '签发前必须填写 D2 问题描述与 D4 根本原因');
    }

    await report.update({
      status: 'released',
      releasedBy: operator.userId,
      releasedAt: new Date(),
    });

    // 显式业务级审计（区别于中间件按 HTTP 维度的自动记录）
    await this.service.audit.record({
      userId: operator.userId,
      username: operator.username,
      method: 'POST',
      path: `/api/v1/reports/${id}/release`,
      status: 200,
      durationMs: 0,
      ip: this.ctx.ip,
      requestSummary: JSON.stringify({ reportNo: report.reportNo, action: 'release_8d_report' }),
      errorMessage: null,
      traceId: this.ctx.tracer.traceId,
    });

    if ((this.service as any).notification) {
      await (this.service as any).notification.createNotification({
        userId: report.createdBy,
        type: 'system',
        title: '8D 报告已签发',
        content: `报告 ${report.reportNo} 已由 ${operator.username} 签发。`,
        sourceId: report.id,
      }).catch((e: Error) => this.ctx.logger.warn('[report] 通知发送失败: %s', e.message));
    }

    return report;
  }
}
```

### 18.1.4 Controller：app/controller/report.ts（完整内容）

```typescript
import { Controller } from 'egg';

const updateRule = {
  title: { type: 'string', required: false, allowEmpty: true, max: 255 },
  d1Team: { type: 'string', required: false, allowEmpty: true, max: 20000 },
  d2Problem: { type: 'string', required: false, allowEmpty: true, max: 20000 },
  d3Containment: { type: 'string', required: false, allowEmpty: true, max: 20000 },
  d4Rootcause: { type: 'string', required: false, allowEmpty: true, max: 20000 },
  d4EvidenceRefs: { type: 'string', required: false, allowEmpty: true, max: 20000 },
  d5Corrective: { type: 'string', required: false, allowEmpty: true, max: 20000 },
  d6Implementation: { type: 'string', required: false, allowEmpty: true, max: 20000 },
  d7Preventive: { type: 'string', required: false, allowEmpty: true, max: 20000 },
  d8Recognition: { type: 'string', required: false, allowEmpty: true, max: 20000 },
  status: { type: 'enum', values: [ 'draft', 'pending_review' ], required: false },
};

export default class ReportController extends Controller {

  /** GET /api/v1/reports?page=1&pageSize=10&status=draft&keyword= */
  public async list() {
    const { ctx } = this;
    const { pagination } = (ctx.app.config as any).biz;
    ctx.validate({
      page: { type: 'int', required: false, min: 1 },
      pageSize: { type: 'int', required: false, min: 1, max: 500 },
      status: { type: 'enum', values: [ 'draft', 'pending_review', 'released' ], required: false },
      keyword: { type: 'string', required: false, allowEmpty: true, max: 64 },
    }, ctx.query as any);

    const page = Math.max(parseInt(String(ctx.query.page ?? ''), 10) || pagination.defaultPage, 1);
    const pageSize = Math.min(
      parseInt(String(ctx.query.pageSize ?? ''), 10) || pagination.defaultPageSize,
      pagination.maxPageSize,
    );
    const result = await ctx.service.report.list({
      page, pageSize,
      status: ctx.query.status || undefined,
      keyword: ctx.query.keyword || undefined,
    });
    ctx.page(result.rows, result.total, page, pageSize);
  }

  /** GET /api/v1/reports/:id（含 D1~D8 全文） */
  public async detail() {
    this.ctx.success(await this.ctx.service.report.detail(parseInt(this.ctx.params.id, 10)));
  }

  /** PUT /api/v1/reports/:id（质量工程师编辑草稿） */
  public async update() {
    const { ctx } = this;
    ctx.validate(updateRule);
    const id = parseInt(ctx.params.id, 10);
    const report = await ctx.service.report.update(id, ctx.request.body as any, ctx.user!.userId);
    ctx.success(report, '更新成功');
  }

  /** POST /api/v1/reports/:id/release（签发） */
  public async release() {
    const { ctx } = this;
    const id = parseInt(ctx.params.id, 10);
    const report = await ctx.service.report.release(id, {
      userId: ctx.user!.userId,
      username: ctx.user!.username,
    });
    ctx.success(report, '报告已签发');
  }
}
```

### 18.1.5 路由与审批挂钩

`app/router.ts` 追加：

```typescript
  // ---------------- 8D 报告（18.1） ----------------
  router.get('/api/v1/reports', auth, controller.report.list);
  router.get('/api/v1/reports/:id', auth, controller.report.detail);
  router.put('/api/v1/reports/:id',
    auth, perm({ permission: 'report:edit' }), controller.report.update);
  router.post('/api/v1/reports/:id/release',
    auth, perm({ permission: 'report:release' }), controller.report.release);
```

**与前文挂钩（关键的一处修改）**：让第 9 章的审批通过自动触发报告骨架。在 `app/service/suggestion.ts` 的 `submitApproval` 方法中，把最后的 `return await this.app.model.transaction(...)` 改为先接住事务结果、挂钩、再返回：

```typescript
    // 原：return await this.app.model.transaction(async t => { ... });
    const result = await this.app.model.transaction(async t => {
      /* ...原事务体一字不改... */
      return { suggestion, approval };
    });

    // ★ 第 18 章挂钩：审批通过后自动生成 8D 报告骨架。
    //   注意必须在事务提交之后调用（放事务内会拉长锁持有时间，且失败会连坐回滚审批本身）
    if (param.action === 'approve') {
      await this.service.report.consumeApproval({
        suggestion: result.suggestion,
        approval: result.approval,
      }).catch(e => this.ctx.logger.warn('[report] 报告骨架生成失败: %s', e.message));
    }

    return result;
```

> 挂钩失败只打警告日志、不影响审批结果——报告骨架是"锦上添花"动作，审批是"法定"动作，二者解耦。

### 18.1.6 curl 验证

前置：重新 `npm run seed`（此时建议/审批数据重建），重启 `npm run dev`，跑一遍第 9.6 节的审批剧本（engineer 触发分析 → reviewer 批准建议 1）。批准动作会自动生成报告骨架。

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
- **WebSocket 实时推送**：事件发生瞬间推到浏览器，帧格式与 IA-SD-005 第 13 节一致（`{ event, eventId, data, ts }`）。

**为什么 WS 服务挂在 agent 进程而不是 worker**：Egg 多 worker 时每个 worker 都独立监听端口会冲突，而 agent 进程在集群中**只有一个**——由它统一持有 WS server，worker 通过 `app.messenger.sendToAgent()` 把推送请求转发给 agent 广播。这也是 Egg 官方推荐的 WebSocket 架构。

### 18.2.2 数据模型：app/model/notification.ts（完整内容）

```typescript
import { Application } from 'egg';

/**
 * 通知表 notifications：用户的站内信箱
 * type: anomaly（异常相关）/ approval（审批相关）/ system（系统公告）
 */
export default (app: Application) => {
  const { INTEGER, STRING, TEXT, DATE, BOOLEAN, ENUM } = app.Sequelize;

  const Notification = app.model.define('notification', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    userId: { type: INTEGER, allowNull: false, comment: '收件人' },
    type: {
      type: ENUM('anomaly', 'approval', 'system'),
      allowNull: false,
      defaultValue: 'system',
    },
    title: { type: STRING(255), allowNull: false },
    content: { type: TEXT, allowNull: true },
    readFlag: { type: BOOLEAN, allowNull: false, defaultValue: false, comment: '已读标记' },
    readAt: { type: DATE, allowNull: true, comment: '已读时间' },
    sourceId: { type: INTEGER, allowNull: true, comment: '来源业务对象 id（报告/事件等）' },
  }, {
    tableName: 'notifications',
    timestamps: true,
    indexes: [
      { fields: [ 'userId', 'readFlag' ] },
    ],
  });

  return Notification;
};
```

> **Migration 说明**：`database/migrations/20261015000003-create-notifications.ts`：
>
> ```typescript
> import { QueryInterface, DataTypes } from 'sequelize';
>
> module.exports = {
>   async up(queryInterface: QueryInterface) {
>     await queryInterface.createTable('notifications', {
>       id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
>       user_id: { type: DataTypes.INTEGER, allowNull: false },
>       type: { type: DataTypes.ENUM('anomaly', 'approval', 'system'), allowNull: false, defaultValue: 'system' },
>       title: { type: DataTypes.STRING(255), allowNull: false },
>       content: { type: DataTypes.TEXT, allowNull: true },
>       read_flag: { type: DataTypes.BOOLEAN, allowNull: false, defaultValue: false },
>       read_at: { type: DataTypes.DATE, allowNull: true },
>       source_id: { type: DataTypes.INTEGER, allowNull: true },
>       created_at: { type: DataTypes.DATE, allowNull: false },
>       updated_at: { type: DataTypes.DATE, allowNull: false },
>     });
>     await queryInterface.addIndex('notifications', { fields: [ 'user_id', 'read_flag' ], name: 'idx_notifications_user_read' });
>   },
>   async down(queryInterface: QueryInterface) {
>     await queryInterface.dropTable('notifications');
>   },
> };
> ```

### 18.2.3 Service：app/service/notification.ts（完整内容）

```typescript
import { Service } from 'egg';
import { Op } from 'sequelize';

export interface CreateNotificationParam {
  userId: number;
  type: 'anomaly' | 'approval' | 'system';
  title: string;
  content?: string;
  sourceId?: number;
}

export default class NotificationService extends Service {

  /**
   * 创建通知（内部 service 方法，供异常事件/审批/报告等业务埋点调用）
   * 落库 + 经 agent 进程向该用户的 WS 房间实时推送
   */
  public async createNotification(param: CreateNotificationParam) {
    const notification = await this.app.model.Notification.create({
      userId: param.userId,
      type: param.type,
      title: param.title,
      content: param.content || null,
      sourceId: param.sourceId || null,
    });

    // 实时推送：worker -> agent -> 目标用户房间（用户不在线则只落库，前端上线后补偿拉取）
    this.app.messenger.sendToAgent('ws:send-to-user', {
      userId: param.userId,
      event: 'notification.push',
      data: {
        id: notification.id,
        type: notification.type,
        title: notification.title,
        content: notification.content,
        sourceId: notification.sourceId,
        createdAt: notification.createdAt,
      },
    });
    return notification;
  }

  /**
   * 异常事件广播（供异常事件创建/根因任务完成等处调用）
   * 与 createNotification 不同：anomaly.push 面向全体在线用户广播（大屏关注）
   */
  public async pushAnomaly(data: {
    anomalyId: number; equipment: string; title: string; severity: string; occurredAt: any;
  }) {
    this.app.messenger.sendToAgent('ws:broadcast', {
      event: 'anomaly.push',
      data: {
        anomalyId: data.anomalyId,
        equipmentCode: data.equipment,
        title: data.title,
        severity: data.severity,
        occurredAt: data.occurredAt,
      },
    });
  }

  /** 未读数 */
  public async unreadCount(userId: number) {
    const count = await this.app.model.Notification.count({
      where: { userId, readFlag: false },
    });
    return { unreadCount: count };
  }

  /** 我的通知列表（分页 + 已读过滤） */
  public async list(param: {
    userId: number; page: number; pageSize: number; readFlag?: boolean;
  }) {
    const where: any = { userId: param.userId };
    if (param.readFlag !== undefined) where.readFlag = param.readFlag;
    const result = await this.app.model.Notification.findAndCountAll({
      where,
      offset: (param.page - 1) * param.pageSize,
      limit: param.pageSize,
      order: [[ 'id', 'DESC' ]],
    });
    return { rows: result.rows, total: result.count };
  }

  /** 标记单条已读 */
  public async markRead(userId: number, id: number) {
    const notification = await this.app.model.Notification.findOne({
      where: { id, userId },
    });
    if (!notification) {
      throw new (require('../core/biz-error').BizError)(
        require('../core/error-codes').ErrorCode.NOT_FOUND,
        `通知不存在（id=${id}）`, 404);
    }
    if (!notification.readFlag) {
      await notification.update({ readFlag: true, readAt: new Date() });
    }
    return notification;
  }

  /** 全部已读 */
  public async markAllRead(userId: number) {
    const [ count ] = await this.app.model.Notification.update(
      { readFlag: true, readAt: new Date() },
      { where: { userId, readFlag: false } },
    );
    return { updated: count };
  }
}
```

> 上面 `markRead` 里为演示 `BizError` 引入方式用了 `require`，与全书风格统一的等价写法是把 import 放到文件头部（推荐）：
>
> ```typescript
> import { BizError } from '../core/biz-error';
> import { ErrorCode } from '../core/error-codes';
> // ...
> throw new BizError(ErrorCode.NOT_FOUND, `通知不存在（id=${id}）`, 404);
> ```
>
> 教程最终代码采用头部 import 写法，`require` 仅为提醒你两种引入方式等价。

### 18.2.4 Controller：app/controller/notification.ts（完整内容）

```typescript
import { Controller } from 'egg';

export default class NotificationController extends Controller {

  /** GET /api/v1/notifications/unread-count */
  public async unreadCount() {
    const result = await this.ctx.service.notification.unreadCount(this.ctx.user!.userId);
    this.ctx.success(result);
  }

  /** GET /api/v1/notifications?page=1&pageSize=10&readFlag=false */
  public async list() {
    const { ctx } = this;
    const { pagination } = (ctx.app.config as any).biz;
    ctx.validate({
      page: { type: 'int', required: false, min: 1 },
      pageSize: { type: 'int', required: false, min: 1, max: 500 },
      readFlag: { type: 'boolean', required: false },
    }, ctx.query as any);

    const page = Math.max(parseInt(String(ctx.query.page ?? ''), 10) || pagination.defaultPage, 1);
    const pageSize = Math.min(
      parseInt(String(ctx.query.pageSize ?? ''), 10) || pagination.defaultPageSize,
      pagination.maxPageSize,
    );
    const readFlag = ctx.query.readFlag === undefined
      ? undefined
      : ctx.query.readFlag === 'true';
    const result = await ctx.service.notification.list({
      userId: ctx.user!.userId, page, pageSize, readFlag,
    });
    ctx.page(result.rows, result.total, page, pageSize);
  }

  /** PUT /api/v1/notifications/:id/read */
  public async markRead() {
    const id = parseInt(this.ctx.params.id, 10);
    const result = await this.ctx.service.notification.markRead(this.ctx.user!.userId, id);
    this.ctx.success(result, '已标记已读');
  }

  /** PUT /api/v1/notifications/read-all */
  public async markAllRead() {
    const result = await this.ctx.service.notification.markAllRead(this.ctx.user!.userId);
    this.ctx.success(result, '全部已读');
  }
}
```

### 18.2.5 WebSocket 服务端：agent.ts（完整内容）

Egg 约定：项目根的 `agent.ts`（与 `app.ts` 同级，脚手架已生成空文件）在 **agent 进程**加载。用以下内容替换：

```typescript
import { Agent } from 'egg';
import { WebSocketServer, WebSocket } from 'ws';

/** 房间名约定：用户房间 user:<userId> */
function userRoom(userId: number): string {
  return `user:${userId}`;
}

/** 推送帧格式（与 IA-SD-005 第 13 节一致） */
function frame(event: string, eventId: number, data: any): string {
  return JSON.stringify({ event, eventId: `evt-${eventId}`, data, ts: Date.now() });
}

export default class WsAgent extends Agent {
  private wss: WebSocketServer | null = null;
  /** 房间表：room -> 连接集合（agent 进程唯一，天然无需分布式锁） */
  private rooms = new Map<string, Set<WebSocket>>();
  /** 事件自增序号：前端断线补偿游标（生产建议改用 Redis INCR 全局递增） */
  private eventSeq = 0;

  public async didLoad() {
    const port = (this.app.config as any).biz.websocket.port;
    this.wss = new WebSocketServer({ port });

    this.wss.on('connection', (ws: WebSocket & { isAlive?: boolean; meta?: any }, req) => {
      // ---------- 1. 连接鉴权：URL query 携带 access token ----------
      const url = new URL(req.url || '/', 'http://localhost');
      const token = url.searchParams.get('token') || '';
      let payload: any = null;
      try {
        // agent 进程同样加载了 egg-jwt 插件（app.jwt 可用）
        payload = (this.app as any).jwt.verify(token, this.app.config.jwt.secret);
      } catch {
        ws.close(4001, 'unauthorized: invalid or expired token');
        return;
      }
      ws.isAlive = true;
      ws.meta = { userId: payload.userId, username: payload.username, roles: payload.roles || [] };
      this._join(userRoom(payload.userId), ws);
      this.logger.info('[ws] 用户上线：%s（userId=%s），当前连接数 %s',
        payload.username, payload.userId, this.wss!.clients.size);

      // ---------- 2. 客户端心跳应答 ----------
      ws.on('pong', () => { ws.isAlive = true; });

      // ---------- 3. 业务消息（客户端上行，本教程仅要求心跳） ----------
      ws.on('message', raw => {
        let msg: any = null;
        try { msg = JSON.parse(String(raw)); } catch { return; }
        if (msg.event === 'ping') {
          ws.send(JSON.stringify({ event: 'pong', ts: Date.now() }));
        }
      });

      ws.on('close', () => {
        this._leave(userRoom(payload.userId), ws);
        this.logger.info('[ws] 用户离线：%s，当前连接数 %s',
          payload.username, this.wss!.clients.size);
      });
      ws.on('error', () => { /* close 会随后触发，无需处理 */ });
    });

    // ---------- 4. 服务端心跳：30s 轮询，两次无应答即踢 ----------
    const heartbeat = setInterval(() => {
      if (!this.wss) return;
      this.wss.clients.forEach((ws: any) => {
        if (ws.isAlive === false) return ws.terminate();
        ws.isAlive = false;
        ws.ping();
      });
    }, 30000);
    this.wss.on('close', () => clearInterval(heartbeat));

    // ---------- 5. 接收 worker 转发来的推送请求 ----------
    this.messenger.on('ws:broadcast', (data: { event: string; data: any }) => {
      this._broadcast(data.event, data.data);
    });
    this.messenger.on('ws:send-to-user', (data: { userId: number; event: string; data: any }) => {
      this._sendToRoom(userRoom(data.userId), data.event, data.data);
    });

    this.logger.info('[ws] WebSocket server 已启动，端口 %s', port);
  }

  /** 加入房间 */
  private _join(room: string, ws: WebSocket) {
    if (!this.rooms.has(room)) this.rooms.set(room, new Set());
    this.rooms.get(room)!.add(ws);
  }

  /** 离开房间 */
  private _leave(room: string, ws: WebSocket) {
    const set = this.rooms.get(room);
    if (!set) return;
    set.delete(ws);
    if (set.size === 0) this.rooms.delete(room);
  }

  /** 广播到全体已认证连接（anomaly.push 用） */
  private _broadcast(event: string, data: any) {
    const message = frame(event, ++this.eventSeq, data);
    this.wss?.clients.forEach(ws => ws.readyState === WebSocket.OPEN && ws.send(message));
  }

  /** 定向推送到房间（notification.push 用，房间隔离：A 用户收不到 B 用户的通知） */
  private _sendToRoom(room: string, event: string, data: any) {
    const set = this.rooms.get(room);
    if (!set) return; // 目标用户不在线：只落库不推送，前端上线后拉列表补偿
    const message = frame(event, ++this.eventSeq, data);
    set.forEach(ws => ws.readyState === WebSocket.OPEN && ws.send(message));
  }
}
```

> ** typings 补充**：`ws` 的类型来自 `@types/ws`（已装）。`ws.on('pong')` 中给连接挂 `isAlive/meta` 用了类型断言写法，无需改 `typings/index.d.ts`。

### 18.2.6 业务埋点：在哪里调用通知

三个埋点位置（都在 **service 层**，不在 controller——通知是业务副作用，与 HTTP 无关）：

**埋点 1：异常事件发生时广播**。前文异常事件由监控 Agent / FDC 推送写入（教程主线暂无创建接口）。若你按 8 章扩展了 `POST /api/v1/anomaly-events`，在事件 create 之后加：

```typescript
    // 广播异常事件 + 通知相关批次责任人（演示固定通知 engineer）
    await this.service.notification.pushAnomaly({
      anomalyId: event.id, equipment: event.equipment,
      title: event.title, severity: event.severity, occurredAt: event.occurredAt,
    });
```

**埋点 2：根因分析完成时通知触发人**。在 `app/service/rootcause.ts` 的 `analyze` 方法中，`task.update({ status: 'succeeded', ... })` 之后、`return task` 之前加：

```typescript
      // 通知任务触发人：分析已完成
      if (param.createdBy) {
        await this.service.notification.createNotification({
          userId: param.createdBy,
          type: 'anomaly',
          title: `批次 ${param.batchNo} 根因分析完成`,
          content: `Top-1 根因：${(agentResult.root_causes || [])[0]?.description || '见任务详情'}`,
          sourceId: task.id,
        });
      }
```

**埋点 3：审批完成时通知建议发起人**。在 `app/service/suggestion.ts` 的 `submitApproval` 中、第 18.1.5 节挂钩代码的 `if (param.action === 'approve')` 块**之后**追加（approve / reject 都要通知）：

```typescript
    // 通知建议发起人审批结果
    if (result.suggestion.createdBy) {
      await this.service.notification.createNotification({
        userId: result.suggestion.createdBy,
        type: 'approval',
        title: param.action === 'approve' ? '你的建议已批准' : '你的建议被驳回',
        content: `审批人 ${param.approverName}，理由：${param.reason}`,
        sourceId: result.suggestion.id,
      }).catch(e => this.ctx.logger.warn('[notify] %s', e.message));
    }
```

（18.1.3 `consumeApproval` 与 `release` 中的通知调用已经在各自 service 内部埋好。）

### 18.2.7 路由

`app/router.ts` 追加。**注意顺序**：`unread-count` 与 `read-all` 必须注册在 `/:id` 之前，否则会被当成 id 解析：

```typescript
  // ---------------- 通知（18.2） ----------------
  router.get('/api/v1/notifications/unread-count', auth, controller.notification.unreadCount);
  router.get('/api/v1/notifications', auth, controller.notification.list);
  router.put('/api/v1/notifications/read-all', auth, controller.notification.markAllRead);
  router.put('/api/v1/notifications/:id/read', auth, controller.notification.markRead);
```

### 18.2.8 前端 WS 客户端如何对接（呼应前端教程）

前端（React）侧的对接要点（完整实现见《01-前端开发教程-React从0到1》相应章节）：

```typescript
// 1. 建连：token 走 query（WS 无法自定义 Header，这是通行做法）
const ws = new WebSocket(`ws://127.0.0.1:7002/?token=${accessToken}`);

// 2. 收到推送：按 event 分发
ws.onmessage = e => {
  const { event, eventId, data } = JSON.parse(e.data);
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
//    再对比本地已处理的 eventId，补上断线期间漏掉的事件（IA-SD-005 12.3 的"断线补偿"）。
```

### 18.2.9 curl 验证

重启 `npm run dev`（agent 进程随 master 一起启动，日志应出现 `[ws] WebSocket server 已启动，端口 7002`）。跑一遍审批剧本让通知落库后：

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

**④ WebSocket 推送验证**（需要 Node 一行脚本做客户端）：

```bash
node -e "
const WebSocket = require('ws');
const token = process.argv[1];
const ws = new WebSocket('ws://127.0.0.1:7002/?token=' + token);
ws.on('open', () => console.log('[client] 已连接'));
ws.on('message', m => console.log('[client] 收到推送:', m.toString()));
ws.on('close', (c, r) => console.log('[client] 连接关闭:', c, r.toString()));
setTimeout(() => process.exit(0), 15000);
" "$ENGINEER_TOKEN"
```

保持这个客户端挂着，另开终端触发一次审批（对 pending 状态的建议 approve），客户端应立即打印：

```text
[client] 已连接
[client] 收到推送: {"event":"notification.push","eventId":"evt-1","data":{"id":4,"type":"approval","title":"你的建议已批准","content":"审批人 reviewer，理由：...","sourceId":2,"createdAt":"..."},"ts":1760493305000}
```

再验证鉴权（不带 token 连接应被 4001 关闭）：

```bash
node -e "
const WebSocket = require('ws');
const ws = new WebSocket('ws://127.0.0.1:7002/');
ws.on('close', (c, r) => console.log('[client] 关闭码:', c, '原因:', r.toString()));
"
# 预期输出：[client] 关闭码: 4001 原因: unauthorized: invalid or expired token
```

---

## 18.3 缺陷模块（MinIO 对象存储 + 预签名 URL）

### 18.3.1 做什么 / 为什么

缺陷图（AOI 显微照片）单张几 MB 到几十 MB，存 PostgreSQL 会把库拖垮；正确姿势是**对象存储**（MinIO，S3 兼容），数据库只存 `image_object_key`。前端看图走**预签名 URL**：后端用凭证替用户签一个限时 GET URL，图片流量直连 MinIO，不经过业务后端，凭证也不下发到前端——这就是"数据不出域"的图片访问方案（对应 IA-SD-004 1.1 节 MinIO `defect-images` 桶与 IA-SD-005 7.2 节接口）。

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

浏览器打开 `http://127.0.0.1:9001`（industry / industry123）可看到控制台。桶由下方 service 的 `ensureBucket` 在首次访问时自动创建，无需手工操作。

### 18.3.3 数据模型：app/model/defect_record.ts（完整内容）

字段语义对齐《数据库设计说明书（IA-SD-004）》的 `defect_records`（wafer/批次、设备、缺陷类型、坐标、尺寸、图片键、严重度、检出时间）；按教程惯例用整型主键与 camelCase 字段（差异说明见 18.7.3）。

```typescript
import { Application } from 'egg';

/**
 * 缺陷记录表 defect_records
 * 由 AOI / 缺陷检测设备检出写入；image_object_key 指向 MinIO 的缺陷图
 */
export default (app: Application) => {
  const { INTEGER, STRING, DECIMAL, DATE, ENUM } = app.Sequelize;

  const DefectRecord = app.model.define('defect_record', {
    id: { type: INTEGER, primaryKey: true, autoIncrement: true },
    waferId: { type: INTEGER, allowNull: true, comment: '关联晶圆（教程简化：批次维度）' },
    batchNo: { type: STRING(32), allowNull: true, comment: '批次号' },
    equipmentCode: { type: STRING(64), allowNull: false, comment: '检出设备，如 ETCH-03' },
    processStep: { type: STRING(32), allowNull: true, comment: '工序步，如 ETCH_MAIN' },
    defectType: { type: STRING(32), allowNull: false, comment: '缺陷类型：PATTERN/PARTICLE/SCRATCH/RESIDUE...' },
    defectCount: { type: INTEGER, allowNull: false, defaultValue: 1, comment: '该类型缺陷个数' },
    xPos: { type: DECIMAL(8, 2), allowNull: true, comment: '缺陷位置 X（mm）' },
    yPos: { type: DECIMAL(8, 2), allowNull: true, comment: '缺陷位置 Y（mm）' },
    sizeUm: { type: DECIMAL(8, 2), allowNull: true, comment: '缺陷尺寸（微米）' },
    imageObjectKey: { type: STRING(255), allowNull: true, comment: 'MinIO 对象键，如 defects/2026/10/xxx.png' },
    severity: {
      type: ENUM('low', 'medium', 'high'),
      allowNull: false,
      defaultValue: 'low',
      comment: '严重度',
    },
    detectedAt: { type: DATE, allowNull: false, comment: '检出时间' },
  }, {
    tableName: 'defect_records',
    timestamps: true,
    indexes: [
      { fields: [ 'defectType' ] },
      { fields: [ 'detectedAt' ] },
    ],
  });

  return DefectRecord;
};
```

> **Migration 说明**：`database/migrations/20261015000004-create-defect-records.ts`：
>
> ```typescript
> import { QueryInterface, DataTypes } from 'sequelize';
>
> module.exports = {
>   async up(queryInterface: QueryInterface) {
>     await queryInterface.createTable('defect_records', {
>       id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
>       wafer_id: { type: DataTypes.INTEGER, allowNull: true },
>       batch_no: { type: DataTypes.STRING(32), allowNull: true },
>       equipment_code: { type: DataTypes.STRING(64), allowNull: false },
>       process_step: { type: DataTypes.STRING(32), allowNull: true },
>       defect_type: { type: DataTypes.STRING(32), allowNull: false },
>       defect_count: { type: DataTypes.INTEGER, allowNull: false, defaultValue: 1 },
>       x_pos: { type: DataTypes.DECIMAL(8, 2), allowNull: true },
>       y_pos: { type: DataTypes.DECIMAL(8, 2), allowNull: true },
>       size_um: { type: DataTypes.DECIMAL(8, 2), allowNull: true },
>       image_object_key: { type: DataTypes.STRING(255), allowNull: true },
>       severity: { type: DataTypes.ENUM('low', 'medium', 'high'), allowNull: false, defaultValue: 'low' },
>       detected_at: { type: DataTypes.DATE, allowNull: false },
>       created_at: { type: DataTypes.DATE, allowNull: false },
>       updated_at: { type: DataTypes.DATE, allowNull: false },
>     });
>     await queryInterface.addIndex('defect_records', { fields: [ 'defect_type' ] });
>     await queryInterface.addIndex('defect_records', { fields: [ 'detected_at' ] });
>   },
>   async down(queryInterface: QueryInterface) {
>     await queryInterface.dropTable('defect_records');
>   },
> };
> ```

### 18.3.4 Service：app/service/defect.ts（完整内容）

```typescript
import { Service } from 'egg';
import * as crypto from 'crypto';
import { Op } from 'sequelize';
import * as Minio from 'minio';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';

/** 预签名有效期：7 天（秒）——质检复盘窗口 */
const PRESIGN_EXPIRES_SECONDS = 7 * 24 * 3600;

export default class DefectService extends Service {
  private minioClient: Minio.Client;

  constructor(ctx: any) {
    super(ctx);
    const cfg = (ctx.app.config as any).biz.minio;
    this.minioClient = new Minio.Client({
      endPoint: cfg.endPoint,
      port: cfg.port,
      useSSL: cfg.useSSL,
      accessKey: cfg.accessKey,
      secretKey: cfg.secretKey,
    });
  }

  /** 确保桶存在（幂等，首次调用时创建） */
  private async ensureBucket(): Promise<string> {
    const bucket = (this.app.config as any).biz.minio.bucket;
    const exists = await this.minioClient.bucketExists(bucket);
    if (!exists) await this.minioClient.makeBucket(bucket, 'cn-north-1');
    return bucket;
  }

  /** 分页查询（多条件过滤：批次/设备/类型/严重度/时间范围） */
  public async list(param: {
    page: number; pageSize: number;
    batchNo?: string; equipmentCode?: string; defectType?: string; severity?: string;
    startTime?: string; endTime?: string;
  }) {
    const where: any = {};
    if (param.batchNo) where.batchNo = { [Op.like]: `%${param.batchNo}%` };
    if (param.equipmentCode) where.equipmentCode = { [Op.like]: `%${param.equipmentCode}%` };
    if (param.defectType) where.defectType = param.defectType;
    if (param.severity) where.severity = param.severity;
    if (param.startTime || param.endTime) {
      where.detectedAt = {};
      if (param.startTime) where.detectedAt[Op.gte] = new Date(param.startTime);
      if (param.endTime) where.detectedAt[Op.lte] = new Date(param.endTime);
    }
    const result = await this.app.model.DefectRecord.findAndCountAll({
      where,
      offset: (param.page - 1) * param.pageSize,
      limit: param.pageSize,
      order: [[ 'detectedAt', 'DESC' ]],
    });
    return { rows: result.rows, total: result.count };
  }

  /** 详情 */
  public async detail(id: number) {
    const defect = await this.app.model.DefectRecord.findByPk(id);
    if (!defect) throw new BizError(ErrorCode.NOT_FOUND, `缺陷记录不存在（id=${id}）`, 404);
    return defect;
  }

  /**
   * 缺陷图预签名 GET URL（7 天有效）
   * 凭证只在本服务端使用，URL 本身限时、可分享给内网浏览器直接访问
   */
  public async getImageUrl(id: number) {
    const defect = await this.detail(id);
    if (!defect.imageObjectKey) {
      throw new BizError(ErrorCode.NOT_FOUND, `该缺陷记录没有关联图片（id=${id}）`, 404);
    }
    const bucket = await this.ensureBucket();
    const url = await this.minioClient.presignedGetObject(
      bucket, defect.imageObjectKey, PRESIGN_EXPIRES_SECONDS);
    return {
      url,
      objectKey: defect.imageObjectKey,
      expiresIn: PRESIGN_EXPIRES_SECONDS,
    };
  }

  /**
   * 上传缺陷图（multipart stream 直写 MinIO，不落本地磁盘）
   * 返回新的 objectKey 并回写到缺陷记录
   */
  public async uploadImage(id: number, stream: any) {
    const defect = await this.detail(id);

    // 扩展名白名单校验（config.multipart.whitelist 的第二道防线）
    const ext = '.' + String(stream.filename || '').split('.').pop().toLowerCase();
    if (![ '.png', '.jpg', '.jpeg', '.bmp', '.webp' ].includes(ext)) {
      throw new BizError(ErrorCode.PARAM_INVALID, `不支持的图片类型：${ext}`);
    }

    const bucket = await this.ensureBucket();
    const now = new Date();
    const objectKey = `defects/${now.getFullYear()}/${String(now.getMonth() + 1).padStart(2, '0')}/${crypto.randomUUID()}${ext}`;

    try {
      await this.minioClient.putObject(bucket, objectKey, stream, undefined, {
        'Content-Type': stream.mime || 'application/octet-stream',
      });
    } catch (e: any) {
      this.ctx.logger.error('[defect] MinIO 上传失败: %s', e.message);
      throw new BizError(50000, '对象存储上传失败，请稍后重试', 502);
    }

    await defect.update({ imageObjectKey: objectKey });
    return { id: defect.id, imageObjectKey: objectKey };
  }
}
```

### 18.3.5 Controller：app/controller/defect.ts（完整内容）

```typescript
import { Controller } from 'egg';

export default class DefectController extends Controller {

  /** GET /api/v1/defects?batchNo=&equipmentCode=&defectType=&severity=&startTime=&endTime= */
  public async list() {
    const { ctx } = this;
    const { pagination } = (ctx.app.config as any).biz;
    ctx.validate({
      page: { type: 'int', required: false, min: 1 },
      pageSize: { type: 'int', required: false, min: 1, max: 500 },
      batchNo: { type: 'string', required: false, allowEmpty: true },
      equipmentCode: { type: 'string', required: false, allowEmpty: true },
      defectType: { type: 'string', required: false, allowEmpty: true },
      severity: { type: 'enum', values: [ 'low', 'medium', 'high' ], required: false },
      startTime: { type: 'datetime', required: false },
      endTime: { type: 'datetime', required: false },
    }, ctx.query as any);

    const page = Math.max(parseInt(String(ctx.query.page ?? ''), 10) || pagination.defaultPage, 1);
    const pageSize = Math.min(
      parseInt(String(ctx.query.pageSize ?? ''), 10) || pagination.defaultPageSize,
      pagination.maxPageSize,
    );
    const result = await ctx.service.defect.list({
      page, pageSize,
      batchNo: ctx.query.batchNo || undefined,
      equipmentCode: ctx.query.equipmentCode || undefined,
      defectType: ctx.query.defectType || undefined,
      severity: ctx.query.severity || undefined,
      startTime: ctx.query.startTime || undefined,
      endTime: ctx.query.endTime || undefined,
    });
    ctx.page(result.rows, result.total, page, pageSize);
  }

  /** GET /api/v1/defects/:id/image-url（MinIO 预签名） */
  public async imageUrl() {
    const id = parseInt(this.ctx.params.id, 10);
    this.ctx.success(await this.ctx.service.defect.getImageUrl(id));
  }

  /** POST /api/v1/defects/:id/image（multipart 上传，stream 直写 MinIO） */
  public async uploadImage() {
    const { ctx } = this;
    if (!ctx.isMultipart) {
      ctx.status = 422;
      ctx.body = { code: 10001, message: '请使用 multipart/form-data 上传图片', traceId: ctx.tracer.traceId };
      return;
    }
    const id = parseInt(ctx.params.id, 10);
    const stream = await ctx.getFileStream(); // 已按 config.multipart 白名单校验
    const result = await ctx.service.defect.uploadImage(id, stream);
    ctx.status = 201;
    ctx.success(result, '上传成功');
  }
}
```

### 18.3.6 路由与种子数据

`app/router.ts` 追加：

```typescript
  // ---------------- 缺陷记录（18.3） ----------------
  router.get('/api/v1/defects', auth, controller.defect.list);
  router.get('/api/v1/defects/:id/image-url', auth, controller.defect.imageUrl);
  router.post('/api/v1/defects/:id/image',
    auth, perm({ permission: 'defect:upload' }), controller.defect.uploadImage);
```

`scripts/init-db.ts` 追加种子数据（放在传感器读数之后）：

```typescript
  // ---------- 种子数据：缺陷记录（18.3） ----------
  const DefectRecord = (await import('../app/model/defect_record')).default(miniApp);
  await DefectRecord.bulkCreate([
    { batchNo: 'B20261015001', equipmentCode: 'ETCH-03', processStep: 'ETCH_MAIN', defectType: 'PATTERN', defectCount: 6, xPos: 42.10, yPos: 88.30, sizeUm: 1.80, severity: 'high', detectedAt: hoursAgo(28) },
    { batchNo: 'B20261015001', equipmentCode: 'ETCH-03', processStep: 'ETCH_MAIN', defectType: 'PARTICLE', defectCount: 2, xPos: 12.50, yPos: 30.00, sizeUm: 0.90, severity: 'medium', detectedAt: hoursAgo(27) },
    { batchNo: 'B20261015003', equipmentCode: 'CVD-11', processStep: 'CVD_MAIN', defectType: 'SCRATCH', defectCount: 3, xPos: 66.20, yPos: 15.80, sizeUm: 5.20, severity: 'medium', detectedAt: hoursAgo(6) },
    { batchNo: 'B20261015003', equipmentCode: 'LITH-02', processStep: 'LITHO_DEV', defectType: 'RESIDUE', defectCount: 1, xPos: 50.00, yPos: 50.00, sizeUm: 2.40, severity: 'low', detectedAt: hoursAgo(30) },
    { batchNo: 'B20261015002', equipmentCode: 'CVD-11', processStep: 'CVD_MAIN', defectType: 'PATTERN', defectCount: 1, xPos: 20.10, yPos: 71.60, sizeUm: 1.10, severity: 'low', detectedAt: hoursAgo(25) },
    { batchNo: 'B20261015001', equipmentCode: 'CMP-02', processStep: 'CMP_MAIN', defectType: 'PARTICLE', defectCount: 4, xPos: 88.00, yPos: 9.90, sizeUm: 0.60, severity: 'medium', detectedAt: hoursAgo(20) },
  ]);
  console.log('[seed] 缺陷记录 6 条写入完成');
```

重新执行 `npm run seed` 并重启 `npm run dev`（确认 ia-minio 容器在跑）。

### 18.3.7 curl 验证

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
printf '\x89PNG\r\n\x1a\n\x00\x00\x00\rIHDR\x00\x00\x00\x01\x00\x00\x00\x01\x08\x06\x00\x00\x00\x1f\x15\xc4\x89\x00\x00\x00\nIDATx\x9cc\x00\x01\x00\x00\x05\x00\x01\r\n\x2d\xb4\x00\x00\x00\x00IEND\xaeB\x60\x82' > /tmp/defect.png

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

### 18.4.2 Service：app/service/dashboard.ts（完整内容）

```typescript
import { Service } from 'egg';

/** 缓存键（IA-SD-005 12.1 规定 dash:overview 前缀，教程沿用 ia: 命名空间） */
const OVERVIEW_KEY = 'ia:stats:dashboard:overview';

export default class DashboardService extends Service {

  /**
   * 总览（cache-aside，同第 10 章 StatsService 的模式）
   * 聚合：当前良率 / OEE / 今日异常统计 / 待审批数 + 扩展指标
   */
  public async overview() {
    const ttl = (this.app.config as any).biz.cache.dashboardTtlSeconds;

    const cached = await this.app.redis.get(OVERVIEW_KEY);
    if (cached) {
      return { source: 'cache', data: JSON.parse(cached) };
    }

    const QueryTypes = this.app.Sequelize.QueryTypes;

    // ---- 1. 近 7 天完工/出货批次平均良率 ----
    const [ yieldRow ] = await this.app.model.query(
      `SELECT ROUND(AVG(yield_rate), 2) AS avg_yield
         FROM wafers
        WHERE status IN ('completed', 'shipped')
          AND produced_at >= CURRENT_DATE - INTERVAL '7 days'`,
      { type: QueryTypes.SELECT },
    );
    const yieldRate = yieldRow && yieldRow.avg_yield !== null ? Number(yieldRow.avg_yield) : null;

    // ---- 2. OEE（教程简化口径，见 18.4.1 说明） ----
    // 2a. 时间稼动率：近 24h 出现过读数的 (设备, 小时) 组合数 / (设备数 × 24)
    const [ availRow ] = await this.app.model.query(
      `SELECT COUNT(DISTINCT (equipment_id, date_trunc('hour', recorded_at))) AS active_slots,
              COUNT(DISTINCT equipment_id) AS equip_count
         FROM sensor_readings
        WHERE recorded_at >= NOW() - INTERVAL '24 hours'`,
      { type: QueryTypes.SELECT },
    );
    const equipCount = availRow ? Number(availRow.equip_count) : 0;
    const activeSlots = availRow ? Number(availRow.active_slots) : 0;
    const availability = equipCount > 0
      ? Math.min(activeSlots / (equipCount * 24), 1)
      : 0;
    // 2b. 性能稼动率：演示固定值（接入 MES 后按实际节拍/理论节拍计算）
    const performance = 0.92;
    // 2c. 良品率
    const quality = yieldRate === null ? 0 : yieldRate / 100;
    const oee = Math.round(availability * performance * quality * 10000) / 100;

    // ---- 3. 今日异常统计（按严重度分组） ----
    const anomalyRows = await this.app.model.query(
      `SELECT severity, COUNT(*) AS cnt
         FROM anomaly_events
        WHERE occurred_at >= CURRENT_DATE
        GROUP BY severity`,
      { type: QueryTypes.SELECT },
    );
    const anomalyToday = anomalyRows.reduce((s: number, r: any) => s + Number(r.cnt), 0);
    const anomalyBySeverity: Record<string, number> = {};
    anomalyRows.forEach((r: any) => { anomalyBySeverity[r.severity] = Number(r.cnt); });

    // ---- 4. 待审批数 ----
    const pendingApprovals = await this.app.model.Suggestion.count({
      where: { status: 'pending' },
    });

    // ---- 5. Top 根因因子（近 30 天任务结果的 JSONB 展开统计） ----
    const topRootcauses = await this.app.model.query(
      `SELECT rc->>'factor' AS name, COUNT(*) AS count
         FROM agent_tasks t,
              jsonb_array_elements(t.result->'root_causes') AS rc
        WHERE t.status = 'succeeded'
          AND t.created_at >= CURRENT_DATE - INTERVAL '30 days'
        GROUP BY rc->>'factor'
        ORDER BY count DESC
        LIMIT 5`,
      { type: QueryTypes.SELECT },
    );

    const data = {
      yieldRate,
      yieldTarget: 95.0,
      oee,
      oeeTarget: 85.0,
      availability: Math.round(availability * 10000) / 100,
      performance: Math.round(performance * 10000) / 100,
      anomalyToday,
      anomalyBySeverity,
      pendingApprovals,
      topRootcauses: topRootcauses.map((r: any) => ({ name: r.name, count: Number(r.count) })),
      generatedAt: new Date().toISOString(),
    };

    await this.app.redis.set(OVERVIEW_KEY, JSON.stringify(data), 'EX', ttl);
    return { source: 'db', data };
  }

  /** 总览缓存失效（建议在审批/异常写入路径调用，同第 10 章模式） */
  public async invalidateOverview() {
    await this.app.redis.del(OVERVIEW_KEY);
  }

  /** 良率趋势（近 N 天，按完工批次日聚合；generate_series 补零保证连续日期） */
  public async yieldTrend(days: number) {
    const QueryTypes = this.app.Sequelize.QueryTypes;
    const rows = await this.app.model.query(
      `SELECT to_char(d.day, 'YYYY-MM-DD') AS date,
              COALESCE(ROUND(AVG(w.yield_rate), 2), 0) AS avg_yield,
              COUNT(w.id) AS batches
         FROM generate_series(
                CURRENT_DATE - make_interval(days => :days - 1),
                CURRENT_DATE,
                INTERVAL '1 day'
              ) AS d(day)
         LEFT JOIN wafers w
           ON date_trunc('day', w.produced_at) = d.day
          AND w.status IN ('completed', 'shipped')
        GROUP BY d.day
        ORDER BY d.day`,
      { type: QueryTypes.SELECT, replacements: { days } },
    );
    return rows.map((r: any) => ({
      date: r.date,
      avgYield: Number(r.avg_yield),
      batches: Number(r.batches),
    }));
  }

  /** 缺陷分类统计（饼图数据源） */
  public async defectCategory() {
    const QueryTypes = this.app.Sequelize.QueryTypes;
    const rows = await this.app.model.query(
      `SELECT defect_type, COUNT(*) AS records, SUM(defect_count) AS total_defects
         FROM defect_records
        WHERE detected_at >= CURRENT_DATE - INTERVAL '30 days'
        GROUP BY defect_type
        ORDER BY total_defects DESC`,
      { type: QueryTypes.SELECT },
    );
    return rows.map((r: any) => ({
      defectType: r.defect_type,
      records: Number(r.records),
      totalDefects: Number(r.total_defects),
    }));
  }

  /** 异常高发设备 Top N（柱图数据源） */
  public async equipmentTop(limit: number) {
    const QueryTypes = this.app.Sequelize.QueryTypes;
    const rows = await this.app.model.query(
      `SELECT equipment,
              COUNT(*) AS anomaly_count,
              COUNT(*) FILTER (WHERE severity = 'critical') AS critical_count
         FROM anomaly_events
        WHERE occurred_at >= CURRENT_DATE - INTERVAL '30 days'
        GROUP BY equipment
        ORDER BY anomaly_count DESC, equipment
        LIMIT :limit`,
      { type: QueryTypes.SELECT, replacements: { limit } },
    );
    return rows.map((r: any) => ({
      equipment: r.equipment,
      anomalyCount: Number(r.anomaly_count),
      criticalCount: Number(r.critical_count),
    }));
  }
}
```

### 18.4.3 Controller：app/controller/dashboard.ts（完整内容）

```typescript
import { Controller } from 'egg';

export default class DashboardController extends Controller {

  /** GET /api/v1/dashboard/overview */
  public async overview() {
    this.ctx.success(await this.ctx.service.dashboard.overview());
  }

  /** GET /api/v1/dashboard/yield-trend?days=14 */
  public async yieldTrend() {
    const { ctx } = this;
    ctx.validate({
      days: { type: 'int', required: false, min: 1, max: 90 },
    }, ctx.query as any);
    const days = parseInt(String(ctx.query.days ?? ''), 10) || 14;
    ctx.success(await ctx.service.dashboard.yieldTrend(days));
  }

  /** GET /api/v1/dashboard/defect-category */
  public async defectCategory() {
    this.ctx.success(await this.ctx.service.dashboard.defectCategory());
  }

  /** GET /api/v1/dashboard/equipment-top?limit=5 */
  public async equipmentTop() {
    const { ctx } = this;
    ctx.validate({
      limit: { type: 'int', required: false, min: 1, max: 20 },
    }, ctx.query as any);
    const limit = parseInt(String(ctx.query.limit ?? ''), 10) || 5;
    ctx.success(await ctx.service.dashboard.equipmentTop(limit));
  }
}
```

### 18.4.4 路由

`app/router.ts` 追加：

```typescript
  // ---------------- 看板统计（18.4） ----------------
  router.get('/api/v1/dashboard/overview', auth, controller.dashboard.overview);
  router.get('/api/v1/dashboard/yield-trend', auth, controller.dashboard.yieldTrend);
  router.get('/api/v1/dashboard/defect-category', auth, controller.dashboard.defectCategory);
  router.get('/api/v1/dashboard/equipment-top', auth, controller.dashboard.equipmentTop);
```

缓存失效挂钩（同第 10 章做法）：在 `app/service/suggestion.ts` 的 `submitApproval` 末尾、以及未来异常事件的写路径中追加：

```typescript
    // 看板总览缓存失效（18.4）
    await this.service.dashboard.invalidateOverview();
```

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

### 18.5.2 Service：给 app/service/audit.ts 追加 list 方法

在 `AuditService` 类中追加（与 6.2 节的 `record` 并列；同时文件头部追加 `import { Op } from 'sequelize';`）：

```typescript
  /** 审计日志检索（管理员专用）：操作人 / 路径前缀 / 时间范围 */
  public async list(param: {
    page: number; pageSize: number;
    username?: string; path?: string;
    startTime?: string; endTime?: string;
  }) {
    const where: any = {};
    if (param.username) where.username = { [Op.like]: `%${param.username}%` };
    if (param.path) where.path = { [Op.like]: `${param.path}%` };
    if (param.startTime || param.endTime) {
      where.created_at = {};
      if (param.startTime) where.created_at[Op.gte] = new Date(param.startTime);
      if (param.endTime) where.created_at[Op.lte] = new Date(param.endTime);
    }
    const result = await this.app.model.AuditLog.findAndCountAll({
      where,
      offset: (param.page - 1) * param.pageSize,
      limit: param.pageSize,
      order: [[ 'id', 'DESC' ]],
    });
    return { rows: result.rows, total: result.count };
  }
```

### 18.5.3 Controller：app/controller/audit.ts（完整内容）

```typescript
import { Controller } from 'egg';

export default class AuditController extends Controller {

  /** GET /api/v1/audits?username=&path=&startTime=&endTime= */
  public async list() {
    const { ctx } = this;
    const { pagination } = (ctx.app.config as any).biz;
    ctx.validate({
      page: { type: 'int', required: false, min: 1 },
      pageSize: { type: 'int', required: false, min: 1, max: 500 },
      username: { type: 'string', required: false, allowEmpty: true, max: 64 },
      path: { type: 'string', required: false, allowEmpty: true, max: 255 },
      startTime: { type: 'datetime', required: false },
      endTime: { type: 'datetime', required: false },
    }, ctx.query as any);

    const page = Math.max(parseInt(String(ctx.query.page ?? ''), 10) || pagination.defaultPage, 1);
    const pageSize = Math.min(
      parseInt(String(ctx.query.pageSize ?? ''), 10) || pagination.defaultPageSize,
      pagination.maxPageSize,
    );
    const result = await ctx.service.audit.list({
      page, pageSize,
      username: ctx.query.username || undefined,
      path: ctx.query.path || undefined,
      startTime: ctx.query.startTime || undefined,
      endTime: ctx.query.endTime || undefined,
    });
    ctx.page(result.rows, result.total, page, pageSize);
  }
}
```

### 18.5.4 路由

`app/router.ts` 追加：

```typescript
  // ---------------- 审计查询（18.5，仅 admin） ----------------
  router.get('/api/v1/audits',
    auth, perm({ permission: 'audit:read' }), controller.audit.list);
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

### 18.6.2 Service：app/service/user.ts（完整内容）

```typescript
import { Service } from 'egg';
import * as bcrypt from 'bcryptjs';
import { Op } from 'sequelize';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';

/** 用户表敏感列：任何查询都不允许返回 */
const SAFE_ATTRIBUTES = [
  'id', 'username', 'realName', 'email', 'status', 'lastLoginAt', 'createdAt', 'updatedAt',
];

export default class UserService extends Service {

  /** 用户分页列表（含角色；keyword 模糊匹配用户名/姓名） */
  public async list(param: { page: number; pageSize: number; keyword?: string; status?: string }) {
    const where: any = {};
    if (param.keyword) {
      where[Op.or] = [
        { username: { [Op.like]: `%${param.keyword}%` } },
        { realName: { [Op.like]: `%${param.keyword}%` } },
      ];
    }
    if (param.status) where.status = param.status;

    const result = await this.app.model.User.findAndCountAll({
      where,
      attributes: SAFE_ATTRIBUTES,
      offset: (param.page - 1) * param.pageSize,
      limit: param.pageSize,
      order: [[ 'id', 'ASC' ]],
    });

    // 批量补角色（两条查询，避免 N+1）
    const userIds = result.rows.map(u => u.id);
    const rolesMap = await this._getRolesMap(userIds);
    const rows = result.rows.map(u => ({
      ...u.toJSON(),
      roles: rolesMap.get(u.id) || [],
    }));
    return { rows, total: result.count };
  }

  /** 创建用户（事务：建用户 + 绑角色；密码 bcrypt 哈希，永不明文） */
  public async create(param: {
    username: string; password: string; realName?: string; email?: string;
    roleCodes: string[];
  }) {
    return await this.app.model.transaction(async t => {
      const exists = await this.app.model.User.findOne({
        where: { username: param.username },
        transaction: t,
      });
      if (exists) {
        throw new BizError(ErrorCode.DUPLICATE, `用户名 ${param.username} 已存在`);
      }
      const user = await this.app.model.User.create({
        username: param.username,
        passwordHash: bcrypt.hashSync(param.password, 10),
        realName: param.realName || null,
        email: param.email || null,
        status: 'active',
      }, { transaction: t });

      await this._bindRoles(user.id, param.roleCodes, t);
      return { ...user.toJSON(), passwordHash: undefined, roles: param.roleCodes };
    });
  }

  /** 更新用户（资料与启用/禁用；不许改用户名） */
  public async update(id: number, param: {
    realName?: string; email?: string; status?: 'active' | 'disabled';
  }) {
    const user = await this.app.model.User.findByPk(id);
    if (!user) throw new BizError(ErrorCode.NOT_FOUND, `用户不存在（id=${id}）`, 404);

    if (param.status === 'disabled' && id === this.ctx.user!.userId) {
      throw new BizError(ErrorCode.STATE_CONFLICT, '不能禁用当前登录的自己');
    }
    const patch: any = {};
    if (param.realName !== undefined) patch.realName = param.realName;
    if (param.email !== undefined) patch.email = param.email;
    if (param.status !== undefined) patch.status = param.status;
    await user.update(patch);
    return { ...user.toJSON(), passwordHash: undefined };
  }

  /** 改绑角色（事务：删旧全量重插） */
  public async updateRoles(id: number, roleCodes: string[]) {
    return await this.app.model.transaction(async t => {
      const user = await this.app.model.User.findByPk(id, { transaction: t });
      if (!user) throw new BizError(ErrorCode.NOT_FOUND, `用户不存在（id=${id}）`, 404);

      // 校验角色码全部合法（IA-SD-005 5.2：IA-11002 角色码非法）
      const roles = await this.app.model.Role.findAll({
        where: { code: roleCodes }, transaction: t,
      });
      if (roles.length !== roleCodes.length) {
        const found = roles.map(r => r.code);
        const missing = roleCodes.filter(c => !found.includes(c));
        throw new BizError(ErrorCode.PARAM_INVALID, `角色码非法：${missing.join(',')}`);
      }

      await this.app.model.UserRole.destroy({ where: { userId: id }, transaction: t });
      await this._bindRoles(id, roleCodes, t);
      return { userId: id, roles: roleCodes };
    });
  }

  /** 重置密码（bcrypt 新哈希） */
  public async resetPassword(id: number, newPassword: string) {
    const user = await this.app.model.User.findByPk(id);
    if (!user) throw new BizError(ErrorCode.NOT_FOUND, `用户不存在（id=${id}）`, 404);
    await user.update({ passwordHash: bcrypt.hashSync(newPassword, 10) });
    return { id, passwordReset: true };
  }

  /** 批量绑定角色（内部方法） */
  private async _bindRoles(userId: number, roleCodes: string[], t?: any) {
    if (!roleCodes || roleCodes.length === 0) return;
    const roles = await this.app.model.Role.findAll({ where: { code: roleCodes } });
    if (roles.length !== roleCodes.length) {
      const found = roles.map(r => r.code);
      const missing = roleCodes.filter(c => !found.includes(c));
      throw new BizError(ErrorCode.PARAM_INVALID, `角色码非法：${missing.join(',')}`);
    }
    await this.app.model.UserRole.bulkCreate(
      roles.map(r => ({ userId, roleId: r.id })), { transaction: t },
    );
  }

  /** userId -> 角色码数组 的批量映射 */
  private async _getRolesMap(userIds: number[]): Promise<Map<number, string[]>> {
    const map = new Map<number, string[]>();
    if (userIds.length === 0) return map;
    const userRoles = await this.app.model.UserRole.findAll({
      where: { userId: userIds },
    });
    if (userRoles.length === 0) return map;
    const roleIds = userRoles.map(ur => ur.roleId);
    const roles = await this.app.model.Role.findAll({ where: { id: roleIds } });
    const roleIdToCode = new Map(roles.map(r => [ r.id, r.code ]));
    for (const ur of userRoles) {
      const code = roleIdToCode.get(ur.roleId);
      if (!code) continue;
      if (!map.has(ur.userId)) map.set(ur.userId, []);
      map.get(ur.userId)!.push(code);
    }
    return map;
  }
}
```

### 18.6.3 Controller：app/controller/user.ts（完整内容）

```typescript
import { Controller } from 'egg';

const createRule = {
  username: { type: 'string', required: true, min: 3, max: 64 },
  password: { type: 'string', required: true, min: 8, max: 64 },
  realName: { type: 'string', required: false, allowEmpty: true, max: 64 },
  email: { type: 'email', required: false, allowEmpty: true },
  roleCodes: { type: 'array', required: true, itemType: 'string', min: 1, max: 5 },
};

const updateRule = {
  realName: { type: 'string', required: false, allowEmpty: true, max: 64 },
  email: { type: 'email', required: false, allowEmpty: true },
  status: { type: 'enum', values: [ 'active', 'disabled' ], required: false },
};

const rolesRule = {
  roleCodes: { type: 'array', required: true, itemType: 'string', min: 0, max: 5 },
};

const passwordRule = {
  password: { type: 'string', required: true, min: 8, max: 64 },
};

export default class UserController extends Controller {

  /** GET /api/v1/users?keyword=&status= */
  public async list() {
    const { ctx } = this;
    const { pagination } = (ctx.app.config as any).biz;
    ctx.validate({
      page: { type: 'int', required: false, min: 1 },
      pageSize: { type: 'int', required: false, min: 1, max: 500 },
      keyword: { type: 'string', required: false, allowEmpty: true, max: 64 },
      status: { type: 'enum', values: [ 'active', 'disabled' ], required: false },
    }, ctx.query as any);

    const page = Math.max(parseInt(String(ctx.query.page ?? ''), 10) || pagination.defaultPage, 1);
    const pageSize = Math.min(
      parseInt(String(ctx.query.pageSize ?? ''), 10) || pagination.defaultPageSize,
      pagination.maxPageSize,
    );
    const result = await ctx.service.user.list({
      page, pageSize,
      keyword: ctx.query.keyword || undefined,
      status: ctx.query.status || undefined,
    });
    ctx.page(result.rows, result.total, page, pageSize);
  }

  /** POST /api/v1/users（管理员创建） */
  public async create() {
    const { ctx } = this;
    ctx.validate(createRule);
    const user = await ctx.service.user.create(ctx.request.body as any);
    ctx.status = 201;
    ctx.success(user, '创建成功');
  }

  /** PUT /api/v1/users/:id（资料 + 启用禁用） */
  public async update() {
    const { ctx } = this;
    ctx.validate(updateRule);
    const user = await ctx.service.user.update(parseInt(ctx.params.id, 10), ctx.request.body as any);
    ctx.success(user, '更新成功');
  }

  /** PUT /api/v1/users/:id/roles（改绑角色） */
  public async updateRoles() {
    const { ctx } = this;
    ctx.validate(rolesRule);
    const result = await ctx.service.user.updateRoles(
      parseInt(ctx.params.id, 10), (ctx.request.body as any).roleCodes);
    ctx.success(result, '角色已更新');
  }

  /** PUT /api/v1/users/:id/password（重置密码） */
  public async resetPassword() {
    const { ctx } = this;
    ctx.validate(passwordRule);
    const result = await ctx.service.user.resetPassword(
      parseInt(ctx.params.id, 10), (ctx.request.body as any).password);
    ctx.success(result, '密码已重置');
  }
}
```

### 18.6.4 路由

`app/router.ts` 追加：

```typescript
  // ---------------- 用户管理（18.6，仅 admin） ----------------
  router.get('/api/v1/users',
    auth, perm({ permission: 'user:read' }), controller.user.list);
  router.post('/api/v1/users',
    auth, perm({ permission: 'user:create' }), controller.user.create);
  router.put('/api/v1/users/:id',
    auth, perm({ permission: 'user:update' }), controller.user.update);
  router.put('/api/v1/users/:id/roles',
    auth, perm({ permission: 'user:update' }), controller.user.updateRoles);
  router.put('/api/v1/users/:id/password',
    auth, perm({ permission: 'user:reset_password' }), controller.user.resetPassword);
```

### 18.6.5 curl 验证

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

`scripts/init-db.ts` 中追加两个模型的 import（无需种子数据，仅为 `sequelize.sync()` 能建表；缺陷模型已在 18.3.6 加入）：

```typescript
  // ---------- 18 章新表（仅建表，无种子） ----------
  await import('../app/model/report_8d');
  await import('../app/model/notification');
  console.log('[seed] reports_8d / notifications 表结构同步完成');
```

重新执行 `npm run seed`（会清库重建，重跑一遍 9.6 审批剧本让 18 章数据就位），重启 `npm run dev`。

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
| #34 GET /notifications | #27~30 | 教程拆为列表/未读数/已读标记四个接口（IA-SD-005 的 sinceEventId 断线补偿由前端结合 eventId 帧实现） |

**约定级差异（以本教程为准）**：

1. **分页参数**：教程统一 `page` / `pageSize`（IA-SD-005 写作 `pageNum`）；分页响应为 `data: { rows, total, page, pageSize }`（IA-SD-005 为 `{ list, total, pageNum, pageSize }`）。若需严格对齐 IA-SD-005，只需改 `ctx.page` helper 一处 + Controller 取参处。
2. **错误码**：教程用整型段（10xxx/20xxx/30xxx/50000，见 5.3 节）；IA-SD-005 用 `IA-1xxxx` 字符串段。映射关系一一对应（如 10002↔IA-11001、20001↔IA-10401），网关层可无损转换。
3. **统一响应**：教程成功 message 为 `ok`（IA-SD-005 为 `success`），且响应体带 `traceId`（教程增强项）。
4. **JWT 签名**：HS256 自签（教学简化）→ 第 19 章给出 Keycloak RS256 完整升级方案。

结论：**IA-SD-005 的 34 个接口中 32 个已实现（含等价替代），2 个（retry/takeover、approvals/stat）列为后续迭代**；教程另补充 9 个实用端点（编辑报告、通知四件套、密码重置、看板两扩展、旧 stats 端点）。

### 18.7.4 一键冒烟脚本

`scripts/smoke-test.sh`（完整内容；Git Bash 执行 `bash scripts/smoke-test.sh`）。脚本顺序调用 18 个接口，登录提取 token，逐个断言 `code:0`：

```bash
#!/usr/bin/env bash
# industryagent-server 冒烟测试：跑通前先确保 PG/Redis/MinIO/Python Mock/server 都在跑，
# 且已执行过 npm run seed + 9.6 节审批剧本（保证有 pending 建议与生成的报告）。
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

控制台手点只适合第一次探索。**生产与团队协作的正确姿势**是把 realm 配置导出为 JSON 纳入 git（配置即代码，同第 4 章"代码与环境隔离"的精），新环境 `--import-realm` 一键重建。

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

## 19.3 后端改造：验签 RS256 公钥

**改造目标**：`auth` 中间件在 `keycloak` 模式下改验 Keycloak 签发的 RS256 token；`ctx.user` 从 claims 映射并**同步到本地 users 表（upsert）**；权限判定**不动**（仍走本地 RBAC 表）。用配置开关切换两种模式，随时可回退。

### 19.3.1 获取 realm 公钥

Keycloak 每个 realm 的 RS256 公钥（x509）可从 realm 端点直接拿：

```bash
curl -s http://127.0.0.1:8080/realms/ia | grep -o '"public_key":"[^"]*"' | cut -d'"' -f4
# 预期输出：MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...（一长串 base64）
```

> 生产环境更推荐 **JWKS 动态轮换**（`/realms/ia/protocol/openid-connect/certs`，key 轮换时自动刷新，用 `createRemoteJWKSet`——见 19.3.3 的方案 B 注释）。教程用静态公钥注入配置，路径最短、依赖最少。

### 19.3.2 配置与 .env 变更

`.env` 追加：

```bash
# ---- 认证模式（19.3）：local=自签JWT（默认） / keycloak ----
IA_AUTH_MODE=keycloak
IA_KEYCLOAK_BASE_URL=http://127.0.0.1:8080
IA_KEYCLOAK_REALM=ia
IA_KEYCLOAK_CLIENT_ID=industryagent-server
IA_KEYCLOAK_CLIENT_SECRET=industryagent-server-secret-change-me
IA_KEYCLOAK_PUBLIC_KEY=把 19.3.1 复制的 public_key 粘贴到这里
```

`config.default.ts` 的 `(config as any).biz.auth` 扩展为：

```typescript
    auth: {
      accessExpiresIn: '2h',
      refreshExpiresInSeconds: 7 * 24 * 3600,
      // ---- 19.3：认证模式与 Keycloak ----
      mode: env('IA_AUTH_MODE', 'local'),   // 'local' | 'keycloak'
      keycloak: {
        baseUrl: env('IA_KEYCLOAK_BASE_URL', 'http://127.0.0.1:8080'),
        realm: env('IA_KEYCLOAK_REALM', 'ia'),
        clientId: env('IA_KEYCLOAK_CLIENT_ID', 'industryagent-server'),
        clientSecret: env('IA_KEYCLOAK_CLIENT_SECRET', ''),
        // realm RS256 公钥（19.3.1 获取；keycloak 模式必填）
        publicKey: env('IA_KEYCLOAK_PUBLIC_KEY', ''),
      },
    },
```

### 19.3.3 用户模型与 auth 中间件改造

**① users 表加 keycloak 关联字段**：`app/model/user.ts` 的字段定义中追加一行（IA-SD-004 3.1 的 `keycloak_id`）：

```typescript
    keycloakId: { type: STRING(64), allowNull: true, unique: true, comment: 'Keycloak sub（OIDC 关联，19.3）' },
```

**② auth 中间件改造**：`app/middleware/auth.ts` 完整替换为（local 分支 = 原逻辑一字不差，keycloak 分支 = 新增）：

```typescript
import { Context, Next } from 'egg';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';

/** 把 Keycloak 的 base64 DER 公钥包成 PEM */
function toPem(base64Key: string): string {
  return `-----BEGIN PUBLIC KEY-----\n${base64Key}\n-----END PUBLIC KEY-----`;
}

/**
 * JWT 鉴权中间件（路由级按需挂载）
 * - local 模式：验自签 HS256（第 7 章，完全保留）
 * - keycloak 模式：验 Keycloak RS256 + claims -> 本地用户 upsert
 * 用法不变：router.get('/api/v1/xxx', auth(), controller.xxx)
 */
export default () => {
  return async function jwtAuth(ctx: Context, next: Next) {
    const header = ctx.get('authorization');
    const token = header ? header.replace(/^Bearer\s+/i, '') : '';
    if (!token) {
      throw new BizError(ErrorCode.UNAUTHORIZED, '缺少访问令牌（Authorization: Bearer <token>）', 401);
    }

    const authCfg = (ctx.app.config as any).biz.auth;
    try {
      if (authCfg.mode === 'keycloak') {
        // ---------- Keycloak RS256 验签 ----------
        const payload = ctx.app.jwt.verify(token, toPem(authCfg.keycloak.publicKey), {
          algorithms: [ 'RS256' ],          // 必须显式锁定算法，防 alg 混淆攻击
          issuer: `${authCfg.keycloak.baseUrl}/realms/${authCfg.keycloak.realm}`,
        });
        if (payload.typ === 'Refresh') {
          throw new BizError(ErrorCode.UNAUTHORIZED, '请使用访问令牌调用本接口', 401);
        }
        // claims -> 本地用户 upsert（sub 优先，其次 preferred_username）
        ctx.user = await ctx.service.auth.syncKeycloakUser(payload);
      } else {
        // ---------- 自签 HS256（第 7 章原逻辑） ----------
        const payload = ctx.app.jwt.verify(token, ctx.app.config.jwt.secret);
        ctx.user = {
          userId: payload.userId,
          username: payload.username,
          roles: payload.roles || [],
        };
      }
    } catch (e: any) {
      if (e instanceof BizError) throw e;
      if (e && e.name === 'TokenExpiredError') {
        throw new BizError(ErrorCode.TOKEN_EXPIRED, '登录已过期，请刷新令牌或重新登录', 401);
      }
      throw new BizError(ErrorCode.UNAUTHORIZED, '无效的访问令牌', 401);
    }
    await next();
  };
};
```

> **方案 B（可选）：openid-client introspection**。若需要"管理员在 Keycloak 侧强制登出后 token 立即失效"（本地验签做不到），改用内省：`npm install openid-client` 后，把上面 keycloak 分支的验签替换为每次请求调 Keycloak 的 introspection 端点（confidential client 凭据 + token），返回 `active: true` 才放行。代价是每请求一次内网调用（可用 Redis 按 token 哈希缓存 30~60s 摊薄）。静态公钥验签（方案 A）与内省（方案 B）的取舍：**只关心真伪选 A（快），关心即时吊销选 B（准）**。教程主线用 A。

**③ 用户同步 Service**：`app/service/auth.ts` 中追加方法（放在 `_getUserRoles` 前面）：

```typescript
  /**
   * Keycloak claims -> 本地用户 upsert（19.3）
   * 按 sub（keycloakId）查，查不到按 username 兜底，仍没有则建档。
   * 返回的 ctx.user 角色来自本地 RBAC 表——权限永远在本地。
   */
  public async syncKeycloakUser(claims: any) {
    const { app } = this;
    let user = claims.sub
      ? await app.model.User.findOne({ where: { keycloakId: claims.sub } })
      : null;
    if (!user && claims.preferred_username) {
      user = await app.model.User.findOne({ where: { username: claims.preferred_username } });
      // 老用户首次用 Keycloak 登录：回填 keycloakId 建立绑定
      if (user && !user.keycloakId) {
        await user.update({ keycloakId: claims.sub });
      }
    }
    if (!user) {
      user = await app.model.User.create({
        username: claims.preferred_username || `kc_${claims.sub.slice(0, 12)}`,
        passwordHash: bcrypt.hashSync(crypto.randomUUID(), 10), // 占位哈希：密码认证已外包给 Keycloak
        realName: claims.name || claims.preferred_username || null,
        email: claims.email || null,
        keycloakId: claims.sub,
        status: 'active',
      });
    }
    if (user.status !== 'active') {
      throw new BizError(ErrorCode.USER_DISABLED, '账号已被禁用，请联系管理员', 403);
    }
    const roles = await this._getUserRoles(user.id);
    return { userId: user.id, username: user.username, roles };
  }
```

> 注意 `crypto` 与 `bcrypt` 在文件头部已有 import（第 7 章），无需重复引入。**权限映射完全不动**：`permission` 中间件、`ROLE_PERMISSIONS`、`user_roles` 表原样生效——Keycloak 负责"你是谁"，本地 RBAC 负责"你能干什么"。

**④ users 表结构变更生效**：模型加了字段，开发环境直接重跑 `npm run seed` 重建（清库），或手工执行：

```bash
docker exec -it ia-pg psql -U industry -d industryagent \
  -c "ALTER TABLE users ADD COLUMN IF NOT EXISTS keycloak_id VARCHAR(64) UNIQUE;"
```

## 19.4 登录流程变化：后端代理 token 端点（推荐方案）

**方案选择**：授权码流程（前端跳 Keycloak 登录页）是 OAuth2 最标准玩法，但需要前端处理回调、PKCE、跨域会话，改造成本高且体验割裂（内外网两套登录页）。**教程推荐"后端代理"**：前端表单照旧 POST 用户名密码到后端，后端持 client_secret 走 ROPC 找 Keycloak 换 token——前端零改造、密钥不出服务器、响应结构与第 7 章 `login` 完全一致。

`app/service/keycloak.ts`（完整内容，新增文件）：

```typescript
import { Service } from 'egg';
import { BizError } from '../core/biz-error';
import { ErrorCode } from '../core/error-codes';

/**
 * Keycloak 对接（19.4）：后端代理登录/刷新/登出
 * 前端只见本服务返回的统一结构，client_secret 永不下发浏览器
 */
export default class KeycloakService extends Service {

  private get cfg() {
    return (this.app.config as any).biz.auth.keycloak;
  }

  private get tokenUrl(): string {
    return `${this.cfg.baseUrl}/realms/${this.cfg.realm}/protocol/openid-connect/token`;
  }

  /** 密码直换 token（ROPC） */
  public async login(username: string, password: string) {
    const res = await this.ctx.curl(this.tokenUrl, {
      method: 'POST',
      contentType: 'form',
      dataType: 'json',
      data: {
        grant_type: 'password',
        client_id: this.cfg.clientId,
        client_secret: this.cfg.clientSecret,
        username, password,
      },
      timeout: 5000,
    });
    if (res.status !== 200 || !(res.data as any).access_token) {
      // Keycloak 对"用户不存在"与"密码错误"同样返回 400 invalid_grant，天然防用户名枚举
      throw new BizError(ErrorCode.LOGIN_FAILED, '用户名或密码错误');
    }
    return this._normalize(res.data as any);
  }

  /** 刷新 token */
  public async refresh(refreshToken: string) {
    const res = await this.ctx.curl(this.tokenUrl, {
      method: 'POST',
      contentType: 'form',
      dataType: 'json',
      data: {
        grant_type: 'refresh_token',
        client_id: this.cfg.clientId,
        client_secret: this.cfg.clientSecret,
        refresh_token: refreshToken,
      },
      timeout: 5000,
    });
    if (res.status !== 200 || !(res.data as any).access_token) {
      throw new BizError(ErrorCode.TOKEN_EXPIRED, '刷新令牌无效或已过期，请重新登录', 401);
    }
    return this._normalize(res.data as any);
  }

  /** 登出：吊销 refresh token（access token 短时效自然过期） */
  public async logout(refreshToken?: string) {
    if (!refreshToken) return { revoked: false };
    await this.ctx.curl(
      `${this.cfg.baseUrl}/realms/${this.cfg.realm}/protocol/openid-connect/logout`,
      {
        method: 'POST',
        contentType: 'form',
        dataType: 'json',
        data: {
          client_id: this.cfg.clientId,
          client_secret: this.cfg.clientSecret,
          refresh_token: refreshToken,
        },
        timeout: 5000,
      },
    );
    return { revoked: true };
  }

  /** Keycloak 响应 -> 第 7 章 login 接口的返回结构（前端零改造的关键） */
  private async _normalize(data: any) {
    // 从 access_token 解出用户信息（验签在 auth 中间件，这里只 decode 取 claims）
    const payload: any = this.app.jwt.decode(data.access_token);
    const user = await this.service.auth.syncKeycloakUser(payload);
    return {
      accessToken: data.access_token,
      refreshToken: data.refresh_token,
      tokenType: 'Bearer',
      expiresInSeconds: data.expires_in,
      user: {
        userId: user.userId,
        username: user.username,
        roles: user.roles,
      },
    };
  }
}
```

`app/controller/auth.ts` 中追加三个方法（与原有 login/refresh/logout 并列，**原有方法保留**作为 local 模式入口）：

```typescript
  /** POST /api/v1/auth/keycloak/login（19.4：后端代理 Keycloak 登录） */
  public async kcLogin() {
    const { ctx } = this;
    ctx.validate(loginRule); // 复用第 7 章规则：username/password
    const { username, password } = ctx.request.body as any;
    ctx.success(await ctx.service.keycloak.login(username, password));
  }

  /** POST /api/v1/auth/keycloak/refresh */
  public async kcRefresh() {
    const { ctx } = this;
    ctx.validate(refreshRule); // 复用：refreshToken
    ctx.success(await ctx.service.keycloak.refresh((ctx.request.body as any).refreshToken));
  }

  /** POST /api/v1/auth/keycloak/logout */
  public async kcLogout() {
    const { refreshToken } = (this.ctx.request.body || {}) as any;
    this.ctx.success(await this.ctx.service.keycloak.logout(refreshToken));
  }
```

路由追加：

```typescript
  // ---------------- Keycloak 认证（19.4，与第 7 章端点并存） ----------------
  router.post('/api/v1/auth/keycloak/login', controller.auth.kcLogin);
  router.post('/api/v1/auth/keycloak/refresh', controller.auth.kcRefresh);
  router.post('/api/v1/auth/keycloak/logout', controller.auth.kcLogout);
```

> 前端改造：把登录/刷新的请求地址从 `/auth/login` 换成 `/auth/keycloak/login`（或由网关按 `IA_AUTH_MODE` 转发），**请求体与响应结构完全不变**——这就是代理方案的回报。

## 19.5 验证与回退

### 19.5.1 全链路验证

确保 ia-keycloak 在跑、`.env` 已按 19.3.2 配置（`IA_AUTH_MODE=keycloak`），重启 `npm run dev`：

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

重启 `npm run dev` 即恢复第 7 章全流程（`/auth/login` 端点从未被删除，双 token + Redis 白名单照常工作）。建议生产切换时灰度观察一周：两套端点并存，前端按环境变量决定调哪套，确认无回归后再下线 local 模式。

## 19.6 本章小结：全书完结

### 19.6.1 全书模块清单（1~19 章交付物总览）

| 能力 | 关键实现 | 章节 |
|---|---|---|
| 工程骨架 | Egg.js 4.0 + TS 约定式分层（router/controller/service/model/middleware） | 3 |
| 配置体系 | default/local/prod/test 四层 + dotenv 环境变量注入 | 4 |
| CRUD 模板 | wafer 模块：egg-validate 校验、事务查重、统一响应、curl 全流程验证 | 5 |
| 横切关注点 | BizError 错误体系、审计日志、结构化请求日志 + traceId | 5~6 |
| 鉴权（本地） | bcrypt + JWT 双 token + Redis 白名单 + RBAC 多角色 | 7 |
| 智能体协作 | httpclient 超时/重试/熔断降级调用 Python 服务，任务表全生命周期 | 8 |
| 业务闭环 | 分析 → 建议落库 → 审批（事务 + 行锁）→ 审计留痕 | 9 |
| Redis 进阶 | cache-aside 看板缓存 + 写路径失效、Idempotency-Key 幂等 | 10 |
| 后台任务 | schedule 断流告警 + 良率日报（interval / cron / worker 类型） | 11 |
| 质量保障 | sinon 单测 mock 模型 + egg-mock 集成测试 + 覆盖率 | 12 |
| MCP | stdio Server 复用模型定义，三个工具暴露给智能体，客户端脚本验收 | 13 |
| 可观测 | prom-client QPS/延迟指标、/healthz 探活 | 14 |
| 部署 | tsc 构建 + 多阶段 Dockerfile + compose 编排 + 容器健康检查 | 15 |
| 8D 报告 | reports_8d 状态机（draft→pending_review→released）、审批通过自动生成骨架、签发写审计 | 18.1 |
| 通知 + WebSocket | notifications 站内信四接口、agent 进程 WS server（鉴权/房间隔离/心跳）、IA-SD-005 帧格式 | 18.2 |
| 缺陷 + 对象存储 | defect_records 多条件检索、MinIO stream 上传、7 天预签名 GET URL | 18.3 |
| 看板统计 | overview（良率/OEE/今日异常/待审批，cache-aside 60s）、yield-trend、defect-category、equipment-top | 18.4 |
| 审计查询 | audit_logs 管理员检索（操作人/路径/时间范围） | 18.5 |
| 用户管理 | 创建（bcrypt）/启停/改绑角色/重置密码，全部 admin 权限 | 18.6 |
| Keycloak | RS256 公钥验签、claims→本地用户 upsert、后端代理登录、一键回退 | 19 |

**接口规模**：43 个 REST 端点（18.7.2 总表）+ 1 个 WS 推送通道（2 类事件）+ 3 个 MCP 工具。与 IA-SD-005 的对照结论见 18.7.3：34 个接口中 32 个已实现（含等价替代），2 个（任务 retry/takeover、审批 SLA 统计）列为后续迭代。

### 19.6.2 与文档群的映射

| 文档 | 编号 | 与本教程的对应关系 |
|---|---|---|
| 用户需求说明书 / SRS | IA-URS-001 / IA-SRS-001 | 业务能力来源（异常监控、根因分析、人机协同审批） |
| 系统架构设计文档 | IA-SD-001 | 1.2 节架构图即其 Egg.js 后端部分的落地 |
| 后端详细设计说明书 | IA-SD-002 | 模块/分层设计的实现依据 |
| 数据库设计说明书 | IA-SD-004 | 表结构语义依据（教程简化：整型主键、camelCase 字段，差异见 18.7.3） |
| 接口设计说明书 | IA-SD-005 | 34 接口 + WS 事件 + MCP 工具的对照基准（18.7.3 差异表） |
| 前端开发教程 | IA-TU-002 | 消费本教程全部 REST API 与 WS 通道（18.2.8 对接要点） |
| 智能体开发教程 | IA-TU-003 | 替换 agent-mock，对接 POST /api/v1/rootcause/analyze 协议与 MCP 工具 |
| 部署上线教程 | IA-TU-004 | 第 15 章 compose 的生产化延伸 |

### 19.6.3 三条心法（贯穿全书）

1. **分层纪律**：Controller 薄、Service 厚、Model 纯——18 章的六个新模块全部按 wafer 模板的五步走（Model → Service → Controller → Router → curl 验证），没有一处例外；
2. **约束前置**：错误码、审计、traceId 在第 5~6 天就立好规矩，18 章补模块时**零成本**继承——18.7 的差异表里所有"教程增强项"（traceId、幂等、审计）都是早期约束的红利；
3. **复用技巧**："迷你 app + 模型工厂"让同一份模型在 Egg 进程、seed 脚本、MCP Server 三处复用；"认证外包、授权本地"让 Keycloak 迁移只动了 auth 中间件一个分支——**好的架构让变更局部化**。

### 19.6.4 下一步

- **《03-智能体开发教程》**：把 `agent-mock/main.py` 替换为真正的多智能体服务（LangGraph 编排监控/根因/建议 Agent，接入知识图谱与 RAG），并对接本教程的 `POST /api/v1/rootcause/analyze` 协议与 13 章的 MCP 工具；
- **《01-前端开发教程-React从0到1》**：基于 43 个 REST 端点与 WS 通道构建管理台（登录、批次管理、看板大屏、审批工作流、通知铃铛），落地 token 静默刷新与断线补偿；
- **进阶方向（按优先级）**：补齐 retry/takeover 与审批 SLA 统计两接口 → OpenTelemetry 分布式追踪打通 traceId → GitHub Actions CI（lint + test + build 镜像）→ migration 替代 sync → WebSocket 集群化（Redis pub/sub 替代 agent 单点）→ K8s 编排与灰度发布。

**全教程完。** 从 `npm init egg` 到 43 个端点、从自签 JWT 到 Keycloak、从单表 CRUD 到"分析→建议→审批→报告→通知"的完整人机协同闭环——这套代码与文档群，就是你的第一份企业级全栈交付物。

---

> 文档编号：IA-TU-001 | 版本 V1.1 | 编制：李吉 | 日期：2026-10-15
> 本文档属于 IndustryAgent 项目文档体系，修订请遵循修订记录表规范。
