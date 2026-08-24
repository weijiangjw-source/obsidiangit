---
标题: HyperFrames 全套配置与操作指南
创建时间: 2026-08-17 17:47
修改时间: 2026-08-17 17:47
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---

# HyperFrames 全套配置与操作指南

> 基于 macOS + Claude Code + PM2 环境的完整配置复盘，从零开始打通 AI 视频制作全流程

---

## 一、工具全景说明

在开始配置之前，先理清这套工具链中每个组件的作用，避免概念混淆。

### 1.1 核心组件关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                    你的电脑 (macOS)                            │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │ Claude Code │  │   PM2      │  │  终端 CLI   │            │
│  │ (AI助手)    │  │ (进程守护) │  │ (手动命令)  │            │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘            │
│         │                │                │                    │
│         └────────────────┼────────────────┘                    │
│                          ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              HyperFrames (视频渲染引擎)                  │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  Skills (技能包，安装在 ~/.agents/skills/)        │  │  │
│  │  │  ├── hyperframes (路由入口)                       │  │  │
│  │  │  ├── hyperframes-core (核心创作契约)              │  │  │
│  │  │  ├── faceless-explainer (无面讲解)                │  │  │
│  │  │  ├── captions-overlay (字幕叠加)                  │  │  │
│  │  │  └── ... (共11个已安装技能)                       │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          │                                     │
│                          ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  依赖工具层                                              │  │
│  │  ├── Node.js 22+ (JavaScript 运行时)                    │  │
│  │  ├── FFmpeg (音视频编解码)                              │  │
│  │  └── Chrome/Chromium (浏览器渲染引擎)                   │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 各组件职责速查

| 组件 | 类型 | 职责 | 是否需要常驻 |
|------|------|------|-------------|
| **Claude Code** | AI 编程助手 | 接收自然语言指令，调用 Skills 生成/修改视频代码 | 按需启动 |
| **HyperFrames** | 视频渲染框架 | 将 HTML/CSS/JS 渲染为 MP4 视频 | 预览时需常驻 |
| **PM2** | 进程守护工具 | 让 HyperFrames 预览服务在后台持续运行 | 推荐常驻 |
| **FFmpeg** | 系统工具 | 视频编码、格式转换、音视频处理 | 无需常驻 |
| **Chrome** | 浏览器引擎 | 渲染视频画面（无头模式） | 预览时被调用 |

### 1.3 已安装的 Skills 清单

| Skill 名称 | 用途 |
|-----------|------|
| `hyperframes` | **入口路由** — 最先使用的 Skill，自动路由到合适的工作流 |
| `hyperframes-core` | **核心创作契约** — 定义视频的 HTML 规范和 timeline 机制 |
| `hyperframes-animation` | **动画知识库** — GSAP、Lottie、Three.js 等动画适配器 |
| `hyperframes-creative` | **创意指导** — 调色板、排版、旁白、节拍规划 |
| `hyperframes-cli` | **CLI 开发循环** — 所有 `npx hyperframes` 命令的说明 |
| `hyperframes-registry` | **可复用组件库** — Blocks 和 Components 的安装与管理 |
| `hyperframes-keyframes` | **关键帧动画** — 精确的 2D/3D 关键帧控制 |
| `media-use` | **媒体资产预处理** — TTS 配音、语音转文字、背景移除 |
| `faceless-explainer` | **无面讲解视频** — 任意文本 → 无人出镜的讲解视频 |
| `captions-overlay` | **字幕叠加** — TikTok 风格字幕（drop/rail/embed 模式） |
| `hyperframes-audio` | **音频处理** — 背景音乐、音效、配音合成 |

---

## 二、环境准备

### 2.1 必需的依赖工具

| 工具 | 最低版本 | 安装命令 | 验证命令 |
|------|---------|---------|---------|
| Node.js | 22+ | 从 nodejs.org 下载 | `node -v` |
| npm | 10+ | 随 Node.js 自带 | `npm -v` |
| FFmpeg | 6.0+ | `brew install ffmpeg` | `ffmpeg -version` |
| Chrome | 最新 | 从 google.com/chrome 下载 | 直接打开应用 |
| Git | 2.0+ | `brew install git` | `git --version` |

