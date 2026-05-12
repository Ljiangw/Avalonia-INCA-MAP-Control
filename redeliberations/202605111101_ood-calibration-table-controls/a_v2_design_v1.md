# Avalonia 标定数据表格编辑控件 — 架构级 OOD 设计方案 v4

## 概述

本方案设计一组基于 Avalonia V12 的标定数据表格编辑控件，面向汽车 ECU 标定工具场景。在统一交互范式下支撑 1 维（曲线）、2 维（Map）、3 维（数据立方体）三种标定数据的可视化编辑，并将控件与后端数据模型、图表渲染引擎解耦，确保可复用性和可替换性。

相对于 v3 方案，v4 的核心修订集中在：
- **数据契约层补充**：`ICalibrationData` 及各维度特化接口补全核心方法签名与精度展示契约；`ITableDataSource` 泛型化以消除类型歧义
- **表格交互策略明确**：行级高亮与 z 切片级高亮的驱动源分离；`CubeEditor` 的 z 列改为每行重复显示以保障虚拟化滚动下的信息完整性
- **图表与编辑契约补全**：`IChartPresenter` 和 `ICurveTablePresenter` 明确最小接口契约；图表交互深度按阶段划分
- **变更反馈与列宽策略**：明确即时同步的回写模式、脏标记机制、列宽/行高调整策略

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
| **图表抽象层** | 定义折线图和 3D 曲面图的渲染契约，隔离具体图表库实现。 | 被 Views 层依赖，自身无 UI 框架依赖（接收原始数据） |
| **激活追踪层** | CubeEditor 中 z 切片激活状态的专门管理者，将表格行/单元格选中事件映射为 z 值切换。 | 依赖表格适配层的 `CubeRowData`，被 CubeEditor 使用 |
| **Curve 表格契约层** | `CurveEditor` 专用的横向表格视图契约（`ICurveTablePresenter`），将一维 x/z 数据映射为两行横向表格，处理选中、编辑与高亮。 | 依赖数据契约层，被 CurveEditor 使用 |

依赖规则：数据契约层为最底层，无 outward 依赖；适配层和图表抽象层仅依赖数据契约；激活追踪层仅依赖表格适配层；Curve 表格契约层仅依赖数据契约层；Views 层可依赖所有下层模块，但下层模块不可反向依赖 Views 层。

## 核心抽象

### `ICalibrationData`（接口）

**角色**：所有维度标定数据的公共契约。

**职责**：暴露标定变量的元信息（名称、变量组、显示名称、单位、各轴变量名及单位）、数值精度展示策略，以及数据变更通知能力。

**协作**：被 `ICurveData`、`IMapData`、`ICubeData` 继承；被 `CalibrationEditorBase<T>` 作为泛型约束使用；被表格适配层、图表抽象层和 Curve 表格契约层消费。

**类型形态**：接口，**明确继承 `INotifyPropertyChanged`**。具体数据模型由后端标定系统定义，控件组只消费契约，不拥有实现。将 `INotifyPropertyChanged` 纳入继承列表，强制后端实现方具备数据变更通知能力，确保单元格编辑后绑定系统能正确感知变更并刷新 UI。

**核心契约成员**：
- `string VariableName { get; }` — 变量原始名称
- `string GroupName { get; }` — 变量所属分组名称（如「曲线变量组」），用于下拉框显示格式化
- `string DisplayName { get; }` — 格式化后的显示文本（如 `<曲线变量组> AllMkn_n_AirMax`），供下拉框直接绑定展示
- `string Unit { get; }` — 变量单位
- `IReadOnlyList<AxisInfo> Axes { get; }` — 各轴信息（轴名称、轴单位），轴顺序固定为 x → y → z
- `int DisplayPrecision { get; }` — 数值展示精度（小数位数），供表格单元格、图表轴标签统一使用
- `string FormatString { get; }` — 可选的自定义格式字符串（如 `"F3"`），优先级高于 `DisplayPrecision`

### `ICurveData` / `IMapData` / `ICubeData`（接口）

**角色**：维度特化的数据契约。

**职责**：在 `ICalibrationData` 基础上，各自声明维度特定的数据访问方式——`ICurveData` 提供一维 x/z 值序列的读写访问，`IMapData` 提供二维 (x,y)→z 的矩阵读写访问，`ICubeData` 提供三维 (x,y,z)→value 的张量读写访问。

**协作**：作为 `CalibrationEditorBase<T>` 的泛型参数具体化类型；被各自的表格适配器或 Curve 表格呈现器消费。

**类型形态**：接口，继承自 `ICalibrationData`。不采用抽象类，因为后端数据模型可能已经继承自其他领域基类，接口避免单继承限制。

**核心契约成员**：

`ICurveData`：
- `int Count { get; }` — 数据点数量
- `IReadOnlyList<double> XValues { get; }` — x 轴（自变量）值序列
- `IReadOnlyList<double> ZValues { get; }` — z 轴（因变量）值序列
- `void SetXValue(int index, double value)` — 修改指定索引的 x 值
- `void SetZValue(int index, double value)` — 修改指定索引的 z 值

`IMapData`：
- `int XAxisLength { get; }` / `int YAxisLength { get; }` — 各轴维度长度
- `IReadOnlyList<double> XValues { get; }` — x 轴刻度值（列头来源）
- `IReadOnlyList<double> YValues { get; }` — y 轴刻度值（行头来源）
- `double GetValue(int xIndex, int yIndex)` — 读取矩阵指定位置的 z 值
- `void SetValue(int xIndex, int yIndex, double value)` — 写入矩阵指定位置

