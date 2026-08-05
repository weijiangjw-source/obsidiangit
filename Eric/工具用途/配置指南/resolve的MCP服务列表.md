---
标题: resolve的MCP服务列表
创建时间: 2026-08-05 20:22
修改时间: 2026-08-05 20:22
引用渠道: claude code
是否修改: true
标签: []
参考资料: ""
难度: 简单
状态: 已成功
---

# DaVinci Resolve MCP (Resolve) Skills

  

以下列出了 **DaVinci Resolve MCP**（*Resolve* 服务器）提供的所有工具（可以视为“技能”），并按功能类别做了简要说明。

> **说明**：所有工具均以 **`mcp__resolve-mcp__`** 为前缀，表示它们在 Resolve 的 MCP 服务器上运行。

---

## 1️⃣ 项目与数据库

| 工具 | 作用 |
|------|------|
| `resolve_get_info` | 获取当前项目的基本信息（项目名、时间线数量、Bin 树等） |
| `resolve_load_project` | 打开指定项目（会关闭当前项目） |
| `resolve_save_project` | 保存当前项目 |
| `resolve_list_projects` | 列出当前数据库中的所有项目 |
| `resolve_create_project` | 创建新项目 |
| `resolve_delete_project` | 删除项目 |
| `resolve_set_current_database` | 切换数据库 |
| `resolve_get_database_list` | 列出所有可用数据库 |
| `resolve_list_db_folders` | 列出数据库文件夹结构 |

---

## 2️⃣ 画布 & 界面

| 工具 | 作用 |
|------|------|
| `resolve_switch_page` | 跳转至指定页面（media、cut、edit、fusion、color、fairlight、deliver） |
| `resolve_get_product_name` | 返回 Resolve 版本与产品名 |
| `resolve_get_version` | 详细版本信息 |

---

## 3️⃣ 资源 & 媒体池

| 工具 | 作用 |
|------|------|
| `resolve_list_bins` | 列出媒体池 Bin 及其子夹 |
| `resolve_import_media` | 直接把媒体文件导入媒体池 |
| `resolve_import_folder_to_media_pool` | 递归导入文件夹 |
| `resolve_import_project` | 导入 `.drp` 项目 |
| `resolve_import_render_preset` | 导入渲染预设 |
| `resolve_import_burn_in_preset` | 导入烧录预设 |
| `resolve_import_layout_preset` | 导入界面布局预设 |
| `resolve_set_clip_color` | 给媒体池 Clip 设置颜色标签 |
| `resolve_set_clip_metadata` | 给媒体池 Clip 写入自定义元数据 |
| `resolve_clear_transcription` | 清除媒体池 Clip 的转录数据 |
| `resolve_clear_flags` | 清空 Clip 标记 |
| `resolve_link_clips` / `resolve_unlink_clips` | 链接/解除链接媒体 |
| `resolve_import_timeline_from_file` | 从外部文件导入时间线 |
| `resolve_search_clips` | 按名称搜索媒体池中的 Clip |

---

## 4️⃣ 轨道 & 片段

| 工具 | 作用 |
|------|------|
| `resolve_add_track` | 给当前时间线添加新轨道（video/audio/subtitle） |
| `resolve_set_track_enabled` / `resolve_set_track_locked` | 启用/锁定轨道 |
| `resolve_set_track_name` | 重命名轨道 |
| `resolve_list_clips_on_track` | 列出指定轨道的所有 Clip |
| `resolve_move_clips` | 把 Clip 移到其它 Bin |
| `resolve_set_clip_color_on_timeline` | 给时间线上的 Clip 设置颜色标签 |
| `resolve_set_clip_enabled` | 启用/禁用时间线 Clip |
| `resolve_create_compound_clip` | 创建复合 Clip |
| `resolve_add_marker` / `resolve_delete_marker` | 在轨道或 Clip 上添加/删除标记 |
| `resolve_get_marker_data` | 读取标记的自定义数据 |
| `resolve_update_marker_data` | 更新标记自定义数据 |
| `resolve_get_clip_info` / `resolve_get_clip_source_info` | 读取 Clip 的详细信息 |
| `resolve_get_item_properties` / `resolve_set_item_properties` | 读取/写入时间线项目属性（transform、复合模式等） |
| `resolve_get_item_flags` / `resolve_set_item_flags` | 读取/写入项目标记 |
| `resolve_get_item_id` | 获取项目的唯一 ID |
| `resolve_apply_grade_from_album` | 给 Clip 应用已存在的调色 |
| `resolve_set_color_group` / `resolve_get_color_group` | 给 Clip 设置/读取颜色分组 |
| `resolve_add_version` / `resolve_delete_version` / `resolve_load_version` | 管理 Clip 版本 |
| `resolve_stabilize_clip` | 稳定时间线 Clip |
| `resolve_smart_reframe` | 自动裁剪适应不同宽高比 |

