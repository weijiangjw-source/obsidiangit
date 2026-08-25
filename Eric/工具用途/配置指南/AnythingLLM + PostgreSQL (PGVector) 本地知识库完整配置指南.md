---
标题: AnythingLLM + PostgreSQL (PGVector) 本地知识库完整配置指南
创建时间: 2026-08-25 23:05
修改时间: 2026-08-25 23:05
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 中等
状态: 已成功
---

# AnythingLLM + PostgreSQL (PGVector) 本地知识库完整配置指南

> 本文档基于 macOS (Homebrew) 环境的实际操作经验整理，涵盖从零开始配置 AnythingLLM 本地知识库的全流程，包括向量数据库、嵌入模型、Notion MCP 等核心组件的配置方法与踩坑复盘。


## 一、概述与工具说明

### 1.1 这套系统能做什么？

- **本地知识库问答（RAG）** ：上传 PDF、Word、TXT 等文档，AI 基于文档内容回答问题
- **Notion 笔记检索与操作**：通过 MCP 协议让 AI 读取和操作 Notion 内容
- **数据完全私有**：所有向量数据存储在本地 PostgreSQL 数据库中，不上传云端

### 1.2 核心组件说明

| 组件 | 作用 | 本指南使用方案 |
|------|------|----------------|
| **AnythingLLM** | RAG 应用框架，提供聊天界面和文档管理 | 桌面版（Desktop） |
| **PostgreSQL + pgvector** | 向量数据库，存储文档的向量 embeddings | Homebrew 安装 PostgreSQL 17 |
| **嵌入模型（Embedder）** | 将文档文本转换为向量 | AnythingLLM 内置的 `multilingual-e5-small`（支持中文） |
| **大语言模型（LLM）** | 负责生成回答 | 用户自行配置（Ollama / OpenAI 等） |


## 二、环境准备：PostgreSQL 与 pgvector

### 2.1 安装 PostgreSQL

macOS 使用 Homebrew 安装：

```bash
brew install postgresql@17
brew services start postgresql@17
```

**⚠️ 关键踩坑点（Homebrew 安装特有）** ：

Homebrew 安装 PostgreSQL 时，**不会**创建传统的 `postgres` 超级用户，而是会**创建一个与当前 macOS 登录名同名的超级用户**。例如，如果你的 Mac 用户名是 `eric`，则数据库超级用户名就是 `eric`，而不是 `postgres`。

❌ 错误示例（会报错 `role "postgres" does not exist`）：
```bash
psql -U postgres -d postgres
```

✅ 正确示例：
```bash
psql -U eric -d postgres
# 或直接省略 -U 参数（默认使用当前系统用户名）
psql -d postgres
```

### 2.2 安装并启用 pgvector 扩展

pgvector 是 PostgreSQL 的向量扩展，AnythingLLM 的 PGVector 功能依赖它。

登录数据库后执行：

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

验证是否已安装：

```sql
\dx
```

如果看到 `vector` 在列表中，说明安装成功。

**📌 连接字符串格式**（后续在 AnythingLLM 中填写）：

```text
postgresql://用户名:密码@127.0.0.1:5432/postgres
```

如果密码为空：
```text
postgresql://用户名@127.0.0.1:5432/postgres
```

**⚠️ 关键建议**：连接地址**强烈建议使用 `127.0.0.1` 而非 `localhost`**。macOS 上 `localhost` 可能优先解析为 IPv6 地址 `::1`，而 pgAdmin 或某些应用通过 IPv6 连接时可能遇到认证问题。使用 `127.0.0.1` 强制走 IPv4 可避免此类问题。


## 三、安装 pgAdmin（可选，用于可视化查看数据库）

pgAdmin 是 PostgreSQL 的官方图形化管理工具，便于查看向量数据表中的内容。

### 3.1 安装

```bash
brew install --cask pgadmin4
```

如果 Homebrew 下载失败，可直接从官网下载 `.dmg` 安装包：
https://www.pgadmin.org/download/pgadmin-4-macos/

