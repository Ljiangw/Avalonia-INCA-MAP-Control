# OOD 设计方案审查报告（v1）

## 审查结果

APPROVED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 设计中所有类型形态选择在 C# 类型系统内完全可行：
- `ICalibrationData` 继承 `INotifyPropertyChanged`、`ICurveData`/`IMapData`/`ICubeData` 继承 `ICalibrationData` 符合接口多继承规则
- `CalibrationEditorBase<T>` 作为泛型抽象类约束 `T : ICalibrationData` 并继承 `TemplatedControl` 是 Avalonia 中合法的泛型控件模式
- `ITableDataSource<TRow> where TRow : struct` 泛型约束合法，`readonly struct` 行数据类型可行
- `IChartPresenter` → `IAvaloniaChartPresenter` → `ILineChartPresenter`/`ISurfaceChartPresenter` 的接口继承层次清晰，无多继承冲突
- `ZSliceActivationTracker` 作为独立非静态类，通过纯 .NET 事件与 `CubeEditor` 交互，类型耦合方向合理
- `CurveTableRow` 枚举、`ActiveZSliceChangedEventArgs`/`ColumnWidthChangedEventArgs` 等事件参数类型均符合 C# 规范

**[轻微]** `Array.AsReadOnly` 仅能作用于一维数组（`T[]`），不适用于 `double[,]`。虽然设计同时提供了"自定义只读包装器"和"深拷贝"作为替代方案，整体可行性不受影响，但建议在文档中修正该示例，避免对后端实现者产生误导。

### 2. 标准库与生态覆盖

**[通过]** 设计中依赖的能力均在 C# 标准库和 Avalonia V12 覆盖范围内：
- `INotifyPropertyChanged`、`INotifyCollectionChanged`、`IReadOnlyList<T>`、`TimeProvider`（.NET 8+）、`Task`、`EventHandler<T>` 均为标准库类型
- `Dispatcher.UIThread.Post`、`TemplatedControl`、`DataGrid`（含 `LoadingRow`/`UnloadingRow`/`CellEditEnding` 事件、`DataGridCellEditEndingEventArgs.Cancel`）、`ComboBox` 均为 Avalonia 标准控件/API，已通过网络资源验证 `DataGridCellEditEndingEventArgs` 确实继承自 `CancelEventArgs` 并提供 `Cancel` 属性
- `FakeTimeProvider` 可通过 `Microsoft.Extensions.TimeProvider.Testing` NuGet 包获得，满足测试注入需求
- `double.TryParse` 配合 `NumberStyles.Float` 和 `CultureInfo.InvariantCulture` 为标准解析模式
- `DataGrid` 动态列绑定 `Values[0]` 等索引器路径在 Avalonia 绑定引擎中受支持
- MVVM 支持通过 `INotifyPropertyChanged` 接口继承和 Avalonia 绑定机制实现，符合 Avalonia 规范

### 3. 语言特性可行性

**[通过]** 所有语言层面设计决策与 C# / Avalonia 能力匹配：
- 错误处理策略（`TryParse` 就地验证 + 异常传播 + `RenderFailed` 事件降级）与 C# 错误处理模式完全匹配
- 并发设计（UI 线程亲和性通过 `Dispatcher.UIThread.Post` 保障、图表渲染后台准备 + UI 提交）符合 Avalonia 单线程渲染模型
- `ZSliceActivationTracker` 的防抖策略基于 `TimeProvider` 定时器，定时器回调运行于线程池，`CubeEditor` 事件处理器中调度回 UI 线程，线程模型正确
- 控件架构基于 `TemplatedControl` + `ControlTemplate` + `PART_` 模板部件约定，完全符合 Avalonia 控件组织方式
- `AvaloniaProperty.Register` 用于泛型控件基类的依赖属性注册在 C# / Avalonia 中可行

