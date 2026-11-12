> **历史演进产物，不代表当前设计，不作为当前实现或开发依据。**
>
> 本目录集中存放从现行发布区移出的历史文档。这里的每一篇都是历史演进资料，不代表当前设计，不能作为当前实现或开发依据。
>
> 现行架构请阅读仓库主导航中的现行文档，而不是本目录。

# 历史演进资料

本目录在 2026-11-12（Asia/Shanghai）的文档整理中设立。当时按源仓库 `main` 短 SHA `615a8d70e`、应用版本 v0.6.1113 核实现行架构后，将非现行架构正文从 `release/` 移入此处，并保留原相对路径，便于对照演进过程。

**本次共归档 79 篇。** 没有删除这些历史文档。每篇顶部都有与上文相同的声明，并记录原路径；若原文件名含有日期或版本标识，只作为文件名中的标识抄录，不另行推定写作或生效日期。

## 如何阅读

- 现行架构只以仓库主导航列出的文档为准，权威稿是 `release/02-架构设计/20261104-ARCH-全仓整体架构设计.md`。
- 本目录中既有明确过时的早期综述、故事板 / 场次 / 旧片段方案，也有仍可能谈到工程约定的专题指南或规范。即便后者在技术上仍有参考价值，它们也**不是现行架构的单一来源**，不能代替主导航中的现行文档。
- 历史文档已经过脱敏；保留的是技术演进意义，而不是敏感实现。开源版同样不包含登录、认证、付费等实现细节。
- 文档内部的相对链接已按移动后的位置校正。指向现行架构文档的链接会回到 `release/`；指向同批归档文档的链接仍在本目录内解析。

## 归档清单

### `release/01-项目概览/`（9 篇）

| 文档 | 原路径 |
|------|--------|
| [ComfyUI与开源模型接入.md](./release/01-项目概览/ComfyUI与开源模型接入.md) | `release/01-项目概览/ComfyUI与开源模型接入.md` |
| [vs-SaaS对比分析.md](./release/01-项目概览/vs-SaaS对比分析.md) | `release/01-项目概览/vs-SaaS对比分析.md` |
| [产品介绍.md](./release/01-项目概览/产品介绍.md) | `release/01-项目概览/产品介绍.md` |
| [目录结构与模块说明-v0.3.801.md](./release/01-项目概览/目录结构与模块说明-v0.3.801.md) | `release/01-项目概览/目录结构与模块说明-v0.3.801.md` |
| [通用模块架构总览.md](./release/01-项目概览/通用模块架构总览.md) | `release/01-项目概览/通用模块架构总览.md` |
| [项目架构总览.md](./release/01-项目概览/项目架构总览.md) | `release/01-项目概览/项目架构总览.md` |
| [项目概要说明.md](./release/01-项目概览/项目概要说明.md) | `release/01-项目概览/项目概要说明.md` |
| [项目综述-v0.3.801.md](./release/01-项目概览/项目综述-v0.3.801.md) | `release/01-项目概览/项目综述-v0.3.801.md` |
| [项目综述-早期版.md](./release/01-项目概览/项目综述-早期版.md) | `release/01-项目概览/项目综述-早期版.md` |

### `release/02-架构设计/`（10 篇）

