---
标题: DaVinci Resolve + Claude Code MCP 桥接配置指南
创建时间: 2026-08-05 10:24
修改时间: 2026-08-05 12:00
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---



# DaVinci Resolve 免费版 + Claude Code MCP 桥接配置指南

> 基于 macOS (Apple Silicon) + DaVinci Resolve 21.0.3 免费版 + resolve-mcp 成功经验总结  
> 最后更新：2026年8月5日


## 📋 目录

1. [概述与架构](#概述与架构)
2. [准备工作](#准备工作)
3. [安装桥接器](#安装桥接器)
4. [启动桥接器](#启动桥接器)
5. [安装 MCP 服务器](#安装-mcp-服务器)
6. [配置 Claude Code](#配置-claude-code)
7. [日常使用流程](#日常使用流程)
8. [常用操作示例](#常用操作示例)
9. [常见问题与排错](#常见问题与排错)
10. [关键经验复盘](#关键经验复盘)
11. [安全与权限设置](#安全与权限设置)


## 概述与架构

### 完整数据流

```
DaVinci Resolve 免费版
        ↓
  桥接器 (resolve_bridge_server.py)
  作用：在达芬奇内部运行，暴露控制接口
        ↓
  MCP 服务器 (resolve-mcp)
  作用：将 Claude Code 指令翻译为桥接器命令
        ↓
  Claude Code (终端版)
  作用：接收自然语言指令，调用 MCP 工具
```

### 组件说明

| 组件 | 位置 | 作用 |
| :--- | :--- | :--- |
| **桥接器** | `davinci-resolve-free-mcp-bridge/` | 在达芬奇内部运行，使外部可控制 |
| **MCP 服务器** | 虚拟环境 `venv/bin/resolve-mcp` | Claude Code 与桥接器的翻译层 |
| **Claude Code** | 全局安装 | 终端 AI 助手，接收自然语言指令 |


## 准备工作

### 系统要求

- macOS (Apple Silicon / Intel)
- DaVinci Resolve 免费版 已安装
- 网络连接（用于下载依赖）

### 1. 安装 Homebrew（如未安装）

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 2. 安装 Python 3.11+

> ⚠️ **重要**：不要使用 Python 3.14（开发版），某些依赖可能不兼容。推荐 Python 3.11 或 3.12。

```bash
brew install python@3.11
```

验证安装：

```bash
python3.11 --version
```

### 3. 安装必要系统工具

```bash
brew install ffmpeg
brew install git
```


## 安装桥接器

### 1. 克隆项目仓库

```bash
cd ~
git clone https://github.com/Andreansx/davinci-resolve-free-mcp-bridge.git
cd davinci-resolve-free-mcp-bridge
```

### 2. 运行安装脚本

```bash
./install.sh
```

**成功输出示例**：
```
==> 1. copy bridge files -> /Users/xxx/.resolve-free-bridge
==> 2. install shim over Blackmagic's module (with one-time backup)
==> 3. deploy menu script -> Workspace -> Scripts
Done.
```

### 3. 验证桥接器文件

```bash
ls -la "/Library/Application Support/Blackmagic Design/DaVinci Resolve/Fusion/Scripts/Utility/"
```

应该能看到 `resolve_bridge_server.py`。


## 启动桥接器

> ⚠️ **每次重启达芬奇后都需要重新执行此步骤**

### 方法一：通过菜单启动（推荐）

1. 打开 DaVinci Resolve，加载任意项目
2. 顶部菜单栏：**Workspace → Scripts → resolve_bridge_server**
3. 会弹出黑色终端窗口，**保持开启不要关闭**

### 方法二：通过控制台启动（备用）

1. 打开 **Workspace → Console**
2. 点击底部切换到 **Py3** 模式
3. 执行：

```python
exec(open('/Library/Application Support/Blackmagic Design/DaVinci Resolve/Fusion/Scripts/Utility/resolve_bridge_server.py').read())
```

### 验证桥接器是否运行

在**新终端**中执行：

```bash
PYTHONPATH="/Library/Application Support/Blackmagic Design/DaVinci Resolve/Developer/Scripting/Modules" python3 "/Users/eric/.resolve-free-bridge/test_connection.py"
```

**成功输出**：
```
CONNECTED: DaVinci Resolve 21.0.3.7
Current project : 你的项目名
```


## 安装 MCP 服务器

### 1. 进入项目目录并创建虚拟环境

```bash
cd ~/davinci-resolve-free-mcp-bridge
python3.11 -m venv venv
source venv/bin/activate
```

> 终端提示符前出现 `(venv)` 表示成功激活。

### 2. 安装 resolve-mcp

```bash
pip install resolve-mcp
```

### 3. 测试 MCP 服务器

```bash
resolve-mcp
```

**成功输出**：
```
Starting MCP server 'resolve-mcp' with transport 'stdio'
```

> 此命令会占用终端，按 `Ctrl+C` 可停止。


## 配置 Claude Code

### 1. 获取虚拟环境中的 resolve-mcp 路径

```bash
which resolve-mcp
```

输出类似：
```
/Users/eric/davinci-resolve-free-mcp-bridge/venv/bin/resolve-mcp
```

### 2. 添加 MCP 服务器到 Claude Code

```bash
claude mcp add resolve-mcp -- /Users/eric/davinci-resolve-free-mcp-bridge/venv/bin/resolve-mcp
```

### 3. 验证配置

```bash
claude mcp list
```

应该能看到 `resolve-mcp` 状态为 `✓ Connected`，并显示工具数量（如 `211 tools`）。

### 4. 重启 Claude Code

完全退出后重新启动：

```bash
claude
```

### 5. 测试连接

在 Claude Code 中输入：

```
获取当前达芬奇项目名称
```

**成功返回**：
```
当前达芬奇项目名称是 “新建项目 1”
```


## 日常使用流程

每次使用达芬奇 + Claude Code 控制时，按以下顺序操作：

### 步骤 1：启动达芬奇并打开项目

### 步骤 2：启动桥接器

菜单栏：**Workspace → Scripts → resolve_bridge_server**

保持黑色窗口开启。

### 步骤 3：确保 MCP 服务器运行（PM2 托管）

```bash
pm2 list | grep resolve
```

状态应为 `online`。如果未运行：

```bash
pm2 start resolve-mcp
```

### 步骤 4：使用 Claude Code

打开终端：

```bash
claude
```

输入自然语言指令控制达芬奇。


## 常用操作示例

### 项目管理

| 操作 | 指令示例 |
| :--- | :--- |
| 获取项目名称 | `获取当前达芬奇项目名称` |
| 重命名项目 | `将当前项目重命名为“我的漫画测试”` |
| 创建新项目 | `创建一个名为“新项目”的达芬奇项目` |

### 媒体导入

| 操作 | 指令示例 |
| :--- | :--- |
| 导入视频到媒体池 | `导入视频 /Users/eric/Movies/video.mp4` |
| 导入到当前时间线 | `将 /path/to/video.mp4 导入到当前时间线` |

> 💡 **关键技巧**：指令要精确。例如“导入视频 /path/to/file.mp4”比“将视频导入到达芬奇”更容易被正确解析。

### 时间线操作

| 操作 | 指令示例 |
| :--- | :--- |
| 列出所有时间线 | `列出当前项目中的所有时间线` |
| 创建时间线 | `创建一个名为“第一集”的新时间线` |
| 切换时间线 | `切换到名为“第一集”的时间线` |

### 组合操作

```
将当前项目重命名为“我的漫画测试”，然后将 /Users/eric/Movies/video.mp4 导入到媒体池
```

> ⚠️ **注意**：组合操作可能被拆分为多个工具调用，如果某一步失败，请拆分为单步指令重试。


## 常见问题与排错

### 问题 1：菜单中找不到 resolve_bridge_server

**原因**：安装脚本未正确部署。

**解决**：
```bash
sudo cp ~/.resolve-free-bridge/resolve_bridge_server.py "/Library/Application Support/Blackmagic Design/DaVinci Resolve/Fusion/Scripts/Utility/"
```

重启达芬奇。

### 问题 2：控制台 Py3 提示 "Python 3 was not found"

**原因**：达芬奇在默认路径找不到 Python 3。

**解决**：
```bash
sudo ln -sf /opt/homebrew/bin/python3.11 /usr/local/bin/python3
```

重启达芬奇。

### 问题 3：test_connection.py 返回 NOT CONNECTED

**原因**：桥接器未运行。

**解决**：
1. 确认达芬奇已打开并加载项目
2. 重新在菜单中执行 `resolve_bridge_server`
3. 确认黑色窗口保持开启

### 问题 4：Claude Code 提示 "Unknown skill: resolve-mcp"

**原因**：MCP 服务器未正确加载。

**解决**：
1. 确认 `resolve-mcp` 正在运行（`pm2 list`）
2. 重启 Claude Code
3. 重新添加 MCP 配置：
   ```bash
   claude mcp remove resolve-mcp -s local
   claude mcp add resolve-mcp -- /path/to/venv/bin/resolve-mcp
   ```

### 问题 5：resolve-mcp 启动报错 No module named 'mcp'

**原因**：虚拟环境中缺少依赖。

**解决**：
```bash
cd ~/davinci-resolve-free-mcp-bridge
source venv/bin/activate
pip install mcp resolve-mcp
```

### 问题 6：视频导入失败

**可能原因**：
- 文件路径不存在
- 指令表述不够精确
- 达芬奇中未加载项目

**解决**：
1. 确认文件存在：`ls -la "/path/to/file.mp4"`
2. 使用精确指令：`导入视频 /完整/路径/文件.mp4`
3. 先单独执行导入，不要与其他操作组合
4. 查看日志：`pm2 logs resolve-mcp --lines 30`

### 问题 7：PM2 管理的 resolve-mcp 反复崩溃

**原因**：PM2 默认用 Node.js 解析 Python 脚本。

**解决**：使用配置文件启动（见第 6 节 PM2 配置）。


## 关键经验复盘

### 踩过的坑与教训

#### 失误 1：混淆了 `davinci-resolve-mcp` 和 `resolve-mcp`

- **问题**：`davinci-resolve-mcp` 是为付费版设计的，在免费版中无法工作。
- **教训**：免费版用户应使用 `resolve-mcp`（注意名称不带 `davinci-` 前缀）。

#### 失误 2：使用 `uvx` 临时运行导致依赖丢失

- **问题**：`uvx` 每次创建临时环境，依赖不持久。
- **教训**：使用 `python -m venv` 创建持久虚拟环境，用 `pip` 安装。

#### 失误 3：Python 版本过新（3.14）

- **问题**：Python 3.14 是开发版，某些包不兼容。
- **教训**：使用 Python 3.11 或 3.12 LTS 版本。

#### 失误 4：分不清两个目录的作用

- **问题**：`davinci-resolve-free-mcp-bridge/` 是项目源码，`.resolve-free-bridge/` 是运行时文件。
- **教训**：源码目录用于开发和安装，运行时目录由 `install.sh` 自动管理。

#### 失误 5：在错误的环境执行命令

- **问题**：有时在全局环境而非虚拟环境中操作。
- **教训**：执行任何 `pip install` 前，确保终端提示符前有 `(venv)`。

#### 失误 6：指令表述不够精确导致工具未触发

- **问题**：模糊的指令（如“将视频导入”）可能不会被正确解析。
- **教训**：使用明确的动词 + 对象格式，如“导入视频 /完整/路径/文件.mp4”。


## 安全与权限设置

### 关闭工具确认提示（可选）

每次调用 MCP 工具时，Claude Code 会弹出确认提示。可以选择以下任一方式关闭：

**方式一：针对单个工具（推荐）**

当弹出确认提示时，选择：
```
2. Yes, and don't ask again for resolve-mcp - <工具名> commands in /Users/eric
```

**方式二：全局关闭**

在 Claude Code 中执行：
```
/settings set toolUseConfirmation off
```

**方式三：启动参数**

```bash
claude --dangerously-skip-permissions
```

> ⚠️ 建议仅放行只读操作（如列出时间线），保留修改操作的确认提示。


## 附录：文件结构总览

```
~/davinci-resolve-free-mcp-bridge/          # 项目根目录（必须保留）
├── venv/                                   # 虚拟环境（必须保留）
│   └── bin/
│       └── resolve-mcp                     # MCP 服务器可执行文件
├── resolve_bridge_server.py               # 桥接器核心脚本
├── test_connection.py                     # 连接测试脚本
├── install.sh                             # 安装脚本
└── uninstall.sh                           # 卸载脚本

~/.resolve-free-bridge/                    # 运行时目录（必须保留）
├── resolve_bridge_server.py               # 桥接器副本
└── test_connection.py                     # 测试脚本副本

/Library/Application Support/Blackmagic Design/DaVinci Resolve/Fusion/Scripts/
├── Utility/
│   └── resolve_bridge_server.py           # 达芬奇菜单中的桥接器
├── Edit/
│   └── resolve_bridge_server.py           # 桥接器副本（菜单显示）
├── Comp/
│   └── resolve_bridge_server.py           # 桥接器副本（菜单显示）
└── Color/
    └── resolve_bridge_server.py           # 桥接器副本（菜单显示）
```


## 快速命令速查

```bash
# 进入项目并激活虚拟环境
cd ~/davinci-resolve-free-mcp-bridge
source venv/bin/activate

# PM2 管理 resolve-mcp
pm2 start resolve-mcp          # 启动
pm2 stop resolve-mcp           # 停止
pm2 restart resolve-mcp        # 重启
pm2 list                       # 查看状态
pm2 logs resolve-mcp           # 查看日志

# 测试桥接器连接
PYTHONPATH="/Library/Application Support/Blackmagic Design/DaVinci Resolve/Developer/Scripting/Modules" python3 ~/.resolve-free-bridge/test_connection.py

# 配置 Claude Code MCP
claude mcp add resolve-mcp -- ~/davinci-resolve-free-mcp-bridge/venv/bin/resolve-mcp

# 查看 MCP 列表
claude mcp list

# 移除 MCP 配置
claude mcp remove resolve-mcp -s local
```


## 最终确认清单

- [ ] 桥接器已安装（`./install.sh` 成功）
- [ ] 达芬奇菜单中出现 `resolve_bridge_server`
- [ ] `test_connection.py` 返回 `CONNECTED`
- [ ] 虚拟环境已创建并安装 `resolve-mcp`
- [ ] PM2 管理 `resolve-mcp` 状态为 `online`
- [ ] Claude Code 中 `claude mcp list` 显示 `resolve-mcp ✓ Connected`
- [ ] Claude Code 能成功获取达芬奇项目名称
- [ ] Claude Code 能成功导入视频到媒体池

---

> **最后建议**：将本指南保存为 `DAVINCI_MCP_GUIDE.md`，遇到问题时可按步骤逐项排查。