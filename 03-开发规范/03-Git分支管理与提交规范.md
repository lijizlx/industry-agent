# Git 分支管理与提交规范

| 文档控制 | 内容 |
|---|---|
| 文档编号 | IA-DEV-004 |
| 文档名称 | Git 分支管理与提交规范 |
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

## 1. 分支模型

IndustryAgent 四个仓库（industryagent-server / web / agent / infra）统一采用下述分支模型：

```text
                          ┌─────────────── main（受保护，仅 release/hotfix 合入）───────┐
                          │                        │                                    │
                          │                        │ merge（squash）                     │ merge（squash）
                    release/1.2.0 ─────────────────┘                                    │
                          │ tag: v1.2.0                                               │
                          │                                                           │
   develop ───────────────┴────────────────────────────────────────────────────────────┴────────→
      │        ▲                ▲                ▲                ▲
      │        │ merge（squash） │                │                │
      │  feature/IA-101-xxx  feature/IA-102-xxx  │         bugfix/IA-205-xxx
      │                                          │
      │                                     release/1.1.1（回归修复）
      │                                          │
   hotfix/1.1.2 ───────── 同时合入 main 与 develop（cherry-pick / merge-back） ──────────────→
                          │
                          │ tag: v1.1.2
```

| 分支 | 来源 | 去向 | 命名规则 | 保护策略 |
|---|---|---|---|---|
| main | — | — | main | 禁止直推；仅接受 release/hotfix 的 PR；合并须 2 人评审 + CI 绿 |
| develop | main | — | develop | 禁止直推；仅接受 PR 合并 |
| feature/* | develop | develop | `feature/IA-<issue>-kebab-desc` | 个人分支，可不评审 |
| bugfix/* | develop 或 release/* | 同源分支 | `bugfix/IA-<issue>-kebab-desc` | 合入 develop 可 1 人评审；合入 release 须 2 人 |
| release/* | develop | main + develop | `release/<SemVer>` | 禁止直推；仅接受 bugfix PR |
| hotfix/* | main | main + develop | `hotfix/<SemVer>-IA-<issue>` | 合并须 2 人评审 |

## 2. 分支命名规范

```bash
# ✅ 正例
feature/IA-123-rootcause-agent
feature/IA-201-wafer-defect-map
bugfix/IA-305-lot-query-timeout
hotfix/1.1.2-IA-401-token-expired
release/1.2.0

# ❌ 反例
feature/rootcause                 # 缺 issue 号，无法追溯
feature/IA-123                    # 只有编号，分支列表里无法辨识
bugfix/修复超时问题                # 中文 + 无编号
dev-liji                          # 个人长期分支（禁止长期私人分支）
```

规则：issue 编号必填（GitHub Issue / 项目看板卡片）；描述用英文 kebab-case，不超过 5 个单词；一个分支只做一件事。

## 3. Commit Message 规范（Conventional Commits）

格式：

```text
<type>(<scope>): <subject>

[body]

[footer]
```

- **type**：`feat | fix | docs | refactor | test | chore | perf | ci | build`
- **scope**（可选，枚举）：`server | web | agent | infra | db | api | auth | yield | agent-core`
- **subject**：祈使句、英文小写开头、≤ 72 字符、结尾不加句号
- **footer**：关联 issue `Closes #123`；破坏性变更 `BREAKING CHANGE: <说明>`
- commitlint 在 `commit-msg` hook 强制校验（配置见 IA-DEV-001 第 8 章）

### 10 组正反例

```text
✅ 1. feat(yield): add z-score anomaly detection for lot yield
❌ 1. 新功能：加了异常检测                    # 非 Conventional 类型、中文 subject、无法解析

✅ 2. fix(server): resolve N+1 query in lot list API. Closes #305
❌ 2. fix: fixed some bugs                    # 无 scope、subject 无信息量

✅ 3. docs(api): update error code table for IA-40xxx agent errors
❌ 3. update docs                             # 缺 type/scope、描述含糊

✅ 4. refactor(agent-core): extract graph node builders from orchestrator
❌ 4. REFACTOR: refactor code                 # type 大写、subject 重复无意义

✅ 5. test(web): add interaction tests for TaskStatusTag component
❌ 5. add test file                          # 缺 type、无被测对象

✅ 6. chore(infra): bump postgres image to 16.4 in docker-compose
❌ 6. chore: 修改了一下配置                    # 中文、无具体对象

✅ 7. perf(db): add covering index idx_wafer_lots_product_status. Closes #401
❌ 7. make it faster                          # 无 type/scope、无法定位优化点

✅ 8. feat(api)!: rename /v1/lots to /v1/yield/lots

    BREAKING CHANGE: API path renamed, old path returns 410 for two releases. Closes #512
❌ 8. feat: 改了接口路径                       # 破坏性变更未声明 BREAKING CHANGE

✅ 9. fix(auth): mask user phone in access log output
❌ 9. fix(server): fix                        # subject 为空话

✅ 10. ci: require postgres migration dry-run job before deploy
❌ 10. Update .github/workflows/ci.yml        # 文件路径不是 subject、无 type
```

## 4. PR / MR 规范

### 4.1 合并三要素

1. **至少 1 名评审人 Approve**（核心模块——鉴权、错误处理、数据库迁移、智能体编排——须 2 人，其中 1 人为模块 Owner）；
2. **CI 全绿**（lint + type-check + 单测 + 覆盖率 + 迁移 dry-run）；
3. **Squash merge**：一个 PR 压缩为一个 commit 进入目标分支，保持主线历史干净；commit 标题即 PR 标题。

### 4.2 PR 标题模板

```text
<type>(<scope>): <subject> (IA-<issue>)
```

示例：`feat(yield): add root-cause report export (IA-123)`

### 4.3 PR 描述模板（全文，保存为 `.github/pull_request_template.md`）

```markdown
## 关联
- Issue: Closes #123
- 看板卡片: IA-123

## 变更说明
<!-- 做了什么、为什么这么做；50 字以上，说清背景 -->
实现根因分析报告导出为 PDF：后端新增 /v1/yield/lots/:id/report/export 接口，
前端审批详情页增加导出按钮，复用 8D 报告渲染模板。

## 变更类型
- [ ] feat 新功能
- [ ] fix 缺陷修复
- [ ] refactor 重构（无行为变化）
- [ ] docs 文档
- [ ] chore/ci 工程化

## 影响面
<!-- 涉及的接口/表/组件；破坏性变更必须勾选并说明迁移方案 -->
- [x] API: 新增 GET /v1/yield/lots/:id/report/export
- [ ] DB migration: 无
- [ ] 前端页面: 审批详情页
- [ ] 破坏性变更: 无

## 自测清单
- [ ] 本地 lint / type-check / 单测通过
- [ ] 新增/修改的接口有 OpenAPI 注解与示例
- [ ] 涉及 DB 的变更已提交 migration 文件（可回滚）
- [ ] 敏感信息未打印至日志

## 截图/演示
<!-- 前端变更必附；后端接口变更附 curl 示例响应 -->
```

### 4.4 分支同步

- feature 分支保持与 develop 同步：优先 `git rebase develop`（PR 未评审前），冲突大时再 merge。
- PR 过期 30 天未处理将被机器人自动关闭（可申请重开）。

## 5. 版本号管理（SemVer + release 分支）

版本格式：`MAJOR.MINOR.PATCH`，预发布加后缀：`v1.2.0-rc.1`。

| 变更类型 | 版本动作 | 示例 |
|---|---|---|
| 破坏性 API 变更、大架构调整 | MAJOR | v1.x → v2.0.0 |
| 向后兼容的功能新增 | MINOR | v1.2.x → v1.3.0 |
| 缺陷修复 | PATCH | v1.2.0 → v1.2.1 |

流程：develop 功能收敛 → 从 develop 切 `release/1.3.0` → 仅允许 bugfix 进入 → 回归测试通过 → PR 合入 main（2 人评审）→ 打 `v1.3.0` tag → 同步合回 develop。

## 6. 冲突处理

1. **禁止**在共享分支（develop/main/release）上直接解决冲突后直推；
2. feature 冲突：在自己的分支上 `git fetch origin && git rebase origin/develop`，解决后 `git push --force-with-lease`（禁止裸 `--force`）；
3. 冲突涉及他人代码时，必须 @ 原作者共同确认，不允许静默覆盖他人改动；
4. release 分支冲突由该 release 负责人（后端负责人指定）协调。

## 7. 紧急 Hotfix 流程（完整命令）

适用场景：生产环境 P0/P1 缺陷（如根因分析接口 500、鉴权失效）。

```bash
# 1. 确认生产版本与缺陷现象，建 issue（如 IA-901）
git fetch origin
git checkout main
git pull origin main
git tag --list 'v*' | tail -1        # 确认当前生产 tag，如 v1.2.0

# 2. 从 main（生产 tag）切出 hotfix 分支
git checkout -b hotfix/1.2.1-IA-901 main

# 3. 最小化修复（只修缺陷，禁止夹带功能），提交
git add -A
git commit -m "fix(server): prevent 500 when agent service timeout. Closes #901"

# 4. 补充/更新回归测试，本地验证
npm run test

# 5. 推送并发起两个 PR：
#    PR-1 → base: main  （2 人评审 + CI 绿 + squash merge）
#    PR-2 → base: develop（merge-back，保持 develop 同步修复）
git push origin hotfix/1.2.1-IA-901

# 6. PR-1 合并后，负责人在 main 上打 tag 并推送
git checkout main && git pull origin main
git tag -a v1.2.1 -m "hotfix: agent timeout crash (IA-901)"
git push origin v1.2.1

# 7. CI/CD 自动触发生产部署（industryagent-infra 流水线），部署后验证 + 复盘登记
```

事后 48 小时内补交复盘记录（模板见手册 14.3 节）到 `docs/postmortem/`。

## 8. Tag 规范

| 规则 | 示例 |
|---|---|
| 正式版 | `v1.0.0`、`v1.2.1` |
| 候选版 | `v1.0.0-rc.1`（release 回归阶段打在 release 分支） |
| 格式 | `v` + SemVer；统一 annotated tag（`git tag -a`） |

```bash
# ✅ 正例
git tag -a v1.2.0 -m "release: yield root-cause report v1.2.0"
git push origin v1.2.0

# ❌ 反例
git tag v1.2                      # 非 SemVer
git tag release-final             # 命名随意
```

## 9. 检查清单速览

- [ ] 分支名含 issue 号且 kebab-case
- [ ] commit 全部符合 Conventional Commits（commitlint 校验通过）
- [ ] PR 使用标准模板，描述完整
- [ ] 评审人数达标（核心模块 2 人），CI 全绿
- [ ] 使用 squash merge，未直推共享分支
- [ ] 版本/tag 符合 SemVer，hotfix 已 merge-back 到 develop