`ICubeData`：
- `int XAxisLength { get; }` / `int YAxisLength { get; }` / `int ZAxisLength { get; }` — 各轴维度长度
- `IReadOnlyList<double> XValues { get; }` — x 轴刻度值
- `IReadOnlyList<double> YValues { get; }` — y 轴刻度值
- `IReadOnlyList<double> ZValues { get; }` — z 轴刻度值
- `double GetValue(int xIndex, int yIndex, int zIndex)` — 读取张量指定位置的值
- `void SetValue(int xIndex, int yIndex, int zIndex, double value)` — 写入张量指定位置

### `CalibrationEditorBase<T>`（抽象类）

**角色**：三个标定编辑控件的公共基座。

**职责**：
- 定义并管理 Avalonia 依赖属性：`ItemsSource`（`IList<T>` 类型，绑定到变量选择下拉框）、`SelectedVariable`（当前选中的标定数据项）
- 提供统一的顶部信息栏模板结构（左侧 `ComboBox`、右侧单位/轴信息展示）
- 处理变量切换时的选中变更事件，向子类发出数据切换通知
- 管理空状态：当 `ItemsSource` 为 `null`、空列表，或 `SelectedVariable` 为 `null` 时，进入空状态展示友好提示

**协作**：内部聚合一个 `ComboBox` 和一个信息展示面板；通过模板约定或受保护的虚方法向子类（`CurveEditor`、`MapEditor`、`CubeEditor`）暴露数据切换钩子。

**类型形态**：泛型抽象类（`T : ICalibrationData`），继承自 Avalonia `TemplatedControl`。选用抽象类而非接口，是因为需要封装具体的依赖属性注册、模板部件约定和公共视觉状态逻辑。

**信息栏文本格式化**：默认格式模板为 `[{Unit}] x: {XAxisName} [{XAxisUnit}] y: {YAxisName} [{YAxisUnit}] z: {ZAxisName} [{ZAxisUnit}]`，其中不存在的轴信息字段自动省略。子类可通过覆盖受保护的虚方法 `FormatInfoText(T variable)` 提供自定义格式。

**空状态管理**：定义受保护的虚方法 `OnEnterEmptyState(string reason)` 和 `OnExitEmptyState()`，供子类在空状态切换时执行视觉状态更新（如展示/隐藏占位提示）。

### `CurveEditor` / `MapEditor` / `CubeEditor`（密封类）

**角色**：面向终端使用者的三个具体控件。

**职责**：
- `CurveEditor`：主内容区上方为折线图区域、下方为 1 维两行表格（x 行 + z 行）。表格采用横向布局，通过 `ICurveTablePresenter` 构建，不使用 `DataGrid` 纵向行模型
- `MapEditor`：主内容区左侧为 3D 曲面图、右侧为 2 维表格（x 列头 + y 行头 + z 数据单元格）
- `CubeEditor`：主内容区左侧为 3D 曲面图（展示当前激活 z 切片的 (x, y) → value 数据）、右侧为平铺 3 维表格（z/y 双列行头 + x 列头 + 数据单元格）。表格以平铺形式展示完整数据立方体的所有 (z, y) 行 × x 列，而非仅展示当前激活切片。z 切片激活由表格内的选中/导航交互驱动

**协作**：`MapEditor` 与 `CubeEditor` 各自内部持有对应的表格适配器实例和图表呈现器实例；`CubeEditor` 额外持有 `ZSliceActivationTracker`。`CurveEditor` 内部持有 `ICurveTablePresenter` 实例和 `IChartPresenter` 实例。

**类型形态**：密封类，继承 `CalibrationEditorBase<T>`（`T` 分别为 `ICurveData`、`IMapData`、`ICubeData`）。密封是因为这些控件是领域终态产品，不存在合理的进一步派生场景。

### `ICurveTablePresenter`（接口）

**角色**：`CurveEditor` 横向两行表格的视图构建与交互处理契约。

**职责**：
- 接收 `ICurveData`，构建两行横向表格的视觉树。第一行展示所有 x 轴值，第二行展示所有 z 轴值，列数与数据点数量一致
- 管理单元格选中状态（当前活动单元格）和单元格编辑状态（进入/退出编辑模式）
- 提供单元格编辑完成事件，携带数据点索引（列位置）和行标识（X 行或 Z 行），供 `CurveEditor` 回写到原始数据模型
- 提供选中单元格变更通知，供 `CurveEditor` 同步驱动折线图的数据点高亮
- 暴露视觉树根元素，供 `CurveEditor` 嵌入到主内容区布局中
- 支持列宽调整（手动拖拽或自适应），保证不同数据量下的可读性

**协作**：被 `CurveEditor` 内部持有；向下消费 `ICurveData`；向上通过事件通知 `CurveEditor` 单元格编辑完成和选中变更。实现方基于 Avalonia 自定义布局（如 `Grid` + `ItemsControl` 或自定义 `Panel`）构建横向视觉树，不依赖 `DataGrid`。

**类型形态**：接口。

**核心契约成员**：
- `void LoadData(ICurveData data)` — 加载数据并重建横向表格视觉树
- `Control VisualRoot { get; }` — 横向表格的视觉树根元素
- `EventHandler<CurveCellSelectedEventArgs> SelectedCellChanged` — 选中单元格变更事件，参数包含列索引、行标识（X/Z）
- `EventHandler<CurveCellEditCompletedEventArgs> CellEditCompleted` — 单元格编辑完成事件，参数包含列索引、行标识（X/Z）、解析后的数值
- `EventHandler<CurveCellValidationFailedEventArgs> CellValidationFailed` — 单元格输入验证失败事件，参数包含列索引、行标识、原始输入文本、失败原因
- `void Refresh()` — 请求刷新显示（在数据模型触发 `PropertyChanged` 后由 `CurveEditor` 调用）
- `bool CanResizeColumns { get; set; }` — 是否允许用户拖拽调整列宽

