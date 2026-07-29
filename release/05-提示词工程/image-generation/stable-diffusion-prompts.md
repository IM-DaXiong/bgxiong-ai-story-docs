# Stable Diffusion 提示词指南

> 来源：综合整理自多个 Stable Diffusion 提示词资源
> 说明：SD 图像生成提示词结构、常用标签和示例

---

## 提示词结构

### 基础结构
```
[质量标签], [主体描述], [风格修饰], [技术参数]
```

### 完整示例
```
masterpiece, best quality, highly detailed, 
a beautiful anime girl with long flowing hair, 
standing in a cherry blossom garden, 
soft lighting, pastel colors, 
8k wallpaper, ultra-detailed
```

---

## 质量标签

### 通用质量标签
- `masterpiece` - 杰作
- `best quality` - 最佳质量
- `high quality` - 高质量
- `highres` - 高分辨率
- `ultra-detailed` - 超详细
- `highly detailed` - 高度详细
- `8k wallpaper` - 8K 壁纸
- `4k` - 4K 分辨率
- `sharp focus` - 清晰对焦
- `professional` - 专业
- `award-winning` - 获奖
- `trending on artstation` - 在 ArtStation 上流行

### 负面质量标签（用于 negative prompt）
- `low quality` - 低质量
- `worst quality` - 最差质量
- `lowres` - 低分辨率
- `bad anatomy` - 错误解剖
- `bad hands` - 错误的手
- `missing fingers` - 缺少手指
- `extra digits` - 多余的手指
- `fewer digits` - 手指数量不对
- `cropped` - 裁剪不当
- `watermark` - 水印
- `signature` - 签名
- `text` - 文字
- `error` - 错误
- `blurry` - 模糊
- `jpeg artifacts` - JPEG 伪影
- `duplicate` - 重复
- `morbid` - 病态
- `mutilated` - 残缺

---

## 主体描述

### 人物描述

#### 外貌
- `beautiful face` - 美丽的面容
- `pretty face` - 漂亮的脸
- `cute face` - 可爱的脸
- `detailed eyes` - 细致的眼睛
- `glowing eyes` - 发光的眼睛
- `sparkling eyes` - 闪闪发光的眼睛
- `long hair` - 长发
- `short hair` - 短发
- `flowing hair` - 飘逸的头发
- `curly hair` - 卷发
- `straight hair` - 直发
- `ponytail` - 马尾
- `twintails` - 双马尾
- `bun` - 丸子头

#### 服装
- `dress` - 连衣裙
- `school uniform` - 校服
- `kimono` - 和服
- `armor` - 盔甲
- `casual clothes` - 休闲装
- `formal wear` - 正装
- `fantasy outfit` - 奇幻服装
- `elegant dress` - 优雅的裙子

#### 姿势
- `standing` - 站立
- `sitting` - 坐着
- `walking` - 行走
- `running` - 奔跑
- `jumping` - 跳跃
- `floating` - 漂浮
- `dancing` - 跳舞
- `fighting pose` - 战斗姿势

### 场景描述

#### 自然场景
- `mountain` - 山
- `forest` - 森林
- `ocean` - 海洋
- `river` - 河流
- `lake` - 湖泊
- `sky` - 天空
- `sunset` - 日落
- `sunrise` - 日出
- `starlight` - 星光
- `moonlight` - 月光

#### 室内场景
- `bedroom` - 卧室
- `living room` - 客厅
- `library` - 图书馆
- `classroom` - 教室
- `office` - 办公室
- `cafe` - 咖啡馆
- `studio` - 工作室

#### 奇幻场景
- `castle` - 城堡
- `dungeon` - 地牢
- `floating island` - 浮空岛
- `magical forest` - 魔法森林
- `crystal cave` - 水晶洞穴
- `ancient ruins` - 古代遗迹

### 物品描述

#### 自然物品
- `flower` - 花
- `tree` - 树
- `rock` - 岩石
- `water` - 水
- `fire` - 火
- `cloud` - 云

#### 人造物品
- `sword` - 剑
- `shield` - 盾牌
- `book` - 书
- `potion` - 药水
- `crystal` - 水晶
- `gem` - 宝石

