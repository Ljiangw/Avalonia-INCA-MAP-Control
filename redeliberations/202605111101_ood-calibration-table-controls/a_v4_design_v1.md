# Avalonia 标定数据表格编辑控件 — 架构级 OOD 设计方案 v7

## 概述

本方案设计一组基于 Avalonia V12 的标定数据表格编辑控件，面向汽车 ECU 标定工具场景。在统一交互范式下支撑 1 维（曲线）、2 维（Map）、3 维（数据立方体）三种标定数据的可视化编辑，并将控件与后端数据模型、图表渲染引擎解耦，确保可复用性和可替换性。

相对于 v6 方案，v7 的核心变化集中在以下方面：

1. **明确 CubeEditor z 列展示方案为需求折中**：在「设计决策2」开头增加显式声明，说明因 Avalonia DataGrid V12 原生不支持 `RowSpan` 式跨行单元格，"合并单元格"需求降级为"每行重复显示 + 视觉分组线"方案，并补充未来升级路径。
2. **修复 `ICurveTablePresenter` 接口契约与职责描述不一致**：补充列宽调整相关契约成员（`MinColumnWidth`、`ColumnWidthChanged` 事件及事件参数），使接口成员与"暴露列宽调整能力"的职责描述对齐。
3. **正式定义 `CalibrationEditorBase<T>` 模板部件**：补充模板部件表格（部件名称、预期类型、角色）及缺失时的降级行为说明。
4. **明确 `IMapData.GetValueMatrix()` / `ICubeData.GetSliceMatrix()` 返回数组的可变性语义**：约定返回数组为只读视图或深拷贝，调用方不应修改；若需修改，应通过数据模型写接口进行。
5. **补全编辑模式下方向键行为**：明确编辑模式下方向键在 `TextBox` 内部移动光标（不退出编辑），Tab/Enter 确认并退出编辑，Esc 取消编辑。
6. **定义空状态默认视觉呈现**：补充基类默认空状态行为（主内容区中央显示"无有效数据"占位文本，灰色调，居中），子类可覆盖。
7. **补充数值输入解析策略**：统一使用 `CultureInfo.InvariantCulture` + `NumberStyles.Float` 解析；千分位和本地化小数点不纳入支持；`double` 溢出输入视为解析失败。
8. **明确 `IAsyncSaveCapable.SaveAsync()` 异常语义**：保存失败时抛出异常；控件层 try/catch 捕获后显示错误提示但不阻断编辑；`UnsavedChangesChanged` 仅在 `HasUnsavedChanges` / `UnsavedChangeCount` 实际值变化时触发。
9. **定义键盘 Tab 导航跨 z 切片边界行为**：明确采用"自然延续到下一 z 切片第一行"策略，与 DataGrid 默认行为一致。
10. **补充 `ISurfaceChartPresenter` 轴标签设置契约**：增加 `SetAxisLabels` 方法，接收各轴名称及单位信息。
11. **补充 `ActiveZSliceChangedEventArgs` z 轴实际值**：增加 `OldZValue` / `NewZValue` 属性。
12. **参数化 `ZSliceActivationTracker` 防抖延迟**：构造函数增加 `TimeSpan debounceDelay` 参数，默认值为 150ms。
13. **细化精度展示策略后备格式**：明确 `FormatString` 为空且 `DisplayPrecision` 为 0 时的默认后备格式为 `G6`；编辑完成后重新格式化规则与展示格式化策略一致。

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

**职责**：暴露标定变量的元信息（名称、单位、x 轴变量名及单位）以及数据变更通知能力。轴信息属性按接口隔离原则下沉到维度特化子接口，本接口仅保留所有维度均适用的公共元信息。

**关键成员**：
- `string VariableName { get; }` — 变量名称，用于下拉框选项的内部标识
- `string GroupName { get; }` — 变量所属组名（如「曲线变量组」），用于下拉框显示格式的分组前缀
- `string DisplayName { get; }` — 格式化后的显示文本（如 `<曲线变量组> AllMkn_n_AirMax`），直接作为下拉框选项的展示内容；若后端不提供，控件层可按 `[{GroupName}] {VariableName}` 自动合成
- `string Unit { get; }` — 变量单位（如 `mgpl`）
- `string XAxisName { get; }`、`string XAxisUnit { get; }` — x 轴变量名及单位（所有维度均适用）
- `int DisplayPrecision { get; }` — 默认显示精度（小数位数），作为 `FormatString` 的后备策略
- `string FormatString { get; }` — 数值格式字符串（如 `F3`、`G6`），优先于 `DisplayPrecision` 使用；若两者同时存在，`FormatString` 优先，`DisplayPrecision` 仅在 `FormatString` 为空时生效
- 继承 `INotifyPropertyChanged` — 强制后端实现方具备数据变更通知能力

**元素级变更通知约定**：
`ICalibrationData` 的子接口（`ICurveData`、`IMapData`、`ICubeData`）声明的元素级写方法（`SetXValue`、`SetZValue`、`SetValue` 等）在成功修改数据后应触发 `PropertyChanged` 事件。考虑到单次写操作可能影响的数据范围不确定（如修改 x 轴值可能同时影响图表和表格列头），约定属性名使用 `string.Empty`（表示所有属性均可能变更），由控件层在收到通知后执行全量或增量刷新。若后端实现方能精确知道变更范围，也可触发具体属性名，但控件层不依赖此精确性。

**协作**：被 `ICurveData`、`IMapData`、`ICubeData` 继承；被 `CalibrationEditorBase<T>` 作为泛型约束使用；被表格适配层、图表抽象层和 Curve 表格契约层消费。

**类型形态**：接口，**明确继承 `INotifyPropertyChanged`**。具体数据模型由后端标定系统定义，控件组只消费契约，不拥有实现。

### `ICurveData` / `IMapData` / `ICubeData`（接口）

**角色**：维度特化的数据契约。

**职责**：在 `ICalibrationData` 基础上，各自声明维度特定的数据访问方式。轴信息属性按维度实际需求声明，避免一维模型被迫暴露无关轴属性。

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
- `string YAxisName { get; }`、`string YAxisUnit { get; }` — y 轴变量名及单位（从 `ICalibrationData` 下沉至此）
- `double[,] GetValueMatrix()` — 返回二维值矩阵的**只读视图或深拷贝**，供 `ISurfaceChartPresenter.LoadMapData` 直接消费，避免调用方逐点遍历构建数组的 O(n) 开销。**调用方不应修改返回的数组**；若需修改数据，应通过 `SetValue` 写接口进行。后端实现方可选择返回内部数组的只读包装（如通过 `Array.AsReadOnly` 或自定义只读包装器）或执行深拷贝，但无论哪种方式，都应确保调用方修改不会直接影响后端数据模型的一致状态

