# 质量审查报告 — Avalonia 标定数据表格编辑控件 OOD 设计方案 v4

**审查对象**：`a_v2_design_v4.md`
**审查轮次**：第 2 轮
**审查维度**：需求响应充分度、事实准确性、深度与完整性、落地可行性

---

## 严重问题

### 问题 1：`readonly struct` 与 DataGrid 标准编辑机制不兼容的根因分析错误

**问题描述**：设计文档将 `readonly struct` 行数据无法使用 DataGrid 标准编辑提交机制归因于「不可变性」：

> `readonly struct` 的不可变性在函数式语义上更清晰……`readonly struct` 与 DataGrid 标准单元格编辑的双向绑定自动提交机制不兼容：DataGrid 尝试将编辑值自动写回绑定数据源时，不可变结构体的实例替换无法生效。

这个归因是**不准确的**。核心问题并非 `readonly`，而是**值类型（`struct`）本身**。即使将行数据降级为**可变**的 `struct`，DataGrid 在 `DataContext` 中拿到的是已 boxing 的副本，标准编辑提交对该副本的修改不会反映回 `Rows` 集合中的原始实例。只有改为 `class`（引用类型）才能让 DataGrid 的标准编辑机制直接修改原集合中的对象。当前归因会让实现者误以为"改为可变 struct 就能启用标准编辑"，从而做出错误的技术决策。

**所在位置**：「设计决策 9」→「`readonly struct` vs `class` 的权衡分析」

**严重程度**：严重

**改进建议**：
- 修正根因说明：明确指出**值类型的 boxing 副本问题**是根本原因，`readonly` 只是在此基础上额外禁止了 setter 调用。
- 若保留手动事件拦截策略，应明确说明：无论 `readonly struct` 还是可变 `struct`，都必须采用手动事件拦截；只有降级为 `class` 才能利用 DataGrid 内置编辑提交。
- 补充 `class` 方案的 GC 影响和坐标索引的内联存储损失，使权衡分析完整。

---

### 问题 2：单元格编辑后的全量刷新策略存在选中状态丢失和性能风险，但未给出应对方案

**问题描述**：设计文档规定编辑完成后采用 `e.Cancel = true` + `ITableDataSource<TRow>.Refresh()` + 重新设置 `DataGrid.ItemsSource` 的刷新路径。该策略存在两个未处理的副作用：

1. **选中状态丢失**：替换 `ItemsSource` 后，DataGrid 的 `CurrentItem`、`CurrentCell` 和 `SelectedItem` 会被重置。用户在编辑一个单元格后，全量刷新会导致焦点跳回表格顶部（或默认位置），严重破坏编辑连续性。已有社区实践（WPF/Avalonia DataGrid 替换 ItemsSource 后保持选中状态）表明需要显式保存/恢复选中状态。

2. **性能风险**：对于 `CubeEditor`，`Refresh()` 需要重新生成 `z × y` 个行数据 struct 并构造新的 `IReadOnlyList<TRow>`。每次编辑单个单元格都执行 O(z×y) 的全量重建，在数据量大时（如 100×100 = 10,000 行）会造成可感知的 UI 卡顿。

设计文档未说明如何在 `Refresh()` 后恢复 `CurrentCell`/`SelectedItem`，也未评估全量刷新的性能影响或提供增量刷新替代方案。

**所在位置**：「关键行为契约」→「单元格编辑与数据回写」→「MapEditor / CubeEditor（DataGrid 纵向行模型）」小节

**严重程度**：严重