---

## 风格修饰

### 艺术风格

#### 动漫风格
- `anime` - 动漫
- `manga` - 漫画
- `cel shading` - 色块阴影
- `flat color` - 平涂
- `anime style` - 动漫风格
- `japanese animation` - 日本动画

#### 写实风格
- `realistic` - 写实
- `photorealistic` - 照片级真实
- `hyperrealistic` - 超写实
- `cinematic` - 电影级
- `photography` - 摄影
- `real life` - 真实生活

#### 艺术风格
- `oil painting` - 油画
- `watercolor` - 水彩
- `digital art` - 数字艺术
- `concept art` - 概念艺术
- `illustration` - 插画
- `sketch` - 素描
- `pixel art` - 像素艺术
- `3D render` - 3D 渲染

#### 特殊风格
- `cyberpunk` - 赛博朋克
- `steampunk` - 蒸汽朋克
- `fantasy` - 奇幻
- `sci-fi` - 科幻
- `gothic` - 哥特
- `vintage` - 复古
- `retro` - 怀旧
- `minimalist` - 极简主义

### 光线修饰

#### 自然光
- `natural lighting` - 自然光
- `sunlight` - 阳光
- `golden hour` - 黄金时刻
- `soft light` - 柔光
- `warm light` - 暖光
- `cool light` - 冷光

#### 人工光
- `studio lighting` - 影棚灯光
- `dramatic lighting` - 戏剧性灯光
- `rim lighting` - 轮廓光
- `backlighting` - 逆光
- `neon lights` - 霓虹灯
- `glowing` - 发光

#### 特殊光
- `volumetric lighting` - 体积光
- `god rays` - 耶稣光
- `bioluminescence` - 生物发光
- `ethereal light` - 空灵的光

### 色彩修饰

#### 色调
- `warm tones` - 暖色调
- `cool tones` - 冷色调
- `pastel colors` - 粉彩色
- `vibrant colors` - 鲜艳色彩
- `muted colors` - 柔和色彩
- `monochromatic` - 单色
- `desaturated` - 去饱和

#### 特殊色彩
- `neon colors` - 霓虹色
- `iridescent` - 彩虹色
- `holographic` - 全息
- `metallic` - 金属色

---

## 技术参数

### 分辨率参数
- `8k` - 8K 分辨率
- `4k` - 4K 分辨率
- `highres` - 高分辨率
- `ultra-detailed` - 超详细
- `extremely detailed` - 极其详细

### 质量参数
- `masterpiece` - 杰作
- `best quality` - 最佳质量
- `high quality` - 高质量
- `professional` - 专业
- `award-winning` - 获奖

### 风格参数
- `trending on artstation` - 在 ArtStation 上流行
- `featured on pixiv` - 在 Pixiv 上精选
- `unreal engine` - 虚幻引擎
- `octane render` - Octane 渲染
- `cinema 4d` - Cinema 4D

---

## 提示词示例

### 1. 动漫女孩
```
masterpiece, best quality, highly detailed, 
1girl, beautiful face, detailed eyes, 
long flowing hair, cherry blossom petals, 
school uniform, standing, 
soft lighting, pastel colors, 
anime style, 8k wallpaper
```

**负面提示词**：
```
low quality, worst quality, lowres, bad anatomy, 
bad hands, missing fingers, extra digits, 
fewer digits, cropped, watermark, signature, 
text, error, blurry, jpeg artifacts
```

### 2. 奇幻战士
```
masterpiece, best quality, highly detailed, 
fantasy warrior, armor, sword, shield, 
dramatic lighting, epic pose, 
standing on battlefield, 
cinematic, 8k, professional
```

**负面提示词**：
```
low quality, worst quality, lowres, bad anatomy, 
bad hands, missing fingers, extra digits, 
fewer digits, cropped, watermark, signature, 
text, error, blurry, jpeg artifacts, duplicate
```

### 3. 风景画
```
masterpiece, best quality, highly detailed, 
beautiful landscape, mountain, lake, 
sunset, golden hour, 
reflection in water, 
oil painting style, 8k, professional
```

