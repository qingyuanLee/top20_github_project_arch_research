<!-- 数据说明：快照日期 2026-09-21，来源 GitHub Trending daily（since=daily）。当日仅 13 个项目（周一 UTC 午夜刚重置），非完整 20 个。本项目排名 #10。 -->

# anthropics/financial-services — 架构研究分析

> 抓取时间：2026-09-21 | Stars：35,499 | 排名：#10 | 主语言：Python（仓库本体为 Markdown+YAML+Shell 模板）
> 项目类别：B 类（代码架构 — Anthropic 官方金融行业 Claude 插件/Agent 模板集合）
> 仓库地址：https://github.com/anthropics/financial-services

**下一步可执行：** 在 Cowork 里 `Settings → Plugins → Add plugin` 粘贴 `https://github.com/anthropics/financial-services`，先装 `financial-analysis` 核心包，再按业务挑 1-2 个 agent。

**TL;DR：** Anthropic 官方出的金融行业"参考实现"，把投研/投行/PE/基金运营/财富管理 5 大 FSI 垂直，封装成 **10 个自包含 agent 插件 + 6 个 vertical 技能包 + 12 个 MCP 数据连接器**，同一份 markdown/YAML 既能当 Cowork 插件装，也能通过 Managed Agents API 部署到自己的编排层。

---

## 1. 场景问题：该项目主要解决什么问题

### 场景 1：投行/PE 分析师半夜赶 pitch book
- **目标用户：**  bulge-bracket 投行分析师、PE 投资经理
- **痛点：**  comps / precedents / LBO 模型手搭，CIM 草稿、buyer list、process letter 重复劳动，晚上 11 点还在改 Excel 公式
- **如何介入：** 装 `pitch-agent` / `investment-banking` vertical，自带 `/comps` `/dcf` `/lbo` `/cim` `/teaser` `/buyer-list` `/merger-model` 等 slash command，技能文件里写死了卖方模板的步骤和 QC 清单
- **效果：**  从"零到初稿"从半天压到 1-2 小时，QC 由 `ib-check-deck` / `audit-xls` 自动扫公式硬编码和平衡检查

### 场景 2：基金运营月底关账
- **目标用户：**  fund admin、会计、财务 ops
- **痛点：**  GL 对账找 break、accruals、roll-forward、LP 报表审计，跨系统手工追溯 root cause 极慢
- **如何介入：**  `gl-reconciler`、`month-end-closer`、`statement-auditor`、`valuation-reviewer` 4 个 agent 串成一条关账流水线，每个 agent 是 `managed-agent-cookbooks/<slug>/` 里的 `agent.yaml` + leaf-worker subagents
- **效果：**  break 自动定位 + 自动路由给 sign-off 人，LP 报表分发前先过 `statement-auditor` 一轮

### 场景 3：券商研究员更新 earnings note
- **目标用户：**  equity research associate
- **痛点：**  财报电话会 transcript + 10-Q/10-K 解析 → 模型 update → 写 note，三步割裂
- **如何介入：**  `earnings-reviewer` agent 自动 ingest earnings call + filings → 触发 `model-update` skill → 起草 note；`/earnings` `/earnings-preview` `/initiate` `/thesis` `/catalysts` 覆盖覆盖全生命周期
- **效果：**  电话会后 30 分钟出 model delta + note 草稿

### 场景 4：财富顾问面对客户前 10 分钟
- **目标用户：**  RIA / 私人财富顾问
- **痛点：**  会前 briefing pack 要从 CRM、组合、estate 系统拼数据，合规预扫容易漏
- **如何介入：**  `claude-for-financial-advisors` 子包直连顾问的 CRM / 组合 / 规划系统，自带 meeting prep、prospect intake、rebalance review、compliance pre-check
- **效果：**  会前自动出 briefing pack，合规预扫在发送前跑一遍

---

## 2. 组成结构与技术组件

### 2.1 目录/内容结构（基于实际 `gh api contents/` 输出）

