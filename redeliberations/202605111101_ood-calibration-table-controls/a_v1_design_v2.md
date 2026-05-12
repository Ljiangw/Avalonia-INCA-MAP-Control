# Avalonia 标定数据表格编辑控件 — 架构级 OOD 设计方案 v3（修订版）

## 概述

本方案设计一组基于 Avalonia V12 的标定数据表格编辑控件，面向汽车 ECU 标定工具场景。在统一交互范式下支撑 1 维（曲线）、2 维（Map）、3 维（数据立方体）三种标定数据的可视化编辑，并将控件与后端数据模型、图表渲染引擎解耦，确保可复用性和可替换性。

相对于 v2 方案，v3 的核心变化集中在 **CubeEditor** 的数据组织与交互模型：取消独立的 Z 轴数据列区域，将 z 轴信息整合进表格行头，以平铺形式展示完整的 3 维数据立方体（所有 z 切片的 (z, y) 行 × x 列）。z 切片激活完全由表格内的选中/导航交互驱动，左侧 3D 曲面图始终展示当前「激活/聚焦」z 切片的 (x, y) → value 数据。

**本轮修订（v2）的核心调整**：明确 `CurveEditor` 的横向两行表格布局采用独立的自定义布局方案，与 `MapEditor`/`CubeEditor` 的纵向 `DataGrid` 行集合抽象解耦。`CurveEditor` 不再依赖 `ITableDataSource` 和 `CurveRowData`，转而通过 `ICurveTablePresenter` 契约构建横向表格视图。同时补充 `CubeEditor` 中 z 切片级高亮状态与 `DataGrid` 虚拟化行视觉元素的联动策略。

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
| **表格适配层** | 将多维数据投影为二维行集合，封装单元格坐标到原始数据的反向映射，支撑编辑回写。`ITableDataSource` 及 `MapRowData`/`CubeRowData` 供 `MapEditor` 与 `CubeEditor` 使用；`CurveEditor` 不经过此抽象，而由 `ICurveTablePresenter` 直接处理横向布局。 | 依赖数据契约层 |
| **图表抽象层** | 定义折线图和 3D 曲面图的渲染契约，隔离具体图表库实现。 | 被 Views 层依赖，自身无 UI 框架依赖（接收原始数据） |
| **激活追踪层** | CubeEditor 中 z 切片激活状态的专门管理者，将表格行/单元格选中事件映射为 z 值切换。 | 依赖表格适配层的 `CubeRowData`，被 CubeEditor 使用 |
| **Curve 表格契约层** | `CurveEditor` 专用的横向表格视图契约（`ICurveTablePresenter`），将一维 x/z 数据映射为两行横向表格，处理选中、编辑与高亮。 | 依赖数据契约层，被 CurveEditor 使用 |

依赖规则：数据契约层为最底层，无 outward 依赖；适配层和图表抽象层仅依赖数据契约；激活追踪层仅依赖表格适配层；Curve 表格契约层仅依赖数据契约层；Views 层可依赖所有下层模块，但下层模块不可反向依赖 Views 层。

## 核心抽象

### `ICalibrationData`（接口）

**角色**：所有维度标定数据的公共契约。

**职责**：暴露标定变量的元信息（名称、单位、各轴变量名及单位）以及数据变更通知能力。

**协作**：被 `ICurveData`、`IMapData`、`ICubeData` 继承；被 `CalibrationEditorBase<T>` 作为泛型约束使用；被表格适配层、图表抽象层和 Curve 表格契约层消费。

**类型形态**：接口，**明确继承 `INotifyPropertyChanged`**。具体数据模型由后端标定系统定义，控件组只消费契约，不拥有实现。将 `INotifyPropertyChanged` 纳入继承列表，强制后端实现方具备数据变更通知能力，确保单元格编辑后绑定系统能正确感知变更并刷新 UI。

### `ICurveData` / `IMapData` / `ICubeData`（接口）

**角色**：维度特化的数据契约。

**职责**：在 `ICalibrationData` 基础上，各自声明维度特定的数据访问方式——`ICurveData` 提供一维 x/z 值序列的读写访问，`IMapData` 提供二维 (x,y)→z 的矩阵读写访问，`ICubeData` 提供三维 (x,y,z)→value 的张量读写访问。

