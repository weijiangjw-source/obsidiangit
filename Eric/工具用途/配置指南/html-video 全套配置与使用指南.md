---
标题: html-video 全套配置与使用指南
创建时间: 2026-08-05 11:56
修改时间: 2026-08-05 12:00
引用渠道: DeepSeek
是否修改: false
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---



# html-video 全套配置与使用指南

> 基于 Mac M5 + MLX 本地模型环境，从零到一打通自然语言驱动视频生成全流程


## 📋 目录

1. [工具生态全景概览](#1-工具生态全景概览)
2. [环境准备](#2-环境准备)
3. [html-video 安装与构建](#3-html-video-安装与构建)
4. [本地模型配置（核心难点）](#4-本地模型配置核心难点)
5. [配音方案：Edge-TTS 配置](#5-配音方案edge-tts-配置)
6. [创建便捷命令（告别长路径）](#6-创建便捷命令告别长路径)
7. [后台服务管理（PM2）](#7-后台服务管理pm2)
8. [视频合成完整流程](#8-视频合成完整流程)
9. [工具必要性说明](#9-工具必要性说明)
10. [踩坑总结与最佳实践](#10-踩坑总结与最佳实践)


## 1. 工具生态全景概览

在开始之前，先理解你手中这套工具链的层级关系：

| 层级 | 工具 | 作用 |
| :--- | :--- | :--- |
| **设计理念层** | Open Design | 开源 AI 设计平台，`html-video` 是其视频方向的具体实现 |
| **视频生成框架** | html-video | 将 HTML 动画渲染为 MP4 的核心工具 |
| **AI 大脑** | Claude Code + 本地 MLX 模型 | 理解你的自然语言，生成对应的 HTML/CLI 代码 |
| **配音引擎** | Edge-TTS | 免费、高质量的文本转语音（需联网） |
| **视频合成** | ffmpeg | 将画面和配音合成为最终 MP4 |
| **进程管理** | PM2 | 让 Studio 服务在后台持续运行 |

**核心工作流**：
```
你输入自然语言 → Claude Code 理解并生成命令 → html-video CLI 执行渲染 → 输出 MP4 画面
                                                              ↓
                                            Edge-TTS 生成配音 → ffmpeg 合并 → 最终带配音视频
```


## 2. 环境准备

### 2.1 基础依赖安装

| 工具 | 最低版本 | 检查命令 | 安装方式 |
| :--- | :--- | :--- | :--- |
| Node.js | 20+ | `node --version` | 官网下载或 `brew install node` |
| pnpm | 9+ | `pnpm --version` | `npm install -g pnpm` |
| ffmpeg | 任意最近版本 | `ffmpeg -version` | `brew install ffmpeg` |
| Python 3 | 3.6+ | `python3 --version` | macOS 自带或官网下载 |
| Chromium | — | — | `npx playwright install chromium` |

### 2.2 Python 环境注意事项（重要）

macOS 自带的 Python 3 已启用 **PEP 668** 保护机制，直接使用 `pip3 install` 会报错：

```
error: externally-managed-environment
```

**解决方案：使用 pipx 安装命令行工具**

```bash
brew install pipx
pipx ensurepath
# 关闭并重新打开终端使 PATH 生效
```

后续所有 Python 命令行工具的安装统一使用 `pipx install <包名>`。


## 3. html-video 安装与构建

### 3.1 ⚠️ 注意：不要使用 npm 全局安装

`@nexu-io/html-video` **并未发布到 npm 仓库**，直接执行 `npm install -g @nexu-io/html-video` 会报 404 错误。必须从 GitHub 克隆源码。

### 3.2 正确安装步骤

```bash
# 1. 克隆项目
git clone https://github.com/nexu-io/html-video.git
cd html-video

# 2. 安装依赖（耗时约 2-5 分钟）
pnpm install

# 3. 构建项目（耗时约 5-15 分钟）
pnpm -r build
```

### 3.3 构建过程中的警告说明

执行 `pnpm -r build` 时可能出现以下警告，**均不影响使用**：

| 警告内容 | 原因 | 是否处理 |
| :--- | :--- | :--- |
| `Some chunks are larger than 500 kB` | 前端界面文件较大 | 忽略 |
| `Failed to create bin at ... webpack` | Remotion 适配器尚未构建 | 忽略（暂不用 Remotion） |
| `Update available! 9.15.0 → 11.18.0` | pnpm 有新版本 | 忽略或按需更新 |

### 3.4 验证安装

```bash
node packages/cli/dist/bin.js --help
```

如显示帮助信息，则安装成功。


## 4. 本地模型配置（核心难点）

### 4.1 模型选型建议

根据实测，以下是适用于 `html-video` 的本地模型推荐排序：

| 排名 | 模型 | 大小 | 推荐度 | 原因 |
| :--- | :--- | :--- | :--- | :--- |
| 1 | **Qwen2.5-Coder-14B-Instruct-4bit** | ~8.3GB | ⭐⭐⭐⭐⭐ | 代码生成专精、稳定、内存友好 |
| 2 | **DeepSeek-Coder-V2-Lite-Instruct-4bit-mlx** | ~8.2GB | ⭐⭐⭐⭐⭐ | 专为代码优化，Mac 高效运行 |
| 3 | **gemma-4-12b-coder-fable5-composer2.5-8bit** | ~8GB | ⭐⭐⭐⭐ | 社区微调版，代码能力不错 |
| ❌ | **DeepSeek-R1-0528-Qwen3-8B-MLX-8bit** | ~8GB | 不推荐 | 回复逻辑混乱（汽车维修微调版） |
| ❌ | **gpt-oss-20b-MXFP4-Q8** | ~10GB | 不推荐 | 不认识 html-video，偏题 |

### 4.2 ⚠️ 关键教训：不要用垃圾模型浪费时间

**遇到的问题**：使用 `DeepSeek-R1-0528-Qwen3-8B-MLX-8bit` 时，发送“你好”却回复“解决汽车副驾驶加热问题”，说明该模型被社区微调成了垂直领域专用模型，完全不适合通用对话和代码生成。

**教训**：下载模型前先确认其用途描述，避免使用名称混乱、来源不明的社区版本。

### 4.3 MLX 模型内存限制处理

Mac 系统默认限制 GPU 可锁定内存约 25GB。如果模型运行时触顶报错：

```
process memory limit exceeded (usage 24.9 GB, abort threshold 23.7 GB)
```

**解决方案**：
1. **先换模型**（推荐 14B 级别而非 32B+）
2. 关闭浏览器等占用内存的应用
3. 如仍不足，可提升内核限制：
   ```bash
   sudo sysctl iogpu.wired_limit_mb=28672
   ```
   （重启后失效，需重新执行）


## 5. 配音方案：Edge-TTS 配置

### 5.1 为什么选择 Edge-TTS

- **完全免费**：无需 API Key，无调用次数限制
- **音质优秀**：使用微软 Azure 同源神经网络语音
- **命令行可编程**：适合集成到自动化流程中
- **多语言支持**：60+ 语言，多种情感语音

### 5.2 安装 Edge-TTS

```bash
pipx install edge-tts
```

安装后验证：
```bash
edge-tts --list-voices | grep zh-CN
```

### 5.3 常用命令

```bash
# 基础配音
edge-tts --text "你的文本" --voice zh-CN-XiaoxiaoNeural --write-media output.mp3

# 调整语速（-50% 到 +100%）
edge-tts --rate=-50% --text "慢速朗读" --voice zh-CN-XiaoxiaoNeural --write-media output.mp3

# 调整音调
edge-tts --pitch=+20Hz --text "高音调" --voice zh-CN-XiaoxiaoNeural --write-media output.mp3
```

### 5.4 常见中文语音推荐

| 语音名称 | 风格 | 性别 |
| :--- | :--- | :--- |
| `zh-CN-XiaoxiaoNeural` | 标准女声 | 女 |
| `zh-CN-YunxiNeural` | 标准男声 | 男 |
| `zh-CN-YunjianNeural` | 新闻播报风格 | 男 |
| `zh-CN-XiaoyiNeural` | 温暖亲切 | 女 |


## 6. 创建便捷命令（告别长路径）

### 6.1 创建全局命令 `hv`

每次输入 `node packages/cli/dist/bin.js generate ...` 太过冗长。创建一个全局命令：

```bash
sudo tee /usr/local/bin/hv > /dev/null << 'EOF'
#!/bin/bash
cd /Users/eric/html-video
node packages/cli/dist/bin.js "$@"
EOF

sudo chmod +x /usr/local/bin/hv
```

### 6.2 测试验证

```bash
hv --help
```

### 6.3 原理说明

- 系统 `PATH` 环境变量包含 `/usr/local/bin`
- 输入 `hv` 时，系统在此目录找到可执行文件
- 脚本自动 `cd` 到项目目录，`"$@"` 传递所有参数

### 6.4 其他推荐的全局命令

```bash
# 启动 Studio 图形界面（前台）
alias hvs='cd /Users/eric/html-video && node packages/cli/dist/bin.js studio'

# 或者使用 PM2 后台启动（见下一节）
```


## 7. 后台服务管理（PM2）

### 7.1 为什么需要 PM2

`html-video Studio` 是一个 Web 服务，需要持续运行。使用 PM2 可以：
- 在后台运行，关闭终端不影响服务
- 开机自启（可选）
- 方便查看日志、重启、停止

### 7.2 安装 PM2

```bash
npm install pm2 -g
```

### 7.3 创建后台启动命令 `hvs`

```bash
sudo tee /usr/local/bin/hvs > /dev/null << 'EOF'
#!/bin/bash
cd /Users/eric/html-video
if pm2 list | grep -q "html-video-studio"; then
    pm2 restart html-video-studio
else
    pm2 start node --name "html-video-studio" -- packages/cli/dist/bin.js studio
fi
echo ""
pm2 status html-video-studio
echo ""
echo "🌐 访问地址: http://127.0.0.1:3071"
echo "📋 查看日志: pm2 logs html-video-studio"
EOF

sudo chmod +x /usr/local/bin/hvs
```

### 7.4 PM2 常用管理命令

| 用途 | 命令 |
| :--- | :--- |
| 启动/重启服务 | `hvs` |
| 查看状态 | `pm2 status` |
| 查看日志 | `pm2 logs html-video-studio` |
| 停止服务 | `pm2 stop html-video-studio` |
| 删除服务 | `pm2 delete html-video-studio` |
| 开机自启 | `pm2 startup && pm2 save` |

### 7.5 ⚠️ 重要提示

**CLI 模式（在终端/Claude Code 中生成视频）不依赖 Studio 服务**。只有当你需要打开浏览器使用图形界面时，才需要启动 `hvs`。


## 8. 视频合成完整流程

### 8.1 方式一：在 Claude Code 中自然语言驱动

在 Claude Code 中输入：

```
请使用 hv 生成一个关于 AI 发展历程的 15 秒短视频，内容涵盖图灵测试到现代大语言模型的关键节点。
```

### 8.2 方式二：手动 CLI 命令

```bash
hv generate "关于 AI 发展历程的 15 秒短视频" --template modern
```

### 8.3 配音合成（画面生成后）

画面输出为 `/tmp/ai_timeline.mp4` 后：

```bash
# 1. 生成配音
edge-tts --text "你的旁白文本" --voice zh-CN-XiaoxiaoNeural --write-media /tmp/voice.mp3

# 2. 合并音视频
ffmpeg -y -i /tmp/ai_timeline.mp4 -i /tmp/voice.mp3 -c:v copy -c:a aac -shortest /tmp/final_video.mp4

# 3. 播放查看
open /tmp/final_video.mp4
```

### 8.4 在 Claude Code 中合成配音的问题与解决

**遇到的问题**：Claude Code 无法直接执行 `edge-tts` 或 `ffmpeg`，只会生成脚本让你手动运行。

**原因**：Claude Code 的 Bash 工具对这类命令存在安全限制，不会自动批准执行。

**解决方案**：
1. **手动复制命令到终端执行**（最直接，也是最推荐的）
2. 封装成 Skill 供 Claude Code 调用（进阶）

### 8.5 ⚠️ 音频时长与视频时长不匹配

使用 `-shortest` 参数让 ffmpeg 以较短的流为准，确保视频与音频同步：

```bash
ffmpeg -y -i video.mp4 -i audio.mp3 -c:v copy -c:a aac -shortest output.mp4
```


## 9. 工具必要性说明

### 9.1 你不需要的工具

| 工具 | 是否必需 | 说明 |
| :--- | :--- | :--- |
| **OpenMontage** | ❌ 不需要 | 庞大的全自动化视频生产系统，对你当前需求过剩。先掌握 `html-video` 即可。 |
| **Automator** | ❌ 不需要 | 图形化自动化工具，与你的命令行工作流不匹配 |
| **MiniMax API** | ❌ 不需要 | Edge-TTS 已提供免费配音，无需额外付费 |

### 9.2 你需要的工具清单

| 工具 | 必要性 | 说明 |
| :--- | :--- | :--- |
| **html-video** | ✅ 核心 | 视频画面生成引擎 |
| **Claude Code** | ✅ 核心 | 理解自然语言，驱动整个流程 |
| **本地 MLX 模型** | ✅ 核心 | AI 推理后端（Qwen2.5-Coder 等） |
| **Edge-TTS** | ✅ 推荐 | 免费配音 |
| **ffmpeg** | ✅ 必需 | 视频合成与处理 |
| **PM2** | ⚠️ 可选 | 仅在使用 Studio 图形界面时需要 |


## 10. 踩坑总结与最佳实践

### 10.1 配置阶段常见错误

| 错误现象 | 原因 | 解决方案 |
| :--- | :--- | :--- |
| `npm install -g ...` 404 | 包未发布到 npm | 必须从 GitHub 克隆源码 |
| `pnpm -r build` 警告 | 子包未构建或文件过大 | 忽略警告，不影响使用 |
| Python `externally-managed-environment` | PEP 668 保护机制 | 使用 `pipx` 而非 `pip3` |
| 模型回复离谱内容 | 使用了被污染/微调错误的模型 | 换用官方或知名社区的 Coder 模型 |
| 内存超限报错 | 模型过大或上下文过载 | 换 14B 模型或提升内核限制 |

### 10.2 使用阶段常见问题

| 问题 | 原因 | 解决方案 |
| :--- | :--- | :--- |
| Studio 显示 "Not logged in" | Agent 未登录 | 该提示仅针对云端 API，本地模式可忽略 |
| Studio 聊天框无法使用本地模型 | 设计如此 | 在 Claude Code 终端中使用 CLI 方式 |
| Claude Code 不执行 edge-tts | 安全限制 | 手动复制命令到终端执行 |
| 视频无声音 | 未配置配音 | 使用 Edge-TTS 生成配音后用 ffmpeg 合并 |

### 10.3 最佳实践总结

1. **模型选择要谨慎**：优先选择官方发布的 Coder 系列模型（Qwen2.5-Coder / DeepSeek-Coder），避免使用名称混乱、来源不明的社区模型。

2. **区分两种工作模式**：
   - **终端/Claude Code 模式**（推荐）：使用 `hv` 命令，不依赖 Studio 服务
   - **图形界面模式**：需启动 `hvs` 服务，适合预览模板和手动调整

3. **配音与画面分开处理**：`html-video` 处理画面，Edge-TTS + ffmpeg 处理配音，两者独立且灵活。

4. **善用全局命令**：`hv` 和 `hvs` 大幅提升操作效率，值得花 5 分钟配置。

5. **不要过度复杂化**：对于内容创作者的需求，`html-video` + Edge-TTS 已经足够，不需要一开始就引入 OpenMontage 等大型系统。


## 📎 快速参考卡片

### 日常使用命令速查

```bash
# 生成视频（在 Claude Code 中或终端）
hv generate "你的视频描述"

# 启动 Studio 图形界面（后台）
hvs

# 查看 Studio 状态
pm2 status

# 停止 Studio
pm2 stop html-video-studio

# 配音生成
edge-tts --text "文本" --voice zh-CN-XiaoxiaoNeural --write-media /tmp/voice.mp3

# 合并音视频
ffmpeg -y -i video.mp4 -i voice.mp3 -c:v copy -c:a aac -shortest output.mp4

# 播放视频
open /tmp/output.mp4
```

---

*本指南基于实际配置经验整理，适用于 Mac M5 + MLX 本地模型环境。如有问题，可根据错误信息针对性排查。*