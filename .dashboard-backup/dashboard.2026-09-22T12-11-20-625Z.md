---
dashboard: true
banner:
  quote: "The mind is everything. What you think you become."
  author: "jiang wei"
  image: "https://images.pexels.com/photos/2307638/pexels-photo-2307638.jpeg"
  quoteColor: "#d46d0c"
  images:
    - "https://images.pexels.com/photos/2307638/pexels-photo-2307638.jpeg"
    - "https://images.pexels.com/photos/10664504/pexels-photo-10664504.jpeg"
  mode: stats
  statsConfig:
    excludeFolders:
      - "English"
      - "Eric/工具用途"
      - "Eric/提示词"
      - "Eric/选题方向"
      - "Templates"
      - "Wiki/raw"
      - "Wiki/wiki/sources"
      - "clippings"
      - "copilot"
    accent: "#e85d11"
    showDetails: true
    showLeft: true
    showCenter: true
    showRight: true
    leftStat: totalNotes
    centerStat: streak
    rightStats:
      - taskCompletion
      - connectivity
      - avgLinksPerNote
columns:
  - name: Memo
    color: "#f59e0b"
    type: memo
  - name: 文件夹
    color: "#6366f1"
    type: folder
    library:
      viewMode: gallery
      sortBy: "created"
      sortDesc: true
      folders:
        - "Eric/公众号图文/镜鉴野望"
        - "Eric/公众号图文/品味流溢"
      excludeFolders:
        - "English"
        - "Eric/工具用途"
        - "Eric/提示词"
        - "Others"
        - "Templates"
        - "Wiki/raw"
        - "Wiki/wiki/sources"
        - "copilot"
      templatePath: "Templates/record/图文模板.md"
      kanbanShowCovers: true
      propertyLimit: 4
      quickDateFilter:
        property: "created"
        start: ""
        end: ""
        days: 7
---

## Memo

### ffmpeg 使用命令
id: demo-memo-1
type: generic
ffmpeg 使用命令：
你问到了关键点：ffmpeg 默认会在你“当前所在的目录”里找这个文件。
当前工作目录（Current Working Directory）
当你打开终端，会有一个“当前位置”，比如：
```bash
/Users/你的用户名
```
这就是当前工作目录。如果直接写 视频.mp4，ffmpeg 只会在这个目录里找。
如何查看当前目录？
```bash
pwd
```
会输出类似 /Users/你的用户名/Desktop。
如何指定文件路径？
1. 使用绝对路径（从根目录写起）
```bash
ffmpeg -i /Users/你的用户名/Downloads/视频.mp4
```
2. 使用相对路径（相对于当前目录）
· 当前目录下的文件：视频.mp4
· 当前目录里的子文件夹：素材/视频.mp4
· 上级目录：../视频.mp4（两个点表示上一级）
3. 先切换目录，再运行
```bash
cd /Users/你的用户名/Downloads
ffmpeg -i 视频.mp4
```
实际操作建议
· 如果你知道文件在 下载 文件夹，可以：
```bash
ffmpeg -i ~/Downloads/视频.mp4
```
~ 是当前用户主目录（/Users/你的用户名）的简写。
· 如果你不想打完整路径，也可以直接把文件从访达拖进终端，会自动填入完整路径：
1. 输入 ffmpeg -i （后面留一个空格）
2. 把视频文件从文件夹拖到终端窗口
3. 终端会自动显示类似 ffmpeg -i /Users/.../视频.mp4
所以答案是：需要指定路径，只是如果你把文件放在当前终端所在的目录，可以只写文件名；否则必须写清楚它在哪个文件夹下。

### Claude环境变量
id: demo-memo-path
type: generic
Claude code在终端中切换本地登录命令：
Claude-local.   本地
Claude-offical. 官方
在claudian中本地模型的环境变量：
ANTHROPIC_BASE_URL=http://localhost:1234
ANTHROPIC_AUTH_TOKEN=any-value-here
在claudian中云端模型的环境变量：
ANTHROPIC_BASE_URL=https://open.bigmodel.cn/api/anthropic
ANTHROPIC_AUTH_TOKEN=8b39eba5a85e47ee958f8d9722b38d06.yvSnbDixvJiovoW2
ANTHROPIC_DEFAULT_OPUS_MODEL=glm-4.7
ANTHROPIC_DEFAULT_SONNET_MODEL=glm-4.7
ANTHROPIC_DEFAULT_HAIKU_MODEL=glm-4.5-air
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
API_TIMEOUT_MS=3000000
这个填完还需要在Claudia选项卡中的Claude CLI path里配置路径：/Users/eric/.local/bin/claude，这两步完成后云端模型才能正常工作。

