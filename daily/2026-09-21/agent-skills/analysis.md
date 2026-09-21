# 下一步：`npx skills add addyosmani/agent-skills` 装全部 25 个 skill，或 `/plugin marketplace add addyosmani/agent-skills` 在 Claude Code 里原生装。

> 快照日期：2026-09-21 | 来源：GitHub Trending daily（since=daily）| 当日仅 13 个项目（周一 UTC 午夜刚重置），非完整 20 个
> 抓取时间：2026-09-21 | Stars：97,842 | 当日新增：736 | 主语言：JavaScript（实际纯 Markdown skill + TOML 命令包装）
> 项目类别：B 类（代码架构 / skill 集合）
> 仓库地址：https://github.com/addyosmani/agent-skills

**TL;DR：** Addy Osmani（Chrome 团队）出品的"AI coding agent 生产级工程技能集"——25 个 skill + 9 个 slash command + 4 个 agent persona + 7 个 reference checklist，覆盖 define→plan→build→verify→review→ship 全生命周期。核心理念：把 Google 工程实践（Hyrum's Law、Beyonce Rule、Chesterton's Fence、trunk-based、Shift Left）编码成 agent 可执行的 workflow，每个 skill 都带"反合理化表"和"验证要求"。

---

## 1. 场景问题：该项目主要解决什么问题

- [ ] **场景 A：AI 写代码快但不生产级**
  - 目标用户：用 Claude Code / Cursor / Codex 的工程师
  - 痛点：agent 默认最短路径——不写 spec、跳过测试、不做 review
  - 介入方式：25 个 skill 把 senior engineer 的纪律（TDD、spec-driven、constraint-driven、doubt-driven）变成强制 workflow
  - 效果：每次 build 都自动走 plan→test→verify→commit，而不是一把梭

- [ ] **场景 B：团队要统一 AI 编程标准**
  - 目标用户：tech lead / 工程效能团队
  - 痛点：每个人用不同的 prompt，代码质量参差
  - 介入方式：9 个 slash command（/spec /plan /build /test /review /ship /constraints /webperf /code-simplify）作为团队入口
  - 效果：全团队走同一套 lifecycle，code review 标准统一

- [ ] **场景 C：agent 会给自己找借口跳步骤**
  - 目标用户：资深工程师
  - 痛点：agent 说"我先不写测试了，后面补"、"这个改动小不用 review"
  - 介入方式：每个 skill 带 **anti-rationalization table**（借口 + 反驳），如 "I'll add tests later" → 反驳
  - 效果：agent 被自己的规则卡住，无法自欺

- [ ] **场景 D：跨多个 agent 工具用同一套 skill**
  - 目标用户：同时用 Claude Code + Cursor + Codex + Gemini CLI 的人
  - 痛点：每个工具有自己的 skill 格式
  - 介入方式：仓库根 `skills/` 是 portable core，`.claude/ .gemini/ .codex-plugin/ .cursor/` 等 adapter 目录各自包装
  - 效果：改一次 skill，所有工具同步生效

---

## 2. 组成结构与技术组件

### 2.1 目录结构（来自 `git/trees/main`）

```
skills/                 # 25 个 portable SKILL.md（核心）
  using-agent-skills/   # meta-skill：路由到正确 skill
  interview-me/         # 一次一问，问到 95% 置信
  idea-refine/          # 发散/收敛
  spec-driven-development/
  constraint-driven-development/
  planning-and-task-breakdown/
  incremental-implementation/
  test-driven-development/
  context-engineering/
  source-driven-development/
  doubt-driven-development/   # CLAIM→EXTRACT→DOUBT→RECONCILE→STOP
  frontend-ui-engineering/
  api-and-interface-design/
  browser-testing-with-devtools/
  debugging-and-error-recovery/
  code-review-and-quality/
  code-simplification/
  security-and-hardening/
  performance-optimization/
  git-workflow-and-versioning/
  ci-cd-and-automation/
  deprecation-and-migration/
  documentation-and-adrs/
  observability-and-instrumentation/
  shipping-and-launch/
agents/                 # 4 个 persona：code-reviewer / test-engineer / security-auditor / web-performance-auditor
references/             # 7 个 checklist：definition-of-done / testing-patterns / security / perf / a11y / observability / orchestration-patterns
commands/ + .claude/commands/ + .gemini/commands/   # 9 个 slash command 三平台包装
.codex-plugin/ .agents/plugins/   # Codex marketplace 注册
hooks/                  # sdd-cache-pre/post/test + session-start + simplify-ignore
evals/                  # 25 个 eval case + fixtures（真实代码片段）
docs/                   # 每个 host 的 setup 指南 + adoption-guide + comparison
plugin.json             # 根 plugin manifest
```

