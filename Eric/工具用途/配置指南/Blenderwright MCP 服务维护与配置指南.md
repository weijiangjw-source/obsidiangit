---
标题: Blenderwright MCP 服务维护与配置指南
创建时间: 2026-11-18 18:17
修改时间: 2026-11-18 18:17
引用渠道: gemini
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---

这是一份为你量身定制的 **Blenderwright (原 blend-ai) MCP 服务更新与配置操作指南**，方便你后期维护和重新部署时直接对照操作。

# 🛠️ Blenderwright MCP 服务维护与配置指南

## 一、 终端代码与 Python 依赖更新

当项目开源仓库有更新时，通过 Git 强制同步并使用 `uv` 刷新本地虚拟环境。

1. **打开终端并进入项目目录：**
    
    ```bash
    cd /Users/eric/projects/blend-ai
    ```
    
2. **执行强制同步与依赖更新：**
    
    ```bash
    git fetch origin
    git reset --hard origin/main
    uv sync
    ```
    

## 二、 Blender 内部插件更新

底层代码和依赖更新后，Blender 软件内的插件端也需要同步进行“干净重装”，避免旧缓存冲突。

1. **卸载旧版插件：**
    
    - 打开 Blender，进入 `Edit → Preferences → Add-ons`。
        
    - 找到旧版插件（`blend-ai` 或旧的 `blenderwright`），展开并点击 **Remove**（移除）。
        
    - **彻底重启一次 Blender**（清空 Python 缓存）。
        
2. **安装最新插件：**
    
    - 再次打开 `Add-ons`，点击右上角 **Install...**。
        
    - 定位到本地目录 `/Users/eric/projects/blend-ai`，选中最新的插件包进行安装并**勾选启用**。
        
3. **确认服务启动：**
    
    - 在 Blender 3D 视口面板中确认服务已成功运行（正常会显示 `**Server: Running (port 9876)**`）。
        

## 三、 Toonflow 客户端 MCP 挂载配置

由于 Toonflow 作为 macOS 桌面端 GUI 应用无法直接读取终端的环境变量，配置时必须使用 `**uv**` **的绝对路径**。

在 Toonflow 的 **“添加 MCP” → “快速创建”** 界面中，按如下规范填写：

- **名称 (Name)：** `blenderwright`
    
- **连接方式：** 本地 (`stdio`)
    
- **启动命令 (Command)：** `/opt/homebrew/bin/uv`
    
    > _注：若系统路径不同，可在终端运行 `which uv` 获取准确的绝对路径填入。_
    
- **参数 (Arguments) （每行一个）：**
    
    ```text
    --directory
    /Users/eric/projects/blend-ai
    run
    blenderwright
    ```
    
    _(注：根据后续具体的入口命令，可在 `run` 后面补充对应的启动参数或脚本)_
    
- **保存与生效：** 配置完成后点击 **“测试连接”**，通过后点击保存即可在下一轮 Agent 对话中无缝调用。