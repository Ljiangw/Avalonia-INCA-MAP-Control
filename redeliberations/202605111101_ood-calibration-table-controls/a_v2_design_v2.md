# Avalonia 标定数据表格编辑控件 — 架构级 OOD 设计方案 v4

## 概述

本方案设计一组基于 Avalonia V12 的标定数据表格编辑控件，面向汽车 ECU 标定工具场景。在统一交互范式下支撑 1 维（曲线）、2 维（Map）、3 维（数据立方体）三种标定数据的可视化编辑，并将控件与后端数据模型、图表渲染引擎解耦，确保可复用性和可替换性。

相对于 v3 方案，v4 的核心变化包括：
- **高亮驱动源修正**：行级高亮由 DataGrid 选中状态独立驱动，与 z 切片级高亮彻底分离
- **z 列展示策略调整**：从「首行显示」改为「每行重复显示」，消除虚拟化滚动下的信息丢失风险
- **接口契约补全**：`ICurveData`/`IMapData`/`ICubeData`、`IChartPresenter`、`ICurveTablePresenter` 均补充最小方法签名
- **精度与格式契约**：在 `ICalibrationData` 中引入显示精度与格式化属性，定义流转路径
- **列宽/行高调整策略**：明确 DataGrid 列宽调整能力边界（Avalonia DataGrid 不支持用户拖拽调整行高）
- **编辑回写模式**：明确采用「即时同步 + 脏标记」模式
- **空状态机**：补充变量切换的完整状态机和边界条件处理
- **图表交互深度**：MVP 阶段仅支持默认视角静态展示
- **`ITableDataSource` 泛型化**：明确 `ITableDataSource<TRow>` 约束关系

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
│  │  ITableDataSource   │  IChartPresenter              │    │
│  │  (表格适配接口)      │  (图表渲染抽象)                │    │
│  │  [Map/Cube 专用]    │                               │    │
│  └─────────────────────┘                               │    │
│  ┌─────────┐ ┌────────┐ ┌─────────────┐                │    │
│  │ICurveData│ │IMapData│ │  ICubeData  │                │    │
│  └─────────┘ └────────┘ └─────────────┘                │    │
│         ▲              ▲              ▲                │    │
│         └──────────────┴──────────────┘                │    │
│              ICalibrationData (公共数据契约)            │    │
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
| **图表抽象层** | 定义折线图和 3D 曲面图的渲染契约，隔离具体图表库实现。允许包含轻量的 Avalonia 基础类型依赖（如 `Visual`），但不耦合具体渲染实现。 | 被 Views 层依赖 |
| **激活追踪层** | CubeEditor 中 z 切片激活状态的专门管理者，将表格行/单元格选中事件映射为 z 值切换。 | 依赖表格适配层的 `CubeRowData`，被 CubeEditor 使用 |
| **Curve 表格契约层** | `CurveEditor` 专用的横向表格视图契约（`ICurveTablePresenter`），将一维 x/z 数据映射为两行横向表格，处理选中、编辑与高亮。 | 依赖数据契约层，被 CurveEditor 使用 |

依赖规则：数据契约层为最底层，无 outward 依赖；适配层和图表抽象层仅依赖数据契约；激活追踪层仅依赖表格适配层；Curve 表格契约层仅依赖数据契约层；Views 层可依赖所有下层模块，但下层模块不可反向依赖 Views 层。

## 核心抽象

### `ICalibrationData`（接口）

**角色**：所有维度标定数据的公共契约。

**职责**：暴露标定变量的元信息（名称、单位、各轴变量名及单位）、数据变更通知能力、显示精度与格式化信息，以及变量分组信息。

**协作**：被 `ICurveData`、`IMapData`、`ICubeData` 继承；被 `CalibrationEditorBase<T>` 作为泛型约束使用；被表格适配层、图表抽象层和 Curve 表格契约层消费。

**类型形态**：接口，**明确继承 `INotifyPropertyChanged`**。具体数据模型由后端标定系统定义，控件组只消费契约，不拥有实现。将 `INotifyPropertyChanged` 纳入继承列表，强制后端实现方具备数据变更通知能力，确保单元格编辑后绑定系统能正确感知变更并刷新 UI。

**最小契约成员**：
- `string VariableName { get; }` — 变量内部标识名
- `string GroupName { get; }` — 变量所属分组（用于下拉框按 `<曲线变量组> VariableName` 格式展示）
- `string DisplayName { get; }` — 格式化后的显示文本，默认可由 `GroupName` 和 `VariableName` 组合生成
- `string Unit { get; }` — 变量值单位
- `string FormatString { get; }` — 数值显示格式字符串（如 `"F2"`、`"G6"`），由后端统一提供以确保表格和图表的精度一致性
- `int DisplayPrecision { get; }` — 显示小数位数，作为 `FormatString` 的辅助信息
- 各轴信息属性：`XAxisName`、`XAxisUnit`、`YAxisName`、`YAxisUnit`、`ZAxisName`、`ZAxisUnit`（各轴名及单位，不存在的轴返回空字符串）

### `ICurveData` / `IMapData` / `ICubeData`（接口）

**角色**：维度特化的数据契约。

**职责**：在 `ICalibrationData` 基础上，各自声明维度特定的数据访问方式。

**协作**：作为 `CalibrationEditorBase<T>` 的泛型参数具体化类型；被各自的表格适配器或 Curve 表格呈现器消费。

**类型形态**：接口，继承自 `ICalibrationData`。不采用抽象类，因为后端数据模型可能已经继承自其他领域基类，接口避免单继承限制。

**`ICurveData` 最小契约成员**：
- `int XAxisLength { get; }` — x 轴数据点数量
- `IReadOnlyList<double> XValues { get; }` — x 轴值序列（只读）
- `IReadOnlyList<double> ZValues { get; }` — z 轴值序列（因变量，只读）
- `void SetXValue(int index, double value)` — 更新 x 轴指定位置的值
- `void SetZValue(int index, double value)` — 更新 z 轴指定位置的值