### 2.2 Node.js 版本检查（重要）

```bash
# 确认版本 >= 22
node -v
# 输出示例: v22.11.0
```

如果版本低于 22，建议使用 `nvm` 管理多版本 Node.js：

```bash
# 安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash

# 安装并切换 Node.js 22
nvm install 22
nvm use 22
nvm alias default 22
```

### 2.3 FFmpeg 安装与验证

```bash
# 使用 Homebrew 安装
brew install ffmpeg

# 验证安装（必须能正常输出版本信息）
ffmpeg -version
```

### 2.4 环境变量配置

将以下内容添加到 `~/.zshrc`（或 `~/.bash_profile`），避免每次手动设置：

```bash
# 告诉 Puppeteer 跳过 Chromium 下载，直接使用系统 Chrome
export PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true
export PUPPETEER_EXECUTABLE_PATH="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"

# 可选：国内用户设置 npm 镜像加速
# export NPM_CONFIG_REGISTRY=https://registry.npmmirror.com
```

使配置生效：

```bash
source ~/.zshrc
```

---

## 三、安装 HyperFrames 与 Skills

### 3.1 全局安装 vs 项目安装

HyperFrames 采用 **全局安装 CLI + 项目级安装 Skills** 的混合模式：

| 组件 | 安装方式 | 存放位置 | 说明 |
|------|---------|---------|------|
| `hyperframes` CLI | 全局安装或 npx 临时使用 | 随 npm 全局 | 推荐用 `npx` 无需全局安装 |
| HyperFrames Skills | 项目级或全局 | `~/.agents/skills/` | Skill 文件统一存储 |
| Skills 快捷方式 | 符号链接 | `~/.claude/skills/` | 指向 `~/.agents/skills/` |

### 3.2 为 Claude Code 安装 Skills

```bash
# 进入你的项目目录
cd /Users/eric/projects

# 安装 HyperFrames Skills 到 Claude Code
npx skills add heygen-com/hyperframes --full-depth
```

**交互式安装菜单操作指南**：

1. **第一步：选择 Skills**
   - 你会看到 `Core Skills` 和 `Other` 两个分类
   - 光标默认在 `Core Skills` 上，按 `Enter` 即可一键选择全部核心技能
   - 如需选择其他技能，用 `↑/↓` 移动，`Space` 勾选

2. **第二步：选择安装目标**
   ```
   Which agents do you want to install to?
   ○ Claude Code    ← 务必选中
   ○ Cursor         ← 如果使用则选中
   ○ Codex          ← 如果使用则选中
   ```
   - 用 `Space` 勾选你实际使用的 AI 编程助手
   - **推荐只勾选 Claude Code + Cursor**

3. **第三步：选择安装方式**
   ```
   Installation method
   ● Symlink (Recommended)   ← 选这个！
   ○ Copy to all agents
   ```
   - **必须选 Symlink**：所有 AI 助手共享同一份 Skill 源文件，方便更新

4. **按 `Enter` 确认安装**

### 3.3 验证 Skill 安装成功

```bash
# 查看 Skills 是否在 Claude Code 目录下
ls -la ~/.claude/skills/
# 应该看到 hyperframes -> ../../.agents/skills/hyperframes 等符号链接

# 查看 Skills 总数
ls ~/.agents/skills/ | wc -l
# 应该显示 11 个以上
```

### 3.4 更新 Skills

```bash
# 更新所有 HyperFrames Skills
npx hyperframes skills update

# 或重新安装最新版本
npx skills add heygen-com/hyperframes --full-depth --force
```

---

## 四、Claude Code 配置与模型切换

### 4.1 启动 Claude Code

