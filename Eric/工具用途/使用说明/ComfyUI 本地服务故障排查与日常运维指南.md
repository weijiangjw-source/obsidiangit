---
标题: ComfyUI 本地服务故障排查与日常运维指南
创建时间: 2026-10-29 15:03
修改时间: 2026-10-29 15:03
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---
# ComfyUI 本地服务故障排查与日常运维指南

> 适用环境：macOS（Apple Silicon），用户 `eric`，项目路径 `/Users/eric/projects/ComfyUI`，PM2 托管，端口 8188。
> 本文档整合了 2026-10-29 的完整排查与更新过程，可直接按症状查阅。

---

## 一、环境速查（出问题时先对照）

| 项目 | 值 |
|---|---|
| 项目目录 | `/Users/eric/projects/ComfyUI` |
| 虚拟环境 | `/Users/eric/projects/ComfyUI/venv` |
| Python 解释器 | `venv/bin/python` → Python 3.11.17（Homebrew `python@3.11`） |
| 启动入口 | `main.py` |
| 默认地址 | `http://127.0.0.1:8188/` |
| PM2 进程名 | `comfyui` |
| PM2 数据目录 | `~/.pm2/`（dump 文件：`~/.pm2/dump.pm2`） |
| 是否配开机自启 | **否**（按需手动启动） |
| **当前版本** | **`v0.43.1`（正式版 tag，detached HEAD，commit `086099f5`）** |
| 更新前版本 | `v0.30.0-17-g2eb60976`（master 开发版） |
| 回滚目标 | `git checkout 2eb60976` 或 `git switch master` |

> 版本信息以实际 `git describe --tags` 为准，更新后记得回来改这张表。

---

## 二、核心概念（先搞懂，能避免 80% 的误操作）

### 1. `pm2 start` vs `pm2 restart` —— 最容易踩的坑

| 命令 | 作用 | 何时用 |
|---|---|---|
| `pm2 start ...` | **新建**一个进程并启动 | 进程不存在时（重启电脑后、首次启动） |
| `pm2 restart comfyui` | **重启**一个已存在的进程 | 进程已在 PM2 列表里，只是要重启 |
| `pm2 stop comfyui` | 停止但**保留配置** | 暂时不用，想释放内存 |
| `pm2 delete comfyui` | 停止并**删除配置** | 彻底清理，下次需重新 `start` |

> ⚠️ **典型错误**：重启电脑后 PM2 列表为空，此时敲 `pm2 restart comfyui` 会报 `Process or Namespace comfyui not found`。正确做法是执行 `comfy`（即 `pm2 start ...`）。

### 2. 两种启动方式，同一时间只能用一种

| 方式 | 命令 | 特点 | 适用场景 |
|---|---|---|---|
| **PM2 后台托管** | `comfy`（alias） | 后台运行，关终端不影响；由 PM2 管理 | 日常使用 |
| **前台直接启动** | `cd 项目目录 && venv/bin/python main.py` | 占着终端，日志直接可见，`Ctrl+C` 停止 | 排查报错、更新后验证 |

两者都占 8188 端口，**不能同时启动**，否则报 `Address already in use`。切换前先停掉另一个。

### 3. `pm2 save` 的作用

把当前 PM2 进程列表写入 `~/.pm2/dump.pm2`。**不配开机自启时，`save` 不是必须的**，但养成 `start` 后 `save` 的习惯，能让 `pm2 resurrect` 在需要时恢复进程。注意：dump 可能被后续操作覆盖，不能完全依赖它。

---

## 三、故障速查表（按症状直接跳转）

| 症状 | 最可能原因 | 跳转 |
|---|---|---|
| `pm2 restart comfyui` 报 `Process or Namespace comfyui not found` | PM2 列表为空，用了 restart 而非 start | §4.1 |
| `http://127.0.0.1:8188/` 打不开，`lsof -i :8188` 无输出 | ComfyUI 根本没在运行 | §4.2 |
| 端口在监听但浏览器打不开 | 监听地址不对 / 远程访问用了 127.0.0.1 | §4.3 |
| 前台启动卡住、`ModuleNotFoundError`、`import torch` 失败 | 依赖缺失、venv 断链、custom_nodes 插件报错 | §4.4 |
| venv 里的 python 报 `No such file or directory` | Homebrew 升级/清理导致符号链接断链 | §4.5 |
| `pip install` 末尾出现 `[notice] A new release of pip is available` | pip 自身有新版本，**非错误** | §4.6 |
| 切 tag 后提示 `You are in 'detached HEAD' state` | 正常现象，**非错误** | §5.4 |

