# 下一步：`uv tool install 'cua-bench[browser]'` 跑一个无 VM 的 simulated task，或 `irm https://cua.ai/driver/install.ps1 | iex` 装 Driver 连本机 macOS/Windows/Linux。

> 快照日期：2026-09-21 | 来源：GitHub Trending daily（since=daily）| 当日仅 13 个项目（周一 UTC 午夜刚重置），非完整 20 个
> 抓取时间：2026-09-21 | Stars：25,322 | 当日新增：1,018 | 主语言：HTML（仓库 monorepo，实际含 Rust/Python/TS/Swift）
> 项目类别：B 类（代码架构 / computer-use 2.0 平台）
> 仓库地址：https://github.com/trycua/cua

**TL;DR：** Cua 把"给 AI agent 一台能用的电脑"拆成 5 个可独立使用的产品：**Fleets**（隔离云桌面池）、**Driver**（跨 OS 桌面驱动，CLI/MCP/SDK）、**CUA-S1**（小而专的 System 1 决策模型）、**Lume**（Apple Silicon 本地 macOS/Linux VM）、**Cua-Bench**（任务+评估+轨迹导出）。它本身不做 agent，只做"电脑 + 工具 + benchmark"，让你带自己的 model 进来。

---

## 1. 场景问题：该项目主要解决什么问题

- [ ] **场景 A：训练/评估 computer-use agent 缺统一沙箱**
  - 目标用户：LLM 研究员 / agent 团队
  - 痛点：每家自己搭 VM、自己录轨迹、自己写 evaluator，无法横向对比
  - 介入方式：Cua-Bench 支持无 VM 的 simulated task，`uv tool install` 一行装好，跑完导出轨迹用于训练
  - 效果：从 0 到第一个 reward=1.0 的 task 只要几分钟

- [ ] **场景 B：让现有 coding agent（Claude Code/Codex/Cursor）操作本机 GUI**
  - 目标用户：想让 agent 点 Calc 验证结果、操作 LibreOffice 的工程师
  - 痛点：macOS 辅助功能权限、Windows UI Automation、Linux X11 三套 API 各写一遍
  - 介入方式：Cua Driver 提供 CLI / MCP / typed SDK，支持后台投递（不抢鼠标焦点）
  - 效果：agent 问 "compute 6×7 in Calculator" 就能跑通并自检显示 42

- [ ] **场景 C：跑 fleet 级 GUI 自动化**
  - 目标用户：SRE / 自动化测试团队
  - 痛点：要批量起 Linux 桌面、跑命令、截图、销毁资源
  - 介入方式：run.cua.ai 云桌面池，Sandbox SDK 本地和云共享同一套 API
  - 效果：代码从池里 claim 一台桌面、跑命令、存截图、释放，成本按秒算

- [ ] **场景 D：在 Apple Silicon 上跑本地 macOS Tahoe VM**
  - 目标用户：隐私敏感 / 离线场景
  - 痛点：云桌面太贵且数据出域；官方 Virtualization.Framework 包装层少
  - 介入方式：Lume 用 Apple Virtualization.Framework 起 vanilla macOS Tahoe，SSH 接入
  - 效果：从 restore image 一键起 VM，不用自己写 hypervisor 胶水

---

## 2. 组成结构与技术组件

### 2.1 目录结构（monorepo，来自 `git/trees/main` 与 `contents/libs`）

```
libs/
├── cua-bench/          # 任务定义、evaluator、轨迹导出（Python, uv 分发）
├── cua-driver/         # 跨 OS 桌面驱动（Rust 核心 + MCP/CLI/SDK 绑定）
├── cua-driver-rs/      # Rust 驱动原生实现
├── cua-s1/             # System 1 小模型（Python 训练/评估代码，权重在 HF）
├── cuabot/             # 内部 bot 容器
├── fleet/              # 云 fleet 控制面
├── kasm/               # KasmVNC 集成（MIT 子组件）
├── lume/               # Swift 写的 macOS/Linux VM 管理器
├── lumier/             # Lume 的 GUI/容器
├── python/             # Python 绑定与 agent 库
├── typescript/         # TS SDK 与 CLI
├── qemu-docker/        # QEMU 容器镜像
├── xfce/ + xfce-cua/    # XFCE 桌面镜像
.github/workflows/      # 60+ 个 cd-*/ci-* workflow（按组件拆分）
.github/scripts/        # 大量 release 编排与兼容性校验脚本
```

### 2.2 技术栈

- **多语言 monorepo**：Rust（driver 核心）、Python 3.12/3.13 + uv（bench/训练/SDK）、TypeScript（CLI/SDK）、Swift（Lume，依赖 Apple Virtualization.Framework）
- **容器/镜像**：Dockerfile + docker-compose，覆盖 xfce / Kasm / QEMU(linux/windows/android) / cuabot / lumier
- **外部依赖**：Playwright（bench 浏览器）、Kasm（MIT）、Microsoft OmniParser（CC-BY-4.0，可选）、ultralytics（AGPL-3.0，可选 `cua-agent[omni]`）
- **分发**：PyPI `cua-bench`、curl install.sh / install.ps1、Hugging Face 模型权重 `cua-ai/cua-s1-forms`

