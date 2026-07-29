# ComfyUI 架构最优解：三分离模型（原子级实现方案）

**文档日期**：2026-05-03
**修订日期**：2026-05-03（v2 — 补充完整数据流、类型定义、组件实现）
**性质**：架构设计 — 不考虑兼容性、不考虑兜底、以项目最优解为准
**前置文档**：`20260503-DIAGNOSE-ComfyUI生图页面模型选择器问题与改进方案.md`、`20260503-PLAN-ComfyUI原子级开发计划.md`
**基线代码**：`appSettings.ts`（`listConfiguredImageModelPicks` / `imageServiceOverrideForPick`）、`useSegmentShotGeneration.ts`（`executeSegmentShot`）、`ShotModelPickerRow.tsx`、`SettingsComfyuiWorkflowPanel.tsx`、`comfyui/pipeline/image.rs`、`comfyui/workflow/templates.rs`、`comfyui/workflow/resolver.rs`

---

## 〇、前置：完整数据流审计

在讨论架构之前，必须先理清**当前代码中 ComfyUI 生图的完整数据流**。这是理解一切改动的基石。

### 0.1 当前数据流：设置页 → profile.extra

```
用户操作（设置页第四栏 SettingsComfyuiWorkflowPanel）:
  点击 "basic_t2i"
    → applyTemplateValue("basic_t2i", caps, json)
      → profile.extra.comfyui_workflow_template = "basic_t2i"
        → 用户点击"保存" → 写入磁盘
```

### 0.2 当前数据流：生图页 → backend

```
用户点击「生成」(ShotActionButtons → executeSegmentShot):

  1. effectiveShotImgKey = shotImagePickHook.userPickKey
     = JSON.stringify([profileId, model, backendId])
     对 ComfyUI: model = "" → key = '["comfyui-xxx", "", "comfyui"]'

  2. parseImageModelPickKey(effectiveShotImgKey)
     → { profileId: "comfyui-xxx", model: "", backendId: "comfyui" }

  3. imageServiceOverrideForPick(appSettings, profileId, model)
     → 查找 profile → 构造 AiServiceOverride {
         baseUrl: "http://127.0.0.1:8188",
         providerKind: "image_comfyui",
         model: undefined (因为 model === "")
         extra: profile.extra   ← 这里包含了 comfyui_workflow_template!
       }

  4. api.generateSegmentStoryboardImage({
       imageProvider: "comfyui",
       imageModel: "",              ← model 为空
       imageServiceOverride: {...},  ← extra 中有 workflow 信息
       ...
     })

  后端 (generate_segment_storyboard_image):
    5. resolve_effective_image_service(cfg, ComfyUi, override)
       → merge_ai_service(base, override)
       → image_service.extra = override.extra  ← workflow 信息进入 AiServiceConfig

    6. IMAGE_PROVIDERS.get("comfyui").generate(image_service, request)
       → resolve_workflow_json(service, request)
         → merge_extra(service, request) → ex
         → template_id_from_extra(ex) → "basic_t2i"  ← 从 extra 中取！
         → workflow_json_for_template_id("basic_t2i") → 返回 JSON
       → analyzer::analyze_capabilities(...)
       → node_deps::check_dependencies(...)
       → negative_prompt::generate(...)
       → apply_placeholders(ckpt, seed, w, h, ...)
       → post_prompt(workflow)
       → poll_until_done(...)
       → download_view_to_path(...)
```

### 0.3 关键发现

**当前流程其实是通的**：工作流信息通过 `profile.extra.comfyui_workflow_template` → `AiServiceOverride.extra` → `AiServiceConfig.extra` → `resolve_workflow_json` 这条链路正确传递到了后端。

**真正的问题只有两个**：

| # | 问题 | 影响 |
|---|------|------|
| 1 | 生图页面"模型选择器"显示 `—`（空字符串），用户不知道选了什么 | 用户体验：语义错误 |
| 2 | 能力卡片从 `providerKind` 元数据获取，不反映当前选中工作流的真实能力 | 能力展示：数据源错误 |

### 0.4 结论

**不需要改造整个执行链路**。`ImageProvider` trait 不变，`ImageGenRequest` 不变，`extra` 透传机制不变。

**只需要改两层**：
1. **UI 渲染层**：生图页面 ComfyUI 时渲染工作流选择器，而非模型选择器
2. **能力数据源层**：ComfyUI 时能力卡片从工作流分析获取，而非 providerKind 元数据

---

## 一、根因分析

### 1.1 问题不在 UI，在抽象

当前架构的根本问题不是"生图页面的模型选择器对 ComfyUI 显示为空"，而是：

**项目将两种完全不同的计算范式强行统一到一个抽象中。**

| 维度 | 云端 API（即梦/可灵/火山） | ComfyUI 本地服务 |
|------|--------------------------|------------------|
| 计算模型 | Stateless HTTP 请求-响应 | 有状态图计算引擎（工作流 = 计算图） |
| 用户选择什么 | 模型名（字符串标识符） | 工作流（完整计算图定义） |
| 谁定义能力边界 | Provider kind（固定元数据） | 工作流本身（每个工作流能力不同） |
| 提示词参数 | prompt + negative_prompt 是 API 参数 | 工作流中的节点输入（可能有/没有/多个） |
| 参考图 | API 请求体中的 image 字段 | 工作流中的 LoadImage 节点（可能 0 个或 N 个） |
| 出图尺寸 | API 参数 | EmptyLatentImage 节点的 width/height |
| 模型/权重 | 由 API 端决定 | 由工作流中的 CheckpointLoader 节点决定 |
| 依赖管理 | 无（API 端保证可用） | 需要检查 ComfyUI 是否安装了工作流所需的自定义节点 |

