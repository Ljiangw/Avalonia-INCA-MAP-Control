# OOD 设计方案审查报告（v4）

## 审查结果

APPROVED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 设计中所有类型形态选择均与 C# 类型系统能力匹配。
- `ICalibrationData` 继承 `INotifyPropertyChanged`、`ICurveData`/`IMapData`/`ICubeData` 继承 `ICalibrationData`：接口多继承合法。
- `CalibrationEditorBase<T>` 为泛型抽象类，`T : ICalibrationData`，继承 `TemplatedControl`：Avalonia 支持泛型控件基类。
- `CurveEditor`/`MapEditor`/`CubeEditor` 为密封类，分别继承 `CalibrationEditorBase<ICurveData>`/`CalibrationEditorBase<IMapData>`/`CalibrationEditorBase<ICubeData>`：C# 泛型实例化后的具体基类可被继承，密封修饰合理。
- `ITableDataSource<TRow> where TRow : struct` 及 `readonly struct` 行数据：值类型约束和只读结构体均为 C# 合法类型形态。
- `ILineChartPresenter`/`ISurfaceChartPresenter` 继承 `IChartPresenter`：接口继承合法。
- `ZSliceActivationTracker` 为独立类，不依赖 Avalonia 视觉类型：纯 .NET 类，类型形态合理。

**[轻微]** `MapRowData`/`CubeRowData` 的 x 列值描述为"通过动态属性或索引器按列顺序暴露"。由于行数据类型被约束为 `readonly struct`，运行时动态属性方案（如继承 `DynamicObject`）在 C# 中对结构体不可行，实际上只能采用索引器方案。建议在设计文档中明确排除动态属性方向，仅保留索引器方案，避免实现阶段走弯路。

### 2. 标准库与生态覆盖

**[通过]** 设计所需能力均在 Avalonia 11.x/V12 生态覆盖范围内。
- `DataGrid.CellEditEnding` 事件及其参数 `DataGridCellEditEndingEventArgs.Cancel`：Avalonia API 确认存在（继承自 `CancelEventArgs`），支持取消默认提交。
- `DataGrid.LoadingRow` 事件：Avalonia API 确认存在（`EventHandler<DataGridRowEventArgs>`）。
- `DataGrid.CanUserResizeColumns`：Avalonia API 确认存在。
- `DataGridRow.DataContextChanged` 事件：继承自 `StyledElement`，Avalonia API 确认存在。
- `Dispatcher.UIThread.Post`：Avalonia 标准调度 API。
- `DataGrid` 样式选择器支持 `DataGridRow` 的 `MinHeight`/`MaxHeight`：Avalonia `DataGridRow` 继承自 `TemplatedControl` → `Control`，具备 `MinHeight`/`MaxHeight` 依赖属性，样式系统完全支持。
- `IReadOnlyList<T>`、`INotifyPropertyChanged`、泛型事件 `EventHandler<T>`：均为 .NET 标准库能力。

**[轻微]** 设计文档 v4 修订说明称"Avalonia `DataGrid` 不存在 `UnloadingRow` 事件"，并以此为由删除了对该事件的依赖。经查 Avalonia 11.3.12 API 文档，`DataGrid` 明确包含 `UnloadingRow` 事件（`Occurs when a DataGridRow object becomes available for reuse`）。备选方案（`LoadingRow` 先清理再判断 + `DataContextChanged` 后备清理）同样可行，不影响整体设计，但设计决策基于了错误的前提。建议实现阶段评估是否直接使用 `UnloadingRow` 进行样式清理，以简化虚拟化状态同步逻辑。

### 3. 语言特性可行性

**[通过]** 所有策略与 C# 及 Avalonia 的语言特性兼容。
- 错误处理：就地验证反馈（手动设置 `TextBox` 边框/工具提示）、空状态展示、`IChartPresenter.RenderFailed` 降级通知，均可在 C#/Avalonia 中实现。
- 并发：`Dispatcher.UIThread.Post`/`Invoke` 确保 UI 线程亲和性；`ZSliceActivationTracker` 的 `Task.Delay` 防抖 + 调度回 UI 线程模式符合 Avalonia 的并发模型。
- 资源管理：`IChartPresenter.Detach()` 提供显式解绑点，符合 C# 资源管理惯例。
- 模块结构：按 Views / 数据契约 / 表格适配 / 图表抽象 / 激活追踪分层，符合 NuGet 多项目组织的惯例。
- Avalonia 控件：`TemplatedControl` 基类、`AvaloniaProperty.Register`、ControlTemplate 部件约定（`PART_VariableSelector` 等）、样式类动态附加（`Classes`），均符合 Avalonia V12 控件开发规范。
- MVVM 支持：`ICalibrationData` 继承 `INotifyPropertyChanged`，控件通过绑定消费数据模型，适配层承担视图模型职责，整体支持 MVVM 模式。