```bash
# 进入你的项目目录
cd /Users/eric/projects/my-video-project

# 启动 Claude Code
claude
```

### 4.2 关键：切换 AI 模型（避坑重点）

⚠️ **这是整个配置中最容易出问题的环节**

部分企业环境会配置内部模型网关（如 `agnes-2.5-flash`），该网关在解析包含 HTML 标签的请求时可能触发 JSON 解析错误，导致所有指令（包括 `/hyperframes` 和普通对话）都报 `400 BadRequestError`。

**在 Claude Code 中执行以下步骤**：

```
# 输入以下命令查看可用模型
/model
```

如果列表中有标准模型，用 `↑/↓` 选择后按 `Enter`：
- `claude-3.5-sonnet`（推荐）
- `claude-3-opus`
- `claude-3-haiku`

**验证模型切换成功**：

```
# 输入一个简单英文指令测试
list files in this project
```

如果正常返回文件列表，说明模型切换成功，可以正常使用了。

**备用方案**：
- 如无法切换模型，考虑使用 **Cursor** 替代 Claude Code
- 或联系管理员确认 `agnes-2.5-flash` 网关的 JSON 解析限制

---

## 五、初始化视频项目

### 5.1 创建项目

```bash
# 进入你的 projects 目录
cd /Users/eric/projects

# 创建新的 HyperFrames 项目
npx hyperframes init my-video-project
```

**选择模板时的建议**：
- 如果你打算用 AI 从头生成视频内容 → 选 `Blank`（最干净）
- 如果你想参考现有示例 → 选 `Product Promo` 或 `Kinetic Type`
- 选错也没关系，可以随时删除项目重新创建

### 5.2 项目结构说明

```
my-video-project/
├── index.html          # 主入口文件（视频的"总控台"）
├── hyperframes.json    # 项目配置文件
├── package.json        # npm 依赖管理
├── compositions/       # 视频组件存放目录（AI 生成的内容放这里）
├── renders/            # 渲染输出的 MP4 文件
└── captures/           # 临时截图/素材缓存
```

### 5.3 验证项目可用

```bash
cd my-video-project

# 启动预览服务
npx hyperframes preview
```

浏览器打开 `http://localhost:3002`，如果能看到空白画面（或示例画面），说明项目创建成功。

---

## 六、PM2 进程守护配置

### 6.1 为什么需要 PM2

`hyperframes preview` 是一个常驻服务，需要保持运行才能让浏览器预览实时生效。PM2 可以：
- 让服务在**后台持续运行**（关闭终端也不影响）
- **自动重启**崩溃的进程
- 提供统一的**日志管理**

### 6.2 安装 PM2

```bash
npm install -g pm2
```

### 6.3 创建配置文件（方案二：推荐）

⚠️ **特别注意：文件扩展名必须为 `.cjs`**

由于项目 `package.json` 包含 `"type": "module"`，Node.js 会将 `.js` 文件视为 ES Module。PM2 配置文件使用 `module.exports`（CommonJS 语法），因此必须用 `.cjs` 扩展名，否则会报 `ReferenceError: module is not defined`。

在项目根目录创建 `ecosystem.config.cjs`：

```bash
cd /Users/eric/projects/my-video-project
touch ecosystem.config.cjs
```

编辑内容：

```javascript
module.exports = {
  apps: [
    {
      name: 'hf',                              // 简短进程名
      cwd: '/Users/eric/projects/my-video-project',  // 项目绝对路径
      script: 'npx',
      args: 'hyperframes preview',
      interpreter: 'none',                     // 直接执行 npx
      autorestart: true,
      max_memory_restart: '500M',
      env: {
        NODE_ENV: 'production',
      },
      out_file: '/Users/eric/.pm2/logs/hf-out.log',
      error_file: '/Users/eric/.pm2/logs/hf-error.log',
      merge_logs: true,
      log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    },
  ],
};
```

### 6.4 启动服务