---

## 四、详细排查流程

### 4.1 `pm2 restart` 报 Process not found

**诊断：**
```bash
pm2 list          # 看列表是否为空
```

**解决：**
```bash
# 列表为空 → 用 start 而非 restart
comfy
# 或完整命令：
cd /Users/eric/projects/ComfyUI
pm2 start main.py --name comfyui \
  --interpreter /Users/eric/projects/ComfyUI/venv/bin/python \
  -- --listen 127.0.0.1 --port 8188

pm2 save          # 可选，保存列表
```

### 4.2 网页打不开，端口无监听

**诊断（三条命令，全空即确认没在运行）：**
```bash
lsof -i :8188
ps aux | grep -i "main.py" | grep -v grep
pm2 list
```

**解决：** 按 §4.1 启动。若启动后仍无监听，转 §4.4 前台看报错。

### 4.3 端口在监听但网页打不开

**诊断：**
```bash
lsof -i :8188          # 看监听地址是 127.0.0.1 还是 *:8188
curl -I http://127.0.0.1:8188
```

**处理：**
- 本机访问：若启动带 `--listen 0.0.0.0`，本机用 `127.0.0.1` 也应能通；不通就试 `http://localhost:8188`。
- 远程设备访问：`127.0.0.1` 指向访问者自己，连不上。需启动加 `--listen 0.0.0.0`，再用 `ipconfig getifaddr en0` 查到的局域网 IP 访问。

### 4.4 前台启动报错或卡住

**操作（先停 PM2 里的，再前台跑）：**
```bash
pm2 stop comfyui
cd /Users/eric/projects/ComfyUI
venv/bin/python main.py
```

| 现象 | 含义 | 处理 |
|---|---|---|
| 刷加载信息，最后 `To see the GUI go to: http://127.0.0.1:8188` | 启动成功 | 浏览器访问 |
| 卡在某个 `Loading ...` 不动 | 加载大模型或某 custom_node 卡住 | 等几分钟；仍卡则禁用可疑插件 |
| `ModuleNotFoundError: No module named 'xxx'` | 依赖缺失 | `source venv/bin/activate && pip install -r requirements.txt` |
| `import torch` 报错 | PyTorch 与系统不匹配 | 单独重装 torch（按报错选 wheel） |
| 某 custom_node 抛异常 | 插件问题 | 目录改名加 `.disabled` 后缀再启动，逐个排查 |
| `Address already in use` | 端口被占 | 先 `pm2 stop comfyui`，或 `lsof -i :8188` 找占用进程 |

**快速验 torch：**
```bash
venv/bin/python -c "import torch; print(torch.__version__)"
```

### 4.5 venv 断链

**诊断：**
```bash
cd /Users/eric/projects/ComfyUI
ls -la venv/bin/python          # 看符号链接指向
venv/bin/python --version       # 能输出版本即正常
```

**若断链：**
```bash
which python3.11 && python3.11 --version
brew list --versions | grep -i python

# 有 python@3.11 → 备份旧 venv 后重建
mv venv venv.bak
python3.11 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt

# 没有 → 先 brew install python@3.11 再重建
```

> 2026-10-29 排查中 venv 经验证**正常**（Python 3.11.17），无需重建。

### 4.6 `pip` 的 `[notice]` 提示

```
[notice] A new release of pip is available: 26.2.1 -> 26.2.2
[notice] To update, run: pip install --upgrade pip
```

这是 **pip 自身有新版本**的提示，与 ComfyUI 依赖无关，**不是错误，可忽略**。想消除就执行 `pip install --upgrade pip`，不执行也不影响。

---

## 五、查看版本与更新 ComfyUI

### 5.1 查看当前版本

```bash
cd /Users/eric/projects/ComfyUI

git describe --tags
```

