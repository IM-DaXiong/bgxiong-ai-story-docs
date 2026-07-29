**文档时间**：2026-06-30T12:00:00+0800

# GUIDE · VoxCPM 2 开发者手册

> **用途**：本仓集成 VoxCPM 2 本地 TTS 时的**技术参考 SSOT**；涵盖官方 API、生成模式、质量调优、部署、微调与故障排查。  
> **官方文档来源**（整理基准）：[API 参考](https://voxcpm.readthedocs.io/zh-cn/latest/reference/api.html) · [快速开始](https://voxcpm.readthedocs.io/zh-cn/latest/quickstart.html) · [安装](https://voxcpm.readthedocs.io/zh-cn/latest/installation.html) · [使用指南](https://voxcpm.readthedocs.io/zh-cn/latest/usage_guide.html) · [最佳实践](https://voxcpm.readthedocs.io/zh-cn/latest/cookbook.html) · [FAQ](https://voxcpm.readthedocs.io/zh-cn/latest/faq.html) · [微调指南](https://voxcpm.readthedocs.io/zh-cn/latest/finetuning/finetune.html) · [微调 FAQ](https://voxcpm.readthedocs.io/zh-cn/latest/finetuning/faq.html)  
> **本仓集成**：Sidecar / CDN / Rich TTS → **`plans/20260628T140000+0800-PLAN-VoxCPM2全能力Sidecar-CDN与Rich本地TTS-架构审核版.md`** · Sidecar 源码 **`runtimes/tts/voxcpm/`**

---

## 目录

1. [概述与版本选型](#1-概述与版本选型)
2. [环境要求与安装](#2-环境要求与安装)
3. [运行时设备选择](#3-运行时设备选择)
4. [Python API 完整参考](#4-python-api-完整参考)
5. [命令行接口（CLI）](#5-命令行接口cli)
6. [三种生成模式](#6-三种生成模式)
7. [文本输入与韵律控制](#7-文本输入与韵律控制)
8. [参考音频与音色一致性](#8-参考音频与音色一致性)
9. [质量调优与长文本策略](#9-质量调优与长文本策略)
10. [流式生成](#10-流式生成)
11. [LoRA 与微调](#11-lora-与微调)
12. [性能、显存与部署](#12-性能显存与部署)
13. [故障排查 FAQ](#13-故障排查-faq)
14. [本仓 Sidecar 集成契约](#14-本仓-sidecar-集成契约)
15. [产品开发速查表](#15-产品开发速查表)

---

## 1. 概述与版本选型

### 1.1 VoxCPM 2 定位

VoxCPM 2（推荐 checkpoint **`openbmb/VoxCPM2`**）是 OpenBMB 发布的**扩散式 TTS** 模型（约 2B 参数），特点：

| 能力 | 说明 |
|------|------|
| 多语种 | 官方宣称支持约 **30 种语言**（中/英/日/韩及多欧洲、亚洲、中东语言） |
| Voice Design | 无参考音频，用括号内控制指令描述音色 |
| 声音克隆 | VoxCPM 2 支持 **`reference_wav_path` 仅参考音**（无需转写） |
| Hi-Fi 克隆 | `prompt_wav_path` + `prompt_text` + `reference_wav_path` 三件套，保真度最高 |
| 流式 | `generate_streaming()` 增量产出波形块 |
| LoRA / 全量微调 | 单说话人克隆、领域适配、新语言扩展 |

> **架构注意**：VoxCPM 生成的是**连续音频 latent**（扩散 DiT），**不兼容**标准 LLM 推理框架（vLLM、lmdeploy 等）。高吞吐部署请用 [NanoVLLM-VoxCPM](https://voxcpm.readthedocs.io/zh-cn/latest/deployment/nanovllm_voxcpm.html)。

### 1.2 版本对照

| 版本 | 参数量 | 编码采样率 | 输出采样率 | 备注 |
|------|--------|-----------|-----------|------|
| VoxCPM 1.0 | ~0.5B | 16 kHz | — | 克隆需 prompt+转写，无 `reference_wav_path` |
| VoxCPM 1.5 | ~0.8B | 44.1 kHz | — | 同上 |
| **VoxCPM 2** | ~2B | **16 kHz**（AudioVAE 编码） | **48 kHz**（解码输出） | **新项目默认** |

本仓 Sidecar 冻结：**PyPI `voxcpm`** + 权重 **`openbmb/VoxCPM2`**（详见 PLAN §2.1）。

---

## 2. 环境要求与安装

### 2.1 软件依赖

| 依赖 | 要求 |
|------|------|
| Python | **3.10–3.12**（3.10–3.11 测试最充分；**勿用 3.14+**） |
| PyTorch | **2.5.0+** |
| CUDA | 可选；NVIDIA GPU 加速建议 **CUDA 12.0+** |
| 磁盘 | 模型权重数 GB（VoxCPM 2 完整 checkpoint 较大） |

### 2.2 pip 安装（推荐）

```bash
pip install voxcpm
```

从源码（Web Demo / 微调 / 贡献）：

```bash
git clone https://github.com/OpenBMB/VoxCPM.git
cd VoxCPM
pip install -e .
```

验证：

```bash
python -c "from voxcpm import VoxCPM; print('VoxCPM is ready')"
```

### 2.3 uv 安装（备选）

```bash
uv pip install voxcpm
# 源码开发
git clone https://github.com/OpenBMB/VoxCPM.git && cd VoxCPM && uv sync
uv run python app.py   # Web Demo
```

### 2.4 Hugging Face 镜像（中国大陆）

```bash
export HF_ENDPOINT=https://hf-mirror.com
```

首次 `from_pretrained("openbmb/VoxCPM2")` 会自动下载权重。

### 2.5 最小可运行示例

```python
from voxcpm import VoxCPM
import soundfile as sf

model = VoxCPM.from_pretrained(
    "openbmb/VoxCPM2",
    load_denoiser=False,
)

wav = model.generate(
    text="VoxCPM 2 is the current recommended release for realistic multilingual speech synthesis.",
    cfg_value=2.0,
    inference_timesteps=10,
)
sf.write("demo.wav", wav, model.tts_model.sample_rate)
print("saved: demo.wav")
```

---

## 3. 运行时设备选择

### 3.1 自动选择

| API / CLI | 行为 |
|-----------|------|
| `device=None` 或 `device="auto"` | 回退顺序：**cuda → mps → cpu** |
| CLI `--device auto` | 同上 |

### 3.2 显式指定

| 值 | 说明 |
|----|------|
| `"cpu"` | 强制 CPU |
| `"mps"` | Apple Silicon MPS |
| `"cuda"` / `"cuda:0"` | 指定 CUDA 设备 |

**重要**：显式指定时**不会静默回退**；设备不可用会直接报错。

### 3.3 torch.compile 优化（`optimize`）

| 场景 | 建议 |
|------|------|
| NVIDIA CUDA 生产推理 | `optimize=True`（默认） |
| CPU / MPS / Windows 调试 / 多线程服务 | `optimize=False` 或 CLI `--no-optimize` |
| Mac MPS 报错 | 回退 `device="cpu", optimize=False` |

`optimize=True` 启用 `torch.compile`，主要对 **CUDA** 有意义；非 CUDA 环境可能触发 Triton / einops 兼容问题。

---

## 4. Python API 完整参考

### 4.1 类 `VoxCPM`

#### 构造函数

```python
VoxCPM(
    voxcpm_model_path,
    zipenhancer_model_path='iic/speech_zipenhancer_ans_multiloss_16k_base',
    enable_denoiser=True,
    optimize=True,
    device=None,
    lora_config=None,
    lora_weights_path=None,
)
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `voxcpm_model_path` | str | 本地模型目录（含权重、config、分词器） |
| `zipenhancer_model_path` | str \| None | ModelScope 降噪模型 id 或本地路径；`None` 跳过降噪器 |
| `enable_denoiser` | bool | 是否初始化 ZipEnhancer 降噪流水线 |
| `optimize` | bool | 是否 `torch.compile` 加速 |
| `device` | str \| None | 见 §3 |
| `lora_config` | LoRAConfig \| None | LoRA 配置 |
| `lora_weights_path` | str \| None | 预训练 LoRA 权重（`.pth` 或含 `lora_weights.ckpt` 的目录） |

模型架构（`voxcpm` / `voxcpm2`）根据 `config.json` 的 `architecture` 字段自动检测。

#### 类方法 `from_pretrained`

```python
VoxCPM.from_pretrained(
    hf_model_id='openbmb/VoxCPM2',
    load_denoiser=True,
    zipenhancer_model_id='iic/speech_zipenhancer_ans_multiloss_16k_base',
    cache_dir=None,
    local_files_only=False,
    optimize=True,
    device=None,
    lora_config=None,
    lora_weights_path=None,
    **kwargs,
)
```

| 参数 | 说明 |
|------|------|
| `hf_model_id` | Hugging Face 仓库 id 或**本地目录路径** |
| `load_denoiser` | 是否加载降噪器 |
| `local_files_only` | `True` 时仅本地文件，不联网下载 |
| `cache_dir` | Hub 快照缓存目录 |

本仓 Sidecar 使用：

```python
VoxCPM.from_pretrained(
    str(CHECKPOINTS_DIR),
    load_denoiser=False,
    local_files_only=True,
)
```

### 4.2 `generate()` — 非流式合成

```python
wav: np.ndarray = model.generate(
    text,                          # 必填
    prompt_wav_path=None,
    prompt_text=None,
    reference_wav_path=None,       # 仅 VoxCPM 2
    cfg_value=2.0,
    inference_timesteps=10,
    min_len=2,
    max_len=4096,
    normalize=False,
    denoise=False,
    retry_badcase=True,
    retry_badcase_max_times=3,
    retry_badcase_ratio_threshold=6.0,
)
```

| 参数 | 默认 | 说明 |
|------|------|------|
| `text` | — | 待合成文本；Voice Design 时在正文前加 `(控制指令)` |
| `prompt_wav_path` | None | 延续式克隆 prompt 音频，须与 `prompt_text` 配对 |
| `prompt_text` | None | prompt 音频逐字转写 |
| `reference_wav_path` | None | **VoxCPM 2**：独立音色克隆参考音，可单独使用 |
| `cfg_value` | 2.0 | CFG 引导强度，建议 **1.0–3.0** |
| `inference_timesteps` | 10 | 扩散步数，建议 **4–30** |
| `min_len` / `max_len` | 2 / 4096 | token 长度上下限 |
| `normalize` | False | 文本规范化（展开数字、日期等） |
| `denoise` | False | 对 prompt/参考音频降噪（需已加载降噪器） |
| `retry_badcase` | True | 音频时长异常偏短/偏长时自动重试 |
| `retry_badcase_max_times` | 3 | 坏例重试上限 |
| `retry_badcase_ratio_threshold` | 6.0 | 坏例检测阈值 |

**返回**：一维 `float32` 波形；采样率 = `model.tts_model.sample_rate`（VoxCPM 2 输出 **48 kHz**）。

**异常**：

- `ValueError`：`text` 为空；`prompt_wav_path`/`prompt_text` 未同时提供；VoxCPM 1.x 使用 `reference_wav_path`
- `FileNotFoundError`：音频路径不存在

### 4.3 `generate_streaming()` — 流式合成

参数与 `generate()` 完全一致；返回 `Generator[np.ndarray, None, None]`，增量产出波形块。

```python
import numpy as np

chunks = []
for chunk in model.generate_streaming(text="Streaming output."):
    chunks.append(chunk)
wav = np.concatenate(chunks)
```

> 官方建议：交互场景按**句**切分后逐句流式，比对不断增长的长文本流式更稳。真正的「文本 token 与音频双向同步流」目前不支持。

### 4.4 LoRA 运行时 API

| 方法 | 说明 |
|------|------|
| `load_lora(lora_weights_path)` | 加载 LoRA；返回 `(loaded_keys, skipped_keys)` |
| `unload_lora()` | 重置 LoRA 权重（层保留但不起作用） |
| `set_lora_enabled(enabled: bool)` | 启用/禁用 LoRA |
| `get_lora_state_dict()` | 获取当前 LoRA 参数 |
| `lora_enabled: bool` | 是否已加载 LoRA 配置 |

热切换与 `torch.compile` 兼容。

---

## 5. 命令行接口（CLI）

默认模型：`openbmb/VoxCPM2`。旧版扁平 CLI（`voxcpm --text ...`）已弃用，请用子命令。

### 5.1 `voxcpm design` — 声音设定 / 直接合成

```bash
voxcpm design --text "Hello world" --output out.wav
voxcpm design --text "Hello world" --control "warm female voice" --output out.wav
voxcpm design --text "Hello" --device cpu --no-optimize --output out.wav
```

### 5.2 `voxcpm clone` — 声音克隆

```bash
# Reference-only（VoxCPM 2）
voxcpm clone --text "Hello" --reference-audio ref.wav --output out.wav

# Hi-Fi 克隆
voxcpm clone --text "Hello" \
  --prompt-audio ref.wav --prompt-text "Transcript of ref.wav" \
  --reference-audio ref.wav --output out.wav

# 克隆 + 风格控制
voxcpm clone --text "Hello" --reference-audio ref.wav \
  --control "speaking slowly" --output out.wav
```

### 5.3 `voxcpm batch` — 批量

```bash
voxcpm batch --input texts.txt --output-dir ./outs
voxcpm batch --input texts.txt --output-dir ./outs --reference-audio ref.wav
```

每行文本生成 `output_001.wav`、`output_002.wav` …

### 5.4 常用 CLI 参数

| 类别 | 参数 | 默认 | 说明 |
|------|------|------|------|
| 生成 | `--text` / `-t` | — | 待合成文本 |
| | `--control` | — | 音色/风格指令；**不可**与 `--prompt-text` 同用 |
| | `--cfg-value` | 2.0 | CFG，建议 1.0–3.0 |
| | `--inference-timesteps` | 10 | 扩散步数，建议 4–30 |
| | `--normalize` | off | 文本规范化 |
| 音频 | `--prompt-audio` / `-pa` | — | 延续 prompt 音频 |
| | `--prompt-text` / `-pt` | — | prompt 转写 |
| | `--reference-audio` / `-ra` | — | 参考音（VoxCPM 2） |
| | `--denoise` | off | prompt/参考音降噪 |
| 模型 | `--model-path` | — | 本地模型目录（优先于 hf-model-id） |
| | `--hf-model-id` | openbmb/VoxCPM2 | HF 仓库 id |
| | `--device` | auto | auto / cpu / mps / cuda / cuda:N |
| | `--local-files-only` | off | 仅本地 |
| | `--no-denoiser` | off | 跳过降噪模型 |
| | `--no-optimize` | off | 禁用 torch.compile |
| LoRA | `--lora-path` | — | LoRA 权重目录 |
| | `--lora-r` / `--lora-alpha` / `--lora-dropout` | 32 / 16 / 0.0 | LoRA 超参 |

---

## 6. 三种生成模式

### 6.1 模式 A：声音设定（Voice Design）

**无参考音频**。在正文前用括号写控制指令，模型从零生成新声音。

```python
wav = model.generate(
    text="(A young woman, gentle and sweet voice)Hello, welcome to VoxCPM!",
    cfg_value=2.0,
    inference_timesteps=10,
)
```

- 中英文控制指令均可：`(年轻女性，温柔甜美)`、`(an excited young man)`
- 每次调用音色可能略有随机性
- CLI 等价：`voxcpm design --control "warm female voice" ...`

### 6.2 模式 B：可控声音克隆（Reference + Style）

提供 **`reference_wav_path`** 克隆音色；括号内指令调节语速、情绪、风格；**无需参考音转写**。

```python
wav = model.generate(
    text="(slightly faster, cheerful tone)This is a cloned voice with style control.",
    reference_wav_path="speaker.wav",
    cfg_value=2.0,
    inference_timesteps=10,
)
```

> 克隆不能任意改变说话人身份，适合在原始音色上调节情绪、语速、表达方式。

### 6.3 模式 C：高保真克隆（Hi-Fi）

同时提供参考音 + **逐字转写**，对齐 prompt 音频，保真度最高。

```python
wav = model.generate(
    text="This is a high-fidelity cloned voice.",
    prompt_wav_path="speaker.wav",
    prompt_text="The exact transcript of speaker.wav goes here.",
    reference_wav_path="speaker.wav",
    cfg_value=2.0,
    inference_timesteps=10,
)
```

- 转写建议用 ASR（Web Demo 用 SenseVoice-Small 自动转写）
- **Hi-Fi 模式下控制指令会被忽略**
- VoxCPM 1.x **不支持** `reference_wav_path`，仅 prompt+转写

### 6.4 模式对照表（产品开发）

| 产品语义 | VoxCPM API | 本仓 Sidecar `mode` |
|----------|-----------|---------------------|
| 纯文本 / 随机音色 | `generate(text=...)` | `plain` |
| 声音设定 | `generate(text="(control)正文")` | `voice_design` + `voiceDesign.controlPrompt` |
| 参考音克隆 | `reference_wav_path` | `reference_clone`（无 `promptText`） |
| Hi-Fi 克隆 | prompt + reference 三件套 | `reference_clone` + `promptText` |
| 克隆 + 风格 | reference + `(style)正文` | `reference_clone` + `styleControl` |

---

## 7. 文本输入与韵律控制

### 7.1 普通文本 vs 音素

| 类型 | 用法 |
|------|------|
| 普通文本 | 默认；数字/日期需规范朗读时设 `normalize=True` |
| 音素 | 细粒度发音控制；**关闭** `normalize` |

```python
# 中文拼音（带声调数字）
wav = model.generate(text="{ni3}{hao3}{shi4}{jie4}", normalize=False)

# 英文 CMUDict 音素
wav = model.generate(text="{HH AH0 L OW1}", normalize=False)
```

### 7.2 文本规范化

```python
wav = model.generate(text="总建筑面积为5640平方米", normalize=True)
```

部分专名、商品名可能仍需人工预处理。

### 7.3 标点与停顿

| 标点 | 效果 |
|------|------|
| 句号、问号 | 较清晰句末停顿 |
| 逗号 | 较短停顿 |
| 省略号 | 迟疑、拖尾 |

需要更强停顿时，**拆成更短句**，不要只依赖标点。

### 7.4 短文本

极短输入（如 `"Hello"`、`"好的"`）可能偏弱（训练最短约 1 秒）。自然产出数秒语音的输入更稳定。

### 7.5 方言

**正文须用地道方言词汇**，不要用标准普通话硬写：

| 方言 | 推荐 | 不推荐 |
|------|------|--------|
| 粤语 | `(广东话，中年男性)伙計，唔該一個A餐，凍奶茶少甜！` | 用普通话写「伙计，麻烦来一个 A 餐…」 |
| 四川话 | `幺儿，哈戳戳得你屋头来噶！` | 普通话对应句 |
| 东北话 | `你搁这整啥玩意儿呢？` | — |

不会写方言时，可先用 LLM 把普通话改写成方言版本；控制指令里写方言名即可（如 `Cantonese`）。

### 7.6 非语言标签（VoxCPM 2 最佳实践）

在正文中插入**英文方括号**标签增加表现力（少量使用）：

| 类别 | 推荐标签 |
|------|----------|
| 笑与叹息 | `[laughing]`, `[sigh]` |
| 停顿与思考 | `[Uhm]`, `[Shh]` |
| 疑问语气 | `[Question-ah]`, `[Question-ei]`, `[Question-en]`, `[Question-oh]` |
| 情绪 | `[Surprise-wa]`, `[Surprise-yo]`, `[Dissatisfaction-hnn]` |

使用全小写形式（如 `[laughing]`）通常比变体更稳定；一句内不要叠太多标签。

### 7.7 Voice Design 控制指令要素（Cookbook）

合并身份、质感、场景于一条 Control Instruction：

| 要素 | 示例 |
|------|------|
| 身份 | middle-aged male broadcaster, elderly woman |
| 质感 | low-pitched, raspy, magnetic |
| 表达 | passionate, shouting campaign slogans, historical narration |

示例（中文）：`热情洋溢的中年男性播音员，声音较为低沉，富有磁性与感染力，带着逐渐密集的节奏感呼喊宣讲口号`

---

## 8. 参考音频与音色一致性

### 8.1 音频质量要求

| 项 | 建议 |
|----|------|
| 时长 | **5–30 秒**（至少 5 秒） |
| 格式 | WAV / FLAC / MP3 等 torchaudio 支持格式 |
| 音质 | 越干净越利于保留音色 |
| 语言 | VoxCPM 2 支持 30 种语言 |

### 8.2 无参考音时

每次调用会**随机**一种声音；风格可从文本推断，但**跨调用音色不一致**。

### 8.3 保持音色一致

1. 每次复用**同一段参考音频**
2. 克隆模式使用 `reference_wav_path`
3. 生产级一致性可考虑 **LoRA 微调**（见 §11）

### 8.4 降噪（`denoise`）

- 改善的是 **prompt/参考音频**，不是生成结果本身
- 参考音嘈杂时 `denoise=True` 有用
- 参考音已干净时 `denoise=False` 保留原声特征
- ZipEnhancer 在 16 kHz 流水线运行，可能轻微改变声线；克隆变差时可关闭

---

## 9. 质量调优与长文本策略

### 9.1 CFG（`cfg_value`）

| 取值 | 效果 |
|------|------|
| 1.0–2.0 | 更自然，可能略偏离文本 |
| **2.0** | 默认均衡 |
| 2.0–3.0 | 更贴文本，难例上易有噪声/瑕疵 |

长音频发糊、发嗡：降到 **1.5–1.6** 附近往往更稳。

### 9.2 扩散步数（`inference_timesteps`）

- 越多：细节与自然度越好，速度越慢
- 建议范围：**4–30**；默认 **10**

### 9.3 长文本

长文本易触发：逐渐加速/嗡嗡声、KV cache 显存不足、生成停不下来。

**推荐**：切分为较短段落，逐段生成后拼接：

```python
import numpy as np

segments = ["第一段……", "第二段……", "第三段……"]
all_wavs = []
for seg in segments:
    wav = model.generate(text=seg, reference_wav_path="voice.wav")
    all_wavs.append(wav)
full_wav = np.concatenate(all_wavs)
```

### 9.4 首尾伪影

- VoxCPM 1.x：检查 `prompt_text` 是否与参考音完全一致
- 启用 `retry_badcase=True`
- 降低 `cfg_value`
- 后处理裁切：`librosa.effects.trim(wav)`

### 9.5 坏例重试

`retry_badcase=True`（默认）：当生成音频相对文本异常偏短或偏长时自动重试，参数 `retry_badcase_max_times`、`retry_badcase_ratio_threshold` 可调。

---

## 10. 流式生成

```python
import numpy as np
import soundfile as sf

chunks = []
for chunk in model.generate_streaming(
    text="Streaming text to speech is easy with VoxCPM!",
):
    chunks.append(chunk)
wav = np.concatenate(chunks)
sf.write("streaming.wav", wav, model.tts_model.sample_rate)
```

**推荐交互流程**：

1. 按句切分输入
2. 每句调用 `generate_streaming()`
3. 按序播放或缓冲

本仓 Sidecar **当前未交付**流式 HTTP（chunk SSE）；见 PLAN「明确不交付」。

---

## 11. LoRA 与微调

### 11.1 选型指南

| 目标 | 数据量 | 推荐 |
|------|--------|------|
| 克隆单个说话人 | 5–50 条 | **LoRA** |
| 适配领域/风格 | 50–500 条 | LoRA（r=32–64） |
| 新增语言 | 500+ 小时 | **全量微调** + 混入部分中英数据 |
| 大规模定制 | 1000+ 条 | 全量微调 |

LoRA（r=32）在单说话人基准上相似度约为全量微调的 **98%**，显存约减半。

### 11.2 训练数据格式（JSONL）

每行一个样本：

```json
{"audio": "path/to/audio1.wav", "text": "Transcript of audio 1."}
{"audio": "path/to/audio2.wav", "text": "Transcript 2.", "ref_audio": "path/to/ref.wav"}
{"audio": "path/to/audio3.wav", "text": "Optional fields.", "duration": 3.5, "dataset_id": 1}
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `audio` | 是 | 音频路径（建议 WAV） |
| `text` | 是 | 与音频一致的转写 |
| `ref_audio` | 否 | 同说话人参考 clip；建议 **30–50%** 样本含此字段 |
| `duration` | 否 | 秒，加速过滤 |
| `dataset_id` | 否 | 多数据集混合 ID |

### 11.3 音频与预处理

| 项 | 要求 |
|----|------|
| 采样率（VoxCPM 2 训练） | `sample_rate: 16000`（**编码器**输入；非 48 kHz 输出） |
| 片段时长 | **3–30 秒**；<1 s 不稳定 |
| 尾静音 | 裁到 **≤0.5 s**（过长是「生成停不下来」最常见原因） |
| 转写 | 须与音频逐字一致 |
| 音量 | 不一致时做归一化 |
| 噪声 | 剔除噪声样本 |

### 11.4 LoRA 训练配置要点（VoxCPM 2）

```yaml
pretrained_path: /path/to/VoxCPM2/
train_manifest: /path/to/train.jsonl
sample_rate: 16000
out_sample_rate: 48000
batch_size: 16
learning_rate: 0.0001      # LoRA
max_batch_tokens: 8192

lora:
  enable_lm: true
  enable_dit: true          # 对音质至关重要
  enable_proj: false
  r: 32                     # 克隆 32；风格/语言 64
  alpha: 32
  dropout: 0.0
```

启动：

```bash
python scripts/train_voxcpm_finetune.py --config_path conf/your_lora_config.yaml
# 多卡
CUDA_VISIBLE_DEVICES=0,1,2,3 torchrun --nproc_per_node=4 \
  scripts/train_voxcpm_finetune.py --config_path conf/your_lora_config.yaml
```

### 11.5 LoRA 推理

```python
model = VoxCPM.from_pretrained(
    "openbmb/VoxCPM2",
    lora_weights_path="/path/to/checkpoints/lora/latest",
)
wav = model.generate(text="Hello from the fine-tuned model.")
```

或运行时热切换：`load_lora` / `unload_lora` / `set_lora_enabled`。

### 11.6 全量微调要点

- `learning_rate`: **1e-5**（约为 LoRA 的 1/10）
- VoxCPM 2 约需 **~40 GB** 显存（batch 16）
- checkpoint 为完整模型目录，可直接 `from_pretrained(ckpt_dir)`
- 更易过拟合：常 **1–2 epoch** 即最优

### 11.7 训练监控与停止条件

| 指标 | 含义 |
|------|------|
| `loss/diff` | 扩散损失，应下降后趋平 |
| `loss/stop` | 停止预测损失，应较早稳定 |
| `grad_norm` | 尖峰可能表示坏样本或 LR 过高 |

**停止信号**：

- 单说话人通常 **1–3 epoch** 足够
- 模型开始**忽略输入文本**（过拟合）→ 回退较早 checkpoint
- 验证损失与听感不一致时，多存几个 checkpoint 试听

### 11.8 微调常见问题

| 问题 | 处理 |
|------|------|
| OOM | 降 `batch_size` / `max_batch_tokens`；增 `grad_accum_steps`；改 LoRA |
| 损失不收敛 | 降 LR；增 warmup；查数据质量/转写 |
| 忽略文本（过拟合） | 保持 `training_cfg_rate=0.1`；`weight_decay=0.01`；早停 |
| 生成停不下来 | 裁训练数据尾静音；推理 `retry_badcase=True` |
| LoRA 效果差 | 增大 `r`；确保 `enable_dit: true`；核对推理侧 LoRA 配置一致 |

---

## 12. 性能、显存与部署

### 12.1 显存与 RTF（RTX 4090，compile + timesteps=10）

| 模型 | 显存 | RTF（官方 generate） | RTF（NanoVLLM） |
|------|------|---------------------|-----------------|
| VoxCPM 1.0 | ~5 GB | ~0.17 | ~0.10 |
| VoxCPM 1.5 | ~6 GB | ~0.15 | ~0.08 |
| **VoxCPM 2** | **~8 GB** | **~0.3** | **~0.13** |

本仓 manifest 标注 **`minVramGb: 8`**。

### 12.2 高吞吐部署

| 方案 | 说明 |
|------|------|
| [NanoVLLM-VoxCPM](https://voxcpm.readthedocs.io/zh-cn/latest/deployment/nanovllm_voxcpm.html) | 官方推荐并发服务 |
| vLLM-Omni | 生态选项 |
| VoxCPM.cpp / ONNX / ANE / MLX | 各平台专用部署 |

**禁止**用标准 vLLM / lmdeploy 直接跑 VoxCPM。

### 12.3 多线程与 CUDA Graphs

默认 `optimize=True` 使用 CUDA Graphs，**与多线程不兼容**。

| 场景 | 方案 |
|------|------|
| 后台线程推理 | `optimize=False` |
| 并发 HTTP 服务 | NanoVLLM-VoxCPM |
| Gradio | `queue(default_concurrency_limit=1)` |

本仓 Sidecar 使用 **`asyncio.Lock` 单请求互斥** + 默认 `optimize=False`（Windows 兼容）。

### 12.4 生态集成（参考）

ComfyUI-VoxCPM、TTS WebUI、voxcpm_rs 等见[官方生态页](https://voxcpm.readthedocs.io/zh-cn/latest/index.html)。

---

## 13. 故障排查 FAQ

### 13.1 Windows Triton 报错

**现象**：`Python int too large to convert to C long` 或 Triton 导入失败。

**处理**：

1. 安装 [triton-windows](https://github.com/woct0rdho/triton-windows)（版本须匹配 PyTorch）
2. 或 `optimize=False`
3. 特定 DLL 问题见 PyTorch issue 补丁方案

| PyTorch | Triton |
|---------|--------|
| 2.4 / 2.5 | 3.1 |
| 2.6 | 3.2 |
| 2.7 | 3.3 |
| 2.8 | 3.4 |

### 13.2 torchcodec / libtorchcodec

**现象**：克隆时报 `Could not load libtorchcodec`。

**处理**：

1. 系统安装 FFmpeg（4–7）并加入 PATH
2. `pip install torchcodec`
3. 或强制 soundfile 后端：`torchaudio.set_audio_backend("soundfile")`

### 13.3 首次运行 torch.compile 报错

**现象**：`torch._dynamo.exc.Unsupported`（einops 等）。

**处理**：`optimize=False`；或 PyTorch 2.5.1+ / Triton 3.1+ / einops 0.8.1 / Python 3.10–3.11。

### 13.4 Mac / MPS

- MPS 可用但不保证所有路径成功；报错时 `device="cpu", optimize=False`
- 降噪器在 MPS 上仍跑 CPU；不需要时可 `load_denoiser=False`

### 13.5 Python 版本

- 推荐 **3.10–3.12**
- 3.14+ 可能安装失败
- `No module named 'pkg_resources'` → `pip install setuptools`

### 13.6 WSL2 / ROCm

社区 workaround：monkey-patch torchcodec I/O；`optimize=False`；必要时先 CPU 验证。

---

## 14. 本仓 Sidecar 集成契约

> 产品链 SSOT：**Rust adapter** → HTTP Sidecar → `VoxCPM.generate()`。禁止 TS/Rust 双写厂商参数映射。

### 14.1 源码与运行时

| 路径 | 说明 |
|------|------|
| `runtimes/tts/voxcpm/sidecar_server.py` | FastAPI HTTP 服务 |
| `runtimes/tts/voxcpm/start-sidecar.ps1` | Windows 启动脚本 |
| `runtimes/tts/voxcpm/manifest.json` | CDN / 三级下载 manifest |
| `runtimes/tts/voxcpm/checkpoints/` | 本地 HF 权重目录 |

### 14.2 HTTP 端点

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/health` | `{ status, modelLoaded }` |
| GET | `/v1/models` | OpenAI 兼容模型列表 |
| POST | `/v1/audio/speech` | 合成，返回 `audio/wav` |

监听 **`127.0.0.1`**；端口环境变量 **`BGX_TTS_SIDECAR_PORT`**（默认 18100）。

### 14.3 请求体（`SpeechRequest`）

```json
{
  "model": "openbmb/VoxCPM2",
  "input": "待合成正文",
  "mode": "plain",
  "voiceDesign": { "controlPrompt": "warm female voice" },
  "reference": {
    "audioPath": "C:\\ref\\speaker.wav",
    "promptText": "可选，Hi-Fi 转写",
    "styleControl": "可选，克隆+风格"
  },
  "inference": {
    "cfgValue": 2.0,
    "inferenceTimesteps": 10,
    "denoiseReference": false
  }
}
```

| `mode` | 约束 |
|--------|------|
| `plain` | 仅 `input` |
| `voice_design` | 可选 `voiceDesign.controlPrompt` → 映射为 `(control)input`；**不得**带 `reference` |
| `reference_clone` | 必填 `reference.audioPath`；有 `promptText` 时走 Hi-Fi；**不得**带 `voiceDesign` |

### 14.4 映射到 `generate()`（Sidecar SSOT）

| Sidecar 字段 | VoxCPM 参数 |
|--------------|-------------|
| `input` | `text` |
| `voiceDesign.controlPrompt` | 前缀 `(control)` 拼入 `text` |
| `reference.audioPath`（无 promptText） | `reference_wav_path` |
| `reference.audioPath` + `promptText` | `prompt_wav_path` + `prompt_text` + `reference_wav_path` |
| `reference.styleControl` | 前缀 `(style)` 拼入 `text` |
| `inference.cfgValue` | `cfg_value` |
| `inference.inferenceTimesteps` | `inference_timesteps` |
| `inference.denoiseReference` | `denoise` |

### 14.5 并发与约束

- 单进程 **`asyncio.Lock`**：同时仅 1 个合成；忙时 **503** + `Retry-After: 5`
- **`input` 禁止 truncate**（INV-PROMPT-USER-TEXT 同源原则）
- 合成日志须记录完整 `input` 正文（排障）
- 参考音路径由 Rust **`resolve_tts_reference_audio_path`** 校验（防穿越）

### 14.6 开发安装与启动

```powershell
cd L:\00Dev\bgxiong-ai-story
.\scripts\dev-install-voxcpm-runtime.ps1
# 产物：data\runtimes\tts\voxcpm\{version}\

cd data\runtimes\tts\voxcpm\{version}
powershell -ExecutionPolicy Bypass -File start-sidecar.ps1 --port 18100
```

Bootstrap 进度：`sidecar.bootstrap.jsonl`  
首次启动可能较久（pip + 权重）；manifest `bootstrapTimeoutSec`: 1800。

### 14.7 相关 PLAN / 门禁

| 文档 / 脚本 | 用途 |
|-------------|------|
| `plans/20260628T140000+0800-PLAN-VoxCPM2全能力Sidecar-CDN与Rich本地TTS-架构审核版.md` | 母 PLAN |
| `plans/20260607T120000+0800-PLAN-本地TTS引擎框架整合-VoxCPM-IndexTTS-可扩展架构.md` | 引擎框架 |
| `SPEC-站外资源三级下载-RemoteArtifact.md` | CDN 下载契约 |
| `check-voxcpm-runtime-ssot.ps1` | 运行时门禁 |
| `check-tts-engine-isolation-ssot.ps1` | 引擎隔离 |

---

## 15. 产品开发速查表

### 15.1 场景 → 推荐参数

| 场景 | mode / API | cfg | timesteps | 备注 |
|------|-----------|-----|-----------|------|
| 旁白默认 | plain | 2.0 | 10 | 无参考音每次音色随机 |
| 固定角色 | reference_clone | 2.0 | 10 | 5–30 s 干净参考音 |
| 最高相似度 | reference_clone + promptText | 2.0 | 10 | ASR 转写对齐 |
| 虚拟音色 | voice_design | 2.0 | 10 | control 写身份+质感+场景 |
| 方言 | voice_design 或 plain | 1.8–2.0 | 10 | **正文用方言** |
| 长对白 | 分段 plain/clone | 1.5–1.6 | 10 | 段间 `np.concatenate` |
| 数字密集 | 任意 | 2.0 | 10 | `normalize=True` |
| 嘈杂参考音 | reference_clone | 2.0 | 10 | `denoiseReference=true` |

### 15.2 红线（本仓）

1. **禁止** clone 失败静默降级 plain  
2. **禁止** truncate 用户 `input` 正文  
3. **禁止** sidecar 监听 `0.0.0.0`  
4. **禁止** TS 与 Rust 各写一套 VoxCPM 参数映射  
5. 参考音须 Rust 侧路径校验 + 大小/时长校验  

### 15.3 与 IndexTTS-2 选型对照

| 维度 | VoxCPM 2 | IndexTTS-2 |
|------|----------|------------|
| 声音设定 | 括号 control 指令 | 内置默认说话人 + 情感 |
| 克隆 | reference-only / Hi-Fi | speaker audioPath |
| 情感 | 文本 control + 非语言标签 | emotionControl（text/vector/audio） |
| 显存 | ~8 GB | ~4 GB |
| 流式 | API 有，Sidecar 未交付 | 同上 |

两者均为本地 Sidecar；产品 DTO 统一 **`SpeechSynthesisRequestV1`**，引擎差异仅在 Rust adapter。

---

## 附录 A · 官方链接索引

| 主题 | URL |
|------|-----|
| 文档首页 | https://voxcpm.readthedocs.io/zh-cn/latest/ |
| GitHub | https://github.com/OpenBMB/VoxCPM |
| HF 权重 | https://huggingface.co/openbmb/VoxCPM2 |
| NanoVLLM 部署 | https://voxcpm.readthedocs.io/zh-cn/latest/deployment/nanovllm_voxcpm.html |
| 模型架构 | https://voxcpm.readthedocs.io/zh-cn/latest/models/architecture.html |
| 版本历史 | https://voxcpm.readthedocs.io/zh-cn/latest/models/version_history.html |

---

**维护**：官方 API 或本仓 Sidecar 契约变更时，同步更新本 GUIDE 与 INDEX「本地 TTS」行；重大版本 bump 须新 PLAN + manifest 版本号。
