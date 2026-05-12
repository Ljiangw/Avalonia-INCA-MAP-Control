# Avalonia 标定数据表格编辑控件 — 架构级 OOD 设计方案 v5

## 概述

本方案设计一组基于 Avalonia V12 的标定数据表格编辑控件，面向汽车 ECU 标定工具场景。在统一交互范式下支撑 1 维（曲线）、2 维（Map）、3 维（数据立方体）三种标定数据的可视化编辑，并将控件与后端数据模型、图表渲染引擎解耦，确保可复用性和可替换性。

相对于 v4 方案，v5 的核心变化集中在以下方面：

1. **修正 `readonly struct` 与 DataGrid 编辑机制不兼容的根因分析**：明确值类型的 boxing 副本问题是根本原因，`readonly` 只是额外禁止了 setter 调用；补充 `class` 降级方案的 GC 影响与坐标索引内联存储损失。
2. **完善编辑后刷新策略**：补充选中状态保持步骤；评估增量刷新可行性（`INotifyCollectionChanged` 单元素替换）；明确大数据量下的性能兜底方案。
3. **补全数据模型元素级变更通知约定**：明确元素级写方法触发 `PropertyChanged` 时使用 `string.Empty` 作为属性名，表示所有属性均可能变更。
4. **统一图表抽象层参数风格**：`ISurfaceChartPresenter` 的两个加载方法均接收原始数组，消除 `IMapData` 与原始数组混用的不对称性。
5. **补充 `ZSliceActivationTracker` 核心事件签名与输入方法**：定义 `ActiveSliceChanged` 事件、`ActiveZSliceChangedEventArgs` 参数类型，以及 `OnSelectionChanged`/`OnZColumnClicked` 输入方法。
6. **补充 `ICurveTablePresenter` 程序化选中能力**：新增 `SelectCell`/`ClearSelection` 方法及 `CurveTableRow` 枚举，支撑变量切换后重置选中和图表联动反向选中。
7. **扩展 `IAsyncSaveCapable` 为含状态成员的契约接口**：从纯标记接口扩展为包含 `HasUnsavedChanges` 和 `SaveAsync` 的完整契约，使脏标记跟踪具备数据支撑。
8. **修正图表抽象层 UI 依赖声明**：将 `IChartPresenter` 拆分为平台无关的基接口和 `IAvaloniaChartPresenter` 子接口，消除"无 UI 框架依赖"声明与 `AttachTo(Control host)` 之间的矛盾。
9. **补充高亮叠加策略的第四层视觉反馈**：当前行选中时 y 列文字加粗/变色、当前 z 切片激活时 z 列文字加粗/变色。
10. **明确 CubeEditor 默认 z 切片语义、列索引语义、3D 视图切换动画决策、键盘跨切片视觉不一致处理策略**。

## 模块划分

```
┌─────────────────────────────────────────────────────────────┐
│                     Views 层（控件定义）                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ CurveEditor │  │  MapEditor  │  │     CubeEditor      │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
│         │                └────────────────────┘             │
│         │                       ▲                           │
│         │    CalibrationEditorBase<T>  (抽象基类)            │
│         └───────────────────────┘                           │
│         ▲                                                   │
│  ICurveTablePresenter                                       │
│  (CurveEditor 横向表格契约)                                  │
└─────────────────────────────────────────────────────────────┘
                         │
┌────────────────────────┼────────────────────────────────────┐
│              ViewModel / 适配层（数据契约与转换）               │
│  ┌─────────────────────┼───────────────────────────────┐    │
│  │ ITableDataSource<TRow> │  IChartPresenter          │    │
│  │ (表格适配接口)        │  (图表渲染抽象)              │    │
│  │ [Map/Cube 专用]      │                               │    │
│  └─────────────────────┘                               │    │
│  ┌─────────┐ ┌────────┐ ┌─────────────┐                │    │
│  │ICurveData│ │IMapData│ │  ICubeData  │                │    │
│  └─────────┘ └────────┘ └─────────────┘                │    │
│         ▲              ▲              ▲                │    │
│         └──────────────┴──────────────┘                │    │
│              ICalibrationData (公共数据契约)            │    │
│              IAsyncSaveCapable (保存能力契约)            │    │
└────────────────────────────────────────────────────────┘    │
                         ▲                                    │
┌────────────────────────┘────────────────────────────────────┐
│              激活追踪层（CubeEditor 专用）                      │
│              ZSliceActivationTracker                        │
│              · 依赖表格适配层的 CubeRowData                  │
│              · 被 CubeEditor 使用                            │
└─────────────────────────────────────────────────────────────┘
```

### 模块职责与依赖方向

| 模块 | 职责 | 依赖方向 |
|------|------|---------|
| **Views 层** | 定义 Avalonia 控件模板、视觉结构、交互路由。包含三个具体控件及其公共抽象基类。 | 向下依赖适配层与数据契约 |
| **数据契约层** | 定义标定数据的领域接口（`ICalibrationData` 及维度特化子接口），不依赖任何 UI 框架类型。 | 被 Views 层和适配层依赖，自身无 UI 依赖 |
| **表格适配层** | 将多维数据投影为二维行集合，封装单元格坐标到原始数据的反向映射，支撑编辑回写。`ITableDataSource<TRow>` 及 `MapRowData`/`CubeRowData` 供 `MapEditor` 与 `CubeEditor` 使用；`CurveEditor` 不经过此抽象，而由 `ICurveTablePresenter` 直接处理横向布局。 | 依赖数据契约层 |
| **图表抽象层** | 定义折线图和 3D 曲面图的渲染契约，隔离具体图表库实现。基接口 `IChartPresenter` 自身无 UI 框架依赖；Avalonia 宿主关联能力下沉到 `IAvaloniaChartPresenter` 子接口。 | 被 Views 层依赖 |
| **激活追踪层** | CubeEditor 中 z 切片激活状态的专门管理者，将表格行/单元格选中事件映射为 z 值切换。 | 依赖表格适配层的 `CubeRowData`，被 CubeEditor 使用 |
| **Curve 表格契约层** | `CurveEditor` 专用的横向表格视图契约（`ICurveTablePresenter`），将一维 x/z 数据映射为两行横向表格，处理选中、编辑与高亮。 | 依赖数据契约层，被 CurveEditor 使用 |

依赖规则：数据契约层为最底层，无 outward 依赖；适配层和图表抽象层仅依赖数据契约；激活追踪层仅依赖表格适配层；Curve 表格契约层仅依赖数据契约层；Views 层可依赖所有下层模块，但下层模块不可反向依赖 Views 层。

## 核心抽象

### `ICalibrationData`（接口）

**角色**：所有维度标定数据的公共契约。

**职责**：暴露标定变量的元信息（名称、单位、各轴变量名及单位）以及数据变更通知能力。

**关键成员**：
- `string VariableName { get; }` — 变量名称，用于下拉框选项的内部标识
- `string GroupName { get; }` — 变量所属组名（如「曲线变量组」），用于下拉框显示格式的分组前缀
- `string DisplayName { get; }` — 格式化后的显示文本（如 `<曲线变量组> AllMkn_n_AirMax`），直接作为下拉框选项的展示内容；若后端不提供，控件层可按 `[{GroupName}] {VariableName}` 自动合成
- `string Unit { get; }` — 变量单位（如 `mgpl`）
- `string XAxisName { get; }`、`string XAxisUnit { get; }` — x 轴变量名及单位
- `string YAxisName { get; }`、`string YAxisUnit { get; }` — y 轴变量名及单位（2D/3D 数据适用）
- `string ZAxisName { get; }`、`string ZAxisUnit { get; }` — z 轴变量名及单位（3D 数据适用）
- `int DisplayPrecision { get; }` — 默认显示精度（小数位数），作为 `FormatString` 的后备策略
- `string FormatString { get; }` — 数值格式字符串（如 `F3`、`G6`），优先于 `DisplayPrecision` 使用；若两者同时存在，`FormatString` 优先，`DisplayPrecision` 仅在 `FormatString` 为空时生效
- 继承 `INotifyPropertyChanged` — 强制后端实现方具备数据变更通知能力

**元素级变更通知约定**：
`ICalibrationData` 的子接口（`ICurveData`、`IMapData`、`ICubeData`）声明的元素级写方法（`SetXValue`、`SetZValue`、`SetValue` 等）在成功修改数据后应触发 `PropertyChanged` 事件。考虑到单次写操作可能影响的数据范围不确定（如修改 x 轴值可能同时影响图表和表格列头），约定属性名使用 `string.Empty`（表示所有属性均可能变更），由控件层在收到通知后执行全量或增量刷新。若后端实现方能精确知道变更范围，也可触发具体属性名，但控件层不依赖此精确性。

**协作**：被 `ICurveData`、`IMapData`、`ICubeData` 继承；被 `CalibrationEditorBase<T>` 作为泛型约束使用；被表格适配层、图表抽象层和 Curve 表格契约层消费。

