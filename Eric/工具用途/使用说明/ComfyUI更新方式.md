---
标题: ComfyUI更新方式
创建时间: 2026-11-07 21:51
修改时间: 2026-11-07 21:51
引用渠道: DeepSeek
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---

确实，GitHub 上已经发布了 **`v0.44.0`**（带有 `Latest` 标签）。

从你现在的 `v0.43.1` 升级到 `v0.44.0` 属于相邻版本的小幅更新，风险比上次跨 13 个版本要小得多，但由于涉及前端和插件，**还是建议按照我们之前定好的“标准流程”来操作**。

请打开终端，按顺序执行以下步骤：

### 第一步：停服务并备份
```bash
# 1. 停掉当前在跑的 ComfyUI
pm2 stop comfyui

# 2. 进入目录
cd /Users/eric/projects/ComfyUI

# 3. 备份（虽然是小更新，但备份 custom_nodes 是个好习惯）
cp -R custom_nodes custom_nodes.bak-v0.43.1
```

### 第二步：拉取最新代码并切换
```bash
# 4. 获取最新的 tag 信息
git fetch --tags

# 5. 切换到新版本 v0.44.0
git checkout v0.44.0

# 6. 确认当前版本（应该输出纯净的 v0.44.0）
git describe --tags
```

### 第三步：更新依赖
```bash
# 7. 激活虚拟环境
source venv/bin/activate

# 8. 安装/更新依赖（如果有变化会下载，没有变化会显示 already satisfied）
pip install -r requirements.txt
```

### 第四步：前台测试（关键！）
```bash
# 9. 前台启动
python main.py
```
启动后，观察终端输出：
- 如果没有报错，最后出现 `To see the GUI go to: http://127.0.0.1:8188`。
- 打开浏览器，进去随便跑一个简单的工作流，确保出图正常，插件面板能正常打开。
- *顺便看看之前那些 `DEPRECATION WARNING` 有没有变少，或者有没有引发新的报错。*

### 第五步：交回 PM2 托管
如果前台测试一切正常：
1. 在终端按 `Ctrl+C` 停掉前台进程。
2. 重新用 PM2 启动：
```bash
pm2 restart comfyui
pm2 list
```

---

### 🚨 如果更新后出问题（回滚方案）
万一 v0.44.0 出现严重 Bug 导致无法使用，立刻执行以下命令退回今天的版本：
```bash
pm2 stop comfyui
cd /Users/eric/projects/ComfyUI
git checkout v0.43.1
source venv/bin/activate
pip install -r requirements.txt
pm2 restart comfyui
```

祝你更新顺利！过程其实很快，遇到卡住的地方随时发我。