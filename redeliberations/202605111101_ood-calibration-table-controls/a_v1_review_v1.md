# OOD 设计方案审查报告（v3）

## 审查结果

REJECTED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 设计中所有类型形态选择均与 C# 类型系统及 Avalonia 控件体系兼容：
- `ICalibrationData` 继承 `INotifyPropertyChanged` 符合 C# 接口多继承规则；
- `CalibrationEditorBase<T>` 作为泛型抽象类继承 `TemplatedControl`，其子类 `CurveEditor` / `MapEditor` / `CubeEditor` 为具体密封类，泛型约束 `T : ICalibrationData` 合理，Avalonia XAML 可通过具体子类正常实例化和应用样式；
- `ICurveData` / `IMapData` / `ICubeData` 采用接口而非抽象类，避免了后端领域模型可能面临的单继承冲突；
- `ZSliceActivationTracker` 作为独立类，不依赖 Avalonia 视觉类型，纯 .NET 事件输出，类型隔离清晰；
- `readonly struct` 行数据对象（`CurveRowData` / `MapRowData` / `CubeRowData`）在 C# 中完全合法，坐标索引以内联属性传递，无泛型系统能力超限问题。

### 2. 标准库与生态覆盖

**[通过]** 经查阅 Avalonia 官方 API 文档确认，设计方案依赖的核心能力均在 Avalonia 生态覆盖范围内：
- `DataGrid` 控件（`Avalonia.Controls.DataGrid`）在 Avalonia 中提供 `CellEditEnding`（编辑即将结束前触发）与 `CellEditEnded`（编辑结束后触发）事件，设计文档中引用的 `CellEditEnding` 事件名称与 Avalonia API 一致；
- `DataGrid` 支持动态列生成（`DataGridColumn` 派生类运行时创建）与行虚拟化（`EnableRowVirtualization`），满足 `CubeEditor` 大数据量场景；
- `Dispatcher.UIThread.Post` / `Invoke` 为 Avalonia 标准线程调度 API；
- Avalonia V12 未移除 `TemplatedControl`、`DataGrid`、`INotifyPropertyChanged` 等核心类型，生态基础能力无缺失；
- 3D 曲面图无内置控件，通过 `IChartPresenter` 接口抽象隔离具体图表库，假设合理且符合 Avalonia 生态现状。

**[轻微]** `readonly struct` 作为 `DataGrid` 行数据源时，`DataGrid` 标准单元格编辑的双向绑定自动提交机制无法更新不可变结构体实例，编辑回写需完全依赖手动事件拦截（`CellEditEnding` / `CellEditEnded`），实现复杂度较高，需在详细设计阶段明确编辑模板与事件处理策略。

### 3. 语言特性可行性

**[通过]** 设计方案涉及的语言特性与 C# / Avalonia 能力完全匹配：
- 错误处理策略（就地验证反馈、空状态降级、图表渲染降级）均可通过 Avalonia `IDataErrorInfo` / `INotifyDataErrorInfo`、`DataGrid` 数据验证模板、异常捕获实现；
- UI 线程亲和性约束通过 `Dispatcher.UIThread` 调度满足；
- `ZSliceActivationTracker` 内部防抖可通过 `DispatcherTimer` 或 `Task.Delay` + `CancellationToken` 实现；
- 控件封装为独立 `TemplatedControl`，符合 NuGet 包中可复用 Avalonia 控件库的项目组织方式；
- 模块依赖方向（数据契约层 → 适配层 → Views 层）无反向依赖，符合 C# 项目引用规范。

### 4. 设计一致性

**[一般]** `CurveEditor` 的横向两行表格布局与 `ITableDataSource` 纵向行集合抽象之间的映射关系缺失。需求明确要求 `CurveEditor` "以两行形式展示：第一行为 x 轴（自变量）值，第二行为 z 轴（因变量）值"，这是一种横向转置的表格形态。但当前设计中 `CurveRowData` 及 `ITableDataSource` 抽象均面向纵向行集合（每行承载一个数据点及其坐标索引），未说明该抽象如何支撑 `CurveEditor` 的横向布局。若实现者直接套用 `DataGrid` + `CurveRowData`，将得到纵向多行列表（每行一个 x/z 数据点），与需求的两行横向展示严重不符；若 `CurveEditor` 采用其他布局方案（如自定义 `Grid`），则现有表格适配抽象不适用，存在需求与设计的衔接缺口，实现阶段易产生方向偏差。

