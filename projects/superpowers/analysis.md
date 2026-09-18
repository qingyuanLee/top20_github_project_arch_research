# obra/superpowers — 架构研究分析

> 抓取时间：2026-09-18 | Stars：288183 | 排名：#12 | 主语言：Shell
> 项目类别：B类（代码架构）
> 仓库地址：https://github.com/obra/superpowers
> ⚠️ star数疑似异常：仓库创建于 2025-10-09，截至抓取仅约 11 个月即达 288k★，增速远超任何成熟开源项目。该项目在 2025 H2「Agent Skills」生态爆发期借 Claude Code/Codex 等官方插件市场病毒式分发而走红，star 数可能含平台推广与批量关注成分。本分析以架构机制为主，不以 star 数作为质量证明。

## 1. 场景问题：该项目主要解决什么问题

### 场景一：编码 Agent 「拿到需求就乱写」的失控问题
- **场景名称**：把编码 Agent 从「冲动型代码生成器」调教成「有工程纪律的协作者」。
- **目标用户**：日常使用 Claude Code / Codex / Cursor 等编码 Agent 的开发者。
- **解决的痛点**：裸 Agent 收到「加个功能」就立刻跳进写代码，跳过需求澄清、不写测试、不做设计评审，产出一堆要返工的代码；用户只能靠反复 prompt 提醒，且每次新会话都要重新教。
- **典型使用方式**：装好 Superpowers 后，SessionStart hook 自动注入 `using-superpowers` bootstrap；当 Agent 识别到「要构建东西」，自动触发 `brainstorming` skill，先用对话把 spec 问清楚、分段给用户确认，获得批准才允许进入实现（README「How it works」「The Basic Workflow」）。

### 场景二：多套编码 Agent 工具链无法复用同一套方法论
- **场景名称**：跨 harness 的技能与方法论一次编写、处处运行。
- **目标用户**：同时使用 Claude Code、Codex、Cursor、Gemini CLI、Copilot CLI 等多个编码 Agent 的团队/个人。
- **解决的痛点**：每家 Agent 工具有自己的插件机制，团队好不容易沉淀的「TDD + 计划 + 子 Agent 审查」流程无法跨工具复用，每换一个 harness 就要重配。
- **典型使用方式**：仓库根目录同时维护 `.claude-plugin/`、`.codex-plugin/`、`.cursor-plugin/`、`.devin-plugin/`、`.hermes-plugin/`、`.kimi-plugin/`、`.opencode/`、`.pi/` 等十余个 harness 适配目录，核心 `skills/` 与 `hooks/` 只有一份；README 给出每个 harness 的一行安装命令（Claude Code、Codex、Cursor、Gemini、Copilot、Grok、Kimi、OpenCode、Pi、Hermes、Devin、Factory Droid、Antigravity 共 13+ 种）。

### 场景三：长任务无人值守时 Agent 偏离计划
- **场景名称**：用子 Agent 流水线实现「可 autonomous 跑几小时不跑偏」。
- **目标用户**：希望 Agent 并行处理一组工程任务、自己只在关键节点确认的工程师。
- **解决的痛点**：让 Agent 一口气写 10 个文件，它常在中途「自由发挥」偏离原始意图，且没有质量门禁。
- **典型使用方式**：`writing-plans` 先把工作拆成 2–5 分钟一个、带精确文件路径与验证步骤的小任务；`subagent-driven-development` 为每个任务派一个全新子 Agent，并做两段式审查（先查 spec 符合性，再查代码质量）；关键问题按严重度阻断，实现「一次性 autonomous 工作数小时不偏离计划」（README「How it works」第 4 步）。

## 2. 组成结构与技术组件

### 2.1 目录/内容结构

根目录（实测 `contents/`）：
- `skills/`：核心技能库，14 个技能目录，每个内含一个 `SKILL.md`。
  - 测试：`test-driven-development`
  - 调试：`systematic-debugging`、`verification-before-completion`
  - 协作：`brainstorming`、`writing-plans`、`executing-plans`、`dispatching-parallel-agents`、`requesting-code-review`、`receiving-code-review`、`using-git-worktrees`、`finishing-a-development-branch`、`subagent-driven-development`
  - 元技能：`writing-skills`（如何写新技能）、`using-superpowers`（技能系统总引导/bootstrap）
