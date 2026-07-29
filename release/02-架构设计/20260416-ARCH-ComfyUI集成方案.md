# ComfyUI 本地/内网生图生视频集成方案

## 1. 总体结论

**当前项目架构已具备接入 ComfyUI 的完整基础**。ComfyUI 可作为新的 `ImageProviderKind` / `VideoProviderKind` 插件化接入，不需改动管线、命令层、前端 UI 主体逻辑。

### 1.1 无限扩展能力评估

> 本节说明当 ComfyUI 能力演进（多图参考图、九宫格/三视图、对口型、动作绑定等）时，架构如何支撑。

| 能力变更 | 当前架构是否支持 | 需要的改动 |
|---------|----------------|-----------|
| **参考图支持 1→N 张** | ✅ 已支持（`upload_reference_images` 循环上传，`inject_references_into_workflow` 按序填入） | 无需改动 |
| **一次生成九宫格（9 张）** | ✅ 已支持（`parse_outputs` 收集全部 `images` 数组，`download_and_save_outputs` 循环下载） | 前端展示适配 |
| **三视图（正/侧/背同批）** | ✅ 同上；通过文件名前缀分类可映射用途 | 管线需约定前缀规则 |
| **对口型（视频）** | ✅ `VideoProviderKind::ComfyUi` 复用同一套 submit/poll/download | Workflow 模板不同 |
| **动作绑定 / ControlNet / IPAdapter** | ✅ 不关节点语义，只提交 JSON + 替换占位符 | Workflow 模板含对应节点 |
| **新版 ComfyUI 节点类型变化** | ⚠️ 需占位符引擎（见 §11 解耦设计） | 模板用 `{{PLACEHOLDER}}` 标记，零代码适配 |

**架构能无限扩展的核心原因**：
1. **不解析节点语义**：只提交完整 JSON → 轮询 `/history` → 收集 `outputs` 中所有文件
2. **输出处理无类型假设**：`parse_outputs` 同时读 `images` + `videos`，数量不限
3. **配置通过 `extra` 存任意 JSON**：`comfyui_workflow_template` 是字符串，可存任意复杂工作流
4. **占位符替换引擎（§11）**：注入规则从代码中提取为可配置模板变量，换能力 = 换模板，不需改代码

---

## 2. 现有架构总览

### 2.1 数据流

```
用户操作 → 前端 UI → Tauri IPC → commands.rs
                              ↓
                    ImageProviderKind / VideoProviderKind 解析
                              ↓
                    resolve_image_service() → AiServiceConfig
                              ↓
                    open_platform/{provider}.rs (专用模块)
                    或 image.rs / video.rs (内联调度)
                              ↓
                    HTTP → 云端/本地 API → 返回 b64_json / url
                              ↓
                    管线层 → 写入 portrait_cache / media 目录 → SQLite → 前端展示
```

### 2.2 关键抽象

| 抽象 | 位置 | 作用 |
|------|------|------|
| `ImageProviderKind` (enum) | `src-tauri/src/open_platform/image.rs:224` | 后端图像服务商路由标识 |
| `VideoProviderKind` (enum) | `src-tauri/src/open_platform/video.rs:101` | 后端视频服务商路由标识 |
| `AiServiceConfig` | `src-tauri/src/config.rs:130` | 统一配置结构：`provider` / `base_url` / `api_key` / `model` 等 |
| `GenerateImageParams` | `image.rs:184` | 生图参数统一结构 |
| `AiProviderProfile` | `ui/src/settings/appSettings.ts` | 前端设置页服务商配置 |
| `AppSettings` | SQLite + `AppSettingsProvider` | 运行时权威数据源 |

### 2.3 现有 ImageProviderKind 清单

| Enum 变体 | `as_str()` | 认证方式 | 协议 |
|-----------|-----------|----------|------|
| `VolcengineArk` | `"volcengine_ark"` | Bearer API Key | HTTP POST |
| `JimengOpenAi` | `"jimeng"` | Bearer API Key | HTTP POST (OpenAI 兼容) |
| `VolcengineVisualI2iPortraitV3` | `"volcengine_visual_i2i_portrait_3"` | IAM AK/SK 签名 | HTTP 提交+轮询 |
| `VolcengineJimengT2iV40` | `"volcengine_jimeng_t2i_v40"` | IAM AK/SK 签名 | HTTP 同步异步 |
| `VolcengineJimengT2iV46` | `"volcengine_jimeng_t2i_v46"` | IAM AK/SK 签名 | HTTP 同步异步 |
| `DashscopeWanImage` | `"dashscope_wan_image"` | Bearer API Key | HTTP 同步 |
| `KlingImageGenerationT2i/I2i` | `"kling_image_generation_*"` | Bearer API Key | HTTP 提交+轮询 |
| `KlingOmniImageT2i/I2i` | `"kling_omni_image_*"` | Bearer API Key | HTTP 提交+轮询 |
| **`ComfyUi`** ⭐新增 | **`"comfyui"`** | 可选 API Key | HTTP 提交+轮询 + WebSocket |

> ⚠️ 新增 `ComfyUi` 变体时须同步修改：
> 1. `ImageProviderKind` 枚举定义（`image.rs`）新增 `ComfyUi`
> 2. `ImageProviderKind::as_str()` 方法增加 `ComfyUi => "comfyui"`
> 3. `ImageProviderKind::parse()` 方法增加 `"comfyui" => ComfyUi`（⚠️ 当前为 chained if-else，需在合适位置插入）
> 4. `resolve_image_service()` 函数增加 `ImageProviderKind::ComfyUi` 分支
> 5. `generate_image()` 调度增加 `if kind == ImageProviderKind::ComfyUi` 分支

### 2.4 现有 VideoProviderKind 清单

| Enum 变体 | 认证方式 | 协议 |
|-----------|----------|------|
| `OpenApiCompat` | Bearer API Key | HTTP REST 提交+轮询 |
| `VolcJimengT2v(*)` | IAM AK/SK 签名 | 智能视觉异步 |
| `KlingOmniVideo` | Bearer API Key | HTTP 提交+轮询 |
| **`ComfyUi`** ⭐新增（Phase 3） | 可选 API Key | HTTP 提交+轮询 + WebSocket |

> ⚠️ 视频变体与图片共用 `comfyui.rs` 核心逻辑，仅在 Workflow 模板上有差异。
> 同样须修改 `VideoProviderKind` 枚举、`as_str()`、`parse()` 与调度方法。

### 2.5 前端设置类别

| 设置分类 | 前端常量 | 对应后端类型 |
|----------|----------|-------------|
| `AI_IMAGE` | `"cat.ai.image"` | `imageProviders` |
| `AI_IMAGE_TO_IMAGE` | `"cat.ai.imageToImage"` | `imageToImageProviders` |
| `AI_VIDEO` | `"cat.ai.video"` | `videoProviders` |
| `AI_KEYFRAME` | `"cat.ai.keyframe"` | `keyframeProviders` |

---

## 3. ComfyUI 核心差异分析

| 维度 | 云端服务商 | ComfyUI |
|------|-----------|---------|
| **认证** | Bearer Key / IAM AK/SK | 通常无需认证（可选 API Key） |
| **协议** | REST API | HTTP + **WebSocket** |
| **提交** | `POST /endpoint` | `POST /prompt`（workflow JSON） |
| **状态** | REST 轮询 `GET /status/{id}` | **WebSocket 推送** + `GET /history/{prompt_id}` |
| **结果获取** | 直接返回 / REST 下载 | `GET /view?filename=&type=output` |
| **输入形态** | `prompt` + `model` + 参数 | **完整 workflow JSON**（含节点图） |
| **模型选择** | 云端模型 ID | 本地 checkpoint 名称 |
| **网络** | 公网 HTTPS | 局域网 HTTP (`http://127.0.0.1:8188`) |

---

## 4. ComfyUI API 核心端点

| 端点 | 方法 | 用途 | 备注 |
|------|------|------|------|
| `/prompt` | POST | 提交工作流生成任务 | 返回 `prompt_id` |
| `/history/{prompt_id}` | GET | 查询任务状态与结果 | `status: success/failed` |
| `/view` | GET | 下载输出文件 | `?filename=&type=output&subfolder=` |
| `/queue` | GET | 查看队列状态 | 可选 |
| `/ws` | WebSocket | 实时任务状态推送 | 推荐用于异步进度 |

### 4.1 `/prompt` 请求体

```json
{
  "prompt": {
    "3": {
      "class_type": "KSampler",
      "inputs": {
        "seed": 12345,
        "steps": 20,
        "cfg": 7.0,
        "sampler_name": "euler",
        "scheduler": "normal",
        "denoise": 1.0,
        "positive": ["6", 0],
        "negative": ["7", 0],
        "latent_image": ["5", 0],
        "model": ["4", 0]
      }
    },
    "4": { "class_type": "CheckpointLoaderSimple", "inputs": { "ckpt_name": "v1-5-pruned.safetensors" } },
    "6": { "class_type": "CLIPTextEncode", "inputs": { "text": "提示词", "clip": ["4", 1] } },
    "7": { "class_type": "CLIPTextEncode", "inputs": { "text": "负面提示词", "clip": ["4", 1] } },
    "5": { "class_type": "EmptyLatentImage", "inputs": { "width": 1024, "height": 1024, "batch_size": 1 } },
    "8": { "class_type": "VAEDecode", "inputs": { "samples": ["3", 0], "vae": ["4", 2] } },
    "9": { "class_type": "SaveImage", "inputs": { "images": ["8", 0], "filename_prefix": "bgxiong" } }
  }
}
```

### 4.2 `/history/{prompt_id}` 响应

```json
{
  "prompt_id": {
    "status": { "status_str": "success", "completed": true },
    "outputs": {
      "9": {
        "images": [
          { "filename": "bgxiong_00001_.png", "subfolder": "", "type": "output" }
        ]
      }
    }
  }
}
```

---

## 5. 后端实现方案

### 5.1 新增文件清单

| 文件路径 | 用途 | 预估行数 |
|----------|------|----------|
| `src-tauri/src/open_platform/comfyui.rs` | **ComfyUI 客户端核心** | ~500-700 |
| `src-tauri/src/open_platform/comfyui/workflows.rs` | 工作流模板管理 | ~150-250 |
| `src-tauri/src/model_capabilities.rs` (修改) | ComfyUI 能力边界配置 | +50 |

