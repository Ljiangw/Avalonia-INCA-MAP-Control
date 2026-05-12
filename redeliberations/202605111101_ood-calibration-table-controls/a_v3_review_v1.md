# OOD 设计方案审查报告（v5）

## 审查结果

REJECTED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 设计中所有类型形态选择均与 C# 类型系统能力匹配：
- `ICalibrationData` 作为接口继承 `INotifyPropertyChanged`，C# 支持接口继承接口。
- `CalibrationEditorBase<T>` 作为泛型抽象类继承 `TemplatedControl`，C# 支持泛型类的继承与依赖属性注册。
- `ITableDataSource<TRow> where TRow : struct` 的泛型约束合法，`readonly struct` 行数据类型在 C# 中完全支持。
- `sealed` 具体控件（`CurveEditor`/`MapEditor`/`CubeEditor`）的密封声明合理。
- 接口拆分策略（`IChartPresenter` → `ILineChartPresenter`/`ISurfaceChartPresenter`）在 C# 单继承多接口实现的约束下合理。

**[轻微]** `CubeRowData` 的 x 列值描述为"通过动态属性或索引器按列顺序暴露"，其中"动态属性"在 `readonly struct` 语境下较模糊（`readonly struct` 无法继承 `DynamicObject`）。建议明确以索引器（如 `double this[int index] { get; }`）作为主要暴露方式，并在详细设计阶段确认 Avalonia `DataGridTextColumn` 绑定到值类型索引器的路径解析行为。

### 2. 标准库与生态覆盖

**[通过]** 设计方案中引用的标准库和 Avalonia API 均在覆盖范围内：
- `Dispatcher.UIThread.Post`、`ObservableCollection<T>`、`Task.Delay`、`INotifyPropertyChanged` 均为 .NET 标准库/API。
- `DataGrid` 的 `CellEditEnding`、`LoadingRow`、行虚拟化、`CanUserResizeColumns` 均为 Avalonia DataGrid 的标准能力。
- `DataGridRow.DataContextChanged` 事件（继承自 `StyledElement`）在 Avalonia 中存在，可用于虚拟化行回收时的状态清理。
- `string.Empty` 作为 `PropertyChanged` 属性名表示"所有属性均可能变更"，Avalonia 绑定系统支持此约定。

**[轻微]** 增量刷新方案建议将 `ITableDataSource<TRow>.Rows` 改为 `ObservableCollection<TRow>` 并通过 `Rows[index] = newRow` 实现单元素替换，但接口返回类型为 `IReadOnlyList<TRow>`，调用方无法在不进行类型检查的情况下触发单元素替换。建议在接口层面为增量刷新提供明确的契约支撑（如增加 `bool SupportsIncrementalRefresh { get; }` 或暴露 `INotifyCollectionChanged`）。

### 3. 语言特性可行性

**[通过]** 错误处理、并发、资源管理等策略与 C# 能力匹配：
- 手动事件拦截策略（`CellEditEnding` + `e.Cancel = true`）在 Avalonia DataGrid 中可行。
- `Task.Delay` 防抖 + `Dispatcher.UIThread.Post` 调度在 C# 异步模型下可行。
- 模块/包结构设计符合 NuGet 项目组织方式。
- `TemplatedControl` + `ControlTemplate` + `Style` 的组织方式符合 Avalonia 控件规范。
- MVVM 模式通过 `INotifyPropertyChanged` 和绑定系统得到支持。

**[轻微]** `ZSliceActivationTracker` 的防抖实现未提及时间抽象（如 `TimeProvider`），不利于单元测试中的时间控制。建议 Tracker 构造函数接受可选的 `TimeProvider` 参数（.NET 8+ 可用），默认使用 `TimeProvider.System`，测试中注入 `FakeTimeProvider` 以快速推进防抖延迟。

### 4. 设计一致性

**[一般]** **图表抽象层 `IChartPresenter.AttachTo(Control host)` 引入 Avalonia 依赖，与"无 UI 框架依赖"声明矛盾。** 模块职责与依赖方向表明确声明图表抽象层"自身无 UI 框架依赖（接收原始数据）"，但 `IChartPresenter.AttachTo(Control host)` 的参数 `Control` 是 Avalonia 核心类型，直接引入了 UI 框架依赖。此矛盾若不修正，实现者无法遵循文档声明的解耦目标，且将来若需将图表引擎替换为独立渲染（如离屏导出）时，`Control` 参数将成为障碍。