**关键洞察**：云端 API 是"调函数"，ComfyUI 是"跑程序"。用一个 `ImageModelPick { model: string }` 去抽象两者，就像用同一个下拉框让用户选"函数名"和"应用程序"一样荒谬。

### 1.2 原子级开发计划没有覆盖

`20260503-PLAN-ComfyUI原子级开发计划.md` 设计了设置页第四栏的工作流管理器（Phase C1），解决了"在设置页选择工作流"的问题。但它**没有解决生图执行页面**的交互：

- 用户在设置页选了工作流 → 回到生图页面 → 看到的仍然是"模型选择器"显示 `—`
- 能力卡片从 providerKind 元数据获取，不反映当前选中工作流的真实能力
- 两个页面之间存在信息断层

---

## 二、最优解：三层分离模型

### 2.1 设计哲学

```
┌─────────────────────────────────────────────────────────────┐
│                    三层分离模型                               │
│                                                             │
│  Layer 1: 执行边界（Execution Boundary）                     │
│    → ImageProvider trait，内部契约，不变                     │
│    → 编排层调用 provider.generate()，不关心内部怎么实现       │
│    → ImageGenRequest 不变，extra 透传不变                    │
│                                                             │
│  Layer 2: 用户配置边界（User Configuration Boundary）        │
│    → 每种 providerKind 声明"选择模式"                        │
│    → UI 根据模式渲染不同的选择器                             │
│    → ComfyUI 用 workflow 数据填充 ImageModelPick.model       │
│    → 云端 API 用 model 数据填充 ImageModelPick.model         │
│    → 下游 infrastructure（hooks/types/keys）完全复用         │
│                                                             │
│  Layer 3: 能力体系（Capability System）                      │
│    → 云端 API：能力 = providerKind 的静态元数据              │
│    → ComfyUI：能力 = 当前选中工作流的动态分析结果            │
│                                                             │
│  核心原则：                                                   │
│  - 执行边界保持统一（编排层不需要知道差异）                   │
│  - 用户配置按范式分离（UI 语义正确）                         │
│  - 能力体系数据源正确（反映真实运行时状态）                   │
│  - 下游 infrastructure 复用（hooks/keys/types 不变）         │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 为什么不是"统一抽象"？

有人可能会想：为什么不把模型和工作流统一成一个概念（比如都叫 "ExecutionProfile"）？

**答案是：因为它们的本质不同。**

- 模型是一个**字符串标识符**，指向 API 端的一个端点
- 工作流是一个**计算图**，包含节点、连接、参数、依赖

统一后的抽象必然是两者的最小公倍数，会丢失重要信息（工作流的节点数、复杂度、依赖关系），同时引入不必要的复杂度（云端 API 也需要处理 workflow 字段）。

**正确做法是：在执行边界保持统一，在用户配置层面按范式分离，下游 infrastructure 复用。**

---

## 三、具体架构设计（原子级）

### 3.1 Layer 2 基础：SelectionMode 声明

#### 3.1.1 后端：集中映射（新建文件）

```rust
// src-tauri/src/open_platform/provider_metadata.rs

use serde::Serialize;

/// 每个 providerKind 声明它在生图页面上需要什么类型的"选择器"。
/// 这是全局唯一的硬编码映射点。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize)]
#[serde(rename_all = "snake_case")]
pub enum SelectionMode {
    /// 选择模型名（云端 API：即梦/可灵/火山等）
    Model,
    /// 选择工作流（ComfyUI）
    Workflow,
}

impl SelectionMode {
    /// 从 providerKind 字符串推导选择模式。
    pub fn from_kind(kind: &str) -> Self {
        match kind {
            "comfyui" => SelectionMode::Workflow,
            _ => SelectionMode::Model,
        }
    }
}
```

```rust
// src-tauri/src/open_platform/mod.rs — 新增一行：
pub mod provider_metadata;
```

#### 3.1.2 前端：集中映射

```typescript
// ui/src/settings/appSettings.ts — 在 IMAGE_KIND_COMFYUI 定义之后添加：

export type SelectionMode = "model" | "workflow";

/** 每种 providerKind 声明其选择模式。集中定义，UI 统一读取。 */
export const PROVIDER_SELECTION_MODES: Record<string, SelectionMode> = {
  [IMAGE_KIND_COMFYUI]: "workflow",
};

/** 查询 providerKind 的选择模式。未定义时默认 "model"。 */
export function getSelectionMode(kind: string): SelectionMode {
  return PROVIDER_SELECTION_MODES[kind] ?? "model";
}
```

### 3.2 Layer 2 核心：ComfyUI 时 `ImageModelPick` 填充工作流数据

**这是最关键的洞察**：不需要创建新的 `WorkflowPick` 类型与 `ImageModelPick` 平行。

**原因**：现有基础设施（`useEffectiveModelPick`、`ShotPickHook<ImageModelPick>`、`parseImageModelPickKey`、`imageModelPickKey`、`imageServiceOverrideForPick`）全部围绕 `ImageModelPick` 构建。引入新类型意味着所有这些都要改造，引入不必要的复杂度。

**正确做法**：保持 `ImageModelPick` 类型不变，**改变 ComfyUI 时的数据来源**。

#### 3.2.1 前端：新增工作流列表构建函数

```typescript
// ui/src/settings/appSettings.ts

/** ComfyUI 专用：将工作流列表转换为 ImageModelPick 格式。
 *
 * 关键设计：
 * - workflowId 写入 model 字段（下游通过 parseImageModelPickKey 解析）
 * - label 展示为用户友好的名称
 * - backendId 固定为 "comfyui"
 * - 下游 infrastructure 完全不需要知道差异
 */