### `ITableDataSource<TRow>`（接口）

**角色**：多维数据到二维纵向表格结构的投影器。供 `MapEditor` 与 `CubeEditor` 使用。

**职责**：将原始标定数据（二维矩阵、三维张量）转换为适合 Avalonia `DataGrid` 绑定的行集合。各维度实现返回各自强类型的行数据对象集合，行数据对象内联携带原始数据坐标索引，以支撑编辑后的值回写。

**协作**：被 `MapEditor` 和 `CubeEditor` 内部使用，作为 `DataGrid` `ItemsSource` 的实际来源。编辑完成时，Editor 从强类型行数据对象中提取坐标索引，调用原始数据模型的写接口。

**类型形态**：泛型接口 `ITableDataSource<TRow> where TRow : struct`。`MapEditor` 与 `CubeEditor` 分别实例化为 `ITableDataSource<MapRowData>` 和 `ITableDataSource<CubeRowData>`，通过泛型参数消除返回 `object` 或基类型的歧义。

**核心契约成员**：
- `IReadOnlyList<TRow> Rows { get; }` — 投影后的行集合
- `void Refresh()` — 在原始数据变更后重新投影行集合

### `MapRowData` / `CubeRowData`（`readonly struct`）

**角色**：承载一行纵向表格数据的展示值及其在原始多维数据中的索引坐标。供 `MapEditor` 与 `CubeEditor` 使用。

**职责**：
- `MapRowData`：内联 `XIndex`、`YIndex` 属性，标识该单元格在二维矩阵中的坐标
- `CubeRowData`：内联 `ZIndex`、`YIndex` 属性，标识该数据行在三维张量中的坐标；同时承载 `ZValue` 和 `YValue` 文本（用于 z/y 双列行头展示）。每行对应一个 (z, y) 组合下所有 x 列的数据值

**协作**：由对应维度的表格适配器生成并填充，作为 `DataGrid` 的 `ItemsSource` 行集合。Editor 在编辑确认时从行数据对象中直接读取坐标索引属性，回写到原始数据模型。

**类型形态**：`readonly struct`。值类型避免堆分配；坐标索引作为结构体的属性直接内联，不通过接口引用传递，彻底消除接口装箱带来的 GC 压力。

**编辑可行性说明**：`readonly struct` 的不可变性意味着 `DataGrid` 的标准双向绑定自动提交机制无法直接更新行数据源。设计明确编辑回写完全通过 `CellEditEnding` / `CellEditEnded` 事件拦截实现——Editor 从行数据对象提取坐标索引后，直接调用后端数据模型的写接口，再触发 `PropertyChanged` 刷新 UI。`DataGrid` 的验证模板（`IDataErrorInfo` / `INotifyDataErrorInfo`）在此场景下可能无法直接作用于不可变结构体，验证反馈需通过事件拦截 + 编辑模板视觉状态控制实现（见「错误处理策略」）。

### `IChartPresenter`（接口）

**角色**：图表渲染的抽象契约。

**职责**：接收标定数据，在宿主控件区域内渲染折线图（1D）或 3D 曲面图（2D/3D）。阶段 1（MVP）仅支持默认视角的静态展示；阶段 2 可扩展为支持鼠标拖拽旋转（3D）和滚轮缩放。

**协作**：被 `CurveEditor`、`MapEditor`、`CubeEditor` 内部持有。实现方可选用 OxyPlot、LiveCharts、ScottPlot、自定义 Skia 渲染等具体技术。

**类型形态**：接口。因为 Avalonia V12 生态中没有内置的 3D 曲面图标准控件，图表实现必须可替换，接口是最低耦合的抽象形式。

**核心契约成员**：
- `void LoadData(ICalibrationData data)` — 加载标定数据（实际消费时按维度向下转型为 `ICurveData` / `IMapData` / `ICubeData`）
- `void AttachTo(Control host)` — 将图表渲染目标关联到指定的宿主控件
- `void Refresh()` — 显式触发重绘
- `EventHandler<ChartRenderFailedEventArgs> RenderFailed` — 渲染失败事件，供宿主控件降级展示占位矩形或错误提示
- `bool EnableInteraction { get; set; }` — 是否启用交互（旋转、缩放等），默认 `false`（阶段 1）
- `EventHandler<ChartInteractionEventArgs> InteractionOccurred` — 交互事件（阶段 2 可选），通知宿主图表视角变化

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
- `ItemsSource` 为 `null` 或空列表时，`ComboBox` 无选项，`SelectedVariable` 为 `null`，控件进入空状态，主内容区展示占位提示（如 "无可用的标定变量"）
- `SelectedVariable` 为 `null` 时，信息栏显示默认文本，图表区域和表格区域均进入空状态
- 变量切换过程中（从旧数据卸载到新数据加载完成），`CalibrationEditorBase<T>` 调用 `OnEnterEmptyState` 进入过渡状态，防止旧数据残留显示

### 单元格编辑与数据回写

**编辑回写模式：即时同步**

编辑完成后立即调用数据模型的写接口更新对应位置，不采用显式保存按钮模式。理由：标定数据编辑场景要求用户能够实时看到修改效果（尤其是图表联动），即时同步符合工程工具的使用习惯。