> ⚠️ 行数预估已较初版增加：新增了 `/upload/image` 上传参考图、`inject_references_into_workflow` 注入逻辑、
> `extract_comfyui_error` 错误解析、中间进度写入 `ai_task` 等功能。

### 5.2 修改文件清单

| 文件路径 | 修改内容 |
|----------|----------|
| `src-tauri/src/open_platform/image.rs` | 新增 `ComfyUi` 变体 + `parse()` 别名 + `resolve_image_service()` 分支 + `generate_image()` 调度；`generate_image()` 函数签名需新增 `db: &DbState` 参数 |
| `src-tauri/src/open_platform/video.rs` | 新增 `ComfyUi` 变体 + `submit_text2video()`/`query_video_task()` 调度 |
| `src-tauri/src/open_platform/mod.rs` | `pub mod comfyui;` |
| `src-tauri/src/config.rs` | **必须**：`AiServiceConfig` 新增 `extra: Option<HashMap<String, Value>>` 字段（当前 struct 无此字段，**必须新增**） |
| `src-tauri/Cargo.toml` | **新增依赖**：`url-encoding = "2"`（URL 编码）；Phase 3 需 `tokio-tungstenite = "0.26"` + `futures-util = "0.3"`（WebSocket）；确认 `reqwest` 已启用 `multipart` feature（参考图上传需要） |

### 5.3 后端核心实现

#### 5.3.1 `comfyui.rs` 核心结构

```rust
// src-tauri/src/open_platform/comfyui.rs

use reqwest::Client;
use serde_json::{json, Value, Map};
use tokio::time::{sleep, Duration, Instant};
use base64::engine::general_purpose::STANDARD as BASE64;
use base64::Engine;

/// 从 ComfyUI 获取生成图片（完整链路：上传参考图 → 提交 → 轮询 → 下载 → 写盘）
/// 返回：(本地文件路径列表, base64 列表 — 用于前端即时预览)
///
/// ⚠️ 注意：`GenerateImageParams` 不含 `db` / `task_id` 字段，这两个参数由调用方
/// （`generate_image()` 调度层）从管线上下文单独传入，不通过 params 传递。
pub async fn generate_comfyui_image(
    service: &AiServiceConfig,
    params: &GenerateImageParams,
    workflow_override: Option<String>,
    task_id: Option<&str>,
    db: &DbState,
) -> Result<(Vec<String>, Vec<String>), String> {
    let base_url = service.base_url.as_deref()
        .filter(|s| !s.is_empty())
        .unwrap_or("http://127.0.0.1:8188");

    // 1. 构建 workflow JSON
    let workflow = build_workflow(
        params.prompt.clone(),
        params.model.as_deref(),
        params.size.as_deref(),
        params.reference_images_b64.first().cloned(),
        workflow_override,
        service,
    )?;

    // 2. 如有参考图，先通过 /upload/image 上传到 ComfyUI 服务器
    let uploaded_filenames = upload_reference_images(base_url, &params.reference_images_b64).await?;

    // 3. 将上传后的文件名注入 workflow（LoadImage 节点）
    let workflow = if !uploaded_filenames.is_empty() {
        inject_references_into_workflow(workflow, &uploaded_filenames)?
    } else {
        workflow
    };

    // 🔴 开发规范（强制）：提交前必须记录最终提示词
    // 将最终 workflow JSON 序列化为字符串，通过 dev_log_final_prompt 落盘
    let final_prompt_for_log = serde_json::to_string_pretty(&workflow)
        .unwrap_or_else(|_| workflow.to_string());
    dev_log_final_prompt(
        "image",
        "comfyui",
        Some(base_url),
        params.model.as_deref().or(service.model.as_deref()),
        &final_prompt_for_log,
    );

    // 4. 可达性检查（快速失败）
    check_comfyui_alive(base_url).await?;

    // 5. 生成 clientId（用于 WebSocket 事件关联；即使不用 WS 也建议发送）
    let client_id = uuid::Uuid::new_v4().to_string();

    // 6. POST /prompt 提交任务
    let prompt_id = submit_prompt(base_url, &workflow, service.api_key.as_deref(), &client_id).await?;

    tracing::info!("ComfyUI prompt submitted: prompt_id={prompt_id} base_url={base_url}");

    // 7. 轮询 /history/{prompt_id} 直到完成（含中间进度写入 ai_task）
    let outputs = poll_until_complete(
        base_url, &prompt_id,
        service.timeout_secs.unwrap_or(300),
        task_id, db,
    ).await?;

    // 8. 下载输出文件 → 写入磁盘（story_media/comfyui/），返回 (本地路径, b64)
    download_and_save_outputs(base_url, &outputs, db).await
}

/// ComfyUI 可达性检查（GET /system_stats，3 秒超时快速失败）
pub async fn check_comfyui_alive(base_url: &str) -> Result<(), String> {
    let client = Client::builder()
        .timeout(Duration::from_secs(3))
        .build()
        .map_err(|e| e.to_string())?;

    let res = client
        .get(format!("{base_url}/system_stats"))
        .send()
        .await
        .map_err(|e| format!("ComfyUI 服务未启动（{base_url}）: {e}"))?;

    if !res.status().is_success() {
        return Err(format!("ComfyUI 可达但返回异常状态: {}", res.status()));
    }
    Ok(())
}
```

#### 5.3.1b `ComfyOutput` 结构体与辅助函数

```rust
/// ComfyUI 输出文件描述
#[derive(Debug, Clone)]
struct ComfyOutput {
    filename: String,
    subfolder: String,
    /// 通常为 "output"
    r#type: String,
}

/// 从 history entry 中解析输出文件列表
fn parse_outputs(entry: &Value) -> Result<Vec<ComfyOutput>, String> {
    let outputs = entry.get("outputs")
        .and_then(|v| v.as_object())
        .ok_or_else(|| "history 响应缺少 outputs 字段".to_string())?;

    let mut result = Vec::new();
    for (_node_id, node_output) in outputs {
        // images 字段
        if let Some(images) = node_output.get("images").and_then(|v| v.as_array()) {
            for img in images {
                if let Some(filename) = img.get("filename").and_then(|v| v.as_str()) {
                    result.push(ComfyOutput {
                        filename: filename.to_string(),
                        subfolder: img.get("subfolder").and_then(|v| v.as_str()).unwrap_or("").to_string(),
                        r#type: img.get("type").and_then(|v| v.as_str()).unwrap_or("output").to_string(),
                    });
                }
            }
        }
        // videos 字段（VHS_VideoCombine 等节点输出视频）
        if let Some(videos) = node_output.get("videos").and_then(|v| v.as_array()) {
            for vid in videos {
                if let Some(filename) = vid.get("filename").and_then(|v| v.as_str()) {
                    result.push(ComfyOutput {
                        filename: filename.to_string(),
                        subfolder: vid.get("subfolder").and_then(|v| v.as_str()).unwrap_or("").to_string(),
                        r#type: vid.get("type").and_then(|v| v.as_str()).unwrap_or("output").to_string(),
                    });
                }
            }
        }
    }

    if result.is_empty() {
        return Err("history 响应中未找到输出文件".to_string());
    }
    Ok(result)
}

/// 解析 "1024x768" 格式字符串为 (width, height)
fn parse_size_to_wh(size: &str) -> (u32, u32) {
    let parts: Vec<&str> = size.split('x').collect();
    if parts.len() == 2 {
        let w = parts[0].parse::<u32>().unwrap_or(1024);
        let h = parts[1].parse::<u32>().unwrap_or(1024);
        (w, h)
    } else {
        (1024, 1024)
    }
}

/// 🔴 上传参考图到 ComfyUI 服务器（必须先上传才能在 workflow 中引用）
///
/// ComfyUI 要求图片先通过 POST /upload/image 上传至服务器本地存储，
/// 然后 workflow 中的 LoadImage 节点通过 filename 引用。
/// 这与云端服务商直接传 base64 的方式不同。
async fn upload_reference_images(
    base_url: &str,
    images_b64: &[String],
) -> Result<Vec<String>, String> {
    if images_b64.is_empty() {
        return Ok(Vec::new());
    }

    let client = Client::builder()
        .timeout(Duration::from_secs(30))
        .build()
        .map_err(|e| e.to_string())?;

    let mut uploaded_filenames = Vec::new();

    for (idx, b64) in images_b64.iter().enumerate() {
        let bytes = BASE64.decode(b64)
            .map_err(|e| format!("参考图 #{idx} base64 解码失败: {e}"))?;

        // 生成唯一文件名避免冲突
        let ts = chrono::Utc::now().format("%Y%m%d_%H%M%S_%3f");
        let filename = format!("bgxiong_ref_{ts}_{idx}.png");

        let part = reqwest::multipart::Part::bytes(bytes)
            .file_name(filename.clone())
            .mime_str("image/png")
            .map_err(|e| e.to_string())?;

        let form = reqwest::multipart::Form::new()
            .part("image", part)
            .text("overwrite", "true");

        let res = client
            .post(format!("{base_url}/upload/image"))
            .multipart(form)
            .send()
            .await
            .map_err(|e| format!("上传参考图 #{idx} 到 ComfyUI 失败: {e}"))?;

        if !res.status().is_success() {
            let status = res.status();
            let text = res.text().await.unwrap_or_default();
            return Err(format!("ComfyUI /upload/image 返回 {status}: {text}"));
        }

        // 响应包含服务器实际保存的文件名（可能与上传不同）
        let v: Value = res.json().await.map_err(|e| e.to_string())?;
        let server_filename = v.get("name")
            .and_then(|v| v.as_str())
            .unwrap_or(&filename)
            .to_string();
        let subfolder = v.get("subfolder")
            .and_then(|v| v.as_str())
            .unwrap_or("")
            .to_string();

        tracing::info!(
            "ComfyUI 参考图上传成功: idx={idx}, server_filename={server_filename}, subfolder={subfolder}"
        );
        uploaded_filenames.push(server_filename);
    }

    Ok(uploaded_filenames)
}

/// 将上传后的参考图文件名注入 workflow（替换/添加 LoadImage 节点）
///
/// 策略：在 workflow 中查找已有的 LoadImage 节点并替换其 image 字段；
/// 若无 LoadImage 节点，则动态添加并连接到 KSampler 的 latent_image 输入。
fn inject_references_into_workflow(
    mut workflow: Value,
    uploaded_filenames: &[String],
) -> Result<Value, String> {
    if let Some(obj) = workflow.as_object_mut() {
        // 查找所有 LoadImage 节点，按上传顺序注入文件名
        let load_image_nodes: Vec<String> = obj.iter()
            .filter(|(_, node)| {
                node.get("class_type").and_then(|v| v.as_str()) == Some("LoadImage")
            })
            .map(|(id, _)| id.clone())
            .collect();

        for (idx, node_id) in load_image_nodes.iter().enumerate() {
            if idx < uploaded_filenames.len() {
                if let Some(node) = obj.get_mut(node_id) {
                    if let Some(inputs) = node.get_mut("inputs").and_then(|v| v.as_object_mut()) {
                        inputs["image"] = json!(uploaded_filenames[idx]);
                    }
                }
            }
        }
    }
    Ok(workflow)
}
```

