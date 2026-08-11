---
标题: " Claude Code + Blender 自然语言控制 配置指南"
创建时间: 2026-08-11 17:42
修改时间: 2026-08-11 17:42
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 中等
状态: 已成功
---

# Claude Code + Blender 自然语言控制 配置指南

本指南基于 macOS（M系列芯片）环境，使用 Claude Code（命令行版本）和 **blend-ai** MCP 服务器，实现通过自然语言指令操控 Blender 进行 3D 建模、材质、动画等操作。

---

## 一、工具简介

| 工具 | 作用 |
|------|------|
| **Claude Code** | Anthropic 推出的命令行 AI 编程助手，支持 MCP（Model Context Protocol）扩展，可调用外部工具。 |
| **blend-ai** | 一个基于 MCP 的服务器，提供 164+ 个工具，连接 Blender 并执行 Python 代码，实现 3D 场景的自动创建与编辑。 |
| **Blender** | 开源 3D 创作套件，支持 Python 脚本（bpy），作为最终执行环境。 |
| **uv** | 极速 Python 包管理工具，用于安装和管理 blend-ai 依赖。 |

> **核心通信链路**：  
> 用户 → Claude Code（自然语言）→ blend-ai MCP 服务器（通过 stdio）→ Blender 插件（WebSocket 9876 端口）→ Blender（执行 bpy 代码）

---

## 二、前置条件

