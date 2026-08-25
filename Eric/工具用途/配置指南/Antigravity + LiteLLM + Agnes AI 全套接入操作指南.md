---
标题: Antigravity + LiteLLM + Agnes AI 全套接入操作指南
创建时间: 2026-08-25 23:25
修改时间: 2026-08-25 23:25
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 中等
状态: 已成功
---

# Antigravity + LiteLLM + Agnes AI 全套接入操作指南

本指南基于实际打通经验总结，旨在帮助你**从零开始**在 Antigravity 中通过 LiteLLM 调用 Agnes AI（或其他 OpenAI 兼容 API）的模型。文中包含详细配置、常见失误复盘及排障方法，可直接复用。

---

## 一、整体架构与原理

- **Antigravity**：IDE/CLI 工具，支持通过 **MCP（模型上下文协议）** 接入外部服务。
- **LiteLLM**：一个轻量级 API 网关，能将不同厂商的 API 格式（如 Anthropic、Gemini）转换为 OpenAI 兼容格式，反之亦然。**它充当 Antigravity 与 Agnes AI 之间的“翻译官”**。
- **Agnes AI**：提供 OpenAI 兼容的 API 接口（`https://apihub.agnes-ai.com/v1`），但可能因网络原因需要代理访问。

**数据流向**：  
`Antigravity` → `MCP Server (litellm-mcp-server)` → `本地 LiteLLM 服务` → `(代理) → Agnes AI`

---

## 二、前提条件

- 已安装 Antigravity（版本 ≥ 2.0）
- 已安装 Node.js（用于运行 `litellm-mcp-server`）
- 已安装 Python 环境，并全局安装 LiteLLM（`pip install 'litellm[proxy]'`）
- 拥有有效的 Agnes AI API Key
- 本地有可用的代理软件（如 Clash、V2Ray、Surge 等），且代理端口已知（本例假设为 `127.0.0.1:7890`）

---

## 三、配置步骤（按顺序执行）

### 步骤 1：编写 LiteLLM 配置文件 `config.yaml`

在任意目录（例如 `~/litellm/`）创建 `config.yaml`，内容如下：

```yaml
model_list:
  - model_name: agnes-2.5-flash          # 在 Antigravity 中显示的名称
    litellm_params:
      model: openai/agnes-2.5-flash       # LiteLLM 内部模型标识
      api_base: https://apihub.agnes-ai.com/v1
      api_key: "你的 Agnes AI API Key"    # 请替换为真实 Key

# 全局设置：自动丢弃模型不支持的参数（防止 Antigravity 传参导致报错）
litellm_settings:
  drop_params: true
```

**注意**：
- `model_name` 可以任意命名，但在 Antigravity 中引用时需使用此名称。
- 如果 Agnes AI 提供多个模型（如 `agnes-2.0-flash`），请根据实际情况修改 `model` 字段。
- 建议添加 `drop_params: true`，因为 Antigravity 可能会携带 `thinking`、`tool_choice` 等扩展参数，而 Agnes AI 可能不支持，开启后 LiteLLM 会自动忽略这些参数，避免报错。

---

### 步骤 2：启动 LiteLLM 服务（必须带代理环境变量）

由于 Agnes AI 的 API 地址（`apihub.agnes-ai.com`）需要代理才能访问，**必须在启动 LiteLLM 的终端中显式设置代理环境变量**。

```bash
# 进入配置文件所在目录
cd ~/litellm

# 启动 LiteLLM，假设代理监听在 127.0.0.1:7890
HTTP_PROXY=http://127.0.0.1:7890 \
HTTPS_PROXY=http://127.0.0.1:7890 \
NO_PROXY=localhost,127.0.0.1 \
litellm --config config.yaml --port 4000
```

**关键说明**：
- **`--port 4000`** 必须与后续 Antigravity MCP 配置中的端口一致。
- **`NO_PROXY=localhost,127.0.0.1`** 必不可少，它能确保 LiteLLM 在访问本地资源时**不走代理**，防止循环代理或连接失败。
- 如果代理端口不是 `7890`，请替换为实际端口（例如 `7897`、`1080`）。

启动后，终端应显示类似 `Server running on http://0.0.0.0:4000` 的日志，且无错误信息。如有 `ProxyError` 或 `Timeout`，请检查代理软件是否运行、端口是否正确。

---

### 步骤 3：配置 Antigravity 的 MCP 连接

Antigravity 通过 MCP 与 LiteLLM 通信，配置文件位于 `~/.gemini/config/mcp_config.json`（**正确路径**，部分旧文档可能提及别的路径，务必使用此路径）。

