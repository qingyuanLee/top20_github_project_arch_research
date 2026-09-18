# GitHub Stars Top20 项目架构研究 — 每日报告

> **报告日期**：2026-09-18
> **数据来源**：GitHub Search API（`search/repositories?q=stars:>1&sort=stars&order=desc&per_page=20`），已认证调用
> **原始数据**：[data/top20_raw_2026-09-18.json](data/top20_raw_2026-09-18.json)
> **分析项目数**：20 / 20
> **ProcessOn 全景图**：20 张（均已开启永久分享）
> **幕布大纲文档**：20 篇（均归入 `top20_github_arch_research` 文件夹）

---

## ⚠️ 数据说明

以下 5 个项目的 star 数疑似异常（短期内增速远超正常开源项目，个人/品牌项目与数万 fork 的量级反差），均来自真实 GitHub API 返回，仓库真实存在，但分析结论仅供参考：

| # | 项目 | Stars | 异常说明 |
|---|---|---|---|
| 6 | openclaw/openclaw | 390,022 | AI Agent 框架，描述模糊（"The lobster way"），TypeScript 项目 |
| 12 | obra/superpowers | 288,183 | 2025-10 创建，11 个月达 288k★，Agent Skills 框架 |
| 15 | mattpocock/skills | 264,609 | 2026-02 创建，7 个月达 264k★，Agent Skills 集合 |
| 16 | affaan-m/ECC | 261,256 | "Agent harness performance optimization system"，JavaScript 项目 |
| 19 | NousResearch/hermes-agent | 246,597 | "The agent that grows with you"，Python 项目 |

---

## Top20 总览

