根据以下审查结果，迭代上一轮的产出，形成新版的文件，从而更好地满足用户需求。

## 当前审查结果

本轮审查共发现 12 项质量问题，其中 3 项严重、9 项一般，需在 v6 中修复。

### 严重问题

1. **事实错误 — Avalonia DataGrid `UnloadingRow` 事件存在，设计文档错误声明其不存在**
   - 设计文档在「z 切片级高亮状态与 DataGrid 虚拟化联动（CubeEditor）」中明确声明"Avalonia `DataGrid` 不存在 `UnloadingRow` 事件"，并基于此构建了整个虚拟化状态清理策略（采用「`LoadingRow` 先清理再判断」+「`DataContextChanged` 后备清理」的双重机制）。经查阅 Avalonia 官方 API 文档，`DataGrid` 明确包含 `UnloadingRow` 事件，其定义与 WPF 一致。在 Avalonia 官方 GitHub issue #6027 中，核心维护者亦明确推荐使用 `LoadingRow` + `UnloadingRow` 配对来管理行级事件订阅。
   - 改进建议：修正事实声明，将虚拟化状态清理策略改为标准的 `LoadingRow`/`UnloadingRow` 配对模式；`DataContextChanged` 可作为后备机制保留，但不应作为主力方案。

2. **接口设计缺陷 — `ITableDataSource<TRow>.Rows` 返回 `IReadOnlyList<TRow>` 与增量刷新机制矛盾**
   - 设计文档描述增量刷新策略为"通过单元素替换（如 `Rows[index] = newRow`）仅刷新对应行"，然而 `IReadOnlyList<TRow>` 仅提供只读访问能力，不支持索引赋值操作。这意味着文档描述的增量刷新路径在接口层面即被阻断。
   - 改进建议：保留 `IReadOnlyList<TRow>` 作为只读视图，同时新增 `void ReplaceRow(int index, TRow newRow)` 方法作为增量刷新的显式契约入口。此方法由实现方内部执行集合替换并触发 `INotifyCollectionChanged`（若实现）。

3. **接口设计与需求矛盾 — `IAsyncSaveCapable` 的 `bool HasUnsavedChanges` 无法支撑"未保存数量"显示**
   - `CalibrationEditorBase<T>` 的职责描述中包含"在信息栏区域显示未保存数量（如 '3 处未保存'）"。但 `IAsyncSaveCapable` 接口仅提供 `bool HasUnsavedChanges { get; }`，只能表达"有/无"未保存变更的二元状态，无法提供具体数量。
   - 改进建议：在 `IAsyncSaveCapable` 中补充 `int UnsavedChangeCount { get; }`，使控件层能够显示具体数量；若后端无法提供精确数量，则 `CalibrationEditorBase<T>` 的职责描述应降级为布尔状态提示（如"存在未保存变更"）。需确保接口契约与控件层职责描述一致。

### 一般问题

4. **`ZSliceActivationTracker.OnZColumnClicked` 的"选中第一个数据单元格"通知机制缺失**
   - `ZSliceActivationTracker` 的职责描述包含"通知 `CubeEditor` 选中该切片内的第一个数据单元格"，但 Tracker 对外暴露的唯一通知机制 `ActiveSliceChanged` 事件无法区分触发源（`OnZColumnClicked` vs `OnSelectionChanged`），`CubeEditor` 无法按承诺执行"选中第一个数据单元格"的行为。
   - 改进建议：推荐将"选中第一个数据单元格"行为从 Tracker 的职责中移除，改为 `CubeEditor` 在调用 `OnZColumnClicked` 后自行处理选中逻辑，Tracker 仅负责 z 切片索引变更。

5. **`SupportsIncrementalRefresh` 与 `INotifyCollectionChanged` 的关联机制未定义**
   - `ITableDataSource<TRow>` 新增了 `SupportsIncrementalRefresh` 属性，文档说明"若返回 `true`，实现方应同时实现 `INotifyCollectionChanged`"。但接口本身未继承 `INotifyCollectionChanged`，也未提供获取通知器的属性或方法，导致 `MapEditor`/`CubeEditor` 在运行时只能通过类型强制转换检测，增加了运行时不确定性。
   - 改进建议：在 `ITableDataSource<TRow>` 中新增 `INotifyCollectionChanged? CollectionChangedNotifier { get; }` 属性。当 `SupportsIncrementalRefresh` 为 `true` 时返回非 null 通知器实例。

