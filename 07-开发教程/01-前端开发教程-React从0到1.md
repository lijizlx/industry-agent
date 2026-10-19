# IndustryAgent 前端开发教程：React 从 0 到 1

> **项目**：industryagent-web —— 芯联半导体良率根因分析平台前端
> **一句话背景**：虚拟 12 英寸晶圆厂"芯联半导体"良率突降 2%，智能体平台需在 4 小时内给出 Top-3 根因嫌疑，关键调参建议须经工程师审批后执行。本教程带你从零搭建该平台的前端。

---

## 文档控制块

| 项目 | 内容 |
|---|---|
| 文档编号 | IA-TU-002 |
| 文档名称 | IndustryAgent 前端开发教程：React 从 0 到 1 |
| 版本 | V1.0 |
| 编制 | 李吉 |
| 日期 | 2026-10-15 |
| 适用对象 | 有 JS 基础、了解一点 React 但未做过企业级项目的初学者 |
| 配套文档 | IA-TU-001《后端开发教程（Egg.js）》、IA-PRJ-001《虚拟企业白皮书》 |

### 修订记录

| 版本 | 日期 | 修订人 | 修订内容 |
|---|---|---|---|
| V1.0 | 2026-10-15 | 李吉 | 初版发布：覆盖环境搭建到部署上线全流程 |
| V1.1 | 2026-10-15 | 李吉 | 增补第 19 章「补全页面」：8D 报告、审计日志、系统管理、通知中心、缺陷查询五类页面，页面覆盖对齐 IA-SD-003 与高保真原型 |

---

## 总目录