#### 5.3.2 `submit_prompt` 实现

```rust
async fn submit_prompt(
    base_url: &str,
    workflow: &Value,
    api_key: Option<&str>,
    client_id: &str,     // 用于 WebSocket 事件关联
) -> Result<String, String> {
    let client = Client::builder()
        .timeout(Duration::from_secs(30))
        .build()
        .map_err(|e| e.to_string())?;

    // ⚠️ 必须包含 client_id，否则 WebSocket 事件无法按会话关联
    let body = json!({
        "prompt": workflow,
        "client_id": client_id
    });

    let mut req = client
        .post(format!("{base_url}/prompt"))
        .header("Content-Type", "application/json")
        .json(&body);

    // ComfyUI-Manager API Key（可选）
    if let Some(key) = api_key.filter(|k| !k.is_empty()) {
        req = req.header("Authorization", format!("Bearer {key}"));
    }

    let res = req.send().await.map_err(|e| format!("ComfyUI 提交失败: {e}"))?;

    if !res.status().is_success() {
        let status = res.status();
        let text = res.text().await.unwrap_or_default();
        // 尝试解析 ComfyUI 结构化错误（含 node_errors）
        let detail = serde_json::from_str::<Value>(&text).ok()
            .and_then(|v| {
                v.get("node_errors")
                    .map(|ne| ne.to_string())
                    .or_else(|| v.get("error")
                        .and_then(|e| e.as_str().map(|s| s.to_string())))
            })
            .unwrap_or(text);
        return Err(format!("ComfyUI /prompt 返回 {status}: {detail}"));
    }

    let v: Value = res.json().await.map_err(|e| e.to_string())?;
    v.get("prompt_id")
        .and_then(|v| v.as_str())
        .map(|s| s.to_string())
        .ok_or_else(|| format!("ComfyUI 响应缺少 prompt_id: {v}"))
}
```

#### 5.3.3 `poll_until_complete` 实现

```rust
async fn poll_until_complete(
    base_url: &str,
    prompt_id: &str,
    timeout_secs: u64,
    task_id: Option<&str>,
    db: &DbState,
) -> Result<Vec<ComfyOutput>, String> {
    let client = Client::builder()
        .timeout(Duration::from_secs(10))
        .build()
        .map_err(|e| e.to_string())?;

    let max_time = Instant::now() + Duration::from_secs(timeout_secs);
    let mut poll_count: u64 = 0;

    loop {
        if Instant::now() > max_time {
            // 🔴 超时时写入 ai_task 进度，方便前端显示
            if let Some(tid) = task_id {
                let _ = db.with_conn(|conn| {
                    ai_task::update_progress_merge(conn, tid, json!({
                        "status": "timeout",
                        "comfyuiPromptId": prompt_id
                    }))
                });
            }
            return Err(format!("ComfyUI 任务超时 ({timeout_secs}s, prompt_id={prompt_id})"));
        }

        let res = client
            .get(format!("{base_url}/history/{prompt_id}"))
            .send()
            .await
            .map_err(|e| e.to_string())?;

        let v: Value = res.json().await.map_err(|e| e.to_string())?;

        if let Some(entry) = v.get(prompt_id) {
            if let Some(status) = entry.get("status") {
                if let Some(status_str) = status.get("status_str").and_then(|v| v.as_str()) {
                    match status_str {
                        "success" => {
                            let outputs = parse_outputs(entry)?;
                            return Ok(outputs);
                        }
                        "error" => {
                            // 解析 ComfyUI 结构化错误信息（含 node_errors / messages）
                            let err_detail = extract_comfyui_error(entry);
                            return Err(format!("ComfyUI 任务失败: {err_detail}"));
                        }
                        _ => {
                            // 排队中或执行中，写入中间进度
                            poll_count += 1;
                            if let Some(tid) = task_id {
                                let _ = db.with_conn(|conn| {
                                    ai_task::update_progress_merge(conn, tid, json!({
                                        "comfyuiPollCount": poll_count,
                                        "comfyuiStatus": status_str,
                                        "comfyuiPromptId": prompt_id
                                    }))
                                });
                            }
                            sleep(Duration::from_secs(2)).await;
                            continue;
                        }
                    }
                }
            }
        }
        // 任务尚未出现在 history 中（排队中）
        poll_count += 1;
        if let Some(tid) = task_id {
            let _ = db.with_conn(|conn| {
                ai_task::update_progress_merge(conn, tid, json!({
                    "comfyuiPollCount": poll_count,
                    "comfyuiStatus": "queued",
                    "comfyuiPromptId": prompt_id
                }))
            });
        }
        sleep(Duration::from_secs(2)).await;
    }
}

/// 从 ComfyUI history entry 中提取结构化错误信息
fn extract_comfyui_error(entry: &Value) -> String {
    let status = entry.get("status").unwrap_or(&Value::Null);
    let messages = status.get("messages")
        .and_then(|m| m.as_array())
        .map(|arr| {
            arr.iter()
                .filter_map(|m| m.as_str().or_else(|| Some(&m.to_string())))
                .collect::<Vec<_>>()
                .join("; ")
        })
        .unwrap_or_default();
    let node_errors = entry.get("node_errors")
        .map(|ne| ne.to_string())
        .unwrap_or_default();
    if !messages.is_empty() {
        format!("messages: {messages}")
    } else if !node_errors.is_empty() {
        format!("node_errors: {node_errors}")
    } else {
        "执行失败（无详细错误信息）".to_string()
    }
}
```

#### 5.3.4 `download_and_save_outputs` 实现

> **开发规范**：图片文件必须存到硬盘，数据库只记录存放路径。下载后的文件写入 `{app_data_dir}/story_media/comfyui/` 目录。

```rust
/// 下载 ComfyUI 输出文件 → 写入磁盘 → 返回 (本地相对路径, base64)
async fn download_and_save_outputs(
    base_url: &str,
    outputs: &[ComfyOutput],
    db: &DbState,
) -> Result<(Vec<String>, Vec<String>), String> {
    let client = Client::builder()
        .timeout(Duration::from_secs(120))
        .build()
        .map_err(|e| e.to_string())?;

    let app_data_dir = paths::app_data_dir();
    let media_dir = app_data_dir.join("story_media").join("comfyui");
    std::fs::create_dir_all(&media_dir)
        .map_err(|e| format!("创建 story_media/comfyui 目录失败: {e}"))?;

    let mut local_paths = Vec::new();
    let mut all_b64 = Vec::new();

    for out in outputs {
        // 构建下载 URL
        let mut url = format!("{base_url}/view?filename={}&type={}",
            url_encoding::encode(&out.filename),
            url_encoding::encode(&out.r#type)
        );
        if !out.subfolder.is_empty() {
            url.push_str(&format!("&subfolder={}", url_encoding::encode(&out.subfolder)));
        }

        // 下载文件字节
        let bytes = client
            .get(&url)
            .send()
            .await
            .map_err(|e| format!("ComfyUI 下载文件失败: {e}"))?
            .bytes()
            .await
            .map_err(|e| e.to_string())?;

        // 生成唯一文件名（带时间戳避免冲突）
        let ts = chrono::Utc::now().format("%Y%m%d_%H%M%S_%3f");
        let ext = std::path::Path::new(&out.filename)
            .extension()
            .and_then(|e| e.to_str())
            .unwrap_or("png");
        let save_filename = format!("comfyui_{ts}.{ext}");
        let save_path = media_dir.join(&save_filename);

        // 写入磁盘
        std::fs::write(&save_path, &bytes)
            .map_err(|e| format!("写入文件失败: {save_path:?} - {e}"))?;

        // 记录相对路径（相对于 app_data_dir）
        let relative_path = format!("story_media/comfyui/{save_filename}");
        tracing::info!("ComfyUI 输出已保存: {relative_path} ({} bytes)", bytes.len());

        local_paths.push(relative_path);

        // 同时返回 base64 用于前端即时预览
        let b64 = BASE64.encode(&bytes);
        all_b64.push(b64);
    }

    Ok((local_paths, all_b64))
}
```

#### 5.3.5b 前端图片加载路径处理（⚠️ Windows 兼容性）

