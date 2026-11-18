# 本地统计服务提案

- 状态：提案草案，待需求与架构评审
- 对应任务：GitHub Issue [#13](https://github.com/troywjz/Pomodoro/issues/13)
- 来源：`docs/requirements.md`、`docs/architecture.md`、`docs/interface-contract.md`、`docs/acceptance.md`
- 范围：本地统计查询及其与计时记录、Repository、数据库和导出的边界；不包含业务实现
- 阶段门槛：Notion需求与新版Figma原型经用户确认、关键架构获批前，本文件保持提案状态，不启动业务代码、Cargo项目、实现型试验或数据库执行

## 1. 统计定义候选

### 1.1 日期归属

每个计时阶段开始时，协调器一次性采集并随该阶段记录保存：

| 字段 | 类型/格式候选 | 含义 |
|---|---|---|
| `started_at_utc` | UTC epoch 毫秒或 RFC 3339 UTC | 阶段开始瞬间；持久化格式由数据库提案冻结 |
| `start_local_date` | ISO 8601 `YYYY-MM-DD` | 阶段开始瞬间在当时系统时区对应的日历日期，作为统计归属日 |
| `start_offset_minutes` | 有符号整数 | 开始时本地 UTC 偏移分钟数；东经为正、西经为负 |

`start_local_date` 与 `start_offset_minutes` 是开始时快照，之后系统时区变化、夏令时切换或用户修改时钟都不重算它们。阶段跨越午夜时，整段结果归到 `start_local_date`。查询“今天”时由调用方提供当前本地日期值；统计服务不自行读取系统时钟或猜测时区。历史日期查询按同一日期字段筛选。

UTC 时间戳标识绝对时刻，偏移快照解释该时刻在开始地的民用时间。偏移值不包含时区规则身份；本提案不承诺根据它重建完整时区或未来夏令时规则。

### 1.2 时长与结果分类

- `actual_active_ms` 是计时内核累计的该段活动时长，单位毫秒。暂停区间不累计；统计服务不从墙钟起止时间相减推导它。
- focus 阶段的实际专注分钟为所有归属日期内 focus 记录 `actual_active_ms` 之和除以 60,000。建议内部始终返回整数毫秒，UI按明确舍入规则展示分钟；展示舍入方式仍待产品确认。
- 已完成轮次为 `phase=focus AND outcome=completed` 的唯一阶段记录数。`completed` 表示计时目标正常到期并成功持久化，不按实际时长阈值推断。
- `interrupted` 是用户提前结束、重置、恢复时放弃等未正常完成的阶段结果。focus 的 interrupted 活动时长计入实际专注时长，但不计完成轮次。
- `skipped` 是用户跳过休息的结果；当前命令契约只允许短休/长休阶段跳过。其活动时长照实保留，但不计入专注分钟或完成轮次。
- `outcome` 是单值枚举 `completed | interrupted | skipped`，每个 `session_id` 最多对应一条最终历史记录。活动中的阶段不作为历史结果计入汇总。
- 当前统计至少返回已完成专注轮次、实际专注毫秒/展示分钟，以及 completed、interrupted、skipped 分类计数。分类计数按阶段记录计数并保留 phase 维度，避免把跳过休息与中断专注混成一个总量。

同一 `session_id` 发生重试或重复完成投递时，Repository 幂等约束保证统计中只出现一条结果。查询仅汇总已提交历史；写入尚未确认时不乐观增加数字。

## 2. 服务输入/输出契约候选

统计服务为纯本地查询组件：输入明确日期与 Repository 查询端口，输出由已提交历史计算的不可变汇总值。它不写数据库、不控制计时、不读取系统时钟、不负责本地化格式化。

```text
TodayQuery {
  local_date: LocalDate,       // 调用方从当前系统时区取得；格式 YYYY-MM-DD
  // 可选：要求的统计定义版本，初版省略并使用当前批准定义
}

TodayStatistics {
  local_date: LocalDate,
  completed_focus_rounds: u64,
  actual_focus_ms: u64,
  outcome_counts: {
    focus: { completed: u64, interrupted: u64, skipped: u64 },
    short_break: { completed: u64, interrupted: u64, skipped: u64 },
    long_break: { completed: u64, interrupted: u64, skipped: u64 }
  },
  source_revision: Revision
}

StatisticsError {
  invalid_date,
  repository_read_failed { cause: RepositoryError }
}
```

`source_revision` 是 Repository 同一一致性读视图对应的历史 revision，用于诊断与界面刷新；revision 的来源和类型待数据库契约冻结。当前模型按 session 最终结果计数；如果未来允许对历史更正，Repository revision 变化后重新查询即可。

底层候选端口：

```text
Repository.query_sessions_by_start_date(local_date)
  -> (sessions: [SessionHistory], source_revision: Revision)
  | RepositoryError
```

候选记录投影仅需 `session_id`、`phase`、`outcome`、`actual_active_ms`、`start_local_date`。具体接口由数据库角色定义并与该提案协调。若数据库采用索引查询，日期过滤应在 Repository/数据库边界执行；若数据规模允许全量加载，统计服务仍使用相同语义，具体执行策略由数据库评审确定。

错误采用既有接口约定：Repository 读取失败原样转为可识别统计错误，调用方展示错误并保留上次成功值或明确标示不可用；失败不返回看似有效的零统计。

## 3. 时间变化与边界行为

| 情况 | 统计行为候选 |
|---|---|
| 一段从 23:55 跨到次日 00:20 | 该段完整实际活动时长和 outcome 归于开始时的 `start_local_date`。 |
| 暂停跨午夜后恢复 | 暂停毫秒不进入 `actual_active_ms`；恢复后活动累计仍归原开始日期。 |
| 时钟向前或向后调整 | 统计时长使用计时内核活动时长；不以 UTC 墙钟差计算。日期归属仍使用开始快照。 |
| 运行中系统时区变化 | 当前段归属字段不变；之后新开始的阶段使用新时区产生的日期与偏移。 |
| 夏令时跳变 | 统计不从本地时间差求时长；开始偏移记录跳变前/后的实际开始偏移。跨越 DST 的段仍按活动时长和开始日归属。 |
| 进程崩溃/睡眠恢复 | 依据已持久化历史的 `actual_active_ms` 汇总。恢复策略确认并产生最终历史结果后才计 outcome；无历史最终结果时不纳入汇总。 |
| 活动阶段尚未完成 | 不纳入完成轮次、结果计数或已归档专注分钟；UI可另行展示计时内核当前段快照。 |
| 同一完成事务重放 | 以唯一 `session_id` 和已提交历史保证一次计数。 |

UTC/偏移采集责任属于应用协调器/计时开始命令边界。计时内核提供可持久化的实际活动时长；Repository负责在同一历史记录中保存时间快照和结果，避免统计层拼接不同版本数据。

## 4. 可手算虚构样本

查询日期：`2026-04-03`。偏移为虚构样本字段，仅用于验证归属和计算，不描述任何真实用户。

| session_id | phase | 开始本地时间（仅便于阅读） | `start_local_date` | 偏移 | 实际活动 | outcome |
|---|---|---|---|---:|---:|---|
| `s-001` | focus | 2026-04-03 09:00 | 2026-04-03 | -300 | 25 分钟 | completed |
| `s-002` | short_break | 2026-04-03 09:25 | 2026-04-03 | -300 | 5 分钟 | completed |
| `s-003` | focus | 2026-04-03 23:55 | 2026-04-03 | -300 | 20 分钟（跨到次日） | interrupted |
| `s-004` | focus | 2026-04-04 00:10 | 2026-04-04 | -300 | 25 分钟 | completed |
| `s-005` | long_break | 2026-04-03 12:00 | 2026-04-03 | -300 | 0 分钟 | skipped |

日期 `2026-04-03` 的预期：

- `completed_focus_rounds = 1`（仅 `s-001`）
- `actual_focus_ms = (25 + 20) × 60,000 = 2,700,000 ms`，展示为 45 分钟
- focus 结果计数：completed 1、interrupted 1、skipped 0
- short_break：completed 1、interrupted 0、skipped 0
- long_break：completed 0、interrupted 0、skipped 1
- `s-004` 不属于该日期；`s-003` 整段归属开始日；`s-005` 不计专注分钟

人工校验只证明本例符合上述定义，不是代码或数据库测试记录。

## 5. 数据库与导出依赖

### 数据库角色依赖

- 依赖数据库提案提供最终 outcome、phase、毫秒时长、唯一 session ID、开始 UTC 时间、开始本地日期、开始偏移及 Repository 查询接口。
- 依赖一条历史记录作为一个阶段结果；focus 轮次与阶段记录一一对应。若数据库模型把一轮拆分为多条记录，需由数据库与统计角色共同确定稳定的轮次标识。
- 依赖唯一约束或等效事务规则，让同一 `session_id` 的重复完成/恢复提交不会制造重复统计。
- 依赖一致性读：一个汇总返回值中的记录与 `source_revision` 对应同一可见提交点。
- 日期字段、时间单位、枚举拼写和索引由 `docs/team/database/` 提案冻结；本文件中的类型为契约候选，不修改公共数据库文档。

### 导出角色依赖

- JSON/CSV 导出需使用相同日期归属、完成轮次和实际活动时长定义，避免导出数字与界面统计分叉。
- JSON 应保留原始 UTC、开始偏移、`start_local_date`、实际毫秒、phase 和 outcome，使用户可解释历史归属；schema version 与字段名由数据管理角色定义。
- CSV 应包含稳定日期列和时间偏移列，并明确实际时长单位或列标题；日期按保存的开始本地日期导出，不因导出机器当前时区重算。
- 若导出含每日汇总，每日聚合通过此统计定义计算；导出范围过滤需显式日期边界，日期边界采用包含式 `start_date` / `end_date` 候选。
- 导出时点的一致性快照和 revision 由 Repository 与数据管理角色确定；统计服务不自行写导出文件。

## 6. 待协调和验收项

1. 产品确认：展示分钟采用向下取整、四舍五入或 `分钟+秒`；服务原始字段保持毫秒以保留精度。
2. 数据库冻结：毫秒字段类型、日期序列化、时区偏移单位、唯一历史约束、查询投影与 `source_revision`。
3. 产品/计时角色确认：skip 是否只适用于休息，和 interrupted reason 枚举是否进入统计查询投影。
4. 数据管理角色确认：JSON/CSV schema、每日汇总一致性快照及导出日期边界。
5. Issue #1 的“今日”归属规则经需求评审确认后，本提案中的开始日期快照规则转为冻结契约。

### 验收矩阵（计划，待执行）

| ID | 输入/操作 | 预期结果 | 状态 |
|---|---|---|---|
| STAT-01 | 按第4节虚构样本查询 `2026-04-03` | 得到手算轮次、毫秒、分钟和按 phase 区分的 outcome 计数 | 计划 / 待执行 |
| STAT-02 | 查询 `2026-04-04` | `s-004` 纳入；跨午夜的 `s-003` 不纳入 | 计划 / 待执行 |
| STAT-03 | 一条 focus interrupted 记录含非零活动时长 | 时长进入实际专注总量，completed 轮次不增加 | 计划 / 待执行 |
| STAT-04 | paused 区间跨日，计时记录含活动时长 | 汇总仅采用 `actual_active_ms`，暂停区间不累计，归属开始日期 | 计划 / 待执行 |
| STAT-05 | 开始后修改系统时间或时区 | 已保存日期、偏移和实际活动毫秒不被重新计算 | 计划 / 待执行 |
| STAT-06 | Repository 对同一 `session_id` 重复提交结果 | 查询中仅有一条最终历史贡献 | 计划 / 待执行 |
| STAT-07 | Repository 读取失败 | 返回明确错误，不伪装成零值统计 | 计划 / 待执行 |
| STAT-08 | JSON/CSV 与界面查询同一已提交历史 | 日期归属、实际时长、结果分类及完成轮次定义一致 | 计划 / 待执行 |

以上均为验收计划；当前未创建数据库、未运行软件、未执行测试。