**类型形态**：接口，**明确继承 `INotifyPropertyChanged`**。具体数据模型由后端标定系统定义，控件组只消费契约，不拥有实现。

### `ICurveData` / `IMapData` / `ICubeData`（接口）

**角色**：维度特化的数据契约。

**职责**：在 `ICalibrationData` 基础上，各自声明维度特定的数据访问方式。

**关键成员**：

`ICurveData`：
- `int Length { get; }` — 数据点数量
- `IReadOnlyList<double> XValues { get; }` — 一维 x 轴值序列（自变量）
- `IReadOnlyList<double> ZValues { get; }` — 一维 z 轴值序列（因变量）
- `void SetXValue(int index, double value)` — 更新指定索引处的 x 值；更新完成后触发 `PropertyChanged`，属性名使用 `string.Empty`
- `void SetZValue(int index, double value)` — 更新指定索引处的 z 值；更新完成后触发 `PropertyChanged`，属性名使用 `string.Empty`

`IMapData`：
- `int XAxisLength { get; }`、`int YAxisLength { get; }` — 各轴维度长度
- `double GetValue(int xIndex, int yIndex)` — 读取二维矩阵指定位置的 z 值
- `void SetValue(int xIndex, int yIndex, double value)` — 更新二维矩阵指定位置的 z 值；更新完成后触发 `PropertyChanged`，属性名使用 `string.Empty`
- `IReadOnlyList<double> XValues { get; }` — x 轴坐标值序列
- `IReadOnlyList<double> YValues { get; }` — y 轴坐标值序列

`ICubeData`：
- `int XAxisLength { get; }`、`int YAxisLength { get; }`、`int ZAxisLength { get; }` — 各轴维度长度
- `double GetValue(int xIndex, int yIndex, int zIndex)` — 读取三维张量指定位置的值
- `void SetValue(int xIndex, int yIndex, int zIndex, double value)` — 更新三维张量指定位置的值；更新完成后触发 `PropertyChanged`，属性名使用 `string.Empty`
- `IReadOnlyList<double> XValues { get; }` — x 轴坐标值序列
- `IReadOnlyList<double> YValues { get; }` — y 轴坐标值序列
- `IReadOnlyList<double> ZValues { get; }` — z 轴坐标值序列

**协作**：作为 `CalibrationEditorBase<T>` 的泛型参数具体化类型；被各自的表格适配器或 Curve 表格呈现器消费。

**类型形态**：接口，继承自 `ICalibrationData`。不采用抽象类，因为后端数据模型可能已经继承自其他领域基类，接口避免单继承限制。

### `IAsyncSaveCapable`（接口）

**角色**：标识后端数据模型支持异步持久化保存，并为脏标记跟踪提供状态查询能力。

**职责**：
- 提供未保存变更状态的查询能力，支撑控件层的脏标记视觉反馈和状态栏未保存提示
- 提供异步保存触发入口，供控件层在用户请求保存时调用

**关键成员**：
- `bool HasUnsavedChanges { get; }` — 是否存在尚未持久化的变更；控件层据此决定是否显示脏标记（橙色边框）和状态栏未保存数量提示
- `Task SaveAsync()` — 异步触发保存操作；保存成功后 `HasUnsavedChanges` 应返回 `false`
- `event EventHandler? UnsavedChangesChanged` — 未保存状态变更事件，供控件层实时响应（如用户在外部触发保存后自动清除脏标记）

**协作**：由后端数据模型选择性实现；被 `CalibrationEditorBase<T>` 在检测到数据模型实现此接口时启用脏标记跟踪和状态栏提示。若数据模型未实现此接口（即时持久化模式），控件层自动禁用脏标记相关反馈。

**类型形态**：接口，独立于 `ICalibrationData` 继承树。不强制所有数据模型实现，保持向后兼容性。

### `CalibrationEditorBase<T>`（抽象类）

**角色**：三个标定编辑控件的公共基座。

**职责**：
- 定义并管理 Avalonia 依赖属性：`ItemsSource`（`IList<T>` 类型，绑定到变量选择下拉框）、`SelectedVariable`（当前选中的标定数据项）
- 提供统一的顶部信息栏模板结构（左侧 `ComboBox`、右侧单位/轴信息展示）
- 处理变量切换时的选中变更事件，向子类发出数据切换通知
- 信息栏文本格式化：默认格式为 `[{Unit}] x: {XAxisName} [{XAxisUnit}]`，若存在 y/z 轴则依次追加 `y: {YAxisName} [{YAxisUnit}]`、`z: {ZAxisName} [{ZAxisUnit}]`；子类可通过覆盖受保护的虚方法 `FormatInfoText(T variable)` 自定义模板
- 空状态管理：当 `ItemsSource` 为 `null`、空列表或 `SelectedVariable` 为 `null` 时，进入空状态展示（如显示 "无有效数据" 占位提示）；提供受保护的虚方法 `OnEnterEmptyState()` / `OnExitEmptyState()` 供子类覆盖
- 脏标记状态栏提示：当数据模型实现 `IAsyncSaveCapable` 时，订阅其 `UnsavedChangesChanged` 事件，在信息栏区域显示未保存数量（如 "3 处未保存"）

**协作**：内部聚合一个 `ComboBox` 和一个信息展示面板；通过模板约定或受保护的虚方法向子类（`CurveEditor`、`MapEditor`、`CubeEditor`）暴露数据切换钩子。

**类型形态**：泛型抽象类（`T : ICalibrationData`），继承自 Avalonia `TemplatedControl`。选用抽象类而非接口，是因为需要封装具体的依赖属性注册、模板部件约定和公共视觉状态逻辑。

### `CurveEditor` / `MapEditor` / `CubeEditor`（密封类）

**角色**：面向终端使用者的三个具体控件。

**职责**：
- `CurveEditor`：主内容区上方为折线图区域、下方为 1 维两行表格（x 行 + z 行）。表格采用横向布局，通过 `ICurveTablePresenter` 构建，不使用 `DataGrid` 纵向行模型
- `MapEditor`：主内容区左侧为 3D 曲面图、右侧为 2 维表格（x 列头 + y 行头 + z 数据单元格）
- `CubeEditor`：主内容区左侧为 3D 曲面图（展示当前激活 z 切片）、右侧为平铺 3 维表格（z/y 双列行头 + x 列头 + 数据单元格）。表格以平铺形式展示完整数据立方体的所有 (z, y) 行 × x 列，z 切片激活由表格内的选中/导航交互驱动

**协作**：`MapEditor` 与 `CubeEditor` 各自内部持有对应的表格适配器实例和图表呈现器实例；`CubeEditor` 额外持有 `ZSliceActivationTracker`。`CurveEditor` 内部持有 `ICurveTablePresenter` 实例和 `IChartPresenter` 实例。

**类型形态**：密封类，继承 `CalibrationEditorBase<T>`（`T` 分别为 `ICurveData`、`IMapData`、`ICubeData`）。密封是因为这些控件是领域终态产品，不存在合理的进一步派生场景。

### `ICurveTablePresenter`（接口）

**角色**：`CurveEditor` 横向两行表格的视图构建与交互处理契约。

**职责**：
- 接收 `ICurveData`，构建两行横向表格的视觉树。第一行展示所有 x 轴值，第二行展示所有 z 轴值，列数与数据点数量一致
- 管理单元格选中状态（当前活动单元格）和单元格编辑状态（进入/退出编辑模式）
- 提供单元格编辑完成事件，携带数据点索引（列位置）和行标识（X 行或 Z 行），供 `CurveEditor` 回写到原始数据模型
- 提供选中单元格变更通知，供 `CurveEditor` 同步驱动折线图的数据点高亮
- 暴露列宽调整能力：实现方可通过拖拽列分隔线调整列宽，或通过均分/自适应策略初始化列宽
- 支持程序化选中设置和清除，供 `CurveEditor` 在变量切换后重置选中，以及在图表联动时反向选中表格单元格

**关键成员**：
- `void LoadData(ICurveData data)` — 加载数据并重建视觉树
- `Control VisualRoot { get; }` — 视觉树的根元素，供 `CurveEditor` 嵌入到主内容区
- `event EventHandler<CurveCellSelectedEventArgs> SelectedCellChanged` — 选中单元格变更事件，参数携带列索引和行标识（X/Z）
- `event EventHandler<CurveCellEditCompletedEventArgs> CellEditCompleted` — 单元格编辑完成事件，参数携带列索引、行标识（X/Z）和新数值
- `void Refresh()` — 响应外部数据变更通知，刷新显示内容
- `void SelectCell(int columnIndex, CurveTableRow row)` — 程序化选中指定列和行的单元格
- `void ClearSelection()` — 清除当前选中状态

