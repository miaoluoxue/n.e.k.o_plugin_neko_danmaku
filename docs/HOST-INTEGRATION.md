# 宿主接入与权限清单

`neko_danmaku` 把宿主聊天以弹幕形式绘制到游戏/窗口上。当前清单只声明面板
`state:read`、`action:call`，但代码还使用对话通道、本机窗口枚举和本地 HTTP。

> 红线结论：不写 N.E.K.O 本体记忆库、对话数据库或角色记忆卡。本插件不生成
> 新的猫娘台词，因此不调用 `push_message`；弹幕只读取宿主已经生成的对话。

## 高风险权限使用矩阵（发布前必读）

本表按当前代码事实描述，**不把“需要用户确认”写成“默认关闭”**。

| 高风险能力 | 使用插件 | 为什么必须 | 当前默认状态 | 用户确认与限制 | 回滚 |
|---|---|---|---|---|---|
| 宿主记忆/对话写入 | 无 | 没有此能力也能完成弹幕 | 未使用 | 兼容 SQLite 连接强制 `mode=ro`，只执行 `SELECT` | 无需回滚 |
| 猫娘回复/隐藏上下文 | 本插件不适用 | 弹幕读取宿主已有台词，不生成新台词 | 不调用 `push_message` | 默认 `sdk` conversations 只读；不把内容上传第三方 | 停用插件或切换回 `sdk` |
| 代理 DLL / UE4SS | 仅 `neko_pbz` | PBZ 游戏侧状态和字幕需要 UE4SS 加载自有 Mod | `neko_danmaku` 未使用 | 本插件随包 PySide6 DLL 只是 GUI 运行时，不是游戏代理 DLL | 移除插件包即可 |
| 键鼠输入注入 | 仅 `neko_pawpilot` | 用户明确同意后接力驾驶 | `neko_danmaku` 未使用 | 本插件不注入键鼠 | 不适用 |
| 启动游戏进程 | 仅 `neko_terraria` | 拉起用户主动连接的 AI 游戏客户端 | `neko_danmaku` 未使用 | 本插件不启动游戏进程 | 不适用 |
| 屏幕截图 | `neko_pbz`、`neko_fishpower`、`neko_pawpilot`、`neko_terraria` | 游戏视觉侦察或 HUD OCR | `neko_danmaku` 未使用 | 本插件不做桌面截图，只枚举用户选择的目标窗口 | 不适用 |
| 私有照片/文件读写 | `neko_arcade`、`neko_pawpilot` 保存照片；六个插件均有各自的数据目录 | 图库、截图、存档和游标必须持久化 | 只在功能使用或插件运行需要时写入 | 仅写插件 `data_path()`/`cache_path()`，不扫描私人相册，不写宿主记忆库 | 删除对应插件数据目录 |

## 声明=实际

## 宿主能力

| 能力 | 实际用途 | 当前声明 |
|---|---|---|
| `@lifecycle` | 启动/停止 Qt 浮层与对话桥；配置变化时热重启 | 隐式 |
| `@plugin_entry` | 面板查询、连接测试、目标窗口选择、数据源切换 | `action:call` |
| `ctx.bus.conversations.get()` | 增量读取主人和猫娘的对话回合 | 未单列 |
| `data_path()` | 保存覆盖层样式、游标和状态 | 未单列 |
| 静态 UI | 配置浮层、目标窗口、字体、速度和显示效果 | `state:read` |

本插件不通过 `push_message` 生成猫娘台词；弹幕内容来自宿主已有对话。

## 兼容数据源

为旧宿主保留两个只读回退：

- `memory/<角色>/time_indexed.db`：只读 `time_indexed_original`；
- `recent.json`：只读最新对话窗口。

默认优先 `sdk` conversations。回退源只读，不写、不改、不迁移宿主记忆库。

## 本机能力

| 能力 | 用途 | 约束 |
|---|---|---|
| Qt/PySide6 浮层 | 透明、鼠标穿透的弹幕窗口 | 在插件进程内 daemon 线程运行 |
| Win32 窗口枚举 | 让用户选择游戏窗口或显示器 | 只用于目标选择，不注入进程 |
| 本地 HTTP `127.0.0.1` | 插件与浮层之间推送文本、配置和关闭信号 | 仅 loopback，不监听公网 |
| 插件数据目录 | 样式、游标、状态 | 不缓存完整聊天历史 |

明确不使用：游戏 Hook、代理 DLL、键鼠注入、进程内存读取、远程网络上传。
随包 PySide6 DLL 是 GUI 运行时，不是注入游戏的代理 DLL。

## 红线

- 不写 N.E.K.O 本体记忆库、对话库或角色记忆卡。
- 不修改宿主聊天记录；旧数据源只读。
- 不扫描与当前弹幕目标无关的窗口内容。
- 不把聊天内容上传第三方服务。

## 权限差距与回滚

云玩家权限模板发布后，应把 `conversations` 读取、数据目录和本地 loopback
服务映射为正式能力。回滚方式：

- 关闭 `auto_start_overlay` 并停用插件；
- 将 `data_source` 固定为 `sdk`；
- 删除 `state/bridge_cursor.json` 重置增量游标；
- 浮层随插件 shutdown 退出，不常驻独立进程。

## 完整权限清单

下表 ID 是本插件的审计标签，不是当前 `plugin.toml` 已有的正式权限字段。

| 审计 ID | 权限 | 实际 API/资源 | 降级 |
|---|---|---|---|
| `HOST_LIFECYCLE` | 生命周期与配置热更新 | `@lifecycle startup/shutdown/config_change` | 浮层和桥停止 |
| `HOST_ENTRY` | 面板操作 | `@plugin_entry`、静态 UI | 只保留已有浮层 |
| `HOST_CONVERSATION_BUS` | 对话回合读取 | `ctx.bus.conversations.get(since_ts=...)` | 回退兼容文件源 |
| `HOST_MEMORY_DB_READ` | 宿主时间索引只读 | `memory/<角色>/time_indexed.db` 的 `SELECT` | 找不到时桥停用 |
| `HOST_RECENT_FILE_READ` | 宿主最近对话读取 | `recent.json` | 找不到时桥停用 |
| `HOST_DATA_PATH` | 插件私有状态 | `plugin.data_path()` | 旧目录兼容 |
| `FS_PLUGIN_STATE_RW` | 状态/样式游标 | `config/overlay_style.json`、`state/bridge_cursor.json` | 丢失时从当前尾部开始 |
| `UI_QT_OVERLAY` | 桌面浮层 | 随包 PySide6、Qt 事件循环 | 缺少运行时则浮层不可用 |
| `WIN_WINDOW_ENUM` | 目标窗口枚举 | Win32 窗口句柄、标题、进程信息 | 无法选择时仅显示器模式 |
| `NET_LOOPBACK_HTTP` | 插件到浮层通信 | `127.0.0.1:<overlay_port>` | 端口占用时启动失败 |

明确**不使用**：`push_message` 生成新台词、宿主记忆写操作、游戏 Hook、代理 DLL、
键鼠输入、游戏内存读取、远程服务器上传。

红线审计：不写宿主记忆库、不修改对话记录；两个旧数据源均只读。
`time_indexed.db` 每次建立连接都使用 SQLite URI `file:<path>?mode=ro` 和
`uri=True`，源码只执行 `SELECT`；`tests/test_sources.py` 有回归测试锁定该行为。