如果文件不存在，请手动创建。编辑内容如下：

```json
{
  "mcpServers": {
    "litellm": {
      "command": "npx",
      "args": [
        "-y",
        "litellm-mcp-server@latest"
      ],
      "env": {
        "LITELLM_BASE_URL": "http://localhost:4000",
        "LITELLM_API_KEY": "任意占位符（如 dummy）"
      }
    }
  }
}
```

**注意**：
- `LITELLM_BASE_URL` 必须与步骤 2 中启动的 LiteLLM 服务的地址和端口完全一致。
- `LITELLM_API_KEY` 如果 LiteLLM 服务未开启认证，可以随便填（如 `"dummy"`），不影响使用。
- 保存文件后，**完全退出并重新启动 Antigravity IDE**，以便 MCP 配置生效。

---

### 步骤 4：在 Antigravity 中验证与使用

1. 重启 Antigravity 后，打开右上角 `...` 菜单 → **MCP Store**。
2. 点击 **Refresh** 按钮，应能看到名为 `litellm` 的服务器，且显示 **“11 tools enabled”**（或其他工具数量）。
3. 在对话框输入 `@`，选择 `litellm` 服务器，然后输入你的问题（可明确指定模型，例如：`@litellm 使用 agnes-2.5-flash 写一个冒泡排序`）。
4. 如果一切正常，你将收到来自 Agnes AI 的回复。

---

## 四、常见问题与排障（经验复盘）

| 问题现象 | 可能原因 | 解决方案 |
|--------|--------|--------|
| **Antigravity 中 MCP 服务器未出现** | 配置文件路径不正确；JSON 格式错误 | 确认文件在 `~/.gemini/config/mcp_config.json`，并检查 JSON 语法（可用在线校验工具）。 |
| **MCP 服务器出现但工具为 0** | LiteLLM 服务未启动或端口不对；`LITELLM_BASE_URL` 配置错误 | 检查 LiteLLM 终端是否运行；确保 `BASE_URL` 端口与 `--port` 一致。 |
| **调用时报错 `Connection Error`** | LiteLLM 无法访问 Agnes AI（代理未生效） | 确认启动 LiteLLM 时是否带上了 `HTTP_PROXY`/`HTTPS_PROXY`；检查代理软件是否开启；尝试 `curl -x http://127.0.0.1:7890 https://apihub.agnes-ai.com/v1/models` 测试连通性。 |
| **调用时报错 `400 Bad Request` 或参数错误** | Antigravity 传递了 Agnes AI 不支持的参数 | 确保 `config.yaml` 中已设置 `litellm_settings: drop_params: true`，并重启 LiteLLM。 |
| **Antigravity 访问本地 LiteLLM 极慢或超时** | Antigravity 的全局代理设置也代理了 `localhost` | 在 Antigravity 的设置（`Cmd + ,`）中，将 `localhost`、`127.0.0.1` 加入 `NO_PROXY` 或 `http.noProxy` 列表。 |
| **LiteLLM 启动后访问 Agnes AI 返回 `401 Unauthorized`** | API Key 错误或已过期 | 重新核对 `config.yaml` 中的 `api_key`；尝试在 Agnes AI 官网重新生成 Key。 |

---

## 五、核心注意事项总结

1. **路径正确性**：Antigravity 的 MCP 配置文件务必放在 `~/.gemini/config/mcp_config.json`。
2. **代理三要素**：
   - 启动 LiteLLM 时必须添加 `HTTP_PROXY`、`HTTPS_PROXY`。
   - 务必设置 `NO_PROXY=localhost,127.0.0.1` 避免本地通信被代理。
   - Antigravity IDE 本身的网络设置（如果有）需排除 localhost。
3. **参数兼容性**：始终开启 `drop_params: true`，这是保证 Antigravity 与 Agnes AI 畅通交互的关键。
4. **端口统一**：LiteLLM 启动端口（`--port`）与 MCP 配置的 `BASE_URL` 端口必须相同。
5. **重启生效**：修改任何配置文件或启动参数后，均需重启对应的服务（LiteLLM 或 Antigravity）。

---

## 六、后续扩展

此套流程同样适用于接入其他 OpenAI 兼容的 API（如 DeepSeek、Moonshot 等），只需修改 `config.yaml` 中的 `api_base`、`api_key` 和 `model` 字段即可。如果目标 API 无需代理，则直接启动 LiteLLM（不加代理变量）即可。

---

通过以上指南，你可以稳定地在 Antigravity 中调用 Agnes AI 模型，享受更灵活的开发体验。如果在实施过程中遇到其他问题，请参考 LiteLLM 官方文档或查阅终端日志进行定位。