**协作**：作为 `CalibrationEditorBase<T>` 的泛型参数具体化类型；被各自的表格适配器或 Curve 表格呈现器消费。

**类型形态**：接口，继承自 `ICalibrationData`。不采用抽象类，因为后端数据模型可能已经继承自其他领域基类，接口避免单继承限制。

### `CalibrationEditorBase<T>`（抽象类）

**角色**：三个标定编辑控件的公共基座。

**职责**：
- 定义并管理 Avalonia 依赖属性：`ItemsSource`（`IList<T>` 类型，绑定到变量选择下拉框）、`SelectedVariable`（当前选中的标定数据项）
- 提供统一的顶部信息栏模板结构（左侧 `ComboBox`、右侧单位/轴信息展示）
- 处理变量切换时的选中变更事件，向子类发出数据切换通知

**协作**：内部聚合一个 `ComboBox` 和一个信息展示面板；通过模板约定或受保护的虚方法向子类（`CurveEditor`、`MapEditor`、`CubeEditor`）暴露数据切换钩子。

**类型形态**：泛型抽象类（`T : ICalibrationData`），继承自 Avalonia `TemplatedControl`。选用抽象类而非接口，是因为需要封装具体的依赖属性注册、模板部件约定和公共视觉状态逻辑——这些是具体的实现代码，接口无法承载。

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

**类型形态**：接口。`CurveEditor` 的横向布局与 `MapEditor`/`CubeEditor` 的纵向 `DataGrid` 行模型本质不同：前者是固定两行、动态多列的矩阵转置形态，后者是纵向行集合。若强行将 `CurveEditor` 纳入 `ITableDataSource` 纵向抽象，需要复杂的列动态生成和索引器绑定，与 DataGrid 的行模型语义冲突。独立的 `ICurveTablePresenter` 契约允许实现者选择最适合横向布局的 Avalonia 布局容器，保持架构清晰。

### `ITableDataSource`（接口）

**角色**：多维数据到二维纵向表格结构的投影器。供 `MapEditor` 与 `CubeEditor` 使用。

**职责**：将原始标定数据（二维矩阵、三维张量）转换为适合 Avalonia `DataGrid` 绑定的行集合。各维度实现返回各自强类型的行数据对象集合，行数据对象内联携带原始数据坐标索引，以支撑编辑后的值回写。

**协作**：被 `MapEditor` 和 `CubeEditor` 内部使用，作为 `DataGrid` `ItemsSource` 的实际来源。编辑完成时，Editor 从强类型行数据对象中提取坐标索引，调用原始数据模型的写接口。

**类型形态**：接口。`MapEditor` 与 `CubeEditor` 共享纵向行集合抽象，`CurveEditor` 不经过此抽象。

### `MapRowData` / `CubeRowData`（`readonly struct`）

**角色**：承载一行纵向表格数据的展示值及其在原始多维数据中的索引坐标。供 `MapEditor` 与 `CubeEditor` 使用。

**职责**：
- `MapRowData`：内联 `XIndex`、`YIndex` 属性，标识该单元格在二维矩阵中的坐标
- `CubeRowData`：内联 `ZIndex`、`YIndex` 属性，标识该数据行在三维张量中的坐标；同时承载 `ZValue` 和 `YValue` 文本（用于 z/y 双列行头展示），以及 `IsFirstRowInZSlice` 标识（用于控制 z 列的分组显示逻辑）。每行对应一个 (z, y) 组合下所有 x 列的数据值

**协作**：由对应维度的表格适配器生成并填充，作为 `DataGrid` 的 `ItemsSource` 行集合。Editor 在编辑确认时从行数据对象中直接读取坐标索引属性，回写到原始数据模型。

**类型形态**：`readonly struct`。值类型避免堆分配；坐标索引作为结构体的属性直接内联，不通过接口引用传递，彻底消除接口装箱带来的 GC 压力。

### `IChartPresenter`（接口）

**角色**：图表渲染的抽象契约。

**职责**：接收标定数据，在宿主控件区域内渲染折线图（1D）或 3D 曲面图（2D/3D）。不处理交互事件，仅负责数据到图形的映射。

**协作**：被 `CurveEditor`、`MapEditor`、`CubeEditor` 内部持有。实现方可选用 OxyPlot、LiveCharts、ScottPlot、自定义 Skia 渲染等具体技术。

