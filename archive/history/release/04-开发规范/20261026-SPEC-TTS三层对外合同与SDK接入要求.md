> **历史演进产物，不代表当前设计，不作为当前实现或开发依据。**
>
> - 原路径：`release/04-开发规范/20261026-SPEC-TTS三层对外合同与SDK接入要求.md`
> - 文档标识：取自原文件名 `20261026-SPEC-TTS三层对外合同与SDK接入要求.md`（这是原文件名中的标识，不是本次整理推定的写作或生效日期）
>
> 现行架构请从仓库主导航进入查阅；本文只作为历史演进资料，不作为当前实现或开发依据。

# TTS 三层对外合同与 SDK 接入要求

> **开源整理版**（2026-11-12）：已去除或改写私有路径、未公开端点、凭证示例，以及登录 / 认证 / 付费相关实现细节。若某节全部属于上述范围，将改为一句能力边界说明。


日期：2026-10-26。版本：语义合同 v1.0。状态：**语义 v1.0 已机器化；S0 开发候选，主程序／模型包未开发，产品声音验收 NOT_RUN。** 用户随后授权 S0；本轮先更新 main 至 `956c9efdd` 再实施，原文基线记录保留为历史。机器源及作者入口见「Speech v1」（开源版未收录该内部文档或源码路径），实际验收和剩余范围见 PLAN 的 S0 记录。

本轮基线为 main `090f37384bbfddb4d381c0b24cf6fed7ba0e0033`，已执行 `git pull --ff-only origin main` 确认最新。此前产品源码基线 `272794db1` 未因上一轮文档提交改变。本文是新 TTS 外挂路径的字段与行为唯一设计来源；「开发计划」（开源版未收录该内部文档或源码路径）只安排实施与验收。既有 SDK 信封仍以当前公开 schema 为准；本文明确标出的新增项必须先实现、验证并发布，不能宣称当前 SDK 已支持。

提交收尾时远端推进到 `e64032ca5`（包含视频提示词修复 `d27b13a37`）；首次推送因非快进被拒后已 fetch 并将本次文档提交无冲突顺接到该 main。核对增量未修改本文所查的 TTS/SDK 文件，相关源码结论仍适用；远端其他功能原样保留，本轮没有实施或验收这些功能。

## 1. 冻结范围与三层边界

1. **主程序 ↔ 中间模块**：选择、意图、媒体授权、绑定预览、任务及成果。UI 可任意调整布局、组件和文案，但输出同一请求。业务写回、权限和持久化属于主程序。
2. **中间模块 ↔ 第三方包**：公共 speech-v1 输入、结果、可选能力、包描述和受管 runtime 生命周期。中间模块不认识厂商；adapter 在包内把公共合同翻译成引擎私有请求。adapter 与其 companion 的模型业务 HTTP 格式是包内实现，允许随包换，不能成为宿主依赖。
3. **公开 SDK ↔ 宿主／包作者**：SDK V2 六操作、RCD、schema 身份、媒体、错误、审计、流和作者交付物。共用同一套工具和校验，不新造一套 TTS 私有执行信封。

冻结的是字段、含义、缺省、单位、归属和兼容规则，不是 UI 外观、厂商型号、模型权重或内部推理算法。本文之后实施不得边写代码边重新决定这些语义；发现真实缺口先记录受影响调用并修订合同版本，只暂停依赖它的工作。

### 1.1 最小可用与可选能力

- 具备普通合成且已有默认声音的模型，领域最小输入只有 `schemaVersion` 与 `text`；基础输出只需一个可读取的音频。SDK 信封、媒体身份、包完整性由工具提供，不要求模型原生理解这些字段。
- 可选能力未声明是 **unknown**，不是 false、不是保证支持、也不是安装失败。发现缺失不启动模型做探测，不阻断基本调用；所选操作是否实际可执行由真实实现决定。
- 不支持的选填风格／情绪／语言提示等保留在原始意图，生成 `Notice`，有效请求不发送该项；同一规则必须在公开 profile 库和作者入口生效。未知能力的合法选填项可交 adapter 判断；未确认采用时只显示未确认，不假称生效。
- `mode=reference_clone`、`delivery=stream`、明确的 `voiceId` 和 `runtimeProfile` 是用户选择，不可悄悄替换。缺所选能力只使该次选择不可执行，普通合成仍可用。纯克隆模型可安装，但不能伪造默认声音；只有确实必需的参考/转写成为该模式的前提。
- 缺 UI 控件、声音目录、时间戳、性能、评分、usage、报告、附带源码，不拒装、不拒已有功能。坏可选输出附件单独隔离，不连带丢弃可用音频。
- 真实传输不可解析、资源越权、包摘要错配、必需权重不存在、音频本体损坏仍明确失败；不能把写作建议或“资料不齐”解释为执行安全问题。

### 1.2 全文类型和缺省规则

下文 `type` 代码块是文档类型记法，不是已生成的 TypeScript API。没有 `?` 的字段必填且无隐式缺省；`?` 只表示可省略，不表示可传 `null`。允许 null 的字段必须写 `| null`。省略可选对象即整组不提供；对象存在时其必填子字段仍需提供。没有专门缺省说明的选填字段，省略表示不提供／不提出偏好／未知；不补造业务数据。

```ts
type Json = null | boolean | number | string | Json[] | { [key: string]: Json };
type Id = string;                 // 非空，不透明，区分大小写，不含 NUL
type Int = number;                // JSON 安全整数，非 NaN/Infinity
type UInt = Int;                  // >= 0
type PositiveInt = Int;           // >= 1
type Positive = number;           // 有限且 > 0
type Sha256 = string;             // 64 位小写十六进制
type U64Text = string;            // 十进制无符号整数，0..18446744073709551615
type Pointer = string;            // RFC 6901 JSON Pointer，禁止通配符
type Namespace = string;          // 已验证发布者命名空间；std 为公共保留
type RelativePath = string;       // 包内 / 分隔相对路径，禁绝对路径、..、NUL、逃逸链接
type MediaRef = { mediaId: Id };   // 只有此一个字段，禁任意路径/URL
type Selection = { bindingId: Id; modelId: Id };
type Extension = { namespace: Namespace; key: Id; value: Json };
type Notice = {
  code: "optional_not_applied" | "optional_unconfirmed" | "metadata_invalid"
      | "metadata_unknown" | "capability_unknown" | "partial_output";
  pointer: Pointer;               // 相对当前 input 或 data；根为 ""
  message: string;                // 展示说明，不用于匹配/分支
  itemId?: Id;                    // 仅 batch；省略表示本次整体
};
type SdkErrorCode = "INVALID_PACKAGE" | "UNSUPPORTED_VERSION" | "UNSUPPORTED_FEATURE"
  | "INVALID_INPUT" | "UNAUTHORIZED" | "PERMISSION_DENIED" | "UPSTREAM_ERROR"
  | "RATE_LIMITED" | "TIMEOUT" | "ACCEPTANCE_UNKNOWN" | "RESOURCE_UNAVAILABLE"
  | "CANCEL_UNSUPPORTED" | "INTERNAL_ERROR";
type SdkError = {
  code: SdkErrorCode; message: string; retryable: boolean;
  vendorCode: string | null; vendorRequestId: string | null;
};
type Reply<T> = { ok: true; value: T } | { ok: false; error: SdkError };
type Page<T> = { items: T[]; nextCursor: Id | null };
```

除 `Json`、明确的字典和独立解析的可选附件外，对象字段闭合。未知顶层字段是协议形状错误；扩展只进入 `extensions`。数组顺序保留，ID 按其作用域唯一；重复扩展 `(namespace,key)` 拒绝。字段无任意业务数量上限；沿 SDK 真实帧／字节／内存预算，分页可续。可选参数存在但类型非法是输入结构错误，不能把 `"fast"` 伪转成数值；缺少该参数完全合法。

原文 `text` 不 trim 后替换、不翻译、不截断、不按关键词拒收。空串不可合成；只有空白的输入可报没有可合成文本，原文仍保留。缺省 `mode=plain`、`delivery=complete` 是公开语义，不是宿主擅自修复用户内容。

## 2. 第一层：主程序与统一语音模块

这是所有 GUI、F1、MCP 消费的公共领域端口。实现时挂到现有 Creator 能力目录及生成 schema；名称表示逻辑端口，**不是当前已存在的 IPC 名或第七类 SDK 操作**。端口统一返回 `Reply<T>`；现有 Creator 外层回执再包装它，不复制任务系统。

### 2.1 输入、媒体与绑定