**改进建议**（按优先级排序）：
- **必选（高优先级）**：在编辑流程中补充「选中状态保持」步骤：编辑前保存 `CurrentCell` 的坐标（行索引 + 列索引），`Refresh()` 后根据新 `ItemsSource` 重新定位并恢复选中。此修复与后续是否采用增量刷新无关，是维持编辑连续性的最低要求。
- **推荐（中优先级）**：评估增量刷新可行性：若将 `ITableDataSource<TRow>.Rows` 改为 `ObservableCollection<TRow>`，可通过 `Rows[index] = newRow` 实现单元素替换，DataGrid 仅刷新对应行而无需全量重建。此方案可从根本上消除选中状态丢失和性能问题，但实现复杂度高于方案一。
- **兜底（低优先级，可与上述方案叠加）**：若坚持全量刷新或数据量极端大，应在「并发设计」或「设计决策」中明确说明性能兜底方案（如异步刷新、延迟刷新、编辑后批量刷新等）。

> **适用条件说明**：方案一（选中状态保持）是任何采用全量刷新策略时的必选修复；方案二（增量刷新）是长期推荐方向，建议在实现阶段先实施方案一确保可用性，再逐步迁移至方案二优化体验；方案三作为大数据量场景下的补充手段。

---

## 中等问题

### 问题 3：数据模型元素级变更的 `PropertyChanged` 通知约定缺失

**问题描述**：`ICalibrationData` 继承 `INotifyPropertyChanged`，但设计文档未定义当调用 `ICurveData.SetXValue(int, double)`、`IMapData.SetValue(int, int, double)`、`ICubeData.SetValue(int, int, int, double)` 等**元素级写方法**时，应触发什么 `PropertyChanged` 事件。

具体缺失包括：
- 属性名应使用什么？（空字符串/ `null` 表示全部变更？固定字符串如 `"Values"`？还是各维度特定名称？）
- 若使用空字符串/`null`，控件层订阅方如何区分「仅数据值变更」与「元信息（如轴长度）变更」？
- 后端实现方和控件消费者在这一通知约定上缺乏契约，将导致 UI 刷新不及时或过度刷新。

**所在位置**：「核心抽象」→「`ICurveData` / `IMapData` / `ICubeData`（接口）」小节；「关键行为契约」→「单元格编辑与数据回写」小节

**严重程度**：中等

**改进建议**：
- 在 `ICalibrationData` 或其子接口的文档中明确约定：元素级写方法触发 `PropertyChanged` 事件，属性名建议为 `string.Empty`（表示所有属性均可能变更），或增加一个专门的 `ValuesChanged` 事件（独立于 `INotifyPropertyChanged`）以精确通知数据矩阵变更。
- 若采用 `string.Empty`，应说明控件层在收到该通知后应执行全量刷新；若采用更细粒度的事件，应定义事件参数（如包含变更坐标范围）。

---

### 问题 4：`ISurfaceChartPresenter` 两个加载方法的参数风格不一致

**问题描述**：`ISurfaceChartPresenter` 定义了两个加载方法：

- `LoadMapData(IMapData data)` — 接收数据模型接口
- `LoadSliceData(double[,] sliceData, IReadOnlyList<double> xValues, IReadOnlyList<double> yValues)` — 接收原始数组

参数风格不统一会增加调用方困惑：`MapEditor` 调用 `LoadMapData` 时只需传入 `IMapData`，但 `CubeEditor` 调用 `LoadSliceData` 前需要自行将 `ICubeData` 的 z 切片数据遍历组装为 `double[,]`。为什么 `LoadMapData` 不也需要 `double[,]` + `xValues` + `yValues`？这种不对称没有在设计文档中给出理由。

**所在位置**：「核心抽象」→「`ILineChartPresenter` / `ISurfaceChartPresenter`（接口）」小节

**严重程度**：中等

**改进建议**：
- 统一参数风格。推荐方案：将 `LoadMapData` 也拆分为 `LoadMapData(double[,] mapData, IReadOnlyList<double> xValues, IReadOnlyList<double> yValues)`，使两个方法均接收原始数组，由调用方（`MapEditor`/`CubeEditor`）负责从数据模型提取数据。
- 若保留 `IMapData` 参数以简化 `MapEditor` 调用，应在「设计决策 5」中补充说明：为何 `MapEditor` 不需要像 `CubeEditor` 那样手动组装 `double[,]`（例如 `IMapData` 可直接被图表库消费，而 `ICubeData` 需要切片提取）。