> **开发规范（强制）**：ComfyUI 输出文件写入 `{app_data_dir}/story_media/comfyui/` 的相对路径（如
> `story_media/comfyui/comfyui_20260415_120000_000.png`），前端必须经过
> `resolveAppDataMediaPath` + `convertFileSrc` 加载。**Windows 上 `convertFileSrc` 不会自动将
> `\` 转为 `/`**，必须对绝对路径先行 `\` → `/` 替换（参见 AGENTS.md
> 「Tauri 2 本地文件 URL」一节与 `mediaSrcFromAppDataRelative`）。
>
> ```typescript
> // 正确做法（与现有 portrait_cache / story_media 一致）
> const absPath = await api.resolveAppDataMediaPath(relativePath);
> const src = convertFileSrc(absPath.replace(/\\/g, '/'));
> ```
>
> 存入 `portrait_versions` 或 `ai_tasks.last_payload` 时仅使用相对路径（`story_media/comfyui/...`），
> 由 `resolve_app_data_media_path`（后端 Rust）动态解析为绝对路径。

#### 5.3.6 Workflow 构建器

```rust
fn build_workflow(
    prompt: String,
    model: Option<&str>,
    size: Option<&str>,
    reference_image_b64: Option<String>,
    workflow_override: Option<String>,
    service: &AiServiceConfig,  // ⚠️ 需 AiServiceConfig 新增 extra 字段
) -> Result<Value, String> {
    // 方案 A：用户上传的自定义 workflow JSON（优先）
    if let Some(override_json) = workflow_override {
        return inject_prompt_into_workflow(override_json, &prompt, None, model);
    }

    // 方案 B：从 service.extra 读取预存 workflow 模板
    // ⚠️ 当前 AiServiceConfig 无 extra 字段，需新增 extra: Option<HashMap<String, Value>>
    if let Some(extra) = &service.extra {
        if let Some(template) = extra.get("comfyui_workflow_template").and_then(|v| v.as_str()) {
            if !template.trim().is_empty() {
                return inject_prompt_into_workflow(template.to_string(), &prompt, None, model);
            }
        }
    }

    // 方案 C：内置默认 workflow（最小 KSampler 流程）
    // 注意：reference_image_b64 此处仅供参考信息；实际上传已在 generate_comfyui_image 中完成
    build_default_workflow(prompt, model, size, reference_image_b64.is_some())
}
```

#### 5.3.7 可达性检查 + 模型列表获取

```rust
/// 获取 ComfyUI 可用 checkpoint 列表（用于前端模型选择器）
pub async fn list_comfyui_checkpoints(base_url: &str) -> Result<Vec<String>, String> {
    let client = Client::builder()
        .timeout(Duration::from_secs(5))
        .build()
        .map_err(|e| e.to_string())?;

    let res = client
        .get(format!("{base_url}/object_info/CheckpointLoaderSimple"))
        .send()
        .await
        .map_err(|e| format!("ComfyUI 连接失败: {e}"))?;

    let v: Value = res.json().await.map_err(|e| e.to_string())?;

    let checkpoints = v
        .get("input")
        .and_then(|i| i.get("required"))
        .and_then(|r| r.get("ckpt_name"))
        .and_then(|c| c.get(0))
        .and_then(|c| c.as_array())
        .map(|arr| {
            arr.iter()
                .filter_map(|v| v.as_str().map(|s| s.to_string()))
                .collect::<Vec<_>>()
        })
        .unwrap_or_default();

    Ok(checkpoints)
}
```

### 5.4 接入 `image.rs` 调度

```rust
// 在 image.rs 的 generate_image() 函数中新增分支（在 VolcengineJimengT2iV40 之前）：

// ⚠️ dev_log_final_prompt 已在上层统一调用（generate_image 函数开头），
//    因此 ComfyUI 分支不需要重复调用；但 comfyui.rs 内部会在提交前
//    额外记录完整的 workflow JSON（包含节点拓扑），以便排障对比。
if kind == ImageProviderKind::ComfyUi {
    // 从 service.extra 读取自定义 workflow 模板
    // ⚠️ AiServiceConfig 需新增 extra: Option<HashMap<String, Value>> 字段
    let workflow_override = service.extra
        .as_ref()
        .and_then(|e| e.get("comfyui_workflow_template"))
        .and_then(|v| v.as_str())
        .filter(|s| !s.trim().is_empty())
        .map(|s| s.to_string());

    // ⚠️ task_id 和 db 由管线层传入，不在 GenerateImageParams 中
    // 若在非管线上下文（如独立命令）调用，传 None
    return super::comfyui::generate_comfyui_image(
        service,
        &params,
        workflow_override,
        None,   // task_id: 由管线层传入时替换
        &db,    // db: 由调用方传入（如 commands.rs 已有的 DbState）
    )
    .await
    .map_err(|e| decorate_image_provider_error(kind, e))
    .map(|(local_paths, b64)| GenerateImageOutput {
        urls: vec![],
        b64_json: b64,
        provider: kind.as_str().to_string(),
        final_prompt_sent: Some(final_prompt_sent.clone()),
        warnings: vec![],
        // local_paths 可由调用方写入 portrait_versions / media 记录
        // 前端通过 resolveAppDataMediaPath + convertFileSrc 加载
    });
}
```

> **注意**：`generate_image()` 函数签名目前为 `(kind, service, params, prep_text)`，
> **不含** `db` 参数。接入时需要将 `db: &DbState` 作为新增参数传入（或从 `commands.rs`
> 调用层传入）。同时 `ImageProviderKind` 枚举需新增 `ComfyUi` 变体，并更新
> `parse()` / `as_str()` 方法（`as_str()` → `"comfyui"`，`parse("comfyui")` → `ComfyUi`）。

### 5.5 WebSocket 进度推送 + AI Task 集成

ComfyUI 的 WebSocket `clientId` 是**会话级持久 ID**（同一客户端可复用），不应使用 `prompt_id`。

> **策略选择**：WebSocket 与 HTTP 轮询是**两种可选的进度获取方式**，实现上推荐：
> - **Phase 1/2**：仅使用 HTTP 轮询（`poll_until_complete`），每轮写入 `ai_task` 进度（已在 5.3.3 实现）；
> - **Phase 3**：可选启用 WebSocket，在提交任务后**额外**连接 WS 接收实时进度，轮询仅作兜底（WS 断连时仍可完成）。
>
> 推荐将 WebSocket 监听与 `poll_until_complete` 协调：
> WS 监听在后台运行，实时推送进度（每步/每节点）；轮询在 WS 断连或超时后接管。
> 两者通过 `prompt_id` 和 `client_id` 关联同一任务。

```rust
/// WebSocket 实时进度推送（集成 ai_task 进度）
use tokio_tungstenite::{connect_async, tungstenite::Message};
use futures_util::{StreamExt, SinkExt};

/// WebSocket 事件类型
#[derive(Debug, Clone)]
pub enum ComfyProgressEvent {
    /// 任务正在执行某个节点
    Executing { node: String, prompt_id: String },
    /// 进度更新（当前步数 / 总步数）
    Progress { value: u64, max: u64, prompt_id: String },
    /// 任务执行完成
    Executed { node: String, prompt_id: String },
}

/// 解析 ComfyUI WebSocket 消息为进度事件
fn parse_comfy_event(data: &Value) -> Option<ComfyProgressEvent> {
    let msg_type = data.get("type").and_then(|v| v.as_str())?;
    match msg_type {
        "progress" => {
            let value = data.get("value").and_then(|v| v.as_u64()).unwrap_or(0);
            let max = data.get("max").and_then(|v| v.as_u64()).unwrap_or(1);
            Some(ComfyProgressEvent::Progress { value, max, prompt_id: String::new() })
        }
        "executing" => {
            let node = data.get("node").and_then(|v| v.as_str()).unwrap_or("").to_string();
            let pid = data.get("prompt_id").and_then(|v| v.as_str()).unwrap_or("").to_string();
            Some(ComfyProgressEvent::Executing { node, prompt_id: pid })
        }
        "executed" => {
            let node = data.get("node").and_then(|v| v.as_str()).unwrap_or("").to_string();
            let pid = data.get("prompt_id").and_then(|v| v.as_str()).unwrap_or("").to_string();
            Some(ComfyProgressEvent::Executed { node, prompt_id: pid })
        }
        _ => None,
    }
}

/// WebSocket 监听 + 写入 ai_task 进度
///
/// ⚠️ 此函数应与 poll_until_complete 并行运行：
/// - WS 提供实时步骤级进度（KSampler 步数、节点执行状态）
/// - 轮询作为兜底，在 WS 断连后仍能检测完成
/// 推荐模式：tokio::select! { ws_result = comfyui_ws_watch(...) => ..., poll_result = poll_until_complete(...) => ... }
pub async fn comfyui_ws_watch(
    base_url: &str,
    prompt_id: &str,
    client_id: &str,     // 会话级持久 ID，非 prompt_id
    task_id: &str,       // ai_task 的 task_id，用于写入进度
    db: &DbState,
) -> Result<(), String> {
    let ws_url = base_url.replace("http://", "ws://").replace("https://", "wss://");
    let url = format!("{ws_url}/ws?clientId={client_id}");

    let (ws_stream, _) = connect_async(&url)
        .await
        .map_err(|e| format!("ComfyUI WebSocket 连接失败: {e}"))?;

    let (_, mut read) = ws_stream.split();

    while let Some(Ok(msg)) = read.next().await {
        if let Message::Text(text) = msg {
            let event: Value = serde_json::from_str(&text).unwrap_or(Value::Null);
            if let Some(data) = event.get("data") {
                if let Some(ev) = parse_comfy_event(data) {
                    match ev {
                        ComfyProgressEvent::Progress { value, max, .. } => {
                            let pct = if max > 0 { (value as f64 / max as f64 * 100.0) as u64 } else { 0 };
                            // 写入 ai_task 进度，前端 TasksTab 可实时显示
                            db.with_conn(|conn| {
                                ai_task::update_progress_merge(conn, task_id, json!({
                                    "comfyuiProgress": { "step": value, "total": max, "percent": pct }
                                }))
                            })?;
                        }
                        ComfyProgressEvent::Executing { node, .. } => {
                            tracing::info!("ComfyUI WS executing node={node} prompt_id={prompt_id}");
                        }
                        ComfyProgressEvent::Executed { node, .. } => {
                            tracing::info!("ComfyUI WS executed node={node} prompt_id={prompt_id}");
                        }
                    }
                }
            }
        }
    }

    Ok(())
}
```

---

## 6. 前端实现方案

### 6.1 修改文件清单

| 文件路径 | 修改内容 |
|----------|----------|
| `ui/src/settings/appSettings.ts` | 新增 `IMAGE_KIND_COMFYUI` 常量、预设、凭证校验；`AiProviderProfile` 扩展 `extra` 字段 |
| `ui/src/i18n/zh-CN.ts` + `en.ts` | 服务商显示文案 |
| `ui/src/components/ComfyUIWorkflowEditor.tsx` | **可选** 工作流编辑器/上传器 |

### 6.2 `appSettings.ts` 修改

```typescript
// 新增 provider kind 常量
export const IMAGE_KIND_COMFYUI = "image_comfyui";
export const VIDEO_KIND_COMFYUI = "video_comfyui";

// preset 入口（withOfficialPresetProviders 或等效位置）
{
    id: uuidv4(),
    name: "ComfyUI 本地服务",
    providerKind: IMAGE_KIND_COMFYUI,
    baseUrl: "http://127.0.0.1:8188",
    apiKey: "",
    accessKeyId: "",
    secretAccessKey: "",
    models: "",
    extra: {
        comfyui_workflow_template: "" // 可选，用户上传的 workflow JSON
    }
}

