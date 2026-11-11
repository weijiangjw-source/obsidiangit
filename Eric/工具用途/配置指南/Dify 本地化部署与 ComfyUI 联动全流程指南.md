---
标题: Dify 本地化部署与 ComfyUI 联动全流程指南
创建时间: 2026-11-12 02:13
修改时间: 2026-11-12 02:13
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---

这份指南基于我们刚刚在 Mac 环境下（特别是你的 `M 系列芯片` 和 `/Users/eric/projects` 路径）成功打通 Dify 部署的全过程复盘。

这份 SOP（标准作业程序）不仅记录了踩坑点，还详细说明了各工具的定位及核心的网络打通配置，供你未来复用或迁移到新环境时参考。

---

# 🚀 Dify 本地化部署与 ComfyUI 联动全流程指南

## 🎯 工具角色说明（定位澄清）
在漫剧自动化流程中，明确各工具的“分工”是关键：
*   **Dify（总调度台 / 大脑）**：负责流程逻辑编排、剧本拆解、提示词优化、条件判断，以及触发外部工具（如调用 ComfyUI）。
*   **ComfyUI（生产车间 / 画笔）**：负责具体的图像/视频渲染（如你接入的 Krea 2 和 MiniMax H3）。
*   **Docker（基础设施）**：提供独立的容器环境，让 Dify 的各种依赖（PostgreSQL, Redis, Web 等）互不干扰地稳定运行。

---

## 🛠️ 一、 部署前环境准备
1.  **Docker Desktop**：已安装，且建议在 `Preferences -> Resources` 中分配至少 **8GB 内存**。
2.  **终端环境**：Mac 系统的 Terminal（或 iTerm2）。
3.  **端口占用检查**：Dify 默认占用 80 和 443 端口。如果被占用，需修改 `.env` 文件中的 `EXPOSE_NGINX_PORT`（例如改为 8080）。

---

## 📦 二、 Dify 部署标准流程（Mac 环境）

### 1. 获取代码并定位目录
在终端中执行，将代码克隆到你指定的项目目录：
```bash
cd /Users/eric/projects
git clone https://github.com/langgenius/dify.git
cd dify/docker
```

### 2. 配置文件初始化
复制环境变量模板并进入文件准备修改：
```bash
cp .env.example .env
```

### 3. 启动 Dify 服务
确保 Docker Desktop 正在运行，执行一键拉起：
```bash
docker compose up -d
```
此时浏览器访问 `http://localhost`，如能出现“设置管理员账户”界面，代表核心服务部署成功。

---

## 🚨 三、 核心复盘：踩坑记录与修复方法

### 💥 踩坑复盘 1：`agent_backend` 和 `local_sandbox` 无限重启
*   **现象**：执行 `docker compose ps` 时，发现这俩服务状态一直为 `Restarting`。
*   **排查过程**：使用 `docker compose logs local_sandbox --tail=50` 查看日志，报错为 `shellctl: auth token is required` 和 `Value error, local_sandbox_auth_token is required for the local runtime backend`。
*   **原因分析**：Dify 较新版本（如 1.19.0）引入了高级 Agent 沙盒执行环境，官方 `.env` 模板默认将沙盒的鉴权 Token 留空，导致前后端无法通信，陷入崩溃。
*   **彻底修复方法**：
    1. 生成随机 Token：`openssl rand -hex 32` （复制生成的字符串）。
    2. 修改 `/Users/eric/projects/dify/docker/.env`，找到 `DIFY_AGENT_LOCAL_SANDBOX_AUTH_TOKEN=`，将刚生成的 Token 填在等号后。
    3. **不要用 `restart` 命令**，必须用 `docker compose up -d --force-recreate local_sandbox agent_backend` 强制重建容器加载新变量。
*   **验证**：再次 `docker compose ps`，看到两者状态变为 `Up (healthy)` 即大功告成。

### 💥 踩坑复盘 2：初始化账户设置
*   **现象**：初次访问 `http://localhost` 要求设置管理员。
*   **正确操作**：这是本地数据库（PostgreSQL）的初始化，填写的邮箱不需要接收验证码，但**密码必须包含字母和数字且不少于 8 位**。这是你后续登录的最高权限账号。

---

## 🔗 四、 重头戏：Dify 与本地 ComfyUI 的网络打通

因为 Dify 运行在 Docker 虚拟网络中，而 ComfyUI 直接运行在 Mac 宿主机上，**两者的 `localhost` 不是一回事**。

### 1. 启动 ComfyUI 的必要姿势
必须让你的 ComfyUI 监听所有网络接口，而不是仅监听本地。
*   **终端启动法**：`python main.py --listen 0.0.0.0` （或者 `python main.py --listen`）
*   **Mac 桌面版/一键包**：如果通过代码启动，确保启动脚本里加上了 `--listen` 参数。

### 2. 在 Dify 中配置 HTTP 请求节点
当你在 Dify 工作流里添加了 **HTTP 请求（HTTP Request）** 节点，准备调用 ComfyUI 的 API（例如 `http://127.0.0.1:8188/prompt`）时：
*   **错误写法**：`http://localhost:8188` 或 `http://127.0.0.1:8188` （Dify 的 Docker 容器会认为在找它自己，导致连接拒绝）。
*   **正确写法**：**`http://host.docker.internal:8188`**
    *   *`host.docker.internal` 是 Docker 提供的内部 DNS 域名，专门用于让容器访问宿主机（你的 Mac）。*

---

## 🛡️ 五、 日常维护与重要注意事项

1.  **修改 `.env` 后的生效规则**：
    任何时候修改了 `/Users/eric/projects/dify/docker/.env` 文件，**绝不能**只敲 `docker compose restart`。必须执行 `docker compose up -d`，Compose 会检测到环境变更并自动重建相关容器。
2.  **数据备份（极度重要）**：
    Dify 的代码可以随时重新克隆，但你的账号、剧本、工作流配置都保存在 Docker 数据卷中。定期备份 `dify/docker/volumes` 目录（或了解 Docker Volume 备份机制），防止误删导致心血白费。
3.  **Mac 磁盘访问权限**：
    如果未来 Dify 需要读取你 Mac 上的本地文件（例如通过插件读取 `~/projects` 下的剧本文件），务必前往 Mac 的 **系统设置 -> 隐私与安全性 -> 完全磁盘访问权限**，将 Docker 和终端勾选上。
4.  **系统升级策略**：
    以后 Dify 出新版，在 `dify` 目录下执行 `git pull` 拉取新代码后，**一定要先看官方的 Release Note**。若有类似我们踩坑的 `.env` 新增变量，最好将 `.env.example` 对比一下，把新变量合并到你的旧 `.env` 里，然后再执行 `docker compose up -d`。

---
**下一步建议**：现在，你可以进入 Dify 的“设置 -> 模型供应商”，把你常用的 LLM（如 DeepSeek 或 OpenAI）API Key 填进去，然后新建一个“工作流”应用，在画布上体验一下流程编排了。