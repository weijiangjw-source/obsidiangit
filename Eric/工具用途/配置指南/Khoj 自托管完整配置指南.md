---
标题: Khoj 自托管完整配置指南
创建时间: 2026-08-06 22:52
修改时间: 2026-08-06 22:52
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 较难
状态: 已成功
---
# Khoj 自托管完整配置指南

> 本文档基于实际配置经验整理，涵盖从零开始部署 Khoj 自托管服务的完整流程，并记录了操作中遇到的所有关键问题与解决方案。


## 一、Khoj 是什么

Khoj 是一个开源的、可自托管的个人 AI 助手，核心目标是把你的所有个人知识库——包括 Obsidian 笔记、Markdown 文件、PDF 文档、图片等——变成一个你可以随时用自然语言对话的“第二大脑”。它的核心工作流程是：先对你的文档进行索引（Indexing），使用嵌入模型将文本转换成向量并存储在向量数据库中；当你提问时，系统在向量数据库中找出语义最相似的文本片段，作为上下文提交给大语言模型生成回答。

Khoj 支持两种运行模式：云服务（已停止运营）和自托管。本文档聚焦于**自托管模式**——所有数据留在本地，100% 隐私，无需联网即可使用核心功能。


## 二、环境准备

### 2.1 必要的环境依赖

在开始安装 Khoj 之前，确保你的 Mac 上已具备以下环境：

| 依赖项 | 用途 | 安装命令（如未安装） |
|--------|------|----------------------|
| **Homebrew** | macOS 包管理器 | `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"` |
| **Python 3.11 或 3.12** | Khoj 运行环境 | `brew install python@3.11` |
| **PostgreSQL 17** | 向量数据库（需 pgvector 扩展） | `brew install postgresql@17` |
| **pgvector** | 向量存储扩展 | `brew install pgvector` |
| **pipx** | Python 应用安装工具 | `brew install pipx` |

### 2.2 关键版本说明

Khoj 官方要求 Python 版本 >= 3.10，推荐使用 3.11 或 3.12。**Python 3.14 目前不被支持**，因为 PyTorch 等核心依赖尚未适配该版本，会导致依赖冲突（`ResolutionImpossible` 错误）。如果你的默认 Python 是 3.14，必须在安装时显式指定使用 3.11。

### 2.3 确认现有 Python 版本

```bash
ls /opt/homebrew/bin/python*
```

如果输出中包含 `python3.11`，说明你已具备合适的版本。如果只有 `python3.14`，需先安装 Python 3.11：

```bash
brew install python@3.11
```


## 三、PostgreSQL 数据库安装与配置

### 3.1 安装 PostgreSQL 和 pgvector

```bash
# 安装 PostgreSQL
brew install postgresql@17

# 安装 pgvector 扩展
brew install pgvector
```

安装过程中，Homebrew 会询问是否继续安装依赖（如 `krb5`），输入 `y` 确认即可。

### 3.2 启动 PostgreSQL 服务

```bash
brew services start postgresql@17
```

验证服务是否正常运行：

```bash
brew services list | grep postgresql
```

状态应显示为 `started`。

### 3.3 在数据库中启用 pgvector 扩展

```bash
psql -d postgres -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

如果提示权限错误，可指定用户：

```bash
psql -U eric -d postgres -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

### 3.4 创建数据库

```bash
createdb postgres
```

> **注意**：通过 Homebrew 安装 PostgreSQL 时，默认创建的超级用户是你的 macOS 用户名（如 `eric`），而非 `postgres`。这是后续配置环境变量时需要特别留意的关键点。


## 四、安装 Khoj

### 4.1 使用 pipx 安装（推荐）

```bash
pipx install khoj --python /opt/homebrew/bin/python3.11 --pip-args="-i https://pypi.tuna.tsinghua.edu.cn/simple"
```

**参数说明：**
- `--python /opt/homebrew/bin/python3.11`：指定使用 Python 3.11（避开不兼容的 3.14）
- `--pip-args="-i https://pypi.tuna.tsinghua.edu.cn/simple"`：使用清华镜像源加速下载

### 4.2 安装失败的常见原因与处理

**问题一：`ResolutionImpossible` 依赖冲突**

原因：使用了 Python 3.14。解决方案：如上所述，显式指定 Python 3.11。

**问题二：安装进度条长时间不动（卡在 `⣻ installing khoj`）**

原因：Khoj 依赖 PyTorch 等大型机器学习库（约 1-2GB），下载需要时间，但 `pipx` 不显示下载进度。解决方案：改用 `pip` + 虚拟环境方式安装，可以看到实时进度：