---

### 问题 5：`ZSliceActivationTracker` 核心事件签名未定义

**问题描述**：`ZSliceActivationTracker` 被定义为「提供激活切片变更通知」的核心类，但设计文档未给出其公开事件的签名。实现者不知道：

- 事件名称是什么？（如 `ActiveSliceChanged`？）
- 事件参数类型是什么？（仅包含 `int ZIndex`？还是包含更多上下文？）
- 事件在防抖后触发还是在每次潜在变更时都触发？

**所在位置**：「核心抽象」→「`ZSliceActivationTracker`（类）」小节

**严重程度**：中等

**改进建议**：
- 补充 `ZSliceActivationTracker` 的最小事件定义，例如：
  ```csharp
  public event EventHandler<ActiveZSliceChangedEventArgs>? ActiveSliceChanged;
  public class ActiveZSliceChangedEventArgs : EventArgs
  {
      public int ZIndex { get; }
      public ActiveZSliceChangedEventArgs(int zIndex) => ZIndex = zIndex;
  }
  ```
- 明确事件触发语义：防抖结束后触发一次，还是在用户停止导航后的最终状态变更时触发。

---

### 问题 6：`ICurveTablePresenter` 缺少程序化选中设置能力

**问题描述**：`ICurveTablePresenter` 定义了 `SelectedCellChanged` 事件（输出方向）和 `CellEditCompleted` 事件，但没有定义输入方向的选中控制方法。这意味着：

- `CurveEditor` 无法在变量切换后将选中单元格重置为默认位置（如第 0 列 x 单元格）。
- `CurveEditor` 无法在图表联动时（如用户点击折线图数据点）反向选中表格对应单元格。
- 接口契约不完整， downstream consumer（如外部代码通过 `CurveEditor` 操作表格选中状态）无法实现。

**所在位置**：「核心抽象」→「`ICurveTablePresenter`（接口）」小节

**严重程度**：中等

**改进建议**：
- 在 `ICurveTablePresenter` 中补充选中设置方法，例如：
  ```csharp
  void SelectCell(int columnIndex, CurveTableRow row);
  void ClearSelection();
  ```
  其中 `CurveTableRow` 为枚举（`X` / `Z`），与 `CurveCellSelectedEventArgs` 中的行标识一致。

---

### 问题 7：脏标记跟踪的 `IAsyncSaveCapable` 接口未定义但被引用

**问题描述**：设计文档在「变更反馈机制」中引入了一个标记接口 `IAsyncSaveCapable`：

> 控件层通过检测后端是否实现 `IAsyncSaveCapable` 标记接口来决定是否启用脏标记跟踪。

但该接口在文档中没有任何定义，也没有出现在「模块职责与依赖方向」或「核心抽象」中。实现者不知道：

- 该接口应定义在哪个命名空间/程序集？
- 接口的成员是什么？（纯标记接口还是有 `HasUnsavedChanges` 属性/`SaveAsync` 方法？）
- 控件层如何检测？（`data is IAsyncSaveCapable`？）

**所在位置**：「关键行为契约」→「单元格编辑与数据回写」→「变更反馈机制（即时同步模式）」小节

**严重程度**：中等

**改进建议**：
- 在「核心抽象」中补充 `IAsyncSaveCapable` 的定义。若仅为标记接口，明确声明：
  ```csharp
  public interface IAsyncSaveCapable { }
  ```
- 若需要更丰富的契约（如脏数据查询、保存触发），应补充相应成员并说明控件层的检测和消费逻辑。

---

### 问题 8：z/y 行头文字在激活状态下的加粗/颜色变化需求未响应

**问题描述**：需求文档 v3 第 169 行明确要求：

> z 列和 y 列的行头文字在激活状态下可采用加粗或颜色变化作为辅助标识

v4 设计文档详细定义了三种高亮（单元格级蓝色、行级轻微背景色、z 切片级左侧边框/分组背景色），但**完全没有涉及 z 列和 y 列行头文字本身的加粗或颜色变化**。这是一个明确的需求响应遗漏。