### 3.2 连接配置

1. 打开 pgAdmin，点击 **“Add New Server”**
2. 在 **General** 标签页，**Name** 填写任意名称（如 `My Local DB`）
3. 在 **Connection** 标签页填写：

| 字段 | 填写内容 |
|------|----------|
| Host name/address | `127.0.0.1`（**不要用 localhost**） |
| Port | `5432` |
| Maintenance database | `postgres` |
| Username | **你的 Mac 用户名**（如 `eric`，不是 `postgres`） |
| Password | 你设置的数据库密码 |

**⚠️ 踩坑点**：Username 必须填写 Homebrew 创建的那个与系统用户名同名的账号，而不是 `postgres`。如果填错会报 `role "xxx" does not exist`。


## 四、AnythingLLM 基础配置

### 4.1 安装 AnythingLLM 桌面版

从官网下载安装包：https://anythingllm.com/

### 4.2 配置大语言模型（LLM）

进入 **设置 → 大语言模型（LLM）** ，根据你的情况选择：
- **Ollama**：本地模型，需先安装 Ollama 并下载模型
- **OpenAI**：需填写 API Key
- 其他提供商按需配置

### 4.3 配置向量数据库（PGVector）

进入 **设置 → 向量数据库**：

| 选项 | 填写内容 |
|------|----------|
| 向量数据库提供商 | 选择 **PGVector** |
| Postgres Connection String | `postgresql://用户名:密码@127.0.0.1:5432/postgres` |
| Vector Table Name | `anythingllm_vectors`（保持默认） |

点击 **保存更改**，然后**重启 AnythingLLM**。

### 4.4 配置嵌入模型（Embedder）—— 最关键的一步

进入 **设置 → 嵌入器（Embedder）** ，这是决定文档能否被正确向量化的核心环节。

**推荐配置**：

| 选项 | 推荐值 |
|------|--------|
| 嵌入引擎提供商 | **AnythingLLM Embedder**（内置，零配置） |
| 模型 | **`multilingual-e5-small`**（支持中文，效果最好） |

**⚠️ 踩坑复盘**：

1. **不要选 `Local AI`**：除非你确实在本地运行了 LocalAI 或 Ollama 的嵌入服务，否则会因连接不到服务而导致向量化为 0。
2. **中文文档务必选 `multilingual-e5-small`**：`all-MiniLM-L6-v2` 主要针对英文优化，处理中文效果很差。`multilingual-e5-small` 对中文语义的理解远优于前者。
3. **首次使用会自动下载模型**：`multilingual-e5-small` 约 400MB，下载需要几分钟，请耐心等待，不要强行关闭应用。

选择模型后点击 **保存**。


## 五、文档上传与向量化 —— 最容易踩坑的环节

### 5.1 ❌ 错误方式：通过聊天框的 `+` 号上传

聊天输入框旁边的 `+` 号上传文件，**只会把文件内容作为当前对话的临时上下文**，**不会触发向量化存储**。这是本指南中**最容易踩的坑**。

### 5.2 ✅ 正确方式：通过“文档”管理页面上传并嵌入

1. 在 AnythingLLM 左侧边栏，点击 **“文档”**（Document）图标（不是聊天框的 `+` 号）
2. 点击 **上传** 按钮，选择文件
3. 上传完成后，**勾选该文件**
4. 点击 **“保存并嵌入”**（Save and Embed）按钮
5. 等待处理完成，文件状态变为绿色 ✅ **“就绪”**（Ready）

### 5.3 验证向量化是否成功

进入 **设置 → 向量数据库**，查看 **“向量数量”**：
- 如果 > 0，说明嵌入成功
- 如果仍为 0，说明嵌入失败，检查嵌入模型配置

**📌 触发向量化的另一种方式**：如果在“文档”页面找不到“保存并嵌入”按钮（不同版本 UI 可能有差异），也可以通过聊天框上传文件后，**发送一条消息**（如“请处理这份文档”），系统会自动触发向量化。但“文档”页面的方式更可控、更清晰。