**协作**：被 `CurveEditor` 内部持有；向下消费 `ICurveData`；向上通过事件通知 `CurveEditor` 单元格编辑完成和选中变更。实现方基于 Avalonia 自定义布局（如 `Grid` + `ItemsControl` 或自定义 `Panel`）构建横向视觉树，不依赖 `DataGrid`。

**类型形态**：接口。`CurveEditor` 的横向布局与 `MapEditor`/`CubeEditor` 的纵向 `DataGrid` 行模型本质不同，独立的契约允许实现者选择最适合横向布局的 Avalonia 布局容器。

### `CurveTableRow`（枚举）

**角色**：标识 `CurveEditor` 横向表格中的行类型。

**值**：
- `X` — 表示 x 轴（自变量）行
- `Z` — 表示 z 轴（因变量）行

**类型形态**：枚举。行类型只有两种固定取值，枚举在语义上最精确。

### `ITableDataSource<TRow>`（接口）

**角色**：多维数据到二维纵向表格结构的投影器。供 `MapEditor` 与 `CubeEditor` 使用。

**职责**：将原始标定数据（二维矩阵、三维张量）转换为适合 Avalonia `DataGrid` 绑定的行集合。各维度实现返回各自强类型的行数据对象集合，行数据对象内联携带原始数据坐标索引，以支撑编辑后的值回写。

**关键成员**：
- `IReadOnlyList<TRow> Rows { get; }` — 纵向行集合，可直接绑定到 `DataGrid.ItemsSource`
- `int GetXIndexFromColumn(int columnIndex)` — 将 DataGrid 的原始列索引（包含所有行头列）映射为原始数据的 x 轴索引（供列头生成和编辑回写使用）。实现方内部处理行头列的偏移映射
- `void Refresh()` — 响应外部原始数据模型的变更通知，重新生成 `Rows` 行集合；Editor 在检测到数据模型 `PropertyChanged` 后调用此方法，随后重新设置 `DataGrid.ItemsSource` 以刷新表格显示
- `IReadOnlyList<string> ColumnHeaders { get; }` — x 轴数据列的列头文本集合（不包含行头列），长度等于数据列数，供 Editor 在变量切换时动态生成 `DataGrid` 列
- `int DataColumnCount { get; }` — x 轴数据列的数量（不包含行头列）
- `bool SupportsIncrementalRefresh { get; }` — 标识当前实现是否支持增量刷新。若返回 `true`，实现方应同时实现 `INotifyCollectionChanged`（如通过 `ObservableCollection<TRow>`），使 Editor 可通过单元素替换实现行级增量刷新

**协作**：被 `MapEditor` 和 `CubeEditor` 内部使用，作为 `DataGrid` `ItemsSource` 的实际来源。编辑完成时，Editor 从强类型行数据对象中提取坐标索引属性，调用原始数据模型的写接口。非编辑路径下的数据变更（如后端标定数据从 ECU 更新后）由 Editor 订阅原始模型的 `PropertyChanged`，再调用 `Refresh()` 重新生成行集合并更新 `ItemsSource`。

**类型形态**：泛型接口（`ITableDataSource<TRow> where TRow : struct`），分别实例化为 `ITableDataSource<MapRowData>` 和 `ITableDataSource<CubeRowData>`。泛型约束 `TRow : struct` 确保行数据为值类型，避免堆分配。

### `MapRowData` / `CubeRowData`（`readonly struct`）

**角色**：承载一行纵向表格数据的展示值及其在原始多维数据中的索引坐标。供 `MapEditor` 与 `CubeEditor` 使用。

**职责**：
- `MapRowData`：内联 `YIndex` 属性，标识该行在二维矩阵中的 y 坐标；承载 `YValue` 文本（用于 y 轴行头展示）。每行对应一个 y 值下所有 x 列的数据值，x 列的值通过索引器按列顺序暴露
- `CubeRowData`：内联 `ZIndex`、`YIndex` 属性，标识该数据行在三维张量中的坐标；承载 `ZValue` 和 `YValue` 文本（用于 z/y 双列行头展示）。每行对应一个 (z, y) 组合下所有 x 列的数据值。x 列的值通过索引器按列顺序暴露，列索引与 x 轴索引一一映射

**协作**：由对应维度的表格适配器生成并填充，作为 `DataGrid` 的 `ItemsSource` 行集合。Editor 在编辑确认时从行数据对象中直接读取坐标索引属性，回写到原始数据模型。

**类型形态**：`readonly struct`。值类型避免堆分配，坐标索引作为结构体的属性直接内联，不通过接口引用传递，消除接口引用传递中的装箱开销。需注意 `DataGrid` 将行数据赋给 `DataGridRow.DataContext`（`object` 类型属性）时仍会发生一次 boxing，此机制由 Avalonia 框架决定，无法避免；但行数据本身避免了堆分配和接口引用的额外装箱，总体 GC 压力仍显著低于 `class` 方案。

### `IChartPresenter`（接口）

**角色**：图表渲染的抽象契约，平台无关的基接口。

**职责**：定义图表渲染的最小通用能力（刷新、渲染失败通知），不依赖任何 UI 框架类型。Avalonia 特定的宿主关联能力下沉到 `IAvaloniaChartPresenter` 子接口。

**关键成员**：
- `void Refresh()` — 显式触发重绘
- `event EventHandler<ChartRenderFailedEventArgs> RenderFailed` — 渲染失败通知事件，参数携带降级原因描述，供宿主控件展示错误占位

**协作**：被 `CurveEditor`、`MapEditor`、`CubeEditor` 内部持有。实现方可选用 OxyPlot、LiveCharts、ScottPlot、自定义 Skia 渲染等具体技术。

**类型形态**：接口。自身无 UI 框架依赖，确保将来若需将图表引擎替换为离屏渲染或跨平台实现时，基接口不受影响。

### `IAvaloniaChartPresenter`（接口）

**角色**：`IChartPresenter` 的 Avalonia 平台特化子接口，承载宿主控件关联能力。

**职责**：将图表渲染目标关联到指定的 Avalonia 宿主控件，以及解除关联并释放资源。

**关键成员**：
- `void AttachTo(Control host)` — 将图表渲染目标关联到指定的 Avalonia 宿主控件
- `void Detach()` — 解除与宿主控件的关联并释放资源

**协作**：`CurveEditor`、`MapEditor`、`CubeEditor` 在内部持有 `IAvaloniaChartPresenter`（或其维度特化子接口）实例，在模板加载完成后调用 `AttachTo` 传入宿主控件。

**类型形态**：接口，继承自 `IChartPresenter`。拆分原因：将平台无关的渲染契约与 Avalonia 特定的宿主关联能力分离，保持基接口的跨平台纯洁性。

### `ILineChartPresenter` / `ISurfaceChartPresenter`（接口）

**角色**：`IAvaloniaChartPresenter` 的维度特化子接口。

**职责**：
- `ILineChartPresenter`：专用于 1 维折线图渲染，接收 `ICurveData`，绘制 x-z 关系的折线图（含数据点标记和连线）
- `ISurfaceChartPresenter`：专用于 2D/3D 曲面图渲染，接收原始数组数据，绘制 (x, y) → value 的彩色网格曲面

**关键成员**：

`ILineChartPresenter`：
- `void LoadData(ICurveData data)` — 加载曲线数据
- `void HighlightDataPoint(int index)` — 高亮指定索引的数据点（响应表格选中联动）

`ISurfaceChartPresenter`：
- `void LoadMapData(double[,] data, IReadOnlyList<double> xValues, IReadOnlyList<double> yValues)` — 加载二维 Map 数据（MapEditor 使用）
- `void LoadSliceData(double[,] sliceData, IReadOnlyList<double> xValues, IReadOnlyList<double> yValues)` — 加载激活 z 切片的二维矩阵数据（CubeEditor 使用）

**协作**：`CurveEditor` 消费 `ILineChartPresenter`；`MapEditor` 消费 `ISurfaceChartPresenter`（加载完整 Map 数据）；`CubeEditor` 消费 `ISurfaceChartPresenter`（加载当前激活 z 切片的二维矩阵）。

**类型形态**：接口，继承自 `IAvaloniaChartPresenter`。拆分为两个特化接口，是因为折线图和 3D 曲面图的数据加载方式、交互反馈（如数据点高亮）差异显著，单一接口会导致大量不适用的成员。两个加载方法均统一为接收原始数组的形式，由调用方（Editor）负责从数据模型提取数据，消除参数风格的不对称性。

### `ZSliceActivationTracker`（类）

**角色**：CubeEditor 中 z 切片激活状态的专门管理者。

