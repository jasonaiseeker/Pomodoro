# M0-UI：UI v2评审收尾

目标：保留四项UI要求，补齐可编辑设计和独立证据，交用户确认。

## 输入

- [产品v0.2](../../product/requirements-v0.2.md)，只读UI相关内容。
- [SVG](../../design/ui-v2/pomodoro-ui-v2.svg)、[PNG](../../design/ui-v2/pomodoro-ui-v2-preview.png)、[说明](../../design/ui-v2/ui-v2-review.md)。
- [验收](../../acceptance.md)AC-43—46与[未决UI项](../../quality/review-register.md)。
- 执行任务创建时填写统一基线commit；Figma文件4U9TTroE0k4Rkef4K9LIG7归当前设计任务唯一写入。

## 范围

仅写docs/design/ui-v2和本任务交付记录；外部写入限Figma独立UI v2页面。产品变化经Issue/Lead记录，不直接改公共契约。当前禁止业务源码、Cargo工程、实现型试验、付费升级和共享变更。

## 验收与停止

1. 圆圈与数字同源，全状态明确，idle无目标/已选目标分开。
2. 标签水平垂直居中，整按钮热区、键盘/焦点/禁用态有证据。
3. 浅深覆盖主窗/设置/确认/错误/恢复，Light默认及失败回退明确。
4. 展开收起同宽，四固定屏/DPI与混合屏标注物理/逻辑单位，保持10%。
5. 实际渲染和独立检查；原生Figma同步与用户批准分开。通道受限交具体错误和可编辑本地结果。
6. 返回commit、真实验证和未决项后结束本轮，Lead安排评审。
