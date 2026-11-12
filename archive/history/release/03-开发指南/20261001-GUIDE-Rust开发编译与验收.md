> **历史演进产物，不代表当前设计，不作为当前实现或开发依据。**
>
> - 原路径：`release/03-开发指南/20261001-GUIDE-Rust开发编译与验收.md`
> - 文档标识：取自原文件名 `20261001-GUIDE-Rust开发编译与验收.md`（这是原文件名中的标识，不是本次整理推定的写作或生效日期）
>
> 现行架构请从仓库主导航进入查阅；本文只作为历史演进资料，不作为当前实现或开发依据。

# GUIDE · Rust 开发编译与验收

> **开源整理版**（2026-11-12）：已去除或改写私有路径、未公开端点、凭证示例，以及登录 / 认证 / 付费相关实现细节。若某节全部属于上述范围，将改为一句能力边界说明。


**开发/人工/发行 EXE 均须遵守「EXE与数据库合同配套交付」（开源版未收录该内部文档或源码路径）**：源码须包含升级库的完整生产/消费合同，记录实际 EXE 身份，以一致性副本核实际 EXE 的 Current 与 Creator/工作流/片段读取。编译成功不等于启动或可用；不能将已升级库再次转换来适配漏合代码的 EXE。

数据库生命周期遵循[INV-DB-SHORT-TXN全仓规范](./20261028-GUIDE-数据库短事务与文件先落盘规范.md)：库外完成文件/模型/网络和等待，短事务登记；验收要检查外层owner，证明耗时阶段独立连接仍可写。非Send约束和启发式搜索均不能单独证明事务短。

开发态仅启动已有产物（2026-10-21）：根目录 `开发态免编译启动.bat` 连接正式云端，直接复用 `src-tauri/target/debug` 中主程序及 SDK CLI/worker，仅运行已有 EXE 启动 GUI；旧库转换使用独立升级工具，不在免编译入口执行；不调用 Cargo/Tauri CLI、不重新准备本地推理资源。Vite 读取当前 UI 源码，复用本仓已运行服务或按需创建，退出仅停止本次创建的 Vite。缺产物、端口被其他服务占用、同一 EXE 已运行或数据库合同不匹配会明确退出。Rust 改动须先用原开发入口构建一次；免编译入口会显示 EXE 时间，不承诺当前 Rust 源码已被编入。可传 `-TargetDir <绝对路径>` 指定其他 target；默认沿用 `CARGO_TARGET_DIR` 或仓内默认路径。`-ValidateOnly` 检查必要文件与端口/进程，存在同仓Vite时还实查其监听地址的HTTP；不迁移、不启动、不构建。Vite不存在时不宣称HTTP已验。

- 状态：现行（2026-10-01，Rust 提速 PLAN S04/S05/S07/S08/S12/S15/S17 产出）。计划 「PLAN」（开源版未收录该内部文档或源码路径） · 台账 「执行台账」（开源版未收录该内部文档或源码路径） · 基线 「S03 基线矩阵」（开源版未收录该内部文档或源码路径）。
- 规则摘要同步在 `.cursor/rules/agent-shell-batch-execution.mdc`「单任务验证节奏」与 `.cursor/rules/release-preflight-sync.mdc`「Agent 交付清单」。INV-PREFLIGHT-DEV 分层（L-fast / L-compile / L-heavy）不变。

## 1. 先看数字（本机基线，i9-10980XE，rustc 1.94.0，中位数）

| 动作 | 耗时 |
|---|---|
| 空跑 `cargo check --bin bgxiong-ai-story-tauri`（FRESH） | ≈5 s |
| 改一个业务 .rs 后 `cargo check --bin …` | ≈89–94 s |
| 改一个业务 .rs 后 `cargo test --bin … --no-run` | ≈136–221 s（取决于改动位置） |
| 只改测试代码后 `cargo check --bin … --tests` / `test --no-run` | ≈109 s / ≈209 s |
| 已构建后跑 filtered 测试 | ≈4 s（测试本身 0.02 s） |
| `check:affected`（1 个 Rust 文件，fast+compile） | ≈221 s（S03：clippy ≈121 s + cargo check ≈86 s）→ **D3 后 131 s**（cargo check 由同一轮 clippy 满足，单样本） |
| `check:affected:heavy` | ≈428 s（S03：clippy ≈121 s + check ≈85 s + 全量 test ≈201 s + 其它）；D3 后 G3 check 同样可由 clippy 满足 |
| `check:affected`（仅 UI 文件） | ≈15 s，不触发 Rust 编译 |