**职责**：
- 维护当前激活的 z 值索引；默认激活策略为 `ZIndex = 0`（第一个 z 切片），变量切换时重置为此默认值
- 接收 `CubeEditor` 传递的表格选中变更事件（`OnSelectionChanged(CubeRowData? row)`），从 `CubeRowData` 中提取 `ZIndex`，判断是否需要切换激活切片
- 接收 z 列点击事件（`OnZColumnClicked(CubeRowData row)`）：将该单元格所属行的 z 值设为激活切片，并通知 `CubeEditor` 选中该切片内的第一个数据单元格
- 提供激活切片变更通知（防抖结束后触发，供 3D 图表刷新和 z 切片级高亮使用）
- 处理键盘快速跨 z 切片导航时的防抖/延迟策略（若数据量大）。防抖期间，表格立即响应导航并更新行选中，3D 曲面图等待防抖结束后刷新

**关键成员**：
- `void OnSelectionChanged(CubeRowData? row)` — 输入方法：表格当前选中行变更时由 `CubeEditor` 调用；`null` 表示无选中项
- `void OnZColumnClicked(CubeRowData row)` — 输入方法：用户点击 z 列单元格时由 `CubeEditor` 调用
- `int ActiveZIndex { get; }` — 当前激活的 z 值索引
- `event EventHandler<ActiveZSliceChangedEventArgs> ActiveSliceChanged` — 激活切片变更事件。参数携带旧 `ZIndex` 和新 `ZIndex`。事件仅在防抖计时结束后（用户停止快速导航）触发，而非每次潜在变更都触发，以避免频繁通知 3D 图表重绘
- `TimeProvider TimeProvider { get; }` — 时间提供器（.NET 8+），默认使用 `TimeProvider.System`；测试中可注入 `FakeTimeProvider` 以快速推进防抖延迟

**协作**：由 `CubeEditor` 实例化并注入；向下消费表格适配器提供的 `CubeRowData` 行数据（通过 `ZIndex` 属性按 z 值分组）；向上通过纯 .NET 事件（`EventHandler<T>`）通知 `CubeEditor`。`CubeEditor` 在事件处理器中通过 `Dispatcher.UIThread.Post` 将状态变更调度到 UI 线程，确保 Tracker 不依赖 Avalonia 视觉类型，保持可测试性。

**类型形态**：独立类（非静态）。将这段逻辑从 `CubeEditor` 中剥离，使控件类聚焦模板和交互路由，状态映射逻辑可独立单元测试。

### `ActiveZSliceChangedEventArgs`（类）

**角色**：`ZSliceActivationTracker.ActiveSliceChanged` 事件的事件参数。

**职责**：携带 z 切片变更前后的索引值。

**关键成员**：
- `int OldZIndex { get; }` — 变更前的 z 切片索引
- `int NewZIndex { get; }` — 变更后的 z 切片索引

**类型形态**：密封类，继承 `EventArgs`。

## 关键行为契约

### 变量切换流程

```
[用户操作 ComboBox]
    │
    ▼
[CalibrationEditorBase<T> 更新 SelectedVariable]
    │
    ├── ItemsSource 为 null / 空列表 ──→ [触发 OnEnterEmptyState] ──→ [显示空状态占位]
    │
    ├── ItemsSource 非空，SelectedVariable 为 null ──→ [触发 OnEnterEmptyState] ──→ [显示空状态占位]
    │
    └── ItemsSource 非空，SelectedVariable 有效 ──→ [触发 OnExitEmptyState] ──→ [触发 OnSelectedVariableChanged]
                                                │
                                                ├── CurveEditor: 通知 ICurveTablePresenter 重建，通知 ILineChartPresenter 加载
                                                ├── MapEditor: 重建 ITableDataSource，通知 ISurfaceChartPresenter 加载
                                                └── CubeEditor: 重建 ITableDataSource，重置 ZSliceActivationTracker 为默认 z 切片（ZIndex = 0），通知 ISurfaceChartPresenter 加载
```

### 单元格编辑与数据回写

#### 编辑触发方式

表格数据单元格采用 **双击进入编辑** 模式，单击仅用于选中/导航。行头列（z 列、y 列）不参与编辑，单击仅触发选中或 z 切片激活。

#### MapEditor / CubeEditor（DataGrid 纵向行模型）

用户触发单元格进入编辑模式（双击）→ 单元格展示编辑模板（如 `TextBox`）→ 用户输入数值 → 输入完成时（Enter/失去焦点），Editor 拦截 `CellEditEnding` 事件，手动解析并验证输入值：

1. **验证失败**：通过设置编辑模板内 `TextBox` 的 `BorderBrush` 和 `ToolTip` 实现红色边框就地反馈，阻止该次编辑回写，但不阻断用户继续编辑其他单元格
2. **验证通过**：
   - 从对应类型的行数据对象中提取坐标索引属性
   - 调用 `ITableDataSource<TRow>.GetXIndexFromColumn(columnIndex)` 将 DataGrid 原始列索引（含行头列）映射为 x 轴索引
   - 调用原始数据模型的写方法更新对应位置
   - 设置 `e.Cancel = true` 取消 DataGrid 的默认提交（因为 `readonly struct` 行数据无法通过标准绑定机制写回）
   - 模型触发 `INotifyPropertyChanged.PropertyChanged`（属性名为 `string.Empty`）
   - Editor 订阅到该通知后，根据 `ITableDataSource<TRow>.SupportsIncrementalRefresh` 选择刷新策略：
     - 若支持增量刷新：通过单元素替换（如 `Rows[index] = newRow`）仅刷新对应行，DataGrid 仅重绘该行
     - 若不支持增量刷新：调用 `ITableDataSource<TRow>.Refresh()` 重新生成行集合
   - 重新设置 `DataGrid.ItemsSource`（全量刷新时）或确认绑定已自动更新（增量刷新时）
   - **选中状态保持**：编辑前保存 `CurrentCell` 的列索引和行对象引用（或行索引），刷新后根据新 `ItemsSource` 重新定位对应单元格并恢复 `CurrentCell` / `SelectedItem`，确保焦点不跳回表格顶部
   - 若该单元格值同时影响图表，图表呈现器接收通知后重绘

#### CubeEditor 编辑流程补充

在 CubeEditor 中，编辑操作发生在平铺表格的任意数据单元格。编辑完成后，Editor 从该单元格对应的 `CubeRowData` 中读取 `ZIndex`、`YIndex`，并结合该单元格所在的 x 列索引（通过 `ITableDataSource<TRow>.GetXIndexFromColumn` 映射），调用 `ICubeData` 的写接口更新三维张量中对应位置的值。

#### CurveEditor（横向自定义布局）

用户触发单元格进入编辑模式（双击）→ `ICurveTablePresenter` 在对应列的对应行展示编辑控件 → 用户输入数值 → 输入完成时，`ICurveTablePresenter` 触发 `CellEditCompleted` 事件，携带列索引、行标识（X 或 Z）以及解析后的数值 → `CurveEditor` 在事件处理器中调用 `ICurveData` 的写接口更新对应位置 → 模型触发 `PropertyChanged`（属性名为 `string.Empty`）→ `ICurveTablePresenter` 刷新对应单元格显示，`ILineChartPresenter` 重绘折线图。

#### 变更反馈机制（即时同步模式）

采用 **即时同步** 模式：编辑完成后立即调用数据模型写接口，数据变更实时反映到表格和图表。

变更反馈的具体形式：
- **脏标记视觉反馈**：已编辑但尚未持久化的单元格以边框变色（如橙色细边框）提示已修改。此反馈仅在数据模型实现 `IAsyncSaveCapable` 时生效；控件层通过检测后端是否实现 `IAsyncSaveCapable` 接口来获取 `HasUnsavedChanges` 状态，决定是否启用脏标记跟踪。若后端为即时持久化（无存盘按钮/无异步保存概念），此反馈自动禁用，避免永久不触发的无效状态指示
- **状态栏提示**：`CalibrationEditorBase<T>` 的信息栏区域在存在未保存修改时显示未保存数量（如 "3 处未保存"），数据源为 `IAsyncSaveCapable.HasUnsavedChanges`。启用条件与脏标记视觉反馈一致
- **模型变更通知**：数据模型写接口触发 `PropertyChanged`（属性名 `string.Empty`）后，Editor 调用 `ITableDataSource<TRow>.Refresh()`（MapEditor/CubeEditor）或 `ICurveTablePresenter.Refresh()`（CurveEditor）刷新单元格显示，作为最基础的变更确认

### z 切片激活与 3D 视图联动（CubeEditor）

用户点击/键盘导航到表格中的某个单元格或行 → `CubeEditor` 将选中的 `CubeRowData` 通过 `OnSelectionChanged` 传递给 `ZSliceActivationTracker` → Tracker 读取该行的 `ZIndex` 属性，与当前激活 z 比较：若相同则无动作；若不同则启动防抖计时 → 防抖结束后（用户停止快速导航），Tracker 更新激活 z 值并通过 `ActiveSliceChanged` 事件发出变更通知 → `CubeEditor` 在事件处理器中通过 `Dispatcher.UIThread.Post` 调度 UI 更新：通知 `ISurfaceChartPresenter` 切换数据源到新的 z 切片矩阵，同时触发表格的 z 切片级高亮样式更新。

