# 下一步：`npx ecc-universal@2.2.2 setup` 跑 guided wizard，或在 Claude Code 里 `/plugin marketplace add https://github.com/affaan-m/ECC` + `/plugin install ecc@ecc`。

> 快照日期：2026-09-21 | 来源：GitHub Trending daily（since=daily）| 当日仅 13 个项目（周一 UTC 午夜刚重置），非完整 20 个
> 抓取时间：2026-09-21 | Stars：264,018 ⚠️ **数值异常标注**：该 star 数在 Trending daily 上下文中明显偏高（前 20 项目中罕见 264k 量级），与当日"826 今日新增"不成比例，疑似 star 历史累积或 API 返回口径异常，本分析不以此作为质量背书
> 当日新增：826 | 主语言：JavaScript（仓库实际含 Bash/TS/Python/Go/Java/Perl）
> 项目类别：B 类（代码架构 / agent harness 操作系统）
> 仓库地址：https://github.com/affaan-m/ECC

**TL;DR：** ECC（Everything Claude Code，现已泛化到多 harness）是一个"agent harness 操作系统"——把 senior engineer 的工作流（plan→test→implement→review→verify→remember→improve）打包成 68 agents + 292 skills + 94 commands + hooks + rules + memory + AgentShield 安全扫描，一次安装让 Claude Code/Codex/Cursor/OpenCode/Codex/Kimi 等 7+ harness 共享同一套工程纪律。

---

## 1. 场景问题：该项目主要解决什么问题

- [ ] **场景 A：AI coding agent 总是跳步骤**
  - 目标用户：每天用 Claude Code / Codex 的工程师
  - 痛点：agent 默认走最短路径——跳过 spec、不写测试、review 自己代码不换上下文
  - 介入方式：ECC 把 plan/test/review/verify 流程做成 skill 自动触发，hook 强制在关键节点停下来
  - 效果：`plan -> test -> implement -> review -> verify -> remember -> improve` 成为默认循环

- [ ] **场景 B：跨多个 coding agent 工具，配置不统一**
  - 目标用户：同时用 Claude Code + Codex + Cursor 的团队
  - 痛点：每个工具有自己的 rules/commands，跨工具配置漂移
  - 介入方式：ECC 一个仓库分发到 Claude plugin / Codex marketplace / Cursor rules / OpenCode / Gemini / Zed / Copilot / Antigravity / Kimi 等
  - 效果：装一次，多 harness 共享同一套工程标准

- [ ] **场景 C：prompt 工程知识无法沉淀**
  - 目标用户：资深工程师 / tech lead
  - 痛点：自己摸索的好 prompt 散落在聊天记录里，换个项目就丢
  - 介入方式：ECC 把反复验证的工作流做成 skill 包，含 references/ 和 hooks/
  - 效果：团队的工程判断变成可版本化、可分发的资产

- [ ] **场景 D：第三方 MCP / hooks 有供应链风险**
  - 目标用户：在生产环境用 agent 的团队
  - 痛点：装上的 skill 可能偷 secrets、跑恶意 hook
  - 介入方式：内置 AgentShield 扫描 prompts/hooks/MCP config/permissions/secrets/agent files
  - 效果：安装前自动扫一遍，把恶意插件挡在门外

---

## 2. 组成结构与技术组件

### 2.1 目录结构（来自 `git/trees/main` 与 `contents/`）

```
.agents/skills/         # 40+ 跨 harness 通用 skill（每个含 SKILL.md + agents/openai.yaml）
.claude/                # Claude Code 专属：commands/rules/homunculus/research/team/workflows
.claude-plugin/         # Claude marketplace plugin.json + marketplace.json
.codex/ + .codex-plugin/# Codex CLI 的 AGENTS.md、agents/*.toml、plugin.json
.cursor/                # Cursor 的 hooks/ + rules/（含 golang 专用）
.gemini/                # Gemini CLI commands
.opencode/ .zed/ .qwen/ .trae/ .kiro/ .hermes/ .openclaw/  # 其他 harness 适配
.agents/plugins/marketplace.json

agents/                 # 专用 sub-agent 定义
commands/               # 94 个 legacy command shim
hooks/                  # 运行时 hook（session-start / pre-compact / subagent-stop 等）
rules/                  # common / typescript / golang 等语言规则包
schemas/                # 各种 YAML/JSON schema
scripts/                # ecc.js / sync-to-codex / install 等
skills/                 # 主 skill 集
mcp-configs/            # MCP 服务器配置
workflows/              # 工作流编排
ecc2/                   # 2.x 新版代码
docker/ examples/ tests/ docs/
```

### 2.2 技术栈

