# 质量审查报告 — OOD 设计方案 v5

## 审查结论

产出整体结构清晰，前两轮迭代中提出的 14 项问题在 v5 中均已得到响应，接口定义的完整度较 v4 有显著提升。但在事实准确性、接口自洽性、边界条件覆盖等方面仍存在影响后续编码落地的关键问题。

---

## 严重问题

### 问题 1：事实错误 — Avalonia DataGrid `UnloadingRow` 事件存在，设计文档错误声明其不存在

**问题描述**：设计文档在「z 切片级高亮状态与 DataGrid 虚拟化联动（CubeEditor）」中明确声明"Avalonia `DataGrid` 不存在 `UnloadingRow` 事件"，并基于此错误声明构建了整个虚拟化状态清理策略（采用「`LoadingRow` 先清理再判断」+「`DataContextChanged` 后备清理」的双重机制）。经查阅 Avalonia 官方 API 文档（api-docs.avaloniaui.net），`DataGrid` 明确包含 `UnloadingRow` 事件，其定义与 WPF 一致——"Occurs when a DataGridRow object becomes available for reuse"。在 Avalonia 官方 GitHub issue #6027 中，核心维护者 maxkatz6 亦明确推荐使用 `LoadingRow` + `UnloadingRow` 配对来管理行级事件订阅。

**所在位置**：「z 切片级高亮状态与 DataGrid 虚拟化联动（CubeEditor）」小节第 412 行

**严重程度**：严重

**改进建议**：
1. 修正事实声明：`UnloadingRow` 事件在 Avalonia DataGrid 中存在且可用。
2. 将虚拟化状态清理策略改为标准的 `LoadingRow`/`UnloadingRow` 配对模式：在 `LoadingRow` 中附加 z 切片高亮样式类，在 `UnloadingRow` 中移除样式类。
3. `DataContextChanged` 可作为虚拟化行重用的后备机制保留，但不应作为主力方案。当前以错误事实为基础设计的双重机制增加了不必要的复杂度和 `DataContextChanged` 事件泄漏风险。

---

### 问题 2：接口设计缺陷 — `ITableDataSource<TRow>.Rows` 返回 `IReadOnlyList<TRow>` 与增量刷新机制矛盾

**问题描述**：设计文档在「关键行为契约」→「单元格编辑与数据回写」中描述增量刷新策略为"通过单元素替换（如 `Rows[index] = newRow`）仅刷新对应行"。然而 `ITableDataSource<TRow>` 中 `Rows` 的类型被定义为 `IReadOnlyList<TRow>`，该接口仅提供只读访问能力，不支持索引赋值操作。这意味着文档描述的增量刷新路径在接口层面即被阻断，实现者无法在满足接口契约的前提下执行单元素替换。

**所在位置**：「核心抽象」→「`ITableDataSource<TRow>`（接口）」；「关键行为契约」→「单元格编辑与数据回写」→ 增量刷新策略

**严重程度**：严重

**改进建议**：
1. 方案 A：将 `Rows` 的类型改为支持索引赋值的接口（如 `IList<TRow>` 或 `ObservableCollection<TRow>`），但这会削弱接口对只读语义的表达。
2. 方案 B（推荐）：保留 `IReadOnlyList<TRow>` 作为展示只读视图，同时新增 `void ReplaceRow(int index, TRow newRow)` 方法作为增量刷新的显式契约入口。此方法由实现方内部执行集合替换并触发 `INotifyCollectionChanged`（若实现）。
3. 无论采用哪种方案，需在接口中明确增量刷新的调用方式，消除当前"`Rows[index] = newRow`"这种无法通过接口实现的描述。

---

### 问题 3：接口设计与需求矛盾 — `IAsyncSaveCapable` 的 `bool HasUnsavedChanges` 无法支撑"未保存数量"显示

**问题描述**：`CalibrationEditorBase<T>` 的职责描述中包含"在信息栏区域显示未保存数量（如 '3 处未保存'）"。但 `IAsyncSaveCapable` 接口仅提供 `bool HasUnsavedChanges { get; }`，只能表达"有/无"未保存变更的二元状态，无法提供具体数量。接口设计与上层需求之间存在不可调和的矛盾——要么接口需要增加 `int UnsavedChangeCount { get; }`，要么 `CalibrationEditorBase<T>` 的职责描述需要降级为仅显示"存在未保存变更"的定性提示。

**所在位置**：「核心抽象」→「`IAsyncSaveCapable`（接口）」；「`CalibrationEditorBase<T>`（抽象类）」脏标记状态栏提示职责

**严重程度**：严重

