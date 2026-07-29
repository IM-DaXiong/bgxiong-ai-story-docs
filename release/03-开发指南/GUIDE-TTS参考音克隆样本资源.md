**文档时间**：2026-07-09T14:45:00+0800

# GUIDE · TTS 参考音克隆样本资源（VoxCPM2 / IndexTTS-2）

> 本仓 Voice Tab 本地 TTS 克隆用的**中文参考音频**获取与导入指南。  
> 自动化脚本：`scripts/download-tts-reference-samples.ps1` · `scripts/import-tts-reference-samples.ps1`  
> 输出目录：`data/tts-reference-samples/`（gitignore，本机生成）

---

## 1. 本仓引擎约束

两套引擎共用 [`src-tauri/src/speech/tts_reference_prepare.rs`](../src-tauri/src/speech/tts_reference_prepare.rs)：

| 约束 | 值 |
|------|-----|
| 输出 | mono WAV **22050 Hz** |
| 时长 | **0.5–60 s**（推荐 5–15 s；超长自动裁切） |
| VoxCPM2 | 参考音即可克隆；可选 `promptText` Hi-Fi |
| IndexTTS-2 | **仅参考音**；不支持 `promptText` |

---

## 2. 一键下载（推荐）

```powershell
.\scripts\download-tts-reference-samples.ps1
.\scripts\import-tts-reference-samples.ps1
# 或复制到应用 refs 目录：
.\scripts\import-tts-reference-samples.ps1 -CopyToAppData
```

生成 `data/tts-reference-samples/manifest.json` 索引全部样本。

---

## 3. 资源清单

### 3.1 第一梯队（最快）

| 资源 | 下载 | 男女声 | 说明 |
|------|------|--------|------|
| **IndexTTS examples** | [GitHub examples/](https://github.com/index-tts/index-tts/tree/main/examples) · [Google Drive 打包](https://drive.google.com/file/d/1o_dCMzwjaA2azbGOxAE7-4E7NbJkgdgO/view) | 混合 | `voice_01`–`12`，脚本自动拉取 |
| **本仓 default_spk_zh** | [CDN launcher](https://r.bgxiong.com/client/ai-story/runtimes/indextts-windows-x64-2.1.5.zip) | 1 女声 | 安装 IndexTTS runtime 后位于 `assets/default_spk_zh.wav` |
| **zhvoice sample** | 百度网盘 https://pan.baidu.com/s/1uHXE2WIt0kdm_dPSej-TtA 提取码 **`i5b3`** | ~3249 人 | 解压 `sample/` + `metadata/` 到 `data/tts-reference-samples/zhvoice/` 后重跑下载脚本 |

### 3.2 第二梯队（高质量 / 批量）

| 资源 | 下载 | 规模 |
|------|------|------|
| **AISHELL-3** | [OpenSLR #93](https://www.openslr.org/93/) · [国内镜像](https://openslr.magicdatatech.com/resources/93/data_aishell3.tgz) | 218 人（43 男 / 175 女），`spk_info.txt` 含 gender |
| **YodasSpeakerPool** | [HF](https://huggingface.co/datasets/fangningshao/YodasSpeakerPool) · hf-mirror | ~3400 中文，4–15 s，脚本自动筛选 |
| **MagicData SLR68** | [openslr.org/68](https://www.openslr.org/68/) | 1080 人，755 h（CC BY-NC-ND，非商用） |
| **ST-CMDS SLR38** | [openslr.org/38](https://www.openslr.org/38/) | 855 人，100 h |

### 3.3 第三梯队

| 资源 | 下载 |
|------|------|
| **DiDiSpeech** | https://outreach.didichuxing.com/research/opendata/ （6000 人，需申请） |
| **标贝 BZNSYP** | https://www.data-baker.com/open_source.html （1 女声，已含于 zhvoice） |
| **AI柠檬网盘汇总** | https://wiki.ailemon.net/docs/asrt-doc/asrt-doc-1deoef82nv83e （THCHS30 `5szx`、AISHELL-1 `q05t` 等提取码） |

---

## 4. 导入 Voice Tab

1. 运行 `import-tts-reference-samples.ps1` 得到 `prepared/*.wav`（22050 Hz mono）
2. 打开应用 **Voice Tab** → 上传参考音 → 选择 `prepared/` 下文件
3. **VoxCPM2**：可选将 manifest 中 `promptText` 填入续写文稿（Hi-Fi）
4. **IndexTTS-2**：仅选参考音即可合成

---

## 5. License

| 资源 | 商用 |
|------|------|
| AISHELL-3 | Apache 2.0 |
| MagicData / 标贝 | **禁止商用** |
| zhvoice | 各子集原 license |
| IndexTTS examples | 随仓库，研究/测试 |

商用配音前务必核对数据集 license。

---

## 6. 相关文档

- VoxCPM2：[`GUIDE-VoxCPM2-开发者手册.md`](GUIDE-VoxCPM2-开发者手册.md) §8
- IndexTTS2：[`GUIDE-IndexTTS2-Sidecar-HTTP外用.md`](GUIDE-IndexTTS2-Sidecar-HTTP外用.md)
- 参考音管线 PLAN：`plans/20260704T160000+0800-PLAN-VoiceTab参考音续写文稿统一管线-架构审核版.md`