**类型形态**：接口。因为 Avalonia V12 生态中没有内置的 3D 曲面图标准控件，图表实现必须可替换，接口是最低耦合的抽象形式。

### `ZSliceActivationTracker`（类）

**角色**：CubeEditor 中 z 切片激活状态的专门管理者。

**职责**：
- 维护当前激活的 z 值索引
- 监听表格选中项变更，从 `CubeRowData` 中提取 `ZIndex`，判断是否需要切换激活切片
- 处理 z 列分组首行的点击事件：当用户点击属于某 z 切片分组的首行（或合并 z 单元格区域）时，将该 z 值设为激活切片，并联动选中该切片内的第一个数据单元格
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

### 单元格编辑与数据回写

#### MapEditor / CubeEditor（DataGrid 纵向行模型）

用户触发单元格进入编辑模式（双击进入编辑，单击仅用于选中/导航）→ 单元格展示编辑模板（如 `TextBox`）→ 用户输入数值 → 输入完成时（Enter/失去焦点），Editor 从对应类型的行数据对象中提取坐标索引属性，调用原始数据模型的写方法更新对应位置 → 模型触发 `INotifyPropertyChanged.PropertyChanged` → 绑定系统刷新关联单元格；若该单元格值同时影响图表，图表呈现器接收通知后重绘。

在 CubeEditor 中，编辑操作发生在平铺表格的任意数据单元格。编辑完成后，Editor 从该单元格对应的 `CubeRowData` 中读取 `ZIndex`、`YIndex`，并结合该单元格所在的 x 列索引 `XIndex`，调用 `ICubeData` 的写接口更新三维张量中对应位置的值。

#### CurveEditor（横向自定义布局）

用户触发单元格进入编辑模式（双击进入编辑，单击仅用于选中/导航）→ `ICurveTablePresenter` 在对应列的对应行（X 行或 Z 行）展示编辑控件（如 `TextBox`）→ 用户输入数值 → 输入完成时，`ICurveTablePresenter` 触发编辑完成事件，携带数据点索引（列位置）、行标识（X 或 Z）以及解析后的数值 → `CurveEditor` 在事件处理器中调用 `ICurveData` 的写接口更新对应位置的数据 → 模型触发 `INotifyPropertyChanged.PropertyChanged` → `ICurveTablePresenter` 刷新对应单元格显示，`IChartPresenter` 重绘折线图。

### z 切片激活与 3D 视图联动（CubeEditor）

用户点击/键盘导航到表格中的某个单元格或行 → `CubeEditor` 将选中的 `CubeRowData` 传递给 `ZSliceActivationTracker` → Tracker 读取该行的 `ZIndex` 属性，与当前激活 z 比较：若相同则无动作；若不同则更新激活 z 值并发出变更事件 → `CubeEditor` 在事件处理器中通过 `Dispatcher.UIThread.Post` 调度 UI 更新：通知图表呈现器切换数据源到新的 z 切片矩阵，同时触发表格的视觉状态更新（z 切片级高亮）。

当用户点击 z 列分组区域（或属于某 z 切片的合并 z 单元格）时，`ZSliceActivationTracker` 将该 z 值设为激活切片，并通知 `CubeEditor` 选中该切片内的第一个数据单元格。

键盘跨 z 切片导航时（如从当前 z 切片的最后一行 y 值按方向键移入下一 z 切片的第一行 y 值），`ZSliceActivationTracker` 内部引入短时间防抖（如 100–200ms），仅在用户停止快速导航后才真正刷新 3D 曲面图，避免频繁重绘造成卡顿。

### z 切片级高亮状态与 DataGrid 虚拟化联动（CubeEditor）

`DataGrid` 仅对当前可视区域内的行创建视觉树实例（行虚拟化）。`ZSliceActivationTracker` 输出的激活 `ZIndex` 需通过以下机制传递到行视觉元素：

1. **`CubeEditor` 订阅 `ZSliceActivationTracker` 的激活变更事件**。当激活 `ZIndex` 变化时，`CubeEditor` 执行两项操作：通知 `IChartPresenter` 刷新 3D 曲面图；遍历当前已加载的 `DataGridRow` 容器，为属于激活 z 切片的行附加样式类（如 `active-z-slice`），为非激活行移除该样式类。