### oMLX启动命令
id: demo-memo-rename
type: generic
⚙️ oMLX 服务管理核心命令
启动服务并设开机自启：brew services start omlx
停止服务并取消开机自启：brew services stop omlx
重启服务：brew services restart omlx
查看服务当前状态：brew services info omlx
这会显示服务的运行状态（running/stopped）、日志位置等信息，是排查问题的第一步。
列出所有服务：brew services list
✅ 如何验证服务已成功运行？
除了用 brew services info 查看状态，访问 http://localhost:8000/admin 能打开 oMLX 管理后台，也是服务启动成功的最直观证明。
找出真正的 API Key
在普通终端里运行以下命令，直接查看 oMLX 配置文件里的密钥：
cat /Users/eric/.omlx/settings.json | grep api_key
先在当前窗口彻底清理干净之前可能残留的后台：
killall omlx 2>/dev/null
重新加载修改后的环境变量：
命令：source ~/.zshrc
启动 oMLX 服务端（带上正确的路径和指定的 Key）
为了让 oMLX 能够准确抓取到你的 Gemma 模型，并且把暗号（API Key）固定下来，请在运行服务端的终端里，使用以下命令启动：
omlx serve --model-dir /Users/eric/.omlx/models/mlx-community --api-key sk-ant-omlx
💡 启动后请肉眼检查日志：
•	确认是否输出了 Discovered 1 models 或成功识别到了你的 Gemma 模型。
•	确认是否有 API key authentication: enabled。

### Yt-下载命令
id: card-mucmnpag
type: generic
Yt-dlp常用下载命令：
1、下载最佳画质
yt-dlp "https://fxtwitter.com/VibePonyAI/status/2073756013484982370?s=20"
2、如果你想明确指定下载最高质量的视频和音频并合并，可以使用 -f 参数：
yt-dlp -f 'bestvideo+bestaudio' "https://fxtwitter.com/VibePonyAI/status/2073756013484982370?s=20"
3、指定下载目录：使用 -o 参数可以指定下载路径和文件名：
yt-dlp -o "~/Downloads/%(title)s.%(ext)s" "视频链接"
4、查看所有格式：
yt-dlp -F "视频链接"
5、更改下载位置：
yt-dlp -P ~/Videos/ "视频链接"
6、直接使用浏览器Cookies：
yt-dlp --cookies-from-browser chrome "视频链接"
7、下载为MP4格式：
yt-dlp --merge-output-format mp4 "视频链接"

### html-video和comfyUI
id: card-mucmoq75
type: generic
手动在终端启动html-video与comfyUI
1、进入 html-video 项目目录：
cd /Users/eric/html-video
2、重新启动服务：
node packages/cli/dist/bin.js studio
3、当终端显示 Server running at http://127.0.0.1:3071 时，再次打开浏览器访问这个地址。
http://127.0.0.1:3071/
4、直接终端启动命令：hvs
5、每次生成视频需要指向位置：
/Users/eric/html-video/packages/cli/dist/bin.js（已经创建脚本，可以在任何目录下用 hv 开头）
比如：hv generate "生成一个 15 秒的 AI 发展史视频"
6、comfyui启动一键命令：comfy
pm2 list               # 查看状态
pm2 logs comfyui       # 查看日志
pm2 restart comfyui    # 重启
pm2 stop comfyui       # 停止
pm2 delete comfyui     # 删除
🧰 PM2 常用管理命令
用途
命令
查看服务状态
pm2 status
查看实时日志
pm2 logs html-video-studio
停止服务
pm2 stop html-video-studio
重启服务
pm2 restart html-video-studio
删除服务
pm2 delete html-video-studio
🔄 设置开机自启（可选）
如果你希望电脑重启后 Studio 自动运行，执行：
bash
pm2 startup
pm2 save
Litellm日常理命令:
操作
命令
启动（若关机后）
pm2 start litellm
重启（修改配置后）
pm2 restart litellm
停止
pm2 stop litellm
查看实时日志
pm2 logs litellm
设置开机自启
pm2 startup（复制输出执行）-> pm2 save

### 临时接触MAC内存限制
id: card-mucmrbcg
type: generic
系统内存临时解除限制：
sudo sysctl iogpu.wired_limit_mb=27648

### Claude调用gemini命令
id: card-mucmsmui
type: generic
Claude code调用gemini命令：
1、终端启动：export LITELLM_LOCAL_MODEL_COST_MAP=True
litellm --config ~/.litellm/config.yaml --detailed_debug
2、另一个终端执行测试：
export ANTHROPIC_BASE_URL="http://localhost:4000"
export ANTHROPIC_AUTH_TOKEN="sk-1234567890"
export ANTHROPIC_MODEL="gemini-model"
claude
3、查看日志：tail -f ~/.litellm/litellm_background.log
4、彻底关掉后台：kill $(lsof -t -i:4000)

## 文件夹
