# 故事板与多角度参考图提示词指南

> 整理日期：2026-06-04
> 说明：故事板生成、多角度参考图的提示词技巧，以及中国大陆 API 访问情况

---

## 一、故事板（Storyboard）提示词

### 1.1 基础结构

```
Create a [数量]-panel storyboard in a grid layout showing 
[角色描述] performing [动作/场景].

Each panel should show a different camera angle:
1. [镜头类型1]
2. [镜头类型2]
3. ...

Maintain consistent character design and art style across all panels.
```

### 1.2 常用镜头类型

| 中文 | 英文 | 说明 |
|------|------|------|
| 全景 | Wide establishing shot | 展示整体环境 |
| 中景 | Medium shot | 腰部以上 |
| 特写 | Close-up | 面部或细节 |
| 过肩镜头 | Over-the-shoulder shot | 从肩膀后方拍摄 |
| 仰拍 | Low angle looking up | 从下往上拍 |
| 俯瞰 | Bird's eye view | 从上往下拍 |
| 平视 | Eye level | 水平视角 |
| 荷兰角 | Dutch angle | 倾斜视角，制造紧张感 |

### 1.3 完整示例

#### 示例 1：漫画风格故事板

```
Create a 6-panel storyboard in manga style showing 
a young woman with black hair walking through a rainy city at night.

Panel layout (2x3 grid):
1. Wide shot: She walks alone on empty street, neon lights reflect on wet pavement
2. Medium shot: She holds umbrella, looking at phone
3. Close-up: Rain drops on her face, melancholic expression
4. Over-shoulder: What she sees - glowing shop signs
5. Low angle: Towering buildings frame her small figure
6. Bird's eye: She's the only person in the vast city grid

Style: manga, black and white with selective color (neon pink/blue)
Consistent character design, cinematic lighting
```

#### 示例 2：电影风格故事板

```
Create a 4-panel cinematic storyboard showing 
a detective investigating a crime scene.

Panel layout (2x2 grid):
1. Wide shot: Detective arrives at abandoned warehouse, police tape
2. Medium shot: Detective examines evidence with flashlight
3. Close-up: Detective's face, intense concentration
4. Over-shoulder: What detective sees - mysterious footprints

Style: film noir, dramatic lighting, high contrast
Black and white, professional storyboard quality
```

#### 示例 3：动画分镜

```
Create an 8-panel animation storyboard showing 
a cat chasing a butterfly in a garden.

Panel layout (2x4 grid):
1. Wide shot: Cat sleeping in sunny garden
2. Medium shot: Butterfly lands nearby, cat notices
3. Close-up: Cat's eyes widen with interest
4. Action shot: Cat pounces toward butterfly
5. Wide shot: Butterfly flies away, cat follows
6. Medium shot: Cat climbs tree after butterfly
7. Close-up: Cat's paw reaches for butterfly
8. Wide shot: Butterfly escapes, cat looks disappointed

Style: cute animation, soft colors, expressive characters
Consistent cat design throughout all panels
```

### 1.4 布局控制

```
# 2行3列
2x3 grid layout

# 3行2列
3x2 grid layout

# 横排
arranged in a row (4 panels in a horizontal strip)

# 竖排
arranged in a column (4 panels in a vertical strip)

# 自由布局
scattered layout, comic book style
```

### 1.5 保持一致性

```
# 关键短语
Same character throughout all panels
Consistent design, same outfit, same hairstyle
Maintain character proportions
Same art style across all panels
Unified lighting direction
Same line weight, same color palette
```

---

## 二、多角度参考图（Reference Sheet）

### 2.1 核心关键词

| 关键词 | 作用 |
|--------|------|
| `character sheet` | 角色设定图 |
| `reference sheet` | 参考图 |
| `turnaround sheet` | 转面图 |
| `multiple views` | 多角度 |
| `front view, side view, back view` | 前/侧/后视图 |
| `3/4 view` | 3/4 视角 |
| `orthographic view` | 正交视图 |
| `T-pose` / `A-pose` | 标准姿势 |
| `full body` | 全身 |
| `white background` | 白色背景 |
| `concept art` | 概念艺术 |

### 2.2 角色参考图模板

```
Character turnaround reference sheet, 
[角色描述],

Multiple views arranged in a grid:
- Front view (正面)
- Side view (侧面)  
- Back view (背面)
- 3/4 view (3/4 视角)
- Close-up face (面部特写)

Full body, T-pose, 
white background, 
consistent design across all views,
concept art style, professional, high detail
```

### 2.3 道具参考图模板