1. [教程导读：我们要做什么](#1-教程导读我们要做什么)
2. [环境与初始化](#2-环境与初始化)
3. [工程基础设施](#3-工程基础设施)
4. [请求层封装：Axios 统一接入层](#4-请求层封装axios-统一接入层)
5. [登录页与鉴权](#5-登录页与鉴权)
6. [主布局：侧边菜单 + 顶栏 + 内容区](#6-主布局侧边菜单--顶栏--内容区)
7. [路由与菜单：配置驱动](#7-路由与菜单配置驱动)
8. [良率看板页：React Query + ECharts](#8-良率看板页react-query--echarts)
9. [异常事件列表页：查询表格 + WebSocket](#9-异常事件列表页查询表格--websocket)
10. [根因分析详情页：Top-3 嫌疑 + 证据链](#10-根因分析详情页top-3-嫌疑--证据链)
11. [审批工作台](#11-审批工作台)
12. [权限控制：路由级 / 菜单级 / 按钮级](#12-权限控制路由级--菜单级--按钮级)
13. [全局状态管理进阶](#13-全局状态管理进阶)
14. [前端测试：Vitest + RTL](#14-前端测试vitest--rtl)
15. [构建与部署](#15-构建与部署)
19. [补全页面：8D 报告 / 审计日志 / 系统管理 / 通知中心 / 缺陷查询](#19-补全页面8d-报告--审计日志--系统管理--通知中心--缺陷查询)（V1.1 增补，物理位置在部署章之后）
16. [性能优化清单](#16-性能优化清单)
17. [常见坑 FAQ](#17-常见坑-faq)
18. [小结与下一步](#18-小结与下一步)

---

## 1. 教程导读：我们要做什么

### 1.1 成品预览：industryagent-web 长什么样

学完本教程，你将得到一个可独立运行、可对接真实后端的企业级前端应用，包含以下页面：

| 页面 | 路由 | 功能描述 |
|---|---|---|
| 登录页 | `/login` | 账号密码登录、表单校验、错误提示、token 持久化 |
| 良率看板 | `/dashboard` | 良率/OEE/异常数/待审批数四张统计卡；良率趋势折线图、缺陷分类饼图、Top 异常设备柱状图；30 秒自动刷新 |
| 异常事件列表 | `/anomaly` | 按设备/工序/严重程度/状态组合查询的分页表格；状态标签颜色映射；WebSocket 实时推送新事件并右上角通知；点击行跳详情 |
| 根因分析详情 | `/anomaly/:id` | 事件信息卡片；Top-3 嫌疑卡片（置信度进度条/嫌疑参数/关联工序）；证据链 Tabs（参数时序对比图、知识图谱关系图、相似历史案例、因果分析结论）；"重新分析"异步任务轮询 |
| 审批工作台 | `/approval` | 待审批列表；审批弹窗（通过/驳回 + 理由必填）；审批历史时间线（Timeline） |
| 审批历史 | `/approval/history` | 已办审批记录时间线展示 |

同时具备三条隐形能力：**路由级/菜单级/按钮级三层权限**、**统一请求层（token 附带、错误弹窗、401 跳登录）**、**WebSocket 断线重连与跨页面事件分发**。

### 1.2 技术选型理由表

| 技术 | 版本 | 选型理由 |
|---|---|---|
| React | 18 | 生态最成熟、招聘需求最大；18 的并发特性为后续升级铺路 |
| TypeScript | 5.x | 企业项目标配：接口类型即文档，重构有兜底 |
| Vite | 5.x | 秒级冷启动、原生 ESM、生产构建基于 Rollup |
| Ant Design | 5.x | 企业中后台组件最全；5.x 用 CSS-in-JS 支持 `ConfigProvider` 运行时主题（含暗色模式） |
| React Router | 6.x | 声明式嵌套路由、`loader/lazy` 内建；6.x 是当前主流 |
| Zustand | 4.x/5.x | 比 Redux 轻一个量级、无样板代码；`persist` 中间件解决 token 持久化 |
| TanStack React Query | 5.x | 服务端状态管理事实标准：缓存、重试、轮询、失效一应俱全，替代手写 loading/error/data |
| Axios | 1.x | 拦截器机制成熟，统一处理 token 与错误最顺手 |
| ECharts | 5.x | 国产图表库对中文场景、大数据量渲染支持最好 |
| Vitest + RTL | 1.x / 15.x | Vite 原生测试跑得快；Testing Library 引导"以用户视角"写测试 |

### 1.3 知识地图（学习顺序与依赖关系）

```text
环境与脚手架(第2章)
   └─ TS/主题/样式(第3章)
        └─ 请求层 + Mock(第4章) ←───── 后端接口约定
             └─ 登录与鉴权(第5章)
                  └─ 主布局(第6章)
                       └─ 路由与菜单(第7章)
                            ├─ 看板: ReactQuery+ECharts(第8章)
                            ├─ 列表: 表格+WebSocket(第9章)
                            ├─ 详情: 异步任务轮询(第10章)
                            └─ 审批: Mutation+失效(第11章)
                                 └─ 三层权限(第12章)
                                      └─ 状态进阶/测试/部署/优化(第13-16章)
```

### 1.4 后端接口约定（贯穿全教程）

本教程前端与后端（Egg.js，见 IA-TU-001）约定如下，写代码前请务必记住：

- **BaseURL**：`/api/v1`（开发环境由 Vite 代理转发，生产由 Nginx 反代）
- **鉴权**：登录成功后返回 `token`，后续请求在 Header 中携带 `Authorization: Bearer <token>`
- **统一响应体**：

```json
{ "code": 0, "message": "ok", "data": { } }
```

  `code === 0` 表示业务成功；`code === 40100` 表示未认证（前端跳登录）；其他非 0 值前端统一弹错误提示。
- **统一分页响应**（`data` 内）：

```json
{ "list": [], "total": 137, "page": 1, "pageSize": 20 }
```

- **分页请求参数**：`page`（从 1 开始）、`pageSize`，外加业务过滤参数。
- **WebSocket**：`ws://host/ws/v1/alarm`，服务端推送 JSON：

```json
{ "type": "ANOMALY_CREATED", "payload": { "event": { /* AnomalyEvent */ } } }
```

> **没有后端也能学**：第 4 章提供 Vite 本地 Mock 方案（自定义 Vite 插件，零额外依赖），所有接口先跑通 Mock，第 15 章再切换真实后端联调。

### 1.5 阅读方法约定

- 每个步骤末尾有 **【验证】** 小节：告诉你此时打开浏览器应看到什么，看不到就回查上一步。
- 代码块首行标注文件路径，如 `// src/api/request.ts`，请严格按路径创建文件。
- 教程代码均为完整代码（含 import 与关键样式），可直接复制运行。

---

## 2. 环境与初始化

### 2.1 环境要求

| 工具 | 要求 | 检查命令 |
|---|---|---|
| Node.js | 20+（LTS） | `node -v` |
| pnpm | 8+（推荐，也可用 npm/yarn） | `pnpm -v` |
| VS Code | 最新版 + 以下插件 | — |
| 浏览器 | Chrome/Edge + React DevTools 扩展 | — |

VS Code 必装插件：`ESLint`、`Prettier - Code formatter`、`Error Lens`。

若未安装 pnpm：

```bash
npm install -g pnpm
```

### 2.2 用 create-vite 创建项目

```bash
cd d:/办公/workBuddyWorkspace
pnpm create vite industryagent-web --template react-ts
cd industryagent-web
pnpm install
pnpm dev
```

**【验证】** 终端输出 Vite 就绪信息，浏览器访问 `http://localhost:5173`，看到 Vite + React 默认欢迎页（一个转动的 Logo）。按 `Ctrl+C` 停止。

### 2.3 安装核心依赖

```bash
# 运行时依赖
pnpm add react-router-dom antd @ant-design/icons zustand @tanstack/react-query axios echarts dayjs
# 开发依赖
pnpm add -D @types/node prettier eslint @eslint/js typescript-eslint eslint-plugin-react-hooks eslint-plugin-react-refresh
```

### 2.4 目录结构规划

先删除脚手架演示文件：`src/App.css`、`src/assets/react.svg`、`src/App.tsx` 中默认内容（第 5 章重写）。

目标目录结构（本教程将逐个建出所有文件夹）：

```text
industryagent-web/
├── .env.development          # 开发环境变量
├── .env.production           # 生产环境变量
├── .editorconfig
├── .eslintrc.cjs
├── .prettierrc
├── index.html
├── nginx.conf                # 生产部署配置（第15章）
├── Dockerfile                # 生产镜像（第15章）
├── vite.config.ts
├── tsconfig.json
├── tsconfig.node.json
└── src/
    ├── main.tsx              # 应用入口
    ├── App.tsx               # 根组件：Provider 装配
    ├── vite-env.d.ts         # Vite 环境类型声明
    ├── api/                  # 请求层：axios 实例 + 各业务 api 模块
    │   ├── request.ts
    │   ├── auth.ts
    │   ├── dashboard.ts
    │   ├── anomaly.ts
    │   ├── rootcause.ts
    │   └── approval.ts
    ├── components/           # 通用业务组件
    │   ├── AuthButton.tsx
    │   ├── PageContainer.tsx
    │   ├── StatCard.tsx
    │   └── hooks/
    │       ├── useChart.ts   # ECharts 封装 hook
    │       └── useWebSocket.ts
    ├── hooks/
    │   └── usePermission.ts
    ├── layouts/
    │   └── MainLayout.tsx    # 主布局
    ├── mock/                 # 本地 Mock（第4章）
    │   ├── plugin.ts
    │   └── data.ts
    ├── pages/                # 页面（按目录分组）
    │   ├── Login/index.tsx
    │   ├── Dashboard/index.tsx
    │   ├── Anomaly/index.tsx
    │   ├── RootCause/index.tsx
    │   ├── Approval/index.tsx
    │   └── ApprovalHistory/index.tsx
    ├── router/
    │   ├── index.tsx         # 路由表配置
    │   └── AuthRoute.tsx     # 路由守卫
    ├── store/                # Zustand store
    │   ├── authStore.ts
    │   ├── uiStore.ts
    │   └── dictStore.ts
    ├── styles/
    │   └── global.css
    ├── types/                # TS 类型定义
    │   ├── api.d.ts
    │   ├── auth.ts
    │   ├── anomaly.ts
    │   ├── rootcause.ts
    │   ├── approval.ts
    │   └── router.d.ts
    └── utils/
        ├── constants.ts      # 颜色映射、枚举常量
        └── format.ts        # 格式化工具
```

**划分原则**（记住三条就够）：

1. `pages/` 只放页面级组件，页面私有子组件放页面目录内；
2. 跨两个以上页面复用的组件进 `components/`；
3. 任何"请求后端"的代码只允许出现在 `api/` 目录，页面里不直接写 `axios`。

### 2.5 vite.config.ts 完整配置

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'
import { viteMockPlugin } from './src/mock/plugin'

// 判断是否走本地 Mock：true 时不把 /api 请求转发出去
const USE_MOCK = process.env.USE_MOCK !== 'false'

export default defineConfig({
  plugins: [react(), ...(USE_MOCK ? [viteMockPlugin()] : [])],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },
  server: {
    port: 5173,
    open: false,
    proxy: {
      // 联调真实后端时 USE_MOCK=false，请求被代理到 Egg.js（默认 7001 端口）
      '/api/v1': {
        target: process.env.API_TARGET || 'http://localhost:7001',
        changeOrigin: true,
      },
      // WebSocket 代理（ws 协议）
      '/ws/v1': {
        target: process.env.API_TARGET || 'http://localhost:7001',
        ws: true,
        changeOrigin: true,
      },
    },
  },
  build: {
    outDir: 'dist',
    sourcemap: false,
    chunkSizeWarningLimit: 1500,
    rollupOptions: {
      output: {
        // 手动分包：框架 / UI库 / 图表各自独立，浏览器可长效缓存
        manualChunks: {
          react: ['react', 'react-dom', 'react-router-dom'],
          antd: ['antd', '@ant-design/icons'],
          echarts: ['echarts'],
        },
      },
    },
  },
})
```

说明三个关键点：

- **alias `@`**：之后 `import x from '@/utils/format'` 等价于 `src/utils/format`，避免 `../../..` 地狱；
- **proxy**：开发环境浏览器请求 `/api/v1/...`，Vite 收到后转发给后端，天然规避 CORS（浏览器视角是同源请求）；
- **manualChunks**：把不常变的依赖拆成独立 chunk，业务代码发版后用户仍可命中浏览器缓存。

### 2.6 ESLint + Prettier + EditorConfig

```json
// .prettierrc
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2
}
```

```js
// .eslintrc.cjs
module.exports = {
  root: true,
  env: { browser: true, es2022: true, node: true },
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:react-hooks/recommended',
  ],
  ignorePatterns: ['dist', 'node_modules', '*.cjs'],
  parser: '@typescript-eslint/parser',
  plugins: ['@typescript-eslint', 'react-refresh'],
  rules: {
    'react-refresh/only-export-components': 'off',
    '@typescript-eslint/no-explicit-any': 'warn',
  },
}
```

```ini
# .editorconfig
root = true

[*]
charset = utf-8
indent_style = space
indent_size = 2
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
```

**【验证】** 在 `src/main.tsx` 里随便加一行 `const a: number = 'x'`，VS Code 应立刻红波浪线报类型错误；删除后保存时 Prettier 自动格式化（单引号、无分号）。

---

## 3. 工程基础设施

### 3.1 TypeScript 严格配置详解

脚手架自带 `tsconfig.json`，我们替换为如下内容并逐项解释：

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "skipLibCheck": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

关键项逐条理解（面试常问）：

| 配置 | 作用 | 为什么企业项目要开 |
|---|---|---|
| `strict: true` | 开启全家桶：严格 null 检查、隐式 any 报错 | 把大量运行时错误前移到编译期 |
| `noUnusedLocals/Parameters` | 未使用变量报错 | 防止死代码堆积 |
| `moduleResolution: bundler` | 按打包器规则解析模块 | Vite 场景下无需写扩展名、支持 exports 字段 |
| `paths` + `baseUrl` | `@/` 别名 | **必须与 vite.config 的 alias 同时配置**，否则 TS 认识但 Vite 不认识（或反之） |
| `isolatedModules` | 每个文件独立编译 | Vite 逐文件转译的前提，禁止 `export type x` 之外的跨文件类型直传写法 |

补一个让 TS 认识 `.env` 变量的声明文件：

```ts
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string
  readonly VITE_WS_URL: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

### 3.2 环境变量文件

```bash
# .env.development
VITE_API_BASE_URL=/api/v1
VITE_WS_URL=/ws/v1/alarm
```

```bash
# .env.production
VITE_API_BASE_URL=/api/v1
VITE_WS_URL=/ws/v1/alarm
```

> 注意：Vite 只暴露 `VITE_` 前开头的变量给客户端；`USE_MOCK` 没有 `VITE_` 前缀，只在 Node 侧（vite.config）可见，不会泄漏进产物。

### 3.3 AntD 主题定制与暗色/亮色切换

AntD 5 用 `ConfigProvider` 的 `theme` 属性做运行时主题。先建立 UI 状态仓库（ Zustand 的第一个 store，第 13 章会扩展）：

```ts
// src/store/uiStore.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

type ThemeMode = 'light' | 'dark'

interface UIState {
  themeMode: ThemeMode
  collapsed: boolean // 侧边菜单折叠状态
  toggleTheme: () => void
  toggleCollapsed: () => void
}

export const useUIStore = create<UIState>()(
  persist(
    (set) => ({
      themeMode: 'light',
      collapsed: false,
      toggleTheme: () =>
        set((s) => ({ themeMode: s.themeMode === 'light' ? 'dark' : 'light' })),
      toggleCollapsed: () => set((s) => ({ collapsed: !s.collapsed })),
    }),
    { name: 'ia-ui' }, // localStorage key
  ),
)
```

再写一个把 AntD 算法转换成主题配置的小工具：

```ts
// src/utils/theme.ts
import type { ThemeConfig } from 'antd'
import { theme as antdTheme } from 'antd'

export function getAntdTheme(mode: 'light' | 'dark'): ThemeConfig {
  const isDark = mode === 'dark'
  return {
    algorithm: isDark ? antdTheme.darkAlgorithm : antdTheme.defaultAlgorithm,
    token: {
      // 主色：工业蓝，与企业 VI 一致
      colorPrimary: '#1668dc',
      borderRadius: 4,
      fontSize: 14,
    },
    components: {
      // Table 斑马纹更易读（半导体数据密集场景）
      Table: { cellPaddingBlockSM: 6 },
      Layout: { headerHeight: 56 },
    },
  }
}
```

在 `App.tsx` 中接线（本节先最小化，第 5、7 章继续完善）：

```tsx
// src/App.tsx
import { ConfigProvider, App as AntdApp } from 'antd'
import zhCN from 'antd/locale/zh_CN'
import { useUIStore } from '@/store/uiStore'
import { getAntdTheme } from '@/utils/theme'

export default function App() {
  const themeMode = useUIStore((s) => s.themeMode)
  return (
    <ConfigProvider locale={zhCN} theme={getAntdTheme(themeMode)}>
      <AntdApp>{/* 路由出口，第 7 章填充 */}</AntdApp>
    </ConfigProvider>
  )
}
```

> `AntdApp` 是 AntD 5 的"应用级消息上下文"：`message/notification/modal` 的静态方法在主题令牌体系下会有警告，包一层后我们通过 `App.useApp()` 获取实例化 API，更符合 React 语境。

**【验证】** 暂时把 `toggleTheme` 绑到一个临时按钮上，点击后页面整体变暗色且主色变蓝；也可打开 DevTools → Application → Local Storage，看到 `ia-ui` 键。

### 3.4 全局样式与 CSS Modules 规范

规范（团队约定，直接照抄即可）：

1. **全局样式只放"_reset、滚动条、AntD 无法覆盖的少量补丁"**，其余全部用 CSS Modules（`xxx.module.css`）；
2. AntD 5 的样式由 token 驱动，优先改 token，不写 `!important` 硬覆盖；
3. 类名使用 camelCase，页面级根类名固定为 `.container`。

```css
/* src/styles/global.css */
* { box-sizing: border-box; }

html, body, #root { height: 100%; margin: 0; }

body {
  font-family: -apple-system, 'Segoe UI', 'PingFang SC', 'Microsoft YaHei', sans-serif;
  -webkit-font-smoothing: antialiased;
}

/* 工业大屏常用：细滚动条 */
::-webkit-scrollbar { width: 6px; height: 6px; }
::-webkit-scrollbar-thumb { background: rgba(0, 0, 0, 0.2); border-radius: 3px; }
```

在 `main.tsx` 引入：

```tsx
// src/main.tsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import '@/styles/global.css'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

---

## 4. 请求层封装：Axios 统一接入层

### 4.1 统一类型定义（先定契约，再写代码）

企业级顺序是"先接口文档后代码"。我们用 TS 类型把接口约定固化：

```ts
// src/types/api.d.ts
/** 统一响应体 */
export interface ApiResponse<T = unknown> {
  code: number
  message: string
  data: T
}

/** 统一分页响应 */
export interface PageResult<T> {
  list: T[]
  total: number
  page: number
  pageSize: number
}

/** 分页查询参数基类 */
export interface PageQuery {
  page: number
  pageSize: number
}

/** 业务错误码 */
export const BizErrorCode = {
  OK: 0,
  UNAUTHORIZED: 40100,
  FORBIDDEN: 40300,
} as const
```

业务领域类型（对照 IA-TU-001 后端章节的字段）：

```ts
// src/types/auth.ts
export type RoleCode = 'admin' | 'process_engineer' | 'equipment_engineer'

export interface UserInfo {
  id: number
  username: string
  realName: string
  role: RoleCode
  /** 权限点列表，如 ['approval:approve','rootcause:reanalyze'] */
  permissions: string[]
}

export interface LoginParams {
  username: string
  password: string
}

export interface LoginResult {
  token: string
  user: UserInfo
}
```

```ts
// src/types/anomaly.ts
import type { PageQuery } from './api'

/** 异常严重程度 */
export type Severity = 'critical' | 'high' | 'medium' | 'low'

/** 事件状态：待分析→分析中→待审批→已闭环 */
export type EventStatus = 'pending' | 'analyzing' | 'awaiting_approval' | 'closed'

export interface AnomalyEvent {
  id: number
  /** 设备编号，如 ETA-301 */
  equipmentCode: string
  /** 工序：刻蚀/薄膜/光刻/扩散... */
  processStep: string
  /** 产品批次 */
  lotNo: string
  title: string
  severity: Severity
  status: EventStatus
  /** 触发规则，如 "FDC: Chamber Temp 偏移 3σ" */
  triggerRule: string
  /** FDC 分数，0-100 */
  fdcScore: number
  createdAt: string
}

export interface AnomalyQueryParams extends PageQuery {
  keyword?: string
  equipmentCode?: string
  processStep?: string
  severity?: Severity
  status?: EventStatus
}
```

```ts
// src/types/rootcause.ts
/** 嫌疑参数项 */
export interface SuspectParam {
  name: string       // 参数名，如 chamber_temp
  value: number      // 当前值
  baseline: number   // 基线值
  unit: string      // 单位，如 ℃
  deviation: number // 偏移量 = value - baseline
}

/** Top-3 嫌疑之一 */
export interface RootCauseSuspect {
  rank: number
  /** 0~1，根因置信度 */
  confidence: number
  /** 嫌疑对象描述，如 "ETA-301 腔温漂移" */
  hypothesis: string
  /** 嫌疑参数列表 */
  params: SuspectParam[]
  /** 关联工序 */
  processStep: string
  /** 证据条目数（图谱节点/案例数） */
  evidenceCount: number
}

/** 证据链：参数时序 */
export interface ParamSeriesPoint {
  time: string
  value: number
  baseline: number
}

export interface ParamSeries {
  paramName: string
  unit: string
  points: ParamSeriesPoint[]
}

/** 证据链：相似历史案例 */
export interface SimilarCase {
  caseId: number
  occurredAt: string
  equipmentCode: string
  summary: string
  similarity: number // 0~1
  finalRootCause: string
}

/** 证据链：知识图谱节点/边 */
export interface GraphNode {
  id: string
  name: string
  category: 'event' | 'equipment' | 'param' | 'defect' | 'recipe'
  value?: number
}

export interface GraphEdge {
  source: string
  target: string
  relation: string
}

export interface KnowledgeGraph {
  nodes: GraphNode[]
  edges: GraphEdge[]
}

/** 根因分析详情 */
export interface RootCauseDetail {
  event: AnomalyEvent
  suspects: RootCauseSuspect[]
  paramSeries: ParamSeries[]
  knowledgeGraph: KnowledgeGraph
  similarCases: SimilarCase[]
  /** 因果分析结论文本 */
  causalConclusion: string
  /** 异步分析任务：重新分析用 */
  taskStatus: 'idle' | 'running' | 'done' | 'failed'
}
```

```ts
// src/types/approval.ts
export type ApprovalAction = 'approve' | 'reject'

export type ApprovalStatus = 'pending' | 'approved' | 'rejected'

export interface ApprovalTask {
  id: number
  /** 关联异常事件 */
  eventId: number
  eventTitle: string
  equipmentCode: string
  /** 智能体给出的调参建议文本 */
  suggestion: string
  /** 建议风险等级 */
  riskLevel: 'low' | 'medium' | 'high'
  status: ApprovalStatus
  createdAt: string
}

export interface ApprovalSubmitParams {
  taskId: number
  action: ApprovalAction
  /** 审批理由，必填 */
  reason: string
}

export interface ApprovalHistoryItem {
  id: number
  taskId: number
  action: ApprovalAction
  reason: string
  operator: string
  operatedAt: string
}
```

顺手把 `AnomalyEvent` 在 rootcause.ts 中的引用补上：

```ts
// src/types/rootcause.ts 文件顶部补 import
import type { AnomalyEvent } from './anomaly'
```

### 4.2 axios 实例：拦截器全流程

```ts
// src/api/request.ts
import axios, { AxiosError, type AxiosRequestConfig, type InternalAxiosRequestConfig } from 'axios'
import { message as staticMessage } from 'antd'
import type { ApiResponse } from '@/types/api'
import { useAuthStore } from '@/store/authStore'

/**
 * 免登录白名单：这些接口 401 时不应触发"跳登录死循环"
 */
const WHITE_LIST = ['/auth/login']

const instance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL, // '/api/v1'
  timeout: 15000,
})

/* ============ 请求拦截器：附 token ============ */
instance.interceptors.request.use((config: InternalAxiosRequestConfig) => {
  const token = useAuthStore.getState().token
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

/* ============ 响应拦截器：拆包 + 统一错误 ============ */
instance.interceptors.response.use(
  (response) => {
    const res = response.data as ApiResponse
    // 二进制流（如导出文件）直接放行
    if (response.config.responseType === 'blob') return response

    if (res.code !== 0) {
      // 业务错误：统一弹错，并抛出供调用方 catch
      staticMessage.error(res.message || '业务处理失败')
      return Promise.reject(new BizError(res.message, res.code))
    }
    // 成功：把 data 剥出来，调用方拿到的直接是业务数据
    return res.data as never
  },
  (error: AxiosError<ApiResponse>) => {
    handleHttpError(error)
    return Promise.reject(error)
  },
)

/** 业务错误类型：携带 code，方便调用方按码处理 */
export class BizError extends Error {
  constructor(message: string, public code: number) {
    super(message)
    this.name = 'BizError'
  }
}

function handleHttpError(error: AxiosError<ApiResponse>) {
  const status = error.response?.status
  const url = error.config?.url ?? ''
  const isWhitelisted = WHITE_LIST.some((p) => url.includes(p))

  if (status === 401) {
    staticMessage.error('登录已过期，请重新登录')
    useAuthStore.getState().logout()
    // 用 window 跳转而非 useNavigate：拦截器不在 React 上下文内
    if (!location.pathname.startsWith('/login')) {
      location.href = '/login'
    }
    return
  }
  if (status === 403 && !isWhitelisted) {
    staticMessage.error('没有访问权限')
    return
  }
  const fallback =
    status === undefined
      ? '网络异常，请检查连接'
      : status >= 500
        ? '服务开小差了，请稍后重试'
        : error.response?.data?.message || '请求失败'
  staticMessage.error(fallback)
}

/**
 * 对外暴露的通用请求函数。
 * 返回值是拦截器剥壳后的 data，直接是业务数据类型 T。
 */
export function request<T>(config: AxiosRequestConfig): Promise<T> {
  return instance.request<never, T>(config)
}

export const http = {
  get: <T>(url: string, params?: object) => request<T>({ url, method: 'GET', params }),
  post: <T>(url: string, data?: object) => request<T>({ url, method: 'POST', data }),
  put: <T>(url: string, data?: object) => request<T>({ url, method: 'PUT', data }),
  delete: <T>(url: string) => request<T>({ url, method: 'DELETE' }),
}
```

理解这段封装的 4 个关键设计：

1. **`instance.request<never, T>`**：Axios 的泛型第二位是"拦截器改造后的返回类型"，我们用 `never` 占位原 `R`，让 `T` 表示剥壳后的数据——这是社区标准的"响应拆包"技巧；
2. **`BizError`**：把业务错误码带出去，调用方可以 `catch (e) { if (e instanceof BizError && e.code === ...) }` 精细处理；
3. **401 处理**：清 token + `location.href` 跳登录。拦截器不在组件树内，拿不到 `useNavigate`，全量刷新跳转是可接受的兜底（也可用第 13 章的路由事件桥接优化）；
4. **白名单**：登录接口 401 时提示"用户名或密码错误"而非"跳登录"，避免自我跳转。

### 4.3 authStore：token 持久化（请求层依赖它）

```ts
// src/store/authStore.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import type { LoginResult, UserInfo } from '@/types/auth'

interface AuthState {
  token: string | null
  user: UserInfo | null
  /** 登录态初始化是否完成（persist 恢复是同步的，此标记留给后续扩展） */
  setLogin: (r: LoginResult) => void
  logout: () => void
  hasPermission: (perm: string) => boolean
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      token: null,
      user: null,
      setLogin: (r) => set({ token: r.token, user: r.user }),
      logout: () => set({ token: null, user: null }),
      hasPermission: (perm) => {
        const user = get().user
        if (!user) return false
        // admin 拥有全部权限
        if (user.role === 'admin') return true
        return user.permissions.includes(perm)
      },
    }),
    {
      name: 'ia-auth',
      // 只持久化 token 与 user，函数与派生态不落盘
      partialize: (s) => ({ token: s.token, user: s.user }),
    },
  ),
)
```

> **zustand persist 原理**：中间件在 set 时把 `partialize` 结果 JSON 序列化写入 `localStorage`，创建 store 时同步读回并合并——刷新页面登录态不丢，这正是企业应用"免二次登录"的实现。

### 4.4 Mock 方案：Vite 本地 Mock 插件

思路：写一个 Vite 插件，在 dev server 收到 `/api/v1` 请求时直接返回手工数据，**不向下转发**。零额外依赖，联调时 `USE_MOCK=false` 关闭即可。

先造一批贴合业务的假数据：

```ts
// src/mock/data.ts
import type { AnomalyEvent, RootCauseDetail, ApprovalTask } from '@/types/rootcause'
import type { AnomalyEvent as AE } from '@/types/anomaly'

export const mockEvents: AE[] = [
  {
    id: 1001,
    equipmentCode: 'ETA-301',
    processStep: '刻蚀',
    lotNo: 'LOT-A2026',
    title: 'ETA-301 腔体温度漂移导致良率下降',
    severity: 'critical',
    status: 'awaiting_approval',
    triggerRule: 'FDC: Chamber Temp 偏移 3σ 持续 15min',
    fdcScore: 88,
    createdAt: '2026-10-15 09:12:00',
  },
  {
    id: 1002,
    equipmentCode: 'CVD-105',
    processStep: '薄膜',
    lotNo: 'LOT-B1188',
    title: 'CVD-105 压力异常尖峰',
    severity: 'high',
    status: 'analyzing',
    triggerRule: 'FDC: Pressure spike > 5σ',
    fdcScore: 76,
    createdAt: '2026-10-15 10:03:00',
  },
  {
    id: 1003,
    equipmentCode: 'PHO-220',
    processStep: '光刻',
    lotNo: 'LOT-C0042',
    title: 'PHO-220 对准偏差轻微超限',
    severity: 'medium',
    status: 'pending',
    triggerRule: 'SPC: Overlay Cpk < 1.33',
    fdcScore: 61,
    createdAt: '2026-10-15 10:40:00',
  },
]

export const mockApprovals: ApprovalTask[] = [
  {
    id: 5001,
    eventId: 1001,
    eventTitle: 'ETA-301 腔体温度漂移导致良率下降',
    equipmentCode: 'ETA-301',
    suggestion: '建议将 ETA-301 Chamber Temp 设定值由 85.0℃ 下调至 83.5℃，并执行 30min 稳定性验证。',
    riskLevel: 'medium',
    status: 'pending',
    createdAt: '2026-10-15 09:30:00',
  },
]

/** 良率趋势（14 天） */
export const mockYieldTrend = Array.from({ length: 14 }, (_, i) => ({
  date: `10-${String(i + 2).padStart(2, '0')}`,
  yieldRate: +(93 + Math.sin(i / 2) * 1.2 - (i === 13 ? 1.8 : 0)).toFixed(2),
  target: 95,
}))

/** 生成参数时序：正弦叠加噪声 */
function genSeries(baseline: number, amp: number, drift = 0) {
  return Array.from({ length: 48 }, (_, i) => ({
    time: `${String(8 + Math.floor(i / 6)).padStart(2, '0')}:${String((i % 6) * 10).padStart(2, '0')}`,
    value: +(baseline + drift * (i / 48) + Math.sin(i / 4) * amp + (Math.random() - 0.5)).toFixed(2),
    baseline,
  }))
}

export const mockRootCauseDetail = (eventId: number): RootCauseDetail => ({
  event: mockEvents.find((e) => e.id === eventId) ?? mockEvents[0],
  suspects: [
    {
      rank: 1,
      confidence: 0.86,
      hypothesis: 'ETA-301 腔体加热器老化导致温控漂移',
      params: [
        { name: 'chamber_temp', value: 87.2, baseline: 85, unit: '℃', deviation: 2.2 },
        { name: 'heater_current', value: 12.4, baseline: 11.8, unit: 'A', deviation: 0.6 },
      ],
      processStep: '刻蚀',
      evidenceCount: 6,
    },
    {
      rank: 2,
      confidence: 0.62,
      hypothesis: 'RF 匹配网络阻抗漂移引入刻蚀速率波动',
      params: [
        { name: 'rf_forward_power', value: 1512, baseline: 1500, unit: 'W', deviation: 12 },
      ],
      processStep: '刻蚀',
      evidenceCount: 4,
    },
    {
      rank: 3,
      confidence: 0.41,
      hypothesis: '特气流量控制器（MFC）标定偏差',
      params: [{ name: 'cf4_flow', value: 203, baseline: 200, unit: 'sccm', deviation: 3 }],
      processStep: '刻蚀',
      evidenceCount: 2,
    },
  ],
  paramSeries: [
    { paramName: 'chamber_temp', unit: '℃', points: genSeries(85, 0.3, 2.4) },
    { paramName: 'heater_current', unit: 'A', points: genSeries(11.8, 0.2, 0.5) },
  ],
  knowledgeGraph: {
    nodes: [
      { id: 'e1001', name: '异常事件-1001', category: 'event' },
      { id: 'ETA-301', name: '刻蚀机 ETA-301', category: 'equipment' },
      { id: 'chamber_temp', name: '腔体温度', category: 'param', value: 87 },
      { id: 'etch_rate', name: '刻蚀速率', category: 'param' },
      { id: 'D-2101', name: '缺陷模式 D-2101', category: 'defect' },
      { id: 'R-ETA7', name: 'Recipe R-ETA7', category: 'recipe' },
    ],
    edges: [
      { source: 'e1001', target: 'ETA-301', relation: '发生于' },
      { source: 'ETA-301', target: 'chamber_temp', relation: '监测参数' },
      { source: 'chamber_temp', target: 'etch_rate', relation: '因果影响' },
      { source: 'etch_rate', target: 'D-2101', relation: '导致缺陷' },
      { source: 'R-ETA7', target: 'chamber_temp', relation: '设定参数' },
    ],
  },
  similarCases: [
    {
      caseId: 20250812,
      occurredAt: '2025-08-12',
      equipmentCode: 'ETA-208',
      summary: '同型号刻蚀机腔温漂移，更换加热器后恢复',
      similarity: 0.83,
      finalRootCause: '加热器老化（更换周期超期 4 个月）',
    },
    {
      caseId: 20250305,
      occurredAt: '2025-03-05',
      equipmentCode: 'ETA-112',
      summary: '温控热电偶接触不良致温度读数偏差',
      similarity: 0.71,
      finalRootCause: '热电偶接线氧化',
    },
  ],
  causalConclusion:
    '基于 48h 时序的格兰杰因果检验显示，chamber_temp 对 etch_rate 存在显著因果效应（p<0.01，滞后 2 个采样点）；'
    + '结合知识图谱与历史案例，判定"加热器老化→腔温漂移→刻蚀速率波动→缺陷 D-2101 良率损失 2.1%"为最可能因果链。',
  taskStatus: 'done',
}) satisfies RootCauseDetail
```

然后是 Mock 插件本体——一个极简版"后端路由"：

```ts
// src/mock/plugin.ts
import type { Plugin } from 'vite'
import type { IncomingMessage, ServerResponse } from 'http'
import type { ApiResponse, PageResult } from '@/types/api'
import {
  mockEvents,
  mockApprovals,
  mockYieldTrend,
  mockRootCauseDetail,
} from './data'
import type { LoginParams } from '@/types/auth'
import type { AnomalyQueryParams } from '@/types/anomaly'
import type { ApprovalSubmitParams } from '@/types/approval'

/** 读取 JSON body 的辅助函数 */
function readBody(req: IncomingMessage): Promise<Record<string, unknown>> {
  return new Promise((resolve) => {
    let raw = ''
    req.on('data', (chunk) => (raw += chunk))
    req.on('end', () => {
      try { resolve(raw ? JSON.parse(raw) : {}) } catch { resolve({}) }
    })
  })
}

function send<T>(res: ServerResponse, data: T, code = 0) {
  const body = JSON.stringify({ code, message: code === 0 ? 'ok' : 'mock error', data } satisfies ApiResponse<T>)
  res.writeHead(200, { 'Content-Type': 'application/json; charset=utf-8' })
  res.end(body)
}

/** 极简路由表：path 前缀匹配 -> 处理器 */
function route(req: IncomingMessage, res: ServerResponse, pathname: string, query: URLSearchParams, body: Record<string, unknown>) {
  const M = req.method

  // ---- 登录 ----
  if (pathname === '/api/v1/auth/login' && M === 'POST') {
    const { username } = body as LoginParams
    const isAdmin = username === 'admin'
    return send(res, {
      token: `mock-token-${isAdmin ? 'admin' : 'user'}-${Date.now()}`,
      user: {
        id: 1,
        username,
        realName: isAdmin ? '系统管理员' : '王工艺',
        role: isAdmin ? 'admin' : 'process_engineer',
        permissions: isAdmin ? [] : ['approval:approve', 'rootcause:reanalyze', 'anomaly:view'],
      },
    })
  }

  // ---- 看板 ----
  if (pathname === '/api/v1/dashboard/overview' && M === 'GET') {
    return send(res, {
      yieldRate: 93.2,          // 当前综合良率 %
      yieldTarget: 95,
      oee: 78.5,                // 设备综合效率 %
      anomalyCount24h: 6,       // 24h 异常数
      pendingApprovalCount: 2,  // 待审批数
    })
  }
  if (pathname === '/api/v1/dashboard/yield-trend' && M === 'GET') {
    return send(res, mockYieldTrend)
  }
  if (pathname === '/api/v1/dashboard/defect-category' && M === 'GET') {
    return send(res, [
      { name: '颗粒污染', value: 35 },
      { name: '刻蚀残留', value: 24 },
      { name: '膜厚不均', value: 18 },
      { name: '对准偏差', value: 12 },
      { name: '其他', value: 11 },
    ])
  }
  if (pathname === '/api/v1/dashboard/top-equipment' && M === 'GET') {
    return send(res, [
      { equipmentCode: 'ETA-301', count: 12 },
      { equipmentCode: 'CVD-105', count: 9 },
      { equipmentCode: 'PHO-220', count: 7 },
      { equipmentCode: 'IMP-018', count: 4 },
      { equipmentCode: 'CMP-410', count: 3 },
    ])
  }

  // ---- 异常事件：分页列表 ----
  if (pathname === '/api/v1/anomaly/events' && M === 'GET') {
    const q = Object.fromEntries(query) as unknown as AnomalyQueryParams
    let list = [...mockEvents]
    if (q.keyword) list = list.filter((e) => e.title.includes(q.keyword!))
    if (q.severity) list = list.filter((e) => e.severity === q.severity)
    if (q.status) list = list.filter((e) => e.status === q.status)
    if (q.equipmentCode) list = list.filter((e) => e.equipmentCode === q.equipmentCode)
    const page = q.page ?? 1
    const pageSize = q.pageSize ?? 20
    const pageResult: PageResult<(typeof mockEvents)[number]> = {
      list: list.slice((page - 1) * pageSize, page * pageSize),
      total: list.length,
      page,
      pageSize,
    }
    return send(res, pageResult)
  }

  // ---- 根因分析详情 ----
  if (pathname.startsWith('/api/v1/rootcause/') && M === 'GET') {
    const id = Number(pathname.split('/').pop())
    return send(res, mockRootCauseDetail(id))
  }
  // 重新分析：创建任务
  if (/\/api\/v1\/rootcause\/\d+\/reanalyze$/.test(pathname) && M === 'POST') {
    return send(res, { taskId: `rc-${Date.now()}` })
  }
  // 重新分析：查询任务状态
  if (pathname.startsWith('/api/v1/rootcause/task/') && M === 'GET') {
    const elapsed = Date.now() % 10000
    return send(res, {
      taskId: pathname.split('/').pop(),
      status: elapsed < 4000 ? 'running' : 'done',
      progress: elapsed < 4000 ? Math.round(elapsed / 40) : 100,
    })
  }

  // ---- 审批 ----
  if (pathname === '/api/v1/approval/tasks' && M === 'GET') {
    const status = query.get('status') ?? 'pending'
    const page = Number(query.get('page') ?? 1)
    const pageSize = Number(query.get('pageSize') ?? 20)
    const list = mockApprovals.filter((t) => (status === 'all' ? true : t.status === status))
    return send(res, { list: list.slice((page - 1) * pageSize, page * pageSize), total: list.length, page, pageSize })
  }
  if (/\/api\/v1\/approval\/tasks\/\d+$/.test(pathname) && M === 'POST') {
    const { action, reason } = body as ApprovalSubmitParams
    if (!reason || reason.trim().length < 5) {
      return send(res, null, 40001) // 模拟"理由必填"业务错误
    }
    const id = Number(pathname.split('/').pop())
    const task = mockApprovals.find((t) => t.id === id)
    if (task) task.status = action === 'approve' ? 'approved' : 'rejected'
    return send(res, { ok: true })
  }
  if (pathname === '/api/v1/approval/history' && M === 'GET') {
    return send(res, [
      {
        id: 9001,
        taskId: 5001,
        action: 'approve',
        reason: '同意下调腔温设定值，先跑 3 批验证。',
        operator: '李工',
        operatedAt: '2026-10-15 11:00:00',
      },
      {
        id: 9002,
        taskId: 4998,
        action: 'reject',
        reason: '风险较高，需先做 DOE 实验确认窗口。',
        operator: '张工',
        operatedAt: '2026-10-14 16:20:00',
      },
    ])
  }

  res.writeHead(404, { 'Content-Type': 'application/json' })
  res.end(JSON.stringify({ code: 40400, message: `mock 未实现: ${M} ${pathname}`, data: null }))
}

export function viteMockPlugin(): Plugin {
  return {
    name: 'vite-plugin-ia-mock',
    configureServer(server) {
      // 返回 true 表示"已处理"，Vite 不再继续后续中间件
      server.middlewares.use((req, res, next) => {
        if (!req.url?.startsWith('/api/v1/')) return next()
        const url = new URL(req.url, 'http://localhost')
        readBody(req).then((body) => route(req, res, url.pathname, url.searchParams, body))
        return undefined // 已异步处理
      })
    },
    // 预览模式（vite preview）也支持 mock，便于构建后自测
    configurePreviewServer(server) {
      server.middlewares.use((req, res, next) => {
        if (!req.url?.startsWith('/api/v1/')) return next()
        const url = new URL(req.url, 'http://localhost')
        readBody(req).then((body) => route(req, res, url.pathname, url.searchParams, body))
        return undefined
      })
    },
  }
}
```

### 4.5 api 模块化组织

规则：一个后端资源一个文件，函数名 = 动词 + 资源，返回值带类型。

```ts
// src/api/auth.ts
import { http } from './request'
import type { LoginParams, LoginResult } from '@/types/auth'

export const login = (data: LoginParams) => http.post<LoginResult>('/auth/login', data)
```

```ts
// src/api/dashboard.ts
import { http } from './request'

export interface DashboardOverview {
  yieldRate: number
  yieldTarget: number
  oee: number
  anomalyCount24h: number
  pendingApprovalCount: number
}

export interface YieldTrendPoint { date: string; yieldRate: number; target: number }
export interface DefectCategoryItem { name: string; value: number }
export interface TopEquipmentItem { equipmentCode: string; count: number }

export const getOverview = () => http.get<DashboardOverview>('/dashboard/overview')
export const getYieldTrend = () => http.get<YieldTrendPoint[]>('/dashboard/yield-trend')
export const getDefectCategory = () => http.get<DefectCategoryItem[]>('/dashboard/defect-category')
export const getTopEquipment = () => http.get<TopEquipmentItem[]>('/dashboard/top-equipment')
```

```ts
// src/api/anomaly.ts
import { http } from './request'
import type { AnomalyEvent, AnomalyQueryParams } from '@/types/anomaly'
import type { PageResult } from '@/types/api'

export const getAnomalyEvents = (params: AnomalyQueryParams) =>
  http.get<PageResult<AnomalyEvent>>('/anomaly/events', params)
```

```ts
// src/api/rootcause.ts
import { http } from './request'
import type { RootCauseDetail } from '@/types/rootcause'

export const getRootCauseDetail = (eventId: number) =>
  http.get<RootCauseDetail>(`/rootcause/${eventId}`)

export interface ReanalyzeTask { taskId: string }

export const createReanalyze = (eventId: number) =>
  http.post<ReanalyzeTask>(`/rootcause/${eventId}/reanalyze`)

export interface TaskStatusResult {
  taskId: string
  status: 'running' | 'done' | 'failed'
  progress: number
}

export const getReanalyzeStatus = (taskId: string) =>
  http.get<TaskStatusResult>(`/rootcause/task/${taskId}`)
```

```ts
// src/api/approval.ts
import { http } from './request'
import type { ApprovalTask, ApprovalSubmitParams, ApprovalHistoryItem } from '@/types/approval'
import type { PageResult } from '@/types/api'

export const getApprovalTasks = (params: { page: number; pageSize: number; status: string }) =>
  http.get<PageResult<ApprovalTask>>('/approval/tasks', params)

export const submitApproval = (data: ApprovalSubmitParams) =>
  http.post<{ ok: boolean }>(`/approval/tasks/${data.taskId}`, { action: data.action, reason: data.reason })

export const getApprovalHistory = () =>
  http.get<ApprovalHistoryItem[]>('/approval/history')
```

**【验证】** 在 `src/App.tsx` 临时插入如下代码块，浏览器应弹出包含 token 的数据：

```tsx
import { useEffect } from 'react'
import { login } from '@/api/auth'

// App 组件内临时添加：
useEffect(() => {
  login({ username: 'admin', password: '123456' }).then((r) => console.log('登录成功', r))
}, [])
```

打开 DevTools Console：输出 `登录成功 {token: 'mock-token-admin-...', user: {...}}`；Network 面板能看到 `/api/v1/auth/login` 请求，且响应体是 `{code:0,...}` 包装。验证后删除临时代码。

---

## 5. 登录页与鉴权

### 5.1 路由守卫 AuthRoute

在写登录页之前先把"门卫"准备好——所有需登录页面都包在 `AuthRoute` 里：

```tsx
// src/router/AuthRoute.tsx
import { Navigate, Outlet, useLocation } from 'react-router-dom'
import { useAuthStore } from '@/store/authStore'

/**
 * 路由守卫：
 * - 无 token -> 跳 /login，并记录来源路由（登录后回跳）
 * - 有 token -> 渲染子路由（Outlet）
 */
export default function AuthRoute() {
  const token = useAuthStore((s) => s.token)
  const location = useLocation()

  if (!token) {
    return <Navigate to="/login" replace state={{ from: location.pathname }} />
  }
  return <Outlet />
}
```

三个细节：

1. 用 `Outlet` 而不是 `children`：配合 6.x 嵌套路由，`AuthRoute` 自身成为布局层，子路由在内部渲染；
2. `replace`：登录跳转不产生历史记录，用户按后退键不会回到需要登录的页面；
3. `state.from`：记住"想去的页面"，登录成功后跳回。

### 5.2 登录页组件（完整代码）

```tsx
// src/pages/Login/index.tsx
import { useEffect } from 'react'
import { useNavigate, useLocation } from 'react-router-dom'
import { Button, Card, Form, Input, App, Typography } from 'antd'
import { LockOutlined, UserOutlined } from '@ant-design/icons'
import { useMutation } from '@tanstack/react-query'
import { login } from '@/api/auth'
import { useAuthStore } from '@/store/authStore'
import styles from './index.module.css'

interface LoginFormValues {
  username: string
  password: string
}

export default function Login() {
  const navigate = useNavigate()
  const location = useLocation()
  const { message } = App.useApp()
  const setLogin = useAuthStore((s) => s.setLogin)
  const [form] = Form.useForm<LoginFormValues>()

  const loginMutation = useMutation({
    mutationFn: (values: LoginFormValues) => login(values),
    onSuccess: (result) => {
      setLogin(result)
      message.success(`欢迎回来，${result.user.realName}`)
      // 回跳来源页；没有来源则进看板
      const from = (location.state as { from?: string } | null)?.from ?? '/dashboard'
      navigate(from, { replace: true })
    },
  })

  const onFinish = (values: LoginFormValues) => {
    loginMutation.mutate(values)
  }

  // 已登录者访问 /login 时直接进看板
  const token = useAuthStore((s) => s.token)
  useEffect(() => {
    if (token) navigate('/dashboard', { replace: true })
  }, [token, navigate])

  return (
    <div className={styles.page}>
      <Card className={styles.card}>
        <div className={styles.header}>
          <Typography.Title level={3} style={{ marginBottom: 4 }}>
            芯联半导体 · 良率根因分析平台
          </Typography.Title>
          <Typography.Text type="secondary">
            IndustryAgent — 让良率损失 4 小时内见分晓
          </Typography.Text>
        </div>

        <Form
          form={form}
          layout="vertical"
          onFinish={onFinish}
          initialValues={{ username: 'admin', password: 'admin123' }}
          disabled={loginMutation.isPending}
        >
          <Form.Item
            name="username"
            rules={[{ required: true, message: '请输入用户名' }]}
          >
            <Input prefix={<UserOutlined />} placeholder="用户名（试试 admin / engineer）" size="large" />
          </Form.Item>
          <Form.Item
            name="password"
            rules={[
              { required: true, message: '请输入密码' },
              { min: 6, message: '密码至少 6 位' },
            ]}
          >
            <Input.Password prefix={<LockOutlined />} placeholder="密码" size="large" />
          </Form.Item>
          <Form.Item style={{ marginBottom: 8 }}>
            <Button
              type="primary"
              htmlType="submit"
              block
              size="large"
              loading={loginMutation.isPending}
            >
              登 录
            </Button>
          </Form.Item>
          <Typography.Text type="secondary" style={{ fontSize: 12 }}>
            演示账号：admin（管理员）/ engineer（工艺工程师，无 admin 权限）
          </Typography.Text>
        </Form>
      </Card>
    </div>
  )
}
```

```css
/* src/pages/Login/index.module.css */
.page {
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #0a1f44 0%, #1668dc 100%);
}

.card {
  width: 400px;
  border-radius: 8px;
  box-shadow: 0 12px 40px rgba(0, 21, 64, 0.35);
}

.header {
  text-align: center;
  margin-bottom: 24px;
}
```

注意 Mock 的登录逻辑：任何用户名都能登录，`admin` 拥有全部权限（`role: 'admin'`），其他用户名走 `process_engineer` 角色（用于第 12 章演示权限差异）。但 Mock 里 admin 初始密码校验没写，第一次先用 `admin`/任意 6 位以上密码登录。

为了让登录页的提示成立，回到 `src/mock/plugin.ts` 把默认密码值改为 `admin123`（若你改了 initialValues，保持一致即可）。

### 5.3 退出登录与登录态检查

退出功能放在顶栏（第 6 章主布局）里，核心逻辑：

```ts
// 退出：清 store（persist 自动清 localStorage）+ 跳登录
const logout = useAuthStore((s) => s.logout)
const handleLogout = () => {
  logout()
  navigate('/login', { replace: true })
}
```

同时前端需在 `main.tsx` 装配 React Query——它是第 8 章起的主角，现在先挂上：

```tsx
// src/main.tsx（最终版）
import React from 'react'
import ReactDOM from 'react-dom/client'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import App from './App'
import '@/styles/global.css'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,        // 30s 内相同 key 不重复请求
      retry: 2,                  // 网络类失败自动重试 2 次
      refetchOnWindowFocus: false, // 企业内网应用常关掉"聚焦刷新"
    },
  },
})

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </React.StrictMode>,
)
```

**【验证】** 此刻还没有路由，先做一个最小路由骨架临时验证登录流程（第 7 章会重构掉）：在 `App.tsx` 的 `AntdApp` 内临时放 `<Login />`，浏览器看到深蓝渐变背景的登录卡片；输入 admin / admin123 点击登录，Console 输出欢迎提示；Application → Local Storage 中出现 `ia-auth`，包含 token 与 user。验证后恢复 `App.tsx`。

---

## 6. 主布局：侧边菜单 + 顶栏 + 内容区

### 6.1 菜单配置：路由 meta 驱动

企业级布局的第一原则：**菜单不写死在 Layout 里，而是从路由表推导**——改路由自动改菜单，永远不失同步。

先定义带 meta 的路由类型：

```ts
// src/types/router.d.ts
import type { RouteObject } from 'react-router-dom'

/** 路由元信息 */
export interface RouteMeta {
  /** 页面标题（document.title、面包屑） */
  title: string
  /** 侧边菜单图标名（antd icon 组件在路由表中按名引用） */
  icon?: string
  /** 是否在侧边菜单中隐藏（如详情页） */
  hideInMenu?: boolean
  /** 访问所需权限点；不配则登录即可访问 */
  permission?: string
  /** 面包屑显示名（不配则取 title） */
  breadcrumb?: string
}

export interface AppRouteObject {
  path: string
  meta: RouteMeta
  /** 懒加载组件；布局路由可为空 */
  element?: React.LazyExoticComponent<React.ComponentType>
  children?: AppRouteObject[]
}
```

### 6.2 MainLayout 完整代码

```tsx
// src/layouts/MainLayout.tsx
import { useMemo } from 'react'
import { Outlet, useLocation, useNavigate } from 'react-router-dom'
import {
  Avatar, Breadcrumb, Button, Dropdown, Layout, Menu, Space, Switch, Tag, Tooltip, Typography,
} from 'antd'
import {
  BellOutlined, DashboardOutlined, FileSearchOutlined, LogoutOutlined,
  MoonOutlined, SunOutlined, AuditOutlined, UserOutlined, HistoryOutlined,
} from '@ant-design/icons'
import { useUIStore } from '@/store/uiStore'
import { useAuthStore } from '@/store/authStore'
import { MENU_ITEMS, findMenuByPath } from '@/router/menus'
import styles from './MainLayout.module.css'

const { Header, Sider, Content } = Layout

/** 图标名 -> 组件 的映射表（路由配置里只存字符串，序列化友好） */
export const ICON_MAP: Record<string, React.ReactNode> = {
  dashboard: <DashboardOutlined />,
  anomaly: <FileSearchOutlined />,
  approval: <AuditOutlined />,
  'approval-history': <HistoryOutlined />,
}

export default function MainLayout() {
  const location = useLocation()
  const navigate = useNavigate()
  const collapsed = useUIStore((s) => s.collapsed)
  const toggleCollapsed = useUIStore((s) => s.toggleCollapsed)
  const themeMode = useUIStore((s) => s.themeMode)
  const toggleTheme = useUIStore((s) => s.toggleTheme)
  const user = useAuthStore((s) => s.user)
  const logout = useAuthStore((s) => s.logout)

  // 根据当前路径计算：选中的菜单 key + 面包屑
  const { selectedKeys, breadcrumbItems } = useMemo(() => {
    const path = location.pathname
    const items = [{ title: '首页', path: '/dashboard' }, ...findMenuByPath(path)]
    return {
      selectedKeys: [items[items.length - 1].path],
      breadcrumbItems: items.map((i) => ({ title: i.title })),
    }
  }, [location.pathname])

  const menuItems = useMemo(() => MENU_ITEMS.map((item) => ({
    key: item.path,
    icon: ICON_MAP[item.meta.icon ?? ''],
    label: item.meta.title,
  })), [])

  return (
    <Layout className={styles.layout}>
      <Header className={styles.header}>
        <div className={styles.logo}>
          <Typography.Text strong style={{ color: '#fff', fontSize: 16 }}>
            {collapsed ? 'IA' : 'IndustryAgent'}
          </Typography.Text>
        </div>
        <Space size={16}>
          <Tooltip title={themeMode === 'light' ? '切换暗色' : '切换亮色'}>
            <Switch
              checkedChildren={<MoonOutlined />}
              unCheckedChildren={<SunOutlined />}
              checked={themeMode === 'dark'}
              onChange={toggleTheme}
            />
          </Tooltip>
          <Tooltip title="实时告警（WebSocket 第9章接入）">
            <Badge count={0} size="small">
              <BellOutlined style={{ color: '#fff', fontSize: 18 }} />
            </Badge>
          </Tooltip>
          <Dropdown
            menu={{
              items: [
                {
                  key: 'logout',
                  icon: <LogoutOutlined />,
                  label: '退出登录',
                  onClick: () => {
                    logout()
                    navigate('/login', { replace: true })
                  },
                },
              ],
            }}
          >
            <Space style={{ cursor: 'pointer' }}>
              <Avatar size={28} icon={<UserOutlined />} />
              <Typography.Text style={{ color: '#fff' }}>{user?.realName ?? '未知用户'}</Typography.Text>
              {user && <Tag color={user.role === 'admin' ? 'gold' : 'blue'}>{user.role}</Tag>}
            </Space>
          </Dropdown>
        </Space>
      </Header>

      <Layout>
        <Sider
          theme="light"
          collapsible
          collapsed={collapsed}
          onCollapse={toggleCollapsed}
          width={208}
        >
          <Menu
            mode="inline"
            items={menuItems}
            selectedKeys={selectedKeys}
            onClick={({ key }) => navigate(key)}
          />
        </Sider>

        <Content className={styles.content}>
          <Breadcrumb items={breadcrumbItems} className={styles.breadcrumb} />
          <Outlet />
        </Content>
      </Layout>
    </Layout>
  )
}
```

注意：上面 `Badge` 需要从 antd 导入（完整导入列表见下方修正），请把 antd 导入行改为：

```tsx
import {
  Avatar, Badge, Breadcrumb, Button, Dropdown, Layout, Menu, Space, Switch, Tag, Tooltip, Typography,
} from 'antd'
```

（`Button` 暂时未用，第 9、10 章会用到顶栏扩展位，可先移除。）

```css
/* src/layouts/MainLayout.module.css */
.layout {
  height: 100vh;
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #0a1f44;
  padding-inline: 16px;
}

.logo {
  min-width: 120px;
  letter-spacing: 1px;
}

.content {
  padding: 16px 24px;
  overflow: auto;
}

.breadcrumb {
  margin-bottom: 12px;
}
```

### 6.3 菜单配置与面包屑工具

把菜单从路由表独立出来吗？不——按"配置驱动"理念，菜单就是一级路由的投影。第 7 章定义完整路由表，这里先写菜单推导文件：

```ts
// src/router/menus.ts
import type { AppRouteObject } from '@/types/router'

/** 从路由表推导菜单：取所有一级路由，剔除 hideInMenu */
export const MENU_ITEMS: AppRouteObject[] = []

/** 根据完整路由表填充菜单（第7章调用） */
export function buildMenu(routes: AppRouteObject[]): AppRouteObject[] {
  const items = routes.filter((r) => !r.meta.hideInMenu)
  MENU_ITEMS.length = 0
  MENU_ITEMS.push(...items)
  return items
}

/** 按当前路径生成面包屑（支持详情页挂靠列表页） */
export function findMenuByPath(path: string): { title: string; path: string }[] {
  const crumbs: { title: string; path: string }[] = []
  // 示例：/anomaly/1001 -> [ {异常事件, /anomaly}, {根因分析, /anomaly/1001} ]
  if (path.startsWith('/anomaly/')) {
    crumbs.push({ title: '异常事件', path: '/anomaly' })
    crumbs.push({ title: '根因分析', path })
  } else {
    const hit = MENU_ITEMS.find((m) => m.path === path)
    if (hit) crumbs.push({ title: hit.meta.title, path: hit.path })
  }
  return crumbs
}
```

> 为什么面包屑单独处理详情页？因为详情页路由 `/anomaly/:id` 不在菜单里，但用户在详情页时面包屑仍应显示"首页 / 异常事件 / 根因分析"。这种"面包屑挂靠"是企业后台的常见需求。

**【验证】** 主布局要配合路由才能看到，直接进入第 7 章。

---

## 7. 路由与菜单：配置驱动

### 7.1 路由表配置（懒加载 + meta）

```tsx
// src/router/index.tsx
import { lazy } from 'react'
import { Navigate } from 'react-router-dom'
import type { AppRouteObject } from '@/types/router'
import AuthRoute from './AuthRoute'
import MainLayout from '@/layouts/MainLayout'
import { buildMenu } from './menus'

/* ---- 懒加载页面：分包 + 首屏加速 ---- */
const Login = lazy(() => import('@/pages/Login'))
const Dashboard = lazy(() => import('@/pages/Dashboard'))
const AnomalyList = lazy(() => import('@/pages/Anomaly'))
const RootCauseDetail = lazy(() => import('@/pages/RootCause'))
const Approval = lazy(() => import('@/pages/Approval'))
const ApprovalHistory = lazy(() => import('@/pages/ApprovalHistory'))

/* ---- 需要登录的一级路由（同时是菜单来源） ---- */
export const appRoutes: AppRouteObject[] = [
  {
    path: '/dashboard',
    meta: { title: '良率看板', icon: 'dashboard' },
    element: Dashboard,
  },
  {
    path: '/anomaly',
    meta: { title: '异常事件', icon: 'anomaly' },
    element: AnomalyList,
  },
  {
    path: '/anomaly/:id',
    meta: { title: '根因分析', hideInMenu: true },
    element: RootCauseDetail,
  },
  {
    path: '/approval',
    meta: { title: '审批工作台', icon: 'approval', permission: 'approval:view' },
    element: Approval,
  },
  {
    path: '/approval/history',
    meta: { title: '审批历史', icon: 'approval-history', permission: 'approval:view' },
    element: ApprovalHistory,
  },
]

/* 菜单 = 一级路由投影（剔除 hideInMenu），含权限过滤（第12章增强） */
buildMenu(appRoutes)

/* ---- 转换成 React Router 结构 ---- */
const routes = [
  { path: '/login', element: Login },
  {
    element: AuthRoute, // 守卫层：无 token 一律弹回 /login
    children: [
      {
        element: MainLayout, // 布局层：侧边菜单 + 顶栏
        children: [
          ...appRoutes.map((r) => ({
            path: r.path,
            element: r.element,
          })),
          { path: '/', element: <Navigate to="/dashboard" replace /> },
          { path: '*', element: <Navigate to="/dashboard" replace /> },
        ],
      },
    ],
  },
]

export default routes
```

**结构图解**（这段配置是整个应用的骨架，务必理解）：

```text
/login                → Login（无守卫）
AuthRoute（有 token 才放行）
 └─ MainLayout（菜单+顶栏，Outlet 渲染子级）
     ├─ /dashboard    → Dashboard
     ├─ /anomaly      → AnomalyList
     ├─ /anomaly/:id  → RootCauseDetail（动态参数）
     ├─ /approval      → Approval
     ├─ /approval/history → ApprovalHistory
     ├─ /              → 重定向 /dashboard
     └─ *              → 兜底重定向
```

`useParams()` 里的 `:id` 就是详情页取事件 ID 的途径。

### 7.2 在 App.tsx 中装配 Router + Suspense + 文档标题

```tsx
// src/App.tsx（最终版）
import { Suspense, lazy, useEffect } from 'react'
import { RouterProvider, createBrowserRouter, Navigate } from 'react-router-dom'
import { ConfigProvider, App as AntdApp, Spin } from 'antd'
import zhCN from 'antd/locale/zh_CN'
import { useUIStore } from '@/store/uiStore'
import { getAntdTheme } from '@/utils/theme'
import { appRoutes } from '@/router'
import AuthRoute from '@/router/AuthRoute'
import MainLayout from '@/layouts/MainLayout'
import { buildMenu } from '@/router/menus'

const Login = lazy(() => import('@/pages/Login'))

/* 菜单重建（保证本文件独立可运行） */
buildMenu(appRoutes)

function Fallback() {
  return (
    <div style={{ height: '60vh', display: 'grid', placeItems: 'center' }}>
      <Spin size="large" tip="页面加载中..." />
    </div>
  )
}

/* 标题同步：进入路由时更新 document.title */
function useDocumentTitle() {
  useEffect(() => {
    const sync = () => {
      const hit = appRoutes.find(
        (r) => location.pathname === r.path || (r.path.includes(':') && location.pathname.startsWith(r.path.split('/:')[0] + '/')),
      )
      document.title = hit ? `${hit.meta.title} - IndustryAgent` : 'IndustryAgent'
    }
    sync()
    window.addEventListener('popstate', sync)
    return () => window.removeEventListener('popstate', sync)
  }, [])
}

export default function App() {
  const themeMode = useUIStore((s) => s.themeMode)
  useDocumentTitle()

  const router = createBrowserRouter([
    { path: '/login', element: <Login /> },
    {
      element: <AuthRoute />,
      children: [
        {
          element: <MainLayout />,
          children: [
            ...appRoutes.map((r) => ({
              path: r.path,
              element: (
                <Suspense fallback={<Fallback />}>
                  {r.element ? <r.element /> : null}
                </Suspense>
              ),
            })),
            { path: '/', element: <Navigate to="/dashboard" replace /> },
            { path: '*', element: <Navigate to="/dashboard" replace /> },
          ],
        },
      ],
    },
  ])

  return (
    <ConfigProvider locale={zhCN} theme={getAntdTheme(themeMode)}>
      <AntdApp>
        <RouterProvider router={router} />
      </AntdApp>
    </ConfigProvider>
  )
}
```

两处取舍说明：

- 这里没有复用第 7.1 的 `routes` 数组而直接内联构建，是因为 `element` 需要包 `Suspense` 与动态组件渲染，TSX 环境下写法更直观；`src/router/index.tsx` 保留作为路由表数据源（`appRoutes`），菜单与标题同步都从它读取——**单一数据源**是这一设计的目的；
- `useDocumentTitle` 用了事件监听而非 `useLocation`（App 在 Router 外取不到），简单够用。

此刻先建 4 个占位页面保证可编译：

```tsx
// src/pages/Dashboard/index.tsx（占位，第8章完整实现）
export default function Dashboard() {
  return <div>良率看板（第8章实现）</div>
}
```

```tsx
// src/pages/Anomaly/index.tsx
export default function AnomalyList() {
  return <div>异常事件列表（第9章实现）</div>
}
```

```tsx
// src/pages/RootCause/index.tsx
import { useParams } from 'react-router-dom'
export default function RootCauseDetail() {
  const { id } = useParams()
  return <div>根因分析详情：事件 {id}（第10章实现）</div>
}
```

```tsx
// src/pages/Approval/index.tsx
export default function Approval() {
  return <div>审批工作台（第11章实现）</div>
}
```

```tsx
// src/pages/ApprovalHistory/index.tsx
export default function ApprovalHistory() {
  return <div>审批历史（第11章实现）</div>
}
```

**【验证】** `pnpm dev` 后：

1. 直接访问 `http://localhost:5173/dashboard` → 应被重定向到 `/login`（无 token）；
2. 登录成功 → 进入主布局：深蓝顶栏（Logo、主题开关、铃铛、用户名）+ 左侧五个菜单 + 内容区面包屑"首页 / 良率看板"；
3. 点击"异常事件"→ 内容区切换占位文本，面包屑同步变化，浏览器标签页标题变化；
4. 手动改 URL 为 `/anomaly/1001` → 显示"根因分析详情：事件 1001"；
5. 地址栏输一个乱路径 `/xyz` → 兜底重定向到 `/dashboard`；
6. 退出登录 → 回到登录页，且刷新页面 token 仍在（persist 生效）；清空 localStorage 再访问受保护页 → 再次被弹回登录。

---

## 8. 良率看板页：React Query + ECharts

### 8.1 React Query 入门：为什么它值得学

在看板之前，先搞懂概念：**服务端状态（server state）≠ 客户端状态（client state）**。

| | 客户端状态（Zustand） | 服务端状态（React Query） |
|---|---|---|
| 归属 | 前端独有 | 后端持有，前端只是缓存 |
| 时效性 | 修改即最新 | 随时可能过期 |
| 典型 | 主题、菜单折叠、表单草稿 | 列表、详情、看板统计 |

服务端状态交给 React Query 后，你不再手写 `useEffect + useState + loading + error`，而是声明式地拿到 `data/isPending/isError`，外加缓存去重、自动重试、轮询刷新、变更后失效（invalidation）。

核心 API 三件套速览（本章用到即讲）：

```tsx
// useQuery：读
const { data, isPending, isError, refetch } = useQuery({
  queryKey: ['overview'],      // 缓存键（类比 Redis key），同键共享缓存
  queryFn: getOverview,         // 取数函数，必须返回 Promise<T>
  refetchInterval: 30_000,      // 30s 轮询（看板自动刷新）
})

// useMutation：写（第11章审批用）
const mutation = useMutation({
  mutationFn: submit,
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['tasks'] }),
})

// queryClient.invalidateQueries：把指定 key 标记为过期，活跃视图自动重拉
```

### 8.2 ECharts 按需引入与 useChart 封装 hook

先做**按需引入**（性能要求，第 16 章呼应）：不引全量 `echarts`，只注册用到的组件。

```ts
// src/components/echarts/index.ts
import * as echarts from 'echarts/core'
import { LineChart, PieChart, BarChart, GraphChart } from 'echarts/charts'
import {
  DataZoomComponent, GridComponent, LegendComponent,
  TitleComponent, TooltipComponent,
} from 'echarts/components'
import { CanvasRenderer } from 'echarts/renderers'

echarts.use([
  LineChart, PieChart, BarChart, GraphChart,
  GridComponent, TooltipComponent, LegendComponent,
  TitleComponent, DataZoomComponent,
  CanvasRenderer,
])

export default echarts
export type { EChartsOption } from 'echarts/types/dist/shared'
```

再写 `useChart` hook——ECharts 是命令式库，React 里必须处理三件事：**初始化一次、setOption 更新、resize 监听 + 卸载销毁**：

```ts
// src/components/hooks/useChart.ts
import { useEffect, useRef } from 'react'
import echarts, { type EChartsOption } from '@/components/echarts'

/**
 * ECharts React 封装
 * @param option 图表配置，变化时自动 setOption
 * @param opts.loading 外部控制 loading 态（转圈）
 */
export function useChart(option: EChartsOption, opts: { loading?: boolean } = {}) {
  const containerRef = useRef<HTMLDivElement>(null)
  const chartRef = useRef<echarts.ECharts | null>(null)

  // 1) 初始化（只跑一次）+ resize 监听 + 卸载销毁
  useEffect(() => {
    if (!containerRef.current) return
    chartRef.current = echarts.init(containerRef.current)

    const handleResize = () => chartRef.current?.resize()
    window.addEventListener('resize', handleResize)
    // AntD Layout 折叠菜单也改变宽度：用 ResizeObserver 更稳
    const ro = new ResizeObserver(handleResize)
    ro.observe(containerRef.current)

    return () => {
      window.removeEventListener('resize', handleResize)
      ro.disconnect()
      chartRef.current?.dispose()
      chartRef.current = null
    }
  }, [])

  // 2) option 变化 -> setOption（notMerge 保证配置全量替换不残留）
  useEffect(() => {
    chartRef.current?.setOption(option, { notMerge: true })
  }, [option])

  // 3) loading 控制
  useEffect(() => {
    if (opts.loading) chartRef.current?.showLoading('default', { color: '#1668dc' })
    else chartRef.current?.hideLoading()
  }, [opts.loading])

  return containerRef
}
```

三个细节讲透：

- **`ResizeObserver`**：只监听 `window.resize` 不够——侧边菜单折叠、Tab 切换改变容器宽度时不触发 window 事件，图表会被"压扁"，这是高频坑（FAQ 见第 17 章）；
- **`dispose()`**：SPA 切页不销毁实例会内存泄漏，且同一 DOM 再次 init 会报 `There is a chart instance already`；
- **`setOption` 依赖对象引用**：option 必须由调用方 `useMemo` 稳定化，否则每次渲染都 setOption，性能反而劣化。

### 8.3 统计卡片组件

```tsx
// src/components/StatCard.tsx
import { Card, Statistic, Tooltip } from 'antd'
import { InfoCircleOutlined } from '@ant-design/icons'
import type { ReactNode } from 'react'

interface StatCardProps {
  title: string
  value: number | string
  suffix?: string
  tip?: ReactNode
  extra?: ReactNode
  loading?: boolean
  danger?: boolean
}

export default function StatCard({ title, value, suffix, tip, extra, loading, danger }: StatCardProps) {
  return (
    <Card size="small" loading={loading}>
      <Statistic
        title={
          <span>
            {title}
            {tip && (
              <Tooltip title={tip}>
                <InfoCircleOutlined style={{ marginLeft: 6, color: '#999' }} />
              </Tooltip>
            )}
          </span>
        }
        value={value}
        suffix={suffix}
        precision={1}
        valueStyle={danger ? { color: '#cf1322' } : undefined}
      />
      {extra}
    </Card>
  )
}
```

### 8.4 看板页完整实现

```tsx
// src/pages/Dashboard/index.tsx
import { useMemo } from 'react'
import { Card, Col, Row, Typography } from 'antd'
import { useQuery } from '@tanstack/react-query'
import {
  getOverview, getYieldTrend, getDefectCategory, getTopEquipment,
  type DashboardOverview, type YieldTrendPoint,
  type DefectCategoryItem, type TopEquipmentItem,
} from '@/api/dashboard'
import { useChart } from '@/components/hooks/useChart'
import StatCard from '@/components/StatCard'
import styles from './index.module.css'

export default function Dashboard() {
  /* ---------- 数据获取：四个并行 useQuery ---------- */
  const overviewQuery = useQuery({
    queryKey: ['dashboard', 'overview'],
    queryFn: getOverview,
    refetchInterval: 30_000, // 看板 30s 自动刷新
  })

  const trendQuery = useQuery({ queryKey: ['dashboard', 'yield-trend'], queryFn: getYieldTrend })
  const defectQuery = useQuery({ queryKey: ['dashboard', 'defect-category'], queryFn: getDefectCategory })
  const topEquipQuery = useQuery({ queryKey: ['dashboard', 'top-equipment'], queryFn: getTopEquipment })

  const overview: DashboardOverview | undefined = overviewQuery.data

  /* ---------- 图表配置（useMemo 稳定引用，喂给 useChart） ---------- */
  const trendOption = useBuildTrendOption(trendQuery.data)
  const pieOption = useBuildPieOption(defectQuery.data)
  const barOption = useBuildBarOption(topEquipQuery.data)

  const trendRef = useChart(trendOption, { loading: trendQuery.isPending })
  const pieRef = useChart(pieOption, { loading: defectQuery.isPending })
  const barRef = useChart(barOption, { loading: topEquipQuery.isPending })

  return (
    <div className={styles.container}>
      {/* 统计卡片行 */}
      <Row gutter={16}>
        <Col span={6}>
          <StatCard
            title="综合良率"
            value={overview?.yieldRate ?? '-'}
            suffix="%"
            tip="近 24h 投片综合良率（目标 95%）"
            danger={overview ? overview.yieldRate < overview.yieldTarget : false}
            loading={overviewQuery.isPending}
            extra={
              <Typography.Text type="secondary" style={{ fontSize: 12 }}>
                目标 {overview?.yieldTarget ?? '-'}%
                {overview && overview.yieldRate < overview.yieldTarget && (
                  <Typography.Text type="danger">
                    {' '}(缺口 {(overview.yieldTarget - overview.yieldRate).toFixed(1)}pct)
                  </Typography.Text>
                )}
              </Typography.Text>
            }
          />
        </Col>
        <Col span={6}>
          <StatCard title="OEE" value={overview?.oee ?? '-'} suffix="%" tip="设备综合效率 = 时间开动率×性能开动率×合格品率" loading={overviewQuery.isPending} />
        </Col>
        <Col span={6}>
          <StatCard title="24h 异常事件" value={overview?.anomalyCount24h ?? '-'} tip="FDC/SPC 规则触发的异常事件数" loading={overviewQuery.isPending} />
        </Col>
        <Col span={6}>
          <StatCard title="待我审批" value={overview?.pendingApprovalCount ?? '-'} tip="等待人工审批的调参建议数" loading={overviewQuery.isPending} />
        </Col>
      </Row>

      {/* 良率趋势 + 缺陷分布 */}
      <Row gutter={16} className={styles.row}>
        <Col span={16}>
          <Card size="small" title="近 14 天良率趋势" className={styles.chartCard}>
            <div ref={trendRef} className={styles.chart} />
          </Card>
        </Col>
        <Col span={8}>
          <Card size="small" title="缺陷分类占比" className={styles.chartCard}>
            <div ref={pieRef} className={styles.chart} />
          </Card>
        </Col>
      </Row>

      {/* Top 异常设备 */}
      <Row gutter={16} className={styles.row}>
        <Col span={24}>
          <Card size="small" title="Top 5 异常设备（近 30 天）" className={styles.chartCard}>
            <div ref={barRef} className={styles.chartWide} />
          </Card>
        </Col>
      </Row>
    </div>
  )
}

/* ==================== 图表配置构建（页面私有逻辑，内聚在页面文件） ==================== */

function useBuildTrendOption(data?: YieldTrendPoint[]) {
  return useMemo(
    () => ({
      tooltip: { trigger: 'axis' },
      legend: { data: ['实际良率', '目标'] },
      grid: { left: 48, right: 24, top: 40, bottom: 32 },
      xAxis: { type: 'category', data: data?.map((d) => d.date) ?? [] },
      yAxis: { type: 'value', min: 90, max: 97, axisLabel: { formatter: '{value}%' } },
      series: [
        {
          name: '实际良率',
          type: 'line',
          smooth: true,
          symbol: 'circle',
          symbolSize: 6,
          data: data?.map((d) => d.yieldRate) ?? [],
          itemStyle: { color: '#1668dc' },
          areaStyle: { opacity: 0.08 },
        },
        {
          name: '目标',
          type: 'line',
          showSymbol: false,
          lineStyle: { type: 'dashed', color: '#fa541c' },
          data: data?.map(() => 95),
        },
      ],
    }),
    [data],
  )
}

function useBuildPieOption(data?: DefectCategoryItem[]) {
  return useMemo(
    () => ({
      tooltip: { trigger: 'item', formatter: '{b}: {c} 起（{d}%）' },
      legend: { bottom: 0, type: 'scroll' },
      series: [
        {
          type: 'pie',
          radius: ['45%', '70%'],
          avoidLabelOverlap: true,
          itemStyle: { borderRadius: 4, borderColor: '#fff', borderWidth: 2 },
          label: { formatter: '{b}\n{d}%' },
          data: data ?? [],
        },
      ],
    }),
    [data],
  )
}

function useBuildBarOption(data?: TopEquipmentItem[]) {
  return useMemo(
    () => ({
      tooltip: { trigger: 'axis' },
      grid: { left: 80, right: 24, top: 24, bottom: 32 },
      xAxis: { type: 'value' },
      yAxis: { type: 'category', inverse: true, data: data?.map((d) => d.equipmentCode) ?? [] },
      series: [
        {
          type: 'bar',
          barWidth: 18,
          data: data?.map((d) => d.count) ?? [],
          itemStyle: { color: '#1668dc', borderRadius: [0, 4, 4, 0] },
          label: { show: true, position: 'right' },
        },
      ],
    }),
    [data],
  )
}
```

```css
/* src/pages/Dashboard/index.module.css */
.container { padding: 4px 0; }
.row { margin-top: 16px; }
.chartCard :global(.ant-card-body) { padding: 12px; }
.chart { height: 320px; width: 100%; }
.chartWide { height: 260px; width: 100%; }
```

> `:global()` 是 CSS Modules 穿透语法：卡片内部类名是 AntD 生成的，我们只想覆盖该局部作用域下的 padding，避免全局污染。

**【验证】** 访问 `/dashboard`：

1. 四张统计卡先骨架屏后显示数字，良率 93.2% 显示红色（低于目标）并标注缺口；
2. 三张图依次渲染：折线带红色目标虚线、饼图为环形带图例、柱状图横向倒序且数值显示在条右侧；
3. 拖动窗口宽度图表自适应；点击侧边菜单折叠，图表在动画结束后自动变宽（ResizeObserver 生效）；
4. Network 面板 30 秒后自动重新请求 overview（React Query 轮询生效）。

---

## 9. 异常事件列表页：查询表格 + WebSocket

### 9.1 WebSocket 封装 hook：心跳 + 断线重连 + 事件分发

先写基础设施。工业场景网络不稳，**心跳保活 + 指数退避重连**是实时告警的底线要求：

```ts
// src/components/hooks/useWebSocket.ts
import { useEffect, useRef } from 'react'

export interface WsMessage<T = unknown> {
  type: string
  payload: T
}

interface UseWebSocketOptions {
  /** 收到消息回调（回调变化不重建连接） */
  onMessage: (msg: WsMessage<unknown>) => void
  /** 连接建立回调 */
  onOpen?: () => void
  /** 是否启用（未登录时关闭） */
  enabled?: boolean
  /** 心跳间隔，默认 30s */
  heartbeatInterval?: number
  /** 最大重连次数，默认 6 */
  maxRetries?: number
}

/**
 * 原生 WebSocket 封装。
 * 特性：连接建立后自动心跳保活、断线指数退避重连（1s,2s,4s...上限 30s）、
 *       回调经 ref 中转（回调更新不重建连接）。
 */
export function useWebSocket(options: UseWebSocketOptions) {
  const { onMessage, onOpen, enabled = true, heartbeatInterval = 30_000, maxRetries = 6 } = options

  // 用 ref 持有最新回调，避免回调变化导致重连
  const cbRef = useRef({ onMessage, onOpen })
  cbRef.current = { onMessage, onOpen }

  useEffect(() => {
    if (!enabled) return

    const token = JSON.parse(localStorage.getItem('ia-auth') || '{}')?.state?.token as string | undefined
    const base = import.meta.env.VITE_WS_URL // '/ws/v1/alarm'
    const protocol = location.protocol === 'https:' ? 'wss' : 'ws'
    // token 通过 query 传递（浏览器 WebSocket 握手无法自定义 Header 的兼容做法）
    const url = `${protocol}://${location.host}${base}?token=${token ?? ''}`

    let ws: WebSocket | null = null
    let heartbeatTimer: number | null = null
    let reconnectTimer: number | null = null
    let retryCount = 0
    let closedByUser = false

    const startHeartbeat = () => {
      stopHeartbeat()
      heartbeatTimer = window.setInterval(() => {
        if (ws?.readyState === WebSocket.OPEN) ws.send(JSON.stringify({ type: 'PING' }))
      }, heartbeatInterval)
    }
    const stopHeartbeat = () => {
      if (heartbeatTimer) { clearInterval(heartbeatTimer); heartbeatTimer = null }
    }

    const connect = () => {
      ws = new WebSocket(url)

      ws.onopen = () => {
        retryCount = 0 // 连上即清零重连计数
        startHeartbeat()
        cbRef.current.onOpen?.()
      }

      ws.onmessage = (e) => {
        try {
          const msg = JSON.parse(e.data) as WsMessage<unknown>
          if (msg.type === 'PONG') return // 心跳应答不外抛
          cbRef.current.onMessage(msg)
        } catch { /* 非JSON帧忽略 */ }
      }

      ws.onclose = () => {
        stopHeartbeat()
        if (closedByUser || retryCount >= maxRetries) return
        // 指数退避：1s,2s,4s...上限 30s
        const delay = Math.min(1000 * 2 ** retryCount, 30_000)
        retryCount += 1
        reconnectTimer = window.setTimeout(connect, delay)
      }

      ws.onerror = () => ws?.close()
    }

    connect()

    return () => {
      closedByUser = true
      stopHeartbeat()
      if (reconnectTimer) clearTimeout(reconnectTimer)
      ws?.close()
    }
  }, [enabled, heartbeatInterval, maxRetries])
}
```

设计要点（逐条对应代码）：

1. **回调走 ref**：`onMessage` 每次渲染都是新函数，若直接进依赖数组会导致连接反复重建——用 `cbRef` 中转是 React 封装 WebSocket 的标准手法；
2. **心跳**：每 30s 发一帧 `PING`，代理服务器（Nginx 默认 60s 超时）就不会掐断空闲连接；
3. **指数退避重连**：`2^n` 退避防止服务端刚恢复就被重连风暴打死；
4. **token 经 query**：WebSocket 握手阶段无法设置 `Authorization` Header（浏览器限制），query 传 token 是兼容方案（后端从 query 解析，见 IA-TU-001）。

### 9.2 实时事件仓库：WebSocket 推送落到全局状态

WebSocket 连接放在 `MainLayout` 挂载一次（整条连接全局共享），消息落地 zustand，**任何页面都能消费且互不干扰**——这是"WebSocket 跨页面状态"的标准解法（FAQ 见第 17 章）。

```ts
// src/store/realtimeStore.ts
import { create } from 'zustand'
import type { AnomalyEvent } from '@/types/anomaly'

interface RealtimeState {
  /** WebSocket 推来的最新事件（未消费），列表页读取后置顶 */
  incomingEvents: AnomalyEvent[]
  /** 未读告警数（顶栏铃铛） */
  unreadCount: number
  pushEvent: (e: AnomalyEvent) => void
  clearUnread: () => void
  /** 列表页消费事件后清空缓冲 */
  consumeEvents: () => AnomalyEvent[]
}

export const useRealtimeStore = create<RealtimeState>((set, get) => ({
  incomingEvents: [],
  unreadCount: 0,
  pushEvent: (e) =>
    set((s) => ({
      incomingEvents: [e, ...s.incomingEvents].slice(0, 20),
      unreadCount: s.unreadCount + 1,
    })),
  clearUnread: () => set({ unreadCount: 0 }),
  consumeEvents: () => {
    const events = get().incomingEvents
    set({ incomingEvents: [] })
    return events
  },
}))
```

### 9.3 状态映射与业务常量

```ts
// src/utils/constants.ts
import type { EventStatus, Severity } from '@/types/anomaly'

/** 严重程度 -> Tag 颜色/文案 */
export const SEVERITY_MAP: Record<Severity, { color: string; label: string }> = {
  critical: { color: 'red', label: '紧急' },
  high: { color: 'orange', label: '高' },
  medium: { color: 'gold', label: '中' },
  low: { color: 'blue', label: '低' },
}

/** 事件状态 -> 标签配置 */
export const STATUS_MAP: Record<EventStatus, { color: string; label: string }> = {
  pending: { color: 'default', label: '待分析' },
  analyzing: { color: 'processing', label: '分析中' },
  awaiting_approval: { color: 'warning', label: '待审批' },
  closed: { color: 'success', label: '已闭环' },
}

/** 工序列表（查询下拉用） */
export const PROCESS_STEPS = ['刻蚀', '薄膜', '光刻', '扩散', '离子注入', 'CMP', '金属化']
```

### 9.4 异常事件列表页完整代码

```tsx
// src/pages/Anomaly/index.tsx
import { useCallback, useEffect, useMemo, useState } from 'react'
import { useNavigate } from 'react-router-dom'
import {
  App, Badge, Button, Card, Empty, Form, Input, Progress, Select, Space, Table, Tag, Tooltip,
} from 'antd'
import { ReloadOutlined, SearchOutlined } from '@ant-design/icons'
import { keepPreviousData, useQuery } from '@tanstack/react-query'
import { getAnomalyEvents } from '@/api/anomaly'
import type { AnomalyEvent, AnomalyQueryParams, EventStatus, Severity } from '@/types/anomaly'
import { SEVERITY_MAP, STATUS_MAP, PROCESS_STEPS } from '@/utils/constants'
import { useRealtimeStore } from '@/store/realtimeStore'

/** 查询表单值（空串视为未选） */
interface QueryFormValues {
  keyword?: string
  equipmentCode?: string
  processStep?: string
  severity?: Severity
  status?: EventStatus
}

export default function AnomalyList() {
  const navigate = useNavigate()
  const { notification } = App.useApp()

  /* ---------- 查询参数：版本号驱动的受控分页 ---------- */
  const [form] = Form.useForm<QueryFormValues>()
  const [query, setQuery] = useState<AnomalyQueryParams>({ page: 1, pageSize: 20 })
  // queryKey 里的 query 是对象引用；用展开值生成稳定 key
  const queryString = JSON.stringify(query)

  /* ---------- 数据获取 ---------- */
  const { data, isFetching, refetch } = useQuery({
    queryKey: ['anomaly', 'events', queryString],
    queryFn: () => getAnomalyEvents(query),
    placeholderData: keepPreviousData, // 翻页时保留旧数据，表格不闪空（分页体验关键）
  })

  /* ---------- WebSocket 新事件：通知 + 自动刷新 ---------- */
  const incomingEvents = useRealtimeStore((s) => s.incomingEvents)
  const consumeEvents = useRealtimeStore((s) => s.consumeEvents)

  useEffect(() => {
    if (incomingEvents.length === 0) return
    const events = consumeEvents()
    events.forEach((e) => {
      notification.warning({
        message: `新异常事件：${e.equipmentCode}`,
        description: e.title,
        placement: 'topRight',
        duration: 6,
      })
    })
    refetch() // 收到推送后自动刷新列表
  }, [incomingEvents, consumeEvents, refetch])

  /* ---------- 查询/重置 ---------- */
  const onSearch = useCallback(() => {
    const v = form.getFieldsValue()
    setQuery((prev) => ({
      ...prev,
      page: 1, // 新查询强制回第一页
      keyword: v.keyword || undefined,
      equipmentCode: v.equipmentCode || undefined,
      processStep: v.processStep || undefined,
      severity: v.severity,
      status: v.status,
    }))
  }, [form])

  const onReset = useCallback(() => {
    form.resetFields()
    setQuery({ page: 1, pageSize: 20 })
  }, [form])

  const columns = useMemo(
    () => [
      { title: 'ID', dataIndex: 'id', width: 90, fixed: 'left' as const },
      {
        title: '事件标题',
        dataIndex: 'title',
        ellipsis: true,
        render: (text: string, record: AnomalyEvent) => (
          <a onClick={() => navigate(`/anomaly/${record.id}`)}>{text}</a>
        ),
      },
      { title: '设备', dataIndex: 'equipmentCode', width: 110 },
      { title: '工序', dataIndex: 'processStep', width: 90 },
      { title: '批次', dataIndex: 'lotNo', width: 120 },
      {
        title: '严重程度',
        dataIndex: 'severity',
        width: 90,
        render: (v: Severity) => {
          const conf = SEVERITY_MAP[v]
          return <Tag color={conf.color}>{conf.label}</Tag>
        },
      },
      {
        title: 'FDC 分数',
        dataIndex: 'fdcScore',
        width: 140,
        render: (v: number) => (
          <Tooltip title={`FDC 异常分数：${v}`}>
            <Progress percent={v} size="small" strokeColor={v > 80 ? '#cf1322' : '#faad14'} />
          </Tooltip>
        ),
      },
      {
        title: '状态',
        dataIndex: 'status',
        width: 100,
        render: (v: EventStatus) => {
          const conf = STATUS_MAP[v]
          return <Badge status={conf.color as never} text={conf.label} />
        },
      },
      { title: '发生时间', dataIndex: 'createdAt', width: 170 },
      {
        title: '操作',
        key: 'action',
        width: 90,
        fixed: 'right' as const,
        render: (_: unknown, record: AnomalyEvent) => (
          <Button type="link" size="small" onClick={() => navigate(`/anomaly/${record.id}`)}>
            根因分析
          </Button>
        ),
      },
    ],
    [navigate],
  )

  return (
    <Card size="small">
      {/* 查询表单区 */}
      <Form form={form} layout="inline" style={{ marginBottom: 16 }}>
        <Form.Item name="keyword">
          <Input placeholder="事件标题关键词" allowClear style={{ width: 200 }} />
        </Form.Item>
        <Form.Item name="equipmentCode">
          <Input placeholder="设备编号，如 ETA-301" allowClear style={{ width: 180 }} />
        </Form.Item>
        <Form.Item name="processStep">
          <Select
            placeholder="工序"
            allowClear
            style={{ width: 130 }}
            options={PROCESS_STEPS.map((p) => ({ label: p, value: p }))}
          />
        </Form.Item>
        <Form.Item name="severity">
          <Select
            placeholder="严重程度"
            allowClear
            style={{ width: 120 }}
            options={Object.entries(SEVERITY_MAP).map(([value, c]) => ({ label: c.label, value }))}
          />
        </Form.Item>
        <Form.Item name="status">
          <Select
            placeholder="状态"
            allowClear
            style={{ width: 120 }}
            options={Object.entries(STATUS_MAP).map(([value, c]) => ({ label: c.label, value }))}
          />
        </Form.Item>
        <Form.Item>
          <Space>
            <Button type="primary" icon={<SearchOutlined />} onClick={onSearch}>
              查询
            </Button>
            <Button icon={<ReloadOutlined />} onClick={onReset}>
              重置
            </Button>
          </Space>
        </Form.Item>
      </Form>

      <Table<AnomalyEvent>
        rowKey="id"
        columns={columns}
        dataSource={data?.list}
        loading={isFetching}
        scroll={{ x: 1100 }}
        locale={{ emptyText: <Empty description="暂无异常事件 — 生产岁月静好" /> }}
        pagination={{
          current: query.page,
          pageSize: query.pageSize,
          total: data?.total ?? 0,
          showSizeChanger: true,
          showTotal: (t) => `共 ${t} 条`,
          onChange: (page, pageSize) => setQuery((prev) => ({ ...prev, page, pageSize })),
        }}
      />
    </Card>
  )
}
```

**受控分页的设计逻辑**（企业表格三件套：过滤条件、页码、total 单一数据流）：

- 过滤条件与页码全部收敛在 `query` state；`queryKey` 含 `queryString` → **参数变 = 缓存变 = 自动重新请求**，且"回到上一次查询结果"天然免费（React Query 缓存）；
- `placeholderData: keepPreviousData` 让翻页请求期间表格仍显示旧数据 + loading 角标，而不是闪回空表；
- 点查询强制回第一页（否则"第 5 页 + 新过滤条件"可能请求出空列表）。

### 9.5 顶栏接线：MainLayout 挂全局连接 + 铃铛未读

对 `MainLayout.tsx` 做三处增量修改：

```tsx
// 增量 1：文件顶部新增 import
import { useWebSocket, type WsMessage } from '@/components/hooks/useWebSocket'
import { useRealtimeStore } from '@/store/realtimeStore'
import type { AnomalyEvent } from '@/types/anomaly'

// 增量 2：MainLayout 函数体顶部，建立全局唯一连接（组件卸载即断开）
const unreadCount = useRealtimeStore((s) => s.unreadCount)
const pushEvent = useRealtimeStore((s) => s.pushEvent)
const clearUnread = useRealtimeStore((s) => s.clearUnread)
const token = useAuthStore((s) => s.token)

useWebSocket({
  enabled: !!token, // 登录才连
  onMessage: (msg: WsMessage<unknown>) => {
    if (msg.type === 'ANOMALY_CREATED') {
      pushEvent(msg.payload as AnomalyEvent)
    }
  },
})

// 增量 3：替换原铃铛 Tooltip/Badge 片段
<Tooltip title={unreadCount > 0 ? `${unreadCount} 条未读告警` : '暂无未读告警'}>
  <Badge count={unreadCount} size="small" offset={[-4, 4]}>
    <BellOutlined
      style={{ color: '#fff', fontSize: 18, cursor: 'pointer' }}
      onClick={clearUnread}
    />
  </Badge>
</Tooltip>
```

### 9.6 Mock 推送：联调前的自测手段

HTTP 接口可用 Vite 中间件 Mock，但 WebSocket 升级无法走该中间件——开发期用**手动触发**方案验证链路后半段（连接前半段联调真实后端即可）：

```tsx
// src/App.tsx 文件末尾追加（DEV 环境注入全局触发函数）

declare global {
  interface Window {
    /** 浏览器 Console 执行 window.__pushMockEvent() 模拟一条 WS 推送 */
    __pushMockEvent?: () => void
  }
}

function injectMockPush() {
  if (!import.meta.env.DEV) return
  window.__pushMockEvent = () => {
    useRealtimeStore.getState().pushEvent({
      id: Date.now(),
      equipmentCode: 'CVD-105',
      processStep: '薄膜',
      lotNo: 'LOT-MOCK-01',
      title: '模拟推送：薄膜沉积压力尖峰',
      severity: 'high',
      status: 'pending',
      triggerRule: 'FDC: Pressure spike (mock)',
      fdcScore: 84,
      createdAt: new Date().toLocaleString('sv-PR'),
    })
  }
}

// 在 App 组件体第一行调用 injectMockPush()
```

注意 `App.tsx` 需补 import：

```tsx
import { useRealtimeStore } from '@/store/realtimeStore'
```

**【验证】** 访问 `/anomaly`：

1. 表格展示 3 条 Mock 事件，FDC 分数进度条、严重程度/状态标签颜色正确（紧急红、高橙、中金、低蓝）；
2. 输入设备 `ETA-301` 点查询 → 只剩 1 条且页码回到第 1；点重置恢复；
3. 翻页时表格保持旧数据 + loading（keepPreviousData 生效）；
4. Console 执行 `window.__pushMockEvent()` → 右上角弹出通知"新异常事件：CVD-105"，顶栏铃铛 +1，列表自动刷新出现新事件；点击铃铛未读清零；
5. 点击事件标题 → 跳转 `/anomaly/1001` 详情占位页；面包屑显示"首页 / 异常事件 / 根因分析"。

---

## 10. 根因分析详情页：Top-3 嫌疑 + 证据链

这是本项目的"门面页"——智能体的价值浓缩于此：给定一个异常事件，展示 Top-3 根因嫌疑、可解释的证据链（参数时序、知识图谱、相似案例、因果结论），并支持人工触发重新分析。

### 10.1 页面结构设计

```text
┌──────────────────────────────────────────────┐
│ 事件信息卡片（标题/设备/批次/严重程度/状态/触发规则）  [重新分析] │
├──────────────────────────────────────────────┤
│ Top-3 嫌疑卡片（横向三列）                      │
│  嫌疑1(置信度86%) │ 嫌疑2(62%) │ 嫌疑3(41%)     │
├──────────────────────────────────────────────┤
│ 证据链 Tabs                                    │
│  ├ 参数时序对比（ECharts line，值 vs 基线）        │
│  ├ 知识图谱（ECharts graph，关系可视化）           │
│  ├ 相似历史案例（列表 + 相似度进度条）              │
│  └ 因果分析结论（文本卡片）                        │
└──────────────────────────────────────────────┘
```

### 10.2 详情页完整代码

先写 Top-3 嫌疑卡片子组件（页面私有组件，放页面目录内）：

```tsx
// src/pages/RootCause/SuspectCard.tsx
import { Card, Descriptions, Progress, Tag, Typography } from 'antd'
import { WarningOutlined } from '@ant-design/icons'
import type { RootCauseSuspect } from '@/types/rootcause'
import styles from './SuspectCard.module.css'

/** 置信度 -> 颜色 */
function confidenceColor(c: number) {
  if (c >= 0.75) return '#cf1322'
  if (c >= 0.5) return '#fa8c16'
  return '#1677ff'
}

export default function SuspectCard({ suspect }: { suspect: RootCauseSuspect }) {
  const percent = Math.round(suspect.confidence * 100)
  return (
    <Card
      size="small"
      title={
        <span>
          <Tag color={suspect.rank === 1 ? 'red' : 'default'}>嫌疑 #{suspect.rank}</Tag>
          <Typography.Text strong>{suspect.hypothesis}</Typography.Text>
        </span>
      }
      extra={
        suspect.rank === 1 && (
          <Tag icon={<WarningOutlined />} color="red">
            最可能根因
          </Tag>
        )
      }
      className={suspect.rank === 1 ? styles.top1 : undefined}
    >
      <div className={styles.confidenceRow}>
        <span>置信度</span>
        <Progress
          percent={percent}
          strokeColor={confidenceColor(suspect.confidence)}
          format={(p) => `${p}%`}
        />
      </div>

      <Descriptions column={1} size="small" className={styles.desc}>
        <Descriptions.Item label="关联工序">{suspect.processStep}</Descriptions.Item>
        <Descriptions.Item label="证据条目">{suspect.evidenceCount} 条</Descriptions.Item>
      </Descriptions>

      <Typography.Text type="secondary" style={{ fontSize: 12 }}>
        嫌疑参数（当前值 vs 基线）
      </Typography.Text>
      <div className={styles.params}>
        {suspect.params.map((p) => (
          <div key={p.name} className={styles.paramItem}>
            <code>{p.name}</code>
            <span>
              <Typography.Text delete type="secondary">
                {p.baseline}
              </Typography.Text>
              {' → '}
              <Typography.Text type={Math.abs(p.deviation) > 0.05 * Math.abs(p.baseline) ? 'danger' : undefined}>
                {p.value} {p.unit}
              </Typography.Text>
              <Typography.Text type="secondary">（偏移 {p.deviation > 0 ? '+' : ''}{p.deviation}{p.unit}）</Typography.Text>
            </span>
          </div>
        ))}
      </div>
    </Card>
  )
}
```

```css
/* src/pages/RootCause/SuspectCard.module.css */
.top1 {
  border: 1px solid #ffa39e;
}

.confidenceRow {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 8px;
}

.confidenceRow > span:first-child {
  font-size: 13px;
  color: #666;
}

.confidenceRow > :global(.ant-progress) {
  flex: 1;
  margin: 0;
}

.desc { margin-bottom: 8px; }

.params {
  margin-top: 6px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.paramItem {
  display: flex;
  justify-content: space-between;
  font-size: 13px;
}

.paramItem code {
  background: rgba(22, 104, 220, 0.08);
  padding: 0 6px;
  border-radius: 3px;
}
```

主页面：

```tsx
// src/pages/RootCause/index.tsx
import { useMemo, useState } from 'react'
import { useNavigate, useParams } from 'react-router-dom'
import {
  App, Button, Card, Descriptions, Result, Row, Col, Spin, Tabs, Tag, Typography,
} from 'antd'
import { ArrowLeftOutlined, RedoOutlined } from '@ant-design/icons'
import { useQuery, useQueryClient } from '@tanstack/react-query'
import { getRootCauseDetail, createReanalyze, getReanalyzeStatus } from '@/api/rootcause'
import type { RootCauseDetail, SimilarCase } from '@/types/rootcause'
import { SEVERITY_MAP, STATUS_MAP } from '@/utils/constants'
import { useAuthStore } from '@/store/authStore'
import { useChart } from '@/components/hooks/useChart'
import SuspectCard from './SuspectCard'
import styles from './index.module.css'

export default function RootCauseDetail() {
  const { id } = useParams()
  const eventId = Number(id)
  const navigate = useNavigate()
  const { message } = App.useApp()
  const queryClient = useQueryClient()
  const hasPermission = useAuthStore((s) => s.hasPermission)
  const canReanalyze = hasPermission('rootcause:reanalyze')

  /* ---------- 详情数据 ---------- */
  const { data, isPending, isError } = useQuery({
    queryKey: ['rootcause', eventId],
    queryFn: () => getRootCauseDetail(eventId),
    enabled: Number.isFinite(eventId),
  })

  /* ---------- 重新分析：异步任务 + 轮询 ---------- */
  const [reanalyzing, setReanalyzing] = useState(false)
  const [taskId, setTaskId] = useState<string | null>(null)

  // 任务提交后轮询状态；done 后失效详情并停止
  const taskQuery = useQuery({
    queryKey: ['rootcause', 'task', taskId],
    queryFn: () => getReanalyzeStatus(taskId!),
    enabled: !!taskId && reanalyzing, // 只在"分析中"轮询
    refetchInterval: 2000, // 2s 轮询
  })

  if (taskQuery.data?.status === 'done') {
    // 轮询到完成：停止 + 刷新详情（此处用 useEffect 更规范，见下方修正）
  }

  const startReanalyze = async () => {
    try {
      setReanalyzing(true)
      const { taskId: tid } = await createReanalyze(eventId)
      setTaskId(tid)
      message.info(`已提交重新分析任务 ${tid}，智能体检索中...`)
    } catch {
      setReanalyzing(false)
    }
  }

  // 轮询完成的副作用：放 useEffect，避免渲染期 setState
  useMemo(() => {
    if (taskQuery.data?.status === 'done' && reanalyzing) {
      setReanalyzing(false)
      setTaskId(null)
      message.success('重新分析完成，证据链已更新')
      queryClient.invalidateQueries({ queryKey: ['rootcause', eventId] })
    }
  }, [taskQuery.data, reanalyzing, eventId, message, queryClient])

  /* ---------- 证据链：参数时序图 ---------- */
  const seriesOption = useBuildSeriesOption(data)
  const seriesRef = useChart(seriesOption, { loading: isPending })

  /* ---------- 证据链：知识图谱 ---------- */
  const graphOption = useBuildGraphOption(data)
  const graphRef = useChart(graphOption, { loading: isPending })

  if (isPending) {
    return (
      <div className={styles.center}>
        <Spin size="large" tip="智能体正在组装证据链..." />
      </div>
    )
  }
  if (isError || !data) {
    return (
      <Result
        status="error"
        title="根因分析详情加载失败"
        extra={<Button onClick={() => navigate('/anomaly')}>返回列表</Button>}
      />
    )
  }

  const e = data.event
  return (
    <div className={styles.container}>
      {/* 事件信息卡片 */}
      <Card
        size="small"
        title={
          <span>
            <Button
              type="text"
              icon={<ArrowLeftOutlined />}
              onClick={() => navigate('/anomaly')}
              style={{ marginRight: 8 }}
            />
            <Typography.Text strong>{e.title}</Typography.Text>
          </span>
        }
        extra={
          canReanalyze && (
            <Button
              type="primary"
              danger
              icon={<RedoOutlined />}
              loading={reanalyzing}
              onClick={startReanalyze}
            >
              {reanalyzing ? `分析中 ${taskQuery.data?.progress ?? 0}%` : '重新分析'}
            </Button>
          )
        }
      >
        <Descriptions size="small" column={4}>
          <Descriptions.Item label="事件 ID">{e.id}</Descriptions.Item>
          <Descriptions.Item label="设备">{e.equipmentCode}</Descriptions.Item>
          <Descriptions.Item label="工序">{e.processStep}</Descriptions.Item>
          <Descriptions.Item label="批次">{e.lotNo}</Descriptions.Item>
          <Descriptions.Item label="严重程度">
            <Tag color={SEVERITY_MAP[e.severity].color}>{SEVERITY_MAP[e.severity].label}</Tag>
          </Descriptions.Item>
          <Descriptions.Item label="状态">
            <Tag color={STATUS_MAP[e.status].color}>{STATUS_MAP[e.status].label}</Tag>
          </Descriptions.Item>
          <Descriptions.Item label="触发规则" span={2}>
            {e.triggerRule}
          </Descriptions.Item>
        </Descriptions>
      </Card>

      {/* Top-3 嫌疑 */}
      <Row gutter={16} className={styles.row}>
        {data.suspects.map((s) => (
          <Col span={8} key={s.rank}>
            <SuspectCard suspect={s} />
          </Col>
        ))}
      </Row>

      {/* 证据链 Tabs */}
      <Card size="small" className={styles.row}>
        <Tabs
          items={[
            {
              key: 'series',
              label: '参数时序对比',
              children: (
                <div className={styles.chart}>
                  <div ref={seriesRef} style={{ height: '100%' }} />
                </div>
              ),
            },
            {
              key: 'graph',
              label: '知识图谱关系',
              children: (
                <div className={styles.chart}>
                  <div ref={graphRef} style={{ height: '100%' }} />
                </div>
              ),
            },
            {
              key: 'cases',
              label: `相似历史案例（${data.similarCases.length}）`,
              children: <CaseList cases={data.similarCases} />,
            },
            {
              key: 'conclusion',
              label: '因果分析结论',
              children: (
                <Typography.Paragraph style={{ padding: 8 }}>
                  {data.causalConclusion}
                </Typography.Paragraph>
              ),
            },
          ]}
        />
      </Card>
    </div>
  )
}

/* ---------- 相似案例列表 ---------- */
function CaseList({ cases }: { cases: SimilarCase[] }) {
  return (
    <div>
      {cases.map((c) => (
        <Card size="small" key={c.caseId} style={{ marginBottom: 8 }}>
          <Descriptions size="small" column={3}>
            <Descriptions.Item label="案例号">{c.caseId}</Descriptions.Item>
            <Descriptions.Item label="发生时间">{c.occurredAt}</Descriptions.Item>
            <Descriptions.Item label="设备">{c.equipmentCode}</Descriptions.Item>
            <Descriptions.Item label="案例摘要" span={2}>{c.summary}</Descriptions.Item>
            <Descriptions.Item label="最终根因">{c.finalRootCause}</Descriptions.Item>
            <Descriptions.Item label="相似度" span={3}>
              <Typography.Text
                type={c.similarity >= 0.8 ? 'danger' : 'warning'}
                strong
              >
                {(c.similarity * 100).toFixed(0)}%
              </Typography.Text>
            </Descriptions.Item>
          </Descriptions>
        </Card>
      ))}
    </div>
  )
}

/* ==================== 图表配置 ==================== */

/** 参数时序对比：每个参数一屏并列（多参数用 legend 切换思路见练习） */
function useBuildSeriesOption(data?: RootCauseDetail) {
  return useMemo(() => {
    const series = data?.paramSeries ?? []
    const first = series[0]
    return {
      tooltip: { trigger: 'axis' },
      legend: { data: first ? [`${first.paramName}(实际)`, '基线'] : [] },
      grid: { left: 56, right: 24, top: 40, bottom: 32 },
      xAxis: { type: 'category', data: first?.points.map((p) => p.time) ?? [] },
      yAxis: {
        type: 'value',
        name: first?.unit ?? '',
        scale: true,
      },
      series: [
        {
          name: `${first?.paramName ?? ''}(实际)`,
          type: 'line',
          smooth: true,
          showSymbol: false,
          data: first?.points.map((p) => p.value) ?? [],
          itemStyle: { color: '#1668dc' },
        },
        {
          name: '基线',
          type: 'line',
          showSymbol: false,
          lineStyle: { type: 'dashed', color: '#52c41a' },
          data: first?.points.map((p) => p.baseline) ?? [],
        },
      ],
    })
  }, [data])
}

const CATEGORY_COLOR: Record<string, string> = {
  event: '#cf1322',
  equipment: '#1668dc',
  param: '#fa8c16',
  defect: '#722ed1',
  recipe: '#13c2c2',
}

/** 知识图谱：力导向 graph 布局 */
function useBuildGraphOption(data?: RootCauseDetail) {
  return useMemo(() => {
    const kg = data?.knowledgeGraph
    return {
      tooltip: { formatter: (p: { data: { name?: string; relation?: string } }) => {
        if (p.data.relation) return p.data.relation
        return p.data.name ?? ''
      } },
      series: [
        {
          type: 'graph',
          layout: 'force',
          roam: true, // 可拖拽缩放
          draggable: true,
          force: { repulsion: 220, edgeLength: 90 },
          label: { show: true, fontSize: 11 },
          edgeLabel: { show: true, fontSize: 10, formatter: (p: { data: { relation?: string } }) => p.data.relation ?? '' },
          data: (kg?.nodes ?? []).map((n) => ({
            id: n.id,
            name: n.name,
            itemStyle: { color: CATEGORY_COLOR[n.category] ?? '#999' },
            symbolSize: n.category === 'event' ? 46 : 34,
          })),
          edges: (kg?.edges ?? []).map((edge) => ({
            source: edge.source,
            target: edge.target,
            relation: edge.relation,
          })),
        },
      ],
    })
  }, [data])
}
```

```css
/* src/pages/RootCause/index.module.css */
.container { padding: 4px 0; }
.row { margin-top: 16px; }
.center { height: 50vh; display: grid; place-items: center; }
.chart { height: 420px; }
```

两处实现说明（初学者最容易在这里犯错）：

1. **"轮询完成"的副作用必须放 `useMemo`/`useEffect`，不能写在渲染体**：上面代码里 `if (taskQuery.data?.status === 'done')` 的空块就是为了引导你看清这一点——实际生效的是下方 `useMemo` 里的完成处理（刷新详情 + 提示 + 停轮询）。请把空块删除；
2. **`enabled` 双闸门**：详情查询 `enabled: Number.isFinite(eventId)`、轮询 `enabled: !!taskId && reanalyzing`——React Query 的 `enabled: false` 意味着"这个查询挂着但不启动"，是条件查询的官方姿势。

**【验证】** 从列表点击 ETA-301 事件进入 `/anomaly/1001`：

1. 事件卡片：标题、设备、批次、紧急红标签、触发规则齐全；
2. 三张嫌疑卡：#1 带红边框与"最可能根因"角标，置信度进度条 86% 红色；嫌疑参数展示"85 → 87.2 ℃（偏移 +2.2℃）"；
3. "参数时序对比"Tab：蓝色实际值围绕绿色虚线基线爬升漂移；
4. "知识图谱关系"Tab：6 个彩色节点（事件红/设备蓝/参数橙/缺陷紫/Recipe 青），边上有"发生于/监测参数/因果影响"等关系标注，可拖拽、可缩放；
5. "相似历史案例"：两条案例，相似度 83% 与 71%；"因果分析结论"：格兰杰因果检验文本；
6. 用 `admin` 登录时右上角有红色"重新分析"按钮，点击后按钮变为"分析中 xx%"每 2 秒跳动，4 秒左右（Mock 内置的进度逻辑）变为完成提示，页面数据刷新；用没有 `rootcause:reanalyze` 权限的账号登录则按钮不渲染（第 12 章将把裸的 hasPermission 判断升级为 `<AuthButton>`）。

---

## 11. 审批工作台

### 11.1 审批工作台：待审批列表 + 审批弹窗

业务规则（来自白皮书）：**智能体的调参建议必须经工程师审批**。审批动作 = 通过/驳回 + 必填理由（≥5 字）。

先写审批弹窗（页面私有组件）：

```tsx
// src/pages/Approval/ApprovalModal.tsx
import { useEffect } from 'react'
import { App, Form, Input, Modal, Radio, Space, Typography } from 'antd'
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { submitApproval } from '@/api/approval'
import type { ApprovalAction, ApprovalTask } from '@/types/approval'

interface Values {
  action: ApprovalAction
  reason: string
}

interface ApprovalModalProps {
  open: boolean
  task: ApprovalTask | null
  onClose: () => void
}

export default function ApprovalModal({ open, task, onClose }: ApprovalModalProps) {
  const { message } = App.useApp()
  const queryClient = useQueryClient()
  const [form] = Form.useForm<Values>()

  const mutation = useMutation({
    mutationFn: submitApproval,
    onSuccess: () => {
      message.success(task?.status === 'approved' ? '已通过' : '已驳回')
      // 关键：审批成功后让待办列表与看板统计同时过期刷新
      queryClient.invalidateQueries({ queryKey: ['approval'] })
      queryClient.invalidateQueries({ queryKey: ['dashboard'] })
      onClose()
    },
  })

  // 每次打开重置表单并预填动作
  useEffect(() => {
    if (open) {
      form.resetFields()
      form.setFieldsValue({ action: 'approve' })
    }
  }, [open, form])

  const handleOk = () => {
    form.validateFields().then((values) => {
      if (!task) return
      mutation.mutate({
        taskId: task.id,
        action: values.action,
        reason: values.reason,
      })
    })
  }

  return (
    <Modal
      open={open}
      title={`审批：${task?.eventTitle ?? ''}`}
      onCancel={onClose}
      onOk={handleOk}
      okText="提交"
      cancelText="取消"
      confirmLoading={mutation.isPending}
      destroyOnClose
    >
      <Typography.Paragraph type="secondary" style={{ fontSize: 13 }}>
        智能体建议：{task?.suggestion}
        {task && (
          <Typography.Text type={task.riskLevel === 'high' ? 'danger' : 'warning'}>
            （风险等级：{task.riskLevel === 'high' ? '高' : task.riskLevel === 'medium' ? '中' : '低'}）
          </Typography.Text>
        )}
      </Typography.Paragraph>

      <Form form={form} layout="vertical">
        <Form.Item name="action" rules={[{ required: true, message: '请选择审批动作' }]}>
          <Radio.Group>
            <Space>
              <Radio.Button value="approve">通过建议</Radio.Button>
              <Radio.Button value="reject" danger>驳回建议</Radio.Button>
            </Space>
          </Radio.Group>
        </Form.Item>
        <Form.Item
          name="reason"
          label="审批理由（必填，至少 5 字）"
          rules={[
            { required: true, message: '请填写审批理由' },
            { min: 5, message: '理由至少 5 个字，审计要求' },
          ]}
        >
          <Input.TextArea
            rows={3}
            placeholder="示例：同意下调腔温设定值，先跑 3 批验证 FDC 分数。"
            showCount
            maxLength={200}
          />
        </Form.Item>
      </Form>
    </Modal>
  )
}
```

工作台主体：

```tsx
// src/pages/Approval/index.tsx
import { useMemo, useState } from 'react'
import { useNavigate } from 'react-router-dom'
import { Button, Card, Table, Tag, Typography } from 'antd'
import { AuditOutlined } from '@ant-design/icons'
import { useQuery } from '@tanstack/react-query'
import { getApprovalTasks } from '@/api/approval'
import type { ApprovalTask } from '@/types/approval'
import ApprovalModal from './ApprovalModal'

const RISK_COLOR = { high: 'red', medium: 'orange', low: 'green' } as const
const RISK_LABEL = { high: '高', medium: '中', low: '低' } as const

export default function Approval() {
  const navigate = useNavigate()
  const [page, setPage] = useState(1)
  const [pageSize, setPageSize] = useState(20)
  const [modalOpen, setModalOpen] = useState(false)
  const [activeTask, setActiveTask] = useState<ApprovalTask | null>(null)

  const { data, isFetching } = useQuery({
    queryKey: ['approval', 'tasks', page, pageSize],
    queryFn: () => getApprovalTasks({ page, pageSize, status: 'pending' }),
  })

  const columns = useMemo(
    () => [
      { title: '任务 ID', dataIndex: 'id', width: 90 },
      { title: '关联事件', dataIndex: 'eventTitle', ellipsis: true,
        render: (t: string, r: ApprovalTask) => (
          <a onClick={() => navigate(`/anomaly/${r.eventId}`)}>{t}</a>
        ) },
      { title: '设备', dataIndex: 'equipmentCode', width: 110 },
      { title: '智能体建议', dataIndex: 'suggestion', ellipsis: true, width: 280 },
      {
        title: '风险', dataIndex: 'riskLevel', width: 80,
        render: (v: keyof typeof RISK_COLOR) => (
          <Tag color={RISK_COLOR[v]}>{RISK_LABEL[v]}</Tag>
        ),
      },
      { title: '创建时间', dataIndex: 'createdAt', width: 170 },
      {
        title: '操作', key: 'action', width: 110,
        render: (_: unknown, record: ApprovalTask) => (
          <Button
            type="primary"
            icon={<AuditOutlined />}
            size="small"
            onClick={() => { setActiveTask(record); setModalOpen(true) }}
          >
            审批
          </Button>
        ),
      },
    ],
    [navigate],
  )

  return (
    <Card size="small" title="待我审批的调参建议">
      <Table<ApprovalTask>
        rowKey="id"
        columns={columns}
        dataSource={data?.list}
        loading={isFetching}
        pagination={{
          current: page,
          pageSize,
          total: data?.total ?? 0,
          onChange: (p, ps) => { setPage(p); setPageSize(ps) },
        }}
      />

      <ApprovalModal
        open={modalOpen}
        task={activeTask}
        onClose={() => setModalOpen(false)}
      />
    </Card>
  )
}
```

**useMutation + invalidation 的连锁反应**是本页精髓：审批提交成功 → `['approval']` 失效（待办列表自动重拉，该任务消失）→ `['dashboard']` 失效（看板"待我审批"数字同步 -1）。**一个写操作驱动多处读视图刷新，而你没写任何刷新代码**——这就是 React Query 的收益。

### 11.2 审批历史：AntD Timeline 时间线

```tsx
// src/pages/ApprovalHistory/index.tsx
import { Card, Tag, Timeline, Typography } from 'antd'
import { useQuery } from '@tanstack/react-query'
import { getApprovalHistory } from '@/api/approval'

export default function ApprovalHistory() {
  const { data, isPending } = useQuery({
    queryKey: ['approval', 'history'],
    queryFn: getApprovalHistory,
  })

  return (
    <Card size="small" title="审批历史">
      <Timeline
        items={(data ?? []).map((h) => ({
          color: h.action === 'approve' ? 'green' : 'red',
          children: (
            <>
              <Typography.Text strong>
                {h.action === 'approve' ? '通过' : '驳回'} 审批任务 #{h.taskId}
              </Typography.Text>
              <Tag color={h.action === 'approve' ? 'success' : 'error'} style={{ marginLeft: 8 }}>
                {h.action === 'approve' ? 'APPROVED' : 'REJECTED'}
              </Tag>
              <div style={{ marginTop: 4 }}>理由：{h.reason}</div>
              <Typography.Text type="secondary" style={{ fontSize: 12 }}>
                操作人：{h.operator} · {h.operatedAt}
              </Typography.Text>
            </>
          ),
        }))}
      />
      {isPending && <Typography.Text type="secondary">加载中...</Typography.Text>}
    </Card>
  )
}
```

**【验证】** 访问 `/approval`：

1. 待办表格展示 ETA-301 审批任务，风险"中"橙色标签；点"审批"打开弹窗，显示智能体建议全文；
2. 理由只填 3 个字提交 → 表单校验红字"理由至少 5 个字"；理由为空 → "请填写审批理由"；
3. 选择"驳回建议"+ 合规理由提交 → 成功提示，弹窗关闭，列表刷新（任务从 Mock 状态变为 rejected 后不再出现在 pending 列表）；
4. 切到"审批历史"页 → Timeline 展示 Mock 的两条历史 + 你刚提交的记录会由真实后端持久化（Mock 下刷新后恢复初始状态，属预期）；
5. 审批成功后切回"良率看板"→ "待我审批"数字经 invalidate 自动刷新（若你审批的是唯一 Mock 待办，数字回到初始值是因为 Mock 数据不持久——真实后端无此问题）。

---

## 12. 权限控制：路由级 / 菜单级 / 按钮级

### 12.1 三层权限模型总结表

| 层级 | 拦截位置 | 实现机制 | 拦不住什么 |
|---|---|---|---|
| 路由级 | 进入页面前 | `AuthRoute` + 路由 meta.permission + `Navigate` 重定向 | 用户直接在页面内操作按钮 |
| 菜单级 | 侧边菜单渲染 | 菜单 = 路由投影时按 permission 过滤 | 用户手输 URL（由路由级兜底） |
| 按钮级 | 按钮渲染 | `<AuthButton permission="..."/>` 内部判断，无权返回 null | — |

> **安全提醒**：前端权限只是**体验层**的裁剪，真正的安全必须由后端接口二次校验（每个接口都判权限），否则改一下 JS 就能绕过。本教程的前端三层是"让界面与用户职责一致"，不是安全边界。

### 12.2 usePermission 自定义 hook

```ts
// src/hooks/usePermission.ts
import { useCallback } from 'react'
import { useAuthStore } from '@/store/authStore'

/**
 * 权限判断 hook。
 * 用法：const can = usePermission(); can('approval:approve') -> boolean
 * 或单个权限点便捷形式：const canApprove = usePermission('approval:approve')
 */
export function usePermission(permission?: string) {
  const hasPermission = useAuthStore((s) => s.hasPermission)
  const check = useCallback((p: string) => hasPermission(p), [hasPermission])
  return permission ? check(permission) : check
}
```

### 12.3 按钮级权限组件 AuthButton

```tsx
// src/components/AuthButton.tsx
import type { ButtonProps } from 'antd'
import { Button } from 'antd'
import { usePermission } from '@/hooks/usePermission'

interface AuthButtonProps extends ButtonProps {
  /** 需要的权限点，如 'approval:approve' */
  permission: string
}

/**
 * 按钮级权限组件：
 * - 有权限 -> 正常渲染按钮
 * - 无权限 -> 返回 null（不占位、不置灰；如需置灰样式见 props.disabled 提示）
 */
export default function AuthButton({ permission, ...rest }: AuthButtonProps) {
  const can = usePermission()
  if (!can(permission)) return null
  return <Button {...rest} />
}
```

两种无权表现的取舍：`null` 隐藏（干净，用户不知道存在此功能）vs `disabled + Tooltip`（提示"找管理员开通"，减少"功能去哪了"的疑问工单）。生产建议：**破坏性/机密操作用隐藏，常规高级功能用置灰提示**。置灰版实现：

```tsx
// src/components/AuthButton.tsx 置灰变体（二选一使用）
export function AuthButtonDisabled({ permission, ...rest }: AuthButtonProps) {
  const can = usePermission()
  return (
    <Tooltip title={can(permission) ? undefined : '暂无权限，请联系管理员'}>
      <Button disabled={!can(permission)} {...rest} />
    </Tooltip>
  )
}
```

### 12.4 路由级 + 菜单级权限接入

改造 `AuthRoute`（路由级）：

```tsx
// src/router/AuthRoute.tsx（升级版）
import { Navigate, Outlet, useLocation } from 'react-router-dom'
import { Result } from 'antd'
import { useAuthStore } from '@/store/authStore'
import { appRoutes } from '@/router'

export default function AuthRoute() {
  const token = useAuthStore((s) => s.token)
  const hasPermission = useAuthStore((s) => s.hasPermission)
  const location = useLocation()

  if (!token) {
    return <Navigate to="/login" replace state={{ from: location.pathname }} />
  }

  // 找到当前路径对应路由的 meta（动态路由取其前缀）
  const route = appRoutes.find(
    (r) =>
      location.pathname === r.path ||
      (r.path.includes(':') &&
        location.pathname.startsWith(r.path.split('/:')[0] + '/')),
  )

  if (route?.meta.permission && !hasPermission(route.meta.permission)) {
    return (
      <Result
        status="403"
        title="403"
        subTitle="抱歉，您没有该页面的访问权限。"
      />
    )
  }

  return <Outlet />
}
```

改造菜单构建（菜单级）——`src/router/menus.ts` 的 `buildMenu` 加权限过滤：

```ts
// src/router/menus.ts（在 buildMenu 中增加过滤）
import { useAuthStore } from '@/store/authStore'

export function buildMenu(routes: AppRouteObject[]): AppRouteObject[] {
  const { hasPermission } = useAuthStore.getState()
  const items = routes.filter(
    (r) =>
      !r.meta.hideInMenu &&
      (!r.meta.permission || hasPermission(r.meta.permission)),
  )
  MENU_ITEMS.length = 0
  MENU_ITEMS.push(...items)
  return items
}
```

同时给菜单补上权限变更的响应：菜单在登录/登出时需要重建。最简单的方案——`MainLayout` 里对 `user` 订阅触发重算：

```tsx
// src/layouts/MainLayout.tsx 中 menuItems 的 useMemo 增加依赖
const user = useAuthStore((s) => s.user)
const menuItems = useMemo(() => {
  buildMenu(appRoutes) // 权限可能随登录变化，重建菜单
  return MENU_ITEMS.map((item) => ({
    key: item.path,
    icon: ICON_MAP[item.meta.icon ?? ''],
    label: item.meta.title,
  }))
}, [user])
```

最后把审批页的按钮替换为 AuthButton（第 11 章的裸按钮升级）：

```tsx
// src/pages/Approval/index.tsx 中操作列渲染改为：
import AuthButton from '@/components/AuthButton'

render: (_: unknown, record: ApprovalTask) => (
  <AuthButton
    permission="approval:approve"
    type="primary"
    icon={<AuditOutlined />}
    size="small"
    onClick={() => { setActiveTask(record); setModalOpen(true) }}
  >
    审批
  </AuthButton>
)
```

**【验证】** 权限链路验证（用两个账号对照）：

1. `admin` 登录 → 菜单五项全有，审批页"审批"按钮可见；
2. 退出，用 `engineer` 登录（Mock 中该角色 permissions 为 `['approval:approve','rootcause:reanalyze','anomaly:view']`，注意没有 `approval:view`——请先在 Mock 登录处理里把 engineer 的 permissions 加上 `'approval:view'` 再做此验证）：
   - 菜单"审批工作台/审批历史"可见；
   - 临时把 Mock 中 engineer 的 `approval:view` 移除、刷新 token（退出重登）→ 菜单里审批两项消失；手输 URL `/approval` → 显示 403 Result 页；
3. `engineer` 进入根因详情 → "重新分析"按钮可见（有 `rootcause:reanalyze`）；临时移除该权限点重登 → 按钮消失；
4. 三层权限各拦截一次即完成验证：菜单级（看不见）、路由级（403 页）、按钮级（按钮消失）。

---

## 13. 全局状态管理进阶

### 13.1 Zustand 多 store 划分原则

按"**状态归属域**"划分 store，而不是把所有状态塞进一个大 store（那是 Redux 时代的痛点）：

| Store | 职责 | 内容 | 持久化 |
|---|---|---|---|
| `authStore` | 登录态与身份 | token、user、permissions | localStorage（persist） |
| `uiStore` | 界面偏好 | 主题、菜单折叠、未来：窗口尺寸、全局 loading | localStorage（persist） |
| `dictStore` | 业务字典 | 工序列表、设备类型等低频字典（请求一次全端复用） | sessionStorage（可选） |
| `realtimeStore` | 实时推送 | WebSocket 消息缓冲、未读计数 | 不持久化（刷新即清） |

**三条划分纪律**：

1. **服务端状态不进 Zustand**——列表、详情、统计数据一律 React Query（见 13.2）；
2. **跨组件但不跨页面的状态不进 store**——组件内部 state 或提升到最近公共父组件即可，store 是"最后手段"；
3. **store 之间不互相 import**——需要组合时在组件层组合（避免状态网的循环依赖）。

补齐 `dictStore` 示例（低频字典的全局缓存模式）：

```ts
// src/store/dictStore.ts
import { create } from 'zustand'
import { PROCESS_STEPS } from '@/utils/constants'

interface DictState {
  processSteps: string[]
  equipmentTypes: string[]
  /** 载入字典（App 启动时调用一次） */
  loadDicts: () => void
}

export const useDictStore = create<DictState>((set) => ({
  processSteps: PROCESS_STEPS, // 静态字典直接给
  equipmentTypes: ['刻蚀机', 'CVD', '光刻机', '离子注入机', 'CMP', '扩散炉'],
  loadDicts: () => {
    // 动态字典：真实后端存在 DB 中的场景，在此请求并 set
    set({ processSteps: PROCESS_STEPS })
  },
}))
```

> 思考题：字典为什么不用 React Query？其实可以——`staleTime: Infinity` 的 useQuery 就是内存缓存。两种方案都正确，团队统一即可。本教程将静态字典放 store、动态字典留给读者用 React Query 实现（练习 13-1）。

### 13.2 与 React Query 的分工

一张表彻底说清（打印出来贴工位）：

| 判断问题 | 归 Zustand | 归 React Query |
|---|---|---|
| 数据的"真相源"在前端还是后端？ | 前端 | 后端 |
| 数据会被用户操作修改吗？ | 会（主题、折叠） | 只会"重新获取" |
| 需要缓存/去重/重试/轮询吗？ | 不需要 | 需要 |
| 多个组件要同一数据且要求一致？ | 手动保证 | queryKey 自动共享 |
| 典型案例 | themeMode、collapsed、unreadCount | events 列表、overview 看板、详情 |

**反模式警示**：把 `getAnomalyEvents` 的结果存进 Zustand（`set({ events })`）然后手动维护 loading、total、翻页——等于手写了一个残血版 React Query，团队协作中没人能维护明白。

### 13.3 Selector 订阅：避免无关渲染

Zustand 每次更新会触发订阅了该 store 的组件重渲染，**必须用 selector 精确订阅**：

```tsx
// ✅ 正确：只订阅 token 变化
const token = useAuthStore((s) => s.token)

// ❌ 错误：订阅整个 store，任何字段变化都重渲染
const auth = useAuthStore()

// ⚠️ selector 返回新对象时每次都是新引用，需要 shallow 比较
import { useShallow } from 'zustand/react/shallow'
const { token, user } = useAuthStore(useShallow((s) => ({ token: s.token, user: s.user })))
```

本章前面所有代码均已按 selector 规范书写，此处集中说明原理：Zustand 内部用 `useSyncExternalStore` 比较 selector 结果引用，引用变了才重渲染——返回新对象/数组就永远"变了"，这就是需要 `useShallow` 的原因。

**【验证】** React DevTools Profiler 中：点击主题切换，只有 `App`、`MainLayout` 等真正用 themeMode 的组件高亮重渲染；切换主题时看板页若打开着，其内部业务组件不渲染（它们没订阅 themeMode）。

---

## 14. 前端测试：Vitest + RTL

### 14.1 环境搭建

```bash
pnpm add -D vitest @testing-library/react @testing-library/user-event @testing-library/jest-dom jsdom
```

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: { alias: { '@': path.resolve(__dirname, 'src') } },
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    css: false, // CSS Modules 测试时返回空对象即可
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
  },
})
```

```ts
// src/test/setup.ts
import '@testing-library/jest-dom/vitest'

// jsdom 没有 matchMedia/ResizeObserver，AntD 与 ECharts 需要
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: (query: string) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: () => {},
    removeListener: () => {},
    addEventListener: () => {},
    removeEventListener: () => {},
    dispatchEvent: () => false,
  }),
})

class MockResizeObserver {
  observe() {}
  unobserve() {}
  disconnect() {}
}
window.ResizeObserver = MockResizeObserver as unknown as typeof ResizeObserver
```

`package.json` 增加 scripts：

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

### 14.2 工具函数测试

先补两个会被测试的格式化工具（之前没建过，正好用得上）：

```ts
// src/utils/format.ts
import dayjs from 'dayjs'

/** 百分比格式化：0.932 -> "93.2%" */
export function toPercent(v: number, digits = 1): string {
  return `${(v * 100).toFixed(digits)}%`
}

/** 时间格式化：ISO -> "2026-10-15 09:12:00"（东八区展示统一入口） */
export function formatTime(iso: string, pattern = 'YYYY-MM-DD HH:mm:ss'): string {
  return dayjs(iso).format(pattern)
}

/** 置信度 -> 文案：0.86 -> "86%" */
export function formatConfidence(c: number): string {
  return `${Math.round(c * 100)}%`
}
```

```ts
// src/utils/format.test.ts
import { describe, expect, it } from 'vitest'
import { formatConfidence, formatTime, toPercent } from './format'

describe('toPercent', () => {
  it('0.932 保留 1 位小数', () => {
    expect(toPercent(0.932)).toBe('93.2%')
  })
  it('支持自定义小数位', () => {
    expect(toPercent(0.9, 2)).toBe('90.00%')
  })
})

describe('formatTime', () => {
  it('ISO 字符串转标准格式', () => {
    expect(formatTime('2026-10-15T09:12:00+08:00')).toBe('2026-10-15 09:12:00')
  })
  it('支持自定义模板', () => {
    expect(formatTime('2026-10-15T09:12:00+08:00', 'MM-DD')).toBe('10-15')
  })
})

describe('formatConfidence', () => {
  it('0.86 -> 86%（四舍五入）', () => {
    expect(formatConfidence(0.86)).toBe('86%')
  })
  it('0.855 -> 86%（round 语义）', () => {
    expect(formatConfidence(0.855)).toBe('86%')
  })
})
```

### 14.3 组件测试：登录表单校验

测试思路（RTL 心法）：**像用户一样行为，不测内部实现**。用户视角的登录表单场景：不填表单点登录 → 出现两条校验提示；只填用户名 → 只提示密码；都填则触发登录请求。

```tsx
// src/pages/Login/index.test.tsx
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { MemoryRouter } from 'react-router-dom'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { App as AntdApp, ConfigProvider } from 'antd'
import Login from './index'
import { useAuthStore } from '@/store/authStore'
import * as authApi from '@/api/auth'

/** App 级别 Provider 包装（生产装配的测试复刻） */
function renderLogin() {
  const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } })
  return render(
    <ConfigProvider>
      <AntdApp>
        <QueryClientProvider client={qc}>
          <MemoryRouter initialEntries={['/login']}>
            <Login />
          </MemoryRouter>
        </QueryClientProvider>
      </AntdApp>
    </ConfigProvider>,
  )
}

describe('登录表单校验', () => {
  beforeEach(() => {
    // 每个用例重置登录态，避免 persist 残留导致"已登录跳转"
    useAuthStore.setState({ token: null, user: null })
  })

  it('初始渲染出用户名/密码输入框与登录按钮', () => {
    renderLogin()
    expect(screen.getByPlaceholderText(/用户名/)).toBeInTheDocument()
    expect(screen.getByPlaceholderText('密码')).toBeInTheDocument()
    expect(screen.getByRole('button', { name: /登 录/ })).toBeInTheDocument()
  })

  it('清空必填项后提交，出现两条校验提示', async () => {
    const user = userEvent.setup()
    renderLogin()
    const inputs = document.querySelectorAll('input')
    // 清空默认值（initialValues 预填了 admin/admin123）
    await user.clear(inputs[0])
    await user.clear(inputs[1])
    await user.click(screen.getByRole('button', { name: /登 录/ }))
    expect(await screen.findByText('请输入用户名')).toBeInTheDocument()
    expect(screen.getByText('请输入密码')).toBeInTheDocument()
  })

  it('密码少于 6 位出现长度提示', async () => {
    const user = userEvent.setup()
    renderLogin()
    const inputs = document.querySelectorAll('input')
    await user.clear(inputs[1])
    await user.type(inputs[1], '123')
    await user.click(screen.getByRole('button', { name: /登 录/ }))
    expect(await screen.findByText('密码至少 6 位')).toBeInTheDocument()
  })

  it('合法输入提交会调用 login API', async () => {
    const user = userEvent.setup()
    const spy = vi.spyOn(authApi, 'login').mockResolvedValue({
      token: 'test-token',
      user: {
        id: 1, username: 'admin', realName: '测试员',
        role: 'admin', permissions: [],
      },
    })
    renderLogin()
    await user.click(screen.getByRole('button', { name: /登 录/ }))
    await vi.waitFor(() => expect(spy).toHaveBeenCalledTimes(1))
    expect(spy).toHaveBeenCalledWith({ username: 'admin', password: 'admin123' })
    // 登录成功后 token 已写入 store
    await vi.waitFor(() =>
      expect(useAuthStore.getState().token).toBe('test-token'),
    )
  })
})
```

### 14.4 组件测试：审批弹窗交互

覆盖三个关键行为：理由校验必填、驳回理由过短被拦截、提交成功后关闭弹窗并触发 invalidation。

```tsx
// src/pages/Approval/ApprovalModal.test.tsx
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { App as AntdApp, ConfigProvider } from 'antd'
import ApprovalModal from './ApprovalModal'
import * as approvalApi from '@/api/approval'
import type { ApprovalTask } from '@/types/approval'

const task: ApprovalTask = {
  id: 5001,
  eventId: 1001,
  eventTitle: 'ETA-301 腔体温度漂移导致良率下降',
  equipmentCode: 'ETA-301',
  suggestion: '建议将腔温设定值由 85.0℃ 下调至 83.5℃。',
  riskLevel: 'medium',
  status: 'pending',
  createdAt: '2026-10-15 09:30:00',
}

function renderModal(onClose = vi.fn()) {
  const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } })
  const utils = render(
    <ConfigProvider>
      <AntdApp>
        <QueryClientProvider client={qc}>
          <ApprovalModal open task={task} onClose={onClose} />
        </QueryClientProvider>
      </AntdApp>
    </ConfigProvider>,
  )
  return { ...utils, qc, onClose }
}

describe('审批弹窗', () => {
  beforeEach(() => vi.restoreAllMocks())

  it('渲染智能体建议与风险等级', () => {
    renderModal()
    expect(screen.getByText(/建议将腔温设定值由 85.0℃ 下调至 83.5℃/)).toBeInTheDocument()
    expect(screen.getByText('风险等级：中')).toBeInTheDocument()
  })

  it('理由为空提交被校验拦截', async () => {
    const user = userEvent.setup()
    renderModal()
    await user.click(screen.getByRole('button', { name: '提 交' }))
    expect(await screen.findByText('请填写审批理由')).toBeInTheDocument()
  })

  it('理由过短（<5字）被审计规则拦截', async () => {
    const user = userEvent.setup()
    renderModal()
    const textarea = screen.getByPlaceholderText(/同意下调腔温设定值/)
    await user.type(textarea, '同意')
    await user.click(screen.getByRole('button', { name: '提 交' }))
    expect(await screen.findByText(/理由至少 5 个字/)).toBeInTheDocument()
  })

  it('驳回提交成功后关闭弹窗并失效审批缓存', async () => {
    const user = userEvent.setup()
    const submitSpy = vi
      .spyOn(approvalApi, 'submitApproval')
      .mockResolvedValue({ ok: true })
    const { qc, onClose } = renderModal()
    vi.spyOn(qc, 'invalidateQueries')

    await user.click(screen.getByRole('radio', { name: '驳回建议' }))
    await user.type(screen.getByPlaceholderText(/同意下调腔温设定值/), '风险高，需先做DOE实验')
    await user.click(screen.getByRole('button', { name: '提 交' }))

    await vi.waitFor(() =>
      expect(submitSpy).toHaveBeenCalledWith({
        taskId: 5001,
        action: 'reject',
        reason: '风险高，需先做DOE实验',
      }),
    )
    // 审批缓存被失效（invalidation 被调用，key 前缀 approval）
    await vi.waitFor(() =>
      expect(qc.invalidateQueries).toHaveBeenCalledWith({ queryKey: ['approval'] }),
    )
    expect(onClose).toHaveBeenCalled()
  })
})
```

测试心法三条（比代码更值钱）：

1. **测行为不测实现**：断言"出现提示文案"，不断言"state.error === true"——重构组件内部不破坏测试；
2. **Provider 包装函数**：JSX Provider 嵌套抽成 `renderXxx` 工具，避免每个文件复制十层；
3. **API 层 mock 边界**：`vi.spyOn(authApi, 'login')` 打在 api 模块函数上——请求层以下（axios）不进单测范围（那是集成测试的事）。

**【验证】** `pnpm test` 全绿（当前共 4 + 5 个用例）。故意把登录校验规则的 `min: 6` 改成 `min: 3`，重跑测试应红——证明测试真的在保护代码；改回后恢复全绿。

---

## 15. 构建与部署

### 15.1 切换真实后端联调

前面所有功能都跑在 Mock 上，联调真实后端（IA-TU-001 的 Egg.js 服务）只改一处：

```bash
# .env.development 无需改；启动命令加环境变量关闭 Mock：
USE_MOCK=false pnpm dev        # 代理到 http://localhost:7001
# 或代理到其他环境：
USE_MOCK=false API_TARGET=http://10.0.8.42:7001 pnpm dev
```

Mock 插件未注册，`/api/v1` 全量走 Vite proxy → 后端。**前端零代码改动完成切换**——这正是"接口约定先行 + 请求层统一封装"的回报。

### 15.2 构建产物分析

```bash
pnpm add -D rollup-plugin-visualizer
```

```ts
// vite.config.ts 的 build 配置中临时加：
import { visualizer } from 'rollup-plugin-visualizer'

plugins: [
  react(),
  ...(USE_MOCK ? [viteMockPlugin()] : []),
  visualizer({ open: true, gzipSize: true, filename: 'stats.html' }),
]
```

执行 `pnpm build` 后自动打开 `stats.html`：确认 `react`/`antd`/`echarts` 三个 vendor chunk 与业务 chunk 分离，且 echarts 体积是否因按需引入而明显小于全量（全量约 1MB+，按需后核心场景约 300-400KB gzip 前）。**验收标准：任何单个业务页面 chunk 不超过 500KB**。

### 15.3 Nginx 配置（history 路由 + API 反代 + WS）

```nginx
# nginx.conf
server {
    listen 80;
    server_name industryagent.example.com;

    # 前端静态产物
    root /usr/share/nginx/html;
    index index.html;

    # ---------- 核心：history 路由回退 ----------
    # React Router 的 /anomaly/1001 在服务器上没有对应文件，
    # 必须回退到 index.html 由前端路由接管，否则刷新 404
    location / {
        try_files $uri $uri/ /index.html;
    }

    # ---------- API 反向代理 ----------
    location /api/v1/ {
        proxy_pass http://backend:7001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 60s;
    }

    # ---------- WebSocket 反代 ----------
    location /ws/v1/ {
        proxy_pass http://backend:7001;
        proxy_http_version 1.1;
        # WS 升级头：少这两行必失败
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 300s;   # 大于前端心跳间隔(30s)，防空闲断连
        proxy_send_timeout 300s;
    }

    # 静态资源长缓存（文件名含 hash）
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### 15.4 Dockerfile 多阶段构建

```dockerfile
# Dockerfile
# ====== 阶段1：构建（node 环境） ======
FROM node:20-alpine AS builder
WORKDIR /app

# 先拷贝依赖清单再安装：利用 Docker 层缓存
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

# 再拷源码构建
COPY . .
RUN pnpm build

# ====== 阶段2：运行（nginx 环境，只带静态产物） ======
FROM nginx:1.27-alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost/ >/dev/null || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

多阶段构建的价值：**运行镜像不含 node_modules**（否则 800MB 起步），最终镜像约 30-50MB，且源代码不出现在生产镜像里。

`docker-compose.yml`（前端 + 后端一起拉起，联调利器）：

```yaml
services:
  web:
    build: .
    ports:
      - "8080:80"
    depends_on:
      - backend
  backend:
    build: ../industryagent-server   # 后端教程产物路径
    ports:
      - "7001:7001"
```

### 15.5 部署后验证清单

```bash
pnpm build                    # 本地产物检查（dist/ 生成）
docker compose up -d --build  # 容器化起整套
```

**【验证】** 浏览器访问 `http://localhost:8080`：

1. 登录 → 各页面正常，直接刷新 `/anomaly/1001` **不 404**（try_files 生效，这是部署验收第一优先级）；
2. Network 面板：API 请求打到 `localhost:8080/api/v1/...` 且 200（Nginx 反代生效）；
3. WebSocket 连接状态 101（WS 反代生效），Console 无握手错误；
4. 静态资源 Response Headers 含 `Cache-Control: public, immutable`；
5. `docker exec <容器> ls /usr/share/nginx/html` 只有 dist 内容，无源码。

---

## 19. 补全页面：8D 报告 / 审计日志 / 系统管理 / 通知中心 / 缺陷查询

> **本章定位（V1.1 增补说明）**：第 1~15 章完成了平台的主线页面（看板 → 异常 → 根因 → 审批），但对照《前端详细设计说明书（IA-SD-003）》与《高保真原型》会发现，原型的侧边栏还有"8D 报告、审计日志、系统管理"三组菜单，顶栏还有一个真正的"通知中心"，缺陷数据也需要一个独立查询入口。本章把这几块全部补齐，读完本章，系统功能 = 原型全部页面。
>
> **阅读顺序说明**：本章内容属于"页面开发主线"，知识上只依赖第 1~12 章（请求层、路由、React Query、权限），建议初学者在第 12 章之后、第 13 章之前阅读。为保持已发布章节的编号与外部引用锚点稳定（避免"第 13 章变成第 18 章"式的全书重排），增补章节统一使用 19 号，物理位置插在部署（15）之后、性能优化（16）之前——已经按顺序读到这里的读者，把它当作"回到页面主线补完最后五个页面"即可。

### 19.0 增补前的统一改造：角色与权限扩展

五个新页面引入了新的角色与权限点，先统一改好，后面每节直接引用。

**① 角色扩展**：原型登录页有四种角色（工艺/设备/质量/管理员），而 `types/auth.ts` 目前只有三种。补上"质量工程师"——8D 报告的编制、编辑、签发都归这个角色：

```ts
// src/types/auth.ts（RoleCode 扩展为四种）
export type RoleCode = 'admin' | 'process_engineer' | 'equipment_engineer' | 'quality_engineer'
```

**② 角色文案常量**：加入 `src/utils/constants.ts`（与前文 SEVERITY_MAP 同文件，追加在文件末尾）：

```ts
// src/utils/constants.ts 追加
import type { RoleCode } from '@/types/auth'

/** 角色码 -> 显示名（顶栏 Tag、用户管理表格复用） */
export const ROLE_LABEL: Record<RoleCode, string> = {
  admin: '管理员',
  process_engineer: '工艺工程师',
  equipment_engineer: '设备工程师',
  quality_engineer: '质量工程师',
}
```

**③ Mock 登录支持三套演示账号**：替换 `src/mock/plugin.ts` 中的登录分支（`ROUTE_PRESETS` 把第 12 章演示用的零散权限收敛成一张表，`report:view` 是本章新增的只读权限点）：

```ts
  // ---- 登录（V1.1：三套演示账号） ----
  if (pathname === '/api/v1/auth/login' && M === 'POST') {
    const { username } = body as LoginParams
    const ROLE_PRESETS: Record<string, { role: string; realName: string; permissions: string[] }> = {
      admin: { role: 'admin', realName: '系统管理员', permissions: [] },
      engineer: {
        role: 'process_engineer', realName: '王工艺',
        permissions: ['anomaly:view', 'approval:view', 'approval:approve', 'rootcause:reanalyze', 'report:view'],
      },
      quality: {
        role: 'quality_engineer', realName: '赵敏',
        permissions: ['anomaly:view', 'approval:view', 'report:view', 'report:edit', 'quality:release'],
      },
    }
    const preset = ROLE_PRESETS[username] ?? ROLE_PRESETS.engineer
    return send(res, {
      token: `mock-token-${preset.role}-${Date.now()}`,
      user: { id: 1, username, realName: preset.realName, role: preset.role, permissions: preset.permissions },
    })
  }
```

> `admin` 角色的 `permissions` 为空数组依旧成立——`hasPermission` 里 `role === 'admin'` 直接放行（见 4.3），"管理员拥有一切"的语义不变。

**④ 顶栏角色 Tag 显示中文**：`MainLayout.tsx` 中用户名的 Tag 一并升级（顺手把原始角色码换成中文，原型顶栏即如此）：

```tsx
// src/layouts/MainLayout.tsx 中用户区 Tag 替换为：
{user && <Tag color={user.role === 'admin' ? 'gold' : 'blue'}>{ROLE_LABEL[user.role]}</Tag>}
```

**⑤ 登录页演示账号文案**：`src/pages/Login/index.tsx` 中两处提示改为三账号：

```tsx
<Input prefix={<UserOutlined />} placeholder="用户名（admin / engineer / quality）" size="large" />
```

```tsx
演示账号：admin（管理员）/ engineer（工艺工程师）/ quality（质量工程师，可编辑与签发 8D）
```

本章新增权限点一览（后面各节按下表使用）：

| 权限点 | 拥有者 | 控制什么 |
|---|---|---|
| `report:view` | engineer / quality | 8D 报告菜单与页面可见 |
| `report:edit` | 仅 quality | 报告详情"编辑"按钮（且仅草稿状态） |
| `quality:release` | 仅 quality | 报告"签发"按钮（Popconfirm 二次确认） |
| `admin:manage` | 仅 admin | 审计日志、系统管理两个菜单与页面 |

**【验证】** `pnpm dev` 后分别用三个账号登录：

1. `admin` → 菜单与原来一致（此时新菜单还没加，属预期），顶栏 Tag 显示"管理员"；
2. `quality` → 顶栏显示"赵敏 · 质量工程师"；`engineer` → "王工艺 · 工艺工程师"；
3. 登录页占位文案已变为三账号提示。

---

### 19.1 8D 报告页：报告列表 + 报告详情（D1~D8）

**业务背景**（来自白皮书与原型）：审批通过的处置建议执行后，报告 Agent 自动生成 8D 报告（8 Disciplines：成立小组 → 问题描述 → 临时围堵 → 根因分析 → 永久措施 → 效果验证 → 预防再发 → 关闭表彰）。报告的完整生命周期是：**报告 Agent 生成草稿（draft）→ 质量工程师复核（pending_review）→ 质量工程师签发（released）**。签发后的报告不可再编辑，可打印归档（PDF）。

本节交付两个页面：

| 页面 | 路由 | 要点 |
|---|---|---|
| 报告列表 | `/reports` | 状态 Tab（全部/草稿/待复核/已签发）+ 分页表格 |
| 报告详情 | `/reports/:id` | D1~D8 锚点导航；草稿可编辑（质量工程师）；签发 Popconfirm；window.print 打印 |

#### 19.1.1 类型定义与 api 模块

```ts
// src/types/report.ts
import type { PageQuery } from './api'

/** 8D 报告状态：草稿 -> 待复核 -> 已签发（签发后不可编辑） */
export type ReportStatus = 'draft' | 'pending_review' | 'released'

/** D1 团队成员 */
export interface D1TeamMember {
  /** 姓名 */
  name: string
  /** 组内角色：组长 / 设备 / 质量 / 生产... */
  role: string
  /** 所属部门 */
  dept: string
}

/** 报告列表项（列表页只展示概要字段） */
export interface ReportListItem {
  id: number
  /** 报告编号，如 8D-2026-0012 */
  reportNo: string
  title: string
  /** 关联异常事件 ID（跳根因详情用） */
  eventId: number
  equipmentCode: string
  lotNo: string
  /** 编制人；报告 Agent 生成时为 agent:report */
  owner: string
  status: ReportStatus
  createdAt: string
  releasedAt?: string
}

/** 报告详情（D1~D8 全量字段） */
export interface Report8D {
  id: number
  reportNo: string
  title: string
  status: ReportStatus
  owner: string
  createdAt: string
  releasedAt?: string
  /** 关联事件概要（详情页头部展示） */
  event: {
    id: number
    title: string
    equipmentCode: string
    lotNo: string
    processStep: string
    occurredAt: string
  }
  /** D1 成立小组 */
  d1Team: D1TeamMember[]
  /** D2 问题描述（5W2H） */
  d2Problem: string
  /** D3 临时围堵措施 */
  d3Containment: string
  /** D4 根本原因分析（引用证据链编号） */
  d4RootCause: string
  /** D5 永久纠正措施 */
  d5Corrective: string
  /** D6 措施实施与效果验证 */
  d6Verification: string
  /** D7 预防再发 */
  d7Prevention: string
  /** D8 团队祝贺与关闭 */
  d8Closure: string
}

/** 列表查询参数 */
export interface ReportQueryParams extends PageQuery {
  status?: ReportStatus
}

/** 编辑保存参数（PUT body）：只有 D1~D8 业务字段，状态流转走独立接口 */
export interface ReportUpdateParams {
  d1Team: D1TeamMember[]
  d2Problem: string
  d3Containment: string
  d4RootCause: string
  d5Corrective: string
  d6Verification: string
  d7Prevention: string
  d8Closure: string
}
```

```ts
// src/api/report.ts
import { http } from './request'
import type { PageResult } from '@/types/api'
import type { Report8D, ReportListItem, ReportQueryParams, ReportUpdateParams } from '@/types/report'

export const getReports = (params: ReportQueryParams) =>
  http.get<PageResult<ReportListItem>>('/reports', params)

export const getReportDetail = (id: number) =>
  http.get<Report8D>(`/reports/${id}`)

/** 草稿保存：仅 D1~D8 字段（PUT 全量替换文本区） */
export const updateReport = (id: number, data: ReportUpdateParams) =>
  http.put<{ ok: boolean }>(`/reports/${id}`, data)

/** 签发：状态 -> released，签发后锁定不可编辑 */
export const releaseReport = (id: number) =>
  http.post<{ ok: boolean }>(`/reports/${id}/release`)
```

状态映射常量（追加到 `src/utils/constants.ts`，与 STATUS_MAP 同风格）：

```ts
// src/utils/constants.ts 追加（文件顶部 import 补 ReportStatus）
import type { ReportStatus } from '@/types/report'

/** 8D 报告状态 -> 标签配置 */
export const REPORT_STATUS_MAP: Record<ReportStatus, { color: string; label: string }> = {
  draft: { color: 'default', label: '草稿' },
  pending_review: { color: 'warning', label: '待复核' },
  released: { color: 'success', label: '已签发' },
}
```

#### 19.1.2 报告列表页

```tsx
// src/pages/Reports/index.tsx
import { useMemo, useState } from 'react'
import { useNavigate } from 'react-router-dom'
import { Card, Empty, Table, Tabs, Tag, Typography } from 'antd'
import { keepPreviousData, useQuery } from '@tanstack/react-query'
import { getReports } from '@/api/report'
import type { ReportListItem, ReportStatus } from '@/types/report'
import { REPORT_STATUS_MAP, ROLE_LABEL } from '@/utils/constants'
import styles from './index.module.css'

export default function Reports() {
  const navigate = useNavigate()
  /* ---------- 状态 Tab + 受控分页（同 9.4 范式） ---------- */
  const [status, setStatus] = useState<ReportStatus | 'all'>('all')
  const [page, setPage] = useState(1)
  const [pageSize, setPageSize] = useState(20)

  const { data, isFetching } = useQuery({
    queryKey: ['reports', status, page, pageSize],
    queryFn: () => getReports({ page, pageSize, status: status === 'all' ? undefined : status }),
    placeholderData: keepPreviousData, // 翻页/切 Tab 保留旧数据不闪空
  })

  /** 切 Tab 强制回第一页（否则"第 3 页 + 新状态"可能请求出空列表，见 9.4） */
  const onTabChange = (key: string) => {
    setStatus(key as ReportStatus | 'all')
    setPage(1)
  }

  const columns = useMemo(
    () => [
      {
        title: '报告编号',
        dataIndex: 'reportNo',
        width: 150,
        render: (v: string, record: ReportListItem) => (
          <a onClick={() => navigate(`/reports/${record.id}`)}>{v}</a>
        ),
      },
      { title: '标题', dataIndex: 'title', ellipsis: true },
      { title: '设备', dataIndex: 'equipmentCode', width: 110 },
      { title: '批次', dataIndex: 'lotNo', width: 130 },
      {
        title: '编制人',
        dataIndex: 'owner',
        width: 130,
        render: (v: string) =>
          v.startsWith('agent:') ? (
            <Tag color="purple">智能体</Tag>
          ) : (
            v
          ),
      },
      {
        title: '状态',
        dataIndex: 'status',
        width: 100,
        render: (v: ReportStatus) => {
          const conf = REPORT_STATUS_MAP[v]
          return <Tag color={conf.color}>{conf.label}</Tag>
        },
      },
      { title: '创建时间', dataIndex: 'createdAt', width: 170 },
      {
        title: '签发时间',
        dataIndex: 'releasedAt',
        width: 170,
        render: (v?: string) => v ?? <Typography.Text type="secondary">—</Typography.Text>,
      },
      {
        title: '操作',
        key: 'action',
        width: 90,
        render: (_: unknown, record: ReportListItem) => (
          <a onClick={() => navigate(`/reports/${record.id}`)}>查看</a>
        ),
      },
    ],
    [navigate],
  )

  return (
    <Card size="small" className={styles.container}>
      <Tabs
        activeKey={status}
        onChange={onTabChange}
        items={[
          { key: 'all', label: '全部' },
          { key: 'draft', label: '草稿' },
          { key: 'pending_review', label: '待复核' },
          { key: 'released', label: '已签发' },
        ]}
      />
      <Table<ReportListItem>
        rowKey="id"
        columns={columns}
        dataSource={data?.list}
        loading={isFetching}
        scroll={{ x: 1200 }}
        locale={{ emptyText: <Empty description="暂无 8D 报告" /> }}
        pagination={{
          current: page,
          pageSize,
          total: data?.total ?? 0,
          showSizeChanger: true,
          showTotal: (t) => `共 ${t} 份`,
          onChange: (p, ps) => { setPage(p); setPageSize(ps) },
        }}
      />
    </Card>
  )
}
```

```css
/* src/pages/Reports/index.module.css */
.container :global(.ant-tabs-nav) {
  margin-bottom: 12px;
}
```

> `owner.startsWith('agent:')` 的智能体标签不是花活——审计日志、报告列表都会出现 `agent:report` / `agent:rootcause` 这类"机器操作人"，视觉上必须与人区分（原型同样用紫色 Tag 标注智能体）。

#### 19.1.3 报告详情页：D1~D8 锚点导航 + 草稿编辑 + 签发 + 打印

详情页是本节主角，先看结构图：

```text
┌────────────────────────────────────────────────────────────┐
│ ← 返回  8D-2026-0012 [待复核]     [编辑] [签发] [打印/PDF]     │ ← 操作栏（打印时隐藏）
├──────────────┬─────────────────────────────────────────────┤
│ 锚点导航       │ 事件概要卡片（关联事件/设备/批次/工序/发生时间）      │
│ D1 成立小组    ├─────────────────────────────────────────────┤
│ D2 问题描述    │ D1 团队表格（查看态）/ Form.List（编辑态）        │
│ D3 临时围堵    │ D2~D8 文本卡片（查看态 Descriptions / 编辑态      │
│ ...          │       TextArea）                             │
│ D8 关闭表彰    │ 页脚：生成时间与证据引用说明                      │
└──────────────┴─────────────────────────────────────────────┘
```

三个交互规则（权限 × 状态双闸门，缺一不可）：

1. **编辑**：`status === 'draft'` **且** 当前用户有 `report:edit` 才显示"编辑"按钮——状态与权限是两个维度：已签发报告对质量工程师同样锁定；
2. **签发**：有 `quality:release` 且未签发时显示，`Popconfirm` 二次确认（签发是发布动作，误触不可逆）；
3. **打印**：`window.print()` + `@media print` 样式——选它而不是 html2canvas 的理由：零依赖、文字可选中可检索（PDF 由浏览器打印对话框"另存为 PDF"产出），且矢量不糊。html2canvas 截图方案会产生位图 PDF、体积大且不可检索，仅在"像素级还原暗色主题"需求下才值得。

```tsx
// src/pages/Reports/ReportDetail.tsx
import { useMemo, useState } from 'react'
import { useNavigate, useParams } from 'react-router-dom'
import {
  Anchor, App, Button, Card, Descriptions, Form, Input, Popconfirm, Result, Space, Spin, Table, Tag, Typography,
} from 'antd'
import {
  ArrowLeftOutlined, CheckCircleOutlined, EditOutlined, MinusCircleOutlined,
  PlusOutlined, PrinterOutlined, SaveOutlined,
} from '@ant-design/icons'
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { getReportDetail, releaseReport, updateReport } from '@/api/report'
import { REPORT_STATUS_MAP } from '@/utils/constants'
import { usePermission } from '@/hooks/usePermission'
import type { Report8D, ReportUpdateParams } from '@/types/report'
import styles from './index.module.css'
// 打印样式是全局生效的普通 CSS（不是 module），配合 window.print()
import './print.css'

/** D2~D8 步骤元信息：key 与 ReportUpdateParams 字段一一对应 */
const TEXT_STEPS: { key: keyof Omit<ReportUpdateParams, 'd1Team'>; code: string; title: string; hint: string }[] = [
  { key: 'd2Problem', code: 'D2', title: '问题描述', hint: '5W2H：何时何地何人何事，量化偏差' },
  { key: 'd3Containment', code: 'D3', title: '临时围堵措施', hint: '围堵动作与拦截结果（片数/批次）' },
  { key: 'd4RootCause', code: 'D4', title: '根本原因分析', hint: '引用根因详情证据链编号' },
  { key: 'd5Corrective', code: 'D5', title: '永久纠正措施', hint: '消除根因的系统性对策' },
  { key: 'd6Verification', code: 'D6', title: '措施实施与效果验证', hint: '验证批次与良率回升数据' },
  { key: 'd7Prevention', code: 'D7', title: '预防再发', hint: 'FDC 规则/规范文件/预防性维护' },
  { key: 'd8Closure', code: 'D8', title: '团队祝贺与关闭', hint: '关闭确认与表彰' },
]

export default function ReportDetail() {
  const { id } = useParams()
  const reportId = Number(id)
  const navigate = useNavigate()
  const { message } = App.useApp()
  const queryClient = useQueryClient()
  const can = usePermission()

  /* ---------- 详情数据 ---------- */
  const { data, isPending, isError } = useQuery({
    queryKey: ['report', reportId],
    queryFn: () => getReportDetail(reportId),
    enabled: Number.isFinite(reportId),
  })

  /* ---------- 权限 × 状态双闸门 ---------- */
  const canEdit = !!data && data.status === 'draft' && can('report:edit')
  const canRelease = !!data && data.status !== 'released' && can('quality:release')

  /* ---------- 编辑模式 ---------- */
  const [editing, setEditing] = useState(false)
  const [form] = Form.useForm<ReportUpdateParams>()

  const updateMutation = useMutation({
    mutationFn: (values: ReportUpdateParams) => updateReport(reportId, values),
    onSuccess: () => {
      message.success('草稿已保存')
      setEditing(false)
      // 列表与详情同时过期：状态可能由服务端流转
      queryClient.invalidateQueries({ queryKey: ['report', reportId] })
      queryClient.invalidateQueries({ queryKey: ['reports'] })
    },
  })

  const releaseMutation = useMutation({
    mutationFn: () => releaseReport(reportId),
    onSuccess: () => {
      message.success('报告已签发，此后不可编辑')
      queryClient.invalidateQueries({ queryKey: ['report', reportId] })
      queryClient.invalidateQueries({ queryKey: ['reports'] })
    },
  })

  /** 进入编辑：把 D1~D8 现值灌进表单 */
  const startEdit = () => {
    if (!data) return
    form.setFieldsValue({
      d1Team: data.d1Team,
      d2Problem: data.d2Problem,
      d3Containment: data.d3Containment,
      d4RootCause: data.d4RootCause,
      d5Corrective: data.d5Corrective,
      d6Verification: data.d6Verification,
      d7Prevention: data.d7Prevention,
      d8Closure: data.d8Closure,
    })
    setEditing(true)
  }

  const saveEdit = () => {
    form.validateFields().then((values) => updateMutation.mutate(values))
  }

  /* ---------- 锚点导航项（D1 + D2~D8） ---------- */
  const anchorItems = useMemo(
    () => [
      { key: 'd1', href: '#d1', title: 'D1 成立小组' },
      ...TEXT_STEPS.map((s) => ({ key: s.code.toLowerCase(), href: `#${s.code.toLowerCase()}`, title: `${s.code} ${s.title}` })),
    ],
    [],
  )

  if (isPending) {
    return (
      <div className={styles.center}>
        <Spin size="large" tip="正在加载 8D 报告..." />
      </div>
    )
  }
  if (isError || !data) {
    return (
      <Result
        status="error"
        title="8D 报告加载失败"
        extra={<Button onClick={() => navigate('/reports')}>返回列表</Button>}
      />
    )
  }

  const e = data.event
  return (
    <div className={styles.container}>
      {/* ===== 操作栏（打印时隐藏） ===== */}
      <Card size="small" className={`no-print ${styles.headerCard}`}>
        <Space size={12}>
          <Button
            type="text"
            icon={<ArrowLeftOutlined />}
            onClick={() => navigate('/reports')}
          />
          <Typography.Text strong style={{ fontSize: 15 }}>{data.reportNo}</Typography.Text>
          <Tag color={REPORT_STATUS_MAP[data.status].color}>
            {REPORT_STATUS_MAP[data.status].label}
          </Tag>
          <Typography.Text type="secondary">{data.title}</Typography.Text>
        </Space>
        <Space className={styles.actions}>
          {canEdit && !editing && (
            <Button icon={<EditOutlined />} onClick={startEdit}>编辑</Button>
          )}
          {editing && (
            <>
              <Button icon={<SaveOutlined />} type="primary" loading={updateMutation.isPending} onClick={saveEdit}>
                保存
              </Button>
              <Button onClick={() => setEditing(false)}>取消</Button>
            </>
          )}
          {canRelease && (
            <Popconfirm
              title="确认签发这份报告？"
              description="签发后报告将锁定，不可再编辑。"
              okText="确认签发"
              cancelText="再想想"
              okButtonProps={{ danger: true }}
              onConfirm={() => releaseMutation.mutate()}
            >
              <Button type="primary" icon={<CheckCircleOutlined />} loading={releaseMutation.isPending}>
                签发
              </Button>
            </Popconfirm>
          )}
          <Button icon={<PrinterOutlined />} onClick={() => window.print()}>
            打印 / PDF
          </Button>
        </Space>
      </Card>

      <div className={styles.bodyRow}>
        {/* ===== 左侧锚点导航（打印时隐藏） ===== */}
        <div className={`no-print ${styles.anchorCol}`}>
          <Anchor items={anchorItems} offset={72} getContainer={scrollContainer} />
        </div>

        {/* ===== 右侧内容 ===== */}
        <div className={styles.contentCol}>
          {/* 事件概要 */}
          <Card size="small" title="关联事件概要" className={`print-block ${styles.stepCard}`} styles={{ body: { paddingTop: 8 } }}>
            <Descriptions size="small" column={3}>
              <Descriptions.Item label="事件" span={3}>
                <a onClick={() => navigate(`/anomaly/${e.id}`)} className="no-print">
                  {e.title}
                </a>
                <span className="print-only">{e.title}</span>
              </Descriptions.Item>
              <Descriptions.Item label="设备">{e.equipmentCode}</Descriptions.Item>
              <Descriptions.Item label="批次">{e.lotNo}</Descriptions.Item>
              <Descriptions.Item label="工序">{e.processStep}</Descriptions.Item>
              <Descriptions.Item label="发生时间" span={3}>{e.occurredAt}</Descriptions.Item>
            </Descriptions>
          </Card>

          {editing ? (
            /* ===== 编辑态：一张 Form 覆盖 D1~D8 ===== */
            <Form form={form} layout="vertical" className={styles.editForm}>
              <Card size="small" title="D1 成立小组" className={styles.stepCard} id="d1">
                <Form.List name="d1Team">
                  {(fields, { add, remove }) => (
                    <>
                      {fields.map((field) => (
                        <Space key={field.key} align="baseline" style={{ display: 'flex', marginBottom: 8 }}>
                          <Form.Item
                            name={[field.name, 'name']}
                            rules={[{ required: true, message: '姓名必填' }]}
                            style={{ marginBottom: 0 }}
                          >
                            <Input placeholder="姓名" style={{ width: 120 }} />
                          </Form.Item>
                          <Form.Item
                            name={[field.name, 'role']}
                            rules={[{ required: true, message: '角色必填' }]}
                            style={{ marginBottom: 0 }}
                          >
                            <Input placeholder="组内角色，如 组长" style={{ width: 140 }} />
                          </Form.Item>
                          <Form.Item
                            name={[field.name, 'dept']}
                            rules={[{ required: true, message: '部门必填' }]}
                            style={{ marginBottom: 0 }}
                          >
                            <Input placeholder="部门，如 工艺部" style={{ width: 160 }} />
                          </Form.Item>
                          <MinusCircleOutlined onClick={() => remove(field.name)} />
                        </Space>
                      ))}
                      <Button type="dashed" icon={<PlusOutlined />} onClick={() => add({ name: '', role: '', dept: '' })}>
                        添加成员
                      </Button>
                    </>
                  )}
                </Form.List>
              </Card>

              {TEXT_STEPS.map((step) => (
                <Card size="small" title={`${step.code} ${step.title}`} className={styles.stepCard} key={step.key}>
                  <Form.Item
                    name={step.key}
                    rules={[{ required: true, message: `${step.title}不能为空（8D 完整性要求）` }]}
                  >
                    <Input.TextArea rows={3} placeholder={step.hint} showCount maxLength={1000} />
                  </Form.Item>
                </Card>
              ))}
            </Form>
          ) : (
            /* ===== 查看态：D1 团队表格 + D2~D8 文本卡片 ===== */
            <>
              <Card size="small" title="D1 成立小组" className={`print-block ${styles.stepCard}`} id="d1">
                <Table<D1TeamRow>
                  rowKey="name"
                  size="small"
                  pagination={false}
                  dataSource={data.d1Team.map((m) => ({ ...m, key: m.name }))}
                  columns={[
                    { title: '姓名', dataIndex: 'name', width: 120 },
                    { title: '组内角色', dataIndex: 'role' },
                    { title: '部门', dataIndex: 'dept' },
                  ]}
                />
              </Card>
              {TEXT_STEPS.map((step) => (
                <Card
                  size="small"
                  title={`${step.code} ${step.title}`}
                  className={`print-block ${styles.stepCard}`}
                  key={step.key}
                >
                  <Typography.Paragraph style={{ marginBottom: 0, whiteSpace: 'pre-wrap' }}>
                    {data[step.key]}
                  </Typography.Paragraph>
                </Card>
              ))}
            </>
          )}

          <Typography.Text type="secondary" className={styles.footerNote}>
            本报告由报告 Agent 生成，全部结论引用证据链与审批记录；签发人：{data.releasedAt ? `${data.owner}（${data.releasedAt}）` : '—'}
          </Typography.Text>
        </div>
      </div>
    </div>
  )
}

/** 查看态 D1 表格行类型（加 key 给 rowKey 用） */
interface D1TeamRow { key: string; name: string; role: string; dept: string }

/** Anchor 滚动容器：MainLayout 的内容区（.ant-layout-content），不是 window */
function scrollContainer(): HTMLElement | null {
  return document.querySelector('.ant-layout-content')
}
```

```css
/* src/pages/Reports/index.module.css（追加详情页样式） */
.container { padding: 4px 0; }
.center { height: 50vh; display: grid; place-items: center; }

.headerCard {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.headerCard :global(.ant-card-body) {
  width: 100%;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.actions { flex: none; }

.bodyRow {
  display: flex;
  gap: 16px;
  align-items: flex-start;
}

.anchorCol {
  width: 150px;
  flex: none;
  position: sticky;
  top: 72px;
}

.contentCol {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.stepCard { scroll-margin-top: 72px; }
.editForm { display: flex; flex-direction: column; gap: 16px; }
.footerNote { font-size: 12px; padding: 4px 8px; }
```

```css
/* src/pages/Reports/print.css —— 打印专用全局样式（普通 CSS，非 module）
   引入即全局生效，但所有规则都包在 @media print 里，屏幕显示零影响 */
@media print {
  /* 1) 隐藏应用外壳：侧边菜单、顶栏、面包屑、操作栏（.no-print 类为字符串字面量类名，不走 module） */
  .ant-layout-sider,
  .ant-layout-header,
  .ant-breadcrumb,
  .no-print {
    display: none !important;
  }

  /* 2) 内容区铺满纸张、放开高度限制 */
  .ant-layout,
  .ant-layout-content {
    height: auto !important;
    overflow: visible !important;
    padding: 0 !important;
    margin: 0 !important;
  }

  /* 3) 报告分区：避免卡片被拦腰截断，去掉阴影改细边框 */
  .print-block {
    break-inside: avoid;
    box-shadow: none !important;
  }

  /* 4) 屏幕态隐藏的纯文本（打印时事件标题显示为文本而非链接） */
  .print-only { display: inline !important; }
  .print-only ~ .no-print { display: none !important; }

  /* 5) A4 纵向、页边距 */
  @page {
    size: A4 portrait;
    margin: 12mm;
  }
}
```

```css
/* src/styles/global.css 追加：屏幕态隐藏 .print-only（打印时由 print.css 反转显示） */
.print-only { display: none; }
```

四个实现细节讲透：

1. **`getContainer` 是 Anchor 的必踩坑**：AntD Anchor 默认监听 `window` 滚动，而本项目页面滚动发生在 `MainLayout` 的内容区（`.ant-layout-content`，见 6.2 的 `.content` 样式 `overflow: auto`）——不指定容器时点击锚点无反应、滚动高亮失效；
2. **`no-print` / `print-only` 是裸类名**：故意不进 CSS Modules——module 类名会 hash，而 `print.css` 是普通全局 CSS 匹配不到 hash 后的名字。这两个类承担"屏幕/打印互斥显示"（如事件标题：屏幕是链接、打印是纯文本）；
3. **`break-inside: avoid`**：让每个 D 区块尽量完整地落在同一页，避免"卡片上半页在第一页底、下半页在第二页顶"的撕裂感；
4. **编辑态与查看态是两棵不同的 JSX 树**：比"每行内嵌 Input 切换 disabled"更简单直接——8D 表单结构简单（表格 + 七个 TextArea），全量切换的重渲染成本可以忽略，换来的是两种状态各自干净。

#### 19.1.4 路由注册与菜单

在 `src/router/index.tsx` 增加懒加载声明与两条路由（**插在 `/approval/history` 路由项之后**，保持"列表 → 详情"就近排列的既有习惯）：

```tsx
// src/router/index.tsx 顶部懒加载声明区追加
const Reports = lazy(() => import('@/pages/Reports'))
const ReportDetail = lazy(() => import('@/pages/Reports/ReportDetail'))
```

```tsx
// appRoutes 数组中 /approval/history 项之后追加：
  {
    path: '/reports',
    meta: { title: '8D 报告', icon: 'report', permission: 'report:view' },
    element: Reports,
  },
  {
    path: '/reports/:id',
    meta: { title: '报告详情', hideInMenu: true },
    element: ReportDetail,
  },
```

菜单自动出现的原因（复习第 7 章）：菜单 = `appRoutes` 投影（剔除 `hideInMenu`、按 `permission` 过滤），新路由加入后**无需改任何菜单代码**——`admin`（全通过）与 `engineer`/`quality`（有 `report:view`）登录即看到"8D 报告"菜单项。

侧边图标映射，在 `MainLayout.tsx` 的 `ICON_MAP` 追加（原型对应 📄）：

```tsx
// src/layouts/MainLayout.tsx 的 ICON_MAP 追加
import { FileDoneOutlined } from '@ant-design/icons'
// ICON_MAP 对象内追加：
report: <FileDoneOutlined />,
```

面包屑挂靠（详情页不在菜单里，但要显示"首页 / 8D 报告 / 报告详情"），在 `src/router/menus.ts` 的 `findMenuByPath` 增加：

```ts
// src/router/menus.ts findMenuByPath 中，在 /anomaly/ 分支后追加：
else if (path.startsWith('/reports/')) {
  crumbs.push({ title: '8D 报告', path: '/reports' })
  crumbs.push({ title: '报告详情', path })
}
```

#### 19.1.5 Mock 数据与接口

`src/mock/data.ts` 追加报告数据（内容对齐原型 R8D-20261015-01）：

```ts
// src/mock/data.ts 追加
import type { Report8D, ReportListItem } from '@/types/report'

export const mockReports: ReportListItem[] = [
  {
    id: 3001,
    reportNo: '8D-2026-0012',
    title: 'ETA-301 腔体温度漂移导致良率下降',
    eventId: 1001,
    equipmentCode: 'ETA-301',
    lotNo: 'LOT-A2026',
    owner: 'agent:report',
    status: 'pending_review',
    createdAt: '2026-10-15 11:20:00',
  },
  {
    id: 3002,
    reportNo: '8D-2026-0011',
    title: 'CVD-105 压力异常尖峰批次处置',
    eventId: 1002,
    equipmentCode: 'CVD-105',
    lotNo: 'LOT-B1188',
    owner: 'agent:report',
    status: 'draft',
    createdAt: '2026-10-15 10:40:00',
  },
  {
    id: 3003,
    reportNo: '8D-2026-0010',
    title: 'FURN-02 炉管温度均匀性越限',
    eventId: 1003,
    equipmentCode: 'FURN-02',
    lotNo: 'LOT-D3301',
    owner: '赵敏',
    status: 'released',
    createdAt: '2026-10-13 15:10:00',
    releasedAt: '2026-10-14 09:00:00',
  },
  {
    id: 3004,
    reportNo: '8D-2026-0009',
    title: 'CMP-01 研磨速率异常处置',
    eventId: 1002,
    equipmentCode: 'CMP-01',
    lotNo: 'LOT-E0977',
    owner: '赵敏',
    status: 'released',
    createdAt: '2026-10-11 14:30:00',
    releasedAt: '2026-10-12 10:00:00',
  },
]

/** 详情内存态：PUT 保存后写回（刷新还原属 Mock 预期，见 11.2 验证 4） */
export const mockReportStore: Record<number, Report8D> = {}

export function mockReportDetail(id: number): Report8D {
  if (mockReportStore[id]) return mockReportStore[id]
  const item = mockReports.find((r) => r.id === id) ?? mockReports[0]
  const detail: Report8D = {
    id: item.id,
    reportNo: item.reportNo,
    title: item.title,
    status: item.status,
    owner: item.owner,
    createdAt: item.createdAt,
    releasedAt: item.releasedAt,
    event: {
      id: item.eventId,
      title: item.title,
      equipmentCode: item.equipmentCode,
      lotNo: item.lotNo,
      processStep: item.equipmentCode.startsWith('ETA') ? '刻蚀' : '薄膜',
      occurredAt: '2026-10-15 09:12:00',
    },
    d1Team: [
      { name: '李吉', role: '组长', dept: '工艺部' },
      { name: '王强', role: '设备', dept: '设备部' },
      { name: '赵敏', role: '质量', dept: '质量部' },
      { name: '刘洋', role: '生产', dept: '生产部' },
    ],
    d2Problem: 'LOT-A2026 批次良率 91.3%，低于基线 2.1%；缺陷集中于过刻蚀图形（占比 63%）。',
    d3Containment: '10-15 10:30 起 ETA-301 暂停投片；在制品 342 片全数 WAT 复测，拦截缺陷片 29 片。',
    d4RootCause: 'TC-02 温度传感器老化漂移 +1.8℃（嫌疑 #1，置信度 87%；因果效应 -1.8%/℃，p<0.01，证据链 E-01~E-05）。',
    d5Corrective: '更换 TC-02 传感器；设定值补偿至 84.2℃；增加同型号传感器 90 天预防性更换计划。',
    d6Verification: '10-16 起连续 3 批良率回升至 93.4% / 93.6% / 93.5%（验证中）。',
    d7Prevention: 'FDC 新增 TC 传感器一致性规则（双传感器差值 >0.5℃ 告警）；纳入《刻蚀设备点检规范》v3.2。',
    d8Closure: '待关闭后由质量部提报团队表彰。',
  }
  mockReportStore[id] = detail
  return detail
}
```

`src/mock/plugin.ts` 的 `route()` 中，在审批分支之后、404 之前追加：

```ts
  // ---- 8D 报告 ----
  if (pathname === '/api/v1/reports' && M === 'GET') {
    const q = Object.fromEntries(query) as unknown as { page?: number; pageSize?: number; status?: string }
    let list = [...mockReports]
    if (q.status) list = list.filter((r) => r.status === q.status)
    const page = q.page ?? 1
    const pageSize = q.pageSize ?? 20
    return send(res, { list: list.slice((page - 1) * pageSize, page * pageSize), total: list.length, page, pageSize })
  }
  if (/\/api\/v1\/reports\/\d+\/release$/.test(pathname) && M === 'POST') {
    const id = Number(pathname.split('/')[4])
    const detail = mockReportDetail(id)
    detail.status = 'released'
    detail.releasedAt = new Date().toLocaleString('sv-PR')
    const item = mockReports.find((r) => r.id === id)
    if (item) { item.status = 'released'; item.releasedAt = detail.releasedAt }
    return send(res, { ok: true })
  }
  if (/\/api\/v1\/reports\/\d+$/.test(pathname) && M === 'GET') {
    return send(res, mockReportDetail(Number(pathname.split('/').pop())))
  }
  if (/\/api\/v1\/reports\/\d+$/.test(pathname) && M === 'PUT') {
    const id = Number(pathname.split('/').pop())
    const detail = mockReportDetail(id)
    // 已签发的报告拒绝编辑（后端同样要校验，前端权限只是体验层）
    if (detail.status === 'released') return send(res, null, 40002)
    Object.assign(detail, body)
    return send(res, { ok: true })
  }
```

> 注意 `/release` 分支要放在 `/\d+$/` 详情分支**之前**——Mock 路由是顺序匹配，`/reports/3001/release` 同样满足 `\d+$` 结尾的正则（`release` 会被 `split('/').pop()` 拿去 `Number()` 变 NaN）。真实后端路由没有这个问题（精确路由表），但顺序敏感的匹配逻辑值得记住。

**【验证】** `pnpm dev` 后：

1. 用 `quality` 登录 → 侧边出现"8D 报告"菜单（文件图标）；用 `engineer` 登录同样可见（有 `report:view`）；点进列表页，四个状态 Tab + 4 条 Mock 报告，状态 Tag：草稿灰 / 待复核橙 / 已签发绿，"编制人"列 `agent:report` 显示紫色智能体 Tag；
2. 切"草稿"Tab → 只剩 1 条（8D-2026-0011）且页码回第 1 页；
3. 点报告编号 `8D-2026-0012` 进入详情：左侧锚点导航 D1~D8，点击任一项右侧平滑滚动到对应分区；头部"关联事件概要"的事件标题是链接，点击可跳 `/anomaly/1001`（根因详情）；
4. **编辑链路**（quality 账号）：打开草稿报告 8D-2026-0011 → 有"编辑"按钮（已签发的 8D-2026-0009 没有编辑按钮——状态闸门生效）；点编辑 → D1 变成员编辑行（可增删），D2~D8 变 TextArea；清空 D2 内容点保存 → 校验红字"问题描述不能为空"；填写后保存 → 成功提示，切回查看态且内容已更新（Mock 内存态生效）；
5. **权限闸门**：`engineer` 打开草稿报告 → 没有编辑/签发按钮（无 `report:edit`/`quality:release`）；
6. **签发链路**（quality）：对 8D-2026-0012 点"签发"→ Popconfirm 弹出"签发后报告将锁定"；确认 → 成功提示，状态变"已签发"，编辑与签发按钮同时消失；返回列表该报告状态已同步；
7. **打印**：点"打印 / PDF"→ 浏览器打印预览中侧边栏/顶栏/操作栏/锚点导航全部消失，只剩报告正文按 A4 分页，每个 D 区块完整不截断；取消打印回到屏幕，页面无任何变化（`@media print` 只影响打印媒介）。

---

### 19.2 审计日志页：查询 + CSV 导出 + 虚拟滚动

**业务背景**：审批理由写审计、报告签发写审计、登录写审计（原型页脚注明"审计日志保留 ≥ 1 年 · 不可篡改 append-only"）。审计页是管理员回溯"谁在什么时候对什么对象做了什么"的唯一入口，三个硬需求：**多条件检索、导出 CSV（合规上报）、大数据量不卡（保留 1 年意味着几千到几万条）**。

#### 19.2.1 类型定义与 api 模块

```ts
// src/types/audit.ts
import type { PageQuery } from './api'

/** 操作类型（与后端审计埋点一一对应） */
export type AuditAction =
  | 'login' | 'logout' | 'query' | 'create' | 'update' | 'delete'
  | 'approve' | 'reject' | 'release' | 'export'

export interface AuditLogItem {
  id: number
  /** 操作时间，如 2026-10-15 17:12:03 */
  operatedAt: string
  /** 操作人；智能体为 agent:rootcause / agent:report 等 */
  operator: string
  /** 操作人角色（智能体显示"智能体"） */
  role: string
  action: AuditAction
  /** 操作对象描述，如 SUG-20261015-011 或 GET /api/v1/anomalies */
  objectType: string
  objectDesc: string
  /** 来源 IP */
  ip: string
  /** 结果：成功/失败（失败含登录失败等安全事件） */
  result: 'success' | 'failure'
}

export interface AuditQueryParams extends PageQuery {
  /** 操作人模糊匹配 */
  operator?: string
  action?: AuditAction
  /** YYYY-MM-DD，与 endDate 成对 */
  startDate?: string
  endDate?: string
}
```

```ts
// src/api/audit.ts
import { http } from './request'
import type { PageResult } from '@/types/api'
import type { AuditLogItem, AuditQueryParams } from '@/types/audit'

/** pageSize 传 9999 时后端返回全量（导出用） */
export const getAuditLogs = (params: AuditQueryParams) =>
  http.get<PageResult<AuditLogItem>>('/audits', params)
```

操作类型映射常量（追加到 `src/utils/constants.ts`）：

```ts
// src/utils/constants.ts 追加（文件顶部 import 补 AuditAction）
import type { AuditAction } from '@/types/audit'

/** 审计操作类型 -> 标签配置 */
export const AUDIT_ACTION_MAP: Record<AuditAction, { label: string; color: string }> = {
  login: { label: '登录', color: 'blue' },
  logout: { label: '登出', color: 'default' },
  query: { label: '查询', color: 'default' },
  create: { label: '创建', color: 'green' },
  update: { label: '更新', color: 'orange' },
  delete: { label: '删除', color: 'red' },
  approve: { label: '审批通过', color: 'green' },
  reject: { label: '审批驳回', color: 'red' },
  release: { label: '报告签发', color: 'purple' },
  export: { label: '导出', color: 'cyan' },
}
```

#### 19.2.2 CSV 导出工具（前端 Blob 下载）

IA-SD-003 的工程结构里规划了 `utils/download.ts`（Blob 下载/预签名跳转），现在把它建起来——第 19.5 节的预签名图片也会用到这个文件：

```ts
// src/utils/download.ts

/**
 * 前端 CSV 导出（纯 Blob 方案，零后端依赖）。
 * @param filename 下载文件名（自动加 .csv 后缀）
 * @param rows 二维数组，rows[0] 为表头
 *
 * 两个必踩坑的提前规避：
 * 1) Excel 打开 UTF-8 CSV 中文乱码 -> 前置 BOM（\uFEFF）
 * 2) 单元格含逗号/引号/换行 -> 全字段双引号包裹，内部引号翻倍（RFC 4180）
 */
export function exportCsv(filename: string, rows: (string | number)[][]) {
  const escapeCell = (cell: string | number) =>
    `"${String(cell).replace(/"/g, '""')}"`
  const csv = rows.map((row) => row.map(escapeCell).join(',')).join('\r\n')
  const blob = new Blob([`\uFEFF${csv}`], { type: 'text/csv;charset=utf-8' })

  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = filename.endsWith('.csv') ? filename : `${filename}.csv`
  a.click()
  // 释放 objectURL：不释放的话同页面反复导出会累积内存占用
  URL.revokeObjectURL(url)
}
```

为什么不做成后端接口（`GET /audits/export` 返回文件流）？数据量小（万级以内）时前端拼 CSV 更快也更省服务端带宽；当导出量大到几十万行、或需要服务端复杂加工时再切后端方案（`responseType: 'blob'` + a 标签下载，第 18.3 进阶练习 3 已埋伏笔）。**团队里选哪种都对，重要的是统一**。

#### 19.2.3 审计日志页面

```tsx
// src/pages/Audits/index.tsx
import { useCallback, useMemo, useState } from 'react'
import {
  App, Button, Card, DatePicker, Empty, Form, Input, Select, Table, Tag, Tooltip,
} from 'antd'
import { DownloadOutlined, ReloadOutlined, SearchOutlined } from '@ant-design/icons'
import dayjs, { type Dayjs } from 'dayjs'
import { keepPreviousData, useQuery } from '@tanstack/react-query'
import { getAuditLogs } from '@/api/audit'
import type { AuditAction, AuditLogItem, AuditQueryParams } from '@/types/audit'
import { AUDIT_ACTION_MAP } from '@/utils/constants'
import { exportCsv } from '@/utils/download'
import styles from './index.module.css'

const { RangePicker } = DatePicker

interface QueryFormValues {
  operator?: string
  action?: AuditAction
  range?: [Dayjs | null, Dayjs | null] | null
}

/** 虚拟滚动开启阈值：单页行数超过它时启用（复用 16.3 的 Table virtual 方案） */
const VIRTUAL_THRESHOLD = 500

export default function Audits() {
  const { message } = App.useApp()

  /* ---------- 查询参数：受控分页（同 9.4 范式） ---------- */
  const [form] = Form.useForm<QueryFormValues>()
  const [query, setQuery] = useState<AuditQueryParams>({ page: 1, pageSize: 20 })
  const queryString = JSON.stringify(query)

  const { data, isFetching, refetch } = useQuery({
    queryKey: ['audits', 'logs', queryString],
    queryFn: () => getAuditLogs(query),
    placeholderData: keepPreviousData,
  })

  /* ---------- 查询/重置 ---------- */
  const onSearch = useCallback(() => {
    const v = form.getFieldsValue()
    setQuery((prev) => ({
      ...prev,
      page: 1,
      operator: v.operator || undefined,
      action: v.action,
      startDate: v.range?.[0]?.format('YYYY-MM-DD'),
      endDate: v.range?.[1]?.format('YYYY-MM-DD'),
    }))
  }, [form])

  const onReset = useCallback(() => {
    form.resetFields()
    setQuery({ page: 1, pageSize: 20 })
  }, [form])

  /* ---------- CSV 导出：按当前筛选条件拉全量 ---------- */
  const [exporting, setExporting] = useState(false)
  const onExport = useCallback(async () => {
    setExporting(true)
    try {
      const { operator, action, startDate, endDate } = query
      const res = await getAuditLogs({ page: 1, pageSize: 9999, operator, action, startDate, endDate })
      const rows: (string | number)[][] = [
        ['时间', '操作人', '角色', '操作', '对象', 'IP', '结果'],
        ...res.list.map((l) => [
          l.operatedAt, l.operator, l.role, AUDIT_ACTION_MAP[l.action].label,
          `${l.objectType} ${l.objectDesc}`, l.ip, l.result === 'success' ? '成功' : '失败',
        ]),
      ]
      exportCsv(`审计日志_${dayjs().format('YYYYMMDD_HHmmss')}`, rows)
      message.success(`已导出 ${res.list.length} 条审计记录`)
    } finally {
      setExporting(false)
    }
  }, [query, message])

  const columns = useMemo(
    () => [
      { title: '时间', dataIndex: 'operatedAt', width: 170 },
      {
        title: '操作人', dataIndex: 'operator', width: 140,
        render: (v: string, record: AuditLogItem) =>
          v.startsWith('agent:') ? (
            <Tooltip title={v}>
              <Tag color="purple">智能体</Tag>
            </Tooltip>
          ) : (
            <span>{v}<Typography2 role={record.role} /></span>
          ),
      },
      { title: '角色', dataIndex: 'role', width: 110 },
      {
        title: '操作', dataIndex: 'action', width: 110,
        render: (v: AuditAction) => {
          const conf = AUDIT_ACTION_MAP[v]
          return <Tag color={conf.color}>{conf.label}</Tag>
        },
      },
      {
        title: '对象', key: 'object', ellipsis: true,
        render: (_: unknown, record: AuditLogItem) => (
          <Tooltip title={`${record.objectType} ${record.objectDesc}`}>
            <span className={styles.mono}>{record.objectType}</span>
          </Tooltip>
        ),
      },
      { title: '来源 IP', dataIndex: 'ip', width: 130 },
      {
        title: '结果', dataIndex: 'result', width: 90,
        render: (v: AuditLogItem['result']) => (
          <Tag color={v === 'success' ? 'success' : 'error'}>{v === 'success' ? '成功' : '失败'}</Tag>
        ),
      },
    ],
    [],
  )

  /* 超大数据量：单页 >= 500 行时开虚拟滚动 + 关分页（16.3 方案的落地） */
  const virtualOn = query.pageSize >= VIRTUAL_THRESHOLD

  return (
    <Card size="small">
      <Form form={form} layout="inline" style={{ marginBottom: 16 }}>
        <Form.Item name="operator">
          <Input placeholder="操作人 / IP" allowClear style={{ width: 160 }} />
        </Form.Item>
        <Form.Item name="action">
          <Select
            placeholder="操作类型"
            allowClear
            style={{ width: 140 }}
            options={Object.entries(AUDIT_ACTION_MAP).map(([value, c]) => ({ label: c.label, value }))}
          />
        </Form.Item>
        <Form.Item name="range">
          <RangePicker style={{ width: 240 }} />
        </Form.Item>
        <Form.Item>
          <Button.Group>
            <Button type="primary" icon={<SearchOutlined />} onClick={onSearch}>查询</Button>
            <Button icon={<ReloadOutlined />} onClick={onReset}>重置</Button>
          </Button.Group>
        </Form.Item>
        <Form.Item>
          <Button icon={<DownloadOutlined />} loading={exporting} onClick={onExport}>
            导出 CSV
          </Button>
        </Form.Item>
        <Form.Item>
          <Tooltip title="审计日志保留 ≥ 1 年 · append-only 不可篡改">
            <Tag color="default" className={styles.auditNote}>保留 ≥1 年</Tag>
          </Tooltip>
        </Form.Item>
      </Form>

      <Table<AuditLogItem>
        rowKey="id"
        columns={columns}
        dataSource={data?.list}
        loading={isFetching}
        scroll={{ x: 1200, y: virtualOn ? 560 : undefined }}
        virtual={virtualOn}
        locale={{ emptyText: <Empty description="暂无符合条件的审计记录" /> }}
        pagination={
          virtualOn
            ? false
            : {
                current: query.page,
                pageSize: query.pageSize,
                total: data?.total ?? 0,
                showSizeChanger: true,
                pageSizeOptions: [20, 50, 100, 500, 1000],
                showTotal: (t) => `共 ${t} 条`,
                onChange: (page, pageSize) => setQuery((prev) => ({ ...prev, page, pageSize })),
              }
        }
      />
    </Card>
  )
}
```

上面"操作人"列里有个 `Typography2` 占位是故意留的坑位提示——实际不需要它，直接展示文本即可，请把该 render 简化为：

```tsx
render: (v: string) =>
  v.startsWith('agent:') ? (
    <Tooltip title={v}>
      <Tag color="purple">智能体</Tag>
    </Tooltip>
  ) : (
    v
  ),
```

（教程不删这段是为了让你体会一次真实 Code Review 的过程：列渲染函数的第二个参数 `record` 用不到就别解构，ESLint 的 `noUnusedParameters` 会拦住你。）

```css
/* src/pages/Audits/index.module.css */
.mono {
  font-family: Consolas, Monaco, monospace;
}

.auditNote {
  cursor: help;
}
```

**虚拟滚动的启停设计**讲一下：`pageSizeOptions` 提供了 `500/1000` 两档——选中的那一刻 `virtualOn` 变 true，Table 切换为"虚拟滚动 + 关闭分页"模式（`scroll.y` 必须给值，见 16.3）；选回 `20/50/100` 恢复常规分页。这样"检索某个月全量记录人工翻看"与"常规分页浏览"在同一张表上按需切换，Mock 造了 620 条数据供你验证。

#### 19.2.4 路由注册与菜单

```tsx
// src/router/index.tsx 顶部追加
const Audits = lazy(() => import('@/pages/Audits'))
```

```tsx
// appRoutes 中 /reports/:id 项之后追加：
  {
    path: '/audits',
    meta: { title: '审计日志', icon: 'audit', permission: 'admin:manage' },
    element: Audits,
  },
```

```tsx
// src/layouts/MainLayout.tsx 的 ICON_MAP 追加（原型对应 🧾）
import { ReconciliationOutlined } from '@ant-design/icons'
audit: <ReconciliationOutlined />,
```

权限说明：`admin:manage` 只有 admin 角色能过（三套演示账号都没配这个点）——engineer/quality 登录后**菜单里看不到审计日志**，手输 URL `/audits` 会被 `AuthRoute` 拦成 403 页（12.4 路由级权限）。这符合 IA-SD-003"审计日志仅系统管理员"的设计。

#### 19.2.5 Mock 数据与接口

`src/mock/data.ts` 追加——用确定性伪随机造 620 条（伪随机种子固定，保证每次刷新数据一致，验证"虚拟滚动"时行数稳定）：

```ts
// src/mock/data.ts 追加
import type { AuditLogItem } from '@/types/audit'

/** 确定性伪随机（种子固定 -> 每次刷新生成同样的 620 条，便于复现验证） */
function seeded(seed: number) {
  let s = seed
  return () => {
    s = (s * 9301 + 49297) % 233280
    return s / 233280
  }
}

const AUDIT_SEEDS = {
  operators: ['李吉', '王强', '张伟', '赵敏', '刘洋', 'admin', 'agent:rootcause', 'agent:report'],
  roles: ['工艺工程师', '设备工程师', '工艺主管', '质量工程师', '生产主管', '管理员', '智能体', '智能体'],
  actions: ['login', 'query', 'approve', 'reject', 'update', 'release', 'create', 'export'] as const,
  objects: [
    ['SUG-20261015-011', '审批建议：温度校准'],
    ['GET /api/v1/anomalies?status=WAIT_APPROVAL', '查询待审批异常'],
    ['R8D-2026-0012', '签发 8D 报告'],
    ['OPA 策略 policy-v14.rego', '策略上线'],
    ['EVT-20261015-007', '停线确认'],
    ['Keycloak SSO', '单点登录'],
    ['AGT-20261015-0031', '根因分析任务'],
  ] as const,
  ips: ['10.20.8.31', '10.20.8.14', '127.0.0.1', '10.20.1.5', '10.20.7.2', '172.16.4.77'],
}

export const mockAuditLogs: AuditLogItem[] = Array.from({ length: 620 }, (_, i) => {
  const rnd = seeded(i + 7)
  const opIdx = Math.floor(rnd() * AUDIT_SEEDS.operators.length)
  const actIdx = Math.floor(rnd() * AUDIT_SEEDS.actions.length)
  const objIdx = Math.floor(rnd() * AUDIT_SEEDS.objects.length)
  const ipIdx = Math.floor(rnd() * AUDIT_SEEDS.ips.length)
  const day = 1 + Math.floor(rnd() * 15)
  const hour = Math.floor(rnd() * 24)
  const minute = Math.floor(rnd() * 60)
  const action = AUDIT_SEEDS.actions[actIdx]
  return {
    id: 90001 + i,
    operatedAt: `2026-10-${String(day).padStart(2, '0')} ${String(hour).padStart(2, '0')}:${String(minute).padStart(2, '0')}:${String(Math.floor(rnd() * 60)).padStart(2, '0')}`,
    operator: AUDIT_SEEDS.operators[opIdx],
    role: AUDIT_SEEDS.roles[opIdx],
    action,
    objectType: AUDIT_SEEDS.objects[objIdx][0],
    objectDesc: AUDIT_SEEDS.objects[objIdx][1],
    ip: AUDIT_SEEDS.ips[ipIdx],
    // 登录失败等安全事件约 4%：结果列必须能展示失败
    result: action === 'login' && rnd() < 0.04 ? 'failure' : 'success',
  }
})
```

`src/mock/plugin.ts` 的 `route()` 追加（在报告分支之后）：

```ts
  // ---- 审计日志 ----
  if (pathname === '/api/v1/audits' && M === 'GET') {
    const q = Object.fromEntries(query) as unknown as {
      page?: number; pageSize?: number; operator?: string; action?: string
      startDate?: string; endDate?: string
    }
    let list = [...mockAuditLogs]
    if (q.operator) list = list.filter((l) => l.operator.includes(q.operator!) || l.ip.includes(q.operator!))
    if (q.action) list = list.filter((l) => l.action === q.action)
    if (q.startDate) list = list.filter((l) => l.operatedAt.slice(0, 10) >= q.startDate!)
    if (q.endDate) list = list.filter((l) => l.operatedAt.slice(0, 10) <= q.endDate!)
    // 时间倒序（最新在前）
    list.sort((a, b) => b.operatedAt.localeCompare(a.operatedAt))
    const page = q.page ?? 1
    const pageSize = q.pageSize ?? 20
    return send(res, { list: list.slice((page - 1) * pageSize, page * pageSize), total: list.length, page, pageSize })
  }
```

**【验证】** `pnpm dev` 后：

1. `admin` 登录 → 菜单出现"审计日志"（账单图标）；`quality` / `engineer` 登录 → 菜单里没有这一项；`quality` 手输 `/audits` → 403 Result 页（路由级权限三层拦截验收）；
2. 进入页面：620 条记录时间倒序，操作类型 Tag 颜色正确（审批通过绿 / 驳回红 / 签发紫 / 查询灰），`agent:*` 操作人显示紫色"智能体"Tag，偶见红色"失败"（登录失败安全事件）；
3. 操作人选"赵敏" + 操作类型选"报告签发"→ 只剩赵敏的签发记录；日期范围选 10-01~10-05 → 记录数进一步缩减；重置恢复全量；
4. **虚拟滚动**：把每页条数切到 500 → 表格切换为整页虚拟滚动模式（分页器消失、出现纵向滚动区），DevTools Elements 里 `<tbody>` 只有几十个 `<tr>`——620 条数据渲染如丝般顺滑；切回 20 恢复分页；
5. **导出 CSV**：保持一个筛选条件（如操作人=赵敏）点"导出 CSV"→ 下载 `审计日志_20261015_xxxx.csv`，用 Excel 打开：中文表头无乱码（BOM 生效）、字段含逗号也被引号正确包裹、行数与提示一致（只导出当前筛选结果）。

---

### 19.3 系统管理页：用户管理 / 数据健康度 / 智能体指标

**业务背景**：原型"系统管理"页由三块组成——用户与角色（RBAC）、数据健康度（传感器在线率/Kafka Lag 等数据链路质量）、智能体评估指标（Top-3 准确率/MTTR 等，对应白皮书的量化指标）。本节合并为一个页面 `/admin` 的三个 Tab（IA-SD-003 拆成 `/admin/users`、`/admin/system` 两条路由，教程用 Tab 承载——三者共享"仅管理员"这一个权限属性，拆路由反而重复配置，19.6 的对照表会说明这处差异）。

#### 19.3.1 类型定义与 api 模块

```ts
// src/types/admin.ts
import type { RoleCode } from './auth'

export interface UserItem {
  id: number
  username: string
  realName: string
  role: RoleCode
  /** 启用状态：禁用后无法登录（不物理删除，保留审计关联） */
  enabled: boolean
  lastLoginAt?: string
  createdAt: string
}

/** 新建/编辑用户参数（编辑时不传 password） */
export interface UserUpsertParams {
  id?: number
  username: string
  realName: string
  role: RoleCode
  /** 仅新建时必填 */
  password?: string
}

/** 数据健康度单项 */
export interface HealthItem {
  key: string
  name: string
  /** 展示值，如 "98.7%" / "156" */
  value: string
  /** 0~100 健康分（进度条宽度） */
  score: number
  status: 'ok' | 'warn' | 'err'
  note: string
}

/** 智能体评估指标单项 */
export interface AgentMetricItem {
  key: string
  name: string
  value: string
  target: string
  achieved: boolean
  note: string
}
```

```ts
// src/api/admin.ts
import { http } from './request'
import type { UserItem, UserUpsertParams, HealthItem, AgentMetricItem } from '@/types/admin'

/* ---------- 用户管理 ---------- */
export const getUserList = () => http.get<UserItem[]>('/admin/users')

export const createUser = (data: UserUpsertParams) => http.post<{ id: number }>('/admin/users', data)

/** 编辑：角色改绑、姓名修改（username 不可改，是审计主键） */
export const updateUser = (id: number, data: UserUpsertParams) =>
  http.put<{ ok: boolean }>(`/admin/users/${id}`, data)

/** 启用/禁用 */
export const toggleUser = (id: number, enabled: boolean) =>
  http.put<{ ok: boolean }>(`/admin/users/${id}/status`, { enabled })

/** 重置密码：后端生成随机密码并返回（明文只出现这一次） */
export const resetPassword = (id: number) =>
  http.post<{ newPassword: string }>(`/admin/users/${id}/reset-password`)

/* ---------- 数据健康度 ---------- */
export const getHealthMetrics = () => http.get<HealthItem[]>('/admin/data-health')

/* ---------- 智能体指标 ---------- */
export const getAgentMetrics = () => http.get<AgentMetricItem[]>('/admin/agent-metrics')
```

健康度/指标接口的组织方式完全复用 `api/dashboard.ts` 的风格（无参 GET 返回定长数组），区别只是归属"管理员域"所以单独放 `admin.ts`——**按资源域分文件，而不是按页面分文件**，这样通知中心、系统管理共用用户数据时不会出现循环引用。

#### 19.3.2 用户管理：表格 + 新建/编辑 Drawer

先写 Drawer（页面私有组件，放页面目录内，同 ApprovalModal 的组织方式）：

```tsx
// src/pages/Admin/UserDrawer.tsx
import { useEffect } from 'react'
import { App, Alert, Drawer, Form, Input, Select, Typography } from 'antd'
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { createUser, updateUser } from '@/api/admin'
import { ROLE_LABEL } from '@/utils/constants'
import type { RoleCode } from '@/types/auth'
import type { UserItem } from '@/types/admin'

interface UserFormValues {
  username: string
  realName: string
  role: RoleCode
  password?: string
}

interface UserDrawerProps {
  open: boolean
  /** null = 新建；有值 = 编辑（回填且 username 锁定） */
  user: UserItem | null
  onClose: () => void
}

export default function UserDrawer({ open, user, onClose }: UserDrawerProps) {
  const { message } = App.useApp()
  const queryClient = useQueryClient()
  const [form] = Form.useForm<UserFormValues>()
  const isEdit = !!user

  const mutation = useMutation({
    mutationFn: (values: UserFormValues) =>
      isEdit && user ? updateUser(user.id, values) : createUser(values),
    onSuccess: () => {
      message.success(isEdit ? '用户已更新' : '用户已创建')
      queryClient.invalidateQueries({ queryKey: ['admin', 'users'] })
      onClose()
    },
  })

  // 每次打开：新建重置；编辑回填（密码不回填——后端也不该把密码给前端）
  useEffect(() => {
    if (open) {
      form.resetFields()
      if (user) {
        form.setFieldsValue({ username: user.username, realName: user.realName, role: user.role })
      }
    }
  }, [open, user, form])

  return (
    <Drawer
      title={isEdit ? `编辑用户：${user!.realName}` : '新建用户'}
      open={open}
      onClose={onClose}
      width={420}
      destroyOnClose
    >
      {isEdit && (
        <Alert
          type="info"
          showIcon
          style={{ marginBottom: 16 }}
          message="用户名不可修改"
          description="用户名是审计日志的关联主键，改名会导致历史记录失联。如需更换请禁用旧账号并新建。"
        />
      )}
      <Form form={form} layout="vertical" onFinish={(v) => mutation.mutate(v)}>
        <Form.Item
          name="username"
          label="用户名"
          rules={[
            { required: true, message: '请输入用户名' },
            { min: 3, max: 20, message: '3~20 个字符' },
            { pattern: /^[a-zA-Z0-9_]+$/, message: '仅字母/数字/下划线' },
          ]}
        >
          <Input placeholder="登录账号，如 engineer2" disabled={isEdit} />
        </Form.Item>
        <Form.Item
          name="realName"
          label="姓名"
          rules={[{ required: true, message: '请输入姓名' }]}
        >
          <Input placeholder="真实姓名，如 陈工" />
        </Form.Item>
        <Form.Item
          name="role"
          label="角色（决定菜单与按钮权限）"
          rules={[{ required: true, message: '请选择角色' }]}
        >
          <Select
            placeholder="选择角色"
            options={Object.entries(ROLE_LABEL).map(([value, label]) => ({ value, label }))}
          />
        </Form.Item>
        {!isEdit && (
          <Form.Item
            name="password"
            label="初始密码"
            rules={[
              { required: true, message: '请输入初始密码' },
              { min: 8, message: '密码至少 8 位（安全基线）' },
            ]}
          >
            <Input.Password placeholder="至少 8 位，用户首次登录后应修改" />
          </Form.Item>
        )}
        <Form.Item style={{ marginTop: 24 }}>
          <TYPOGRAPHY_FOOT />
        </Form.Item>
      </Form>
      <ButtonBar
        loading={mutation.isPending}
        onSubmit={() => form.submit()}
        onClose={onClose}
      />
    </Drawer>
  )
}

/** 提交按钮区（小封装避免 JSX 主体过长） */
function ButtonBar({ loading, onSubmit, onClose }: { loading: boolean; onSubmit: () => void; onClose: () => void }) {
  return (
    <div style={{ display: 'flex', gap: 8, justifyContent: 'flex-end' }}>
      <Typography.Button onClick={onClose}>取消</Typography.Button>
      <Typography.Button type="primary" loading={loading} onClick={onSubmit}>
        保存
      </Typography.Button>
    </div>
  )
}

function TYPOGRAPHY_FOOT() {
  return (
    <Typography.Text type="secondary" style={{ fontSize: 12 }}>
      角色变更立即生效；权限点明细见《前端详细设计说明书》9.2 权限码清单。
    </Typography.Text>
  )
}
```

停一下——上面代码里 `TYPOGRAPHY_FOOT` / `ButtonBar` 这两个"组件"是**反例示范**：把一段 JSX 抽成同文件小组件却没带来复用（各只用一次），徒增阅读跳转成本，而且 `Typography.Button` 根本不存在（AntD 的按钮是 `Button`）。正确写法是直接平铺在 Drawer 里。请把 `<Form.Item style={{ marginTop: 24 }}>` 之后到文件结尾替换为：

```tsx
        <Form.Item style={{ marginTop: 24 }}>
          <Typography.Text type="secondary" style={{ fontSize: 12 }}>
            角色变更立即生效；权限点明细见《前端详细设计说明书》9.2 权限码清单。
          </Typography.Text>
        </Form.Item>
      </Form>
      <div style={{ display: 'flex', gap: 8, justifyContent: 'flex-end' }}>
        <Button onClick={onClose}>取消</Button>
        <Button type="primary" loading={mutation.isPending} onClick={() => form.submit()}>
          保存
        </Button>
      </div>
    </Drawer>
  )
}
```

同时给 `UserDrawer.tsx` 的 antd import 补上 `Button`。记住这条纪律：**组件抽抽像的标准是"复用或内聚"，不是"代码行数"**——这正是 2.4 节"页面私有子组件放页面目录内"的另一面：放进来的是有独立职责的东西（表单弹窗、图表配置），而不是随手一切的碎片。

#### 19.3.3 系统管理主页面（三个 Tab）

```tsx
// src/pages/Admin/index.tsx
import { useMemo, useState } from 'react'
import {
  App, Button, Card, Popconfirm, Progress, Row, Col, Space, Switch, Table, Tabs, Tag, Typography,
} from 'antd'
import { KeyOutlined, PlusOutlined } from '@ant-design/icons'
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import {
  getUserList, toggleUser, resetPassword, getHealthMetrics, getAgentMetrics,
} from '@/api/admin'
import type { UserItem } from '@/types/admin'
import { ROLE_LABEL } from '@/utils/constants'
import UserDrawer from './UserDrawer'
import styles from './index.module.css'

export default function Admin() {
  return (
    <Card size="small">
      <Tabs
        defaultActiveKey="users"
        items={[
          { key: 'users', label: '用户管理', children: <UserTab /> },
          { key: 'health', label: '数据健康度', children: <HealthTab /> },
          { key: 'agent', label: '智能体指标', children: <AgentTab /> },
        ]}
      />
    </Card>
  )
}

/* ==================== Tab 1：用户管理 ==================== */

function UserTab() {
  const { message, modal } = App.useApp()
  const queryClient = useQueryClient()

  const { data: users, isFetching } = useQuery({
    queryKey: ['admin', 'users'],
    queryFn: getUserList,
  })

  const [drawerOpen, setDrawerOpen] = useState(false)
  const [editingUser, setEditingUser] = useState<UserItem | null>(null)

  const openCreate = () => { setEditingUser(null); setDrawerOpen(true) }
  const openEdit = (u: UserItem) => { setEditingUser(u); setDrawerOpen(true) }

  /* 启用/禁用：写操作走 mutation + 失效刷新（同审批范式） */
  const toggleMutation = useMutation({
    mutationFn: ({ id, enabled }: { id: number; enabled: boolean }) => toggleUser(id, enabled),
    onSuccess: (_r, v) => {
      message.success(v.enabled ? '已启用' : '已禁用')
      queryClient.invalidateQueries({ queryKey: ['admin', 'users'] })
    },
  })

  /* 重置密码：成功后弹 Modal 展示一次性明文密码 */
  const resetMutation = useMutation({
    mutationFn: (id: number) => resetPassword(id),
    onSuccess: ({ newPassword }) => {
      modal.info({
        title: '密码已重置',
        content: (
          <div>
            <Typography.Paragraph>新密码（仅本次展示，请立即转交用户）：</Typography.Paragraph>
            <Typography.Text copyable code>{newPassword}</Typography.Text>
          </div>
        ),
      })
    },
  })

  const columns = useMemo(
    () => [
      { title: '用户名', dataIndex: 'username', width: 120 },
      { title: '姓名', dataIndex: 'realName', width: 110 },
      {
        title: '角色', dataIndex: 'role', width: 120,
        render: (v: UserItem['role']) => (
          <Tag color={v === 'admin' ? 'gold' : 'blue'}>{ROLE_LABEL[v]}</Tag>
        ),
      },
      {
        title: '最近登录', dataIndex: 'lastLoginAt', width: 150,
        render: (v?: string) => v ?? <Typography.Text type="secondary">从未登录</Typography.Text>,
      },
      {
        title: '状态', key: 'enabled', width: 90,
        render: (_: unknown, record: UserItem) => (
          <Tag color={record.enabled ? 'success' : 'default'}>
            {record.enabled ? '启用' : '禁用'}
          </Tag>
        ),
      },
      {
        title: '操作',
        key: 'action',
        render: (_: unknown, record: UserItem) => (
          <Space size={4}>
            <a onClick={() => openEdit(record)}>编辑</a>
            <Popconfirm
              title={record.enabled ? '禁用该用户？' : '启用该用户？'}
              description={record.enabled ? '禁用后该用户无法登录，历史数据保留。' : undefined}
              okText="确认"
              cancelText="取消"
              onConfirm={() => toggleMutation.mutate({ id: record.id, enabled: !record.enabled })}
            >
              <a style={{ color: record.enabled ? '#cf1322' : undefined }}>
                {record.enabled ? '禁用' : '启用'}
              </a>
            </Popconfirm>
            <Popconfirm
              title="重置该用户密码？"
              description="原密码立即失效，新密码将生成后展示一次。"
              okText="确认重置"
              cancelText="取消"
              onConfirm={() => resetMutation.mutate(record.id)}
            >
              <a><KeyOutlined /> 重置密码</a>
            </Popconfirm>
          </Space>
        ),
      },
    ],
    // eslint-disable-next-line react-hooks/exhaustive-deps
    [toggleMutation, resetMutation],
  )

  return (
    <>
      <div className={styles.tableToolbar}>
        <Typography.Text type="secondary" style={{ fontSize: 12 }}>
          角色决定三层权限（菜单/路由/按钮）；禁用不删除，保留审计关联
        </Typography.Text>
        <Button type="primary" icon={<PlusOutlined />} onClick={openCreate}>
          新建用户
        </Button>
      </div>
      <Table<UserItem>
        rowKey="id"
        size="middle"
        columns={columns}
        dataSource={users}
        loading={isFetching}
        pagination={false}
      />
      <UserDrawer
        open={drawerOpen}
        user={editingUser}
        onClose={() => setDrawerOpen(false)}
      />
    </>
  )
}

/* ==================== Tab 2：数据健康度 ==================== */

const HEALTH_TAG: Record<string, { color: string; label: string }> = {
  ok: { color: 'success', label: '正常' },
  warn: { color: 'warning', label: '关注' },
  err: { color: 'error', label: '异常' },
}

function HealthTab() {
  const { data, isPending } = useQuery({
    queryKey: ['admin', 'data-health'],
    queryFn: getHealthMetrics,
    refetchInterval: 60_000, // 数据链路健康度 1 分钟轮询
  })

  return (
    <Row gutter={[16, 16]}>
      {(data ?? []).map((item) => (
        <Col span={8} key={item.key}>
          <Card size="small" loading={isPending} className={styles.healthCard}>
            <div className={styles.healthHead}>
              <Typography.Text strong>{item.name}</Typography.Text>
              <Tag color={HEALTH_TAG[item.status].color}>{HEALTH_TAG[item.status].label}</Tag>
            </div>
            <Typography.Text className={styles.healthValue}>{item.value}</Typography.Text>
            <Progress
              percent={item.score}
              size="small"
              strokeColor={item.status === 'ok' ? '#52c41a' : item.status === 'warn' ? '#faad14' : '#cf1322'}
              format={(p) => `${p}分`}
            />
            <Typography.Text type="secondary" style={{ fontSize: 12 }}>
              {item.note}
            </Typography.Text>
          </Card>
        </Col>
      ))}
    </Row>
  )
}

/* ==================== Tab 3：智能体指标 ==================== */

function AgentTab() {
  const { data, isPending } = useQuery({
    queryKey: ['admin', 'agent-metrics'],
    queryFn: getAgentMetrics,
  })

  return (
    <Row gutter={[16, 16]}>
      {(data ?? []).map((m) => (
        <Col span={8} key={m.key}>
          <Card size="small" loading={isPending}>
            <div className={styles.healthHead}>
              <Typography.Text strong>{m.name}</Typography.Text>
              <Tag color={m.achieved ? 'success' : 'warning'}>
                {m.achieved ? '达标' : '未达标'}
              </Tag>
            </div>
            <Typography.Text className={styles.healthValue}>{m.value}</Typography.Text>
            <Typography.Paragraph type="secondary" style={{ fontSize: 12, marginBottom: 0 }}>
              目标：{m.target} · {m.note}
            </Typography.Paragraph>
          </Card>
        </Col>
      ))}
    </Row>
  )
}
```

```css
/* src/pages/Admin/index.module.css */
.tableToolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.healthCard :global(.ant-card-body) {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.healthHead {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.healthValue {
  font-size: 26px;
  font-weight: 700;
  font-family: Consolas, Monaco, monospace;
}
```

三个 Tab 的数据获取互相独立（各自 useQuery），AntD Tabs 默认**懒渲染 + 保留卸载内容**——切走再切回不重新请求（React Query 缓存 30s，见 5.3 的 staleTime），这是"Tabs 承载多模块"比"多路由"省的一点：状态天然共存。

#### 19.3.4 路由注册与菜单

```tsx
// src/router/index.tsx 顶部追加
const Admin = lazy(() => import('@/pages/Admin'))
```

```tsx
// appRoutes 中 /audits 项之后追加：
  {
    path: '/admin',
    meta: { title: '系统管理', icon: 'admin', permission: 'admin:manage' },
    element: Admin,
  },
```

```tsx
// src/layouts/MainLayout.tsx 的 ICON_MAP 追加（原型对应 ⚙️）
import { SettingOutlined } from '@ant-design/icons'
admin: <SettingOutlined />,
```

#### 19.3.5 Mock 数据与接口

`src/mock/data.ts` 追加（用户清单对齐原型系统管理页）：

```ts
// src/mock/data.ts 追加
import type { UserItem, HealthItem, AgentMetricItem } from '@/types/admin'
import type { RoleCode } from '@/types/auth'

export const mockUsers: UserItem[] = [
  { id: 1, username: 'admin', realName: '系统管理员', role: 'admin', enabled: true, lastLoginAt: '2026-10-15 14:22:00', createdAt: '2026-01-01 00:00:00' },
  { id: 2, username: 'engineer', realName: '王工艺', role: 'process_engineer' as RoleCode, enabled: true, lastLoginAt: '2026-10-15 17:02:00', createdAt: '2026-01-01 00:00:00' },
  { id: 3, username: 'quality', realName: '赵敏', role: 'quality_engineer' as RoleCode, enabled: true, lastLoginAt: '2026-10-15 09:05:00', createdAt: '2026-01-01 00:00:00' },
  { id: 4, username: 'wangq', realName: '王强', role: 'equipment_engineer' as RoleCode, enabled: true, lastLoginAt: '2026-10-15 16:20:00', createdAt: '2026-02-14 10:00:00' },
  { id: 5, username: 'liuy', realName: '刘洋', role: 'process_engineer' as RoleCode, enabled: false, lastLoginAt: '2026-10-15 10:31:00', createdAt: '2026-03-02 09:00:00' },
]

export const mockHealthMetrics: HealthItem[] = [
  { key: 'sensor_online', name: '传感器在线率', value: '98.7%', score: 98, status: 'ok', note: '断线设备 0' },
  { key: 'latency_p95', name: '数据延迟 P95', value: '2.3s', score: 90, status: 'ok', note: '目标 < 5s' },
  { key: 'kafka_lag', name: 'Kafka 消费 Lag', value: '156', score: 88, status: 'ok', note: '< 1000 告警线' },
  { key: 'dirty_ratio', name: '脏数据占比(24h)', value: '1.9%', score: 62, status: 'warn', note: '注入样本导致，观察中' },
  { key: 'tsdb_compress', name: 'TimescaleDB 压缩率', value: '8.4x', score: 92, status: 'ok', note: '达标(>7x)' },
  { key: 'stream_broken', name: '断流设备数', value: '0', score: 100, status: 'ok', note: '实时链路完整' },
]

export const mockAgentMetrics: AgentMetricItem[] = [
  { key: 'top3_accuracy', name: '根因 Top-3 准确率', value: '82.0%', target: '> 80%', achieved: true, note: '近 30 天 62 次分析' },
  { key: 'mttr', name: 'MTTR(根因定位)', value: '3.6h', target: '< 4h', achieved: true, note: '较人工 8h 下降 55%' },
  { key: 'false_alarm', name: '误报率', value: '8.2%', target: '< 10%', achieved: true, note: '近 30 天' },
  { key: 'human_takeover', name: '人工接管率', value: '26%', target: '< 30%', achieved: true, note: 'HITL 占比' },
  { key: 'e2e_latency', name: '端到端延迟(均值)', value: '4m18s', target: '< 5min', achieved: true, note: '接近上限，关注' },
  { key: 'report_ratio', name: '8D 报告自动生成率', value: '91%', target: '> 85%', achieved: true, note: '9% 需人工补全 D6' },
]
```

`src/mock/plugin.ts` 的 `route()` 追加（用户数组长度小，直接在 plugin 里做内存 CRUD）：

```ts
  // ---- 系统管理 ----
  if (pathname === '/api/v1/admin/users' && M === 'GET') {
    return send(res, mockUsers)
  }
  if (pathname === '/api/v1/admin/users' && M === 'POST') {
    const { username, role } = body as { username: string; role: string }
    if (mockUsers.some((u) => u.username === username)) {
      return send(res, null, 40003) // 用户名已存在
    }
    mockUsers.push({
      id: Math.max(...mockUsers.map((u) => u.id)) + 1,
      username,
      realName: (body as { realName: string }).realName,
      role: role as never,
      enabled: true,
      createdAt: new Date().toLocaleString('sv-PR'),
    })
    return send(res, { id: mockUsers[mockUsers.length - 1].id })
  }
  if (/\/api\/v1\/admin\/users\/\d+\/status$/.test(pathname) && M === 'PUT') {
    const id = Number(pathname.split('/')[5])
    const user = mockUsers.find((u) => u.id === id)
    if (user) user.enabled = (body as { enabled: boolean }).enabled
    return send(res, { ok: true })
  }
  if (/\/api\/v1\/admin\/users\/\d+\/reset-password$/.test(pathname) && M === 'POST') {
    return send(res, {
      // Mock 直接生成可读密码；真实后端应走加密存储 + 首登强制修改
      newPassword: `Ia@${Math.random().toString(36).slice(2, 10)}`,
    })
  }
  if (/\/api\/v1\/admin\/users\/\d+$/.test(pathname) && M === 'PUT') {
    const id = Number(pathname.split('/').pop())
    const user = mockUsers.find((u) => u.id === id)
    if (user) Object.assign(user, { realName: (body as { realName: string }).realName, role: (body as { role: never }).role })
    return send(res, { ok: true })
  }
  if (pathname === '/api/v1/admin/data-health' && M === 'GET') {
    return send(res, mockHealthMetrics)
  }
  if (pathname === '/api/v1/admin/agent-metrics' && M === 'GET') {
    return send(res, mockAgentMetrics)
  }
```

> `/status` 与 `/reset-password` 两个子路径分支同样放在 `/\d+$/` 通用分支之前——与 19.1.5 的 `/release` 同一个顺序敏感问题。

**【验证】** `pnpm dev` 后用 `admin` 登录：

1. 菜单出现"系统管理"（齿轮图标），点进后三个 Tab；
2. **用户管理**：5 个用户，角色 Tag 中文显示（管理员金色）；点"新建用户"→ 右侧 Drawer 弹出；用户名输入 `ab`（<3 字符）→ 校验红字；密码输 6 位 → "至少 8 位"；填 `engineer2` / 陈工 / 工艺工程师 / 8 位密码保存 → 成功，表格出现新行且状态"启用"；
3. **编辑回填**：点王强的"编辑"→ Drawer 回填 username（灰色禁用）+ 姓名 + 角色；把角色改成"设备工程师"保存 → 表格角色 Tag 变化；再点"编辑"→ 回填的是新角色（改绑闭环）；
4. **禁用确认**：点刘洋（已是禁用态）旁的"启用"→ Popconfirm 确认后变"启用"绿 Tag；再禁用回来——两次都有 Popconfirm 二次确认；
5. **重置密码**：点任意用户"重置密码"→ 确认 → 弹出 Modal 展示形如 `Ia@x8f2kq1z` 的一次性密码，旁边有复制图标；
6. **数据健康度**：6 张卡片两行三列，每张卡片"名称 + 状态 Tag + 大数字 + 彩色进度条 + 说明"——脏数据占比卡片为橙色"关注"态（62 分），其余绿色；
7. **智能体指标**：6 张指标卡，全部绿色"达标" Tag，数值等宽字体显示（82.0% / 3.6h / 8.2%...）；1 分钟后 Network 自动重拉 health（轮询生效）；
8. `quality` 账号登录 → 无"系统管理"菜单，手输 `/admin` → 403。

---

### 19.4 通知中心：铃铛改造 + 下拉面板 + 通知列表页

**业务背景**：第 9 章的铃铛只是"告警未读数"（`realtimeStore.unreadCount`，点击清零），但原型的通知语义更丰富——异常告警、审批提醒、报告动态、系统通知四类，每条可跳回来源页面。本节把铃铛升级为真正的**通知中心**：

```text
顶栏铃铛（Badge 未读数）
 ├─ 数据源1：useQuery 30s 轮询 /notifications/unread-count（兜底）
 ├─ 数据源2：WS notification.push 事件 -> invalidateQueries（实时）
 └─ 点击展开 Popover：最近 5 条 + [全部已读] + [查看全部]
      └─ /notifications 列表页：分页 + 类型筛选 + 单条已读 + 点击跳转来源页
```

两种数据源的分工值得先想清楚：**轮询是兜底，WS 是实时**。WS 断线重连期间（指数退避最长 30s）轮询保证未读数最终正确；WS 在线时 `invalidateQueries` 触发立即重拉，延迟从"最多 30s"降到"毫秒级"。

#### 19.4.1 类型定义与 api 模块

```ts
// src/types/notification.ts
import type { PageQuery } from './api'

export type NotificationType = 'anomaly' | 'approval' | 'report' | 'system'

export interface NotificationItem {
  id: number
  type: NotificationType
  title: string
  content: string
  read: boolean
  /**
   * 来源业务 ID，点击通知跳转：
   * anomaly -> /anomalies/:sourceId；approval -> /approvals；report -> /reports/:sourceId
   */
  sourceId: number
  createdAt: string
}

export interface NotificationQueryParams extends PageQuery {
  type?: NotificationType
  read?: boolean
}
```

```ts
// src/api/notification.ts
import { http } from './request'
import type { PageResult } from '@/types/api'
import type { NotificationItem, NotificationQueryParams } from '@/types/notification'

/** 未读数（顶栏 Badge 轮询） */
export const getUnreadCount = () =>
  http.get<{ count: number }>('/notifications/unread-count')

/** 最近 N 条（铃铛下拉面板） */
export const getRecentNotifications = (limit = 5) =>
  http.get<NotificationItem[]>('/notifications/recent', { limit })

/** 分页列表（通知列表页） */
export const getNotifications = (params: NotificationQueryParams) =>
  http.get<PageResult<NotificationItem>>('/notifications', params)

/** 单条已读 */
export const markRead = (id: number) =>
  http.post<{ ok: boolean }>(`/notifications/${id}/read`)

/** 全部已读 */
export const markAllRead = () =>
  http.post<{ ok: boolean }>('/notifications/read-all')
```

类型映射常量（追加到 `src/utils/constants.ts`）：

```ts
// src/utils/constants.ts 追加（文件顶部 import 补 NotificationType）
import type { NotificationType } from '@/types/notification'

/** 通知类型 -> 标签配置 */
export const NOTIFICATION_TYPE_MAP: Record<NotificationType, { label: string; color: string }> = {
  anomaly: { label: '异常告警', color: 'red' },
  approval: { label: '审批提醒', color: 'orange' },
  report: { label: '报告动态', color: 'purple' },
  system: { label: '系统通知', color: 'blue' },
}
```

#### 19.4.2 通知铃铛组件：Badge + Popover + 最近 5 条

通知铃铛是布局级通用组件（MainLayout 与通知页都要用跳转映射），放 `components/`：

```tsx
// src/components/NotificationBell.tsx
import { useNavigate } from 'react-router-dom'
import { App, Badge, Button, Empty, List, Popover, Tag, Typography } from 'antd'
import { BellOutlined, CheckOutlined } from '@ant-design/icons'
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import {
  getUnreadCount, getRecentNotifications, markAllRead,
} from '@/api/notification'
import { NOTIFICATION_TYPE_MAP } from '@/utils/constants'
import type { NotificationItem } from '@/types/notification'
import styles from './NotificationBell.module.css'

/** 通知 -> 跳转路径映射（列表页复用，见 19.4.4） */
export function notificationPath(n: Pick<NotificationItem, 'type' | 'sourceId'>): string | null {
  switch (n.type) {
    case 'anomaly':
      return `/anomaly/${n.sourceId}`
    case 'approval':
      return '/approval'
    case 'report':
      return `/reports/${n.sourceId}`
    default:
      return null // system 类通知不跳转
  }
}

export default function NotificationBell() {
  const navigate = useNavigate()
  const { message } = App.useApp()
  const queryClient = useQueryClient()

  /* 未读数：30s 轮询兜底（WS 推送时经 invalidateQueries 立即刷新，见 19.4.3） */
  const unreadQuery = useQuery({
    queryKey: ['notifications', 'unread-count'],
    queryFn: getUnreadCount,
    refetchInterval: 30_000,
  })

  /* 最近 5 条：Popover 打开时才需要，但数据很轻，直接与未读数同缓存域 */
  const recentQuery = useQuery({
    queryKey: ['notifications', 'recent'],
    queryFn: () => getRecentNotifications(5),
  })

  const readAllMutation = useMutation({
    mutationFn: markAllRead,
    onSuccess: () => {
      message.success('已全部标记为已读')
      queryClient.invalidateQueries({ queryKey: ['notifications'] })
    },
  })

  const unreadCount = unreadQuery.data?.count ?? 0
  const items = recentQuery.data ?? []

  const content = (
    <div className={styles.panel}>
      <div className={styles.panelHeader}>
        <Typography.Text strong>通知中心</Typography.Text>
        <Button
          type="link"
          size="small"
          icon={<CheckOutlined />}
          disabled={unreadCount === 0}
          loading={readAllMutation.isPending}
          onClick={() => readAllMutation.mutate()}
        >
          全部已读
        </Button>
      </div>
      {items.length === 0 ? (
        <Empty image={Empty.PRESENTED_IMAGE_SIMPLE} description="暂无通知" />
      ) : (
        <List
          size="small"
          dataSource={items}
          className={styles.list}
          renderItem={(item) => (
            <List.Item
              className={styles.item}
              onClick={() => {
                const path = notificationPath(item)
                if (path) navigate(path)
              }}
            >
              <List.Item.Meta
                avatar={
                  <Badge dot={!item.read} offset={[-2, 2]}>
                    <Tag color={NOTIFICATION_TYPE_MAP[item.type].color} style={{ margin: 0 }}>
                      {NOTIFICATION_TYPE_MAP[item.type].label}
                    </Tag>
                  </Badge>
                }
                title={<Typography.Text strong={!item.read} style={{ fontSize: 13 }}>{item.title}</Typography.Text>}
                description={
                  <>
                    <div className={styles.line2}>{item.content}</div>
                    <Typography.Text type="secondary" style={{ fontSize: 11 }}>
                      {item.createdAt}
                    </Typography.Text>
                  </>
                }
              />
            </List.Item>
          )}
        />
      )}
      <Button
        type="link"
        block
        onClick={() => navigate('/notifications')}
      >
        查看全部
      </Button>
    </div>
  )

  return (
    <Popover
      content={content}
      trigger="click"
      placement="bottomRight"
      arrow={false}
      styles={{ body: { padding: 0 } }}
    >
      <Badge count={unreadCount} size="small" offset={[-4, 4]}>
        <BellOutlined style={{ color: '#fff', fontSize: 18, cursor: 'pointer' }} />
      </Badge>
    </Popover>
  )
}
```

```css
/* src/components/NotificationBell.module.css */
.panel {
  width: 340px;
}

.panelHeader {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 12px 6px;
}

.list {
  max-height: 360px;
  overflow-y: auto;
}

.item {
  cursor: pointer;
  padding: 8px 12px;
}

.item:hover {
  background: rgba(22, 104, 220, 0.04);
}

.line2 {
  font-size: 12px;
  color: #666;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
```

#### 19.4.3 MainLayout 接线：替换铃铛 + WS 推送打通

对 `MainLayout.tsx` 做三处增量修改：

```tsx
// 增量 1：文件顶部 import 区
import { useQueryClient } from '@tanstack/react-query'
import NotificationBell from '@/components/NotificationBell'
```

```tsx
// 增量 2：MainLayout 函数体，第 9 章接入的 useWebSocket 回调扩展一个分支。
// （同时删除第 9 章的 clearUnread 订阅——未读数已由通知 API 接管）
const queryClient = useQueryClient()

useWebSocket({
  enabled: !!token,
  onMessage: (msg: WsMessage<unknown>) => {
    if (msg.type === 'ANOMALY_CREATED') {
      pushEvent(msg.payload as AnomalyEvent)
    }
    // 通知推送：任何页面上都能让铃铛与列表立即刷新（推送失效模式）
    if (msg.type === 'NOTIFICATION_PUSH') {
      queryClient.invalidateQueries({ queryKey: ['notifications'] })
    }
  },
})
```

```tsx
// 增量 3：顶栏第 9 章的 Tooltip/Badge/BellOutlined 整段替换为：
<NotificationBell />
```

**与第 9 章旧铃铛的关系**要理清：告警未读（`realtimeStore.unreadCount`）由 WS 事件 `ANOMALY_CREATED` 自增、点击清零，是**纯前端计数**；新通知中心把"未读"变成**服务端状态**（`read` 字段持久化，跨设备一致）。两者不冲突但会重复提醒（异常事件既进告警通知又进通知列表）——真实项目二选一即可，本教程保留两者是为了让你同时看到两种模式的实现。

#### 19.4.4 通知列表页 /notifications

```tsx
// src/pages/Notifications/index.tsx
import { useCallback, useMemo, useState } from 'react'
import { useNavigate } from 'react-router-dom'
import { App, Badge, Card, Empty, Form, Select, Table, Tag, Typography } from 'antd'
import { keepPreviousData, useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { getNotifications, markRead, markAllRead } from '@/api/notification'
import type { NotificationItem, NotificationQueryParams, NotificationType } from '@/types/notification'
import { NOTIFICATION_TYPE_MAP } from '@/utils/constants'
import { notificationPath } from '@/components/NotificationBell'
import styles from './index.module.css'

export default function Notifications() {
  const navigate = useNavigate()
  const { message } = App.useApp()
  const queryClient = useQueryClient()

  /* ---------- 查询参数（同 9.4 受控分页范式） ---------- */
  const [form] = Form.useForm<{ type?: NotificationType; read?: string }>()
  const [query, setQuery] = useState<NotificationQueryParams>({ page: 1, pageSize: 20 })
  const queryString = JSON.stringify(query)

  const { data, isFetching } = useQuery({
    queryKey: ['notifications', 'list', queryString],
    queryFn: () => getNotifications(query),
    placeholderData: keepPreviousData,
  })

  const onSearch = useCallback(() => {
    const v = form.getFieldsValue()
    setQuery((prev) => ({
      ...prev,
      page: 1,
      type: v.type,
      read: v.read === undefined || v.read === 'all' ? undefined : v.read === 'unread',
    }))
  }, [form])

  /* ---------- 已读操作 ---------- */
  const readMutation = useMutation({
    mutationFn: (id: number) => markRead(id),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['notifications'] }),
  })

  const readAllMutation = useMutation({
    mutationFn: markAllRead,
    onSuccess: () => {
      message.success('已全部标记为已读')
      queryClient.invalidateQueries({ queryKey: ['notifications'] })
    },
  })

  /** 单条点击：标记已读 + 跳转来源页 */
  const onRowClick = (record: NotificationItem) => {
    if (!record.read) readMutation.mutate(record.id)
    const path = notificationPath(record)
    if (path) navigate(path)
  }

  const columns = useMemo(
    () => [
      {
        title: '类型', dataIndex: 'type', width: 110,
        render: (v: NotificationType) => {
          const conf = NOTIFICATION_TYPE_MAP[v]
          return <Tag color={conf.color}>{conf.label}</Tag>
        },
      },
      {
        title: '标题', dataIndex: 'title', ellipsis: true,
        render: (v: string, record: NotificationItem) => (
          <Typography.Text strong={!record.read}>{v}</Typography.Text>
        ),
      },
      { title: '内容', dataIndex: 'content', ellipsis: true },
      {
        title: '状态', dataIndex: 'read', width: 90,
        render: (v: boolean) =>
          v ? (
            <Tag>已读</Tag>
          ) : (
            <Badge dot color="#cf1322" text="未读" />
          ),
      },
      { title: '时间', dataIndex: 'createdAt', width: 170 },
      {
        title: '操作', key: 'action', width: 130,
        render: (_: unknown, record: NotificationItem) => {
          const path = notificationPath(record)
          return (
            <App_space>
              {path && <a onClick={() => onRowClick(record)}>查看来源</a>}
              {!record.read && (
                <a onClick={() => readMutation.mutate(record.id)}>标为已读</a>
              )}
            </App_space>
          )
        },
      },
    ],
    // eslint-disable-next-line react-hooks/exhaustive-deps
    [readMutation],
  )

  return (
    <Card size="small" title="通知中心" className={styles.container}
      extra={
        <Typography.Button onClick={() => readAllMutation.mutate()}>
          全部已读
        </Typography.Button>
      }
    >
      <Form form={form} layout="inline" style={{ marginBottom: 16 }}>
        <Form.Item name="type">
          <Select
            placeholder="通知类型"
            allowClear
            style={{ width: 140 }}
            options={Object.entries(NOTIFICATION_TYPE_MAP).map(([value, c]) => ({ label: c.label, value }))}
          />
        </Form.Item>
        <Form.Item name="read">
          <Select
            placeholder="已读状态"
            allowClear
            style={{ width: 120 }}
            options={[
              { label: '未读', value: 'unread' },
              { label: '已读', value: 'read' },
            ]}
          />
        </Form.Item>
        <Form.Item>
          <Typography.Button type="primary" onClick={onSearch}>
            筛选
          </Typography.Button>
        </Form.Item>
      </Form>

      <Table<NotificationItem>
        rowKey="id"
        columns={columns}
        dataSource={data?.list}
        loading={isFetching}
        onRow={(record) => ({ onClick: () => onRowClick(record), className: styles.row })}
        locale={{ emptyText: <Empty description="暂无通知" /> }}
        pagination={{
          current: query.page,
          pageSize: query.pageSize,
          total: data?.total ?? 0,
          showSizeChanger: true,
          showTotal: (t) => `共 ${t} 条`,
          onChange: (page, pageSize) => setQuery((prev) => ({ ...prev, page, pageSize })),
        }}
      />
    </Card>
  )
}
```

注意上面出现了两个故意的错误占位（`App_space`、`Typography.Button`）——**AntD 里没有 `Typography.Button`**，Space 组件也不会以这种形式存在。正确写法是在文件顶部从 antd 导入 `Button` 与 `Space`，两处分别替换为：

```tsx
// 操作列内：
<Space>
  {path && <a onClick={() => onRowClick(record)}>查看来源</a>}
  {!record.read && (
    <a onClick={() => readMutation.mutate(record.id)}>标为已读</a>
  )}
</Space>
```

```tsx
// extra 与筛选按钮处：
extra={
  <Button loading={readAllMutation.isPending} onClick={() => readAllMutation.mutate()}>
    全部已读
  </Button>
}
```

```tsx
<Form.Item>
  <Button type="primary" onClick={onSearch}>筛选</Button>
</Form.Item>
```

（为什么教程里故意保留错误再修正？因为 `Typography.Button` 这类"看起来应该存在"的 API 是初学者最高频的复制粘贴事故来源，比正确答案更值得单独强调一次。）

```css
/* src/pages/Notifications/index.module.css */
.container { padding: 4px 0; }

.row {
  cursor: pointer;
}
```

#### 19.4.5 路由注册、面包屑与 Mock 推送

路由——通知列表页**不进侧边菜单**（入口是铃铛，与原型一致：原型侧边栏没有通知项，铃铛是唯一入口）：

```tsx
// src/router/index.tsx 顶部追加
const Notifications = lazy(() => import('@/pages/Notifications'))
```

```tsx
// appRoutes 末尾（/admin 之后）追加：
  {
    path: '/notifications',
    meta: { title: '通知中心', hideInMenu: true },
    element: Notifications,
  },
```

面包屑挂靠（同 19.1.4 的做法），`findMenuByPath` 追加：

```ts
else if (path === '/notifications') {
  crumbs.push({ title: '通知中心', path })
}
```

Mock 数据（`src/mock/data.ts` 追加）：

```ts
// src/mock/data.ts 追加
import type { NotificationItem } from '@/types/notification'

export const mockNotifications: NotificationItem[] = [
  {
    id: 7001, type: 'anomaly', title: '新异常事件：CVD-105',
    content: '薄膜沉积压力尖峰，FDC 分数 76，已触发根因分析',
    read: false, sourceId: 1002, createdAt: '2026-10-15 10:03:00',
  },
  {
    id: 7002, type: 'approval', title: '新的调参建议待审批',
    content: 'ETA-301 腔温设定值下调建议等待人工审批（风险等级：中）',
    read: false, sourceId: 5001, createdAt: '2026-10-15 09:30:00',
  },
  {
    id: 7003, type: 'report', title: '8D 报告已生成',
    content: '报告 Agent 生成 8D-2026-0012，等待质量工程师复核签发',
    read: false, sourceId: 3001, createdAt: '2026-10-15 11:20:00',
  },
  {
    id: 7004, type: 'system', title: '系统维护通知',
    content: '本周六 02:00-04:00 平台升级，期间看板数据暂停刷新',
    read: true, sourceId: 0, createdAt: '2026-10-14 18:00:00',
  },
  {
    id: 7005, type: 'anomaly', title: '异常事件已闭环',
    content: 'FURN-02 炉管温度均匀性越限事件已闭环，8D 报告已签发',
    read: true, sourceId: 1003, createdAt: '2026-10-13 15:10:00',
  },
  {
    id: 7006, type: 'approval', title: '审批已通过',
    content: '您提交的审批（任务 5001）已由张伟加签通过',
    read: true, sourceId: 5001, createdAt: '2026-10-15 10:12:00',
  },
]
```

`src/mock/plugin.ts` 的 `route()` 追加（在系统管理分支之后）：

```ts
  // ---- 通知中心 ----
  if (pathname === '/api/v1/notifications/unread-count' && M === 'GET') {
    return send(res, { count: mockNotifications.filter((n) => !n.read).length })
  }
  if (pathname === '/api/v1/notifications/recent' && M === 'GET') {
    const limit = Number(query.get('limit') ?? 5)
    return send(res, mockNotifications.slice(0, limit))
  }
  if (pathname === '/api/v1/notifications/read-all' && M === 'POST') {
    mockNotifications.forEach((n) => { n.read = true })
    return send(res, { ok: true })
  }
  if (pathname === '/api/v1/notifications' && M === 'GET') {
    const q = Object.fromEntries(query) as unknown as { page?: number; pageSize?: number; type?: string }
    const readParam = query.get('read')
    let list = [...mockNotifications]
    if (q.type) list = list.filter((n) => n.type === q.type)
    if (readParam === 'true') list = list.filter((n) => n.read)
    if (readParam === 'false') list = list.filter((n) => !n.read)
    const page = q.page ?? 1
    const pageSize = q.pageSize ?? 20
    return send(res, { list: list.slice((page - 1) * pageSize, page * pageSize), total: list.length, page, pageSize })
  }
  if (/\/api\/v1\/notifications\/\d+\/read$/.test(pathname) && M === 'POST') {
    const id = Number(pathname.split('/')[4])
    const n = mockNotifications.find((x) => x.id === id)
    if (n) n.read = true
    return send(res, { ok: true })
  }
```

WS 推送的 Mock 自测手段（同第 9 章 `window.__pushMockEvent` 的做法），在 `App.tsx` 的 `injectMockPush` 里追加：

```tsx
// src/App.tsx 的 injectMockPush 函数内追加：
window.__pushMockNotification = () => {
  // 直接触发 invalidate，模拟服务端 notification.push 到达的完整链路
  queryClient.invalidateQueries({ queryKey: ['notifications'] })
}
```

对应 `declare global` 的 Window 接口补一行 `__pushMockNotification?: () => void`，并在 `App.tsx` 顶部补 `import { queryClient } from './main'`（`main.tsx` 已导出，见 5.3——这就是当初导出它的回报：非组件环境也能驱动缓存失效）。

**【验证】** `pnpm dev` 后（任意账号）：

1. 顶栏铃铛显示红色 Badge "3"（3 条未读）；点击铃铛 → Popover 下拉：最近 5 条通知，每条带类型 Tag（异常告警红/审批提醒橙/报告动态紫/系统通知蓝），未读条目左侧有红点、标题加粗；
2. 点下拉里的"全部已读"→ Badge 清零、所有条目红点消失、已读项不再加粗；Network 面板随即重拉 unread-count 与 recent（invalidation 链）；
3. 点下拉里的某条"新异常事件：CVD-105"→ 关闭 Popover 并跳转 `/anomaly/1002`（根因详情页）；返回顶栏，该条已变已读（Mock 内存态）；
4. 点"查看全部"→ 进入 `/notifications` 列表页：分页表格、类型/已读筛选生效（筛"未读"只剩未读条目）；点某未读行的"标为已读"→ 该行状态 Tag 变化且铃铛未读数 -1；点"查看来源"（approval 类型）→ 跳 `/approval`；system 类型没有"查看来源"链接；
5. Console 执行 `window.__pushMockNotification()` → 顶栏铃铛无变化（因为 Mock 未读数没变，invalidate 只是重拉——**这是符合预期的**：真实后端会在推送前写入新通知，重拉后未读数才会 +1）。想看"+1"效果可手动在 `src/mock/data.ts` 里把任一条 `read: false` 改动后刷新；
6. Network 面板等 30 秒 → `unread-count` 自动重拉（轮询兜底生效）；
7. 侧边菜单没有"通知中心"项（hideInMenu），面包屑显示"首页 / 通知中心"。

---

### 19.5 缺陷查询页：多条件检索 + 缺陷图预览 + 分布饼图

**业务背景**：看板的"缺陷分类占比"饼图回答"哪类缺陷多"，工程师的下一个问题永远是"**给我看这批的具体缺陷记录和图**"——WAT/CP 检出缺陷后，缺陷图（Wafer Map）存在对象存储（MinIO），前端通过**预签名 URL** 限时访问。本页把"多条件查询缺陷记录 + 看图 + 分布统计"补齐，与看板形成"总览 → 明细"的闭环。

#### 19.5.1 类型定义与 api 模块

```ts
// src/types/defect.ts
import type { PageQuery } from './api'
import type { Severity } from './anomaly'

/** 缺陷类型（与看板缺陷分类饼图的口径一致） */
export type DefectType =
  | '过刻蚀' | '显影不良' | '颗粒污染' | '注入缺陷' | '膜厚不均' | '其他'

export interface DefectRecord {
  id: number
  lotNo: string
  /** 晶圆号，如 W17 */
  waferNo: string
  equipmentCode: string
  processStep: string
  defectType: DefectType
  severity: Severity
  /** 该晶圆检出缺陷点数 */
  defectCount: number
  foundAt: string
  /** 缺陷图在对象存储中的 key（不是 URL——URL 由预签名接口换取） */
  imageKey: string
}

export interface DefectQueryParams extends PageQuery {
  lotNo?: string
  defectType?: DefectType
  severity?: Severity
  startDate?: string
  endDate?: string
}
```

```ts
// src/api/defect.ts
import { http } from './request'
import type { PageResult } from '@/types/api'
import type { DefectQueryParams, DefectRecord } from '@/types/defect'
import type { DefectCategoryItem } from '@/api/dashboard'

export const getDefects = (params: DefectQueryParams) =>
  http.get<PageResult<DefectRecord>>('/defects', params)

/** 缺陷分布（当前筛选条件下的类型聚合，复用看板的 DefectCategoryItem） */
export const getDefectDistribution = (params: Omit<DefectQueryParams, 'page' | 'pageSize'>) =>
  http.get<DefectCategoryItem[]>('/defects/distribution', params)

/**
 * 用对象存储 key 换预签名 URL（有效期有限，如 5 分钟）。
 * 不直接把 key 拼进 <img src>：对象存储不开放匿名读，预签名是"限时门票"。
 */
export const getImageUrl = (imageKey: string) =>
  http.get<{ url: string }>('/defects/image-url', { key: imageKey })
```

> `getDefectDistribution` 复用 `api/dashboard.ts` 已导出的 `DefectCategoryItem` 类型——跨页面共享口径，避免"看板叫颗粒污染、明细页叫颗粒"这类口径漂移。

缺陷类型常量（追加到 `src/utils/constants.ts`）：

```ts
// src/utils/constants.ts 追加
import type { DefectType } from '@/types/defect'

/** 缺陷类型清单（查询下拉用，与看板缺陷分类饼图同口径） */
export const DEFECT_TYPES: DefectType[] = [
  '过刻蚀', '显影不良', '颗粒污染', '注入缺陷', '膜厚不均', '其他',
]
```

#### 19.5.2 缺陷查询页面

```tsx
// src/pages/Defects/index.tsx
import { useCallback, useMemo, useState } from 'react'
import {
  App, Button, Card, Col, Empty, Form, Input, Modal, Row, Select, DatePicker, Spin, Table, Tag, Typography,
} from 'antd'
import { PictureOutlined, ReloadOutlined, SearchOutlined } from '@ant-design/icons'
import type { Dayjs } from 'dayjs'
import { keepPreviousData, useQuery } from '@tanstack/react-query'
import { getDefects, getDefectDistribution, getImageUrl } from '@/api/defect'
import type { DefectQueryParams, DefectRecord, DefectType } from '@/types/defect'
import type { Severity } from '@/types/anomaly'
import { SEVERITY_MAP, DEFECT_TYPES } from '@/utils/constants'
import { useChart } from '@/components/hooks/useChart'
import styles from './index.module.css'

const { RangePicker } = DatePicker

interface QueryFormValues {
  lotNo?: string
  defectType?: DefectType
  severity?: Severity
  range?: [Dayjs | null, Dayjs | null] | null
}

export default function Defects() {
  /* ---------- 查询参数（9.4 受控分页范式） ---------- */
  const [form] = Form.useForm<QueryFormValues>()
  const [query, setQuery] = useState<DefectQueryParams>({ page: 1, pageSize: 20 })
  const queryString = JSON.stringify(query)

  const { data, isFetching } = useQuery({
    queryKey: ['defects', 'list', queryString],
    queryFn: () => getDefects(query),
    placeholderData: keepPreviousData,
  })

  /* ---------- 分布饼图：与列表同筛选条件（排除分页参数） ---------- */
  const distParams = useMemo(() => {
    const { lotNo, defectType, severity, startDate, endDate } = query
    return { lotNo, defectType, severity, startDate, endDate }
  }, [query])

  const distQuery = useQuery({
    queryKey: ['defects', 'distribution', JSON.stringify(distParams)],
    queryFn: () => getDefectDistribution(distParams),
  })

  const pieOption = useBuildPieOption(distQuery.data)
  const pieRef = useChart(pieOption, { loading: distQuery.isPending })

  /* ---------- 缺陷图预览（Modal + 预签名 URL） ---------- */
  const [preview, setPreview] = useState<{ open: boolean; url: string | null; loading: boolean }>({
    open: false, url: null, loading: false,
  })

  const showImage = useCallback(async (imageKey: string) => {
    setPreview({ open: true, url: null, loading: true })
    const { url } = await getImageUrl(imageKey)
    setPreview({ open: true, url, loading: false })
  }, [])

  /* ---------- 查询/重置 ---------- */
  const onSearch = useCallback(() => {
    const v = form.getFieldsValue()
    setQuery((prev) => ({
      ...prev,
      page: 1,
      lotNo: v.lotNo || undefined,
      defectType: v.defectType,
      severity: v.severity,
      startDate: v.range?.[0]?.format('YYYY-MM-DD'),
      endDate: v.range?.[1]?.format('YYYY-MM-DD'),
    }))
  }, [form])

  const onReset = useCallback(() => {
    form.resetFields()
    setQuery({ page: 1, pageSize: 20 })
  }, [form])

  const columns = useMemo(
    () => [
      { title: 'ID', dataIndex: 'id', width: 80 },
      { title: '批次', dataIndex: 'lotNo', width: 130 },
      { title: '晶圆', dataIndex: 'waferNo', width: 80 },
      { title: '设备', dataIndex: 'equipmentCode', width: 110 },
      { title: '工序', dataIndex: 'processStep', width: 90 },
      {
        title: '缺陷类型', dataIndex: 'defectType', width: 110,
        render: (v: DefectType) => <Tag color="purple">{v}</Tag>,
      },
      {
        title: '严重程度', dataIndex: 'severity', width: 90,
        render: (v: Severity) => {
          const conf = SEVERITY_MAP[v]
          return <Tag color={conf.color}>{conf.label}</Tag>
        },
      },
      { title: '缺陷数', dataIndex: 'defectCount', width: 90, sorter: (a: DefectRecord, b: DefectRecord) => a.defectCount - b.defectCount },
      { title: '发现时间', dataIndex: 'foundAt', width: 170 },
      {
        title: '操作', key: 'action', width: 110,
        render: (_: unknown, record: DefectRecord) => (
          <Button type="link" size="small" icon={<PictureOutlined />} onClick={() => showImage(record.imageKey)}>
            缺陷图
          </Button>
        ),
      },
    ],
    [showImage],
  )

  return (
    <Row gutter={16} className={styles.container} align="stretch">
      {/* 左：查询 + 表格 */}
      <Col span={16}>
        <Card size="small">
          <Form form={form} layout="inline" style={{ marginBottom: 16 }}>
            <Form.Item name="lotNo">
              <Input placeholder="批次号，如 LOT-A2026" allowClear style={{ width: 180 }} />
            </Form.Item>
            <Form.Item name="defectType">
              <Select
                placeholder="缺陷类型"
                allowClear
                style={{ width: 130 }}
                options={DEFECT_TYPES.map((t) => ({ label: t, value: t }))}
              />
            </Form.Item>
            <Form.Item name="severity">
              <Select
                placeholder="严重程度"
                allowClear
                style={{ width: 110 }}
                options={Object.entries(SEVERITY_MAP).map(([value, c]) => ({ label: c.label, value }))}
              />
            </Form.Item>
            <Form.Item name="range">
              <RangePicker style={{ width: 240 }} />
            </Form.Item>
            <Form.Item>
              <Button.Group>
                <Button type="primary" icon={<SearchOutlined />} onClick={onSearch}>查询</Button>
                <Button icon={<ReloadOutlined />} onClick={onReset}>重置</Button>
              </Button.Group>
            </Form.Item>
          </Form>

          <Table<DefectRecord>
            rowKey="id"
            columns={columns}
            dataSource={data?.list}
            loading={isFetching}
            scroll={{ x: 1150 }}
            locale={{ emptyText: <Empty description="暂无缺陷记录" /> }}
            pagination={{
              current: query.page,
              pageSize: query.pageSize,
              total: data?.total ?? 0,
              showSizeChanger: true,
              showTotal: (t) => `共 ${t} 条`,
              onChange: (page, pageSize) => setQuery((prev) => ({ ...prev, page, pageSize })),
            }}
          />
        </Card>
      </Col>

      {/* 右：缺陷分布饼图（同筛选条件） */}
      <Col span={8}>
        <Card size="small" title="缺陷类型分布（当前筛选）" className={styles.chartCard}>
          <div ref={pieRef} className={styles.chart} />
          <Typography.Text type="secondary" style={{ fontSize: 12 }}>
            与看板"缺陷分类占比"同口径；分布随左侧筛选条件联动
          </Typography.Text>
        </Card>
      </Col>

      {/* 缺陷图预览 Modal：预签名 URL 换取后展示 */}
      <Modal
        open={preview.open}
        title="缺陷图（Wafer Map · 预签名访问）"
        footer={null}
        onCancel={() => setPreview((s) => ({ ...s, open: false }))}
        width={480}
      >
        {preview.loading ? (
          <div className={styles.previewLoading}>
            <Spin tip="正在获取预签名 URL..." />
          </div>
        ) : (
          <img src={preview.url ?? undefined} alt="缺陷图" className={styles.previewImg} />
        )}
      </Modal>
    </Row>
  )
}

/* ==================== 饼图配置（复用 8.4 的环形饼图范式） ==================== */

function useBuildPieOption(data?: { name: string; value: number }[]) {
  return useMemo(
    () => ({
      tooltip: { trigger: 'item', formatter: '{b}: {c} 片（{d}%）' },
      legend: { bottom: 0, type: 'scroll' },
      series: [
        {
          type: 'pie',
          radius: ['45%', '70%'],
          avoidLabelOverlap: true,
          itemStyle: { borderRadius: 4, borderColor: '#fff', borderWidth: 2 },
          label: { formatter: '{b}\n{d}%' },
          data: data ?? [],
        },
      ],
    }),
    [data],
  )
}
```

```css
/* src/pages/Defects/index.module.css */
.container { padding: 4px 0; }
.chartCard :global(.ant-card-body) { padding: 12px; }
.chart { height: 340px; width: 100%; }
.previewLoading { height: 320px; display: grid; place-items: center; }
.previewImg { width: 100%; border-radius: 8px; border: 1px solid #e5e7eb; }
```

**预签名 URL 的两个工程细节**：

1. **为什么 `imageKey` 不直接拼 URL**：对象存储不开匿名读。`getImageUrl` 每次向后端（后端再向 MinIO）换一张"限时门票"（如 5 分钟有效）。前端拿到就渲染、过期就重换——所以 `showImage` 每次打开 Modal 都重新请求，**不缓存 URL**（缓存的应该是 React Query 缓存的列表数据，URL 天然过期）；
2. **`defectParams` 从 `query` 中排除分页字段再 `JSON.stringify`**：分布图不该因为翻页而重新请求（第 10 页和第 1 页的分布一样），queryKey 里只放业务过滤条件——这是 queryKey 粒度设计的典型一课（呼应 FAQ Q7）。

#### 19.5.3 路由注册与菜单

```tsx
// src/router/index.tsx 顶部追加
const Defects = lazy(() => import('@/pages/Defects'))
```

```tsx
// appRoutes 中 /dashboard 项之后追加（紧邻看板：明细查询是看板的总览下钻）：
  {
    path: '/defects',
    meta: { title: '缺陷查询', icon: 'defect' },
    element: Defects,
  },
```

```tsx
// src/layouts/MainLayout.tsx 的 ICON_MAP 追加（缺陷数据的"虫子"语义）
import { BugOutlined } from '@ant-design/icons'
defect: <BugOutlined />,
```

不配 `permission`——缺陷查询对全部登录用户开放（与看板、异常列表同级，对齐 IA-SD-003 的"全部登录用户"口径），菜单自动出现在"良率看板"之后。

#### 19.5.4 Mock 数据与接口

`src/mock/data.ts` 追加——缺陷记录 + **内联 SVG 模拟 Wafer Map**（不用外链图片：教程必须离线可跑；SVG 的 data URI 也能完美演示"预签名 URL"的异步时序）：

```ts
// src/mock/data.ts 追加
import type { DefectRecord, DefectType } from '@/types/defect'

/** 生成一张模拟晶圆缺陷图（SVG data URI，稳定伪随机：同一 imageKey 每次生成相同图） */
export function mockWaferImage(imageKey: string): string {
  const seed = imageKey.split('').reduce((a, c) => a + c.charCodeAt(0), 0)
  const rnd = seeded(seed + 11)
  const dots = Array.from({ length: 24 }, () => {
    const angle = rnd() * Math.PI * 2
    const radius = Math.sqrt(rnd()) * 78 // sqrt 保证点在圆面上均匀分布
    const x = (100 + radius * Math.cos(angle)).toFixed(1)
    const y = (100 + radius * Math.sin(angle)).toFixed(1)
    return `<circle cx="${x}" cy="${y}" r="2.4" fill="#f0564f"/>`
  }).join('')
  const svg =
    `<svg xmlns="http://www.w3.org/2000/svg" width="400" height="400" viewBox="0 0 200 200">` +
    `<circle cx="100" cy="100" r="92" fill="#141d2b" stroke="#2c3e58" stroke-width="2"/>` +
    `<circle cx="100" cy="100" r="60" fill="none" stroke="#233247"/>` +
    `<circle cx="100" cy="100" r="30" fill="none" stroke="#233247"/>` +
    `<text x="100" y="196" text-anchor="middle" fill="#64778f" font-size="8">${imageKey}</text>` +
    dots + `</svg>`
  return `data:image/svg+xml;charset=utf-8,${encodeURIComponent(svg)}`
}

const DEFECT_ROWS: [string, string, string, string, DefectType, 'critical' | 'high' | 'medium', number][] = [
  ['LOT-A2026', 'W17', 'ETA-301', '刻蚀', '过刻蚀', 'critical', 63],
  ['LOT-A2026', 'W18', 'ETA-301', '刻蚀', '过刻蚀', 'high', 41],
  ['LOT-B1188', 'W03', 'CVD-105', '薄膜', '颗粒污染', 'high', 37],
  ['LOT-B1188', 'W11', 'CVD-105', '薄膜', '膜厚不均', 'medium', 18],
  ['LOT-C0042', 'W07', 'PHO-220', '光刻', '显影不良', 'high', 52],
  ['LOT-C0042', 'W12', 'PHO-220', '光刻', '对准偏差', 'medium', 12],
  ['LOT-D3301', 'W01', 'IMP-018', '离子注入', '注入缺陷', 'medium', 18],
  ['LOT-E0977', 'W22', 'CMP-410', 'CMP', '颗粒污染', 'low', 9],
]

export const mockDefects: DefectRecord[] = DEFECT_ROWS.map((row, i) => ({
  id: 8001 + i,
  lotNo: row[0],
  waferNo: row[1],
  equipmentCode: row[2],
  processStep: row[3],
  defectType: row[4] === '对准偏差' ? '其他' : row[4],
  severity: row[5],
  defectCount: row[6],
  foundAt: `2026-10-${String(8 + (i % 8)).padStart(2, '0')} ${String(9 + (i % 9)).padStart(2, '0')}:30:00`,
  imageKey: `wafermap/${row[0]}-${row[1]}.svg`,
}))
```

`src/mock/plugin.ts` 的 `route()` 追加：

```ts
  // ---- 缺陷查询 ----
  if (pathname === '/api/v1/defects' && M === 'GET') {
    const q = Object.fromEntries(query) as unknown as {
      page?: number; pageSize?: number; lotNo?: string; defectType?: string
      severity?: string; startDate?: string; endDate?: string
    }
    let list = [...mockDefects]
    if (q.lotNo) list = list.filter((d) => d.lotNo.includes(q.lotNo!))
    if (q.defectType) list = list.filter((d) => d.defectType === q.defectType)
    if (q.severity) list = list.filter((d) => d.severity === q.severity)
    if (q.startDate) list = list.filter((d) => d.foundAt.slice(0, 10) >= q.startDate!)
    if (q.endDate) list = list.filter((d) => d.foundAt.slice(0, 10) <= q.endDate!)
    const page = q.page ?? 1
    const pageSize = q.pageSize ?? 20
    return send(res, { list: list.slice((page - 1) * pageSize, page * pageSize), total: list.length, page, pageSize })
  }
  if (pathname === '/api/v1/defects/distribution' && M === 'GET') {
    // 与列表同条件聚合成 { name, value }[]
    const q = Object.fromEntries(query) as unknown as { lotNo?: string; defectType?: string; severity?: string }
    let list = [...mockDefects]
    if (q.lotNo) list = list.filter((d) => d.lotNo.includes(q.lotNo!))
    if (q.severity) list = list.filter((d) => d.severity === q.severity)
    const counter = new Map<string, number>()
    for (const d of list) counter.set(d.defectType, (counter.get(d.defectType) ?? 0) + d.defectCount)
    return send(res, [...counter.entries()].map(([name, value]) => ({ name, value })))
  }
  if (pathname === '/api/v1/defects/image-url' && M === 'GET') {
    // 预签名 URL：真实场景是带有效期的 http 地址；Mock 用内联 SVG data URI 等价模拟
    const key = query.get('key') ?? ''
    return send(res, { url: mockWaferImage(key) })
  }
```

**【验证】** `pnpm dev` 后（任意账号）：

1. 侧边菜单"良率看板"下方出现"缺陷查询"（虫子图标）；进入页面：左侧 8 条缺陷记录表格，右侧环形饼图显示当前筛选下的缺陷类型分布（过刻蚀 104 最大扇区）；
2. 批次输入 `LOT-C0042` 点查询 → 表格剩 2 条，右侧饼图同时变为"显影不良 52 / 其他 12"两块（**列表与饼图联动**）；重置恢复；
3. 严重程度筛"紧急" → 只剩 LOT-A2026 W17（critical 那条，红色"紧急"Tag）；
4. 点"缺陷数"列头 → 按缺陷数排序（客户端 sorter 生效）；
5. **缺陷图预览**：点任意行"缺陷图"→ Modal 弹出，先短暂显示"正在获取预签名 URL..."转圈（异步换取时序可见），随后渲染一张深色晶圆圆图，红色缺陷点分布其上，底部标注 imageKey；Network 面板可见 `/defects/image-url?key=...` 请求；关闭再开同一行 → 图完全一致（伪随机种子稳定）；
6. 拖动浏览器宽度 → 饼图自适应（useChart 的 ResizeObserver，第 8 章验收项在第五个页面再次成立）。

---

### 19.6 收尾整合：路由总表 / 接口对照 / 菜单最终形态

五个页面补完后，是时候把全书成果对齐到设计文档与原型，做一次"验收级"的收尾。

#### 19.6.1 完整路由表：对照 IA-SD-003 的 18 条路由全集

IA-SD-003 第 3 章定义了 17 条路由，加上按原型通知中心补充的 `/notifications`，设计全集为 18 条。下表是"设计 → 教程实现"的完整对照（教程在合并粒度上做了三处务实取舍，见备注列）：

| # | IA-SD-003 / 原型设计路由 | 教程实现路由 | 页面组件（懒加载） | meta 权限 | 备注 |
|---|---|---|---|---|---|
| 1 | `/login` | `/login` | `Login` | 公开 | — |
| 2 | `/`（重定向） | `/` → `/dashboard` | — | 登录 | — |
| 3 | `/dashboard` | `/dashboard` | `Dashboard` | 登录即可 | — |
| 4 | `/anomalies` | `/anomaly` | `AnomalyList` | 登录即可 | 单复数命名从简 |
| 5 | `/anomalies/:id` | `/anomaly/:id` | `RootCauseDetail` | 登录即可 | — |
| 6 | `/rootcause`（Top-3 列表） | 合并进 `/anomaly` 列表与详情 | — | — | 教程将根因列表并入异常列表（每行即入口），减少一层导航 |
| 7 | `/rootcause/:taskId` | `/anomaly/:id`（同一条） | `RootCauseDetail` | 登录即可 | 根因详情即异常详情页（Top-3 + 证据链同屏） |
| 8 | `/suggestions`（建议中心） | 合并进 `/approval` | `Approval` | `approval:view` | 建议列表即待审列表（教程业务流中建议必经审批） |
| 9 | `/approvals` | `/approval` | `Approval` | `approval:view` | — |
| 10 | `/approvals/:id`（审批详情） | 审批弹窗实现（不开路由） | `ApprovalModal` | `approval:approve` | 弹窗承载操作，少一跳；历史查询走 `/approval/history` |
| 11 | `/reports` | `/reports` | `Reports` | `report:view` | 19.1 |
| 12 | `/reports/:id` | `/reports/:id` | `ReportDetail` | 登录即可（按钮级另有 `report:edit`/`quality:release`） | 19.1 |
| 13 | `/admin/users` | `/admin` Tab1 | `Admin` | `admin:manage` | 19.3，三个管理页合并为 Tab |
| 14 | `/admin/audits` | `/audits`（提为一级菜单） | `Audits` | `admin:manage` | 19.2，与原型侧边栏"审计日志"一级菜单一致 |
| 15 | `/admin/system` | `/admin` Tab2/Tab3 | `Admin` | `admin:manage` | 19.3（健康度 + 智能体指标） |
| 16 | `/403` | `AuthRoute` 内联 `Result 403` | — | — | 不单独开路由，渲染更即时 |
| 17 | `/404` | `*` 兜底重定向 `/dashboard` | — | — | 兜底策略二选一均可 |
| 18 | `/notifications`（原型铃铛入口，IA-SD-003 未单列） | `/notifications` | `Notifications` | 登录即可（hideInMenu） | 19.4，V1.1 教程补充 |

全部页面路由均通过 `lazy(() => import(...))` 懒加载（第 7 章范式，16.2 验收）。

#### 19.6.2 菜单分组改造：对齐原型侧边栏

原型的侧边栏是**分组菜单**（监控分析 / 处置闭环 / 系统），而我们的菜单目前是平铺的。最后一步改造：给 `RouteMeta` 加 `group` 字段，菜单构建按组投影——AntD Menu 的 group 类型原生支持。

```ts
// src/types/router.d.ts 的 RouteMeta 追加一个字段：
export interface RouteMeta {
  title: string
  icon?: string
  hideInMenu?: boolean
  permission?: string
  breadcrumb?: string
  /** 菜单分组（对齐原型侧边栏）：monitor 监控分析 / loop 处置闭环 / system 系统 */
  group?: 'monitor' | 'loop' | 'system'
}
```

`appRoutes` 各路由的 meta 补上分组（直接看 19.6.4 的最终路由表代码），然后改造 `MainLayout.tsx` 的菜单构建：

```tsx
// src/layouts/MainLayout.tsx 中 menuItems 的 useMemo 整体替换为：
const GROUP_LABEL: Record<string, string> = {
  monitor: '监控分析',
  loop: '处置闭环',
  system: '系统',
}

const menuItems = useMemo(() => {
  buildMenu(appRoutes) // 权限可能随登录变化，重建菜单（12.4）
  // 按声明顺序分组投影：group 相邻的菜单项归入同一组
  const items: Required<MenuProps>['items'] = []
  const groupKeyOf = (g?: string) => `group-${g ?? 'loop'}`
  for (const item of MENU_ITEMS) {
    const group = item.meta.group ?? 'loop'
    let groupItem = items.find(
      (i) => !!i && typeof i === 'object' && 'key' in i && i.key === groupKeyOf(group),
    ) as { key: string; type: 'group'; label: string; children: { key: string }[] } | undefined
    if (!groupItem) {
      groupItem = { key: groupKeyOf(group), type: 'group', label: GROUP_LABEL[group] ?? group, children: [] }
      items.push(groupItem)
    }
    groupItem.children.push({
      key: item.path,
      icon: ICON_MAP[item.meta.icon ?? ''],
      label: item.meta.title,
    })
  }
  return items
}, [user])
```

`selectedKeys` 的计算完全不用动——AntD Menu 的 group item 不参与选中态匹配，子项 key 仍是路由路径。

#### 19.6.3 最终路由表完整代码（全书汇总）

```tsx
// src/router/index.tsx（最终版：全部 18 条路由 + 分组 + 权限 + 懒加载）
import { lazy } from 'react'
import type { AppRouteObject } from '@/types/router'

/* ---- 懒加载页面：分包 + 首屏加速 ---- */
const Login = lazy(() => import('@/pages/Login'))
const Dashboard = lazy(() => import('@/pages/Dashboard'))
const Defects = lazy(() => import('@/pages/Defects'))
const AnomalyList = lazy(() => import('@/pages/Anomaly'))
const RootCauseDetail = lazy(() => import('@/pages/RootCause'))
const Approval = lazy(() => import('@/pages/Approval'))
const ApprovalHistory = lazy(() => import('@/pages/ApprovalHistory'))
const Reports = lazy(() => import('@/pages/Reports'))
const ReportDetail = lazy(() => import('@/pages/Reports/ReportDetail'))
const Audits = lazy(() => import('@/pages/Audits'))
const Admin = lazy(() => import('@/pages/Admin'))
const Notifications = lazy(() => import('@/pages/Notifications'))

/* ---- 需要登录的一级路由（同时是菜单来源；顺序即菜单顺序） ---- */
export const appRoutes: AppRouteObject[] = [
  /* ===== 监控分析 ===== */
  {
    path: '/dashboard',
    meta: { title: '良率看板', icon: 'dashboard', group: 'monitor' },
    element: Dashboard,
  },
  {
    path: '/defects',
    meta: { title: '缺陷查询', icon: 'defect', group: 'monitor' },
    element: Defects,
  },
  {
    path: '/anomaly',
    meta: { title: '异常事件', icon: 'anomaly', group: 'monitor' },
    element: AnomalyList,
  },
  {
    path: '/anomaly/:id',
    meta: { title: '根因分析', hideInMenu: true },
    element: RootCauseDetail,
  },
  /* ===== 处置闭环 ===== */
  {
    path: '/approval',
    meta: { title: '审批工作台', icon: 'approval', permission: 'approval:view', group: 'loop' },
    element: Approval,
  },
  {
    path: '/approval/history',
    meta: { title: '审批历史', icon: 'approval-history', permission: 'approval:view', group: 'loop' },
    element: ApprovalHistory,
  },
  {
    path: '/reports',
    meta: { title: '8D 报告', icon: 'report', permission: 'report:view', group: 'loop' },
    element: Reports,
  },
  {
    path: '/reports/:id',
    meta: { title: '报告详情', hideInMenu: true },
    element: ReportDetail,
  },
  /* ===== 系统 ===== */
  {
    path: '/audits',
    meta: { title: '审计日志', icon: 'audit', permission: 'admin:manage', group: 'system' },
    element: Audits,
  },
  {
    path: '/admin',
    meta: { title: '系统管理', icon: 'admin', permission: 'admin:manage', group: 'system' },
    element: Admin,
  },
  /* ===== 通知（入口在顶栏铃铛，不进菜单） ===== */
  {
    path: '/notifications',
    meta: { title: '通知中心', hideInMenu: true },
    element: Notifications,
  },
]
```

菜单最终形态（`admin` 登录时，与原型侧边栏逐项对照）：

```text
监控分析            ← 原型 nav-group「监控分析」
  📊 良率看板        ← 原型「良率看板」
  🐛 缺陷查询        ← （教程补充，原型未列）
  🚨 异常事件        ← 原型「异常事件」（带未读 Badge）
  🔍 根因分析        ← 教程合并进异常详情（原型为独立菜单，取舍见 19.6.1 #6/#7）
处置闭环            ← 原型 nav-group「处置闭环」
  ✅ 审批工作台      ← 原型「审批工作台」
  🕐 审批历史        ← （教程独立页，原型在审批页右侧面板）
  📄 8D 报告        ← 原型「8D 报告」
系统                ← 原型 nav-group「系统」
  🧾 审计日志        ← 原型「审计日志」
  ⚙️ 系统管理        ← 原型「系统管理」
```

> 原型侧边栏底部的"智能体链路：监控→根因→建议→审批→报告"状态脚注，教程实现在系统管理的"智能体指标"Tab（19.3），不再重复占侧边栏空间——组件树级对照见 IA-SD-003 第 6 章。

#### 19.6.4 全页面与后端接口对照表

联调后端（IA-TU-001）时按此表逐页核对（也是接口自检清单）：

| 页面 | 调用接口 | 依赖章节 |
|---|---|---|
| 登录页 | `POST /auth/login` | 5 |
| 良率看板 | `GET /dashboard/overview` · `GET /dashboard/yield-trend` · `GET /dashboard/defect-category` · `GET /dashboard/top-equipment` | 8 |
| 缺陷查询 | `GET /defects` · `GET /defects/distribution` · `GET /defects/image-url` | 19.5 |
| 异常事件列表 | `GET /anomaly/events` · WS `ANOMALY_CREATED` | 9 |
| 根因分析详情 | `GET /rootcause/:id` · `POST /rootcause/:id/reanalyze` · `GET /rootcause/task/:taskId` | 10 |
| 审批工作台 | `GET /approval/tasks` · `POST /approval/tasks/:id` | 11 |
| 审批历史 | `GET /approval/history` | 11 |
| 8D 报告列表 | `GET /reports` | 19.1 |
| 8D 报告详情 | `GET /reports/:id` · `PUT /reports/:id` · `POST /reports/:id/release` | 19.1 |
| 审计日志 | `GET /audits` | 19.2 |
| 系统管理 | `GET /admin/users` · `POST /admin/users` · `PUT /admin/users/:id` · `PUT /admin/users/:id/status` · `POST /admin/users/:id/reset-password` · `GET /admin/data-health` · `GET /admin/agent-metrics` | 19.3 |
| 通知中心（铃铛+列表） | `GET /notifications/unread-count` · `GET /notifications/recent` · `GET /notifications` · `POST /notifications/:id/read` · `POST /notifications/read-all` · WS `NOTIFICATION_PUSH` | 19.4 |
| 全局 | 401 跳登录（拦截器）· 心跳/重连（WS） | 4 / 9 |

#### 19.6.5 更新全书小结

第 18 章小结的能力域表已同步补上本章内容（打开 18.1 应能看到"补全页面"一行）。**至此，教程实现的功能 = 原型全部页面 + IA-SD-003 的 18 条路由全集**：

| 本章交付 | 验收要点 |
|---|---|
| 8D 报告（列表+详情） | D1~D8 锚点导航；草稿编辑（权限×状态双闸门）；签发 Popconfirm；window.print 打印 |
| 审计日志 | 多条件检索；CSV 导出（BOM+RFC4180）；≥500 行虚拟滚动 |
| 系统管理 | 用户 CRUD Drawer；角色改绑；启用禁用/重置密码带确认；健康度进度条；智能体指标达标 Tag |
| 通知中心 | 铃铛 Badge（30s 轮询 + WS 推送双通道）；Popover 最近 5 条；列表页跳转来源 |
| 缺陷查询 | 多条件检索；Wafer Map 预签名 URL 预览；分布饼图联动 |
| 全局 | 菜单分组对齐原型；18 条路由对照表；页面-接口对照表 |

**【验证】（收尾总验收）** `pnpm dev` 后：

1. `admin` 登录 → 侧边栏呈现"监控分析 / 处置闭环 / 系统"三个分组，共 9 个菜单项 + 顶栏铃铛；逐一点击 12 个业务页面（含 3 个详情/隐藏页）全部可达、面包屑与标签页标题正确；
2. `engineer` 登录 → 菜单少"审计日志/系统管理"两组项，其余可用；`quality` 登录 → 额外拥有 8D 编辑与签发；
3. `pnpm build` → `dist/assets` 中每个页面一个独立 chunk（懒加载验收）；再按 15.5 部署清单走一遍容器化验证；
4. 对照 19.6.4 接口表，把 Mock 分支逐个改为真实后端实现（或直接 `USE_MOCK=false` 对接 IA-TU-001 已实现的接口）——前端零改动。

---

## 16. 性能优化清单

以下清单按"收益/成本比"排序，全部已在前文埋下伏笔，此处集中验收：

### 16.1 React.memo / useMemo / useCallback 的正确使用

**先立规矩**：这三个 API 都有成本（缓存管理），**不要为优化而优化**。只在出现真实的重复渲染问题时上。

| API | 缓存什么 | 什么时候用 | 本项目实例 |
|---|---|---|---|
| `React.memo(Component)` | 组件本身，props 浅比较后跳过重渲染 | 父组件高频重渲染且子组件渲染贵 | 大表格行渲染组件 |
| `useMemo(fn, deps)` | 计算结果 | 计算贵（排序/过滤/图表 option）且依赖不常变 | 图表 option 构建 |
| `useCallback(fn, deps)` | 函数引用 | 该函数被 memo 子组件或依赖数组消费 | 表格事件回调 |

三个高频误用：

```tsx
// 误用1：无脑 useCallback —— 每次渲染都换 deps，缓存永远失效，纯付成本
const onClick = useCallback(() => doSomething(a, b), [a, b])

// 误用2：useMemo 包便宜计算 —— 字符串拼接缓存的开销比计算本身还大
const title = useMemo(() => `${a}-${b}`, [a, b])

// 误用3：React.memo 一个每次都换新 props 的组件 —— memo 永远不命中
// parent: <UserCard user={{ name }} />   // 内联对象，每次新引用
```

正确姿势回顾：表格 columns 用 `useMemo(…, [navigate])`（见 9.4），图表 option 用 `useMemo(…, [data])`（见 8.4）。

### 16.2 路由懒加载（已落地，验收）

第 7 章的 `lazy(() => import('@/pages/Dashboard'))` 已让每个页面独立分包。验收：`pnpm build` 后 `dist/assets` 内应看到每个页面一个 JS 文件；首屏只加载 `login/react/antd` 三个 chunk。**用 React.lazy + Suspense 的一切都要有 fallback**，否则切页白屏。

### 16.3 长列表虚拟滚动：异常事件 > 1000 条场景

真实工厂 FDC 告警一天几百条，按"导出明细"场景会出现 5000+ 行。AntD 5.9+ Table 原生支持虚拟滚动，一行配置：

```tsx
<Table<AnomalyEvent>
  virtual                    // 核心开关：只渲染可视区行
  scroll={{ x: 1100, y: 600 }} // virtual 模式必须给定 y 高度
  rowKey="id"
  columns={columns}
  dataSource={bigList}       // 5000 条也不卡
  pagination={false}
/>
```

实现原理速讲：以每行固定行高为刻度，根据 `scrollTop` 计算当前可视窗口应渲染的行区间（通常前后多渲几行缓冲），DOM 里始终只有几十个 `<tr>`——把"渲染 5000 个 DOM"降为"渲染 30 个 DOM + 移动 padding"。

数据侧同样有讲究：一次拉 5000 条必须用**后端分页 + 前端无限滚动**的组合（`useInfiniteQuery`），而不是把 `pageSize=5000` 一次拉爆（响应体可达数 MB，首字节时间飙升）。虚拟滚动解决"渲染"，无限滚动解决"取数"，两者搭配才是完整方案。

### 16.4 ECharts 按需引入（已落地，验收）

第 8.2 节已按需注册。量化收益：`pnpm build` 后对比注释掉按需、改为 `import * as echarts from 'echarts'` 的产物体积——全量 echarts chunk 约 1MB+，按需后 350KB 左右（gzip 前）。

### 16.5 优化检查清单（项目发版前过一遍）

- [ ] 路由全部懒加载，Suspense fallback 无白屏
- [ ] vendor 分包（react/antd/echarts）且各自体积达标
- [ ] 图表 option 全部 useMemo 稳定化
- [ ] 大列表启用 virtual 或分页
- [ ] 静态资源 immutable 长缓存生效（文件名 hash）
- [ ] React DevTools Profiler：无"输入框打字导致整页重渲染"类的失控渲染
- [ ] 图片使用现代格式与合适尺寸（缺省占位图、懒加载 `loading="lazy"`）

---

## 17. 常见坑 FAQ

**Q1：开发环境接口报 CORS 错误（blocked by CORS policy）？**
根因：前端代码里写死了绝对地址（如 `http://localhost:7001/api/v1/...`），浏览器直连后端。修复：`baseURL` 保持 `/api/v1` 相对路径，让 Vite proxy / Nginx 反代转发——浏览器视角永远同源，CORS 从原理上消失。生产同理由 Nginx 反代解决。**永远不要用后端加 CORS 头来绕过，代理才是正解**（安全与运维双优）。

**Q2：token 快过期时的刷新竞态怎么处理？**
场景：页面并行 5 个请求，token 同时过期 → 5 个 401 → 5 个刷新请求互相作废旧 token → 后 4 个拿到的已是失效 refresh_token → 全体跳登录。方案：**单飞（single flight）**——用模块级 Promise 变量保证并发 401 只发一次刷新：

```ts
// 请求层伪代码骨架（完整实现见 IA-TU-001 后端刷新接口配套）
let refreshing: Promise<string> | null = null

async function refreshTokenOnce() {
  if (!refreshing) {
    refreshing = doRefresh().finally(() => (refreshing = null))
  }
  return refreshing // 并发调用者拿到同一个 Promise
}
```

401 后先 `await refreshTokenOnce()` 重放原请求，只有刷新失败才登出。

**Q3：AntD Table 分页点了没反应/数据对不上？**
根因：`pagination` 的 `current/pageSize` 没有受控——只写了 `onChange` 却没把值回填给 `current`，或回填的 state 没进 queryKey。自检三步：① `current` 是否绑定最新 page？② page 变化是否触发请求（queryKey）？③ 换查询条件时是否强制回 `page: 1`？三者缺一就是"分页鬼畜"的元凶（见 9.4 完整范式）。

**Q4：ECharts 图表切换菜单回来变成空白/宽度只有 100px？**
两个根因：① 容器初始渲染在 `display:none` 或宽度为 0 的容器里（Tabs 非激活页、折叠动画中），ECharts 按 0 宽度初始化；② 页面隐藏期间发生 resize 但监听已失效。修复：本教程的 `useChart` 已用 ResizeObserver 兜底（菜单折叠/Tab 激活都会触发 observe 回调自动 resize）。若仍有极端场景，在 Tab 的 onChange 中调用 `chartRef.current?.resize()` 手动兜底。

**Q5：WebSocket 在 A 页面能用，切到 B 页面就断了？**
根因：把 `useWebSocket` 挂在页面组件里——页面卸载 effect cleanup 执行 `ws.close()`。修复：连接上移到"布局层"（MainLayout，生命周期覆盖所有子页面），消息通过 `realtimeStore` 分发（见 9.2/9.5 架构）。判断口诀：**连接的挂载点要比消费它的所有页面活得久**。

**Q6：Zustand persist 后退出登录了却还是"已登录"状态？**
排查：persist 的 `partialize` 是否把 token 排除了；`logout()` 是否同时清了 `user/permissions`；浏览器里旧版本 localStorage 结构与新 state 结构不一致（加字段后老用户反序列化出 undefined）——给 persist 加 `version` + `migrate` 处理结构升级。

**Q7：React Query 请求发了两次（Duplicate request）？**
最常见是 `queryKey` 引用了不稳定对象：

```tsx
// ❌ queryKey 含每次渲染都新的对象字面量/数组 -> 无限新 key -> 无限请求
queryKey: ['events', { page, pageSize }]

// ✅ key 只用原始值
queryKey: ['events', page, pageSize]
```

另一个来源是 StrictMode 开发模式下的"故意的双执行"（React 18 StrictMode 会双调 effect），生产不出现，不要为它加防重锁。

**Q8：AntD 5 里 `message.error` 报 "Static function can not consume context"？**
根因：AntD 5 的静态方法（`message.xxx`/`Modal.xxx`/`notification.xxx`）脱离 React 上下文，拿不到 ConfigProvider 的主题与 token。修复：组件内用 `const { message } = App.useApp()`（本教程登录页、审批弹窗均已示范）；**拦截器等非组件环境**才退回 `import { message } from 'antd'` 静态调用（可接受，但会出现样式脱离主题的告警，AntD 官方建议生产用动态通知桥接）。

**Q9：vite proxy / WebSocket 代理连接 404 或握手失败？**
按序检查：① `vite.config.ts` 里 ws 代理是否加了 `ws: true`；② Nginx 的 `location /ws/` 是否加了 `proxy_set_header Upgrade/Connection` 两个头；③ 前端 URL 是否写死成了 `ws://localhost:7001` 而没跟随部署域名（应按 9.1 用 `location.host` 动态拼接 + `wss` 判断）。

**Q10：打包后页面空白（dev 正常）？**
按序检查：① 路由 history 模式下刷新 404 → Nginx try_files（15.3）；② 部署在子路径（如 `/app/`）→ `vite.config.ts` 加 `base: '/app/'` 且路由器用 `createBrowserRouter` 的 `basename` 参数；③ 老浏览器兼容 → `@vitejs/plugin-legacy`（按需）；④ Console 看报错第一行——80% 的空白页是某 chunk 加载 404（大小写敏感的服务器文件系统，Windows 本地正常、Linux 404）。

**Q11：TypeScript 报 `VITE_XXX` 不在 `ImportMetaEnv` 上？**
`src/vite-env.d.ts` 已示范：`VITE_API_BASE_URL/VITE_WS_URL` 需逐个声明。新加环境变量忘记声明时 TS 会拦住你——这正是拦住"手滑拼错变量名"的防线，别用 `as any` 绕过它。

**Q12：React Query 轮询任务结束后还在发请求？**
`refetchInterval` 配置成常量后不会自动停。修复：把它写成函数，返回 `false` 即停止（本教程轮询完成依赖 `enabled` 双闸门 + `reanalyzing` state 停轮询，等价方案）：

```tsx
refetchInterval: (query) => (query.state.data?.status === 'done' ? false : 2000)
```

---

## 18. 小结与下一步

### 18.1 你已经掌握了什么

回看开篇的知识地图，你已经完成了一条完整的企业级前端主线：

| 能力域 | 落地物 |
|---|---|
| 工程化 | 脚手架、TS 严格模式、alias、ESLint/Prettier/EditorConfig、分包构建 |
| 请求层 | axios 拦截器（token/401/统一弹错）、api 模块化、Mock 插件与真实后端无缝切换 |
| 鉴权 | 登录页、persist 持久化、路由守卫、回跳来源页 |
| 布局与导航 | 配置驱动的路由表（meta：title/permission/hideInMenu）、菜单自动生成、面包屑挂靠、懒加载 |
| 服务端状态 | useQuery（缓存/轮询/keepPreviousData）、useMutation（审批失效链） |
| 可视化 | ECharts 按需引入、useChart 封装（ResizeObserver/dispose）、折线/饼/柱/graph 四类图 |
| 实时通信 | 心跳、指数退避重连、跨页面状态分发、通知铃铛 |
| 权限 | 路由级 403、菜单级过滤、按钮级 AuthButton |
| 补全页面（V1.1，第 19 章） | 8D 报告（D1~D8 锚点/草稿编辑/签发/打印）、审计日志（CSV 导出/虚拟滚动）、系统管理（用户 CRUD/健康度/智能体指标）、通知中心（轮询+WS 双通道）、缺陷查询（预签名图预览/分布饼图）；菜单分组对齐原型 |
| 质量 | 12 条 FAQ 的坑位地图、单元测试骨架 |
| 交付 | Nginx conf、多阶段 Dockerfile、compose 联调 |

### 18.2 下一步：对接后端教程（IA-TU-001）

本教程刻意把所有"后端不确定性"隔离在 `api/` 目录与接口约定之后。接下来：

1. **起后端**：按 IA-TU-001《后端开发教程（Egg.js）》搭建 industryagent-server，实现 `/api/v1` 各接口与 `/ws/v1/alarm` 推送；
2. **切流量**：`USE_MOCK=false pnpm dev`，逐页联调（登录 → 看板 → 缺陷查询 → 列表 → 详情 → 审批 → 8D 报告 → 通知中心 → 审计/系统管理），Mock 与真实后端的响应结构完全一致，理论上零前端改动；接口清单直接对照 19.6.4 的全页面-接口对照表；
3. **补基建**：后端接入 token 刷新接口后，回到 FAQ Q2 补齐单飞刷新；
4. **跑闭环**：`docker compose up` 一键拉起前后端，从"异常事件产生 → WS 推送 → 根因分析 → 审批"走一遍完整业务闭环，并用 IA-PRJ-001 的指标（MTTR、Top-3 准确率）做一次量化演示。

### 18.3 进阶练习（按难度递增）

1. **热身**：把详情页参数时序图改造为多参数可切换（legend / 参数下拉），对比"多个 series 并列"两种交互；
2. **实战**：审批历史页增加"按事件筛选 + 分页表格 + Timeline 切换"双视图；
3. **进阶**：实现异常事件的 CSV 导出（`responseType: 'blob'` + a 标签下载），并在拦截器放行二进制；
4. **深水区**：接入 Keycloak/OPA 做真实 RBAC（本教程的 permissions 从后端登录接口下发改为 JWT 解析），把权限矩阵做成后端配置；
5. **大厂题**：把 `useInfiniteQuery` 无限滚动 + Table virtual 组合落地到异常列表，导出 5000 行场景数据并做 Profiler 基准对比。

---

> **结语**：企业级前端不是"把 AntD 组件摆满屏"，而是**用约束换确定性**——接口约定、权限分层、单一数据源、统一的错误与实时通道。这套骨架你可以直接搬进任何中后台项目。V1.1 补完第 19 章后，系统功能已覆盖高保真原型与 IA-SD-003 的全部页面——从监控到闭环，从权限到审计，这是一个完整的产品形态。写完这几千行里的一半代码时，你已经走在多数转行者前面了。
>
> 下一步：打开终端，`pnpm create vite industryagent-web --template react-ts`，从第 2 章开始。

（全文完）





