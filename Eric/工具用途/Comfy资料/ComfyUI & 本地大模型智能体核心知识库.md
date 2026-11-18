---
标题: ComfyUI & 本地大模型智能体 核心知识库
创建时间: 2026-11-18 21:54
修改时间: 2026-11-18 21:54
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---
这份总结为你梳理了本次对话中涉及的所有 ComfyUI 核心知识点、避坑指南、模型参数与底层架构。你可以直接将其收藏，作为你 Mac 本地部署的**“百科全书”**。

---

# 🛠️ ComfyUI & 本地大模型智能体 核心知识库

## 一、 ComfyUI 智能体 (Agent) 集成架构

### 1. `comfyui-mcp` 插件的本质
*   **定位**：它不是一个单纯的模型，而是一个驻扎在 ComfyUI 内部的全功能智能体（Agent）。它提供“手脚”（操作节点工具），而“大脑”由外部大模型提供。
*   **控制方式**：通过 **MCP (Model Context Protocol)** 协议，实时读取画布状态并直接修改节点连线，无需读写本地 JSON 文件。
*   **后端选择器 (Backend Picker)**：
    *   **官方通道**：Claude、ChatGPT (Codex)、Gemini（直接利用你的订阅额度，无需 API Key 配置）。
    *   **本地/自定义通道**：Ollama、LM Studio 或 **Custom Endpoint**（直连本地端口或 Magpie 网关）。

### 2. 外部模型的三种接入方式
*   **官方 API**：在面板 `API Keys` 中填入密钥（如 OpenRouter、Kimi、GLM、DeepSeek）。
*   **Custom Endpoint（自定义端点）**：在设置面板中填入 `Base URL`（如 `http://127.0.0.1:3425/v1`）和 API Key。适合直连 Magpie 网关或 LM Studio。
*   **路由组 (Routing Group)**：在 Magpie 中创建，ComfyUI 中将模型名设为 `group/<组ID>`。可实现多个模型（如 DeepSeek、Kimi）的自动负载均衡和故障转移。

### 3. 智能体行为控制 (System Prompts)
*   **入口**：面板的 `Prompts` 或 `Settings` 页面。
*   **配置对象**：修改 “Panel agent persona (all backends)” 文本框（保存在 `~/.comfyui-mcp/panel-prompts.json`）。
*   **避坑指南**：系统内置提示词包含操作 ComfyUI 的核心说明书，**切忌直接覆盖清空**。如需追加约束（如“不要反复检查 UI 节点”），请先复制默认内容，再在其后追加你的规则。
*   **上下文超限 (Token 压缩)**：对话过长时，系统会压缩旧记录并提示。可通过设置环境变量 `COMFYUI_MCP_OLLAMA_NUM_CTX="131072"` 或开新对话解决。

### 4. macOS 终极部署方案 (PM2 管控)
*   **痛点**：官方自带的 `launcher` 自动拉起机制在 Mac 上常因环境变量（找不到 npx 路径）或权限问题失效。
*   **解法**：放弃 `launcher`，全局安装后用 PM2 统一管理。
    ```bash
    # 1. 全局安装
    npm install -g comfyui-mcp@latest
    # 2. 清理旧的启动器
    comfyui-mcp launcher uninstall
    ```
*   **优雅的 `.zshrc` 别名**：
    ```bash
    # 一键启动（防乱码/防路径丢失）
    alias comfy='pm2 start /Users/eric/projects/ComfyUI/venv/bin/python3 --name comfyui -- /Users/eric/projects/ComfyUI/main.py --enable-assets && pm2 start "comfyui-mcp connect" --name comfyui-mcp'
    # 一键全部关闭
    alias comfy-stop='pm2 stop comfyui comfyui-mcp'
    ```

---

## 二、 ComfyUI 核心机制与排错

### 1. 资产系统 (Media Assets) 丢失问题
*   **原因**：默认启动时仅为临时读取，不建立持久化数据库，重启后清空。
*   **解决**：启动命令必须加上 `--enable-assets` 参数。
*   **数据位置**：
    *   物理文件：`/Users/eric/projects/ComfyUI/output/`
    *   索引数据库：`/Users/eric/projects/ComfyUI/user/comfyui.db`