```
Prop reference sheet for [道具描述],

Multiple views arranged in a grid:
- Front view (正面)
- Side view (侧面)
- Back view (背面)
- 3/4 perspective (3/4 视角)
- Close-up of [细节1] (细节特写)
- Close-up of [细节2] (细节特写)

Clean white background, 
orthographic view, 
concept art style, 
highly detailed, professional game art
```

### 2.4 完整示例

#### 示例 1：角色设定图

```
Character turnaround reference sheet for a cyberpunk ninja,

Multiple views arranged in a grid:
- Front view, full body, T-pose
- Side view, full body, T-pose
- Back view, full body, T-pose
- 3/4 view, full body, action pose
- Close-up face with mask removed

Appearance: athletic build, black tactical suit with neon blue accents, 
glowing visor mask, utility belt, armored boots

White background, consistent design across all views,
cyberpunk anime style, professional concept art, highly detailed
```

#### 示例 2：武器道具图

```
Prop reference sheet for a futuristic energy sword,

Multiple views arranged in a grid:
- Front view (正面完整视图)
- Side view (侧面完整视图)
- 3/4 perspective (3/4 视角)
- Close-up of handle (把手细节)
- Close-up of energy blade (能量刃细节)
- Exploded view (分解图)

Design: sleek hilt with glowing energy blade, 
blue plasma core, chrome accents

White background, orthographic view,
sci-fi concept art style, highly detailed, professional game art
```

#### 示例 3：场景道具图

```
Prop reference sheet for a magical crystal staff,

Multiple views arranged in a grid:
- Front view (正面)
- Side view (侧面)
- Back view (背面)
- Close-up of crystal head (水晶头特写)
- Close-up of wood grain texture (木纹特写)
- Size comparison with human hand (尺寸对比)

Design: ancient gnarled wood staff topped with glowing purple crystal, 
runic engravings along shaft, leather grip wrapping

White background, fantasy concept art style,
highly detailed, professional illustration
```

### 2.5 负面提示词（Stable Diffusion）

```
# 排除不需要的元素
deformed, merged, overlapping, inconsistent style, blurry,
different faces, different outfits, different proportions,
low quality, worst quality, watermark, text
```

---

## 三、组合技巧：故事板 + 多角度

### 3.1 完整案例

```
Create a character design sheet for a cyberpunk ninja,

Top row (3 panels) - Character Turnaround:
1. Front view, full body, T-pose
2. Side view, full body, T-pose  
3. Back view, full body, T-pose

Bottom row (3 panels) - Action Storyboard:
4. Wide shot: Ninja leaps between neon-lit buildings
5. Medium shot: Ninja draws glowing sword
6. Close-up: Ninja's masked face, glowing eyes

Style: cyberpunk anime, dark atmosphere, neon accents
Consistent character design, professional concept art
White background for top row, dark city background for bottom row
```

### 3.2 道具使用场景图

```
Create a prop usage guide for a futuristic drone,

Top row (3 panels) - Drone Reference:
1. Front view, closed state
2. Side view, deployed state
3. Top view, showing sensor layout

Bottom row (4 panels) - Usage Scenarios:
4. Wide shot: Drone flying over city skyline
5. Medium shot: Drone scanning a building
6. Close-up: Drone's camera lens focusing
7. Action shot: Drone avoiding obstacles

Style: sci-fi technical illustration, clean design
White background for reference panels, 
scene backgrounds for usage panels
```

---

## 四、各平台支持情况

| 平台 | 故事板 | 多角度参考图 | 推荐设置 |
|------|--------|--------------|----------|
| **GPT-Image-2** | ✅ 优秀 | ✅ 优秀 | 默认即可 |
| **Midjourney v6** | ✅ 优秀 | ✅ 优秀 | `--style raw` 更写实 |
| **Midjourney niji 6** | ✅ 动漫风格 | ✅ 动漫风格 | 适合二次元 |
| **Stable Diffusion** | ✅ 需要 ControlNet | ✅ 需要精细提示词 | 控制力最强 |
| **Leonardo AI** | ✅ 内置故事功能 | ✅ 角色参考功能 | 适合游戏开发 |
| **通义万象** | ✅ 支持 | ✅ 支持 | 国内首选 |
| **CogView-4** | ✅ 支持 | ✅ 支持 | 中文理解强 |

---

## 五、中国大陆 API 访问情况

### 5.1 GPT-Image-2 API

| 方式 | 状态 | 说明 |
|------|------|------|
| OpenAI 直连 | ❌ 被封锁 | 2024年7月起封锁中国大陆 |
| VPN/代理 | ⚠️ 违反 ToS | 有封号风险 |
| Azure OpenAI（中国区） | ⏳ 未上线 | 世纪互联运营，图像模型尚未部署 |
| Azure OpenAI（Global） | ⚠️ 需合规评估 | 数据出境问题 |