6. **`ICalibrationData` 轴属性对所有维度可见，一维数据模型被迫暴露无关属性**
   - `ICalibrationData` 定义了 `YAxisName`、`YAxisUnit`、`ZAxisName`、`ZAxisUnit` 等属性。`ICurveData`（一维）继承后被迫包含这些无意义属性，违反接口隔离原则。
   - 改进建议：将轴信息属性从 `ICalibrationData` 中移除，下沉到各维度子接口：`ICurveData` 仅保留 `XAxisName`/`XAxisUnit`；`IMapData` 增加 `YAxisName`/`YAxisUnit`；`ICubeData` 增加 `ZAxisName`/`ZAxisUnit`。`CalibrationEditorBase<T>` 的信息栏格式化模板应通过类型检查动态构建显示文本。

7. **`ISurfaceChartPresenter` 的 `double[,]` 参数与 `IMapData` 逐点访问接口之间的转换开销未考虑**
   - `ISurfaceChartPresenter.LoadMapData` 和 `LoadSliceData` 均接收 `double[,]` 原始数组，但 `IMapData`/`ICubeData` 提供的是逐点访问接口（`GetValue`），没有直接返回二维/三维数组的属性。Editor 在调用图表加载方法前必须自行遍历数据构建 `double[,]` 数组，每次变量切换或 z 切片切换都会产生 O(n) 的内存分配和数据拷贝开销。
   - 改进建议：在 `IMapData` 中增加 `double[,] GetValueMatrix()` 方法（或 `ReadOnlyMemory2D<double>` 等现代内存抽象），由后端数据模型决定是返回内部数组的视图还是执行拷贝；或在 `ISurfaceChartPresenter` 接口中增加逐点数据填充的替代方法。

8. **动态列绑定 `readonly struct` 索引器与 Avalonia V12 CompiledBinding 的兼容性风险**
   - `MapRowData`/`CubeRowData` 通过索引器暴露 x 列数据，`MapEditor`/`CubeEditor` 需要动态生成 DataGrid 列，每列绑定路径为 `[0]`、`[1]` 等索引器语法。Avalonia V12 默认启用 CompiledBinding，动态生成列时无法在 XAML 中静态声明 `x:DataType`，CompiledBinding 对运行时动态构造的索引器绑定的支持情况未经验证。
   - 改进建议：补充对 CompiledBinding + 动态索引器绑定的技术验证项；备选方案：将 `MapRowData`/`CubeRowData` 改为通过 `IReadOnlyList<double> Values { get; }` 暴露 x 列数据，DataGrid 列绑定到 `Values[0]`、`Values[1]` 等，`IReadOnlyList<T>` 的属性绑定在 CompiledBinding 中的兼容性更好。

9. **空状态未覆盖数据内容为零维度的场景**
   - `CalibrationEditorBase<T>` 的空状态管理覆盖了 `ItemsSource` 为 null、空列表、`SelectedVariable` 为 null 三种场景，但未覆盖 `SelectedVariable` 有效但其内部数据维度为零的情况（如 `ICurveData.Length == 0`、`IMapData.XAxisLength == 0`、`ICubeData.ZAxisLength == 0`）。这些场景下控件可能尝试构建 0 列表格或初始化越界的 `ZIndex`，导致运行时异常或控件白屏。
   - 改进建议：在「变量切换流程」中增加对数据内容维度的检查，若 `SelectedVariable` 有效但维度为零，同样进入空状态；在 `ZSliceActivationTracker` 初始化逻辑中增加 `ZAxisLength == 0` 的防护。

10. **多个事件参数类型未定义**
    - 设计文档引用了以下事件参数类型但未给出结构定义：`CurveCellSelectedEventArgs`（`ICurveTablePresenter.SelectedCellChanged`）、`CurveCellEditCompletedEventArgs`（`ICurveTablePresenter.CellEditCompleted`）、`ChartRenderFailedEventArgs`（`IChartPresenter.RenderFailed`）。此外 `DataGridCellEditEndingEventArgs` 在 Avalonia 中的具体属性未确认。
    - 改进建议：在「核心抽象」中补充上述事件参数类型的完整结构定义；确认 Avalonia `DataGridCellEditEndingEventArgs` 的可用属性。

