# UI 原型评审快照

> v1历史快照。当前四项UI调整和本地设计证据见[UI v2](design/ui-v2/ui-v2-review.md)，原生Figma同步状态与未决检查单独记录。

状态：草案，等待用户评审。来源：Figma 原型（Page 1）。关联任务：[Issue #3](https://github.com/troywjz/Pomodoro/issues/3)。

## 原型

[Figma：Pomodoro](https://www.figma.com/design/4U9TTroE0k4Rkef4K9LIG7/Pomodoro)

画布有五个 280×220 主卡片（idle 25:00、running 18:42、paused 18:42、break 05:00、completed 00:00）、200×64 mini、设置、休眠/崩溃恢复、数据管理、两类删除确认与成功示意，以及托盘菜单说明。主卡片、状态板、对话框均由可编辑文字和矢量图层构成。

整体采用深色中性底、橙色专注、绿色休息。产品文本使用 Noto Sans SC，计时数字使用 Inter。复用已导入的 Material 3 Button 和 Switch variants：设置完成按钮与置顶/音效两个开关为组件实例。Button variant 自带 Roboto 标签及紫色样式；此外观作为本次评审提案。

## 点击流程

Figma 原型设置了 ON_CLICK / NAVIGATE：

- idle 开始 → running；running 暂停 → paused；paused 继续 → running。
- completed 开始休息 → break；break 跳过 → idle。
- mini 展开 → running。各主卡片的“设置 / 收起”共用标签当前跳转settings；mini画板可单独预览。将设置与收起拆为两个独立入口列入设计修订项。
- 设置完成按钮的透明点击热区 → idle。
- 数据管理清空历史 → 确认；取消 → 数据管理；确认 → 清空完成；返回 → 数据管理。
- 永久清除 → 单独确认；取消 → 数据管理；确认 → 永久清除完成示意。

## 产品语义

自动开始下一轮首版关闭，界面显示静态说明，已移除该开关入口。完成状态等待用户操作，并可选择内置短提示音，默认关闭。休眠时暂停，锁屏时继续；崩溃恢复时询问并保持暂停时间。

清空历史先备份后删除，保留配置和当前活动。永久清除在结束活动后删除数据库、配置和应用备份，并停止新建备份。托盘菜单列出显示/隐藏、暂停/继续、设置、退出；进行中退出需确认，关闭窗口隐藏到托盘。

## 画板框架 IDs

- Idle `8:2`; Running `8:9`; Paused `8:16`; Break `8:23`; Completed `8:30`。
- Mini `8:37`; Settings `8:41`; Recovery `8:49`; Data management `8:55`。
- Clear history confirmation `9:488`; Delete success `9:527`。
- Permanent erase confirmation `10:27`; Permanent erase complete `10:34`。

## 验收记录

- 已成功导入并使用 Button component set key `ab924dce8b851fd820bd0d56da24a9c489311ae6`、Switch component set key `1f6ff0e7afa1181d166c5dbd3856ba6135b75019`；变体与实例 ID 记录在本地 ledger。
- 设计Agent检查框架尺寸、层级及主流程reaction；字体断言为Noto Sans SC、Inter及库Button内部的Roboto。Lead随后通过浏览器检查实际画布。
- Figma连接器达到Starter调用上限后，Lead改用浏览器原生UI复核。实际修复永久清除正文截断、清空历史确认的重复标题/正文，并移除自动开始开关。编辑过程产生的一处多余顶层文字已移除，最终图层列表及画布已复核。
- 浏览器实际点击确认：idle `8:2` → running `8:9` → paused `8:16` → running `8:9` → settings `8:41`，URL中的node-id与画面状态对应。
- 最终截图检查包括两类删除确认的放大视图、设置页以及完整画布。截图证据保留在本次聊天的工具记录；[此前组合截图](https://www.figma.com/api/mcp/asset/ee18d548-6491-47ea-a90d-8577127253b2.png)为修订前快照。
- 本任务未创建 Cargo 工程或业务代码。产品语义仍为评审提案。

## 本次评审需关注

- 原型为固定示例状态；计时数字和删除成功画面用于交互评审。
- 当前主要动作的prototype热区依附文字，点击按钮边缘时有区域未覆盖。设计确认后的下一轮修订将热区扩到完整按钮，并拆分设置/收起入口。
- 设置时长暂以文字显示，字段校验、退出确认、错误与全部恢复分支按需求及验收矩阵在下一轮交互细化中补齐。现有13个画板覆盖核心状态及两类数据删除流程。