```bash
python3.11 -m venv ~/khoj-env
source ~/khoj-env/bin/activate
pip install khoj -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**问题三：`'khoj' already seems to be installed` 但命令找不到**

原因：`pipx` 创建了虚拟环境但符号链接未正确注册。解决方案：
```bash
pipx ensurepath --force
```
然后**完全关闭终端，重新打开**，再试 `khoj --help`。

如果仍无效，强制重装：
```bash
pipx install khoj --force --python /opt/homebrew/bin/python3.11 --pip-args="-i https://pypi.tuna.tsinghua.edu.cn/simple"
```

### 4.3 验证安装

```bash
khoj --help
```

如显示帮助信息，说明安装成功。


## 五、环境变量配置（关键步骤）

### 5.1 为什么需要环境变量

Khoj 默认尝试用 `postgres` 用户连接数据库，但 Homebrew 安装的 PostgreSQL 默认用户是你的 macOS 用户名（如 `eric`）。不设置环境变量会导致 `FATAL: role "postgres" does not exist` 错误。

### 5.2 必需的环境变量

每次启动 Khoj 前，需在终端设置以下变量：

```bash
export POSTGRES_USER=eric              # 替换为你的 macOS 用户名
export POSTGRES_PASSWORD=              # 留空（Homebrew 默认无密码）
export POSTGRES_HOST=localhost
export POSTGRES_PORT=5432
export POSTGRES_DB=postgres
export KHOJ_DOMAIN=localhost
export HF_ENDPOINT=https://hf-mirror.com   # Hugging Face 国内镜像加速
```

### 5.3 持久化配置（一劳永逸）

为避免每次打开新终端都要重新设置，将上述变量写入 `~/.zshrc`：

```bash
echo 'export POSTGRES_USER=eric' >> ~/.zshrc
echo 'export POSTGRES_PASSWORD=' >> ~/.zshrc
echo 'export POSTGRES_HOST=localhost' >> ~/.zshrc
echo 'export POSTGRES_PORT=5432' >> ~/.zshrc
echo 'export POSTGRES_DB=postgres' >> ~/.zshrc
echo 'export KHOJ_DOMAIN=localhost' >> ~/.zshrc
echo 'export HF_ENDPOINT=https://hf-mirror.com' >> ~/.zshrc
source ~/.zshrc
```


## 六、首次启动 Khoj

### 6.1 启动服务

```bash
khoj
```

首次启动时，Khoj 会：
1. 自动创建数据库表（Django 迁移）
2. 提示输入管理员邮箱和密码（用于访问 Web 管理后台 `http://localhost:42110/server/admin`）

### 6.2 配置 AI 模型

启动过程中会依次询问是否添加各种聊天模型（OpenAI、Gemini、Anthropic、本地模型等）。**如果你只把 Khoj 用作语义检索引擎，而不打算在 Web 界面聊天，可以全部输入 `n` 跳过。** 跳过后，Khoj 的语义搜索功能仍然完全可用。

### 6.3 模型下载问题

Khoj 需要下载嵌入模型（默认 `thenlper/gte-small`），约 400-500MB。如果下载卡住（`Connection to huggingface.co timed out`），是因为网络无法直接访问 Hugging Face。解决方案已在环境变量中设置了 `HF_ENDPOINT=https://hf-mirror.com`，会走国内镜像加速下载。

### 6.4 启动成功的标志

终端最后显示：

```
Uvicorn running on http://127.0.0.1:42110 (Press CTRL+C to quit)
```

> **重要**：这个终端窗口必须保持打开，Khoj 服务才能持续运行。关闭终端则服务停止。


## 七、配置数据源（让 Khoj 知道你的笔记在哪）

### 7.1 方法一：通过配置文件（推荐）

Khoj 的 Web 管理后台（`/server/admin`）可能没有 `File sources` 入口（访问 `/server/admin/database/filesource/` 可能返回 403）。最可靠的方式是编辑配置文件。

**创建配置文件：**

```bash
mkdir -p ~/.config/khoj
nano ~/.config/khoj/khoj.yml
```

**写入以下内容：**

```yaml
content-type:
  markdown:
    input-directories:
      - /Users/eric/Obsidian    # 替换为你的 Obsidian 仓库路径
    input-files: []
```

保存（`Ctrl+O` → 回车 → `Ctrl+X`）。

### 7.2 方法二：通过 `khoj configure` 命令

```bash
khoj configure
```

按提示输入 Markdown 文件目录、PDF 目录等。**注意**：在某些版本中，`khoj configure` 可能不会进入交互模式而是直接启动服务，此时应使用方法一。


## 八、连接 Obsidian 插件

### 8.1 安装插件

1. 打开 Obsidian → 设置 → 社区插件 → 浏览
2. 搜索 **Khoj**，点击安装并启用

### 8.2 配置插件

在 Obsidian 的 Khoj 插件设置中：

| 配置项 | 值 |
|--------|-----|
| **Khoj URL** | `http://127.0.0.1:42110` |
| **Khoj API Key** | **留空**（因为运行在匿名模式） |

