# Code Review 检查清单

| 文档控制 | 内容 |
|---|---|
| 文档编号 | IA-DEV-005 |
| 文档名称 | Code Review 检查清单 |
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

## 1. CR 流程

```text
开发者                          评审人                        CI / 合并
  │  提 PR（模板齐全） ──────────→ │
  │                              │  24h 内首次响应（SLA）
  │                              │  按清单逐项检查
  │   ← 请求变更 / 批准 ────────── │
  │  逐条回复并修改，rebase develop │
  │  ─────────────────────────────│────→ CI 必须全绿
  │                              │  核心模块需第 2 位评审人 Approve
  │  ← squash merge ──────────────│────→ 自动合并、删除远端分支
```

1. **提 PR**：PR 模板（见 IA-DEV-004 第 4.3 节）必填完整，缺模板直接打回；
2. **至少 1 人评审通过**：核心模块（鉴权、错误处理中间件、DB 迁移、智能体编排、审批流）须 **2 人**，其中 1 人为该模块 Owner；
3. **CI 全绿才可合并**：lint、type-check、单测、覆盖率（后端 ≥ 70%、前端工具函数 ≥ 80%）、迁移 dry-run；
4. **合并策略**：squash merge；
5. **评审时效 SLA**：评审人须在 **24 小时内**给出首次响应（Approve 或 Request Changes）；超时未响应，PR 作者可 @ 备选评审人（各模块设主/备评审人各 1 名）。
6. 评审意见分级：`[Must]` 必须修改才能通过；`[Should]` 强烈建议本次修改；`[Nit]` 可选（作者自行决定，可不回复）。

## 2. 分级检查清单

评审时逐项勾选。任一 Must 项未通过即 Request Changes。

### 2.1 通用项（所有仓库适用）

- [ ] 命名符合 IA-DEV-001 总则（camelCase/PascalCase/UPPER_SNAKE，见名知义）
- [ ] 无重复代码：≥ 3 处相似代码块必须抽公共函数/组件
- [ ] 无死代码：未使用的变量、导入、函数、注释掉的代码块已删除
- [ ] 无魔法值：阈值/状态码/错误码已收敛为常量并注明出处
- [ ] 日志规范：分级正确、含关键上下文（requestId/lotId）、无敏感信息
- [ ] 函数圈复杂度 ≤ 15、参数 ≤ 5、嵌套 ≤ 4 层
- [ ] 注释解释"为什么"，公共导出有 JSDoc
- [ ] 无 TODO 无主：所有 TODO 均关联 issue 号

### 2.2 后端项（industryagent-server）

- [ ] 分层正确：Controller 薄、Service 厚，无跨层调用（Controller 直查 DB 一票否决）
- [ ] **SQL 注入**：全部参数化查询 / ORM where 条件，无字符串拼接 SQL
- [ ] **事务边界**：事务在 Service 闭合；事务内无 HTTP/Kafka 等慢 IO；失败可回滚
- [ ] **N+1 查询**：循环内无逐条查询，列表聚合用 JOIN 或批量 IN 查询
- [ ] **错误码**：抛出 BizError 且错误码已登记 error-codes.ts，HTTP 状态码与语义匹配
- [ ] **权限校验**：新接口已在 router 上声明权限中间件；资源级越权（IDOR）已校验归属
- [ ] 参数校验：DTO 规则完整，未校验的入参不存在
- [ ] 定时任务有分布式锁与失败告警
- [ ] 单文件 ≤ 500 行

```typescript
// ❌ N+1 反例（CR 必打回）
for (const lot of lots) {
  const product = await this.ctx.model.Product.findByPk(lot.productId);
}
// ✅ 批量查询
const products = await this.ctx.model.Product.findAll({
  where: { id: { [Op.in]: lots.map((l) => l.productId) } },
});
```

### 2.3 前端项（industryagent-web）

- [ ] **类型安全**：无 any、无 @ts-ignore；API 响应有接口建模
- [ ] **内存泄漏**：setInterval/setTimeout/事件监听/WebSocket 在卸载时清理；ECharts dispose
- [ ] **竞态**：异步请求有竞态防护（useEffect cleanup 中丢弃过期响应或使用 AbortController）
- [ ] **加载态与错误态**：每个数据视图三态齐全（loading / error+重试 / empty）
- [ ] hooks 依赖数组完整；组件 ≤ 300 行、单文件单组件
- [ ] 长列表（>100 条）虚拟化
- [ ] 无组件内裸 fetch，请求统一走 `src/api/`
- [ ] 受控组件有受控来源，无警告日志

```typescript
// ❌ 竞态反例：快速切换批次时旧响应可能覆盖新响应
useEffect(() => {
  getLotDetail(lotId).then(setDetail);
}, [lotId]);

// ✅ 正例：过期请求丢弃
useEffect(() => {
  let stale = false;
  getLotDetail(lotId).then((d) => { if (!stale) setDetail(d); });
  return () => { stale = true; };
}, [lotId]);
```