**改进建议**：
1. 在 `IAsyncSaveCapable` 中补充 `int UnsavedChangeCount { get; }`（或 `IReadOnlyCollection<...>` 形式），使控件层能够显示具体数量。
2. 若后端无法提供精确数量（某些持久化框架难以追踪变更计数），则 `CalibrationEditorBase<T>` 的职责描述应降级为布尔状态提示（如"存在未保存变更"）。
3. 无论选择哪种方案，需确保接口契约与控件层职责描述一致。

---

## 一般问题

### 问题 4：`ZSliceActivationTracker.OnZColumnClicked` 的"选中第一个数据单元格"通知机制缺失

**问题描述**：`ZSliceActivationTracker` 的职责描述明确包含"通知 `CubeEditor` 选中该切片内的第一个数据单元格"。然而 Tracker 对外暴露的唯一通知机制是 `ActiveSliceChanged` 事件，其参数 `ActiveZSliceChangedEventArgs` 仅携带 `OldZIndex` 和 `NewZIndex`，没有任何标志表明此次变更源自 `OnZColumnClicked`（而非 `OnSelectionChanged`）。`CubeEditor` 在事件处理器中无法区分两种触发源，因此无法按 Tracker 的职责承诺执行"选中第一个数据单元格"的行为。若 `CubeEditor` 对所有 `ActiveSliceChanged` 事件都执行"选中第一个单元格"，则会破坏键盘导航时的用户选中位置保持。

**所在位置**：「核心抽象」→「`ZSliceActivationTracker`（类）」职责描述；「z 切片激活与 3D 视图联动（CubeEditor）」

**严重程度**：一般

**改进建议**：
1. 方案 A：在 `ActiveZSliceChangedEventArgs` 中增加 `SliceChangeSource Source { get; }` 枚举（`SelectionChange` / `ZColumnClick`），`CubeEditor` 根据来源决定是否执行"选中第一个数据单元格"。
2. 方案 B：将"选中第一个数据单元格"行为从 Tracker 的职责中移除，改为 `CubeEditor` 在调用 `OnZColumnClicked` 后自行处理选中逻辑，Tracker 仅负责 z 切片索引变更。
3. 推荐方案 B，因为"选中第一个单元格"本质上是 UI 层行为，不应由纯逻辑层的 Tracker 承担。

---

### 问题 5：`SupportsIncrementalRefresh` 与 `INotifyCollectionChanged` 的关联机制未定义

**问题描述**：`ITableDataSource<TRow>` 新增了 `SupportsIncrementalRefresh` 属性，文档说明"若返回 `true`，实现方应同时实现 `INotifyCollectionChanged`"。但 `ITableDataSource<TRow>` 本身未继承 `INotifyCollectionChanged`，也未提供获取 `INotifyCollectionChanged` 通知器的属性或方法（如 `INotifyCollectionChanged? CollectionChangedNotifier { get; }`）。这导致 `MapEditor`/`CubeEditor` 在运行时无法通过接口契约检测到 `INotifyCollectionChanged` 的实现，只能依赖类型强制转换（`Rows as INotifyCollectionChanged`），增加了运行时不确定性和耦合度。

**所在位置**：「核心抽象」→「`ITableDataSource<TRow>`（接口）」

**严重程度**：一般

**改进建议**：
1. 在 `ITableDataSource<TRow>` 中新增 `INotifyCollectionChanged? CollectionChangedNotifier { get; }` 属性。当 `SupportsIncrementalRefresh` 为 `true` 时，该属性返回非 null 的通知器实例；为 `false` 时返回 `null`。
2. 或者，让 `ITableDataSource<TRow>` 直接继承 `INotifyCollectionChanged`，使所有实现方必须提供集合变更通知能力（不支持增量刷新的实现方可返回空实现或永不触发事件）。

---

### 问题 6：`ICalibrationData` 轴属性对所有维度可见，一维数据模型被迫暴露无关属性

**问题描述**：`ICalibrationData` 接口定义了 `YAxisName`、`YAxisUnit`、`ZAxisName`、`ZAxisUnit` 等属性，并通过注释说明"2D/3D 数据适用"或"3D 数据适用"。`ICurveData`（一维）继承 `ICalibrationData` 后，被迫包含这些对其无意义的属性。接口设计违反了接口隔离原则：一维数据模型的实现者必须为这些属性提供虚假返回值（如空字符串），增加了实现负担，也可能导致下游消费者误用（如 `CurveEditor` 错误地显示 y/z 轴信息）。

**所在位置**：「核心抽象」→「`ICalibrationData`（接口）」

**严重程度**：一般

**改进建议**：
1. 将轴信息属性从 `ICalibrationData` 中移除，下沉到各维度子接口：
   - `ICurveData` 仅保留 `XAxisName`/`XAxisUnit`
   - `IMapData` 增加 `YAxisName`/`YAxisUnit`
   - `ICubeData` 增加 `ZAxisName`/`ZAxisUnit`
