> **历史演进产物，不代表当前设计，不作为当前实现或开发依据。**
>
> - 原路径：`release/04-开发规范/20261002-SPEC-Agent-Skills包加载与执行合同.md`
> - 文档标识：取自原文件名 `20261002-SPEC-Agent-Skills包加载与执行合同.md`（这是原文件名中的标识，不是本次整理推定的写作或生效日期）
>
> 现行架构请从仓库主导航进入查阅；本文只作为历史演进资料，不作为当前实现或开发依据。

# SPEC · Agent Skills 包加载与执行合同

> **开源整理版**（2026-11-12）：已去除或改写私有路径、未公开端点、凭证示例，以及登录 / 认证 / 付费相关实现细节。若某节全部属于上述范围，将改为一句能力边界说明。


> **2026-10-07管理补正接手提示**：「联审REPORT」（开源版未收录该内部文档或源码路径）及「补正PLAN」（开源版未收录该内部文档或源码路径）定义后续分类/草稿/版本/公共MCP管理设计；尚未变成已实现合同。SG01按同批更新本SPEC与机读合同，SG19回写as-built；当前字段、格式校验和已冻结运行行为不得因本提示被假称已升级。

- 文档时间：2026-10-02（Asia/Shanghai）；版本：v1.3（K01 冻结；P00 补 §4.4 YAML 库与测例；R07 补 §14.1 应用数据表；W00 填写 §15 SKILL worker 合同）。其他章节修改须先改本 SPEC 并在执行台账登记。
- 权威：「100 项文件级原子开发计划」（开源版未收录该内部文档或源码路径） §3；本 SPEC 是它的可编码展开。机读合同：`packages/app-contracts/v1/agent-skills.json`、`packages/app-contracts/v1/agent-skill-tools.schema.json`（K02）、`packages/app-contracts/v1/agent-skill-worker.json`（W00）。诊断码、状态集合、预算数值以 K02 JSON 为机读单源；本 SPEC 与 JSON 不一致时视为缺陷，须同批修正，不得各自实现。
- 状态：K01 冻结。实现任务 P00–P07、R00–R09、L00–L06、W00–W12 按本 SPEC 编码；本 SPEC 不代表任何生产代码已实施。

## 0. 2026-10-25 创作包发布边界