### 2. UI-only 节点检查死循环 (Tool Loop)
*   **报错现象**：`Checked UI-only node types — 4 types` 反复刷屏。
*   **原因**：Agent 在操作前检查 4 种前端专用节点（Note, MarkdownNote, Reroute, PrimitiveNode），小模型因逻辑能力不足陷入死循环，白白消耗 Token。
*   **解决**：换用大模型（如 DeepSeek V3、Claude 3.5、GPT-4o），或通过 System Prompt 约束，或直接开新对话。

### 3. 端口占用冲突
*   MCP 默认通信端口为 `9199`。如果面板一直连接失败，可通过终端 `lsof -nP -iTCP:9199 -sTCP:LISTEN` 检查是否被占用（如 Logitech G HUB 的 `9180` 端口）。
*   可在面板 `Advanced` 中将 `Bridge URL` 强制改为 `ws://127.0.0.1:9199`。

---

## 三、 核心模型解析与文件部署

### 1. 图像模型：Qwen-Image 2.1 (阿里通义千问团队)
*   **特点**：主打图像生成与排版，**文字渲染**（中英文）能力极强。
*   **Mac 适配**：截图中的工作流使用了全套 **INT8 量化版本**，总占用约 26GB（32GB 内存可跑，但需分阶段加载防 Swap）。
*   **组件参数**：
    *   `unet_name`: `qwen_image_2.1_int8_convrot.safetensors` (约 7.26 GB)
    *   `clip_name`: `qwen3vl_8b_int8_convrot.safetensors` (约 9.35 GB)
    *   `vae_name`: `qwen_image_2.1_vae_bf16.safetensors` (约 676 MB)
    *   `PE_model`: `qwen3.5_9b...int8...` (约 9 GB)

### 2. 视频模型对比
*   **Wan 2.1 (14B - 阿里云)**：
    *   **地位**：VBench 霸榜，画质极佳。
    *   **Mac 适配**：必须用 **GGUF 量化版**（配合 ComfyUI-GGUF 节点）才能跑。32GB Mac 生成 5 秒 480P 视频约需 30 分钟。可配合 **PhobosStream** 插件实现零显存流式加载。
*   **LTX-Video 2B (Lightricks/LTX)**：
    *   **地位**：极致的轻量化，适合快速迭代。
    *   **版本差异**：`dev`（质量优先，需多步推理）、`distilled`（蒸馏版，8步出图，需专用工作流）。
    *   **Mac 适配**：0.9.6 版本主模型需放在 `models/diffusion_models/`，而不是 checkpoints。
*   **MiniMax H3**：
    *   **特性**：占用极其夸张（数百 GB 磁盘），但依靠**流式加载**和 INT8 量化，32GB 内存可跑（峰值内存约 16.8GB）。
    *   **核心逻辑**：硬盘空间换内存。必须避免文本编码器和扩散模型同时驻留内存。

### 3. 文本编码器 (Text Encoders)
*   **T5-XXL (LTX/Flux 使用)**：
    *   体积巨大，精度决定大小：`FP16` (约 9.79 GB，质量最好)、`FP8` (约 4.89 GB，**Mac 首选**)、`GGUF` (约 5 GB)。
    *   存放路径：`models/text_encoders/`
    *   国内极速镜像（ModelScope）：`https://modelscope.cn/models/AI-ModelScope/flux_text_encoders/resolve/master/t5xxl_fp8_e4m3fn.safetensors`

---

## 四、 常用节点参数说明 (截图实例)

在截图提供的 KSampler（采样器）节点中：
*   **sampler (采样器)**：`euler`（最基础、速度快的采样算法）。
*   **scheduler (调度器)**：`simple`（控制去噪步数的噪声衰减曲线，simple 较为平缓）。
*   **seed (种子)**：`891836227295375`。固定种子可复现同一张图；旁边带有骰子图标点击可随机。
*   **dtype (数据类型)**：`default`。通常保持默认，若不支持半精度可在此切换（如 `fp32`，但会更慢）。
*   **cache_device (缓存设备)**：`auto`。让系统自动决定模型权重放在内存还是显存中。

---

## 💡 后期维护小贴士
1.  **大模型比小模型更省钱**：复杂任务（如生成工作流、修正连线）绝对不要用 `flash` 小模型，它陷入死循环烧掉的 Token 和花费的时间，远比用一次 Pro 模型多得多。
2.  **国内镜像下载慢**：浏览器单线程下载极易断流，强烈推荐下载 **Motrix** 这类支持多线程的下载工具，速度会有质的飞跃。