- `hooks/`：运行时引导机制——`hooks.json`（SessionStart 钩子声明）、`hooks-cursor.json`（Cursor 版）、`run-hook.cmd`、`session-start/`（会话启动脚本）。
- 多 harness 适配目录：`.claude-plugin/`（含 `plugin.json` + `marketplace.json`）、`.codex-plugin/`、`.cursor-plugin/`、`.devin-plugin/`、`.hermes-plugin/`、`.kimi-plugin/`、`.opencode/`、`.pi/`、`.agents/`。
- `docs/`：分 harness 的安装文档（如 `README.kimi.md`、`README.opencode.md`）。
- `scripts/`、`tests/`：插件基础设施测试（`run-*.sh` / `npm test`）。
- 多 Agent 入口文件：`AGENTS.md`、`CLAUDE.md`、`GEMINI.md`、`gemini-extension.json`——同一套指令面向不同 Agent 运行时。
- 工程文件：`package.json`、`.pre-commit-config.yaml`、`.version-bump.json`、`RELEASE-NOTES.md`。

**SKILL.md 定义格式**（实测 `skills/brainstorming/SKILL.md`）：
- YAML frontmatter：`name` + `description`，description 用强触发措辞（"You MUST use this before any creative work..."）——这是 Agent 自动检索/触发技能的关键信号。
- Markdown 正文：含 `<HARD-GATE>` 显式硬门控（"Do NOT invoke any implementation skill... until approved"）、按任务规模分三条路径（Spike / Bounded / Full spec）、把「仪式感随任务伸缩、审批门永不伸缩」写成规则。

### 2.2 技术栈/工程化清单

- **运行时形态**：无传统后端。主语言 Shell（`hooks/run-hook.cmd`、`session-start`、各 `run-*.sh`），配少量 Node/TypeScript（`package.json` `main: .opencode/plugins/superpowers.js`；`.pi/extensions/superpowers.ts`）。本质是「技能 + 钩子 + 多 harness 清单」的分发包。
- **版本**：`package.json` / `.claude-plugin/plugin.json` 均锁 `version: 6.3.0`，author Jesse Vincent，MIT。
- **引导机制**：`hooks/hooks.json` 声明 `SessionStart` 钩子，matcher=`startup|clear|compact`，调用 `${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd session-start`，同步执行（`async:false`）。即会话开始、清空、压缩（compaction）后都会重新注入 bootstrap——解决长会话压缩后技能「失忆」问题。
- **质量门禁**：`.pre-commit-config.yaml` 本地提交前检查；`tests/` 跑插件基础设施测试；技能行为测试用独立的 `superpowers-evals`（drill eval harness，克隆到 `evals/`）。
- **分发渠道**：Anthropic 官方 Claude 插件市场、OpenAI Codex 官方市场、xAI Grok 官方市场，以及自建 `obra/superpowers-marketplace`。
- **遥测**：brainstorming 的可选 visual companion 从官网加载 logo，URL 带版本号以粗略统计使用量，可用 `SUPERPOWERS_DISABLE_TELEMETRY` 关闭。

### 2.3 核心数据流/协作流

这是一条「会话启动 → 技能自动触发 → 门控式工作流 → 子 Agent 执行与审查」的闭环：
1. **会话启动**：用户打开任一编码 Agent → `SessionStart` 钩子运行 `session-start` → 注入 `using-superpowers` bootstrap（告诉 Agent「任何任务前先检查相关技能」）。
2. **需求澄清门**：用户说要做东西 → `brainstorming` 自动触发 → 分类任务规模（Spike/Bounded/Full）→ Socratic 提问把 spec 分段呈现 → 用户批准（`<HARD-GATE>` 阻断实现）。
3. **隔离环境**：`using-git-worktrees` 建隔离分支与干净测试基线。
4. **计划拆解**：`writing-plans` 把工作拆成 2–5 分钟、带文件路径和验证步骤的任务。
5. **执行+审查**：`subagent-driven-development` 逐任务派新子 Agent，两段式 review（spec 符合 → 代码质量）；`test-driven-development` 强制 RED-GREEN-REFACTOR；`requesting-code-review` 按严重度阻断关键问题。
6. **收尾**：`finishing-a-development-branch` 验证测试、给 merge/PR/丢弃选项、清理 worktree。
7. **防失忆**：会话一旦 compaction，`SessionStart` 钩子再次触发，重新注入 bootstrap。

