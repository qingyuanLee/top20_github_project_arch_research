# public-apis/public-apis — 架构研究分析

> 抓取时间：2026-09-18 | Stars：481255 | 排名：#3 | 主语言：Python（校验脚本）/ 内容为 Markdown
> 项目类别：A类（资源聚合）
> 仓库地址：https://github.com/public-apis/public-apis

## 1. 场景问题：该项目主要解决什么问题

本仓库是一份**手工策展的免费公共 REST API 目录**，用 Markdown 表格按约 50 个领域分类收录可免费调用的公开 API，并标注认证方式、HTTPS、CORS、Postman 集合等可直接上手的元信息。

- **场景一：做黑客松/原型开发、急需免费数据接口的独立开发者**
  - 场景角色：参加 hackathon、周末项目或 MVP 验证的开发者。
  - 痛点：临时需要天气、图片、汇率、笑话等测试数据，不知道哪些 API 免费、要不要 key、支不支持前端直连（CORS），逐个查官方文档太慢。
  - 该仓库如何介入：按领域分表，每条 API 一行，固定列出 Auth（No/apiKey/OAuth…）、HTTPS、CORS、Postman Run 按钮，一眼判断能否直接 fetch。
  - 效果：10 分钟内挑到一个"无需 key、支持 CORS"的 API 直接用进 demo，跳过注册流程。

- **场景二：为产品找第三方数据源的后端工程师**
  - 场景角色：要为应用选型天气/汇率/地理位置 API 的工程师。
  - 痛点：商业 API 对比要看免费额度、认证复杂度、是否绑设备购买；网上测评带广告。
  - 该仓库如何介入：CONTRIBUTING 明确"只收有免费额度、不强制购买设备/服务的 API"，并把认证方式枚举化，便于横向过滤。
  - 效果：在对应分类表内按 Auth=No / CORS=Yes 筛出候选，再去官方文档对比免费额度，选型效率提升。

- **场景三：想贡献一个 API 的社区成员**
  - 场景角色：发现某个好用但冷门的免费 API 的用户。
  - 痛点：不知道收录格式、命名规范、是否被接受。
  - 该仓库如何介入：CONTRIBUTING.md 给出表格模板、枚举字段值、PR 标题格式（`Add X API`）、每条 PR 只加一条、描述≤100 字等硬规则。
  - 效果：照着模板加一行提 PR，CI 自动校验格式与死链，合并路径清晰。

## 2. 组成结构与技术组件

### 2.1 目录/内容结构

根目录（`gh api .../contents/` 实际返回）：

| 路径 | 职责 |
|---|---|
| `README.md`（约 26 万字节） | 主体：APILayer 商业推荐区 + Learn more + Index 索引 + 约 50 个分类的 Markdown 表格 |
| `CONTRIBUTING.md` | 贡献规范：表格 schema、枚举字段值、PR 规则、反营销声明 |
| `scripts/` | 工程化脚本：`validate/`、`tests/`、`github_pull_request.sh`、`requirements.txt` |
| `.github/workflows/` | 三个 CI：`validate_links.yml`、`test_of_validate_package.yml`、`test_of_push_and_pull.yml` |
| `LICENSE` | MIT |

**内容分类体系**：README 底部 Index 列出约 50 个领域分类（Animals、Anime、Art、Books、Cloud Storage、Cryptocurrency、Development、Finance、Machine Learning、News、Weather 等），每个分类一张 Markdown 表，表内按字母序排列。

**表格化条目 schema**（CONTRIBUTING.md 实测）：
```
| API(链接到文档) | 描述(≤100字) | Auth | HTTPS | CORS | Call this API(Postman) |
```
枚举字段：
- Auth 仅接受 `OAuth` / `apiKey` / `X-Mashape-Key` / `No` / `User-Agent`；
- CORS 仅接受 `Yes` / `No` / `Unknown`；
- 描述 ≤100 字符、首字母大写；表内按字母序、每列两侧留一空格。

### 2.2 技术栈/工程化清单（A 类）

