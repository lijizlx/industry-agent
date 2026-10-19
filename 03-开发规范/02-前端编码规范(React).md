# 前端编码规范（React）

| 文档控制 | 内容 |
|---|---|
| 文档编号 | IA-DEV-003 |
| 文档名称 | 前端编码规范（React） |
| 版本 | V1.0 |
| 密级 | 内部 |
| 编制 | 李吉 |
| 审核 | （待填） |
| 批准 | （待填） |
| 日期 | 2026-10-15 |

## 修订记录

| 版本 | 日期 | 修订内容 | 编制/修订人 | 审核人 |
|---|---|---|---|---|
| V1.0 | 2026-10-15 | 初始版本发布 | 李吉 | — |

---

## 1. 适用范围与技术基线

本规范适用于 `industryagent-web` 仓库。技术基线：React 18 + TypeScript(strict) + Vite + Ant Design 5。所有规则由 ESLint（`eslint-plugin-react-hooks`、`@typescript-eslint`）与 CI 强制执行，冲突时以本规范为准。

## 2. 组件规范

| 规则 | 说明 |
|---|---|
| 只用函数组件 + hooks | 禁止 class 组件（新代码），存量 class 组件改造时重写 |
| 命名 PascalCase | 组件名与文件名一致：`YieldTrendChart.tsx` 导出 `YieldTrendChart` |
| 单文件单组件 | 一个 `.tsx` 只导出一个组件；类型、常量可随文件导出 |
| 行数上限 300 | 超过 300 行必须拆分子组件或抽自定义 hook |
| props 接口化 | props 必须显式 `interface XxxProps`，禁止解构后重命名混乱 |
| 组件内禁止直接请求 | 请求必须走统一请求层（见第 7 章） |

```tsx
// ✅ 正例：标准组件结构
import { useState } from 'react';
import { Table, Tag } from 'antd';
import { useLotList } from '@/hooks/use-lot-list';

/** 良率批次列表属性 */
interface LotTableProps {
  /** 默认产品过滤 ID */
  productId?: number;
  /** 行点击回调 */
  onRowClick?: (lotId: string) => void;
}

/**
 * 良率批次列表
 * @description 展示批次良率并高亮异常批次（Z-Score 超阈值）
 */
export function LotTable({ productId, onRowClick }: LotTableProps) {
  const [page, setPage] = useState(1);
  const { data, isLoading, isError } = useLotList({ productId, page });

  if (isError) return <LotTableErrorRetry onRetry={() => refetch()} />;
  return <Table loading={isLoading} ... />;
}
```

```tsx
// ❌ 反例：class 组件 + 多职责 + 越权请求
class LotTable extends React.Component {
  state = { data: [], loading: false };
  componentDidMount() {
    fetch('http://localhost:7001/api/lots')       // 组件内裸 fetch
      .then((r) => r.json())
      .then((data) => this.setState({ data }));
  }
  render() { /* 300+ 行 JSX 混合表格、弹窗、表单 */ }
}
```

## 3. TypeScript 规范

1. **禁 any**：`@typescript-eslint/no-explicit-any: error`。第三方库缺类型时写 `.d.ts` 声明。
2. 接口先建模：所有 API 响应、业务实体必须先定义 `interface`/`type`，与后端 DTO 对齐。
3. 联合类型 + 类型收窄代替宽松类型。

```typescript
// ✅ 正例：显式建模 + 类型收窄
type AgentTaskStatus = 'pending' | 'running' | 'succeeded' | 'failed';

interface RootCauseReport {
  taskId: string;
  status: AgentTaskStatus;
  suspects: Suspect[];
}

function renderStatus(report: RootCauseReport) {
  switch (report.status) {
    case 'succeeded': return <Tag color="green">完成</Tag>;
    case 'failed':    return <Tag color="red">失败</Tag>;
    default:          return <Tag color="blue">进行中</Tag>;
  }
}

// ❌ 反例
const report: any = await getData();       // any 逃逸
function renderStatus(status: string) { ... } // 宽松 string，无法穷举
```

