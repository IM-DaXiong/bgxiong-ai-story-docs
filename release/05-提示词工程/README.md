# 提示词库索引

> 整理自 GitHub 高星仓库，覆盖通用、系统、图像生成、提示工程、中文等 7 大类，共 30+ 个优质资源。

## 目录结构

```
prompts/
├── README.md              ← 本文件（总览）
├── general/               ← 通用提示词（写作、编程、商业、创意）
├── system-prompts/        ← 系统提示词（AI 工具、GPTs）
├── image-generation/      ← 图像生成提示词（GPT-4o、Midjourney、SD）
├── prompt-engineering/    ← 提示工程指南与框架
└── chinese/               ← 中文提示词专区
```

## 快速索引（按星标排序 Top 10）

| 排名 | 仓库 | 星标 | 分类 | 说明 |
|------|------|------|------|------|
| 1 | [f/awesome-chatgpt-prompts](https://github.com/f/awesome-chatgpt-prompts) | 163k | 通用 | 全球最大开源提示词库，支持多模型 |
| 2 | [x1xhlol/system-prompts-and-models-of-ai-tools](https://github.com/x1xhlol/system-prompts-and-models-of-ai-tools) | 139k | 系统提示词 | 33+ AI 工具完整 system prompt |
| 3 | [microsoft/generative-ai-for-beginners](https://github.com/microsoft/generative-ai-for-beginners) | 112k | 提示工程 | 微软 21 课生成式 AI 教程 |
| 4 | [dair-ai/Prompt-Engineering-Guide](https://github.com/dair-ai/Prompt-Engineering-Guide) | 75.3k | 提示工程 | 最全面的提示工程指南 |
| 5 | [PlexPt/awesome-chatgpt-prompts-zh](https://github.com/PlexPt/awesome-chatgpt-prompts-zh) | 60.6k | 中文 | 中文 ChatGPT 提示词指南 |
| 6 | [asgeirtj/system_prompts_leaks](https://github.com/asgeirtj/system_prompts_leaks) | 41.2k | 系统提示词 | Anthropic/OpenAI/Google 等泄露提示词 |
| 7 | [linexjlin/GPTs](https://github.com/linexjlin/GPTs) | 32k | 系统提示词 | OpenAI 自定义 GPTs 系统提示词 |
| 8 | [linshenkx/prompt-optimizer](https://github.com/linshenkx/prompt-optimizer) | 30.4k | 中文 | AI 提示词优化工具 |
| 9 | [langfuse/langfuse](https://github.com/langfuse/langfuse) | 28.5k | 工具 | 开源 LLM 工程平台 |
| 10 | [elder-plinius/CL4R1T4S](https://github.com/elder-plinius/CL4R1T4S) | 26.4k | 系统提示词 | 25 家 AI 提供商系统提示词 |

## 使用建议

1. **快速查找**：直接访问各子目录的 README.md
2. **深度使用**：clone 对应仓库到本地，很多提供 CSV/JSON 格式方便程序化使用
3. **学习提升**：先看 `prompt-engineering/` 目录的指南，再参考具体提示词

## 比格熊项目映射（SSOT 权威在 `项目文档/`）

本目录为 **外部参考库**，不是项目运行时 SSOT。实现与维护 prompt 时以以下为准：

| docs 子目录 | 比格熊 SSOT 模块 | 权威文档 |
|-------------|------------------|----------|
| `prompt-engineering/` | `src-tauri/src/ai_prompts/` | [`项目文档/20260605T103000+0800-GUIDE-全仓提示词工程SSOT与维护规范.md`](../项目文档/20260605T103000+0800-GUIDE-全仓提示词工程SSOT与维护规范.md) |
| `image-generation/` | `src-tauri/src/image_generation/visual_style/` | 同上 · Wave 2 lexicon |
| `system-prompts/` | LLM system 六段式结构 | 同上 §5.2 |
| `general/` · `chinese/` | 按需参考 | — |

机器可读索引：`src-tauri/resources/prompt_ssot_index.json` · CI：`scripts/check-prompt-ssot-index.ps1`

升级 PLAN：`项目文档/20260604T253000+0800-PLAN-docs提示词库对标分析与项目提示词工程升级计划.md`

## 更新记录

- 2026-06-05：增加比格熊项目映射节 · 链到 GUIDE v1.0

- 2026-06-04：首次整理，30 个仓库，7 大分类