| 文档 | 原路径 |
|------|--------|
| [20260410-ARCH-故事板设计文档.md](./release/02-架构设计/20260410-ARCH-故事板设计文档.md) | `release/02-架构设计/20260410-ARCH-故事板设计文档.md` |
| [20260410-ARCH-无限画布瀑布流性能优化设计与排列算法.md](./release/02-架构设计/20260410-ARCH-无限画布瀑布流性能优化设计与排列算法.md) | `release/02-架构设计/20260410-ARCH-无限画布瀑布流性能优化设计与排列算法.md` |
| [20260410-ARCH-视频片段生成架构指南.md](./release/02-架构设计/20260410-ARCH-视频片段生成架构指南.md) | `release/02-架构设计/20260410-ARCH-视频片段生成架构指南.md` |
| [20260410-ARCH-高性能量化画布模块设计.md](./release/02-架构设计/20260410-ARCH-高性能量化画布模块设计.md) | `release/02-架构设计/20260410-ARCH-高性能量化画布模块设计.md` |
| [20260416-ARCH-ComfyUI集成方案.md](./release/02-架构设计/20260416-ARCH-ComfyUI集成方案.md) | `release/02-架构设计/20260416-ARCH-ComfyUI集成方案.md` |
| [20260420-ARCH-片段画布撤销重做系统设计文档.md](./release/02-架构设计/20260420-ARCH-片段画布撤销重做系统设计文档.md) | `release/02-架构设计/20260420-ARCH-片段画布撤销重做系统设计文档.md` |
| [20260503-ARCH-ComfyUI三层分离最优架构方案.md](./release/02-架构设计/20260503-ARCH-ComfyUI三层分离最优架构方案.md) | `release/02-架构设计/20260503-ARCH-ComfyUI三层分离最优架构方案.md` |
| [外部依赖说明书-202607.md](./release/02-架构设计/外部依赖说明书-202607.md) | `release/02-架构设计/外部依赖说明书-202607.md` |
| [架构设计文档.md](./release/02-架构设计/架构设计文档.md) | `release/02-架构设计/架构设计文档.md` |
| [语音波形播放器方案.md](./release/02-架构设计/语音波形播放器方案.md) | `release/02-架构设计/语音波形播放器方案.md` |

### `release/03-开发指南/`（21 篇）

