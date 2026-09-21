<!-- 数据说明：快照日期 2026-09-21，来源 GitHub Trending daily（since=daily）。当日仅 13 个项目（周一 UTC 午夜刚重置），非完整 20 个。本项目排名 #11。 -->

# mihail911/modern-software-dev-assignments — 架构研究分析

> 抓取时间：2026-09-21 | Stars：4,643 | 排名：#11 | 主语言：Python
> 项目类别：A/B 混合 — Stanford CS146S《The Modern Software Developer》Fall 2025 课程作业仓库（B 类代码模板 + A 类教学内容编排）
> 仓库地址：https://github.com/mihail911/modern-software-dev-assignments

**下一步可执行：** 按 README 跑 `conda create -n cs146s python=3.12 && poetry install --no-interaction`，然后从 `week1/k_shot_prompting.py` 的第一个 TODO 开始改 prompt。

**TL;DR：** 斯坦福一门新课的 8 周作业集：Week 1 用本地 Ollama 练 6 种 prompting 技术，Week 2 起用 Cursor/Claude Code 迭代一个 FastAPI+SQLite 全栈 app，Week 3 手写 MCP server，Week 4-5 做自治编码 agent，Week 6 Semgrep 扫漏洞，Week 8 demo day。本质是"教 2026 年软件工程师怎么用 AI 工具链干活"。

---

## 1. 场景问题：该项目主要解决什么问题

### 场景 1：CS 专业学生不知道"用 AI 写代码"该怎么教
- **目标用户：** 斯坦福本科生（CS146S Fall 2025 选课学生）+ 全球自学者
- **痛点：** 传统 CS 课教数据结构/操作系统，但"如何用 Cursor/Claude Code/MCP/自治 agent 做产品"没有成体系教材；网上教程是零散博客
- **如何介入：** 8 周渐进式作业：先练 prompting 基础（week1）→ 在已有 starter 上用 AI 工具迭代产品（week2）→ 自己写 MCP server（week3）→ 用 AI agent 自动化（week4-5）→ 安全扫描（week6）→ 全栈 demo（week8）
- **效果：** 一学期下来学生能独立用 AI 工具链把一个 idea 变成可演示的全栈 app

### 场景 2：企业想培训工程师"AI 辅助开发"
- **目标用户：** 内部培训负责人、技术 Lead
- **痛点：** 不知道从哪开始教、教什么顺序、怎么验收
- **如何介入：** 直接把这套周作业搬来用；每周有明确 assignment.md + writeup.md + rubric
- **效果：** 8 周出一个内部"AI 工程师"训练营雏形

### 场景 3：想入门 AI 工程但不会搭环境
- **目标用户：** 自学开发者
- **痛点：** 网上示例要么只讲 prompt 工程、要么只讲框架，没有"从零到能跑的产品"的完整路径
- **如何介入：** 顶层 README 给了 Anaconda + Poetry + Ollama 的一键环境；每周 starter 代码可跑；TODO 标记明确"你只需要改这里"
- **效果：** 按顺序做下来，week1 跑通 6 个 prompting 脚本，week2 起有一个真能打开的 web app

### 场景 4：想理解 MCP / Agent 生态的工程师
- **目标用户：** 后端/平台工程师
- **痛点：** MCP 官方 spec 抽象，不知道怎么写第一个 server；自治 agent 概念满天飞但缺可跑例子
- **如何介入：** week3 assignment.md 明确要求"包一个真实外部 API 成 MCP server"；week4/5 用 Claude Code 自动化 repo 内任务
- **效果：** 周末就能跑通一个自己的 MCP server 接到 Claude Desktop

---

## 2. 组成结构与技术组件

### 2.1 目录/内容结构（基于实际 `gh api contents/` 输出）