2. **`CubeEditor` 订阅 `DataGrid` 的 `LoadingRow` 事件**。当用户滚动导致新行进入可视区域时，`DataGrid` 创建对应的行视觉元素并触发 `LoadingRow` 事件。`CubeEditor` 在事件处理器中读取该行的 `DataContext`（`CubeRowData`），若其 `ZIndex` 等于当前激活 `ZIndex`，则立即为新行附加 z 切片高亮样式类。这确保了虚拟化机制下，新进入可视区域的行也能正确显示激活切片的高亮状态。

3. **`CubeEditor` 订阅 `DataGrid` 的 `UnloadingRow` 事件**（若可用）。在行视觉元素被回收前清理其附加的样式类，避免回收后状态残留影响后续行的视觉呈现。

通过上述机制，`ZSliceActivationTracker` 的状态输出（纯 .NET 事件）与 `DataGrid` 视觉树的状态同步解耦：Tracker 不依赖 Avalonia 视觉类型，而 `CubeEditor` 作为 UI 层协调者负责将逻辑状态映射到视觉状态，兼容 `DataGrid` 的行虚拟化和行回收机制。

### 高亮叠加策略

三种高亮在 CubeEditor 中可能同时存在，视觉优先级从高到低为：

1. **单元格级高亮**（蓝色背景，当前活动单元格）
2. **行级高亮**（轻微背景色，当前选中单元格所在的整行）
3. **z 切片级高亮**（左侧边框或分组背景色，当前激活切片的所有行）

通过 Avalonia 的样式选择器（Style Selector）和 `Classes` 动态附加机制实现层级叠加。单元格级直接通过表格选中状态驱动；行级和 z 切片级通过 `ZSliceActivationTracker` 输出的状态值驱动附加样式类。

## 错误处理策略

### 输入验证错误

单元格编辑阶段的数值解析失败（如非法字符、越界值）属于局部错误。错误处理策略为「就地反馈」：在单元格层面通过红色边框或工具提示标识错误，阻止该次编辑回写，但不阻断用户继续编辑其他单元格或切换变量。

在 `MapEditor`/`CubeEditor` 中，利用 Avalonia `DataGrid` 的数据验证模板（`IDataErrorInfo` / `INotifyDataErrorInfo`）实现就地反馈。在 `CurveEditor` 中，`ICurveTablePresenter` 的实现负责拦截非法输入并在编辑控件层面呈现错误标识。

### 数据绑定不匹配错误

控件绑定到不符合维度期望的数据模型（如 `CubeEditor` 接收的 `ICubeData` 中 z 维度长度为 0）属于结构性错误。控件应进入「空状态」展示友好提示（如 "无有效数据"），而非抛出未处理异常导致控件白屏。

### 图表渲染错误

图表库渲染失败（如 GPU 不可用、3D 数据异常）属于可降级错误。`IChartPresenter` 实现方应内部捕获渲染异常，降级为显示占位矩形或错误图标，保证表格编辑功能不受影响。

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

**决策**：不追求原生跨行合并，采用「z 值首行显示 + 视觉分组线 + 左侧边框标识」的组合方案。

**理由**：Avalonia 的 `DataGrid`（包括 V12）不提供 WPF 中 `RowSpan` 式的跨行单元格能力。若强行在模板层模拟合并单元格，会严重破坏虚拟化机制和行回收，在数据量大时性能急剧下降。替代方案中，z 列的自定义 `CellTemplate` 只在每组的第一个可见行渲染 z 值文本（通过 `CubeRowData.IsFirstRowInZSlice` 控制），其余行留空；配合行样式在 z 切片边界处渲染水平分隔线；激活切片通过整组行的左侧边框或轻微背景色统一标识。此方案在视觉认知上等效于合并单元格，且完全兼容 `DataGrid` 的虚拟化。

### 3. 为什么用抽象基类 `CalibrationEditorBase<T>` 而非纯组合或接口？

**决策**：采用泛型抽象类承载公共 UI 结构和依赖属性。

**理由**：三个控件共享的不仅是行为契约，还有大量的具体 UI 结构——顶部栏的 `ComboBox`、单位信息面板、变量切换后的数据刷新管线。抽象类可以在 Avalonia 的类型系统中直接注册这些公共依赖属性（`AvaloniaProperty.Register`），并约定公共 ControlTemplate 的部件名称（如 `PART_VariableSelector`、`PART_InfoPanel`）。若用纯接口，每个控件需要重复注册相同的依赖属性和模板逻辑，违背 DRY 原则。组合模式（如注入一个「顶部栏管理器」对象）在 Avalonia 的模板/样式系统中反而增加不必要的间接层。

