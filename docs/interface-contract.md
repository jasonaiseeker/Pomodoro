# 模块接口与数据契约提案

状态：待用户架构审批。本文定义协作边界，业务实现尚未开始。

## 单一状态来源

应用协调器串行处理 UI、托盘和 Windows 事件；计时内核接收显式命令与时钟样本，返回新 Snapshot 和待执行 Effect。UI 与托盘读取同一份 Snapshot，禁止各自维护计时进度。

| 接口 | 输入 | 输出与规则 |
| --- | --- | --- |
| TimerCore | Start、Pause、Resume、Reset、SkipBreak、Tick、Suspend、ResumeDetected、Recover/Discard | Snapshot、Persist、NotifyCompletion；状态转移按 session_id + revision 幂等 |
| Clock | 活动时间、系统启动经过时间、UTC墙钟、时区偏移 | 活动时间用于倒计时；启动经过时间与活动时间差用于辅助检测漏掉的睡眠事件；墙钟用于历史记录和恢复提示 |
| Repository | SaveTransition(expected_revision)、Checkpoint、LoadRecovery、QueryToday、Export、Clear | 成功返回已提交 revision；失败返回可展示错误，保留最后已提交状态 |
| WindowsShell | ApplyTopmost、SetVisible、SetMini、SetTray、RestorePosition | Suspend/Resume、DpiChanged、DisplayChanged、TrayAction、TrayFailed、CloseRequested；转换为协调器命令 |
| SlintView | Snapshot、可操作错误、恢复/删除/退出确认 | 发送命令，不直接读写数据库或调用 Windows API |

## 状态约束

- 阶段 focus / short_break / long_break，与 idle / running / paused / completed / recovery 状态分开。
- idle 开始进入 running；暂停冻结剩余时间；恢复建立新的活动时钟截止值；完成等待用户下一步。
- 提前结束/重置计 interrupted，跳过休息计 skipped；二者均不增加已完成专注轮次。
- 常规到期事务一次性完成：关闭活动段、追加唯一历史、更新轮次与快照 revision。提交成功后展示已保存结果。
- 睡眠通知触发暂停；唤醒进入恢复提示，用户确认后恢复。漏通知检测进入 recovery 并采用最后可靠快照，不自动补完成；检测阈值与 Windows 时钟 API 方案经专项测试定稿。
- 锁屏继续；改墙钟不扣当前剩余时间；崩溃重启从最近可靠快照询问恢复，默认保持暂停。
- Start/Pause/Resume/Reset/Complete/Exit 等转换先持久化；失败时展示保存失败、暂停新进度并提供重试，继续保留原快照。后台存储结果用 revision 检查，拒绝过期响应覆盖新状态。
- running 提案每 15 秒 checkpoint；异常终止恢复损失范围目标为最近 15 秒活动进度。状态转换不等待周期 checkpoint。周期与实际损失界限需要故障注入验收。

## 数据模型 v1 提案

所有时长使用整数毫秒；格式升级有 schema_version。单调时钟值只用于同一次系统启动内的运行，持久化恢复依靠已累计活动时间和剩余时间，不跨重启复用单调时钟原点。

| 实体 | 字段 |
| --- | --- |
| Settings | schema_version、focus/short/long_ms、long_break_interval、topmost、sound_enabled、window_logical_rect、monitor_hint |
| ActiveSession | session_id、phase、state、target_ms、accumulated_active_ms、remaining_ms、revision、last_checkpoint_utc、start_utc、start_local_date、start_offset_minutes |
| SessionHistory | session_id唯一、phase、target_ms、actual_active_ms、start/end_utc、start_local_date、start_offset_minutes、outcome(completed/interrupted/skipped)、reason |
| ExportEnvelope | schema_version、exported_at_utc、settings、sessions；CSV含UTF-8、固定列、时区字段与转义规则 |

存储候选为 `%LOCALAPPDATA%/Pomodoro` 中的 SQLite 单库及应用管理的 backups 目录。配置与历史共用 schema 迁移。应用单实例；第二次启动发送显示请求给既有进程，IPC 命名与访问控制在 Windows 实现任务中评审。

统计提案：今天完成专注轮次及实际专注分钟；按开始时本地日期归属，保存当时UTC偏移。历史和手动备份默认保留至用户清理。清空历史保留设置与活动段且先备份；永久清除结束活动段后删除库、配置和应用管理备份，用户自行导出的文件范围在确认中说明。

## 依赖与验收

先批准状态表、字段与保存失败语义，再并行实现内核、存储、WindowsShell 和 UI；接口变更由 Lead 审核并更新关联 Issue。验收覆盖假时钟、重复事件、写入失败、睡眠漏通知、强杀、跨日、迁移、导出、备份/永久清除。所有实测等待开发阶段。