本轮更新创作 Skill、作者规范和旧包执行代际处理；标准包格式、权限、冻结资源读取协议继续遵守现有合同。执行代际以 `agent-skills.json` 的 `skillMetadataExtensions.workflow` 为唯一机读来源。当前包声明 `metadata.bgx-workflow: clips-v1`；缺失或不匹配不判格式损坏，但新执行为 unavailable，库卡片显示原因。安装包覆盖官方随包文件并刷新官方缓存，外部旧包不转换、不删除，旧冻结资源保留。库卡片执行诊断字段见 `agent-skill-management-tools.schema.json`。创作用途与内容遵守[当前作者规则](../03-开发指南/20261002-GUIDE-Agent-Skills导入创作与开发.md#当前作者与开发智能体必须遵守)及「片段主计划」（开源版未收录该内部文档或源码路径）。旧场次、旧故事板与旧编排用途不再用于新发行包；历史运行器 ID 留在 registry 不代表仍是作者可用映射。新包不提供旧字段、私有步骤工作流或旧 schema 的兼容资源。输出由当前宿主提供 schema，创作方法不得声明尚未发布的工具或绕过用户媒体确认。下文关于旧包迁移和场次绑定的既有实现描述不构成当前创作流程要求。

## 1. 范围与规范来源

- **2026-10-20 双插槽解释补充**：「内容优化与提示词格式通用 GUIDE」（开源版未收录该内部文档或源码路径）解释两个阶段与技能包的关系。一个包可声明一个或两个有效用途；用途不授予目标格式兼容性，用户用途覆盖仍由现有 host 合同决定。执行时各阶段使用宿主下发的结构化合同，独立聊天的终稿约定不能覆盖它。候选状态、识别来源及负面结果等后续扩展见「计划」（开源版未收录该内部文档或源码路径），本说明不新增任何运行时字段或私有包格式。

- 作者格式采用 Agent Skills 开放规范，2026-10-02 联网逐条核验：
  - [Agent Skills Specification](https://agentskills.io/specification)
  - [How to add skills support to your agent](https://agentskills.io/client-implementation/adding-skills-support)
- 开放规范只定义包目录、`SKILL.md` frontmatter/正文、可选目录约定、渐进加载与文件引用；**未定义**分镜 schema、故事数据库、生图 API、脚本运行时、WASM ABI、安装目录或应用 IPC。
- 本 SPEC 定义本应用（比格熊数字导演工作站）如何**导入、校验、保存、导出、发现、激活、读取资源、判定可执行性**。其中的诊断码、SkillRef、packageDigest、工具 IPC 都是**本应用合同**，不得对外标称为行业标准。
- 一份标准包应能原样交给其他兼容客户端使用；能否在某客户端完成任务取决于该客户端的工具与运行时（计划 §1）。

## 2. 术语

| 术语 | 含义 |
|---|---|
| 包（skill package） | 一个包根目录及其下全部文件；最少只有 `SKILL.md`。 |
| 包根 | 直接包含 `SKILL.md`（文件名精确匹配，区分大小写）的目录。 |
| 容器 | 用户提交的导入对象：目录、ZIP 或单个 `SKILL.md` 文件。 |
| 容器名 | 用于“name 与父目录一致”校验的目录名：包装目录名 > ZIP 文件名去扩展名 > 选中目录名 > 单文件所在目录名。 |
| frontmatter | `SKILL.md` 开头两条 `---` 定界行之间的 YAML 映射。 |
| 正文 | 结束定界行之后的全部内容。 |
| 资源 | 包内除 `SKILL.md` 外的任意文件（`scripts/`、`references/`、`assets/` 只是约定，不是白名单）。 |
| stableId | 本应用给已安装包的稳定 ID：`<source>:<name>`，见 §8。模型、绑定、工具调用只用它定位包。 |
| packageDigest | 整包内容指纹 `sha256:<64hex>`，见 §8。 |
| 激活 | 把某包的正文与资源清单载入一次会话/运行，并冻结快照，见 §9。 |
| 格式状态 | 包是否符合本 SPEC 的格式规则（`formatStatus`）。 |
| 执行就绪 | 当前环境是否具备包所需工具/运行时（`executionReadiness`）。与格式状态**相互独立**，见 §10。 |

## 3. 包结构与容器识别

### 3.1 容器规则

1. **目录**：选中目录直接含 `SKILL.md` → 它是包根；否则若**恰好一个**直接子目录含 `SKILL.md` → 该子目录是包根（单层包装）；多于一个 → `AGENT_SKILL_MULTIPLE_ROOTS`；没有 → `AGENT_SKILL_SKILL_MD_MISSING`。不向更深层递归搜索。
2. **ZIP**：先忽略顶层 `__MACOSX/` 整棵子树（记 `AGENT_SKILL_MACOSX_IGNORED`，info）；然后归档根含 `SKILL.md` → 根是包根；否则若全部条目都位于**唯一**顶层目录下且该目录含 `SKILL.md` → 该目录是包根；其余情况同目录规则报错。只有大小写变体（如 `skill.md`）不算找到。
3. **单个 `SKILL.md` 文件**：包 = 仅该文件；容器名取其所在目录名。
4. **技能库根**（导入入口，不改变上面的单包规则）：选中的目录或 ZIP 根上没有 `SKILL.md`，且有两个及以上直接子目录含 `SKILL.md` 时，每个这样的子目录是一个包，一次预览、一次导入。不向更深层递归。把这个根直接交给单包读取仍是 `AGENT_SKILL_MULTIPLE_ROOTS`（N03）。零个或恰好一个子目录仍按第 1、2 条处理。
5. 包根下**全部**文件和子目录都属于包，完整读取、完整保存、完整导出；不存在“只提取固定文件”的路径（替代 `content_skill_host/archive.rs` 的固定文件提取）。
6. 不要求、也不读取 `skill.manifest.json`、`pipeline.json`、`generation-policy.json`、`outputSchema` 或任何 WASM；这些文件若存在，只作为普通资源原样保存。

### 3.2 路径与安全（两档均为 error）

| 规则 | 诊断码 |
|---|---|
| 绝对路径、盘符（`C:`）、UNC、`..` 或 `.` 段、空段、NUL/控制字符 | `AGENT_SKILL_PATH_UNSAFE` |
| Windows 保留名（CON、PRN、AUX、NUL、COM1–9、LPT1–9，含带扩展名形式）、以点或空格结尾的段 | `AGENT_SKILL_PATH_RESERVED_NAME` |
| 同一包内路径按 Unicode NFC + 大小写折叠后冲突 | `AGENT_SKILL_PATH_CASE_COLLISION` |
| 符号链接、Windows 重解析点/联接、硬链接、ZIP 中的 symlink 条目 | `AGENT_SKILL_SYMLINK_FORBIDDEN` |
| ZIP 内重复条目名 | `AGENT_SKILL_DUPLICATE_ENTRY` |
| ZIP 条目名既无 UTF-8 标志、又不是合法 UTF-8（如 GBK/CP437 旧归档）；不猜代码页 | `AGENT_SKILL_PATH_ENCODING_UNSUPPORTED` |
| ZIP 损坏、加密条目、不支持的压缩方法 | `AGENT_SKILL_ARCHIVE_CORRUPT` |
| 文件数、目录深度、单路径长度、单文件/总解压/归档大小、压缩比、`SKILL.md`/frontmatter 大小超过 K02 预算 | `AGENT_SKILL_LIMIT_EXCEEDED`（`details.limitKey` 指明哪一项） |

- ZIP 条目名中的 `\` 视为目录分隔符并规范成 `/`，记 `AGENT_SKILL_ZIP_BACKSLASH_PATH`（warning）。原因：Windows PowerShell 5.1 `Compress-Archive` 生成的归档使用 `\`，直接拒绝会误伤常见用户包。
- 路径保存为解码后的 UTF-8 字符串，不做 Unicode 规范化改名；NFC+大小写折叠只用于冲突检测。
- 所有文件 IO 经 `platform::fs`（计划 §3.3）；纯核心只接收字节和路径值。

## 4. SKILL.md 解析

1. 编码：必须是合法 UTF-8，否则 `AGENT_SKILL_SKILL_MD_NOT_UTF8`。允许 UTF-8 BOM（仅解析时跳过）；LF 与 CRLF 均接受。**原始字节原样保存**，解析结果只是索引，任何流程都不改写 `SKILL.md`。
2. 开始定界：首行（BOM 之后）必须是 `---`（允许行尾空白），否则 `AGENT_SKILL_FRONTMATTER_MISSING`。
3. 结束定界：之后第一条内容为 `---`（允许行尾空白）的行；找不到 → `AGENT_SKILL_FRONTMATTER_UNCLOSED`。不接受 `...` 作结束符。
4. YAML：
   - 解析失败 → `AGENT_SKILL_FRONTMATTER_YAML_INVALID`；顶层不是映射 → `AGENT_SKILL_FRONTMATTER_NOT_MAPPING`；空 frontmatter 视为空映射（随后报 name/description 缺失）。
   - 重复键 → `AGENT_SKILL_FRONTMATTER_DUPLICATE_KEY`；锚点/别名/合并键 → `AGENT_SKILL_FRONTMATTER_ALIAS_FORBIDDEN`（防膨胀攻击）；非核心 schema 标签视为 YAML 无效。
   - YAML 库（P00 固定）：`yaml-rust2` **=0.13.0**（crates.io，MIT/Apache-2.0；`default-features = false`，不启用 `encoding`）。只通过 `parser::Parser::next_token` 拉取事件；**不得**使用 `Parser::load`/`YamlLoader`：其递归下降在约 5000 层块嵌套（约 10 KB 输入）时栈溢出，会让整个进程崩溃（P00 box 实测）。`serde_yaml` 已归档停止维护，不采用。
   - 核心按事件自行判定：重复键（任意层级，按解析后的键值比较）；锚点、别名、合并键 `<<` → `ALIAS_FORBIDDEN`；纯量按 YAML 1.2 core schema 判型（仅 plain 纯量判 null/bool/int/float，引号与块纯量一律字符串）；只允许核心标签 `!!str`/`!!int`/`!!float`/`!!bool`/`!!null`/`!!seq`/`!!map` 与非特定标签 `!`，其余标签或标签与值不符 → `YAML_INVALID`；一个 frontmatter 只允许一个 YAML 文档（`...` 后再有内容 → `YAML_INVALID`）；集合作为键 → `YAML_INVALID`；流式嵌套超过库上限（255 层）→ `YAML_INVALID`。解析树为数组 arena，构建与释放均不递归。
   - YAML 测例：`tests/yaml_library.rs`（P00，固定库行为：重复键不被库合并、锚点/别名可见、`!!str`/`!foo` 标签解析、`description: Use when: x` 解析失败、plain/引号/折叠纯量可区分、`...` 产生第二文档、30000 层块嵌套不溢出、1000 层流式嵌套报错、空/纯注释流无文档）；`tests/frontmatter.rs`（P01，固定核心判定）。升级 YAML 库须先改本节并通过两组测例。
5. **不自动修复畸形 YAML**。官方客户端指南建议可选地为“含冒号的未加引号值”加引号后重试；本应用不做：修复会让同一字节在不同客户端得到不同含义，且违反“禁止额外自动修复”（计划 §3.2-6）。这类包报 `AGENT_SKILL_FRONTMATTER_YAML_INVALID`，提示作者加引号。列为已知兼容限制（§12），用户可另行决定是否开启。
6. 正文：结束定界行之后的全部内容。保存不变；交给模型时去掉开头空行和末尾空白（与官方指南“trimmed”一致）。

## 5. Frontmatter 字段逐项对照

两档严格度：**import**（导入、安装、列表、激活，宽松）与 **author**（编辑器保存、导出前检查、P06 验证命令、仓库发布 fixture，严格）。见 §7。

| 字段 | 官方约束（2026-10-02 原文要点） | import | author | 本应用用途 |
|---|---|---|---|---|
| `name` | 必填；1–64 字符；只含 Unicode 小写字母数字（a-z、0-9）和 `-`；不以 `-` 开头/结尾；无 `--`；须与父目录名一致 | 缺失/空/非字符串 → error（`NAME_MISSING`/`NAME_NOT_STRING`）；含路径分隔符、控制字符、`.`/`..` 或超过硬上限 128 → error（`NAME_UNSAFE`）；超 64 → warning（`NAME_TOO_LONG`）；字符集不符 → warning（`NAME_CHARSET`）；连字符规则不符 → warning（`NAME_HYPHEN`）；与容器名不一致 → warning（`NAME_DIR_MISMATCH`） | 上述 warning 全部升为 error | 显示名；stableId 的名称部分；目录 enum 的值。字符判定与参考校验器 `skills-ref` 一致：NFKC 后每个字符为 Unicode 字母/数字且等于其小写形式，或为 `-`。非 ASCII 名称两档都另记 `AGENT_SKILL_NAME_NON_ASCII`（warning，跨客户端可移植性提示） |
| `description` | 必填；1–1024 字符；非空 | 缺失、空或全空白 → error（`DESCRIPTION_MISSING`，与官方“缺 description 跳过”一致）；非字符串 → error（`DESCRIPTION_NOT_STRING`）；超 1024 → warning（`DESCRIPTION_TOO_LONG`），目录展示截断到 1024 并标注 | 超长 → error | 目录（第 1 层）唯一的语义描述；模型据此决定是否激活 |
| `license` | 可选；许可证名或包内许可文件名 | 非字符串 → warning（`LICENSE_NOT_STRING`），保留原文 | error | 仅展示与导出；不校验许可证文本 |
| `compatibility` | 可选；提供时 1–500 字符；环境要求 | 非字符串、空或超 500 → warning（`COMPATIBILITY_INVALID`），保留原文 | error | 原文给模型和用户看；**不解析、不正则猜依赖**；执行就绪由实际工具/运行时协商（§10） |
| `metadata` | 可选；字符串到字符串的映射；键名建议唯一 | 非映射或含非字符串值 → warning（`METADATA_INVALID`），原文保留，解析视图只暴露字符串值 | error | 原样保留；**不是安装必填**；`metadata.bgx` 等任何键都不被要求。本应用工作台可见性看 `metadata.bgx-hosts`（空格分隔字符串，闭集见 `agent-skill-hosts.json`）：有效列表为空则不出现在任何工作台选择器。该键不是安装格式必填，但是工作台选用的硬约束 |
| `allowed-tools` | 可选；空格分隔字符串；实验性 | 非字符串 → warning（`ALLOWED_TOOLS_INVALID`） | error | 只作“声明”展示与就绪提示；**绝不自动授予任何工具权限**（L05） |
| 其他未知字段 | 规范未定义（参考校验器视为错误） | warning（`UNKNOWN_FIELD`），原文保留 | error | 不解释、不丢弃 |

- 字符数按 YAML 解析后的 Unicode 标量值计。
- 诊断码全名均加前缀 `AGENT_SKILL_`（例：`AGENT_SKILL_NAME_CHARSET`），完整列表与各档严重度见 K02 `agent-skills.json` 的 `diagnostics`。
- 不从旧 `local.` 命名空间或旧 manifest 推导标准 `name`（计划 §3.1）；旧包升级由 M00 显式转换并经 author 档验证。

## 6. 正文与附件

1. 正文格式不受限制；不按 Markdown 标题、关键词或正则推导 capabilities、工作台适用性或输出合同（计划 §3.2-5）。
2. 资源完整保存、按需读取；激活时只列清单不预读（§9）。
3. 正文 > 500 行记 `AGENT_SKILL_BODY_LARGE`（warning，作者提示，对应官方“< 500 行、< 5000 token 推荐”）；空正文记 `AGENT_SKILL_BODY_EMPTY`（warning）。二者都不阻止导入或激活。
4. 文件引用：使用相对包根的路径，建议一层深。正文中指向包内不存在文件的相对链接，P07 可记 `AGENT_SKILL_REFERENCE_MISSING`（warning，仅提示，不阻断，不据此改写正文）。
5. 二进制资源（图片、`.wasm` 等）原样保存与导出；读取接口只返回元数据与 hash，不把二进制塞进模型上下文。

## 7. 两档严格度与诊断模型

- 每条诊断：`code`、`severity`（`error`/`warning`/`info`）、`phase`（`container`/`parse`/`validate`/`install`/`catalog`/`activate`/`read`/`execute`）、`message`（人话，可本地化）、可选 `path`（包内相对路径）与 `details`。
- **import 档**：用于导入、安装、注册列表、目录、激活。有任何 error → 不安装（`formatStatus=invalid`）；有 warning 无 error → 安装，`formatStatus=valid_with_warnings`；只有 info 或无诊断 → `formatStatus=valid`。warning 诊断在库页面可见（U01/U06）。
- **author 档**：用于编辑器保存（R05/U02）、导出前检查（R04/U05）、`skills validate` 类命令（P06）、仓库内发布用 fixture（M02–M09、M12 门禁）。官方规范的每条“必须”都是 error。编辑器保存若 author 档不通过：阻止保存并显示诊断，不静默修正。
- 同一错误只有一个码、一个含义；同一码在两档的严重度由 K02 `diagnostics[].severity.{import,author}` 唯一给出。
- 用户可见文案走 i18n 人话；机码与细节只进日志（AGENTS「INV-TOAST-USER」）。

## 8. 包身份与指纹

### 8.1 stableId 与来源

- `source` 闭集：`builtin`（随应用发布）、`user`（用户导入）。当前产品**不支持** project 作用域（工作区目录扫描）；不虚构该来源（R02）。
- `stableId = "<source>:<name>"`，`name` 为 frontmatter 原值（import 档允许的宽松名称）。同一 `source` 下同名再次导入 = 替换候选，须用户显式确认（P05/R01），不自动改名、不并存。
- 不同来源同名：两者都安装；模型目录中 `user` 遮蔽 `builtin`（与官方“更具体作用域优先”一致），并对用户显示“已遮蔽内置同名 SKILL”提示；用户在选择器中按 stableId 显式选择仍可选任何一个（U03）。
- 模型、工具、绑定、运行快照只使用 stableId 与 packageDigest，**不向模型暴露文件系统路径**（官方指南允许省略 location；本应用激活结果以资源清单替代路径）。

### 8.2 packageDigest v1

```text
lines  = for each regular file f in package (sorted by relpath UTF-8 bytes ascending):
           lowercase_hex(sha256(bytes(f))) + "  " + relpath(f) + "\n"
digest = "sha256:" + lowercase_hex(sha256("bgx-agent-skill-digest-v1\n" + concat(lines)))
```

- `relpath`：相对包根、`/` 分隔、§3 规范化之后的路径（包装目录与 `__MACOSX/` 已去除）。
- 只计普通文件内容与路径；不计空目录、mtime、权限位、ZIP 注释/压缩参数、宿主注册信息或快照。
- 黄金向量（P04 必须复现）：
  - 仅 `SKILL.md`，内容为 `---\nname: hello-skill\ndescription: Say hello in the user's language. Use when the user greets you.\n---\n\n# Hello\n\nReply with a short greeting.\n`（LF、无 BOM）→ `sha256:7dd89a71d2fbb598e4fc8541214b83d252d7125dfb527307847ebd1e46d016cb`。
  - 再加 `references/tone.md` 内容 `Keep it warm and brief.\n` → `sha256:00ce28a71e699cd068e0415e485ef93bd9ab9dc45d21152f2fd6bcc99f8b8ab2`。
- 编辑保存使用 `expectedPackageDigest` 做 CAS（R05）；运行冻结 digest，运行中包被修改不静默替换（计划 §3.2-4）。

## 9. 发现、目录、激活与资源读取

1. **目录（第 1 层）**：每个启用且格式有效的包只提供 `skillId`（= stableId）、`name`、`description`。禁用、被权限拒绝、格式无效的包**不出现在目录中**（不列出后再在激活时拒绝）。没有可用包时不提供目录，也不注册 SKILL 工具。
2. **激活（第 2 层）**：`skills.activate` 的 `skillId` 参数在运行时收窄为当前目录的枚举（L02/L03），模型不能编造 ID。用户显式选择（选择器/绑定）与模型激活走**同一个激活服务**（计划 §3.2-3）。激活结果：正文（去掉 frontmatter）、资源清单（不读内容）、`compatibility` 原文、`allowed-tools` 声明、执行就绪与限制（§10）、`activationId`。正文超过 K02 `activationBodyMaxChars` 时截断并标 `bodyTruncated`，剩余部分用 `skills.read_resource` 读取 `SKILL.md`。
3. **快照**：激活冻结 `packageDigest`、正文 sha256、资源清单（路径/大小/sha256/文本或二进制）、`source`、`activationSource`（`user`/`model`/`binding`）。同一运行重复激活同一 skillId 返回同一 activationId，不重复注入（官方“去重”）。单次运行激活数受 K02 `maxActivationsPerRun` 限制。
4. **资源读取（第 3 层）**：`skills.list_resources` 分页列清单；`skills.read_resource` 只接受该激活快照中的 `resourceRef`，按字符 offset 分页读取文本；读取时核对 sha256，与快照不符 → `AGENT_SKILL_RESOURCE_HASH_MISMATCH`，不返回新内容。
5. **上下文管理**：激活内容带可识别包装，压缩/摘要时受保护；压缩或续跑后按 packageDigest 重载（L06）。
6. 工作台、助手、绑定的运行都只读激活快照，不再读取旧 JSON 包字段（计划 §3.2-3）。

## 10. 可执行能力与限制

1. `formatStatus`（格式）与 `executionReadiness`（环境）分开记录、分开展示：格式有效但缺工具/运行时的包**照常导入**，执行时显示明确限制；不得静默换模型、换工具、跳过 helper 或报假成功（计划 §3.2-7）。
2. `executionReadiness` 闭集：`ready`（所需能力均可用）、`limited`（可执行但部分声明的能力不可用，限制已列出）、`unavailable`（当前环境无法执行该包要求的步骤）。纯文本方法包在有可用模型时为 `ready`。
3. `limitationKind` 闭集：`tool_unavailable`、`runtime_unavailable`、`helper_language_unsupported`、`permission_not_granted`。判定依据是实际工具注册表和运行时探测，不是对 `compatibility` 文本的正则（L05）。
4. `allowed-tools` 只是声明：可用于提示“该包期望的工具”，**不授予权限**；越权调用照常走应用权限判定（`AGENT_SKILL_PERMISSION_DENIED`）。
5. `scripts/` 等程序资源完整保存、可读，但本应用**不执行任意 Python/JS/shell**，不自动安装解释器（计划 §10）。唯一可执行的程序附件是经 W 批次独立 worker 运行的 `.wasm` helper（现有 BGX WIT，标注为本应用 helper 兼容性，不称通用 ABI）；宿主进程不链接 Wasmtime。
6. 落库、媒体生成仍走既有 prepare/accept/apply、Creator Kernel 与 `ai_tasks`；SKILL 不直接访问数据库或凭据（计划 §3.2-6/8/10）。

## 11. 明确不要求的 BGX 扩展

以下都**不是**导入、安装、激活或执行的前置条件；缺少它们不得产生 error：

- `skill.manifest.json`、`pipeline.json`、`generation-policy.json`、`outputSchema`、旧 `capabilities` 列表（`content-skill-capabilities.json` 在 M10 退役）；
- `metadata.bgx` 或任何 metadata 键；`local.` 命名空间；
- WASM component、`content-skill-v1` 协议版本、TypeScript SDK、`buildPlan`、BGX WIT；
- 任何固定资源路径（如必须存在 `references/layout.json`）。

结构化资源（如 `references/layout.json`、`references/output.schema.json`、`references/workflow.json`）是**可选**的：存在时由对应应用工具读取并按应用合同校验，不存在时应用提供默认输出合同（C00）。旧 `content-skill-v1` 包只通过 M00 一次性升级工具读取，不再作为新安装格式（M10）。

## 12. 已知兼容性限制

| 限制 | 处理 | 可否由用户改变 |
|---|---|---|
| 畸形 YAML（未加引号的冒号值等）不自动修复 | `FRONTMATTER_YAML_INVALID`，提示加引号 | 可；需另立任务评审“修复后重试”方案 |
| 不支持 project 作用域（工作区 `.agents/skills/` 扫描） | 只有 builtin/user | 可；需新任务与信任门禁 |
| 非 ASCII `name` | 允许（与参考校验器一致），记可移植性 warning | — |
| GBK/CP437 编码的 ZIP 文件名 | `PATH_ENCODING_UNSUPPORTED`，提示用 UTF-8 重新打包 | — |
| Windows 保留名、大小写冲突路径 | error（Windows 上无法可靠落盘） | — |
| 任意脚本执行 | 不支持；仅 W worker 的 `.wasm` helper | 可；未来执行器作为同一资源工具扩展（W12） |

## 13. 正反例覆盖矩阵

“落地”列为实现该用例测试的任务与 fixture 目录（`tests/fixtures/agent-skills/` 下，按用例 ID 命名）。

| ID | 输入 | import | author | 诊断码 | 落地 |
|---|---|---|---|---|---|
| P01 | 仅 `SKILL.md`，name/description 合规 | valid | valid | — | P01/P06/R00 |
| P02 | 加全部可选字段（license、compatibility、metadata 字符串映射、allowed-tools） | valid | valid | — | P01 |
| P03 | 含 scripts/references/assets 与自定义目录、二进制文件 | valid，全量保存 | valid | — | R00/R01/R04 |
| P04 | ZIP 单层包装目录 | valid | valid | — | R00 |
| P05 | ZIP 带顶层 `__MACOSX/` | valid | valid | `MACOSX_IGNORED`(info) | R00 |
| P06 | UTF-8 BOM + CRLF | valid，字节不变 | valid | — | P01/R04 |
| P07 | description 多行折叠标量 | valid | valid | — | P01 |
| P08 | 无 manifest/pipeline/outputSchema/WASM/metadata | valid | valid | — | P01/T00 |
| P09 | 中文 name（Unicode 小写字母数字） | valid_with_warnings | valid_with_warnings | `NAME_NON_ASCII`(warning) | P02 |
| P10 | 正文 > 500 行 | valid_with_warnings | valid_with_warnings | `BODY_LARGE`(warning) | P01 |
| P11 | ZIP 条目使用 `\` 分隔 | valid_with_warnings | —（导出不产生） | `ZIP_BACKSLASH_PATH`(warning) | R00 |
| W01 | name 含大写 | valid_with_warnings | invalid | `NAME_CHARSET` | P02 |
| W02 | name 65 字符 | valid_with_warnings | invalid | `NAME_TOO_LONG` | P02 |
| W03 | name 以 `-` 开头/含 `--` | valid_with_warnings | invalid | `NAME_HYPHEN` | P02 |
| W04 | name 与容器名不一致 | valid_with_warnings | invalid | `NAME_DIR_MISMATCH` | P02 |
| W05 | description 1025 字符 | valid_with_warnings | invalid | `DESCRIPTION_TOO_LONG` | P01 |
| W06 | compatibility 501 字符或空串 | valid_with_warnings | invalid | `COMPATIBILITY_INVALID` | P01 |
| W07 | metadata 值为数字/列表 | valid_with_warnings | invalid | `METADATA_INVALID` | P01 |
| W08 | allowed-tools 为列表 | valid_with_warnings | invalid | `ALLOWED_TOOLS_INVALID` | P01 |
| W09 | 未知字段（如 `version`、`disable-model-invocation`） | valid_with_warnings，原文保留 | invalid | `UNKNOWN_FIELD` | P01 |
| W10 | license 为数字 | valid_with_warnings | invalid | `LICENSE_NOT_STRING` | P01 |
| N01 | 无 `SKILL.md`（只有 skill.manifest.json） | invalid | invalid | `SKILL_MD_MISSING` | R00 |
| N02 | 只有小写 `skill.md` | invalid | invalid | `SKILL_MD_MISSING` | R00 |
| N03 | ZIP 两个顶层目录各有 `SKILL.md` | invalid | invalid | `MULTIPLE_ROOTS` | R00 |
| N04 | 首行不是 `---` | invalid | invalid | `FRONTMATTER_MISSING` | P01 |
| N05 | 无结束 `---` | invalid | invalid | `FRONTMATTER_UNCLOSED` | P01 |
| N06 | 未加引号的冒号值 `description: Use when: x` | invalid | invalid | `FRONTMATTER_YAML_INVALID` | P01 |
| N07 | frontmatter 为列表 | invalid | invalid | `FRONTMATTER_NOT_MAPPING` | P01 |
| N08 | 重复键 `name` | invalid | invalid | `FRONTMATTER_DUPLICATE_KEY` | P01 |
| N09 | YAML 锚点/别名 | invalid | invalid | `FRONTMATTER_ALIAS_FORBIDDEN` | P01 |
| N10 | `SKILL.md` 非 UTF-8 | invalid | invalid | `SKILL_MD_NOT_UTF8` | P01 |
| N11 | name 缺失 / 空串 / 数字 | invalid | invalid | `NAME_MISSING` / `NAME_NOT_STRING` | P02 |
| N12 | name 含 `/`、`\`、控制字符或为 `..` | invalid | invalid | `NAME_UNSAFE` | P02 |
| N13 | description 缺失 / 全空白 / 列表 | invalid | invalid | `DESCRIPTION_MISSING` / `DESCRIPTION_NOT_STRING` | P01 |
| N14 | ZIP 条目 `../x`、`/etc/x`、`C:/x` | invalid | invalid | `PATH_UNSAFE` | R00 |
| N15 | 路径含 `CON.txt` 或以点结尾 | invalid | invalid | `PATH_RESERVED_NAME` | R00 |
| N16 | `A.md` 与 `a.md` 并存 | invalid | invalid | `PATH_CASE_COLLISION` | R00 |
| N17 | 符号链接 / 重解析点 / ZIP symlink | invalid | invalid | `SYMLINK_FORBIDDEN` | R00 |
| N18 | 超文件数、单文件、总大小、压缩比、frontmatter 大小 | invalid | invalid | `LIMIT_EXCEEDED` | R00 |
| N19 | 损坏/加密 ZIP | invalid | invalid | `ARCHIVE_CORRUPT` | R00 |
| N20 | GBK 文件名 ZIP（无 UTF-8 标志） | invalid | invalid | `PATH_ENCODING_UNSUPPORTED` | R00 |
| N21 | ZIP 重复条目名 | invalid | invalid | `DUPLICATE_ENTRY` | R00 |
| E01 | 格式有效、含 `scripts/run.py`、无 Python 执行器 | valid；`executionReadiness=limited`，`helper_language_unsupported` | valid | 格式无诊断；执行限制单列 | W10/L05 |
| E02 | 格式有效、`allowed-tools: Bash(*)` | valid；不授予 Bash | valid | 越权调用 `PERMISSION_DENIED` | L05/T05 |
| E03 | 含 `.wasm` helper、worker 缺失 | valid；`unavailable`，`runtime_unavailable` | valid | 执行期 `RUNTIME_UNAVAILABLE` | W04/W10 |

## 14. 合同落点与任务映射

| 本 SPEC 章节 | 机读合同（K02） | 实现任务 |
|---|---|---|
| §3 容器/路径/预算 | `agent-skills.json` `budgets`、`resourceRef`、`diagnostics` | P03、P07、R00、R01 |
| §4–§5 解析与字段 | `diagnostics`、`profiles` | P01、P02、P06 |
| §8 身份/指纹 | `skillRef`、`stableId`、`packageDigest`、`sources` | P04、P05、R01、R08 |
| §9 目录/激活/读取 | `activationSnapshot`、`agent-skill-tools.schema.json` | L00–L04、L06、R03 |
| §10 执行就绪 | `statusSets.executionReadiness`、`statusSets.limitationKind` | L05、W04、W10、W12 |
| §7 诊断 | `diagnostics` | P06、U06、T00 |

### 14.1 应用数据表 schema（R07 冻结）

应用库（SQLite，迁移 `src-tauri/src/db/migrations/agent_skills.rs`）只新增两张表，字段一律 snake_case；读写只经仓储 `src-tauri/src/infrastructure/persistence/sqlite/agent_skills_sqlite.rs`（写入前校验 stableId、packageDigest 与枚举值，SQL 侧再用 CHECK 兜底）。不建故事方法重构的作品表。

| 表 | 主键 | 字段 | 说明 |
|---|---|---|---|
| `agent_skill_bindings` | (`binding_scope`, `target_id`, `slot`) | `binding_scope`（`scene_storyboard`/`scene_optimize`/`image_generation`）、`target_id`、`slot`（子槽，如优化能力；无则空串）、`story_id`、`stable_id`、`package_digest`、`bound_at` | 绑定只 pin `stableId + packageDigest`（§8），不存 pipeline/plan hash；索引 `story_id`、(`stable_id`,`package_digest`)。 |
| `agent_skill_activation_snapshots` | `activation_id` | `run_id`、`stable_id`、`package_digest`、`source`（`builtin`/`user`）、`activation_source`（`user`/`model`/`binding`）、`activated_at`、`body_sha256`、`resource_count`（≥0）、`snapshot_json`（K02 `activationSnapshot` 原样 camelCase JSON）、`created_at` | 只插入不覆盖；索引 `run_id`、(`stable_id`,`package_digest`)。 |

- 迁移：`CREATE … IF NOT EXISTS`，在 SAVEPOINT 内执行，可重复运行；任一语句失败整段回滚，既有表与行保持原值，启动按既有约定报 `DB_MIGRATE`。
- 安装索引不入库：已安装包版本与 `index.json` 放在应用数据目录 `agent_skills/`（内容寻址版本目录 + 原子替换的索引文件，R01/R02），保证“版本目录与索引”在同一文件系统内原子切换。
- 旧表 `scene_storyboard_skill_binding`、`scene_optimize_skill_binding`、`content_skill_decision_drafts` 保留不动，由 M01 迁移/退役；旧绑定（`skill_id + plan_hash`）不能自动映射为 pin，须用户重新选择（R08）。

## 15. SKILL worker 合同（W00 冻结）

机读合同：`packages/app-contracts/v1/agent-skill-worker.json`（`contractScope: internal-worker-ipc`）。常量、错误码与报文字段表由 `bgx-agent-skills-core/build.rs` 生成到 `worker_protocol` 模块（INV-CONTRACT-SSOT），worker（`bgx-skill-runtime`）与 GUI transport（`bgx-skill-transport`）共用同一份类型，不各自手抄。

### 15.1 定位与边界

- worker 只承载**已有 BGX 动态 guest**：组件验证（`validateComponent`）与 `produce-plan` 执行（`producePlan`）。现有 WIT `bgxiong:content-skill/content-skill@1.0.0`（`wit/content-skill-v1/world.wit`）标记为 `helper-compat`：它是已发布 guest 的兼容 helper ABI，**不称通用 SKILL ABI**；普通 Markdown SKILL 不经过 worker（§10）。
- 不依赖 Adapter：不复用 `adapter-worker.json` 报文、broker 或权限 policy；与 Adapter A47 只共享“GUI 闭包无 Wasmtime”的目标，交叉登记见执行台账 §3。
- worker 不读写文件、不联网、不启动子进程、不写库，只返回字节或 typed failure。Plan 解析、验收（accept/prepare）与写入仍由宿主完成。

### 15.2 帧与报文

- 帧：`u32` 小端长度 + UTF-8 JSON 对象，一帧一条报文；长度上限 `framing.frameMaxBytes`，超限在分配前拒绝。stdout 只传协议帧，stderr 只作有界诊断（`stderrTailMaxChars`）。
- 父→worker：`hello{protocolVersion}`、`validateComponent{requestId, component}`、`producePlan{requestId, component, input}`、`cancel{requestId}`、`shutdown{}`。
- worker→父：`helloAck{protocolVersion, contractSha256, workerVersion, limits, abiWorld}`、`validated{requestId, componentSha256, imports}`、`planProduced{requestId, componentSha256, inputSha256, output, fuelConsumed}`、`failed{requestId, code, message}`、`cancelAck{requestId, active}`、`shutdownAck{}`。
- 字段闭集（未知字段、未知 `type` 一律 `AGENT_SKILL_WORKER_FRAME_MALFORMED`）；请求报文**没有**限额字段，不能放宽限额。
- `blob{bytesBase64, sha256}`：标准 base64；`sha256` 为 `sha256:<小写 hex>`。接收方先按上限检查解码长度，再校验 hash；不符不执行（`AGENT_SKILL_WORKER_HASH_MISMATCH`）。`componentSha256`/`inputSha256` 回显在终态中，供宿主核对 package pin。

### 15.3 限额与 import 闸

- 限额与 allowlist 同源于 `content-skill-limits.json`：`wasm.memoryBytes`、`wasm.fuel`、`wasm.wallMs`、`allowedHostImports`、`maxSnapshotBytes`（输入上限）、`maxPlanBytes`（输出上限）；另有 `componentMaxBytes`。helloAck 回显实际限额。
- 每个请求独立 Store：fuel 计量、内存 limiter、epoch 中断（`epochTickMs` 一个 tick；墙钟到 `wallMs` 或收到 cancel 即中断）；不链接 WASI、HTTP、FS 进口；import 不在 allowlist 即 `AGENT_SKILL_WORKER_IMPORT_FORBIDDEN`。
- 平台差异（W01 Windows 实测，2026-10-02）：Wasmtime 28 的 fuel/epoch trap 经 libcall `longjmp` 抛出，在 Windows（MSVC）上会跨 `extern "C"` 帧展开，进程 Fast Fail（0xC0000409），与 `vendor_adapter_gateway` 既有结论一致。因此 Windows 上 worker 不计 fuel（`planProduced.fuelConsumed = 0`），也不在进程内触发 epoch trap；墙钟与 cancel（含 shutdown/EOF）由 worker 看门狗执行：先写 `failed WALL_TIMEOUT`/`CANCELLED` 终态并放弃该 guest 线程（不 join），worker 继续响应；宿主随后的 `shutdown`/EOF 结束 worker 进程，guest 随进程结束（worker 内不使用进程 API）。内存 limiter、硬件 trap、import 闸、大小上限各平台一致；无限循环在 Windows 上最迟 `wallMs` 后以 `WALL_TIMEOUT` 结束。helloAck 仍回显合同限额。

### 15.4 生命周期、取消与错误

- 首帧必须是 `hello`；worker 回 `helloAck`，宿主核对 `protocolVersion` 与 `contractSha256`（合同文本 LF 归一后的 sha256），不一致即 `AGENT_SKILL_WORKER_INCOMPATIBLE`，不发请求。
- 每个 `requestId` 恰好一个终态（`validated`/`planProduced`/`failed`）；同时最多一个在途请求（`maxInflightRequests = 1`），其余立即 `failed BUSY`。
- `cancel` 只终止当前实例：worker 回 `cancelAck{active}`，并给该请求 `failed CANCELLED` 终态；`cancelGraceMs` 内无终态时宿主按子进程句柄终止（不按进程名强杀）。`shutdown` 取消在途请求、等其终态后回 `shutdownAck` 并以 0 退出；stdin EOF 视为父进程消失，取消在途请求后退出。
- 错误码闭集：worker 侧 `workerErrorCodes`（协议、hash、组件、import、ABI、实例化、内存、fuel、墙钟、取消、trap、guest err、输出过大、内部），宿主 transport 侧 `transportErrorCodes`（路径不存在、启动失败、不兼容、握手超时、终态前退出、协议违例）。

### 15.5 旧 WIT 映射

| WIT `produce-plan(snapshot-json: list<u8>) -> result<list<u8>, string>` | worker 协议 |
|---|---|
| `snapshot-json` 原字节 | `producePlan.input` |
| `ok(list<u8>)` 原字节 | `planProduced.output`（worker 不解析 Plan） |
| `err(string)` | `failed{code: AGENT_SKILL_WORKER_GUEST_ERROR, message}`（截断到 `stderrTailMaxChars`） |

## 16. 变更记录

- 2026-10-02 v1.0（K01）：初版冻结；规范来源当日联网核验。
- 2026-10-02 v1.1（P00，分支 `wip/agent-skills`）：§4.4 固定 YAML 库 `yaml-rust2 =0.13.0`（只用拉取式事件 API）、核心判定细则与 YAML 测例文件；其余章节未改。
- 2026-10-02 v1.2（R07，分支 `wip/agent-skills`）：新增 §14.1 应用数据表 schema（绑定 pin 与激活快照两表、迁移与仓储约束、安装索引不入库、旧表保留）；其余章节未改。
- 2026-10-02 v1.3（W00，分支 `wip/agent-skills`）：填写 §15 SKILL worker 合同（机读合同 `agent-skill-worker.json`）；其余章节未改。


## SG管理合同补正（2026-10-07；合同已定义，运行升级待实施）

`packages/app-contracts/v1/agent-skill-library.schema.json`是Host索引v2的结构单一源，`agent-skills.json`只引用其entryToken/category，不手写跨语言镜像。当前运行库仍v1，禁止按本节自行改写用户index；转换只走SG03原生离线升级及真实数据副本验证。

- v2根：schema_version/revision/entries/categories/entry_tombstones/pending_effect_commits/builtin_suppressions。entries保留原始JSON行，逐行按storedEntry校验；坏行继续可诊断、可原字节保留，不能靠放宽有效entry掩盖损坏。索引顶层闭集。
- 每次安装entry_id为新UUID，entry_revision为安全整数；IPC entryToken为entryId/revision/packageDigest，落盘storedEntryToken为entry_id/revision/package_digest。卸载重装、undo恢复都生成新身份及当前revision，不复活旧token；CAS比较整个token，禁止只比digest。
- 删除代次：entry_tombstones按stableId记录最新deletion_id/deleted_entry_token；undo要求对象仍不存在且同一deletion_id，防删除重装ABA。绑定另沿公共bindingToken合同，不借索引token代替。
- 分类字典内置8个主题直接由schema builtinCategories消费；user分类为user:UUID，多选；label可改，ID不变。未知作者类别只保留declared_categories/declared_category_labels，不自动建字典、不作为权限；未分类为空membership投影。
- metadata.bgx-categories为字符串空格列表；bgx-tags为JSON字符串数组的字符串值；bgx-category-labels为JSON字符串映射的字符串值；扩展损坏只诊断本扩展，不阻断通用方法激活。包字节不变。category_override/tag_override的null跟包、[]显式清空；effective值只作DTO投影，不能写第二主库。
- origin保存来源方式、导入摘要、公开来源引用、derived_from及可信Builtin基线摘要；未证明官方来源的历史差异不升级覆盖。来源引用禁止凭据及本机私有源路径；无可信基线保持null。Builtin显式卸载的抑制记录保存before业务状态和相邻顺序锚点，列表/seed不得自动重装。
- pending_effect_commits是同一索引原子提交证据，按stableId记录effect_ref/before_token/after_token/result_summary，不是Skills请求状态机。公共协调器对账完成前禁止同目标后续写入，其他目标可继续；清marker不变更业务revision。对账、receipt、undo耐久保护根沿E02。
- 没有全库200包/16激活累计上限；有界单页/单文件/worker预算仍沿既有合同。未交付公共FileRef与request工厂时，禁止新增raw-path管理工具或Skill私有执行流程。

离线结构正反例：`scripts/test-agent-skill-library-contract.py`。它只验证合同，不表示SG01.04真实Rust serde、迁移、UI、MCP/F1或安装已通过。执行状态只记原Skills台账。


## 视频提示词格式方法（VP-VS，2026-10-09）

格式方法是标准Skill包：SKILL.md及可选references，库保留原包字节，导出不包装成私有格式。格式有效与应用兼容绑定是两个状态；绑定存应用库，原包不注入projection配置。派生/升级后展示拟议重绑定，再由用户按既有编辑发布流程确认。

选择器可选当前目标默认方法或自选stableId+packageDigest。打开和查询只读；用户点击生成/优化时，共享VideoPromptService冻结目标、文本模型、格式/内容方法及有限资源子集。包缺失、digest变化、绑定不兼容或资源缺失显式失败，不换模型/包。

`project_existing`只换已审母版的目标格式，必须带Rust返回的canonicalDocumentRef与精确正文指纹；不重新生成母版。源稿或语义依赖变更失效，方法/目标格式变更仅重新投影。代码只校验结构、闭集ID及数值，内容阻断由评审模型的结构化blocking结论决定，minor只记录。

GUI/F1/MCP同一能力目录、参数schema、任务及投影回执。回执含母版来源、冻结方法、资源证据、目标快照与projectionFingerprint；canonical ref来自真实提交。正文、母版、投影marker、canonical effect与请求证据在同一CAS事务落盘；冲突返回候选，保留用户内容和已完成兄弟项。已确认正文在预览/提交时逐字直通，标签展示只读已编译inventoryLabels。

离线执行状态以 「VS85报告」（开源版未收录该内部文档或源码路径） 为准。实模首响/零修复轮、GUI跨窗/安装包/成片仍须分别验收；当前未执行部分不得宣称完成。H3源包授权仍pending_release_review。

## 内容整理 / 格式适配与Skill扩写（2026-10-20，开发版合同）

UI名称已确定；既有host ID不变。默认保真，任一所选Skill明确允许即可按正文范围扩写，格式Skill同样适用，不另加程序总开关。用户本次要求优先，相反的显式Skill策略需说明冲突。

可选metadata.bgx-expansion（allow/preserve）、策略来源、新事实引用与正负通道由agent-skills.json及video-prompt-channels.schema.json定义；本次开发版按「正式PLAN」（开源版未收录该内部文档或源码路径）接入，旧发布版仍以其实际schema为准。新增内容必须进入新版母版与审阅；现有project_existing仍保持不改母版的含义，适配扩写使用adapt_expand和Rust下发的canonicalDocumentRef，不静默改变旧API语义。当前版本不得接受未发布字段来冒充支持。作者与执行说明见「通用GUIDE」（开源版未收录该内部文档或源码路径）。