### 4. 表格编辑如何定位回原始数据？

**决策**：通过维度特化的 `readonly struct` 行数据对象建立可见单元格到原始多维索引的映射，坐标索引直接内联为结构体属性，取消独立的坐标接口抽象。

**理由**：Avalonia 表格控件（`DataGrid`）天然面向二维行集合（`IEnumerable` 的扁平行）。原始标定数据是 2D/3D 矩阵，无法直接作为 `ItemsSource`。引入表格适配层将矩阵投影为行集合后，每一行必须携带「我来自原始数据的哪个位置」这一元信息，否则编辑后无法回写。v2 已验证将坐标索引直接内联到各维度行数据类型（`MapRowData`、`CubeRowData`）中的方案，以强类型属性传递坐标信息，彻底消除接口装箱，同时保持编辑回写时的坐标定位能力。v3 中 `CubeRowData` 额外承载 `ZValue`、`YValue` 和 `IsFirstRowInZSlice`，以支撑 z/y 双列行头的展示需求。

### 5. 3D 曲面图没有 Avalonia 内置方案，如何设计？

**决策**：通过 `IChartPresenter` 接口抽象图表渲染，控件不耦合具体图表库。

**理由**：Avalonia V12 标准控件集中不包含 3D 图表。社区中 OxyPlot、LiveCharts、ScottPlot 等库对 3D 的支持程度和 API 差异很大，且部分库在 V12 下的兼容性尚在发展。将图表渲染抽象为接口后，控件只关心「把数据给你，你在指定区域内画出来」。实现阶段可以先以 2D 热力图或伪 3D 网格作为最小可行方案，后续替换为真正的 3D 引擎时不影响控件架构。这避免了在设计阶段就锁定可能不合适的图表技术。

### 6. CubeEditor 的 z 切片激活逻辑为何独立为 `ZSliceActivationTracker`？

**决策**：将 z 切片激活的状态管理和事件映射抽取为独立类，而非内嵌在 `CubeEditor` 中。

**理由**：CubeEditor 的表格同时承担多重职责——数据展示、单元格编辑、z 切片选择器。若将 z 切片激活逻辑（行→z 值提取、激活变更判断、防抖、通知联动、z 列分组首行点击处理）全部写在 `CubeEditor` 的代码后台，会导致该类臃肿且难以测试。`ZSliceActivationTracker` 封装了「从表格选中状态到 z 切片语义」的纯逻辑转换，不依赖 Avalonia 控件的具体视觉类型，仅消费 `CubeRowData` 行数据对象。其状态输出采用纯 .NET 事件（`EventHandler<T>`），由 `CubeEditor` 在事件处理器中调度 UI 线程更新，这使得 Tracker 逻辑可以在无头环境中单元测试，也便于后续调整 z 切片切换的交互策略。

### 7. 编辑触发方式：单击还是双击？

**决策**：表格数据单元格采用 **双击进入编辑** 模式，单击仅用于选中/导航。

**理由**：在标定数据表格中，用户频繁进行选中浏览（查看相邻单元格数值、跨行比较）。若单击即进入编辑，会频繁弹出 `TextBox` 编辑模板，干扰浏览体验。双击编辑是桌面表格工具（Excel、INCA 等标定工具）的广泛惯例，与用户的工程工具心智模型一致。行头列（z 列、y 列）不参与编辑，单击它们仅触发选中或 z 切片激活。

### 8. CurveEditor 的横向两行表格为何不用 DataGrid，而采用独立布局契约？

**决策**：`CurveEditor` 不使用 `DataGrid` 和 `ITableDataSource` 纵向行集合抽象，而采用独立的 `ICurveTablePresenter` 接口，由实现方基于 Avalonia 自定义布局容器构建横向视觉树。