当用户点击 z 列的任意单元格时，`CubeEditor` 调用 `ZSliceActivationTracker.OnZColumnClicked`，Tracker 将该单元格所属行的 z 值设为激活切片（同样经过防抖），并通知 `CubeEditor` 选中该切片内的第一个数据单元格。

键盘跨 z 切片导航时（如从当前 z 切片的最后一行 y 值按方向键移入下一 z 切片的第一行 y 值），表格的 `CurrentCell` / `SelectedItem` 立即响应键盘输入而切换，`ZSliceActivationTracker` 内部引入短时间防抖（如 100–200ms），仅在用户停止快速导航后才通过 `ActiveSliceChanged` 事件真正刷新 3D 曲面图。此期间存在约 100–200ms 的视觉不一致（表格已切换行，3D 视图仍显示旧切片），在标定工具场景下属于可接受的轻微延迟；若需缓解，可在防抖期间为 3D 视图区域施加半透明遮罩（如 30% 透明度灰色覆盖），提示"正在加载"，防抖结束后移除遮罩并刷新图表。

3D 曲面图数据源切换采用**直接瞬时刷新**策略，不引入过渡动画，与表格即时响应保持一致。轻量淡入淡出过渡动画作为可选增强，在阶段 2 评估引入。

### z 切片级高亮状态与 DataGrid 虚拟化联动（CubeEditor）

`DataGrid` 仅对当前可视区域内的行创建视觉树实例（行虚拟化）。`ZSliceActivationTracker` 输出的激活 `ZIndex` 需通过以下机制传递到行视觉元素：

1. **`CubeEditor` 订阅 `ZSliceActivationTracker` 的 `ActiveSliceChanged` 事件**。当激活 `ZIndex` 变化时，`CubeEditor` 执行两项操作：通知 `ISurfaceChartPresenter` 刷新 3D 曲面图；遍历当前已加载的 `DataGridRow` 容器，为属于激活 z 切片的行附加样式类（如 `active-z-slice`），为非激活行移除该样式类。

2. **`CubeEditor` 订阅 `DataGrid` 的 `LoadingRow` 事件**。当用户滚动导致新行进入可视区域时，`DataGrid` 创建对应的行视觉元素（新创建或从重用车池中取出）并触发 `LoadingRow` 事件。`CubeEditor` 在事件处理器中**先无条件移除该行已附加的 z 切片高亮样式类**（避免重用车池中的残留状态），再读取该行的 `DataContext`（`CubeRowData`），若其 `ZIndex` 等于当前激活 `ZIndex`，则为新行附加 z 切片高亮样式类。

3. **`CubeEditor` 订阅 `DataGridRow` 的 `DataContextChanged` 事件**。对于因虚拟化而被重用的行容器，`DataContext` 变更时触发此事件。`CubeEditor` 在事件处理器中同样先无条件移除 z 切片高亮样式类，再根据新的 `DataContext` 中的 `ZIndex` 判断是否重新附加。此机制作为 `LoadingRow` 策略的后备，确保行重用时视觉状态始终与当前数据一致。

> **注意**：Avalonia `DataGrid` 不存在 `UnloadingRow` 事件，因此不依赖该行卸载事件进行清理。上述「`LoadingRow` 先清理再判断」+「`DataContextChanged` 后备清理」的双重机制足以覆盖虚拟化场景下的状态一致性需求。

通过上述机制，`ZSliceActivationTracker` 的状态输出（纯 .NET 事件）与 `DataGrid` 视觉树的状态同步解耦：Tracker 不依赖 Avalonia 视觉类型，而 `CubeEditor` 作为 UI 层协调者负责将逻辑状态映射到视觉状态，兼容 `DataGrid` 的行虚拟化和行回收机制。

### 高亮叠加策略

四种高亮在 CubeEditor 中可能同时存在，视觉优先级从高到低为：

1. **单元格级高亮**（蓝色背景，当前活动单元格）— 由 DataGrid 内置选中状态驱动
2. **行级高亮**（轻微背景色，当前选中单元格所在的整行）— 由 DataGrid 的 `CurrentItem` / 当前活动单元格所在行驱动，通过 DataGrid 内置选中行样式或基于 `CurrentCell` 状态的样式选择器实现
3. **z 切片级高亮**（左侧边框或分组背景色，当前激活切片的所有行）— 仅由 `ZSliceActivationTracker` 输出的激活 `ZIndex` 驱动
4. **行头文字样式变化**（加粗或颜色变化）— 当前行被选中时 y 列文字加粗或变色；当前 z 切片激活时 z 列文字加粗或变色。通过 `DataGridCell` 样式选择器（根据所属列和行的选中/激活状态匹配）或 `LoadingRow` 事件中对行头单元格附加/移除样式类实现

四者的驱动来源完全独立：
- 单元格级高亮 → DataGrid 内置选中机制
- 行级高亮 → DataGrid `CurrentItem` / `CurrentCell` 状态
- z 切片级高亮 → `ZSliceActivationTracker` 激活状态
- 行头文字样式变化 → DataGrid `CurrentItem` 状态（y 列）+ `ZSliceActivationTracker` 激活状态（z 列）

通过 Avalonia 的样式选择器（Style Selector）和 `Classes` 动态附加机制实现层级叠加。

### 列宽/行高调整策略

#### MapEditor / CubeEditor（DataGrid）

- **列宽调整**：启用 `CanUserResizeColumns`，允许用户拖拽列分隔线调整列宽。x 轴数据列初始宽度采用 `DataGridLength.Auto` 或固定像素值（如 80px），确保列头文本完整显示；y/z 行头列宽度略宽于数据列（如 100px），容纳轴值文本
- **行高调整**：Avalonia `DataGrid` 无 `MinRowHeight`/`MaxRowHeight` 属性，通过 `DataGrid.Styles` 中为 `DataGridRow` 设置样式实现行高限制：
  ```
  Style Selector="DataGridRow"
      Setter Property="MinHeight" Value="24"
      Setter Property="MaxHeight" Value="48"
  ```
- **最小/最大列宽约束**：动态生成的 x 轴数据列设置最小宽度（如 60px），防止数据量过大时列宽被压缩至不可读；最大宽度（如 200px）防止单列过宽占用过多水平空间

#### CurveEditor（横向自定义布局）

`ICurveTablePresenter` 的实现方负责列宽调整能力：
- 默认均分列宽（总宽度 / 列数）
- 支持用户拖拽列分隔线调整单列宽度
- 提供最小列宽约束（如 60px），确保数值可读

### 精度展示策略

精度信息流转路径：
1. **数据源提供**：`ICalibrationData` 通过 `FormatString` 和 `DisplayPrecision` 提供精度策略
2. **表格适配层消费**：`MapTableAdapter` / `CubeTableAdapter` 在生成行数据时，按 `FormatString`（优先）或 `DisplayPrecision`（后备）格式化数值文本
3. **图表消费**：`ILineChartPresenter` / `ISurfaceChartPresenter` 的轴标签和数据标签使用同一格式策略，确保表格与图表的数值展示一致性
4. **编辑控件消费**：单元格进入编辑模式时，`TextBox` 的初始文本按相同格式策略格式化，用户输入完成后解析为数值

## 错误处理策略

### 输入验证错误

单元格编辑阶段的数值解析失败（如非法字符、越界值）属于局部错误。错误处理策略为「就地反馈」：在单元格层面通过红色边框或工具提示标识错误，阻止该次编辑回写，但不阻断用户继续编辑其他单元格或切换变量。

在 `MapEditor`/`CubeEditor` 中，由于行数据类型为 `readonly struct`，DataGrid 的标准编辑双向绑定自动提交机制无法直接写回不可变结构体。根本原因并非单纯的「不可变性」，而是**值类型（`struct`）在通过 `object` 类型（如 `DataGridRow.DataContext`）传递时发生 boxing，DataGrid 拿到的是已 boxing 的副本；即使将 `readonly struct` 降级为可变 `struct`，标准编辑提交对该副本的修改仍不会反映回 `Rows` 集合中的原始实例**。只有降级为 `class`（引用类型）才能让 DataGrid 的标准编辑机制直接修改原集合中的对象。设计采用 **完全手动事件拦截** 策略应对此限制：
- 在 `CellEditEnding` 事件中拦截编辑值，手动解析并验证
- 解析失败时，通过设置编辑模板内 `TextBox` 的自定义验证状态（如附加 `DataValidationErrors` 属性或边框样式变更）实现红色边框反馈
- 验证通过后，调用原始数据模型写接口，然后取消 `CellEditEnding` 事件的默认提交（`e.Cancel = true`），由 Editor 监听模型的 `PropertyChanged`（属性名 `string.Empty`）后主动调用 `ITableDataSource<TRow>.Refresh()` 并重新设置 `DataGrid.ItemsSource`，完成 UI 刷新；同时执行选中状态保持步骤