**`IMapData` 最小契约成员**：
- `int XAxisLength { get; }` — x 轴维度长度
- `int YAxisLength { get; }` — y 轴维度长度
- `double GetValue(int xIndex, int yIndex)` — 读取二维矩阵指定位置的值
- `void SetValue(int xIndex, int yIndex, double value)` — 更新二维矩阵指定位置的值
- `IReadOnlyList<double> XAxisBreakpoints { get; }` — x 轴断点值序列
- `IReadOnlyList<double> YAxisBreakpoints { get; }` — y 轴断点值序列

**`ICubeData` 最小契约成员**：
- `int XAxisLength { get; }` — x 轴维度长度
- `int YAxisLength { get; }` — y 轴维度长度
- `int ZAxisLength { get; }` — z 轴维度长度
- `double GetValue(int xIndex, int yIndex, int zIndex)` — 读取三维张量指定位置的值
- `void SetValue(int xIndex, int yIndex, int zIndex, double value)` — 更新三维张量指定位置的值
- `IReadOnlyList<double> XAxisBreakpoints { get; }` — x 轴断点值序列
- `IReadOnlyList<double> YAxisBreakpoints { get; }` — y 轴断点值序列
- `IReadOnlyList<double> ZAxisBreakpoints { get; }` — z 轴断点值序列

### `CalibrationEditorBase<T>`（抽象类）

**角色**：三个标定编辑控件的公共基座。

**职责**：
- 定义并管理 Avalonia 依赖属性：`ItemsSource`（`IList<T>` 类型，绑定到变量选择下拉框）、`SelectedVariable`（当前选中的标定数据项）
- 提供统一的顶部信息栏模板结构（左侧 `ComboBox`、右侧单位/轴信息展示）
- 处理变量切换时的选中变更事件，向子类发出数据切换通知
- 管理空状态：当 `ItemsSource` 为 `null` 或空列表、`SelectedVariable` 为 `null` 时，进入空状态展示友好提示

**协作**：内部聚合一个 `ComboBox` 和一个信息展示面板；通过模板约定或受保护的虚方法向子类（`CurveEditor`、`MapEditor`、`CubeEditor`）暴露数据切换钩子。

**类型形态**：泛型抽象类（`T : ICalibrationData`），继承自 Avalonia `TemplatedControl`。选用抽象类而非接口，是因为需要封装具体的依赖属性注册、模板部件约定和公共视觉状态逻辑。

**空状态虚方法**：
- `protected virtual void OnEnterEmptyState()` — 进入空状态时调用，子类可覆盖以自定义空状态展示
- `protected virtual void OnExitEmptyState()` — 退出空状态时调用，子类可覆盖以恢复主内容区

**信息栏格式化**：顶部信息栏默认采用格式模板 `[{Unit}] x: {XAxisName} [{XAxisUnit}] y: {YAxisName} [{YAxisUnit}] z: {ZAxisName} [{ZAxisUnit}]`。不存在的轴信息字段自动省略。子类可通过覆盖受保护的 `BuildInfoText()` 方法自定义格式。

### `CurveEditor` / `MapEditor` / `CubeEditor`（密封类）

**角色**：面向终端使用者的三个具体控件。

**职责**：
- `CurveEditor`：主内容区上方为折线图区域、下方为 1 维两行表格（x 行 + z 行）。表格采用横向布局，通过 `ICurveTablePresenter` 构建，不使用 `DataGrid` 纵向行模型
- `MapEditor`：主内容区左侧为 3D 曲面图、右侧为 2 维表格（x 列头 + y 行头 + z 数据单元格）
- `CubeEditor`：主内容区左侧为 3D 曲面图（展示当前激活 z 切片）、右侧为平铺 3 维表格（z/y 双列行头 + x 列头 + 数据单元格）。表格以平铺形式展示完整数据立方体的所有 (z, y) 行 × x 列，而非仅展示当前激活切片。z 切片激活由表格内的选中/导航交互驱动

**协作**：`MapEditor` 与 `CubeEditor` 各自内部持有对应的表格适配器实例和图表呈现器实例；`CubeEditor` 额外持有 `ZSliceActivationTracker`。`CurveEditor` 内部持有 `ICurveTablePresenter` 实例和 `IChartPresenter` 实例。

**类型形态**：密封类，继承 `CalibrationEditorBase<T>`（`T` 分别为 `ICurveData`、`IMapData`、`ICubeData`）。密封是因为这些控件是领域终态产品，不存在合理的进一步派生场景。

### `ICurveTablePresenter`（接口）

**角色**：`CurveEditor` 横向两行表格的视图构建与交互处理契约。

**职责**：
- 接收 `ICurveData`，构建两行横向表格的视觉树。第一行展示所有 x 轴值，第二行展示所有 z 轴值，列数与数据点数量一致
- 管理单元格选中状态（当前活动单元格）和单元格编辑状态（进入/退出编辑模式）
- 提供单元格编辑完成事件，携带数据点索引（列位置）和行标识（X 行或 Z 行），供 `CurveEditor` 回写到原始数据模型
- 提供选中单元格变更通知，供 `CurveEditor` 同步驱动折线图的数据点高亮

**协作**：被 `CurveEditor` 内部持有；向下消费 `ICurveData`；向上通过事件通知 `CurveEditor` 单元格编辑完成和选中变更。实现方基于 Avalonia 自定义布局（如 `Grid` + `ItemsControl` 或自定义 `Panel`）构建横向视觉树，不依赖 `DataGrid`。

**类型形态**：接口。`CurveEditor` 的横向布局与 `MapEditor`/`CubeEditor` 的纵向 `DataGrid` 行模型本质不同：前者是固定两行、动态多列的矩阵转置形态，后者是纵向行集合。独立的 `ICurveTablePresenter` 契约允许实现者选择最适合横向布局的 Avalonia 布局容器，保持架构清晰。