**[轻微]** 设计文档将 `DataGrid.CurrentItem` 列为行级高亮的驱动源之一（"由 DataGrid 的 `CurrentItem` / 当前活动单元格所在行驱动"）。经查 Avalonia 11.3.12 API 文档，`DataGrid.CurrentItem` 是 `protected` 属性，外部代码和样式选择器无法直接访问。虽然文档同时提供了"DataGrid 内置选中行样式"的备选方案（基于 `SelectedItem`，该属性为 public），但建议将首选驱动源明确为 `SelectedItem`/`SelectionChanged`，避免实现阶段对 `CurrentItem` 的不可访问性产生困惑。

### 4. 设计一致性

**[通过]** 各抽象职责清晰，协作关系形成闭环，模块间依赖方向合理。
- 职责划分：Views 层（控件模板与交互路由）→ 表格适配层（多维→二维投影）→ 数据契约层（领域接口）→ 图表抽象层（渲染隔离），职责边界清晰。
- 依赖方向：数据契约层无 outward 依赖；适配层仅依赖数据契约；激活追踪层仅依赖表格适配层；Views 层可依赖所有下层，无反向依赖。无循环依赖。
- 协作闭环：
  - 变量切换：`ComboBox` → `CalibrationEditorBase<T>` → 空状态或子类刷新 → 表格重建/图表加载，闭环完整。
  - 单元格编辑：双击编辑 → `CellEditEnding` 拦截 / `ICurveTablePresenter.CellEditCompleted` → 模型写接口 → `PropertyChanged` → `Refresh()` → `ItemsSource` 重置，闭环完整。
  - z 切片激活：选中变更 → `ZSliceActivationTracker` → 激活事件 → `CubeEditor` 调度 UI → 3D 图表刷新 + z 切片高亮更新，闭环完整。
- 行为契约：关键行为契约（变量切换流程、单元格编辑与数据回写、z 切片激活与 3D 视图联动、高亮叠加策略、列宽/行高调整、精度展示）均有明确的触发条件和执行路径描述，足以指导后续实现。

### 5. 设计质量

**[通过]** 职责划分遵循单一职责原则，抽象层次恰当，便于后续实现和测试。
- 单一职责：`ZSliceActivationTracker` 独立封装 z 切片状态管理；`ITableDataSource<TRow>` 独立封装多维→二维投影；`IChartPresenter` 独立隔离图表库。各抽象职责单一。
- 抽象层次：架构级设计聚焦模块划分、接口契约和协作关系，未过度侵入实现细节（如具体字段类型、XAML 结构），层次恰当。
- 可测试性：`ZSliceActivationTracker` 不依赖 Avalonia 视觉类型，仅消费 `CubeRowData` 和输出纯 .NET 事件，可在无头环境中单元测试。`ITableDataSource<TRow>` 和 `IChartPresenter` 均为接口契约，便于 mock。`CalibrationEditorBase<T>` 提供受保护的虚方法钩子（`FormatInfoText`、`OnEnterEmptyState`、`OnExitEmptyState`），便于子类行为测试。
- 可替换性：`IChartPresenter` 拆分为 `ILineChartPresenter`/`ISurfaceChartPresenter`，允许不同图表库实现替换；`ICurveTablePresenter` 允许横向布局的多种实现方案。

**[轻微]** `ITableDataSource<TRow>.Refresh()` 采用全量重新生成 `Rows` 集合并重新设置 `DataGrid.ItemsSource` 的策略。对于 CubeEditor 的大数据量场景（z 数 × y 数可能达数千行），全量刷新在频繁外部数据更新（如 ECU 实时推送）时可能引发 UI 闪烁和性能压力。此为架构层面的简化选择，实现阶段可通过引入 `ObservableCollection` 增量更新或局部刷新机制优化。设计文档已提及实现阶段可评估折叠/展开机制，建议同样在架构层面留注 `Refresh()` 的潜在性能优化方向。

## 修改要求

无严重或一般问题，不涉及 REJECTED 修改要求。上述 5 项轻微问题均为澄清或改进建议，不阻塞设计通过，可在后续详细设计或实现阶段酌情处理。