```
modern-software-dev-assignments/
├── README.md                 # 环境安装（Anaconda + Poetry）
├── pyproject.toml            # Poetry 依赖（FastAPI/SQLAlchemy/Pydantic/OpenAI/Ollama）
├── poetry.lock
├── week1/                    # Prompting 技术（本地 Ollama）
│   ├── assignment.md         # 60 分 rubric（6 个技术 × 10 分）
│   ├── k_shot_prompting.py
│   ├── chain_of_thought.py
│   ├── tool_calling.py
│   ├── self_consistency_prompting.py
│   ├── rag.py
│   └── reflexion.py
├── week2/                    # Action Item Extractor（FastAPI + SQLite starter）
│   ├── assignment.md         # 用 Cursor 改进现有 app
│   ├── app/  frontend/  tests/
│   └── writeup.md            # 学生记录 prompt + 改动
├── week3/                    # 手写 MCP Server
│   └── assignment.md
├── week4/                    # 自治编码 agent IRL
│   ├── backend/  frontend/  data/  docs/
│   ├── Makefile  pre-commit-config.yaml
│   └── writeup.md
├── week5/                    # 最小全栈 starter（自治 agent 实验）
│   └── README.md / assignment.md / ...
├── week6/                    # Semgrep 安全扫描
│   └── assignment.md（要求修复 ≥3 个漏洞）
├── week7/                    # 增强版全栈 starter
│   └── README.md / assignment.md / ...
└── week8/                    # Demo day：多栈 AI 加速全栈 web app
    └── assignment.md / writeup.md
```

### 2.2 技术栈/工程化清单（从 `pyproject.toml` 提取）

- **Python：** `>=3.10,<4.0`（README 推荐 3.12）
- **Web：** FastAPI ≥0.11.0 + uvicorn[standard] ≥0.23 + SQLAlchemy ≥2.0 + Pydantic ≥2.0
- **LLM：** openai ≥1.0 + ollama ^0.5.3（week1 用本地 mistral-nemo:12b / llama3.1:8b）
- **Dev 工具：** pytest ≥7 + httpx ≥0.24 + black ≥24 + ruff ≥0.4（E/F/I/UP/B）+ pre-commit ≥3.6
- **包管理：** Poetry（`package-mode = false`，因为是作业集不是库）
- **Editor：** 作业明确推荐 Cursor（含 Cursor Pro 学生免费兑换链接）
- **本地模型：** Ollama（week1 必须 pull mistral-nemo:12b + llama3.1:8b）
- **Week 6 专用：** Semgrep（静态分析）

### 2.3 核心数据流 / 学习路径

```
week1  学 prompting 6 法（本地 Ollama）
   │   改 TODO → 跑测试 → 写 writeup
   ▼
week2  FastAPI+SQLite starter → Cursor 改产品
   │   学会"AI 改已有代码"
   ▼
week3  自己写 MCP server（STDIO/HTTP）
   │   学会"给 AI 工具挂工具"
   ▼
week4  在 repo 里用 Claude Code 做 ≥2 个 automation
   │   学会"自治 agent 跑在真实代码库"
   ▼
week5  最小全栈 starter 跑自治 agent
   ▼
week6  Semgrep 扫漏洞 + 修 3 个
   │   学会"AI 生成代码要扫安全"
   ▼
week7  增强版 starter
   ▼
week8  Demo day：多栈 AI 加速全栈 web app
```

每周作业的共同模式：**assignment.md（任务+rubric）→ 改 TODO / 用 Cursor → writeup.md（记录 prompt 和改动）**。

---

## 3. 前 10 结构性优越点

### 优越点 1：教学曲线按"AI 工程栈"真实顺序排
- **结构依据：** week1 prompting → week2 在 starter 上迭代 → week3 写 MCP → week4 自治 agent → week6 安全 → week8 全栈 demo
- **为什么优越：** 不是先讲 LLM 理论再写代码，而是先让学生在本地 Ollama 上亲手改 prompt，再逐步引入工具、协议、agent、安全；每一步都能跑起来
- **对比：** 多数 AI 工程课是"先 Transformer 数学再 RAG 框架"，劝退 90% 实操派学生

### 优越点 2：starter 代码可跑，TODO 标记明确
- **结构依据：** week1 每个 `.py` 文件顶部写任务，学生"look for all the places labeled `TODO`… That should be the only thing you have to change (i.e. don't tinker with the model)"
- **为什么优越：** 学生不会卡在"环境搭不起来"，只聚焦在该学的点（prompt 设计）
- **对比：** 很多课程作业给一份空 README 让学生从零写，光环境配置就耗一周