**理由**：需求明确要求 `CurveEditor` "以两行形式展示：第一行为 x 轴（自变量）值，第二行为 z 轴（因变量）值"，这是一种矩阵转置的表格形态——固定两行、动态多列。而 `DataGrid` 的设计范式是纵向行集合（每行一个数据对象，列对应属性），与横向布局的语义天然冲突。若强行套用 `DataGrid`，需要为每个数据点动态生成一列，并将两个行对象（x 行和 z 行）的数组索引绑定到各列，实现复杂度极高且违背 `DataGrid` 的设计意图。`CurveEditor` 的数据规模通常较小（一维曲线的数据点数量有限），不需要 `DataGrid` 的行虚拟化、列排序等重型功能。通过 `ICurveTablePresenter` 接口将横向布局的构建和交互抽象化后，实现方可选用 `Grid` + `ItemsControl`、`UniformGrid` 或自定义 `Panel` 等轻量方案，精确控制列宽、单元格间距和选中高亮行为，与需求的横向两行展示完全对齐。同时，`CurveEditor` 与 `MapEditor`/`CubeEditor` 在表格实现上的差异被封装在各自的适配/呈现层中，不影响公共基类的统一性。

## 修订说明（v2）

| 审查意见 | 修改措施 |
|---------|---------|
| `CurveEditor` 的横向两行表格布局与 `ITableDataSource` 纵向行集合抽象不匹配。需求要求横向展示（x 行 + z 行），但 `ITableDataSource` 和 `CurveRowData` 面向纵向行集合，直接套用 `DataGrid` 会得到纵向多行列表，与需求冲突。 | **采用方案 B（自定义布局）**：<br>1. 明确 `CurveEditor` 不使用 `DataGrid`、`ITableDataSource` 和 `CurveRowData`；<br>2. 新增 `ICurveTablePresenter` 接口作为 `CurveEditor` 专用的横向表格视图契约，负责构建两行横向视觉树、管理选中/编辑状态、输出编辑完成事件；<br>3. 更新模块划分图和职责表，将 `CurveEditor` 从 `ITableDataSource` 依赖中剥离；<br>4. 在「关键行为契约」中补充 `CurveEditor` 的横向表格编辑回写链路；<br>5. 新增「设计决策 8」详细阐述为何 CurveEditor 采用独立布局契约而非 DataGrid。 |
| `CubeEditor` 的 z 切片级高亮状态如何与 `DataGrid` 虚拟化行视觉元素联动，方案缺少状态传递机制说明。`DataGrid` 仅对可见行创建视觉树实例，激活状态需传递到行视觉元素。 | 在「关键行为契约」中新增 **「z 切片级高亮状态与 DataGrid 虚拟化联动」** 小节，明确三层机制：<br>1. 激活 `ZIndex` 变化时，`CubeEditor` 遍历当前已加载的 `DataGridRow` 容器更新样式类；<br>2. 订阅 `LoadingRow` 事件，为新进入可视区域的行根据当前激活 `ZIndex` 设置高亮样式；<br>3. 订阅 `UnloadingRow` 事件清理回收行的样式状态。确保 Tracker 不依赖 Avalonia 视觉类型，而由 `CubeEditor` 作为 UI 协调者完成逻辑状态到视觉状态的映射。 |
| `readonly struct` 作为 `DataGrid` 行数据源时，标准单元格编辑的双向绑定自动提交机制无法更新不可变结构体实例，编辑回写需完全依赖手动事件拦截，实现复杂度较高。 | 本轮未在架构层面改动。该问题属于实现阶段细节：设计中已明确编辑回写通过 `CellEditEnding`/`CellEditEnded` 事件拦截实现，由 Editor 从行数据对象提取坐标索引后调用原始数据模型写接口。`readonly struct` 的不可变性不影响此回写链路（Editor 不修改行数据对象，而是直接修改后端模型并触发 `PropertyChanged` 刷新 UI）。实现阶段需明确 `DataGrid` 的编辑模板和事件处理策略。 |
| 输入验证的抽象未在数据契约层定义。需求要求"输入的数据应能正确解析为数值类型"，但 `ICalibrationData` 未声明数值解析、精度、取值范围验证契约。 | 本轮在架构层面做方向性补充：在「错误处理策略」的「输入验证错误」小节中明确，`MapEditor`/`CubeEditor` 利用 `DataGrid` 的数据验证模板（`IDataErrorInfo`/`INotifyDataErrorInfo`）实现就地反馈，`CurveEditor` 由 `ICurveTablePresenter` 实现方拦截非法输入。详细设计阶段可在数据契约层或表格适配层补充验证接口（如 `ICellValidator`），本轮架构设计保留此扩展方向但不强制引入新抽象。 |