| 输出形式 | 含义 |
|---|---|
| `v0.43.1`（纯 tag） | 在**正式版**上 |
| `v0.30.0-17-g2eb60976` | 在 master **开发版**上：基于 v0.30.0 后跟 17 个提交，当前 commit `2eb60976` |

辅助命令：
```bash
git log --oneline -5           # 最近 5 个提交
git branch --show-current      # 当前分支（detached HEAD 时输出为空）
git status -sb                 # 是否落后远程
cat comfyui_version.py         # 项目自带的版本文件
curl -s http://127.0.0.1:8188/system_stats   # 服务在跑时从 API 查
```

### 5.2 判断“是不是正式版”

- 在 **`master` 分支**上 → 开发版，即使 `git describe` 显示某个 tag 也是“tag 之后又跟了 N 个提交”。
- **`detached HEAD` 指向某个纯 tag** → 正式版。
- `git fetch --tags` 只更新本地的 tag/分支信息，**不改代码**；要看远程最新 tag：
  ```bash
  git fetch --tags
  git tag --sort=-v:refname | head -5
  ```

### 5.3 更新流程（标准步骤）

> 核心原则：**备份 → 切 tag → 更新依赖 → 前台验证 → 再交回 PM2**。不要直接 `git pull` 到 master 就完事。

```bash
# 1. 停服务
pm2 stop comfyui
lsof -i :8188                 # 确认端口已释放

# 2. 备份（关键，尤其跨多个版本时）
cd /Users/eric/projects/ComfyUI
cp -R custom_nodes custom_nodes.bak-$(date +%Y%m%d)
cp -R user user.bak-$(date +%Y%m%d)
# models/ 一般不受 git 影响，不必备份

# 3. 拉取 tag 信息并切到最新正式版
git fetch --tags
git tag --sort=-v:refname | head -5     # 确认最新 tag
git checkout v0.43.1                     # 换成你要的 tag

# 4. 在现有 venv 里更新依赖（不是重建 venv）
source venv/bin/activate
pip install -r requirements.txt

# 5. 前台启动验证（先别用 PM2）
python main.py
```

**验证重点：**
- 无 `ModuleNotFoundError` / `import torch` 报错
- custom_nodes 逐个加载正常（跨版本更新最易出问题处）
- 无数据库迁移报错（目录含 `alembic_db/`、`api_server/`、`app/`）
- 最后出现 `To see the GUI go to: http://127.0.0.1:8188`，浏览器能打开并跑通一个简单工作流

**验证通过后交回 PM2：**
```bash
# 前台 Ctrl+C 停掉，然后
pm2 restart comfyui
# 若提示 not found（进程被删过）：
# pm2 start main.py --name comfyui --interpreter /Users/eric/projects/ComfyUI/venv/bin/python -- --listen 127.0.0.1 --port 8188
pm2 list
pm2 logs comfyui --lines 20
```

### 5.4 关于 `detached HEAD`（切 tag 后的提示）

切换 tag 后 Git 会提示：

```
You are in 'detached HEAD' state.
HEAD is now at 086099f5 ComfyUI v0.43.1
```

**这是正常提醒，不是错误。** tag 不像分支会跟随新提交，所以 HEAD 不指向任何分支。对“只想运行这个正式版”来说正是想要的状态。

| 能做 | 不要做 |
|---|---|
| 正常启动、跑工作流、更新依赖 | 不要在此状态 `git commit`（提交会悬空） |
| 想回 master：`git switch master` | 不要在此状态直接 `git pull` |

### 5.5 更新出问题时的回滚

```bash
pm2 stop comfyui
cd /Users/eric/projects/ComfyUI

# 回到更新前的 commit（本次是 2eb60976）
git checkout 2eb60976
# 或直接回 master 分支：git switch master

source venv/bin/activate
pip install -r requirements.txt

# 若插件也出问题，恢复备份
# rm -rf custom_nodes && mv custom_nodes.bak-YYYYMMDD custom_nodes

pm2 restart comfyui
```

> `git checkout` 是原地替换被跟踪文件，**不需要删除重装**。venv、models、custom_nodes、user、output 都不在 Git 跟踪范围，切换时不受影响；唯一要重做的是 `pip install -r requirements.txt`。

### 5.6 更新后的清理（稳定几天后再做）

