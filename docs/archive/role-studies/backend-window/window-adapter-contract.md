# 窗口平台适配研究与契约候选

- 状态：研究草案，待接口冻结与架构审批；当前无实现记录。
- 责任：后端·窗口适配；关联 Issue [#11](https://github.com/troywjz/Pomodoro/issues/11)。
- 需求输入：[UI 第二轮调整](../../../ui-revision-v2.md)、[需求草案](../../../requirements.md)、[架构提案](../../../architecture.md)、[接口契约](../../../interface-contract.md)、[验收矩阵](../../../acceptance.md)。
- 边界：仅研究 Windows 宿主窗口、几何与系统应用主题事件。Guest 内部不注入窗口/计时器；不承诺 UAC 安全桌面、Winlogon 或独占全屏覆盖。

## 结论摘要

1. Geometry 的来源是当前窗口中心所在显示器；若窗口中心落在所有显示器之外，选择主显示器。窗口拖动到其他显示器后，按新显示器的有效 DPI 与物理宽度重新计算尺寸。
2. `Wpx = round_half_up(display_width_physical_px / 10)`；`Wdip = Wpx × 96 / dpi`。例如 1920@100% 得 192px / 192DIP；3840@150% 得 384px / 256DIP。展开和收起共用同一几何宽度，不按空间限制缩小或增大目标比例。
3. DPI 改变、显示配置改变和主题颜色改变均成为平台事件，经 UI 线程交付协调器。平台适配不直接操作业务状态。
4. Windows 官方文档支持普通窗口置顶、DPI 与显示器枚举 API 的机制。Hyper-V VMConnect、混合 DPI 拖动、断屏恢复及具体 Slint backend 交互需要实机验收；本文没有执行这些测试。

## Geometry 契约候选

### 单位与公式

- `display_width_physical_px`：目标显示器可用显示模式的水平像素宽度，使用该显示器分辨率；不使用拼接虚拟桌面总宽。显示模式和像素宽度由显示器信息查询获得。
- `dpi`：目标窗口所在显示器的有效 DPI。缩放因子 `s = dpi / 96`；百分比显示缩放如 150% 对应 `dpi=144`、`s=1.5`。
- `round_half_up(x) = floor(x + 0.5)`，只用于物理目标宽度。计算顺序固定为先从显示器物理宽度得整数 `Wpx`，再换算 `Wdip`。DIP 使用浮点值传给布局；映射到 OS 整数窗口像素时采用最近整数、.5 向上。
- `Wpx = floor(display_width_physical_px / 10 + 0.5)`。
- `Wdip = Wpx × 96 / dpi`。禁止在窗口尺寸或布局尺寸上另加改变 10% 比例的 min/max clamp。显示器可用工作区不足时只修正位置并保留尺寸；内容可读性由产品和设计另行评审。
- 展开与收起持有同一个 `Wpx` 和 `Wdip`；仅高度及内容状态变化。等比设计参数由 UI 设计按 `W` 提供，不由 Windows adapter 猜测控件比例。

### 数值核对

| 显示模式 | 缩放 / DPI | `round_half_up(width/10)` | 逻辑宽度 |
|---|---:|---:|---:|
| 1366×768 | 100% / 96 | 137px | 137DIP |
| 1920×1080 | 100% / 96 | 192px | 192DIP |
| 2560×1440 | 125% / 120 | 256px | 204.8DIP |
| 3840×2160 | 150% / 144 | 384px | 256DIP |

在混合 DPI 双屏上，活动显示器自身宽度和 `GetDpiForWindow` 返回值构成一次几何输入；例如 1920px、96 DPI 的屏幕为 192DIP，3840px、144 DPI 的屏幕为 256DIP。不能把显示器物理宽度除以缩放后再计算 10%，也不能用桌面拼接宽度。

### 显示器选择和位置修复

1. 优先选择窗口外框中心所在的显示器。该规则支持窗口大部分在另一屏时仍由中心决定，且便于稳定触发 DPI/布局更新。
2. 若中心不与任何连接显示器相交，选择当前 Windows 主显示器。
3. 分辨率、DPI、显示器连接状态或主屏变化后重新枚举显示器并刷新选择。显示器句柄只作短期引用；每次拓扑变化后重查，不能持久化并复用旧 HMONITOR。
4. 选择结果变化时，以目标显示器工作区修复窗口位置，使标题/可交互窗口主体可重新找回。位置修复不得缩放或夹紧窗口宽度来改变 10% 规则。
5. 断开窗口所在显示器时将窗口移到主显示器工作区可见区域，再用目标显示器 Geometry 重新布局。

## Win32 置顶与 Hyper-V 范围

- `TopmostPolicy` 提案为用户明确启用的普通宿主桌面置顶。Windows `SetWindowPos` 使用 `HWND_TOPMOST` 可让窗口高于所有非 topmost 窗口，并在失焦后保持 topmost；应组合 `SWP_NOACTIVATE` 避免置顶/定位本身激活窗口。窗口被其他 topmost 窗口覆盖仍符合该定义。
- 不把置顶作为周期性抢焦点行为；窗口显示/恢复由用户操作触发，置顶状态与激活状态独立管理。
- Hyper-V 覆盖范围为宿主 Windows 交互桌面：Hyper-V Manager、普通宿主应用、窗口化 VMConnect 分别按宿主普通窗口置顶规则验收。Guest 画面属于 VMConnect 内容，应用不向 Guest 安装/注入组件。
- VMConnect 全屏、独占全屏应用、安全桌面分别记录观察结果和焦点；普通 topmost 不推导出对这些表面的可见性承诺。
- Windows 文档描述窗口管理器的 Z 序语义，没有为 VMConnect 所有显示模式定义 app-specific 保证。准确结论是宿主普通桌面置顶机制适用，VMConnect/全屏显示结果待目标版本实测。

## Theme 契约候选

- `ThemeMode` 初始策略固定为 `FollowSystem`，应用主题只有 `Light` 与 `Dark` 两值。
- 启动时先用 Light 初始化可绘制界面，再查询 Windows 应用颜色偏好；查询成功后发送当前主题。查询不可用、结果无效或订阅失败时保留 Light fallback。
- 订阅 Windows UI 颜色变化；变化事件重新查询偏好，并向 UI 线程发布最新 `ThemeChanged`。重复事件可合并，以最后一次查询结果为准。
- 初始查询或颜色变化订阅失败后保持 Light，并启动重试序列；候选间隔为 1、2、5、15、60 秒，之后每 60 秒重试一次。应用重新激活时额外触发一次立即重试，并与既有定时重试合并。
- 任一时刻只保留一个有效颜色变化订阅。重试前撤销旧订阅令牌；每次订阅使用递增 generation，旧 generation 回调丢弃。查询或订阅成功后停止定时重试、重置间隔，后续系统颜色事件触发查询；重复注册和重复通知合并成一次查询与至多一个主题版本更新。
- 主题切换更新全部语义色 token：背景/表面、正文/次级正文、边框、倒计时圆环已完成/未完成、主要/次要/禁用按钮、焦点/悬停/按下状态、错误与确认提示。平台适配只产出 Light/Dark 模式，不决定产品色值。
- Windows 提供 `UISettings.ColorValuesChanged` 颜色变化事件和 `GetColorValue` 颜色读取接口。颜色变化可用来重新检查系统偏好。具体 Windows API/WinRT bridge 的兼容性和依赖选择待架构审批。
- 深色标题栏属于 Win32 非客户区个别处理；Slint 客户区和系统标题栏分别验收主题匹配。

## Geometry / Theme 事件契约候选

事件通过 UI event loop 顺序投递；平台层不得从 Windows 回调直接修改 Slint 属性或持久化。几何更新按以下顺序合并，避免把一个 monitor 的宽度和另一个 monitor 的 DPI 拼成同一输入：

1. 窗口创建、窗口中心跨 monitor、`WM_DPICHANGED`、显示拓扑变化均只记录原因并安排一次 UI event loop 几何刷新；同一轮中的重复信号合并。
2. 刷新时重新枚举当前连接显示器，读取窗口中心、目标 monitor、完整物理矩形、工作区和该窗口有效 DPI；所有字段来自同一轮读取。遇到枚举期间再次变化时，丢弃该次候选并安排下一轮重新枚举。
3. 候选完整后分配严格递增的 `revision` 并发布一个 `GeometryChanged`。接收方拒绝 `revision <= current_revision` 的事件；排队中的旧版本不覆盖新几何状态。
4. `WM_DPICHANGED` 的建议矩形以物理屏幕像素解释，仅参与位置修复；之后仍按完整新快照重算宽度和布局。跨屏期间先完成 monitor 选择，再使用该 monitor 的 DPI 和显示宽度计算。

| 事件 | 字段候选 | 触发与处理 |
|---|---|---|
| `GeometryChanged` | `revision`（单调递增）、`monitor_id`（瞬时标识）、`monitor_physical_rect_px`、`monitor_work_area_physical_px`、`dpi_x`、`dpi_y`、`display_width_physical_px`、`window_center_physical_px`、`reason` | 首次建窗及屏幕选择、DPI、显示拓扑、窗口中心变化后发布完整且自洽快照；UI 仅采用比当前更新的 revision 并重算 W/H 与比例布局。 |
| `DpiChanged` | `old_dpi`、`new_dpi`、`suggested_rect_physical_px`、`monitor_id` | 来自 `WM_DPICHANGED`；只作为待处理原因入队。合并后重枚举 monitor 与 DPI，再发布完整 Geometry；建议矩形可作位置修复候选，最终宽度遵守 10% 规则。 |
| `DisplayTopologyChanged` | `reason`（resolution/attach/detach/primary/unknown） | 显示配置变化只作为待处理原因入队；同一批次合并 DPI/拓扑/窗口中心变化，重新枚举显示器并发布快照，不依赖旧句柄。 |
| `ThemeChanged` | `mode`（light/dark）、`source`（system/fallback）、`revision` | 首次系统查询及颜色变化重新查询后发布；UI 原子更新语义色集合。主题 revision 独立递增。 |
| `ThemeQueryFailed` | `stage`（initial/query/subscribe）、`error_kind`（不含私人路径或系统数据） | 发布诊断事件并继续 Light；失败恢复可在下次颜色变化或重试时转入系统主题。 |

协调器向 UI 暴露的稳定输入可冻结为：

```text
GeometrySnapshot {
  revision: u64,
  monitor_id: MonitorId,
  display_width_physical_px: u32,
  dpi_x: u32,
  dpi_y: u32,
  monitor_rect_physical_screen_px: RectI,
  work_area_physical_screen_px: RectI,
  window_center_physical_screen_px: PointI,
  reason: GeometryChangeReason
}

ThemeSnapshot {
  mode: Light | Dark,
  source: System | LightFallback,
  revision: u64
}
```

此为契约形状候选，最终字段名、monitor identity 生命周期、revision 共享方和事件版本需由 Lead 冻结。`RectI` 和 `PointI` 均采用 Windows per-monitor aware 的物理屏幕像素坐标；`suggested_rect_physical_px` 同样采用物理屏幕像素。转换为 Slint DIP 只在布局边界执行一次。`ThemeSnapshot.revision` 与 `GeometrySnapshot.revision` 各自独立单调递增。

## 官方依据

- Microsoft Learn: [`SetWindowPos`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowpos) 定义 `HWND_TOPMOST` 和 `SWP_NOACTIVATE`。
- Microsoft Learn: [`GetDpiForWindow`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getdpiforwindow) 说明 per-monitor-aware 窗口返回其所在显示器 DPI。
- Microsoft Learn: [`WM_DPICHANGED`](https://learn.microsoft.com/en-us/windows/win32/hidpi/wm-dpichanged) 说明窗口移动至不同 DPI 显示器或显示器 DPI 改变时的通知。
- Microsoft Learn: [High DPI Desktop Application Development](https://learn.microsoft.com/en-us/windows/win32/hidpi/high-dpi-desktop-application-development-on-windows) 描述 per-monitor DPI 与收到变化后重新布局/绘制。
- Microsoft Learn: [Multiple Monitor System Metrics](https://learn.microsoft.com/en-us/windows/desktop/gdi/multiple-monitor-system-metrics) 推荐用 `EnumDisplayMonitors` 和 `GetMonitorInfo` 取得单屏信息与工作区。
- Microsoft Learn: [`WM_DISPLAYCHANGE`](https://learn.microsoft.com/en-us/windows/win32/gdi/wm-displaychange) 说明分辨率改变通知；[HMONITOR and the Device Context](https://learn.microsoft.com/en-us/windows/win32/gdi/hmonitor-and-the-device-context) 指出显示变化后 monitor handle 可能失效。
- Microsoft Learn: [`UISettings.ColorValuesChanged`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.colorvalueschanged) 与 [`UISettings`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings) 描述颜色变化通知和系统颜色读取。
- Microsoft Learn: [Support Dark and Light themes in Win32 apps](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/apply-windows-themes) 提供 Win32 桌面应用探测主题变化和标题栏主题的指导。

## 验收计划与执行状态

| 案例 | 验收重点 | 状态 |
|---|---|---|
| W-01 | 四种指定分辨率/DPI 按表计算；重点复核 1920@100=192px/192DIP、3840@150=384px/256DIP | 公式人工复核；Windows UI 实测待执行 |
| W-02 | 展开/收起前后物理宽度恒定，DPI 改变后按新屏重算 | 待执行 |
| W-03 | 混合 DPI 双屏穿越边界，中心屏规则触发且布局重算；含窗口横跨屏幕中心变化 | 待执行 |
| W-04 | 断开窗口所在屏幕、调整分辨率、切换主屏；窗口移入仍连接屏的工作区并重算 | 待执行 |
| W-05 | 切换 Windows 应用浅/深主题运行时，全部颜色 token 同步更新；查询失败时 Light | 待执行 |
| W-06 | 普通窗口、Hyper-V Manager、窗口化 VMConnect、VMConnect 全屏、安全桌面分别记录可见性与焦点 | 待执行；VMConnect/全屏依目标 Windows 版本验证 |
| W-07 | 显示/定位窗口不抢焦点；窗口保持普通 topmost 定义，其他 topmost 窗口可覆盖 | 待执行 |

已执行工作限于官方文档检索、规格文档编辑与公式核算；Windows API、Slint event loop、主题订阅重试和事件幂等的运行行为均待 Windows 实机执行 W-01 至 W-07 验收。