#### MapEditor / CubeEditor（DataGrid 纵向行模型）

用户触发单元格进入编辑模式（双击进入编辑，单击仅用于选中/导航）→ 单元格展示编辑模板（如 `TextBox`）→ 用户输入数值 → 输入完成时（Enter/失去焦点），Editor 拦截 `CellEditEnding` 事件，解析输入文本为数值：
- 若解析失败 → 阻止编辑提交，通过编辑模板视觉状态（红色边框 + 工具提示）就地反馈错误，不调用数据模型写接口
- 若解析成功 → 从对应类型的行数据对象中提取坐标索引属性，调用原始数据模型的写方法更新对应位置 → 模型触发 `INotifyPropertyChanged.PropertyChanged` → 绑定系统刷新关联单元格；若该单元格值同时影响图表，图表呈现器接收通知后重绘

在 CubeEditor 中，编辑操作发生在平铺表格的任意数据单元格。编辑完成后，Editor 从该单元格对应的 `CubeRowData` 中读取 `ZIndex`、`YIndex`，并结合该单元格所在的 x 列索引 `XIndex`，调用 `ICubeData` 的写接口更新三维张量中对应位置的值。

#### CurveEditor（横向自定义布局）

用户触发单元格进入编辑模式（双击进入编辑，单击仅用于选中/导航）→ `ICurveTablePresenter` 在对应列的对应行（X 行或 Z 行）展示编辑控件（如 `TextBox`）→ 用户输入数值 → 输入完成时，`ICurveTablePresenter` 内部验证输入：
- 若验证失败 → 触发 `CellValidationFailed` 事件，编辑控件保持红色错误状态，不提交修改
- 若验证成功 → 触发 `CellEditCompleted` 事件，携带数据点索引（列位置）、行标识（X 或 Z）以及解析后的数值 → `CurveEditor` 在事件处理器中调用 `ICurveData` 的写接口更新对应位置的数据 → 模型触发 `INotifyPropertyChanged.PropertyChanged` → `ICurveTablePresenter` 刷新对应单元格显示，`IChartPresenter` 重绘折线图。

**变更反馈机制**：
- 即时同步模式下，编辑成功的反馈为单元格值即时更新 + 图表即时重绘
- 若实现阶段需要增强反馈，可在 `CalibrationEditorBase<T>` 的状态栏区域展示「未保存更改计数」或「最后编辑时间」提示（非强制性）
- 数据模型写接口抛出异常时（如越界、只读），控件捕获异常并通过单元格级别的错误提示展示失败原因

### z 切片激活与 3D 视图联动（CubeEditor）

用户点击/键盘导航到表格中的某个单元格或行 → `CubeEditor` 将选中的 `CubeRowData` 传递给 `ZSliceActivationTracker` → Tracker 读取该行的 `ZIndex` 属性，与当前激活 z 比较：若相同则无动作；若不同则更新激活 z 值并发出变更事件 → `CubeEditor` 在事件处理器中通过 `Dispatcher.UIThread.Post` 调度 UI 更新：通知图表呈现器切换数据源到新的 z 切片矩阵，同时触发表格的视觉状态更新（z 切片级高亮）。

当用户点击 z 列分组区域（或属于某 z 切片的 z 单元格）时，`ZSliceActivationTracker` 将该 z 值设为激活切片，并通知 `CubeEditor` 选中该切片内的第一个数据单元格。

键盘跨 z 切片导航时（如从当前 z 切片的最后一行 y 值按方向键移入下一 z 切片的第一行 y 值），`ZSliceActivationTracker` 内部引入短时间防抖（如 100–200ms），仅在用户停止快速导航后才真正刷新 3D 曲面图，避免频繁重绘造成卡顿。

### z 切片级高亮状态与 DataGrid 虚拟化联动（CubeEditor）

`DataGrid` 仅对当前可视区域内的行创建视觉树实例（行虚拟化）。`ZSliceActivationTracker` 输出的激活 `ZIndex` 需通过以下机制传递到行视觉元素：

1. **`CubeEditor` 订阅 `ZSliceActivationTracker` 的激活变更事件**。当激活 `ZIndex` 变化时，`CubeEditor` 执行两项操作：通知 `IChartPresenter` 刷新 3D 曲面图；遍历当前已加载的 `DataGridRow` 容器，为属于激活 z 切片的行附加样式类（如 `active-z-slice`），为非激活行移除该样式类。

2. **`CubeEditor` 订阅 `DataGrid` 的 `LoadingRow` 事件**。当用户滚动导致新行进入可视区域时，`DataGrid` 创建对应的行视觉元素并触发 `LoadingRow` 事件。`CubeEditor` 在事件处理器中读取该行的 `DataContext`（`CubeRowData`），若其 `ZIndex` 等于当前激活 `ZIndex`，则立即为新行附加 z 切片高亮样式类。这确保了虚拟化机制下，新进入可视区域的行也能正确显示激活切片的高亮状态。

3. **`CubeEditor` 订阅 `DataGrid` 的 `UnloadingRow` 事件**（若可用）。在行视觉元素被回收前清理其附加的样式类，避免回收后状态残留影响后续行的视觉呈现。

通过上述机制，`ZSliceActivationTracker` 的状态输出（纯 .NET 事件）与 `DataGrid` 视觉树的状态同步解耦：Tracker 不依赖 Avalonia 视觉类型，而 `CubeEditor` 作为 UI 层协调者负责将逻辑状态映射到视觉状态，兼容 `DataGrid` 的行虚拟化和行回收机制。