### 2.4 智能体项（industryagent-agent）

- [ ] **提示词版本化**：prompt 模板入库并带版本号（如 `prompts/rootcause_v3.txt`），修改提示词必须走 PR，禁止硬编码在函数体内
- [ ] **工具权限**：Agent 可调用的工具（DB 查询、图谱查询、RAG 检索、模型推理）已按角色声明白名单；高危工具（调参建议下发）必须走人工审批
- [ ] **Token 成本**：上下文构造有截断/压缩策略；超长输入有预算上限并记录实际 Token 消耗到指标
- [ ] **幻觉防护**：输出包含结论引用来源（图谱节点/文档片段）；关键数值字段经规则校验（如良率范围 0-1）
- [ ] **超时与重试**：LLM/工具调用设超时（默认 30s）与有限重试（≤ 3 次、指数退避）；最终失败返回 IA-40001/IA-40010 而非挂死
- [ ] 人机协同：审批 Agent 记录审批人、时间、理由，审计日志完整

```python
# ❌ 反例：提示词硬编码 + 无重试 + 无预算
async def analyze(event):
    prompt = "你是根因分析专家，请分析：" + str(event)   # 提示词未版本化
    return await llm.chat(prompt)                        # 无超时、无重试、无成本记录

# ✅ 正例
async def analyze(event: FdcEvent) -> RootCauseResult:
    prompt = load_prompt("rootcause", version=PROMPT_VERSION)  # 版本化
    async for attempt in AsyncRetrying(stop=stop_after_attempt(3),
                                        wait=wait_exponential()):
        with attempt:
            return await asyncio.wait_for(llm.chat(prompt), timeout=30)
```

### 2.5 安全项（全仓库，Must 级）

- [ ] **密钥硬编码**：无任何密码/token/API Key/连接串字面量（一票否决）
- [ ] **越权**：接口有资源归属校验；前端隐藏按钮不作为权限依据
- [ ] **注入**：SQL/Neo4j Cypher/命令注入均已参数化
- [ ] **敏感日志**：手机号/邮箱/token/晶圆坐标等敏感字段已脱敏或不出现在日志
- [ ] 上传文件类型/大小校验；MinIO 预签名 URL 有效期 ≤ 15 分钟

### 2.6 性能项

- [ ] 接口 P95 延迟有评估：新增查询走索引，慢查询目标 < 200ms
- [ ] 无明显大对象拷贝/深循环嵌套（O(n²) 以上需说明数据规模上限）
- [ ] 缓存使用合理：有失效策略，禁止缓存鉴权敏感数据
- [ ] 前端 bundle 无超标 chunk（> 500KB 触发告警）

## 3. 常见反模式清单（CR 高频打回项）

| # | 反模式 | 后果 | 正确做法 |
|---|---|---|---|
| 1 | Controller 写业务逻辑 | 无法复用、无法测试 | 下沉 Service |
| 2 | 拼接 SQL 字符串 | SQL 注入 | 参数化查询 |
| 3 | catch 后吞异常返回 `{ok:false}` | 故障不可见 | 抛 BizError 或记 error 日志 |
| 4 | 循环内逐条 DB 查询（N+1） | 列表接口秒级超时 | 批量 IN / JOIN |
| 5 | `useEffect` 依赖数组留空数组 + disable 注释 | 闭包读到旧值、状态错乱 | 补全依赖、重设计时器 |
| 6 | 组件内裸 fetch | 无鉴权、无统一错误、竞态 | 统一请求层 + React Query |
| 7 | 提示词写死在代码里 | 无法 A/B、无法回滚 | 提示词文件版本化 |
| 8 | 事务里发 Kafka / 调外部 HTTP | 长事务持锁、死锁 | 事务外发送（outbox 模式） |
| 9 | any + @ts-ignore 连锁 | 类型系统名存实亡 | 建模、类型收窄、@ts-expect-error+原因 |
| 10 | PR 混入无关格式化 diff（全量 prettier） | 淹没真实变更 | 格式化独立成 chore PR |
| 11 | 硬编码密钥"临时先跑通" | 安全事故 | 环境变量 + Secret 管理 |
| 12 | 删代码靠注释"以防万一" | 代码腐烂 | 直接删，信 Git |

## 4. 评审人角色与纪律

| 角色 | 人员 | 责任 |
|---|---|---|
| 评审 Owner | 各模块负责人（后端/前端/算法各指定） | 核心模块必审 |
| 备选评审人 | 同岗位另 1 人 | 主评审 24h 未响应时接替 |
| DevOps 会签 | DevOps 1 人 | infra 仓库、CI 配置、迁移文件变更 |

评审纪律：