| # | 项目 | Stars | 类别 | 分析文档 | ProcessOn 全景图 | 幕布文档 |
|---|---|---|---|---|---|---|
| 1 | codecrafters-io/build-your-own-x | 547,926 | A类·学习路径 | [分析](projects/build-your-own-x/analysis.md) | [全景图](https://www.processon.com/view/link/6aacad177783ce2a62b98555) | `jlQ2bgC4bc` |
| 2 | sindresorhus/awesome | 507,212 | A类·资源导航 | [分析](projects/awesome/analysis.md) | [全景图](https://www.processon.com/view/link/6aacae1bc6646636dfd2957a) | `G0sA-VvbXc` |
| 3 | public-apis/public-apis | 481,255 | A类·资源导航 | [分析](projects/public-apis/analysis.md) | [全景图](https://www.processon.com/view/link/6aacaed26a54b67d4fa606a3) | `2ugTt4hAqHc` |
| 4 | freeCodeCamp/freeCodeCamp | 455,701 | B类·代码架构 | [分析](projects/freeCodeCamp/analysis.md) | [全景图](https://www.processon.com/view/link/6aacb148926a46649d0b3341) | `323wnLy6rc` |
| 5 | EbookFoundation/free-programming-books | 397,067 | A类·学习路径 | [分析](projects/free-programming-books/analysis.md) | [全景图](https://www.processon.com/view/link/6aacb18a9e63607e80fc9516) | `i5hwKQkUrc` |
| 6 | openclaw/openclaw ⚠️ | 390,022 | B类·代码架构 | [分析](projects/openclaw/analysis.md) | [全景图](https://www.processon.com/view/link/6aacad97aa338a4e8a923b4d) | `t2fGspBMbc` |
| 7 | donnemartin/system-design-primer | 370,545 | A类·系统设计 | [分析](projects/system-design-primer/analysis.md) | [全景图](https://www.processon.com/view/link/6aacae796a29601cdfbea9c5) | `2QW98aPUeXc` |
| 8 | nilbuild/developer-roadmap | 367,556 | B类·代码架构 | [分析](projects/developer-roadmap/analysis.md) | [全景图](https://www.processon.com/view/link/6aacaf2cc6646636dfd297c7) | `6E7FcfBOIHc` |
| 9 | jwasham/coding-interview-university | 361,120 | A类·学习路径 | [分析](projects/coding-interview-university/analysis.md) | [全景图](https://www.processon.com/view/link/6aacafcfaa338a4e8a92410c) | `7S9hNlXgCrc` |
| 10 | vinta/awesome-python | 321,363 | A类·资源导航 | [分析](projects/awesome-python/analysis.md) | [全景图](https://www.processon.com/view/link/6aacb0719e63607e80fc925b) | `6s8Sf2VDEbc` |
| 11 | awesome-selfhosted/awesome-selfhosted | 319,966 | A类·资源导航 | [分析](projects/awesome-selfhosted/analysis.md) | [全景图](https://www.processon.com/view/link/6aacaf367783ce2a62b989c5) | `18gUJrxuXc` |
| 12 | obra/superpowers ⚠️ | 288,183 | B类·代码架构 | [分析](projects/superpowers/analysis.md) | [全景图](https://www.processon.com/view/link/6aacb0716a29601cdfbeb39e) | `63v4VRyXNbc` |
| 13 | practical-tutorials/project-based-learning | 283,694 | A类·学习路径 | [分析](projects/project-based-learning/analysis.md) | [全景图](https://www.processon.com/view/link/6aacb145c6646636dfd29d04) | `R3TuF2nnXc` |
| 14 | 996icu/996.ICU | 277,142 | A类·社会运动 | [分析](projects/996.ICU/analysis.md) | [全景图](https://www.processon.com/view/link/6aacb32a9e63607e80fc9898) | `3UuSWXRfKXc` |
| 15 | mattpocock/skills ⚠️ | 264,609 | B类·代码架构 | [分析](projects/skills/analysis.md) | [全景图](https://www.processon.com/view/link/6aacb3f39e63607e80fc9a44) | `5wn_5EPcyXc` |
| 16 | affaan-m/ECC ⚠️ | 261,256 | B类·代码架构 | [分析](projects/ECC/analysis.md) | [全景图](https://www.processon.com/view/link/6aacadb6c66afe02ff75ab6b) | `2bBSs7Y86bc` |
| 17 | react/react | 250,550 | B类·代码架构 | [分析](projects/react/analysis.md) | [全景图](https://www.processon.com/view/link/6aacae609e63607e80fc8788) | `5L8m8Y8q2Xc` |
| 18 | torvalds/linux | 249,359 | B类·代码架构 | [分析](projects/linux/analysis.md) | [全景图](https://www.processon.com/view/link/6aacaee6c0bae4107ed2e1a2) | `5YP3QhSFxrc` |
| 19 | NousResearch/hermes-agent ⚠️ | 246,597 | B类·代码架构 | [分析](projects/hermes-agent/analysis.md) | [全景图](https://www.processon.com/view/link/6aacaf6c6a29601cdfbeac40) | `vjeAAyzsXc` |
| 20 | trimstray/the-book-of-secret-knowledge | 244,410 | A类·学习路径 | [分析](projects/the-book-of-secret-knowledge/analysis.md) | [全景图](https://www.processon.com/view/link/6aacafe5926a46649d0b2ff5) | `4H0cmCADerc` |

---

## 分类统计

| 维度 | 数量 | 项目 |
|---|---|---|
| **A类（资源聚合/文档类）** | 12 | build-your-own-x, awesome, public-apis, free-programming-books, system-design-primer, coding-interview-university, awesome-python, awesome-selfhosted, project-based-learning, 996.ICU, the-book-of-secret-knowledge |
| **B类（代码架构类）** | 8 | freeCodeCamp, openclaw, developer-roadmap, superpowers, skills, ECC, react, linux, hermes-agent |
| **学习路径与动手教程** | 6 | build-your-own-x, free-programming-books, coding-interview-university, project-based-learning, the-book-of-secret-knowledge |
| **资源导航与 awesome 清单** | 5 | awesome, public-apis, awesome-python, awesome-selfhosted |
| **AI Agent 框架/技能** | 5 | openclaw, superpowers, skills, ECC, hermes-agent |
| **真正的大型软件项目** | 3 | freeCodeCamp, react, linux |

---

## 核心发现摘要

### A类资源聚合项目的共性优越点
1. **内容即数据**：awesome-selfhosted 将全部条目拆到姊妹仓库的一项目一 YAML，用 hecat 渲染，5 条 CI 流水线自动化
2. **极严的贡献治理**：awesome-python 采用 shortlist 非 catalog 模式，3+2 上限，PyPI 下载量为准入信号
3. **diff 感知 lint**：project-based-learning 自研 check_readme.py 只校验 PR 新增行，避免历史债务阻塞贡献
4. **词条 schema 标准化**：public-apis 表格化索引（名称/描述/认证/HTTPS/CORS/链接），CI 自动校验链接存活
5. **多语言 i18n**：free-programming-books 覆盖 46 种语言书籍 + 39 种课程，996.ICU 覆盖 35 语种

### B类代码架构项目的共性优越点
1. **Monorepo + 工作区管理**：freeCodeCamp（pnpm + turbo）、react（yarn workspaces）、openclaw（pnpm + Rust crates）
2. **Fiber 双缓冲架构**：react 的 reconcile 阶段可中断、可优先级调度，commit 阶段同步不可中断
3. **子系统解耦**：linux 内核 22 种架构支持、drivers 分类、kernel/fs/net/mm 子系统清晰边界
4. **Agent Skills 工程化**：superpowers 的 `<HARD-GATE>` 硬门控 + SessionStart 钩子防失忆；skills 的 User-invoked/Model-invoked 二元调用边界
5. **内容即代码**：developer-roadmap 的 roadmap 内容以结构化 JSON 存储，TS 脚本同步到网站

---

## 交付物清单

| 交付物 | 数量 | 位置 |
|---|---|---|
| 项目分析文档（analysis.md） | 20 | `projects/{项目名}/analysis.md` |
| ProcessOn 结构性全景图 | 20 | 见上表链接（永久分享） |
| 幕布大纲文档 | 20 | 幕布 `top20_github_arch_research` 文件夹 |
| 原始 API 数据 | 1 | `data/top20_raw_2026-09-18.json` |
| 本报告 | 1 | `2026-09-18-top20-arch-report.md` |

---

## 分析方法论

每个项目的分析均包含以下 6 个章节：
1. **场景问题**：按真实使用场景说明（角色 + 痛点 + 介入 + 效果）
2. **组成结构与技术组件**：目录结构 + 技术栈 + 数据流/协作流
3. **前 10 结构性优越点**：A类侧重思维工具/思维工程/场景问题三维度；B类侧重架构设计/技术选型/工程实践
4. **前 10 优化增强点**：具体可操作的改进方向
5. **ProcessOn 全景图信息**：架构图 + 分享链接
6. **幕布文档信息**：大纲文档 ID

所有事实均来自 GitHub API 实际抓取的 README、目录结构、配置文件，已在各分析文档中标注来源。

---

*报告生成时间：2026-09-18 | 下次更新：手动触发*