**[轻微]** `ZSliceActivationTracker` 声明 `TimeProvider TimeProvider { get; }` 为只读属性，且文档声称"测试中可注入 `FakeTimeProvider`"，但在"关键成员"和构造函数描述中未明确 `TimeProvider` 的构造函数注入参数。建议补充构造函数签名（如 `ZSliceActivationTracker(TimeSpan debounceDelay = default, TimeProvider? timeProvider = null)`），使声称的测试注入能力在设计层面有明确的入口。

### 4. 设计一致性

**[通过]** 设计方案内部一致，各模块协作形成闭环：
- 依赖方向严格遵循：数据契约层 → 适配层/图表抽象层 → Views 层，无循环依赖
- `requirement.md` v3 的全部功能需求（3 个独立控件、顶部信息栏、变量选择器、折线图/3D 曲面图、单元格编辑、选中高亮、数据绑定、精度展示、列宽/行高调整、变更反馈）均在设计中有对应抽象或行为契约覆盖
- 迭代需求中全部 14 项问题在 v7 中均有明确对应修改：z 列折中声明、`ICurveTablePresenter` 列宽契约、`CalibrationEditorBase<T>` 模板部件定义、数组可变性语义、编辑模式方向键行为、空状态默认视觉、数值解析策略、`SaveAsync` 异常语义、Tab 跨边界行为、`SetAxisLabels`、版本号统一、`OldZValue`/`NewZValue`、`debounceDelay` 参数化、`G6` 后备格式
- 变量切换流程覆盖所有边界场景（null/空列表/维度为零/正常数据），形成完整的状态机
- 单元格编辑回写链路（`CellEditEnding` → 手动解析 → 模型写接口 → `PropertyChanged` → 刷新）在三个控件中逻辑一致
- 四种高亮（单元格级/行级/z 切片级/行头文字）的驱动来源完全独立，优先级明确，无冲突
- z 切片激活链路（选中变更 → Tracker 防抖 → 事件通知 → 图表刷新 + 高亮更新）形成完整闭环
- `LoadingRow`/`UnloadingRow`/`DataContextChanged` 三重机制覆盖虚拟化下的 z 切片高亮状态同步，策略完备

### 5. 设计质量

**[通过]** 设计质量良好，满足架构级要求：
- **单一职责原则**：`ZSliceActivationTracker` 仅负责 z 切片激活状态管理；`ITableDataSource<TRow>` 仅负责数据投影；`IChartPresenter` 仅负责图表渲染；三个 Editor 控件仅负责组合与交互路由，职责边界清晰
- **抽象层次恰当**：架构级设计聚焦于模块划分、核心接口契约和关键行为流程，未过度下沉到字段级实现细节（如具体颜色值 `#808080` 仅作为示例而非强制），也未遗漏影响架构决策的关键信息（如 `readonly struct` 与 DataGrid 编辑机制的冲突分析及应对策略）
- **可测试性**：`ZSliceActivationTracker` 不依赖 Avalonia 视觉类型，可通过注入 `FakeTimeProvider` 进行无头单元测试；`ITableDataSource<TRow>`、`IChartPresenter`、`ICurveTablePresenter` 均为接口，支持 Mock 替换；`IAsyncSaveCapable` 的异常语义和事件触发条件定义明确，便于测试验证
- **可扩展性**：图表渲染通过 `IChartPresenter` 接口族抽象，支持 OxyPlot/LiveCharts/ScottPlot/自定义 Skia 等多种实现；`ITableDataSource<TRow>.SupportsIncrementalRefresh` 为性能优化保留可选路径；z 列展示方案明确标注为技术折中并预留未来升级路径
- **设计决策文档化充分**：15 项设计决策均给出明确决策内容、理由和权衡分析（如抽象基类 vs 纯组合、`readonly struct` vs `class`、编辑触发方式、轴信息属性下沉等），为后续详细设计和实现提供清晰依据

## 修改要求（REJECTED 时存在）

无。本方案无严重或一般级别问题，审查通过。