**最小契约成员**：
- `void LoadData(ICurveData data)` — 加载数据并重建横向表格视觉树
- `Control VisualRoot { get; }` — 视觉树根元素，由 `CurveEditor` 将其嵌入主内容区
- `event EventHandler<CurveCellSelectedEventArgs> SelectedCellChanged` — 选中单元格变更事件，参数携带列索引和行标识（X/Z）
- `event EventHandler<CurveCellEditCompletedEventArgs> CellEditCompleted` — 单元格编辑完成事件，参数携带列索引、行标识（X/Z）和新数值
- `void Refresh()` — 刷新显示（当外部数据变更时调用）

### `ITableDataSource<TRow>`（接口）

**角色**：多维数据到二维纵向表格结构的投影器。供 `MapEditor` 与 `CubeEditor` 使用。

**职责**：将原始标定数据（二维矩阵、三维张量）转换为适合 Avalonia `DataGrid` 绑定的行集合。各维度实现返回各自强类型的行数据对象集合，行数据对象内联携带原始数据坐标索引，以支撑编辑后的值回写。

**协作**：被 `MapEditor` 和 `CubeEditor` 内部使用，作为 `DataGrid` `ItemsSource` 的实际来源。编辑完成时，Editor 从强类型行数据对象中提取坐标索引，调用原始数据模型的写接口。

**类型形态**：泛型接口 `ITableDataSource<TRow> where TRow : struct`。`MapEditor` 实例化为 `ITableDataSource<MapRowData>`，`CubeEditor` 实例化为 `ITableDataSource<CubeRowData>`。

**最小契约成员**：
- `IReadOnlyList<TRow> Rows { get; }` — 行数据集合
- `void Refresh()` — 当外部数据变更时重新生成行集合

### `MapRowData` / `CubeRowData`（`readonly struct`）

**角色**：承载一行纵向表格数据的展示值及其在原始多维数据中的索引坐标。供 `MapEditor` 与 `CubeEditor` 使用。

**职责**：
- `MapRowData`：内联 `XIndex`、`YIndex` 属性，标识该单元格在二维矩阵中的坐标
- `CubeRowData`：内联 `ZIndex`、`YIndex` 属性，标识该数据行在三维张量中的坐标；同时承载 `ZValue` 和 `YValue` 文本（用于 z/y 双列行头展示）。每行对应一个 (z, y) 组合下所有 x 列的数据值

**协作**：由对应维度的表格适配器生成并填充，作为 `DataGrid` 的 `ItemsSource` 行集合。Editor 在编辑确认时从行数据对象中直接读取坐标索引属性，回写到原始数据模型。

**类型形态**：`readonly struct`。值类型避免堆分配；坐标索引作为结构体的属性直接内联，不通过接口引用传递，彻底消除接口装箱带来的 GC 压力。

### `IChartPresenter`（接口）

**角色**：图表渲染的抽象契约。

**职责**：接收标定数据，在宿主控件区域内渲染折线图（1D）或 3D 曲面图（2D/3D）。MVP 阶段仅负责数据到图形的静态映射，交互深度留待后续迭代评估。

**协作**：被 `CurveEditor`、`MapEditor`、`CubeEditor` 内部持有。实现方可选用 OxyPlot、LiveCharts、ScottPlot、自定义 Skia 渲染等具体技术。

**类型形态**：接口。因为 Avalonia V12 生态中没有内置的 3D 曲面图标准控件，图表实现必须可替换，接口是最低耦合的抽象形式。

**最小契约成员**：
- `void LoadData(ICalibrationData data)` — 加载标定数据，内部转换为图表所需格式
- `void Refresh()` — 显式触发重绘
- `Control VisualRoot { get; }` — 图表控件的根元素，由 Editor 将其嵌入视觉树
- `event EventHandler<ChartRenderFailedEventArgs> RenderFailed` — 渲染失败事件，实现方内部捕获异常后触发，Editor 监听并降级显示占位矩形

### `ZSliceActivationTracker`（类）

**角色**：CubeEditor 中 z 切片激活状态的专门管理者。

**职责**：
- 维护当前激活的 z 值索引
- 监听表格选中项变更，从 `CubeRowData` 中提取 `ZIndex`，判断是否需要切换激活切片
- 处理 z 列分组首行的点击事件：当用户点击属于某 z 切片分组的区域时，将该 z 值设为激活切片，并联动选中该切片内的第一个数据单元格
- 提供激活切片变更通知（供 3D 图表刷新和 z 切片级高亮使用）
- 处理键盘快速跨 z 切片导航时的防抖/延迟策略（若数据量大）

**协作**：由 `CubeEditor` 实例化并注入；向下消费表格适配器提供的 `CubeRowData` 行数据（通过 `ZIndex` 属性按 z 值分组）；向上通过纯 .NET 事件（`EventHandler<T>`）通知 `CubeEditor`。`CubeEditor` 在事件处理器中通过 `Dispatcher.UIThread.Post` 将状态变更调度到 UI 线程，确保 Tracker 不依赖 Avalonia 视觉类型，保持可测试性。

**类型形态**：独立类（非静态）。将这段逻辑从 `CubeEditor` 中剥离，使控件类聚焦模板和交互路由，状态映射逻辑可独立单元测试。

## 关键行为契约

### 变量切换流程

用户操作 ComboBox 选中不同变量 → `CalibrationEditorBase<T>` 更新 `SelectedVariable` 依赖属性 → 触发受保护的虚方法 `OnSelectedVariableChanged` → 子类 Editor 响应：
- `CurveEditor`：通知 `ICurveTablePresenter` 重建横向表格数据源，通知 `IChartPresenter` 加载新数据
- `MapEditor`/`CubeEditor`：重建表格适配器数据源、通知图表呈现器加载新数据
- `CubeEditor` 额外重置 `ZSliceActivationTracker` 的 z 切片激活状态为默认值（如第一个 z 切片）

**空状态处理**：
- 当 `ItemsSource` 为 `null` 或空列表时，ComboBox 无可用项，`CalibrationEditorBase<T>` 自动调用 `OnEnterEmptyState()`，主内容区显示"无可用变量"提示
- 当 `ItemsSource` 非空但 `SelectedVariable` 为 `null` 时（如初始状态或清空选择），同样进入空状态，主内容区显示"请选择一个变量"
- 变量切换过程中（从旧变量卸载到新变量加载完成前），子类 Editor 可展示过渡状态（如轻量加载指示）

