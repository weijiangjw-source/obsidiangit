---
标题: yt-dlp常用命令操作指南
创建时间: 2026-08-09 18:34
修改时间: 2026-08-09 18:34
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---

这是一份 `yt-dlp` 的常用命令操作指南，涵盖了从基础下载到高级用法的各种场景，方便你随时查阅。

---

### ⚙️ 准备工作

在开始之前，请确保你已经安装了 `yt-dlp` 和 `ffmpeg`。

*   **安装 `yt-dlp` 和 `ffmpeg`** (macOS 使用 Homebrew):
    ```bash
    brew install yt-dlp ffmpeg
    ```
    > **注意**: `ffmpeg` 是**必需**的，它用于合并 yt-dlp 分别下载的视频和音频流。

*   **验证安装**:
    ```bash
    yt-dlp --version
    ffmpeg -version
    ```

---

### 📖 基础用法

#### 1. 下载单个视频
默认情况下，yt-dlp 会下载最高画质，并自动合并音视频。
```bash
yt-dlp "视频URL"
```

#### 2. 指定下载目录和文件名
*   **`-P`**: 指定下载目录。
*   **`-o`**: 自定义文件名和路径。`%(title)s` 是视频标题，`%(ext)s` 是文件扩展名。
```bash
yt-dlp -P "/Users/eric/Downloads" -o "%(title)s.%(ext)s" "视频URL"
```

#### 3. 查看所有可用格式
在决定下载哪种画质前，可以先查看该视频所有可用的格式列表。
```bash
yt-dlp -F "视频URL"
```
输出会显示一个表格，包含格式代码 (`format code`)、分辨率、编码等信息。

---

### 🎨 画质与格式选择

#### 1. 使用 `-f` 选择特定格式
从 `-F` 命令的结果中，选择你想要的**格式代码**进行下载。
```bash
yt-dlp -f 137 "视频URL"    # 下载格式代码为 137 的视频
yt-dlp -f 137+140 "视频URL" # 下载视频(137)和音频(140)并合并
```

#### 2. 使用 `-f` 选择最佳画质
这是下载最高画质的标准方式。
```bash
yt-dlp -f "bestvideo+bestaudio/best" "视频URL"
```
*   `bestvideo+bestaudio`: 选择最好的视频流和最好的音频流，然后用 ffmpeg 合并。
*   `/best`: 如果合并失败，则回退到下载最佳的组合格式。

#### 3. 使用 `-S` 按条件排序（更推荐）
`-S` 参数是更现代、更灵活的画质选择方式。
```bash
# 优先选择分辨率最接近 1080p 的版本
yt-dlp -S "res:1080" "视频URL"

# 优先选择 H.264 编码的 MP4 视频（兼容性最好）
yt-dlp -S "vcodec:h264,acodec:aac" "视频URL"

# 优先选择文件较小的版本
yt-dlp -S "+size,+br" "视频URL"
```

#### 4. 限制最高画质
```bash
# 下载画质不高于 720p 的视频
yt-dlp -S "res:720" "视频URL"

# 或者使用 -f 方式
yt-dlp -f "bestvideo[height<=720]+bestaudio/best" "视频URL"
```

---

### 🎵 只下载音频

使用 `-x` 参数可以提取音频。
```bash
# 下载最佳音质并转换为 mp3
yt-dlp -x --audio-format mp3 "视频URL"

# 下载最佳音质，保留原始格式（如 m4a）
yt-dlp -x -f bestaudio "视频URL"

# 下载最佳音质并嵌入封面和元数据
yt-dlp -x -f bestaudio --add-metadata --embed-thumbnail "视频URL"
```

---

### 📂 处理播放列表

#### 1. 下载整个播放列表
直接提供播放列表的 URL 即可。
```bash
yt-dlp "播放列表URL"
```

#### 2. 只下载播放列表中的部分视频
```bash
# 只下载第1到第5个，以及第8个视频
yt-dlp --playlist-items 1-5,8 "播放列表URL"
```

#### 3. 只下载单个视频（忽略播放列表）
如果链接是一个播放列表，但你只想下载其中的某一个视频。
```bash
yt-dlp --no-playlist "视频URL"
```

---

### 🚀 高级与实用功能

#### 1. 使用浏览器 Cookies（绕过登录限制）
对于需要登录或受年龄限制的视频，可以使用浏览器的 Cookies。
```bash
# 从 Safari 浏览器获取 Cookies
yt-dlp --cookies-from-browser safari "视频URL"

# 从 Chrome 浏览器的特定用户配置中获取
yt-dlp --cookies-from-browser "chrome:/path/to/profilefolder" "视频URL"
```

#### 2. 使用代理
```bash
# 使用 SOCKS5 代理
yt-dlp --proxy "socks5://127.0.0.1:1080" "视频URL"

# 使用 HTTP 代理
yt-dlp --proxy "http://127.0.0.1:7890" "视频URL"
```

#### 3. 多线程下载（加速）
使用 `aria2c` 作为外部下载器可以显著提升速度。
```bash
# 使用 aria2c，并开启16个线程
yt-dlp --downloader aria2c --downloader-args "aria2c:-x 16 -k 1M" "视频URL"
```

#### 4. 断点续传
默认情况下，yt-dlp 会自动续传未完成的下载。如需强制重新下载：
```bash
yt-dlp --no-continue "视频URL"
```

#### 5. 下载字幕
```bash
# 下载所有可用字幕
yt-dlp --write-subs --all-subs "视频URL"

# 只下载英文字幕
yt-dlp --write-subs --sub-langs "en" "视频URL"
```

#### 6. 嵌入元数据和缩略图
```bash
yt-dlp --embed-metadata --embed-thumbnail "视频URL"
```

#### 7. 批量下载
创建一个文本文件（如 `urls.txt`），每行一个 URL，然后使用 `-a` 参数。
```bash
yt-dlp -a urls.txt
```

---

### ⚙️ 配置与维护

#### 1. 更新 yt-dlp
```bash
yt-dlp -U
```

#### 2. 配置文件
你可以创建一个配置文件来设置默认选项，避免每次都输入长串命令。文件位置如下：
*   **用户配置**: `~/.config/yt-dlp/config`
*   **系统配置**: `/etc/yt-dlp.conf`

配置文件每行一个选项，例如：
```bash
# ~/.config/yt-dlp/config
-P /Users/eric/Downloads
-o %(title)s.%(ext)s
-S res:1080
--no-playlist
```

---

### 💡 实用技巧

*   **获取帮助**: 任何时候都可以使用 `yt-dlp --help` 查看所有选项。
*   **忽略错误**: 在处理大型播放列表时，使用 `-i` 可以跳过下载失败的视频，继续下载后面的内容。
*   **网络问题**: 如果网络不稳定，可以使用 `--retries infinite` 让 yt-dlp 无限重试。