### 高亮叠加策略

三种高亮在 CubeEditor 中可能同时存在，各自由独立的驱动源控制，视觉优先级从高到低为：

1. **单元格级高亮**（蓝色背景，当前活动单元格）— 由 `DataGrid` 的选中状态（`CurrentCell` / `SelectedItem`）直接驱动，通过 `DataGrid` 内置的选中单元格样式实现
2. **行级高亮**（轻微背景色，当前选中单元格所在的整行）— 由 `DataGrid` 的 `CurrentItem` / `SelectedItem` 或当前活动单元格所在行驱动，通过 `DataGrid` 内置的选中行样式（如 `DataGridRow` 的 `IsSelected` 状态样式）或基于 `CurrentCell` 状态的自定义样式选择器实现。**注意：行级高亮不依赖 `ZSliceActivationTracker`**
3. **z 切片级高亮**（左侧边框或分组背景色，当前激活切片的所有行）— 仅由 `ZSliceActivationTracker` 输出的激活 `ZIndex` 驱动，通过 `CubeEditor` 动态为 `DataGridRow` 附加/移除样式类实现

三者的驱动源完全独立：单元格级和行级由 `DataGrid` 自身的选中/导航状态驱动；z 切片级由 `ZSliceActivationTracker` 的语义状态驱动。`CubeEditor` 作为协调者分别订阅这两类状态变更，独立更新对应的视觉样式，避免状态耦合导致的逻辑矛盾。

### 列宽/行高调整策略

#### MapEditor / CubeEditor（DataGrid）

- `CanUserResizeColumns` 对所有动态生成的 x 轴数据列启用，允许用户拖拽调整列宽，保证不同数据量下的可读性
- `CanUserResizeRows` 启用，允许用户调整行高
- 动态生成列的宽度初始化策略：`DataGrid` 列的 `Width` 初始化为 `DataGridLength.Auto`（根据内容自适应），用户调整后保存为星号或像素宽度。最小列宽约束为 `MinWidth = 60`，防止列宽过窄导致数值截断
- z 列和 y 列作为行头列，固定较窄宽度（如 `Width = 80`），允许用户调整但设置最小宽度限制

#### CurveEditor（横向自定义布局）

- `ICurveTablePresenter` 的实现方负责列宽管理。通过暴露 `CanResizeColumns` 属性控制是否允许用户拖拽调整列宽
- 列宽初始化采用均分策略（所有列等宽），或根据数值字符串长度自适应
- 实现方可通过自定义 `Panel` 的 `MeasureOverride` / `ArrangeOverride` 精确控制列宽分配逻辑

### 数值精度一致性展示

精度信息从 `ICalibrationData.FormatString` 或 `DisplayPrecision` 属性获取，流转路径如下：

1. **表格单元格**：`MapTableAdapter` / `CubeTableAdapter` 在生成行数据时，将 `FormatString` 传递给单元格的绑定转换器（如 `StringFormatConverter`），统一控制单元格数值的显示格式
2. **图表轴标签**：`IChartPresenter` 在渲染轴标签时，消费 `ICalibrationData` 的 `FormatString` 或 `DisplayPrecision`，确保图表与表格的数值展示精度一致
3. **CurveEditor 横向表格**：`ICurveTablePresenter` 的实现方在构建单元格文本时，直接应用 `ICurveData` 的 `FormatString`

若 `FormatString` 为 `null`，则回退到基于 `DisplayPrecision` 的默认格式（如 `"F" + DisplayPrecision`）。

## 错误处理策略

### 输入验证错误

单元格编辑阶段的数值解析失败（如非法字符、越界值）属于局部错误。错误处理策略为「就地反馈」：在单元格层面通过红色边框或工具提示标识错误，阻止该次编辑回写，但不阻断用户继续编辑其他单元格或切换变量。

#### MapEditor / CubeEditor（DataGrid + readonly struct）

由于行数据源为 `readonly struct`，`DataGrid` 的标准双向绑定自动提交机制无法直接更新结构体实例，`DataGrid` 内置的数据验证模板（`IDataErrorInfo` / `INotifyDataErrorInfo`）在此场景下的作用受限——验证错误无法通过绑定系统自动反映到单元格视觉状态。

**验证反馈路径**：
1. **事件拦截为主**：Editor 在 `CellEditEnding` 事件中拦截编辑值，执行数值解析和范围验证
2. **手动视觉状态控制**：验证失败时，Editor 通过直接操作单元格的 `TextBox` 编辑模板（如设置 `BorderBrush` 为红色、显示 `ToolTip`）实现就地反馈，并设置 `e.Cancel = true` 阻止编辑提交
3. **编辑模板状态绑定**：编辑模板内的 `TextBox` 可绑定到 `ValidationErrors` 附加属性（由 Editor 在事件拦截中动态设置），利用 Avalonia 的验证错误视觉状态（红色边框）实现标准化错误呈现

此方案放弃了 `DataGrid` 行数据级别的验证模板，完全采用「事件拦截 + 编辑模板手动状态控制」的验证反馈路径。权衡记录：`readonly struct` 的零 GC 优势在标定数据大数据量场景下具有显著性能价值，值得以略高的验证实现复杂度为代价换取。若实现阶段发现验证反馈实现过于复杂，可将行数据类型降级为可变的 `class`，保留架构回退空间。

#### CurveEditor（横向自定义布局）

