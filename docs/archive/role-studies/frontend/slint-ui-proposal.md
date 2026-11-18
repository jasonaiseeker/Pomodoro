# Slint 前端结构与接口提案

**状态：草案，待新版 Figma 原型、计时接口和窗口接口对齐后冻结。**

**范围：** Slint 前端组件、属性、回调、比例布局、无障碍及主题/展开交互。本文不定义业务实现。

## 设计约束

- 倒计时数字前方显示剩余进度圆圈；数字是精确读数，圆弧是剩余时间的视觉概览。
- 所有按钮标签水平与垂直居中，按钮完整矩形都是点击与触控热区。
- 主题默认浅色，后续跟随系统浅/深主题，并响应运行时主题变化。
- 展开和收起共用同一窗口宽度：当前窗口所在显示器物理宽度的 10%。布局以物理宽度和 DPI 转换得到逻辑尺寸；不添加最小/最大宽度夹限。
- 展开状态增加内容并调整高度，收起状态保留计时与必要操作；两种状态的宽度不变。
- 该比例规则优先于可读性偏好。窄屏上的字形、信息密度和操作目标需由原型显式评审。

## 组件树候选

```text
PomodoroWindow
└── WindowSurface (Geometry、Theme、expanded、windowStatus)
    ├── CompactTimerPanel
    │   ├── HeaderRow (phaseLabel, statusLabel, SettingsButton, ExpandButton)
    │   ├── TimerRow
    │   │   ├── CountdownRing (progress, accessibleProgress)
    │   │   └── CountdownReadout (timeText, phase/status accessible description)
    │   └── PrimaryActionButton (Start / Pause / Resume / Continue 等状态标签)
    ├── ExpandedDetails (仅 expanded)
    │   ├── SessionSummary (今日统计等，字段依产品确认)
    │   └── SecondaryActions (Reset / Skip / End 等按状态启用)
    ├── StatusMessage (保存错误、恢复提示、托盘错误等)
    └── ConfirmationDialog (退出、放弃、数据操作等；按需显示)
```

组件职责边界：`PomodoroWindow` 将应用快照、几何和主题状态传入视图，并把 UI 意图上送协调器；`CompactTimerPanel` 只呈现计时及主操作；`CountdownRing` 只消费归一化进度；`ExpandedDetails` 只呈现展开专属信息；弹窗与消息只呈现协调器给出的提示和选项。Slint 视图不自行累计时间、不调用 Windows API、不直接访问 Repository。

## 属性与回调候选

字段类型是接口讨论用的语义候选，不是已批准的 Rust / Slint 类型签名。所有时间值单位建议为毫秒；显示字符串由协调器或纯展示适配层生成，视图不解析本地化时间文本。

| 接口 | 候选字段 | 用途 / 约束 | 状态 |
|---|---|---|---|
| `Geometry` | `display_width_physical_px`、`dpi_x`、`dpi_y`、`work_area_physical_px`、`window_center_physical_px`、`reason`、`Wpx = floor(display_width_physical_px / 10 + 0.5)`、`Wdip = Wpx × 96 / dpi_x`、`expanded_height_dip`、`compact_height_dip` | 按窗口适配契约候选，坐标在平台边界使用物理屏幕像素。两种状态都使用同一个 `Wdip`；高度由 Figma 比例规则给出。窗口中心选择显示器；中心无有效屏时回退主屏；跨屏、DPI、分辨率变化时重算并修正位置至工作区可见。 | 待窗口接口冻结 |
| `Theme` | `mode`（`light` / `dark`）、`source`（`system` / `light_fallback`）、`revision`、`surface`、`text_primary`、`text_secondary`、`accent_focus`、`accent_break`、`ring_track`、`ring_value`、`button_primary`、`button_disabled`、`focus_outline`、`error` | 主题快照的 `mode/source/revision` 与窗口适配契约候选对齐，颜色值映射为语义 token。默认 `light`；系统主题查询成功即采用当前主题，查询前或失败保持浅色。系统变化更新整套 token，包括弹窗与错误/禁用/焦点态。 | 颜色值待 Figma / 产品冻结 |
| `TimerSnapshot` | `phase`、`state`、`target_ms`、`remaining_ms`、`elapsed_ms`、`time_text`、`phase_label`、`status_label`、`primary_action_label`、`can_start`、`can_pause`、`can_resume`、`can_reset`、`can_skip`、`can_end`、`revision` | 同源计时快照；UI 不维护第二份倒计时状态。`time_text` 是权威显示读数。圆圈进度候选为 `clamp(remaining_ms / target_ms, 0, 1)`；目标为 0 或无活动阶段时显示静态空闲态，不除零。 | 时间与枚举字段待计时接口冻结 |
| `ViewState` | `expanded`、`status_message`、`dialog_kind`、`dialog_options`、`focused_control`（若键盘恢复焦点需外部控制） | 描述仅 UI 展开/提示状态；持久设置是否记录展开状态另行决定。 | 待设计确认 |