### 单元格编辑与数据回写

#### 编辑触发方式

表格数据单元格采用 **双击进入编辑** 模式，单击仅用于选中/导航。行头列（z 列、y 列）不参与编辑，单击它们仅触发选中或 z 切片激活。

#### MapEditor / CubeEditor（DataGrid 纵向行模型）

用户触发单元格进入编辑模式（双击进入编辑，单击仅用于选中/导航）→ 单元格展示编辑模板（如 `TextBox`）→ 用户输入数值 → 输入完成时（Enter/失去焦点），Editor 拦截 `CellEditEnding` 事件：
1. 从对应类型的行数据对象中提取坐标索引属性
2. 调用原始数据模型的写方法更新对应位置
3. 模型触发 `INotifyPropertyChanged.PropertyChanged`
4. 绑定系统刷新关联单元格；若该单元格值同时影响图表，图表呈现器接收通知后重绘

**回写模式**：采用 **即时同步** 模式。编辑完成后立即调用数据模型写接口更新原始数据，无需显式保存按钮。同时引入 **脏标记** 机制：
- `ICalibrationData` 的实现方在数据被修改后标记自身为 Dirty
- `CalibrationEditorBase<T>` 可通过绑定或事件监听脏状态，在控件底部状态栏显示未保存提示（如 "* 已修改"）
- 脏状态的持久化/保存动作由后端标定系统负责，控件层仅负责展示提示

在 CubeEditor 中，编辑操作发生在平铺表格的任意数据单元格。编辑完成后，Editor 从该单元格对应的 `CubeRowData` 中读取 `ZIndex`、`YIndex`，并结合该单元格所在的 x 列索引 `XIndex`，调用 `ICubeData` 的写接口更新三维张量中对应位置的值。

#### CurveEditor（横向自定义布局）

用户触发单元格进入编辑模式（双击进入编辑，单击仅用于选中/导航）→ `ICurveTablePresenter` 在对应列的对应行（X 行或 Z 行）展示编辑控件（如 `TextBox`）→ 用户输入数值 → 输入完成时，`ICurveTablePresenter` 触发编辑完成事件，携带数据点索引（列位置）、行标识（X 或 Z）以及解析后的数值 → `CurveEditor` 在事件处理器中调用 `ICurveData` 的写接口更新对应位置的数据 → 模型触发 `INotifyPropertyChanged.PropertyChanged` → `ICurveTablePresenter` 刷新对应单元格显示，`IChartPresenter` 重绘折线图。

### z 切片激活与 3D 视图联动（CubeEditor）

用户点击/键盘导航到表格中的某个单元格或行 → `CubeEditor` 将选中的 `CubeRowData` 传递给 `ZSliceActivationTracker` → Tracker 读取该行的 `ZIndex` 属性，与当前激活 z 比较：若相同则无动作；若不同则更新激活 z 值并发出变更事件 → `CubeEditor` 在事件处理器中通过 `Dispatcher.UIThread.Post` 调度 UI 更新：通知图表呈现器切换数据源到新的 z 切片矩阵，同时触发表格的视觉状态更新（z 切片级高亮）。

当用户点击 z 列分组区域时，`ZSliceActivationTracker` 将该 z 值设为激活切片，并通知 `CubeEditor` 选中该切片内的第一个数据单元格。

键盘跨 z 切片导航时（如从当前 z 切片的最后一行 y 值按方向键移入下一 z 切片的第一行 y 值），`ZSliceActivationTracker` 内部引入短时间防抖（如 100–200ms），仅在用户停止快速导航后才真正刷新 3D 曲面图，避免频繁重绘造成卡顿。

### z 切片级高亮状态与 DataGrid 虚拟化联动（CubeEditor）

`DataGrid` 仅对当前可视区域内的行创建视觉树实例（行虚拟化）。`ZSliceActivationTracker` 输出的激活 `ZIndex` 需通过以下机制传递到行视觉元素：

1. **`CubeEditor` 订阅 `ZSliceActivationTracker` 的激活变更事件**。当激活 `ZIndex` 变化时，`CubeEditor` 执行两项操作：通知 `IChartPresenter` 刷新 3D 曲面图；遍历当前已加载的 `DataGridRow` 容器，为属于激活 z 切片的行附加样式类（如 `active-z-slice`），为非激活行移除该样式类。

2. **`CubeEditor` 订阅 `DataGrid` 的 `LoadingRow` 事件**。当用户滚动导致新行进入可视区域时，`DataGrid` 创建对应的行视觉元素并触发 `LoadingRow` 事件。`CubeEditor` 在事件处理器中**总是先清除**该行已有的 `active-z-slice` 样式类（覆盖虚拟化行回收复用场景），再根据该行的 `DataContext`（`CubeRowData`）中的 `ZIndex` 是否等于当前激活 `ZIndex`，决定是否重新附加 z 切片高亮样式类。这确保了虚拟化机制下，新进入可视区域的行和回收复用的行都能正确显示激活切片的高亮状态。

通过上述机制，`ZSliceActivationTracker` 的状态输出（纯 .NET 事件）与 `DataGrid` 视觉树的状态同步解耦：Tracker 不依赖 Avalonia 视觉类型，而 `CubeEditor` 作为 UI 层协调者负责将逻辑状态映射到视觉状态，兼容 `DataGrid` 的行虚拟化和行回收机制。

### 高亮叠加策略

三种高亮在 CubeEditor 中可能同时存在，视觉优先级从高到低为：

1. **单元格级高亮**（蓝色背景，当前活动单元格）— 由 DataGrid 的 `SelectedItem` / `CurrentItem` 选中状态直接驱动
2. **行级高亮**（轻微背景色，当前选中单元格所在的整行）— 由 DataGrid 的 `CurrentItem` 所在行驱动，通过 DataGrid 内置的选中行样式或基于 `CurrentCell` 状态的样式选择器实现
3. **z 切片级高亮**（左侧边框或分组背景色，当前激活切片的所有行）— 由 `ZSliceActivationTracker` 输出的激活 `ZIndex` 驱动，通过 `CubeEditor` 动态附加/移除样式类实现

