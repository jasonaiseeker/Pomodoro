# 托盘与应用生命周期提案

状态：草案，待 UI 原型、公共接口与关键架构审批；业务实现及实机验证未开始。
来源：GitHub Issue [#12](https://github.com/troywjz/Pomodoro/issues/12)、`docs/interface-contract.md`、`docs/requirements.md`、`docs/acceptance.md`。

## 官方依据与版本候选

- Slint 官方 `SystemTrayIcon` 提供原生系统托盘入口：Windows 使用 Shell notification area / `Shell_NotifyIcon`；图标实例创建后在 event loop 运行时注册，释放实例后移除。托盘为顶层组件，需与主窗口分别创建；两者的 Slint globals 各自独立。可见托盘图标可维持事件循环运行。[Slint SystemTrayIcon](https://docs.slint.dev/latest/docs/slint/reference/window/systemtrayicon/)
- Slint 官方 Rust 文档要求大多数后端在主线程运行 event loop，UI 组件在该线程创建；后台线程通过 `invoke_from_event_loop` 投递 UI 工作。[Slint Rust threading/event loop](https://docs.slint.dev/latest/docs/rust/slint/)
- Slint `run_event_loop()` 在最后一个窗口关闭且没有可见托盘图标时返回；`run_event_loop_until_quit()` 则等显式退出。[Slint run_event_loop](https://docs.slint.dev/latest/docs/rust/slint/fn.run_event_loop)
- Windows Shell 在 `TaskbarCreated` 广播后，应用应假设旧托盘图标已移除并重新注册。[Microsoft Taskbar](https://learn.microsoft.com/en-us/windows/win32/shell/taskbar)
- Windows named mutex 可用于单实例互斥；named pipe 可用于进程间通信，但需定义 ACL，且本地管道应拒绝网络访问。[CreateMutex](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createmutexa)、[Named Pipe Security](https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipe-security-and-access-rights)

版本候选：采用 Slint 官方当前稳定文档的 `SystemTrayIcon` API；截至本提案查询时 Rust API 文档展示 Slint 1.19.0，候选依赖为 Slint 1.19.x。锁定具体 patch、Rust 工具链、feature 与许可证清单须在工具链/架构审批中重新核验。托盘 feature 需显式确认启用，候选 `system-tray`；Windows 后端行为以获批版本官方文档和实际构建验证为准。没有实测记录。

## 生命周期状态

应用状态由主线程应用协调器单一持有；托盘注册状态与窗口可见状态作为 Shell 状态，不复制计时状态。

| 状态 | 说明 | 可接受事件 |
|---|---|---|
| Starting | 获得本用户单实例锁；创建协调器、窗口和托盘 | 初始化成功/失败、二次启动激活请求 |
| ReadyVisible | 主窗口可见，托盘可用 | Hide、CloseRequested、TrayActivate、ExitRequested |
| ReadyHidden | 主窗口隐藏，托盘可用；计时由 TimerCore 按状态继续 | TrayActivate、TrayShow、ExitRequested、TaskbarCreated |
| TrayDegradedVisible | 托盘初始化或恢复失败；强制主窗口可见并显示错误 | TrayRetry、TrayActivate、ExitRequested、CloseRequested |
| ExitConfirmPaused | 退出确认打开前先将活动段暂停并持久化；窗口可见 | SaveAndExit、DiscardAndExit、CancelExit |
| Exiting | 已选择退出，等待要求的持久化回执并清理 Shell 资源 | PersistSucceeded/Failed、CleanupComplete |
| Stopped | 释放互斥量及资源，event loop 返回 | 无 |

窗口关闭、托盘菜单退出、第二实例唤醒均须通过协调器命令；UI 和托盘 callback 不直接调用数据库或 Windows API。重复/过期命令遵循 `command_id` 与 `expected_revision` 契约。

## 命令与结果表

| 来源与命令 | 协调器动作 | 成功结果 | 失败处理 |
|---|---|---|---|
| 主窗口 `CloseRequested` | 若托盘可用，`SetVisible(false)`；TimerCore 状态不变 | ReadyHidden；计时照常按内核时钟推进 | 隐藏失败时保持可见并提示 |
| 托盘左键 `TrayActivate` / 菜单“显示” | 同一 UI 线程调用窗口 show，恢复至当前可见工作区；按窗口策略请求前置 | ReadyVisible；重算窗口位置后呈现 | show/定位失败提示并保持可操作入口 |
| 托盘菜单“隐藏” | 隐藏主窗口 | ReadyHidden | 失败保持可见 |
| 托盘菜单“退出”或 UI 退出 | 派发 `ExitRequested`；运行/暂停/恢复中段先暂停并等待保存回执，再展示退出选择 | ExitConfirmPaused | 暂停持久化失败时停留可见，提示重试，不进入可丢失状态 |
| `SaveAndExit` | 保存 paused 快照，收到提交成功回执后清理托盘/窗口并退出 loop | Stopped；下次启动恢复为 paused | 保持窗口/托盘可用，显示保存错误和重试 |
| `DiscardAndExit` | 写入 interrupted 历史并确认提交，再清理退出 | Stopped | 保留进程及数据，提示失败 |
| `CancelExit` | 不恢复运行，关闭确认页，保持 paused 并显示继续按钮 | ReadyVisible；paused | UI 更新失败时维持暂停并重试 UI 同步 |
| 空闲或 completed 状态退出 | 保存必要设置/快照，确认回执后退出 | Stopped | 错误提示、保持运行 |
| 第二实例 `ActivateRequest` | 主实例接收 IPC；串行派发 show/focus 命令 | 主窗口显示，已存在进程继续唯一计时 | 请求超时向启动方返回失败；启动方退出，不创建第二计时器 |
| Windows `TaskbarCreated` | 标记旧图标失效并重新注册图标 | 托盘恢复，窗口状态保持 | TrayDegradedVisible；呈现错误与退出入口 |

### 退出选择语义

- 用户触发退出后，若有活动阶段，先发 TimerCore `Pause(reason=user)` 并持久化；确认页显示阶段、剩余时间及选择影响。
- “保存后退出”保留 paused 恢复快照，不记 interrupted。
- “放弃后退出”记录 interrupted，再结束进程。
- “取消”关闭确认并保持 paused；用户可明确点击继续。
- idle/completed 可直接持久化后退出；recovery 退出沿用保存快照，仍需保留状态提示。
- 退出不因关闭窗口触发；隐藏与进程结束是两条明确路径。

## 线程归属与消息路由

| 线程/组件 | 归属与职责 |
|---|---|
| 主线程 Slint event loop | 创建/持有 Slint Window、SystemTrayIcon、协调器；接收窗口关闭、托盘 callback、系统消息与 IPC 投递；串行处理命令和发布快照。长 I/O 由后台线程处理。 |
| WindowsShell adapter | Windows 事件接收与 Shell 操作。若所选 Slint 平台后端能提供所需的 TaskbarCreated / 托盘失败通知，直接桥接；否则需在审批后评估最小原生窗口消息适配。Shell 回调只形成事件，回投协调器。 |
| IPC listener | 后台线程监听当前用户范围激活请求；限制消息为版本化 `Activate` 请求并设置长度/超时；用 `slint::invoke_from_event_loop` 投递。IPC 失败只影响第二实例激活，不创建第二协调器。 |
| TimerCore / Repository workers | 依现有接口契约执行纯状态计算与持久化；结果携带 revision 回投主线程，主线程拒绝陈旧结果。写入失败沿用暂停并展示错误语义。 |

消息路由：UI/Tray/Windows/IPC → `AppCommand` 或 `ShellEvent` → 主线程协调器校验前置状态与 revision → TimerCore/Repository effect → 结果回投 → 协调器更新单一 Snapshot 与窗口可见状态 → Slint 与托盘菜单读取同一 Snapshot。托盘菜单只读取协调器发布的 Snapshot 与 session 状态。

单实例候选：每用户命名互斥量负责选主；获胜者启动本地 named pipe listener；后续启动者连接并发送 `Activate` 后退出。管道 ACL 限定当前用户 SID，拒绝远程客户端，协议只接受有限、版本化消息。命名规则、启动竞争超时、管道 ACL 代码方案、安全审查及打包模式待架构审批。

## 错误及回退规则

- 初始托盘创建失败：应用进入 TrayDegradedVisible，主窗保持可见；窗口显示“托盘不可用”，提供“重试托盘”和始终可见的退出入口。CloseRequested 被转换为最小化/保持可见并说明原因。
- Explorer 重启：接收 TaskbarCreated 后重新注册托盘；注册期间主窗口现有可见性不变。连续失败进入降级态，用户可见提示并可重试。
- Shell 重建与退出竞态：Exiting 后忽略新的重建请求；先等待持久化决议，再删除托盘图标并释放组件。
- IPC 互斥量创建错误：显示单实例启动错误并停止初始化 UI，记录可诊断错误；互斥量名称冲突及被预先创建风险通过 ACL/名称策略审查。
- IPC client/listener 错误：已运行实例不受影响。第二次启动尝试连接既有进程；连接失败则显示可操作错误并结束该启动请求，防止重复计时实例。
- `invoke_from_event_loop` 失败或 event loop 已关闭：后台任务停止提交 UI 更新；关闭过程以持久化回执为退出门槛，避免无主异步任务。
- 托盘菜单“退出”与窗口关闭并发：协调器按事件队列顺序接受一个退出事务；相同 `command_id` 返回既有结果，重复确认不重复写历史。

## Windows 实机验收计划

状态：全部待执行；当前仅为计划，未构建、未安装软件、未做实现试验。

| ID | 操作 | 预期观察与证据 |
|---|---|---|
| TR-01 | 首次启动后观察窗口、通知区域图标、左键与右键菜单 | 单窗口单图标；左键显示，右键菜单项可操作；记录 Windows 版本、Slint/Rust 构建标识及截图 |
| TR-02 | 运行计时后点窗口关闭，再由托盘显示 | 窗口隐藏后计时按 TimerCore 继续；显示后剩余时间一致，无第二计时实例 |
| TR-03 | 分别在 running、paused、recovery 和 idle 选择退出 | 活动段先暂停；保存/放弃/取消路径符合上节语义；验证实际持久化记录与退出进程状态 |
| TR-04 | 用第二进程再次启动应用 | 原进程窗口显示并获得预期焦点；第二进程退出；进程列表和历史只有一个活动会话 |
| TR-05 | Explorer 正常重启期间保持应用隐藏并运行 | Explorer 重启后托盘图标自动恢复；应用和计时进程继续运行；显示/退出菜单可操作 |
| TR-06 | 受控条件阻止初次托盘注册、再恢复条件并点重试 | 窗口保持可见，错误与退出入口可操作；重试成功回到可托盘状态，不丢失 Snapshot |
| TR-07 | Explorer 重启后模拟重新注册失败 | 显示降级状态及可执行下一步；已有隐藏状态回退为可见窗口，运行状态由 TimerCore 决定 |
| TR-08 | 托盘菜单退出与窗口关闭快速并发触发 | 一个退出流程、一次历史转换；不挂起、不重复持久化、不失联 |
| TR-09 | 托盘图标被 Windows 溢出区隐藏、用户关闭主窗口 | UI 明确说明应用仍在运行，测试人员从通知区域溢出入口恢复；记录用户可发现性 |
| TR-10 | Windows 注销/关机请求发生于 running 与写入期间 | 系统关闭处理符合批准策略；退出前保存回执或呈现保存失败；证据记录系统日志和应用状态 |

执行记录字段沿用 `docs/acceptance.md`：日期、Windows 版本/补丁、屏幕和 DPI、构建标识、步骤结果、截图/日志、缺陷号与复核人。每项结果目前均为待执行。

## 待冻结接口与依赖

- UI 设计冻结托盘菜单名称、图标资源、隐藏时用户反馈及主窗口 show/focus 行为。
- Lead 冻结 AppCoordinator 消息类型、退出回执边界与 Snapshot/revision 发布顺序。
- Windows 窗口适配角色确认 TaskbarCreated 消息接入点及 Explorer/DPI 广播的区分策略。
- 架构评审批准 Slint 版本、`system-tray` feature、IPC 机制和 ACL、单实例按用户还是会话隔离、崩溃/注销期间的保存策略。
- Issue #12 依赖窗口 Shell 契约、TimerCore 持久化回执和 UI 的退出确认原型；本提案已独立给出可供冻结的候选规则。