- **贡献者协议**：`CONTRIBUTING.md` 开头即反营销声明——"This API list is not a marketing tool"，拒绝为付费/需购买设备的 API 打广告（例：需先买智能插座设备的 API 直接拒）。
- **PR 硬规则**：一 PR 只加一条；标题 `Add X API`；名称不含 TLD、不以 "API" 结尾；描述≤100 字；按字母序；跨分类时归入"最贴合主用途"的分类（如 Instagram 归 Social 而非 Photography）；提交前 squash commits；不收录已有 API 的新版本。
- **自动化 CI**（`.github/workflows/`）：
  - `validate_links.yml`：死链/链接存活检查；
  - `test_of_validate_package.yml` + `test_of_push_and_pull.yml`：跑 `scripts/validate` 与 `scripts/tests`，校验表格格式、排序、枚举字段合法性。
- **脚本工具**：`scripts/validate` 做规则校验，`scripts/tests` 做单元测试，`requirements.txt` 固化 Python 依赖——这是同类 awesome 清单中少有的"带测试套件"工程化。
- **社区与商业**：由社区成员 + APILayer 员工共同维护，README 顶部有 APILayer 自家 API 推荐横幅与 Postman Run 按钮，Discord 社区运营。

### 2.3 核心协作流

贡献者在 fork 上建分支 → 按表格 schema 加一行（命名/描述/枚举字段全部合规）→ 发 `Add X API` 标题的 PR → CI 自动跑死链检查与表格格式/排序/枚举校验 → 维护者人工审核（去重、反营销、免费额度确认）→ squash 合并入对应分类表 → 通过 README Index 暴露，全球开发者按 CORS/Auth 过滤使用。

## 3. 前 10 结构性优越点（为什么它能突出）

### 优越点 1：表格化 schema 把"能否直接用"变成可过滤维度
- 结构依据：CONTRIBUTING 定义六列表格，Auth/HTTPS/CORS 为枚举字段。
- 为什么优越：开发者不用点进文档，仅凭 CORS=Yes、Auth=No 就判断能否前端直连，信息密度远高于纯链接列表。
- 对比维度：build-your-own-x 类清单只有"语言+标题+URL"，无运行时元信息，需逐一点开试。

### 优越点 2：用 CI 自动校验表格格式与排序
- 结构依据：`.github/workflows/test_of_validate_package.yml` + `scripts/validate`/`tests`。
- 为什么优越：字母序、列对齐、枚举合法这类机械规则由脚本兜底，人工只判断内容价值。
- 对比维度：纯人工维护的 Markdown 表长期必然格式漂移，本仓库靠测试套件保持一致。

### 优越点 3：死链由 CI 主动巡检
- 结构依据：`validate_links.yml` workflow。
- 为什么优越：数百个外部 API URL 随时间失效，自动检查避免读者点进死链。
- 对比维度：无 CI 的 awesome 清单死链靠用户举报，反应慢。

### 优越点 4：反营销、反"购买设备"准入，保住中立性
- 结构依据：CONTRIBUTING 首段明确"not a marketing tool"，拒收需先买硬件/订阅的 API。
- 为什么优越：把"免费可用"作为硬门槛，防止仓库变成付费 API 的广告位，维护社区信任。
- 对比维度：不设此门槛的 API 目录很快被企业投放淹没。

### 优越点 5：枚举字段约束保证全表可机读
- 结构依据：Auth 五选一、CORS 三选一的闭合枚举。
- 为什么优越：枚举使表格可被脚本解析、可二次开发成搜索/筛选工具，而非纯人类阅读。
- 对比维度：自由文本描述的清单无法程序化过滤。

### 优越点 6：描述≤100 字 + 一 PR 一条，控制合并冲突
- 结构依据：PR 规则"Description 不超 100 字""Add one link per Pull Request"。
- 为什么优越：短描述保持表格紧凑；一 PR 一条让每个 PR diff 极小、review 快、冲突少。
- 对比维度：大型批量 PR 一次改几十行，review 成本高、易冲突。

### 优越点 7：命名规范（去 TLD、不结尾 API）统一品牌
- 结构依据：CONTRIBUTING "Don't mention the TLD"、"name should not end with API"。
- 为什么优越：全表 API 名称风格一致（Gmail 而非 Gmail.com / Gmail API），视觉整齐、便于排序。
- 对比维度：命名混乱的清单同一 API 可能出现多种写法，造成重复。

### 优越点 8：Postman Run 按钮把"发现"变"即调"
- 结构依据：表格末列"Call this API"链到 Postman Collection。
- 为什么优越：开发者点 Run in Postman 即可发起真实请求，零配置验证 API 是否可用。
- 对比维度：只给文档链接的目录，验证一个 API 仍需手动写请求。