三种高亮的驱动来源完全独立：单元格级和行级由 DataGrid 自身的选中状态系统驱动；z 切片级由 `ZSliceActivationTracker` 驱动。`ZSliceActivationTracker` **不**参与行级高亮的决策。

### 列宽/行高调整策略

#### MapEditor / CubeEditor（DataGrid）

- **列宽调整**：启用 `CanUserResizeColumns`，允许用户通过拖拽列头边界调整 x 轴数据列的宽度。z 列和 y 列作为行头标识列，列宽固定或允许有限调整。
- **动态列宽度初始化**：`DataGrid` 动态生成列时，x 轴数据列采用统一宽度（如 `DataGridLength.SizeToHeader` 或固定像素值），避免列数多时出现极端列宽。
- **最小/最大列宽约束**：通过 `DataGridColumn.MinWidth` / `MaxWidth` 限制，防止用户将列宽调整到不可读或过度占用空间的极端值。
- **行高调整**：Avalonia `DataGrid` **不支持**用户拖拽调整行高（不存在 `CanUserResizeRows` 属性）。行高通过 `DataGrid.RowHeight` 设置默认值，或通过 `MinRowHeight` / `MaxRowHeight` 限制范围。行内容采用 `DataGridLength.Auto` 保证可读性。如需更大的行高，可在控件外层提供独立的缩放/字号调整机制。

#### CurveEditor（横向自定义布局）

`ICurveTablePresenter` 的实现方负责横向布局的列宽分配策略。建议采用等宽分配（所有数据列相同宽度）或基于内容自适应（`Grid` 列的 `Auto` / `*` 混合）。用户列宽调整能力由 `ICurveTablePresenter` 的实现方决定是否暴露（如通过列头拖拽手势）。

### 精度一致性展示

精度信息从 `ICalibrationData.FormatString` 和 `DisplayPrecision` 出发，流向两个消费端：
1. **表格层**：`MapTableAdapter` / `CubeTableAdapter` 在生成行数据时，将 `FormatString` 应用到单元格的显示值（如通过 `StringFormat` 绑定或适配器预格式化）；`CurveEditor` 的 `ICurveTablePresenter` 实现方在渲染单元格文本时应用同一 `FormatString`
2. **图表层**：`IChartPresenter` 实现方在渲染轴标签、数据标签时，消费 `ICalibrationData.FormatString` 或 `DisplayPrecision`，确保图表 tooltip 和轴刻度与表格中的数值展示格式一致

精度信息由后端标定系统统一提供，控件层只负责消费，不自行推断精度。

## 错误处理策略

### 输入验证错误

单元格编辑阶段的数值解析失败（如非法字符、越界值）属于局部错误。错误处理策略为「就地反馈」：在单元格层面通过红色边框或工具提示标识错误，阻止该次编辑回写，但不阻断用户继续编辑其他单元格或切换变量。

在 `MapEditor`/`CubeEditor` 中，由于行数据源采用 `readonly struct`，DataGrid 的标准单元格编辑双向绑定自动提交机制无法直接更新不可变结构体实例。编辑回写完全依赖 `CellEditEnding` 事件拦截，由 Editor 从行数据对象提取坐标索引后调用原始数据模型写接口。**在此模式下，放弃 DataGrid 内置的 `DataErrorValidationRule` 验证模板，改为完全手动的事件拦截 + 编辑模板状态控制**：
- `CellEditEnding` 事件处理器中解析用户输入，若解析失败或数值越界，设置编辑模板 `TextBox` 的 `DataValidationErrors.Errors` 附加属性或边框样式为错误状态
- 验证失败时设置 `e.Cancel = true` 阻止默认提交行为
- 用户修正输入后再次触发 `CellEditEnding`，验证通过后执行回写

在 `CurveEditor` 中，`ICurveTablePresenter` 的实现方负责拦截非法输入并在编辑控件层面呈现错误标识。

### 数据绑定不匹配错误

控件绑定到不符合维度期望的数据模型（如 `CubeEditor` 接收的 `ICubeData` 中 z 维度长度为 0）属于结构性错误。控件应进入「空状态」展示友好提示（如 "无有效数据"），而非抛出未处理异常导致控件白屏。

### 图表渲染错误

图表库渲染失败（如 GPU 不可用、3D 数据异常）属于可降级错误。`IChartPresenter` 实现方应内部捕获渲染异常，通过 `RenderFailed` 事件通知 Editor，Editor 降级为显示占位矩形或错误图标，保证表格编辑功能不受影响。

## 并发设计

本场景为单用户桌面交互，无显式多线程并发需求。但需注意以下边界：

- **UI 线程亲和性**：所有 Avalonia 控件状态变更必须在 UI 线程执行。若后端标定数据模型由后台线程更新（如来自 ECU 通信线程），数据模型应在触发 `PropertyChanged` 前通过 `Dispatcher.UIThread.Invoke` 调度。
- **图表渲染隔离**：图表重绘可能是 CPU/GPU 密集型操作。`IChartPresenter` 实现时应考虑在后台线程准备顶点/网格数据，仅在提交渲染时回到 UI 线程，避免阻塞交互。
- **防抖策略**：CubeEditor 中键盘快速跨 z 切片导航时，`ZSliceActivationTracker` 内部可对激活切片变更引入短时间防抖（如 100–200ms），仅在用户停止快速导航后才真正刷新图表。
- **大数据量虚拟化**：CubeEditor 平铺所有 z 切片后，表格行数可达 z 数 × y 数，数据量显著增加。`DataGrid` 的行虚拟化（`EnableRowVirtualization`）必须保持开启；`CubeTableAdapter` 生成行集合时应避免在热路径中进行复杂计算，必要时采用惰性加载或分页策略。

## 设计决策

### 1. CubeEditor 表格从「当前切片展示」改为「全量平铺展示」，行数膨胀后性能如何保障？