## 3. 前 10 结构性优越点（为什么它能突出）

### 优越点 1：用「技能 + 强制门控」把工程纪律写成可执行的 Agent 约束
- 结构依据：每个 `SKILL.md` 用 frontmatter 的强触发 description + 正文 `<HARD-GATE>` 块（brainstorming 原文："Do NOT invoke any implementation skill... until approved"）。
- 为什么优越：把「先设计后编码、先测试后实现」这类团队规范，从「靠人提醒的 prompt 工程」变成「Agent 启动时必检的硬门」，纪律不再随会话丢失。
- 对比维度：多数 Agent 提示词只是「建议式」软约束；Superpowers 用 HARD-GATE 把它做成阻断式流程。

### 优越点 2：一次编写、13+ harness 复用的适配层架构
- 结构依据：根目录并存 `.claude-plugin/`、`.codex-plugin/`、`.cursor-plugin/`、`.devin-plugin/`、`.hermes-plugin/`、`.kimi-plugin/`、`.opencode/`、`.pi/` 等适配目录，核心 `skills/` 单一来源。
- 为什么优越：Agent 编码工具市场高度碎片化，方法论与具体 harness 的插件机制解耦，新增一个 harness 只需加一份清单/启动脚本，核心技能零改动。
- 对比维度：同类技能包通常只绑定一家 Agent；Superpowers 用「核心 + 适配壳」把分发面扩到整个生态。

### 优越点 3：SessionStart 钩子解决「长会话压缩后失忆」
- 结构依据：`hooks/hooks.json` 的 SessionStart matcher=`startup|clear|compact`，同步运行 `run-hook.cmd session-start`。
- 为什么优越：Agent 上下文压缩（compaction）后常把 bootstrap/规则忘掉；把注入挂到 `compact` 事件，等于在每次记忆截断后自动重新「开机自检」。README 也坦承 Hermes 因无 post-compaction hook 会丢 bootstrap，说明作者精准识别了这一关节点。
- 对比维度：多数插件只在启动时注入一次，长会话后规则失效；Superpowers 把注入点覆盖到压缩事件。

### 优越点 4：任务规模自适应的「三路径」设计，避免过度仪式化
- 结构依据：brainstorming SKILL.md 明确分 Spike / Bounded / Full 三档——Spike 只要答案不留代码、Bounded 在对话里给短设计、Full 才写 spec 文件。
- 为什么优越：一刀切走完整 spec 流程会让改个标点也要写设计文档；按任务规模伸缩仪式感（但保留审批门），既保纪律又不拖慢日常小改。
- 对比维度：教条式方法论要么全流程太重、要么完全放任；Superpowers 用分级路径平衡了严谨与效率。

### 优越点 5：子 Agent 两段式审查流水线，把质量门禁分层
- 结构依据：`subagent-driven-development` + `requesting-code-review`：每任务派新子 Agent，先审 spec 符合性、再审代码质量，关键问题按严重度阻断（README 第 4、6 步）。
- 为什么优越：单 Agent 自评有「自己写的自己看不出问题」的盲点；用全新上下文的子 Agent 做规格符合性审查，再做代码质量审查，相当于把 code review 内建进执行循环。
- 对比维度：多数 Agent 编码是「一气呵成写到底」；Superpowers 用多 Agent 串行审查逼近人工 PR review。

### 优越点 6：把 TDD 做成不可绕过的循环，而非最佳实践口号
- 结构依据：`test-driven-development` 技能强制 RED-GREEN-REFACTOR，并要求「删掉先于测试写出的代码」（README 第 5 步、Philosophy 第一条）。
- 为什么优越：Agent 天然倾向先写实现再补测试；把「先看测试失败、再写最小实现、再看通过」写成步骤约束，从根上抑制 Agent 跳过测试的冲动。
- 对比维度：其他提示词只说「请写测试」；Superpowers 把它拆成可执行的红绿重构闭环并配反模式参考。

