# 电源与崩溃恢复协议提案

**状态：待架构审批；本文件为设计提案，所有验证均待执行。**

**来源：** GitHub Issue [#10](https://github.com/troywjz/Pomodoro/issues/10)；需求 [requirements.md](../../../requirements.md)；接口候选 [interface-contract.md](../../../interface-contract.md)。
**边界：** 本模块协调 Windows 电源/锁屏信号、TimerCore 与 Repository；不拥有倒计时算法或数据库 schema。

## 1. 决策摘要

1. 运行段收到 `PBT_APMSUSPEND` 后，协调器在通知处理时限内尽力持久化 `SuspendObserved` 和冻结后的 `recovery` 快照。Windows 文档给应用约两秒处理该通知，时限届满后系统可中断应用；应用不请求或承诺阻止、延迟系统挂起。提交成功后，唤醒时显示恢复界面，用户选择继续或保持暂停。超时、进程中断或写入失败时，重启/唤醒采用最后可靠快照进入 `recovery`，显示保存错误并提供重试；本提案的恢复数据损失目标仍待实测。
2. 锁屏/解锁只作为可观测事件，不改变计时状态。运行段按活动时间继续推进；系统会话锁屏期间进程仍运行时，TimerCore 照常更新。解锁后用时钟差重算倒计时，不按 UI 刷新次数累计。
3. 崩溃重启时先加载最后提交的快照，不使用上次进程的单调时钟原点。若数据库标记 `running`，应用启动进入 `recovery` 并等待用户选择；不扣除进程退出期间时长，不补记完成。
4. `active_ms` 是倒计时唯一时间基准；`boot_ms` 用于跨 suspend/休眠漏通知的检测；UTC 与偏移只用于历史与墙钟变更提示。墙钟校正不改变活动时间，不触发完成。
5. 每 15 秒提交 checkpoint 是当前目标；转换事务须立即提交。写入失败后，协调器停止推进可见的已保存状态，显示明确错误并重试；成功后才发布新 revision。周期 checkpoint 延后、重试未完成期间的实际数据损失上界须由故障注入验证。

## 2. 责任和依赖

| 组件 | 所有权 | 本协议依赖/交付 |
| --- | --- | --- |
| Windows 适配 | 订阅 `WM_POWERBROADCAST`、锁屏/解锁会话通知；捕获消息到达顺序与采样失败 | 向协调器投递原始 `PowerEvent` / `SessionEvent`；消息源与注册/注销方式待 Windows 适配模块冻结 |
| 应用协调器 | 单一串行事件队列；接收系统、UI、时钟、存储回执；给事件编号；每次决策统一采样 Clock；调度 checkpoint、存储重试和恢复界面 | 不直接修改倒计时字段或数据库；保证事件和 `command_id` 幂等 |
| TimerCore（计时模块） | 纯状态转移：running/paused/recovery、剩余时间、完成/中断判定 | 接收已排序事件及 `ClockSample`，返回候选 Snapshot 与 effects；活动时间语义、截止值与数值单位由计时模块冻结 |
| Repository（数据库模块） | SQLite 事务、revision 比较、幂等事件表/唯一约束、checkpoint 保存及加载 | `SaveTransition`、`SaveCheckpoint`、`LoadRecovery` 原子操作；schema、journal/synchronous、幂等键保留期和回执细节待数据库模块冻结 |
| UI | 显示恢复选择、保存错误、重试状态；发出用户选择 | 不读取 DB，不自行推导是否补时或完成 |

**当前接口候选依赖：** `active_ms`、`boot_ms`、`utc_ms`、`start_offset_minutes`；`session_id`、`command_id`、`expected_revision`、`revision`；`remaining_ms`、`accumulated_active_ms`、`target_ms`；`Checkpoint`、`SaveTransition`、`LoadRecovery`。字段名与类型仍属待冻结接口。

## 3. 恢复决策表

| 观察到的情形 | 协调器动作 | 持久化/状态 | 用户可见结果 |
| --- | --- | --- | --- |
| suspend 通知，当前 `running` | 入队一次 `SuspendObserved`；采样并请求 TimerCore 冻结活动时间；在通知处理时限内尽力提交 | 成功时原子更新 ActiveSession 到 `recovery`、递增 revision 并保存 suspend 来源和快照；超时/失败时保留最后提交版本，标记本次 suspend 快照未提交 | 成功时显示“设备休眠前已暂停，唤醒后请选择”；超时/失败路径在唤醒或重启后以最后可靠快照进入 `recovery`，展示保存错误与重试入口 |
| suspend 通知，当前 `paused` / `idle` / `completed` | 记录系统事件诊断，不改业务状态 | 无状态转换事务；可选诊断日志须由数据模块定义 | 不出现恢复确认 |
| 自动唤醒通知 | 与 suspend 事务配对；检查 boot/active 差 | 维持 `recovery`，不自动继续 | 待用户操作；不要求用户已登录 |
| 用户交互后 resume 通知 | 若仍是对应恢复 epoch，仅更新恢复界面可交互标记；重复通知幂等 | 不改计时 revision | 显示恢复选择 |
| 没有先前 suspend，resume 到达 | 作为 `UnpairedResume`；进行时钟差检查 | 运行段采用最后可靠快照转为 `recovery` | 显示“检测到休眠/电源事件缺口，请核对后选择”；不补完成 |
| `boot_ms` 差大于 `active_ms` 差且超阈值 | 记录 `MissedSuspendDetected`；running 转 `recovery`，恢复剩余值采用最近可靠 checkpoint | 保存恢复原因与检测时钟样本；不得将两计数差额扣除 | 显示疑似睡眠或漏通知的中性提示；等待选择 |
| 锁屏/解锁事件 | 只记录事件和时钟样本；不执行暂停 | 状态不变；非必要时不触发独立 DB 写入 | 锁屏继续计时，解锁后显示实时剩余值 |
| UTC 前后差与 boot/active 差显著不同，或墙钟跳变 | 生成 `WallClockAdjusted` 提示事件；继续以活动时间推进 | 更新墙钟诊断信息；不重置 `start_utc`，不改变 remaining | 可显示系统时间变化提示；不据 UTC 推断睡眠时长或完成 |
| 启动时加载到 `running` 快照 | 不复用旧 boot/session 单调基点；`StartupRecoveryDetected` | 从最后提交的 `remaining_ms` 进入 `recovery`，递增 revision | 恢复弹窗提供“继续计时 / 保持暂停 / 放弃本段” |
| 启动时为 `recovery` / `paused` / `completed` / `idle` | 按持久化状态加载 | 不自动变更业务状态；如需记录启动可用独立诊断事件 | 按存储状态显示 |
| 用户选择继续 | 重新采样 Clock；用冻结剩余值建立新的活动计时基点 | 原子写入 `running` 与新 revision，成功回执后开始推进 | 保存成功后关闭询问并继续；失败显示错误，仍停在恢复态 |
| 用户选择保持暂停 | 提交 `paused` 与恢复剩余值 | 成功后新 revision | 显示暂停计时器 |
| 用户选择放弃本段 | 用恢复快照的已累计活动时长追加唯一 `interrupted` 历史 | 同一事务将 ActiveSession 置 `idle` 并追加 history；session_id 唯一 | 成功后回空闲；失败保留恢复态并允许重试 |
| 任一状态转换写入失败 | 冻结业务事件处理在失败转换处；保留最后提交 Snapshot；通过单一退避队列重试。suspend 通知超时后由系统继续挂起流程 | 不发布候选 revision，不重复追加 history；失败状态只用于运行时 UI 展示 | 展示操作、原因及“重试”；不显示已保存成功；唤醒/重启后按最后可靠快照恢复 |

阈值 `missed_suspend`、显著墙钟跳变阈值以及“冻结”期间 Tick 的排序策略需要 TimerCore/Windows/数据库共同评审。初始测试候选是 `boot_delta - active_delta > 2s` 视作睡眠缺口，墙钟异常候选为与 boot 差相差超过 2s；数值仅用于测试设计，不作为已批准产品策略。

## 4. 事件 payload 候选

全部事件有公共 envelope：

```text
RecoveryEvent {
  event_id: UUID,                 // 投递/重放幂等标识
  event_seq: u64,                 // 协调器单调递增序号，仅本进程排序
  observed_at: ClockSample,       // active_ms, boot_ms, utc_ms, utc_offset_minutes
  session_id: UUID | null,
  expected_revision: u64 | null,
  source: windows | startup | user | timer | repository
}
```

候选专有字段：

| 事件 | 额外 payload | 结果/幂等规则 |
| --- | --- | --- |
| `SuspendObserved` | `power_event=PBT_APMSUSPEND`、`power_epoch_id` | 同一 epoch 仅产生一次暂停转换；已有提交结果可重放 |
| `ResumeObserved` | `power_event=PBT_APMRESUMEAUTOMATIC | PBT_APMRESUMESUSPEND`、`power_epoch_id?` | 自动恢复与用户交互恢复视为同一电源周期；后到事件只补充诊断 |
| `LockStateChanged` | `locked: bool`、`session_source` | 同状态重复事件忽略；不改变 TimerCore 状态 |
| `ClockSampleCompared` | `previous_active_ms`、`previous_boot_ms`、`previous_utc_ms`、`active_delta_ms`、`boot_delta_ms`、`utc_delta_ms` | 派生 `missed_suspend` / `wall_clock_adjusted` 标志；样本来源失败时标记 unavailable，不估算时间 |
| `StartupRecoveryDetected` | `stored_state`、`stored_revision`、`stored_checkpoint_id`、`previous_boot_marker?` | 每个启动尝试 ID 唯一；加载事务不得覆盖较新 revision |
| `CheckpointDue` | `checkpoint_interval_ms=15000`、`last_checkpoint_revision` | 只对相同 session 和活动 revision 有效；重复 tick 合并 |
| `RecoveryChoice` | `choice=continue | keep_paused | discard`、`choice_id` | choice_id 幂等；以 `expected_revision` 防止旧弹窗覆盖新状态 |
| `PersistRetry` | `failed_operation_id`、`attempt`、`next_delay_ms`、`last_error_code` | 重试原 operation_id 与相同候选快照；不重新计算历史或 revision |

`ClockSample` 的来源、单位及有效性标志由 Timer 模块定义；本文件不规定 Rust 类型或 Windows API 封装。SQLite 表/列、WAL 与同步级别由 Database 模块决定。

## 5. 事务、排序与提交时序

### 5.1 状态转换

1. Windows/UI/Timer 事件进入协调器单线程队列，分配 `event_seq`。Windows 回调只投递，不执行 SQLite 或等待用户界面。
2. 协调器取得同一采样点的 ClockSample，检查 event_id/command_id 是否已处理，再按 `expected_revision` 调用 TimerCore 生成候选 Snapshot 和 effects。
3. 状态转换候选作为单个 Repository 事务提交：检查数据库当前 revision；若不匹配则返回 `Rejected(current_snapshot)`；若操作幂等键存在返回原结果；否则写 ActiveSession、唯一 SessionHistory（若适用）、处理结果/事件去重记录，并将 revision 加一。
4. 只有 Repository 返回 committed revision 后，协调器才将候选 Snapshot 发布给 UI、重置下一 checkpoint deadline 或执行外部 effect。通知/提示音 effect 自身按 `session_id` 去重。
5. 提交失败时，协调器保留旧已提交 Snapshot，暂存原候选 operation_id，并进入可重试错误态。该候选期间不处理会改变同一活动段 revision 的用户命令；系统电源消息可排队并合并。重试成功后按序处理队列，失败则保持暂停推进。

### 5.2 周期 checkpoint

- `running` 每 15 秒提出一次 checkpoint，保存同一 session 下新的 `remaining_ms`、`accumulated_active_ms`、时钟样本及 checkpoint 序号。
- checkpoint 只允许更新活动快照，不追加历史、不改变阶段 outcome。保存采用 CAS revision，过期 checkpoint 拒绝；协调器从成功回执开始计算下一间隔。
- 完成/暂停/睡眠/退出等关键转换优先进入状态转换事务，并取消或合并同一 session 的待处理 checkpoint。
- checkpoint 期间崩溃：按 Repository 最后提交值恢复。目标进度损失不超过一个 checkpoint 周期；此目标排除存储失败或系统掉电，存储失败期间的上界取决于持续时间，界面需呈现未保存错误。

### 5.3 睡眠时序

```text
WM_POWERBROADCAST(PBT_APMSUSPEND)
  -> Windows Adapter enqueue PowerEvent
  -> Coordinator sample clocks + order event
  -> TimerCore: running -> recovery(frozen remaining)
  -> best-effort Repository transaction within notification handling budget (~2s)
  -> if committed: record recovery snapshot and finish event handling
  -> if timeout/failure: retain last reliable checkpoint; system proceeds with suspend and may interrupt app
  -> PBT_APMRESUMEAUTOMATIC: compare clocks, remain in recovery
  -> user choice -> transaction -> publish committed state
```

Microsoft 将 `PBT_APMSUSPEND` 定义为即将挂起的通知，并给出约两秒处理时限；时限届满后系统可中断仍在执行的应用操作。该通知不提供应用阻止/延迟系统挂起的承诺。本设计只在有限时限内尽力提交恢复快照；超时/失败时系统继续挂起，唤醒/重启以最后可靠快照进入 `recovery`。数据损失目标及具体设备上的提交表现待实测。唤醒通知组合由 Microsoft 文档定义；通知缺失路径仍须通过 AC 与故障注入验证。事务提交耐久性取决于 SQLite 配置和设备，须由 Database 模块给出选型及边界。

### 5.4 错误与重试

| 错误类别 | 协调器处理 | UI 信息 | 重试约束 |
| --- | --- | --- | --- |
| transient I/O、busy、锁冲突 | 保留旧提交快照、排队同一操作 | “保存失败，当前显示最近已保存状态”并提供重试 | 同 operation_id、候选内容、revision；退避间隔待实现设计 |
| revision conflict | 丢弃陈旧候选并加载 Repository 当前快照；重新对用户命令作前置状态判断 | 更新状态；若操作失效说明原因 | 前置状态通过后再生成新的转换候选 |
| disk full / permission / corruption | 保留 DB 原件和旧快照，停止涉及数据修改的继续操作 | 显示错误类别和可操作恢复建议 | 修复介质/权限后重新尝试；损坏库走数据库恢复流程 |
| Clock API unavailable | 将运行态转为保守恢复待确认；无效样本不参与时间差计算 | “系统时间样本不可用，请确认计时状态” | 用户选择后再次采样；无有效活动时间时继续选项保持禁用或采用已批准策略 |
| repeated power events | 同 epoch/event ID 合并 | 不重复弹窗 | 按幂等键返回原事务结果 |

## 6. 墙钟变更与漏通知检测

协调器在启动、系统唤醒、恢复选择及必要的周期 checkpoint 采集活动时钟、开机经过时钟与 UTC。令 `Δa` 为 active 差，`Δb` 为 boot 差，`Δu` 为 UTC 差：

- 若 `Δb - Δa` 超过批准阈值，检测到活动时钟未计入的系统停顿，生成恢复候选，不以差值扣除 remaining。
- 若 `Δu` 与 `Δb` 差值超过批准阈值，标记墙钟改变；倒计时仍只依据 `Δa`。`start_utc` 与原当地日期/偏移保持不变，历史结束时间使用新的 UTC 样本，并存储校时诊断。
- 若时钟样本跨启动、倒退、缺失或不一致，标记样本无效，按最近已提交剩余值进入 `recovery`。UTC 差值仅用于校时诊断。
- 每次完成事务同时带有基于活动时钟计算的截止判断；完成结果由活动时间截止值触发。用户恢复选择后，TimerCore 建立新的同启动活动基点。

Microsoft 将 `QueryUnbiasedInterruptTime` 定义为排除睡眠/休眠且不受用户或 Windows Time 服务系统时间校正影响的活动态时钟候选；实际选用 API、单位转换与失败语义由 Windows/计时接口评审确定。休眠时间包含在 `QueryInterruptTime`/系统 UTC，而 `QueryUnbiasedInterruptTime` 只计工作态，适合交叉检测。

## 7. 故障注入及验收计划（待执行）

所有项目属于未来实现后的计划；当前没有测试执行或通过证据。验收对应 [AC-07–AC-10](../../../acceptance.md#执行环境和证据模板)、AC-25、AC-36、AC-41。

| ID | 注入/步骤 | 预期观察 | 证据 |
| --- | --- | --- | --- |
| RI-01 | running 时投递 suspend；在提交前/提交后分别触发进程终止 | 提交前终止时下次按最近 checkpoint 恢复；提交后状态为 recovery；任何分支均不自动完成 | event trace、DB revision、重启 UI 截图 |
| RI-02 | 同一 suspend/resume 消息重复投递或乱序投递 | 同一 power epoch 仅一次转换；无负时长、重复 history | 输入序列、revision、history 行数 |
| RI-03 | 不投递 suspend，只增加 boot 时钟并保持 active 增量小 | 检测漏通知，转 recovery 并用最后 checkpoint；不扣除睡眠时长 | 三类时钟样本、恢复原因、快照值 |
| RI-04 | 锁屏运行已知时长后解锁；分别在屏幕锁定期间跨过截止点 | active 时间推进；完成只由 TimerCore 一次提交；事件不造成暂停或重复完成 | 锁状态、计时样本、唯一 history |
| RI-05 | 调整 UTC 前后时钟并运行阶段；测试向前、向后跳变 | remaining 连续，不因墙钟跳变负数/完成；保留原 start UTC 与时区归属 | UTC/active/boot 采样、历史字段 |
| RI-06 | 将 checkpoint 写入暂停/失败若干次，再强杀应用 | 只能恢复最后成功提交值；显示未保存错误；不会报告失败候选已保存 | I/O 故障日志、DB snapshot、UI 状态 |
| RI-07 | Sleep/Resume、恢复选择、checkpoint 转换事务分别在 BEGIN、写行、COMMIT 前后注入失败 | 原子性成立：要么旧 revision+旧快照，要么新 revision+完整关联 history；不出现半状态 | SQLite 只读检查、事务日志/测试报告 |
| RI-08 | Recover continue/keep_paused/discard 使用同 choice_id 重投；再用旧 revision 重投 | 重投回原结果；discard 仅一条 interrupted；旧 revision 返回 conflict 并刷新 | command/result 记录、唯一键检查 |
| RI-09 | Repository 持续报 busy，释放后重试原 operation_id | 协调器停止推进并持续提示；解除故障后一次提交并恢复后续队列 | 尝试序号、延时、提交 revision |
| RI-10 | 注入磁盘满、只读目录、损坏库及 Clock API 失败 | 保留最后数据；分类错误；时钟无效时不估算、不补完成；按数据模块恢复流程处理损坏库 | 错误码、源库/备份检查、界面提示 |
| RI-11 | 强杀时分别处于 running、recovery、paused、completed；重启 | running/recovery 进入或保持恢复等待；其它状态按已提交状态展示；不复用旧单调基点 | 启动快照、状态序列、选择前倒计时观察 |
| RI-12 | 在 Hyper-V 宿主运行 VMConnect、锁屏与休眠场景并重复上述电源测试 | 宿主通知、clock delta、恢复选择与普通桌面承诺一致；Guest 内行为不纳入承诺 | 宿主系统版本、VM 状态、事件 trace |

执行环境、Windows build、设备电源模式、数据库配置、构建标识和每个注入点须记录。测试代码、SQLite 故障装置、软件安装及实现型试验均属于批准开发阶段工作，当前阶段不执行。

## 8. 待冻结与待评审

1. Timer 模块冻结 `ClockSample` 字段、单调时间来源、失效表示、截止值、活动时长/剩余时长计算及暂停事件参数。
2. Database 模块冻结 SQLite schema、revision CAS、幂等记录持久化、ActiveSession 与 SessionHistory 单事务边界、checkpoint 原子更新及失败分类。
3. Windows 适配模块明确 `PBT_APMSUSPEND` 处理回调是否可同步等候写入、锁屏事件接入方式及消息丢失边界。
4. Lead/用户审批漏通知和墙钟异常阈值、写失败时继续挂起策略，以及 checkpoint 目标所对应的数据损失承诺。
5. 建立 UI 文案和恢复动作可访问性；“继续计时”是否从冻结 remaining 开始、确认前保持恢复态，与需求草案一致。

## 官方依据

- Microsoft：[QueryUnbiasedInterruptTime](https://learn.microsoft.com/en-us/windows/win32/api/realtimeapiset/nf-realtimeapiset-queryunbiasedinterrupttime) 说明该计数排除睡眠/休眠且不受系统时间校正影响。
- Microsoft：[PBT_APMSUSPEND](https://learn.microsoft.com/en-us/windows/win32/power/pbt-apmsuspend) 说明应用约有两秒处理通知，超时后系统可中断应用操作。
- Microsoft：[WM_POWERBROADCAST](https://learn.microsoft.com/en-us/windows/win32/power/wm-powerbroadcast) 与[电源广播消息](https://learn.microsoft.com/en-us/windows/win32/power/wm-powerbroadcast-messages)说明 suspend、automatic resume 与交互 resume 消息。
- Microsoft：[Windows Time](https://learn.microsoft.com/en-us/windows/win32/sysinfo/windows-time)区分包含睡眠时间的 elapsed clock 与仅计算 working state 的 unbiased interrupt time。
- SQLite：[Atomic Commit](https://www.sqlite.org/atomiccommit.html)与[WAL](https://www.sqlite.org/wal.html)说明事务原子提交与耐久性配置边界；具体 journal/synchronous 选择待数据库模块提出并审批。