在 `CurveEditor` 中，`ICurveTablePresenter` 的实现方负责拦截非法输入并在编辑控件层面呈现错误标识。

### 数据绑定不匹配错误

控件绑定到不符合维度期望的数据模型（如 `CubeEditor` 接收的 `ICubeData` 中 z 维度长度为 0）属于结构性错误。控件应进入「空状态」展示友好提示（如 "无有效数据"），而非抛出未处理异常导致控件白屏。

空状态覆盖场景：
- `ItemsSource` 为 `null`
- `ItemsSource` 为空列表（`Count == 0`）
- `SelectedVariable` 为 `null`
- 变量切换过程中（数据加载前）

`CalibrationEditorBase<T>` 提供受保护的虚方法 `OnEnterEmptyState()` 和 `OnExitEmptyState()`，子类可覆盖以自定义空状态视觉呈现。

### 图表渲染错误

图表库渲染失败（如 GPU 不可用、3D 数据异常）属于可降级错误。`IChartPresenter` 实现方应内部捕获渲染异常，通过 `RenderFailed` 事件通知宿主控件，宿主控件降级为显示占位矩形或错误图标，保证表格编辑功能不受影响。

## 并发设计

本场景为单用户桌面交互，无显式多线程并发需求。但需注意以下边界：

- **UI 线程亲和性**：所有 Avalonia 控件状态变更必须在 UI 线程执行。若后端标定数据模型由后台线程更新（如来自 ECU 通信线程），数据模型应在触发 `PropertyChanged` 前通过 `Dispatcher.UIThread.Invoke` 调度。
- **图表渲染隔离**：图表重绘可能是 CPU/GPU 密集型操作。`IChartPresenter` 实现时应考虑在后台线程准备顶点/网格数据，仅在提交渲染时回到 UI 线程，避免阻塞交互。
- **防抖策略**：CubeEditor 中键盘快速跨 z 切片导航时，`ZSliceActivationTracker` 内部可对激活切片变更引入短时间防抖（如 100–200ms），仅在用户停止快速导航后才真正刷新图表。Tracker 构造函数接受可选的 `TimeProvider` 参数（.NET 8+ 可用），默认使用 `TimeProvider.System`，测试中注入 `FakeTimeProvider` 以快速推进防抖延迟。
- **大数据量虚拟化**：CubeEditor 平铺所有 z 切片后，表格行数可达 z 数 × y 数，数据量显著增加。`DataGrid` 的行虚拟化（`EnableRowVirtualization`）必须保持开启；`CubeTableAdapter` 生成行集合时应避免在热路径中进行复杂计算，必要时采用惰性加载或分页策略。
- **编辑后全量刷新性能兜底**：若 `ITableDataSource` 不支持增量刷新，且数据量极端大（如 >10,000 行），全量 `Refresh()` + `ItemsSource` 替换可能造成 UI 卡顿。此时采用以下兜底策略：
  - **异步刷新**：在后台线程执行 `Refresh()` 的行集合重建，完成后通过 `Dispatcher.UIThread.Post` 回 UI 线程替换 `ItemsSource`
  - **延迟刷新**：若用户在短时间内连续编辑多个单元格，合并多次 `Refresh()` 为一次批量刷新（如 500ms 去抖动）
  - **批量刷新**：在实现阶段评估是否需要引入显式「保存/应用」按钮，将编辑暂存到本地缓冲区，用户确认后一次性批量写回并刷新

## 设计决策

### 1. CubeEditor 表格从「当前切片展示」改为「全量平铺展示」，行数膨胀后性能如何保障？

**决策**：保留 `DataGrid` 的行虚拟化机制，`CubeTableAdapter` 一次性生成完整的平铺行集合但依赖 UI 层的虚拟化只渲染可见行；若 z 数 × y 数极端大（如 >10,000 行），在实现阶段评估是否需要引入 z 切片折叠/展开或分页机制。

**理由**：v3 要求 CubeEditor 表格平铺展示所有 z 切片的完整数据，行数从原来的 y 数膨胀到 z 数 × y 数。Avalonia `DataGrid` 的行虚拟化机制可以处理数千行的滚动场景，只要行模板保持轻量即可。`CubeRowData` 作为 `readonly struct` 的数组一次性分配后由 `DataGrid` 按需访问，不会产生每行的堆分配压力。但若标定数据的 z 和 y 维度都很大（如各 100 以上），1 万行以上的虚拟化滚动体验可能下降，此时可在实现阶段引入 z 切片分组的折叠/展开机制作为性能兜底。架构设计阶段不预先引入折叠抽象，避免过度设计。

### 2. Avalonia DataGrid 原生不支持跨行合并单元格，z 列如何分组展示？

**决策**：不追求原生跨行合并，采用「z 值每行重复显示 + 视觉分组线」的组合方案。

**理由**：Avalonia 的 `DataGrid`（包括 V12）不提供 WPF 中 `RowSpan` 式的跨行单元格能力。若强行在模板层模拟合并单元格，会严重破坏虚拟化机制和行回收，在数据量大时性能急剧下降。

替代方案：z 列的自定义 `CellTemplate` 在每一行都渲染 z 值文本（相同 z 值的连续行显示相同文本），配合行样式在 z 切片边界处渲染水平分隔线；激活切片通过整组行的左侧边框或轻微背景色统一标识。此方案在虚拟化滚动下，即使某 z 切片的首行滚出可视区域，该切片剩余的所有可见行仍然显示 z 值，信息完整性得到保障。视觉上的重复文本通过分组线区分，认知负担可控。

### 3. 为什么用抽象基类 `CalibrationEditorBase<T>` 而非纯组合或接口？

**决策**：采用泛型抽象类承载公共 UI 结构和依赖属性。

**理由**：三个控件共享的不仅是行为契约，还有大量的具体 UI 结构——顶部栏的 `ComboBox`、单位信息面板、变量切换后的数据刷新管线。抽象类可以在 Avalonia 的类型系统中直接注册这些公共依赖属性（`AvaloniaProperty.Register`），并约定公共 ControlTemplate 的部件名称（如 `PART_VariableSelector`、`PART_InfoPanel`）。若用纯接口，每个控件需要重复注册相同的依赖属性和模板逻辑，违背 DRY 原则。组合模式（如注入一个「顶部栏管理器」对象）在 Avalonia 的模板/样式系统中反而增加不必要的间接层。

### 4. 表格编辑如何定位回原始数据？

**决策**：通过维度特化的 `readonly struct` 行数据对象建立可见单元格到原始多维索引的映射，坐标索引直接内联为结构体属性，取消独立的坐标接口抽象。

**理由**：Avalonia 表格控件（`DataGrid`）天然面向二维行集合（`IEnumerable` 的扁平行）。原始标定数据是 2D/3D 矩阵，无法直接作为 `ItemsSource`。引入表格适配层将矩阵投影为行集合后，每一行必须携带「我来自原始数据的哪个位置」这一元信息，否则编辑后无法回写。将坐标索引直接内联到各维度行数据类型（`MapRowData`、`CubeRowData`）中，以强类型属性传递坐标信息，彻底消除接口装箱，同时保持编辑回写时的坐标定位能力。`CubeRowData` 额外承载 `ZValue`、`YValue`，以支撑 z/y 双列行头的展示需求。

### 5. 3D 曲面图没有 Avalonia 内置方案，如何设计？

**决策**：通过 `IChartPresenter` 接口抽象图表渲染，拆分为平台无关的 `IChartPresenter` 基接口和 `IAvaloniaChartPresenter` 子接口，再按维度特化为 `ILineChartPresenter` 和 `ISurfaceChartPresenter`。

**理由**：Avalonia V12 标准控件集中不包含 3D 图表。社区中 OxyPlot、LiveCharts、ScottPlot 等库对 3D 的支持程度和 API 差异很大，且部分库在 V12 下的兼容性尚在发展。将图表渲染抽象为接口后，控件只关心「把数据给你，你在指定区域内画出来」。

图表交互深度分阶段决策：
- **阶段 1（MVP）**：仅支持默认视角静态展示，不支持缩放、平移、旋转
- **阶段 2**：支持鼠标拖拽旋转（3D 曲面图）和滚轮缩放（折线图和 3D 曲面图）；若确定支持交互，`IChartPresenter` 的实现方在内部处理交互事件，不向上暴露交互契约

3D 曲面图数据源切换动画决策：采用**直接瞬时刷新**，不引入过渡动画，与表格即时响应保持一致。轻量淡入淡出过渡动画作为可选增强，在阶段 2 评估引入。理由是标定工具场景下用户更关注数据切换的即时性和准确性，过渡动画可能引入不必要的视觉延迟；若后续用户反馈切换过程存在视觉跳跃感，再评估引入 100–200ms 的淡入淡出过渡。

