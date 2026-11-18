# 应用层数据操作契约提案

状态：草案，待产品、数据库与架构评审；业务实现未启动。
来源：GitHub Issue [#14](https://github.com/troywjz/Pomodoro/issues/14)、[需求草案](../../../requirements.md)、[架构提案](../../../architecture.md)、[模块接口与数据契约](../../../interface-contract.md)、[验收矩阵](../../../acceptance.md)。

## 范围与边界

本角色定义应用层 JSON/CSV 导出、应用管理备份、清空历史和永久清除的流程及用户可见结果。应用协调器负责确认、操作编排和反馈；SlintView 只发起命令并呈现状态；Repository 按冻结后的接口读取一致快照、生成备份、清理指定数据并返回结构化结果。此提案不定义 SQLite 表、文件布局、事务实现或存储引擎。

| 数据范围 | JSON | CSV | 清空历史 | 永久清除 |
| --- | --- | --- | --- | --- |
| 配置 | 导出快照 | 不导出 | 保留 | 删除 |
| 已完成历史 | 导出 | 导出 | 先备份再删除 | 删除 |
| 活动会话/恢复快照 | 默认不导出；用户可选“包含恢复状态” | 不导出 | 保留 | 确认前先结束活动会话，之后删除 |
| 应用管理的备份 | 不作为普通导出内容 | 不导出 | 清空历史后保留新备份及既有备份 | 删除，不新建备份 |
| 用户另存的导出文件 | 用户选择保存位置 | 用户选择保存位置 | 不触碰 | 不触碰，确认页明示由用户管理 |

JSON 的“包含恢复状态”选项仅供用户备份/迁移；其中包含活动阶段、状态、剩余时间和 revision，不包含单调时钟原点。普通 JSON 导出不包含此类可恢复状态，避免把查看用途误作恢复输入。导出均为用户显式发起，路径由系统保存对话框选定；应用不扫描、移动或删除用户外部文件。

## 操作流程

| 操作 | 确认与执行顺序 | 成功结果 | 取消/失败结果 |
| --- | --- | --- | --- |
| 导出 JSON | 选择范围与保存路径 → Repository 读取单一一致快照 → 序列化版本化 envelope → 写目标文件 → 完成后报告路径 | 报告记录数、是否含配置/恢复状态及文件路径 | 保存对话框取消不读写；写入失败报告目标不可用等原因，移除本次不完整临时文件，源数据不变 |
| 导出 CSV | 选择保存路径 → 读取同一统计口径的历史快照 → 固定列序和编码序列化 → 写目标文件 | 报告导出行数和文件路径 | 取消不读写；读写/磁盘失败时报告失败，源数据不变，不显示成功 |
| 手动备份 | 选择“立即备份” → Repository 创建数据库一致性备份并验证可读性、schema 支持及完整性 → 记录备份清单 → 报告位置与时间 | 备份进入应用管理备份范围，默认保留到用户清理 | 取消无变化；磁盘写入或验证失败时不标记成功、不删除旧备份、不改变源数据 |
| 清空历史 | 展示影响摘要 → 二次确认 → 检查活动状态及 revision → 创建并验证包含被删历史的前置备份 → 原子清除历史并更新统计 → 返回提交 revision | 历史为空；配置和活动会话保留；备份保留；显示条数、时间及备份位置 | 首次页面取消不变；确认后若备份、revision 检查或清理失败则中止，历史保持原状并给出恢复/重试指引 |
| 永久清除 | 展示范围和外部文件提示 → 若存在活动会话，要求先结束并明确结束结果 → 单独二次确认，明确该操作不会新建备份 → 检查 revision → 清除历史、配置、活动快照及应用管理备份 | 清除完成后进入干净初始数据状态；用户导出的外部文件保持原位 | 任一确认取消则不执行清除；若清除未提交则保留原数据和备份，报告失败；不可在部分成功时报告整体成功 |

二次确认须使用明确动作词，例如“清空历史”或“永久删除本机全部应用数据”，确认按钮不得使用“确定”作为唯一说明。永久清除前结束活动会话按计时契约选择保存为 paused 或记录 interrupted；用户取消退出流程时不开始永久删除。数据清理操作期间禁止接受会改变目标范围的并行写入；Repository 以 expected_revision 拒绝过期命令，协调器刷新快照后由用户重新确认。

## 版本化 JSON envelope 候选

文件为 UTF-8 JSON；字段名使用 snake_case。`schema_version` 是导出格式版本，与内部数据库 schema 版本分离。时间戳为 RFC 3339 UTC；本地归属日期及当时偏移独立保留。未知的未来 envelope 主版本应被拒绝导入（导入不属于本轮实现范围），不允许静默忽略字段。

```json
{
  "format": "pomodoro.export",
  "schema_version": 1,
  "exported_at_utc": "2026-01-02T03:04:05Z",
  "scope": {
    "settings": true,
    "sessions": true,
    "active_session": false
  },
  "settings": {
    "focus_minutes": 25,
    "short_break_minutes": 5,
    "long_break_minutes": 15,
    "long_break_interval": 4
  },
  "sessions": [
    {
      "session_id": "example-id",
      "phase": "focus",
      "target_ms": 1500000,
      "actual_active_ms": 1500000,
      "start_utc": "2026-01-02T02:39:05Z",
      "end_utc": "2026-01-02T03:04:05Z",
      "start_local_date": "2026-01-01",
      "start_offset_minutes": -360,
      "outcome": "completed",
      "reason": null
    }
  ],
  "active_session": null
}
```

会话按 `start_utc`、再按 `session_id` 升序输出，保证相同快照序列化顺序稳定。`phase` 取 `focus`、`short_break`、`long_break`；`outcome` 取 `completed`、`interrupted`、`skipped`。`reason` 使用已批准的枚举或 null，不写本机路径、用户名、机器名、内部网络信息或运行环境诊断。选择活动快照时 `active_session` 使用 `session_id`、`phase`、`state`、`target_ms`、`accumulated_active_ms`、`remaining_ms`、`revision` 和最近 checkpoint UTC 字段；不得含可跨启动复用的单调时钟值。以上字段为契约候选，需数据库角色确认映射。

## CSV 列与转义候选

编码采用 UTF-8 BOM，便于常见表格应用识别；首行固定列头，列顺序如下：

```text
session_id,phase,outcome,target_ms,actual_active_ms,start_utc,end_utc,start_local_date,start_offset_minutes,reason
```

所有字段按 RFC 4180 规则处理：含逗号、双引号、CR 或 LF 的字段用双引号括起，字段内双引号重复为两个双引号；每条记录 CRLF 结束。整数时长为毫秒，UTC 时间为 RFC 3339，`start_local_date` 为 `YYYY-MM-DD`，偏移为从 UTC 起算的整数分钟。无值输出空字段；CSV 中不夹带汇总行，以保证每行均为单条历史记录。

`reason` 若允许未来扩展为用户可编辑文本，需在最终定稿中决定表格软件公式注入保护策略，并明确原值与显示值差异；当前 reason 枚举不含自由文本。数据库字段、reason 允许值与 BOM/导入兼容性是待冻结项。

## 错误与中止语义

| 错误码候选 | 用户反馈 | 必须保持的条件 |
| --- | --- | --- |
| `operation_cancelled` | 操作已取消 | 数据及既有备份不变；不弹出失败告警 |
| `revision_conflict` | 数据已变化，请刷新后重新确认 | 不清理任何记录；刷新快照与摘要 |
| `snapshot_read_failed` | 无法读取一致数据，请重试 | 源数据不变，不生成部分成功导出 |
| `destination_write_failed` | 无法写入所选位置，请检查磁盘空间和权限后重试 | 删除本次未完成临时文件；源数据不变 |
| `backup_create_failed` | 备份未完成，清空已中止 | 历史、配置、活动会话、旧备份保持；不继续删除 |
| `backup_validation_failed` | 备份校验失败，清空已中止 | 不将备份列为有效备份；源数据保持 |
| `history_clear_failed` | 历史清理未完成，请刷新并重试 | Repository 原子语义保证全部历史仍在或全部清除；失败结果不宣称成功 |
| `active_session_requires_resolution` | 请先结束或保存当前计时，再永久清除 | 永久清除阶段尚未开始；活动快照保持 |
| `permanent_clear_failed` | 永久清除未完成，应用数据仍可检查 | 不报告整体完成；展示可执行的重试或支持步骤 |
| `unsupported_export_version` | 文件格式版本不受支持 | 本版本不导入；源数据不变 |

所有结果携带 `operation_id`、最终 revision（若适用）、影响范围摘要和可展示的内部错误分类。界面错误不显示 SQL、绝对本机路径或诊断转储。操作取消、读取失败、磁盘写入失败和活动会话阻止清理均为可重复验收情景。

## Repository 边界候选

应用服务请求/结果候选：

- `ExportSnapshot(scope, expected_revision) -> Snapshot | OperationError`：读取设置、历史和可选活动快照的同一 revision；序列化与目标文件交给应用层。
- `CreateBackup(reason, expected_revision) -> BackupReceipt | OperationError`：为迁移、手动备份或清空前备份创建一致性备份并返回校验状态；不得暴露存储布局给 UI。
- `ClearHistory(expected_revision) -> ClearReceipt | OperationError`：与配置及活动快照保留语义原子绑定。
- `ClearAllLocalData(expected_revision) -> ClearReceipt | OperationError`：清除应用管理数据范围，不创建备份；成功后返回空初始快照/revision。

名称和参数是供数据库角色对齐的候选，不替代其 Repository 接口设计。配置字段、活动会话快照和历史字段以数据库最终契约为准；字段不匹配时由两个角色联合提案变更，再由 Lead 冻结。

## 验收案例（计划，待开发阶段执行）

| ID | 步骤 | 预期 | 当前状态 |
| --- | --- | --- | --- |
| DATA-01 | 选择 JSON/CSV 保存对话框后取消 | 不读写数据、不创建文件，界面返回原状态 | 计划，待执行 |
| DATA-02 | 在导出过程中模拟目标磁盘写入失败 | 显示失败；不保留伪成功文件；本地数据和 revision 不变 | 计划，待执行 |
| DATA-03 | 改变数据后提交旧 revision 的导出/清理命令 | 返回 revision conflict，清理不开始，摘要刷新 | 计划，待执行 |
| DATA-04 | JSON 导出并解析 envelope | 版本、scope、UTC 时间、配置、排序历史与选择性活动快照符合字段契约 | 计划，待执行 |
| DATA-05 | 将逗号、引号、换行测试值序列化为 CSV 并用解析器读取 | 解析后字段值和固定列顺序一致，CRLF 与编码符合契约 | 计划，待执行 |
| DATA-06 | 对 CSV/JSON 导出目标注入不可写磁盘 | 报错并中止，无数据变更 | 计划，待执行 |
| DATA-07 | 清空历史页首次确认后选择取消 | 历史、设置、活动快照和备份均不变 | 计划，待执行 |
| DATA-08 | 有活动会话时查看清空历史摘要并确认 | 明示历史删除范围；清空后配置/活动快照保留 | 计划，待执行 |
| DATA-09 | 清空历史时模拟前置备份写入或验证失败 | 删除流程中止；所有原历史仍可读取；错误可操作 | 计划，待执行 |
| DATA-10 | 清空历史时模拟数据库清理失败 | Repository 原子失败；原历史保留；备份可读取；不报告成功 | 计划，待执行 |
| DATA-11 | 永久清除页面首次确认后取消 | 配置、历史、活动快照、备份全部不变 | 计划，待执行 |
| DATA-12 | 有活动会话时发起永久清除 | 要求先明确保存暂停快照或记录 interrupted；未处理前禁止清除 | 计划，待执行 |
| DATA-13 | 永久清除后检查应用和用户文件范围 | 配置、历史、快照及应用备份清除；不生成新备份；用户外部导出保留 | 计划，待执行 |
| DATA-14 | 永久清除期间模拟磁盘/Repository 错误 | 不给整体成功反馈；返回可重试错误与清晰状态 | 计划，待执行 |
| DATA-15 | 运行、暂停与恢复状态分别执行导出及清空历史 | 导出快照来自单一 revision；清空保留会话；计时协调器无竞态覆盖 | 计划，待执行 |

未执行软件测试、数据库操作、实现型试验或安装。以上均为计划，需新版 Figma/Notion 与关键架构经用户确认并进入开发阶段后执行。

## 依赖与待冻结项

1. 数据库角色确认 Snapshot 一致读取、备份校验回执、清理原子性、revision 协议与永久清除失败边界。
2. 产品/Lead 冻结普通 JSON 是否包含设置（此稿提案为包含）、是否允许可选活动快照，以及导出隐私提示文案。
3. 数据库角色确认 `reason` 域、历史字段、UTC/偏移字段及恢复快照字段映射。
4. 架构评审确定 CSV BOM、格式版本演进与公式注入策略；若 reason 引入自由文本，须在实现前完成安全与数据保真取舍。
5. 明确应用管理备份的实际用户可见位置和备份清理入口；本角色不决定文件布局。