所有热构建里**只有主 crate 一个单元**在重编（依赖与 lib 全部 Fresh）；等待来自“这个 crate 很大”和“同一轮里重复跑 test-build/affected”，不是依赖失效。

> **2026-10-29 注**：上表是 10-01 基线，已不代表现状（10-13 观测 bin 测试增量 12–14 分钟，10-28 观测 `cargo check` 6–23 分钟）。主要新增原因：主 crate 编译身份随升版、`Cargo.lock`/依赖清单变化而改变，增量缓存作废后各模式冷编。现状、复现命令与处置见 「提速 PLAN v2」（开源版未收录该内部文档或源码路径） §2 与阶段 B；新基线由该计划 B0 重测后回填本表。日常改动不要顺带升版或改依赖清单。

## 2. 单任务验证节奏（S04）

1. **同一逻辑任务中的小步反馈**：只跑
   ```powershell
   cd src-tauri
   cargo check --bin bgxiong-ai-story-tauri          # 改业务代码
   cargo check --bin bgxiong-ai-story-tauri --tests  # 改了测试代码
   ```
   不要每保存一次就跑 `check:affected` 或 `cargo test`。实测（S08）：同一任务 3 次小修改，旧节奏（每次 related test + affected）1125 s → 新节奏 634 s（−43.7%），见 「S08 报告」（开源版未收录该内部文档或源码路径）。
2. **任务最终树**：按**完整任务 diff** 前台跑**一次** `cd ui && npm run check:affected`；命中 heavy 路径或改了 `Cargo.toml` 时跑 `npm run check:affected:heavy`。交付回复引用 stdout 的 `PREFLIGHT_LAYER=…`。
3. **失效条件**（任一发生都要重跑所需检查，heavy 路径仍 heavy）：最终验收失败后的修复；开始新任务；接口/合同/配置改变。
4. **禁止**：用任务途中某次已绿的结果冒充最终树已绿；新增或使用任何跳过 compile/heavy 的开关（没有，也不得加）。

## 3. 测试 target 与缓存合同（S05）

- 主 bin 的 check / test **一律** `--bin bgxiong-ai-story-tauri`、默认 features、默认 dev/test profile、同一个 target-dir（仓内默认 `src-tauri/target`）。换 features/profile/target-dir 会让主 crate 整体重编。
- **feature 集已统一（D2，2026-10-01）**：tauri-cli 2.x 的 `dev_options` 总是传 `--no-default-features` 并把 `default` 成员展开为显式 feature，因此只要 `[features]` 里存在 `default` 键，`tauri dev` 与 `cargo check/test` 的 feature 集就必然不同。现在 `src-tauri/Cargo.toml` 不定义 `default`，原 `local-ai-text-only`（dev/发行恒开启）的 candle/hf-hub/tokenizers 改为常驻依赖，`tauri.conf.json` 不再设 `build.features`。实测：`cargo check --bin …` 后再用 `tauri dev` 形式 `--no-default-features` 为 Fresh（2.5 s），主 crate 产物共用。仍可选的 feature：`local-ai-whisper-inprocess`、`merge_shadow_audit`（开启它们才会另编一份）。
- 相关测试：`cargo test --bin bgxiong-ai-story-tauri <filter>` 只减少**执行**数量，不减少编译范围（仍构建整个 test harness）。必须看 `test result:` 行的实际运行数量；**0 个测试不算通过**（S02 测量工具对 0 个会以 exit 3 失败）。
- 不要先 `cargo test --no-run` 再 `cargo build`/`cargo test` 重复构建；`cargo test <filter>` 本身会先构建。
- 不带 `--bin` 的 `cargo build` / `cargo test <filter>` / `cargo check --bins` 会把 `src/main.rs` 以 `bgx-adapter` 名义**再完整编一次**（本机首次 `check --bins` 该单元 305 s，见 S18）。开发与门禁命令都要带 `--bin`。
- 路径选择：

