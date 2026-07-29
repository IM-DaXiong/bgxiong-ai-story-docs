# GUIDE · IndexTTS-2 Sidecar HTTP 外用

独立 HTTP 服务，**无需 Tauri / 桌面应用**。适用于达芬奇插件、脚本、其它本地工具。

**契约 SSOT**：`runtimes/tts/indextts/CONTRACT.md` · `sidecar_server.py`

---

## 1. 安装运行时

### 方式 A：CDN launcher（推荐）

1. 下载 manifest：`https://r.bgxiong.com/client/ai-story/runtimes/indextts-windows-x64.json`
2. 按 manifest 下载 zip，解压到任意目录，例如：
   `C:\Tools\indextts\2.1.1\`
3. 确保七件套 + `assets/default_spk_zh.wav` 齐全（见 manifest `requiredFiles`）

### 方式 B：开发仓库

```powershell
cd L:\00Dev\bgxiong-ai-story
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
cd C:\Tools\indextts\2.1.1
# 或 data\runtimes\tts\indextts\2.1.1
powershell -ExecutionPolicy Bypass -File start-sidecar.ps1 --port 18100
```

环境变量：

| 变量 | 默认 | 说明 |
|------|------|------|
| `BGX_TTS_SIDECAR_PORT` | `18100` | HTTP 端口（`--port` 优先） |
| `BGX_TTS_SIDECAR_DEBUG` | off | 设为 `1` 开启 DEBUG 日志 |

首次启动可能 10–30 分钟（pip + `IndexTeam/IndexTTS-2` 权重）。

进度：`sidecar.bootstrap.jsonl`  
错误：`sidecar.stderr.log`

---

## 4. 健康与能力探测

```powershell
curl -s http://127.0.0.1:18100/health
curl -s http://127.0.0.1:18100/v1/indextts/capabilities
```

`health.modelLoaded=true` 后方可合成。

---

## 5. 合成示例

### Plain（内置默认说话人）

```powershell
curl -s -X POST http://127.0.0.1:18100/v1/indextts/synthesize `
  -H "Content-Type: application/json" `
  -d '{"text":"你好，世界。"}' `
  -o hello.wav
```

### 说话人克隆

```powershell
$body = @{
  text = "这是克隆音色测试。"
  speaker = @{ audioPath = "C:\ref\speaker.wav" }
} | ConvertTo-Json -Compress

curl -s -X POST http://127.0.0.1:18100/v1/indextts/synthesize `
  -H "Content-Type: application/json" `
  -d $body `
  -o clone.wav
```

### 情感文本（Qwen）

```powershell
$body = @{
  text = "今天真是美好的一天。"
  speaker = @{ audioPath = "C:\ref\speaker.wav" }
  emotion = @{ mode = "text"; text = "开心、兴奋"; alpha = 1.0 }
} | ConvertTo-Json -Depth 5 -Compress

curl -s -X POST http://127.0.0.1:18100/v1/indextts/synthesize `
  -H "Content-Type: application/json" `
  -d $body `
  -o emotion.wav
```

### 情感向量（8 维）

顺序：`happy, angry, sad, afraid, disgusted, melancholic, surprised, calm`

```json
{
  "text": "……",
  "speaker": { "audioPath": "C:\\ref\\speaker.wav" },
  "emotion": {
    "mode": "vector",
    "vector": [0, 0, 0.8, 0, 0, 0, 0, 0],
    "alpha": 1.0
  }
}
```

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
| 端点 | `POST /v1/indextts/synthesize` | `POST /v1/audio/speech` |
| 说话人 | **必须**（plain 用默认 wav） | plain 可无参考 |
| 情感 | text / audio / vector |  mainly styleControl 括号前缀 |
| Voice design | 不支持 | 支持 |

外部项目应分别对接各自端点，**禁止**共用 VoxCPM OpenAI 兼容 schema。

---

## 8. 排障

1. 读 `sidecar.bootstrap.jsonl` 末尾 — 是否 `manual_required`
2. 读 `sidecar.stderr.log`
3. 手动权重目录：`{runtime_root}\checkpoints\` 须含 `config.yaml`、`gpt.pth` 等
4. 详细步骤：`本地运行时手动安装指南.txt` §九
