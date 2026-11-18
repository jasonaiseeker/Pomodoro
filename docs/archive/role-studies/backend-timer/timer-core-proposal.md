# 计时内核契约提案

- 状态：草案，待产品/架构评审；未批准实现。
- 来源：GitHub Issue [#9](https://github.com/troywjz/Pomodoro/issues/9)、[需求草案](../../../requirements.md)、[架构提案](../../../architecture.md)、[接口契约](../../../interface-contract.md)、[验收矩阵](../../../acceptance.md)及[UI 第二轮要求](../../../ui-revision-v2.md)。
- 范围：纯 Rust 计时状态机接口，不含 Windows API、Slint、数据库实现或依赖选择。
- 阶段门槛：新版 Notion 需求/Figma 原型及关键架构获用户确认前，本文件保持设计提案。

## 1. 核心不变量

1. 阶段枚举为 `focus | short_break | long_break`；状态枚举为 `idle | running | paused | completed | recovery`。阶段与状态分开表示。
2. 一个活动段有唯一 `session_id`、固定的 `target_ms` 和单调活动时间上的 `deadline_active_ms`。启动后修改配置不改变该段目标。
3. `running` 的剩余时间由 `deadline_active_ms - now_active_ms` 推导并饱和到 `[0, target_ms]`；不按 UI 刷新次数递减。暂停时冻结剩余毫秒，恢复时以当前活动时钟建立新截止值。
4. 活动时钟来自注入的 `Clock`。其采样为单调、非负毫秒值；墙钟调整不改变它。运行态遇到休眠/漏掉睡眠通知时，由协调器按既有恢复契约转入 `recovery` 并采用最近可靠快照，等待用户选择。锁屏期间计时继续。活动时钟与系统启动经过时间的具体 Windows 采样策略由平台适配提案确定。
5. `completed` 只在运行段到期并且完成事务成功后发布。完成事务以 `session_id` 唯一化；重复 Tick/完成投递不重复写历史、增加轮次或触发提示。
6. `recovery` 在用户确认前不推进、不自动完成。`continue` 按最近快照剩余时间重新建立活动截止值；`keep_paused` 冻结该剩余值；`discard` 写一条 `interrupted` 历史后进入 idle。
7. 状态转移先形成候选快照并提交持久化效果；成功回执后才对外发布新 revision。失败保留最后已提交快照，冻结推进并报告保存错误，按接口契约提供重试。
8. `remaining_ms <= target_ms`。圆圈使用 `remaining_ms / target_ms`，初始为 1，到期为 0；暂停和恢复状态按冻结/继续后的精确快照呈现。`target_ms` 必须大于 0。

## 2. 时钟和倒计时计算

纯内核依赖调用方提供的 `ClockSample { active_ms, boot_ms, utc_ms }`。`active_ms` 用于倒计时，`boot_ms` 与 `active_ms` 的变化关系供协调器识别可能漏掉的睡眠事件，`utc_ms` 用于历史时间戳及提示。内核不得自行读系统时间，也不得把 UTC 墙钟差应用到剩余时间。

运行段的候选剩余时间：

```text
remaining_ms = clamp(deadline_active_ms - sample.active_ms, 0, target_ms)
progress_ratio = remaining_ms / target_ms
```

比率是剩余比例，数值范围 `[0.0, 1.0]`；整型计时值是数据源。UI 可低频读取 Snapshot 及重新计算比率，刷新频率不影响内核推进。`paused`、`recovery`、`completed` 返回冻结/完成快照中的剩余值。`idle` 无活动段时返回 `remaining_ms = 0`、`target_ms = null`、`progress_ratio = null`；界面可按空闲视觉规范隐藏圆圈或展示全空状态，设计待冻结。

活动时钟样本必须不早于内核上一接受样本。早到样本返回 `clock_regressed`，当前 Snapshot 不变；相同样本允许重放，结果相同。溢出运算使用饱和或经检查算术，禁止绕回产生大剩余值。阶段开始时将实际采纳的时长保存为 `target_ms`，转换单位和上限验证在接受命令前完成。

## 3. 命令协议与状态转移

每条命令带 `command_id`、`expected_revision`；命令由单一协调器串行送入内核。读取当前 Snapshot 后，客户端应以其 revision 发命令。所有成功状态变更增加 revision 一次。无状态变化的重复 Tick 可返回当前 revision。

| 命令 | 合法前置状态 | 成功转移与效果 | 主要拒绝条件 |
| --- | --- | --- | --- |
| `Start(phase, duration_ms)` | `idle`；或 `completed` 且用户明确选择开始下一段/重开 | 创建新 `session_id`；状态 `running`；设置目标与活动截止值；保留旧完成记录 | 阶段非法、时长非正数/超配置范围、其他状态、revision 冲突 |
| `Pause(user)` | `running` | 采样并计算剩余值，进入 `paused`，持久化快照 | 非 running、时钟倒退、revision 冲突 |
| `Pause(suspend)` | `running` | 保存剩余值并进入 `recovery`，等待唤醒后用户选择 | 非 running、时钟倒退、revision 冲突 |
| `Resume` | `paused` | 用冻结的剩余值建立活动截止值，进入 `running` | 非 paused、剩余值为零但未提交完成（需由协调器先处理到期）、revision 冲突 |
| `Tick(sample)` | `running` | 未到期：返回更新后的只读 Snapshot；到期：提交唯一完成历史并进入 `completed` | 时钟倒退、revision 冲突；其他状态不推进并返回当前快照 |
| `ResumeDetected(power/missed_suspend)` | `running` 或 `recovery` | 取最近可靠快照进入 `recovery`；不扣睡眠/离线时长 | 非活动会话、无可靠快照、revision 冲突 |
| `Recover(continue)` | `recovery` | 以冻结剩余值建新截止值，进入 `running` | 非 recovery、剩余值无效、revision 冲突 |
| `Recover(keep_paused)` | `recovery` | 保持剩余值，进入 `paused` | 非 recovery、revision 冲突 |
| `Recover(discard)` | `recovery` | 追加唯一 `interrupted` 历史，进入同阶段 `idle` | 非 recovery、持久化失败、revision 冲突 |
| `Reset` | `running | paused | recovery` | 计算/采用可靠快照实际活动时长，追加 `interrupted`，同阶段进入 `idle` | 其他状态、持久化失败、revision 冲突 |
| `SkipBreak` | `short_break | long_break` 且 `running | paused` | 追加 `skipped`，进入 `focus/idle`，不自动开始 | focus 段、其他状态、持久化失败、revision 冲突 |
| `Complete(session_id)` 内部效果 | 同 ID 活动段且到期 | 完成历史唯一写入、轮次更新一次、状态 `completed`；完成通知按 ID 去重 | ID 不匹配、未到期；同 ID 已完成时回放既有结果 |

`completed` 下一步由用户明确发起 `Start`；不得自动启动。`Reset`、`SkipBreak` 与完成命令的历史记录和轮次语义沿用 requirements/interface-contract。退出确认和 Shell 行为由协调器负责，内核只提供暂停、丢弃和快照转换。

## 4. 幂等、版本和错误

- `command_id` 在协调器持久化的幂等表/日志范围内唯一。同 ID 同载荷重投返回原始结果；同 ID 不同载荷返回 `command_id_reused`，不执行新操作。
- 每次状态变更的持久化请求包含 `expected_revision`、候选完整 Snapshot、关联 session ID 和待追加 History。提交成功后才发布候选值。重复效果按 `(session_id, outcome)` 去重；完成通知按 `session_id` 去重。
- Tick 是可重放观察命令；重复相同或更旧的有效样本不得产生额外状态副作用。并发命令经协调器排序后逐个校验 revision。
- 错误码至少区分 `invalid_phase`、`invalid_duration`、`invalid_state`、`revision_conflict`、`command_id_reused`、`clock_regressed`、`snapshot_unavailable`、`persistence_failed`、`completion_not_due`。拒绝结果携带当前已提交 Snapshot/revision；持久化失败同时保留可重试效果标识。
- 转换保存失败时，不得向 UI 发布新状态、完成提示或退出成功；按已有恢复契约停止推进并提供重试。此处描述目标行为，尚无实现或故障注入证据。

## 5. Snapshot 字段候选

公共接口字段为草案；字段名、序列化格式和 revision 所有权待 Lead 与数据库/恢复角色冻结。

| 字段 | 类型候选 | 语义 |
| --- | --- | --- |
| `revision` | `u64` | 已提交状态版本；持久化成功后递增 |
| `session_id` | 可空 UUID/string | 当前活动段标识；idle 时为空 |
| `phase` | `focus | short_break | long_break` | 当前段类型；idle 初始值可空，丢弃后可保留上一段展示信息，需冻结 |
| `state` | 状态枚举 | 五态之一 |
| `target_ms` | 可空 `u64` | 本段启动时锁定的目标时长 |
| `remaining_ms` | `u64` | 当前计算或冻结的精确剩余毫秒数 |
| `progress_ratio` | 可空有限 `f64` | `remaining_ms / target_ms`，范围 `[0,1]`，仅派生展示字段；idle 为 null |
| `accumulated_active_ms` | `u64` | 最近可靠快照记录的已活动毫秒；暂停/恢复时精确更新 |
| `deadline_active_ms` | 可空 `u64` | 仅内存中的运行截止值；重启后不可跨进程复用其单调时钟原点 |
| `last_sample_active_ms` | 可空 `u64` | 检测倒退样本及计算当前进度 |
| `last_checkpoint_utc_ms` | 可空 `i64` | 最近可靠持久化检查点的 UTC 时间 |
| `start_utc_ms` | 可空 `i64` | 记录开始时间，不参与倒计时 |
| `start_local_date` | 可空日期 | 统计归属，按开始时本地日期保存 |
| `start_offset_minutes` | 可空 `i32` | 开始时 UTC 偏移，配合统计/导出 |
| `pending_effect` | 可空效果标识 | 持久化未确认时用于重试/恢复去重；存储归属待数据库契约确定 |

Snapshot 对 UI 只读。`progress_ratio` 应从整数值推导，避免独立存储产生与准确读数不一致。内核逻辑以整数毫秒为准；显示格式由 UI 决定。启动/恢复返回已提交 Snapshot 后方可驱动圆圈及数字。

## 6. 假时钟验收计划（待执行）

这些是内核验收计划，不是测试通过记录。开发阶段批准后，用可控 `Clock` 和持久化假仓储执行；当前未创建 Cargo 项目、未运行代码或测试。

| ID | 初始条件与操作 | 预期结果 |
| --- | --- | --- |
| T-01 起始边界 | `idle` 启动 1 ms 阶段，读取同一时钟样本 | running、目标 1、剩余 1、比率 1.0 |
| T-02 逐毫秒 | 目标 10,000 ms；样本前进 1、9999 ms | 剩余分别 9999、1；比率由整数比推导，未依赖读取次数 |
| T-03 精确到期 | 目标 10,000 ms；样本到截止值，重复 Tick | 首次持久化一次并进 completed、剩余 0/比率 0；重复返回原完成结果，无重复历史/提示/轮次 |
| T-04 超过截止 | 样本大于截止值并多次 Tick | 剩余饱和为 0，不发生整数绕回，完成副作用一次 |
| T-05 暂停冻结 | 运行 4,000/10,000 ms 后 Pause；时钟前进 60,000 ms | paused 仍剩 6,000 ms；暂停时间不计活动时长 |
| T-06 恢复截止 | T-05 后 Resume，时钟前进 2,500 ms | remaining=3,500；新截止点基于恢复时样本 |
| T-07 墙钟调整 | running 时 UTC 前跳/回拨，active_ms 连续 | remaining 只随 active_ms 变化；UTC 只影响记录/提示元数据 |
| T-08 时钟倒退 | 已接受 active_ms=5000 后输入 4999 | 返回 `clock_regressed`，状态和最近快照不变 |
| T-09 样本重放 | 重复同一有效 Tick 样本 | Snapshot 和效果不重复、不回退 |
| T-10 恢复选择 | recovery 剩余 7000 ms，分别执行 continue/keep_paused/discard | 分别 running、paused、idle；确认前时间冻结；discard 仅写一条 interrupted |
| T-11 阶段和时长错误 | 非法 phase、0、配置上/下界外时长 | 返回区分明确的错误；不创建 session，不变更 revision |
| T-12 状态前置 | 对 idle Pause、running Resume、completed SkipBreak 等非法组合发命令 | 返回 `invalid_state` 和当前 Snapshot，保持状态不变 |
| T-13 revision 并发 | 两条命令携带同 expected_revision，第一条成功后提交第二条 | 第二条返回 `revision_conflict` 与新 Snapshot，不覆盖首条结果 |
| T-14 command 去重 | 重投相同 command_id/载荷；再以同 ID 改参数重投 | 首次返回原始结果；改参数返回 `command_id_reused`，无第二次副作用 |
| T-15 提交失败 | Pause、完成、discard 分别令假仓储失败后重试 | 发布状态冻结在旧已提交 Snapshot；无虚假完成通知；重试按同效果去重并成功一次 |
| T-16 数值边界 | 极大有效时长、截止差值边界、算术上溢输入 | 验证输入上限；算术受检/饱和，不产生负剩余或绕回值 |

## 7. 依赖、决策与验证记录

- 依赖已有 `TimerSnapshot`、`command_id`、`expected_revision` 和成功提交后发布状态的通用协议；本稿候选与公共接口一致，需 Lead 冻结字段和错误码。
- 依赖后端恢复角色确认唤醒/漏通知进入 recovery 的事件来源，以及最近可靠快照与 15 秒周期 checkpoint 的交接字段。
- 依赖数据库角色确认 revision、活动段、History 唯一键、命令去重记录及事务边界。
- 依赖产品/设计确定 `idle` 圆圈呈现和 completed 的剩余值呈现；本稿定义进度数学语义。
- 本次执行证据：已检查 `git status -sb`，确认初始工作区干净；从 detached HEAD 切换至已存在的 `team/backend-timer` 分支。仅写入此目录中的本文件。未运行测试、未执行实现型试验、未访问远程写入接口。
- 尚待评审：idle 的 phase 取值、completed 是否保留 target、recovery 快照字段、命令去重保留期限、错误码对 UI 的映射、UTC/活动钟采样与 Windows 休眠识别策略。
