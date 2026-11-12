> **历史演进产物，不代表当前设计，不作为当前实现或开发依据。**
>
> - 原路径：`release/03-开发指南/GUIDE-VendorAdapter-开发者手册.md`
>
> 现行架构请从仓库主导航进入查阅；本文只作为历史演进资料，不作为当前实现或开发依据。

# GUIDE · 第三方算力适配器（INV-VENDOR-ADAPTER）

> **开源整理版**（2026-11-12）：已去除或改写私有路径、未公开端点、凭证示例，以及登录 / 认证 / 付费相关实现细节。若某节全部属于上述范围，将改为一句能力边界说明。


权威设计：`docs/dev/plans/20260830T163700+0800-PLAN-长尾厂商适配器公开API-架构审核版.md`。
第一波 as-built：`docs/dev/plans/20260904T002800+0800-PLAN-长尾厂商适配器落地缺口修复与开箱可用-架构审核版.md`。
**全量落地（本轮已接活路径）**：`docs/dev/plans/20260904T053500+0800-PLAN-长尾厂商适配器公开API全量落地补齐-架构审核版.md`。
**宿主/运行时解耦（2026-10-02 起的代码结构，以此为准）**：「PLAN」（开源版未收录该内部文档或源码路径） · 「SPEC 宿主执行与 Worker 内部合同」（开源版未收录该内部文档或源码路径） · 「需求」（开源版未收录该内部文档或源码路径）。下文凡与这三份冲突，以它们为准。

## 编写路径

| 路径 | 入口 | 执行 | 状态 |
|---|---|---|---|
| profile | `profile.json` | 唯一 `HttpPort::execute_plan` | **可用** |
| Rust | WIT `invoke` | 未签名 crate：设置页选文件夹即编译；或预编译 `adapter.wasm`。hello WIT 只是 L2 | **源码导入即编译已验收**（`init rust` 生成匿名 crate；T0=本机 rustc/`cargo-component`/`wasm32-wasip2`，不进安装包；L2 hello ≠ guest probe ≠ 可生成） |
| TinyGo | WIT `invoke` | — | **暂不支持**（不可导入、不可选） |
| Python | 解释器组件 | — | **暂不支持**（无 CDN 制品，不可导入） |
| TypeScript | 解释器组件 | — | **暂不支持**（同上） |

禁止：pip / npm / 真实文件系统 / `socket` / `subprocess`。

## 包与职责（A47 后）

| 包 / 目录 | 是什么 | 不许有 |
|---|---|---|
| `src-tauri/crates/bgx-adapter-core` | 纯逻辑：协议常量、invocation/response、错误码、manifest/能力槽/RCD、执行上下文与用途、`policy`（出网判定纯函数）、worker 协议 DTO、`conformance` 判定、`redact` | IO（fs/net/process）、tokio、Wasmtime、宿主 |
| `src-tauri/crates/bgx-adapter-runtime` | Wasmtime 引擎/Store/Linker、WASI 虚拟化、guest 与 profile 执行器、worker server；bin **`bgx-adapter-worker`** | reqwest、rusqlite、tauri、宿主 |
| `src-tauri/crates/bgx-adapter-transport` | 帧编解码、`WorkerClient`（起 worker、握手、取消、超时）、broker 会话公共块、媒体分块/落盘、作者服务（compile/pack/verify/sign）、隐藏子进程 | Wasmtime、tauri、rusqlite、reqwest |
| `src-tauri/crates/bgx-adapter-tools` | bin **`bgx-adapter`**（开发者 CLI）、DevBroker（live/record/replay）、transcript、conform | Wasmtime、宿主、对 runtime 的库依赖（只按进程启动 worker） |
| `src-tauri/src/adapter_host/` | GUI 生产装配：`execution`（single-flight、submit 串行、关机取消）、`worker_client`、`broker/`（出站请求与任务媒体等；开源版已省略凭证与签名实现）、`task_recording`、`package_management/`、`trust/`、`bridges/`、`resume/`、`results/`、Tauri `commands` | Wasmtime（宿主 `wasmtime` 依赖只服务 `content_skill_host`，与 adapter 无关） |

依赖方向由 `scripts/check-workspace-crates.ps1` 强制，违例即失败。旧 `vendor_adapter_gateway`、`dev_cli.rs`、GUI exe 内的隐藏 worker 入口均已删除；sandbox gate 锁住它们不得复活（A43/A47 段）。