### 优越点 3：本地模型优先，零 API 成本
- **结构依据：** week1 强制 Ollama + `mistral-nemo:12b` / `llama3.1:8b`，assignment.md 给了 macOS/Linux/Windows 三种安装方式
- **为什么优越：** 学生不需要 OpenAI key、不花钱、可离线；课程可扩展性极强
- **对比：** 大多数 LLM 课程作业默认 `OPENAI_API_KEY`，学生没 key 直接放弃

### 优越点 4：writeup.md 强制记录 prompt 和改动
- **结构依据：** week2 assignment.md："use `writeup.md` to document your progress… include the prompts you use, as well as any changes made by you or Cursor"
- **为什么优越：** 把"和 AI 对话的过程"变成可评分产物；学生被迫反思哪条 prompt 有效
- **对比：** 普通作业只看最终代码，学生用 AI 一键生成却学不到 prompt 工程

### 优越点 5：Rubric 量化到点
- **结构依据：** week1："60 pts total — 10 for each completed prompt across the 6 different prompting techniques"
- **为什么优越：** 学生知道"做完什么得多少分"；TA 批改客观
- **对比：** 很多课程作业只说"做个 RAG 系统"，没有验收标准

### 优越点 6：工具链对齐 2026 真实业界
- **结构依据：** week2 推荐 Cursor（含 Pro 学生兑换）、week4 用 Claude Code 做 automation、week3 手写 MCP server、week6 Semgrep
- **为什么优越：** 教的就是 2026 年工程师日常在用的工具，不是教科书里的老旧栈
- **对比：** 高校课程普遍还在教 2018 年的 Spring/Vue 模板

### 优越点 7：MCP 手写课是稀缺资源
- **结构依据：** week3 assignment.md："Design and implement a Model Context Protocol (MCP) server that wraps a real external API… locally (STDIO) or remotely (HTTP)"
- **为什么优越：** MCP 是 2025-2026 AI 工具互操作的事实标准，多数课程只讲"用 MCP client"，这门课让学生自己写 server
- **对比：** 公开 MCP 教程多是"怎么用现成 server"，教怎么写 server 的极少

### 优越点 8：安全周单独成块
- **结构依据：** week6 专门用 Semgrep 扫 starter app，要求修 ≥3 个漏洞
- **为什么优越：** 承认"AI 生成代码会有安全问题"，让学生从第 6 周就养成扫描习惯
- **对比：** 大多数 AI 编码课程完全跳过安全，学生带着漏洞上线

### 优越点 9：Poetry + pre-commit + ruff 工程规范从第一周就上
- **结构依据：** `pyproject.toml` 配 black/ruff/pre-commit；week4/5/7 自带 `Makefile` + `pre-commit-config.yaml`
- **为什么优越：** 学生写作业的同时就在学工业级 Python 工程规范，不是 `python script.py` 野路子
- **对比：** 课程作业仓库常见做法是"扔几个 .py 文件"，没有 lint/format/CI

### 优越点 10：可演化的 starter 架构（week4→5→7）
- **结构依据：** week4 初版 starter、week5 是"minimal full-stack starter for experimenting with autonomous coding agents"、week7 是"slightly enhanced full-stack starter (copied from Week 5) with a few backend improvements"
- **为什么优越：** 同一个代码库在 4 周内迭代三次，学生能看到"真实项目怎么演进"而不是每周换个全新代码
- **对比：** 多数课程作业每周独立，学生学不到"代码演化"这件事

---

## 4. 前 10 优化增强点

### 优化点 1：week3/8 没有 starter 代码
- **当前状态：** week3 只有 assignment.md，week8 也只有 assignment.md
- **优化方向：** 给一个空 MCP server 模板（FastMCP 骨架）和 week8 全栈项目脚手架
- **预期收益：** 学生不会卡在"从零建项目结构"上

### 优化点 2：测试/评测集缺失
- **当前状态：** week1 说 "iterate to improve results until the test script passes"，但测试脚本未在公开 repo
- **优化方向：** 把 golden test 脚本也开源（脱敏后），让自学者也能跑评分
- **预期收益：** 课程对外可自学性大幅提升