```
financial-services/
├── plugins/
│   ├── agent-plugins/          # 10 个自命名 agent（每个 slug 一个目录）
│   │   ├── pitch-agent/
│   │   ├── meeting-prep-agent/
│   │   ├── market-researcher/
│   │   ├── earnings-reviewer/
│   │   ├── model-builder/
│   │   ├── valuation-reviewer/
│   │   ├── gl-reconciler/
│   │   ├── month-end-closer/
│   │   ├── statement-auditor/
│   │   └── kyc-screener/
│   ├── vertical-plugins/      # 6 个 FSI 垂直技能包 + MCP 连接器
│   │   ├── financial-analysis/   # 核心：11 个建模 skill + 全部 MCP 连接器
│   │   ├── investment-banking/
│   │   ├── equity-research/
│   │   ├── private-equity/
│   │   ├── fund-admin/
│   │   └── operations/
│   └── partner-built/          # 合作伙伴共建：lseg、spglobal
├── managed-agent-cookbooks/    # 每个 agent 一份 headless 部署模板（agent.yaml + subagents）
├── claude-for-msft-365-install/  # M365 插件管理员装机工具
├── scripts/
│   ├── check.py                # lint manifest + 跨文件引用 + 漂移检测
│   ├── validate.py
│   ├── sync-agent-skills.py    # 把 vertical 的 skill 同步到所有引用它的 agent
│   ├── orchestrate.py          # 参考 event loop：路由 handoff_request
│   ├── deploy-managed-agent.sh # 部署到 /v1/agents
│   ├── version_bump.py
│   └── test-cookbooks.sh
├── .githooks/  .github/
└── CLAUDE.md
```

### 2.2 技术栈/工程化清单

- **零构建：**  全部内容是 Markdown + YAML + JSON + Shell，没有 npm/pip 构建链（README 原文："Everything is file-based — markdown and JSON, no build step."）
- **MCP 连接器（12 个，集中在 `financial-analysis/.mcp.json`）：** Daloopa、Morningstar、S&P Global (Kensho)、FactSet、Moody's、MT Newswires、Aiera、LSEG、PitchBook、Chronograph、Egnyte、Box
- **三种分发通道同源：** Cowork plugin / Claude Code CLI (`claude plugin install ...`) / Managed Agents API (`POST /v1/agents`)
- **CI 钩子：** `scripts/check.py` 在 push 前 lint 所有 manifest、校验跨文件引用、检测 bundled skill 是否从 vertical 源漂移
- **分发渠道：** `.claude-plugin/` 目录定义 marketplace 元数据，`claude plugin marketplace add anthropics/financial-services` 即可拉到

### 2.3 核心数据流

```
用户在 Cowork/CLI/Managed Agent 触发
        │
        ▼
   选 agent（自包含插件：system prompt + skills/）
        │
        ▼
   agent 自动触发相关 skill（如 /dcf /comps）
        │
        ▼
   通过 .mcp.json 中的 MCP server 拉数据
   （FactSet / PitchBook / Box / Egnyte …）
        │
        ▼
   产出 .xlsx / .pptx / memo / note
   （xlsx-author / pptx-author 头less 生成）
        │
        ▼
   staged for human sign-off（不自动执行交易/过账）
```

Managed Agent 部署路径额外多一层：`deploy-managed-agent.sh` 解析 file references → 上传 skills → 创建 depth-1 leaf-worker subagents → POST orchestrator 到 `/v1/agents`；`orchestrate.py` 提供 `handoff_request` 事件路由参考实现。

---

## 3. 前 10 结构性优越点

### 优越点 1：一份源，三种部署形态
- **结构依据：** README "Everything here is available two ways from one source" + `plugins/agent-plugins/<slug>/` 同时被 Cowork 和 `managed-agent-cookbooks/<slug>/` 引用
- **为什么优越：** 同一份 system prompt + 同一份 skills，既是交互式 Cowork 插件，又是 headless API agent；客户不用维护两套实现
- **对比：**  多数 Agent 模板仓库只给一种入口（要么 SDK 代码、要么 prompt 片段），Anthropic 直接把"交互 vs 自动"两条产品路线收敛到同一目录

### 优越点 2：agent 插件自包含（self-contained）
- **结构依据：** README："Each agent plugin is self-contained — it bundles the skills it uses, so installing the agent is all you need."
- **为什么优越：** 装一个 pitch-agent 不用再去手动选 6 个依赖 skill；插件即"交付单元"
- **对比：**  很多 skill 仓库要求用户自己拼"agent = prompt + 5 个 skill + 3 个 connector"，上手心智成本高

### 优越点 3：vertical 与 agent 分层解耦
- **结构依据：** `plugins/vertical-plugins/<vertical>/skills/` 是 skill 源；`plugins/agent-plugins/<slug>/skills/` 是同步副本；`scripts/sync-agent-skills.py` 负责同步
- **为什么优越：**  skill 改一次，所有用到它的 agent 自动跟上；同时 agent 仍可独立微调
- **对比：**  单层扁平 skill 仓库容易出现"同一个 DCF skill 在 5 个 agent 里各改各的"的漂移