## CLI

`bgx-adapter`（`bgx-adapter-tools` 的 bin）是独立可执行文件，不链接 Wasmtime，也不链接 GUI；要跑 guest 时按进程启动 `bgx-adapter-worker`，与 GUI 用同一个 worker、同一套 worker 协议。命令：`init|check|pack|sign|build|dev|record|replay|conform|trust-sign|trust-check`；退出码 0 成功、1 校验或运行失败、2 用法错误（含不支持的 init 语言）。`pack` 写 `hash relative/path`；`sign` 写 `manifest.sha256.minisig`；`dev` 真 invoke；`record` 真出网并把每次往返写成 transcript；`replay <transcript>` 校验 fixture，带包目录时离线重跑（零网络）；`conform` 对 fixture 做 replay 和 hello，合同目录有 `fixtures/same-source` 时再跑同源用例，禁止只数文件。

```text
cargo build -p bgx-adapter-runtime --bin bgx-adapter-worker   # 先放好 worker（同一 target 目录）
cargo run -p bgx-adapter-tools --bin bgx-adapter -- init profile
cargo run -p bgx-adapter-tools --bin bgx-adapter -- init rust
cargo run -p bgx-adapter-tools --bin bgx-adapter -- check .
cargo run -p bgx-adapter-tools --bin bgx-adapter -- pack .
cargo run -p bgx-adapter-tools --bin bgx-adapter -- build .
cargo run -p bgx-adapter-tools --bin bgx-adapter -- dev rust
cargo run -p bgx-adapter-tools --bin bgx-adapter -- record <包目录> out.json
cargo run -p bgx-adapter-tools --bin bgx-adapter -- replay out.json <包目录>
cargo run -p bgx-adapter-tools --bin bgx-adapter -- conform
cargo run -p bgx-adapter-tools --bin bgx-adapter -- trust-sign payload.json
cargo run -p bgx-adapter-tools --bin bgx-adapter -- trust-check payload.json payload.minisig
```

安装版：`bgx-adapter.exe`、`bgx-adapter-worker.exe` 与 GUI 同在安装根，合同树在安装根 `vendor-adapter-contract/v1`，CLI 不需要任何环境变量。

### 定位 worker 与合同目录

- worker：`BGX_ADAPTER_WORKER_EXE`（显式，指向不存在的文件直接报错）→ 调用方（GUI 或 CLI）同目录的 `bgx-adapter-worker`。不搜 PATH，不回退到 GUI exe。找不到时报「未找到适配器运行组件…（开发环境请 cargo build -p bgx-adapter-runtime --bin bgx-adapter-worker；安装版请重新安装）」。`cargo tauri dev` 下 worker 必须构建到与 GUI 相同的 target 目录。
- 合同目录（CLI 的 `init rust` 模板、`conform` 的 fixture 与同源用例）：`BGX_ADAPTER_CONTRACT_DIR` → `bgx-adapter` 旁的 `vendor-adapter-contract/v1` → 本源码树；都没有就报错并列出查找位置，不静默回退。
- 打包：唯一入口 `scripts/build-adapter-binaries.ps1`（`build-before.ps1` Step 2d 调用，先于任何宿主 cargo 编译），覆盖配置 `src-tauri/tauri.adapter-bundle.conf.json` 带 sidecar 与合同树；主 `tauri.conf.json` 不含 `externalBin`，日常 `cargo check` / `tauri dev` 不需要先暂存 sidecar。

### 定位错误

- 握手失败（版本不符、拿错 exe、输出不是 worker 协议）：`adapter worker handshake failed: the executable is not a matching bgx-adapter-worker build (rebuild it, or reinstall the app); <细节>`，分类 protocol，厂商零请求。重建当前树的 worker，或重装。
- worker 崩溃：错误里带退出码与有界 stderr 尾部；Windows `0xC0000409` 多为 guest trap（见下文 RCD-20 / fuel / memoryBytes）。
- 超时：hello 墙钟、guest 墙钟（epoch）、静默 worker 都会被杀进程并给出分类，不会挂住 GUI。
- guest 内部：worker 日志里 `ADAPTER_WASM_INVOKE stage=enter/returned/error`。
- 给用户的 toast 是人话，技术码只进日志（`user_messages_never_leak_machine_codes`）；日志经 `redact` 脱敏，不出现原始密钥。

