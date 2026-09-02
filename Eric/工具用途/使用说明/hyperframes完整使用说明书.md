---
标题: hyperframes完整使用说明书
创建时间: 2026-08-15 23:55
修改时间: 2026-08-15 23:55
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---


## 一、快速上手：在 Claude Code 中使用 Skills

安装完成后，**需要重启 Claude Code 会话**，Skills 才会生效。在 Claude Code 中，这些 Skills 会注册为**斜杠命令（Slash Commands）**。

**核心使用方式**：在 Claude Code 对话框中输入斜杠命令 + 你的需求描述即可。

```bash
/hyperframes 创建一个10秒的产品介绍视频，包含淡入标题和背景音乐
```

---

## 二、核心 Skills（Core Skills）

这些是**每个 HyperFrames 项目都必需的基础技能**，你已全部安装。

### 1. `/hyperframes` — 入口路由 Skill

**这是最先要读的 Skill**，是整个系统的能力地图和意图路由器。

- **作用**：接收“帮我做个视频”这类请求，自动判断应该使用哪个具体工作流
- **使用场景**：任何视频创作的**起点**，不确定用哪个 Skill 时就用它

### 2. `/hyperframes-core` — 核心创作契约

HyperFrames 从 **HTML 渲染视频**。一个 composition（视频作品）就是一个 HTML 文件，通过 `data-*` 属性声明时间轴。

**核心概念**：
- **Clip（片段）**：任何带有 `data-start`（开始时间）和 `data-duration`（持续时间）的 DOM 元素
- **Track（轨道）**：通过 `data-track-index` 控制图层叠放顺序，同一轨道上的片段**不能时间重叠**

**关键参考文档**（位于 `hyperframes-core/references/`）：
| 文档 | 用途 |
|------|------|
| `minimal-composition.md` | 最小可渲染的 composition 骨架 |
| `data-attributes.md` | 所有 `data-*` 属性速查 |
| `tracks-and-clips.md` | 轨道和片段的时间规则 |
| `sub-compositions.md` | 子 Composition 的组装方式 |
| `review-loop.md` | 计划→草图→构建的审核流程 |
| `determinism-rules.md` | 确定性渲染规则（禁止使用 `Date.now()` 等）|

### 3. `/hyperframes-animation` — 所有动画知识

涵盖 7 种运行时适配器：**GSAP**（默认）、Lottie、Three.js、Anime.js、CSS Keyframes、Web Animations API、TypeGPU。

**工作方式**：
- **默认**：从 `rules-index.md` 中挑选 2-4 条原子规则，用单个暂停的 GSAP 时间线组合
- **Blueprint（蓝图）**：当场景匹配现成的多阶段场景模板时加载（如品牌揭示、社交证明等）

**路由指南**：
| 需求 | 查阅 |
|------|------|
| 按触发条件/标签选原子运动模式 | `rules-index.md` |
| 读取某个规则的完整 HTML/CSS/GSAP 配方 | `rules/<name>.md` |
| 选多阶段场景模板 | `blueprints-index.md` |
| 场景间转场 | `transitions/` |
| GSAP API（时间线/tween/position 参数）| `adapters/gsap.md` |

### 4. `/hyperframes-creative` — 非动画创意指导

负责：**调色板、排版、旁白、节拍规划、音频响应式视觉、构图模式、品牌/风格决策**。

**⚠️ 最重要的事**：在写任何 HTML 之前，**必须先读这两个文件**——它们是避免产出“网页感”视频的关键：
- `house-style.md` — 如何将 prompt 转化为真实内容
- `video-composition.md` — 视频媒介的尺度、景深和前景细节

**路由指南**：
| 需求 | 查阅 |
|------|------|
| 采用现成的 frame-preset | `frame-presets/` |
| 默认调色板/动效/排版 | `house-style.md` |
| 命名风格预设 | `visual-styles.md` |
| 构图模式（画中画、文字衬底等）| `composition-patterns.md` |
| 逐拍指导、节奏规划 | `beat-direction.md` |

### 5. `/hyperframes-cli` — CLI 开发循环

涵盖所有 CLI 命令：`init`、`add`、`catalog`、`capture`、`lint`、`check`、`preview`、`render`、`publish`、`doctor`、`transcribe`、`tts`、`remove-background` 等。

**标准开发流程**：
```bash
# 1. 脚手架
npx hyperframes init <项目名>

# 2. 编写 composition（使用 /hyperframes-core）

# 3. 快速反馈 — 每次 HTML 修改后运行
npx hyperframes lint

# 4. 最终检查（包含 lint + 运行时错误 + 布局检查）
npx hyperframes check

# 5. 预览
npx hyperframes preview

# 6. 渲染（只有审核通过后才渲染）
npx hyperframes render --quality high --output out.mp4
```

**⚠️ 两个不同的预览界面不要混淆**：
| 界面 | 何时打开 | 用途 |
|------|----------|------|
| Storyboard Board | composition 检查之前 | 审核计划卡片和线框图 |
| Final Composition Preview | `check` 通过后 | 审核组装好的完整时间线 |

### 6. `/media-use` — 媒体资产预处理

一个 Skill 搞定所有媒体需求：**背景音乐、音效、图片、图标、品牌 Logo、配音、调色、LUT**。

**核心能力**：
- **TTS 配音**（Kokoro）
- **音频/视频转录**（Whisper，默认使用 Parakeet，WER 更低且快 5-10 倍）
- **背景移除**（u2net，用于透明叠加层）