### 5.2 国产替代方案

| 平台 | 模型 | API 地址 | 特点 |
|------|------|----------|------|
| **阿里云** | 通义万象 | [百炼平台](https://bailian.console.aliyun.com) | 文生图、图生图、编辑 |
| **智谱 AI** | CogView-4 | [bigmodel.cn](https://bigmodel.cn) | 中文理解强，持续更新 |
| **快手** | 可灵 (Kling) | [klingai.kuaishou.com](https://klingai.kuaishou.com) | 图像+视频生成 |
| **字节跳动** | 即梦 (Jimeng) | [jimeng.jianying.com](https://jimeng.jianying.com) | 抖音生态 |
| **百度** | 文心一格 | [yige.baidu.com](https://yige.baidu.com) | 文心大模型系列 |
| **腾讯** | 混元 | [腾讯云](https://cloud.tencent.com) | 腾讯云集成 |

### 5.3 建议方案

#### 方案 1：主力国产 API（推荐）

```
优点：
- 合规，无封号风险
- 国内访问速度快
- 中文理解好
- 成本可控

缺点：
- 模型能力可能略逊于 GPT-Image-2
- 部分功能可能需要等待更新
```

#### 方案 2：Azure OpenAI Global

```
优点：
- 使用 OpenAI 最新模型
- 企业级稳定性

缺点：
- 需要海外 Azure 账号
- 数据出境合规问题
- 成本较高
```

#### 方案 3：API 中转服务

```
优点：
- 无需自己处理网络问题
- 可能提供多个模型选择

缺点：
- 稳定性和合规性存疑
- 可能有额外费用
- 服务质量参差不齐
```

### 5.4 项目建议

对于比格熊 AI 故事项目：

1. **主力用国产 API**：通义万象、CogView-4 质量已经很不错
2. **保留 GPT-Image 接口**：代码层抽象，方便切换
3. **关注 Azure 中国区**：等 GPT-Image 在中国区上线
4. **多模型对比**：不同场景选择最适合的模型

---

## 六、提示词优化技巧

### 6.1 提高一致性

```
# 关键短语
consistent character design
same art style throughout
unified color palette
matching line weight
coherent lighting direction
```

### 6.2 控制布局

```
# 网格布局
2x3 grid, 3x2 grid, 4x4 grid

# 排列方式
arranged in a row (横排)
arranged in a column (竖排)
scattered layout (自由布局)
comic panel layout (漫画分镜)
```

### 6.3 指定细节程度

```
# 高细节
highly detailed, intricate details, sharp focus, 8k

# 中等细节
detailed, clear, good quality

# 低细节（草图）
rough sketch, concept sketch, quick draft
```

### 6.4 风格控制

```
# 写实风格
photorealistic, realistic, photography style

# 动漫风格
anime style, manga style, cel shading

# 概念艺术
concept art, digital painting, illustration

# 技术插图
technical illustration, blueprint style, schematic
```

---

## 七、常见问题

### Q: 如何保持角色在多个面板中一致？
A: 使用 `consistent character design, same outfit, same hairstyle, maintain proportions`

### Q: 如何控制面板数量和布局？
A: 明确指定 `6-panel storyboard, 2x3 grid layout`

### Q: 如何让背景和角色分离？
A: 角色参考图用 `white background`，故事板可以指定每个面板的背景

### Q: 哪个平台最适合故事板生成？
A: GPT-Image-2 和 Midjourney v6 效果最好，国产推荐通义万象

### Q: 国内用哪个 API 最稳定？
A: 阿里云通义万象和智谱 CogView-4 都比较稳定，建议都试试

---

## 八、参考资源

### 8.1 在线工具
- [ChatGPT](https://chat.openai.com) - GPT-Image-2
- [Midjourney](https://midjourney.com) - MJ v6
- [Leonardo AI](https://leonardo.ai) - 角色参考功能
- [通义万象](https://tongyi.aliyun.com) - 阿里云
- [智谱清言](https://chatglm.cn) - CogView-4

### 8.2 学习资源
- [awesome-gpt4o-images](https://github.com/jamez-bondos/awesome-gpt4o-images)
- [awesome-gpt-image-2](https://github.com/YouMind-OpenLab/awesome-gpt-image-2)
- [Prompt Engineering Guide](https://github.com/dair-ai/Prompt-Engineering-Guide)

### 8.3 社区
- [r/midjourney](https://reddit.com/r/midjourney)
- [r/StableDiffusion](https://reddit.com/r/StableDiffusion)
- [Civitai](https://civitai.com) - SD 模型和提示词
