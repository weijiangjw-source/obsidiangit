
要实现你的目标，需要一个核心思路：因为 `claude` 命令原生只支持 `ANTHROPIC_BASE_URL` 这一个接口地址，所以它不能同时连接多个后端，需要通过外挂工具来实现。我们可以用一个叫做 **Claude Code Router (CCR)** 的中转服务作为“AI总机”，来统一管理这两个模型，让终端和 Obsidian 都能通过它随意切换。下面是具体的配置步骤：

### 🧱 1. 配置中转服务：Claude Code Router (CCR)
*   **安装与认证**：在终端依次运行以下命令：
    ```bash
    npm install -g claude-code-router @vitorcen/gemini-cli-openai
    gemini
    ```
    在Gemini界面输入 `/auth` 登录你的Google账号，LM Studio不需要认证，只需确保你已启动本地服务器。
*   **创建配置文件**：在终端执行 `mkdir -p ~/.claude && nano ~/.claude/ccr-config.json`，粘贴以下内容后保存。
```json
{
  "Providers": [
    {
      "name": "LMStudio",
      "api_base_url": "http://localhost:1234/v1/messages",
      "api_key": "lmstudio",
      "models": ["local-model"]
    },
    {
      "name": "Gemini",
      "api_base_url": "http://127.0.0.1:8787/v1/chat/completions",
      "api_key": "sk-or-v1_not-need",
      "models": ["gemini-2.5-pro", "gemini-2.5-flash"]
    }
  ],
  "Router": { "default": "Gemini,gemini-2.5-pro" }
}
```
*   **启动服务**：运行 `gemini-cli-openai` 和 `claude-code-router start`。

### ✍️ 2. 配置 Obsidian 插件：Claudian
*   **让 CCR 接管**：打开Obsidian设置中的Claudian，在**Codex Provider**部分开启并填入 **CLI Path**: `claude-code-router`，**Provider**: `Codex`。

### 💻 3. 配置终端环境变量（可选）
你可以在 `~/.zshrc` 文件中添加以下内容，让终端也能无缝使用CCR：
```bash
# Claude Code Router 配置
export ANTHROPIC_BASE_URL="http://localhost:6060"
export ANTHROPIC_API_KEY="your-api-key"
```
然后运行 `source ~/.zshrc` 使配置生效。

### 🔄 4. 在终端和 Obsidian 中切换模型
*   **在Obsidian中切换**：打开Claudian聊天窗口，在左下角选择 **Codex**，CCR会根据你的默认配置自动处理。
*   **在终端中切换**：
    *   **临时切换**：执行 `claude -p "Hello" --model gemini-2.5-pro`。
    *   **交互界面切换**：输入 `/model` 并选择模型。
    *   **全局切换**：修改 `ccr-config.json` 中的 `default` 字段。

---

这样，你的个人 AI 工作台就搭建好了：本地用 LM Studio（省钱、私密），云端用 Gemini（处理复杂任务），无论写笔记还是写代码，都能在终端和 Obsidian 里无缝切换。

根据配置文档，首次安装启动 Gemini CLI 时会要求 Google OAuth 认证，记得检查终端交互界面。LM Studio 的默认模型名 `local-model`，你可以在启动本地服务器后通过 API 查询确切的 ID。

推荐优先配置 Gemini，等云端模型跑通后，再尝试接入本地 LM Studio，这样排查问题会更清晰～






# 一、模型更改方法

是的，你可以随时在这段配置代码的 `models` 数组里更换或添加 Gemini 模型。比如，想把默认的 `gemini-2.5-pro` 换成 `gemini-1.5-flash`，只需修改对应条目：

```json
"models": ["gemini-1.5-flash", "gemini-2.0-flash-exp"]
```

### 📝 具体操作建议

1. **修改模型名称**：在 `"Gemini"` 这个 Provider 的 `"models"` 数组中，直接替换或增加你想用的模型标识符（可从 [Google Gemini 文档](https://ai.google.dev/models/gemini) 获取最新模型 ID）。
2. **同步修改默认路由（可选）**：如果希望 `claude` 命令默认就使用新模型，记得同时更新 `"Router"` 下的 `"default"` 字段，例如：
   ```json
   "Router": { "default": "Gemini,gemini-1.5-flash" }
   ```
3. **生效方式**：修改保存 `ccr-config.json` 后，**无需重启 Claude Code Router**，下一次调用 `claude` 命令时会自动读取最新配置。但如果你已经在一个持续对话中，可能需要重新启动对话才能生效。

### ⚠️ 注意事项
- 确保你填入的模型名称是 **Google 官方支持的、你 API 密钥有权限访问** 的模型（例如 `gemini-2.5-pro` 可能需要白名单，请先在 Gemini CLI 或 AI Studio 中测试能否正常调用）。
- LM Studio 的 `models` 数组同理，可以换成你本地实际加载的模型名称（通常可在 LM Studio 的开发者面板看到）。

总之，随时编辑这个 JSON 文件即可灵活控制可用模型列表，非常方便。


# 三、模型更改文件位置

根据最新的文档查询，你之前参照的配置文件路径可能有些更新了，`ccr-config.json` 这个文件名现在已不再使用。

目前 **Claude Code Router (CCR)** 的配置文件路径和名称统一为：**`~/.claude-code-router/config.json`**。其中，波浪线 `~` 代表你的用户主目录。

### 📂 如何找到并编辑这个文件？

在 `macOS` 系统中，你可以通过以下三种方法之一找到它：

*   **方法一：终端命令行（最直接）**
    打开终端（Terminal.app），输入以下命令后回车：
    ```bash
    open ~/.claude-code-router
    ```
    这个命令会直接在“访达”（Finder）中打开配置文件夹，你就能看到 `config.json` 文件了。

*   **方法二：用文本编辑器直接打开**
    如果你习惯使用终端下的 `nano` 或 `vim` 等文本编辑器，也可以直接运行：
    ```bash
    nano ~/.claude-code-router/config.json
    ```

*   **方法三：手动在访达中查找**
    1.  打开“访达”（Finder）。
    2.  在屏幕顶部的菜单栏中，点击 **前往** -> **前往文件夹...**（或按快捷键 `Shift + Command + G`）。
    3.  在弹出的窗口中输入 `~/.claude-code-router` 并点击“前往”。