| 文档 | 原路径 |
|------|--------|
| [20260421-GUIDE-排障与日志指南.md](./release/03-开发指南/20260421-GUIDE-排障与日志指南.md) | `release/03-开发指南/20260421-GUIDE-排障与日志指南.md` |
| [20260517T250000+0800-GUIDE-角色叙事外形与生图统一设计理念.md](./release/03-开发指南/20260517T250000+0800-GUIDE-角色叙事外形与生图统一设计理念.md) | `release/03-开发指南/20260517T250000+0800-GUIDE-角色叙事外形与生图统一设计理念.md` |
| [20260521T180000+0800-GUIDE-布局持久化跨重启排障与开发约束.md](./release/03-开发指南/20260521T180000+0800-GUIDE-布局持久化跨重启排障与开发约束.md) | `release/03-开发指南/20260521T180000+0800-GUIDE-布局持久化跨重启排障与开发约束.md` |
| [20260528T120000+0800-GUIDE-故事板双栏画布布局与排障经验.md](./release/03-开发指南/20260528T120000+0800-GUIDE-故事板双栏画布布局与排障经验.md) | `release/03-开发指南/20260528T120000+0800-GUIDE-故事板双栏画布布局与排障经验.md` |
| [20260716-8G显存可跑无审查GGUF热门模型精选.md](./release/03-开发指南/20260716-8G显存可跑无审查GGUF热门模型精选.md) | `release/03-开发指南/20260716-8G显存可跑无审查GGUF热门模型精选.md` |
| [20260729-GUIDE-本地音乐音效模型下载清单与地址.md](./release/03-开发指南/20260729-GUIDE-本地音乐音效模型下载清单与地址.md) | `release/03-开发指南/20260729-GUIDE-本地音乐音效模型下载清单与地址.md` |
| [20260816T114300+0800-GUIDE-生图生视频随机种子与可复现性.md](./release/03-开发指南/20260816T114300+0800-GUIDE-生图生视频随机种子与可复现性.md) | `release/03-开发指南/20260816T114300+0800-GUIDE-生图生视频随机种子与可复现性.md` |
| [20261001-GUIDE-Rust开发编译与验收.md](./release/03-开发指南/20261001-GUIDE-Rust开发编译与验收.md) | `release/03-开发指南/20261001-GUIDE-Rust开发编译与验收.md` |
| [20261002-GUIDE-Agent-Skills导入创作与开发.md](./release/03-开发指南/20261002-GUIDE-Agent-Skills导入创作与开发.md) | `release/03-开发指南/20261002-GUIDE-Agent-Skills导入创作与开发.md` |
| [20261004T060000+0800-GUIDE-ComfyUI工作流映射合同-使用迁移与扩展.md](./release/03-开发指南/20261004T060000+0800-GUIDE-ComfyUI工作流映射合同-使用迁移与扩展.md) | `release/03-开发指南/20261004T060000+0800-GUIDE-ComfyUI工作流映射合同-使用迁移与扩展.md` |
| [20261015-GUIDE-ASR模型模块与接手验收.md](./release/03-开发指南/20261015-GUIDE-ASR模型模块与接手验收.md) | `release/03-开发指南/20261015-GUIDE-ASR模型模块与接手验收.md` |
| [20261026-GUIDE-应用版本号月日递增规范.md](./release/03-开发指南/20261026-GUIDE-应用版本号月日递增规范.md) | `release/03-开发指南/20261026-GUIDE-应用版本号月日递增规范.md` |
| [20261026-GUIDE-自动保存与输入生命周期约束.md](./release/03-开发指南/20261026-GUIDE-自动保存与输入生命周期约束.md) | `release/03-开发指南/20261026-GUIDE-自动保存与输入生命周期约束.md` |
| [20261028-GUIDE-数据库短事务与文件先落盘规范.md](./release/03-开发指南/20261028-GUIDE-数据库短事务与文件先落盘规范.md) | `release/03-开发指南/20261028-GUIDE-数据库短事务与文件先落盘规范.md` |
| [20261101-GUIDE-撤销窗口与会话生命周期.md](./release/03-开发指南/20261101-GUIDE-撤销窗口与会话生命周期.md) | `release/03-开发指南/20261101-GUIDE-撤销窗口与会话生命周期.md` |
| [GUIDE-IndexTTS2-Sidecar-HTTP外用.md](./release/03-开发指南/GUIDE-IndexTTS2-Sidecar-HTTP外用.md) | `release/03-开发指南/GUIDE-IndexTTS2-Sidecar-HTTP外用.md` |
| [GUIDE-TTS参考音克隆样本资源.md](./release/03-开发指南/GUIDE-TTS参考音克隆样本资源.md) | `release/03-开发指南/GUIDE-TTS参考音克隆样本资源.md` |
| [GUIDE-VendorAdapter-开发者手册.md](./release/03-开发指南/GUIDE-VendorAdapter-开发者手册.md) | `release/03-开发指南/GUIDE-VendorAdapter-开发者手册.md` |
| [GUIDE-VoxCPM2-开发者手册.md](./release/03-开发指南/GUIDE-VoxCPM2-开发者手册.md) | `release/03-开发指南/GUIDE-VoxCPM2-开发者手册.md` |
| [GUIDE-持久化层六边形架构.md](./release/03-开发指南/GUIDE-持久化层六边形架构.md) | `release/03-开发指南/GUIDE-持久化层六边形架构.md` |
| [分镜与片段全流程.md](./release/03-开发指南/分镜与片段全流程.md) | `release/03-开发指南/分镜与片段全流程.md` |

### `release/04-开发规范/`（12 篇）