## 六、配置 Notion MCP（Model Context Protocol）

MCP 允许 AnythingLLM 的 AI Agent 读取和操作 Notion 内容。

### 6.1 创建 Notion 集成并获取令牌

1. 访问 https://www.notion.so/my-integrations
2. 点击 **“New integration”**
3. 填写名称，选择关联的工作区
4. 创建后复制 **内部集成令牌**（格式：`secret_xxx`）

**⚠️ 权限配置**：在集成设置的 **“Capabilities”** 中，确保 **“Read content”** 权限已开启。

### 6.2 编辑 MCP 配置文件

配置文件位置：
- **macOS**：`~/Library/Application Support/anythingllm-desktop/storage/plugins/anythingllm_mcp_servers.json`
- **Windows**：`%APPDATA%\anythingllm-desktop\storage\plugins\anythingllm_mcp_servers.json`

如果文件或目录不存在，手动创建。

写入以下内容：

```json
{
  "mcpServers": {
    "notion-mcp": {
      "command": "/opt/homebrew/bin/npx",
      "args": [
        "-y",
        "@notionhq/notion-mcp-server"
      ],
      "env": {
        "NOTION_TOKEN": "你的secret_xxx令牌"
      }
    }
  }
}
```

**⚠️ 关键踩坑点**：

1. **`command` 必须使用 `npx` 的绝对路径**。在终端执行 `which npx` 获取路径。Apple Silicon Mac 通常是 `/opt/homebrew/bin/npx`，Intel Mac 通常是 `/usr/local/bin/npx`。如果不写绝对路径，AnythingLLM 可能因环境变量 PATH 不同而找不到 `npx` 命令。

2. **环境变量名是 `NOTION_TOKEN`**，不是 `NOTION_API_TOKEN`。用错变量名会导致令牌无效。

3. **Notion 页面必须手动授权给集成**：在 Notion 中打开目标页面 → 点击右上角 `...` → **“Add connections”** → 选择你的集成名称。**这一步经常被忽略**，导致 MCP 搜索返回空结果。

### 6.3 加载 MCP 配置

1. 保存 `anythingllm_mcp_servers.json` 文件
2. 在 AnythingLLM 中进入 **设置 → 代理技能（Agent Skills）**
3. 找到 **MCP 服务器** 区域，点击 **“重新开始；更新”** 按钮
4. 如果配置正确，会看到 `Notion MCP` 出现在列表中，且**前面没有黄色三角警告**

### 6.4 使用 Notion MCP

在聊天框中通过 `@agent` 调用：

```
@agent 在 Notion 中搜索 "关键词"
@agent 读取 Notion 页面 ID [页面ID] 的内容
```

**📌 关于搜索功能**：Notion API 的搜索接口需要集成能访问到目标页面才能返回结果。如果搜索无结果，请检查页面是否已授权给集成。


## 七、社区中心技能（Agent Skills）

AnythingLLM 的社区中心提供了可直接导入的 Agent 技能。

### 7.1 访问社区中心

在 AnythingLLM 界面中找到 **“社区中心”**（Community Hub）入口。

### 7.2 技能类型说明

| 类型 | 标识 | 安全建议 |
|------|------|----------|
| **Verified Skill** | 带有 ✅ 或“已验证”标识 | 官方审核过，可放心导入 |
| **Unverified Skill** | 无标识或“未验证” | 风险自担，建议仅导入功能简单、不涉及文件读写的技能 |

### 7.3 导入与使用

1. 点击技能卡片右下角的 **“Import”** 按钮
2. 导入后在 **设置 → 代理技能** 中找到该技能，打开开关（On）
3. 在聊天中通过自然语言或 `@agent` 调用

**⚠️ 关于安全策略**：如果导入 Unverified 技能时报错 `Community hub bundle downloads are limited to verified public items`，说明当前安全策略为“仅官方审核及私有物品”。如需导入未验证技能，可在 **设置 → 应用偏好设置** 中将 `Community Hub Access Level` 临时改为 `Allow all items`，导入后**立即改回**原设置。