### 优越点 7：用「元技能」实现自举与可演化
- 结构依据：`writing-skills` 技能教 Agent 如何按最佳实践创建新技能（含测试方法），`using-superpowers` 作为总引导；Contributing 明确要求改技能必须跨所有支持的 harness 可用。
- 为什么优越：技能库不是死文件，而是「教 Agent 自己长新技能」的系统，且用跨 harness 兼容性作为合并门槛，保证生态自演化时不退化。
- 对比维度：封闭技能包只能等作者更新；Superpowers 把「如何写技能」本身做成技能，具备元层扩展能力。

### 优越点 8：与 Git worktree 深度绑定，实现并行隔离
- 结构依据：`using-git-worktrees` 在建完设计后自动建隔离工作区、跑项目 setup、确认干净测试基线；`finishing-a-development-branch` 负责合并/PR/清理。
- 为什么优越：Agent 在主工作区直接改文件易污染、易和用户工作冲突；worktree 隔离让多个任务/多个并行子 Agent 互不踩踏，结束时统一决策收尾。
- 对比维度：裸 Agent 常直接在当前目录乱写；Superpowers 把每次任务关进独立 worktree。

### 优越点 9：显式的遥测 opt-out 与隐私克制
- 结构依据：README「Visual companion telemetry」说明只在可选功能里加载带版本号的 logo，不收集项目/prompt，支持 `SUPERPOWERS_DISABLE_TELEMETRY` 及 Claude Code 的 opt-out。
- 为什么优越：编码工具读取用户代码上下文，遥测极易越界；作者主动把遥测限定为「粗略版本统计」并尊重宿主 Agent 的隐私开关，降低企业采用阻力。
- 对比维度：不少 AI 插件默认全量上报；Superpowers 在方法论之外也做了隐私设计。

### 优越点 10：商业可持续但核心开源的双轨
- 结构依据：核心技能 MIT 开源（LICENSE / plugin.json），同时 README 设「Commercial Services」指向 sales@primeradiant.com，为企业提供支持/管理工具。
- 为什么优越：纯开源难以为长周期维护付费，纯商业又会锁死生态；把方法论内核免费开源换增长、企业服务变现换可持续，是开发者工具成熟的双轨范式。
- 对比维度：要么完全社区驱动、要么纯 SaaS 闭源；Superpowers 用「开源内核 + 商业支持」兼顾。

## 4. 前 10 优化增强点（主要关节点可优化方向）

### 优化点 1：star 数异常暴涨带来的信噪比风险
- 当前状态：约 11 个月冲到 288k★（本项目元信息已标注疑似异常），社区反馈、issue 与讨论量未必与 star 数匹配，新用户易被 star 数误导。
- 优化方向：在 README 顶部用「实际活跃用户/插件安装量」而非 GitHub star 作为主可信度指标；用 evals 通过率、真实工作样本替代 star。
- 预期收益：把项目可信度锚定在可验证的工程效果上，避免 star 泡沫影响新用户判断。

### 优化点 2：跨 harness 一致性靠人工纪律，缺自动回归
- 当前状态：Contributing 要求技能改动必须在所有 harness 可用，但这靠人工保证；README 已暴露 Hermes 因无 post-compaction hook 会丢 bootstrap。
- 优化方向：把「每个 harness 的 bootstrap 是否生效」做成 `tests/` 下的矩阵自动化测试，CI 里逐 harness 跑 session-start 注入 smoke test。
- 预期收益：把「跨 harness 兼容」从口头承诺变成可回归保证，减少某 harness 静默失效。

### 优化点 3：技能触发依赖自然语言 description，存在误触发/漏触发
- 当前状态：技能靠 frontmatter 的自然语言 description 让 Agent 自行决定何时调用，没有结构化的触发条件 schema。
- 优化方向：为每个技能补充结构化触发条件（任务类型标签、前置/后置条件），与自然语言 description 双轨检索，降低 Agent 误判。
- 预期收益：减少「该用没用、不该用乱用」，提高技能自动触发的精确率。