// 凭证校验（ComfyUI 仅需 base_url）
export function profileHasImageEndpointCreds(p: AiProviderProfile): boolean {
    // 新增分支
    if (p.providerKind === IMAGE_KIND_COMFYUI) {
        return p.baseUrl.trim().length > 0;
    }
    // ... 现有逻辑
}

// 默认 base URL
export function defaultUrlForProviderKind(kind: string): string {
    switch (kind) {
        case IMAGE_KIND_COMFYUI:
            return "http://127.0.0.1:8188";
        // ... existing
    }
}

// Backend ID 映射（imageServiceOverrideForPick 等）
export function imageProfileBackendId(p: AiProviderProfile): string {
    if (p.providerKind === IMAGE_KIND_COMFYUI) return "comfyui";
    // ... existing
}
```

### 6.3 Settings 设置页表现

ComfyUI 在设置页中的配置项：

| 字段 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| **服务商名称** | 是 | "ComfyUI 本地服务" | 可自定义 |
| **Base URL** | 是 | `http://127.0.0.1:8188` | ComfyUI 监听地址 |
| **API Key** | 否 | 空 | ComfyUI-Manager 可选认证 |
| **模型** | 否 | 空 | Checkpoint 名称（留空时使用 workflow 默认） |
| **自定义 Workflow** | 否 | 空 | 通过 `extra` 存储 JSON 模板 |

---

## 7. ComfyUI 视频支持方案

### 7.1 `VideoProviderKind::ComfyUi` 变体

视频与图片共用同一套 ComfyUI 基础设施，仅在 Workflow 模板上有差异：

```
文生图 Workflow：CheckpointLoader → CLIPTextEncode → KSampler → VAEDecode → SaveImage
视频 Workflow：  CheckpointLoader → CLIPTextEncode → AnimateDiff → KSampler → VAEDecode → VHS_VideoCombine
```

### 7.2 视频集成路径

1. 在 `VideoProviderKind` 中新增 `ComfyUi` 变体
2. `submit_text2video()` 与 `query_video_task()` 复用 `comfyui.rs` 中的 `/prompt` `/history` 逻辑
3. 视频 Workflow 模板通过 `extra.comfyui_video_workflow_template` 区分
4. 输出文件格式为 `mp4`/`gif`，下载后写入 `story_media/` 目录

---

## 8. 工作流模板方案（关键设计决策）

ComfyUI 要求输入完整的 workflow JSON，本项目有 **三种策略** + **占位符扩展**处理：

### 8.1 策略对比

| 策略 | 实现复杂度 | 灵活性 | 用户门槛 | 推荐 |
|------|-----------|--------|----------|------|
| **A. 内置默认** | 低 | 低 | 零 | 初期默认 |
| **B. 用户上传 JSON** | 中 | 高 | 中 | 进阶用户 |
| **C. API 自动生成** | 高 | 极高 | 零 | 最终目标 |
| **D. 占位符模板（新增 §11）** | 低 | **极高** | 低 | **推荐长期方案** |

> **策略 D 说明**：用户在 ComfyUI 导出 API 格式 JSON 后，将需要动态替换的值标记为 `{{PLACEHOLDER}}`。
> 后端通过 `apply_placeholders()` 引擎替换，**零代码适配新能力**。详见 **§11 无限扩展解耦设计**。

### 8.2 内置默认 Workflow 模板

项目启动时内置一个最小可用的 KSampler 工作流：

```rust
fn build_default_workflow(
    prompt: String,
    model: Option<&str>,
    size: Option<&str>,
    has_reference_image: bool,  // 仅用于决定 denoise 等参数；实际上传已在 generate_comfyui_image 中完成
) -> Value {
    let (width, height) = parse_size_to_wh(size.unwrap_or("1024x1024"));
    let ckpt = model.unwrap_or("v1-5-pruned-emaonly.safetensors");

    // 基础节点
    let mut workflow = json!({
        "4": {
            "class_type": "CheckpointLoaderSimple",
            "inputs": { "ckpt_name": ckpt }
        },
        "6": {
            "class_type": "CLIPTextEncode",
            "inputs": { "text": prompt, "clip": ["4", 1] }
        },
        "7": {
            "class_type": "CLIPTextEncode",
            "inputs": { "text": "", "clip": ["4", 1] } // 负面提示
        },
        "5": {
            "class_type": "EmptyLatentImage",
            "inputs": { "width": width, "height": height, "batch_size": 1 }
        },
        // ... KSampler, VAEDecode, SaveImage 等节点
    });

    // 参考图支持：如果提供了参考图，已通过 /upload/image 上传并在 workflow 中注入 LoadImage 节点
    // 此处仅在 KSampler 中将 denoise 降低（img2img 模式）
    if has_reference_image {
        // 在 KSampler 中将 denoise 降低（img2img 模式）
        if let Some(ksampler) = workflow.get_mut("3") {
            if let Some(inputs) = ksampler.get_mut("inputs") {
                inputs["denoise"] = json!(0.7);
                // positive/negative 改为引用 LoadImageEncode 节点输出
            }
        }
    }

    workflow
}
```

### 8.2b 占位符版模板（Phase 2+ 推荐）

与 8.2 等价但使用占位符的模板——未来内置默认将改为此格式，用户也可自行编辑：

```json
{
  "4": {
    "class_type": "CheckpointLoaderSimple",
    "inputs": { "ckpt_name": "{{MODEL_CKPT}}" }
  },
  "6": {
    "class_type": "CLIPTextEncode",
    "inputs": { "text": "{{POSITIVE_PROMPT}}", "clip": ["4", 1] }
  },
  "7": {
    "class_type": "CLIPTextEncode",
    "inputs": { "text": "{{NEGATIVE_PROMPT}}", "clip": ["4", 1] }
  },
  "5": {
    "class_type": "EmptyLatentImage",
    "inputs": { "width": {{WIDTH}}, "height": {{HEIGHT}}, "batch_size": 1 }
  },
  "3": {
    "class_type": "KSampler",
    "inputs": {
      "seed": {{SEED}}, "steps": 20, "cfg": 7.0,
      "sampler_name": "euler", "scheduler": "normal",
      "denoise": 1.0,
      "positive": ["6", 0], "negative": ["7", 0],
      "latent_image": ["5", 0], "model": ["4", 0]
    }
  },
  "8": { "class_type": "VAEDecode", "inputs": { "samples": ["3", 0], "vae": ["4", 2] } },
  "9": { "class_type": "SaveImage", "inputs": { "images": ["8", 0], "filename_prefix": "bgxiong" } }
}
```

> **占位符清单**见 **§11.2**。当前 Phase 1 仍用硬编码 `inject_prompt_into_workflow`；Phase 2 起逐步迁移。

### 8.3 用户上传 Workflow 注入

> **重要**：用户必须在 ComfyUI 网页版中导出 **API 格式 JSON**（菜单 `Save (API Format)`），而非普通的 workflow JSON（`Save`）。普通 workflow JSON 包含 `nodes`/`links`/`last_node_id` 等画布元数据，不是 `/prompt` 端点期望的格式。

后端在提交前通过占位符引擎替换 `{{PLACEHOLDER}}` 值。如模板中不含占位符，则**回退**到现有硬编码 `inject_prompt_into_workflow`（按 `CLIPTextEncode` 节点排序替换）。

```rust
/// 将正/负面提示词注入用户上传的 workflow JSON
fn inject_prompt_into_workflow(
    template: String,
    prompt: &str,
    negative_prompt: Option<&str>,
    model: Option<&str>,
) -> Result<Value, String> {
    let mut workflow: Value = serde_json::from_str(&template)
        .map_err(|e| format!("Workflow JSON 无效: {e}"))?;

    if let Some(obj) = workflow.as_object_mut() {
        // 按节点 ID 排序，确保顺序稳定
        let mut sorted: Vec<_> = obj.iter_mut().collect();
        sorted.sort_by_key(|(id, _)| id.parse::<u64>().unwrap_or(0));

        let mut clip_text_nodes: Vec<&mut Value> = sorted
            .into_iter()
            .map(|(_, node)| node)
            .filter(|node| node.get("class_type").and_then(|v| v.as_str()) == Some("CLIPTextEncode"))
            .collect();

        // 第一个 CLIPTextEncode → 正向提示词
        if let Some(first) = clip_text_nodes.first_mut() {
            if let Some(inputs) = first.get_mut("inputs").and_then(|v| v.as_object_mut()) {
                if inputs.contains_key("text") {
                    inputs["text"] = json!(prompt);
                }
            }
        }

        // 第二个 CLIPTextEncode → 负面提示词
        if let Some(neg) = negative_prompt {
            if let Some(second) = clip_text_nodes.get_mut(1) {
                if let Some(inputs) = second.get_mut("inputs").and_then(|v| v.as_object_mut()) {
                    if inputs.contains_key("text") {
                        inputs["text"] = json!(neg);
                    }
                }
            }
        }
    }

    if let Some(m) = model {
        update_checkpoint_in_workflow(&mut workflow, m);
    }

    Ok(workflow)
}

/// 替换 workflow 中所有 CheckpointLoaderSimple / CheckpointLoader 的 ckpt_name
fn update_checkpoint_in_workflow(workflow: &mut Value, model: &str) {
    if let Some(obj) = workflow.as_object_mut() {
        for (_id, node) in obj.iter_mut() {
            let is_checkpoint_loader = node.get("class_type")
                .and_then(|v| v.as_str())
                .map(|ct| ct == "CheckpointLoaderSimple" || ct == "CheckpointLoader")
                .unwrap_or(false);

            if is_checkpoint_loader {
                if let Some(inputs) = node.get_mut("inputs").and_then(|v| v.as_object_mut()) {
                    if inputs.contains_key("ckpt_name") {
                        inputs["ckpt_name"] = json!(model);
                    }
                }
            }
        }
    }
}
```

---

## 9. 完整集成步骤

### Phase 1：基础文生图支持（核心 MVP）

| 步骤 | 文件 | 工作量 |
|------|------|--------|
| 1. 新增 `ComfyUi` 变体 | `image.rs` | 15 min |
| 2. 实现 `comfyui.rs` | `open_platform/comfyui.rs` | 2-3h |
| 3. 接入 `generate_image()` 调度 | `image.rs` | 10 min |
| 4. 前端新增 preset + 凭证校验 | `appSettings.ts` | 30 min |
| 5. i18n 文案 | `zh-CN.ts` / `en.ts` | 10 min |
| **总计** | | **~3-4 小时** |

