# 屎山审计报告：故事导出导入模块（story_backup）

> 审计日期：2026-06-11
> 审计范围：`src-tauri/src/story_backup/` 全部 13 个文件 + `ui/src/domain/storyExport.ts`
> 评分标准：[屎山代码鉴定准则 v1.0](./屎山代码鉴定准则.md)

---

## 总评

| 维度 | 满分 | 得分 | 说明 |
|------|------|------|------|
| 命名与可读性 | 20 | 17 | 命名规范，注释充分 |
| 结构与设计 | 25 | 11 | **重灾区**：大量重复代码 |
| 错误处理 | 15 | 12 | 整体良好，有小瑕疵 |
| 状态管理与副作用 | 15 | 13 | 取消标志设计合理 |
| 依赖与配置 | 10 | 9 | 依赖清晰 |
| 测试与文档 | 10 | 4 | **几乎无集成测试** |
| 版本控制 | 5 | 5 | 提交规范 |
| **合计** | **100** | **71** | **🟡 及格：勉强能看** |

---

## 第二章：结构与设计 — 重灾区（扣 14 分）

### 问题 1：collector.rs — 查询执行的复制粘贴地狱（-4 分）

**位置**：[collector.rs:68-180](../src-tauri/src/story_backup/collector.rs#L68-L180)

`collect_table` 函数里有一个 **112 行的 match 块**，包含 **10 个几乎一模一样的分支**：

```rust
// 这段模式重复了 10 次，唯一区别是绑定哪个 JSON 参数
ExportSql::NodeIds(_) => {
    let json = serde_json::to_string(&scope.resolved.node_ids).unwrap_or_else(|_| "[]".to_string());
    let rows = stmt.query_map(params![json], |row| row_to_json(row, &column_names))
        .map_err(|e| format!("query {table}: {e}"))?;
    for row_result in rows {
        let json_val = row_result.map_err(|e| format!("row in {table}: {e}"))?;
        write_json_line(&mut writer, table, &json_val)?;
        count += 1;
    }
}
ExportSql::RoleIds(_) => { /* 完全一样，只是 scope.resolved.role_ids */ }
ExportSql::SegmentIds(_) => { /* 完全一样，只是 scope.resolved.segment_ids */ }
ExportSql::CanvasScopeIds(_) => { /* 完全一样，只是 scope.canvas_scope_ids */ }
ExportSql::ActorIds(_) => { /* 完全一样，只是 scope.resolved.actor_ids */ }
// ... 还有 5 个
```

**应改为**：为 `ExportSql` 实现一个 `resolve_params()` 方法返回 `Vec<String>`，然后用 **一个统一的查询执行路径**：

```rust
fn collect_table(...) -> Result<u64, String> {
    let export_sql = table_registry::export_sql_for_table(table);
    let sql = build_query(&export_sql);
    let mut stmt = conn.prepare(&sql)?;
    let params: Vec<String> = export_sql.resolve_params(scope);
    // 统一执行
    let rows = stmt.query_map(rusqlite::params_from_iter(params.iter()), |row| row_to_json(row, &column_names))?;
    // ...
}
```

### 问题 2：media_refs.rs — 13 个收集函数全是同一个模式（-4 分）

**位置**：[media_refs.rs:71-475](../src-tauri/src/story_backup/media_refs.rs#L71-L475)

| 函数 | 行数 | 模式 |
|------|------|------|
| `collect_story_reference_images` | 22 行 | query → unwrap → parse JSON → insert paths |
| `collect_portrait_versions` | 18 行 | 同上 |
| `collect_role_portrait_local_cache` | 18 行 | 同上 |
| `collect_actor_cast_pad_refs` | 30 行 | 同上 |
| `collect_role_pad_frame_refs` | 35 行 | 同上 |
| `collect_segment_video_state` | 35 行 | 同上 |
| `collect_dialogue_audio` | 20 行 | 同上 |
| `collect_clips_video_state` | 20 行 | 同上 |
| `collect_visual_asset_images` | 20 行 | 同上 |
| `collect_tts_studio_previews` | 25 行 | 同上 |
| `collect_edit_mask_paths` | 20 行 | 同上 |

**应改为**：注册表驱动，每个收集规则用一个结构体描述：

```rust
struct MediaRefRule {
    sql: &'static str,
    extract: fn(&str, &mut HashSet<String>),
}
const MEDIA_RULES: &[MediaRefRule] = &[ ... ];
```

### 问题 3：id_remap.rs — 20+ 个 remap 函数几乎相同（-3 分）

**位置**：[id_remap.rs:200-373](../src-tauri/src/story_backup/import/id_remap.rs#L200-L373)

每个 `remap_xxx` 函数的模式都是：

```rust
fn remap_xxx(v: &mut serde_json::Value, remap: &IdRemap) {
    replace_field(v, "field1", &remap.xxx_id_map);   // 唯一区别
    replace_field(v, "field2", &remap.yyy_id_map);   // 字段名不同
    replace_json_paths(v, "json_field", remap);       // 可选
}
```

**应改为**：表驱动配置：

```rust
struct RemapRule {
    direct_fields: &'static [(&'static str, &'static str)], // (field_name, map_key)
    json_path_fields: &'static [&'static str],
}
const REMAP_RULES: &[(&str, RemapRule)] = &[
    ("stories", RemapRule {
        direct_fields: &[("id", "story_id")],
        json_path_fields: &["reference_images"],
    }),
    // ...
];
```

新增表时只需加一行配置，不用写新函数。

### 问题 4：preview_count_sql 与 export_sql_for_table 双重维护（-3 分）

**位置**：
- [export.rs:342-370](../src-tauri/src/story_backup/export.rs#L342-L370) — `preview_count_sql`
- [table_registry.rs:123-189](../src-tauri/src/story_backup/table_registry.rs#L123-L189) — `export_sql_for_table`

**同一张表的 SQL 过滤条件被维护了两遍**：

```rust
// export.rs 里
fn preview_count_sql(table: &str) -> &'static str {
    match table {
        "edges" => "from_node_id",
        "node_content_history" => "node_id",
        // 30+ 行...
    }
}

// table_registry.rs 里
fn export_sql_for_table(table: &str) -> ExportSql {
    match table {
        T03_EDGES => ExportSql::node_ids_pair(...),
        T21_NODE_CONTENT_HISTORY => ExportSql::node_ids(...),
        // 30+ 行...
    }
}
```

**风险**：新增表时必须同步修改两处，漏改一处就会导致预览和实际导出数据不一致。应该从 `ExportSql` 推导 count SQL，而不是独立维护。

---

## 第六章：测试与文档（-6 分）

### 问题 5：无集成测试（-5 分）

整个 `story_backup` 模块只有 `table_registry.rs` 有 4 个单元测试（测 schema fingerprint）。

**完全缺失的测试**：
- 导出 → 导入 round-trip 测试
- DuplicateWithNewId 策略的 ID 重映射正确性测试
- 媒体文件完整性校验测试
- 共享演员导入/跳过逻辑测试
- 取消操作后的状态一致性测试
- 大数据量（>999 条）的 IN 子句边界测试

> table_registry.rs 头注释写了"新增故事级表时须同步更新"，但没有任何测试能捕获遗漏。

### 问题 6：前端导出模块与后端备份模块职责混淆（-1 分）

- `ui/src/domain/storyExport.ts` — 导出为 Markdown/JSON（剧本导出）
- `src-tauri/src/story_backup/` — 导出为 `.bgxstory` 包（完整备份）

两者都叫"导出"但职责完全不同，且后端命令在前端 **没有调用方**（grep 找不到任何 UI 代码调用 `exportStoryBackup`），属于半成品。

---

## 第四章：状态管理（-2 分）

### 问题 7：全局取消标志缺乏作用域隔离（-2 分）

**位置**：[commands.rs:13](../src-tauri/src/story_backup/commands.rs#L13)

```rust
static CANCEL_FLAG: AtomicBool = AtomicBool::new(false);
```

全局 `AtomicBool` 意味着：
- 如果同时触发导出和导入，取消一个会取消两个
- 没有操作 ID 关联，无法精确取消特定操作

对于当前单线程 Tauri command 模型尚可接受，但属于设计债务。

---

## 亮点（做得好的地方）

| 项 | 说明 |
|----|------|
| 模块拆分 | `mod.rs` 职责清晰，export/import/collector/scope/types 分离 |
| 错误类型 | `StoryBackupError` 枚举有结构化错误码，Display 实现完整 |
| 完整性校验 | 导出后 SHA256 校验 + 导入后引用完整性探测 |
| 进度事件 | 通过 Tauri emit 实时进度推送，UI 可接入 |
| schema fingerprint | 基于列结构的指纹，版本不匹配时拒绝导入 |
| 类型对齐 | `#[serde(rename_all = "camelCase")]` 确保 TS/Rust 一致 |
| 模块文档 | 每个文件都有 `//!` 模块级注释，说明职责和 SSOT |

---

## 改进建议（按优先级排序）

### P0：消除 collector.rs 的 10 路重复

为 `ExportSql` 添加 `resolve_params(&self, scope: &StoryScope) -> Vec<String>` 方法，将 10 个分支合并为 1 个通用执行路径。预计减少 ~100 行代码。

### P1：消除 preview_count_sql 重复

从 `ExportSql` 推导 count SQL，删除 `preview_count_sql` 函数。可选：为 `ExportSql` 实现 `count_sql(&self) -> String`。

### P2：表驱动 id_remap

用配置表 + 通用替换引擎替代 20+ 个 `remap_xxx` 函数。新增表时只需添加配置行。

### P3：表驱动 media_refs

将 13 个收集函数抽象为规则注册表，每条规则描述 SQL + JSON 提取路径。

### P4：补充集成测试

至少覆盖：导出 → 导入 round-trip、DuplicateWithNewId 的 ID 一致性、共享演员处理。

---

## 结论

**评分：71/100 — 🟡 及格**

模块架构设计合理，错误处理和文档注释做得不错。主要问题是 **代码重复严重**——`collector.rs`、`media_refs.rs`、`id_remap.rs` 三个文件加起来约 1200 行，其中至少 600 行是可以通过表驱动/泛化消除的重复代码。这导致新增表时需要同步修改 4+ 个文件（table_registry + collector + id_remap + media_refs + preview_count_sql），维护成本高且容易遗漏。

**一句话总结**：骨架是好骨架，但肉堆得太重复。