**决策**：保留 `DataGrid` 的行虚拟化机制，`CubeTableAdapter` 一次性生成完整的平铺行集合但依赖 UI 层的虚拟化只渲染可见行；若 z 数 × y 数极端大（如 >10,000 行），在实现阶段评估是否需要引入 z 切片折叠/展开或分页机制。

**理由**：v3 要求 CubeEditor 表格平铺展示所有 z 切片的完整数据，行数从原来的 y 数膨胀到 z 数 × y 数。Avalonia `DataGrid` 的行虚拟化机制可以处理数千行的滚动场景，只要行模板保持轻量即可。`CubeRowData` 作为 `readonly struct` 的数组一次性分配后由 `DataGrid` 按需访问，不会产生每行的堆分配压力。但若标定数据的 z 和 y 维度都很大（如各 100 以上），1 万行以上的虚拟化滚动体验可能下降，此时可在实现阶段引入 z 切片分组的折叠/展开机制作为性能兜底。架构设计阶段不预先引入折叠抽象，避免过度设计。

### 2. Avalonia DataGrid 原生不支持跨行合并单元格，z 列如何分组展示？

**决策**：不追求原生跨行合并，采用 **「z 值每行重复显示 + 视觉分组线 + 左侧边框标识」** 的组合方案。

**理由**：Avalonia 的 `DataGrid`（包括 V12）不提供 WPF 中 `RowSpan` 式的跨行单元格能力。若强行在模板层模拟合并单元格，会严重破坏虚拟化机制和行回收，在数据量大时性能急剧下降。替代方案中，z 列的自定义 `CellTemplate` 在**每一行**都渲染 z 值文本（通过 `CubeRowData.ZValue`），而非仅在首行显示。这牺牲了部分视觉简洁性，但彻底消除了虚拟化滚动下首行滚出后 z 值标识丢失的问题。配合行样式在 z 切片边界处渲染水平分隔线，激活切片通过整组行的左侧边框或轻微背景色统一标识。此方案在信息完整性上优于合并单元格，且完全兼容 `DataGrid` 的虚拟化。

### 3. 为什么用抽象基类 `CalibrationEditorBase<T>` 而非纯组合或接口？

**决策**：采用泛型抽象类承载公共 UI 结构和依赖属性。

**理由**：三个控件共享的不仅是行为契约，还有大量的具体 UI 结构——顶部栏的 `ComboBox`、单位信息面板、变量切换后的数据刷新管线。抽象类可以在 Avalonia 的类型系统中直接注册这些公共依赖属性（`AvaloniaProperty.Register`），并约定公共 ControlTemplate 的部件名称（如 `PART_VariableSelector`、`PART_InfoPanel`）。若用纯接口，每个控件需要重复注册相同的依赖属性和模板逻辑，违背 DRY 原则。组合模式（如注入一个「顶部栏管理器」对象）在 Avalonia 的模板/样式系统中反而增加不必要的间接层。

### 4. 表格编辑如何定位回原始数据？

**决策**：通过维度特化的 `readonly struct` 行数据对象建立可见单元格到原始多维索引的映射，坐标索引直接内联为结构体属性，取消独立的坐标接口抽象。

**理由**：Avalonia 表格控件（`DataGrid`）天然面向二维行集合（`IEnumerable` 的扁平行）。原始标定数据是 2D/3D 矩阵，无法直接作为 `ItemsSource`。引入表格适配层将矩阵投影为行集合后，每一行必须携带「我来自原始数据的哪个位置」这一元信息，否则编辑后无法回写。v2 已验证将坐标索引直接内联到各维度行数据类型（`MapRowData`、`CubeRowData`）中的方案，以强类型属性传递坐标信息，彻底消除接口装箱，同时保持编辑回写时的坐标定位能力。v3 中 `CubeRowData` 额外承载 `ZValue`、`YValue`，以支撑 z/y 双列行头的展示需求。

### 5. 3D 曲面图没有 Avalonia 内置方案，如何设计？

**决策**：通过 `IChartPresenter` 接口抽象图表渲染，控件不耦合具体图表库。MVP 阶段图表仅支持默认视角的静态展示，缩放、平移、旋转等交互操作留待后续迭代评估。

**理由**：Avalonia V12 标准控件集中不包含 3D 图表。社区中 OxyPlot、LiveCharts、ScottPlot 等库对 3D 的支持程度和 API 差异很大，且部分库在 V12 下的兼容性尚在发展。将图表渲染抽象为接口后，控件只关心「把数据给你，你在指定区域内画出来」。实现阶段可以先以 2D 热力图或伪 3D 网格作为最小可行方案，后续替换为真正的 3D 引擎时不影响控件架构。这避免了在设计阶段就锁定可能不合适的图表技术。

MVP 阶段仅支持静态展示的理由：标定数据表格编辑的核心交互发生在表格中，图表主要起数据可视化参考作用。在 MVP 阶段优先保障表格编辑体验的完整性和稳定性，图表交互深度可在用户反馈后迭代添加。若后续确定支持交互，`IChartPresenter` 可通过扩展方法或新增事件补充交互契约，无需修改现有接口签名。

### 6. CubeEditor 的 z 切片激活逻辑为何独立为 `ZSliceActivationTracker`？

**决策**：将 z 切片激活的状态管理和事件映射抽取为独立类，而非内嵌在 `CubeEditor` 中。

**理由**：CubeEditor 的表格同时承担多重职责——数据展示、单元格编辑、z 切片选择器。若将 z 切片激活逻辑（行→z 值提取、激活变更判断、防抖、通知联动、z 列分组首行点击处理）全部写在 `CubeEditor` 的代码后台，会导致该类臃肿且难以测试。`ZSliceActivationTracker` 封装了「从表格选中状态到 z 切片语义」的纯逻辑转换，不依赖 Avalonia 控件的具体视觉类型，仅消费 `CubeRowData` 行数据对象。其状态输出采用纯 .NET 事件（`EventHandler<T>`），由 `CubeEditor` 在事件处理器中调度 UI 线程更新，这使得 Tracker 逻辑可以在无头环境中单元测试，也便于后续调整 z 切片切换的交互策略。