### Phase 2：自定义 Workflow 支持 + 占位符引擎

| 步骤 | 文件 | 工作量 |
|------|------|--------|
| 1. Workflow 模板存储与注入 | `comfyui/workflows.rs` | 1-2h |
| 2. 设置页增加 Workflow JSON 输入框 | `SettingsTab.tsx` | 30 min |
| 3. Workflow 验证（启动时检查可达性） | `comfyui.rs` | 30 min |
| **4. 占位符替换引擎 `apply_placeholders()`** | `comfyui.rs` | **1h** |
| **5. 占位符模板内置默认 workflow** | `comfyui/workflows.rs` | **30 min** |
| **6. 输出前缀分类 `classify_outputs_by_prefix()`** | `comfyui.rs` | **30 min** |
| **总计** | | **~4-6 小时** |

### Phase 3：视频 + WebSocket 进度推送

| 步骤 | 文件 | 工作量 |
|------|------|--------|
| 1. `VideoProviderKind::ComfyUi` 变体 | `video.rs` | 15 min |
| 2. 视频 Workflow 模板区分 | `comfyui.rs` | 30 min |
| 3. WebSocket 实时进度推送 | `comfyui.rs` | 2-3h |
| 4. 前端进度事件对接 | `TasksTab.tsx` | 1h |
| **总计** | | **~4-5 小时** |

---

## 10. 风险与注意事项

### 10.1 网络

| 风险 | 描述 | 应对 |
|------|------|------|
| **ComfyUI 未启动** | `http://127.0.0.1:8188` 连接拒绝 | 前端提示"ComfyUI 服务未启动"；后端 3 秒快速失败 |
| **内网地址** | 使用 `http://192.168.x.x:8188` | 提高 `timeout_secs`，网络延迟容忍 |
| **防火墙** | 本地/内网端口被防火墙拦截 | 引导用户检查防火墙规则 |

### 10.2 安全

| 项目 | 说明 |
|------|------|
| **无需 API Key** | ComfyUI 通常无认证，前端不应要求必填 |
| **本地路径泄露** | 用户上传的 workflow JSON 可能含本地路径信息 → 不敏感，属于用户本机 |
| **外部模型文件** | Workflow 引用的 checkpoint 由 ComfyUI 本地管理，本项目不做路径校验 |

### 10.3 性能

| 项目 | 说明 |
|------|------|
| **大体积文件** | ComfyUI 输出 4K PNG 约 10-15MB 原始字节。下载后写入磁盘（`story_media/comfyui/`），仅在前端预览时返回 base64，内存峰值可控 |
| **base64 内存开销** | base64 编码约为原始 1.37x；4K 图 b64 约 14-20MB。确认 `MAX_IMAGE_RESPONSE_SIZE` >= 50MB |
| **阻塞轮询** | 已有 `rAF` 聚合、心跳机制，不影响 UI |
| **GPU 资源占用** | ComfyUI 生图期间 GPU 占用高，与项目其它本地 AI 模型（LLM）注意冲突；建议提示用户避免同时运行 |

### 10.4 依赖（⚠️ 项目当前无以下 crate）

| 依赖 | 版本 | 用途 | 阶段 |
|------|------|------|------|
| `url-encoding` | `"2"` | URL 编码（`/view` 端点文件名含中文/空格） | Phase 1 |
| `reqwest` `multipart` feature | — | 参考图上传（`/upload/image` 端点） | Phase 1 ⚠️ 需确认已启用 |
| `tokio-tungstenite` | `"0.26"` | WebSocket 客户端 | Phase 3 |
| `futures-util` | `"0.3"` | StreamExt（WebSocket split + next） | Phase 3 |

> ⚠️ `base64`、`chrono`、`serde_json`、`reqwest` 已在项目依赖中；`uuid` 需确认是否已包含（用于 `client_id` 生成，若无可使用简单替代方案）。
>
> **Phase 1 仅需 `url-encoding`**（用于 `/view` 端点文件名编码）及确认 `reqwest multipart` feature 已启用。
> `tokio-tungstenite` 与 `futures-util` 为 Phase 3 WebSocket 功能所需，Phase 1/2 不必提前引入。

### 10.5 自定义节点依赖

| 说明 |
|------|
| 内置默认 workflow 仅使用 ComfyUI 基础节点（`CheckpointLoaderSimple`、`CLIPTextEncode`、`KSampler`、`VAEDecode`、`SaveImage`），无需额外安装 |
| 用户上传的 workflow 可能依赖 `AnimateDiff`、`ControlNet`、`IPAdapter`、`VHS_VideoCombine` 等自定义节点，由用户自行在 ComfyUI-Manager 中安装 |
| 本项目不对自定义节点做安装或版本管理，提交 workflow 到 ComfyUI 后如缺少节点会返回错误（`history` 中 `status_str: "error"` + `node_errors`），错误信息会通过 `extract_comfyui_error()` 结构化解析后透传到前端 |

### 10.6 参考图上传（⚠️ 与云端服务商关键差异）

> **重要**：ComfyUI 的参考图机制与云端服务商（方舟、即梦等直接传 base64）**完全不同**。
> ComfyUI 要求先通过 `POST /upload/image` 将图片上传到 ComfyUI 服务器本地存储，
> 然后在 workflow JSON 中通过占位符 `{{REFERENCE_IMAGE_N}}` 引用上传后的文件名。
> 这意味着：
> 1. 参考图上传是 ComfyUI 特有的前置步骤，不可跳过
> 2. 上传后的文件名可能与原始不同（ComfyUI 可能重命名或加后缀）
> 3. 多张参考图需依次上传（使用 `bgxiong_ref_{timestamp}_{idx}.png` 避免冲突）
> 4. **占位符替换引擎（§11.2）** 会将 `{{REFERENCE_IMAGE_0}}`~`{{REFERENCE_IMAGE_N}}` 替换为实际文件名
> 5. 若 workflow 不含占位符，回退到硬编码 `inject_references_into_workflow`（查找 `LoadImage` 节点按序替换）
> 6. `reqwest` 的 `multipart` feature 必须启用（用于 `POST /upload/image` 的 multipart/form-data 请求）

### 10.7 `AiServiceConfig.extra` 字段（⚠️ 需新增）

> **实现前提**：
> - **后端** `AiServiceConfig`（`src-tauri/src/config.rs`）**当前仅有 7 个字段**（`provider` / `base_url` / `api_key` / `access_key_id` / `secret_access_key` / `model` / `timeout_secs`），**无 `extra` 字段**。须**新增** `extra: Option<HashMap<String, Value>>`（`serde(default)`，向后兼容）。
> - **前端** `AiProviderProfile`（`ui/src/settings/appSettings.ts`）同样须**新增** `extra?: Record<string, string>` 字段，用于存储 `comfyui_workflow_template` 等 ComfyUI 专属配置。若当前类型定义无此字段，需在接口中新增。

```rust
// config.rs — 必须新增（当前无此字段）
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct AiServiceConfig {
    // ... 现有 7 个字段
    /// 可选扩展配置（ComfyUI workflow 模板等）
    #[serde(default)]
    pub extra: Option<HashMap<String, Value>>,
}
```

```typescript
// appSettings.ts — 必须新增（当前无此字段）
export interface AiProviderProfile {
    // ... 现有字段
    extra?: Record<string, string>; // ComfyUI: { comfyui_workflow_template: "..." }
}
```

⚠️ `AiServiceConfig.extra` 的值类型为 `Value`（JSON 任意值），而前端 `AiProviderProfile.extra` 的值类型为 `string`（JSON 字符串），通过 Tauri IPC 序列化时需确保对齐。

### 10.8 `generate_image()` 签名变更

> 当前 `generate_image()` 函数签名为 `(kind, service, params, prep_text)`，
> **不含** `db: &DbState` 参数。接入 ComfyUI 需要在签名中新增 `db` 参数（用于写入 `ai_task` 进度），
> 或将进度写入改为调用方（`commands.rs`）负责。所有现有调用点均需适配。

---

## 11. 无限扩展解耦设计（占位符引擎 + 输出分类）

> **目标**：当 ComfyUI 能力发生变化（新节点类型 / 新输出格式 / 新参考图通道）时，**无需修改 Rust 代码**，仅更换 Workflow 模板即可接入。

### 11.1 当前硬编码注入 · 风险分析

现有 `inject_prompt_into_workflow` 与 `inject_references_into_workflow` 做了以下**硬编码假设**：

| 硬编码假设 | 失效场景 | 后果 |
|-----------|---------|------|
| `CLIPTextEncode` 节点 → 第一个填正向提示词 | IPAdapter 用 `IPAdapterPromptEncoder`、Flux 用 `CLIPTextEncodeFlux` | 静默注入失败，生成用 workflow 原始提示词 |
| `CLIPTextEncode` 节点 → 第二个填负面提示词 | ControlNet 流程只有一个 `CLIPTextEncode` | 负面提示词丢失 |
| `LoadImage` 节点 → 按顺序填入上传文件名 | IPAdapter 用 `LoadImageToImage`、ControlNet 用 `LoadImage [Preprocessor]` | 参考图未引用，生成纯文生图 |

### 11.2 占位符替换引擎

**核心思路**：将注入规则从代码中提取为 **Workflow JSON 中的模板占位符**。用户在 ComfyUI 中导出 API 格式 JSON 后，将需要动态填入的值标记为 `{{VARIABLE_NAME}}` 字符串。

**支持的占位符**：

| 占位符 | 替换值 | 示例场景 |
|--------|--------|---------|
| `{{POSITIVE_PROMPT}}` | 用户输入的正向提示词 | 任何生图/生视频 workflow |
| `{{NEGATIVE_PROMPT}}` | 用户输入的负面提示词（可为空字符串） | 需要负面提示的 workflow |
| `{{REFERENCE_IMAGE_0}}` | 第 1 张参考图的 ComfyUI 服务器文件名 | IPAdapter 参考图 |
| `{{REFERENCE_IMAGE_1}}` | 第 2 张参考图的 ComfyUI 服务器文件名 | 多图参考图 |
| `{{REFERENCE_IMAGE_N}}` | 第 N+1 张参考图（N = 0..6） | 最多 7 张参考图 |
| `{{MODEL_CKPT}}` | 用户选择的 checkpoint 名称 | 模型切换 |
| `{{WIDTH}}` | 输出宽度（整数） | 动态尺寸 |
| `{{HEIGHT}}` | 输出高度（整数） | 动态尺寸 |
| `{{SEED}}` | 随机种子（可由系统生成） | 精确复现 |