```ts
type HostMedia = { mediaId: Id; assetId: Id }; // assetId 是已有媒体服务授权标识
type SpeechDraft = {
  schemaVersion: 1;
  selection: Selection;
  input: SpeechInput;
  media?: HostMedia[];            // 缺省 []；不需要参考时合法
};
type SubmitSpeech = SpeechDraft & {
  clientRequestId: Id;            // 主程序/客户端一次用户提交生成
  writebackToken?: Id;            // 已有宿主授权的写回准备 token；省略仅保留任务成果
};
type Pin = {
  bindingId: Id; modelId: Id; packageId: Id; packageVersion: Id;
  bundleSha256: Sha256; adapterSha256: Sha256; runtimeSha256: Sha256;
  sdkPackageDigest: Sha256; sdkContractRelease: Id;
  speechProfileVersion: 1; runtimeContractVersion: 1;
  operationId: Id; operationVersion: PositiveInt;
  inputSchemaDigest: Sha256; outputSchemaDigest: Sha256; resumeSchemaDigest: Sha256;
  resourceDigests: { resourceId: Id; sha256: Sha256 }[];
  descriptorDigest: Sha256;
  modelRevision?: string;         // 只有真实来源提供时存在；实际资源摘要始终权威
};
type BindResult = {
  original: SpeechDraft;
  effectiveInput: SpeechInput;    // 仅移除已确认不消费的选填项；不改变 mode/text/模型
  notices: Notice[];              // 无提示 []
  executable: "yes" | "no" | "unknown";
  issues: { code: SdkErrorCode; pointer: Pointer; message: string }[];
  pin: Pin;                      // 预览身份；提交时重新核对并冻结
};
type ModelEntry = {
  selection: Selection; packageId: Id; packageVersion: Id;
  displayName?: string;           // 无则展示 modelId，可由用户别名覆盖
  availability: "not_checked" | "ready" | "needs_resources" | "unavailable";
  descriptor?: SpeechDescriptor; // 缺省 unknown，不关闭基本输入框
};
type ListModels = { cursor?: Id; limit?: PositiveInt }; // 缺省第一页／宿主实际分页预算
type ListVoices = { selection: Selection; cursor?: Id; limit?: PositiveInt };
type VoicePage = { available: "yes" | "no" | "unknown"; page: Page<Voice>;
  media: AudioAsset[]; notices: Notice[] }; // 目录试听原件由宿主接管；无则 []
```

- `listModels(ListModels) → Page<ModelEntry>`：读取已校验安装目录，不启动模型、不联网；有未准备资源仍可列出，不能伪报 ready。
- `listVoices(ListVoices) → VoicePage`：目录是可选能力；无目录返回 available=no、items=[]、nextCursor=null、media=[]，不要求用户先选一个不存在的 voice。目录调用失败只影响目录显示；已有 voiceId 或默认声音的合成不依赖目录成功。preview.mediaId 只可指向本次 media 中已接管的资产，不能向 UI 暴露过期 SDK 句柄。
- `bind(SpeechDraft) → BindResult`：纯结构/能力/引用检查，不推理、不加载权重。`unknown` 允许显式提交尝试，不等于失败；issues 只列真实阻断项。UI 展示 notices，同时保留 original 以便换回模型。UI 不自行静默改 mode。
- `submit(SubmitSpeech) → SubmitReceipt`：重新 bind、核对权限与版本、冻结 pin，创建/附着现有任务，调度真实执行。同一授权主体和 clientRequestId、同一冻结输入重复提交只附着原任务；同 ID 不同输入返回 INVALID_INPUT；失败后想重新生成必须是新的一次显式提交。不能对 ACCEPTANCE_UNKNOWN 自动重提。

writebackToken 在主程序准备并验证，包含业务目标及版本约束；不把 storyId、nodeId、数据库行号塞进 speech input 或发送给包。无写回目标的独立语音合法。媒体由资产服务解析并授予本次调用句柄，不暴露 app_data 绝对路径；输入内所有 mediaId 必须在授权集合中，未使用的资产不读取。

pin 摘要各有明确对象：bundleSha256 是原分发归档字节，adapterSha256 是其中标准 adapter 制品字节，sdkPackageDigest 是 SDK 已验证的 manifest.sha256 原字节摘要，runtimeSha256 是所选 code 清单原字节摘要，descriptorDigest 是 tts-bundle.json 原 UTF-8 字节摘要（即使 speech 描述省略也存在）。资源摘要取实际选用清单，不把用户 text/目录展示名混入包身份。runtime 就绪后再冻结该调用的 SDK Binding revision/configurationDigest；该连接快照属于执行记录，不改已冻结包/模型身份。重连重新取得授权连接，不把旧 origin 用作回退。

批量第一层端口为 `bindBatch(BatchDraft) → BatchBindResult`、`submitBatch(SubmitBatch) → SubmitReceipt`。`BatchDraft` 精确复用 SpeechDraft，只把 input 换成 §3.4 BatchInput；`BatchBindResult` 精确复用 BindResult，把 original 换成 BatchDraft、effectiveInput 换成 BatchInput；`SubmitBatch` 是 BatchDraft 加同样的 clientRequestId/writebackToken。writebackToken 在宿主按 itemId 绑定各目标，不进入包。批量复用同一任务/取消/结果端口，逐条队列仍是普通 submit，不假冒此端口。

### 2.2 任务、取消、结果与事件

```ts
type SubmitReceipt = { taskId: Id; requestId: Id; pin: Pin; notices: Notice[] };
type TaskRef = { taskId: Id };
type AudioAsset = {
  assetId: Id; mediaId: Id; completeness: "complete" | "partial";
  itemId?: Id;                    // batch 归属
};
type TaskView = {
  taskId: Id; requestId: Id;
  state: "queued" | "running" | "completed" | "failed" | "cancelled";
  cancelRequested: boolean;
  progress: { fraction: number | null; message: string | null } | null;
  outputs: AudioAsset[];          // 未产出 []；失败也可有已保存成果
  error: SdkError | null; notices: Notice[];
  writeback: "not_requested" | "pending" | "completed" | "failed";
  items?: BatchItemResult[];      // 普通任务省略；批量未返回前 []，返回后按 itemId 关联
};
type CancelReceipt = {
  taskId: Id; requestRecorded: boolean;
  execution: "stop_requested" | "stopped" | "continues" | "already_finished";
  message?: string;
};
type ReadEvents = { taskId: Id; cursor?: Id; limit?: PositiveInt };
type HostEvent = {
  cursor: Id; taskId: Id;
  event: { type: "snapshot"; value: TaskView }
       | { type: "audio_chunk"; media: AudioAsset }
       | { type: "notice"; notice: Notice };
};
```

- `getTask(TaskRef) → TaskView`、`cancel(TaskRef) → CancelReceipt`、`readEvents(ReadEvents) → Page<HostEvent>`。任务状态投影现有 `ai_tasks`（源码 `domain/entities/ai_task_entity.rs`），不保存第二份私有状态机；SDK succeeded 映射 completed，expired 映射 failed。fraction 在 0..1 或 null，不凭估算填进度。
- 事件 cursor 为宿主不透明续读标识，与 SDK sequence 不同。缺 cursor 从可用会话起点读；已过保留期给当前 snapshot 及新的 cursor，不重新推理。通知/流媒体是临时投影，任务和已保存成果可以重读。流块租约过期只标不可重播该临时块，最终成果仍走资产服务。
- taskId 及媒体读取都核权限。播放、原件另存、持久化和写回复用已有媒体服务，不在 TTS 合同建立第二套文件路径 API。失败后保留原音频，writeback=failed 可独立重试写回，不能重跑推理替代。
- cancel 记录请求不等于引擎停止。没有原生中断时 execution=continues，任务继续由真实结果收尾；不能提前释放 GPU/媒体租约或写成已取消。停止已确认才写 cancelled；自然完成先到则保留 completed。关面板只是退订。
- SDK 事件及终态通知进入现有任务刷新内核，不新增页面私有轮询。GUI/F1/MCP 所见模型、绑定、notices、任务及成果一致。

第一层最小提交示例（binding、资产身份均由本机目录实际产生）：

```json
{
  "schemaVersion": 1,
  "selection": { "bindingId": "binding-demo-1", "modelId": "voice-model-1" },
  "clientRequestId": "user-submit-001",
  "input": { "schemaVersion": 1, "text": "你好，这是一次普通语音合成。" }
}
```

## 3. 第二层 A：speech-v1 领域合同

### 3.1 操作、能力与发现

公共操作冻结为 `std/speech.synthesize@1`、可选 `std/speech.batch@1`、可选 `std/speech.voices.list@1`。均通过 SDK `submit`，capabilityKey=`audio.synthesize`、transportSlot=`speech`。后两项缺失不影响第一项；声音设计/克隆是 synthesize 的 mode，不另造传输操作。标准操作 ID 中的 `@1` 是文档版本记法，实际 JSON 中 id 与 version 分开。

三种操作的 input/output/resume schema 各有固定 `$id` 和原始字节摘要，跨模型完全相同。现有 SDK 对同 ID/version 连 resume 摘要也一起比较，因此三操作各自的同步/异步包均引用同一公共 resume schema，不能一家 null、一家私有 schema。`SpeechResume = null | {schemaVersion:1,token:Json}`，同步填 null；异步把原生不透明 token 包在 token 中，query/cancel 原样回送；该 schema 只验证外壳，任务归属仍靠真实 job ID、pin 和 checkpoint。resumable=false 仍可引用这个 schema，不虚构恢复能力。

