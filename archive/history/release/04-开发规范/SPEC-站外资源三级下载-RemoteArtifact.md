> **历史演进产物，不代表当前设计，不作为当前实现或开发依据。**
>
> - 原路径：`release/04-开发规范/SPEC-站外资源三级下载-RemoteArtifact.md`
>
> 现行架构请从仓库主导航进入查阅；本文只作为历史演进资料，不作为当前实现或开发依据。

# SPEC · 站外资源三级下载（RemoteArtifact · INV-REMOTE-DL）

> **开源整理版**（2026-11-12）：已去除或改写私有路径、未公开端点、凭证示例，以及登录 / 认证 / 付费相关实现细节。若某节全部属于上述范围，将改为一句能力边界说明。


**状态**：V1 · 2026-06-28  
**范围**：Sidecar bootstrap pip、HF/ModelScope 权重、`generic_download` 远程 artifact 等  
**SSOT 代码**：`runtimes/_shared/remote_sources.json` · `runtimes/tts/_shared/bgx_remote_fetch.py` · `src-tauri/src/remote_resource/*`

---

## 1. 三级策略（T1 → T2 → T3）

| Tier | 名称 | 行为 | 失败时 |
|------|------|------|--------|
| **T1** | 官方源 | primary URL / 官方 API | 结构化 log → T2 |
| **T2** | 国内镜像 | 按 manifest 声明顺序 | 记录每次失败 → T3 |
| **T3** | 手动落盘 | **硬 Err** + 完整指引 | 禁止 silent 跳过 / 换模型 / 无限 retry |

## 2. 不变式 INV-REMOTE-DL-01~06

| ID | 规则 |
|----|------|
| **01** | 每个 remote artifact 须在 manifest 或 central registry 声明 T1+T2+`manualInstall` |
| **02** | 下载顺序 T1→T2→T3；禁止跳过 T2 |
| **03** | T3 必须 Err，含 targetDir + requiredFiles + guideAnchor |
| **04** | 禁止 silent 换源/换模型/换版本 |
| **05** | bootstrap JSONL / UI / Err message 三处 manual 指引一致 |
| **06** | 新 sidecar PR 不得合并 unless `manifest.remoteArtifacts` 齐全 |

## 3. 镜像源注册表

文件：`runtimes/_shared/remote_sources.json`

| provider id | 用途 |
|-------------|------|
| `huggingface` | HF 仓库 |
| `hf_mirror` | hf-mirror.com |
| `modelscope` | ModelScope |
| `pypi` | PyPI 官方 |
| `pypi_tuna` | 清华 PyPI 镜像 |
| `bgx_cdn` | 自有 runtime CDN |
## 4. Sidecar manifest 声明

每个 TTS sidecar `manifest.json` 含 `remoteArtifacts[]`：

```json
{
  "id": "example",
  "kind": "pip_requirements | huggingface_repo | ...",
  "sources": [{ "provider": "pypi" }, { "provider": "pypi_tuna" }],
  "manualInstall": {
    "guideAnchor": "§九.2",
    "commands": ["..."],
    "targetDirTemplate": "{runtime_root}/checkpoints/",
    "verify": ["config.yaml"]
  }
}
```

IndexTTS-2 示例：`runtimes/tts/indextts/manifest.json`

## 5. Python 侧（sidecar bootstrap）

- `bgx_remote_fetch.ensure_artifacts(manifest, runtime_root, progress_cb)`
- T3：`ManualInstallRequired` → sidecar JSONL `phase=manual_required`
- 禁止 sidecar 内硬编码 HF URL（须读 `remote_sources.json`）

## 6. Rust 侧（非 sidecar 模块）

- `remote_resource::mirror_chain` — HTTP 字节链式镜像
- `remote_resource::manual_install` — T3 payload 与 IPC Err 结构

## 7. 用户可见指引

- 用户手册：`user-manual/08-离线包投放.html`（离线文件清单、网盘、安装位置；随包为安装根 `用户手册.html`）
- 门禁：`scripts/check-remote-artifact-ssot.ps1`

## 8. 门禁

```powershell
.\scripts\check-remote-artifact-ssot.ps1
```

注册于 `preflight-release-checks.ps1` 与 `_preflight-check-registry.ps1`。