### 7. 编辑触发方式：单击还是双击？

**决策**：表格数据单元格采用 **双击进入编辑** 模式，单击仅用于选中/导航。

**理由**：在标定数据表格中，用户频繁进行选中浏览（查看相邻单元格数值、跨行比较）。若单击即进入编辑，会频繁弹出 `TextBox` 编辑模板，干扰浏览体验。双击编辑是桌面表格工具（Excel、INCA 等标定工具）的广泛惯例，与用户的工程工具心智模型一致。行头列（z 列、y 列）不参与编辑，单击它们仅触发选中或 z 切片激活。

### 8. CurveEditor 的横向两行表格为何不用 DataGrid，而采用独立布局契约？

**决策**：`CurveEditor` 不使用 `DataGrid` 和 `ITableDataSource` 纵向行集合抽象，而采用独立的 `ICurveTablePresenter` 接口，由实现方基于 Avalonia 自定义布局容器构建横向视觉树。

**理由**：需求明确要求 `CurveEditor` "以两行形式展示：第一行为 x 轴（自变量）值，第二行为 z 轴（因变量）值"，这是一种矩阵转置的表格形态——固定两行、动态多列。而 `DataGrid` 的设计范式是纵向行集合（每行一个数据对象，列对应属性），与横向布局的语义天然冲突。若强行套用 `DataGrid`，需要为每个数据点动态生成一列，并将两个行对象（x 行和 z 行）的数组索引绑定到各列，实现复杂度极高且违背 `DataGrid` 的设计意图。`CurveEditor` 的数据规模通常较小（一维曲线的数据点数量有限），不需要 `DataGrid` 的行虚拟化、列排序等重型功能。通过 `ICurveTablePresenter` 接口将横向布局的构建和交互抽象化后，实现方可选用 `Grid` + `ItemsControl`、`UniformGrid` 或自定义 `Panel` 等轻量方案，精确控制列宽、单元格间距和选中高亮行为，与需求的横向两行展示完全对齐。同时，`CurveEditor` 与 `MapEditor`/`CubeEditor` 在表格实现上的差异被封装在各自的适配/呈现层中，不影响公共基类的统一性。

### 9. `readonly struct` 作为行数据源时，输入验证反馈如何保障？

**决策**：保持 `readonly struct` 作为行数据源以优化 GC，但**放弃 DataGrid 内置验证模板**，采用完全手动的 `CellEditEnding` 事件拦截 + 编辑模板状态控制实现验证反馈。

**理由**：`readonly struct` 避免每行的堆分配，在 CubeEditor 行数膨胀到 z 数 × y 数时 GC 优势显著。但 DataGrid 的标准单元格编辑流程会尝试将编辑值自动写回绑定的数据源——对于不可变的 struct，此自动写回必然失败。设计中已明确编辑回写通过 `CellEditEnding` 事件拦截实现（由 Editor 从行数据对象提取坐标索引后调用原始数据模型写接口），`readonly struct` 的不可变性不影响此回写链路（Editor 不修改行数据对象，而是直接修改后端模型并触发 `PropertyChanged` 刷新 UI）。

对于验证反馈，DataGrid 的 `DataErrorValidationRule` 和默认验证模板依赖绑定源对象实现 `IDataErrorInfo` / `INotifyDataErrorInfo`，而 `readonly struct` 作为临时值对象无法有效承载验证错误状态。因此，验证反馈完全在 `CellEditEnding` 事件处理器中手动实现：解析失败时设置编辑模板控件的边框样式为错误状态并附加工具提示；验证通过后执行回写。该方案增加了实现复杂度，但在性能和功能之间取得了合理平衡。若后续发现手动验证反馈的实现成本过高，可在实现阶段评估将行数据类型降级为可变 `class` 的可行性。

### 10. 行级高亮为何不由 `ZSliceActivationTracker` 驱动？

**决策**：行级高亮（当前选中单元格所在整行的轻微背景色）由 DataGrid 自身的 `CurrentItem` / `SelectedItem` 状态独立驱动，与 `ZSliceActivationTracker` 输出的 z 切片级高亮完全分离。

**理由**：需求定义「行级高亮」为当前选中单元格所在的整行附加轻微背景色，其语义是「标识当前操作焦点所在的行」。若由 `ZSliceActivationTracker` 驱动，会导致当前激活 z 切片内的**所有行**都获得行级高亮，混淆了「操作焦点行」与「当前激活切片」两个不同语义。DataGrid 内置的 `CurrentItem` / `SelectedItem` 机制天然支持选中行样式的驱动，通过样式选择器即可实现行级高亮，无需额外逻辑。`ZSliceActivationTracker` 仅负责 z 切片级高亮（当前激活切片的所有行），两者视觉样式不同、驱动源不同、语义独立。

### 11. 编辑回写为何采用即时同步而非显式保存？

**决策**：采用 **即时同步** 模式，编辑完成后立即调用数据模型写接口更新原始数据，同时引入 **脏标记** 机制提示未保存状态。

**理由**：标定数据编辑场景中，用户通常期望所见即所得——修改一个单元格后立即看到图表联动更新。显式保存模式（需点击保存按钮后才生效）会增加操作步骤，且若用户忘记保存就切换变量，可能导致修改丢失。即时同步模式通过 `INotifyPropertyChanged` 确保表格和图表的联动刷新是实时的。脏标记机制在控件层面提示用户「数据已修改但尚未持久化到后端存储」，由后端标定系统负责最终的保存/刷写动作。这兼顾了即时反馈和持久化安全。

## 修订说明（v3）