**[一般]** **`IAsyncSaveCapable` 纯标记接口无法支撑脏标记跟踪的完整功能。** 设计方案将 `IAsyncSaveCapable` 的最小形态定义为纯标记接口（无成员），并声明控件层"通过检测后端是否实现 `IAsyncSaveCapable` 标记接口来决定是否启用脏标记视觉反馈和状态栏未保存提示"。但纯标记接口只能被检测存在性，无法提供 `HasUnsavedChanges` 状态或 `SaveAsync` 方法。控件层检测到标记后，无法获取"未保存数量"（如 "3 处未保存"），也无法触发保存操作，导致变更反馈机制中关于脏标记的描述缺乏数据支撑。

**[轻微]** `ZSliceActivationTracker` 作为"核心抽象"仅定义了输出事件 `ActiveSliceChanged`，未定义 `CubeEditor` 如何向 Tracker 传递选中变更或 z 列点击的输入方法（如 `OnSelectionChanged(CubeRowData?)`、`OnZColumnClicked(CubeRowData)` 等）。虽然「关键行为契约」中描述了交互流程，但核心抽象层面的输入接口缺失导致 Tracker 的公共 API 契约不完整，不利于独立单元测试和实现对接。

**[通过]** 各模块间的依赖方向合理，无循环依赖。数据契约层为最底层，无 outward 依赖；适配层和图表抽象层仅依赖数据契约；激活追踪层仅依赖表格适配层；Views 层向下依赖所有下层模块。协作关系形成闭环，无缺失环节。

### 5. 设计质量

**[通过]** 职责划分整体遵循单一职责原则：
- `CalibrationEditorBase<T>` 封装公共 UI 结构和变量切换逻辑。
- `ZSliceActivationTracker` 独立管理 z 切片激活状态。
- `ITableDataSource<TRow>` 负责多维到二维的投影。
- `IChartPresenter` 隔离图表渲染实现。
- 三个具体控件（`CurveEditor`/`MapEditor`/`CubeEditor`）各自聚焦单一维度。

**[通过]** 抽象层次恰当，`readonly struct` vs `class` 降级路径的权衡分析完整，`CubeEditor` 平铺展示的性能保障策略合理。

**[轻微]** `ITableDataSource` 未暴露列生成所需的元数据（如 x 轴值列表、列数）。`MapEditor`/`CubeEditor` 在变量切换时动态生成 `DataGrid` 列需要直接访问 `IMapData`/`ICubeData` 接口，列生成逻辑分散在 Editor 和表格适配层之间。建议评估是否在 `ITableDataSource` 中补充列定义信息（如 `IReadOnlyList<string> ColumnHeaders { get; }`、`int DataColumnCount { get; }`），使表格适配层更完整地封装 DataGrid 的列生成需求。

## 修改要求（REJECTED 时存在）

### 问题1：图表抽象层 `IChartPresenter.AttachTo(Control host)` 与"无 UI 框架依赖"声明矛盾

- **问题**：`IChartPresenter.AttachTo(Control host)` 的参数 `Control` 是 Avalonia 核心类型，直接引入了 UI 框架依赖。但模块职责表声明图表抽象层"自身无 UI 框架依赖"。
- **原因**：文档声明与接口定义不一致，违背了解耦目标。若将来需要将图表引擎替换为离屏渲染或跨平台实现，`Control` 参数会成为架构障碍。
- **建议方向**：
  - 方案 A：将 `AttachTo` 参数改为 `object host`，由具体实现方在内部转换为 `Control`。保持接口层面无 Avalonia 依赖。
  - 方案 B：拆分为 `IChartPresenter`（平台无关，仅含 `Refresh()` 和 `RenderFailed`）和 `IAvaloniaChartPresenter : IChartPresenter`（含 `AttachTo(Control host)`）。图表抽象层保持无 UI 依赖，Avalonia 特定契约下沉到子接口。
  - 方案 C：在模块职责表中修正声明，明确图表抽象层"仅依赖 Avalonia 的 `Control` 宿主类型，不依赖具体图表库实现"。

### 问题2：`IAsyncSaveCapable` 纯标记接口无法支撑脏标记跟踪功能

- **问题**：纯标记接口（无成员）只能被检测存在性，无法提供 `HasUnsavedChanges` 状态或 `SaveAsync` 方法。控件层无法获取"未保存数量"，脏标记视觉反馈和状态栏提示缺乏数据支撑。
- **原因**：设计方案在变更反馈机制中明确提到"显示未保存数量"和"脏标记视觉反馈"，但纯标记接口无法提供实现这些功能所需的状态信息。
- **建议方向**：
  - 将 `IAsyncSaveCapable` 的最小形态扩展为至少包含 `bool HasUnsavedChanges { get; }`。若需要保存触发，再扩展 `Task SaveAsync()`。
  - 或者，若希望保持纯标记接口的简洁性，则需要在设计文档中明确：控件层自行维护编辑计数作为"未保存数量"的近似值，并说明此近似策略的局限性和适用场景。
