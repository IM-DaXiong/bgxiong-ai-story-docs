> **历史演进产物，不代表当前设计，不作为当前实现或开发依据。**
>
> - 原路径：`release/03-开发指南/20261002-GUIDE-Agent-Skills导入创作与开发.md`
> - 文档标识：取自原文件名 `20261002-GUIDE-Agent-Skills导入创作与开发.md`（这是原文件名中的标识，不是本次整理推定的写作或生效日期）
>
> 现行架构请从仓库主导航进入查阅；本文只作为历史演进资料，不作为当前实现或开发依据。

# GUIDE · Agent Skills 导入、创作与开发

> **开源整理版**（2026-11-12）：已去除或改写私有路径、未公开端点、凭证示例，以及登录 / 认证 / 付费相关实现细节。若某节全部属于上述范围，将改为一句能力边界说明。


> **2026-10-09**：用途字段与随包说明已回写（见 §2a）。分类、标签与 MCP 完整管理仍以「补正PLAN」（开源版未收录该内部文档或源码路径）为准，**没有**写成用户可跟着操作的说明。跨客户端 / 真实迁移 NOT_RUN 不因文档更新变成 PASS。能力对照见 §2a 状态表。

- 文档时间：2026-10-03（Asia/Shanghai）；版本：v0.2（M11 模板与作者文档、M13 发布说明与支持矩阵并入；随 M10 切换继续修订）。
- 权威：[SPEC · Agent Skills 包加载与执行合同](../04-开发规范/20261002-SPEC-Agent-Skills包加载与执行合同.md)（诊断码、状态集合、包指纹以它和 `packages/app-contracts/v1/agent-skills.json` 为准）；总计划 「全项目 Agent Skills 通用规范统一改造」（开源版未收录该内部文档或源码路径）；执行进度只看 「执行台账」（开源版未收录该内部文档或源码路径），本文不记进度。
- 读者：想导入/编写 SKILL 的用户与作者（第 1–9 节）；要改本应用 SKILL 相关代码的开发者（第 11 节）；要写发布说明或判断“现在能承诺什么”的人看第 10 节。
- 状态说明：应用已按本文第 3、5–7 节的行为实现核心路径；界面入口随 U 波推进，尚未联调的部分在对应小节标注。**没有任何跨客户端或真实模型的实测结论写在本文里**（T 波未执行，记 NOT_RUN）。


> **2026-10-25 当前创作规则**：创作 Skill 以「片段主计划」（开源版未收录该内部文档或源码路径）及「片段核心合同」（开源版未收录该内部文档或源码路径）为准。完整剧集直接规划片段，再分别准备可编辑视频稿和环境图片提示词；不经过旧场次/故事板实体。本文旧包升级机制是历史运行机制说明，不能作为新 Skill 的编写模板，不新增兼容字段或旧流程转换。

### 升级处理（用户 2026-10-25 最新裁决）

当前应用执行采用 `metadata.bgx-workflow: clips-v1`。这是本应用执行代际声明，不是 Agent Skills 开放标准的格式必填项；没有声明的标准包仍可预览、导入、查看和导出，但不能用于新创作执行。作者必须按本文新流程实质重写，不能给旧流程自动贴标冒充已升级。

- 安装新版直接覆盖官方随包文件；技能库中来源明确为 builtin 的旧缓存刷新到当前随包版本。不写旧字段转换、样例指纹迁移或重命名 ID 迁移。用户导入的包保持原文，需重新导入当前方法。
- 仓库不再发行旧包目录或旧协议附件。升级后活动库只采用新版；旧版本字节仅为已有任务冻结资源保留，不重新作为新执行入口。
- 外部旧包、自改旧包、无法核实官方来源的旧包保留原文，库卡片显示不可用及更新说明。既不能因手改用途绕过，也不能用旧固定版本引用绕过；用户请作者按当前规范更新后重新导入。
- 只处理 Skill 库及其绑定/版本，不清理故事、画布、媒体、模型或第三方服务。升级失败明确显示诊断并保留现场，不半写索引、不静默换方法。

### 当前作者与开发智能体必须遵守

