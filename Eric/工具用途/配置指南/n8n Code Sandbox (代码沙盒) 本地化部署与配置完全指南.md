---
标题: n8n Code Sandbox (代码沙盒) 本地化部署与配置完全指南
创建时间: 2026-11-12 19:02
修改时间: 2026-11-12 19:02
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---

这份更新后的 SOP 文档已将**“随机密钥生成”**的完整操作步骤补充到了标准流程中。你可以直接将其作为最终的施工图保存。

---

# 🚀 n8n Code Sandbox (代码沙盒) 本地化部署与配置完全指南

## 🎯 一、 沙盒的作用与架构说明
n8n Assistant 能够帮你写代码，但为了安全，它必须在隔离的环境中运行这些代码。**Code Sandbox** 就是这个隔离环境。
官方沙盒由三个核心微服务组成：
1.  **`sandbox-certs`**：启动时运行一次，负责生成各个服务之间通信所需的 mTLS（双向加密）证书，生成完即退出。
2.  **`sandbox-api`**：控制平面。负责接收 n8n 主程序发来的执行请求，并指挥 Runner 干活。
3.  **`sandbox-runner-1`**：执行平面。真正用来跑代码的容器，底层使用了 Docker-in-Docker (DinD) 技术，**极其消耗内存**。

---

## 📁 二、 目录与前置准备
请确保你已放弃之前的 `docker run` 临时方案，全面转向 `docker-compose`。

*   **配置目录**：`/Users/eric/projects/n8n-config`
*   **数据目录**：`/Users/eric/projects/n8n_data` (请确保之前已映射并保留了数据)
*   **硬件要求**：Mac 的 Docker Desktop 建议分配至少 **8GB 内存**（沙盒非常吃内存）。

---

## 🛠️ 三、 标准配置流程

### 1. 生成安全随机密钥（核心避坑点）
沙盒内部服务之间（API 和 Runner）需要通过多组密钥进行相互认证。在终端中执行以下命令，生成 3 个独立的 32 位随机字符串：

```bash
# 连续运行三次，或者用 && 连在一起一次性生成
openssl rand -hex 16 && openssl rand -hex 16 && openssl rand -hex 16
```
*假设你终端输出的结果分别是：*
*   `a1b2c3d4e5f6...` （记作 **密钥 A**）
*   `f6e5d4c3b2a1...` （记作 **密钥 B**）
*   `9a8b7c6d5e4f...` （记作 **密钥 C**）
*请将这三串字符复制保存在备忘录中，下一步马上用到。*

### 2. 创建 `.env` 文件（环境变量配置）
在 `n8n-config` 目录下创建 `.env` 文件，并将刚才生成的密钥填入：

```ini
# 锁定沙盒版本，避免 latest 更新导致兼容问题
N8N_SANDBOX_VERSION=1.13.0

# 填入你刚生成的 密钥A
N8N_RUNNERS_AUTH_TOKEN=密钥A

# 填入你刚生成的 密钥B（用于 n8n 访问沙盒的凭证）
SANDBOX_API_KEYS=密钥B
N8N_SANDBOX_SERVICE_API_KEY=密钥B

# 填入你刚生成的 密钥C（用于 Runner 向 API 注册和调用）
SANDBOX_API_RUNNER_REGISTRATION_TOKEN=密钥C
SANDBOX_RUNNER_API_KEYS=密钥C
SANDBOX_API_RUNNER_API_KEY=密钥C
```
*(注：本地测试环境为了简单，有些密钥我做了合并复用，只要前后一致即可。)*

### 3. 编写终极版 `docker-compose.yml`
直接复制以下完整代码，**不要随便增减缩进**。所有环境变量和路径均已配齐：