### 优越点 4：MCP 连接器中心化
- **结构依据：**  全部 12 个 MCP connector 集中在 `vertical-plugins/financial-analysis/.mcp.json`，其他 vertical 共享
- **为什么优越：**  加新数据源只改一处；切换供应商只需把 `.mcp.json` 指向新 endpoint（README "Swap connectors" 一节明示）
- **对比：**  企业里常见做法是每个 agent 各自维护一份 MCP 配置，12 个供应商 × N 个 agent 就是 N×12 处配置

### 优越点 5：垂直覆盖完整 FSI 价值链
- **结构依据：**  5 大 vertical（IB / ER / PE / fund-admin / operations）+ 财富顾问子包 + 2 个 partner-built
- **为什么优越：**  从 pitch（投行前端）→ model（建模）→ IC memo（PE 投决）→ GL recon（中后台）→ KYC（运营）一条链路打通，而不是只做"金融聊天机器人"
- **对比：**  多数金融 LLM demo 只做 ER 一条线，缺少 fund admin / ops 这种"不性感但刚需"的后中后台场景

### 优越点 6：强制 human-in-the-loop 边界写死在定位里
- **结构依据：** README 顶部 `[!IMPORTANT]` 块："They do not make investment recommendations, execute transactions, bind risk, post to a ledger, or approve onboarding; every output is staged for human sign-off."
- **为什么优越：**  合规边界不是事后补丁，而是写进 repo 元数据；每个 agent 都默认"草稿级"产出
- **对比：**  金融 AI 项目最容易踩的坑就是"AI 自动下单/自动过账"，这个 repo 从第一行就排除掉

### 优越点 7：partner-built 目录预留生态位
- **结构依据：** `plugins/partner-built/lseg`、`plugins/partner-built/spglobal` 与官方 vertical 平级
- **为什么优越：**  LSEG / S&P Global 这种数据源厂商能直接以"插件"形式贡献，而不是挤进官方仓库；Anthropic 维持中立
- **对比：**  厂商集成通常走 README 一段外链，难以版本化、难以安装

### 优越点 8：可执行的贡献者自检脚本
- **结构依据：** `scripts/check.py` ：lint manifest + 验证 cross-file 引用 + 检测 bundled skill 漂移；贡献者指南要求 "Run `python3 scripts/check.py` before pushing"
- **为什么优越：**  内容型仓库最怕"改了 vertical 源没同步到 agent 副本"，脚本把这种漂移变成 CI 失败
- **对比：**  多数 prompt/skill 仓库只靠人肉 review，跨文件引用漂移几乎必然发生

### 优越点 9：M365 装机作为独立 on-ramp
- **结构依据：** `claude-for-msft-365-install/` 是 Claude Code 插件（不是 Cowork 插件），帮 IT 管理员生成定制 add-in manifest、走 Azure admin consent、写 per-user routing
- **为什么优越：**  企业落地最大的阻力不是 prompt 好不好，而是"怎么在公司 tenant 里合规装上"；这个子包直接解决 IT admin 流程
- **对比：**  多数 LLM 企业部署文档停在"请联系 IT"，没有真正的自动化装机脚本

### 优越点 10：partner 数据端点直接给 URL
- **结构依据：** README "MCP Integrations" 表格里 12 个 provider 直接给出 `https://mcp.factset.com/mcp` 这种可粘贴 URL
- **为什么优越：**  企业安全团队只要白名单这 12 个域名；不需要每个 vendor 自己写 client SDK
- **对比：**  很多"支持 FactSet"的项目实际要客户自己去找文档、申请 sandbox、写 OAuth 流程

---

## 4. 前 10 优化增强点

### 优化点 1：缺少自动化测试覆盖 skill 输出
- **当前状态：** `scripts/check.py` 只校验结构和引用，不校验 skill 产出的模型/PPT 内容
- **优化方向：**  给每个 skill 加 golden fixture（输入 + 期望输出 schema），CI 跑一次端到端 smoke
- **预期收益：**  Anthropic 模型升级时不会静默改坏 DCF 公式输出

### 优化点 2：MCP connector 健康检查缺失
- **当前状态：** `.mcp.json` 只列 URL，没有心跳/版本探测
- **优化方向：**  加 `mcp healthcheck` 脚本，启动前 ping 每个 connector 并报告认证状态
- **预期收益：**  分析师不会在跑 `/comps` 时才发现 FactSet token 过期