`ICubeData`：
- `int XAxisLength { get; }`、`int YAxisLength { get; }`、`int ZAxisLength { get; }` — 各轴维度长度
- `double GetValue(int xIndex, int yIndex, int zIndex)` — 读取三维张量指定位置的值
- `void SetValue(int xIndex, int yIndex, int zIndex, double value)` — 更新三维张量指定位置的值；更新完成后触发 `PropertyChanged`，属性名使用 `string.Empty`
- `IReadOnlyList<double> XValues { get; }` — x 轴坐标值序列
- `IReadOnlyList<double> YValues { get; }` — y 轴坐标值序列
- `IReadOnlyList<double> ZValues { get; }` — z 轴坐标值序列
- `string YAxisName { get; }`、`string YAxisUnit { get; }` — y 轴变量名及单位（从 `ICalibrationData` 下沉至此）
- `string ZAxisName { get; }`、`string ZAxisUnit { get; }` — z 轴变量名及单位（从 `ICalibrationData` 下沉至此）
- `double[,] GetSliceMatrix(int zIndex)` — 返回指定 z 切片的二维值矩阵（x, y）的**只读视图或深拷贝**，供 `ISurfaceChartPresenter.LoadSliceData` 直接消费。**调用方不应修改返回的数组**；若需修改数据，应通过 `SetValue` 写接口进行。后端实现方可选择返回内部数组的只读包装或执行深拷贝，确保调用方修改不会直接影响后端数据模型的一致状态

**协作**：作为 `CalibrationEditorBase<T>` 的泛型参数具体化类型；被各自的表格适配器或 Curve 表格呈现器消费。

**类型形态**：接口，继承自 `ICalibrationData`。不采用抽象类，因为后端数据模型可能已经继承自其他领域基类，接口避免单继承限制。轴信息属性的下沉遵循接口隔离原则：一维曲线数据不需要知道 y/z 轴，二维 Map 数据不需要知道 z 轴。

### `IAsyncSaveCapable`（接口）

**角色**：标识后端数据模型支持异步持久化保存，并为脏标记跟踪提供状态查询能力。

**职责**：
- 提供未保存变更状态的具体数量查询能力，支撑控件层的脏标记视觉反馈和状态栏未保存数量提示
- 提供异步保存触发入口，供控件层在用户请求保存时调用

**关键成员**：
- `bool HasUnsavedChanges { get; }` — 是否存在尚未持久化的变更；控件层据此决定是否显示脏标记（橙色边框）
- `int UnsavedChangeCount { get; }` — 未保存变更的具体数量（如 3），供信息栏显示 "3 处未保存"；若后端无法提供精确数量，可返回 `HasUnsavedChanges ? 1 : 0` 作为降级策略
- `Task SaveAsync()` — 异步触发保存操作。保存成功后 `HasUnsavedChanges` 应返回 `false`，`UnsavedChangeCount` 应返回 0。**保存失败时抛出异常**（具体异常类型由后端实现决定，如 `InvalidOperationException` 或自定义保存异常），控件层通过 try/catch 捕获后在信息栏显示错误提示文本，不阻断用户继续编辑操作。保存过程中发生的异常不应导致控件进入不可恢复状态
- `event EventHandler? UnsavedChangesChanged` — 未保存状态变更事件。**仅在 `HasUnsavedChanges` 或 `UnsavedChangeCount` 的实际值发生变化时触发**。例如：用户编辑单元格导致 `UnsavedChangeCount` 从 2 变为 3 时应触发；保存成功导致两者归零时应触发；重复设置相同值时不应触发。供控件层实时响应（如用户在外部触发保存后自动清除脏标记）

**协作**：由后端数据模型选择性实现；被 `CalibrationEditorBase<T>` 在检测到数据模型实现此接口时启用脏标记跟踪和状态栏提示。若数据模型未实现此接口（即时持久化模式），控件层自动禁用脏标记相关反馈。

**类型形态**：接口，独立于 `ICalibrationData` 继承树。不强制所有数据模型实现，保持向后兼容性。

### `CalibrationEditorBase<T>`（抽象类）

**角色**：三个标定编辑控件的公共基座。

**职责**：
- 定义并管理 Avalonia 依赖属性：`ItemsSource`（`IList<T>` 类型，绑定到变量选择下拉框）、`SelectedVariable`（当前选中的标定数据项）
- 提供统一的顶部信息栏模板结构（左侧 `ComboBox`、右侧单位/轴信息展示）
- 处理变量切换时的选中变更事件，向子类发出数据切换通知
- 信息栏文本格式化：默认格式为 `[{Unit}] x: {XAxisName} [{XAxisUnit}]`。由于 y/z 轴信息已从 `ICalibrationData` 下沉到维度子接口，信息栏格式化模板通过运行时类型检查动态构建：若 `T` 实现了 `IMapData`，追加 `y: {YAxisName} [{YAxisUnit}]`；若 `T` 实现了 `ICubeData`，再追加 `z: {ZAxisName} [{ZAxisUnit}]`。子类可通过覆盖受保护的虚方法 `FormatInfoText(T variable)` 自定义模板
- 空状态管理：当 `ItemsSource` 为 `null`、空列表、`SelectedVariable` 为 `null`，或 `SelectedVariable` 有效但其内部数据维度为零（如 `ICurveData.Length == 0`、`IMapData.XAxisLength == 0`、`ICubeData.ZAxisLength == 0`）时，进入空状态展示。基类默认行为为：在主内容区中央渲染"无有效数据"占位文本，灰色调（如 `#808080`），水平垂直居中，字体大小与信息栏一致。提供受保护的虚方法 `OnEnterEmptyState()` / `OnExitEmptyState()` 供子类覆盖以自定义空状态视觉呈现
- 脏标记状态栏提示：当数据模型实现 `IAsyncSaveCapable` 时，订阅其 `UnsavedChangesChanged` 事件，在信息栏区域显示未保存数量（如 "3 处未保存"），数据源为 `IAsyncSaveCapable.UnsavedChangeCount`。若 `UnsavedChangeCount` 只能提供布尔降级（返回 0 或 1），则显示 "存在未保存变更"

**模板部件（Template Parts）**：

`CalibrationEditorBase<T>` 通过 Avalonia `TemplatedControl` 机制约定以下模板部件，ControlTemplate 中应包含对应名称的元素：