### 优化点 3：缺少参考答案/solution
- **当前状态：** 只有 assignment.md 和 writeup.md 模板，没有 reference solution
- **优化方向：** 开一个 `solutions/` 分支，每周给一份参考实现
- **预期收益：** 自学者卡住时能对比，而不是卡一晚上

### 优化点 4：Windows 体验未验证
- **当前状态：** README 只说 Python 3.12 + Anaconda，week1 才在 Ollama 安装处提了 Windows；assignment.md 里"press Cmd+Shift+V"这种 Mac 快捷键
- **优化方向：** 每节加 Windows/Linux 对应快捷键；CI 跑一次 Windows runner 验证
- **预期收益：** 国内 Windows 用户不会因环境问题弃课

### 优化点 5：LLM provider 锁定在 Ollama/OpenAI
- **当前状态：** pyproject 依赖 `openai` 和 `ollama`，没提 Anthropic / Gemini / local vLLM
- **优化方向：** 用 LiteLLM 或 OpenAI 兼容接口抽象，让学生可切任意 provider
- **预期收益：** 课程更通用，也教"多 provider 抽象"这个工程实践

### 优化点 6：前端栈未指定
- **当前状态：** week2/4/5 有 `frontend/` 目录但没说用什么框架
- **优化方向：** 在 starter 里固定一个现代栈（如 Vite+React），并在 README 说明
- **预期收益：** 学生不用花一周挑前端框架

### 优化点 7：缺少 CI 跑作业
- **当前状态：** 没有 `.github/workflows/` 自动跑 pytest
- **优化方向：** 加一个 CI：学生 push PR 时自动跑 week1 测试、ruff、pytest
- **预期收益：** 助教批改负担下降；学生本地也能提前看 CI 结果

### 优化点 8：版本演进未标注
- **当前状态：** pyproject version = `0.1.0`，没有 changelog；week7 是"copied from Week 5 with improvements"
- **优化方向：** 给每周 starter 打 git tag（`w5-base`、`w7-enhanced`），写 CHANGELOG
- **预期收益：** 学生能 diff week5→week7 学到"怎么改架构"

### 优化点 9：缺少"AI 用错"反例
- **当前状态：** 作业教怎么用 AI 工具，但没有专门一周讲"AI 生成的代码哪里会错"
- **优化方向：** 加一节 prompt 注入、幻觉、越权调用工具的反例实验室
- **预期收益：** 学生不只是"会用"，还知道"哪里会翻车"

### 优化点 10：demo day 评分标准未公开
- **当前状态：** week8 只说"navigate to this form for details"，rubric 不公开
- **优化方向：** 在 week8/assignment.md 里直接列 demo 评分维度（功能/架构/AI 用法/演示）
- **预期收益：** 学生最后一周有明确冲刺目标；外部学习者也能自我对标

---

## 5. ProcessOn 全景图信息

- 文件夹名称：`modern-software-dev-assignments`
- 图表标题：`mihail911/modern-software-dev-assignments 结构性全景图`
- 图表链接：https://www.processon.com/view/link/6ab0d1126a29601cdfc48f88
- 图中应包含：
  - 顶层定位：Stanford CS146S《The Modern Software Developer》8 周作业
  - 中层：8 周递进（week1 prompting → week2 FastAPI starter → week3 MCP server → week4-5 自治 agent → week6 Semgrep → week7 增强 starter → week8 demo）
  - 底层：技术栈（Python 3.12 / Poetry / FastAPI / SQLAlchemy / Ollama / Cursor / Claude Code / Semgrep / pytest+ruff+pre-commit）

## 6. 幕布文档信息

- 文档名称：`modern-software-dev-assignments — 日榜研究`
- 文档 ID：`7zLcHSWjVOc`（[打开](https://mubu.com/doc/7zLcHSWjVOc)）
- 包含内容：场景问题 + 前 10 优越点 + 前 10 优化点 + ProcessOn 链接

---

**2 分钟收尾动作：** 现在就 `conda create -n cs146s python=3.12` 把环境建起来；明天花 30 分钟跑 week1 的 `k_shot_prompting.py`，把 TODO 里的 prompt 改成你自己的一版。