**引擎实现**（替换现有 `inject_prompt_into_workflow`）：

```rust
/// 占位符替换引擎：扫描 workflow JSON 中的所有字符串值，替换 {{VAR}} 标记
///
/// 返回替换后的 workflow，并报告发生了哪些替换（用于日志记录）
fn apply_placeholders(
    workflow_template: &str,
    replacements: &HashMap<String, String>,
) -> Result<(Value, Vec<(String, String)>), String> {
    let mut workflow: Value = serde_json::from_str(workflow_template)
        .map_err(|e| format!("Workflow JSON 无效: {e}"))?;

    let mut applied: Vec<(String, String)> = Vec::new();

    // 递归遍历所有字符串值
    replace_placeholders_recursive(&mut workflow, replacements, &mut applied);

    Ok((workflow, applied))
}

fn replace_placeholders_recursive(
    value: &mut Value,
    replacements: &HashMap<String, String>,
    applied: &mut Vec<(String, String)>,
) {
    match value {
        Value::String(s) => {
            let mut new_s = s.clone();
            for (placeholder, replacement) in replacements {
                if new_s.contains(placeholder) {
                    new_s = new_s.replace(placeholder, replacement);
                    applied.push((placeholder.clone(), replacement.clone()));
                }
            }
            *s = new_s;
        }
        Value::Object(obj) => {
            for (_, v) in obj.iter_mut() {
                replace_placeholders_recursive(v, replacements, applied);
            }
        }
        Value::Array(arr) => {
            for v in arr.iter_mut() {
                replace_placeholders_recursive(v, replacements, applied);
            }
        }
        _ => {}
    }
}

/// 构建占位符映射（由 generate_comfyui_image 调用）
fn build_replacement_map(
    prompt: &str,
    negative_prompt: Option<&str>,
    model: Option<&str>,
    uploaded_filenames: &[String],
    width: u32,
    height: u32,
    seed: u64,
) -> HashMap<String, String> {
    let mut map = HashMap::new();
    map.insert("{{POSITIVE_PROMPT}}".to_string(), prompt.to_string());
    map.insert("{{NEGATIVE_PROMPT}}".to_string(),
        negative_prompt.unwrap_or("").to_string());
    if let Some(m) = model {
        map.insert("{{MODEL_CKPT}}".to_string(), m.to_string());
    }
    for (idx, fname) in uploaded_filenames.iter().enumerate() {
        map.insert(format!("{{REFERENCE_IMAGE_{idx}}}"), fname.clone());
    }
    map.insert("{{WIDTH}}".to_string(), width.to_string());
    map.insert("{{HEIGHT}}".to_string(), height.to_string());
    map.insert("{{SEED}}".to_string(), seed.to_string());
    map
}
```

**用户示例**（三视图 workflow JSON 片段）：

```json
{
  "6": {
    "class_type": "CLIPTextEncode",
    "inputs": {
      "text": "{{POSITIVE_PROMPT}}",
      "clip": ["4", 1]
    }
  },
  "10": {
    "class_type": "LoadImage",
    "inputs": {
      "image": "{{REFERENCE_IMAGE_0}}",
      "upload": "image"
    }
  },
  "20": {
    "class_type": "EmptyLatentImage",
    "inputs": {
      "width": {{WIDTH}},
      "height": {{HEIGHT}},
      "batch_size": 3
    }
  },
  "30": {
    "class_type": "SaveImage",
    "inputs": {
      "images": ["8", 0],
      "filename_prefix": "three_view_front_"
    }
  }
}
```

**优点**：
- **零代码适配新能力**：用户换用 IPAdapter workflow？只需在 ComfyUI 中将参考图节点标记为 `{{REFERENCE_IMAGE_0}}`，上传模板即可
- **透明**：替换日志在 `dev_log_final_prompt` 中完整记录（含 `applied` 列表），排障时一目了然
- **安全**：占位符只替换匹配项，未标记的原始值保持不变
- **向后兼容**：现有硬编码 `inject_*` 函数保留作为 fallback；当 workflow 中不含占位符时自动退回到硬编码模式

### 11.3 多输出 · 文件名前缀分类

九宫格/三视图中，后端收集到的是一组无差别的输出文件。管线需知道哪个文件对应哪个用途。

**方案**：利用 ComfyUI `SaveImage` 节点的 `filename_prefix`。不同输出的模板设置不同前缀，下载后按前缀分类返回。

```rust
/// 按文件名前缀分类输出（用于九宫格、三视图等 batch 输出场景）
///
/// 前缀通过 workflow 模板中的 filename_prefix 字段约定。
/// 返回 HashMap，key 为前缀（去掉末尾下划线），value 为该前缀对应的文件列表（按文件名排序）
fn classify_outputs_by_prefix(
    outputs: &[ComfyOutput],
) -> HashMap<String, Vec<String>> {
    let mut map: HashMap<String, Vec<String>> = HashMap::new();

    for out in outputs {
        // 从文件名中提取前缀（约定格式：{prefix}_[0-9]{4}_.{ext}）
        // 例：three_view_front_00001_.png → prefix = "three_view_front"
        let prefix = extract_prefix_from_filename(&out.filename);
        map.entry(prefix)
            .or_default()
            .push(out.filename.clone());
    }

    // 每个前缀组内按文件名排序（天然序号序）
    for (_, files) in map.iter_mut() {
        files.sort();
    }

    map
}

fn extract_prefix_from_filename(filename: &str) -> String {
    // 正则匹配：去掉末尾 _NNNN_ 部分
    // three_view_front_00001_.png → three_view_front
    let stem = std::path::Path::new(filename)
        .file_stem()
        .and_then(|s| s.to_str())
        .unwrap_or(filename);
    // 去掉末尾的数字序号部分（_00001 等）
    stem.trim_end_matches(|c: char| c.is_ascii_digit() || c == '_').to_string()
}
```

**在 `generate_comfyui_image` 返回值中的体现**：

```rust
/// 返回值改为带分类信息的结构（保持原有 Vec<String> 向后兼容）
pub struct ComfyUiResult {
    /// 所有输出文件的本地相对路径（不分前后缀，向后兼容）
    pub all_paths: Vec<String>,
    /// 所有输出文件的 base64（前端预览）
    pub all_b64: Vec<String>,
    /// 按前缀分类的路径映射（管线层使用，如 {"three_view_front": ["...", "..."]})
    pub by_prefix: HashMap<String, Vec<String>>,
}
```

**管线层消费示例**（场次概念图生成管线）：

```rust
let result = comfyui::generate_comfyui_image(...).await?;
// 按管线语义分类消费
if let Some(front) = result.by_prefix.get("three_view_front") {
    // 写入正面图记录
}
if let Some(side) = result.by_prefix.get("three_view_side") {
    // 写入侧面图记录
}
```

### 11.4 迁移路径

| 阶段 | 注入方式 | 说明 |
|------|---------|------|
| **Phase 1** | 硬编码 `inject_*` | 当前实现，快速上线 MVP |
| **Phase 2** | 占位符 + 硬编码 fallback | 新模板支持占位符，旧模板回退到 `inject_*` |
| **Phase 2.5** | 占位符为主 | 内置默认 workflow 改用占位符模板 |
| **Phase 3** | 仅占位符 | 移除硬编码注入逻辑，全面占位符化 |

---

## 12. 测试方案

