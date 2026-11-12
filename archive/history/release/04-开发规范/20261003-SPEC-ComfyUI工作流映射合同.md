> **历史演进产物，不代表当前设计，不作为当前实现或开发依据。**
>
> - 原路径：`release/04-开发规范/20261003-SPEC-ComfyUI工作流映射合同.md`
> - 文档标识：取自原文件名 `20261003-SPEC-ComfyUI工作流映射合同.md`（这是原文件名中的标识，不是本次整理推定的写作或生效日期）
>
> 现行架构请从仓库主导航进入查阅；本文只作为历史演进资料，不作为当前实现或开发依据。

> **开源整理版**（2026-11-12）：已去除或改写私有路径、未公开端点、凭证示例，以及登录 / 认证 / 付费相关实现细节。若某节全部属于上述范围，将改为一句能力边界说明。

> **2026-10-14 后续合同演进（待实施）**：「全量导入与说明智能提炼PLAN」（开源版未收录该内部文档或源码路径）为本专题增量权威，保留既有身份、冻结与原子apply，扩展编译上下文/序列化证据/稳定歧义ID/知识侧车。本页字段描述仍为现行实现，须在新计划CW08–CW17同批版本化，不能直接当新合同已实现。

# SPEC · ComfyUI 工作流映射合同

**文档时间**：2026-10-03（Asia/Shanghai）  
**版本**：v1.0  
**状态**：G0 冻结；实现从 PLAN S05 开始。  
**权威**：「PLAN」（开源版未收录该内部文档或源码路径） §1–§3。

## 1. 边界

该合同是客户端内部 ComfyUI 平台合同。它描述工作流来源、可执行图快照、工作台语义与具体节点输入之间的绑定。它不是公开 Vendor Adapter SDK，不进入 WIT，不允许通用 host 或通用 UI 识别 `class_type`。

纯核心接受 JSON 数据并返回 JSON 数据，不读文件、不联网、不访问 DB、不启动进程。来源发现、节点定义获取和持久化由 `src-tauri/src/comfyui` 的端口实现；生产任务与媒体生命周期继续由现有宿主负责。

## 2. 身份与摘要

`WorkflowIdentityV1`：

- `schemaVersion = 1`；
- `instanceId`：由规范化 endpoint 身份派生的非秘密稳定 ID；本地导入使用显式 local-library instance；
- `userScope`：Comfy 多用户作用域，单用户显式为 `default`；
- `sourceKind`：`comfyUserData | localFile | clientLibrary`；
- `sourcePath`：来源内部规范化相对路径；禁止绝对路径、`..`、空段与控制字符；
  本地文件/目录导入后，映射指向 `clientLibrary / entries/<entryId>`；绝对原始路径只保留在导入仓的 `SourceIdentity` 中。创建映射时即校验身份，不能等到提交才拒绝。
- `displayName`：只展示，不参与唯一性；
- 稳定键由前四项构成，凭据、token 和 URL query 禁止进入。

`WorkflowSnapshotV1`：`identity + sourceDigest + graphDigest + definitionDigest + ruleRegistryVersion + capturedAtMs`。所有摘要为规范化字节的 `sha256:<64hex>`。提交冻结 snapshot；源更新不改变已提交任务。

原件导入发布的映射 `definitionDigest` 使用 `node_registry::definition::mapping_definition_digest`：仅签名编译后 API 图使用的规范化节点定义，导入与提交共用。明确声明 `image_upload / video_upload / audio_upload` 的 COMBO 不签名动态文件名 `options`；其类型、上传声明及其余约束仍签名，普通模型/采样器枚举仍完整签名。编译产物 `digests.definition` 继续记录完整编译定义快照，二者用途不同，禁止直接比较。

## 3. Binding 合同

`MappingContractV1` 必含：

- `schemaVersion = 1`、`contractVersion`、`snapshot`；
- `outputSelection`：选中的输出节点及媒体种类；
- `bindings[]`：稳定 `bindingId`、`semanticRole`、`valueType`、`targets[]`、可选 default/range/enum、confirmation；
- `diagnostics[]`：闭集 severity `info | warning | blocking`、code、关联候选；
- `status`：`ready | needsConfirmation | unsupported | stale`；
- 可选 `timing`：视频输出的时长/fps 子合同（`DurationContract`，类型 SSOT 为 `bgx-comfy-workflow::media_contracts`，与工作台 caps 的 `durationContract` 同形）；
- 可选 `resolution`：视频输出的分辨率子合同（`ResolutionContract`，同上）；
- 可选 `fixedWrites[]`：`{ nodeId, inputName, value }` 常量写入（如把会改写用户提示词的 prompt-enhance 开关钉为 `false`），无需调用方取值；目标必须存在且不重复，apply 时与 binding 写集一起做冲突检测与读回证明。