实现阶段可以先以 2D 热力图或伪 3D 网格作为最小可行方案，后续替换为真正的 3D 引擎时不影响控件架构。

### 6. CubeEditor 的 z 切片激活逻辑为何独立为 `ZSliceActivationTracker`？

**决策**：将 z 切片激活的状态管理和事件映射抽取为独立类，而非内嵌在 `CubeEditor` 中。

**理由**：CubeEditor 的表格同时承担多重职责——数据展示、单元格编辑、z 切片选择器。若将 z 切片激活逻辑（行→z 值提取、激活变更判断、防抖、通知联动）全部写在 `CubeEditor` 的代码后台，会导致该类臃肿且难以测试。`ZSliceActivationTracker` 封装了「从表格选中状态到 z 切片语义」的纯逻辑转换，不依赖 Avalonia 控件的具体视觉类型，仅消费 `CubeRowData` 行数据对象。其状态输出采用纯 .NET 事件（`EventHandler<T>`），由 `CubeEditor` 在事件处理器中调度 UI 线程更新，这使得 Tracker 逻辑可以在无头环境中单元测试，也便于后续调整 z 切片切换的交互策略。

### 7. 编辑触发方式：单击还是双击？

**决策**：表格数据单元格采用 **双击进入编辑** 模式，单击仅用于选中/导航。

**理由**：在标定数据表格中，用户频繁进行选中浏览（查看相邻单元格数值、跨行比较）。若单击即进入编辑，会频繁弹出 `TextBox` 编辑模板，干扰浏览体验。双击编辑是桌面表格工具（Excel、INCA 等标定工具）的广泛惯例，与用户的工程工具心智模型一致。行头列（z 列、y 列）不参与编辑，单击它们仅触发选中或 z 切片激活。

### 8. CurveEditor 的横向两行表格为何不用 DataGrid，而采用独立布局契约？

**决策**：`CurveEditor` 不使用 `DataGrid` 和 `ITableDataSource` 纵向行集合抽象，而采用独立的 `ICurveTablePresenter` 接口，由实现方基于 Avalonia 自定义布局容器构建横向视觉树。

**理由**：需求明确要求 `CurveEditor` "以两行形式展示：第一行为 x 轴（自变量）值，第二行为 z 轴（因变量）值"，这是一种矩阵转置的表格形态——固定两行、动态多列。而 `DataGrid` 的设计范式是纵向行集合（每行一个数据对象，列对应属性），与横向布局的语义天然冲突。若强行套用 `DataGrid`，需要为每个数据点动态生成一列，并将两个行对象（x 行和 z 行）的数组索引绑定到各列，实现复杂度极高且违背 `DataGrid` 的设计意图。`CurveEditor` 的数据规模通常较小（一维曲线的数据点数量有限），不需要 `DataGrid` 的行虚拟化、列排序等重型功能。通过 `ICurveTablePresenter` 接口将横向布局的构建和交互抽象化后，实现方可选用 `Grid` + `ItemsControl`、`UniformGrid` 或自定义 `Panel` 等轻量方案，精确控制列宽、单元格间距和选中高亮行为，与需求的横向两行展示完全对齐。

### 9. 行数据类型为何选用 `readonly struct` 而非 `class`？

**决策**：`MapRowData` / `CubeRowData` 保持为 `readonly struct`。

**理由**：值类型避免堆分配，对于大数据量表格（CubeEditor 可能数千行）有显著的 GC 优势。`readonly struct` 的不可变性在函数式语义上更清晰。

但 `readonly struct` 与 DataGrid 标准单元格编辑的双向绑定自动提交机制不兼容，根本原因是**值类型（`struct`）本身的 boxing 副本问题**：DataGrid 将行数据赋给 `DataGridRow.DataContext`（`object` 类型属性）时发生 boxing，拿到的是已 boxing 的副本；即使将 `readonly struct` 降级为可变 `struct`，标准编辑提交对该副本的修改也不会反映回 `Rows` 集合中的原始实例。只有降级为 `class`（引用类型）才能让 DataGrid 的标准编辑机制直接修改原集合中的对象。`readonly` 修饰只是在此基础上额外禁止了 setter 调用，使问题暴露得更明显。

设计采用 **完全手动事件拦截** 策略应对此限制：在 `CellEditEnding` 事件中拦截编辑值，手动解析验证后直接调用后端模型写接口，然后取消默认提交（`e.Cancel = true`），由模型 `PropertyChanged`（属性名 `string.Empty`）通知触发 UI 刷新。

此策略的权衡：
- **收益**：消除 GC 压力，保持坐标索引的内联存储，回写链路清晰（不经过行数据对象的中间层）
- **代价**：放弃 DataGrid 内置的数据验证模板（`IDataErrorInfo` / `INotifyDataErrorInfo` 的自动红色边框反馈），验证错误标识需完全在 `CellEditEnding` 事件处理器中手动实现（如设置编辑模板内 `TextBox` 的 `BorderBrush` 和 `ToolTip`）；编辑后需要手动保持选中状态

**`class` 降级方案的补充分析**：
- **GC 影响**：每行一个 `class` 实例在 CubeEditor 大数据量（数千行）下会造成显著的堆分配压力和更频繁的 GC，对于实时性敏感的标定工具场景不利
- **坐标索引内联存储损失**：`class` 的字段通过引用访问，无法像 `struct` 那样将坐标索引直接内联在行数据对象中，缓存局部性下降
- **适用场景**：若实现阶段发现手动验证反馈的维护成本过高，或选中状态保持逻辑过于复杂，可将行数据类型降级为可变 `class`，保留坐标索引属性但允许数据值属性的 setter，恢复 DataGrid 内置编辑提交能力。此降级不影响架构设计层面的职责划分和协作关系

### 10. `FormatString` 与 `DisplayPrecision` 的优先级如何确定？

**决策**：`FormatString` 优先于 `DisplayPrecision`。

**理由**：`FormatString`（如 `F3`、`G6`、`0.0000`）提供了更精确的格式控制（包含科学计数法、前导零、千分位等），而 `DisplayPrecision` 仅表示小数位数，表达能力有限。当两者同时存在时，消费方（表格适配层、图表层）优先使用 `FormatString`；仅当 `FormatString` 为空或 `null` 时，回退到 `DisplayPrecision`，按 `F{displayPrecision}` 格式格式化。此规则确保后端可以精确控制展示格式，同时保留简化的精度声明方式作为后备。

### 11. `IChartPresenter.AttachTo` 的参数类型为何从 `Control` 改为 `object`，但又拆出 `IAvaloniaChartPresenter`？

**决策**：将 `IChartPresenter` 拆分为平台无关的基接口（仅含 `Refresh()` 和 `RenderFailed`）和 `IAvaloniaChartPresenter : IChartPresenter`（含 `AttachTo(Control host)`）。图表抽象层的基接口保持无 UI 依赖，Avalonia 特定契约下沉到子接口。

**理由**：模块职责表明确声明图表抽象层"自身无 UI 框架依赖"，但 v4 中 `IChartPresenter.AttachTo(Control host)` 的参数 `Control` 是 Avalonia 核心类型，直接引入了 UI 框架依赖，造成声明与实现矛盾。将平台无关能力（刷新、渲染失败通知）保留在基接口中，Avalonia 宿主关联能力下沉到子接口后：
- 将来若需将图表引擎替换为离屏渲染（如后台线程导出图片），实现方只需实现 `IChartPresenter` 即可，无需引入 Avalonia 依赖
- Avalonia 桌面场景下的实现方同时实现 `IAvaloniaChartPresenter`，`CurveEditor`/`MapEditor`/`CubeEditor` 内部持有 `IAvaloniaChartPresenter` 引用，调用 `AttachTo` 传入宿主控件
- 职责划分更清晰：基接口定义「渲染什么」，子接口定义「渲染到哪里」

### 12. 编辑后全量刷新为何同时保留增量刷新作为可选路径？

**决策**：`ITableDataSource<TRow>` 通过 `SupportsIncrementalRefresh` 属性让实现方声明是否支持增量刷新；控件层根据该属性选择全量或增量刷新策略。

**理由**：全量 `Refresh()` + 重新设置 `ItemsSource` 虽然实现简单，但存在两个副作用：(1) 替换 `ItemsSource` 后 `CurrentItem`/`CurrentCell`/`SelectedItem` 被重置，焦点跳回表格顶部，破坏编辑连续性（需额外执行选中状态保持步骤）；(2) 对于 `CubeEditor`，每次编辑都执行 O(z×y) 的全量重建，大数据量时造成 UI 卡顿。