`ICurveTablePresenter` 的实现方全权负责输入验证和错误呈现。在编辑控件层面拦截输入变更或完成事件，验证失败时直接在编辑控件上设置错误视觉状态（红色边框、工具提示），并触发 `CellValidationFailed` 事件通知 `CurveEditor`。

### 数据绑定不匹配错误

控件绑定到不符合维度期望的数据模型（如 `CubeEditor` 接收的 `ICubeData` 中 z 维度长度为 0）属于结构性错误。控件应进入「空状态」展示友好提示（如 "无有效数据"），而非抛出未处理异常导致控件白屏。

`CalibrationEditorBase<T>` 在变量切换时检查 `SelectedVariable` 的维度有效性，无效时调用 `OnEnterEmptyState` 进入空状态。

### 图表渲染错误

图表库渲染失败（如 GPU 不可用、3D 数据异常）属于可降级错误。`IChartPresenter` 实现方应内部捕获渲染异常，通过 `RenderFailed` 事件通知宿主控件，宿主降级为显示占位矩形或错误图标，保证表格编辑功能不受影响。

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

**决策**：不追求原生跨行合并，采用「z 值每行重复显示 + 视觉分组线 + z 切片级高亮」的组合方案。

**理由**：Avalonia 的 `DataGrid`（包括 V12）不提供 WPF 中 `RowSpan` 式的跨行单元格能力。若强行在模板层模拟合并单元格，会严重破坏虚拟化机制和行回收，在数据量大时性能急剧下降。

原 v3 方案中「z 值首行显示」策略在虚拟化滚动下存在致命缺陷：当某 z 切片的首行滚出可视区域后，该切片剩余的所有可见行在 z 列中将没有任何 z 值标识，用户无法判断当前行属于哪个 z 切片。

v4 调整为「z 值在切片的每一行都重复显示」，通过以下机制保持视觉分组感：
1. 同一切片内所有行的 z 列均显示相同的 z 值文本，确保无论哪一行在可视区域内，用户都能看到所属 z 切片标识
2. 配合 z 切片边界处的水平分隔线样式（`border-bottom` 加粗），形成视觉分组
3. 激活切片的整组行通过 z 切片级高亮（左侧边框或轻微背景色）统一标识

此方案以牺牲部分视觉简洁性（z 值重复显示）换取信息完整性（滚动场景下 z 标识不丢失），且完全兼容 `DataGrid` 的虚拟化。

### 3. 为什么用抽象基类 `CalibrationEditorBase<T>` 而非纯组合或接口？

**决策**：采用泛型抽象类承载公共 UI 结构和依赖属性。

**理由**：三个控件共享的不仅是行为契约，还有大量的具体 UI 结构——顶部栏的 `ComboBox`、单位信息面板、变量切换后的数据刷新管线。抽象类可以在 Avalonia 的类型系统中直接注册这些公共依赖属性（`AvaloniaProperty.Register`），并约定公共 ControlTemplate 的部件名称（如 `PART_VariableSelector`、`PART_InfoPanel`）。若用纯接口，每个控件需要重复注册相同的依赖属性和模板逻辑，违背 DRY 原则。组合模式（如注入一个「顶部栏管理器」对象）在 Avalonia 的模板/样式系统中反而增加不必要的间接层。

### 4. 表格编辑如何定位回原始数据？

**决策**：通过维度特化的 `readonly struct` 行数据对象建立可见单元格到原始多维索引的映射，坐标索引直接内联为结构体属性，取消独立的坐标接口抽象。

**理由**：Avalonia 表格控件（`DataGrid`）天然面向二维行集合（`IEnumerable` 的扁平行）。原始标定数据是 2D/3D 矩阵，无法直接作为 `ItemsSource`。引入表格适配层将矩阵投影为行集合后，每一行必须携带「我来自原始数据的哪个位置」这一元信息，否则编辑后无法回写。v2 已验证将坐标索引直接内联到各维度行数据类型（`MapRowData`、`CubeRowData`）中的方案，以强类型属性传递坐标信息，彻底消除接口装箱，同时保持编辑回写时的坐标定位能力。v3 中 `CubeRowData` 额外承载 `ZValue`、`YValue`，以支撑 z/y 双列行头的展示需求。v4 移除 `IsFirstRowInZSlice` 属性（因 z 值改为每行重复显示）。

### 5. 3D 曲面图没有 Avalonia 内置方案，如何设计？

**决策**：通过 `IChartPresenter` 接口抽象图表渲染，控件不耦合具体图表库。图表交互深度分阶段实现。

**理由**：Avalonia V12 标准控件集中不包含 3D 图表。社区中 OxyPlot、LiveCharts、ScottPlot 等库对 3D 的支持程度和 API 差异很大，且部分库在 V12 下的兼容性尚在发展。将图表渲染抽象为接口后，控件只关心「把数据给你，你在指定区域内画出来」。

**交互深度**：
- **阶段 1（MVP）**：仅支持默认视角的静态展示，`EnableInteraction` 默认为 `false`
- **阶段 2**：支持鼠标拖拽旋转（3D）和滚轮缩放，`EnableInteraction` 设为 `true`，`IChartPresenter` 通过 `InteractionOccurred` 事件通知宿主视角变化

实现阶段可以先以 2D 热力图或伪 3D 网格作为最小可行方案，后续替换为真正的 3D 引擎时不影响控件架构。这避免了在设计阶段就锁定可能不合适的图表技术。

### 6. CubeEditor 的 z 切片激活逻辑为何独立为 `ZSliceActivationTracker`？

**决策**：将 z 切片激活的状态管理和事件映射抽取为独立类，而非内嵌在 `CubeEditor` 中。