拟发布目录 `packages/vendor-adapter-contract/v2/profiles/speech-v1/` 下冻结文件名 `synthesize.input.schema.json`、`synthesize.output.schema.json`、`batch.input.schema.json`、`batch.output.schema.json`、`voices.input.schema.json`、`voices.output.schema.json`、`resume.schema.json`、`descriptor.schema.json`、`configuration.schema.json`。每个 `$id` 为 `https://bgxiong.local/vendor-adapter/v2/profiles/speech-v1/` 加文件名；全部是本地 registry 标识，不是下载地址。合成 output 为 SpeechData|null，batch output 为 BatchData，voices output 为 VoicesData；非终态 data 的处理沿 SDK 不冒充成功结果。runtime/bundle 文件在 `packages/local-runtime-contract/v1/`，`bundle.schema.json`、`runtime.schema.json`、`control.schema.json` 的 `$id` 为 `https://bgxiong.local/local-runtime/v1/` 加文件名；runtime 定义 RuntimeSpec/LaunchConfig，control 定义就绪/身份/取消与管理端口联合类型。后续生成固定字节并注册；**本轮不编造未来文件摘要**。

可选 `SpeechDescriptor` 位于外层包 `models[].speech`，受 bundle 摘要保护；SDK RCD 闭合，不向其中私塞字段。公开作者工具读取同一 bundle/profile 库，与应用用同一 descriptor；缺描述时所有合法标准选填项仍可送 adapter 判断。模型只按已发布声明暴露额外控件，不猜厂商。

```ts
type Support = "supported" | "unsupported" | "unknown";
type Mode = "plain" | "voice_design" | "reference_clone";
type Range = { min?: number; max?: number }; // 有限，均含端点；同时有值需 min <= max
type FieldSupport = {
  pointer: Pointer; support: Support;
  numericRange?: Range;           // 只对数值字段，真实模型限制，不是质量偏好
  values?: Json[];                // 非空允许值集合；缺省不宣称闭集
};
type Requirement = {
  mode: Mode;
  field: "/reference/audio" | "/reference/promptText" | "/voiceId";
  reason: "model_requires_reference" | "model_requires_transcript" | "model_requires_voice";
};
type SpeechDescriptor = {
  schemaVersion: 1;
  modes?: { mode: Mode; support: Support }[];
  fields?: FieldSupport[];
  requirements?: Requirement[];   // 缺省 []，只支持上述真正推理前提
  languages?: string[];           // BCP 47 标签；省略未知，不自动翻译
  references?: { mimeTypes?: string[]; durationMs?: Range; requiresTranscript?: boolean };
  emotionVector?: { dimensions: PositiveInt; elementRange?: Range };
  outputMimeTypes?: string[];
  textLimit?: { unit: "unicode_scalars" | "utf8_bytes" | "model_tokens"; max: PositiveInt };
  runtimeProfiles?: { id: Id; label?: string; description?: string }[];
  extensions?: { namespace: Namespace; key: Id; schema?: SchemaRef; label?: string }[];
};
type SchemaRef = { id: Id; digest: Sha256 };
```

同一 pointer、mode、runtime profile id 不重复。FieldSupport 只约束选填偏好：未知、未消费或超出支持范围时报告未应用，不擅自 clamp、不改变业务选择。`requirements` 只在所选 mode 检查；禁止要求评分、说明、性能、UI文案、风格或情绪作为合成前置。references.requiresTranscript 只在实际提供参考且模型确需时检查；省略不是自动提供转写。实际未知限制可由 adapter 返回明确真实错误，不能通过无条件强制用户补齐全部参数“预防失败”。

execution 的同步/异步、流、取消和恢复仍取 SDK RCD，是传输事实；不在 descriptor 复制。`outputMimeTypes` 等描述坏件隔离并提示，不改变经验证的 RCD。用于权限/入口/资源选择的 bundle 核心记录有误则拒用该记录，不能将这些安全字段降成可忽略 metadata。

### 3.2 合成输入：完整字段字典

```ts
type SpeechInput = {
  schemaVersion: 1; text: string;
  mode?: Mode;                   // 缺省 plain
  voiceId?: Id;                  // 当前包/模型的声音标识；缺省引擎原有默认声音
  voiceDesign?: {
    description?: string; genderHint?: string; ageHint?: string;
    emotion?: string; speakingStyle?: string; speedHint?: string; controlPrompt?: string;
  };                             // 全为自由文字建议，无字段最短字数/写作规范
  reference?: {
    audio: MediaRef; promptText?: string; styleControl?: string;
  };
  emotionControl?: {
    mode: "none" | "inherit_speaker" | "text" | "audio_reference" | "vector";
    text?: string; audio?: MediaRef; vector?: number[]; alpha?: number;
    useRandom?: boolean; intervalSilenceMs?: UInt; maxTextTokensPerSegment?: PositiveInt;
  };
  language?: string;             // "auto" 或 BCP 47；省略模型自身默认，auto 请求模型检测
  instruction?: string;          // 用户独立表演指令，不强制并入 text
  rate?: { unit: "speed_ratio" | "duration_ratio"; value: Positive };
  volumeDb?: number;             // 有限 dB 增益偏好，0 为不变；不支持则未应用
  pitchSemitones?: number;       // 有限半音偏移，0 为不变
  seed?: U64Text;                // 不造默认随机种子；不能承诺跨模型同音频
  pronunciation?: { text: string; alphabet: "ipa" | "pinyin" | "custom"; value: string }[];
  pauses?: { afterScalar: UInt; durationMs: UInt }[];
  outputPreference?: {
    mime?: string; sampleRateHz?: PositiveInt; channels?: PositiveInt;
  };                            // 偏好，不是必须输出某编码；真实成果 MIME 权威
  execution?: { delivery?: "complete" | "stream"; runtimeProfile?: Id };
  extensions?: Extension[];
};
```

各选填项的补充语义：

- voiceDesign 的空对象合法，表示没有附加建议；只在该模型实际消费时应用。reference 省略表示无说话人参考；对象出现则 audio 必须是授权的音频媒体，promptText 是原录音转写，styleControl 是克隆后的表演建议，不允许把它当转写。voiceId 与 reference 同时存在：mode=reference_clone 以参考为声音来源，voiceId 未应用并提示；plain/voice_design 下参考仅在模型明确支持时使用，不偷偷变更 mode。
- emotionControl 省略表示不请求情绪控制；none 表示不加情绪控制；inherit_speaker 使用本次已选说话人信息，不隐式寻找文件。text/audio_reference/vector 在模型支持且选择该控制时分别读取对应字段；缺载荷只记该可选控制未应用并继续基础合成，不升级为整个 TTS 失败。其他载荷不读取并列未应用。vector 元素有限，alpha 是模型声明的混合强度、无跨模型统一范围；缺维数声明不推测某厂商固定维数。情绪参考与声音参考不可互相替代。
- rate：speed_ratio=2 请求二倍语速；duration_ratio=2 请求二倍时长。它们是相对模型常规速度的偏好，不保证精确时长；缺省不施加比例，非正数是结构错误。未支持不得自动改成另一单位，换算属于声明支持此语义的 adapter。
- pauses.afterScalar 是原 text 的 Unicode 标量边界，0 为文首，text 标量数为文末；相同边界不重复、按位置升序。pronunciation.text 是要读的文字；不替换原始文本，不保证模型有发音控制。无法采用时 notice。省略两数组即无额外指示。
- outputPreference 省略使用模型原生格式；缺单个子字段只是不指定它。宿主不因未满足偏好拒绝有效音频，列实际格式与未应用项；如用户另需转码，走明确的媒体操作。
- execution 缺省 `{delivery:"complete"}`；runtimeProfile 省略使用 bundle 规定的常规启动。显式未知 runtimeProfile 拒绝本次选择；不能默选快模式或另一个设备。没有性能估值也可尝试常规运行。
- extensions 无则 `[]` 语义。只允许与包发布者或 std 匹配的声明来源；未识别的合法扩展保留原意并列未应用。不支持的 namespace 不执行任意代码；扩展不得承载秘密、文件访问授权、进程入口或必须执行的资源条件。可选 schema 解析失败只禁用该扩展，不禁用 plain。

示例：克隆与风格（需要对应能力才执行；媒体实体另放 SDK media 数组）：

```json
{
  "schemaVersion": 1,
  "text": "欢迎回来，我们继续今天的故事。",
  "mode": "reference_clone",
  "reference": {
    "audio": { "mediaId": "speaker-1" },
    "promptText": "这是参考录音的原文。",
    "styleControl": "轻声、自然"
  },
  "emotionControl": { "mode": "text", "text": "温暖" },
  "rate": { "unit": "speed_ratio", "value": 1.1 }
}
```

### 3.3 完成结果与可选附件

完整单条合成的 SDK outputs 角色固定为 `audio`，恰好一个 kind=media、media.kind=audio 的完整原件；不能用 text/url/path 代替。SDK data 为 `SpeechData | null`；**null 合法**，模板无需伪造音频元数据。