### 12.1 单元测试

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_build_default_workflow_has_required_nodes() {
        let wf = build_default_workflow("a cat".to_string(), Some("v1-5.safetensors"), Some("512x512"), false);
        assert!(wf.get("4").is_some()); // CheckpointLoader
        assert!(wf.get("6").is_some()); // CLIPTextEncode (prompt)
        assert!(wf.get("5").is_some()); // EmptyLatentImage
    }

    #[test]
    fn test_inject_prompt_replaces_positive_and_negative() {
        let template = r#"{
            "6": { "class_type": "CLIPTextEncode", "inputs": { "text": "old positive" } },
            "7": { "class_type": "CLIPTextEncode", "inputs": { "text": "old negative" } }
        }"#;
        let result = inject_prompt_into_workflow(
            template.to_string(),
            "new positive",
            Some("new negative"),
            None,
        ).unwrap();
        assert_eq!(result["6"]["inputs"]["text"], "new positive");
        assert_eq!(result["7"]["inputs"]["text"], "new negative");
    }

    #[test]
    fn test_update_checkpoint_in_workflow() {
        let mut wf = json!({
            "4": { "class_type": "CheckpointLoaderSimple", "inputs": { "ckpt_name": "old.ckpt" } }
        });
        update_checkpoint_in_workflow(&mut wf, "new.ckpt");
        assert_eq!(wf["4"]["inputs"]["ckpt_name"], "new.ckpt");
    }

    #[test]
    fn test_parse_size_to_wh() {
        assert_eq!(parse_size_to_wh("1024x768"), (1024, 768));
        assert_eq!(parse_size_to_wh("512x512"), (512, 512));
        assert_eq!(parse_size_to_wh("invalid"), (1024, 1024)); // 回退默认
    }

    // §11 占位符引擎测试
    #[test]
    fn test_placeholder_replaces_prompt() {
        let template = r#"{
            "6": { "class_type": "CLIPTextEncode", "inputs": { "text": "{{POSITIVE_PROMPT}}" } }
        }"#;
        let mut replacements = HashMap::new();
        replacements.insert("{{POSITIVE_PROMPT}}".to_string(), "a black cat".to_string());
        let (result, applied) = apply_placeholders(template, &replacements).unwrap();
        assert_eq!(result["6"]["inputs"]["text"], "a black cat");
        assert_eq!(applied.len(), 1);
    }

    #[test]
    fn test_placeholder_replaces_numeric() {
        // 数字占位符（无引号）在 JSON 中作为 number 值存储
        let template = r#"{
            "5": { "class_type": "EmptyLatentImage", "inputs": { "width": {{WIDTH}}, "height": 512 } }
        }"#;
        let mut replacements = HashMap::new();
        replacements.insert("{{WIDTH}}".to_string(), "1024".to_string());
        let (result, applied) = apply_placeholders(template, &replacements).unwrap();
        assert_eq!(result["5"]["inputs"]["width"].as_u64(), Some(1024));
        assert_eq!(applied.len(), 1);
    }

    #[test]
    fn test_placeholder_no_match_keeps_original() {
        let template = r#"{
            "6": { "class_type": "CLIPTextEncode", "inputs": { "text": "static prompt" } }
        }"#;
        let mut replacements = HashMap::new();
        replacements.insert("{{POSITIVE_PROMPT}}".to_string(), "new".to_string());
        let (result, applied) = apply_placeholders(template, &replacements).unwrap();
        assert_eq!(result["6"]["inputs"]["text"], "static prompt");
        assert_eq!(applied.len(), 0); // 未匹配任何占位符
    }

    #[test]
    fn test_classify_outputs_by_prefix() {
        let outputs = vec![
            ComfyOutput { filename: "three_view_front_00001_.png".to_string(), subfolder: "".to_string(), r#type: "output".to_string() },
            ComfyOutput { filename: "three_view_front_00002_.png".to_string(), subfolder: "".to_string(), r#type: "output".to_string() },
            ComfyOutput { filename: "three_view_side_00001_.png".to_string(), subfolder: "".to_string(), r#type: "output".to_string() },
        ];
        let map = classify_outputs_by_prefix(&outputs);
        assert_eq!(map["three_view_front"].len(), 2);
        assert_eq!(map["three_view_side"].len(), 1);
    }
}
```

### 12.2 集成测试

| 测试场景 | 验证项 |
|----------|--------|
| ComfyUI 正常启动 | `/prompt` 提交成功 → `/history` 返回 `success` → 图片写入 `story_media/comfyui/` → 相对路径返回 |
| 连接失败 | 3 秒内返回明确错误："ComfyUI 服务未启动（http://127.0.0.1:8188）" |
| 可达性检查 | `GET /system_stats` 成功 → 才继续提交；失败立即返回错误 |
| 超时 | 设置 `timeout_secs=30`，30 秒后返回超时错误 |
| Workflow 注入（硬编码 fallback） | 上传 API 格式 workflow 不含占位符时，`CLIPTextEncode` 节点正确替换正/负面提示词 |
| Workflow 注入（占位符） | 上传 workflow 含 `{{POSITIVE_PROMPT}}` 等占位符，替换后提交；`dev_log_final_prompt` 记录 `applied` 列表 |
| 多参考图注入 | 上传 3 张参考图 → `{{REFERENCE_IMAGE_0}}~{{REFERENCE_IMAGE_2}}` 正确替换 |
| Checkpoint 替换 | `{{MODEL_CKPT}}` 占位符被替换为用户指定 checkpoint |
| 九宫格输出前缀分类 | 生成 9 张图（`ninj宫_00001_..png` ~ `_00009_`）→ `classify_outputs_by_prefix()` 返回 `{"ninj宫": [9 files]}` |
| 三视图输出前缀分类 | 生成正/侧/背各 1 张 → `by_prefix` 含 3 个 key，各 1 个文件 |
| 视频生成 | AnimateDiff workflow 提交 → mp4 文件下载并写入 `story_media/comfyui/` |
| 模型列表 | `GET /object_info/CheckpointLoaderSimple` 返回可用 checkpoint 名称列表 |

---

## 13. 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        BACKEND                                │
│                                                              │
│  commands.rs ── ImageProviderKind::parse("comfyui")          │
│       ├── resolve_image_service(cfg, kind) → AiServiceConfig │
│       ├── GenerateImageParams { prompt, model, size, ..     }│
│       └── image::generate_image(kind, &service, params)      │
│                                                              │
│  open_platform/image.rs ── dispatch ──► comfyui.rs           │
│                                                              │
│  comfyui.rs:                                                 │
│    0. GET /system_stats ──► 可达性检查（3s 超时快速失败）    │
│    1. build_workflow(prompt, model, size, ref, override)     │
│    1a. POST /upload/image ──► 上传参考图（如有）获得 filename  │
│    1b. build_replacement_map(prompt, model, uploaded, ..)    │
│    1c. apply_placeholders(workflow, replacements)            │
│        └── 无占位符时 fallback: inject_prompt_into_workflow  │
│    1d. inject_references_into_workflow（硬编码 fallback）    │
│    2. dev_log_final_prompt("image", "comfyui", workflow)    │
│    3. POST /prompt ──────► prompt_id = "abc-123"            │
│    4. GET /history/abc-123 ── 轮询(+进度写入ai_task) ──►     │
│       status: success                                        │
│    5. GET /view?filename=out.png&type=output ───► bytes      │
│    6. bytes → 写入 story_media/comfyui/ + base64             │
│    7. classify_outputs_by_prefix() → by_prefix 分类映射      │
│    8. 返回 ComfyUiResult{ all_paths, all_b64, by_prefix }    │
│                                                              │
│  dev_log_final_prompt("image", "comfyui", ..., final_prompt) │
│  → logs/bgxiong-ai-story.log.YYYY-MM-DD                      │
└──────────────────────┬───────────────────────────────────────┘
                       │ HTTP
┌──────────────────────▼───────────────────────────────────────┐
│                    ComfyUI Server                            │
│                                                              │
│  http://127.0.0.1:8188  (或内网地址)                        │
│  POST /upload/image ◄── 上传参考图（multipart/form-data）       │
│  POST /prompt  ◄──────  workflow JSON (+ client_id)          │
│  GET  /history/{id} ──►  状态/结果                          │
│  GET  /view  ◄─────────  下载输出文件                       │
│  WS   /ws  ──(可选)──►  实时进度推送                         │
│                                                              │
│  内部：Checkpoint → CLIP → KSampler → VAE → SaveImage       │
└──────────────────────────────────────────────────────────────┘
```

---

## 14. 总结

| 维度 | 评估 |
|------|------|
| **架构兼容性** | 优秀。Provider 路由架构成熟，ComfyUI 仅需 ~700 行代码 + 占位符引擎 ~100 行 |
| **前端改动量** | 小。新增 preset + 凭证校验 + Workflow JSON 输入框 + `extra` 字段，约 150 行 |
| **管线无感知** | 完全不影响。所有已有管线按 `ImageProviderKind` 调用，ComfyUI 只是新的路由目标 |
| **无限扩展性** | **高**。占位符引擎 + 输出前缀分类 → 换新能力 = 换模板，**零 Rust 代码改动** |
| **MVP 工作量** | ~4-5 小时（Phase 1，含 `AiServiceConfig.extra` 新增与 `generate_image` 签名变更） |
| **全功能工作量** | ~12-16 小时（含 Phase 2+3 + 占位符引擎 + 输出前缀分类） |
| **风险** | 低-中。ComfyUI API 稳定；参考图上传与云端差异已处理；占位符引擎提供 fallback |

**核心优势**：ComfyUI 复用现有 `AiServiceConfig`（需新增 `extra` 字段）/ `GenerateImageParams` / `dev_log_final_prompt` / `ai_task` 进度体系 / 管线架构。大部分已有功能（模型选择、超时设置、日志记录、磁盘存储）自动继承。

**已补全的关键设计点**：
- ✅ 可达性检查（`GET /system_stats`，3 秒快速失败）
- ✅ 参考图上传（`POST /upload/image`，先上传后在 workflow 中引用文件名）
- ✅ 文件写入磁盘（`story_media/comfyui/`），数据库只存相对路径
- ✅ Windows 路径处理（`convertFileSrc` 需 `\` → `/` 替换，与项目规范一致）
- ✅ AI Task 进度集成（HTTP 轮询写入中间进度 + WebSocket 实时推送）
- ✅ `dev_log_final_prompt`（提交前记录完整 workflow JSON）
- ✅ 正/负面提示词注入（硬编码 fallback + 占位符引擎 `{{POSITIVE_PROMPT}}`）
- ✅ Checkpoint 替换（`{{MODEL_CKPT}}` 占位符 + 硬编码 fallback）
- ✅ 模型列表获取（`GET /object_info/CheckpointLoaderSimple`）
- ✅ 自定义节点依赖说明（由用户自行安装，本项目不管理）
- ✅ ComfyUI 结构化错误解析（`node_errors` + `messages`）
- ✅ `client_id` 在 `/prompt` 请求中传递（WebSocket 事件关联）
- ✅ `AiServiceConfig.extra` 字段新增（当前 struct 无此字段，需新增）
- ✅ `generate_image()` 签名变更（需新增 `db` 参数用于进度写入）
- ✅ `ImageProviderKind` / `VideoProviderKind` 新增 `ComfyUi` 变体及 `parse()`/`as_str()` 映射
- ✅ **占位符替换引擎 `apply_placeholders()`**（零代码适配新能力，§11.2）
- ✅ **输出前缀分类 `classify_outputs_by_prefix()`**（九宫格/三视图用途映射，§11.3）
- ✅ `ComfyUiResult` 结构化返回值（`all_paths` + `all_b64` + `by_prefix`，管线层消费）
- ✅ 向后兼容路径（Phase 1 硬编码 → Phase 2 占位符 + fallback → Phase 3 全面占位符化）

### 14.1 未来能力接入指南

当 ComfyUI 能力发生以下变化时，**无需修改 Rust 代码**，只需：

| 新能力 | 操作 |
|--------|------|
| IPAdapter / ControlNet | 用户在 ComfyUI 中导出含 IPAdapter 节点的 API 格式 JSON，将参考图节点标记为 `{{REFERENCE_IMAGE_0}}`，上传为自定义 workflow 模板 |
| 九宫格批量生图 | workflow 设置 `batch_size: 9` + `filename_prefix: "nine_grid_"`；管线读 `result.by_prefix["nine_grid"]` 获取 9 张图 |
| 三视图 | workflow 设置 3 个 `SaveImage` 节点，`filename_prefix` 分别为 `three_view_front_` / `_side_` / `_back_` |
| 对口型（视频） | 视频 workflow 中设置 `{{POSITIVE_PROMPT}}` + 参考音频节点占位符，通过 `VideoProviderKind::ComfyUi` 提交 |
| 动作绑定 / 骨骼控制 | workflow 含对应节点，标记需要动态注入的字段为占位符；后端只负责替换占位符提交 |

**核心原则**：ComfyUI 的任何新能力 = 新的节点拓扑 + 新的输入字段。只要用户在 workflow 模板中标记了需要动态替换的值为 `{{PLACEHOLDER}}`，后端就能自动处理，**架构对 ComfyUI 能力进化零耦合**。
