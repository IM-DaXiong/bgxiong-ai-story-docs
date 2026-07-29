# GUIDE · 持久化层六边形架构

**文档时间**：2026-06-09  
**状态**：**已落地**（Phase 4 验收 · Phase 5 CI 守护）  
**权威计划**：`docs/dev/specs/20260608-ARCH-持久化层六边形架构升级计划.md`  
**组合根**：`src-tauri/src/composition/app_container.rs`  
**门禁**：`scripts/check-persistence-architecture.ps1` · `scripts/check-platform-boundary.ps1` · `scripts/check-architecture-gates.ps1`

---

## 一、分层与依赖方向

```text
commands / IPC / 流水线编排（薄）
        │
        ▼
application::services（用例 · 返回 ApplicationError）
        │
        ├── application::ports（跨域编排 Port）
        └── domain::ports（细粒度 Repository Port）
                │
                ▼
infrastructure::persistence::sqlite（唯一 rusqlite 业务实现层）
        │
        ▼
db::DbState（连接池 · 仅 composition 内部持有）
```

**硬规则**（与 `AGENTS.md` 一致）：

| 层 | 禁止 |
|----|------|
| `domain/` | `use rusqlite`、Tauri、reqwest |
| `application/` | `use rusqlite`、`with_conn`、直调 `*_sql.rs` / `*_layout.rs` |
| `commands/` + 非 commands IPC | `State<DbState>`、`use rusqlite` |
| 流水线根模块 | 直连 SQL（须经 Service → Port） |

允许 `use rusqlite` 的位置：`infrastructure/persistence/**`、`db/**`、以及 `scripts/persistence-cli-exemptions.txt` 列出的维护 CLI。

---

## 二、组合根（AppContainer）

`main.rs` 仅 `.manage(AppContainer)`，**不得** `.manage(db_state)`。

- `DbState` 在 `AppContainer::new` 内部创建并注入各 `*RepositorySqlite`
- IPC handler 签名：`State<AppContainer>` + 领域 DTO；参数校验在 command，业务在 Service
- 需要多表原子写：优先 `TransactionPort` + `*RepositoryInTx`（见 §四）

获取 Service 模式（command 内）：

```rust
fn svc(c: &AppContainer) -> Arc<StoryboardService> {
    c.storyboard_service()
}
```

---

## 三、Port 清单（摘要）

### 3.1 domain::ports（细粒度 CRUD / 读路径）

| Port | 职责 |
|------|------|
| `NodeRepository` | 层级读、有序列表 |
| `NodeCrudRepository` | 节点 CRUD |
| `EdgeRepository` | 边 CRUD |
| `AiTaskRepository` | AI 任务持久化 |
| `AiQueueRepository` | AI 队列状态机 |
| `GraphRepository` | graph_core 快照 |
| `LayoutRepository` | 故事/剧集/场次/演员列表布局 |
| `StoryRepository` | 故事元数据 |
| `VisualAssetRepository` | 视觉资产 |
| `MediaAssetRepository` | 媒体资产 |
| `ShotSpecsRepository` | 分镜 shot specs |
| `TransactionPort` | 用例级事务 Bundle |

实现目录：`infrastructure/persistence/sqlite/*_repository_sqlite.rs`

### 3.2 application::ports（跨域编排）

典型：`StoryboardMergeRepository`、`PadPolicyRepository`、`ClipsRepository`、`SegmentCanvasWriteRepository`、`DeleteOrchestrationRepository`、`MigrationRepository`、`OrphanReferenceRepository` 等（完整列表见 `application/ports/mod.rs`）。

编排 Port 可引用多个 domain 类型，但**不得**引用 `commands/` 类型。

---

## 四、事务（TransactionPort / UoW）

多 Repository 同一事务：

```rust
transaction_port.with_transaction(|tx| {
    tx.node_crud().upsert(...)?;
    tx.edge().create(...)?;
    Ok(())
})?;
```

- `TransactionPort` 定义于 `domain/ports/transaction.rs`
- SQLite 实现：`infrastructure/persistence/sqlite/transaction_port_sqlite.rs`
- 禁止在 Service 内 `db.with_transaction`；事务边界由 Port 或专用 `*_sqlite` 编排函数承担

---

## 五、新增域 Checklist

1. `domain/entities/` — 领域实体（如需）
2. `domain/ports/xxx_repository.rs` — trait（domain 类型入参/返回值）
3. `infrastructure/persistence/sqlite/xxx_sqlite.rs` — 纯 SQL
4. `infrastructure/persistence/sqlite/xxx_repository_sqlite.rs` — Port 实现（持有 `DbState`）
5. `application/services/xxx_service.rs` — 用例（`Arc<dyn XxxRepository>`）
6. `composition/app_container.rs` — 注册并暴露 getter
7. `commands/xxx.rs` — `State<AppContainer>` 薄封装
8. `infrastructure/persistence/sqlite/repository_integration_tests.rs` — TestDb 集成测试（≥5 cases 推荐）
9. `scripts/check-persistence-architecture.ps1` — 必须通过

---

## 六、门禁与 CI

| 脚本 | DoD | 说明 |
|------|-----|------|
| `check-persistence-architecture.ps1` | G1 | rusqlite / with_conn 三 pattern |
| `check-platform-boundary.ps1` | G2 | 域模块禁止直调 fs/env/Command |
| `check-ipc-handlers-baseline.ps1` | G5 | `generate_handler!` 与 SSOT 零 diff |
| `check-architecture-gates.ps1` | G1+G2+G5 | 本地/CI 聚合 |
| `check-architecture-dod.ps1` | G1–G7 | 全量 DoD 自动化（G8–G12 人工） |

**CI**：`.github/workflows/architecture-gates.yml`（push/PR 自动跑 G1/G2/G5 + `cargo check` + 关键测试）。

**IPC 基线更新**：

```powershell
powershell -File scripts/snapshot-ipc-handlers.ps1
# 同步写入 docs/dev/_snapshots/ipc-handlers-baseline-SSOT.txt
powershell -File scripts/check-ipc-handlers-baseline.ps1
```

`#[cfg(feature = "...")]` 行在比对时规范化为 handler 路径（去 cfg 前缀）。

---

## 七、CLI 豁免

`scripts/persistence-cli-exemptions.txt` 列出离线维护 bin（如 `port_compact_cli.rs`）。清单外任何 `use rusqlite` 均 fail。

---

## 八、与跨平台 PLAN 的关系

- **PlatformContext**（`src-tauri/src/platform/`）：文件 I/O、路径、多窗口能力
- **持久化 Port**：仅数据库与业务持久化；不得把 `std::fs` 混进 Repository
- 组合根并列：`AppContainer`（业务持久化）+ `PlatformContext`（运行时环境）

详见 `docs/dev/plans/20260608T100000+0800-PLAN-跨平台基础架构升级.md`。

---

## 九、排障

1. 门禁 fail → `rg "use rusqlite" src-tauri/src/<模块>` 定位；SQL 下沉到 `infrastructure/persistence/sqlite/`
2. 禁止在 command/service 加 dedupe/双路径兜底掩盖重复写（见 `.cursor/rules/bug-fix-no-compat-fallback.mdc`）
3. BUG 先读 `logs/` 再改（见排障 GUIDE）

记录：`docs/dev/fixes/FIX-BUG-20260608.md` §54–§61（Phase 4 迁移批次）。