### UI 意图回调候选

| 回调 | 载荷 | 协调器动作边界 |
|---|---|---|
| `start_requested(phase)` | phase | 请求开始；合法性及默认时长由应用/计时层验证 |
| `pause_requested` / `resume_requested` | 无 | 发送计时命令；快照回写后再更新按钮状态 |
| `reset_requested` / `skip_break_requested` / `end_requested` | 无 | 转成计时协议命令；需要确认时先打开相应对话 |
| `settings_requested` | 无 | 请求打开设置视图/窗口；属于独立入口 |
| `expanded_changed(expanded)` | bool | 请求展开或收起；不得改变宽度 |
| `dialog_choice(choice_id)` | 稳定选项 ID | 发送退出/恢复/清理等用户选择；具体选项来自协调器 |
| `retry_requested(operation_id)` | 操作 ID | 对可重试错误请求重试 |

`TimerCore` 的命令全集以 [接口契约](../../../interface-contract.md) 为准。上述名称只是 Slint 与协调器之间的候选 UI 意图；最终命名、参数与确认流程需和该契约合并，避免重复定义状态机。

## 环形进度与时间刷新

- 起始满环表示 `remaining_ms == target_ms`；目标耗尽为零进度。暂停冻结剩余值和圆弧，恢复后继续变化；完成状态显示零进度和完成标签。
- 视图使用单个归一化数值驱动画弧。视觉更新建议每秒一次，命令事件可立即刷新；计时精度和到期转换由计时内核根据时钟差负责，不依赖刷新次数。
- 圆弧方向、起始角、线宽、缺口、端点样式、直径/窗口宽度比例及动画策略由新版 Figma 指定。默认不采用持续动画，避免与每秒精确数字产生不同步暗示。
- 空闲无目标时，圆圈展示中性轨道或由设计确定的空闲装饰；辅助技术朗读“未开始”，不报告虚构百分比。

## 比例排版与控件热区

- 几何输入链：窗口适配层给出所在显示器物理宽度和 DPI；先计算 `Wpx = round(physical_width_px / 10)`，再计算 `Wdip = Wpx / (dpi / 96)`。Slint 组件仅收到逻辑宽度和布局令牌，不另算屏幕比例。
- Figma 以逻辑尺寸展示并注记对应物理输出。窗口宽度 W 在紧凑/展开状态保持一致；字体、圆直径、内边距、间距、圆角、按钮高度按 W 的比例定义。
- 设计稿需覆盖 1366×768@100%、1920×1080@100%、2560×1440@125%、3840×2160@150% 和混合 DPI 双屏。物理目标宽度约为 137/192/256/384 px；逻辑宽度按比例换算。测试结论应记录真实显示器识别值与换算值。
- 主要按钮的标签和图标组在完整按钮矩形的水平、垂直中心对齐；完整矩形处理鼠标、触摸和键盘激活。热区不能依附文字节点。焦点环位于按钮边界内外的具体规则由原型标注。
- 10% 约束造成的极窄场景需展示实际字级、折行/省略策略、圆环与时间的空间关系和完整按钮热区；由用户评审可读性取舍，不由前端自行更改比例。

## 键盘与辅助技术

