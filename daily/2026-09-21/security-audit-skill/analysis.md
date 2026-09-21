# 下一步：`npx skills add https://github.com/cloudflare/security-audit-skill --skill security-audit` 安装，然后在代码库里对 coding agent 说 "security audit this codebase"。

> 快照日期：2026-09-21 | 来源：GitHub Trending daily（since=daily）| 当日仅 13 个项目（周一 UTC 午夜刚重置），非完整 20 个
> 抓取时间：2026-09-21 | Stars：18,303 | 当日新增：2,428 | 主语言：JavaScript
> 项目类别：B 类（代码架构 / coding-agent skill 包）
> 仓库地址：https://github.com/cloudflare/security-audit-skill

**TL;DR：** Cloudflare 把自己内部漏洞挖掘 harness 的"单仓起点"开源成一个 coding-agent skill。它不是一个扫描器二进制，而是一套**编排规范**——6 阶段流水线（侦察→覆盖驱动狩猎→候选验证→结构化输出→独立复核→中立报告）+ 零依赖校验器 + JSON Schema。核心价值是"对抗式验证"：发现漏洞的 agent 和验证漏洞的 agent 永远不是同一个。

---

## 1. 场景问题：该项目主要解决什么问题

- [ ] **场景 A：安全团队把 LLM 接入代码审计但结果不可信**
  - 目标用户：AppSec 工程师 / 内部红队
  - 痛点：直接让 Claude/Codex "找漏洞"会产出大量幻觉、把"最佳实践缺失"当漏洞、无法复现
  - 介入方式：skill 强制 6 阶段，候选必须经独立 verifier 反驳，无法反驳才标 `confirmed`
  - 效果：报告里每条 `confirmed` 都带完整 source trace 和有界观测结果

- [ ] **场景 B：中小团队没有专职 SAST 工具链**
  - 目标用户：全栈工程师 / 初创团队
  - 痛点：商业 SAST 贵、误报多；自己写正则扫描器覆盖不到新攻击面
  - 介入方式：13 个攻击类 Markdown 提示词文件（AI/LLM、Web/Auth、客户端、供应链、云部署、RPC、资源耗尽、数据隔离、桌面 IPC 等），开箱即用
  - 效果：`npx skills add` 一条命令装完，对任意 repo 跑一次结构化审计

- [ ] **场景 C：把 LLM 审计接入 CI / 自动化漏洞发现 harness**
  - 目标用户：平台安全 / 工具链工程师
  - 痛点：Cloudflare 自己的漏洞 harness 从单仓演化成 fleet 级系统，需要一个可移植的起点
  - 介入方式：README 明说这是 "the single-repo starting point it evolved from"，产物全部是机器可读 JSON（`findings.json` / `coverage-ledger.json`）
  - 效果：下游可接自己的 fleet 编排、去重、跨仓库聚合

---

## 2. 组成结构与技术组件

### 2.1 目录结构（来自 `git/trees/main`）

```
skills/security-audit/
├── SKILL.md                      # 入口：原则、术语、工作流、反模式
├── RECONNAISSANCE.md             # 阶段1 侦察
├── HUNTING.md                    # 阶段2 编排与狩猎方法论
├── ATTACK-CLASSES.md             # 通用/wildcard/obvious 攻击提示
├── AI-AND-LLM.md                 # 提示注入、agent/工具、输出处理
├── WEB-PROTOCOL-AND-AUTH.md      # HTTP framing、cache、auth 协议
├── CLIENT-SIDE.md                # DOM 注入、消息信任、UI redress、原型污染
├── SUPPLY-CHAIN-AND-RELEASE.md   # 依赖、CI、发布、签名、插件/扩展
├── CLOUD-AND-DEPLOYMENT.md       # IAM、IaC、容器、serverless、ingress
├── PROTOCOLS-RPC-AND-MESSAGING.md# RPC、序列化、队列、broker、webhook
├── RESOURCE-EXHAUSTION-AND-AVAILABILITY.md
├── DATA-ISOLATION-AND-LIFECYCLE.md
├── DESKTOP-MOBILE-AND-LOCAL-IPC.md
├── MEMORY-SAFETY-AND-BINARY.md   # native target 的内存安全/二进制/内核
├── VALIDATION-AND-REPORTING.md   # 阶段3-6
├── report-schema.json            # findings.json 三种 verdict 的 JSON Schema
├── validate-findings.cjs         # 零依赖 findings 校验器
├── validate-findings.test.cjs
├── validate-coverage-ledger.cjs  # 零依赖 coverage-ledger 校验器
└── validate-coverage-ledger.test.cjs
```