```ts
type SpeechData = {
  schemaVersion: 1;
  attachments?: Json[];           // 缺省 []；每条独立解码，不能对整个数组做强语义拒绝
};
type Attachment = {             // 附件解码成功后的结构；原始条目始终保存
  kind: "audio_info" | "alignment" | "segments" | "parameter_receipt"
      | "performance" | "annotation" | "diagnostic" | "extension";
  mediaId?: Id; itemId?: Id;     // 存在时必须指向本次实际媒体/批量项
  payload: Json;
};
type AudioInfo = {
  sampleRateHz?: PositiveInt; channels?: PositiveInt; frames?: U64Text;
  durationMs?: number; codec?: string; language?: string; voiceId?: Id; seed?: U64Text;
};
type Alignment = {
  unit: "sentence" | "word" | "phoneme";
  entries: { startMs: number; endMs: number; text?: string;
    scalarStart?: UInt; scalarEnd?: UInt; confidence?: number }[];
};
type Segments = { entries: { id: Id; startMs: number; endMs: number;
  scalarStart?: UInt; scalarEnd?: UInt; label?: string }[] };
type ParameterReceipt = {
  applied?: { pointer: Pointer; value: Json }[];
  notApplied?: { pointer: Pointer; reason: "unsupported" | "out_of_range"
    | "not_applicable" | "unknown_extension" | "missing_optional_input"; message?: string }[];
};
type Performance = {
  firstAudioMs?: number; inferenceMs?: number; totalMs?: number;
  peakRamBytes?: U64Text; peakVramBytes?: U64Text;
};
type Annotation = {
  label: string; value?: Json; score?: number;
  scale?: { min: number; max: number }; source?: "model" | "adapter" | "human";
};
type Diagnostic = { message: string; code?: string; details?: Json };
```

Attachment.kind 对应 payload 依次为上述类型；extension 的 payload 为 `Extension`。所有可选数值存在时有限；时长非负，时间戳 0≤start≤end≤对应音频解码时长，标量范围来自原 text，confidence 在 0..1。时间轴从对应完整音频 0 开始，不用 wall clock；不同标注可以交叠，不凭自然语言语义拒绝。audio_info/alignment/segments 必须带 mediaId，否则只该附件 invalid。没有模型自报元数据，宿主仍从真实音频解码得到必要播放参数；矛盾时以原件为准并列 notice。

附件原始 JSON 条目可能没有 kind/payload 或包含未来 kind；分别记 `invalid`／`unknown` 并保留原始条目，不能让 outputSchema 把整个已完成音频拒收。valid 附件按上面的闭合类型使用。Schema 只约束 `data` 的最小外壳及 attachments 为 JSON 数组；SDK 核心输出身份/媒体安全仍严格校验。单条音频本体不可读才使此输出失败。

缺 ParameterReceipt 表示实际附加参数是否采用未确认；不默认为全部 applied。宿主合并 bind 已知未应用项并对尚未确认的已提交选填参数列 optional_unconfirmed。adapter 明知未采用时必须返回对应 receipt；公共模板代生成，不要求底层模型提供。appliedParams 可为 null；若有则仅记录 adapter 真实应用事实，不能替代最终请求审计。

示例：只有音频已足够成功；此例是 SDK completed 内部对象，句柄为示意，本机需由 broker 实际创建：

```json
{
  "kind": "completed",
  "vendorTaskId": null,
  "appliedParams": null,
  "outputs": [{
    "kind": "media", "role": "audio",
    "media": {
      "id": "audio-1", "kind": "audio", "handle": "media-handle-demo-1",
      "mime": "audio/wav", "fileName": "result.wav", "byteLength": "48044"
    }
  }],
  "data": null,
  "usage": null
}
```

示例：参数未采用和坏可选附件，音频仍可交付。第二条只产生 metadata_invalid：

```json
{
  "schemaVersion": 1,
  "attachments": [
    { "kind": "parameter_receipt", "payload": {
      "notApplied": [{ "pointer": "/rate", "reason": "unsupported" }]
    } },
    { "kind": "alignment", "mediaId": "audio-1", "payload": { "entries": "invalid example" } }
  ]
}
```

### 3.4 声音目录与批量

```ts
type Voice = { id: Id; label?: string; description?: string; languages?: string[];
  preview?: MediaRef; extensions?: Extension[] };
type VoicesInput = { schemaVersion: 1; cursor?: Id; limit?: PositiveInt };
type VoicesData = { schemaVersion: 1; items: Voice[]; nextCursor: Id | null };
type BatchItem = {
  itemId: Id;
  input: BatchSpeechInput;
  referenceId?: Id;               // 实际引用在根 references；允许 input.reference 留转写/风格
  emotionReferenceId?: Id;
};
type BatchSpeechInput = Omit<SpeechInput, "reference" | "emotionControl" | "execution"> & {
  reference?: { promptText?: string; styleControl?: string };
  emotionControl?: Omit<NonNullable<SpeechInput["emotionControl"]>, "audio">;
  execution?: { delivery?: "complete" };
}; // Omit 删除所列字段；NonNullable 取对象本体。其他所有字段/缺省与 SpeechInput 相同
type BatchInput = {
  schemaVersion: 1; items: BatchItem[];
  references?: MediaRef[];        // 缺省 []，mediaId 去重；单一根级媒体槽
  runtimeProfile?: Id;            // 整批固定；item.execution.runtimeProfile 不允许
};
type BatchItemResult =
  { itemId: Id; status: "succeeded"; mediaId: Id; data?: SpeechData }
  | { itemId: Id; status: "failed"; error: SdkError; data?: SpeechData }
  | { itemId: Id; status: "cancelled"; data?: SpeechData };
type BatchData = { schemaVersion: 1; items: BatchItemResult[] };
```

BatchItem.input 是上面明确的 SpeechInput 变体：reference 不再必需 audio，emotionControl 去掉 audio，execution 只许省略或 delivery=complete。公开 schema 必须生成该变体，不能用完整 SpeechInput 直接校验再私自 strip。referenceId/emotionReferenceId 指向根 references 的 mediaId，绑定后在包内复原单条语义；未引用的媒体不读取。items 至少一条、itemId 唯一，不能静默分批或固定业务条数。

声音目录无副作用，走 submit 完成 `completed.data=VoicesData`，outputs 默认空；preview 存在才以 role=preview 的音频输出关联并授权。label 省略展示 id；语言未知可省略；cursor 过期返回 INVALID_INPUT，仅影响本次分页，可重新从首页读。目录无能力时不伪造 std 操作。

batch 是引擎/运行时实际原生批量；应用逐条队列是多个单条任务，不冒称 std/speech.batch。全批固定 selection、pin、profile、complete 交付。结果按输入顺序，itemId 必须完整且唯一；所有成功项各有一个 role=audio 的实际 output 且 mediaId 不重复，失败项不虚构 output。batch outputSlots 允许 0..任意实际预算数量。SDK completed 只说明调用结束，父任务全成功才 completed，有失败为 failed，全 cancelled 为 cancelled；成功与取消混合为 failed 并保留成果。缺/重复/未知 itemId 属协议错误，保留可确认归属的媒体至恢复区，不自动误写给其他项或重跑。

示例：共享参考，不使用 SDK 不支持的 `/items/*/reference/audio`：

```json
{
  "schemaVersion": 1,
  "references": [{ "mediaId": "speaker-1" }],
  "items": [
    { "itemId": "line-1", "referenceId": "speaker-1", "input": {
      "schemaVersion": 1, "mode": "reference_clone", "text": "第一句。",
      "reference": { "promptText": "录音原文。" }
    } },
    { "itemId": "line-2", "input": { "schemaVersion": 1, "text": "第二句。" } }
  ]
}
```

### 3.5 流式、异步与错误

stream 是可选 delivery。v1 采用现有 `media_chunk`，每块为 **独立可解码 PCM WAV**（16-bit little-endian 或 32-bit float，由 WAV 头确定），同请求采样率/声道/样本格式不变；role=`audio`。音频位置按已解码帧数累加，禁止向闭合 payload 私加 offset、format。SDK sequence 从 1 严格递增，包含 progress/usage 等所有事件，不以音频块序号替代。宿主背压不丢块，每块消费完成才确认释放临时租约。

只有包和宿主都支持 stream-events-v1 且 RCD streaming=true 时接受 stream；不支持只拒该次显式流请求，complete 保留。最后仍需 completed/成功 query 的完整音频输出，声音样本应与按序 chunk 拼合相同，不能在末尾另生成一份不同音频。中断保存 partial 并明确失败/取消；EOF 不是成功。非 WAV 的完整输出合法，流式 PCM 要求只约束 stream，不能反向限制 complete 包。