| 场景 | 小步反馈 | 任务收尾 |
|---|---|---|
| 普通业务修改 | `cargo check --bin …` | 相关 filtered test 一次（记录数量）+ `check:affected` 一次 |
| 只改测试代码 | `cargo check --bin … --tests` | 该测试 filter 一次 + `check:affected` 一次 |
| 合同 JSON / build.rs | `cargo check --bin …`（build.rs 会重跑；非法合同直接失败） | 对应合同测试 + `check:affected`（按映射） |
| heavy 路径 / `Cargo.toml` | `cargo check --bin …` | `check:affected:heavy` 一次 |
| 发版 | — | `npm run check:release-preflight`（不属于日常节奏） |

## 4. 精准 diff 范围（S07）

- 默认 `check:affected` = `-IncludeUnstaged`，扫描**整个工作树 diff**；工作树里若有别的任务的脏文件，会顺带触发它们的检查。
- 只验收本任务时用现有 `-Files`（不新增等价入口），**列全**本任务所有新建/修改/删除/重命名文件及必要合同：
  ```powershell
  powershell -NoProfile -ExecutionPolicy Bypass -File scripts/preflight-affected-by-diff.ps1 `
    -Files src-tauri/src/a.rs,src-tauri/src/b.rs,ui/src/x.ts            # 普通
  powershell -NoProfile -ExecutionPolicy Bypass -File scripts/preflight-affected-by-diff.ps1 `
    -Files src-tauri/src/a.rs -IncludeHeavy                              # heavy
  # 只看会跑什么：加 -ListOnly
  ```
- 引擎会打印完整文件列表与将运行的检查；跨文件门禁（如 quality-ratchet、SSOT 扫描）仍按其规定目录扫描，精准范围不会让任务相关门禁变少。禁止凭扩展名排除属于本任务的文件。
- S06 起每步输出 `PREFLIGHT_STEP layer= id= exit= ms=`、`DOD_STEP …`，末尾 `PREFLIGHT_TIMING total_ms=`；需要机器可读时加 `-TimingJsonPath <路径>`（禁止写进 `logs/`）。

## 5. 进程清理范围（D1，2026-10-01 起）

2026-11-03：已删除 `_invoke-cargo-native.ps1`、发行预检和 Creator runner 按 target-dir 扫杀测试进程的旧逻辑。同一目录不能证明进程归属或失去响应；任何 Cargo 调用前后都不得因此结束别的活跃测试。仅允许明确拥有的进程对象/PID在本次任务超时或收尾时清理，无法证明归属则保留并报告。现有精确主 EXE busy 保护继续生效。实际误杀证据与自测见 「FIX」（开源版未收录该内部文档或源码路径）。

若本命令会重写主程序，且 package/bin、target-dir、target triple 和 profile 能证明唯一输出路径，该 EXE 正在运行时显式报 `CARGO_MAIN_BIN_BUSY`，保留进程。其他 checkout、target 或 profile 的 GUI 不受影响；`check`/`clippy`/`test --bin …` 不因此阻断。target-dir 优先采用 `--target-dir`、`CARGO_TARGET_DIR`，其余由 `cargo metadata` 解析；环境相对路径按本次工作目录解析，metadata 失败的默认目录不能作为主程序归属证明。配置或输出位置无法证明时，不猜测主 GUI 路径，仍执行原 Cargo 并保留其真实错误。此规则针对自动残留清理；显式启动的自有子进程超时处置仍按原调用合同执行。自测：`scripts/test-invoke-cargo-native-scope.ps1`。

## 5b. affected 内 clippy 与 cargo check 去重（D3）

同一次 `check:affected` 中，quality-ratchet 的 `cargo clippy --bin bgxiong-ai-story-tauri` 若**成功**，且 manifest、target-dir、参数（feature 集）相同、源码指纹（HEAD + `git diff` + 未跟踪文件，覆盖 `src-tauri`/`crates`/`packages/app-contracts`）在 clippy 前后及检查时一致，则 compile 层 `cargo check`（heavy 时为 architecture-dod 的 G3）视为通过，输出 `PREFLIGHT_REUSE id=cargo-check …` / `DOD_REUSE …`，步骤 id、退出码与失败聚合不变。clippy 失败、未运行（没改 `.rs`）、或任一条件不符时照常跑 `cargo check`。不跨运行缓存。自测：`scripts/test-clippy-check-reuse.ps1`（真实 helper）、`scripts/test-preflight-affected-engine.ps1`（引擎夹具，含 reuse 用例）。

