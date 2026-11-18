# 模块责任与依赖

组织分组已采用；代码目录和具体实现接口保持架构候选，业务开发待批准。

| 逻辑模块 | 责任 | 显式排除 |
| --- | --- | --- |
| domain | 纯计时、阶段/轮次、状态、注入Clock | Windows、UI和DB副作用 |
| application | 串行命令、候选状态、存储回执、effects、恢复/退出/数据流程 | 直接操纵控件或SQLite细节 |
| platform/windows | 窗口/主题/DPI、托盘、单实例、系统事件采集 | 自行推进业务倒计时 |
| storage | Repository、事务/schema/迁移、备份、查询 | 自行定义统计口径 |
| ui | Slint视图、主题、布局、交互、辅助技术 | 第二份计时状态或直接平台/DB访问 |

计时与应用分组可共同维护domain/application，保留纯内核与副作用协调边界。统计含义由产品确定，application组织，storage按契约查询。

application是跨层状态转换的唯一协调者：系统事件入串行队列，核心输出候选状态/effects，Repository确认事务后发布已保存结果，UI消费同源快照。

共享接口见[契约入口](../contracts/README.md)。当前只建文档，代码结构、renderer、许可和API版本待架构批准后锁定。