### 优化点 3：agent 间 handoff 没有 schema 化
- **当前状态：** `orchestrate.py` 只是"参考 event loop"，`handoff_request` 事件没有正式 JSON schema
- **优化方向：**  把 handoff payload 用 JSON Schema / TypeScript 类型固化，第三方编排层能静态校验
- **预期收益：**  企业自建编排层不必读源码猜事件格式

### 优化点 4：skill 版本化策略不显式
- **当前状态：**  vertical 改了 skill 后，agent bundled 副本靠 `sync-agent-skills.py` 手动跑
- **优化方向：**  给每个 skill 加 semver，agent 声明 `skill@^1.2`，sync 时锁定
- **预期收益：**  升级不会破坏已上线的 pitch-agent 行为

### 优化点 5：合规审计日志不可见
- **当前状态：**  README 强调 human sign-off，但没有"谁审了哪份产出"的 audit trail 模板
- **优化方向：**  给每个 agent 模板加 `audit_log.jsonl` 输出约定（prompt、模型、输出、审核人、时间）
- **预期收益：**  满足 SOC2 / 404B 类审计要求，企业安全团队更容易过

### 优化点 6：多租户/权限模型未示范
- **当前状态：**  README 假设一个 firm 内所有人共享 agent；PE 基金内部 IC memo 通常只对 GP/LP 特定角色可见
- **优化方向：**  在 `agent.yaml` 里加 `rbac` 字段示例，标注哪些 skill 仅 GP 可用
- **预期收益：**  基金管理员开箱即用，不用自己加权限层

### 优化点 7：partner-built 目录缺少贡献者门槛
- **当前状态：**  LSEG / S&P Global 是"partner-built"，但没有 partner 提交模板
- **优化方向：**  加 `CONTRIBUTING-partners.md`：vendor 必须提供 sandbox URL、测试账号、版本 SLA
- **预期收益：**  下一个想接入的 vendor（如 Bloomberg）能按同样格式提交

### 优化点 8：本地离线模式未覆盖
- **当前状态：**  所有 connector 都是 SaaS MCP endpoint；涉密项目（如投行内部 pool）无法断网使用
- **优化方向：**  提供一个 `--offline` 模板，把 connector 换成本地 CSV / 内部 REST
- **预期收益：**  满足投行"不能出内网"的合规要求，扩大客户群

### 优化点 9：writeup / 案例对比缺失
- **当前状态：**  README 只列"是什么"，没有"用 pitch-agent 把一份真实 CIM 从 4 小时压到 40 分钟"这种 before/after 案例
- **优化方向：**  加 `examples/` 目录，每个 vertical 一份端到端 walkthrough（输入文档 → agent 输出 → 人工修改点）
- **预期收益：**  新客户 30 分钟内判断"这东西在我这能不能用"

### 优化点 10：CLI 与 Cowork 行为对齐靠人肉
- **当前状态：**  同一份 markdown 同时喂 Cowork 和 Managed Agent API，但没有测试证明两者行为一致
- **优化方向：**  加一个 golden prompt 测试：同一 agent 在 Cowork 和 Managed Agent 下跑同一输入，diff 输出
- **预期收益：**  消除"我在 Cowork 上好好的，headless 部署就乱了"的常见投诉

---

## 5. ProcessOn 全景图信息

- 文件夹名称：`anthropics-financial-services`
- 图表标题：`anthropics/financial-services 结构性全景图`
- 图表链接：https://www.processon.com/view/link/6ab0d04d6a29601cdfc48da4
- 图中应包含：
  - 顶层定位：Anthropic FSI 参考实现（10 agents + 6 verticals + 12 MCP）
  - 中层：5 大 FSI 垂直（IB / ER / PE / fund-admin / ops）+ 财富顾问 + partner-built
  - 底层：MCP 数据连接器（FactSet / PitchBook / LSEG / Box …）+ 部署通道（Cowork / Claude Code / Managed Agents API）+ 工程脚本（check.py / sync-agent-skills.py / orchestrate.py）

## 6. 幕布文档信息

- 文档名称：`anthropics-financial-services — 日榜研究`
- 文档 ID：`5RvRuztAb2c`（[打开](https://mubu.com/doc/5RvRuztAb2c)）
- 包含内容：场景问题 + 前 10 优越点 + 前 10 优化点 + ProcessOn 链接

---

**2 分钟收尾动作：** 选一个你手上最痛的垂直（建议从 `financial-analysis` 核心 + 你最熟的那条业务线开始），今天就把 `.mcp.json` 指向你已经有 license 的那家数据商，跑一次 `/comps` 看输出。