**[轻微]** `CubeEditor` 的 z 切片级高亮状态如何与 `DataGrid` 虚拟化行视觉元素联动，方案缺少状态传递机制说明。`DataGrid` 仅对可见行创建视觉树实例，`ZSliceActivationTracker` 输出的激活状态需通过 `LoadingRow` 事件、样式绑定或 `DataContext` 属性传递到行视觉元素，建议补充该联动策略。

**[通过]** 其余设计一致性良好：
- 变量切换流程形成完整闭环（`ComboBox` 选中 → 依赖属性更新 → 虚方法通知 → 子类重建数据源/刷新图表）；
- 单元格编辑与数据回写链路清晰（`CellEditEnding` → 行数据对象提取坐标 → 调用 `ICubeData` 写接口 → `INotifyPropertyChanged` 刷新 UI）；
- z 切片激活与 3D 视图联动逻辑完整；
- 模块间依赖无循环。

### 5. 设计质量

**[通过]** 整体设计质量良好：
- 职责划分遵循单一职责原则：`CalibrationEditorBase<T>` 聚焦公共 UI 结构与变量切换，`ZSliceActivationTracker` 独立管理 z 切片激活状态，`ITableDataSource` 负责数据投影，`IChartPresenter` 隔离图表渲染；
- 抽象层次恰当：接口定义契约、抽象类封装公共 UI 逻辑、密封类作为终态控件，无过度设计亦无明显设计不足；
- 可测试性：`ZSliceActivationTracker` 无 Avalonia 视觉依赖，可脱离 UI 线程单元测试；`IChartPresenter`、`ICalibrationData` 等接口便于 mock；
- `CubeEditor` 从 v2 到 v3 的变更集中在数据组织方式，架构其余模块稳定复用，演进合理。

**[轻微]** 输入验证的抽象未在数据契约层定义。需求要求"输入的数据应能正确解析为数值类型"，但 `ICalibrationData` 及其子接口未声明数值解析、精度、取值范围验证契约，验证逻辑可能散落在 Editor 代码中。建议后续详细设计阶段在数据契约层或表格适配层补充验证接口（如 `ICellValidator` 或 `IDataErrorInfo` 支持）。

## 修改要求（REJECTED 时存在）

### 问题 1：`CurveEditor` 横向两行表格布局与现有纵向行集合抽象不匹配

**问题**：需求要求 `CurveEditor` 以两行横向展示（第一行为 x 轴值，第二行为 z 轴值），但当前 `ITableDataSource` 及 `CurveRowData` 抽象面向纵向行集合（每行对应一个数据点），未提供横向布局的实现策略。直接套用 `DataGrid` + `CurveRowData` 会得到纵向多行列表，与需求冲突。

**原因**：`CurveEditor` 的表格形态与 `MapEditor`/`CubeEditor` 的纵向 `DataGrid` 行模型本质不同。若架构设计不明确横向表格的适配方式，实现阶段要么产出错误布局，要么被迫绕过现有抽象自行实现，破坏设计一致性。

**建议方向**：
1. **方案 A（保留 DataGrid）**：为 `CurveEditor` 单独定义横向表格适配抽象。例如，将数据点集合投影为两个行对象（`XRow` 与 `ZRow`），每行对象暴露动态列值集合（如 `IReadOnlyList<double> Values`），`CurveEditor` 据此动态生成 `DataGrid` 列，使 `DataGrid` 纵向行对应横向语义。此时需扩展 `ITableDataSource` 或定义 `ICurveTableAdapter` 特化接口。
2. **方案 B（自定义布局）**：明确 `CurveEditor` 不使用 `DataGrid`，而采用自定义 `Grid` + `ItemsControl` 横向布局。此时需在架构中移除 `CurveEditor` 对 `ITableDataSource`/`CurveRowData` 的依赖，为其定义独立的视觉构建契约（如 `ICurveLayoutBuilder`），避免纵向行集合抽象被误用。

建议在设计文档中显式选择并说明 `CurveEditor` 的表格实现策略，确保与需求一致。