```bash
cd /Users/eric/projects/ComfyUI
ls -d *.bak-*                                  # 查看备份
# 确认无误后删除，释放空间：
# rm -rf custom_nodes.bak-YYYYMMDD user.bak-YYYYMMDD
```

### 5.7 2026-10-29 更新实录（可作为参照）

| 项目 | 内容 |
|---|---|
| 更新前 | `v0.30.0-17-g2eb60976`（master 开发版） |
| 更新后 | `v0.43.1`（正式版 tag，commit `086099f5`，detached HEAD） |
| 跨越版本 | 13 个 minor 版本（0.30 → 0.43） |
| 实际变更的包 | `comfyui-frontend-package` 1.48.6→1.57.4、`comfyui-workflow-templates` 0.11.31→0.11.98、`comfyui-embedded-docs` 0.5.9→0.5.15、`comfy-kitchen` 0.2.26→0.2.40、`comfy-aimdo` 0.4.13→0.6.0，以及 workflow-templates 的若干子包 |
| 未变的核心依赖 | torch 2.13.0、torchvision 0.28.0、numpy 2.4.6、transformers 5.14.1 等均 already satisfied |
| 结果 | 前台启动正常，网站可进入，custom_nodes 兼容 |

> 经验：这次跨 13 个版本风险主要在 **custom_nodes 插件**和**前端包**，核心 Python 依赖没大改，所以相对顺利。以后跨多版本更新，**备份 custom_nodes + 前台验证**这两步一定不能省。

---

## 六、日常命令速查

### 启动 / 停止

```bash
comfy                    # 启动（alias，等价于 pm2 start ...）
pm2 stop comfyui         # 停止，释放内存，保留配置
pm2 restart comfyui      # 进程存在时重启
pm2 delete comfyui       # 彻底删除配置（下次需重新 start）
```

### 状态 / 日志 / 验证

```bash
pm2 list                       # 进程状态（online / stopped）
pm2 logs comfyui               # 实时日志
pm2 logs comfyui --lines 50    # 最近 50 行
lsof -i :8188                  # 端口监听
curl -I http://127.0.0.1:8188  # HTTP 可访问
git describe --tags            # 当前版本
```

### 推荐的 alias（写入 `~/.zshrc`）

```bash
# ComfyUI 一键启停
alias comfy='cd /Users/eric/projects/ComfyUI && pm2 start main.py --name comfyui --interpreter /Users/eric/projects/ComfyUI/venv/bin/python -- --listen 127.0.0.1 --port 8188'
alias comfy-stop='pm2 stop comfyui'
```

改完 `source ~/.zshrc` 生效。相比旧 alias 的改进：① 先 `cd` 保证相对路径正确；② 用 `--interpreter` 明确指定 venv 的 python。

---

## 七、关于开机自启（当前不启用）

选择**不配开机自启**，因为不想让 ComfyUI 常驻占内存（约 600MB）。这是合理的。

**需要知道的行为：**
- 重启电脑后 PM2 daemon 不自动运行，`pm2 list` 可能为空。
- 第一次敲任意 `pm2` 命令会触发 daemon 启动，但**不保证恢复 comfyui**。
- 所以重启后直接执行 `comfy` 最稳，不要用 `pm2 restart`。

**想启用时（只需一次）：**
```bash
pm2 startup
# 复制它打印出的 sudo 命令，执行一次
pm2 save
launchctl list | grep pm2    # 有输出即生效
```
撤销：`pm2 unstartup`（按提示执行），再删 `~/Library/LaunchAgents/com.pm2.eric.plist`。

> 那条 `sudo` 命令来源是 PM2 本机生成，只写 `~/Library/LaunchAgents/` 下你用户级的 plist，不碰系统文件，风险低、可撤销。

---

## 八、本次故障复盘（典型案例）

**现象**：长时间未用后，`pm2 restart comfyui` 报 not found，网页打不开。

**排查链**：
1. `pm2 restart` 报错 → `pm2 list` 为空 → PM2 里没进程
2. `lsof -i :8188`、`ps aux | grep main.py` 均无输出 → ComfyUI 没在运行
3. `venv/bin/python --version` → Python 3.11.17，**环境正常**
4. 前台 `venv/bin/python main.py` → 成功启动