- **分发**：npm 包 `ecc-universal` + `ecc-agentshield`（Node 18+），GitHub App `ecc-tools`，Claude plugin slug `ecc@ecc`
- **多语言实现**：Bash 安装脚本、TypeScript 主逻辑、Python 工具、Go 二进制、Java/Perl 历史脚本
- **hook 体系**：`.cursor/hooks/` 下 15+ 个 adapter.js / after-*.js / before-*.js，覆盖 MCP 执行、shell 执行、文件编辑、tab 读取、session start/end
- **商业层**：ECC Pro（托管 GitHub App，private repo $19/seat/mo），OSS 永远 MIT

### 2.3 核心数据流

```
用户: npx ecc-universal@2.2.2 setup
  → wizard 扫描 marketplace + 所有 native install scope
  → 按所选 harness 安装 skills/agents/commands/hooks/rules
  → 在 harness 内触发 /ecc:configure-ecc
Agent 工作时:
  session-start hook → 加载 rules + memory
  → skill 自动触发（按 description）
  → sub-agent 隔离跑 plan/review/build-repair/security
  → before-shell-execution hook 检查危险命令
  → after-file-edit hook 触发 lint/test
  → session-end hook 做 continuous learning 摘要
  → AgentShield 扫描新增 prompt/hook/MCP
```

---

## 3. 前 10 结构性优越点

### 优越点 1：一次安装覆盖 7+ harness
- 结构依据：README 表 "Claude Code / Codex / Kimi Code" 三列 + `.cursor/ .gemini/ .opencode/ .zed/ .qwen/ .trae/ .kiro/ .hermes/ .openclaw/` 等适配目录
- 为什么优越：每个 harness 都有自己的 plugin 机制，ECC 为每个写原生 adapter，不是简单软链
- 对比维度：多数 skill 包只支持 Claude Code，换 harness 就要重写

### 优越点 2：wizard 先 inventory 再动手
- 结构依据：README "The wizard inventories the official marketplace and every native Claude install scope before making changes"
- 为什么优越：避免重复安装导致 skills/commands/hooks 重复；dry-run 模式先预览
- 对比维度：很多安装脚本直接 cp -R 覆盖，出问题只能手动清

### 优越点 3：明确禁止 stack 安装方法
- 结构依据：README "Pick one path only (per harness)" + "Do not stack install methods"
- 为什么优越：Claude plugin 装完又跑 manual install 会重复 skill；ECC 在文档和脚本两层都拦
- 对比维度：同类项目的 README 经常没警告，用户踩坑后才发现冲突

### 优越点 4：AgentShield 把供应链安全做成内置件
- 结构依据：README 表 "AgentShield: Scanning for prompts, hooks, MCP config, permissions, secrets, and agent files"
- 为什么优越：agent 生态的 hook 权限极大，恶意 skill 可以读 `~/.ssh`；ECC 把扫描做成一等公民
- 对比维度：绝大多数 skill 包完全不考虑 supply-chain

### 优越点 5：hook 粒度细到每个 MCP 调用
- 结构依据：`.cursor/hooks/` 下 `before-mcp-execution.js` / `after-mcp-execution.js` / `before-shell-execution-block-no-verify.js`
- 为什么优越：能在 MCP 调用前拦截、在 shell 执行后审计，形成运行时防护网
- 对比维度：多数项目只有 session-start/session-end 两个 hook 点

### 优越点 6：规则按语言分包按需加载
- 结构依据：README "Start with rules/common plus one language or framework pack you actually use"
- 为什么优越：不是把所有规则塞进 context window，而是按项目技术栈选包
- 对比维度：很多 rules 包全量加载，context 爆炸

### 优越点 7：三标识符解耦（repo / plugin / npm）
- 结构依据：README "Naming + migration note" 段：GitHub `affaan-m/ECC` / Claude `ecc@ecc` / npm `ecc-universal`
- 为什么优越：Anthropic marketplace 对 plugin 名字有长度限制，短 slug 不影响工具命名空间；npm 单独命名不绑死 plugin
- 对比维度：很多项目一个名字到处用，一旦要改名就全线崩

### 优越点 8：install 失败有 doctor / repair 路径
- 结构依据：README "node scripts/ecc.js list-installed / doctor / repair"
- 为什么优越：用户本地 Claude 配置被重置后不用重装，doctor 诊断 + repair 恢复 ECC-managed 文件
- 对比维度：同类项目出问题只能 `rm -rf ~/.claude && 重装`

### 优越点 9：持续学习 + memory + instincts 三件套
- 结构依据：README "Hooks and memory | Runtime | Enforcement, session summaries, continuous learning, instincts, and context controls"
- 为什么优越：把"反复成功的工作流沉淀成可复用 instinct"做成运行时能力，而不是一次性 prompt
- 对比维度：多数 skill 包 stateless，每次会话从零开始