### 优越点 9：跨分类归属有明确仲裁规则
- 结构依据：CONTRIBUTING "归入最贴合主用途的分类"（Instagram 归 Social）。
- 为什么优越：避免同一 API 在多分类重复出现，保持唯一归属。
- 对比维度：无仲裁规则的清单同一项目反复出现，浪费版面。

### 优越点 10：MIT 许可 + 商业赞助续命
- 结构依据：LICENSE 为 MIT；README 由 APILayer 员工与社区共同维护。
- 为什么优越：MIT 允许自由复用，商业赞助提供长期维护人力，比纯志愿仓库更可持续。
- 对比维度：纯志愿维护的资源库常因维护者倦怠而停更。

## 4. 前 10 优化增强点（主要关节点可优化方向）

### 优化点 1：首屏商业横幅过重
- 当前状态：README 顶部大段 APILayer 自家 API 推广表与 banner，占据首屏。
- 优化方向：将商业推荐压缩到顶部一条横幅或底部 Sponsors 区，首屏直接放说明与 Index。
- 预期收益：强化中立清单定位，降低"软文"观感。

### 优化点 2：缺少活跃/已死 API 时间维度
- 当前状态：只列 API 是否存在，不标注最近可用状态、免费额度余量。
- 优化方向：CI 定期 ping 各 API，对连续失败项加 `[dead]` 标记或移出。
- 预期收益：读者不必自己试一个挂掉的 API。

### 优化点 3：无在线搜索/筛选界面
- 当前状态：26 万字节 Markdown，只能 Ctrl-F 翻表。
- 优化方向：从 README 自动生成静态站点，支持按 CORS/Auth/领域组合筛选。
- 预期收益：在数百个 API 中秒级定位符合条件者。

### 优化点 4：免费额度无量化标注
- 当前状态：只说"有免费额度"，不写每日调用上限。
- 优化方向：增加"免费额度"列（如 1000次/月）。
- 预期收益：选型时直接判断是否够生产用。

### 优化点 5：分类间无标签交叉
- 当前状态：一个 API 只能在一个分类，跨领域 API 只能选其一。
- 优化方向：引入标签机制，允许跨分类检索。
- 预期收益：提升边缘 API 的可发现性。

### 优化点 6：数据与展示未分离
- 当前状态：数据直接写在 README.md，CI 脚本反向解析 Markdown 校验。
- 优化方向：将 API 数据抽到独立 JSON/YAML，README 由脚本生成。
- 预期收益：校验更稳、易生成多语言站点、避免正则解析 Markdown 的脆弱性。

### 优化点 7：无用户评价/成功率沉淀
- 当前状态：条目无"被多少人用过""稳定性如何"反馈。
- 优化方向：挂 GitHub Discussion 或配套状态页，收集使用者反馈。
- 预期收益：从静态目录进化为有质量信号的社区库。

### 优化点 8：CORS=Unknown 项缺乏主动补全
- 当前状态：不少 API 标 Unknown，靠贡献者随手填。
- 优化方向：CI 自动发一个带 Origin 头的探测请求，自动判定 CORS。
- 预期收益：减少 Unknown 占比，提升数据质量。

### 优化点 9：无版本/变更日志视图
- 当前状态：API 更新（如改版、停服）无条目级 changelog。
- 优化方向：维护条目历史，记录最近一次验证时间。
- 预期收益：读者知道信息新鲜度，降低踩坑。

### 优化点 10：缺乏非英语社区版本
- 当前状态：仅英文 README。
- 优化方向：提供中/日/西语镜像索引（引用而非翻译整表）。
- 预期收益：扩大非英语开发者覆盖面。

## 5. ProcessOn 全景图信息

- 文件夹名称：public-apis（位于分类文件夹「文档类-资源导航与awesome清单」下）
- 图表标题：public-apis 结构性全景图
- 图表链接：https://www.processon.com/view/link/6aacaed26a54b67d4fa606a3
- 图中应包含：顶层定位（免费 API 目录+APILayer 赞助）、中层三大领域表与约50分类 Index、表格化词条 Schema、底层工程化（贡献硬规则→CI 死链/格式校验→人工去重→MIT 治理），连线标注"加一行→CI→review→合并"流。

## 6. 幕布文档信息

- 文档名称：public-apis — 架构研究
- 文档 ID：2ugTt4hAqHc
- 包含内容：场景问题 + 前10优越点 + 前10优化点 + ProcessOn 链接