### 2.2 技术栈

- **纯 Markdown 工具集**：skill 是 `SKILL.md` + frontmatter（name/description），无运行时代码
- **多 host 分发**：`.claude/commands/*.md`、`.gemini/commands/*.toml`、`commands/*.toml`（Antigravity/Codex）、`.codex-plugin/plugin.json`
- **hooks**：bash 脚本 `hooks/sdd-cache-*.sh` / `hooks/session-start.sh`
- **eval 体系**：`evals/cases/*.json` 25 个 case + `evals/fixtures/` 真实代码 fixture（webhook.js / pagination.js / Button.tsx 等）
- **分发**：`npx skills add addyosmani/agent-skills`（通过 vercel-labs/skills CLI 装到 70+ agent）

### 2.3 核心数据流（lifecycle）

```
/spec    → PRD（spec-driven-development）
/plan    → 任务分解（planning-and-task-breakdown）
/build   → 逐片实现（incremental-implementation + TDD）
           └ 自动触发：api-design / frontend-ui / context-engineering
/test    → 证明（browser-testing + debugging）
/review  → 五轴 review（code-review-and-quality）+ 可选 persona
/webperf → Core Web Vitals 审计
/constraints → 写 CONSTRAINTS.md 定质量门
/code-simplify → Chesterton's Fence / Rule of 500
/ship    → 发布检查 + feature flag + rollback
```

---

## 3. 前 10 结构性优越点

### 优越点 1：skill anatomy 标准化
- 结构依据：README "Every skill follows a consistent anatomy: Overview / When to Use / Process / Rationalizations / Red Flags / Verification"
- 为什么优越：每个 skill 都是同一套模板，agent 学习成本固定；人读起来也能预测结构
- 对比维度：很多 skill 包每个文件结构各异，agent 不知道去哪找 verification 段

### 优越点 2：anti-rationalization 表是独有设计
- 结构依据：README "Anti-rationalization. Every skill includes a table of common excuses agents use to skip steps (e.g., 'I'll add tests later') with documented counter-arguments."
- 为什么优越：直接针对 LLM 的"自我合理化"失败模式，比单纯说"记得写测试"强一个量级
- 对比维度：Superpowers 等同类项目没有这个设计

### 优越点 3：progressive disclosure 控 token
- 结构依据：README "The SKILL.md is the entry point. Supporting references load only when needed"
- 为什么优越：主文件短，reference checklist 按需加载，context 不爆炸
- 对比维度：很多 skill 把所有 checklist 塞在 SKILL.md 里，每次都烧 token

### 优越点 4：4 个 persona 与 skill 正交
- 结构依据：`agents/code-reviewer.md` / `test-engineer.md` / `security-auditor.md` / `web-performance-auditor.md`
- 为什么优越：persona 是视角（"staff engineer 会不会批"），skill 是流程；两者可组合
- 对比维度：很多项目把 persona 和流程揉在一起，无法单独用 security 视角审任意 PR

### 优越点 5：evals 体系是少见的工程实践
- 结构依据：`evals/cases/` 25 个 JSON case + `evals/fixtures/` 真实代码（webhook.js + test.js、Button.tsx + design-system.md）
- 为什么优越：skill 本身可被回归测试，改 prompt 不会静默退化
- 对比维度：99% 的 skill 包没有 eval，全靠人肉感觉

### 优越点 6：Google 工程文化直接编码
- 结构依据：README "Concepts from Software Engineering at Google... Hyrum's Law in API design, Beyonce Rule and test pyramid in testing, Chesterton's Fence in simplification, trunk-based in git workflow, Shift Left in CI/CD"
- 为什么优越：不是泛泛"写测试"，而是把具体原则（test pyramid 80/15/5、change sizing ~100 行）写进步骤
- 对比维度：多数 skill 包是博主个人经验，缺业界共识背书

### 优越点 7：9 个 slash command 映射 lifecycle
- 结构依据：README 表 "9 slash commands that map to the development lifecycle" + `/build auto` 一键全自动
- 为什么优越：用户不用记 25 个 skill 名，9 个命令对应开发阶段
- 对比维度：ECC 有 94 个 command shim，认知负担重

### 优越点 8：多 host adapter 而不是软链
- 结构依据：`.claude/commands/`（Markdown）、`.gemini/commands/`（TOML）、`commands/`（TOML）、`.codex-plugin/`（plugin.json）各自维护
- 为什么优越：每个 host 有自己的 command schema，不能简单复制；ECC 思路类似但这个项目更轻
- 对比维度：很多 skill 包只支持 Claude Code，其他 host 只能手抄