**音频引擎用法**：
```bash
# 完整音频处理（TTS配音 + 背景音乐 + 音效）
node <SKILL_DIR>/audio/scripts/audio.mjs --request ./audio_request.json --out ./audio_meta.json

# 只运行子集
node <SKILL_DIR>/audio/scripts/audio.mjs --only tts,bgm --request ./audio_request.json --out ./audio_meta.json
```

### 7. `/hyperframes-registry` — 可复用组件库

安装和复用预制的**块（Blocks）**和**组件（Components）**。

**常用命令**：
```bash
npx hyperframes add data-chart        # 安装一个 block
npx hyperframes add grain-overlay     # 安装一个 component
npx hyperframes add captions          # 安装所有 tagged captions 的块
npx hyperframes catalog               # 浏览可用组件
```

**Block 的接入方式**：
```html
<div data-composition-id="data-chart" 
     data-composition-src="compositions/data-chart.html" 
     data-start="2" data-duration="15" 
     data-track-index="1" 
     data-width="1920" data-height="1080">
</div>
```

### 8. `/hyperframes-keyframes` — 关键帧动画

用于需要 **2D/3D 关键帧、GSAP 时间线、CSS 关键帧、Anime.js、WAAPI、FLIP、路径、蒙版、SVG 变形/绘制、文字拖尾、3D 景深**等精确动画控制。

**关键帧契约**：
- 可见状态、连续的主体身份、可 seek 的运行时、经过像素验证

**GSAP 关键帧骨架**：
```javascript
const root = document.querySelector("[data-composition-id]");
const compositionId = root.dataset.compositionId;
const tl = gsap.timeline({ paused: true });
tl.addLabel("state-a", 0);
tl.to(".subject", { 
  keyframes: [ 
    { x: 0, opacity: 1, duration: 0.2 },
    { x: 120, opacity: 1, duration: 0.4, ease: "power2.out" }
  ] 
});
window.__timelines[compositionId] = tl;
```

**⚠️ 确定性渲染禁令**（以下操作不能用于渲染关键动画）：
- `Date.now()` / `performance.now()`
- 未种子的 `Math.random()`
- hover/scroll 触发器、定时器
- 异步创建的时间线
- 无限循环

---

## 三、创作工作流 Skills（你已安装的扩展）

### 9. `/faceless-explainer` — 无面讲解视频

将**任意文本**（文章、笔记、话题、brief）转化为无人出镜的讲解视频。

- **时长**：最长约 3 分钟，最佳 30-90 秒
- **视觉风格**：所有画面由 LLM 创造（字体排版、抽象图形、图表、数据可视化）
- **适用场景**：主题讲解、概念拆解、教程、清单体、叙事型讲解

**工作流**（7 步）：
```
Step 0: Setup → hyperframes.json
Step 1: Brief → capture/extracted/
Step 2: Design System → frame.md
Step 3: Storyboard/Script → STORYBOARD.md + SCRIPT.md
Step 3.1: Audio → audio_meta.json
Step 4: Visual Design →  enriched STORYBOARD.md
Step 5: Frames → compositions/frames/NN-*.html + index.html
Step 6: Final Render → renders/video.mp4
```

**保持最新**：
```bash
npx hyperframes skills update faceless-explainer
```

### 10. `/captions-overlay` — 字幕叠加

为**口播视频或发布视频**添加 TikTok 风格的字幕叠加。

- 支持三种模式：**drop**（下沉）、**rail**（轨道）、**embed**（嵌入）
- 引用自 `embedded-captions` 的 rail+embed 模型

---

## 四、你可能还需要了解的其他 Workflow Skills

根据官方文档，HyperFrames 还包含以下工作流 Skills（你可能未全部安装，但可按需添加）：

| 斜杠命令 | 输入 → 输出 |
|----------|-------------|
| `/product-launch-video` | 任何网站 URL / brief → 产品发布/宣传视频 |
| `/pr-to-video` | GitHub PR → 代码变更解说视频 |
| `/embedded-captions` | 现有口播视频 → 添加字幕的同一视频 |
| `/talking-head-recut` | 现有口播视频 → 带图形叠加层的包装视频 |
| `/motion-graphics` | 短小无旁白的动态图形（logo sting、统计数字等）|
| `/music-to-video` | 音乐曲目 → 节拍同步视频 |
| `/general-video` | 不匹配任何专项工作流时的通用创作 |

**按需安装**：
```bash
# 安装单个 workflow
npx skills add heygen-com/hyperframes --skill product-launch-video --full-depth

# 安装全部（跳过选择器）
npx skills add heygen-com/hyperframes --all --full-depth
```

---

## 五、实用提示

### 保持 Skills 最新
```bash
npx hyperframes skills update   # 默认更新核心集
```

### 环境检查
```bash
npx hyperframes doctor          # 检查所有依赖是否就绪
```

### 提示词最佳实践
- **始终以 `/hyperframes` 开头**，它会加载路由 + composition 上下文，让 agent 第一次就选对工作流
- **冷启动示例**：
  > `Using /hyperframes, create a 10-second product intro with a fade-in title, a background video, and background music.`
- **从现有内容创作**：
  > `Take a look at this GitHub repo [url] and explain its uses to me using /hyperframes.`
  > `Summarize the attached PDF into a 45-second pitch video using /hyperframes.`

### 重要资源
- 官方文档：https://hyperframes.heygen.com/introduction
- 提示词指南：https://hyperframes.heygen.com/guides/prompting
- 组件目录（Catalog）：https://hyperframes.heygen.com/catalog/blocks/data-chart
- 在线 Playground：https://www.hyperframes.dev/