同步 complete 不假造 vendorTaskId；真正异步运行时才 accepted，必须带真实 job ID 并在回执前 checkpoint。query 沿 SDK queued/running/succeeded/failed/cancelled/expired；成功 data 按本节对应操作类型，非终态可 null。cancel/query 复用已 pin 的 configuration 和 resumeToken，不重新 submit。默认不承诺本地进程崩溃或软件退出后续推；没有恢复能力不影响本次合成，只明确不能重启续办。

错误严格使用 SdkError；同步 `Response.error` 没有 details/data。异步失败使用现有 TerminalFailure（category 为 vendorTaskFailed/vendorTaskCancelled/vendorTaskExpired，message，vendorCode|null，details:object|null）。详细原生诊断保存在受管会话日志，不在宿主按 vendorCode 写特判。retryable 只说明用户可以重试，不授权自动重提；受理状态不明用 ACCEPTANCE_UNKNOWN，保留 checkpoint 和部分成果。

## 4. 第二层 B：独立分发包与受管 runtime

### 4.1 包结构与字段

归档根 `tts-bundle.json` 为新增外层描述；SDK adapter 子包仍是原有 manifest/hash/签名结构。所有资源与描述受归档摘要和 SDK 既有信任策略约束；签名状态与内容 hash 分开。zip 解包拒绝路径逃逸，禁止执行安装脚本和提权。说明/源码/许可证/报告不进入运行能力判定。

```ts
type Bundle = {
  schemaVersion: 1; packageId: Id; version: Id;
  adapter: { path: RelativePath; sha256: Sha256 }; // 指向标准 SDK pack 制品
  runtimes: RuntimeSpec[];         // 至少一项；id 唯一
  models: ModelSpec[];             // 至少一项；modelId 唯一且与 adapter RCD 匹配
  resources?: Resource[];          // 缺省 []；无外部权重的原生包无需虚构
  displayName?: string; description?: string;
  publisher?: { id: Id; keyId: Id }; // 只有外层已签名时提供；不能借adapter签名认证runtime
  documentation?: RelativePath; license?: string; sourceUrl?: string;
};
type ModelSpec = {
  modelId: Id; runtimeId: Id;
  resourceIds?: Id[];             // 缺省 []，仅当前模型真正依赖的资源
  revision?: string; speech?: SpeechDescriptor;
};
type RuntimeSpec = {
  id: Id; version: Id;
  os: "windows" | "linux" | "macos"; arch: "x86_64" | "aarch64";
  code: { path: RelativePath; sha256: Sha256 }; // runtime 目录内容清单路径及清单摘要
  executable: { source: "code"; path: RelativePath }
            | { source: "resource"; resourceId: Id; path: RelativePath };
  args?: string[];                 // 缺省 []，逐参数传递，禁止 shell 拼接
  resourceIds?: Id[];              // 缺省 []，解释器/动态库等公共依赖
  pythonAbi?: string;              // 用 Python 时声明；原生入口可省
  defaultProfile?: Id;             // 缺省 "default"，启动配置唯一且可定位
  profiles?: { id: Id; args?: string[]; resourceIds?: Id[] }[];
  readinessTimeoutMs?: PositiveInt; // 缺省 120000，真实加载需要可由作者调大
  concurrency?: PositiveInt;       // 缺省 1，安全串行租约，不固定业务条数
  cancel?: "none" | "request";     // 缺省 none；模型无需原生 cancel
  device?: "cpu" | "gpu" | "either"; // 缺省 either，宿主按实际可用和用户选择仲裁
  residentBytes?: U64Text;          // 可选估值，不作为静态显存硬拒条件
};
type Resource = {
  id: Id; version: Id; kind: "model" | "runtime" | "dependency";
  sha256: Sha256; byteLength: U64Text;
  source: { kind: "bundled"; path: RelativePath }
        | { kind: "download"; url: string }; // HTTPS、明确授权后下载
  license?: string; description?: string;
};
type CodeFile = { path: RelativePath; sha256: Sha256; byteLength: U64Text };
// code.path 文件内容：{ schemaVersion: 1, files: CodeFile[] }，排序不限、路径唯一
```

包 id/version 不可覆盖不同内容。下载引用与携带资源均需真实 digest/byteLength；缺资源可登记 needs_resources，不能称 ready。code 清单只涵盖执行树，不把可变缓存/日志/模型强制放进版本目录；模型资源安装到既有 models/speech_synthesis 托管树，执行/依赖在版本 runtime store。资源源文件是单件普通文件或独立资源包；资源包若需展开必须带现有离线资源安装器的清单/摘要与安全校验，不能由任意脚本解释。

外层归档同时携带 `bundle.sha256`，其逐文件格式复用现有 SDK hash inventory（包括 tts-bundle.json、adapter 制品、runtime代码/清单及所有随包文件；排除 inventory 自身和签名文件）。`bundle.sha256.minisig` 可选，签名算法和可信发布者校验复用现有 minisign-prehashed；publisher 与外层签名必须同时存在或同时省略。无签名按既有本地开发包信任/执行权限路径处理，身份用 local. 命名空间；没有签名不伪装已认证。只签 adapter 不代表 companion 已受信；新增外层 inventory/签名覆盖是 T04/T09 必须实现的最小机器完整性，不以作者报告代替。

runtime.profiles 省略时仅 defaultProfile 或 "default" 代表 base args；存在时必须包含默认项，选择项的 args **追加** base args，不做 shell 展开。资源集合是 runtime、model、所选 profile 的 resourceIds 并集，全部必须能解析。runtime 根为 code.path 所在目录，CodeFile.path 和 source=code 的 executable.path 相对该根；source=resource 的 resourceId 必须指向当前 runtime 已声明的 runtime/dependency 资源包，path 相对其经校验的安装根。工作目录固定 runtime 根，args 中相对脚本路径按该根解析，启动文件参数最后追加；禁搜索作者机器 PATH、猜解释器或跨资源逃逸。运行时 env 不由包任意注入，宿主仅提供白名单基础环境和下述启动文件；需要模型专用常量放包内配置。支持枚举内但与当前机器不匹配的 OS/arch 只令该 runtime 不适用，不执行错平台程序；枚举外值是未支持合同版本。首轮只验 Windows x86_64，其他平台声明不等于已实现验收。包/runtime version 使用 SemVer 字符串，资源 version/revision 是不透明来源标识；不因缺模型 revision 拒绝已具摘要的资源。

外层最小形状示例：假定原生 runtime 已内置其必需的小模型，无独立 resources。下面的全零摘要仅标示字段类型，**不是有效制品摘要，不能据此导入**；未来作者工具必须按实际字节生成。

```json
{
  "schemaVersion": 1, "packageId": "local.example.tts", "version": "1.0.0",
  "adapter": { "path": "adapter/package.zip", "sha256": "0000000000000000000000000000000000000000000000000000000000000000" },
  "runtimes": [{
    "id": "runtime-1", "version": "1.0.0", "os": "windows", "arch": "x86_64",
    "code": { "path": "runtime/files.json", "sha256": "0000000000000000000000000000000000000000000000000000000000000000" },
    "executable": { "source": "code", "path": "runner.exe" }
  }],
  "models": [{ "modelId": "voice-model-1", "runtimeId": "runtime-1" }]
}
```

### 4.2 最小握手与连接配置

宿主以受管方式启动本地包装器运行时：先取得驻留资源许可，再以隐藏子进程（Windows）启动包装器，并完成就绪确认与取消。包装器是第三方 runtime 的一部分，也可以包装没有现成健康检查或取消接口的模型。

**开源版已省略**启动配置中的会话秘密、凭证槽、内部管理端点、鉴权头格式、握手校验字段与配置键。公开合同只保留下列能力边界：

- 就绪必须以约定的就绪消息为准，不能把任意字符串当成已经就绪。
- 启动失败、超时或进程退出时，不得报告就绪，并应清理本次预留。
- 业务推理端点与请求形状由适配包与包装器自行约定，公共宿主不解释。
- 取消能力按包装器声明提供；若声明不支持，应给出明确的不支持结果，而不是假装取消成功。
- 用户不可覆盖由宿主注入的受管运行时连接参数。
- 会话秘密不得写入普通日志、AI 任务记录或报错文案。


### 4.3 管理端口与租约

管理端口供应用和独立参考宿主共用，不暴露给 WASM 随意 spawn；通过同一 Creator runtime 管理能力呈现，而非每引擎新增私有工具。

```ts
type RuntimeBinding = { bindingId: Id; modelId: Id; runtimeProfile?: Id };
type InstallState = {
  installed: boolean;
  resources: "not_checked" | "ready" | "missing" | "invalid";
  environment: "not_checked" | "ready" | "unavailable";
  process: "stopped" | "starting" | "ready" | "busy" | "stopping" | "failed";
  issues: SdkError[];
};
type RuntimeInstance = {
  instanceId: Id; ownerToken: Id; configuration: RuntimeConfiguration;
  residencyLeaseId: Id;
};
type RuntimeLease = { leaseId: Id; instanceId: Id; requestId: Id };
type ReleaseReceipt = { leaseId: Id; state: "released" | "pending_stop" | "already_released" };
type StopReceipt = { instanceId: Id; state: "stopped" | "stop_requested" | "busy" | "already_stopped" };
```

