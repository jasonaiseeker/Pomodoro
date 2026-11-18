# SQLite 与 Repository 契约提案

状态：草案，待用户架构审批；未创建数据库、未执行 SQL、未进行实现试验。
来源：GitHub Issue [#15](https://github.com/troywjz/Pomodoro/issues/15)，并对齐 [模块接口契约](../../../interface-contract.md)、[需求草案](../../../requirements.md)、[验收矩阵](../../../acceptance.md)及 [UI 第二轮输入](../../../ui-revision-v2.md)。

## 1. 设计边界

- SQLite 单库作为设置、唯一活动会话、历史与幂等请求账本的候选存储。所有业务写入经过一个串行 Repository 写入队列；查询可使用独立只读连接。
- 状态转换事务同时提交活动快照变化、历史追加、会话轮次更新与命令去重结果。只有事务提交成功后，协调器才发布新 Snapshot 或完成提示。
- 单调时钟起点不落盘。数据库记录累计活动毫秒、剩余毫秒和 UTC 时间；启动恢复采用最近已提交快照，不扣进程离线时间。
- 本文表名、字段和约束是 schema v1 候选。字段、journal/synchronous 策略、数据目录、保留规则与升级兼容政策均待架构审批。

## 2. Schema v1 表提案

所有时间戳统一为 UTC Unix 毫秒整数，时长统一为非负整数毫秒，时区偏移为开始时的分钟数。随机 UUID 以规范小写文本保存。枚举使用固定小写 ASCII 值。数据库约束负责拒绝非法值，应用层仍需提供可读校验错误。

| 表 | 主键 / 唯一性 | 字段与约束提案 | 用途 |
|---|---|---|---|
| `schema_meta` | 单行 `id=1` | `schema_version INTEGER NOT NULL`、`app_data_version INTEGER NOT NULL`、`state_revision INTEGER NOT NULL`、`updated_at_utc_ms INTEGER NOT NULL`；`id CHECK(id=1)` | 应用管理的 schema 标识、全局状态修订与可选数据版本；另用 SQLite `user_version` 镜像迁移版本，读取不一致进入诊断 |
| `settings` | 单行 `id=1` | `focus_ms` 60000..10800000；`short_break_ms` 60000..3600000；`long_break_ms` 60000..7200000；`long_break_interval` 1..12；`topmost`、`sound_enabled` 为 0/1；`theme_mode` 候选 `system`；`window_width_dip`、`window_height_dip` 为正数；位置 x/y 可负；`monitor_hint` 可空；`updated_at_utc_ms`、`revision` 非负 | MVP 配置。时间范围来自需求草案。主题与窗口字段待产品/设计冻结；几何值保存 DIP，不保存物理像素 |
| `active_session` | 单行 `id=1`，约束 `id=1` | `session_id TEXT NOT NULL UNIQUE`；`phase`=`focus/short_break/long_break`；`state`=`running/paused/recovery`；`target_ms > 0`；`accumulated_active_ms >= 0`；`remaining_ms BETWEEN 0 AND target_ms`；`revision >= 0`；`start_utc_ms`、`checkpoint_utc_ms`；`start_local_date` ISO `YYYY-MM-DD`；`start_offset_minutes BETWEEN -840 AND 840`；`pause_reason`、`recovery_reason` 可空；`updated_at_utc_ms` | 只保留当前未结算段。空表代表无活动段；`idle`、`completed` 由空活动行及最近历史/协调器 Snapshot 表达，不复制为活动行 |
| `session_history` | `session_id TEXT PRIMARY KEY` | `phase`；`target_ms > 0`；`actual_active_ms >= 0`；`start_utc_ms`、`end_utc_ms >= start_utc_ms`；`start_local_date`；`start_offset_minutes`；`outcome`=`completed/interrupted/skipped`；`reason` 固定枚举候选；`completed_focus_round INTEGER` 为 0/1；`recorded_at_utc_ms` | 每个 session 至多结算一次。只允许终态记录。跳过休息的 session phase 为休息阶段且 outcome 为 skipped；专注完成轮次只由 completed focus 贡献 |
| `session_day_index` | `session_id PRIMARY KEY`，FK 到 history | `local_date` ISO 日期；`focus_round_delta` 非负；可选 `focus_active_ms` | 可选派生索引，不作为事实来源；首版统计可直接从历史查询。若保留，必须与 history 同一事务维护，并可由 history 重建 |
| `command_receipt` | `command_id TEXT PRIMARY KEY` | `session_id` 可空；`command_kind`；`expected_revision`；`result_code`；`result_revision`；`result_payload_json`；`created_at_utc_ms` | 有界幂等账本，重放同一 command_id 返回原结果；保留期/上限待定，删除前必须确认最老可重试期限 |

约束补充：`active_session` 最多一行是“至多一个活动 session”的数据库级保证；`session_history.session_id` 主键与 `active_session.session_id UNIQUE` 防止重复结算。FK 建议启用并逐连接验证。应用启动时检查活动行状态、范围和关联完整性；违反约束进入只读诊断/恢复流程，不静默修补。

`settings.revision` 是配置修订号；`active_session.revision` 是活动快照修订号；不存在活动行时使用持久化全局 `state_revision` 候选值（放入 `schema_meta`）保证 idle/completed 命令也能做 compare-and-swap。各自职责避免把设置修改与计时状态更新混为同一版本。

## 3. Revision、命令幂等与唯一结算

每个改变计时状态的命令带 `command_id`、`expected_revision` 和完整候选 Snapshot。写事务开始后按以下顺序工作：

1. 若 `command_id` 已存在，返回存储的原始结果；同一 ID 若请求类型或内容摘要不同，返回 `IdempotencyKeyReuse` 并保留数据。
2. 比对当前 `state_revision`（活动行存在时与其 revision 一致）。不相等时不写入，返回 `RevisionConflict(current_snapshot)`。
3. 校验合法状态迁移、session ID、计时范围与终态。
4. 条件更新活动行或全局 revision，并在同一事务追加唯一 history（若结束）和 command receipt。
5. 写入新的 `state_revision`，提交。只有提交成功后才向调用方返回 `Accepted`。

同一命令重试返回同一结果；不同 command_id 对相同 session 的重复完成由 history 主键拦截，并读取既有终态形成稳定的 `AlreadyFinalized` 结果。revision 冲突不自动重试业务命令，协调器加载当前状态后由调用方决定是否重新发起。

周期 checkpoint 对现有活动行执行 revision 条件更新，只更新累计活动时间、剩余时间、checkpoint 时间和 revision；它不写 history。转换命令与 checkpoint 进入同一串行写队列，旧 checkpoint 的较小 revision 被拒绝。checkpoint 可合并排队的中间样本；关键状态转换不得合并或丢弃。

`command_receipt.result_payload_json` 仅承载可重建的协议结果，不含隐私机器信息。receipt 与业务变更同事务，避免出现业务已提交而幂等回执缺失的窗口。时间戳、参数正负和最大值由业务层先校验，数据库层再以 CHECK 防止越界。

## 4. Repository 方法与语义

| 方法 | 输入 | 原子性与输出 |
|---|---|---|
| `open_and_inspect` | 应用支持的最高 schema、迁移策略 | 只读检查 schema、元数据、活动行与完整性状态；返回 `Ready`、`NeedsMigration`、`NewerSchemaReadOnly` 或 `CorruptReadOnly` |
| `load_snapshot` | 无 | 返回 Settings、全局 revision、零/一活动段与最近已提交状态；不以墙钟推算推进时间 |
| `save_transition` | command ID、expected revision、完整新状态、可选终态记录 | 单事务 CAS + 快照写入 + 可选 history + receipt；返回 Accepted/Duplicate/Conflict/Rejected |
| `checkpoint` | session ID、expected revision、累计毫秒、剩余毫秒、UTC 样本 | 单条 CAS 更新；不改变 phase/outcome；冲突返回当前 revision，不覆盖新状态 |
| `save_settings` | expected settings revision、已校验设置 | 单事务 CAS 更新；运行段 target 保持原值，新设置只用于下一段 |
| `query_today` | 开始时本地日期 `YYYY-MM-DD` | 按 `start_local_date` 查询 `completed_focus_rounds`、`actual_focus_ms` 候选值（completed + interrupted focus）及 `completed_focus_ms`（仅 completed focus）；边界包含完整本地日期，不依赖当前时区重算 |
| `query_history` | 日期范围、分页游标、过滤项 | 稳定按 `start_utc_ms DESC, session_id DESC` 分页；返回原记录时区与状态 |
| `export_snapshot` | 导出范围、格式 | 在一致性读事务中取 settings/history；JSON 包 schema 版本与导出时间；CSV 固定列且保留 UTC、原始本地日期和偏移 |
| `create_backup` | 目标临时路径、用途标签 | 在线一致性备份到临时文件，校验可读与完整性后原子改名；失败清理临时产物、源库不变 |
| `clear_history` | expected state revision、确认令牌/操作 ID | 先成功完成备份，再在事务内删历史与相关派生索引；保留设置及活动快照；失败时历史保持完整 |
| `restore_backup` | 已选择备份、确认 | 先在隔离临时文件验证版本与完整性，保留原数据库副本，再交换/恢复；失败回到原文件并返回错误 |
| `clear_all` | 无活动 session 的状态确认、确认令牌 | 结束活动会话后才可执行；移除库与应用管理备份，清单和错误回报由数据管理流程负责；不创建新备份 |

Repository 错误类型至少区分：`Busy`、`Io`、`DiskFull`、`ConstraintViolation`、`RevisionConflict`、`IdempotencyKeyReuse`、`UnsupportedNewerSchema`、`CorruptDatabase`、`BackupFailed`、`MigrationFailed`、`Serialization`。UI 不接触 SQLite 错误码；错误对象含操作、可恢复建议和是否保留原状态。

## 5. 事务、同步与备份候选

- 每个业务状态改变使用显式事务；完整计时转换、唯一历史、receipt 和 revision 在一个事务提交。SQLite 同一时刻只允许一个写事务，因此单写入队列让调用顺序与协调器事件顺序一致。[SQLite 事务说明](https://www.sqlite.org/lang_transaction.html)
- 事务提交失败时回滚，调用方继续持有最后已提交 Snapshot，不触发完成音效、不退出、不报告成功。Busy 使用有限、可取消的重试预算；超出预算返回可见错误并暂停状态推进。
- journal 模式及 `synchronous` 级别仍待架构决策。候选首版 rollback journal；是否选择 WAL 必须同时评估 checkpoint/多文件备份和关闭流程，发布前根据掉电目标做决定。
- 一致性备份首选 SQLite Online Backup API 等效封装，目标先写临时文件，再进行可读性与完整性检查后改名。SQLite 官方说明该 API 可取得备份开始时的数据库快照，并适用于运行中数据库。[SQLite Backup API](https://www.sqlite.org/backup.html)
- 不以复制单个数据库文件作为运行中备份方案。SQLite 官方指出该做法会阻塞写入，且复制期间系统故障可使副本损坏。[SQLite Backup API](https://www.sqlite.org/backup.html)
- schema 迁移执行顺序：打开只读检查 → 判断是否新版本/损坏 → 生成并验证升级前备份 → 单一升级事务 → 更新 schema 元数据与 `user_version` → 提交 → 重新检查。任一步失败保留原库并进入只读错误页。
- 迁移期间事务失败回滚全部 DDL 与数据变更；升级前备份保留到用户清理。备份失败时不开始迁移。未来版本 schema 一律拒绝业务写入，保留原文件并提示使用兼容版本或导出/恢复支持范围内的备份。
- 清空历史先备份再事务删除；备份或事务失败均不删除数据。永久清除遵循已批准的数据管理流程，清单/设置/活动行/应用管理备份范围在确认界面列明。

## 6. 损坏检测与恢复

启动诊断检查数据库是否可打开、schema 是否受支持、关键表与单行约束、外键和 SQLite 完整性检查结果。完整性检查具体采用 `integrity_check` 或较轻量启动检查加按需完整检查，待性能与 SQLite 版本评审定稿。

损坏路径采取只读保护：保留原数据库原位并创建带时间戳的诊断副本（是否包含用户数据、保存位置及清理方式待隐私评审）；禁止覆盖原件或静默新建空库。恢复页列出可用备份的时间、schema 版本、完整性校验结果，用户选择已验证备份或明确确认创建空库。无有效备份时保留原件并允许导出可读取部分（如技术上可行，需单独验收）。

更新版本 schema 的处置与损坏分开：数据库保持只读且原文件不变，提供应用版本要求与兼容版本恢复路径，不把未来 schema 误判成损坏库，也不允许低版本程序降级写入。

## 7. 统计与导出查询契约

- “今日”按段开始时保存的 `start_local_date` 归属；跨午夜结束的 focus 整段计入开始日期。`completed_focus_rounds` 只统计 outcome=completed 的 focus；interrupted focus 不增加轮次。
- `actual_focus_ms` 候选口径为所有已提交 focus 的 `actual_active_ms` 总和，包含 completed 与 interrupted；`completed_focus_ms` 单独表示 outcome=completed 的 focus 总和。两个时长指标名称与口径保持分开，均保留精确毫秒。是否将其中哪项作为产品主显示指标、分钟如何取整，待产品裁决。
- short/long break 与 skipped 均不增加完成轮次。历史记录保存每种终态及原因，统计可按 outcome/phase 重建。
- 会话时间同时记录 `start_utc_ms`、`end_utc_ms`、`start_local_date` 与 `start_offset_minutes`。系统时区变更后历史归属保持原值；导出不可用当前时区覆盖原值。
- `query_today` 入参由计时/统计协调器提供当前本地日期及当前偏移采样。SQLite 不调用本机时区环境；日期确定性由应用层提供。
- JSON 导出契约为版本化 envelope：`schema_version`、`exported_at_utc`、`settings`、`sessions`、字段单位说明；增量导出不属于 MVP 提案。CSV 使用固定列顺序、UTF-8、CRLF、RFC 4180 引号规则，字段包括 session ID、phase、outcome、target/actual ms、开始结束 UTC、原本地日期、原始偏移、reason。确切列标题与日期呈现待产品/数据管理冻结。
- 导出读取使用一致性快照；写出失败报告失败并保留用户原有文件（写临时文件后替换），不显示成功通知。

## 8. 与其他角色的接口依赖

| 依赖方 | 数据库契约候选 | 待冻结项 |
|---|---|---|
| 计时内核 | session ID + global state revision；转换命令幂等；单调时间只在进程内；终态 history 同事务 | Snapshot 完整字段、事件的 wall/active/boot clock 样本、checkpoint 周期与同步优先级 |
| 恢复 | 读最近提交的 running 快照进入 recovery；不扣离线时间；继续/暂停/放弃均为带 revision 的转换 | 唤醒与重启的恢复 reason 编码、快照可信度/损失提示 |
| 统计 | start_local_date 固定归属；`completed_focus_rounds` 仅含 completed；`actual_focus_ms` 候选含 completed + interrupted；`completed_focus_ms` 单独仅含 completed | 产品裁决实际专注主指标；分钟舍入显示、日历范围/时区变更行为 |
| 数据管理 | 清空历史前备份，保留 settings/active；全清要求无活动段并清应用备份；导出不改源数据 | 备份命名/列表/保留、恢复确认、永久清除 UI 结果及 CSV 标题 |
| 产品/设计 | Theme system、window DIP 几何、monitor hint 为 settings 候选字段 | 新版 Figma 完整原型、几何字段与用户隐私/显示器标识边界 |

## 9. 故障与预期结果矩阵

| 故障/输入 | 预期 Repository 行为 | 调用方可见结果 |
|---|---|---|
| 提交前磁盘满、权限拒绝或 I/O 错误 | 回滚转换、history 与 receipt；保持旧 revision | 保存失败，暂停状态推进；保留最后已保存快照并可重试 |
| 提交结果不确定后同 command 重投 | 查 receipt；若已提交则返回原结果，否则安全执行一次 | 不重复历史、轮次或提示音 |
| 同 session 重复完成、不同 command ID | history 主键阻止第二条终态；返回既有终态摘要 | 一次完成记录及一次轮次增量 |
| 旧 checkpoint 晚到 | revision CAS 拒绝更新 | 刷新当前快照；旧值不覆盖新状态 |
| 旧 expected revision 命令 | 不做业务写入 | 返回冲突及当前 Snapshot |
| 相同 command ID 携带不同请求 | 不执行第二个请求 | 返回幂等键复用错误 |
| 两个启动/写连接同时争取活动段 | 单行约束/CAS 仅允许一个活动 session | 第二请求收到活动冲突并加载当前会话 |
| 忙锁超过重试预算 | 回滚并返回 Busy | 显示稍后重试；不丢命令结果或推进计时 |
| 迁移中断/约束失败 | 单一事务回滚；升级前备份仍在 | 进入只读迁移错误页 |
| 备份目标不可写或校验失败 | 停止迁移/清空历史；删除不完整临时备份 | 原数据库与历史保留，说明备份失败 |
| 打开时发现更高 schema_version | 保留文件，拒绝所有业务写入 | 显示版本兼容说明和支持路径 |
| 损坏库且备份无效 | 保留原件，不覆盖、不自动初始化 | 只读恢复页，列出失败备份和人工选择 |
| 导出目标不可写 | 源库读事务结束；目标临时文件清理 | 显示导出失败，已有目标文件不变 |
| 清空历史事务失败 | 回滚历史删除；备份保留 | 原历史可查询，结果明确为失败 |

## 10. 验收映射与执行状态

| 现有验收 | 数据库证据点 | 状态 |
|---|---|---|
| AC-21/22 | JSON/CSV 字段、单位、时区与读取一致 | 待实现阶段执行 |
| AC-23/24/25/33 | 备份先行、取消无副作用、清空范围与失败原子性 | 待实现阶段执行 |
| AC-34/36 | 唯一 session 终态、命令重放、写失败和恢复重试 | 待实现阶段执行 |
| AC-37 | 跨午夜和时区变更仍按开始时日期归属 | 待实现阶段执行 |
| AC-39/40 | 迁移备份/事务回滚、未来版本拒写、损坏库恢复 | 待实现阶段执行 |
| AC-41/42 | 最近已提交快照恢复、旧 revision 拒绝 | 待实现阶段执行 |

本轮仅完成文档设计；没有数据库、SQL、Rust/Cargo 工程、依赖安装、程序执行或故障注入。所有验收状态为待执行。

## 11. 评审问题

1. 是否批准 SQLite 单库、单写队列、唯一活动行、session 主键终态与 command receipt 方案？
2. journal/synchronous 策略与掉电耐久性目标采用何种组合？
3. 迁移前备份是否长期保留至用户清理；备份列表与诊断副本是否纳入应用管理？
4. `command_receipt` 的保留时长/容量及 request 内容摘要算法如何冻结？
5. 全局 revision 与活动 session revision 是否使用单一 `state_revision`，设置 revision 独立？
6. 显示器 hint 的稳定标识范围及隐私要求是什么？
7. 是否首版省略 `session_day_index` 派生表，所有统计直接从历史计算？
8. 统计主显示采用 `actual_focus_ms`（completed + interrupted）还是 `completed_focus_ms`（仅 completed）？两个字段口径分立，完成轮次始终只统计 completed focus。

## 12. 依据

- SQLite [Transaction](https://www.sqlite.org/lang_transaction.html)：显式事务与提交/回滚语义；同一时刻只允许一个写事务。
- SQLite [Online Backup API](https://www.sqlite.org/backup.html)：运行中数据库快照备份和传统文件复制的边界。
- SQLite [PRAGMA](https://www.sqlite.org/pragma.html)：`user_version`、完整性检查及 SQLite 特定 PRAGMA 的行为须按获批 SQLite 版本核验。
- 项目规格：[模块接口契约](../../../interface-contract.md)、[验收矩阵](../../../acceptance.md)、[需求草案](../../../requirements.md)、[架构提案](../../../architecture.md)。