### 开发 broker 与生产 broker

两者都调用 `bgx_adapter_core::policy` 做出网判定，并共用 transport `broker_session::prepare_guest_http` / `http_reply`，所以同一请求在 CLI 与 GUI 的放行/拒绝一致（同源用例 `fixtures/same-source` 两条路径各跑一遍）。区别：

| | 生产（GUI，`adapter_host::broker`） | 开发（CLI，`bgx-adapter-tools::dev_broker`） |
|---|---|---|
| 用途 | Production（先写任务与 prompt 记录再出网，记录失败即不出网）、Polling、Diagnostic（probe / caps 刷新，不写任务记录） | Development：不打开生产 DB，不写任务记录 |
| 访问凭证 | 生产环境由宿主代管，适配器不内嵌密钥。开源版已省略凭证句柄、签名算法、模式字段与拒绝码等实现细节 | 开发 CLI 不使用生产凭证链路。开源版已省略具体拒绝策略 |
| trust / denylist | 受 trust 名单与 denylist 管理 | 不受 trust 管理，无 denylist；allowlist、受保护头、禁 redirect 同生产 |
| 网络 | 统一经 `crate::outbound_http` | live/record 经 `dev_broker/http.rs`（ureq + rustls，固定已判定 IP）；replay 零网络 |

### 新增一个能力槽的检查单

1. core：在 `capability_slot.rs` 登记槽名（`all_capability_slots`；桥接通后再进 `RELEASE_BRIDGED_SLOTS`），必要时扩 `query_observation` / `rcd`；只改内部合同，公开 WIT/SDK 字段不动（要动就另立公开 API 计划）。
2. 宿主：`adapter_host/domain/capability_slot.rs` 的 OPEN 清单、`bridges/<槽>_provider.rs` 桥、`invoke_bridge` toast；`open_slots_match_inventory_and_registry`、`only_implemented_slots_have_bridges`、`slot_without_bridge_is_rejected_at_manifest_validation` 必须一起改绿。
3. 合同：`packages/app-contracts/v1/vendor-adapter.json` 与 vendored SDK 副本同 hash；需要示例时放 `packages/vendor-adapter-contract/v1/examples/`，并补同源用例 `fixtures/same-source/*.json`（GUI 与 CLI 各跑一遍）。
4. 结果与恢复：异步任务走 checkpoint（accepted 前不交付）、`resume/query`、`results/finalize`；终态失败要保留 `terminalFailure`。
5. 门禁：`check-workspace-crates.ps1`、`check-vendor-adapter-sandbox-ssot.ps1`、ratchet；未支持的槽显式 `UNSUPPORTED`，禁止静默跳过。
6. **提交映射**：对照官方 apiCallDemo 与 UI dump，列出增量覆盖时必须每次提交的权重闭集（CLIP/UNet/VAE/采样器等）。漏钉 = 不合格（INV-VENDOR-ADAPTER-MAPPED-PIN）。宿主 `payloadJson` 没有厂牌 `nodeInfoList`。

随包案例（职责一人一义，禁止合成一个 crate 的 App/V2/Comfy 双解析）：

| 路径 | 角色 | HTTP | pluginId |
|---|---|---|---|
| `packages/cloud-adapters/runninghub-h3-dynamic/` | 独立 SDK v2 动态图音参考 | 完整 API 图提交与任务查询 | `local.thirdparty.runninghub-h3-dynamic` |
| `packages/cloud-adapters/runninghub-minimax-h3/` | MiniMax H3 **r2v 1/2/3/4-slot + 单图参考 turbo + 首帧 fl2va + 音频对口型** OpenAPI V2 AI App（每次 submit 钉 CLIP/UNet/VAE/采样器；独立 SDK v2 1.0.1） | `/openapi/v2/run/ai-app/{id}` + `/query` + upload | `local.example.runninghub-minimax-h3` |
| `packages/vendor-adapter-contract/v1/examples/comfyui-lan-minimax-h3/` | 局域网 Comfy **H3 t2v / turbo 单图 / turbo 多参** | `GET /system_stats` + `POST /upload/image` + `POST /prompt` + `GET /history/{id}` | `local.example.comfyui-lan-h3` |
| `packages/vendor-adapter-contract/v1/examples/comfyui-lan-z-image-turbo/` | 局域网 Comfy **Z-Image-Turbo 文生图** | `GET /system_stats` + `POST /prompt` + `GET /history/{id}`（无 upload） | `local.example.comfyui-lan-z-image` |
| `packages/vendor-adapter-contract/v1/examples/lan-openai-compat-text/` | 局域网 **OpenAI 兼容文本** | `GET /v1/models` + `POST /v1/chat/completions`（同步 completed；禁 query） | `local.example.lan-openai-compat-text` |