| 部件名称 | 预期类型 | 角色 | 缺失时降级行为 |
|---------|---------|------|--------------|
| `PART_VariableSelector` | `ComboBox` | 变量选择下拉框，绑定到 `ItemsSource` 和 `SelectedVariable` | 静默跳过变量选择器初始化；控件仍可运行，但用户无法通过 UI 切换变量 |
| `PART_InfoPanel` | `Panel` 或 `TextBlock` | 信息展示面板，显示单位、轴信息和未保存状态 | 静默跳过信息栏初始化；控件仍可运行，但顶部信息栏区域为空 |
| `PART_ContentHost` | `Panel` | 主内容区容器，子类将具体编辑器内容（表格、图表）添加至此 | 抛出 `InvalidOperationException`，因为主内容区是控件的核心功能载体，缺失时无法继续 |

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
- `event EventHandler<ColumnWidthChangedEventArgs>? ColumnWidthChanged` — 列宽变更事件，当用户拖拽列分隔线或程序化调整列宽时触发
- `void Refresh()` — 响应外部数据变更通知，刷新显示内容
- `void SelectCell(int columnIndex, CurveTableRow row)` — 程序化选中指定列和行的单元格
- `void ClearSelection()` — 清除当前选中状态
- `double MinColumnWidth { get; set; }` — 最小列宽约束（如 60px），防止数据量过大时列宽被压缩至不可读

**事件参数类型**：

`CurveCellSelectedEventArgs`（密封类，继承 `EventArgs`）：
- `int ColumnIndex { get; }` — 选中单元格所在的列索引
- `CurveTableRow Row { get; }` — 选中单元格所在的行标识（X 或 Z）

`CurveCellEditCompletedEventArgs`（密封类，继承 `EventArgs`）：
- `int ColumnIndex { get; }` — 编辑完成的列索引
- `CurveTableRow Row { get; }` — 编辑完成的行标识（X 或 Z）
- `double NewValue { get; }` — 用户输入并解析后的新数值

`ColumnWidthChangedEventArgs`（密封类，继承 `EventArgs`）：
- `int ColumnIndex { get; }` — 列宽发生变更的列索引
- `double NewWidth { get; }` — 变更后的列宽（像素值）

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
- `IReadOnlyList<TRow> Rows { get; }` — 纵向行集合的只读视图，可直接绑定到 `DataGrid.ItemsSource`
- `void ReplaceRow(int index, TRow newRow)` — 增量刷新的显式契约入口。由实现方内部执行集合替换（如在 `ObservableCollection<TRow>` 中替换指定索引处的元素）并触发 `INotifyCollectionChanged`。调用方（Editor）在检测到数据模型变更且 `SupportsIncrementalRefresh` 为 `true` 时调用此方法，避免全量重建行集合
- `INotifyCollectionChanged? CollectionChangedNotifier { get; }` — 集合变更通知器引用。当 `SupportsIncrementalRefresh` 为 `true` 时返回非 null 实例（通常与 `Rows` 的底层集合为同一对象）；为 `false` 时返回 `null`。Editor 无需运行时强制转换即可订阅集合变更事件
- `int GetXIndexFromColumn(int columnIndex)` — 将 DataGrid 的原始列索引（包含所有行头列）映射为原始数据的 x 轴索引（供列头生成和编辑回写使用）。实现方内部处理行头列的偏移映射
- `void Refresh()` — 响应外部原始数据模型的变更通知，重新生成 `Rows` 行集合；Editor 在检测到数据模型 `PropertyChanged` 后调用此方法，随后重新设置 `DataGrid.ItemsSource` 以刷新表格显示
- `IReadOnlyList<string> ColumnHeaders { get; }` — x 轴数据列的列头文本集合（不包含行头列），长度等于数据列数，供 Editor 在变量切换时动态生成 `DataGrid` 列
- `int DataColumnCount { get; }` — x 轴数据列的数量（不包含行头列）
- `bool SupportsIncrementalRefresh { get; }` — 标识当前实现是否支持增量刷新。若返回 `true`，`CollectionChangedNotifier` 返回非 null 通知器实例，Editor 可通过 `ReplaceRow` 实现行级增量刷新

**协作**：被 `MapEditor` 和 `CubeEditor` 内部使用，作为 `DataGrid` `ItemsSource` 的实际来源。编辑完成时，Editor 从强类型行数据对象中提取坐标索引属性，调用原始数据模型的写接口。非编辑路径下的数据变更（如后端标定数据从 ECU 更新后）由 Editor 订阅原始模型的 `PropertyChanged`，再调用 `Refresh()` 重新生成行集合并更新 `ItemsSource`。

**类型形态**：泛型接口（`ITableDataSource<TRow> where TRow : struct`），分别实例化为 `ITableDataSource<MapRowData>` 和 `ITableDataSource<CubeRowData>`。泛型约束 `TRow : struct` 确保行数据为值类型，避免堆分配。

### `MapRowData` / `CubeRowData`（`readonly struct`）

**角色**：承载一行纵向表格数据的展示值及其在原始多维数据中的索引坐标。供 `MapEditor` 与 `CubeEditor` 使用。

**职责**：
- `MapRowData`：内联 `YIndex` 属性，标识该行在二维矩阵中的 y 坐标；承载 `YValue` 文本（用于 y 轴行头展示）。x 列数据值通过 `IReadOnlyList<double> Values { get; }` 按列顺序暴露
- `CubeRowData`：内联 `ZIndex`、`YIndex` 属性，标识该数据行在三维张量中的坐标；承载 `ZValue` 和 `YValue` 文本（用于 z/y 双列行头展示）。x 列数据值通过 `IReadOnlyList<double> Values { get; }` 按列顺序暴露，列索引与 x 轴索引一一映射

**协作**：由对应维度的表格适配器生成并填充，作为 `DataGrid` 的 `ItemsSource` 行集合。Editor 在编辑确认时从行数据对象中直接读取坐标索引属性，回写到原始数据模型。

**类型形态**：`readonly struct`。值类型避免堆分配，坐标索引作为结构体的属性直接内联，不通过接口引用传递，消除接口引用传递中的装箱开销。x 列数据通过 `IReadOnlyList<double> Values { get; }` 暴露（而非索引器），使 DataGrid 动态列的绑定路径为 `Values[0]`、`Values[1]` 等属性索引器语法，在 Avalonia V12 CompiledBinding 中的兼容性优于直接索引器绑定。`DataGrid` 将行数据赋给 `DataGridRow.DataContext`（`object` 类型属性）时仍会发生一次 boxing，此机制由 Avalonia 框架决定，无法避免；但行数据本身避免了堆分配和接口引用的额外装箱，总体 GC 压力仍显著低于 `class` 方案。

### `IChartPresenter`（接口）

**角色**：图表渲染的抽象契约，平台无关的基接口。

**职责**：定义图表渲染的最小通用能力（刷新、渲染失败通知），不依赖任何 UI 框架类型。Avalonia 特定的宿主关联能力下沉到 `IAvaloniaChartPresenter` 子接口。