### 2.2 技术栈

- **无运行时框架**：纯 Markdown 提示词 + Node.js 零依赖 CJS 校验脚本（`validate-*.cjs`），不引入 npm 依赖
- **分发渠道**：[Skills CLI](https://skills.sh)（`npx skills add`），兼容任何支持工具调用 + 并行子 agent 的 coding agent
- **强约束外部依赖**：要求 OS 级沙箱（禁外网、allowlist 环境、资源限额、只能写 scratch 路径）——没有沙箱就把 lead 留在 `needs_validation`，不执行目标代码

### 2.3 核心数据流（6 阶段）

```
用户: "security audit this codebase"
  → [1 Recon]        architecture.md + coverage-ledger.json（覆盖账本）
  → [2 Hunting]      从 ledger 分配独立 hunter，coverage critic 找盲区
  → [3 Validation]  每个候选交给全新 verifier 尝试证伪
  → [4 Structured]   findings.json (confirmed/needs_validation/rejected)
                     跑 validate-findings.cjs
  → [5 Record Verif] 新 agent 复核最终 source 主张；实质替换再换一个 verifier
  → [6 Report]       REPORT.md / FINDINGS-DETAIL.md / NEEDS-VALIDATION.md
多次跑是增量的：旧 ledger + findings 用来定位盲区、重验变更、不带陈旧结论。
```

---

## 3. 前 10 结构性优越点

### 优越点 1：发现者与验证者强制隔离
- 结构依据：README "Adversarial validation. The agent that checks a finding is never the agent that found it."
- 为什么优越：LLM 有自我确认偏见，同一上下文既写又验会系统性高估；隔离后 verifier 带着"证伪任务"上下文跑
- 对比维度：多数 AI 安全工具（包括早期 LLM 扫描器）是单轮生成单轮验证，误报率高一个量级

### 优越点 2：三态 verdict 强制区分置信度
- 结构依据：`report-schema.json` + README "verdicts are distinct: confirmed / needs_validation / rejected"
- 为什么优越：`needs_validation` 必须写"未解决的精确事实"且不带 severity，把"还没查清楚"和"确认有问题"从数据层就分开，下游报告不会混淆
- 对比维度：传统 SAST 只有 "warning / error" 两档，无法表达"我怀疑但跑不起来"

### 优越点 3：coverage-ledger 作为一等公民
- 结构依据：`coverage-ledger.json` + `validate-coverage-ledger.cjs` + 阶段 2 "coverage critics"
- 为什么优越：审计不是"问一圈 LLM"，而是先建"我查过哪些入口/边界"的账本，再按账本分配 hunter；未覆盖的地方显式留空
- 对比维度：随机 prompting 的 LLM 审计会重复扫同一类 bug、漏掉边缘路径

### 优越点 4：零依赖校验器把"schema 合规"从 agent 输出中剥离
- 结构依据：`validate-findings.cjs` / `validate-coverage-ledger.cjs` 是纯 Node CJS，无 npm 依赖
- 为什么优越：agent 可能忘记必填字段、写错枚举；父进程在阶段 1/4/5 反复跑校验器，失败立刻修，不依赖 agent 自觉
- 对比维度：很多 prompt-only skill 把格式约束写在提示词里，LLM 经常违反

### 优越点 5：攻击类按"信任边界"而非"语言"切分
- 结构依据：13 个攻击类文件分别覆盖 AI/LLM、Web/Auth、客户端、供应链、云、RPC、资源耗尽、数据隔离、桌面 IPC、内存安全
- 为什么优越：审计的最小单元是"信任边界 + 输入面"，按语言切分会漏掉跨层问题；按边界切分可直接映射到 coverage-ledger 的覆盖单元
- 对比维度：OWASP Top 10 是横向清单，不能直接当 hunt 任务卡

### 优越点 6：沙箱缺失时主动降级而非硬跑
- 结构依据：README Requirements 段 "Without these controls, the workflow keeps the lead as needs_validation instead of executing target code."
- 为什么优越：审计工具本身可能被目标仓库的恶意代码反制；缺沙箱时宁可不出 `confirmed` 也不执行不可信代码，是安全工具该有的安全姿态
- 对比维度：很多 AI 代码执行 agent 默认在用户主环境跑测试，存在 supply-chain 反制风险

### 优越点 7：多轮增量而非一次性
- 结构依据：README "Multiple runs against the same repo are additive... In our test runs, a single run found roughly half of the vulnerabilities that repeated runs found in total."
- 为什么优越：把"覆盖率随轮次衰减"作为产品假设写进设计——第二轮用旧 ledger 定位盲区，而不是从零重跑
- 对比维度：一次性 prompt 审计天然无法跨 run 学习

### 优越点 8：severity 必须基于影响而非"偏离 checklist"
- 结构依据：Design principles "Severity requires impact. Likelihood × impact, not deviation from a checklist." + "Defense-in-depth gaps are not vulnerabilities."
- 为什么优越：禁止把"少加了一层防护"报成漏洞，避免安全报告被硬ening note 淹没
- 对比维度：很多 LLM 审计会把"没加 Helmet 中间件"直接报高危

### 优越点 9：target-neutral 报告
- 结构依据：阶段 6 从 verified records + ledger 派生 `REPORT.md` / `FINDINGS-DETAIL.md` / `NEEDS-VALIDATION.md`
- 为什么优越：报告格式与目标技术栈解耦，下游可直接接不同仓库、不同语言的审计产物做聚合
- 对比维度：定制化审计脚本的报告格式每次都要重写

### 优越点 10：Cloudflare 生产环境反哺
- 结构依据：README "This is the skill that seeded Cloudflare's vulnerability discovery harness, described in Build your own vulnerability harness. The harness grew into a multi-stage, fleet-wide system."
- 为什么优越：设计原则不是论文推演，是从一个真实跑过 fleet 的 harness 里抽出来的最小子集；13 个攻击类文件的切分方式来自生产狩猎经验
- 对比维度：多数 AI 安全 skill 是博主把 OWASP 翻译成 prompt，没有实战验证

---

## 4. 前 10 优化增强点

### 优化点 1：校验器只有 Node 单语言
- 当前状态：`validate-*.cjs` 是 CJS，Python/Rust 生态团队无法在 CI 里直接跑
- 优化方向：把 JSON Schema 抽成纯 JSON（已经是 `report-schema.json`），再提供一个 `jsonschema` 通用命令入口或 prebuilt binary
- 预期收益：非 Node 团队也能把 findings 校验接进 CI

### 优化点 2：缺少"已修复回归"机制
- 当前状态：多次跑是 additive，但 README 没说已确认的修复如何自动从 findings.json 移除
- 优化方向：ledger 增加 `fixed_by_commit` 字段，下次跑自动 diff 已修复项
- 预期收益：把审计从"一次性报告"变成"持续监控"

### 优化点 3：攻击类提示词是静态 Markdown，没有版本化演化
- 当前状态：`AI-AND-LLM.md` 等文件一旦写死，新攻击技巧（如新提示注入变体）只能靠 PR
- 优化方向：把每个攻击类拆成"核心方法论 + 可热更新的技巧库"，支持外部 YAML 注入
- 预期收益：新威胁出现时不用等发版

### 优化点 4：缺少对 LLM 输出 token 成本的显式预算
- 当前状态：6 阶段会 spawn 大量子 agent，README 没给 token 预算或降级开关
- 优化方向：在 SKILL.md 加 `--max-hunters` / `--budget` 参数，小规模 repo 可关掉 memory-safety / desktop 类
- 预期收益：小项目跑一次不再花几十美元

### 优化点 5：coverage-ledger 的 schema 没公开示例
- 当前状态：仓库里有 validator 和测试，但 README 没贴一个最小可用 `coverage-ledger.json` 样例
- 优化方向：在 README 加"5 行最小 ledger"代码块
- 预期收益：新用户不用先读 validator 源码就能手写 ledger

### 优化点 6：沙箱只描述要求，没给参考实现
- 当前状态：Requirements 段列了"禁外网、allowlist、资源限额"，但没给 Docker/firecracker 参考配置
- 优化方向：加 `examples/sandbox/Dockerfile` + 网络禁用脚本
- 预期收益：中小团队不用自己拼沙箱

### 优化点 7：`needs_validation` 闭环缺失
- 当前状态：未验证项写进 `NEEDS-VALIDATION.md` 后，没有机制在新信息到达时自动重验
- 优化方向：ledger 记录"需要的环境/凭证"，提供 `resume.md` 入口让用户补凭证后从断点续跑
- 预期收益：不会因为一次没跑成而永远丢掉线索

### 优化点 8：报告缺跨仓库聚合视图
- 当前状态：每次 run 是独立目录 `run-<N>`，没有汇总所有 run 的 dashboard
- 优化方向：加一个 `audit-summary` 脚本，扫 `~/security-audit-skill/*/run-*` 输出跨仓库趋势
- 预期收益：安全团队可以看"哪些类 bug 在我们 org 反复出现"

### 优化点 9：测试只覆盖 validator，没覆盖 prompt 行为
- 当前状态：`.test.cjs` 只测 JSON schema，没有"给一个已知漏洞仓库，skill 是否能找到"的回归测试
- 优化方向：在 evals/ 下加 5-10 个带 ground-truth 漏洞的最小 repo fixture
- 预期收益：skill 演化时能防 prompt 退化

### 优化点 10：分发只走 Skills CLI，缺少市场插件
- 当前状态：`npx skills add` 是主路径，没有 Claude Code marketplace / Codex plugin 原生一键装
- 优化方向：发一个 `plugin.json`，让 `/plugin marketplace add cloudflare/security-audit-skill` 也能装
- 预期收益：复用各 agent 已有的插件生态

---

## 5. ProcessOn 全景图信息

- 文件夹名称：security-audit-skill
- 图表标题：cloudflare/security-audit-skill 结构性全景图 v4
- 图表链接：https://www.processon.com/view/link/6ab0e8da1a8a2243131ab44c （v4：13节点/14连线/4分组，techblue，auto_layout）
- 图中应包含：6 阶段流水线（Recon→Hunting→Validation→Structured→Record Verif→Report）、13 个攻击类文件、2 个零依赖校验器、report-schema.json、沙箱外部依赖

## 6. 幕布文档信息

- 文档名称：security-audit-skill — 日榜研究
- 文档 ID：7qgLzq234Oc
- 包含内容：场景问题 + 前10优越点 + 前10优化点 + ProcessOn 链接

---

## 2 分钟动作

- [ ] 装一下：`npx skills add https://github.com/cloudflare/security-audit-skill --skill security-audit --global`
- [ ] 找一个你自己的小 repo，对 agent 说 "security audit this codebase"
- [ ] 把第一次跑的 `findings.json` 留下来当 baseline，过一周再跑第二次对比覆盖率增量