增量刷新通过 `ObservableCollection<TRow>` 和单元素替换（`Rows[index] = newRow`）实现，DataGrid 仅刷新对应行，无需全量重建行集合，也无需替换 `ItemsSource`，自然保留选中状态。但增量刷新要求 `Rows` 集合实现 `INotifyCollectionChanged`，而接口层面返回 `IReadOnlyList<TRow>` 无法直接触发通知。通过 `SupportsIncrementalRefresh` 属性让实现方声明能力，控件层在检测到支持时走增量路径，不支持时回退到全量路径并执行选中状态保持。

性能兜底方案：若数据量极端大且不支持增量刷新，采用异步刷新（后台线程重建行集合）+ 延迟刷新（合并短时间内多次编辑为一次批量刷新），或在实现阶段引入显式「保存/应用」按钮，将编辑暂存到本地缓冲区后一次性批量写回。

## 修订说明（v5）

| 审查意见 | 修改措施 |
|---------|---------|
| **严重-问题1**：`readonly struct` 与 DataGrid 标准编辑机制不兼容的根因分析错误，将原因错误地归为「不可变性」而非值类型的 boxing 副本问题。 | 在「错误处理策略」→「输入验证错误」中修正根因说明：明确指出值类型（`struct`）在通过 `object` 类型传递时的 boxing 副本问题是根本原因；`readonly` 只是额外禁止了 setter 调用；明确说明无论 `readonly struct` 还是可变 `struct` 都必须采用手动事件拦截，只有降级为 `class` 才能利用 DataGrid 内置编辑提交。在设计决策 9 中补充 `class` 降级方案的 GC 影响和坐标索引内联存储损失分析。 |
| **严重-问题2**：单元格编辑后的全量刷新策略存在选中状态丢失和性能风险，但未给出应对方案。 | 在「单元格编辑与数据回写」→ MapEditor/CubeEditor 编辑流程中补充「选中状态保持」步骤：编辑前保存 `CurrentCell` 坐标，刷新后根据新 `ItemsSource` 重新定位并恢复选中。在 `ITableDataSource<TRow>` 中新增 `SupportsIncrementalRefresh` 属性，为增量刷新提供接口层面的契约支撑。在「并发设计」中补充性能兜底方案：异步刷新、延迟刷新、批量刷新。 |
| **中等-问题3**：数据模型元素级变更的 `PropertyChanged` 通知约定缺失。 | 在 `ICalibrationData`「核心抽象」中新增「元素级变更通知约定」小节：明确元素级写方法（`SetXValue`、`SetValue` 等）触发 `PropertyChanged` 时属性名使用 `string.Empty`（表示所有属性均可能变更），由控件层收到通知后执行刷新。在 `ICurveData`/`IMapData`/`ICubeData` 的关键成员描述中标注各写方法触发 `PropertyChanged`（属性名 `string.Empty`）。 |
| **中等-问题4**：`ISurfaceChartPresenter` 两个加载方法的参数风格不一致。 | 将 `LoadMapData(IMapData data)` 改为 `LoadMapData(double[,] data, IReadOnlyList<double> xValues, IReadOnlyList<double> yValues)`，与 `LoadSliceData` 统一为接收原始数组的形式。由调用方（Editor）负责从数据模型提取数据。在设计决策 5 中删除不对称性相关描述。 |
| **中等-问题5**：`ZSliceActivationTracker` 核心事件签名未定义。 | 在「核心抽象」中补充 `ActiveSliceChanged` 事件及 `ActiveZSliceChangedEventArgs` 密封类的完整定义。明确事件在防抖结束后触发，而非每次潜在变更都触发。补充 `TimeProvider` 构造函数参数以支持测试中的时间控制。 |
| **中等-问题6**：`ICurveTablePresenter` 缺少程序化选中设置能力。 | 在 `ICurveTablePresenter` 中新增 `void SelectCell(int columnIndex, CurveTableRow row)` 和 `void ClearSelection()` 方法。新增 `CurveTableRow` 枚举定义（`X` / `Z`）。 |
| **中等-问题7**：脏标记跟踪的 `IAsyncSaveCapable` 接口未定义但被引用。 | 在「核心抽象」中新增 `IAsyncSaveCapable` 的完整定义：包含 `bool HasUnsavedChanges { get; }`、`Task SaveAsync()`、`event EventHandler? UnsavedChangesChanged`。在 `CalibrationEditorBase<T>` 职责中补充脏标记状态栏提示逻辑。 |
| **中等-问题8**：z/y 行头文字在激活状态下的加粗/颜色变化需求未响应。 | 在「高亮叠加策略」中补充第四层视觉反馈：当前行被选中时 y 列文字加粗或变色；当前 z 切片激活时 z 列文字加粗或变色。明确通过 `DataGridCell` 样式选择器或 `LoadingRow` 事件中对行头单元格附加样式类实现。 |
| **轻微-问题9**：`CubeEditor` 初始默认 z 切片语义未定义。 | 在 `ZSliceActivationTracker`「职责」中明确默认激活策略为 `ZIndex = 0`（第一个 z 切片）。在「变量切换流程」中补充 CubeEditor 变量切换时"重置 ZSliceActivationTracker 为默认 z 切片（ZIndex = 0）"。 |
| **轻微-问题10**：`ITableDataSource.GetXIndexFromColumn` 的列索引语义不明确。 | 在 `ITableDataSource<TRow>` 关键成员中明确 `columnIndex` 为 DataGrid 的原始列索引（包含所有行头列），由实现方内部处理偏移映射。 |
| **轻微-问题11**：键盘跨 z 切片导航时表格与 3D 视图的视觉不一致未处理。 | 在「z 切片激活与 3D 视图联动」中明确：此不一致在标定工具场景下属于可接受的轻微延迟；若需缓解，可在防抖期间为 3D 视图区域施加半透明遮罩（30% 透明度灰色覆盖），提示"正在加载"，防抖结束后移除遮罩并刷新图表。 |
| **轻微-问题12**：3D 曲面图数据源切换动画未给出决策结论。 | 在设计决策 5 中补充明确决策：采用直接瞬时刷新（与表格即时响应保持一致），不引入过渡动画；轻量淡入淡出过渡动画作为可选增强在阶段 2 评估引入。 |
| **审查报告-问题1**：图表抽象层 `IChartPresenter.AttachTo(Control host)` 与"无 UI 框架依赖"声明矛盾。 | 将 `IChartPresenter` 拆分为平台无关的基接口（`Refresh()` + `RenderFailed`）和 `IAvaloniaChartPresenter : IChartPresenter`（含 `AttachTo(Control host)`）。`ILineChartPresenter` / `ISurfaceChartPresenter` 继承自 `IAvaloniaChartPresenter`。在「模块职责」中修正图表抽象层声明：基接口无 UI 依赖，Avalonia 特定契约下沉到子接口。 |
| **审查报告-问题2**：`IAsyncSaveCapable` 纯标记接口无法支撑脏标记跟踪功能。 | 将 `IAsyncSaveCapable` 从纯标记接口扩展为含成员的完整契约接口：新增 `bool HasUnsavedChanges { get; }`、`Task SaveAsync()`、`event EventHandler? UnsavedChangesChanged`。在「变更反馈机制」中更新脏标记和状态栏提示的数据来源为 `HasUnsavedChanges`。 |
| **审查报告-轻微**：`CubeRowData` 的 x 列值暴露方式中"动态属性"在 `readonly struct` 语境下模糊。 | 将 `MapRowData` / `CubeRowData` 的 x 列值暴露方式明确为「索引器」（如 `double this[int index] { get; }`），删除"动态属性"的表述。 |
| **审查报告-轻微**：增量刷新方案中接口返回 `IReadOnlyList<TRow>`，调用方无法触发单元素替换。 | 在 `ITableDataSource<TRow>` 中新增 `SupportsIncrementalRefresh` 属性，为增量刷新提供明确的契约支撑。若返回 `true`，实现方应同时实现 `INotifyCollectionChanged`。 |
| **审查报告-轻微**：`ZSliceActivationTracker` 的防抖实现未提及时间抽象，不利于单元测试。 | 在 `ZSliceActivationTracker` 中补充 `TimeProvider` 构造函数参数（.NET 8+），默认使用 `TimeProvider.System`，测试中注入 `FakeTimeProvider` 以快速推进防抖延迟。 |
| **审查报告-轻微**：`ZSliceActivationTracker` 缺少输入方法定义。 | 在 `ZSliceActivationTracker` 关键成员中补充 `OnSelectionChanged(CubeRowData? row)` 和 `OnZColumnClicked(CubeRowData row)` 输入方法，明确 `CubeEditor` 向 Tracker 传递选中变更和 z 列点击的公共 API。 |
| **审查报告-轻微**：`ITableDataSource` 未暴露列生成所需的元数据。 | 在 `ITableDataSource<TRow>` 中新增 `IReadOnlyList<string> ColumnHeaders { get; }` 和 `int DataColumnCount { get; }`，使表格适配层更完整地封装 DataGrid 的列生成需求。 |
