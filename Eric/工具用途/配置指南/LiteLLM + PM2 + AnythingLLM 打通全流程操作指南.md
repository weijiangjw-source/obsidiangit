---
标题: LiteLLM + PM2 + AnythingLLM 打通全流程操作指南
创建时间: 2026-08-25 22:54
修改时间: 2026-08-25 22:54
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 中等
状态: 已成功
---

基于我们刚刚从头到尾的实战经验，我将整个过程复盘整理成这份**《LiteLLM + PM2 + AnythingLLM 打通全流程操作指南》**。

这份指南不仅包含成功的步骤，更重点标注了我们在配置中踩过的**5个关键坑**，确保你（或其他人）下次能一次性成功。

---

## 一、 工具说明与适用场景

在开始前，先明确我们用了什么，以及为什么用：

| 工具 | 角色 | 为什么用它 |
| :--- | :--- | :--- |
| **LiteLLM** | 模型网关（Proxy） | 将不同厂商的API（OpenAI、Anthropic、本地模型）统一转换成OpenAI兼容格式。 |
| **PM2** | 进程守护工具 | 让LiteLLM在后台常驻运行，崩溃后自动重启，且支持开机自启。 |
| **pipx** | Python应用安装工具 | 将LiteLLM安装进独立的虚拟环境，避免污染系统Python（我们的路径是 `~/.local/pipx/venvs/litellm`）。 |
| **AnythingLLM** | AI应用客户端 | 连接LiteLLM的网关，实现统一模型调用。 |

---

## 二、 前置准备（确认环境）

在终端执行以下命令，确认基础环境：

```bash
# 1. 确认 Python3 位置（通常为 /opt/homebrew/bin/python3 或 /usr/bin/python3）
which python3

# 2. 确认 pipx 是否已安装（若未安装，先执行 brew install pipx）
which pipx

# 3. 确认 LiteLLM 已通过 pipx 安装
pipx list | grep litellm
```

> **注意**：若 LiteLLM 未安装，执行 `pipx install 'litellm[proxy]'`。

---

## 三、 核心步骤：配置 PM2 管理 LiteLLM

### 1. 创建 PM2 配置文件
**文件路径**：`~/.pm2/litellm.config.js`  
**关键点**：必须使用 **pipx 虚拟环境中的 Python 解释器**，否则会报 `ModuleNotFoundError`。

```javascript
module.exports = {
  apps: [{
    name: 'litellm',                     // 进程名（短命名，方便管理）
    script: '/Users/eric/.local/bin/litellm',  // 可执行文件绝对路径（which litellm 的结果）
    interpreter: '/Users/eric/.local/pipx/venvs/litellm/bin/python', // 关键！虚拟环境Python
    args: '--config /Users/eric/.litellm/config.yaml --port 4000',   // 强制指定4000端口
    autorestart: true,
    max_memory_restart: '1G',
  }]
};
```

**如何确认你的解释器路径？**
```bash
# 查看 litellm 命令的实际 Python 解释器
head -1 /Users/eric/.local/bin/litellm
# 如果显示 #!/usr/bin/env python3，但系统Python没有litellm包，
# 则必须直接用虚拟环境的绝对路径：/Users/eric/.local/pipx/venvs/litellm/bin/python
```

---

### 2. 启动并保存进程

```bash
# 启动服务（首次）
pm2 start ~/.pm2/litellm.config.js

# 保存进程列表（关键！实现开机自启和 pm2 start litellm 短命令）
pm2 save
```

---

## 四、 深度复盘：我们踩过的 5 个关键坑（必看！）

### 坑 1：配置文件写入错误（`SyntaxError: Invalid regular expression flags`）
**现象**：`cat ~/.pm2/litellm.config.js` 后发现文件内容第一行是 `cat > ... << 'EOF'`。  
**原因**：复制终端命令时只复制了中间部分，导致 Shell 命令被写入了 JS 文件。  
**解决方案**：**不要用 `cat` 命令创建 JS 配置文件！** 改用 `nano ~/.pm2/litellm.config.js` 手动输入内容，或使用 `printf` 纯文本写入。