11. **键盘导航行为未在设计中充分定义**
    - 需求文档明确要求"键盘导航（方向键移动、Tab 切换、Enter 确认等）"，但设计文档仅在编辑触发方式中提到"Enter/失去焦点"作为输入完成触发。方向键在表格中的移动行为（尤其是 `CubeEditor` 中跨 z 切片的上下导航）、Tab 键切换顺序、`CurveEditor` 横向两行布局中的方向键行为均未定义。
    - 改进建议：在「关键行为契约」中补充键盘导航的完整行为定义，明确键盘导航触发的 `ZSliceActivationTracker.OnSelectionChanged` 调用与防抖策略的交互细节。

12. **`ZSliceActivationTracker.OnSelectionChanged(null)` 行为未定义**
    - `OnSelectionChanged(CubeRowData? row)` 的参数被标记为可空，文档说明"`null` 表示无选中项"，但未定义传入 `null` 时 Tracker 应如何处理当前激活切片。
    - 改进建议：明确 `OnSelectionChanged(null)` 的语义为"保持当前激活切片不变"；若需要重置行为，应通过独立方法（如 `ResetToDefault()`）显式触发。

## 历史迭代回顾

### 已解决的问题（出现在历史反馈但当前反馈中不再提及）

**迭代第 1 轮（14 项，v5 中已修复）：**
- 行级高亮驱动逻辑与需求定义矛盾
- z 列首行显示方案在虚拟化滚动下丢失 z 值标识
- `ICurveData`/`IMapData`/`ICubeData` 接口缺少具体方法签名
- `IChartPresenter` 缺少关键契约定义
- `ICurveTablePresenter` 缺少接口方法签名
- 数值精度和小数位数一致性展示需求未响应
- 列宽/行高自适应或手动调整需求未响应
- 数据编辑后的变更反馈机制未明确
- 变量选择下拉框显示格式缺少数据契约支撑
- 顶部信息栏单位/轴信息展示格式未定义
- 空状态与边界条件处理不完整
- 图表交互深度需求未在设计中确定
- `readonly struct` 编辑验证反馈路径未充分考虑
- `ITableDataSource` 与不同行数据类型的泛型关系未明确

**迭代第 2 轮（8 项，v5 中已修复）：**
- `readonly struct` 与 DataGrid 标准编辑机制不兼容的根因分析不准确
- 单元格编辑后全量刷新策略选中状态丢失和性能风险
- `ICalibrationData` 元素级写方法 `PropertyChanged` 约定缺失
- `ISurfaceChartPresenter` 两个加载方法参数风格不一致
- `ZSliceActivationTracker` 核心事件签名未定义
- `ICurveTablePresenter` 缺少程序化选中设置方法
- `IAsyncSaveCapable` 标记接口未定义
- z/y 行头文字在激活状态下加粗/颜色变化需求未响应

### 持续存在的问题

无。前两轮提出的问题在 v5 中均已得到响应，本轮审查未发现历史问题复发。

### 新发现的问题

本轮审查发现的 12 项问题均为 v5 中首次识别的新问题，主要集中于：
- 事实准确性（`UnloadingRow` 事件存在性）
- 接口自洽性（`IReadOnlyList` 与增量刷新矛盾、`HasUnsavedChanges` 与数量显示矛盾）
- 边界条件覆盖（零维度空状态、`OnSelectionChanged(null)` 语义）
- 事件契约完备性（事件参数类型缺失、触发源区分缺失）
- 性能与兼容性（`double[,]` 转换开销、CompiledBinding 兼容性风险）
- 交互行为定义（键盘导航未充分定义）

## 上一轮产出路径

C:\Users\jiangwei\Documents\C#\INCA_MAP_Control\redeliberations\202605111101_ood-calibration-table-controls\a_v3_design_v2.md

## 用户需求

C:\Users\jiangwei\Documents\C#\INCA_MAP_Control\redeliberations\202605111101_ood-calibration-table-controls\requirement.md
