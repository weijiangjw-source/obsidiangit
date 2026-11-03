---
标题: DTK v5.1.10 MCP 智能体接入与使用指南
创建时间: 2026-11-03 22:59
修改时间: 2026-11-03 22:59
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---

# DTK v5.1.10 MCP 智能体接入与使用指南

> **MCP (Model Context Protocol)** 能让你的 AI 助手（如 Claude、Cursor）直接连接你的本地 DTK 系统。你只需用自然语言下达指令，AI 就会自动调用 DTK 的接口帮你完成解析、下载、查评论、语音转写等所有操作。

---

## 🛠️ 一、 前置准备（极其重要）

1. **确保核心服务已启动**：你的 Docker 容器必须处于运行状态，特别是 `api`、`worker` 和 `downloader`（如果 AI 要帮你下载视频的话）。
2. **获取 API Key**：
   * 打开 Safari 访问 `http://127.0.0.1:8001`。
   * 左侧菜单进入 **`API Key`** 页面，点击新建。
   * **权限选择**：必须选择 `admin` 或 `operator`，否则 AI 没有权限操作。
   * **复制并保存** 以 `dtk_` 开头的长串密钥（页面刷新后无法再次查看）。
3. **确认 MCP 地址**：在 DTK 控制台左侧点击 **`MCP`**，确认 Endpoint 地址是 `http://127.0.0.1:8001/mcp/`。

---

## 🔌 二、 接入 AI 客户端（三选一）

### 方法 1：接入 Cursor 编辑器（最推荐，图形化操作）
如果你用 Cursor 写代码，这是最直观的：
1. 打开 Cursor 设置（`Cmd + ,`），搜索 `MCP`。
2. 点击 **`+ Add New MCP Server`**。
3. 填写以下信息：
   * **Name**: `DTK-Server`
   * **Type**: `http` 或 `streamable-http`
   * **URL**: `http://127.0.0.1:8001/mcp/`
4. 在 Headers 区域，添加一行：
   * Key: `Authorization`
   * Value: `Bearer 你的真实API_KEY`
5. 保存，如果前面出现绿色小圆点，说明连接成功。

### 方法 2：接入 Claude Code（终端命令行）
打开 Mac 的终端，执行以下命令（**替换为你自己的 API Key**）：
```bash
claude mcp add --transport http dtk http://127.0.0.1:8001/mcp/ \
  --header "Authorization: Bearer dtk_YOUR_API_KEY"
```
执行后在终端输入 `claude mcp list`，看到 `dtk` 即可。

### 方法 3：接入 Claude Desktop 桌面版
1. 打开 Claude Desktop 的配置文件（Mac 路径通常在 `~/Library/Application Support/Claude/claude_desktop_config.json`）。
2. 在 `mcpServers` 节点下添加：
```json
{
  "mcpServers": {
    "dtk": {
      "url": "http://127.0.0.1:8001/mcp/",
      "headers": {
        "Authorization": "Bearer 你的真实API_KEY"
      }
    }
  }
}
```
3. 重启 Claude Desktop。

---

## 🧰 三、 AI 能帮你做什么？（可用工具列表）

成功连接后，你的 AI 就拥有了以下 10 把“武器”：

| 工具名称 | 作用 |
| :--- | :--- |
| `parse_url` | 解析抖音/B站/TikTok链接，提取元数据。 |
| `get_video` | 根据 ID 获取单个视频/图集详情。 |
| `get_user` | 获取作者主页信息。 |
| `list_user_posts` | 翻页获取作者的作品列表。 |
| `list_comments` | 翻页获取帖子的评论（需要登录态 Cookie）。 |
| `get_content_history` | 获取视频或作者的归档指标历史数据。 |
| `pool_status` | 查看你的身份池健康状态。 |
| `download_media` | 将视频/图集下载并存入本地库（返回任务 ID）。 |
| `transcribe_video` | 提取已下载视频的语音并转成文字（依赖 transcriber 容器）。 |
| `get_task_result` | 获取上述长耗时任务的最终结果。 |

---

## 🗣️ 四、 实战演练：如何与 AI 对话？

配置好后，你不需要再点任何网页按钮，直接对 AI 说：

**场景 1：一键下载并转写**
> "帮我把这个 B 站链接的视频下载下来，然后提取语音字幕。链接是：`https://www.bilibili.com/video/BV1xx411c7mD`"

**场景 2：批量抓取与健康检查**
> "先查一下现在的身份池（pool_status）健康吗？如果健康，帮我抓取这个抖音博主的最新 5 条视频的评论，存到数据库里。"

**场景 3：排查错误**
> "我刚刚解析一个抖音链接失败了，你帮我查一下身份池状态，看看是不是 Cookie 过期了？"

*(AI 在后台会自动调用 `parse_url` -> `download_media` -> `transcribe_video`，你只需看着它执行即可。)*

---

## ⚠️ 五、 官方避坑指南（必读）

1. **API Key 权限限制**：如果你生成的 Key 只有 `douyin:read` 权限，AI 就算用 MCP 也绝对查不了 TikTok 或 B站的数据，必须生成全量权限的 Key。
2. **防 AI 卡死**：如果下载大视频或转写大文件，耗时较长。AI 调用 `download_media` 时不会死等，而是立刻收到一个 `task_id`。此时 AI 需要调用 `get_task_result` 去取最终结果，不要让它一直在原地循环。
3. **翻页注意事项**：让 AI 抓取列表时，要提醒 AI：“用返回的 `cursor` 翻页”，而不是直接去改 `count` 参数，否则会被系统拒绝。
4. **本地环境隔离**：MCP 依赖本地 API，所以你的 Mac 必须保持开机，Docker 必须保持运行，Claude/Cursor 才能随时呼叫它。

现在，去生成你的 API Key，选个喜欢的 AI 客户端接上去吧。把之前手动点页面的痛苦全部交给 AI！