**所在位置**：需求 `requirement.md` 第 169 行；设计文档「高亮叠加策略」小节

**严重程度**：中等

**改进建议**：
- 在「高亮叠加策略」中补充第四层视觉反馈：z 列/y 列行头文字在以下场景下的样式变化：
  - 当前行被选中时：y 列文字加粗或变色。
  - 当前 z 切片激活时：z 列文字加粗或变色（属于 z 切片级高亮的文字维度）。
- 明确实现方式：通过 `DataGridCell` 的样式选择器或 `LoadingRow` 事件中对行头单元格的样式类附加来实现。

---

## 轻微问题

### 问题 9：`CubeEditor` 初始默认 z 切片语义未定义

**问题描述**：设计文档在变量切换流程中说明 `CubeEditor` 会「重置 `ZSliceActivationTracker` 为默认 z 切片」，但未定义「默认」的具体含义。是 `ZIndex = 0`？上一次访问的 z 切片？还是其他策略？不同选择会影响用户体验。

**所在位置**：「关键行为契约」→「变量切换流程」小节

**严重程度**：轻微

**改进建议**：明确默认策略为 `ZIndex = 0`（第一个 z 切片），并说明后续可在实现阶段根据用户记忆偏好调整。

---

### 问题 10：`ITableDataSource.GetXIndexFromColumn` 的列索引语义不明确

**问题描述**：`GetXIndexFromColumn(int columnIndex)` 的 `columnIndex` 参数是否包含行头列（y 列、z 列）的偏移没有说明。例如 `CubeEditor` 中，DataGrid 的第 0 列是 z 行头，第 1 列是 y 行头，数据列从第 2 列开始。当 Editor 调用 `GetXIndexFromColumn` 时，传入的是 DataGrid 的原始列索引（2 → 0）还是数据列索引（0 → 0）？实现者会困惑。

**所在位置**：「核心抽象」→「`ITableDataSource<TRow>`（接口）」小节

**严重程度**：轻微

**改进建议**：在接口文档中明确：`columnIndex` 为 DataGrid 的原始列索引（包含所有行头列），由 `ITableDataSource` 实现方内部处理偏移映射。

---

### 问题 11：键盘跨 z 切片导航时表格与 3D 视图的视觉不一致未处理

**问题描述**：设计文档提到 `ZSliceActivationTracker` 对键盘快速跨 z 切片导航引入防抖（100–200ms）。但存在以下未处理的交互细节：

- 用户在当前 z 切片最后一行按「下」键进入下一 z 切片第一行时，DataGrid 的当前行会**立即**切换，但 3D 曲面图因防抖延迟刷新。
- 这导致用户在约 100–200ms 内看到「表格已显示新 z 切片的数据行，但 3D 图仍是旧 z 切片」的视觉不一致状态。

设计文档未说明这种不一致是否可接受，或是否需要在防抖期间给 3D 视图区域添加过渡指示（如淡入淡出、加载遮罩）。

**所在位置**：「关键行为契约」→「z 切片激活与 3D 视图联动（CubeEditor）」小节

**严重程度**：轻微

**改进建议**：
- 在交互设计层面明确该不一致是否可接受（标定工具场景下 100ms 延迟通常可接受）。
- 若需缓解，可在设计文档中补充：防抖期间 3D 视图区域显示轻量过渡效果（如半透明遮罩 + "切换中" 提示），或 3D 视图先立即清空再等待防抖结束后加载新数据。

---

### 问题 12：需求"推断与待确认事项"中 3D 曲面图数据源切换动画未给出决策结论

**问题描述**：需求文档 v3 第 217 行将「`CubeEditor` 切换激活 z 切片时，3D 曲面图是直接刷新还是有过渡动画」列为由设计阶段确定的事项。设计文档「设计决策 5」虽对「图表交互深度」（旋转、缩放）给出了阶段化决策，但**完全未涉及 z 切片切换时 3D 曲面图的刷新方式**（直接瞬时刷新 vs 淡入淡出过渡 vs 其他动画效果）。作为一份架构级 OOD 设计文档，若将此问题完全留给实现阶段，可能导致不同实现者做出不一致的交互决策，影响用户体验的统一性。