## 4. Hooks 规范

1. 自定义 hook 必须 `useXxx` 命名，文件 `use-xxx.ts`，放 `src/hooks/`。
2. **依赖数组必须完整**：`react-hooks/exhaustive-deps: error`。禁止为"修复"渲染问题而删除依赖。
3. hook 内禁止条件调用、禁止在普通函数中调用。
4. 组件逻辑超过两个副作用或复用明显时，抽自定义 hook。

```typescript
// ✅ 正例：依赖完整
useEffect(() => {
  const timer = setInterval(() => poll(taskId), 5000);
  return () => clearInterval(timer);   // 清理副作用，防内存泄漏
}, [taskId]);

// ❌ 反例：依赖缺失（taskId 变化后闭包读到旧值）
useEffect(() => {
  const timer = setInterval(() => poll(taskId), 5000);
  return () => clearInterval(timer);
}, []); // eslint-disable-line react-hooks/exhaustive-deps
```

```typescript
// ✅ 正例：自定义 hook 收敛轮询逻辑
export function useAgentTaskPolling(taskId: string | null) {
  const [report, setReport] = useState<RootCauseReport | null>(null);
  useEffect(() => {
    if (!taskId) return;
    const timer = setInterval(async () => {
      const r = await api.agent.getTask(taskId);
      setReport(r);
      if (r.status === 'succeeded' || r.status === 'failed') clearInterval(timer);
    }, 5000);
    return () => clearInterval(timer);
  }, [taskId]);
  return report;
}
```

## 5. 状态管理规范

| 状态类别 | 方案 | 示例 |
|---|---|---|
| 服务端状态（列表、详情、任务状态） | React Query（TanStack Query v5） | 批次列表、根因分析任务 |
| 客户端全局状态 | Zustand | 当前用户、主题、全局 UI 开关 |
| 组件局部状态 | useState | 弹窗开关、表单草稿 |

**禁止**把服务端数据复制进 Zustand 再手工同步；**禁止**一仓库一 store，按业务域划分：

```typescript
// ✅ 正例：按域划分 store（src/stores/auth-store.ts）
import { create } from 'zustand';

interface AuthState {
  currentUser: User | null;
  permissions: string[];
  setUser: (u: User | null) => void;
}

export const useAuthStore = create<AuthState>((set) => ({
  currentUser: null,
  permissions: [],
  setUser: (u) => set({ currentUser: u, permissions: u?.permissions ?? [] }),
}));

// ✅ 正例：服务端状态用 React Query
const { data } = useQuery({
  queryKey: ['lots', { productId, page }],
  queryFn: () => api.yield.getLotList({ productId, page }),
  staleTime: 60_000,
});
```

```typescript
// ❌ 反例：用 Zustand 缓存服务端数据，手工同步
const useLotStore = create((set) => ({
  lots: [],
  fetchLots: async () => set({ lots: await api.get('/lots') }), // 缓存/失效/重试全部丢失
}));
```

## 6. CSS 规范

**统一选型：Less + Ant Design 5 token（CSS-in-JS 变量）+ 局部 CSS Modules。**

选型理由：项目重度使用 AntD 5，其主题定制依赖 design token；Less 与 AntD 生态天然兼容；CSS Modules 保证局部作用域，避免样式冲突。**不引入 Tailwind**，避免与 AntD 样式体系双轨维护。

```tsx
// ✅ 正例：AntD token 定制主题（全局一次）
import { ConfigProvider, theme } from 'antd';

<ConfigProvider
  theme={{
    token: { colorPrimary: '#165DFF', borderRadius: 4 },
    algorithm: theme.defaultAlgorithm,
  }}
>
  <App />
</ConfigProvider>

// ✅ 正例：组件局部样式用 CSS Modules（lot-table.module.less）
.errorRow { background: var(--ia-error-bg); }
```