| 文档 | 原路径 |
|------|--------|
| [20260509-SPEC-全仓语义命名权威规范.md](./release/04-开发规范/20260509-SPEC-全仓语义命名权威规范.md) | `release/04-开发规范/20260509-SPEC-全仓语义命名权威规范.md` |
| [20260510-SPEC-AI任务作用域UI锁定与任务并发模型.md](./release/04-开发规范/20260510-SPEC-AI任务作用域UI锁定与任务并发模型.md) | `release/04-开发规范/20260510-SPEC-AI任务作用域UI锁定与任务并发模型.md` |
| [20260511-SPEC-图片元数据模块设计.md](./release/04-开发规范/20260511-SPEC-图片元数据模块设计.md) | `release/04-开发规范/20260511-SPEC-图片元数据模块设计.md` |
| [20260511-SPEC-提示词合法性校验与压缩模块.md](./release/04-开发规范/20260511-SPEC-提示词合法性校验与压缩模块.md) | `release/04-开发规范/20260511-SPEC-提示词合法性校验与压缩模块.md` |
| [20260528T210000+0800-SPEC-模型接入分层规范-v1终稿.md](./release/04-开发规范/20260528T210000+0800-SPEC-模型接入分层规范-v1终稿.md) | `release/04-开发规范/20260528T210000+0800-SPEC-模型接入分层规范-v1终稿.md` |
| [20260530T200000+0800-SPEC-EntityHostAiTask全仓AI任务态通用架构-v1终稿.md](./release/04-开发规范/20260530T200000+0800-SPEC-EntityHostAiTask全仓AI任务态通用架构-v1终稿.md) | `release/04-开发规范/20260530T200000+0800-SPEC-EntityHostAiTask全仓AI任务态通用架构-v1终稿.md` |
| [20261002-SPEC-Agent-Skills包加载与执行合同.md](./release/04-开发规范/20261002-SPEC-Agent-Skills包加载与执行合同.md) | `release/04-开发规范/20261002-SPEC-Agent-Skills包加载与执行合同.md` |
| [20261003-SPEC-ComfyUI工作流映射合同.md](./release/04-开发规范/20261003-SPEC-ComfyUI工作流映射合同.md) | `release/04-开发规范/20261003-SPEC-ComfyUI工作流映射合同.md` |
| [20261015-SPEC-语音转文字模型无关接口与选择合同.md](./release/04-开发规范/20261015-SPEC-语音转文字模型无关接口与选择合同.md) | `release/04-开发规范/20261015-SPEC-语音转文字模型无关接口与选择合同.md` |
| [20261019-SPEC-统一开放适配标准与SDK边界.md](./release/04-开发规范/20261019-SPEC-统一开放适配标准与SDK边界.md) | `release/04-开发规范/20261019-SPEC-统一开放适配标准与SDK边界.md` |
| [20261026-SPEC-TTS三层对外合同与SDK接入要求.md](./release/04-开发规范/20261026-SPEC-TTS三层对外合同与SDK接入要求.md) | `release/04-开发规范/20261026-SPEC-TTS三层对外合同与SDK接入要求.md` |
| [SPEC-站外资源三级下载-RemoteArtifact.md](./release/04-开发规范/SPEC-站外资源三级下载-RemoteArtifact.md) | `release/04-开发规范/SPEC-站外资源三级下载-RemoteArtifact.md` |

### `release/05-提示词工程/`（1 篇）

| 文档 | 原路径 |
|------|--------|
| [README.md](./release/05-提示词工程/README.md) | `release/05-提示词工程/README.md` |

### `release/05-提示词工程/chinese/`（2 篇）

| 文档 | 原路径 |
|------|--------|
| [README.md](./release/05-提示词工程/chinese/README.md) | `release/05-提示词工程/chinese/README.md` |
| [awesome-chatgpt-prompts-zh.md](./release/05-提示词工程/chinese/awesome-chatgpt-prompts-zh.md) | `release/05-提示词工程/chinese/awesome-chatgpt-prompts-zh.md` |

### `release/05-提示词工程/general/`（4 篇）

| 文档 | 原路径 |
|------|--------|
| [README.md](./release/05-提示词工程/general/README.md) | `release/05-提示词工程/general/README.md` |
| [awesome-chatgpt-prompts.md](./release/05-提示词工程/general/awesome-chatgpt-prompts.md) | `release/05-提示词工程/general/awesome-chatgpt-prompts.md` |
| [awesome-claude-prompts.md](./release/05-提示词工程/general/awesome-claude-prompts.md) | `release/05-提示词工程/general/awesome-claude-prompts.md` |
| [chatgpt-prompts-yokoffing.md](./release/05-提示词工程/general/chatgpt-prompts-yokoffing.md) | `release/05-提示词工程/general/chatgpt-prompts-yokoffing.md` |

### `release/05-提示词工程/image-generation/`（5 篇）