2. `CalibrationEditorBase<T>` 的信息栏格式化模板 `FormatInfoText` 应通过反射或类型检查判断当前 `T` 支持哪些轴，动态构建显示文本。

---

### 问题 7：`ISurfaceChartPresenter` 的 `double[,]` 参数与 `IMapData` 逐点访问接口之间的转换开销未考虑

**问题描述**：`ISurfaceChartPresenter.LoadMapData` 和 `LoadSliceData` 均接收 `double[,]` 原始数组。但 `IMapData` 和 `ICubeData` 提供的是逐点访问接口（`GetValue(x, y)` / `GetValue(x, y, z)`），没有直接返回二维/三维数组的属性。这意味着 Editor 在调用图表加载方法前，必须自行遍历数据构建 `double[,]` 数组。对于较大的 Map 数据（如 100×100），每次变量切换或 z 切片切换都会产生 O(n) 的内存分配和数据拷贝开销。设计文档未提及这一转换开销，也未讨论缓存策略。

**所在位置**：「核心抽象」→「`ILineChartPresenter` / `ISurfaceChartPresenter`」

**严重程度**：一般

**改进建议**：
1. 在 `IMapData` 中增加 `double[,] GetValueMatrix()` 方法（或 `ReadOnlyMemory2D<double>` 等现代内存抽象），由后端数据模型决定是返回内部数组的视图还是执行拷贝。
2. 或者在 `ISurfaceChartPresenter` 接口中增加逐点数据填充的替代方法（如 `void LoadMapData(IMapData data)`），让图表实现方自行决定最优的数据读取策略。
3. 若坚持当前设计，需在「设计决策 5」或「并发设计」中补充 `double[,]` 构建的性能说明和缓存建议。

---

### 问题 8：动态列绑定 `readonly struct` 索引器与 Avalonia V12 CompiledBinding 的兼容性风险

**问题描述**：`MapRowData`/`CubeRowData` 通过索引器暴露 x 列数据（如 `double this[int index] { get; }`），`MapEditor`/`CubeEditor` 需要在变量切换时动态生成 DataGrid 列，每列的绑定路径为 `[0]`、`[1]` 等索引器语法。Avalonia V12 默认启用 CompiledBinding（`<AvaloniaUseCompiledBindingsByDefault>true</AvaloniaUseCompiledBindingsByDefault>`）。动态生成列时，列的 `Binding` 无法在 XAML 中静态声明 `x:DataType`，CompiledBinding 对运行时动态构造的索引器绑定的支持情况未经验证。若 CompiledBinding 不支持动态索引器绑定，可能需要回退到 ReflectionBinding，影响性能和 AOT 兼容性。

**所在位置**：「核心抽象」→「`MapRowData` / `CubeRowData`」；「设计决策 8」

**严重程度**：一般

**改进建议**：
1. 在「设计决策」中补充对 CompiledBinding + 动态索引器绑定的技术验证项，明确在实现阶段需要验证以下场景：
   - 代码动态构造的 `Binding`（`new Binding("[0]")`）在 CompiledBinding 模式下是否正常工作
   - 若不支持，是否需要回退到 `ReflectionBinding` 或使用 `CompiledBinding.Create(...)` 的代码构造方式
2. 备选方案：将 `MapRowData`/`CubeRowData` 改为通过 `IReadOnlyList<double> Values { get; }` 暴露 x 列数据，DataGrid 列绑定到 `Values[0]`、`Values[1]` 等。`IReadOnlyList<T>` 的属性绑定在 CompiledBinding 中的兼容性更好。

---

### 问题 9：空状态未覆盖数据内容为零维度的场景

**问题描述**：`CalibrationEditorBase<T>` 的空状态管理覆盖了 `ItemsSource` 为 null、空列表、`SelectedVariable` 为 null 三种场景。但未覆盖 `SelectedVariable` 有效但其内部数据维度为零的情况，例如 `ICurveData.Length == 0`、`IMapData.XAxisLength == 0` 或 `ICubeData.ZAxisLength == 0`。在这些场景下，控件会尝试构建 0 列的表格或初始化 `ZSliceActivationTracker` 的 `ZIndex = 0`（当 z 维度为 0 时越界），可能导致运行时异常或控件白屏。

**所在位置**：「错误处理策略」→「数据绑定不匹配错误」；「核心抽象」→「`ZSliceActivationTracker`（类）」

**严重程度**：一般

**改进建议**：
1. 在「变量切换流程」中增加对数据内容维度的检查：若 `SelectedVariable` 有效但维度为零，同样进入空状态。
2. 在 `ZSliceActivationTracker` 初始化逻辑中增加 `ZAxisLength == 0` 的防护：若 z 维度为 0，设置 `ActiveZIndex = -1`（表示无有效切片），并确保 `ActiveSliceChanged` 事件不触发。
3. `CalibrationEditorBase<T>` 的 `OnEnterEmptyState` / `OnExitEmptyState` 虚方法应覆盖所有空数据场景，包括零维度数据。