## 5c. Workspace 纯逻辑 crate（S22–S30）

- `src-tauri/Cargo.toml` 是 workspace 根，members 为主包加 `src-tauri/crates/` 下全部子 crate（2026-10-29 共 19 个，以 `Cargo.toml` 为准）。新 crate 放 `src-tauri/crates/<name>/`，加进 members 并在 `crates/crate-roles.json` 登记角色，host 用 `path = "crates/<name>"` 依赖；旧定义同批删除。主程序内每个 crate 最多一个 `pub use` 门面模块，禁止二级别名与 `#[path]` 兼容层（门面规则见提速 PLAN v2 §4.3）。
- 改 crate 内代码时的“改—测”循环：`cd src-tauri; cargo test -p <crate>`（G5：包 test-build 约 1–7 s，对比 host 的 2–3.5 min）。收尾仍按 §2：host `cargo check --bin bgxiong-ai-story-tauri` + `check:affected`。crate 一改，host 就会整体重编，这一点没变。
- 门禁覆盖：
  - `check:affected` 遇到 `src-tauri/crates/**` 时触发 `workspace-crates`（`scripts/check-workspace-crates.ps1`）：按 `crate-roles.json` 的角色做依赖与 std IO 守卫（`pure-core` 禁 tauri/rusqlite/reqwest/tokio 等；`infrastructure` 允许 rusqlite；其余角色见该文件）+ `cargo test --workspace --exclude bgxiong-ai-story-tauri`（全部非主包；精确到包见提速 PLAN v2 B7）；
  - quality-ratchet 对 crate 文件额外跑 `cargo clippy --workspace --exclude bgxiong-ai-story-tauri`（host 的 `--bin` clippy 不 lint path 依赖）；
  - `serde-internally-tagged-fields` 同时扫描 `src-tauri/crates/**`；
  - heavy/发布前的 architecture-dod 多了 G4b，跑同一组 crate 测试。
- 进程清理（D1）：`-p <非主包>` 或 `--exclude bgxiong-ai-story-tauri` 不选择主 bin；自动残留清理不结束任何主 GUI，精确同输出被占时显式 busy。
- 改 registry 或新增 crate 后跑映射自测：`powershell -NoProfile -ExecutionPolicy Bypass -File scripts/test-workspace-crate-mapping.ps1`（真实 registry；crate 源码/manifest/build.rs/JSON/删除路径的映射，加零匹配守卫）。

## 6. 测量工具（S02）

`scripts/measure-rust-dev-build.ps1`（ASCII，PowerShell 5.1）：固定 `--bin`、显式 target-dir、独立 stdout/stderr、`--timings`、环境/配置指纹，结果追加到 `<OutputDir>/results.jsonl`。
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File scripts/measure-rust-dev-build.ps1 `
  -Case source-test-build -Label my-1 -TargetDir [已省略：本地路径] -OutputDir docs/dev/reports/rust-build-speed/<run-id> -Command test-no-run
```
Case：noop-check / source-check / source-test-build / test-body-build / contract-build / affected-fast / affected-heavy / cold-build / test-run。工具以 BelowNormal 运行；`-EngineDir` 可指定引擎目录（默认本 scripts 目录）。

## 7. 配置层与工具链层结论（S12 / S15 / S16）

**结论：没有采用任何 profile / linker / sccache 配置变更。** 仓库 `src-tauri/Cargo.toml`、`.cargo/`、全局 cargo 配置与环境变量均保持原样。采纳门槛为“配对基线下 `test --no-run` 中位数改善 ≥10% **且** ≥5 s”，同一 target、同一 features、用户空闲时配对测量（配对基线：outbound check 86.2 s / test-build 133.6 s）。