export function comfyuiWorkflowsToModelPicks(
  workflows: WorkflowDisplayInfo[],
  profileId: string,
  profileName: string,
  currentTemplate: string,
): ImageModelPick[] {
  return workflows.map((wf) => ({
    profileId,
    model: wf.workflowId,           // workflow ID 写入 model 字段
    label: wf.label,                 // 展示名称
    backendId: "comfyui",            // 固定
    profileName,
  }));
}

/** 解析 ComfyUI workflow pick key → WorkflowDisplayInfo 的辅助函数。 */
export function parseWorkflowPickKey(key: string): {
  profileId: string;
  workflowId: string;
  backendId: string;
} | null {
  const parsed = parseImageModelPickKey(key);
  if (!parsed) return null;
  return {
    profileId: parsed.profileId,
    workflowId: parsed.model,         // model 字段存的就是 workflowId
    backendId: parsed.backendId,
  };
}

/** 序列化 workflow pick → key 字符串（与 imageModelPickKey 同形）。 */
export function workflowPickKey(pick: {
  profileId: string;
  workflowId: string;
  backendId: string;
}): string {
  return imageModelPickKey({
    profileId: pick.profileId,
    model: pick.workflowId,
    label: pick.workflowId,
    backendId: pick.backendId,
    profileName: "",
  });
}
```

#### 3.2.2 后端：新增 Tauri 命令

```rust
// src-tauri/src/comfyui/commands.rs — 新增命令

use crate::comfyui::analyzer;
use crate::comfyui::workflow::templates::{list_builtin_templates, workflow_json_for_template_id};
use crate::comfyui::workflow::custom::{list_custom_comfyui_workflows, get_comfyui_workflow};
use serde::Serialize;

/// 工作流展示信息（供生图页面下拉使用）。
#[derive(Debug, Clone, Serialize)]
#[serde(rename_all = "camelCase")]
pub struct WorkflowDisplayInfo {
    pub workflow_id: String,
    pub label: String,
    pub source: String,  // "builtin" | "custom"
    pub capabilities: Option<crate::comfyui::types::WorkflowCapabilities>,
}

/// 返回 ComfyUI 的所有可用工作流（内置 + 自定义），供生图页面下拉使用。
#[tauri::command]
pub async fn list_comfyui_workflows_for_generation() -> Result<Vec<WorkflowDisplayInfo>, String> {
    // 1. 内置模板
    let builtins = list_builtin_templates()
        .into_iter()
        .filter_map(|tpl| {
            let json = workflow_json_for_template_id(tpl.id)?;
            let caps = analyzer::analyze_workflow_capabilities(json).ok();
            Some(WorkflowDisplayInfo {
                workflow_id: tpl.id.to_string(),
                label: tpl.name.to_string(),
                source: "builtin".into(),
                capabilities: caps,
            })
        })
        .collect::<Vec<_>>();

    // 2. 自定义工作流
    let customs = list_custom_comfyui_workflows()
        .map_err(|e| e.to_string())?
        .into_iter()
        .filter_map(|wf| {
            let json = get_comfyui_workflow(&wf.name).ok()?;
            let caps = analyzer::analyze_workflow_capabilities(&json).ok();
            Some(WorkflowDisplayInfo {
                workflow_id: format!("custom:{}", wf.name),
                label: wf.name,
                source: "custom".into(),
                capabilities: caps,
            })
        })
        .collect::<Vec<_>>();

    let mut all = builtins;
    all.extend(customs);
    Ok(all)
}
```

```rust
// src-tauri/src/main.rs — 注册新命令（在现有 ComfyUI 命令注册块中添加）：
.manage(comfyui::commands::list_comfyui_workflows_for_generation)
```

#### 3.2.3 前端：新增 API 方法

```typescript
// ui/src/api/methods/apiPart3.ts — 新增：

listComfyuiWorkflowsForGeneration: () =>
  invoke<WorkflowDisplayInfo[]>("list_comfyui_workflows_for_generation", {}),
```

```typescript
// ui/src/api/types.ts — 新增（如果 WorkflowDisplayInfo 不存在）：

export interface WorkflowDisplayInfo {
  workflowId: string;
  label: string;
  source: "builtin" | "custom";
  capabilities?: WorkflowCapabilities;
}
```

#### 3.2.4 前端：新增 hook

```typescript
// ui/src/hooks/useComfyuiWorkflows.ts

import { useState, useEffect, useCallback } from "react";
import { api } from "../api";
import type { WorkflowDisplayInfo } from "../api/types";
import type { ImageModelPick } from "../settings/appSettings";
import { comfyuiWorkflowsToModelPicks } from "../settings/appSettings";

interface UseComfyuiWorkflowsResult {
  /** 原始工作流列表（后端返回） */
  workflows: WorkflowDisplayInfo[];
  /** 转换为 ImageModelPick[] 格式（供 ModelPicker 直接使用） */
  modelPicks: ImageModelPick[];
  loading: boolean;
  error: string | null;
  refresh: () => void;
}