**根因**：
- 上次用 alias `comfy`（即 `pm2 start`），这次误用 `pm2 restart`（要求进程已存在）。
- Mac 重启后 PM2 daemon 重建、列表为空（未配开机自启），`restart` 必然失败。
- 环境（Python/venv/依赖）一直没问题。

**结论**：不是 ComfyUI 坏了，只是没启动 + 用错命令。`comfy` 即可解决。

---

## 九、最简应急流程（贴在显眼处）

```bash
# 1. 网页打不开时，先试这个
comfy

# 2. 还是不行，确认端口
lsof -i :8188

# 3. 端口无监听 → 前台跑，看报错
pm2 stop comfyui
cd /Users/eric/projects/ComfyUI
venv/bin/python main.py
# 把报错最后 20~30 行拿去搜索或求助

# 4. 想更新版本 → 走 §5.3 流程（备份 → git checkout <tag> → pip install → 前台验证）
```

**记住三句话**：
1. 重启电脑后用 `comfy`（start），不要用 `pm2 restart`（restart 要求进程已存在）。
2. 更新版本用 `git checkout <tag>`，不是删除重装；出问题 `git checkout <旧commit>` 回滚。
3. 更新前备份 `custom_nodes` 和 `user`，更新后前台验证通过再交回 PM2。


## 九、最后的补充说明

先澄清一个小误会：**在我们这次对话开始之前，你的 ComfyUI 就已经安装好了（时间戳显示是 8 月份），所以并不是我“当时帮你下载”的**。你当时大概率是参考了某篇网上的安装教程，或者直接使用了默认命令。

不过，你问到了一个非常核心的问题：**为什么网上的教程、甚至 ComfyUI 官方默认的下载方式，都是拉取 `master`（开发/分支版），而不是 `stable`（稳定版）？**

这背后主要有三个原因：

### 1. Git 的默认行为与教程的“偷懒”
在 GitHub 上，ComfyUI 官方仓库的默认分支就是 `master`。当你在教程里看到这条最常见的命令时：
```bash
git clone https://github.com/comfyanonymous/ComfyUI.git
```
如果你不加 `-b` 参数指定分支或 tag，Git 就会**默认拉取 `master` 分支**。绝大多数教程为了让新手“最快跑起来”，都会省略这一步，直接给这个最简命令。

### 2. ComfyUI 的生态特性：“master 才是日常版”
和传统的商业软件（比如微信、Photoshop）不同，ComfyUI 作为一个极其活跃的开源项目，它的迭代速度非常疯狂。
*   **新模型支持极快**：每当有新的 AI 模型（比如最新的 Wan、SD3.5、Flux）发布，开发团队通常会在几小时内把支持代码推送到 `master` 分支。
*   **Tag（正式版）发布滞后**：官方的 `v0.43.1` 这种 tag，往往是隔一段时间打包一次。很多玩家如果想第一时间用上新模型，就必须用 `master` 分支。
*   因此，在 ComfyUI 圈子里，**`master` 其实相当于“日常使用版”**，而 tag 是给那些“装好就不想动、追求绝对稳定”的用户准备的。

### 3. 官方包和第三方包的引导
如果你去下载官方的 Windows 便携版（Portable），或者用 ComfyUI Desktop，它们内置的更新器默认也是从 `master` 拉取更新。这导致整个社区形成了“默认跟车 master”的习惯。

---

### 那么，`master`（开发版）和 `stable tag`（正式版）到底怎么选？

| 维度 | `master` 分支（你之前用的） | 正式版 Tag（如 `v0.43.1`，你现在用的） |
| :--- | :--- | :--- |
| **新功能/新模型** | ⭐⭐⭐⭐⭐ 第一时间支持 | ⭐⭐ 滞后，需要等官方打新 tag |
| **稳定性** | ⭐⭐⭐ 偶尔会有半成品代码或小 Bug | ⭐⭐⭐⭐⭐ 经过官方测试打包，相对最稳 |
| **适合人群** | 喜欢尝鲜、急着用新模型、有一定排错能力 | 追求稳定、只想安静跑图、不想折腾 |

### 以后怎么选？

*   **如果你想用最新出的模型**，发现现在的 ComfyUI 不支持，你可以随时切回 `master` 并拉取最新代码：
    ```bash
    cd /Users/eric/projects/ComfyUI
    git switch master
    git pull
    source venv/bin/activate
    pip install -r requirements.txt
    ```
