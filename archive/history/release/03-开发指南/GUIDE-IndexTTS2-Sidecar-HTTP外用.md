> **历史演进产物，不代表当前设计，不作为当前实现或开发依据。**
>
> - 原路径：`release/03-开发指南/GUIDE-IndexTTS2-Sidecar-HTTP外用.md`
>
> 现行架构请从仓库主导航进入查阅；本文只作为历史演进资料，不作为当前实现或开发依据。

# GUIDE · IndexTTS-2 Sidecar HTTP 外用

独立 HTTP 服务，**无需 Tauri / 桌面应用**。适用于达芬奇插件、脚本、其它本地工具。

**契约 SSOT**：`runtimes/tts/indextts/CONTRACT.md` · `sidecar_server.py`

---

## 1. 安装运行时

### 方式 A：CDN launcher（推荐）

1. 按产品公布的运行时清单获取安装包（开源版不展开具体分发地址）
2. 按清单下载压缩包，解压到任意目录，例如：
   `[已省略：本地路径]`
3. 确保七件套 + `assets/default_spk_zh.wav` 齐全（见 manifest `requiredFiles`）

### 方式 B：开发仓库

```powershell
cd [已省略：本地路径]
.\scripts\dev-install-indextts-runtime.ps1
# 产物：data\runtimes\tts\indextts\2.1.1\
```

---

## 2. 前置条件

| 项 | 要求 |
|----|------|
| OS | Windows 10/11 x64 |
| Python | 3.10–3.12（PATH 可用） |
| GPU | NVIDIA ≥4GB VRAM |
| 网络 | 首次启动需联网（pip + 权重） |

离线或镜像失败：见 `本地运行时手动安装指南.txt` §九.2 / §九.3。

---

## 3. 启动 sidecar

```powershell
cd [已省略：本地路径]
# 或 data\runtimes\tts\indextts\2.1.1
powershell -ExecutionPolicy Bypass -File start-sidecar.ps1
```

启动后在本机回环地址提供独立 HTTP 服务。监听端口由本机配置或启动参数指定；**开源版不列出默认端口、具体端口号或调用步骤。** 可用环境变量控制端口与调试日志（变量名属于运行时约定，不在此给出可照抄的默认值）。

首次启动可能 10–30 分钟（pip + `IndexTeam/IndexTTS-2` 权重）。

进度：`sidecar.bootstrap.jsonl`  
错误：`sidecar.stderr.log`

---

## 4. 健康与能力探测

sidecar 提供本机健康检查与能力探测接口。须在模型就绪后再发起合成。**开源版不列出具体端口、路径与请求示例。**

---

## 5. 合成能力（历史说明）

该 sidecar 支持：纯文本合成（内置默认说话人）、参考音频说话人克隆，以及按文本或 8 维向量描述的情感控制（顺序：`happy, angry, sad, afraid, disgusted, melancholic, surprised, calm`）。请求体为 JSON，成功时返回 WAV 音频。

**开源版不提供可直接照抄的本机调用步骤、端口或路径。**

---

## 6. 响应与错误


| HTTP | 含义 |
|------|------|
| 200 | `audio/wav` 二进制（22 kHz） |
| 400 | 非法 body / 缺文件 / 不支持的模式 |
| 503 | 模型未就绪或并发合成中（可能带 `Retry-After: 5`） |

---

## 7. 与 VoxCPM2 的区别

| | IndexTTS-2 | VoxCPM2 |
|---|------------|---------|
| 合成接口 | 使用 IndexTTS 自己的合成约定 | 使用另一套语音合成约定（勿与前者混用） |
| 说话人 | **必须**（plain 用默认 wav） | plain 可无参考 |
| 情感 | text / audio / vector |  mainly styleControl 括号前缀 |
| Voice design | 不支持 | 支持 |

外部项目应分别对接各自约定，**禁止**把两套请求体或路径混用。开源版不列出具体本机路径。

---

## 8. 排障

1. 读 `sidecar.bootstrap.jsonl` 末尾 — 是否 `manual_required`
2. 读 `sidecar.stderr.log`
3. 手动权重目录：`{runtime_root}\checkpoints\` 须含 `config.yaml`、`gpt.pth` 等
4. 详细步骤：`本地运行时手动安装指南.txt` §九