V1 的 LAN 案例安装包切片仍在 `user-examples/vendor-adapter-v1/`。上表两个 RunningHub 工程是独立 SDK v2 包，源码不依赖宿主，运行包仅含 manifest、预编译 wasm 和 inventory；安装时选择各工程 Build.ps1 输出的安装根目录（`adapter-install.json` 与 `package/` 同级）。用户在宿主中配置访问凭证，不可编进源码或 wasm。开源版已省略凭证引用字段与注入方式。旧 `runninghub-workflow` 教学标本和 V1 H3 源码已退役，不再随 V1 切片分发。

经典七工作流保留原七个 modelId 和 AI App ID，安装后在能力映射／视频工作台重新选择本包声明的模型。每个变体需要准确的 1/2/3/4 张图，音频对口型还需要 1 段参考音；时长 1～15 秒，固定原生 24fps，画幅 1:1、16:9、9:16、4:3，MP 0.25、0.4、0.6、1。具体工作流、上传及节点映射只在独立包，不进入 SDK／宿主。工作台参数必须原样双向传递；不靠省略素材保留厂商图中旧输入。

独立包最低离线验收是原生参数／提交／查询回归、标准 SDK build、source check、pack 与 runtime check。正式检查通过不代表云端已出片；禁止 CI 写「RunningHub 已接通」。安装后的真实素材与云端生成由用户自行验证。LAN Comfy 与 LAN 文本案例仍按各自 README 的入口验证，不能复用其它厂商包的节点或访问配置。

### 提交映射：整图 pin vs 增量覆盖

宿主只给 HOST_PAYLOAD（prompt、时长、画幅、参考图 handle）。厂牌 HTTP 形状只允许写在 guest。

- **增量覆盖**（RunningHub AI App `nodeInfoList`）：请求是「底图默认值 + 列出的字段」。没写进列表的节点跟 **该 App 当前保存的默认值** 走，不是作者网页上刚跑通的那一次，也不是别人账号里的默认值。凡 CLIP / UNet / VAE / 采样器漂移会导致形状或权重错误的，guest **每次 submit 都必须钉死** `nodeId` + `fieldName` + 字符串 `fieldValue`。禁止发超长 `fieldData`。禁止「权重留在厂牌图上」当 mapped_run 策略。上传写入 LoadImage 的值是响应里的 `data.fileName` 原样（含 `openapi/` 前缀）。`download_url` 是公网下载地址，不是 Comfy 输入路径，禁止拿它或剥掉前缀后的裸名去填 `image`。
- **整图 pin**（局域网 Comfy `POST /prompt`）：提交完整 API 图字节，权重已经在 pin 里；guest 只 patch 工作台输入。不要把云案例节点号抄进 LAN 图。

对照该 App 的 apiCallDemo 与 crate `workflows/*.ui.json` 建钉表。一槽一张表，禁止跨 modelId 抄节点。改钉表必须升 `manifest.json` / `PLUGIN_VERSION` 并 cover-import。CI 仍禁止写「已接通」。

先例：`packages/cloud-adapters/runninghub-minimax-h3/`（INV-VENDOR-ADAPTER-MAPPED-PIN）。旧教学标本已删除；新经典包的七个模型各自保留原权重钉表，见独立工程 `docs/HOST_MAPPING.zh-CN.md`。

```text
cargo run -p bgx-adapter-tools --bin bgx-adapter -- init rust
cargo run -p bgx-adapter-tools --bin bgx-adapter -- dev rust
cargo run -p bgx-adapter-tools --bin bgx-adapter -- conform
```