1. 对事不对人，批评代码不批评人；
2. 每条 `[Must]` 意见给出修改建议或参考链接，禁止只说"这样不行"；
3. 作者须**逐条回复**评审意见（采纳/不采纳+理由），未回复不得合并；
4. 评审人对自己 Approve 的代码承担连带责任（线上问题复盘时追溯评审记录）。

## 5. SLA 与度量

| 指标 | 目标 |
|---|---|
| 首次响应时长 | ≤ 24 小时 |
| PR 平均合并时长 | ≤ 3 个工作日 |
| 核心模块 PR 合并时长 | ≤ 5 个工作日（含 2 人评审） |
| CR 打回率 | 20%~40% 为健康区间（过低说明清单失效，过高说明规范培训不足） |

每双周站会公布 CR 指标，连续两次超 SLA 的模块由 PM 协调增加评审人力。

## 6. 评审意见书写规范

评审意见的质量直接决定一轮通过率。统一按 `级别 + 位置 + 问题 + 建议` 四段式书写。

```text
# ✅ 正例：具体、可执行
[Must] app/service/yield/lot.service.ts:87
循环内逐条查询 product，100 批次会打 100 次 DB。
建议：循环外先用 Op.in 批量查出 products，再按 id 建 Map 索引。参考 IA-DEV-002 第 5 章。

[Should] web/src/pages/analysis/TaskPanel.tsx:42
任务状态轮询没有失败终止条件，failed 后会一直轮询。
建议：succeeded/failed 都 clearInterval。

[Nit] 命名 pendingList 建议改为 pendingApprovals，更贴近业务语义。

# ❌ 反例：模糊、不可执行
这里不太好，你改一下。
为什么不用缓存？（无上下文、无替代方案）
这个写法不优雅。
```

## 7. 特殊场景处理

| 场景 | 处理策略 |
|---|---|
| 超大 PR（> 800 行 diff） | 原则上打回拆分；确需大 PR（如批量重命名）须在描述中说明并申请 Owner 专项评审 |
| 紧急修复（生产 P0） | 允许 1 人评审 + 技术负责人口头批准先行合并，24h 内补齐第二人评审并登记 |
| 文档/纯注释 PR | 1 人评审或 PM 自审，可豁免 CI 单测 |
| 迁移文件 PR | 必须 DBA 会签（后端负责人兼任），按 IA-DEV-006 第 8 章检查两阶段兼容性 |
| 生成代码（脚手架/模板） | 须声明生成工具与版本，人写部分单独标注后再评审 |

## 8. 附录 A：PR 自检卡（开发者提 PR 前自查）

提 PR 前 5 分钟自查，可消灭约 60% 的打回项：

- [ ] 本地 `npm run lint`、`tsc --noEmit`、`npm test` 全部通过
- [ ] PR 描述模板填写完整，关联了 issue
- [ ] diff 中没有：console.log、注释掉的代码、调试用任意值、硬编码密钥
- [ ] 新增接口有 OpenAPI 注解；新增表字段有 migration
- [ ] 涉及 UI 的变更附了截图
- [ ] 评审人已按模块指定（核心模块指定 2 人）

## 9. 附录 B：模块 Owner 与评审人映射

| 模块 | 仓库 | Owner | 备选评审人 | 评审级别 |
|---|---|---|---|---|
| 鉴权与用户 | server | 后端负责人 | 后端工程师 B | 2 人 |
| 统一错误处理/中间件 | server | 后端负责人 | 后端工程师 B | 2 人 |
| 良率分析业务 | server | 后端工程师 B | 后端负责人 | 1 人 |
| 数据库迁移 | server | 后端负责人（DBA 会签） | DevOps | 2 人 |
| 智能体编排 | agent | 算法负责人 | 算法工程师 B | 2 人 |
| 审批流与人机协同 | agent / web | 算法负责人 | 前端负责人 | 2 人 |
| 良率看板与图表 | web | 前端负责人 | 前端工程师 B | 1 人 |
| 基础组件与请求层 | web | 前端负责人 | 前端工程师 B | 2 人 |
| CI/CD 与部署配置 | infra | DevOps | 后端负责人 | 1 人 + 会签 |

人员变动时由 PM 在周会更新本表并同步至仓库 `docs/OWNERS.md`。

## 10. 附录 C：争议裁决机制

1. 作者与评审人就 `[Must]` 意见无法达成一致时，48 小时内提交技术委员会（李吉 + 相关模块 Owner）裁决；
2. 裁决结论记录在 PR 评论区，涉及规范条文的回写至本文件修订记录；
3. 裁决期间 PR 保持 open，不得以"先合并后优化"为由绕过。

---

**本规范自发布之日起生效，全员 CR 行为以本清单为准。执行中发现条目缺失或判定歧义，请提 issue（label: spec-cr）并抄送技术委员会。**