### 优化点 4：缺少按项目/团队裁剪技能集的机制
- 当前状态：装上即全套 14 个技能，团队若只想用 TDD + 计划、不需要子 Agent 流水线，无法轻易关闭。
- 优化方向：提供技能 profile（如「极简 TDD 版」「全流程团队版」）与开关配置，让团队按成熟度裁剪。
- 预期收益：降低初次采用门槛，让小团队不必背负过重流程，也便于渐进式引入。

### 优化点 5：eval 体系独立于主仓，新贡献者上手成本高
- 当前状态：技能行为测试要另 clone `superpowers-evals` 到 `evals/`，Contributing 第 4 步才提到。
- 优化方向：把 eval harness 纳入主仓或一键脚本 `make setup-evals`，并在 PR 模板里强制勾选「已跑技能行为 eval」。
- 预期收益：降低贡献者跑测试的门槛，提升 PR 质量与合并速度。

### 优化点 6：HARD-GATE 的粒度未按任务类型做差异化提示
- 当前状态：审批门「永不伸缩」，但 Spike 类可行性问题也要求走完分类+对话，对纯探索略重。
- 优化方向：把审批门做成「按风险分级」——改依赖/删代码为硬门，纯读代码/Spike 为软提示，在 SKILL.md 里显式列出分级矩阵。
- 预期收益：在保住关键安全门的同时，减少探索型任务的仪式开销。

### 优化点 7：子 Agent 审查成本与 token 消耗未透明化
- 当前状态：subagent-driven-development 为每个任务派新子 Agent 并两段审查，长任务 token 消耗可观，用户无法预估。
- 优化方向：在 executing-plans 里加入「预计 token/任务数」预估与预算上限，超预算自动转为批量执行模式。
- 预期收益：控制成本可预期，避免用户在长 autonomous 任务中收到意外高额账单。

### 优化点 8：遥测仅统计版本，无法反哺技能质量
- 当前状态：telemetry 只粗粒度统计用了哪个版本，不知道哪个技能真的提升了产出质量。
- 优化方向：在用户 opt-in 前提下，匿名收集「技能触发后任务是否一次通过 review」等聚合指标，用于排定技能改进优先级。
- 预期收益：用真实效果数据指导技能迭代，而不是凭作者主观判断。

### 优化点 9：缺少面向新手的「5 分钟最小可用示例」
- 当前状态：README 从 13 种 harness 安装命令开始，信息量大；新用户不知道第一次该期待什么。
- 优化方向：在 How it works 之前加一个端到端 walkthrough（一句话需求 → brainstorming 提问 → spec → TDD 实现）的录屏/图文样例。
- 预期收益：降低认知负荷，让新用户 5 分钟内跑通第一个完整工作流。

### 优化点 10：多份入口文件（AGENTS.md/CLAUDE.md/GEMINI.md）存在漂移风险
- 当前状态：根目录为不同 Agent 运行时各维护一份顶层指令文件，内容可能随时间不一致。
- 优化方向：用单一源 + 构建脚本在发布时生成各运行时入口文件，纳入 `.version-bump.json` 的版本流程。
- 预期收益：消除多份指令文件的漂移，保证 Claude/Gemini/通用 Agent 看到的方法论一致。

## 5. ProcessOn 全景图信息

- 文件夹名称：superpowers
- 图表标题：obra/superpowers 结构性全景图
- 图表链接：https://www.processon.com/view/link/6aacb0716a29601cdfbeb39e
- 图中应包含：顶层（项目定位：编码 Agent 的工程方法论框架）→ 中层（14 个技能：brainstorming/plans/TDD/subagent review 等）→ 底层（hooks 引导 + 13 harness 适配壳 + evals/pre-commit 测试基础设施）→ 连线标注（SessionStart→bootstrap→技能触发→子 Agent 两段审查）。

## 6. 幕布文档信息

- 文档名称：superpowers — 架构研究
- 文档 ID：63v4VRyXNbc
- 包含内容：场景问题 + 前10优越点 + 前10优化点 + ProcessOn 链接