```bash
# 在任意目录执行（配置文件使用绝对路径）
pm2 start /Users/eric/projects/my-video-project/ecosystem.config.cjs

# 或先 cd 进项目再启动
cd /Users/eric/projects/my-video-project
pm2 start ecosystem.config.cjs
```

### 6.5 PM2 日常管理命令

```bash
pm2 list          # 查看所有进程状态
pm2 logs hf       # 查看实时日志（帮助排查问题）
pm2 restart hf    # 重启服务
pm2 stop hf       # 停止服务
pm2 delete hf     # 删除进程
pm2 reload hf     # 平滑重启（零停机）
pm2 save          # 保存当前进程列表（用于开机自启）
pm2 startup       # 生成开机自启脚本（按提示执行）
```

### 6.6 验证服务正常运行

```bash
pm2 logs hf --lines 10
```

正常输出应包含：
```
Studio running
Project   my-video-project
Studio    http://localhost:3002
```

> ⚠️ 如果日志中出现 `[StaticGuard] Invalid HyperFrame contract` 警告，这是正常的——说明当前 `index.html` 是空白的，没有包含有效的视频组件。生成视频内容后警告自动消失。

---

## 七、在 Claude Code 中使用 Skills

### 7.1 基本使用流程

```
步骤 1: 确保 Claude Code 在项目目录下启动
步骤 2: 输入 /hyperframes 加载路由上下文（可选，首次建议执行）
步骤 3: 用自然语言描述你的视频需求
步骤 4: AI 自动生成/修改视频代码
步骤 5: 刷新 http://localhost:3002 预览效果
步骤 6: 迭代调整直到满意
步骤 7: 渲染导出 MP4
```

### 7.2 常用指令示例

**基础指令**（不用斜杠，直接对话）：

> "帮我在 index.html 中创建一个 5 秒的产品介绍视频，标题居中淡入，背景用深蓝色到紫色的渐变"

**使用专项 Skill**（先加载 Skill）：

```
/faceless-explainer
→ 等待 AI 响应后说："根据这段文本生成一个 30 秒的讲解视频：[粘贴你的文本]"
```

**使用组件库**：

> "帮我从 catalog 中添加一个数据图表组件到视频中"

### 7.3 渲染最终视频

```bash
# 在项目目录下的终端执行
cd /Users/eric/projects/my-video-project
npx hyperframes render --quality high --output renders/my-video.mp4
```

### 7.4 完整 Skill 使用说明书

> 以下内容已在前期配置过程中整理，此处作为附录完整收录。

**HyperFrames Skills 完整使用说明书（Claude Code 版）**

---

#### A. 核心 Skills

##### `/hyperframes` — 入口路由 Skill

- **作用**：接收"帮我做个视频"这类请求，自动判断应该使用哪个具体工作流
- **使用场景**：任何视频创作的起点，不确定用哪个 Skill 时就用它

##### `/hyperframes-core` — 核心创作契约

HyperFrames 从 HTML 渲染视频。一个 composition（视频作品）就是一个 HTML 文件，通过 `data-*` 属性声明时间轴。

**核心概念**：
- **Clip（片段）**：任何带有 `data-start`（开始时间）和 `data-duration`（持续时间）的 DOM 元素
- **Track（轨道）**：通过 `data-track-index` 控制图层叠放顺序，同一轨道上的片段不能时间重叠

**关键参考文档**（位于 `hyperframes-core/references/`）：

| 文档 | 用途 |
|------|------|
| `minimal-composition.md` | 最小可渲染的 composition 骨架 |
| `data-attributes.md` | 所有 `data-*` 属性速查 |
| `tracks-and-clips.md` | 轨道和片段的时间规则 |
| `sub-compositions.md` | 子 Composition 的组装方式 |
| `review-loop.md` | 计划→草图→构建的审核流程 |
| `determinism-rules.md` | 确定性渲染规则（禁止使用 `Date.now()` 等）|

##### `/hyperframes-animation` — 所有动画知识