```yaml
services:
  n8n:
    image: n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - NODES_EXCLUDE=[]
      # 显式指定沙盒地址和密钥B，防止前端 UI 抽风
      - N8N_INSTANCE_AI_SANDBOX_URL=http://sandbox-api:8080
      - N8N_INSTANCE_AI_SANDBOX_API_KEY=密钥B
    volumes:
      - /Users/eric/projects/n8n_data:/home/node/.n8n
      - /Users/eric/short-drama-project/generated_frames:/data/generated_frames

  sandbox-certs:
    image: ghcr.io/n8n-io/n8n-sandbox-service-api:${N8N_SANDBOX_VERSION}
    user: '0:0'
    entrypoint: ['sh', '-c']
    command:
      - >
        bootstrap-mtls.sh --out-dir /tls --api-san sandbox-api --control-san-prefix sandbox-runner --world-readable
    volumes:
      - sandbox-certs:/tls
    restart: "no"

  sandbox-api:
    image: ghcr.io/n8n-io/n8n-sandbox-service-api:${N8N_SANDBOX_VERSION}
    depends_on:
      sandbox-certs:
        condition: service_completed_successfully
    env_file: .env
    environment:
      SANDBOX_API_GRPC_TLS_CERT_FILE: /tls/api/grpc-server.crt
      SANDBOX_API_GRPC_TLS_KEY_FILE: /tls/api/grpc-server.key
      SANDBOX_API_GRPC_TLS_CLIENT_CA_FILE: /tls/api/ca.crt
      SANDBOX_API_RUNNER_CONTROL_GRPC_TLS_CA_FILE: /tls/api/ca.crt
      SANDBOX_API_RUNNER_CONTROL_GRPC_TLS_CERT_FILE: /tls/api/control-grpc-api-client.crt
      SANDBOX_API_RUNNER_CONTROL_GRPC_TLS_KEY_FILE: /tls/api/control-grpc-api-client.key
      SANDBOX_API_RUNNER_CONTROL_GRPC_TLS_SERVER_NAME: sandbox-runner-1
    volumes:
      - sandbox-certs:/tls:ro
    healthcheck:
      test: ['CMD', 'wget', '-qO-', 'http://localhost:8080/healthz']
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s
    restart: unless-stopped

  sandbox-runner-1:
    image: ghcr.io/n8n-io/n8n-sandbox-service-runner-dind:${N8N_SANDBOX_VERSION}
    privileged: true # DinD 必须拥有特权
    depends_on:
      sandbox-api:
        condition: service_healthy
    env_file: .env
    environment:
      SANDBOX_RUNNER_API_GRPC_ADDR: sandbox-api:9090
      SANDBOX_RUNNER_HTTP_BASE_URL: https://sandbox-runner-1:8080
      SANDBOX_RUNNER_CONTROL_GRPC_LISTEN_ADDR: ':9091'
      SANDBOX_RUNNER_CONTROL_GRPC_ADVERTISE_ADDR: sandbox-runner-1:9091
      SANDBOX_RUNNER_ID: runner-1
      SANDBOX_RUNNER_DOCKER_SANDBOX_IMAGE: ghcr.io/n8n-io/n8n-sandbox-service-sandbox:${N8N_SANDBOX_VERSION}
      
      # 注册 Token（对应 .env 里的密钥C）
      SANDBOX_RUNNER_REGISTRATION_TOKEN: ${SANDBOX_API_RUNNER_REGISTRATION_TOKEN}
      
      # 注册阶段 mTLS 证书
      SANDBOX_RUNNER_REGISTRATION_GRPC_CA_FILE: /tls/runner/ca.crt
      SANDBOX_RUNNER_REGISTRATION_GRPC_CERT_FILE: /tls/runner/grpc-client.crt
      SANDBOX_RUNNER_REGISTRATION_GRPC_KEY_FILE: /tls/runner/grpc-client.key
      SANDBOX_RUNNER_REGISTRATION_GRPC_SERVER_NAME: sandbox-api
      
      # 控制平面 mTLS 证书 (极易写错的变量名)
      SANDBOX_RUNNER_CONTROL_GRPC_TLS_CERT_FILE: /tls/runner/control-grpc-server.crt
      SANDBOX_RUNNER_CONTROL_GRPC_TLS_KEY_FILE: /tls/runner/control-grpc-server.key
      SANDBOX_RUNNER_CONTROL_GRPC_TLS_CLIENT_CA_FILE: /tls/runner/ca.crt
    volumes:
      - sandbox-certs:/tls:ro
    restart: unless-stopped

volumes:
  sandbox-certs:
```
*(⚠️ 注意：请把 `docker-compose.yml` 里的 `密钥B` 替换成你真实生成的字符串。)*