| 审查意见 | 修改措施 |
|---------|---------|
| 行级高亮由 `ZSliceActivationTracker` 驱动，与需求定义矛盾，会导致当前激活 z 切片内的所有行都获得行级高亮 | 修正「高亮叠加策略」：明确行级高亮由 DataGrid `CurrentItem` / `SelectedItem` 独立驱动；z 切片级高亮仅由 `ZSliceActivationTracker` 驱动。两者视觉样式、驱动源、语义完全分离。新增「设计决策 10」阐述分离理由 |
| z 列采用「首行显示」方案，在虚拟化滚动下首行滚出后该切片剩余可见行无 z 值标识 | 修改「设计决策 2」：z 列展示策略从「首行显示」改为「每行重复显示」，`CubeRowData` 的 `ZValue` 在每一行都渲染，彻底消除虚拟化滚动下的信息丢失风险 |
| `ICurveData` / `IMapData` / `ICubeData` 缺少具体方法签名，后端实现方和控件开发者无法确定契约边界 | 在三个接口的描述中补充最小契约成员。`ICurveData` 声明 `XAxisLength`、`XValues`、`ZValues`、`SetXValue`、`SetZValue`；`IMapData` 声明 `XAxisLength`、`YAxisLength`、`GetValue`、`SetValue`、断点序列；`ICubeData` 声明三维索引读写、三轴长度、断点序列 |
| `IChartPresenter` 缺少关键契约定义 | 补充最小契约成员：`LoadData(ICalibrationData)`、`Refresh()`、`VisualRoot { get; }`、`RenderFailed` 事件。同时解决 v1 审查中 `AttachTo(Control host)` 与「无 UI 依赖」描述的矛盾：模块描述统一为「允许包含轻量的 Avalonia 基础类型依赖」；接口不暴露 `AttachTo` 方法，改由 `VisualRoot` 属性让 Editor 直接嵌入 |
| `ICurveTablePresenter` 缺少接口方法签名 | 补充最小契约成员：`LoadData(ICurveData)`、`VisualRoot { get; }`、`SelectedCellChanged` 事件（参数携带列索引和行标识 X/Z）、`CellEditCompleted` 事件（参数携带列索引、行标识 X/Z、新数值）、`Refresh()` |
| 数值精度和小数位数一致性展示需求未响应 | `ICalibrationData` 新增 `FormatString` 和 `DisplayPrecision` 属性；在「关键行为契约」中新增「精度一致性展示」小节，明确精度信息从数据契约层 → 表格适配层 / `ICurveTablePresenter` → 图表层的流转路径 |
| 列宽/行高自适应或手动调整需求未响应 | 在「关键行为契约」中新增「列宽/行高调整策略」小节：DataGrid 启用 `CanUserResizeColumns` 并设置 `MinWidth` / `MaxWidth`；明确 Avalonia DataGrid 不支持 `CanUserResizeRows`，行高通过 `RowHeight` / `MinRowHeight` / `MaxRowHeight` 控制；`CurveEditor` 的列宽调整由 `ICurveTablePresenter` 实现方决定 |
| 数据编辑后的变更反馈机制未明确 | 在「单元格编辑与数据回写」中明确采用「即时同步 + 脏标记」模式：编辑完成后立即调用数据模型写接口，`ICalibrationData` 实现方标记 Dirty，`CalibrationEditorBase<T>` 在状态栏展示未保存提示。后端负责最终持久化 |
| 变量选择下拉框显示格式 `<曲线变量组> VariableName` 缺少数据契约支撑 | `ICalibrationData` 新增 `GroupName` 和 `DisplayName` 属性；`DisplayName` 默认由 `GroupName` 和 `VariableName` 组合生成，也可由后端直接提供格式化后的文本 |
| 顶部信息栏单位/轴信息展示格式未定义 | `CalibrationEditorBase<T>` 职责中补充默认格式模板 `[{Unit}] x: {XAxisName} [{XAxisUnit}] ...`，不存在的轴自动省略；子类可通过覆盖 `BuildInfoText()` 自定义 |
| 空状态与边界条件处理不完整 | 「变量切换流程」中补充完整空状态机：`ItemsSource` null/空列表、`SelectedVariable` null 时调用 `OnEnterEmptyState()`；为 `CalibrationEditorBase<T>` 定义 `OnEnterEmptyState` / `OnExitEmptyState` 虚方法 |
| 图表交互深度需求未在设计中确定 | 「设计决策 5」中明确 MVP 阶段仅支持默认视角静态展示，缩放/平移/旋转留待后续迭代；若后续支持交互，通过扩展方法或新增事件补充契约 |
| `readonly struct` 编辑验证反馈路径未充分考虑 | 新增「设计决策 9」：明确保持 `readonly struct` 但放弃 DataGrid 内置验证模板，采用完全手动的 `CellEditEnding` 事件拦截 + 编辑模板状态控制。记录权衡理由（GC 效率 vs 实现复杂度），保留降级为 `class` 的评估空间 |
| `ITableDataSource` 与不同行数据类型的泛型关系未明确 | 将 `ITableDataSource` 泛型化为 `ITableDataSource<TRow> where TRow : struct`，返回 `IReadOnlyList<TRow>`；`MapEditor` 实例化为 `ITableDataSource<MapRowData>`，`CubeEditor` 实例化为 `ITableDataSource<CubeRowData>` |
| `CanUserResizeRows` 在 Avalonia DataGrid 中不存在（v1 审查问题） | 移除设计文档中所有关于 `CanUserResizeRows` 的声明；行高调整策略改为通过 `RowHeight` / `MinRowHeight` / `MaxRowHeight` 控制，或在外层提供独立缩放机制 |
| `UnloadingRow` 事件在 Avalonia DataGrid 中不存在（v1 审查问题） | 移除对 `UnloadingRow` 的依赖；z 切片高亮联动机制改为仅在 `LoadingRow` 事件中处理：总是先清除 `active-z-slice` 样式类，再根据当前激活 `ZIndex` 决定是否重新附加，天然覆盖回收复用场景 |
| `IChartPresenter.AttachTo(Control host)` 与模块描述矛盾（v1 审查问题） | 移除 `AttachTo` 方法；`IChartPresenter` 改为暴露 `VisualRoot { get; }` 属性（由实现方返回内部创建的控件），Editor 直接将其嵌入视觉树。模块描述统一为「允许包含轻量的 Avalonia 基础类型依赖」 |
