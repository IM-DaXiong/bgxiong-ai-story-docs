# BUG 审查：story_backup 模块

> 审计日期：2026-06-11
> 审计范围：`src-tauri/src/story_backup/` 全部 13 个文件

---

## BUG-1：`count_sql()` 的 `replace` 策略在未来新增子查询表时会静默出错

**严重度**：🟡 中（当前安全，但维护陷阱）
**位置**：[table_registry.rs:259-272](../src-tauri/src/story_backup/table_registry.rs#L259-L272)

```rust
pub fn count_sql(&self) -> String {
    match self {
        Self::Simple(sql) | Self::Subquery(sql) => sql.replace("SELECT *", "SELECT COUNT(*)"),
        Self::StoryOrNodeIds(prefix) => prefix.replace("SELECT *", "SELECT COUNT(*)")
            + " (SELECT value FROM json_each(?2))",
        _ => {
            let query = self.to_query_sql();
            query.replace("SELECT *", "SELECT COUNT(*)")  // ← 替换所有匹配
        }
    }
}
```

`String::replace()` 替换 **所有** 匹配项。当前所有 variant 的子查询都使用 `SELECT value` 或 `SELECT id`（不含 `SELECT *`），所以不会误替换。但如果未来有人写了 `WHERE x IN (SELECT * FROM ...)` 的子查询，`count_sql` 会把子查询也改成 `SELECT COUNT(*)`，产生错误 SQL。

**建议修复**：改为 `replacen("SELECT *", "SELECT COUNT(*)", 1)`，只替换第一个匹配。

---

## BUG-2：`replace_field` 只处理 String 类型字段，非 String 字段静默跳过

**严重度**：🟢 低（当前安全，防御性不足）
**位置**：[id_remap.rs:419-430](../src-tauri/src/story_backup/import/id_remap.rs#L419-L430)

```rust
fn replace_field(v: &mut serde_json::Value, field: &str, map: &HashMap<String, String>) {
    if let Some(obj) = v.as_object_mut() {
        if let Some(serde_json::Value::String(s)) = obj.get(field) {  // ← 只匹配 String
            if let Some(new_val) = map.get(s) {
                obj.insert(field.to_string(), serde_json::Value::String(new_val.clone()));
            }
        }
    }
}
```

如果字段值是 `null`、数字、或数组类型，替换静默跳过。当前所有 ID 字段都是 TEXT（JSON String），所以安全。但若 JSONL 中某行的 `story_id` 因数据损坏变成了 `null`，导入时不会报错也不会 remap，导致数据不一致。

**建议修复**：对非 String 且非 Null 的字段打印 warning 日志。

---

## BUG-3：`ai_queue_tasks` 的 `payload_json` 中嵌入的 `storyId` 不会被 remap

**严重度**：🟡 中（影响 DuplicateWithNewId 策略 + AI 队列表）
**位置**：[id_remap.rs:279](../src-tauri/src/story_backup/import/id_remap.rs#L279)

```rust
// ai_queue_tasks / ai_queue_task_events 不做 remap
```

`ai_queue_tasks` 表的 `payload_json` 字段包含 `$.storyId`（见 [table_registry.rs:183](../src-tauri/src/story_backup/table_registry.rs#L183)）：

```rust
T40_AI_QUEUE_TASKS => ExportSql::subquery(
    "SELECT * FROM ai_queue_tasks WHERE json_extract(payload_json, '$.storyId') = ?1",
),
```

使用 `DuplicateWithNewId` 策略导入时，`ai_queue_tasks` 行被原样插入，但 `payload_json` 中的 `storyId` 仍指向旧故事 ID。

**影响**：
- `include_ai_queue_tasks = false`（默认）时不影响
- `include_ai_queue_tasks = true` 时，导入的队列任务关联到旧故事

**建议修复**：为 `ai_queue_tasks` 添加 remap 规则，对 `payload_json` 执行 `replace_json_paths`。

---

## BUG-4：`OverwriteExisting` 策略执行两次 VACUUM 备份

**严重度**：🟢 低（功能浪费，不丢数据）
**位置**：[import/apply.rs:51-96](../src-tauri/src/story_backup/import/apply.rs#L51-L96)

```rust
StoryBackupImportStrategy::OverwriteExisting => {
    if exists {
        // 第一次备份
        crate::db::backup::vacuum_into_labeled_backup(conn, &app_root, "pre_story_import")?;
        crate::infrastructure::persistence::sqlite::delete_orchestration::delete_story_tx(conn, &story_id)?;
        // ...
    }
    (story_id.clone(), None)
}

// ...

// 第二次备份（OverwriteExisting 已 return，不会执行到这里）
let backup_path = crate::db::backup::vacuum_into_labeled_backup(
    conn, &app_root, "pre_story_backup_import",
)?;
```

实际上 `OverwriteExisting` 在 match 块内直接 return，不会走到外层的第二次备份。**所以这不是 BUG**，只是代码结构让人误以为会执行两次。注释已说明 `OverwriteExisting` 有独立备份。

**结论**：非 BUG，但建议加行注释明确 `// OverwriteExisting 已在 match 内 return，此处仅 DuplicateWithNewId 到达`。

---

## BUG-5：`collector.rs` 参数绑定方式变更 — `params![]` 宏 vs `&[&dyn ToSql]`

**严重度**：🟢 低（已验证等价，但语义略有不同）
**位置**：[collector.rs:67-69](../src-tauri/src/story_backup/collector.rs#L67-L69)

```rust
// 旧代码
let rows = stmt.query_map(params![story_id], |row| ...)?;

// 新代码
let params = export_sql.resolve_params(story_id, scope);
let param_refs: Vec<&dyn rusqlite::types::ToSql> = params.iter().map(|p| p as _).collect();
let rows = stmt.query_map(param_refs.as_slice(), |row| ...)?;
```

`rusqlite::params![]` 宏创建固定大小数组，`&[&dyn ToSql]` 是动态切片。两者都实现 `rusqlite::Params`，功能等价。

**验证**：编译通过 + 测试通过，`resolve_params` 返回的参数顺序与原 `params![]` 绑定顺序一致。

**结论**：非 BUG，等价替换。

---

## BUG-6：`count_sql` 对 `Simple` 和 `Subquery` 使用相同的 `replace` 策略

**严重度**：🟢 低（当前安全）
**位置**：[table_registry.rs:261](../src-tauri/src/story_backup/table_registry.rs#L261)

```rust
Self::Simple(sql) | Self::Subquery(sql) => sql.replace("SELECT *", "SELECT COUNT(*)"),
```

`Simple` 和 `Subquery` 合并处理。`Simple` 的 SQL 不含子查询（如 `SELECT * FROM stories WHERE id = ?1`），替换安全。`Subquery` 的 SQL 含子查询（如 `SELECT * FROM ai_queue_task_events WHERE task_id IN (SELECT id FROM ...)`），但子查询用 `SELECT id` 不用 `SELECT *`，所以也安全。

**结论**：非 BUG，但建议与 BUG-1 统一改为 `replacen` 以提高防御性。

---

## BUG-7（非重构引入）：导出取消后不清理 `tables.json`

**严重度**：🟢 低
**位置**：[export.rs:154-157](../src-tauri/src/story_backup/export.rs#L154-L157)

```rust
if crate::story_backup::is_cancelled() {
    let _ = fs::remove_dir_all(&package_path);  // ← 删除整个包目录
    return Err(StoryBackupError::Cancelled);
}
```

取消时删除整个 `package_path`，包括已写入的 `data/tables.json` 和部分 JSONL 文件。这是正确的——取消意味着不保留任何输出。

**结论**：非 BUG，行为正确。

---

## BUG-8（非重构引入）：`insert_row` 使用 `INSERT OR IGNORE` 可能静默丢弃数据

**严重度**：🟡 中
**位置**：[import/apply.rs:219-237](../src-tauri/src/story_backup/import/apply.rs#L219-L237)

```rust
let sql = format!(
    "INSERT OR IGNORE INTO {} ({}) VALUES ({})",
    table, columns.join(", "), placeholders.join(", ")
);
```

`INSERT OR IGNORE` 在主键/唯一约束冲突时静默跳过。对于 `SkipExisting` 策略，这可能是期望行为。但对于 `DuplicateWithNewId` 策略，如果 remap 漏掉了某个 ID 导致主键冲突，行会被静默丢弃，不会报错。

**建议修复**：在 `DuplicateWithNewId` 策略下，改为 `INSERT INTO`（不用 `OR IGNORE`），让冲突变成显式错误。

---

## 汇总

| 编号 | 严重度 | 描述 | 是否重构引入 |
|------|--------|------|-------------|
| BUG-1 | 🟡 中 | `count_sql` 用 `replace` 全量替换，未来新增子查询表时可能误替换 | 是 |
| BUG-2 | 🟢 低 | `replace_field` 只处理 String，非 String 字段静默跳过 | 否（原有逻辑） |
| BUG-3 | 🟡 中 | `ai_queue_tasks.payload_json` 中的 storyId 不被 remap | 否（原有逻辑） |
| BUG-4 | 🟢 低 | OverwriteExisting 看似两次备份实际不会 | 否（代码结构误导） |
| BUG-5 | 🟢 低 | 参数绑定方式变更 | 是（已验证等价） |
| BUG-6 | 🟢 低 | Simple/Subquery 合并处理 | 是（当前安全） |
| BUG-7 | 🟢 低 | 取消后清理逻辑 | 否 |
| BUG-8 | 🟡 中 | INSERT OR IGNORE 可能静默丢弃数据 | 否（原有逻辑） |

**需要修复**：BUG-1（改为 `replacen`）。
**建议修复**：BUG-3（添加 ai_queue_tasks remap 规则）、BUG-8（DuplicateWithNewId 不用 OR IGNORE）。