未签名 rust crate：设置 → 第三方适配器 → 导入插件包 → **选 crate 根目录**（`Cargo.toml` + `manifest.json` + `wit/` + `vendor-sdk/` + `src/`），不要选 `src`。本机还要有 T0：`rustc`/`cargo`、`cargo install cargo-component`、`rustup target add wasm32-wasip2`（不进安装包）。装完须重启客户端，GUI 进程才能看到用户 PATH。客户端隐藏调用 `cargo component build --release --target wasm32-wasip2`，只把 `manifest.json` / `adapter.wasm` / `manifest.sha256` 装进应用数据目录。作者树不写回 wasm。已签名包必须自带预编译文件，不会重编。缺 T0 或编译失败：toast 人话；设置页同时展示编译器原文。**导入 L3 的空 `probe` 禁止 `host-http`**（`live:false`）；厂商探活只在宿主注入 `endpointBaseUrl` 之后。L2 hello 绿、导入 L3 绿都不是可生成。

卡片身份是 `pluginId@version`，不是文件夹名。两份 crate 要并存，必须改 `manifest.json` 的 `pluginId` 以及 guest 里的 `PLUGIN_ID`；同身份再导入会先问再覆盖，不会多出一张卡。设置页双击卡片名称只改宿主 `display-alias.json`，禁止改已装 `manifest.json`。

`dev` 与客户端是同一个 `bgx-adapter-worker`，引擎标志相同（runtime `engine.rs`：fuel〔Windows 关闭，见下〕+ epoch + component-model）。`STORE_MAX_CORE_INSTANCES` 是 core instance 上限（component 嵌套至少 2），不是「同时一个插件」。用户 `adapter.wasm` 的 parse/instantiate/invoke 以及 L2 hello 都跑在独立隐藏子进程 `bgx-adapter-worker`（GUI 与 CLI 进程都不含 Wasmtime）。Wasmtime abort 只死 worker。私钥不进本仓。

World 只 `use types.{invocation, response}`，`Operation` / `AdapterError` 不在 guest 根模块。`wit_bindgen::generate!` 之后须 `use bgxiong::vendor_adapter::types::{AdapterError, Operation};`。禁止按 rustc 提示改成 `Option::Capabilities`。

Guest 必须 `no_std`（wasm32）+ `dlmalloc` 全局分配器 + `wasm_mem.rs`（`memcmp`/`memcpy`/`memmove`/`memset`/`cabi_realloc`）。这四个 C 内存原语必须**显式逐字节循环**；`slice` 排序或 bulk-memory intrinsic 在 wasm32 上会降级回同名符号并递归，`capabilities` 插入 `Map` 键时必炸。标本与 rust 模板两份 `wasm_mem.rs` 必须相同。`wasm32-wasip2` 的 libstd 会导入 `wasi:io/poll` 等，Host 硬拒一切 `wasi:`。`wit-bindgen` 须关 default（禁 `std`/`async`）。禁止为了过编译去开 WASI 或加 `wasmtime-wasi`。

### 签名信任名单

1. 按 `packages/vendor-adapter-contract/v1/schemas/trust-list.schema.json` 写 **精确** `payload.json`（camelCase；`publishers` 至少 1 条；不要 pretty-print 后再签）。
2. 运维机设置 `BGX_ADAPTER_MINISIGN_KEY` 指向本机 `seckey.json`（`keyIdHex` + `seedB64`；**禁止提交**）。
3. `bgx-adapter trust-sign payload.json` 把 minisig 打到 stdout，原样贴进云端 `/adm/vendor-adapter-trust`。
4. 客户端用内嵌根公钥 `BGX_VENDOR_ADAPTER_TRUST_ROOT_PUBKEY_B64` 对 `payloadJson.as_bytes()` 验签。云端 Java **不验签**。

### 根钥轮换

新根公钥必须随**客户端发版**写入 `src-tauri/keys/vendor-adapter-trust-root.pubkey`（`build.rs` 缺文件即编译失败）。旧客户端会拒收新根签的名单。先发客户端，再在 /adm 发新根签名单。禁止复用 updater 公钥。

## 运行期能力协商（INV-VENDOR-ADAPTER-RCD）

宿主只收 `packages/vendor-adapter-contract/v1/schemas/runtime-caps.schema.json`。第三方 guest 的 `capabilities` 必须返回这份文档，禁止指望宿主认识某一家工作流图或节点 class。

每个可调参数三态：`fixed`（焊死，控件只读）/ `range`（min/max/step/default）/ `enum`（options）。缺省按未声明处理。区间由 guest 自己负责；宿主不编造、不收窄。未知字段硬拒。