**关键成员**：
- `void Refresh()` — 显式触发重绘
- `event EventHandler<ChartRenderFailedEventArgs> RenderFailed` — 渲染失败通知事件，参数携带降级原因描述，供宿主控件展示错误占位

**事件参数类型**：

`ChartRenderFailedEventArgs`（密封类，继承 `EventArgs`）：
- `string Reason { get; }` — 渲染失败的原因描述（如 "GPU 不可用"、"数据格式异常"）
- `bool IsRecoverable { get; }` — 失败是否可恢复（如数据格式异常在数据修正后可恢复，GPU 驱动崩溃可能不可恢复）

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
- `void LoadMapData(double[,] data, IReadOnlyList<double> xValues, IReadOnlyList<double> yValues)` — 加载二维 Map 数据（MapEditor 使用）。`double[,]` 参数由 `IMapData.GetValueMatrix()` 直接提供，避免 Editor 逐点遍历构建数组
- `void LoadSliceData(double[,] sliceData, IReadOnlyList<double> xValues, IReadOnlyList<double> yValues)` — 加载激活 z 切片的二维矩阵数据（CubeEditor 使用）。`double[,]` 参数由 `ICubeData.GetSliceMatrix(zIndex)` 直接提供
- `void SetAxisLabels(string xName, string xUnit, string yName, string yUnit, string valueName, string valueUnit)` — 设置曲面图各轴的名称和单位。x/y 轴为水平坐标轴，value 轴为垂直高度轴。在 `LoadMapData`/`LoadSliceData` 之后调用，确保图表在展示数据的同时正确标注轴信息

**协作**：`CurveEditor` 消费 `ILineChartPresenter`；`MapEditor` 消费 `ISurfaceChartPresenter`（加载完整 Map 数据）；`CubeEditor` 消费 `ISurfaceChartPresenter`（加载当前激活 z 切片的二维矩阵）。

**类型形态**：接口，继承自 `IAvaloniaChartPresenter`。拆分为两个特化接口，是因为折线图和 3D 曲面图的数据加载方式、交互反馈（如数据点高亮）差异显著，单一接口会导致大量不适用的成员。两个加载方法均统一为接收原始数组的形式，由数据模型直接提供矩阵数据，消除调用方的 O(n) 转换开销。

### `ZSliceActivationTracker`（类）

**角色**：CubeEditor 中 z 切片激活状态的专门管理者。

**职责**：
- 维护当前激活的 z 值索引；默认激活策略为 `ZIndex = 0`（第一个 z 切片），但仅在 `ZAxisLength > 0` 时生效。若 `ZAxisLength == 0`，`ActiveZIndex` 保持为 `-1`（表示无有效切片）
- 接收 `CubeEditor` 传递的表格选中变更事件（`OnSelectionChanged(CubeRowData? row)`），从 `CubeRowData` 中提取 `ZIndex`，判断是否需要切换激活切片。传入 `null` 时语义为"保持当前激活切片不变"，Tracker 不执行任何状态变更
- 接收 z 列点击事件（`OnZColumnClicked(CubeRowData row)`）：将该单元格所属行的 z 值设为激活切片（经过防抖），然后通知 `CubeEditor` 激活切片已变更。**Tracker 不再负责"选中该切片内的第一个数据单元格"**——该行为由 `CubeEditor` 在调用 `OnZColumnClicked` 后的后续逻辑中自行处理
- 提供激活切片变更通知（防抖结束后触发，供 3D 图表刷新和 z 切片级高亮使用）
- 处理键盘快速跨 z 切片导航时的防抖/延迟策略（若数据量大）。防抖期间，表格立即响应导航并更新行选中，3D 曲面图等待防抖结束后刷新

**关键成员**：
- `void OnSelectionChanged(CubeRowData? row)` — 输入方法：表格当前选中行变更时由 `CubeEditor` 调用；`null` 表示无选中项，语义为保持当前激活切片不变
- `void OnZColumnClicked(CubeRowData row)` — 输入方法：用户点击 z 列单元格时由 `CubeEditor` 调用；Tracker 将该行的 z 值设为激活切片，经防抖后触发 `ActiveSliceChanged`
- `int ActiveZIndex { get; }` — 当前激活的 z 值索引；若 `ZAxisLength == 0`，返回 `-1`
- `event EventHandler<ActiveZSliceChangedEventArgs> ActiveSliceChanged` — 激活切片变更事件。参数携带旧 `ZIndex` 和新 `ZIndex`。事件仅在防抖计时结束后（用户停止快速导航）触发，而非每次潜在变更都触发，以避免频繁通知 3D 图表重绘
- `TimeProvider TimeProvider { get; }` — 时间提供器（.NET 8+），默认使用 `TimeProvider.System`；测试中可注入 `FakeTimeProvider` 以快速推进防抖延迟
- `TimeSpan DebounceDelay { get; }` — 防抖延迟时间，默认值为 `TimeSpan.FromMilliseconds(150)`。在构造函数中通过可选参数 `TimeSpan debounceDelay` 传入，调用方可根据数据量和性能需求调整。测试中可注入较小的延迟（如 `TimeSpan.Zero`）以禁用防抖
- `void ResetToDefault()` — 显式重置激活切片为默认值（`ZIndex = 0`，若 `ZAxisLength > 0`；否则 `-1`），供 `CubeEditor` 在变量切换时调用

**协作**：由 `CubeEditor` 实例化并注入；向下消费表格适配器提供的 `CubeRowData` 行数据（通过 `ZIndex` 属性按 z 值分组）；向上通过纯 .NET 事件（`EventHandler<T>`）通知 `CubeEditor`。`CubeEditor` 在事件处理器中通过 `Dispatcher.UIThread.Post` 将状态变更调度到 UI 线程，确保 Tracker 不依赖 Avalonia 视觉类型，保持可测试性。

**类型形态**：独立类（非静态）。将这段逻辑从 `CubeEditor` 中剥离，使控件类聚焦模板和交互路由，状态映射逻辑可独立单元测试。

### `ActiveZSliceChangedEventArgs`（类）

**角色**：`ZSliceActivationTracker.ActiveSliceChanged` 事件的事件参数。

**职责**：携带 z 切片变更前后的索引值及对应的实际 z 轴数值。