### 2.3 核心数据流

```
Your agent (Claude Code / Codex / 自研)
   │
   ├── CLI / MCP / typed SDK
   ▼
Cua Driver  ── 本地 macOS / Windows / Linux 原生桌面
   │
   ▼ (也可走云)
Sandbox SDK  ── 本地 sandbox ──┐
                              ├── 同一 API ──► Cua Bench（任务/evaluator/轨迹）
云 Fleet 池  ── run.cua.ai   ──┘
   │
   ▼
CUA-S1 小模型（forms 决策）── 只打分不生成 token，执行交给 Driver
```

---

## 3. 前 10 结构性优越点

### 优越点 1：一个仓库覆盖 computer-use 全栈
- 结构依据：README "Choose your path" 五张卡片 = Fleets / Driver / CUA-S1 / Lume / Cua-Bench
- 为什么优越：别的项目要么只做沙箱、要么只做模型、要么只做 benchmark；Cua 把"训练→评估→部署→fleet"闭环放一起
- 对比维度：OpenAI computer-use、Anthropic computer-use 都只给 agent 端，沙箱/benchmark 要自己拼

### 优越点 2：本地 sandbox 和云 Fleet 共享同一套 Sandbox SDK
- 结构依据：README "Local sandboxes and Fleets share the Sandbox SDK, but credentials, images, operations, and runtime requirements differ."
- 为什么优越：开发时在本地跑，上线切云，业务代码不改；避免"本地能跑云上报错"
- 对比维度：多数云沙箱厂商没有本地等价物，本地开发路径是另一套 API

### 优越点 3：Driver 跨 OS 且支持后台投递
- 结构依据：README "Background delivery lets agents work without moving your pointer or taking focus when the app and platform support it"
- 为什么优越：GUI 自动化最大痛点是"跑 agent 时不能用电脑"；后台投递把 OS 权限差异抽象掉
- 对比维度：PyAutoGUI 等库必须抢焦点，无法在用户办公时跑

### 优越点 4：CUA-S1 是"System 1 决策模型"而非又一个通用 VLM
- 结构依据：README "small, specialized System 1 models... scoring decisions from structured interface elements and document values rather than generating a response token by token"
- 为什么优越：填表、选字段这类高频决策用小模型打分，比让通用 VLM 逐 token 输出快几个数量级
- 对比维度：Grounded-SAM、UI-TARS 都是通用 VLM，延迟和成本都高

### 优越点 5：Cua-Bench 不依赖 VM/Docker/API key 即可起步
- 结构依据：README "Start with a simulated task that requires no VM, Docker, or model API key"
- 为什么优越：降低研究门槛，学生/小团队也能贡献任务和 evaluator
- 对比维度：大多数 computer-use benchmark 需要付费云 + 模型 key

### 优越点 6：CI/CD 按组件极致拆分
- 结构依据：`.github/workflows/` 下有 `cd-py-cua-driver.yml`、`cd-rust-cua-driver.yml`、`cd-ts-fleet.yml`、`cd-swift-lume.yml`、`cd-container-qemu-windows.yml` 等 60+ 文件
- 为什么优越：monorepo 里只改一个 Python 包不会触发 Rust 构建；release 通道按语言/组件解耦
- 对比维度：很多多语言 monorepo 用一个大 workflow，改一行都要等全量构建

### 优越点 7：release 工程化有大量自带测试
- 结构依据：`.github/scripts/tests/` 下有 `test_cua_driver_release_wiring.py`、`test_cua_fleet_release_wiring.py`、`test_lume_release_publication.py` 等几十个 release 装配测试
- 为什么优越：release 本身被测试覆盖，避免"发了个包但安装脚本指错版本"
- 对比维度：多数开源项目的 release 流程靠人手点按钮

### 优越点 8：明确的第三方 license 边界
- 结构依据：README License 段单独列 Kasm(MIT)、OmniParser(CC-BY-4.0)、ultralytics(AGPL-3.0) 三个可选组件
- 为什么优越：AGPL 组件可选引入，避免用户无意中把 AGPL 传染到自己产品
- 对比维度：很多项目把 AGPL 依赖藏在 optional extra 里，README 不提示

### 优越点 9：Lume 直接基于 Apple Virtualization.Framework
- 结构依据：README "Create and manage local macOS and Linux VMs on Apple Silicon using Apple's Virtualization.Framework"
- 为什么优越：比 QEMU 快且原生支持 macOS guest；SSH 接入对自动化友好
- 对比维度：OrbStack/UTM 是产品，Lume 是可编程的 VM 生命周期 API

