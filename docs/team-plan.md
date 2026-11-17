# 第一阶段协作与交付计划

状态：待用户评审。业务开发尚未启动。

## 权威入口

- [Notion 项目页](https://www.notion.so/41be33444f5f80698ee7d99318343c08)：需求、架构和审批记录。
- [Figma 原型](https://www.figma.com/design/4U9TTroE0k4Rkef4K9LIG7/Pomodoro)：可编辑 UI 与交互。
- [GitHub Issues](https://github.com/troywjz/Pomodoro/issues)：任务状态。
- 当前文档分支 `docs/phase1-planning`：版本化评审快照。PR 合并由项目所有者批准。

## 已实际建立的任务

| 角色 | 模型 | 隔离分支 | 写入范围 | 验收 |
| --- | --- | --- | --- | --- |
| Lead | 用户当前选定模型 | docs/phase1-planning | AGENTS、README、集成及验收记录 | 审核真实 diff、来源及产物 |
| Requirements | gpt-6-luna | docs/requirements | requirements.md、acceptance.md | 可执行验收步骤、状态/数据语义完整 |
| Platform | gpt-6-luna | docs/architecture | architecture.md、decisions.md | 官方证据、模块边界、决策待批准 |
| Design | gpt-6-luna | docs/design | Figma、design.md | 编辑图层、状态、主流程、组合截图 |
| Reviewer | gpt-6-luna | 独立审查任务 | 只读实际产物，提交审查结论给 Lead | 发现有据可查、修复后复核 |

三个执行 Worktree 已通过 Codex 原生工具实际创建，均从已提交的规范基线派生。原生多 Agent 与指定 Luna 模型均已启动成功。完成后的文档提交由 Lead 逐一复核并 cherry-pick 到评审分支；任务分支保留供追溯。

## 环境检查结果

| 项目 | 观察结果 | 下一步 |
| --- | --- | --- |
| 仓库 | 公开仓库；已有初始提交、MIT LICENSE 与 Rust .gitignore | 保留初始文件，后续以文档 PR 提交 |
| Git / GitHub | 本地 Git、CLI 与连接器可用；具有仓库写权限 | 每次推送先做公开内容与历史检查 |
| 网络 | Git 直连连接重置；单次本地代理重试成功 | 对后续 Git 网络操作使用单次代理 |
| Windows | Windows 11 x64；检测到 hypervisor | 开发后测试实际 VMConnect 窗口与全屏场景 |
| 原生工具 | MSVC 14.51、Windows SDK 10.0.26100.0、CMake、Ninja 可见 | 编译验证放到用户批准后的环境准备任务 |
| Rust | 当前终端命令查找未找到 rustc/cargo/rustup；默认用户安装位置也未找到 rustc | 开发前安装并锁定工具链，再记录版本与构建结果 |
| Notion | 已读取已有 Pomodoro 项目页并写入项目规划 | 规格与决策关联本项目，任务链接至 GitHub |
| Figma | 指定文件读取成功、初始页面为空 | 完成原型后按图层与截图独立验收 |

以上为环境检查与设计阶段证据。产品编译、性能、Windows 交互与 Hyper-V 实机验收安排在开发阶段。

## 开发阶段拆分提案

前置条件：用户明确确认 Notion 规格和 Figma 原型，批准关键架构与许可路线。下列任务为后续计划。

| 工作包 | 独占范围与输出接口 | 依赖 | 验收 |
| --- | --- | --- | --- |
| 工具链与构建 | Cargo/Slint 版本、依赖 features、Windows CI | 用户批准架构 | Windows release 构建、许可清单 |
| 计时内核 | 纯状态转换、Clock 输入、Snapshot 与 Effect 输出 | 工具链；冻结状态接口 | 假时钟测试、暂停/恢复/边界及幂等完成 |
| 本地存储 | Repository 接口、事务、版本迁移、导出/删除 | 冻结 Snapshot/Session schema | 故障注入、迁移/导出/恢复一致性 |
| Windows 适配 | 窗口句柄封装、置顶、托盘、电源/DPI事件 | Slint backend 选择 | 焦点、Explorer重启、多屏及VMConnect实测 |
| Slint UI | 视图、输入回调、Snapshot绑定、原型状态 | 设计批准；接口冻结 | 主流程、键盘、DPI与截图对照 |
| 集成与发布候选 | 生命周期、资源测量、包和完整验收记录 | 以上工作包 | 独立 review、实机测试、用户合并/发布审批 |

计时内核和存储先约定数据接口，Windows 适配和 UI 先约定命令/事件接口，再启动相互独立的实现 Worktree。接口变更经 Lead 判断后同步相关任务。

## 评审议程

本页是评审准备材料，会议尚未安排。用户在本阶段检查：MVP 范围、休眠/恢复语义、置顶与 Hyper-V 范围、存储/托盘/渲染方案、Slint 许可、资源目标、原型视觉和交互。确认记录包含批准范围、修订项和对应版本；PR 合并与发布另行审批。