- `inspect(RuntimeBinding) → Reply<InstallState>`，纯检查不自动安装/启动。
- `ensureReady({binding:RuntimeBinding,requestId:Id,device?:"cpu"|"gpu"}) → Reply<RuntimeInstance>`；缺 device 交现有宿主资源仲裁一次确定，不能 GPU 失败后静默转 CPU。授权能力、取消信号、预算由宿主调用上下文提供，不是来自包的 JSON 权限声明。请求排队不等于启动成功。
- `acquire({instanceId:Id,ownerToken:Id,requestId:Id}) → Reply<RuntimeLease>`，仅 ready 且有空闲并发额度取得推理租约；忙时由现有调度排队，超出请求 budget 则 TIMEOUT，不能另起无界进程。
- `release({leaseId:Id,outcome:"completed"|"failed"|"cancelled"}) → Reply<ReleaseReceipt>`，幂等；只有确认本次推理已停才能 released，否则 pending_stop 保持实际占用，不因调用方等待结束释放其他请求资源。
- `stop({instanceId:Id,ownerToken:Id,reason:"idle"|"uninstall"|"user_stop"|"app_exit"}) → Reply<StopReceipt>`；idle/uninstall 有活动租约返回 busy。user_stop 只有显式同意停止本实例且不影响别的租约才可终止拥有的进程；app_exit 按现有应用退出规则清理本进程所拥有实例。停止等待不等于停止进程。

驻留租约在加载前取得、进程退出后释放；推理租约只覆盖对应请求。启动失败、崩溃需幂等回收自己持有的许可；普通 release 不释放仍驻留的模型。ownerToken 是管理器能力令牌，不传给 adapter，不给包凭它操作其他进程。无原生取消可等待完成，或在所有权及其他租约安全的条件下停止实例；不强杀共享 runtime。

安装/升级继续用现有受管安装事务与引用清理：staging 校验 → 登记完整版本 → 后续调用选择；缺下载资源记录待资源。每任务 pin 版本和资源，升级不改在途调用；旧包主动回选是用户选择，不是失败自动降级。卸载只删除没有活动引用的包代码，不删用户音频/历史/共享权重。

runtime 消息示例（由包装器产生）：

```json
{ "schemaVersion": 1, "type": "ready", "instanceId": "runtime-instance-1", "nonce": "startup-nonce-1" }
```

## 5. 第三层：公开 SDK 合同与作者交付

### 5.1 沿用的 SDK V2 机器合同

以下字段均已核对当前 schema；完整约束通过链接引用，不能手抄另建宽松版本：

- 「Invocation」（开源版未收录该内部文档或源码路径）：`protocolVersion:2, operation:六操作枚举, profileId:Id, capabilitySlot:槽枚举, requestId:Id, budgetMs:PositiveInt, payloadJson:string, bindingId:Id, negotiatedVersion:2, enabledFeatures:string[]` **全部必填**。bindingId=profileId，TTS capabilitySlot=speech；enabledFeatures 无则显式 `[]`。payloadJson 是序列化 JSON **字符串**，不是嵌套 object。
- 「submit/taskRef」（开源版未收录该内部文档或源码路径）：submit 全部必填 `schemaVersion:2, modelId:Id, capabilityKey:Id, operationId:Id, operationVersion:PositiveInt, inputSchemaDigest:Sha256, input:object, media:MediaInputV2[], configuration:object, uploads:{mediaId:Id,uploadRef:Id}[]`。无媒体/上传显式 []；configuration 在受管 runtime 场景是 RuntimeConfiguration。input 是本 SPEC 对应操作输入。query/cancel 全部必填 `schemaVersion:2,modelId,capabilityKey,operationId,operationVersion,vendorTaskId:Id,resumeToken:Json,configuration:object`；无续办数据写 null。
- 「MediaInputV2」（开源版未收录该内部文档或源码路径）：`id:Id, kind:image|audio|video|file, handle:Id, mime:Id, fileName:Id, byteLength:U64Text` 全必填且对象闭合。fileName 是 basename，无路径；禁止在此私加 duration、sha256 或 url。真正内容、MIME/扩展名由媒体 broker 核验，metadata 另走附件。临时 handle 不持久化为业务媒体路径。
- 「Response」（开源版未收录该内部文档或源码路径）：成功 `{protocolVersion:2,operation,dataJson:string}` 或失败 `{protocolVersion:2,operation,error:SdkError}`，互斥。不加 data/details/requestId；响应与当前请求由会话关联。
- 「Result」（开源版未收录该内部文档或源码路径）：completed 必填 kind、vendorTaskId:string|null、appliedParams:object|null、outputs、data:Json、usage:SDKUsage|null。accepted 必填 kind、vendorTaskId:Id、appliedParams、resumeToken:Json、usage。query 必填 status、outputs、data、usage、progress、terminalFailure、resumeToken；终态 progress=null，成功/非终态 terminalFailure=null，失败/取消/过期必须真实 TerminalFailure。
- 「Usage」（开源版未收录该内部文档或源码路径） 与 「StreamEvent」（开源版未收录该内部文档或源码路径） 原样复用。没有 usage 显式 null，不估算。应用本地推理不计云端厂牌扣费；云端若有真实 usage 才写现有合并入口。流 usage 是最新累计快照，不逐事件重复相加。
- 「控制操作」（开源版未收录该内部文档或源码路径）：capabilities 输入 `{}`，零网络；输出 schemaVersion=2、pluginId、pluginVersion、capabilityContracts 全必填。probe 的 package_validation 零网络，connectivity 显式检查已准备实例，connection/authentication/resources 分别 ready/failed/not_checked，不把 not_checked 当 ready。upload 沿现有 provenance 和媒体上传合同，不能用普通字符串伪造 uploadRef。

SDK Invocation 形状示例，此例 capabilities 不依赖未来 schema 摘要：

```json
{
  "protocolVersion": 2, "operation": "capabilities",
  "profileId": "binding-demo-1", "capabilitySlot": "speech", "requestId": "request-1",
  "budgetMs": 30000, "payloadJson": "{}", "bindingId": "binding-demo-1",
  "negotiatedVersion": 2, "enabledFeatures": []
}
```

SDK 错误示例；message 是展示信息，不是程序判别式：

```json
{
  "protocolVersion": 2, "operation": "submit",
  "error": {
    "code": "RESOURCE_UNAVAILABLE", "message": "本次选择所需的模型资源尚未安装。",
    "retryable": false, "vendorCode": null, "vendorRequestId": null
  }
}
```

### 5.2 RCD、媒体槽、注册及可选能力协商

「CapabilityContractV2」（开源版未收录该内部文档或源码路径） 的 schemaVersion=2、localId、capabilityKey、transportSlot、非空 modelIds、configurationSchema、非空 operations、非空 schemas 全必填。SchemaRef 是 id+digest；schemas 每项为 `{id,digest,schemaJson:string}`。digest 按原始 UTF-8 无 BOM 字节计算，不重排或 Unicode 归一化。所有引用闭合在本地已校验 registry，不能临时联网拉 schema。

operations 每项必填 id、version、label、inputSchema、outputSchema、resumeSchema:SchemaRef|null、uiSchema、mediaSlots、execution、outputSlots。label 可由模板生成稳定操作名，不要求作者设计 UI。本目标合同允许 uiSchema `{controls:[]}`，控件只是提示；**现有 JSON schema 虽允许空数组，validator/input.rs 的 required_controls 仍会拒绝缺必填字段控件，属于 G10 待修缺口，不能声称现有 check 已接受**。execution 全必填 `mode:synchronous|asynchronous,streaming:boolean,cancellation:remote|local_only|unsupported,resumable:boolean,remoteIdempotency:boolean`；模板按真实最小同步实现输出，不把未提供当作功能已支持。speech 操作的 resumeSchema 按 §3.1 固定公共引用，不使用不同包的私有摘要。

媒体槽与输出槽固定规则：

- synthesize 输入 role=speaker、pointer=/reference/audio，以及 role=emotion、pointer=/emotionControl/audio；kind=audio，minCount=0、maxCount=1。是否所选 clone 真需参考由 Requirement 和 adapter 判断，不能把 minCount=1 放成所有 plain 的硬门。
- batch 输入 role=reference、pointer=/references、kind=audio、minCount=0、maxCount=null。SDK 不支持通配符，只使用根数组；每条只能 `{mediaId}`，不能塞转写等字段。
- synthesize output role=audio、kind=audio、minCount=maxCount=1；batch 同角色 minCount=0/maxCount=null，结合逐项结果判定；voices output role=preview、kind=audio、minCount=0/maxCount=null。
- voices 无输入媒体槽。input.media 及 outputs 使用受管 handle，schema 不授予文件/网络权限。缺可选参考时媒体槽计数为零，plain 正常。