---

### 问题 10：多个事件参数类型未定义

**问题描述**：设计文档在接口定义中引用了以下事件参数类型，但未给出它们的结构定义：
- `CurveCellSelectedEventArgs`（`ICurveTablePresenter.SelectedCellChanged` 事件）
- `CurveCellEditCompletedEventArgs`（`ICurveTablePresenter.CellEditCompleted` 事件）
- `ChartRenderFailedEventArgs`（`IChartPresenter.RenderFailed` 事件）
- `DataGridCellEditEndingEventArgs` 在 Avalonia 中的具体属性（`Column`、`EditingElement`、`Cancel` 等）未确认

**所在位置**：「核心抽象」→「`ICurveTablePresenter`（接口）」；「核心抽象」→「`IChartPresenter`（接口）」；「关键行为契约」→「单元格编辑与数据回写」

**严重程度**：一般

**改进建议**：
1. 在「核心抽象」中补充 `CurveCellSelectedEventArgs`、`CurveCellEditCompletedEventArgs`、`ChartRenderFailedEventArgs` 的完整结构定义，包括属性名称、类型和语义。
2. 确认 Avalonia `DataGridCellEditEndingEventArgs` 的可用属性（尤其是 `Column` 和 `EditingElement`），并在编辑流程描述中明确如何从事件参数中获取列索引和编辑后的文本值。

---

### 问题 11：键盘导航行为未在设计中充分定义

**问题描述**：需求文档明确要求"键盘导航（方向键移动、Tab 切换、Enter 确认等）"，但设计文档仅在编辑触发方式中提到"Enter/失去焦点"作为输入完成触发。方向键在表格中的移动行为（尤其是 `CubeEditor` 中跨 z 切片的上下导航）、Tab 键在横向/纵向表格中的切换顺序、Shift+Tab 反向导航、以及 `CurveEditor` 横向两行布局中的方向键行为，均未在设计中定义。这些行为直接影响 `ZSliceActivationTracker` 的防抖触发逻辑和 `ICurveTablePresenter` 的选中管理。

**所在位置**：需求 `requirement.md` 第 160 行、第 221 行；设计文档「关键行为契约」→「单元格编辑与数据回写」→ 编辑触发方式

**严重程度**：一般

**改进建议**：
1. 在「关键行为契约」中补充键盘导航的完整行为定义：
   - `MapEditor`/`CubeEditor`：方向键上下移动选中行，左右移动选中列；在 `CubeEditor` 中从当前 z 切片最后一行按向下键进入下一 z 切片第一行时的行为
   - `CurveEditor`：方向键左右移动选中列，上下键在 X 行和 Z 行之间切换
   - Tab/Shift+Tab 的列顺序导航和行顺序导航
2. 明确键盘导航触发的 `ZSliceActivationTracker.OnSelectionChanged` 调用与防抖策略的交互细节。

---

### 问题 12：`ZSliceActivationTracker.OnSelectionChanged(null)` 行为未定义

**问题描述**：`ZSliceActivationTracker.OnSelectionChanged(CubeRowData? row)` 的参数被标记为可空（`CubeRowData?`），文档说明"`null` 表示无选中项"。但当传入 `null` 时，Tracker 应该如何处理当前激活切片未作定义：是保持当前激活切片不变？还是重置为默认切片（`ZIndex = 0`）？还是将 `ActiveZIndex` 设为 -1（无效状态）？这一缺失会导致 `CubeEditor` 在表格失去焦点或 `ItemsSource` 被清空时的行为不确定。

**所在位置**：「核心抽象」→「`ZSliceActivationTracker`（类）」

**严重程度**：一般

**改进建议**：
1. 明确 `OnSelectionChanged(null)` 的语义：建议定义为"保持当前激活切片不变"，因为无选中项不等于用户意图切换切片。
2. 若需要重置行为，应通过独立的方法（如 `ResetToDefault()`）显式触发，而非通过 `null` 参数隐式表达。

---

## 整体评价

v5 方案在模块划分、职责分离和接口完整度上已达到可指导编码的基本水准。`ICalibrationData` 及维度特化子接口、`ZSliceActivationTracker` 的事件签名、`IAsyncSaveCapable` 的扩展等改进有效填补了 v4 的空白。但上述 3 项严重问题（`UnloadingRow` 事实错误、`IReadOnlyList` 与增量刷新矛盾、`bool HasUnsavedChanges` 与数量显示矛盾）会直接阻塞编码实现或导致需求无法满足，必须在进入实现阶段前修复。一般问题主要涉及边界条件、异常场景和细节契约的完备性，修复后可显著提升设计的落地可靠性。