### 优越点 10：open-core 商业模型健康
- 结构依据：README "OSS stays free. This repo is MIT-licensed forever. ECC Pro is the hosted GitHub App for private repos."
- 为什么优越：公开项目免费，private repo 的 GitHub App 收费 $19/seat/mo，可持续维护
- 对比维度：很多单人项目靠赞助续命，维护不稳定

---

## 4. 前 10 优化增强点

### 优化点 1：star 数异常需官方澄清
- 当前状态：264k star 与"单人维护、周更"画像不符，可能是 star-history 徽章或 API 异常
- 优化方向：在 README 加一行 "Current real star count: X" 并在 issue 区开个讨论
- 预期收益：避免新用户因数字失真而怀疑项目真实性

### 优化点 2：README 过长（4000+ 行）
- 当前状态：英文 README 单文件 3955+ 行，装 Claude/Codex/Kimi 三段挤在一起
- 优化方向：把安装段拆到 `docs/install/` 下，README 只留 3 个命令
- 预期收益：新用户 30 秒找到入口，不用翻屏

### 优化点 3：hook 数量多但缺默认禁用开关
- 当前状态：`.cursor/hooks/` 15+ 个 hook 全装，某些用户不需要 before-read-file
- 优化方向：`--hooks minimal|standard|full` 三档 profile（README 提到了但没在主流程展示）
- 预期收益：降低性能开销，给"我只想要 skills"的用户一个轻量选项

### 优化点 4：legacy command shim 占 94 个
- 当前状态：README 明说 "ECC moves to a skills-first surface"，但 94 个 shim 还在
- 优化方向：在 README 顶部加 deprecation timeline，2.3 开始标 warning
- 预期收益：减少新用户困惑，老用户有迁移计划

### 优化点 5：多语言支持只翻译 README
- 当前状态：顶部有 13 种语言 README 链接，但 skills/rules 本身全英文
- 优化方向：先把 SKILL.md 的 description 段多语言化，让非英文用户的 agent 也能触发
- 预期收益：扩大非英文圈用户

### 优化点 6：AgentShield 扫描规则未开源
- 当前状态：README 说扫描 prompts/hooks/MCP，但没给规则文件
- 优化方向：把扫描规则作为 YAML 公开，让社区贡献检测规则
- 预期收益：透明度提升，也能被其他项目复用

### 优化点 7：缺少最小可复现 demo repo
- 当前状态：安装说明很全，但没有一个"装完 ECC 后跑这个 task"的 sample
- 优化方向：加 `examples/demo-task/` 含一个 README，演示 TDD skill 怎么用
- 预期收益：新用户装完立刻有成就感

### 优化点 8：Codex 同步路径仍标 deprecated
- 当前状态：README 明说 `sync-ecc-to-codex.sh` 是 deprecated compatibility
- 优化方向：给 legacy 用户一个明确的迁移脚本，一键切到 native plugin
- 预期收益：减少双轨维护成本

### 优化点 9：docs 结构分散
- 当前状态：`.claude/`、`.codex/`、`docs/`、`README.*.md` 多处文档
- 优化方向：统一到 `docs/harness/<name>.md`，根 README 只链索引
- 预期收益：贡献者不用在 5 个地方改文档

### 优化点 10：缺性能基准
- 当前状态：README 列了 292 skills / 68 agents，但没说装完后 context window 占用、hook 延迟
- 优化方向：加 `benchmarks/` 目录，对比装 ECC 前后 token 消耗、首 token 延迟
- 预期收益：让用户量化"装上 ECC 的代价"

---

## 5. ProcessOn 全景图信息

- 文件夹名称：ECC
- 图表标题：affaan-m/ECC 结构性全景图 v4
- 图表链接：https://www.processon.com/view/link/6ab0e95767be235e10933615 （v4：10节点/12连线/4分组，techblue，auto_layout）
- 图中应包含：7+ harness adapter 层、68 agents + 292 skills + 94 commands 三层能力、hook 运行时、AgentShield 安全扫描、npm/GitHub App 分发

## 6. 幕布文档信息

- 文档名称：ECC — 日榜研究
- 文档 ID：5H4DPRayx2c
- 包含内容：场景问题 + 前10优越点 + 前10优化点 + ProcessOn 链接

---

## 2 分钟动作

- [ ] 跑 `npx ecc-universal@2.2.2 doctor` 看你当前 harness 状态（不写文件）
- [ ] 打开 `.cursor/hooks/` 或 `.claude/` 看一眼 hook 文件，感受一下粒度
- [ ] 如果你 private repo 多，去 ecc.tools/pricing 看一眼 Pro 是不是值得 $19/seat/mo
