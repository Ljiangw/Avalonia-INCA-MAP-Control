# OOD 设计方案审查报告（v2）

## 审查结果

APPROVED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 类型形态选择合理：`ICalibrationData` 及维度特化子接口采用接口形态，避免了后端领域模型可能存在的单继承冲突；`CalibrationEditorBase<T>` 选用泛型抽象类承载公共依赖属性与模板约定，充分利用了 C# 单继承+多接口实现的类型系统能力；三个具体控件标记为 `sealed`，符合领域终态产品的定位。`readonly struct` 行数据类型（`CurveRowData`、`MapRowData`、`CubeRowData`）与 C# 值类型系统完全兼容，坐标索引以内联属性方式承载，避免了接口引用导致的装箱开销。`ZSliceActivationTracker` 作为独立类抽取，类型职责清晰。

**[轻微]** `CalibrationEditorBase<T>` 作为泛型 `TemplatedControl`，其公共 `ControlTemplate` 在 Avalonia 样式系统中的匹配规则需在详细设计阶段关注。建议为非泛型中间层基类（如 `CalibrationEditorBase : TemplatedControl`）承载样式约定，或确保子类 `CurveEditor`/`MapEditor`/`CubeEditor` 的样式选择器能正确覆盖基类模板部件。

### 2. 标准库与生态覆盖

**[通过]** 核心依赖均在 C# 标准库和 Avalonia 生态覆盖范围内：`INotifyPropertyChanged`（`System.ComponentModel`）、`Dispatcher.UIThread`（Avalonia 线程调度）、`AvaloniaProperty.Register`（依赖属性系统）、`DataGrid`（`Avalonia.Controls.DataGrid` 包）。`IChartPresenter` 接口抽象合理规避了 Avalonia V12 缺乏内置 3D 曲面图控件的问题，实现方可选用 OxyPlot、LiveCharts、ScottPlot 或自定义 Skia 渲染。设计明确识别了 Avalonia `DataGrid` 不支持跨行合并单元格的局限，并给出了兼容虚拟化的替代视觉方案，假设成立且务实。

**[轻微]** `MapEditor` 和 `CubeEditor` 的表格适配层需将多维矩阵投影为二维行集合，且 x 轴列数为动态（不同标定变量的 x 维度长度不同）。`DataGrid` 动态列生成（`DataGrid.Columns.Add` + 索引器绑定）在 Avalonia 中支持，但需在详细设计阶段明确 `MapRowData`/`CubeRowData` 中 x 列值的承载形式（如 `double[] Values` 配合 `{Binding Values[index]}` 动态绑定），建议架构设计在此处补充一句方向性说明。

### 3. 语言特性可行性

**[通过]** 错误处理策略与 C# 能力匹配：输入验证采用就地反馈（红色边框/工具提示）而非异常抛出，符合桌面交互场景；结构性错误（空数据）进入空状态展示，避免未处理异常；图表渲染错误通过 `IChartPresenter` 实现方内部降级，策略完整。并发设计边界清晰：UI 线程亲和性通过 `Dispatcher.UIThread.Post` 保障，`ZSliceActivationTracker` 的防抖策略（100-200ms）在纯 .NET 事件中可基于 `System.Timers.Timer` 或 `CancellationTokenSource` 实现，均在标准库能力范围内。模块组织方式符合 NuGet 包和 Avalonia 控件库的项目结构惯例。

**[轻微]** `ICalibrationData` 强制继承 `INotifyPropertyChanged` 确保了数据变更通知能力，但需注意：若后端数据模型由非 UI 线程（如 ECU 通信线程）更新，实现方必须在触发 `PropertyChanged` 前完成线程调度。建议在「关键行为契约」或「并发设计」章节中补充一句对后端实现方的线程安全约定，避免实现阶段因事件跨线程触发导致绑定异常。

### 4. 设计一致性

**[通过]** 模块间依赖方向合理：数据契约层（`ICalibrationData` 及子接口）为最底层，无 outward 依赖；表格适配层和图表抽象层仅依赖数据契约；Views 层向下依赖所有下层模块，无循环依赖。核心行为契约形成完整闭环：变量切换流程（ComboBox → 依赖属性变更 → 虚方法通知 → 子类重建数据源）链路清晰；单元格编辑回写流程（编辑确认 → 行数据提取坐标索引 → 原始数据模型更新 → `INotifyPropertyChanged` → UI 刷新）闭环完整；z 切片激活联动（表格选中 → `ZSliceActivationTracker` → 纯 .NET 事件 → `CubeEditor` 调度 UI 线程 → 3D 图表刷新 + 视觉状态更新）各环节衔接无缺失。高亮叠加策略的三级优先级定义明确，通过 Avalonia `Classes` 动态附加机制实现，技术路径可行。

**[轻微]** 模块划分图中 `ZSliceActivationTracker` 的层级位置与文字说明的"依赖表格适配层"略有歧义（图中与 `ICubeData` 近似同级）。建议将 Tracker 明确置于适配层或 Views 层的子区域，与文字描述保持一致。

### 5. 设计质量

**[通过]** 职责划分遵循单一职责原则：`CalibrationEditorBase<T>` 专注公共 UI 结构和变量切换管线；各 `Editor` 专注维度特化的布局与交互；`ITableDataSource` 专注多维→二维投影；`IChartPresenter` 专注渲染隔离；`ZSliceActivationTracker` 专注 z 切片状态管理。抽象层次恰当：既未过度设计（如未引入不必要的中间抽象层），也未设计不足（关键的可替换性通过接口保障）。可测试性良好：`ZSliceActivationTracker` 作为纯逻辑组件，仅消费 `CubeRowData` 并通过纯 .NET 事件输出，可在无头环境中独立单元测试；`IChartPresenter` 接口允许图表实现 mock 化；表格适配层与 Views 层解耦，便于隔离测试数据投影逻辑。

**[轻微]** `CubeEditor` 同时承担三重职责（数据展示、单元格编辑、z 切片选择器），虽然 `ZSliceActivationTracker` 已抽离部分逻辑，但 `CubeEditor` 本身仍可能趋于复杂。建议持续关注该类的行数/职责规模，若实现阶段膨胀，可考虑进一步将表格交互路由（选中/编辑/导航事件处理）抽取为内部策略类，但当前架构设计阶段无需调整。

## 修改要求（REJECTED 时存在）

无。本审查无严重或一般级别问题，设计在 C# 语言和 Avalonia 框架层面具有可行性。