| 文档 | 原路径 |
|------|--------|
| [README.md](./release/05-提示词工程/image-generation/README.md) | `release/05-提示词工程/image-generation/README.md` |
| [gpt4o-image-prompts.md](./release/05-提示词工程/image-generation/gpt4o-image-prompts.md) | `release/05-提示词工程/image-generation/gpt4o-image-prompts.md` |
| [midjourney-prompts.md](./release/05-提示词工程/image-generation/midjourney-prompts.md) | `release/05-提示词工程/image-generation/midjourney-prompts.md` |
| [stable-diffusion-prompts.md](./release/05-提示词工程/image-generation/stable-diffusion-prompts.md) | `release/05-提示词工程/image-generation/stable-diffusion-prompts.md` |
| [storyboard-and-reference-sheet.md](./release/05-提示词工程/image-generation/storyboard-and-reference-sheet.md) | `release/05-提示词工程/image-generation/storyboard-and-reference-sheet.md` |

### `release/05-提示词工程/prompt-engineering/`（2 篇）

| 文档 | 原路径 |
|------|--------|
| [README.md](./release/05-提示词工程/prompt-engineering/README.md) | `release/05-提示词工程/prompt-engineering/README.md` |
| [prompt-engineering-guide.md](./release/05-提示词工程/prompt-engineering/prompt-engineering-guide.md) | `release/05-提示词工程/prompt-engineering/prompt-engineering-guide.md` |

### `release/05-提示词工程/system-prompts/`（4 篇）

| 文档 | 原路径 |
|------|--------|
| [README.md](./release/05-提示词工程/system-prompts/README.md) | `release/05-提示词工程/system-prompts/README.md` |
| [chatgpt-system-prompt.md](./release/05-提示词工程/system-prompts/chatgpt-system-prompt.md) | `release/05-提示词工程/system-prompts/chatgpt-system-prompt.md` |
| [claude-code-system-prompt.md](./release/05-提示词工程/system-prompts/claude-code-system-prompt.md) | `release/05-提示词工程/system-prompts/claude-code-system-prompt.md` |
| [system-prompts-leaks.md](./release/05-提示词工程/system-prompts/system-prompts-leaks.md) | `release/05-提示词工程/system-prompts/system-prompts-leaks.md` |

### `release/06-功能演进/`（6 篇）

| 文档 | 原路径 |
|------|--------|
| [20260603-FEAT-两天升级内容汇总.md](./release/06-功能演进/20260603-FEAT-两天升级内容汇总.md) | `release/06-功能演进/20260603-FEAT-两天升级内容汇总.md` |
| [20260612-公众号推文-0.2重大版本升级.md](./release/06-功能演进/20260612-公众号推文-0.2重大版本升级.md) | `release/06-功能演进/20260612-公众号推文-0.2重大版本升级.md` |
| [20260613-FEAT-近期更新汇总.md](./release/06-功能演进/20260613-FEAT-近期更新汇总.md) | `release/06-功能演进/20260613-FEAT-近期更新汇总.md` |
| [20260626-公众号推文-达芬奇插件互通自动剪辑.md](./release/06-功能演进/20260626-公众号推文-达芬奇插件互通自动剪辑.md) | `release/06-功能演进/20260626-公众号推文-达芬奇插件互通自动剪辑.md` |
| [公众号推文-干中学-提示词与分镜.md](./release/06-功能演进/公众号推文-干中学-提示词与分镜.md) | `release/06-功能演进/公众号推文-干中学-提示词与分镜.md` |
| [公众号推文-比格熊数字导演工作站.md](./release/06-功能演进/公众号推文-比格熊数字导演工作站.md) | `release/06-功能演进/公众号推文-比格熊数字导演工作站.md` |

### `release/07-代码质量/`（3 篇）

| 文档 | 原路径 |
|------|--------|
| [BUG审查-story-backup模块.md](./release/07-代码质量/BUG审查-story-backup模块.md) | `release/07-代码质量/BUG审查-story-backup模块.md` |
| [屎山代码鉴定准则.md](./release/07-代码质量/屎山代码鉴定准则.md) | `release/07-代码质量/屎山代码鉴定准则.md` |
| [屎山审计报告-story-backup模块.md](./release/07-代码质量/屎山审计报告-story-backup模块.md) | `release/07-代码质量/屎山审计报告-story-backup模块.md` |

---

返回：[仓库首页](../../README.md) · [现行文档目录](../../release/README.md)