**理由**：CubeEditor 的表格同时承担多重职责——数据展示、单元格编辑、z 切片选择器。若将 z 切片激活逻辑（行→z 值提取、激活变更判断、防抖、通知联动、z 列分组首行点击处理）全部写在 `CubeEditor` 的代码后台，会导致该类臃肿且难以测试。`ZSliceActivationTracker` 封装了「从表格选中状态到 z 切片语义」的纯逻辑转换，不依赖 Avalonia 控件的具体视觉类型，仅消费 `CubeRowData` 行数据对象。其状态输出采用纯 .NET 事件（`EventHandler<T>`），由 `CubeEditor` 在事件处理器中调度 UI 线程更新，这使得 Tracker 逻辑可以在无头环境中单元测试，也便于后续调整 z 切片切换的交互策略。

### 7. 编辑触发方式：单击还是双击？

**决策**：表格数据单元格采用 **双击进入编辑** 模式，单击仅用于选中/导航。

**理由**：在标定数据表格中，用户频繁进行选中浏览（查看相邻单元格数值、跨行比较）。若单击即进入编辑，会频繁弹出 `TextBox` 编辑模板，干扰浏览体验。双击编辑是桌面表格工具（Excel、INCA 等标定工具）的广泛惯例，与用户的工程工具心智模型一致。行头列（z 列、y 列）不参与编辑，单击它们仅触发选中或 z 切片激活。

### 8. CurveEditor 的横向两行表格为何不用 DataGrid，而采用独立布局契约？

**决策**：`CurveEditor` 不使用 `DataGrid` 和 `ITableDataSource` 纵向行集合抽象，而采用独立的 `ICurveTablePresenter` 接口，由实现方基于 Avalonia 自定义布局容器构建横向视觉树。

**理由**：需求明确要求 `CurveEditor` "以两行形式展示：第一行为 x 轴（自变量）值，第二行为 z 轴（因变量）值"，这是一种矩阵转置的表格形态——固定两行、动态多列。而 `DataGrid` 的设计范式是纵向行集合（每行一个数据对象，列对应属性），与横向布局的语义天然冲突。若强行套用 `DataGrid`，需要为每个数据点动态生成一列，并将两个行对象（x 行和 z 行）的数组索引绑定到各列，实现复杂度极高且违背 `DataGrid` 的设计意图。`CurveEditor` 的数据规模通常较小（一维曲线的数据点数量有限），不需要 `DataGrid` 的行虚拟化、列排序等重型功能。通过 `ICurveTablePresenter` 接口将横向布局的构建和交互抽象化后，实现方可选用 `Grid` + `ItemsControl`、`UniformGrid` 或自定义 `Panel` 等轻量方案，精确控制列宽、单元格间距和选中高亮行为，与需求的横向两行展示完全对齐。同时，`CurveEditor` 与 `MapEditor`/`CubeEditor` 在表格实现上的差异被封装在各自的适配/呈现层中，不影响公共基类的统一性。

### 9. `readonly struct` 编辑验证的权衡

**决策**：保留 `readonly struct` 作为 `DataGrid` 行数据源，验证反馈完全通过事件拦截 + 编辑模板手动状态控制实现。

**理由**：`readonly struct` 的零堆分配特性在标定数据大数据量场景（如 CubeEditor 的数千行）下具有显著 GC 优势。其不可变性导致 `DataGrid` 标准验证模板无法直接作用，但编辑回写链路（`CellEditEnding` 事件拦截 → 坐标提取 → 后端模型写接口 → `PropertyChanged` 刷新）本身不依赖行数据的可变性。验证反馈通过在事件拦截中直接操作编辑模板的 `BorderBrush` 和 `ToolTip` 实现，虽然增加了实现复杂度，但局限于 `MapEditor`/`CubeEditor` 的单元格编辑事件处理器中，不影响整体架构的简洁性。若实现阶段验证反馈实现成本过高，保留将 `MapRowData`/`CubeRowData` 降级为可变 `class` 的回退选项。

## 修订说明（v3）