## 八、常见问题与踩坑复盘

### 8.1 向量数量一直为 0

| 可能原因 | 解决方案 |
|----------|----------|
| 嵌入模型未正确配置 | 检查“嵌入器”设置，确保选择了有效的提供商和模型 |
| 模型未下载完成 | `multilingual-e5-small` 约 400MB，等待下载完成 |
| 通过聊天框 `+` 号上传而非“文档”页面上传 | 改用“文档”页面上传并点击“保存并嵌入” |
| 文档状态为 ❌（失败） | 查看错误信息，通常是网络或 API Key 问题 |

### 8.2 PostgreSQL 连接失败

| 错误信息 | 解决方案 |
|----------|----------|
| `role "postgres" does not exist` | Homebrew 安装的用户名是 Mac 用户名，不是 `postgres` |
| `FATAL: role "eric" does not exist`（但 psql 能连上） | pgAdmin 中 Host 改用 `127.0.0.1` 而非 `localhost` |
| `connection to server at "::1" failed` | 同样，改用 `127.0.0.1` 强制 IPv4 |

### 8.3 Notion MCP 黄色三角警告

| 可能原因 | 解决方案 |
|----------|----------|
| `npx` 命令找不到 | 在配置文件中使用 `which npx` 获取的绝对路径 |
| 环境变量名错误 | 使用 `NOTION_TOKEN` 而非 `NOTION_API_TOKEN` |
| Notion 页面未授权给集成 | 在 Notion 中通过 `...` → `Add connections` 授权 |
| Node.js 环境问题 | 确保 Node.js 已安装，`npx` 命令可正常运行 |

### 8.4 时间/日期技能不准

如果安装了 `Contextual DateTime` 等时间技能但返回时间错误：

- **检查 VPN/代理**：如果技能依赖外部时间 API，VPN 可能导致访问超时或返回错误数据。可尝试关闭 VPN 后重试，或在 VPN 中将时间 API 域名加入直连白名单。
- **强制调用**：使用 `@agent 现在几点了？` 强制触发技能调用

### 8.5 Google 日历等需要翻墙的服务与本地服务共存

如果同时使用 Google Calendar MCP（需代理）和本地时间技能（无需代理）：

**方案一（推荐）** ：配置 VPN 分流规则，让 `google.com` 域名走代理，其他流量直连。

**方案二**：在 AnythingLLM 启动前设置环境变量：
```bash
export HTTP_PROXY="http://127.0.0.1:你的代理端口"
export HTTPS_PROXY="http://127.0.0.1:你的代理端口"
open -a "AnythingLLM"
```


## 九、日常使用流程速查

```
1. 上传文档 → “文档”页面 → 上传 → 勾选 → “保存并嵌入”
2. 等待向量化完成（查看向量数量是否增加）
3. 在工作区聊天框提问，AI 将基于文档内容回答
4. （可选）通过 @agent 调用 Notion MCP 等外部工具
```

**📌 核心原则**：**“上传”不等于“向量化”** 。必须执行“保存并嵌入”操作，文档才会真正进入向量数据库。


## 十、备份与恢复

备份整个向量数据库：

```bash
pg_dump -U 用户名 -d postgres -t anythingllm_vectors > backup.sql
```

恢复：

```bash
psql -U 用户名 -d postgres < backup.sql
```


## 附录：关键文件路径速查

| 文件/目录 | 路径（macOS） |
|-----------|---------------|
| AnythingLLM 配置目录 | `~/Library/Application Support/anythingllm-desktop/` |
| MCP 配置文件 | `~/Library/Application Support/anythingllm-desktop/storage/plugins/anythingllm_mcp_servers.json` |
| PostgreSQL 数据目录 | `/opt/homebrew/var/postgresql@17/`（Apple Silicon） |
| pgAdmin 日志 | `~/Library/Logs/pgadmin/` |