- 所有主按钮、设置入口、展开入口、次级动作、对话框选项按视觉顺序进入 Tab 顺序；Tab/Shift+Tab 移动焦点，Enter/Space 激活焦点按钮，Esc 关闭非破坏性面板或返回，具体全局快捷键由产品确认并显示在 UI 提示中。
- 焦点始终可见且对比度满足设计验收；禁用状态可感知并不可触发。鼠标悬停/按下不是唯一的状态反馈。
- 圆环作为进度语义组件向辅助技术报告 phase、剩余时间、状态及可选百分比。时间数字提供易读文本；避免圆环和数字造成重复播报。
- 倒计时每秒视觉刷新，屏幕阅读器不每秒主动播报。焦点在计时区域时允许用户主动读取；暂停、恢复、完成、恢复询问、错误和保存失败以一次性状态通知呈现。
- 弹窗打开后焦点移入标题或首个可操作项，Tab 在弹窗内循环；关闭后焦点回到触发控件。验证 UIA/屏幕阅读器名称、角色、状态和值，以及高对比度下环与按钮的辨识度。
- Slint 官方通用属性文档定义了 `accessible-role`、`accessible-label`、`accessible-value` 和 `accessible-live-region` 等属性；无障碍进度控件需要角色和可访问值。采用的 Slint 版本、Windows 辅助技术映射与实际播报仍须实机验证，当前不记作已验证能力。[Slint Common Properties & Callbacks](https://docs.slint.dev/latest/docs/slint/reference/common/)

## 展开/收起与主题变化

- 展开与收起只改变 `expanded`、高度及可见内容；宽度始终为 Geometry 提供的 W。窗口尺寸更新由窗口适配层执行，Slint 发出意图并消费更新后的 Geometry。
- 收起时保留阶段、精确时间、圆环、主操作、设置入口和展开入口；展开时按原位延伸/增高的方式显示 Figma 指定信息。隐藏内容不进入 Tab 顺序，也不被辅助技术遍历。
- 主题初始化默认为浅色。系统主题读取完成后一次性替换 `Theme` token；运行期间收到系统主题变化时替换全部 token，保持几何、计时和焦点不变。读取失败或暂未完成时显示浅色主题。
- 颜色 token 覆盖底面、前景、文字层级、圆环轨道/值、按钮各状态、焦点环、确认框、错误与提示。文本和状态不能只靠颜色区分。

## 依赖关系与待冻结字段

| 上游依赖 | 前端输入 / 需要冻结的字段 | 前端交付后的下游 |
|---|---|---|
| 新版 Figma（设计角色） | 完整/紧凑画板、浅/深主题 token、圆弧角度与方向、控件顺序、各比例、折行与隐藏规则、焦点/禁用/悬停/按压状态、展开高度和紧凑高度 | 产品确认交互与视觉；测试据此制作截图和可访问性验收步骤 |
| `Geometry` / Windows 窗口适配（backend-window） | `GeometrySnapshot` 候选（`display_width_physical_px`、`dpi_x/y`、`work_area_physical_px`、`window_center_physical_px`、`reason`）、`GeometryChanged`/`DpiChanged`/`DisplayTopologyChanged` 时序；`round_half_up` 规则、DIP 与 OS 整数像素映射 | 前端提供渲染所需几何，发出展开尺寸意图；窗口层控制真实窗口尺寸和位置 |
| `TimerSnapshot` / 计时内核（backend-timer） | phase/state 枚举、时长单位、目标/剩余值更新频率、空闲/完成/恢复字段、revision 及持久化失败快照语义 | 前端展示同源快照并发送候选 UI 命令；协调器串行化并回传新快照 |
| 系统主题检测（窗口/应用协调器） | 初始主题结果、变化事件、失败语义、主题 token 的单一来源 | Slint 根视图替换 Theme 属性；不自行探测 Win32 |
| 产品需求（Notion；仅产品角色编辑） | 主操作状态文案、重置/跳过/结束确认、快捷键、统计呈现、失败/恢复提示文案 | 产品验收文本与 Issue 追踪 |

关键冻结事项：Geometry 是逻辑还是物理坐标、圆弧刷新快照的时序、`remaining_ms` 是否由每次 tick 重发或根据 deadline 求值、当前计时状态枚举、主题事件所有者、设置面板是否另窗、10% 宽度取值来源和四舍五入规则、按钮最小可交互面积与 10% 约束冲突的用户决策、Slint UIA 能力。未冻结前本文仅供接口协调，不作为实现签名。

## 验收矩阵建议

以下均为计划项，未执行：

| 检查 | 预期证据 |
|---|---|
| 四项 UI 要求 | 新版 Figma 上倒计时前有圆环、所有按钮标签居中且热区完整、默认浅色/系统深浅切换、展开收起同宽 10% 计算标注 |
| 尺寸矩阵 | 五种指定环境的 `monitor physical width → Wpx → DPI → Wdip` 对照记录；展开和紧凑宽度相等 |
| 圆环与快照 | 初始满环、暂停冻结、恢复推进、到期归零；数字和状态始终对应同一 TimerSnapshot |
| 操作边界 | Tab 顺序、焦点可见、Enter/Space 激活、对话框焦点进出、禁用控件不触发 |
| 辅助技术 | Windows UIA 检查记录及屏幕阅读器对初始、运行、暂停、完成、错误状态的播报记录；无逐秒重复通知 |
| 主题/展开 | 系统浅/深主题启动及运行中切换；失败回落浅色；几何和焦点保持；两状态同宽 |

上述验证需在原型确认、接口冻结及可执行应用获批后开展。当前没有 Slint/Rust/Cargo 代码、运行时验证或 UIA 实测。