- **操作系统**：macOS（本指南基于 M2/M3 芯片，Intel 通用）
- **已安装**：
  - **Blender 4.2 或更高版本**（[下载](https://www.blender.org/download/)）
  - **Node.js**（[下载](https://nodejs.org/)）
  - **uv**（Python 包管理）：  
    `curl -LsSf https://astral.sh/uv/install.sh | sh`
  - **Claude Code**（全局安装）：  
    `npm install -g @anthropic-ai/claude-code`  
    （确保版本 ≥ 2.0，支持 MCP）
- **网络**：能访问 GitHub 及 Python 包仓库。

---

## 三、安装步骤

### 1. 克隆 blend-ai 项目

```bash
cd ~/projects   # 或你的工作目录
git clone https://github.com/HoldMyBeer-gg/blend-ai.git
cd blend-ai
```

### 2. 安装 Python 依赖（使用 uv）

```bash
uv venv                 # 创建虚拟环境
uv pip install -e .     # 可编辑安装
```

> 如果提示虚拟环境不存在，先执行 `uv venv`。

### 3. 安装 Blender 插件

- 从 [blend-ai Releases](https://github.com/HoldMyBeer-gg/blend-ai/releases) 下载最新的 `.zip` 插件包。
- 在 Blender 中：`编辑` → `偏好设置` → `插件` → `安装...`，选择该 `.zip` 文件并启用。
- 启用后，在 3D 视图中按 `N` 键，侧边栏会出现 **“blend-ai”** 标签页。

### 4. 启动 blend-ai 服务器（Blender 侧）

- 在 Blender 的 blend-ai 面板中，点击 **“Start Server”**，确保状态变为 **“Running”**（默认端口 9876）。

> 注意：不要关闭 Blender，后续所有操作都需保持运行。

---

## 四、配置 Claude Code MCP 服务器

**关键步骤：使用 `claude mcp add` 命令（官方推荐，避免手工编辑配置文件的坑）**

在终端中执行以下命令：

```bash
claude mcp add -s user blend-ai \
  -e BLENDER_HOST=127.0.0.1 \
  -e BLENDER_PORT=9876 \
  -- /opt/homebrew/bin/uv --quiet --directory /Users/eric/projects/blend-ai run blend-ai
```

### 参数说明

| 部分 | 含义 |
|------|------|
| `claude mcp add -s user` | 将服务器添加到用户级配置（`~/.claude/settings.json`） |
| `blend-ai` | 服务器名称（在 `/mcp` 中显示） |
| `-e BLENDER_HOST=127.0.0.1` | 环境变量，告诉服务器 Blender 插件的 IP |
| `-e BLENDER_PORT=9876` | 环境变量，指定 WebSocket 端口 |
| `--` | 分隔符，后面是实际启动命令 |
| `/opt/homebrew/bin/uv` | `uv` 的完整路径（通过 `which uv` 获取） |
| `--quiet` | 减少日志输出，避免干扰 stdio 通信 |
| `--directory /Users/eric/projects/blend-ai` | 工作目录，必须指向 blend-ai 项目根目录 |
| `run blend-ai` | 执行 MCP 服务器 |

> **注意**：  
> - 将 `/opt/homebrew/bin/uv` 替换为你 `which uv` 的实际输出路径。  
> - 将 `/Users/eric/projects/blend-ai` 替换为你的实际克隆路径。

执行后，Claude Code 会自动将配置写入 `~/.claude/settings.json`（或其他用户级配置），无需手动编辑。

---

## 五、验证连接

1. **确保 Blender 中的 blend-ai 服务器处于 “Running” 状态**。
2. 启动 Claude Code：`claude`（或你自定义的命令）。
3. 在 Claude Code 会话中输入 `/mcp`，应该看到：
   ```
   blend-ai · connected · 211 tools
   ```
4. 测试一条简单指令：
   > “在 Blender 场景中心创建一个红色立方体。”

如果 Blender 中出现了红色立方体，则配置成功！

---

## 六、常见问题与解决方案

### 6.1 工作区未信任，MCP 服务器未加载

**现象**：`/mcp` 显示 “No MCP servers configured” 或只显示旧服务器。

**原因**：Claude Code 需要明确信任当前工作目录才能加载 MCP 配置。

**解决**：首次运行 `claude` 时，会弹出信任对话框，选择 **“Yes, I trust this folder”**。

---

### 6.2 配置文件位置错误

**现象**：手动编辑 `~/.claude.json` 或 `~/.claude/mcp-servers.json` 后不生效。

**原因**：Claude Code 优先读取 `~/.claude/settings.json`（用户级）和项目级 `./.claude/settings.local.json`。

**解决**：使用 `claude mcp add` 命令自动写入正确位置，避免手工编辑。

---

### 6.3 `uv` 命令找不到

**现象**：在 Claude Code 中启动 MCP 服务器时报错 “uv: command not found”。

**解决**：在 `mcp add` 命令中使用 `uv` 的完整绝对路径（如 `/opt/homebrew/bin/uv`）。

---

### 6.4 端口冲突

**现象**：同时启动多个 MCP 服务器（如 Blender MCP 和 blend-ai）时，可能出现端口占用。

**解决**：只使用 blend-ai（它已包含 Blender MCP 的所有功能），不要同时启动两个桥接服务。

---

### 6.5 Blender 插件未显示 “Start Server” 按钮

**原因**：插件安装后未启用，或 Blender 版本不兼容。

**解决**：在偏好设置中确认插件已勾选启用，并确保 Blender ≥ 4.2。

---

### 6.6 模型创建后视口看不到

**可能原因**：
- 物体太小或相机位置偏移。
- 显示模式不正确（需切换到 “材质预览” 或 “渲染” 模式才能看到材质）。

**解决方法**：
- 在 Outliner 中选中物体，按 `Cmd + .`（Mac）或 `View` → `Frame Selected` 聚焦。
- 按 `Shift + C` 重置游标并居中视图。
- 调整物体缩放或相机位置（可通过脚本或手动）。

---

### 6.7 骨骼绑定失败（自动权重不生效）

**原因**：当前 blend-ai 工具集未直接提供“自动权重绑定”工具，需要手动执行或通过脚本。

**解决**：在 Blender 中手动绑定：
1. 选中模型，再按住 Shift 选中骨骼。
2. 按 `Ctrl + P` → `Automatic Weights`。

或执行以下 Python 脚本（需根据实际名称调整）：

```python
import bpy
bpy.context.view_layer.objects.active = bpy.data.objects['MyDog']
bpy.data.objects['MyDog'].select_set(True)
bpy.context.view_layer.objects.active = bpy.data.objects['MyDogRig.002']
bpy.data.objects['MyDogRig.002'].select_set(True)
bpy.ops.object.parent_set(type='ARMATURE_AUTO')
```

---

## 七、工作流建议

- **分步执行**：复杂的 3D 任务（如“奔跑的小狗”）需要分解为建模→骨骼→绑定→动画→材质等步骤，逐步指令 Claude。
- **检查工具能力**：某些高级操作（如自动权重绑定）可能不在当前工具集中，可结合手动操作或 Python 脚本补充。
- **善用 `/mcp` 查看工具列表**：随时了解可用工具，以便精确指令。

---

## 八、总结

本指南基于实际踩坑经验，总结了从零配置 Claude Code + blend-ai 到成功操控 Blender 的完整流程。核心要点：

- **使用 `claude mcp add` 命令配置，避免手工编辑配置文件。**
- **务必设置环境变量 `BLENDER_HOST` 和 `BLENDER_PORT`。**
- **保持 Blender 中的 blend-ai 服务器为 “Running” 状态。**
- **首次启动信任工作区。**
- **复杂动画需结合手动操作。**

按此操作，你也能用自然语言驱动 Blender，大大提升 3D 创作效率。

---

*本文基于 macOS M系列芯片、Blender 5.2、Claude Code v2.1.199、blend-ai v1.3.0 实测有效。如有更新，请参考官方文档。*