- 剧集规划用 `episode-clip-plan`，环境图提示词用 `clip-environment-prompt`，片段视频稿用 `clip-video-draft`；剧集写作、角色/道具提炼和名单整合保留各自独立用途。
- 新发布包不得声明 `chapter-scenes`、`scene-storyboard`、`optimize-storyboard-shot`、`storyboard-still-style`、`content-plan`。底层 registry 中尚存的历史 ID 不是当前作者接口，不映射成新用途、不兼容旧 workflow。
- 普通图像优化用 `optimize-image-prompt`，图片摄影风格用 `image-photography-style`（片段与自由画布生图），视频摄影风格用 `video-photography-style`；风格正文读取 `references/layout.json` 的 `visualStyleOverlay`，与优化助手独立。视频内容整理用 `optimize-segment`，模型表达适配用 `video-prompt-projection`。多视图排版只用于明确选择的图像参考用途，不能变成片段固定镜头数。
- 包正文只补充创作方法。结果字段和引用闭集由宿主当次 schema 提供；不附私有 workflow、步骤 op/bind、旧场次/镜头字段或另造输出 schema。现有包目录名即使带 storyboard/shots，也只是包名，不能据此触发旧业务。
- 一个片段可以多镜、切镜、运镜；15秒只作参考。保留分配给本片段的原文对白/旁白，无图也能完整写稿。环境是普通 image 候选，不要求 sceneId；参考身份/帧位属于本次引用上下文，不设为图片永久类型。
- 帧规划只写方法建议和用户明确选择的范围/上限，不宣传尚未发布的 Frame 工具，不固定6/9帧，不自动调用图片或视频。单项、批量、一键共享宿主原子任务，一键当前只产文字。
- 不暗换模型，不编造已读图/不存在的参考 ID。写作评分、标签、模板均为建议；评审未完成仍保留已有成果并如实标记。机器 schema、真实资源、安全与权限约束继续有效。
- 同名 builtin/fixtures 包同步用途；内容变更使用独立版本并冻结 digest。随包文件由安装包直接覆盖，库内官方缓存按下述规则刷新；外部旧包不转换、不覆盖用户原文。

## 1. 这是什么、不是什么