### 优越点 9：`/build auto` 自动模式
- 结构依据：README "Approves the plan once, then it runs autonomously... every task is still test-driven and committed individually, pauses on failures"
- 为什么优越：去掉人在任务间的干预，但保留验证和暂停点
- 对比维度：很多 auto-mode 是一把梭，出错不暂停

### 优越点 10：诚实的 comparison 文档
- 结构依据：README 链 `docs/comparison.md` 与 Superpowers / Matt Pocock's skills 对比，还附 LinkedIn head-to-head 实验
- 为什么优越：不吹自己最好，承认场景差异
- 对比维度：多数项目 README 自吹自擂

---

## 4. 前 10 优化增强点

### 优化点 1：单 skill 安装缺 references 问题未根治
- 当前状态：README 自承 "Installing one skill? A per-skill npx install copies only skills/<name>/, not the repo-level references/"，并 track issue #361
- 优化方向：每个 skill 内部引用 references 时用相对路径复制，或 npx 安装时自动带所需 reference
- 预期收益：单点安装的 skill 不再缺 checklist

### 优化点 2：25 个 skill 缺优先级/入门路径
- 当前状态：README 列了全部 25 个，但新人不知道先学哪 5 个
- 优化方向：加 "Start with these 3: spec-driven-development / test-driven-development / code-review-and-quality"
- 预期收益：降低认知负担

### 优化点 3：evals 没接 CI
- 当前状态：`evals/cases/` 有 25 个 case，但 `.github/workflows/` 只有 `test-plugin-install.yml`
- 优化方向：加一个 workflow，改 SKILL.md 时自动跑 evals 对比基线
- 预期收益：防止 skill 演化退化

### 优化点 4：hooks 是 bash，Windows 不友好
- 当前状态：`hooks/sdd-cache-pre.sh` 等是 bash
- 优化方向：加 `.ps1` 版本或用 Node 重写
- 预期收益：Windows 用户不用装 WSL

### 优化点 5：缺少对国产模型的说明
- 当前状态：README 默认 Claude/GPT 语境，没提 Kimi/GLM/Qwen 上的表现
- 优化方向：加 "Tested on which models" 段，标注哪些 skill 在弱模型上效果差
- 预期收益：中文用户能判断是否值得装

### 优化点 6：persona 只有 4 个
- 当前状态：code-reviewer / test-engineer / security-auditor / web-performance-auditor
- 优化方向：加 docs-writer / data-modeler / devops-engineer persona
- 预期收益：覆盖更多评审场景

### 优化点 7：reference checklist 没版本化
- 当前状态：`references/` 7 个 checklist 是静态 md
- 优化方向：加 `references/CHANGELOG.md`，每次更新 checklist 记录原因
- 预期收益：用户能判断是否需要升级

### 优化点 8：docs/ 有 15+ 个 setup 指南但缺索引
- 当前状态：`docs/cursor-setup.md` / `codex-setup.md` / `gemini-cli-setup.md` 等分散
- 优化方向：加 `docs/README.md` 索引表，按 host 列成熟度
- 预期收益：新用户 10 秒找到自己的 host

### 优化点 9：缺"反例库"
- 当前状态：anti-rationalization 表在每个 skill 内部，没汇总
- 优化方向：加 `references/anti-patterns-catalog.md` 跨 skill 汇总 agent 常见借口
- 预期收益：团队可以基于这个做内部培训

### 优化点 10：license 只 MIT，无商业支持
- 当前状态：README 无 sponsorship / 商业支持链接
- 优化方向：加 GitHub Sponsors 或 consulting 页面
- 预期收益：可持续维护（对比 ECC 有明确 Pro 商业层）

---

## 5. ProcessOn 全景图信息

- 文件夹名称：agent-skills
- 图表标题：addyosmani/agent-skills 结构性全景图
- 图表链接：https://www.processon.com/view/link/6ab0d1cf7783ce2a62bf6d30
- 图中应包含：6 阶段 lifecycle（define/plan/build/verify/review/ship）+ 25 skill 分布、4 persona、7 reference checklist、多 host adapter 层、evals 体系

## 6. 幕布文档信息

- 文档名称：agent-skills — 日榜研究
- 文档 ID：6QFPgi-EJ2c
- 包含内容：场景问题 + 前10优越点 + 前10优化点 + ProcessOn 链接

---

## 2 分钟动作

- [ ] 先装 3 个核心：`npx skills add addyosmani/agent-skills --skill spec-driven-development && npx skills add addyosmani/agent-skills --skill test-driven-development && npx skills add addyosmani/agent-skills --skill code-review-and-quality`
- [ ] 打开 `evals/cases/security-and-hardening.json` 看一个 case 长什么样
- [ ] 读 `references/definition-of-done.md`，这是最值得抄到你团队 README 的一份