### 坑 2：PM2 默认用 Node.js 执行 Python 脚本（`SyntaxError: Unexpected identifier 'litellm'`）
**现象**：日志显示 Node.js 报错，显然 PM2 把 `litellm` 当成了 JS 代码。  
**原因**：PM2 默认解释器是 Node.js，不识别 Python 的 shebang。  
**解决方案**：在配置中**必须显式指定** `interpreter` 字段，指向正确的 Python。

### 坑 3：`interpreter` 错误写法（`Interpreter is NOT AVAILABLE`）
**错误写法**：`interpreter: '/usr/bin/env python3'`（PM2 会把整个字符串当命令，找不到 `env python3` 这个文件）。  
**正确写法**：直接写 Python 绝对路径，如 `interpreter: '/opt/homebrew/bin/python3'` 或虚拟环境路径。

### 坑 4：系统 Python 缺少 litellm 包（`ModuleNotFoundError: No module named 'litellm'`）
**现象**：指定了 `/opt/homebrew/bin/python3`，但仍报错找不到 litellm。  
**原因**：Homebrew 的 Python 环境没有安装 litellm（因为 litellm 装在 pipx 虚拟环境里）。  
**解决方案**：**必须用 pipx 虚拟环境内的 Python**，即 `/Users/xxx/.local/pipx/venvs/litellm/bin/python`。

### 坑 5：端口不统一（LiteLLM 跑在 8397 而非 4000）
**现象**：`curl localhost:4000` 失败，日志显示 `Uvicorn running on http://0.0.0.0:8397`。  
**原因**：配置文件或环境变量中隐藏了 `proxy_port: 8397`。  
**解决方案**：在 PM2 的 `args` 中**硬性追加** `--port 4000`，强制覆盖所有配置。

---

## 五、 最终验证与日常维护

### 1. 验证服务
```bash
# 检查进程状态（应为 online）
pm2 status

# 测试 API 连通性（必须返回 JSON 模型列表）
curl http://localhost:4000/v1/models
```

### 2. 配置 AnythingLLM
打开 AnythingLLM 设置页：
- **Base URL**：`http://localhost:4000`
- **API Key**：留空（除非你的 `config.yaml` 设置了 `master_key`）
- **Chat Model**：下拉框会自动加载模型列表，选择即可。

### 3. 适配 Claude 函数（如果使用）
确保你的 `claude-agnes` 函数检测的是 4000 端口。因为 PM2 已常驻 4000，`lsof -i :4000` 检测到进程后，函数会**跳过重复启动**，直接复用服务。

### 4. 日常管理命令
| 操作 | 命令 |
|------|------|
| 启动（若关机后） | `pm2 start litellm` |
| 重启（修改配置后） | `pm2 restart litellm` |
| 停止 | `pm2 stop litellm` |
| 查看实时日志 | `pm2 logs litellm` |
| 设置开机自启 | `pm2 startup`（复制输出执行）-> `pm2 save` |

### 5. 修改模型配置
当需要增删模型时：
1. 编辑 `~/.litellm/config.yaml`
2. 执行 `pm2 restart litellm`
3. 在 AnythingLLM 中重新选择模型（刷新即可）

---

## 六、 总结：成功配置的核心密码

1. **解释器绝对路径**：必须指向 **pipx 虚拟环境**，而非系统 Python。
2. **端口硬编码**：在启动参数里强制写死 `--port 4000`，避免配置漂移。
3. **配置文件纯代码**：**不要**用 `cat` 命令创建 JS 文件，用 `nano` 或 `vim` 手动录入。

这份指南已经过实战检验，完全复用即可，不会再踩我们掉过的坑。如果后续遇到新问题，随时带着日志来找我。祝你配置顺利！