> **重要**：API Key 必须完全为空，不能有空格或任何字符。如果填入了内容，插件会尝试认证并返回 403 错误。

### 8.3 排除文件夹（可选）

在插件设置中，可以排除不需要索引的文件夹（如 `Templates`、`Excalidraw` 等），以加快索引速度并减少噪音。

### 8.4 同步笔记

点击 **`Force Sync`** 按钮，Khoj 开始索引你的 Obsidian 仓库中的所有 Markdown 文件。


## 九、常见问题与解决方案

### 9.1 数据库连接失败：`FATAL: role "postgres" does not exist`

**原因**：未设置 `POSTGRES_USER` 环境变量，Khoj 默认尝试用 `postgres` 用户连接。

**解决**：在启动 Khoj 前设置 `export POSTGRES_USER=你的用户名`，或将其写入 `~/.zshrc`。

### 9.2 模型下载超时：`Connection to huggingface.co timed out`

**原因**：网络无法直接访问 Hugging Face。

**解决**：设置 `export HF_ENDPOINT=https://hf-mirror.com`，使用国内镜像。

### 9.3 Obsidian 插件报错 `Failed to clear existing content index`

**原因**：索引缓存损坏或状态不一致。

**解决步骤：**

1. 停止 Khoj 服务（`Ctrl+C`）
2. 删除索引目录：`rm -rf ~/.khoj`
3. 重置数据库：`dropdb postgres && createdb postgres`
4. 重新启动 Khoj（会提示重新创建管理员账户）
5. 在 Obsidian 中再次点击 `Force Sync`

### 9.4 Obsidian 插件显示“未连接”（状态不变绿）

**原因**：匿名模式下，插件请求 `/api/v1/user` 接口会返回 403，插件误认为未连接。**这 不影响核心功能**。

**验证服务是否正常**：在终端执行 `curl http://127.0.0.1:42110/api/v1/user`，如返回 `{"detail":"Forbidden"}`，说明服务正常运行。

**解决**：忽略状态指示灯，直接点击 `Force Sync` 即可。

### 9.5 Web 管理后台首页报错 500

**原因**：模板文件 `index.html` 渲染失败，但管理员后台（`/server/admin`）和静态资源正常。

**解决**：这不影响 Obsidian 插件的使用。如需修复，可尝试：
```bash
~/.local/pipx/venvs/khoj/bin/python -m pip install --upgrade starlette jinja2
```

### 9.6 后台登录报错 `CSRF verification failed`

**原因**：使用 `127.0.0.1` 访问但 Khoj 默认信任 `localhost`。

**解决**：改用 `http://localhost:42110/server/admin` 访问，或设置 `export KHOJ_DOMAIN=127.0.0.1`。


## 十、日常使用流程

### 10.1 每次使用 Khoj

1. 打开终端
2. （如未持久化配置）设置环境变量
3. 执行 `khoj` 启动服务
4. 保持终端窗口打开
5. 在 Obsidian 中使用 Khoj 插件进行搜索和对话

### 10.2 后台运行（可选）

如果不想一直占用终端窗口：

```bash
nohup khoj > khoj.log 2>&1 &
```

服务会在后台运行，日志写入 `khoj.log`。

### 10.3 更新 Khoj

```bash
pipx upgrade khoj --pip-args="-i https://pypi.tuna.tsinghua.edu.cn/simple"
```


## 十一、关键教训总结

| 问题 | 原因 | 教训 |
|------|------|------|
| Python 版本冲突 | 使用了 Python 3.14 | 安装时**必须显式指定 Python 3.11** |
| 数据库连接失败 | 默认用户是 `postgres` 而非 macOS 用户名 | **必须设置 `POSTGRES_USER` 环境变量** |
| 模型下载卡住 | 无法访问 Hugging Face | **必须设置 `HF_ENDPOINT` 国内镜像** |
| 索引同步失败 | 索引缓存损坏 | **删除 `~/.khoj` + 重置数据库** 可解决 90% 的索引问题 |
| Obsidian 连不上 | API Key 被误填 | **API Key 必须完全留空**（匿名模式） |
| 安装进度条不动 | pipx 不显示下载进度 | 改用 **`pip` + venv** 方式可看到实时进度 |


## 十二、快速命令速查

```bash
# 启动 PostgreSQL
brew services start postgresql@17

# 启动 Khoj（需先设置环境变量）
khoj

# 重置索引（解决同步失败）
rm -rf ~/.khoj && dropdb postgres && createdb postgres && khoj

# 验证服务是否运行
curl http://127.0.0.1:42110/api/v1/user

# 更新 Khoj
pipx upgrade khoj --pip-args="-i https://pypi.tuna.tsinghua.edu.cn/simple"

# 查看 Khoj 日志（后台运行时）
tail -f khoj.log
```