- 应用现在只有**一种** SKILL 作者格式：[Agent Skills 开放规范](https://agentskills.io/specification)的标准包——一个目录，里面至少有 `SKILL.md`（YAML frontmatter + Markdown 正文），可带任意附件。
- 旧的 `content-skill-v1` 私有包（`skill.manifest.json` + `pipeline.json` + 可选 WASM）**不再是新安装格式**。已安装的旧包通过第 7 节的一次性升级工具转成标准包；升级之外没有第二套运行路径，也没有兼容别名。
- 标准包是普通文件夹：可以原样交给别的支持 Agent Skills 的客户端使用。本应用**不保证**别的客户端能完成同样的任务——能否完成取决于那个客户端有没有对应工具与运行时（见第 9 节）。
- 本应用的“可执行”不等于“格式有效”：格式有效但缺少工具或运行时的包照常导入，执行时会明确显示限制，不会静默换模型、跳过步骤或报假成功。

## 2. 包长什么样（最小包与可选资源）

最小包只有一个文件：

```text
my-skill/
└── SKILL.md
```

`SKILL.md` 示例：

```markdown
---
name: my-skill
description: 一句话说明它做什么、什么时候该用它。模型只凭这句话决定要不要启用。
---

# 正文

这里写具体方法与步骤，Markdown 格式不限。
需要更多材料时引用包内相对路径，例如 `references/tone.md`。
```

要点：

- `name`：必填，小写字母/数字/`-`，不超过 64 字符，不以 `-` 开头结尾，不含连续 `--`，且**应与所在目录名一致**。导入时这些规则多数只给警告，作者保存/导出时全部按错误处理（SPEC §5、§7）。
- `description`：必填，1–1024 字符。它是目录里唯一的语义说明，写清“做什么 + 何时用”。
- `license`、`compatibility`、`metadata`、`allowed-tools`：可选，原样保留。`allowed-tools` 只是声明，**不会授予任何权限**。
- 附件放哪都行；`references/`、`assets/`、`scripts/` 只是约定，不是白名单。所有文件完整保存、完整导出，二进制只提供元数据与 hash，不会塞进模型上下文。
- **不需要**：`skill.manifest.json`、`pipeline.json`、`generation-policy.json`、`outputSchema`、`metadata.bgx`、`local.` 命名空间、WASM、TypeScript SDK（SPEC §11）。这些文件如果存在，只当普通附件保存。
- 结构化附件是**可选**的加速器：例如多视图布局 `references/layout.json`（样例见 `fixtures/agent-skills/native-image-policy/references/layout.json`）。存在时由应用工具读取并按应用合同校验；不存在时应用使用默认合同。普通方法包不需要写应用操作名。

### 展示名规范

发布包和起步模板应声明 `metadata.bgx-display-name`，使用简洁正式的用途名称，并与正文一级标题一致。不在名称中塞括号说明、步骤箭头、内部缩写或规格清单；细节放在 `description` 与正文。模型名称保留准确型号；教学包与起步模板标注“示例”。本轮定稿与全量清单见「展示名整理」（开源版未收录该内部文档或源码路径）。

## 2a. 用途 / bgx-hosts

工作台选择器只看有效用途。包在 `metadata.bgx-hosts` 里声明，值是空格分隔的字符串，每个 id 必须属于 「`agent-skill-hosts.json`」（开源版未收录该内部文档或源码路径）。用户可以在技能库改用途；改过之后以用户的为准。有效列表为空时，技能可以留在库里，但**不会**出现在任何一个工作台选择器。空列表不是默认全开。

图片／视频摄影选择器还核对包自身声明的媒体用途：图片包须声明 `image-photography-style` 或 `optimize-image-prompt`，视频包须声明 `video-photography-style`。同时勾选两个入口不会改变包的媒体用途；声明两类的包允许共用。图片优化方法仍须明确授权图片摄影用途并提供有效风格正文。列表和冻结执行使用同一判断，旧绑定也须通过。

| 当前用途 id | 工作台 / 阶段 |
|---------|--------|
| `story-chapters` | 剧集正文 |
| `chapter-roles` / `chapter-props` / `story-entity-integration` | 独立角色、道具提炼及名单整合 |
| `episode-clip-plan` | 完整剧集到片段规划 |
| `clip-environment-prompt` | 普通环境图片提示词 |
| `clip-video-draft` | 可直接使用的片段视频稿 |
| `clip-frame-plan` | 从已确认视频稿规划画面候选；仅文字 |
| `clip-frame-prompt` | 单个已规划画面的图片提示词；仅文字 |
| `multi-view` | 明确选择的多视图参考排版 |
| `optimize-image-prompt` | 普通图片提示词优化 |
| `optimize-segment` | 视频内容整理 |
| `video-photography-style` | 视频摄影风格 |
| `image-photography-style` | 图片摄影风格（片段与自由画布生图） |
| `video-prompt-projection` | 目标模型格式适配 |
| `postproduction-decoration` | 后期装饰 |

发布用样例 `fixtures/agent-skills` 和起步模板 `templates/agent-skill` 必须至少声明一个合法用途。门禁：`scripts/check-agent-skill-hosts-fixtures.ps1`。`tests/fixtures/agent-skills` 是测试夹具，不在这条门禁里。

同名包若同时在 `builtin-agent-skills` 和 `fixtures/agent-skills`，用途集合必须一致，正文允许教学差异。禁止只改其中一边的用途。视频提示词格式六份（H3 兼容、Seedance、可灵、Wan、通用、LTX 2.5）只在内置资源里，不在案例目录；项目开发智能体必须在同级 bgxiong-content-skills 中维护源码并按锁同步主仓；用户个人可从技能库导出副本修改后导入；H3 与其它包采用同样流程，不按名字限制发行。不要写成案例已经覆盖。`h3-prompt-writing` 已改为本项目自有的 `h3@1` 兼容写法，不包含 MiniMax-H3 仓库中的官方 Skill 原文（见 「发布结论」（开源版未收录该内部文档或源码路径））。

### 动作打斗内容与模型表达

视频工作台中的“内容优化”可以选择内置的“动作打斗提示词”，适用于现代格斗、武侠持械、多人对抗，也会忠实处理没有打斗的原文。先选目标视频模型，再选内容方法与文本模型；点击优化才执行。具有剧情上下文的生成入口同样使用当前所选内容方法，无上下文时仍按工作台提示处理。

该包声明 `optimize-segment` 与 `clip-video-draft`。内容整理按当前合同返回事实或正文；片段成稿直接写出可交给视频模型的完整提示词，不要求先输出母版、先过评审或先有图片。全部必要规则位于 `SKILL.md` 正文，按所选版本冻结后供当次创作使用；不需要附件或私有工作流。目标格式独立选择，默认跟随目标视频模型，由现有 Seedance、MiniMax-H3、可灵、Wan 或通用表达包处理。MiniMax-H3 是当前 `h3@1` 的项目自有兼容写法，不代表其他 MiniMax 型号已获专属适配；任何包都不标称厂商官方。摄影风格继续独立选择。

作者可在技能库导出内置包，改名编辑后作为标准用户包导入。运行源只在 `src-tauri/resources/builtin-agent-skills/`，没有重复的 fixtures 发行副本。内容包不决定厂商参数、参考标记或输出 schema；详细边界与验收见「动作打斗提示词计划」（开源版未收录该内部文档或源码路径）。

### 制作生图提示词优化 Skill

生图工作台沿用统一 Skills 的导入、用途、选择与激活合同。作者在标准包的 `metadata.bgx-hosts` 声明 `optimize-image-prompt`，即可用于生图提示词优化；不需要注册模型专用程序、SDK 或私有 manifest。包里的方法可以面向任意图像模型或工作流，目标模型只影响方法正文，不改变宿主执行流程。

内置 「Qwen Image 2.1 图像提示词」（开源版未收录该内部文档或源码路径） 是完整的单文件标准包，涵盖文生图和参考图编辑，声明 `optimize-image-prompt` 与 `clip-environment-prompt`，可用于普通图片优化和环境图片提示词；不再声明旧分镜优化用途。方法来源固定在官方 revision `ab96caa944157ca28a4f36b695c3047549f48f98`。可以从技能库导出后改名、编辑并作为用户包导入；该包只在内置资源里，不声称案例目录也有一份。

作者要遵守这些通用边界：

- 点击“优化提示词”才调用所选文本模型；不在生图提交时静默再次优化。优化结果保存为新版本，保留原稿。
- 外层输出使用应用当次给定的 JSON schema，目前是仅含 `optimizedPrompt` 的对象。模型或工作流官方增强器的 `rewritten_prompt`、`wh_ratio` 等外壳不能覆盖应用合同；包只决定字段内的提示词写法。
- 工作台参考图顺序是唯一编号依据，第一张 `<image1>`，第二张 `<image2>`，依此类推；优化模型和生图工作流使用相同顺序。不要根据内容重排或重新编号。
- 参考图编辑需要读取实际图片；模型不支持读图时明确报告限制，不凭文件名编造内容或切换成无图优化。画幅、分辨率和参考图集合仍由工作台控制。

随包作者文档在 `fixtures/content-skills/local-samples/`，安装后位于 `user-examples/agent-skills-authoring/`。用途闭集的包内副本是 `hosts-closure.*.md` 与 `agent-skill-hosts.snapshot.json`。随包切片不要链到 `docs/dev` 或 `packages/app-contracts`。

### 能力状态（代码 / 用户说明）

| 能力 | 代码 | 用户手册 / 学习专栏 |
|------|------|---------------------|
| 导入、启用、卸载、库页改用途 | 已有 | 已写；旧场次绑定不作为当前创作入口 |
| 内置 3 个角色多视角 + 1 个场景四向 + 6 个视频提示词格式 | 已有（builtin 资源） | 已写；视频格式案例目录不收录，导出路径已写 |
| 分类 / 标签（`bgx-categories`） | 补正 PLAN 仍待实施，源码无该键 | 不写成已可用 |
| 用 MCP 完整管理技能库 | 补正 PLAN 仍待实施 | 不写成已可用；Creator MCP 的现有能力以随包手册和实际发布目录为准 |
| H3 兼容 Skill | `project_authored` | 项目自有写法；不包含上游官方 Skill 原文 |

## 3. 导入：预览、确认、冲突与诊断

- 可导入对象：**目录**、**ZIP**、**单个 `SKILL.md`**。选中目录直接含 `SKILL.md` 即包根；否则恰好一个直接子目录含 `SKILL.md` 时取该子目录（单层包装）。ZIP 同理，并忽略 `__MACOSX/`。
- 选中的文件夹（或 ZIP）根上没有 `SKILL.md`、但有多个直接子文件夹各有一份 `SKILL.md` 时，这是技能库根：预览列出每一个，确认后全部导入。不合格的子文件夹单独标出，不挡住其余合格包。需要替换已安装版本的，仍要在这次确认里带上预览看到的指纹，未确认的不覆盖。已经相同的不重写。某个包写入失败时，已经导入成功的保留。
- 先**预览**（不安装）：显示名称、描述、文件数、总大小、格式状态、诊断列表、安装决定。再**确认导入**。预览与导入走同一条校验链，不调用模型，不自动修复任何内容。
- 格式状态：`valid`（可用）、`valid_with_warnings`（可用，警告在库页面可见）、`invalid`（有错误，不安装）。
- 同名处理：同一来源（用户导入）下同名包再次导入是“替换候选”，必须由你显式确认，应用不会自动改名，也不会并存两份。内容完全相同则不做任何改动。用户包与内置包同名时两者都安装，目录里用户包遮蔽内置包，库页面会提示；按 stableId 显式选择仍可选中任何一个。
- 常见诊断（完整列表见 `agent-skills.json`）：
  - YAML 写错（例如描述里有未加引号的冒号）→ 提示给值加引号；应用**不会**自动修复。
  - ZIP 里文件名不是 UTF-8（GBK 旧压缩包）→ 请用 UTF-8 重新打包。
  - 路径含 `..`、盘符、Windows 保留名、符号链接、大小写冲突 → 直接拒绝（Windows 上无法可靠落盘）。
  - 文件数/大小/压缩比超限 → 拒绝，并指出哪一项超限。
- 界面里只会看到人话提示；机码与细节写入日志。

## 4. 创作：写好一份 SKILL.md

1. **description 写“触发条件”**：模型第一层只看到 `name` 和 `description`，写清适用场景与不适用场景。
2. **正文写做法，不写应用内部数据结构**：告诉模型步骤、取舍、检查清单；需要应用工具时用普通语言描述“调用工具前要准备什么、拿到结果后怎么用”。
3. **长内容拆到附件**：正文建议 500 行以内；大段规则、示例、模板放 `references/`，正文里用相对路径引用，模型需要时再按需读取。
4. **不要依赖固定路径**：不要要求某个附件必须存在（缺附件的普通 SKILL 仍应可用）。
5. **输出格式由应用合同决定**：剧集、片段、图片优化等输出结构由应用工具提供并校验；SKILL 里可以写偏好，但不能放宽或绕过应用的校验。
6. **有程序附件要如实说明**：`scripts/` 完整保存、可读，但应用**不执行任意 Python/JS/shell**；目前唯一可由应用执行的程序附件是经独立 worker 运行的 `.wasm` helper（BGX 自有 WIT，只能称“本应用 helper 兼容性”）。需要别的解释器的步骤，请在 `compatibility` 里写明。
7. **给别的 Agent 用**：同一个包可以直接交给其他兼容客户端；不要在正文里写只有本应用才有的工具名，除非明确标注。
8. **保存前自检**：作者档校验（编辑器保存、导出前、开发期 `bgx-agent-skills-verify` 命令）比导入档严格，`name`/`description` 规则的警告会升为错误。
9. **从模板起步**：复制 `templates/agent-skill`（安装版是 `user-examples/agent-skills-authoring/agent-skill`）到可写目录，让 `name` 与目录名一致，改 `description` 和正文，即可导入试用；它只有 `SKILL.md` 和一个可选的 `references/` 示例，缺了附件也能用。字段级规范与可整份复制给其它 LLM 的提示包见 `fixtures/content-skills/local-samples/` 下的 `CONTENT_SKILL`、`AUTHORING_SPEC`、`AUTHORING_PROMPT_PACK`（中英各一份，数字都取自 `agent-skills.json`，由 `scripts/check-content-skill-authoring-docs.ps1` 核对）。

## 5. 在应用里使用：激活、绑定与运行

- 目录与激活：启用且格式有效的包才进入目录；模型据 `description` 决定是否激活，也可以由你在选择器中显式选择。两条路径走**同一个激活服务**。激活后只加载正文和资源清单，附件按需分页读取，并校验 sha256。
- 冻结快照：一次运行冻结所用包的 `packageDigest`、正文哈希和资源清单；运行中包被改动不会静默替换。
- 任务用途/优化绑定：绑定记录保存的是 `stableId + packageDigest`（不是文件路径）。包被替换后，绑定会显示“版本已变化”，需要你显式确认升级；包被卸载后，绑定显示“不可用”，作品内容不会被改动。
- 执行就绪：库页面和激活结果会分别显示格式状态与执行就绪（`ready` / `limited` / `unavailable`）及限制原因。
- 界面状态：设置里的 Agent Skills 库页面（导入、启用、排序、编辑、导出、卸载、旧包升级）与各工作台选择器随 L1/L2 联调；联调未完成前以台账为准，不以本文推断界面细节。

## 6. 编辑、导出与交给其他 Agent

- 编辑：对包内任意文本文件做原文编辑，保存时带 `expectedPackageDigest` 做并发保护，保存前按作者档整包重新校验；校验不过则阻止保存并给出诊断，不静默修正。二进制文件只能替换或删除，不能当文本编辑。
- 导出：可导出为 ZIP 或目录，内容与已安装包逐字节一致（`SKILL.md` 不被改写）。导出的 `packageDigest` 与应用内显示的一致，可用来确认“交给别人的就是这一份”。
- 内置编剧方法也可以下载为标准 SKILL 包；其中的 `LICENSE.txt` 目前是占位文字，权利人确认前不要当作正式授权发布。

## 7. 升级旧版 Content Skill 包（M00）

本节仅保留历史机制的定位，新版创作 Skill 不经过旧包升级、旧场次绑定或旧字段转换。作者直接按本页当前用途和随包 AUTHORING_SPEC / AUTHORING_PROMPT_PACK 制作标准包；不读取、扫描或迁移用户旧数据。本轮不对旧升级工具作功能改造，也不将旧机制视为当前验收前置。

## 8. 回滚与恢复

- 真跑升级时的备份位置：`{应用数据}/content_skills/_migration_backup/<时间戳>/`，内含升级前的 `installed/` 目录副本、`_order.json`（如存在）和 `report.json`（逐包结果，含新的 `stableId` 与 `packageDigest`）。
- 升级不会删除或修改旧原件；是否在 M10 切换时清理旧原件，另行决定并记录，决定前以“原件仍在磁盘”为准。
- 想撤销某个已升级的包：在 Agent Skills 库里卸载对应 `stableId`（见 `report.json`）。卸载只移除标准包，不影响你的作品、媒体和绑定记录（绑定会显示“不可用”）。
- 想整体回退软件版本：必须使用**与软件版本匹配**的备份恢复包目录与注册信息；不能只回滚程序文件并继续读取新版安装记录。
- 备份机制的完整演练（旧安装数据副本升级、失败回滚、卸载重装）**尚未执行**。演练报告 「20261002-REPORT-Agent-Skills升级回滚演练.md」（开源版未收录该内部文档或源码路径） 已写好机制说明、全新安装/升级/回滚的复现步骤和演练矩阵，所有实测栏为 NOT_RUN；在它的矩阵有真实证据之前，本节只描述机制，不构成“已验证可回滚”的承诺。
- 演练只能在旧安装数据的**副本**上做：用环境变量 `BGXIONG_APP_DATA_ROOT` 指向副本目录启动，前后用文件哈希和各表行数对照，证明作品与媒体没有被改动。

## 9. 兼容性与已知限制

| 项目 | 说明 |
|---|---|
| 畸形 YAML | 不自动修复，提示加引号 |
| 作用域 | 只支持内置与用户导入；不支持工作区目录（project）扫描 |
| 非 ASCII `name` | 允许，但记可移植性警告（部分客户端可能不接受） |
| ZIP 文件名编码 | 只接受 UTF-8；GBK/CP437 旧压缩包请重新打包 |
| 脚本执行 | 不执行任意 Python/JS/shell，不自动安装解释器；仅 `.wasm` helper 经独立 worker 运行 |
| 其他客户端 | **尚未实测**。是否可用取决于对方客户端的工具与运行时；在 T07 有证据前不对任何具体客户端或模型作兼容承诺 |
| 多视图出图 | 自然语言到布局经模型解释，图像模型不一定严格遵循；`layout.json` 只提高确定性，仍需人工视觉验收 |

## 10. 发布说明与支持工具矩阵

### 10.1 发布说明（面向用户，可直接用作版本说明）

- **只有一种新协议**：从这个版本起，SKILL 的唯一作者格式是标准 `SKILL.md` 包（Agent Skills 开放规范）。旧版 Content Skill 私有包不能再作为新包导入；它们只能通过一次性升级转成标准包。没有兼容别名，也没有第二套运行路径。
- **你需要做什么**（只在你机器上有旧包时）：设置 → 创作智能体 → Skills 库 → 旧包升级。先点**预检**，读完逐包报告和 `notes`（标准包带不过去的内容都写在这里），确认后再**真跑**；然后做一次**绑定迁移**（同样先预检再真跑）。
- **不会发生什么**：不调用模型；不改你的作品和媒体；不删旧原件（旧包、旧绑定行、旧排序文件都保留）；不自动修复任何内容；预检不写任何东西。
- **备份**：真跑前先备份到 `{应用数据}/content_skills/_migration_backup/<时间戳>/`；备份失败则整体中止。
- **失败时怎么办**：

| 情况 | 会发生什么 | 你该做什么 |
|---|---|---|
| 备份写不进去 | 整体中止，什么都没写 | 腾出磁盘空间或修权限后重试 |
| 某个旧包不合格 | 只有它报告 `failed`，其余照常；原件不改 | 看报告里的原因，修好旧包后重跑（重跑幂等） |
| 升级后名称与另一旧包冲突 | 后者 `failed`，报告写明冲突对象 | 改其中一个再重跑 |
| 缺少包升级的报告就做绑定迁移 | 提示先完成旧包升级，不写任何 pin | 先升级旧包 |
| 绑定迁移写到一半失败 | 本次创建的 pin 被撤销，旧数据未动 | 按提示重试 |
| 想撤销某个已升级的包 | 在 Skills 库按 `stableId` 卸载；作品和媒体不变，相关绑定显示“不可用” | 需要时重新导入或重新绑定 |
| 想整体回到升级前 | 用升级前的数据副本整体恢复，并配套对应版本的程序 | 见第 8 节；不要只回滚程序文件 |

- **已知损耗**：生图策略里每个槽位的提示词正文不被标准路径执行，只取槽位顺序（原文件保留在 `references/legacy/`）；含非机位槽位的策略不生成 `layout.json`；旧 WASM 程序保留在 `scripts/legacy/` 且不被执行。
- **没有验证过的东西（如实写进版本说明，不要省略）**：真实旧安装数据上的升级与回滚演练尚未执行；任何外部 Agent 客户端和任何具体模型上的实测尚未执行。

### 10.2 支持工具矩阵

“已实现”表示代码和自动化测试已落地（测试数量与批次只看执行台账）；“NOT_RUN”表示没有实测证据；本表没有一行可以被当成对具体客户端或模型的兼容承诺。

| 对象 | 能做什么 | 依据 | 状态 |
|---|---|---|---|
| 本应用 · 导入、校验、导出标准包 | 目录、ZIP、单个 `SKILL.md`；预览/确认；完整保存与逐字节导出 | SPEC §3–§8 | 已实现 |
| 本应用 · 目录、激活、按需读取资源 | 同一激活服务；冻结 `packageDigest` 与资源清单；资源分页读取并校验 sha256 | SPEC §9 | 已实现；界面联调细节以台账为准 |
| 本应用 · Markdown 方法 SKILL 的实际生成 | 经应用工具合同执行并校验输出 | SPEC §10 | 真实模型端到端 NOT_RUN |
| 本应用 · `.wasm` helper | 经独立 SKILL worker 运行，只能称“本应用 helper 兼容性” | SPEC §15 | 安装包内放置与冒烟 NOT_RUN |
| 本应用 · `scripts/` 里的 Python、JS、shell | 完整保存与导出，**不执行** | SPEC §12 | 执行就绪显示受限（`helper_language_unsupported`） |
| 外部 Agent 客户端 | 标准包是普通文件夹，可原样复制使用；能否完成任务取决于该客户端的工具与运行时 | 「T07 跨客户端验证」（开源版未收录该内部文档或源码路径） | 环境 NOT_SELECTED；实测 NOT_RUN |
| 具体模型 | 无 | — | 未测，不作承诺 |

## 11. 开发者：合同、命令、测试入口

- 机读合同：`packages/app-contracts/v1/agent-skills.json`（状态集合、诊断码、预算）、`agent-skill-tools.schema.json`（模型工具）、`agent-skill-worker.json`（worker 帧与限额）。TS 经 import、Rust 经 `build.rs` 取值，禁止手写镜像。
- 纯核心：`src-tauri/crates/bgx-agent-skills-core`（解析、验证、指纹，无 IO）；宿主：`src-tauri/src/agent_skills/`（导入/存储/激活/资源读取/绑定/升级）；SKILL worker：`bgx-skill-runtime`、`bgx-skill-transport`、`bgx-skill-tools`（宿主不链接 Wasmtime）。
- IPC（前端封装见 `ui/src/api/methods/apiPart1.ts`）：`preview_agent_skill_package`、`import_agent_skill_package`、`list_agent_skills`、`list_agent_skill_resources`、`read_agent_skill_resource`、`get_agent_skill_edit_view`、`save_agent_skill_edit`、`export_agent_skill_package`、`export_screenwriter_skill_package`、`set_agent_skill_order`、`set_agent_skill_enabled`、`uninstall_agent_skill`、`activate_agent_skill_for_run`、`migrate_legacy_content_skills`、`migrate_legacy_skill_bindings`。
- 验证命令（作者档，开发期）：`cargo run --bin bgx-agent-skills-verify`（见 `scripts/check-agent-skills-format.ps1`）。
- 门禁：`scripts/check-agent-skills-format.ps1`（标准格式与 conformance，同时用核心校验器验 `fixtures/agent-skills`、`templates/agent-skill`、安装目录派生副本和提示包里的 SKILL.md 示例）；`scripts/check-agent-skill-hosts-fixtures.ps1`（发布用样例与模板的合法 `bgx-hosts`、同名包用途对齐、随包作者文档不断链、H3 案例再分发拦截）；`scripts/check-content-skills-ssot.ps1`（M10 前验旧协议面与应用内部合同，M10 后验退役状态；始终验标准包来源、同步副本哈希、安装资源通配与新 core/host/worker/UI 层）；`scripts/check-content-skill-authoring-docs.ps1`（本指南、作者文档中英一致与数字取自合同）；`scripts/check-content-skill-template.ps1`（起步模板结构）；`scripts/check-app-contracts-ssot.ps1`（三份机读合同的生成与消费）。
- 随包样例的单一来源：`fixtures/agent-skills` → `installer-user-examples/agent-skills`，作者文档与 `templates/agent-skill` → `installer-user-examples/agent-skills-authoring`，都由 `scripts/sync-user-example-packs.ps1` 生成（gitignored 派生副本，不手改）；安装包经 `tauri.conf.json` 的 `../installer-user-examples/**/*` 带出，不再有旧作者包切片。
- 约束：不得用 `skill.manifest.json` 或旧 capabilities 过滤、拒绝标准包；应用内部的实际输出 schema、资源引用、权限、事务及冻结合同与作者包格式分开维护；历史 Plan/ContentIntent/场次绑定不能作为新包必填内容。
- 新增诊断码、状态或预算：先改 SPEC 与 `agent-skills.json`，再改实现，三者同批。

## 12. 变更记录

- v0.1（2026-10-03，批 B · L7）：首版。覆盖标准包、导入、创作、使用、编辑导出、旧包升级与回滚机制、限制；演练与跨客户端实测均为待做。
- v0.2（2026-10-03，N9 · M11/M13）：第 4 节补起步模板与作者文档入口；第 7 节补旧绑定与排序迁移（M01）的行为；第 8 节指向升级回滚演练报告（全部 NOT_RUN）；新增第 10 节发布说明与支持工具矩阵，开发者一节顺延为第 11 节并补门禁与随包样例来源。

## 视频提示词格式方法（VP-VS，2026-10-09）

**内容整理与格式适配如何分工（2026-10-20）**：见 「通用设计与智能体编排 GUIDE」（开源版未收录该内部文档或源码路径）。两个执行职责有独立意义，但不要求作者制作两个包；支持单用途、两个独立包和一个双用途包。该指南明确用途覆盖、阶段合同、事实扩写边界、资源读取及失败处理。尚未实现的候选展示、识别来源和负面结果交付依「通用实施计划」（开源版未收录该内部文档或源码路径），不得当成已发布功能。

格式方法是标准Skill包：SKILL.md及可选references，库保留原包字节，导出不包装成私有格式。格式有效与应用兼容绑定是两个状态；绑定存应用库，原包不注入projection配置。派生/升级后展示拟议重绑定，再由用户按既有编辑发布流程确认。

选择器可选当前目标默认方法或自选stableId+packageDigest。打开和查询只读；用户点击生成/优化时，共享VideoPromptService冻结目标、文本模型、格式/内容方法及有限资源子集。包缺失、digest变化、绑定不兼容或资源缺失显式失败，不换模型/包。

`project_existing`只换已审母版的目标格式，必须带Rust返回的canonicalDocumentRef与精确正文指纹；不重新生成母版。源稿或语义依赖变更失效，方法/目标格式变更仅重新投影。代码只校验结构、闭集ID及数值，内容阻断由评审模型的结构化blocking结论决定，minor只记录。

GUI/F1/MCP同一能力目录、参数schema、任务及投影回执。回执含母版来源、冻结方法、资源证据、目标快照与projectionFingerprint；canonical ref来自真实提交。正文、母版、投影marker、canonical effect与请求证据在同一CAS事务落盘；冲突返回候选，保留用户内容和已完成兄弟项。已确认正文在预览/提交时逐字直通，标签展示只读已编译inventoryLabels。

离线执行状态以 「VS85报告」（开源版未收录该内部文档或源码路径） 为准。实模首响/零修复轮、GUI 跨窗、安装包和成片仍须分别验收；当前未执行部分不得宣称完成。H3 内置包现为项目自有的 `h3@1` 兼容 Skill，包名与默认绑定不变；来源边界见 「发布结论」（开源版未收录该内部文档或源码路径）。

## 技能管理（2026-10-13 更新）
- 卡片只保留 **编辑 / 停用(启用) / 卸载**；拖动卡片空白处排序；分类下一行显示 `版本 x.y.z`。
- 分类筛选可多选，含“全部”“未分类”；随发布技能已按 `metadata.bgx-categories` 预设分类。
- 点“编辑”打开统一工作台：基本信息（名称/分类/标签/用途/版本方式）、技能内容（草稿）、版本记录（使用此版本）。底部一个“保存”：只改属性不升版本；改了正文按“自动(补丁号+1)/手动”写 `metadata.version` 发布；正文与属性同时修改时一次提交；无改动提示“没有需要保存的修改”。
- 未标版本的旧包首次发布为 1.0.1；手动版本须为更高的标准 SemVer。

本专题最新决定：默认保真，任一所选Skill明确允许即可扩写（包括格式适配），不设额外程序开关；新增事实须与母版/审阅/结果一致。本次开发版已接入可选metadata.bgx-expansion（allow/preserve）、adapt_expand与promptChannels；旧发布版先查询其实际schema。模型与安装验收不由源码接入替代。

## 轻量摄影风格样例维护（2026-10-24）

六个 `large-format-film-{general|portrait|large-scene}-{image|video}` 标准包由 `fixtures/photography-style-authoring/source.json` 共源维护；运行 `python scripts/generate-photography-style-skills.py` 生成 `fixtures/agent-skills/` 固定六包并导出 ZIP，`--export-dir` 可指定独立目录，`--check` 只核生成漂移、不写文件。图片单用途 `image-photography-style`（图片版本由 `imageVersion` 管理），视频单用途 `video-photography-style`（版本由 `version` 管理），附件仅含 `visualStyleOverlay`。视频使用共同核心加短时序句；不再使用旧故事板用途或把图片摄影包混入优化助手。

先修改作者源，再生成六包，执行既有格式、用途、content-skills 和作者文档门禁；最后用 `sync-user-example-packs.ps1` 派生安装样例。不要手改派生目录或复用其它包的稳定 ID。独立 ZIP 位于 `dist/skill-packages/large-format-film-image-<imageVersion>/`，使用同目录回执核对 SHA256。用户导入的旧版本不被自动覆写，应重新导入新版；已有合法风格包也可由用户在技能库明确设置图片摄影用途。

这六包于当前 EXE 完成制品验真后才加入源码，未注册内置，也不在该已构建 EXE 内；本轮以 ZIP 单独导入产品 Skills 库，实际导入/模型效果尚未验。图片摄影质感避免混入视频时序指令；视频不改用户帧率或运动，不新增表情动作。保留用户原文和对白，不以截断或关键词判定质量。参考图自身已蜡质时文字改善可能有限，应在用户选择后做同参数 A/B；本轮没有付费 AI 调用。

开发目录的唯一治理规范见 「三仓平级指南」（开源版未收录该内部文档或源码路径）。