```less
/* ❌ 反例：全局 less 裸类名，造成全局污染 */
.table-row { background: red !important; }
```

禁止：内联 `style={{...}}` 写静态样式（动态计算值除外）、`!important`、覆盖 AntD 内部类名（`.ant-table-cell`）超过 10 处——出现时优先用 token。

## 7. 请求层规范

统一请求入口 `src/api/`，按业务域拆分模块，底层封装 `request.ts`（自动附加 Bearer token、requestId、统一错误提示、401 跳登录）。

```typescript
// src/api/yield.ts
import { request } from './request';

/** 分页查询批次列表 */
export const getLotList = (params: LotListQuery) =>
  request<PageResult<LotItem>>({ url: '/api/v1/yield/lots', method: 'GET', params });
```

```typescript
// ❌ 反例：组件内裸写 fetch，无鉴权、无统一错误处理
useEffect(() => {
  fetch('/api/v1/yield/lots?page=1')
    .then((r) => r.json())
    .then(setData);
}, []);
```

## 8. 性能规范

| 场景 | 规则 |
|---|---|
| `React.memo` | 仅在组件渲染成本可测量地高（如 500+ 行表格行组件）且父组件高频重渲染时使用；禁止无脑包裹 |
| 虚拟列表 | 长度 > 100 的列表/下拉必须用虚拟滚动（`react-window` 或 AntD VirtualTable） |
| 图片 | 缺陷图（MinIO 预签名 URL）必须懒加载 |
| 包体积 | 路由级 `React.lazy` 代码分割；单 chunk > 500KB 触发 CI 告警 |
| 大数据图表 | ECharts 实例在卸载时 `dispose()`，resize 监听随组件清理 |

```tsx
// ✅ 正例：虚拟列表（缺陷点位 5000+）
import { List } from 'react-window';
const Row = ({ index, style }) => <div style={style}>{defects[index].dieCoord}</div>;
<List height={600} itemCount={defects.length} itemSize={36}>{Row}</List>;

// ❌ 反例：5000 条直接 map 渲染 DOM
{defects.map((d) => <DefectRow key={d.id} defect={d} />)}
```

## 9. 可访问性基础要求

1. 所有可交互元素用原生语义标签或 AntD 组件（自带 a11y），禁止 `<div onClick>`。
2. 图片提供 `alt`；图标按钮提供 `aria-label`。
3. 表单控件必须绑定 `label`（AntD Form Item 默认满足）。
4. 键盘可达：弹窗 Esc 关闭、表格可聚焦；禁止移除 outline 而不提供替代焦点样式。
5. 颜色对比度 ≥ 4.5:1（antd token 默认满足，自定义色需走工具校验）。

```tsx
// ❌ 反例：div 当按钮 + 无障碍属性缺失
<div onClick={onExport}>导出报告</div>

// ✅ 正例
<Button onClick={onExport} aria-label="导出8D报告">导出报告</Button>
```

## 10. 测试要求

- vitest + React Testing Library；工具函数单测覆盖率 ≥ 80%，核心组件（良率趋势图、批次表、审批流）必须有交互测试。

```tsx
import { render, screen, fireEvent } from '@testing-library/react';

it('批次状态为 failed 时展示失败标签', () => {
  render(<TaskStatusTag status="failed" />);
  expect(screen.getByText('失败')).toBeInTheDocument();
});
```

## 11. 检查清单速览

- [ ] 函数组件 + hooks，无 class、无裸 fetch
- [ ] 文件单组件，≤ 300 行，PascalCase
- [ ] 无 any，props/响应均有接口建模
- [ ] hooks 依赖数组完整，副作用有清理
- [ ] 服务端状态走 React Query，客户端状态按域分 store
- [ ] 样式走 Less + token / CSS Modules，无全局污染
- [ ] 长列表虚拟化，图表实例清理
- [ ] 可交互元素语义化，基础 a11y 达标