**规则投影（PLAN 决策 D1）**：素材槽、timing 与 resolution 只由平台成熟规则模块（`asset_adapt` / `reference_slots` / `duration_slots` / `resolution_slots`）识别，宿主投影为 `RuleProjectionV1 { assetBindings, timing, resolution }` 交给纯核心；核心只做结构校验（角色闭集、`mediaRef`、槽形状、容量、目标存在、bindingId 唯一）并冻结，不自行猜测素材用途。timing/resolution 以子合同整体冻结，不再拆成扁平 binding 重复表达。

**值绑定投影（PLAN 决策 D2）**：提示词、负向、seed、生图尺寸/宽高比选择器绑定与 `fixedWrites` 由平台生成写集规划器 `plan_generation_writes` 产出，作为 `RuleProjectionV1.valueBindings / fixedWrites` 交给核心校验（bindingId 唯一、非素材语义、1..=上限个目标、节点与输入在图中存在）后冻结；核心不再自行猜测提示词与 seed，只保留 steps/cfg/denoise 标量识别。平台 bindingId 闭集：`prompt.positive`、`prompt.positiveCaption`（Ideogram caption_verifier_v1 JSON 包装）、`prompt.negative`、`prompt.negativeFill`（用户未给负向时只填源图空槽为 `""`）、`sampler.seed`、`image.width`、`image.height`、`image.aspect`、`image.megapixels`、`media.referenceAudio`（flat）。源图不存在的输入不得成为 target。已知偏差：按节点标题识别 Primitive 提示词仍标 `automatic`（与切换前生产行为等价），由 S23 确认界面提供覆盖入口。

素材 binding（`firstFrame`/`lastFrame`/`referenceImage`/`referenceAudio`/`referenceVideo`）携带非空 `assetSlots[]`：`family` 闭集 `named | indexed | flat`（单源 `assetSlotFamilies`），`named` 含 `consumerNodeId + inputKey`，`indexed` 含 `consumerNodeId + parentKey + keyPrefix`，`flat` 无 consumer；均含 `minCount ≤ maxCount`、`maxCount ≥ 1`、`required`。`targets[]` 只列已存在的加载节点输入，待合成槽不预造目标。带 `assetSlots` 的 binding 是结构化素材绑定，只能由平台素材适配器执行，值 apply 必须拒绝。

首版 `semanticRole` 闭集：`positivePrompt`、`negativePrompt`、`seed`、`width`、`height`、`aspect`、`resolution`、`duration`、`fps`、`firstFrame`、`lastFrame`、`referenceImage`、`referenceAudio`、`referenceVideo`、`mask`、`steps`、`cfg`、`denoise`。

`BindingTargetV1` 只能引用图内节点 ID 与输入字段；同一 binding 可含多个关联 target。变换使用闭集 `identity | secondsToFrames | framesToSeconds | aspectSelector | resolutionSelector`，并显式列依赖 binding；禁止脚本、表达式与 eval。

`confirmation`：`automatic | userConfirmed`。只有结构和节点定义能唯一证明用途时可 automatic；名称、标题、ID 顺序、文件名或模型品牌不能单独构成 automatic 证据。

## 4. Apply 与应用证明

`ApplyRequestV1` 包含 frozen contract、workflow JSON 和 `values`（key 为 bindingId）。apply 按以下顺序运行：

1. 校验合同版本、摘要、状态和 outputSelection；
2. 校验每个 bindingId 存在、值类型/范围/枚举合法；
3. 展开 targets 与关联变换，构建完整写集；
4. 若两个 binding 对同一 target 产生不同值，返回 blocking 冲突，零写入；
5. 在副本中应用并重新读取每个 target；
6. 返回 `workflow` 与 `AppliedBindingV1[]`，包含 bindingId、语义、目标和实际值。

工作台只显示 status 为 ready 且有 bindingId 的参数。提交不得重新扫描节点寻找目标；应用证明由最终工作流读回产生，不能由调用方自报。

## 5. 失效规则

- sourceDigest 或 graphDigest 变化：合同 stale，重新分析；
- definitionDigest 变化：仅当变化触及 binding 或输出祖先时 stale，否则可重签新 snapshot；这一优化须有结构证明，首版可全部 stale；
- ruleRegistryVersion 变化：受影响规则的合同 stale；
- endpoint instanceId 或 userScope 变化：视为另一来源，禁止复用；
- 仅 displayName、画布位置或无关注释变化且 graphDigest 不变：不失效。

旧 caps/时长/分辨率确认只能在能精确投影为 targets、变换和摘要时升级；不能证明的项进入 needsConfirmation。生产运行不保留旧注入 fallback。