**负面提示词**：
```
low quality, worst quality, lowres, 
watermark, signature, text, error, 
blurry, jpeg artifacts, duplicate
```

### 4. 赛博朋克城市
```
masterpiece, best quality, highly detailed, 
cyberpunk cityscape, neon lights, 
rain-soaked streets, holographic ads, 
flying cars, dense urban, 
blade runner style, 8k, cinematic
```

**负面提示词**：
```
low quality, worst quality, lowres, 
watermark, signature, text, error, 
blurry, jpeg artifacts, duplicate, 
bad architecture, bad perspective
```

### 5. 可爱宠物
```
masterpiece, best quality, highly detailed, 
cute cat, fluffy fur, big eyes, 
soft lighting, warm colors, 
sitting on windowsill, 
photography style, 8k, professional
```

**负面提示词**：
```
low quality, worst quality, lowres, 
bad anatomy, extra limbs, missing limbs, 
watermark, signature, text, error, 
blurry, jpeg artifacts, duplicate
```

---

## 高级技巧

### 1. 权重调整

使用括号和数字调整权重：
```
(beautiful:1.2) eyes  // 增加 "beautiful" 的权重
(eyes:0.8)           // 减少 "eyes" 的权重
```

### 2. 负面提示词

使用 negative prompt 排除不需要的元素：
```
Positive: a beautiful landscape
Negative: buildings, people, cars
```

### 3. 步数控制

调整采样步数：
- 20-30 步：快速生成
- 30-50 步：标准质量
- 50-100 步：高质量

### 4. CFG Scale

调整提示词遵循程度：
- 7-12：标准范围
- 更高值：更严格遵循提示词
- 更低值：更多创意自由

### 5. 采样器选择

常用采样器：
- `Euler a` - 快速，适合原型
- `DPM++ 2M Karras` - 平衡质量速度
- `DPM++ SDE Karras` - 高质量
- `DDIM` - 确定性结果

### 6. 种子固定

使用固定种子获得一致结果：
```
seed: 12345
```

### 7. 图像到图像

使用 img2img 进行风格转换：
1. 输入参考图像
2. 描述期望风格
3. 调整去噪强度

---

## 常见问题

### Q: 如何提高图像质量？
A: 添加质量标签如 `masterpiece, best quality, highly detailed, 8k`

### Q: 如何获得特定风格？
A: 使用明确的风格标签，如 `anime, oil painting, cyberpunk`

### Q: 如何控制光线？
A: 使用光线标签如 `golden hour, dramatic lighting, soft light`

### Q: 如何调整构图？
A: 使用构图标签如 `close-up, wide angle, rule of thirds`

### Q: 如何获得一致的风格？
A: 使用相同的风格标签组合，保持一致性

### Q: 如何避免常见错误？
A: 使用负面提示词排除问题元素

### Q: 如何提高生成速度？
A: 减少采样步数，使用更快的采样器

### Q: 如何获得更创意的结果？
A: 降低 CFG Scale，增加随机性

---

## 工具和资源

### 1. 在线工具
- [Stable Diffusion WebUI](https://github.com/AUTOMATIC1111/stable-diffusion-webui)
- [ComfyUI](https://github.com/comfyanonymous/ComfyUI)
- [Civitai](https://civitai.com)

### 2. 模型资源
- [Hugging Face](https://huggingface.co)
- [Civitai Models](https://civitai.com/models)
- [Lexica](https://lexica.art)

### 3. 学习资源
- [Stable Diffusion Art](https://stable-diffusion-art.com)
- [Prompt Hero](https://prompthero.com)
- [OpenArt](https://openart.ai)

---

## 总结

Stable Diffusion 提示词编写是一门结合艺术理解和技术参数的实践。通过掌握基础结构、使用合适的标签、调整技术参数，并持续实验优化，你可以生成高质量的图像。

记住：
1. **质量标签**是基础
2. **主体描述**要具体
3. **风格修饰**要明确
4. **技术参数**要合理
5. **负面提示词**很重要
6. **迭代实验**是关键