| 审查意见 | 修改措施 |
|---------|---------|
| **严重-问题1**：行级高亮与 z 切片级高亮驱动源混淆。`ZSliceActivationTracker` 输出激活 z 切片的 ZIndex，若用它驱动行级高亮会导致当前激活 z 切片内的所有行都获得行级高亮，与需求定义矛盾。 | 在「高亮叠加策略」中明确三者的独立驱动源：单元格级高亮由 `DataGrid` 的 `CurrentCell` / `SelectedItem` 驱动；行级高亮由 `DataGrid` 的 `CurrentItem` / `SelectedItem` 或当前活动单元格所在行驱动，**不依赖 `ZSliceActivationTracker`**；z 切片级高亮仅由 `ZSliceActivationTracker` 输出的激活 `ZIndex` 驱动。更新「关键行为契约」→「高亮叠加策略」小节，补充三者独立驱动来源的完整说明。 |
| **严重-问题2**：z 列首行显示方案在虚拟化滚动下丢失 z 值标识。当某 z 切片首行滚出可视区域后，该切片剩余可见行在 z 列中无 z 值。 | **调整 z 列展示策略**：将「z 值首行显示」改为「z 值每行重复显示」。同一切片内所有行的 z 列均显示相同的 z 值文本，确保滚动场景下信息不丢失；配合 z 切片边界水平分隔线和 z 切片级高亮保持视觉分组感。更新「核心抽象」→「`CubeRowData`」小节移除 `IsFirstRowInZSlice` 属性；更新「设计决策 2」详细阐述调整理由。 |
| **中等-问题3**：`ICurveData` / `IMapData` / `ICubeData` 缺少具体方法签名，后端实现方和控件开发者无法确定契约边界。 | 在三个接口的描述中补充**核心契约成员**列表。以自然语言描述每个关键方法/属性的语义，不给出完整 C# 代码签名。`ICurveData` 补充 `Count`、`XValues`、`ZValues`、读写方法；`IMapData` 补充 `XAxisLength`、`YAxisLength`、二维索引读写、`XValues`/`YValues`；`ICubeData` 补充三轴维度、三维索引读写、三轴刻度值序列。 |
| **中等-问题4**：`IChartPresenter` 缺少关键契约定义，无法确定数据传递、渲染目标、重绘触发方式。 | 在「核心抽象」→「`IChartPresenter`」小节补充核心契约成员：`LoadData`（数据加载）、`AttachTo`（宿主关联）、`Refresh`（显式重绘）、`RenderFailed`（渲染失败降级通知）、`EnableInteraction` / `InteractionOccurred`（阶段 2 交互支持）。 |
| **中等-问题5**：`ICurveTablePresenter` 缺少接口方法签名，未定义事件参数类型和视觉树根元素。 | 在「核心抽象」→「`ICurveTablePresenter`」小节补充核心契约成员：`LoadData`、`VisualRoot`、`SelectedCellChanged`、`CellEditCompleted`、`CellValidationFailed`、`Refresh`、`CanResizeColumns`。明确各事件参数携带的语义信息（列索引、行标识、数值等）。 |
| **中等-问题6**：数值精度和小数位数一致性展示需求未响应。 | 在 `ICalibrationData` 中新增 `DisplayPrecision`（`int`，小数位数）和 `FormatString`（`string`，自定义格式如 `"F3"`）属性。在「关键行为契约」中新增「数值精度一致性展示」小节，明确精度信息从数据契约层 → 表格适配层（通过绑定转换器）→ 图表渲染层的统一流转路径。 |
| **中等-问题7**：列宽/行高自适应或手动调整需求未响应。 | 在「关键行为契约」中新增「列宽/行高调整策略」小节。`MapEditor`/`CubeEditor`：启用 `CanUserResizeColumns` 和 `CanUserResizeRows`，动态 x 轴列初始化为 `Auto`，最小列宽约束 60；z/y 行头列固定宽度 80。`CurveEditor`：`ICurveTablePresenter` 暴露 `CanResizeColumns` 属性，实现方通过自定义布局控制列宽。 |
| **中等-问题8**：数据编辑后的变更反馈机制未明确。 | 在「关键行为契约」→「单元格编辑与数据回写」中明确采用**即时同步**模式：编辑完成后立即调用数据模型写接口，不采用显式保存按钮。补充变更反馈的具体形式（单元格即时更新 + 图表即时重绘），以及数据模型写异常时的降级处理策略。 |
| **轻微-问题9**：变量选择下拉框显示格式缺少数据契约支撑。 | 在 `ICalibrationData` 中新增 `GroupName`（变量所属分组）和 `DisplayName`（格式化后的显示文本，如 `<曲线变量组> AllMkn_n_AirMax`）属性。明确 `CalibrationEditorBase<T>` 中下拉框绑定到 `DisplayName`。 |
| **轻微-问题10**：顶部信息栏单位/轴信息展示格式未定义。 | 在「核心抽象」→「`CalibrationEditorBase<T>`」小节中补充信息栏默认格式模板 `[{Unit}] x: {XAxisName} [{XAxisUnit}] ...`，明确不存在的轴字段自动省略。同时声明子类可通过覆盖 `FormatInfoText` 虚方法自定义格式。 |
| **轻微-问题11**：空状态与边界条件处理不完整。 | 在「关键行为契约」→「变量切换流程」中补充空状态处理：明确 `ItemsSource` 为 `null`/空列表、`SelectedVariable` 为 `null` 时的行为。在「核心抽象」→「`CalibrationEditorBase<T>`」中补充 `OnEnterEmptyState` / `OnExitEmptyState` 受保护虚方法。 |
| **轻微-问题12**：图表交互深度需求未在设计中确定。 | 在「设计决策 5」中明确分阶段实现：阶段 1（MVP）仅支持默认视角静态展示，`EnableInteraction` 默认 `false`；阶段 2 支持鼠标拖拽旋转和滚轮缩放。`IChartPresenter` 补充 `EnableInteraction` 属性和 `InteractionOccurred` 事件为阶段 2 预留契约。 |
| **轻微-问题13**：`readonly struct` 编辑验证反馈路径未充分考虑，`DataGrid` 内置验证模板可能无法正常工作。 | 在「错误处理策略」→「输入验证错误」中新增 `MapEditor`/`CubeEditor` 的详细验证反馈路径：明确采用「`CellEditEnding` 事件拦截 + 编辑模板手动视觉状态控制（`BorderBrush`、`ToolTip`）」方案，放弃 `DataGrid` 行数据级别的验证模板。在「设计决策 9」中记录此权衡及回退选项（降级为可变 `class`）。 |
| **轻微-问题14**：`ITableDataSource` 与不同行数据类型的泛型关系未明确。 | 将 `ITableDataSource` 明确泛型化为 `ITableDataSource<TRow> where TRow : struct`，分别实例化为 `ITableDataSource<MapRowData>` 和 `ITableDataSource<CubeRowData>`。核心契约成员声明返回 `IReadOnlyList<TRow>`。更新「模块划分」中的接口名称和「核心抽象」中的描述。 |