**所在位置**：需求 `requirement.md` 第 217 行；设计文档「设计决策 5」小节

**严重程度**：轻微

**改进建议**：在「设计决策 5」或「关键行为契约」→「z 切片激活与 3D 视图联动」中补充明确决策：z 切片切换时 3D 曲面图采用直接瞬时刷新（与表格即时响应保持一致），过渡动画作为可选增强在阶段 2 评估；或明确采用轻量淡入淡出过渡以提升视觉连贯性。

---

## 整体评价

v4 设计方案在响应第 1 轮审查意见方面表现良好：行级高亮与 z 切片级高亮的驱动源分离、z 列展示策略修正、数据契约接口成员补充、`UnloadingRow` 依赖删除、`readonly struct` 外部刷新路径补充等关键修订均已到位。

但在**落地实现层面**仍存在若干缺口：值类型编辑不兼容的根因误述、全量刷新策略的副作用未处理，以及多个接口/类的成员签名不完整（`ZSliceActivationTracker` 事件、`IAsyncSaveCapable`、`ICurveTablePresenter` 选中输入）。这些问题若不在实现前修正，将导致实现者产生困惑或做出错误的技术决策。

建议下一轮迭代优先处理**严重问题 1–2**（根因误述和全量刷新策略），其次处理**中等问题 3–8**（接口契约完整性和需求响应遗漏），最后处理**轻微问题 9–12**（语义明确和待确认事项覆盖）。

---

## 修订说明（v2）

| 质询意见 | 回应 |
|---------|------|
| **问题 1（严重）对 `MinHeight`/`MaxHeight` 的属性类型判定存在事实错误**：审查报告断言它们是 `DirectProperty`、因此不能通过 `Style` 设置，但 Avalonia 中 `Layoutable.MinHeightProperty` 与 `Layoutable.MaxHeightProperty` 实际为 `StyledProperty<double>`，完全支持样式设置。 | **接受质询，删除该问题。** 经重新核实，Avalonia `Layoutable` 的 `MinHeight` 和 `MaxHeight` 确为 `StyledProperty<double>`，通过 `Style` 设置是合法且有效的。上一版审查报告将 `StyledProperty` 误述为 `DirectProperty`，事实前提错误，故将原问题 1 从审查报告中移除。 |
| **逻辑完整性**：问题 1 的改进建议 B（在 `LoadingRow` 中逐行设置）与问题 3（原问题 3，现问题 2）建议的增量刷新方案（`ObservableCollection`）在实现路径上存在潜在冲突，审查报告未说明这些建议的优先级或适用条件。 | **已修正。** 将原问题 3 重新编号为问题 2，并在其「改进建议」中明确标注了三条方案的优先级排序和适用条件：方案一（选中状态保持）为任何全量刷新策略下的必选修复；方案二（增量刷新）为长期推荐方向；方案三（异步刷新兜底）为大数据量场景的补充手段，可与前两者叠加。 |
| **覆盖完备性**：审查报告未覆盖设计文档对需求"推断与待确认事项"的响应情况，例如 z 列合并单元格实现方式、长表格滚动性能与用户体验等。 | **已补充检查并新增问题 12。** 逐条核对需求 v3 中 9 项"推断与待确认事项"后发现：8 项已在设计文档中得到明确响应或决策（图表交互深度、编辑触发方式、z 列展示策略、长表格性能、行头可编辑性、z 切片高亮方案、键盘跨切片导航）。但第 4 项「3D 曲面图数据源切换动画」（需求第 217 行）在设计文档中完全没有涉及——设计决策 5 只讨论了图表交互深度（旋转/缩放），未对 z 切片切换时的刷新/动画方式给出决策结论。新增问题 12 指出此遗漏。 |