**提交时合同供给（PLAN 决策 D3）**：提交只经 `resolve_submission_contract` 获取合同。工作流库来源身份为 `(client-library, default, ClientLibrary, workflows/<name>.json)`；已存合同在 `status=ready`、graphDigest、ruleRegistryVersion、输出媒体种类均一致时复用；过期且含 `userConfirmed` binding → `workflowMappingStale`（零写入）；缺失或纯 automatic 过期 → 重新分析，ready 即原子保存（惰性迁移），否则 `workflowMappingNeedsConfirmation`。内联 JSON 与内置模板用 `inline/<hex16>.json` 临时身份，不落盘。输出选择优先已存选择，其次该媒体种类唯一候选（多候选时排除 PreviewImage），否则 `workflowMappingNeedsConfirmation` / `workflowMappingOutputMissing`。损坏的合同文件 → `workflowMappingCorrupt`。

## 6. 错误与展示

核心错误为结构化 code + details；用户文案由宿主 locale 映射，原始节点错误只进日志。首版错误闭集至少包含：`IDENTITY_INVALID`、`GRAPH_INVALID`、`OUTPUT_AMBIGUOUS`、`BINDING_AMBIGUOUS`、`TARGET_MISSING`、`TYPE_MISMATCH`、`VALUE_OUT_OF_RANGE`、`WRITE_CONFLICT`、`CONTRACT_STALE`、`NODE_DEFINITION_INCOMPLETE`、`RESOURCE_LIMIT_EXCEEDED`。

## 7. 预算

预算数字由 `packages/app-contracts/v1/comfy-workflow-mapping.json` 单源生成。首版需定义最大 JSON 字节、节点数、边数、binding 数、每 binding target 数、诊断数和遍历深度。越界显式失败；不得截断后继续映射或提交。

## 8. 跨语言投影

平台管理 IPC 传递完整私有合同；通用工作台只得到标准能力投影：参数 key、类型、默认值、范围、素材用途、required、bindingId、ready 状态。TS 不重建摘要、图分析、变换或写集规则。每个 IPC 必须用真实前端 camelCase payload 做 Rust serde 测试。

## 迁移说明（2026-10-04 实施后）

- 旧工作流库条目无需手工迁移：首次提交时按稳定身份惰性冻结合同（PLAN D3/D6）；仅含自动 binding 的过期合同按当前规则重新冻结，含用户确认的过期合同显式报 `workflowMappingStale`，需在设置页“重新导入并映射”确认。
- 含 `{{PROMPT}}` 等字符串占位符的旧模板不再支持，提交报 `workflowLegacyPlaceholderUnsupported`；内置模板已改为无占位 API 图（PLAN D9）。
- 画布格式工作流中的未知第三方节点若带位置 `widgets_values`，显式报 `workflowWidgetLayoutUnknown`（含 nodeId/classType）；请在 ComfyUI 中“导出（API）”后导入，或为该节点补 `node_registry` 条目与夹具。
- 能力识别唯一在 Rust（`comfyui/analyzer.rs`）；前端不保留 class_type 分析镜像（PLAN D12）。


## 2026-10-15：视频提交的同图参数投影

`prepare_contract_submission` 显式接收本次服务端节点定义，返回编译图与该冻结合同投影的能力；已发布提交同样返回能力。视频 pipeline 的时长、分辨率、采样和 loader 目标不能重新从旧编译器的 sidecar 读取。仅有自动映射但媒体子合同尚未完成确认的旧合同，按既有 D3 分析路径重新冻结；含用户确认绑定的过期合同仍明确拒绝。

`project_platform_rules` 在有充分节点定义、无歧义时完成媒体子合同确认；新导入 caps 从合同投影。旧库能力读取在 graphDigest 匹配时也使用冻结映射；旧库“重新分析参数”和“确认参数”同步写回该映射，不能只写 caps。旧默认只在目标结构可对应时通过既有默认合并规则保留。

图片的 LoadImage 适配只用于图片提交；视频准备保留素材连线，先校验冻结媒体合同，再由视频资产适配器绑定素材，最后应用时长和分辨率。真实定义漂移继续阻断，禁止提交失败后改写指纹。

LTX conditioning/crop/guide 包装链按实际连接的 CONDITIONING 输出槽追踪文本来源，共用 `text_encode_nodes` 结构识别；规则版本 6 使旧自动映射重新分析。候选列表无歧义不等于可提交，维护/发布验收还必须验证合同为 Ready，不能漏掉 blocking diagnostics。

案例与发版证据：「排障参考」（开源版未收录该内部文档或源码路径）、「本次 FIX」（开源版未收录该内部文档或源码路径）。