### 4. 启动并检查底层状态
```bash
docker compose down -v  # -v 确保清理掉之前失败的证书
docker compose up -d
```
*   **查看日志**：`docker compose logs sandbox-runner-1 --tail=20`。
*   **成功标志**：日志中必须出现 `runner registration stream established` 和 `runner registration heartbeat sent`，且没有任何红色的 `ERROR`。

### 5. 在 n8n UI 界面完成绑定
回到浏览器打开 n8n：
1. 打开 Assistant 设置 -> Code Sandbox。
2. 选择 `n8n Sandbox`。
3. **Service URL** 填入：`http://sandbox-api:8080`
4. **API key** 填入你 `.env` 里配置的 `SANDBOX_API_KEYS`（即 **密钥B**）。
5. 点击 Save。*(如果界面报错，按 F5 刷新一次网页再填即可)*。

---

## 🚨 四、 核心踩坑复盘与排错记录

我们在部署时经历了层层报错，以下是“坑点”及解决办法，极具复用价值：

| 报错现象 (日志或界面) | 根本原因 | 解决办法 |
| :--- | :--- | :--- |
| **`denied`** (拉取镜像时) | 镜像仓库地址错误 | 官方沙盒不在 Docker Hub，而在 GitHub 仓库：`ghcr.io/n8n-io/...` |
| **`mapping key "image" already defined`** | YAML 格式错误 | 追加代码时缩进混乱，把 Runner 写进了 API 的子集里。必须严格对齐顶级 `services:`。 |
| **`SANDBOX_RUNNER_API_KEYS must be set`** | Runner 缺少 API 鉴权 | 在 `.env` 中增加 `SANDBOX_RUNNER_API_KEYS` 变量，并在 Runner 节点中显式传递。 |
| **`...REGISTRATION_GRPC_CA_FILE... are required`** | 缺少注册 mTLS 证书 | Runner 去 API 报到时必须证明身份。补充 `SANDBOX_RUNNER_REGISTRATION_GRPC_*` 变量，指向 `/tls/runner/` 下的证书。 |
| **`...CONTROL_GRPC_TLS_CERT_FILE... are required`** | 缺少控制面 mTLS 证书 | Runner 干活时需要通信加密。补充 `SANDBOX_RUNNER_CONTROL_GRPC_TLS_*` 变量（注意不是 `API_GRPC`）。 |
| **`SANDBOX_RUNNER_REGISTRATION_TOKEN must be set`** | 缺少报到信物 | 在 Runner 环境变量中增加 `SANDBOX_RUNNER_REGISTRATION_TOKEN`。 |
| **界面提示 `no sandbox runners are registered`** | 前端 UI 状态未缓存 | 底层配好后，直接在浏览器按 `F5` 刷新网页，重新填一次 URL 和 Key 即可秒连。 |

---

## 🛡️ 五、 日常维护与使用注意事项

1.  **极度吃内存**：沙盒运行时，Docker Desktop 会占用大量内存。如果你平时主要在画布上拖拽节点，不需要 AI 跑代码，强烈建议执行 `docker compose stop sandbox-runner-1 sandbox-api` 把沙盒停掉。需要用时再 `start`，能极大减轻 Mac 的负担。
2.  **网络隔离大坑**：n8n 运行在容器内，**绝对不要**在沙盒或 HTTP 请求中填 `127.0.0.1` 或 `localhost`。访问 Mac 宿主机必须用 `host.docker.internal`，访问内部容器必须用容器名（如 `sandbox-api`）。
3.  **文件映射防抖**：由于 Mac 的 Docker 文件系统事件有时会丢失，在 n8n 的 `Local File Trigger` 节点中，建议在 Options 里开启 **`Use Polling`**，防止监听不到 Claude Code 生成的剧本文件。