| 候选 | test-build 中位数 | 相对 133.6 s | 其它观察 | 结论 |
|---|---|---|---|---|
| 依赖 `debug=0`（S09） | 131.3 s | −1.7% | PDB 1008→672 MB；cold 678 s | DONE_REJECTED |
| 主 crate `debug=1`（S10） | 129.4 s | −3.1% | PDB →636 MB；cold 611 s；丢变量级调试信息 | DONE_REJECTED |
| 增量编译核查 / 配置层决策（S11/S12） | — | — | incremental 正常，热构建仅主 crate 单元 Dirty；S09/S10 均未过门槛 | NO_CHANGE，G2 PASS_NO_CHANGE |
| rust-lld（S14） | 131.2 s | −1.8% | 真实链接步：link.exe 12.1 s vs rust-lld 11.1 s（≈测试构建的 9%） | DONE_REJECTED |
| linker 落地决策 / 工具链收口（S15/S17） | — | — | 链接不是瓶颈；瓶颈是主 crate 单元自身编译；Cargo.toml / .cargo / RUSTC_WRAPPER 无改动 | S15 DONE_REJECTED，S17 DONE，G3 PASS_NO_CHANGE |
| sccache（S16） | — | — | 依赖 rlib 523 单元 ≈1584 CPU·s 可缓存，但热构建时依赖本就 Fresh；主 test 单元（≈686 s CPU）不可缓存 | 本地 REJECT；CI/依赖冷构建是否使用待用户决定 |

- 原因归纳：热构建只重编主 crate 一个单元，调试信息与链接合计占比小；真正的收益来自**少跑**（第 2 节节奏、第 3 节避免重复构建与 `--bin`）和后续结构拆分（S20/S21，当前 DEFERRED）。
- 详见 「S09–S12 调试信息 profile 对照」（开源版未收录该内部文档或源码路径）、「S13–S17 linker 与工具链」（开源版未收录该内部文档或源码路径）、「S16 sccache 适用面」（开源版未收录该内部文档或源码路径）。
- 节奏收益实测见 S08 工作流回放（第 2 节）：同一任务 3 次小修改，旧节奏 1125.0 s → 新节奏 633.5 s（−43.7%），完整 test-build 与 affected 各 3→1，见 「S08 报告」（开源版未收录该内部文档或源码路径）。

## 8. 不做的事

- 不关闭 incremental、不全局设置 `CARGO_INCREMENTAL=0`、不改用户全局 `~/.cargo/config.toml`、不设用户/机器环境变量。
- 依赖或版本身份变化后，可按 「提速 PLAN v2 §6 B3」（开源版未收录该内部文档或源码路径） 手动回收 `src-tauri/target/debug/incremental/bgxiong_ai_story_tauri-*` 中 LastWriteTime 早于 72 小时的孤儿目录（执行时 `Get-Process cargo,rustc` 须为空；不做自动后台清理）。最近一次盘点：「B3 报告」（开源版未收录该内部文档或源码路径）。
- 不做门禁结果缓存、不加跳过开关。
- 结构拆分已解冻并按 「提速 PLAN v2」（开源版未收录该内部文档或源码路径） 推进（10-13 起已实际拆出 `bgx-video-span-core`、`bgx-db-core`、`bgx-video-span-sqlite` 等 crate，原 S21 冻结不再适用）。
- **2026-10-30 收口**：提速 PLAN 强制项（B/C/GB/C1/C2）与 D 切片 R14–R18 已 DONE；**不要硬推** C3 / R20 / E。详见 「closeout」（开源版未收录该内部文档或源码路径）。

## 9. 测试取舍（2026-10-14，用户要求）

测试按能拦住的回归选择，禁止每修一个 BUG、每个原子任务机械新增文件或凑足正例/拒绝/失败三条。优先扩展现有生产入口测试；可逆低影响的样式与文案无需新增测试。真实 IPC payload、schema 闭集、权限、事务回滚、幂等、任务恢复和资源释放仍须验证。

同一规则的静态扫描由一个既有门禁负责，单测不再重复扫描整棵源码。仅搜索源码局部写法或历史报告中的 PASS/NOT_RUN/任务编号，不构成功能验收。清理须核对调用入口与有效覆盖，不以脚本日期、未接入发版或当前没有失败作为无用证据。常驻规则见 「test-regression-value.mdc」（开源版未收录该内部文档或源码路径）。

本轮清理见 「需求与处置清单」（开源版未收录该内部文档或源码路径）。SP067/SP068/SP069 旧报告搜词用例已退役，原 runner 请求现在显式 NOT_RUN 并返回非零；真实离线执行、模型实测和交接仍需各自证据。历史台账中的 PASS 只表示当时的报告检查，不得用于当前产品验收。