---

## 5️⃣ 颜色 & LUT

| 工具 | 作用 |
|------|------|
| `resolve_get_node_count` | 获取当前 Clip 颜色节点数 |
| `resolve_node_overview` | 列出所有节点信息 |
| `resolve_node_get_label` / `resolve_node_get_enabled` | 读取/设置节点标签、启用状态 |
| `resolve_set_cdl` | 设置 CDL 值 |
| `resolve_get_lut` | 读取节点的 LUT 路径 |
| `resolve_apply_lut` | 给节点应用 LUT |
| `resolve_reset_grades` | 重置所有颜色节点 |
| `resolve_optimize_dolby_vision` | 为 Dolby Vision 优化工作流 |

---

## 6️⃣ Fusion

| 工具 | 作用 |
|------|------|
| `resolve_insert_fusion_composition` / `resolve_insert_fusion_generator` / `resolve_insert_fusion_title` | 在时间线插入 Fusion 片段 |
| `resolve_item_add_fusion_comp` / `resolve_item_load_fusion_comp` / `resolve_item_rename_fusion_comp` / `resolve_item_delete_fusion_comp` | 管理时间线项目上的 Fusion 片段 |
| `resolve_item_import_fusion_comp` / `resolve_item_export_fusion_comp` | 导入/导出 .comp 文件 |
| `resolve_insert_generator` / `resolve_insert_ofx_generator` / `resolve_insert_title` | 插入内置生成器或 OFX 插件 |
| `resolve_set_render_settings` / `resolve_get_render_settings` | 配置渲染设置 |
| `resolve_get_render_status` / `resolve_start_render` / `resolve_stop_render` | 查看/开始/停止渲染任务 |
| `resolve_get_render_mode` / `resolve_set_render_mode` | 切换渲染模式（单片段/全部片段） |

---

## 7️⃣ Fairlight (音频)

| 工具 | 作用 |
|------|------|
| `resolve_set_voice_isolation` / `resolve_get_voice_isolation` | 启用/读取音频轨道的语音隔离功能 |
| `resolve_insert_audio_at_playhead` | 在播放头位置插入音频文件 |
| `resolve_auto_sync_audio` | 对音频轨道进行时间码/波形同步 |

---

## 8️⃣ 标记 & 事件

| 工具 | 作用 |
|------|------|
| `resolve_add_marker_at` | 在时间线起始位置添加标记 |
| `resolve_delete_marker_at` | 删除标记 |
| `resolve_delete_markers` | 批量删除标记 |
| `resolve_update_marker_data` | 更新标记自定义数据 |
| `resolve_list_markers` | 列出时间线或 Clip 标记 |
| `resolve_get_marker_data` | 读取标记的自定义数据 |
| `resolve_create_subtitles` | 自动生成字幕（需配合 AI） |

---

## 9️⃣ 其他实用工具

| 工具 | 作用 |
|------|------|
| `resolve_get_clip_info` | 读取 Clip 详细属性 |
| `resolve_get_clip_source_info` | 读取 Clip 的源文件路径等 |
| `resolve_get_clip_id` | 取得 Clip ID |
| `resolve_get_clip_flags` / `resolve_clear_flags` | 管理 Clip 标记 |
| `resolve_get_clip_metadata` | 读取 Clip 元数据 |
| `resolve_set_clip_metadata` | 写入 Clip 元数据 |
| `resolve_clip_link_proxy` / `resolve_clip_unlink_proxy` | 链接/解除代理媒体 |
| `resolve_clip_replace` | 用新文件替换 Clip 源 |
| `resolve_clip_transcribe` | 开始音频转录（需 Studio） |
| `resolve_clip_apply_lut` | 给媒体池 Clip 应用 LUT |
| `resolve_clip_get_lut` | 读取媒体池 Clip LUT |

---

> **提示**：上述工具需 Resolve 已开启并处于可脚本状态（“外部脚本使用网络”开启）。
> **版权**：本列表基于官方 Resolve MCP 文档，内容仅供学习与参考。