*   **如果你只想稳定出图**，那就保持现在这样（在 `v0.43.1` 上），不折腾。
*   **如果未来你想更新到下一个正式版**（比如 `v0.44.0`），就按我们指南里的办法：
    ```bash
    git fetch --tags
    git checkout v0.44.0
    ```

**总结一下**：当初拉取分支版，是因为教程默认这么教，且整个 ComfyUI 社区都把 `master` 当作日常版在用。你现在已经切到了稳定版 `v0.43.1`，这是一个非常稳妥的避风港，先在这个版本上安稳使用即可。



### 判断哪个才是更好的版本
这是一个非常好的问题。你之前运行 `git fetch --tags` 时，终端刷出过几百行 `[new branch]`，看着确实吓人。

但真相是：**Git 在拉取的时候，它自己根本不“判断”哪个更好，它就是个死脑筋的工具。**

为了让你彻底不再被这堆分支困扰，我们来拆解一下这背后的逻辑，以及以后你怎么一眼挑出“好版本”。

### 1. 那么多分支到底是什么？

你在终端里看到的那几百个分支（如 `synap5e/feat/assets-xxx`、`alexis/chores`、`feat/partner-nodes/hunyuan-image`），**99% 都是全世界各地的开发者在提交代码时的“草稿本”**。

*   **官方是怎么工作的**：开发者 A 想加个新功能，他会从主代码复制一份，建个自己的分支（比如叫 `alexis/add-new-feature`）。在里面改好、测试没崩之后，向官方提交合并申请（PR）。
*   **为什么要放出来**：这些分支存在于远程仓库，是为了方便官方审查代码、跑自动化测试。
*   **对你的意义**：**完全不用管**。这些分支随时会变、随时会被删，代码可能是半成品。

### 2. 为什么你当时会拉到分支版？

Git 的判断逻辑极其简单：**如果你不说要哪个，它就去拉官方默认的主分支**（ComfyUI 官方设定的默认主分支叫 `master`）。

你当时输入 `git clone` 的时候，没指定分支，Git 就老老实实把 `master` 给你拉下来了。它并没有“智能挑选”一个最好的，只是执行了默认项。

### 3. 怎么判断哪个才是真正的好版本？

在 ComfyUI 这种项目里，版本分为四个层级。你以后只需要认准 **前两个** 就行：

| 层级 | 名字长什么样 | 是什么 | 适合谁 |
| :--- | :--- | :--- | :--- |
| **1. 正式标签 (Tag)** | `v0.43.1`、`v0.42.0` | 官方打包的**里程碑版本**。经过测试，相对最稳。 | **绝大多数用户（包括现在的你）** |
| **2. 主干分支 (Branch)** | `master` | **日常开发版**。所有新功能都往这塞，相对新，但也容易有 Bug。 | 尝鲜党、急需新模型支持的人 |
| **3. 版本维护分支** | `release/v0.43` | 官方用来给老版本打补丁用的。 | 极少数遇到特定 Bug 需要修复的人 |
| **4. 开发草稿分支** | `feat/xxx`、`名字/xxx` | 开发者提交的未完成代码。 | 只有开发者自己。**千万别碰** |

### 4. 以后如何一眼找到最好/最新的版本？

不用去拉成百上千的分支，你只需要看 **Tag（标签）**。

在终端里运行这条命令：
```bash
git tag --sort=-v:refname | head -5
```
或者直接看 GitHub 页面的右侧 **Releases** 栏。官方觉得代码足够稳定了，就会给 `master` 打上一个 tag（比如 `v0.43.1`），这就是官方替你做好的“判断”。

### 总结

*   拉取代码时，**不要用网页上的 Update 按钮**，也不要在终端随便 `git pull`。
*   用 `git checkout v0.43.1`，这就等于直接告诉 Git：“我不管那些乱七八糟的分支，我就要这个官方指定的版本。”
*   你现在就在 `v0.43.1` 这个最完美的避风港里。以后哪怕远程仓库又多了几百个乱糟糟的分支，只要你不动，你的代码就永远是 `v0.43.1`，稳如泰山。