缓存：`{app_data}/vendor-adapters/_caps/{pluginId}/{version}.{fingerprint}.json`。指纹是 `rcd_cache_fingerprint(endpointPolicy, baseUrl)`，不含 slot。`staticHosts` 端点段为空（改凭证地址不换文件）；`userConfigurable` 端点段是规范化后的凭证地址，地址为空则刷新失败、不写 `default`。同版本只留一份文件。安装成功不依赖刷新成功。无有效缓存时工作台标 `capsNegotiated=false`，按静态 manifest 投影。提交时按同一把钥匙读；文件不在就用当前凭证再刷新一次，仍没有则提示到设置点「探测」，不会把缺文件说成旧插件。

凭证：adapter 通道与后端 `service_override_sufficient` 同构——模型 id 为 `adapter:{pluginId}:{vendorModelId}` 即可提交预检；不要把用户赶到「设置 → AI 服务商」填云厂牌 Key。

### 图像尺寸 enum 与画幅（INV-VENDOR-ADAPTER-RCD-21~25）

图像槽用 `resolution` **enum** 声明可调尺寸，每个 option 必须带 `aspectRatio`（闭集：`16:9` / `9:16` / `1:1` / `4:3` / `3:4`）。宿主投影为生图工作台**左画幅、右清晰度**两列；**禁止**把视频 RCD-16「有 resolutionOptions 就藏画幅」抄到图像。

- submit wire 字段是 `size`（`WxH`），对应 RCD `resolution` 与回包 `appliedParams.resolution`。不要另声明 `aspect` 参数。
- 图像槽禁止 `resolution.mode=range`。
- `fixed` / 无 RCD：工作台藏两列，提交不下发 `size`。
- SDK：`RcdEnumOption::new(...).with_aspect("16:9").to_json()` 再交给 `RcdParam::enum_of`。

### 参数声明与应用证明（INV-VENDOR-ADAPTER-RCD-11 / 12）

宿主**不读取**任何厂商工作流 JSON。换平台、换 API 时，能映射哪些时长 / fps / 画幅，只能由插件自己声明并自证。

1. **声明即绑定**：`range` / `enum` 必须带非空 `bindingId`。Rust SDK 用 `ParamBinding::new` + `RcdParam::range` / `RcdParam::enum_of`；没有「无绑定却可调」的构造函数。绑定为空则该参数不得出现在 RCD，工作台隐藏，走厂商图焊死值。
2. **应用证明**：`submit` 成功体必须带 `appliedParams`（`duration` / `fps` / `resolution` / `aspect` 中实际写入厂商请求的键）。宿主比对工作台所选值；缺失或不一致则人话失败，日志 `ADAPTER_APPLIED_PARAMS_MISMATCH`。禁止声明可调却静默丢掉。
3. 诚实边界：宿主无法证明 guest 是否对厂商撒谎。能消灭的是「控件可选但请求里没有」的假绿。厂商侧是否真改了节点，靠插件作者按 GUIDE 填绑定 + 厂商 `promptTips` 显式失败。

### no_std alloc 导入与 wasm 真编译（INV-VENDOR-ADAPTER-RCD-17 / 18 / 19）

guest 是 `wasm32` `no_std`。`String` / `Vec` / `format!` / `vec![]` / `to_string()` **不会**从 std prelude 进来。宿主 `cargo test` 全绿不等于 `cargo component build --target wasm32-wasip2` 能过。每个用到这些符号的 `.rs` 必须写对应 `use alloc::…`（与 `config.rs` / `runninghub.rs` 同一写法）。`capabilities` 返回 `Result<Value, String>` 时，`caps.rs` 必有 `use alloc::string::String`。

门禁对**随包案例**都跑 `cargo test --manifest-path`。本机有 T0（`cargo-component` + `wasm32-wasip2` rust-std）时，对两案例与 rust 模板在**临时目录**跑真 component 构建，并做 capabilities roundtrip。缺 T0 只打印 `SKIP … (NOT verified)`，禁止写成已验证。生成的 `src/bindings.rs` / `target/` / `adapter.wasm` 不得写回作者树或安装切片。

### capabilities 分段观测与真 component 回归（INV-VENDOR-ADAPTER-RCD-20）

`probe` 能返回、不代表 `capabilities` 能返回。guest panic_handler 是 `unreachable()`，Windows 上会 Fast Fail **worker**（`0xC0000409`），GUI 进程隔离必须保留，禁止在 GUI 里 `catch_unwind`。