**关键成员**：
- `int OldZIndex { get; }` — 变更前的 z 切片索引
- `int NewZIndex { get; }` — 变更后的 z 切片索引
- `double OldZValue { get; }` — 变更前的 z 轴实际数值（由 `ICubeData.ZValues[OldZIndex]` 提供）
- `double NewZValue { get; }` — 变更后的 z 轴实际数值（由 `ICubeData.ZValues[NewZIndex]` 提供）

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
    ├── ItemsSource 非空，SelectedVariable 有效，但数据维度为零
    │   ├── ICurveData.Length == 0
    │   ├── IMapData.XAxisLength == 0 或 YAxisLength == 0
    │   └── ICubeData.ZAxisLength == 0（或 XAxisLength == 0 或 YAxisLength == 0）
    │       └── [触发 OnEnterEmptyState] ──→ [显示空状态占位]
    │
    └── ItemsSource 非空，SelectedVariable 有效，且数据维度均大于零
        └── [触发 OnExitEmptyState] ──→ [触发 OnSelectedVariableChanged]
                                        │
                                        ├── CurveEditor: 通知 ICurveTablePresenter 重建，通知 ILineChartPresenter 加载
                                        ├── MapEditor: 重建 ITableDataSource，通知 ISurfaceChartPresenter 加载
                                        └── CubeEditor: 重建 ITableDataSource，调用 ZSliceActivationTracker.ResetToDefault()，通知 ISurfaceChartPresenter 加载