### 9.1 Creator注册改动的最低行为验收

全仓新增、移除、迁移能力/handler/owner/module/applier时，catalog中的唯一tool owner、composition中模块的catalog_owners、实际factory返回的handler集合必须闭合。owner是归属引用，不是维护第二份tool ID清单；同一模块可以同时承接多个真实owner。生产和测试复用同一manifest，不能仅在测试注册中补齐。

最终源码编译一次harness后实际执行既有`creator_tools::module_registry::tests`及`composition::creator_modules::s01_changeset_assembly_tests`，再通过已完整装配的注册入口验证受影响读取行为；Comfy复用`creator_tools::comfy_workflow_parity_tests::workflow_list_pages_on_the_same_tool`。可复用官方preset导出所编译的同源码harness，按该次Cargo artifact/日志定位exe并核SHA，直接运行filter，避免再调用Cargo重复编译；无法证明源码/feature一致则不得复用。每组至少1项，记录真实用例数、失败和退出码。

`check-creator-closure-gates.ps1`只证明其实际执行的合同/清单/结构检查，case中存在V1过滤器不等于V1行为已执行。注册改动的完整装配失败、0项或NOT_RUN时，即使编译、更新签名及资源校验通过，也不能交付人工包或正式包。不能放宽owner/覆盖校验、屏蔽模块或新增页面私有旁路。GUI未实际启动不得标启动PASS；模型实跑、安装器向导与内核装配分别记录。

历史教训：「20261020动态模板」（开源版未收录该内部文档或源码路径）及「20261023对白提取」（开源版未收录该内部文档或源码路径）。两次均为局部handler可用但全局装配不成立，Comfy列表等无关入口被连带阻断。本条复用既有真实行为用例，不新增重复搜词门禁。

同类登记也包括AI阶段提示词：调用侧共享step label、`merge_stage_prompt_submit`映射及实际消费点须一起核对，复用真实数据库writer行为用例。自由画布20261023母版/评审两阶段漏映射导致HTTP前失败，另有已生成的canonical原文未投影到节点正文；原文初稿与后续加工应分别保存真实版本，不能以空节点或伪完成代替。
## 10. 分阶段交付与反复返工控制（2026-10-21，用户要求）

较大SDK接入/重构按可独立使用的成果拆阶段，先验证最小真实链路，再决定必要公共重构。阶段冻结基础与修改范围，预先约定墙钟/修正上限；确认已有功能回归、达到上限、定向修正后仍失败或合同缺口时，停止受影响阶段继续实施并保留现场。不能靠换runId或任务清零，也不能用重复编译/全仓核验填充进度。

脚本/环境/首次实现错误与真实回归分别定责；结果复用须同源，修改导致失效则补匹配检查。原最终树 affected/heavy 规则和必要真实回归仍有效；本条不增加跳过门禁的开关。详细失败经验、停止条件和交接见 「分阶段实质交付与失败停工规范」（开源版未收录该内部文档或源码路径）。

## 11. 发行失败先收齐再批量修复（2026-11-08，用户要求）

完整发行检查开始后，先收齐能安全完成的各阶段失败，不因一项普通检查失败立即结束整轮并从头重跑。编译失败、进程占用与业务回归分别记录；没有执行的后段保持未验，既有失败不能被后续成功覆盖。

形成同一批问题清单，逐项注明根因、代码owner和必要验证。按文件互斥分工给同事并行修复，保留其他会话已合入修改；先同步主仓，再统一合并本批修复。没有独立工作的同事保持空闲，不为并行重复编译、重建target或重复调查。

合并后先执行本批对应的定向行为和受影响合同检查，全部通过后冻结源码，再运行一轮完整发行链。完整链如发现下一批失败，仍先集中收齐再修复，不恢复逐项重启整轮的做法。不得提高baseline、删有效安全合同或将旧源码结果套到新产物。GUI、真实模型、安装器和内核读取分别按实际证据记账。

普通开发与开发EXE仍遵守2026-11-06规则，只做本次定向验收；仅用户明确要求整个发行包/发布包时执行全仓门禁。原测试target复用、提交推送与构建前同步主仓规则继续生效。