operationId 必须 namespace/name。`std/speech.synthesize` 等需要先在 `protocol/vendor-adapter-protocol.json` 的 standardProfiles 注册版本与公共 schema 身份，并让 validator 与发行 schema 摘要核对，不能仅靠不同包彼此一致。包不得改变同名同版 std input/output schema 来强制某家选填条件；专属扩展放 descriptor/Extension，不造第二个同名合同。

只有依赖实时流的调用需要协商 `stream-events-v1`；基础包不能把它放 requiredFeatures。stream 能力放 optionalFeatures，未启用时禁止流端口但 complete 仍可调用，RCD 包声明保持不变，宿主 effective capability 根据协商取交集。当前 SDK 是否完整容许这一组合要在 T03 定向验证；若 validator 将 streaming=true 错误提升为整包要求，只修该公共协商缺口，不让作者拆包/伪报 streaming=false。runtime/speech 支持范围是外层合同版本，不往 SDK feature 闭集写未注册字符串。
### 5.3 SDK 必需补充：出站前审计与实时媒体桥

当前 PortObserver 只有 started、completed_port、finished，不能完成最终请求出站前持久化。冻结新增公共宿主交互语义如下；它是 SDK 工具／宿主边界，不是业务请求字段、不是第七类传输操作、不让第三方写生产数据库。

```ts
type BeforeSend = {
  schemaVersion: 1; type: "before_send"; exchangeId: Id; requestId: Id;
  callIndex: PositiveInt;           // 本 request 的实际受管 HTTP 调用序号
  operationId: Id;
  method: "GET" | "POST" | "PUT" | "PATCH" | "DELETE" | "HEAD" | "OPTIONS";
  url: string;                     // 脱敏后，不能泄露 query 凭据
  body: Json;                      // 凭据剔除；二进制/音频以媒体 id/digest/长度描述
  text: string[];                   // 实际请求的全部文字，保持内容/顺序，不摘要/截断
};
type AuditAck = { schemaVersion: 1; type: "audit_ack"; exchangeId: Id;
  result: { persisted: true } | { persisted: false; error: SdkError } };
type MediaBridgeRequest = { schemaVersion: 1; type: "media_read"; exchangeId: Id;
  requestId: Id; handle: Id; offset: U64Text; maxBytes: PositiveInt };
type MediaBridgeReply = { schemaVersion: 1; type: "media_read_reply"; exchangeId: Id;
  result: { bytesBase64: string; eof: boolean } | { error: SdkError } };
type StreamDelivery = { schemaVersion: 1; type: "stream_event"; exchangeId: Id;
  event: SDKStreamEventV1 };
type StreamAck = { schemaVersion: 1; type: "stream_ack"; exchangeId: Id;
  result: { consumed: true } | { error: SdkError } };
```

SDKStreamEventV1 精确引用现有 stream-event.schema，不新增字段：requestId、sequence、type、payload；media_chunk 的 payload 只有 role 与 MediaInputV2。这些新增宿主消息需独立版本化工具消息 schema，JSONL 中依 type 判别；final SDK Response 原样保留，不能把任意 JSON 行当最后结果。exchangeId 在会话内唯一，确认只作用于同一会话；失联/错误按请求预算显式结束，不自动 retry 模型调用。

工具通道选项为 `--host-channel-v1`（**S0 CLI 2.0.2 已实现 before-send；media-stream 执行未实现**）。启用后 CLI 首行发 `ChannelHello={schemaVersion:1,type:"channel_hello",features:ChannelFeature[]}`，宿主回 `ChannelAccept={schemaVersion:1,type:"channel_accept",enabledFeatures:ChannelFeature[]}`；`ChannelFeature="before-send"|"media-stream"`，数组去重、enabled 为双方交集且必须包含 before-send。S0 可只实现 before-send，不要求 media-stream；无参数的独立作者 CLI 使用内部日志确认，不等应用应答。只协商 before-send 时结束仍输出现有原始 Response 并走完整媒体导出；协商 media-stream 时按 FinalDelivery/FinalAck 完成接管。宿主未接受未知协议版本时报 UNSUPPORTED_VERSION，不能忽略握手后出站。channel features 不是 SDK manifest features，不能互相混填。现有 `--host-checkpoints` 请求/确认保持原格式和回执前持久化语义，通道调度显式识别 hostCheckpoint，不吞掉或重复处理。新消息 schema 发布为 `schemas/host-channel-v1.schema.json`（$id=`https://bgxiong.local/vendor-adapter/v2/host-channel-v1.schema.json`）。

出站前必须先对实际请求做脱敏审计投影，确认已经落盘后再发送。开源版已省略凭证注入位置与实现。应用同时持久化 logs/ 与 ai_tasks.last_payload；独立作者宿主只写本次受管会话日志即可，无生产 DB。审计失败零出站。多次调用按 callIndex 保留，任务详情关联本次真正合成请求，不能以健康探测覆盖最终文本。

`text` 从实际请求正文提取：S0 默认提取全部非敏感字符串；未来包可提供公开审计投影，宿主不猜厂商键名、不拼情绪括号。公共出站记录同时保留可审计 body；文本映射不得选择性漏掉用户/模板的实际文字。此投影属于包内转换且与准备出的 HTTP 请求同源，不另调 LLM。任何在 companion 内完成的用户可见模板拼接必须前移到 adapter 的准备过程，保证所记文字就是实际给模型的文字；tokenize 等模型内部算法不作为新用户提示词。

审计投影本身也是冻结的公开作者字段：`AuditProjection={schemaVersion:1,textPointers:Pointer[],binaryPointers:Pointer[],secretPointers:Pointer[]}`。该可选增强在后续版本的 `host-ports.schema.json#/$defs/httpRequest` 增加 `audit:AuditProjection`；S0 不接受该字段。实际 SDK 使用由 schema 生成的 typed WIT，增加字段会改变其类型结构，不能宣称只经 JSON 便无 ABI 影响。按用户避免过度设计的要求，本次只实现省略投影的基础审计，不把投影升级作为普通合成门槛。三个数组均必填，可空。省略 audit 时公共库提取全部非敏感正文字符串，二进制用受管描述，秘密从 broker 来源剔除；不以缺可选投影拒绝可审计请求。

Pointer 相对于实际 HttpRequest.body：JSON 正文按其结构，multipart 按 SDK parts 数组索引，文本正文指向其文本值，不能将同名 multipart 字段合并成字典而丢值。textPointers 顺序为显式展示顺序，未映射的非敏感正文字符串仍完整保留；binaryPointers 只指向真实二进制/媒体，不得掩盖文本；用于标记敏感值的指针只能指向真正的秘密，不能用来隐藏用户提示词。开源版不展开秘密来源的配置与注入。无正文检查可用空数组。投影不是宿主执行代码，不向当前闭合 schema 私塞字段；应用和独立工具共用新版已注册结构。

实时桥在 worker 活跃时读媒体、确认消费：收到 stream_event 后可在 stream_ack 前发 media_read；调度不能阻塞读服务而等待 ack 形成死锁。media_read 受当前 request 句柄授权和 SDK 块预算限制，offset/字节数按真实内容；consumed 确认后释放该临时块。最终 outputs 在消费者接管前仍可读取，不能随着 CLI 退出立即销毁；保持会话到最终媒体接管确认，复用相同 stream_ack 机制时需使用明确 `FinalDelivery={schemaVersion:1,type:"final_response",exchangeId:Id,response:SDKResponseV2}` 与 `FinalAck={schemaVersion:1,type:"final_ack",exchangeId:Id,result:{adopted:true}|{error:SdkError}}`，不得把终态包装误读成业务事件。工具独立调用继续支持既有最终一行 Response，桥模式单独协商。

审计和 complete 结果接管是基础必要条件；实时事件/读块/背压只在 S4 流任务完成。工具采用已支持的本地媒体导出完成 complete 接管时不要求提前实现流桥。任何新工具消息 schema/flag 必须先发布再调用，本文没有给当前 CLI 伪造可执行选项。

### 5.4 第三方收到及交回的内容

我方公开作者开发包必须提供：固定 SDK/CLI/worker 版本；SDK schema 与操作 registry；本 SPEC 对应的 speech/runtime/bundle/schema 文件及摘要；单源生成类型与 profile 校验/未应用回执库；最小 plain 模板；通用 runtime 包装器/独立参考宿主；出站审计接合；标准 build/check/pack 使用说明；最小输入/仅音频/可选缺失/坏附件/权限反例 fixture。不能依赖私有 Tauri 源码或生产数据库才能开发。

第三方最小交付：tts-bundle.json、标准 SDK adapter 制品、所选平台可执行 runtime、真正必需资源及真实摘要。只实现文本到音频的模型，不必交克隆/流/batch/声音目录/评分/性能/UI控件。README、来源许可、升级说明建议随发行提供，但不用于执行器判定基础合成可否运行；开源/商业交付要求另按双方约定，不强制交私有模型源码。

