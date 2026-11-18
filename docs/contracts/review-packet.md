# 共享契约评审包

状态：待产品/架构审批。组织分组已确定，以下保留技术和行为候选状态。

| 议题 | 共同方向 | 待裁决/冻结 |
| --- | --- | --- |
| 计时/时钟 | 单调活动时间差，UI低频刷新，墙钟记录提示 | Windows采样、漏通知阈值、Clock字段/错误 |
| 恢复 | 睡眠暂停，唤醒/崩溃询问，锁屏继续，不扣离线时间 | suspend通知内尽力保存，失败取可靠快照；15秒checkpoint为待实测目标 |
| 持久化 | 候选状态先事务提交再发布，完成按session去重 | global/session/settings revision、Tick版本、receipt保留和故障时序 |
| UI进度 | remaining/target同源，暂停冻结，到期归零 | idle已选/无目标、recovery和错误状态画板 |
| Geometry | 当前屏物理宽10%，先half-up再转DIP，两状态同宽 | API坐标空间、revision/合并时序及跨屏实测 |
| Theme | 跟随系统，首次查询/失败Light | 订阅失败重试、非客户区、辅助技术及状态token |
| 统计 | completed focus计完成轮次，按开始日归属 | actual_focus_ms是否含interrupted，候选拆分completed_focus_ms |
| 清理 | 清历史先备份、保留设置/活动，全清覆盖应用管理数据 | 最终确认、结束活动及各取消节点顺序，部分失败 |
| 导出 | 一致快照、版本JSON/固定CSV、UTC和原偏移 | 配置/活动快照范围、编码/列、导出与恢复权限 |
| 发布选择 | Rust+Slint Windows，仓库MIT保持 | Slint版本/features、renderer、许可归属、SQLite durability、参考机器 |

原始来源见[研究归档](../archive/role-studies/README.md)。评审通过后更新[公共契约](../interface-contract.md)的唯一最终字段定义，再分派实现任务。