/** 加载 ComfyUI 可用工作流列表，转换为 ImageModelPick 格式。 */
export function useComfyuiWorkflows(
  profileId: string,
  profileName: string,
  currentTemplate: string,
): UseComfyuiWorkflowsResult {
  const [workflows, setWorkflows] = useState<WorkflowDisplayInfo[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const load = useCallback(() => {
    setLoading(true);
    setError(null);
    api.listComfyuiWorkflowsForGeneration()
      .then((list) => {
        setWorkflows(list);
      })
      .catch((e) => {
        setError(String(e));
        setWorkflows([]);
      })
      .finally(() => setLoading(false));
  }, []);

  useEffect(() => { load(); }, [load]);

  const modelPicks = comfyuiWorkflowsToModelPicks(
    workflows, profileId, profileName, currentTemplate,
  );

  return { workflows, modelPicks, loading, error, refresh: load };
}
```

### 3.3 Layer 2 UI 渲染：声明式选择器路由

#### 3.3.1 核心组件：SelectionModePicker

```tsx
// ui/src/components/SelectionModePicker.tsx

import { useState, useRef, useEffect } from "react";
import { getSelectionMode, type SelectionMode } from "../settings/appSettings";
import type { ImageModelPick } from "../settings/appSettings";
import { ModelPicker } from "./ModelPicker";
import { useComfyuiWorkflows } from "../hooks/useComfyuiWorkflows";

export interface SelectionModePickerProps {
  /** 当前选中 provider 的 providerKind */
  providerKind: string;
  /** Model 模式：模型列表 */
  modelPicks: ImageModelPick[];
  /** 通用 pick hook（两种模式复用） */
  pickHook: {
    effectivePick: ImageModelPick | null;
    userPickKey: string | null;
    selectPick: (key: string) => void;
  };
  pickKeyFn: (pick: ImageModelPick) => string;
  pickLabelFn: (pick: ImageModelPick) => string;
  disabled?: boolean;
  /** 列标题文案 */
  label?: string;
  /** 辅助说明文案 */
  helpText?: string;
}

/** 声明式选择器路由组件。
 *
 * 根据 providerKind 的 selectionMode 自动渲染正确的选择器：
 * - "model" 模式 → 渲染 ModelPicker（现有逻辑，不变）
 * - "workflow" 模式 → 渲染 WorkflowPicker（新建，ComfyUI 专用）
 */
export function SelectionModePicker({
  providerKind,
  modelPicks,
  pickHook,
  pickKeyFn,
  pickLabelFn,
  disabled,
  label,
  helpText,
}: SelectionModePickerProps) {
  const mode = getSelectionMode(providerKind);

  if (mode === "workflow") {
    return (
      <WorkflowPicker
        profileId={pickHook.effectivePick?.profileId ?? ""}
        profileName={pickHook.effectivePick?.profileName ?? ""}
        pickHook={pickHook}
        pickKeyFn={pickKeyFn}
        pickLabelFn={pickLabelFn}
        disabled={disabled}
        label={label ?? "工作流"}
        helpText={helpText ?? "选择 ComfyUI 工作流（决定出图风格与流程）"}
      />
    );
  }

  return (
    <ModelPicker
      picks={modelPicks}
      effectivePick={pickHook.effectivePick}
      userPickKey={pickHook.userPickKey}
      selectPick={pickHook.selectPick}
      pickKeyFn={pickKeyFn}
      pickLabelFn={pickLabelFn}
      disabled={disabled}
      style={{ width: "100%", padding: "8px 10px", fontSize: 12 }}
    />
  );
}
```

#### 3.3.2 WorkflowPicker 组件

```tsx
// ui/src/components/WorkflowPicker.tsx

import { useState, useEffect } from "react";
import { api } from "../api";
import type { WorkflowDisplayInfo } from "../api/types";
import type { ImageModelPick } from "../settings/appSettings";
import { capabilitiesToDisplayLabels } from "../components/settings-tab/workflowAnalyzer";
import { useI18n } from "../i18n";

interface WorkflowPickerProps {
  profileId: string;
  profileName: string;
  pickHook: {
    effectivePick: ImageModelPick | null;
    userPickKey: string | null;
    selectPick: (key: string) => void;
  };
  pickKeyFn: (pick: ImageModelPick) => string;
  pickLabelFn: (pick: ImageModelPick) => string;
  disabled?: boolean;
  label: string;
  helpText: string;
}

/** ComfyUI 工作流选择器。
 *
 * 下拉项格式示例：
 *   基础文生图 [type:t2i] [neg:yes] [cx:simple]
 *   基础图生图 [type:i2i] [ref:yes] [cx:simple]
 *   my_portrait_v2 [type:i2i] [neg:no] [cx:medium]
 *   ---
 *   [管理我的工作流...]  → 打开设置页第四栏
 */
export function WorkflowPicker({
  profileId,
  profileName,
  pickHook,
  pickKeyFn,
  pickLabelFn,
  disabled,
  label,
  helpText,
}: WorkflowPickerProps) {
  const t = useI18n();
  const [workflows, setWorkflows] = useState<WorkflowDisplayInfo[]>([]);
  const [loading, setLoading] = useState(true);
  const [isOpen, setIsOpen] = useState(false);

  useEffect(() => {
    setLoading(true);
    api.listComfyuiWorkflowsForGeneration()
      .then(setWorkflows)
      .catch(() => setWorkflows([]))
      .finally(() => setLoading(false));
  }, []);

  const builtins = workflows.filter((w) => w.source === "builtin");
  const customs = workflows.filter((w) => w.source === "custom");

  // 当前选中的工作流
  const currentWorkflow = workflows.find(
    (wf) => wf.workflowId === pickHook.effectivePick?.model,
  );

  // 将 WorkflowDisplayInfo 转换为 ImageModelPick（供 pickKeyFn 使用）
  const toPick = (wf: WorkflowDisplayInfo): ImageModelPick => ({
    profileId,
    model: wf.workflowId,
    label: wf.label,
    backendId: "comfyui",
    profileName,
  });

  if (loading) {
    return (
      <div style={pickerStyle}>加载中...</div>
    );
  }

  if (workflows.length === 0) {
    return (
      <div style={pickerStyle} title="无可用工作流，请在设置页管理">
        无可用工作流
      </div>
    );
  }

  return (
    <div style={{ position: "relative" }}>
      {/* 当前选中项展示 */}
      <div
        onClick={() => !disabled && setIsOpen(!isOpen)}
        style={{
          ...pickerStyle,
          cursor: disabled ? "not-allowed" : "pointer",
          opacity: disabled ? 0.5 : 1,
        }}
      >
        <span style={{ overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>
          {currentWorkflow?.label ?? "选择工作流"}
        </span>
        {/* 能力标签紧跟在选择项之后 */}
        {currentWorkflow?.capabilities && (
          <span style={{ fontSize: 10, color: "var(--text3)", marginLeft: 6 }}>
            {capabilitiesToDisplayLabels(currentWorkflow.capabilities).join(" · ")}
          </span>
        )}
        <span style={{ fontSize: 10, opacity: 0.6, marginLeft: "auto" }}>
          {isOpen ? "▲" : "▼"}
        </span>
      </div>

      {/* 下拉列表 */}
      {isOpen && (
        <div
          style={{
            position: "absolute",
            top: "100%",
            left: 0,
            right: 0,
            marginTop: 4,
            background: "var(--surface)",
            border: "1px solid var(--border)",
            borderRadius: 6,
            boxShadow: "0 4px 12px rgba(0,0,0,0.15)",
            zIndex: 1000,
            maxHeight: 300,
            overflow: "auto",
          }}
        >
          {builtins.length > 0 && (
            <>
              <div style={groupHeaderStyle}>内置工作流</div>
              {builtins.map((wf) => (
                <WorkflowOptionItem
                  key={wf.workflowId}
                  workflow={wf}
                  pick={toPick(wf)}
                  isSelected={pickHook.effectivePick?.model === wf.workflowId}
                  onSelect={() => {
                    pickHook.selectPick(pickKeyFn(toPick(wf)));
                    setIsOpen(false);
                  }}
                />
              ))}
            </>
          )}
          {customs.length > 0 && (
            <>
              <div style={groupHeaderStyle}>自定义工作流</div>
              {customs.map((wf) => (
                <WorkflowOptionItem
                  key={wf.workflowId}
                  workflow={wf}
                  pick={toPick(wf)}
                  isSelected={pickHook.effectivePick?.model === wf.workflowId}
                  onSelect={() => {
                    pickHook.selectPick(pickKeyFn(toPick(wf)));
                    setIsOpen(false);
                  }}
                />
              ))}
            </>
          )}
          <div style={{ borderTop: "1px solid var(--border)" }}>
            <button
              type="button"
              onClick={() => {
                // 打开设置页并切换到 ComfyUI provider
                const event = new CustomEvent("open-settings-comfyui");
                window.dispatchEvent(event);
              }}
              style={{
                width: "100%",
                padding: "8px 12px",
                fontSize: 11,
                color: "var(--accent)",
                background: "transparent",
                border: "none",
                cursor: "pointer",
                textAlign: "left",
              }}
            >
              管理我的工作流...
            </button>
          </div>
        </div>
      )}
    </div>
  );
}

// ── 子组件：单个工作流选项 ──

function WorkflowOptionItem({
  workflow,
  pick,
  isSelected,
  onSelect,
}: {
  workflow: WorkflowDisplayInfo;
  pick: ImageModelPick;
  isSelected: boolean;
  onSelect: () => void;
}) {
  const labels = workflow.capabilities
    ? capabilitiesToDisplayLabels(workflow.capabilities)
    : [];

  return (
    <div
      onClick={onSelect}
      style={{
        padding: "8px 12px",
        cursor: "pointer",
        borderBottom: "1px solid var(--border)",
        color: isSelected ? "var(--primary)" : "var(--text)",
        background: isSelected ? "var(--surface2)" : "transparent",
      }}
      onMouseEnter={(e) => { (e.currentTarget as HTMLDivElement).style.background = "var(--surface2)"; }}
      onMouseLeave={(e) => { if (!isSelected) (e.currentTarget as HTMLDivElement).style.background = "transparent"; }}
    >
      <div style={{ fontWeight: 500, fontSize: 12 }}>
        {workflow.source === "builtin" ? "" : ""}{workflow.label}
      </div>
      {labels.length > 0 && (
        <div style={{ fontSize: 10, color: "var(--text3)", marginTop: 2 }}>
          {labels.join(" · ")}
        </div>
      )}
    </div>
  );
}

// ── 样式常量 ──

const pickerStyle: React.CSSProperties = {
  padding: "8px 12px",
  border: "1px solid var(--border)",
  borderRadius: 6,
  background: "var(--surface)",
  color: "var(--text)",
  fontSize: 12,
  display: "flex",
  alignItems: "center",
  gap: 8,
};

const groupHeaderStyle: React.CSSProperties = {
  padding: "6px 12px",
  fontSize: 10,
  fontWeight: 700,
  color: "var(--text3)",
  background: "var(--surface2)",
  borderBottom: "1px solid var(--border)",
};
```

#### 3.3.3 ShotModelPickerRow 改造

```tsx
// ui/src/components/storyboard/modals/shot-workbench/components/ShotModelPickerRow.tsx
// 改动：仅一处 — 将 ModelPicker 替换为 SelectionModePicker

// 在 import 中添加：
import { SelectionModePicker } from "../../../../SelectionModePicker";
import { getSelectionMode } from "../../../../../settings/appSettings";

// 在 JSX 中替换文生图列的 ModelPicker：

{/* 文生图列 */}
<label style={{ flex: 1, minWidth: 260, display: "flex", flexDirection: "column", gap: 6 }}>
  <div style={helpText}>
    {t2iIsComfy
      ? t.storyboard.comfyuiWorkflowColumnLabel ?? "工作流（用于 ComfyUI 生图）"
      : "文生图模型"}
  </div>
  <SelectionModePicker
    providerKind={caps.t2iProviderKind ?? ""}
    modelPicks={imageModelPicks}
    pickHook={shotImagePickHook}
    pickKeyFn={imageModelPickKey}
    pickLabelFn={(p: ImageModelPick) => imageModelPickOptionLabel(p, storyboardImageBackendLabel)}
    disabled={disabled}
  />
  <ModelCapabilityCard
    caps={caps.t2i.caps}
    fields={t2iCapFields}
  />
</label>
```

### 3.4 Layer 3：能力体系 — 数据源切换

#### 3.4.1 前端能力数据源切换（改造现有 hook）

```typescript
// ui/src/components/storyboard/modals/shot-workbench/hooks/useShotWorkbenchCapabilities.ts
// 改动：ComfyUI 时，从工作流分析获取能力，而非 providerKind 元数据

import { useMemo, useState, useEffect } from "react";
import { useModelCapabilitySummary } from "../../../../../hooks/useModelCapabilitySummary";
import { getSelectionMode } from "../../../../../settings/appSettings";
import { api } from "../../../../../api";
import type { WorkflowCapabilities } from "../../../../../api/types";

// 在返回值接口中添加：
export interface UseShotWorkbenchCapabilitiesReturn {
  text: CapsEntry;
  t2i: CapsEntry;
  i2i: CapsEntry;
  t2iProviderKind: string | null;
  i2iProviderKind: string | null;
  /** ComfyUI 专用：工作流能力标签（当 t2iProviderKind 为 ComfyUI 时使用） */
  t2iWorkflowCaps: WorkflowCapabilities | null;
  /** ComfyUI 专用：工作流能力标签（当 i2iProviderKind 为 ComfyUI 时使用） */
  i2iWorkflowCaps: WorkflowCapabilities | null;
}

// 在 hook 实现中添加：

export function useShotWorkbenchCapabilities({
  appSettings,
  textPick,
  t2iPick,
  i2iPick,
}: UseShotWorkbenchCapabilitiesArgs): UseShotWorkbenchCapabilitiesReturn {
  // ... 解析 providerKind 部分不变 ...

  // 文本模型能力（不变）
  const textCaps = useModelCapabilitySummary(effectiveTextProviderKind, "text_generation", effectiveTextModel);

  // 文生图能力：根据选择模式决定数据源
  const t2iMode = getSelectionMode(t2iProviderKind);
  const t2iCaps = t2iMode === "workflow"
    ? { summary: null, loading: false, error: null, providerKind: t2iProviderKind, taskType: "text_to_image" }
    : useModelCapabilitySummary(t2iProviderKind, "text_to_image", effectiveT2iModel);

  // ComfyUI 时从工作流分析获取能力
  const [t2iWorkflowCaps, setT2iWorkflowCaps] = useState<WorkflowCapabilities | null>(null);
  useEffect(() => {
    if (t2iMode !== "workflow" || !t2iPick.effectivePick?.model) {
      setT2iWorkflowCaps(null);
      return;
    }
    // t2iPick.effectivePick.model 此时存的是 workflowId
    const wfId = t2iPick.effectivePick.model;
    // 调用后端分析命令
    void api.getComfyuiWorkflow(wfId.replace("custom:", ""))
      .then((json) => {
        // 前端分析（workflowAnalyzer.ts 已有纯函数实现）
        const { analyzeWorkflowCapabilities } = await import(
          "../../../../../components/settings-tab/workflowAnalyzer"
        );
        setT2iWorkflowCaps(analyzeWorkflowCapabilities(JSON.parse(json)));
      })
      .catch(() => setT2iWorkflowCaps(null));
  }, [t2iMode, t2iPick.effectivePick?.model]);

  // 图生图同理
  const i2iMode = getSelectionMode(i2iProviderKind);
  const i2iCaps = i2iMode === "workflow"
    ? { summary: null, loading: false, error: null, providerKind: i2iProviderKind, taskType: "image_to_image" }
    : useModelCapabilitySummary(i2iProviderKind, "image_to_image", effectiveI2iModel);

  const [i2iWorkflowCaps, setI2iWorkflowCaps] = useState<WorkflowCapabilities | null>(null);
  useEffect(() => {
    if (i2iMode !== "workflow" || !i2iPick.effectivePick?.model) {
      setI2iWorkflowCaps(null);
      return;
    }
    const wfId = i2iPick.effectivePick.model;
    void api.getComfyuiWorkflow(wfId.replace("custom:", ""))
      .then((json) => {
        const { analyzeWorkflowCapabilities } = require(
          "../../../../../components/settings-tab/workflowAnalyzer"
        );
        setI2iWorkflowCaps(analyzeWorkflowCapabilities(JSON.parse(json)));
      })
      .catch(() => setI2iWorkflowCaps(null));
  }, [i2iMode, i2iPick.effectivePick?.model]);

  return {
    text: { caps: textCaps },
    t2i: { caps: t2iCaps },
    i2i: { caps: i2iCaps },
    t2iProviderKind,
    i2iProviderKind,
    t2iWorkflowCaps,
    i2iWorkflowCaps,
  };
}
```

#### 3.4.2 ShotModelPickerRow 能力卡片数据源

```tsx
// ShotModelPickerRow.tsx — 能力卡片字段根据模式切换：

<ModelCapabilityCard
  caps={t2iIsComfy ? (caps.t2iWorkflowCaps ? {
    summary: {
      imageReferenceCaps: {
        supportsReferenceImages: caps.t2iWorkflowCaps.requiresReferenceImage,
        maxReferenceImages: caps.t2iWorkflowCaps.requiresReferenceImage ? 1 : 0,
        kindDefaultNoteZh: "",
      },
      maxPromptChars: 0,
      negativePromptCaps: {
        supportsNegativePrompt: caps.t2iWorkflowCaps.hasNegativePrompt,
        maxChars: 0,
        noteZh: caps.t2iWorkflowCaps.hasNegativePrompt ? "由 AI 自动生成" : "此工作流无负面提示词节点",
      },
      rateLimit: { maxConcurrent: 1, maxQps: 1, retryAfterMs: 0, burstAllowance: 0 },
      estimatedDuration: { minMs: 0, maxMs: 0, typicalMs: 0 },
      cost: { currency: "", amount: 0, unit: "free", note: "本地免费" },
      outputSizeRange: caps.t2iWorkflowCaps.defaultSize
        ? {
            minWidth: caps.t2iWorkflowCaps.defaultSize.width,
            maxWidth: caps.t2iWorkflowCaps.defaultSize.width,
            minHeight: caps.t2iWorkflowCaps.defaultSize.height,
            maxHeight: caps.t2iWorkflowCaps.defaultSize.height,
            recommendedSizes: [],
            aspectRatios: [],
          }
        : { minWidth: 512, maxWidth: 2048, minHeight: 512, maxHeight: 2048, recommendedSizes: [], aspectRatios: [] },
      inputConstraints: { supportedFormats: [], unsupportedFormats: [], maxNestingDepth: 0, sanitizeRules: { removeJson: false, removeMarkdown: false, removeEmojis: false, removeSpecialChars: "", maxListItems: 0 }, requiresTranslation: false, targetLanguage: "", preprocessingNote: "" },
    },
    providerKind: "comfyui",
    loading: false,
    error: null,
  } : caps.t2i.caps) : caps.t2i.caps}
  fields={t2iIsComfy && caps.t2iWorkflowCaps
    ? ["imageReference", "negativePrompt"]
    : t2iCapFields}
/>
```

### 3.5 Layer 1：执行边界 — 保持不变

`ImageProvider` trait **不需要修改**。`ImageGenRequest` **不需要修改**。`extra` 透传机制 **不需要修改**。

后端生图链路完整流程（与当前一致）：

1. `resolve_image_service(cfg, kind, override)` → `merge_ai_service(base, override)` → `AiServiceConfig { extra: { comfyui_workflow_template: "basic_t2i" } }`
2. `IMAGE_PROVIDERS.get("comfyui").generate(service, request)`
3. `resolve_workflow_json(service, request)` → 从 `extra.comfyui_workflow_template` 读取 → 返回工作流 JSON
4. `analyzer::analyze_workflow_capabilities(...)` → 返回 `WorkflowCapabilities`
5. `node_deps::check_workflow_node_dependencies(...)` → 检查缺失节点
6. `negative_prompt::generate_negative_prompt(...)` → LLM 生成负面提示词
7. `apply_workflow_placeholders_legacy(tpl, prompt, neg, ckpt, seed, w, h, ref0)` → 占位符替换
8. `post_prompt(service, headers, workflow, client_id)` → 提交到 ComfyUI
9. `poll_until_done(...)` → 等待完成
10. `download_view_to_path(...)` → 下载结果到本地

**这一层完全不变**。

---

## 四、改动清单（精确到行）

| # | 文件 | 操作 | 说明 |
|---|------|------|------|
| 1 | `src-tauri/src/open_platform/provider_metadata.rs` | **新建** | `SelectionMode` enum + `from_kind()` 函数 |
| 2 | `src-tauri/src/open_platform/mod.rs` | +1 行 | `pub mod provider_metadata;` |
| 3 | `src-tauri/src/comfyui/commands.rs` | **新增命令** | `list_comfyui_workflows_for_generation` + `WorkflowDisplayInfo` struct |
| 4 | `src-tauri/src/main.rs` | +1 行 | 注册 `list_comfyui_workflows_for_generation` 命令 |
| 5 | `ui/src/settings/appSettings.ts` | **+~60 行** | `SelectionMode` type + `PROVIDER_SELECTION_MODES` + `getSelectionMode()` + `comfyuiWorkflowsToModelPicks()` + `parseWorkflowPickKey()` + `workflowPickKey()` |
| 6 | `ui/src/api/types.ts` | **+10 行** | `WorkflowDisplayInfo` interface |
| 7 | `ui/src/api/methods/apiPart3.ts` | **+3 行** | `listComfyuiWorkflowsForGeneration` 方法 |
| 8 | `ui/src/hooks/useComfyuiWorkflows.ts` | **新建** | 加载工作流列表并转换为 `ImageModelPick[]` 格式 |
| 9 | `ui/src/components/SelectionModePicker.tsx` | **新建** | 声明式选择器路由组件 |
| 10 | `ui/src/components/WorkflowPicker.tsx` | **新建** | ComfyUI 工作流下拉选择器 |
| 11 | `ui/src/components/storyboard/modals/shot-workbench/components/ShotModelPickerRow.tsx` | **改 ~10 行** | ModelPicker → SelectionModePicker + 标题切换 |
| 12 | `ui/src/components/storyboard/modals/shot-workbench/hooks/useShotWorkbenchCapabilities.ts` | **+~40 行** | 新增 `t2iWorkflowCaps` / `i2iWorkflowCaps` 字段 + ComfyUI 时从工作流分析获取能力 |
| 13 | `ui/src/i18n/zh-CN.ts` | **+5 行** | 新增 i18n key（见下方完整列表） |
| 14 | `ui/src/i18n/en.ts` | **+5 行** | 同上英文版 |
| 15 | `ui/src/i18n/types.ts` | **+5 行** | i18n 类型声明 |

### 完整 i18n key 清单

```typescript
// zh-CN.ts
storyboard: {
  comfyuiWorkflowColumnLabel: "工作流（用于 ComfyUI 生图）",
},
settings: {
  comfyuiWorkflowManageButton: "管理我的工作流...",
  comfyuiNoWorkflowsAvailable: "无可用工作流，请在设置页管理",
  comfyuiWorkflowGroupBuiltin: "内置工作流",
  comfyuiWorkflowGroupCustom: "自定义工作流",
  comfyuiWorkflowLoading: "加载工作流...",
}

// en.ts
storyboard: {
  comfyuiWorkflowColumnLabel: "Workflow (for ComfyUI generation)",
},
settings: {
  comfyuiWorkflowManageButton: "Manage my workflows...",
  comfyuiNoWorkflowsAvailable: "No workflows available, manage in settings",
  comfyuiWorkflowGroupBuiltin: "Built-in workflows",
  comfyuiWorkflowGroupCustom: "Custom workflows",
  comfyuiWorkflowLoading: "Loading workflows...",
}
```

---

## 五、数据流全景（改造后）

```
┌──────────────────────────────────────────────────────────────────┐
│                          用户视角                                 │
│                                                                  │
│  设置页（不变）：                                                  │
│    1. 选择 ComfyUI Provider → 填 Base URL                        │
│    2. 第四栏选工作流 → 写入 profile.extra.comfyui_workflow_template│
│                                                                  │
│  生图页（改造后）：                                                │
│    1. SelectionModePicker 检测 → ComfyUI → 渲染 WorkflowPicker   │
│    2. WorkflowPicker 调用 list_comfyui_workflows_for_generation  │
│    3. 下拉显示：内置工作流 + 自定义工作流（分组展示）              │
│    4. 每个选项附带能力标签：[type:t2i] [neg:yes] [cx:simple]      │
│    5. 选中后：workflowId 写入 ImageModelPick.model               │
│    6. 能力卡片展示工作流真实能力（从 workflowAnalyzer 获取）       │
│    7. 列标题显示"工作流"而非"模型"                                 │
│    8. 点「生成」→ executeSegmentShot()                            │
└───────────────────────────────────────┬──────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────┐
│                    后端执行边界（不变）                             │
│                                                                  │
│  generate_image(kind=ComfyUi, model=workflowId, ...)             │
│    → resolve_effective_image_service()                           │
│       → merge_ai_service(base, override)                         │
│       → AiServiceConfig { extra: { comfyui_workflow_template }}  │
│    → IMAGE_PROVIDERS.get("comfyui")                              │
│    → ComfyUiImageProvider.generate(service, request)             │
│       ├── resolve_workflow_json() ← extra.comfyui_workflow_template│
│       ├── analyzer::analyze_capabilities()                       │
│       ├── node_deps::check_dependencies()                        │
│       ├── negative_prompt::generate()                            │
│       ├── apply_placeholders(ckpt, seed, w, h, prompt, neg, ref) │
│       ├── post_prompt()                                          │
│       ├── poll_until_done()                                      │
│       └── download_view_to_path()                                │
│    → ImageGenResult { urls, seed, prompt_id, execution_time }   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 六、架构验收标准

- [ ] `SelectionMode` 在 `provider_metadata.rs` 集中定义，不在 UI 组件中散落地判断
- [ ] ComfyUI 生图页下拉显示工作流列表，每项附带能力标签
- [ ] 选中工作流后，workflowId 写入 `ImageModelPick.model` 字段
- [ ] `executeSegmentShot` 时，`imageServiceOverride.extra.comfyui_workflow_template` 正确传递
- [ ] 能力卡片数据来源：ComfyUI → 工作流分析；其他 → providerKind 元数据
- [ ] 列标题：ComfyUI 显示"工作流"，其他显示"文生图模型"
- [ ] `ImageProvider` trait 不变，`ImageGenRequest` 不变
- [ ] 云端 API 生图流程完全不受影响
- [ ] `useEffectiveModelPick`、`parseImageModelPickKey`、`imageModelPickKey` 等 hook 不变

---

## 七、与之前版本的区别

| 维度 | 旧版本（v1） | 新版本（v2） |
|------|-------------|-------------|
| `WorkflowPick` 类型 | 独立于 `ImageModelPick` | 不存在，直接复用 `ImageModelPick` |
| 下游 hooks | 需要改造适配新类型 | 完全复用 `useEffectiveModelPick` |
| pick key 函数 | 需要新增 `workflowPickKey` | 复用 `imageModelPickKey` |
| `imageServiceOverrideForPick` | 需要新增 `workflowServiceOverrideForPick` | 完全复用，不变 |
| 改动范围 | 约 15+ 文件 | 约 15 文件（但下游改动量减半） |
| 架构复杂度 | 中等（引入新类型 + 新 hooks） | 低（仅改数据源 + UI 路由） |

**核心区别**：v1 引入了 `WorkflowPick` 作为独立类型，导致所有使用 `ImageModelPick` 的下游基础设施都要改造。v2 保持 `ImageModelPick` 类型不变，**仅改变 ComfyUI 时的数据来源和 UI 渲染方式**。下游 hooks、key 函数、service 构造函数全部复用。

---

## 八、为什么这是最优解

1. **语义正确**：用户看到的是什么就选什么，不用理解"为什么模型显示为空"
2. **数据源正确**：能力信息来自真实的工作流分析，不是 providerKind 固定元数据
3. **扩展性强**：新增 workflow 模式的 provider 只需在 `SelectionMode` 映射加一行
4. **维护成本低**：判断逻辑集中在 `SelectionMode`，不在 UI 组件中散落
5. **改动量最小**：不引入新类型，下游 infrastructure（hooks/keys/services）完全复用
6. **执行边界不变**：编排层的 `generate_image()` 不需要修改
7. **不破坏现有**：云端 API 的 `ModelPicker` 完全不变

**一句话总结**：不在用户层面假装两种东西一样（UI 语义分离），不在执行层面重复两套代码（trait 不变），在中间层做最小改动（数据来源切换 + 声明式路由）。