作者独立执行与应用执行用相同合同、默认值、未应用回执和媒体行为。现有作者 runtime 目录列 rust/profile，但 `reference-host/src/profile.rs` 明确拒绝非 text 输出槽和 streaming，**首个 TTS adapter 使用 Rust author runtime**；不要求先扩张声明式 profile。Python/CUDA 是 companion，不宣称已有可运行的 Python WASM guest。现有 CLI 有 init/build/check/pack/caps/probe/upload/submit/query/cancel/record/replay/conform，没有 prepare 命令；“准备请求”是 adapter 内纯阶段，不能写成不存在的 CLI。精确未来命令只能随实现后的工具版本交付。

## 6. 现有 SDK 与应用缺口核对

以下 G01–G11 是 S0 前的缺口基线；S0 已补 std 摘要注册、三层 schema/类型、空 UI 解耦、before-send 与作者模板。字段绑定/未应用回执的运行逻辑仍随 T10 接入，runtime、应用及模型仍未实现；不把 schema 存在当作功能完成。具体已验结果以 PLAN S0 记录为准。原基线合同 release `2.0.0` 的 registry 还声明 `applicationSupport=NOT_ADOPTED`，不能据其文件存在宣称 TTS 已被应用接入。

1. **G01 标准语音 profile/注册：基础必需。** `protocol/vendor-adapter-protocol.json` 的 standardProfiles=[]；`crates/validator/src/identity.rs` 按 namespace/name 校验；`capabilities.rs` 有同 ID/version 的 schema 签名一致检查。需发布三操作和公共 schema，并核对登记的固定摘要；不能写 dotted std ID，不能接受各包篡改公共字段。T01/T03。
2. **G02 speech 语义库、可选 descriptor、模板：基础必需。** 现有 RCD 能承载 schema，但没有本 SPEC 的标准缺省、应用/未应用回执、独立附件解析或外层 model descriptor。需单源实现，不在 UI/Rust/CLI 三处各写一次；坏附件不升级成音频失败。T01/T02/T10。
3. **G03 可等待出站前审计：基础必需。** `crates/reference-host/src/dispatch.rs` 的 PortObserver 无 before-send，`crates/tools/src/run.rs` 日志不能代替出站前确认；需公共请求投影、审计请求/ack，应用接 logs 与 last_payload。覆盖普通/流式同一 HTTP 出口，但实时播放可后做。T03/T11。
4. **G04 动态 runtime/bundle：基础必需。** `schemas/manifest.schema.json` 闭合，`host-feature-catalog.json` 无 spawn；`src-tauri/src/runtimes/registry.rs`、`local_ai/speech_synthesis/engine_registry.rs` 静态引擎，`tts_launcher_sync.rs` 嵌入文件。需外层安装合同、受管包装器/租约和动态目录，复用 store/sidecar，禁止向旧 manifest 或 TtsEngineId 逐家加字段。T04/T09/T12。
5. **G05 UI/Creator 接入与成果接管：基础必需。** `speech/synthesis_request.rs`、`tts_params_intake.rs` 目前有旧模型字段和硬能力限制；需按本用户要求将合法不消费选填项变为 notice，保留真实选择/引用错误。`creator_tools/tts_execution.rs` 共用，完整装配/读取回归必做。CLI临时媒体须实际导出到授权应用 writer，再结束会话；现有 process.rs 能跑 CLI 不等于已实现此 TTS 接管。T10–T14。
6. **G06 事件/流媒体桥：仅流增强。** SDK 有 `StreamEventV1`，`crates/tools/src/run.rs` 写 events.jsonl；应用 `adapter_host/v2/process.rs` 只特殊处理 hostCheckpoint，其他 JSON 行作为最终 response，不能据此边生成边播。需分帧、活跃句柄访问、读写复用、ack/背压/取消及最终接管；当前 schema 可用独立 WAV chunk，不必先扩 offset 字段。T18/T19。
7. **G07 原生批量：可选独立。** SDK 通用媒体校验 `crates/validator/src/media.rs` 使用精确 pointer，引用对象只有 mediaId；按根 references 设计，无须加 wildcard。逐项结果、部分失败保存和父任务映射需实现。T20，不依赖流桥。
8. **G08 取消/恢复：依真实声明。** 已有 cancel/query/checkpoint 不等于同步在途 native 推理可取消；包装器与所有权协调需实现，缺原生取消仍可完整合成。本地持久化续推没有证明前声明不支持，不挂远端恢复名单。T04/T07/T13，增强另行记录。
9. **G09 可选 feature 协商：基础兼容定向核对。** 现有 feature catalog 只有 stream-events-v1，`validator/src/package.rs` 已按宿主支持集合取 optionalFeatures 交集；需定向验证连到 worker/端口后 RCD streaming 在未协商 stream 时仍能 complete，不能错误封整个包。公共业务 profile/运行管理不注册成臆造的 SDK requiredFeature。T03；未实现流本身仍留 S4。
10. **G10 UI提示与执行校验耦合：基础必需。** `validator/src/input.rs` 的 required_controls 会拒绝没有对应 UI 控件的必填输入，validate_contract 还会把不兼容的可选控件作为全局错误。这与 UI 可独立变动、无控件声明仍可合成冲突。需在候选 SDK 把 UI 渲染检查分离为非阻断诊断，标准 input/schema/权限校验仍保留，CLI/F1 无 UI 同样能执行；不能为通过门禁伪造一套必需控件。T03 与既有行为用例匹配修正。
11. **G11 声明式 profile 的音频限制：已有明确边界，无需基础修正。** `reference-host/src/profile.rs` 在构建时拒绝媒体输出槽及 streaming，虽然 feature catalog 同时列 rust/profile。T02/T05 用 Rust adapter，companion 可 Python；不给第三方提供会被现有宿主拒绝的纯 JSON 音频模板。日后扩 profile 独立排期，不阻塞基础包。

以上都不要求新增厂商错误码、宿主引擎 if/else 或全量 SDK 重构。当前无需外部网页即可核对本仓冻结源码；未根据“最新网上 SDK”改写本仓协议。

## 7. 合同冻结、验收与本次自审

### 7.1 C0 → 实施 → 发行的顺序

**C0 文档合同冻结（本轮）**：本文字段、类型、缺省、单位、三层职责、可选项与例子经自审，入主仓形成版本锚点。只完成设计合同，不等于 schema 已发布，也不授权继续开发。

**获得开发指令后 S0**：T00 只复核相对本基线变化；T01 把本文生成正式 schema/类型/fixture，T03 发布 registry/必要公共工具修正，再 T02 作者模板。机器化不得重新设计字段；发现无法表达只停止对应缺口，修订本文后再继续。生成的原始 schema 字节/摘要和 SDK 发行版本另冻结，不能借文档冻结假造当前不存在的 hash。

**S1 及以后**：固定已验 SDK/profile/runtime 基础，再做消费者；H1 基础导入/普通合成/播放/保存，B包升级与第二引擎证明独立；流/batch 缺失不阻断 H1 或基础安装版。字段存在不代表模型支持，未测试不写通过。

### 7.2 必须留到实施验证的最小行为例

- 只 text+schemaVersion、空 controls、data=null、缺 descriptor/目录/评分/性能仍能完整合成。
- 同一选填 rate/情绪/扩展换到不支持模型，原意保留、notice 可见；不拒普通功能，不谎称已采用；显式 clone/stream/voiceId 不可用只拒当前选择。
- clone 的真实参考/转写前提只在所选模式检查；越权 mediaId、摘要错配仍失败。未知能力允许显式尝试，不因 unknown 封模型。
- 非法 alignment 单独失效而音频保留；非法核心音频拒用。batch 部分失败保留成功项，根参考无通配符。
- 出站前审计失败零 HTTP；无生产 DB 的作者运行可用；原文、包内最终模板文字和实际调用可对应。
- H1 全组件不改，导入能力不同的包可用；两种 UI 组织生成同一请求。未协商 stream 的包仍可 complete。
- 加载前资源仲裁，没原生取消可正常完成；中止等待不提前释放驻留许可。安装、资源就绪、真实合成和人工音质分别记录。

### 7.3 自审修正与限制

已在设计中处理：SDK ID/信封字段闭合；可选结果不被全局 schema 拒掉；参考角色和批量根槽；缺目录不强制 voiceId；没有 metadata 不伪造值；参数未知不误报采用；流块用现有闭合事件；SDK completed 与业务全成功区分；GPU驻留/推理租约分离；无取消不封包；模型更换不要求 UI/宿主特判；审计和媒体桥必须请求/确认、不能只写事后日志；三层不复制 Creator/任务协议。

本次文档检查：9个JSON示例均可解析，其中已有SDK的Invocation、completed结果、错误Response共3例通过现有Draft 2020-12 schema校验；SPEC/PLAN/需求相对文件链接无缺失；C0、六个实施阶段、25个唯一任务及14项验收与本计划一致。自审另核现有源码中的UI强门、声明式profile媒体限制及resume摘要账本，已修正文档。只检查本轮文档与既有schema，未运行产品、编译、安装、模型或SDK行为验收。示例中的id/handle和包例子中的占位摘要不是真实制品；机器schema、类型生成、摘要注册和SDK发行均留待开发指令后完成。