### 优越点 10：MIT 全栈开放 + 商业化互补
- 结构依据：根 LICENSE MIT，云 Fleet 跑在 run.cua.ai 付费
- 为什么优越：核心技术栈全开源避免 lock-in，付费只买"云桌面容量"，符合 open-core 健康模型
- 对比维度：很多 computer-use 项目只开源 SDK、把沙箱闭源，实际用起来还是要付费

---

## 4. 前 10 优化增强点

### 优化点 1：主语言标 HTML 误导
- 当前状态：GitHub 语言统计显示 HTML（可能是文档/落地页），实际核心是 Rust/Python/TS/Swift
- 优化方向：在 `linguist` 配置里把 docs/ 和 img/ 排除，让主语言显示 Rust 或 TypeScript
- 预期收益：Trending 筛选"我要找 Rust computer-use"时能被找到

### 优化点 2：Driver 平台支持矩阵分散
- 当前状态：README 多次指向 `docs/reference/cua-driver/platform-support` 外部链接
- 优化方向：在 README 放一个紧凑的 "macOS 15+ / Windows 11 / Ubuntu 24.04" 三列支持表
- 预期收益：用户 30 秒内判断能不能跑

### 优化点 3：CUA-S1 模型权重只在 Hugging Face
- 当前状态：README 指向 HF，国内用户访问慢
- 优化方向：加 ModelScope / 阿里云镜像，并在 MODEL_CARD.md 写明
- 预期收益：降低中文用户试用门槛

### 优化点 4：Fleets 免费试用路径不清晰
- 当前状态：README 大按钮是 `run.cua.ai`，但没写免费额度
- 优化方向：在 "Your first result" 段加"首 5 小时免费 / 信用卡可选"
- 预期收益：提高转化率，降低"不敢点"的心理门槛

### 优化点 5：release 脚本太多贡献者门槛高
- 当前状态：`.github/scripts/` 下 30+ Python 脚本，新贡献者不知道改哪个
- 优化方向：加 `docs/release-architecture.md` 画一张"改 X 包 → 触发哪个 workflow"的图
- 预期收益：社区贡献者不用读 30 个脚本才能提 PR

### 优化点 6：bench 任务格式缺官方模板
- 当前状态：`uv tool install` 后要自己看文档写 task
- 优化方向：加 `cua-bench init my-task` 脚手架命令
- 预期收益：从"读文档"到"跑起来"缩短 80%

### 优化点 7：Lume 只支持 Apple Silicon
- 当前状态：README 明说 Apple Silicon，但 Intel Mac / Windows 用户无法用
- 优化方向：明确标注"Intel Mac 请用 qemu-docker 路径"，或提供 UTM 后端
- 预期收益：避免用户装了才发现跑不起来

### 优化点 8：后台投递能力按平台限制，但 README 没说清楚
- 当前状态：README 说 "when the app and platform support it"，但没列哪些 app/平台支持
- 优化方向：在 platform-support 文档加"已验证支持后台投递的 app 清单"
- 预期收益：用户不会在不支持的 app 上浪费调试时间

### 优化点 9：CUA-S1 只有 forms 一个 profile
- 当前状态：README "The first research profile focuses on forms"
- 优化方向：roadmap 里列下一个 profile（如 "file-manager" / "settings"）
- 预期收益：让社区知道方向，也吸引贡献者

### 优化点 10：缺端到端教程视频/录屏
- 当前状态：README 有一个 50 秒 demo 链接到 GitHub user-attachments
- 优化方向：在 docs/ 加 3 个 3 分钟 walkthrough（本地 Driver、云 Fleet、Bench 写任务）
- 预期收益：新用户留存率提升

---

## 5. ProcessOn 全景图信息

- 文件夹名称：cua
- 图表标题：trycua/cua 结构性全景图 v4
- 图表链接：https://www.processon.com/view/link/6ab0e91d23868a0bc6a15b32 （v4：11节点/12连线/4分组，techblue，auto_layout）
- 图中应包含：5 大产品（Fleets/Driver/CUA-S1/Lume/Bench）、跨语言 monorepo 组件（Rust/Python/TS/Swift）、Sandbox SDK 共用层、外部依赖（Kasm/OmniParser/Playwright/HF）

## 6. 幕布文档信息

- 文档名称：cua — 日榜研究
- 文档 ID：35JvNIaMdOc
- 包含内容：场景问题 + 前10优越点 + 前10优化点 + ProcessOn 链接

---

## 2 分钟动作

- [ ] 跑 bench 最小样例：`uv tool install 'cua-bench[browser]' && uv tool run --from 'cua-bench[browser]' playwright install chromium`
- [ ] 如果你在 Windows：跑 `irm https://cua.ai/driver/install.ps1 | iex` 装 Driver
- [ ] 打开 run.cua.ai 看一眼 Fleets 控制台，确认免费额度