```

### 单元格编辑与数据回写

#### 编辑触发方式

表格数据单元格采用 **双击进入编辑** 模式，单击仅用于选中/导航。行头列（z 列、y 列）不参与编辑，单击仅触发选中或 z 切片激活。

#### 数值输入解析策略

所有控件的单元格数值输入采用统一的解析规则：
- **解析方法**：使用 `double.TryParse(text, NumberStyles.Float, CultureInfo.InvariantCulture, out double result)` 进行解析
- **支持的格式**：标准十进制表示（如 `1.5`、`-3.14`）、科学计数法（如 `1.23e-4`）
- **不支持的格式**：千分位分隔符（如 `1,000.5`）、本地化小数点（如 `1,5` 中的逗号作为小数点）。输入包含逗号时视为解析失败
- **溢出处理**：输入值超出 `double` 表示范围（如 `1e309`）时，`TryParse` 返回 `false`，视为解析失败，触发就地验证错误反馈（红色边框）
- **空输入**：空字符串或仅包含空白字符的输入视为解析失败
- **编辑模式初始值**：单元格进入编辑模式时，`TextBox` 的初始文本按当前精度策略格式化（`FormatString` 优先，`DisplayPrecision` 后备；两者均为空时默认使用 `G6`）
- **编辑完成后的重新格式化**：输入确认后，控件层调用数据模型写接口更新原始数据，然后由数据变更通知触发 UI 刷新；刷新后的单元格显示值按相同精度策略重新格式化（与编辑前展示格式一致）

#### MapEditor / CubeEditor（DataGrid 纵向行模型）

用户触发单元格进入编辑模式（双击）→ 单元格展示编辑模板（如 `TextBox`）→ 用户输入数值 → 输入完成时（Enter/失去焦点），Editor 拦截 `CellEditEnding` 事件（`DataGridCellEditEndingEventArgs` 提供 `Column`、`Row`、`EditingElement`、`EditAction` 属性），手动解析并验证输入值：

1. **验证失败**：通过设置编辑模板内 `TextBox` 的 `BorderBrush` 和 `ToolTip` 实现红色边框就地反馈，阻止该次编辑回写，但不阻断用户继续编辑其他单元格
2. **验证通过**：
   - 从对应类型的行数据对象中提取坐标索引属性
   - 调用 `ITableDataSource<TRow>.GetXIndexFromColumn(columnIndex)` 将 DataGrid 原始列索引（含行头列）映射为 x 轴索引
   - 调用原始数据模型的写方法更新对应位置
   - 设置 `e.Cancel = true` 取消 DataGrid 的默认提交（因为 `readonly struct` 行数据无法通过标准绑定机制写回）
   - 模型触发 `INotifyPropertyChanged.PropertyChanged`（属性名为 `string.Empty`）
   - Editor 订阅到该通知后，根据 `ITableDataSource<TRow>.SupportsIncrementalRefresh` 选择刷新策略：
     - 若支持增量刷新：调用 `ReplaceRow(int index, TRow newRow)` 替换对应行，由实现方内部触发 `INotifyCollectionChanged`，DataGrid 仅重绘该行
     - 若不支持增量刷新：调用 `ITableDataSource<TRow>.Refresh()` 重新生成行集合，然后重新设置 `DataGrid.ItemsSource`
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
- **状态栏提示**：`CalibrationEditorBase<T>` 的信息栏区域在存在未保存修改时显示未保存数量（如 "3 处未保存"），数据源为 `IAsyncSaveCapable.UnsavedChangeCount`。若后端只能提供布尔降级，则显示 "存在未保存变更"。启用条件与脏标记视觉反馈一致
- **模型变更通知**：数据模型写接口触发 `PropertyChanged`（属性名 `string.Empty`）后，Editor 调用 `ITableDataSource<TRow>.Refresh()`（MapEditor/CubeEditor）或 `ICurveTablePresenter.Refresh()`（CurveEditor）刷新单元格显示，作为最基础的变更确认

### 键盘导航行为

#### 通用导航规则（适用于所有三个控件）

- **方向键（↑/↓/←/→）**：
  - 在**浏览模式**（非编辑模式）下：在单元格之间移动当前活动单元格（`CurrentCell`）。`MapEditor` 和 `CubeEditor` 中由 DataGrid 内置导航处理；`CurveEditor` 中由 `ICurveTablePresenter` 实现方处理
  - 在**编辑模式**下（`TextBox` 获得焦点时）：方向键在 `TextBox` 内部移动文本光标（←/→ 移动字符位置，↑/↓ 在多行文本中移动行位置），**不退出编辑模式，不移动单元格焦点**
- **Tab**：
  - 在浏览模式下：按视觉顺序切换到下一个可编辑单元格；到达行末时自动换到下一行的首个数据单元格。`Shift+Tab` 反向导航
  - 在编辑模式下：确认当前输入并退出编辑模式，焦点移动到下一个可编辑单元格（与浏览模式下 Tab 的目标单元格一致）
- **Enter**：
  - 在浏览模式（非编辑模式）下：进入当前活动单元格的编辑模式
  - 在编辑模式下：确认输入并退出编辑模式，焦点保持在当前单元格
- **Esc**：在编辑模式下取消当前编辑并退出编辑模式，恢复单元格原始值，焦点保持在当前单元格

#### CubeEditor 跨 z 切片导航的特殊行为

- 用户使用方向键（↑/↓）或 Tab 在 `CubeEditor` 的平铺表格中导航时，DataGrid 的 `CurrentCell` 即时响应键盘输入而切换
- **Tab 跨 z 切片边界行为**：当用户位于某个 z 切片的最后一个数据单元格（该行最后一个 x 列）按 Tab 时，焦点自然延续到下一 z 切片的第一行第一个数据单元格；`Shift+Tab` 反向导航时，从当前 z 切片第一行第一个数据单元格移入上一 z 切片的最后一行最后一个数据单元格。此行为与 DataGrid 默认导航行为一致，无需额外干预
- 当用户从当前 z 切片的最后一行 y 值按 ↓ 移入下一 z 切片的第一行 y 值时（或按 ↑ 从当前 z 切片的第一行 y 值移入上一 z 切片的最末行 y 值时），`CubeEditor` 将新的选中行通过 `OnSelectionChanged` 传递给 `ZSliceActivationTracker`
- Tracker 读取新行的 `ZIndex`，若与当前激活 `ZIndex` 不同，则启动防抖计时（100–200ms）。**防抖期间表格的 `CurrentCell` 和行级高亮立即更新**，仅 3D 曲面图等待防抖结束后刷新
- 用户持续快速按方向键跨 z 切片导航时，Tracker 内部的防抖计时器被连续重置，直到用户停止导航且防抖计时结束后，才真正触发 `ActiveSliceChanged` 事件
- `CubeEditor` 在 `ActiveSliceChanged` 事件处理器中通知 `ISurfaceChartPresenter` 切换数据源，同时更新 z 切片级高亮样式
- 防抖期间 3D 视图区域与表格的视觉不一致处理：可在防抖期间为 3D 视图区域施加半透明遮罩（如 30% 透明度灰色覆盖），提示"正在加载"，防抖结束后移除遮罩并刷新图表

#### ZSliceActivationTracker 与键盘导航的交互细节

- 每次 `CurrentCell` 因键盘导航而变更时，`CubeEditor` 获取新 `CurrentCell` 所在行的 `CubeRowData`，调用 `OnSelectionChanged(row)`
- 若新行的 `ZIndex` 与当前激活 `ZIndex` 相同，Tracker 不启动防抖，不触发事件
- 若新行的 `ZIndex` 不同，Tracker 启动/重置防抖计时器，但不立即变更 `ActiveZIndex`
- 防抖计时结束后，Tracker 更新 `ActiveZIndex` 并触发 `ActiveSliceChanged`
- 若用户在导航过程中释放了方向键且防抖已结束，3D 曲面图刷新为最终停止位置对应的 z 切片
- `OnSelectionChanged(null)` 的调用场景（如用户按 Esc 清除选中）：语义为"保持当前激活切片不变"，Tracker 不执行任何状态变更

### z 切片激活与 3D 视图联动（CubeEditor）

用户点击/键盘导航到表格中的某个单元格或行 → `CubeEditor` 将选中的 `CubeRowData` 通过 `OnSelectionChanged` 传递给 `ZSliceActivationTracker` → Tracker 读取该行的 `ZIndex` 属性，与当前激活 z 比较：若相同则无动作；若不同则启动防抖计时 → 防抖结束后（用户停止快速导航），Tracker 更新激活 z 值并通过 `ActiveSliceChanged` 事件发出变更通知 → `CubeEditor` 在事件处理器中通过 `Dispatcher.UIThread.Post` 调度 UI 更新：通知 `ISurfaceChartPresenter` 切换数据源到新的 z 切片矩阵，同时触发表格的 z 切片级高亮样式更新。

当用户点击 z 列的任意单元格时，`CubeEditor` 调用 `ZSliceActivationTracker.OnZColumnClicked(row)`，Tracker 将该单元格所属行的 z 值设为激活切片（经过防抖），然后通过 `ActiveSliceChanged` 事件通知变更。**`CubeEditor` 在调用 `OnZColumnClicked` 之后，于 `ActiveSliceChanged` 事件处理器中（或直接在其后的同步代码中）自行处理"选中该切片内的第一个数据单元格"逻辑**——该行为不再是 Tracker 的职责。具体实现为：在事件处理器中计算该 z 切片的第一行 y 值对应的数据单元格坐标，程序化设置 `DataGrid.CurrentCell` 到该单元格。

键盘跨 z 切片导航时（如从当前 z 切片的最后一行 y 值按方向键移入下一 z 切片的第一行 y 值），表格的 `CurrentCell` / `SelectedItem` 立即响应键盘输入而切换，`ZSliceActivationTracker` 内部引入短时间防抖（如 100–200ms），仅在用户停止快速导航后才通过 `ActiveSliceChanged` 事件真正刷新 3D 曲面图。此期间存在约 100–200ms 的视觉不一致（表格已切换行，3D 视图仍显示旧切片），在标定工具场景下属于可接受的轻微延迟；若需缓解，可在防抖期间为 3D 视图区域施加半透明遮罩（如 30% 透明度灰色覆盖），提示"正在加载"，防抖结束后移除遮罩并刷新图表。

3D 曲面图数据源切换采用**直接瞬时刷新**策略，不引入过渡动画，与表格即时响应保持一致。轻量淡入淡出过渡动画作为可选增强，在阶段 2 评估引入。

### z 切片级高亮状态与 DataGrid 虚拟化联动（CubeEditor）

`DataGrid` 仅对当前可视区域内的行创建视觉树实例（行虚拟化）。`ZSliceActivationTracker` 输出的激活 `ZIndex` 需通过以下机制传递到行视觉元素：

1. **`CubeEditor` 订阅 `ZSliceActivationTracker` 的 `ActiveSliceChanged` 事件**。当激活 `ZIndex` 变化时，`CubeEditor` 执行两项操作：通知 `ISurfaceChartPresenter` 刷新 3D 曲面图；遍历当前已加载的 `DataGridRow` 容器，为属于激活 z 切片的行附加样式类（如 `active-z-slice`），为非激活行移除该样式类。

2. **`CubeEditor` 订阅 `DataGrid` 的 `LoadingRow` 事件**。当用户滚动导致新行进入可视区域时，`DataGrid` 创建对应的行视觉元素（新创建或从重用车池中取出）并触发 `LoadingRow` 事件。`CubeEditor` 在事件处理器中先无条件移除该行已附加的 z 切片高亮样式类（避免重用车池中的残留状态），再读取该行的 `DataContext`（`CubeRowData`），若其 `ZIndex` 等于当前激活 `ZIndex`，则为新行附加 z 切片高亮样式类。

3. **`CubeEditor` 订阅 `DataGrid` 的 `UnloadingRow` 事件**。当行因滚动滚出可视区域而被回收时，触发此事件。`CubeEditor` 在事件处理器中移除该行的 z 切片高亮样式类，确保重用车池中的行容器不携带旧状态。这是标准的 `LoadingRow`/`UnloadingRow` 配对清理模式，与 Avalonia 官方推荐做法一致（GitHub issue #6027）。

4. **`CubeEditor` 订阅 `DataGridRow` 的 `DataContextChanged` 事件**。对于因虚拟化而被重用的行容器，`DataContext` 变更时触发此事件。`CubeEditor` 在事件处理器中同样先无条件移除 z 切片高亮样式类，再根据新的 `DataContext` 中的 `ZIndex` 判断是否重新附加。此机制作为 `LoadingRow`/`UnloadingRow` 配对策略的后备，确保行重用时视觉状态始终与当前数据一致。

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

**后备格式规则**：
- 当 `FormatString` 非空且非 null 时，直接使用 `FormatString` 格式化
- 当 `FormatString` 为空或 null，但 `DisplayPrecision` > 0 时，使用 `F{displayPrecision}` 格式化（如 `DisplayPrecision = 3` 时使用 `F3`）
- 当 `FormatString` 为空或 null，且 `DisplayPrecision` 为 0 时，使用默认后备格式 **`G6`**（通用格式，保留最多 6 位有效数字）。此默认确保即使后端未提供任何精度信息，数值也能以合理精度展示，避免默认 `F0` 导致所有数值显示为整数

**编辑完成后重新格式化规则**：
- 用户输入确认后，数据模型写接口更新原始数值，随后触发 `PropertyChanged`（属性名 `string.Empty`）
- Editor 收到通知后刷新表格显示，刷新后的单元格显示值按上述精度策略（`FormatString` → `DisplayPrecision` → `G6` 后备）重新格式化
- 重新格式化后的显示值可能与用户输入的文本在精度上不同（如用户输入 `3.14159265` 但 `FormatString` 为 `F3`，显示为 `3.142`），此为预期行为，确保展示精度与数据源策略一致

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
- `SelectedVariable` 有效，但其内部数据维度为零（`ICurveData.Length == 0`、`IMapData.XAxisLength == 0` 或 `YAxisLength == 0`、`ICubeData.ZAxisLength == 0` 或 `XAxisLength == 0` 或 `YAxisLength == 0`）
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

> **显式声明：本方案为与需求原始意图的有意识技术折中。**
>
> 需求 `requirement.md` v3 第119–125行明确要求 z 轴列"相同 z 值的连续行采用合并单元格形式展示"。由于 Avalonia DataGrid V12 原生不支持 `RowSpan` 式跨行单元格，本方案将"合并单元格"降级为"每行重复显示 + 视觉分组线"。
>
> **未来升级路径**：若后续 Avalonia 版本（V12 后续更新或 V13+）引入 DataGrid 跨行单元格支持，或项目引入自定义表格组件替代 DataGrid，可重新评估并恢复真正的合并单元格展示。

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

增量刷新通过 `ReplaceRow(int index, TRow newRow)` 实现，由实现方内部在 `ObservableCollection<TRow>` 上执行替换并触发 `INotifyCollectionChanged`，DataGrid 仅刷新对应行，无需全量重建行集合，也无需替换 `ItemsSource`，自然保留选中状态。`CollectionChangedNotifier` 属性让 Editor 无需运行时强制转换即可获取通知器引用。

性能兜底方案：若数据量极端大且不支持增量刷新，采用异步刷新（后台线程重建行集合）+ 延迟刷新（合并短时间内多次编辑为一次批量刷新），或在实现阶段引入显式「保存/应用」按钮，将编辑暂存到本地缓冲区后一次性批量写回。

### 13. `ICalibrationData` 的轴信息属性为何下沉到维度子接口？

**决策**：将 `YAxisName`/`YAxisUnit`/`ZAxisName`/`ZAxisUnit` 从 `ICalibrationData` 中移除，分别下沉到 `IMapData`（y 轴）和 `ICubeData`（y 轴 + z 轴）。`ICurveData` 仅保留从 `ICalibrationData` 继承的 `XAxisName`/`XAxisUnit`。

**理由**：`ICalibrationData` 作为所有维度标定数据的公共契约，其成员应适用于所有维度。`YAxisName`/`YAxisUnit` 对一维曲线数据毫无意义，`ZAxisName`/`ZAxisUnit` 对一维和二维数据均毫无意义。强迫 `ICurveData` 实现方提供这些无意义属性违反接口隔离原则（ISP），导致实现方被迫抛出异常或返回空字符串，增加错误使用风险。将轴信息按维度实际需求声明，使每个接口的契约精确匹配其使用场景。`CalibrationEditorBase<T>` 的信息栏格式化通过运行时类型检查（`T is IMapData`、`T is ICubeData`）动态构建显示文本，在保持灵活性的同时遵守 ISP。

### 14. `MapRowData`/`CubeRowData` 的 x 列数据为何从索引器改为 `IReadOnlyList<double> Values`？

**决策**：将 x 列数据暴露方式从 `double this[int index] { get; }` 索引器改为 `IReadOnlyList<double> Values { get; }` 属性。

**理由**：`MapEditor`/`CubeEditor` 需要动态生成 DataGrid 列，每列绑定路径为 `[0]`、`[1]` 等索引器语法。Avalonia V12 默认启用 CompiledBinding，动态生成列时无法在 XAML 中静态声明 `x:DataType`，CompiledBinding 对运行时动态构造的索引器绑定的支持情况未经验证，存在兼容性风险。改为 `IReadOnlyList<double> Values { get; }` 后，DataGrid 列绑定路径变为 `Values[0]`、`Values[1]` 等属性索引器语法。`IReadOnlyList<T>` 的属性绑定在 CompiledBinding 中的兼容性更优，因为属性访问路径可通过反射解析，而自定义结构体索引器的绑定解析在动态列场景下可能受限。此变更对行数据对象的内存布局影响极小：`Values` 属性可内联为对内部数组的引用，不引入额外堆分配。

### 15. `IMapData.GetValueMatrix()` 和 `ICubeData.GetSliceMatrix()` 的引入理由

**决策**：在 `IMapData` 中增加 `double[,] GetValueMatrix()`，在 `ICubeData` 中增加 `double[,] GetSliceMatrix(int zIndex)`。

**理由**：`ISurfaceChartPresenter.LoadMapData` 和 `LoadSliceData` 均接收 `double[,]` 原始数组，但 `IMapData`/`ICubeData` 在 v5 中仅提供逐点访问接口（`GetValue`）。Editor 在调用图表加载方法前必须自行遍历数据构建 `double[,]` 数组，每次变量切换或 z 切片切换都会产生 O(n) 的内存分配和数据拷贝开销。由后端数据模型直接提供矩阵数据的优势在于：
- 后端可能已在内部使用 `double[,]` 存储数据，可直接返回内部数组的只读视图（如通过 `Array.AsReadOnly` 包装或返回拷贝），避免调用方逐点遍历
- 矩阵的内存布局（二维数组的行优先/列优先）由后端统一控制，避免调用方和图表渲染器之间因转换逻辑不一致导致的数据错位
- 若后端内部不直接使用 `double[,]`，实现方仍可在此方法中执行一次集中式拷贝，开销与调用方遍历相同但不分散到多个消费者

### 16. `ZSliceActivationTracker` 的"选中第一个数据单元格"行为为何移除？

**决策**：将"选中该切片内的第一个数据单元格"行为从 `ZSliceActivationTracker` 的职责中移除，改为 `CubeEditor` 在调用 `OnZColumnClicked` 后自行处理。

**理由**：`ZSliceActivationTracker` 对外暴露的唯一通知机制 `ActiveSliceChanged` 事件无法区分触发源（`OnZColumnClicked` vs `OnSelectionChanged`）。`CubeEditor` 订阅该事件后，无法判断当前激活切片变更是由 z 列点击触发的（需要执行"选中第一个数据单元格"）还是由普通行选中触发的（不需要）。若在 `ActiveSliceChanged` 事件中无条件执行"选中第一个数据单元格"，会破坏用户通过键盘或鼠标在表格中自由导航的正常体验。将触发源特定的行为下放到调用方（`CubeEditor`），Tracker 仅负责纯 z 切片索引的变更管理，职责边界更清晰，也避免事件参数过度膨胀。

## 修订说明（v6）

| 审查意见 | 修改措施 |
|---------|---------|
| **严重-问题1**：Avalonia DataGrid `UnloadingRow` 事件存在，设计文档错误声明其不存在。 | 修正事实声明：确认 Avalonia `DataGrid` 明确包含 `UnloadingRow` 事件。在「z 切片级高亮状态与 DataGrid 虚拟化联动」中将虚拟化状态清理策略改为标准的 `LoadingRow`/`UnloadingRow` 配对模式；`DataContextChanged` 仅作为后备机制保留。 |
| **严重-问题2**：`ITableDataSource<TRow>.Rows` 返回 `IReadOnlyList<TRow>` 与增量刷新机制矛盾（无法索引赋值）。 | 保留 `IReadOnlyList<TRow>` 作为只读视图；新增 `void ReplaceRow(int index, TRow newRow)` 方法作为增量刷新的显式契约入口，由实现方内部执行集合替换并触发 `INotifyCollectionChanged`。 |
| **严重-问题3**：`IAsyncSaveCapable` 的 `bool HasUnsavedChanges` 无法支撑"未保存数量"显示。 | 在 `IAsyncSaveCapable` 中补充 `int UnsavedChangeCount { get; }`；`CalibrationEditorBase<T>` 的脏标记状态栏提示逻辑更新为使用 `UnsavedChangeCount` 显示具体数量，若后端只能提供布尔降级则显示 "存在未保存变更"。 |
| **一般-问题4**：`ZSliceActivationTracker.OnZColumnClicked` 的"选中第一个数据单元格"通知机制缺失。 | 将"选中第一个数据单元格"行为从 Tracker 的职责中移除，改为 `CubeEditor` 在调用 `OnZColumnClicked` 后自行处理选中逻辑；Tracker 仅负责 z 切片索引变更。 |
| **一般-问题5**：`SupportsIncrementalRefresh` 与 `INotifyCollectionChanged` 的关联机制未定义。 | 在 `ITableDataSource<TRow>` 中新增 `INotifyCollectionChanged? CollectionChangedNotifier { get; }` 属性。当 `SupportsIncrementalRefresh` 为 `true` 时返回非 null 通知器实例，消除运行时强制转换的不确定性。 |
| **一般-问题6**：`ICalibrationData` 轴属性对所有维度可见，一维数据模型被迫暴露无关属性。 | 将 `YAxisName`/`YAxisUnit`/`ZAxisName`/`ZAxisUnit` 从 `ICalibrationData` 中移除，下沉到各维度子接口：`ICurveData` 仅保留 `XAxisName`/`XAxisUnit`；`IMapData` 增加 `YAxisName`/`YAxisUnit`；`ICubeData` 增加 `ZAxisName`/`ZAxisUnit`。`CalibrationEditorBase<T>` 的信息栏格式化模板通过运行时类型检查动态构建显示文本。 |
| **一般-问题7**：`ISurfaceChartPresenter` 的 `double[,]` 参数与 `IMapData` 逐点访问接口之间的转换开销未考虑。 | 在 `IMapData` 中增加 `double[,] GetValueMatrix()` 方法；在 `ICubeData` 中增加 `double[,] GetSliceMatrix(int zIndex)` 方法。由后端数据模型决定返回内部数组视图还是执行拷贝，消除 Editor 逐点遍历构建数组的 O(n) 开销。 |
| **一般-问题8**：动态列绑定 `readonly struct` 索引器与 Avalonia V12 CompiledBinding 的兼容性风险。 | 将 `MapRowData`/`CubeRowData` 的 x 列数据暴露方式从索引器改为 `IReadOnlyList<double> Values { get; }`，DataGrid 列绑定路径为 `Values[0]`、`Values[1]` 等属性索引器语法，在 CompiledBinding 中的兼容性优于自定义结构体索引器。 |
| **一般-问题9**：空状态未覆盖数据内容为零维度的场景。 | 在「变量切换流程」中增加对数据内容维度的检查：若 `SelectedVariable` 有效但任一关键维度为零（`Length == 0`、`XAxisLength == 0`、`YAxisLength == 0`、`ZAxisLength == 0`），同样进入空状态。在 `ZSliceActivationTracker` 初始化逻辑中增加 `ZAxisLength == 0` 的防护（`ActiveZIndex` 保持为 `-1`）。 |
| **一般-问题10**：多个事件参数类型未定义。 | 在「核心抽象」中补充 `CurveCellSelectedEventArgs`、`CurveCellEditCompletedEventArgs`、`ChartRenderFailedEventArgs` 三个事件参数类型的完整结构定义；确认 Avalonia `DataGridCellEditEndingEventArgs` 的可用属性（`Column`、`Row`、`EditingElement`、`EditAction`、`Cancel`）。 |
| **一般-问题11**：键盘导航行为未在设计中充分定义。 | 在「关键行为契约」中新增「键盘导航行为」完整章节，明确定义方向键、Tab、Enter、Esc 在各控件中的行为；明确键盘跨 z 切片导航时 `ZSliceActivationTracker.OnSelectionChanged` 调用与防抖策略的交互细节；明确 `OnSelectionChanged(null)` 的语义。 |
| **一般-问题12**：`ZSliceActivationTracker.OnSelectionChanged(null)` 行为未定义。 | 明确 `OnSelectionChanged(null)` 的语义为"保持当前激活切片不变"；新增 `ResetToDefault()` 方法供显式重置行为使用；在「键盘导航行为」中补充 null 参数的处理说明。 |
