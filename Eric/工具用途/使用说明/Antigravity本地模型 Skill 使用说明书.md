---
标题: Antigravity本地模型 Skill 使用说明书
创建时间: 2026-08-06 22:33
修改时间: 2026-08-06 22:33
引用渠道: antigravity
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---

# 本地模型 Skill 使用说明书

## 1. 简介
本 Skill 用于在 Antigravity IDE 与 Antigravity 2.0 桌面版之间统一管理本地大模型的调用与切换。新增模型已替换原有模型，并提供精简别名，方便快捷使用。

## 2. 精简别名映射表
| 别名 | 完整模型标识 | 模型类型 | 推荐场景 |
|------|------------------------------|--------|---------------------------|
| `qwo` | `Qwen3.5-27B-Claude-4.6-Opus-Distilled-MLX-4bit` | LLM | 通用高质量问答、Claude 蒸馏写作 |
| `qw3` | `Qwen3-Coder-30B-A3B-Instruct-MLX-4bit` | LLM | 超大规模代码生成、复杂编程辅助 |
| `qw2.5` | `Qwen2.5-Coder-14B-Instruct-4bit` | LLM | 中等规模高效代码生成、重构 |
| `24b` | `Devstral-Small-2-24B-Instruct-2512-4bit` | LLM | 多语言对话、指令理解 |
| `gm12b8` | `gemma-4-12B-it-8bit` | VLM | 高精度视觉/多模态对话 |
| `gm12b4` | `gemma-4-12B-it-OptiQ-4bit` | VLM | 轻量视觉解析 |
| `gm4b` | `gemma-4-e4b-it-OptiQ-4bit` | LLM | 超低内存对话、快速总结 |
| `gpt` | `gpt-oss-20b-MXFP4-Q8` | LLM | 开源高精度文本处理 |
| `gm26b` | `diffusiongemma-26B-A4B-it-4bit` | VLM | 多模态图像生成 |

## 3. 快捷指令
### 3.1 切换并锁定默认模型（多轮对话锁定）
```
/sl <别名>
```
示例：
- `/sl qw3` → 将会话默认模型锁定为 **qw3**（后续所有对话自动使用该模型）
- `/sl qwo` → 锁定为 **qwo**

**解锁回云端**：`/cloud` 或 `/reset-cloud`

### 3.2 单条消息临时调用（不改变锁定状态）
```
/lm <别名> <问题描述>
```
示例：
- `/lm qw3 帮我实现一个高性能排序算法`
- `/lm qw2.5 重构这段 Python 代码`
- `/lm 24b 用中文解释量子纠缠`

## 4. 使用场景示例
1. **一次性任务**：`/lm qwo 给我写一段产品介绍`
2. **连续对话**：先执行 `/sl qw3`，随后直接输入自然语言，模型会自动保持 **qw3** 响应。
3. **切回云端**：当需要更强的检索或多模态功能时，输入 `/cloud` 即可恢复 Gemini 云端模型。

## 5. 常见问题排查
- **指令未生效**：确认已经打开项目根目录或使用全局软链接，使 IDE 能读取 `~/.gemini/antigravity/skills/local-model`。
- **别名拼写错误**：别名必须全小写，如 `qw3`、`24b`，否则会被解析为未知模型。
- **模型未加载**：确保对应模型已通过 LiteLLM 正确注册，运行 `litellm list_models` 检查可用模型列表。

## 6. 维护
- 若新增或删除模型，只需编辑 `~/.gemini/antigravity/skills/local-model/SKILL.md` 中的 **模型别名映射表**，保存后重启 IDE 即可生效。
- 建议使用软链接共享全局 Skill，避免每个项目重复维护同一份配置。

---
*本使用说明已保存至本次会话的 Artifact 目录，可在以下路径查看：* `file:///Users/eric/.gemini/antigravity/brain/757e0c5d-1d32-429d-8c4c-95e39a9c2c35/local-model_skill_usage.md`