诊断顺序（禁止先猜 `json!` / 手写第二套 RCD）：

1. Host：`ADAPTER_WASM_INVOKE stage=enter` → `returned`（带 `data_json_len`、剩余 fuel）或 `stage=error`。禁止记录 RCD 正文。
2. Guest `capabilities` 用显式 `serde_json::to_string`（失败映射 `adapterBadOutput`）。**不要**在 capabilities 里调 WIT `host_log`：Windows + Wasmtime 28 异步 Host 返回后 `out_of_gas` 会 Fast Fail worker，连 `CAPS_ENTER` 之后都到不了 framed 错误。probe 仍可 `host_log`（hello fixture 已覆盖）。宿主 `ADAPTER_WASM_INVOKE stage=enter/returned/error` 是 capabilities 的分段观测。
3. 有 T0 时必须对**新鲜** `cargo component build` 产物做 runtime 进程内 invoke（`guest::tests::specimen_capabilities_component_roundtrip`）与独立 worker framed invoke（`worker_client::tests::specimen_capabilities_worker_framed_roundtrip`）；不得 `include_bytes!` 陈旧 wasm，不得用宿主侧解析冒充。

`_caps/{pluginId}/{version}.{fingerprint}.json` 只在 `validate_runtime_caps` 成功后写入。无 RCD 时工作台继续藏档，禁止 catalog 静态窗回填。

Rust 档 `fuel` 必须覆盖 component instantiate + `capabilities` 的 no_std serde 指令数。50M 会在 guest 进入后触发 Wasmtime `out_of_gas`。**Windows + Wasmtime 28** 上 **任何** wasm trap（`out_of_gas` / `memory.grow` 失败 / guest `unreachable`）都走 `longjmp` 并 Fast Fail worker（`0xC0000409`），到不了 framed 错误。因此宿主在 **Windows 关闭 `consume_fuel`**，用 epoch + `StoreLimits` 约束 guest；非 Windows 仍开 instruction fuel（400M）。Windows 上 **`adapterFuelExhausted` 不可达**，超时由 epoch 走 `adapterTimeout`。

Rust 档 **`memoryBytes` 必须 ≥ 32 MiB**。16 MiB 时 instantiate 后 capabilities 的 serde/堆增长会 `memory.grow` 失败，同样 Fast Fail。T0 在 32 MiB 下 `capabilities` `stage=returned`。权威合同 `packages/app-contracts/v1/vendor-adapter.json`（vendored SDK 副本必须同 hash）。

非 Windows 的 runtime `guest.rs` 在 `instantiate_async` 成功后必须再 `set_fuel(policy.fuel)`。instantiate 可能把实例侧 fuel 指针打成 0，而 `Store::get_fuel` 仍显示整罐。

W4（关 fuel + 32 MiB）必须编进**当前 `bgx-adapter-worker`**（与 GUI 同目录的那一份）并重启客户端后再导入 crate 根才算验证。只重编 guest wasm、或工作区已改码但安装根/target 里仍是旧 worker，日志会继续出现 `0xC0000409`，**不得**写成现场已修复。

**submit 带图（INV-VENDOR-ADAPTER-SUBMIT-MEDIA）**：参考/首尾图不得以 Base64 进入 wasm `payload_json`。视频附件走 `video-reference-media-v1` 宿主暂存 + guest `host_http` multipart handle。未协商且仍带图 → 进 worker 前硬拒（升级/重导入）。Windows 上 trap 仍不可 framed；epoch 在 host-http 等待期间照常推进墙钟，合法慢上传走超时，**禁止**靠加大 `memoryBytes` 掩盖内联图。

## 协议

- 区间：`packages/app-contracts/v1/vendor-adapter.json` 的 `protocolMin`/`protocolMax`，N-2 窗口。
- model：`adapter:{pluginId}:{vendorModelId}`；bindingId 走 `profileId`。
- 未签名插件必须 `local.` 前缀。

## 信任

minisign 验 `manifest.sha256`。云端名单 pull 见已落地 PLAN `docs/dev/plans/20260902T150130+0800-PLAN-适配器信任名单云端pull-架构审核版.md`。根钥只验名单；插件只信 `publishers[]`。无成功 pull 时签名插件显式 `adapterPublisherUntrusted`，未签名 `local.` 仍可（宽限期内）。