涵盖 7 种运行时适配器：GSAP（默认）、Lottie、Three.js、Anime.js、CSS Keyframes、Web Animations API、TypeGPU。

**路由指南**：

| 需求 | 查阅 |
|------|------|
| 按触发条件/标签选原子运动模式 | `rules-index.md` |
| 读取某个规则的完整 HTML/CSS/GSAP 配方 | `rules/<name>.md` |
| 选多阶段场景模板 | `blueprints-index.md` |
| 场景间转场 | `transitions/` |
| GSAP API（时间线/tween/position 参数）| `adapters/gsap.md` |

##### `/hyperframes-creative` — 非动画创意指导

负责：调色板、排版、旁白、节拍规划、音频响应式视觉、构图模式、品牌/风格决策。

**⚠️ 最重要的事**：在写任何 HTML 之前，必须先读这两个文件：
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

##### `/hyperframes-cli` — CLI 开发循环

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

##### `/media-use` — 媒体资产预处理

一个 Skill 搞定所有媒体需求：背景音乐、音效、图片、图标、品牌 Logo、配音、调色、LUT。

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

##### `/hyperframes-registry` — 可复用组件库

安装和复用预制的 Blocks 和 Components。

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

##### `/hyperframes-keyframes` — 关键帧动画

用于需要 2D/3D 关键帧、GSAP 时间线、CSS 关键帧、Anime.js、WAAPI、FLIP、路径、蒙版、SVG 变形/绘制、文字拖尾、3D 景深等精确动画控制。

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

#### B. 创作工作流 Skills

##### `/faceless-explainer` — 无面讲解视频

将任意文本（文章、笔记、话题、brief）转化为无人出镜的讲解视频。

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
Step 4: Visual Design → enriched STORYBOARD.md
Step 5: Frames → compositions/frames/NN-*.html + index.html
Step 6: Final Render → renders/video.mp4
```

**保持最新**：

```bash
npx hyperframes skills update faceless-explainer
```

##### `/captions-overlay` — 字幕叠加

为口播视频或发布视频添加 TikTok 风格的字幕叠加。

- 支持三种模式：**drop**（下沉）、**rail**（轨道）、**embed**（嵌入）
- 引用自 `embedded-captions` 的 rail+embed 模型

---

#### C. 实用提示

**保持 Skills 最新**：

```bash
npx hyperframes skills update   # 默认更新核心集
```

**环境检查**：

```bash
npx hyperframes doctor          # 检查所有依赖是否就绪
```

**提示词最佳实践**：
- 始终以 `/hyperframes` 开头，它会加载路由 + composition 上下文
- 冷启动示例：
  > `Using /hyperframes, create a 10-second product intro with a fade-in title, a background video, and background music.`
- 从现有内容创作：
  > `Take a look at this GitHub repo [url] and explain its uses to me using /hyperframes.`
  > `Summarize the attached PDF into a 45-second pitch video using /hyperframes.`

**重要资源**：
- 官方文档：https://hyperframes.heygen.com/introduction
- 提示词指南：https://hyperframes.heygen.com/guides/prompting
- 组件目录（Catalog）：https://hyperframes.heygen.com/catalog/blocks/data-chart

---

## 八、配置过程中的失误与复盘

### 8.1 失误汇总表

| 序号 | 问题现象 | 表面原因 | 根本原因 | 正确做法 |
|------|---------|---------|---------|---------|
| 1 | `npx skills add` 卡在交互菜单 | 以为命令在后台执行 | 进入了交互式选择模式 | 按提示用 `↑/↓` 和 `Space` 操作，按 `Enter` 确认 |
| 2 | 安装依赖时长时间卡住 | 以为进程死锁 | Puppeteer 在下载 300MB Chromium | 设置 `PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true` |
| 3 | Claude Code 持续报 `400 JSON parse error` | 以为中文指令有问题 | 内部模型网关 `agnes-2.5-flash` 解析 HTML 标签时 JSON 格式错误 | 切换模型到标准 Claude 模型 |
| 4 | PM2 报 `module is not defined` | 以为配置文件写错了 | `package.json` 有 `"type": "module"`，`.js` 被当 ES Module 解析 | 配置文件使用 `.cjs` 扩展名 |
| 5 | 预览日志显示 `Invalid HyperFrame contract` | 以为配置有故障 | `index.html` 是空白的，没有视频组件 | 这是正常提示，生成视频内容后消失 |
| 6 | 用 `/hyperframes 中文指令` 报错 | 以为不能用中文 | 根本原因仍是模型网关问题 | 先 `/hyperframes` 加载上下文，再单独发中文描述，或直接切换模型 |

### 8.2 关键经验总结

1. **提前设置环境变量**：`PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true` 应在安装前就配置好，可省去大量等待时间

2. **模型选择是稳定性的关键**：企业代理网关可能对特殊字符敏感，遇到莫名其妙的 JSON 解析错误时，优先尝试切换模型

3. **PM2 配置注意文件扩展名**：如果项目使用 ES Module（`"type": "module"`），PM2 配置文件必须用 `.cjs`

4. **日志是最好的朋友**：遇到问题先用 `pm2 logs` 或查看终端输出，90% 的故障都能从日志中找到线索

5. **Skills 是共享的，项目是独立的**：Skills 安装在 `~/.agents/skills/` 全局共享，每个视频项目单独创建，互不影响

---

## 九、快速排障指南

### 9.1 Claude Code 报 `400 BadRequestError`

```bash
# 1. 检查当前模型
/model

# 2. 切换到标准模型（如 sonnet）
# 在 /model 列表中选择 claude-3.5-sonnet

# 3. 测试基本指令
list files

# 4. 如果仍不行，重置会话
# 完全退出 Claude Code 后重新启动
```

### 9.2 预览服务无法访问 `localhost:3002`

```bash
# 1. 检查 PM2 状态
pm2 list

# 2. 查看日志
pm2 logs hf --lines 20

# 3. 重启服务
pm2 restart hf

# 4. 如果端口被占用，修改配置中的 PORT 环境变量
# 在 ecosystem.config.cjs 的 env 中添加: PORT: 3003
```

### 9.3 渲染视频失败

```bash
# 1. 检查 FFmpeg 是否可用
ffmpeg -version

# 2. 检查项目是否通过 check
npx hyperframes check

# 3. 重新渲染
npx hyperframes render --verbose
```

### 9.4 Skills 无法加载

```bash
# 1. 验证 Skills 是否已安装
ls ~/.claude/skills/

# 2. 重新安装
npx skills add heygen-com/hyperframes --full-depth --force

# 3. 重启 Claude Code
```

---

## 十、附录：完整命令速查表

### 10.1 环境准备

```bash
# 安装必需工具
brew install ffmpeg git

# 安装 Node.js（如未安装）
# 从 nodejs.org 下载或使用 nvm

# 安装 PM2
npm install -g pm2
```

### 10.2 Skills 管理

```bash
# 安装 HyperFrames Skills
npx skills add heygen-com/hyperframes --full-depth

# 更新 Skills
npx hyperframes skills update

# 查看已安装 Skills
ls ~/.claude/skills/
```

### 10.3 项目管理

```bash
# 创建项目
npx hyperframes init my-project

# 进入项目
cd my-project

# 预览
npx hyperframes preview

# 检查
npx hyperframes check

# 渲染
npx hyperframes render --quality high --output renders/out.mp4
```

### 10.4 PM2 管理

```bash
# 启动
pm2 start ecosystem.config.cjs

# 查看状态
pm2 list

# 查看日志
pm2 logs hf

# 重启
pm2 restart hf

# 停止
pm2 stop hf

# 删除
pm2 delete hf

# 开机自启
pm2 save
pm2 startup
```

---

*本指南基于 HyperFrames v0.7.109 + Claude Code v2.1.199 + PM2 实际配置经验整理，适用于 macOS 环境。*