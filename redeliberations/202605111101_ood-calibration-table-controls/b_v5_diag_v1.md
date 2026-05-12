# 质量审查报告 — OOD 设计文档 v7

## 概述

本次审查针对第 5 轮迭代产出的架构级 OOD 设计文档（内部版本号 v7）。经逐项核查，文档在前 4 轮迭代反馈的基础上已做出大量修正，接口契约、行为契约、错误处理、空状态、键盘导航、精度策略等维度均有显著完善。但从实际落地视角审视，仍存在若干事实错误、逻辑矛盾和关键遗漏，可能阻塞编码实现或导致运行时异常。

---

## 发现的问题

### 问题 1：Avalonia DataGrid 不存在 `CurrentCell` 属性，文档中反复引用该属性属于事实错误

**问题描述**：设计文档在多处基于 WPF 概念引用 `DataGrid.CurrentCell`，但 Avalonia DataGrid（含 V12）并不提供 `CurrentCell` 属性。Avalonia DataGrid 仅暴露 `CurrentColumn`（public，可读写）和 `CurrentItem`（protected，只读）。因此，文档中所有涉及"保存/恢复 `CurrentCell`"、"程序化设置 `CurrentCell`"、"`CurrentCell` 即时响应键盘导航"的描述均无法在 Avalonia 中直接落地。实现者若按文档字面意思编码，将无法通过编译。

**所在位置**：
- 「关键行为契约」→「单元格编辑与数据回写」→「MapEditor / CubeEditor」小节（"编辑前保存 `CurrentCell` 的列索引和行对象引用..."）
- 「关键行为契约」→「CubeEditor 跨 z 切片导航的特殊行为」小节（"DataGrid 的 `CurrentCell` 即时响应键盘输入而切换"）
- 「关键行为契约」→「z 切片激活与 3D 视图联动」小节（"程序化设置 `DataGrid.CurrentCell` 到该单元格"）

**严重程度**：严重

**改进建议**：将文档中所有 `DataGrid.CurrentCell` 引用统一替换为 Avalonia 实际 API 的组合策略：
1. 保存/恢复选中状态时，保存 `CurrentColumn` 引用（或 `DisplayIndex`）和行在 `ItemsSource` 中的索引；恢复时设置 `SelectedItem = Rows[index]` 和 `CurrentColumn = column`。
2. 程序化设置当前单元格时，通过 `DataGrid.SelectedItem = targetRow` + `DataGrid.CurrentColumn = targetColumn` 实现。
3. 若需进入编辑模式，在设置 `SelectedItem`/`CurrentColumn` 后调用 `DataGrid.BeginEdit()`。
4. 在「关键行为契约」中新增小节，明确 Avalonia DataGrid 的选中状态恢复机制与 WPF 的差异。

---

### 问题 2：单元格级脏标记（橙色边框）与 `IAsyncSaveCapable` 接口能力之间存在逻辑矛盾，该功能在当前契约下无法实现

**问题描述**：设计文档要求"已编辑但尚未持久化的单元格以边框变色（如橙色细边框）提示已修改"。但 `IAsyncSaveCapable` 仅提供全局级别的 `HasUnsavedChanges`（布尔）和 `UnsavedChangeCount`（整数总量），未暴露任何逐单元格（或逐坐标）的脏标记查询能力。在即时同步模式下，控件层调用 `SetValue` 后数据已实时写入内存模型，模型是否完成持久化对控件透明。控件层若不自行维护编辑历史坐标集合，则根本无法判断"哪个单元格"触发了未保存状态。文档同时声称"控件层不跟踪编辑历史"，这与"单元格级脏标记"的需求形成了不可调和的矛盾。

**所在位置**：
- 「关键行为契约」→「单元格编辑与数据回写」→「变更反馈机制（即时同步模式）」小节（"已编辑但尚未持久化的单元格以边框变色..."）
- 「核心抽象」→「`IAsyncSaveCapable`（接口）」小节

**严重程度**：严重

**改进建议**：二选一：
- **方案 A（降级为全局提示）**：放弃单元格级脏标记，仅保留信息栏的"未保存数量/存在未保存变更"全局提示。此方案与当前 `IAsyncSaveCapable` 契约完全兼容，实现成本最低。
- **方案 B（扩展接口以支持逐点脏标记）**：在 `IAsyncSaveCapable` 中新增逐坐标脏查询能力，如 `bool IsValueUnsaved(int xIndex, int? yIndex, int? zIndex)`，或增加 `IReadOnlyList<ChangedCoordinate> UnsavedChanges { get; }`。控件层在收到 `UnsavedChangesChanged` 后遍历未保存坐标集合，为对应单元格附加/移除脏标记样式类。此方案能完全满足需求，但会显著增加后端实现负担，需在需求侧确认是否值得。

---

### 问题 3：`ActiveZSliceChangedEventArgs.OldZValue` 在 `OldZIndex == -1` 时存在数组越界风险

**问题描述**：`ZSliceActivationTracker` 的默认激活策略规定：当 `ZAxisLength == 0` 时，`ActiveZIndex` 保持为 `-1`。文档同时定义 `ActiveZSliceChangedEventArgs.OldZValue` 的语义为"由 `ICubeData.ZValues[OldZIndex]` 提供"。当变量从空状态切换到第一个有效变量（`OldZIndex = -1` → `NewZIndex = 0`）时，`ICubeData.ZValues[-1]` 将抛出 `IndexOutOfRangeException`。文档未对 `OldZIndex < 0` 时的取值做任何防护约定。

**所在位置**：
- 「核心抽象」→「`ActiveZSliceChangedEventArgs`（类）」小节（`OldZValue` 属性定义）
- 「核心抽象」→「`ZSliceActivationTracker`（类）」小节（`ActiveZIndex = -1` 的默认策略）

**严重程度**：严重

**改进建议**：明确 `ActiveZSliceChangedEventArgs` 的取值规则：当 `OldZIndex < 0` 时，`OldZValue` 返回 `double.NaN`（或文档约定的其他哨兵值）；当 `NewZIndex < 0` 时，`NewZValue` 同样返回 `double.NaN`。并在 `ZSliceActivationTracker` 的语义描述中补充：Tracker 仅在 `OldZIndex >= 0` 且 `NewZIndex >= 0` 时触发 `ActiveSliceChanged` 事件，或事件参数自身携带对无效索引的防御。

---

### 问题 4：`ILineChartPresenter` 缺少折线图轴标签的显式设置契约

**问题描述**：需求明确要求折线图"x 轴和 z 轴均显示变量名及单位作为轴标签"。`ISurfaceChartPresenter` 提供了显式的 `SetAxisLabels` 方法以设置曲面图各轴名称和单位，但 `ILineChartPresenter` 仅提供 `LoadData(ICurveData data)` 和 `HighlightDataPoint`。虽然 `ICurveData` 包含了轴信息（`XAxisName`、`XAxisUnit`、`VariableName`、`Unit`），但接口层面并未强制要求实现方在加载数据时解析并设置轴标签。这种不对称性使得 `ILineChartPresenter` 的实现方可能遗漏轴标签功能，而审查时也难以验证实现是否满足需求。

**所在位置**：
- 「核心抽象」→「`ILineChartPresenter` / `ISurfaceChartPresenter`（接口）」小节
- 需求 `requirement.md` 第 68 行（折线图轴标签要求）

**严重程度**：一般

**改进建议**：在 `ILineChartPresenter` 中补充轴标签设置契约，例如 `void SetAxisLabels(string xName, string xUnit, string zName, string zUnit)`，与 `ISurfaceChartPresenter` 保持对称。并在 `CurveEditor` 的职责中明确：加载数据后依次调用 `LoadData` 和 `SetAxisLabels`。

---

### 问题 5：全量刷新后的选中状态恢复策略存在歧义，行对象引用在 `ItemsSource` 替换后失效

**问题描述**：文档在编辑回写流程中描述："编辑前保存 `CurrentCell` 的列索引和行对象引用（或行索引），刷新后根据新 `ItemsSource` 重新定位对应单元格并恢复..." 但在全量刷新策略下，`ITableDataSource<TRow>.Refresh()` 会重新生成完整的 `Rows` 行集合，随后 Editor 重新设置 `DataGrid.ItemsSource`。新集合中的 `CubeRowData`/`MapRowData` 是全新分配的实例，编辑前保存的"行对象引用"已失效。若实现者按字面意思使用行对象引用恢复选中，将导致 `SelectedItem` 无法匹配或引发异常。文档的"（或行索引）"表述过于模糊，没有明确优先推荐哪种恢复方式。

**所在位置**：
- 「关键行为契约」→「单元格编辑与数据回写」→「MapEditor / CubeEditor」小节（"选中状态保持"段落）

**严重程度**：一般

**改进建议**：明确全量刷新后只能使用「行索引 + 列索引」恢复选中状态，禁止使用行对象引用。具体步骤：
1. 编辑前记录 `rowIndex = DataGrid.SelectedIndex`（或 `Rows.IndexOf(selectedRow)`）和 `columnIndex = DataGrid.CurrentColumn.DisplayIndex`（或 `Columns.IndexOf(CurrentColumn)`）。
2. 全量刷新并重新设置 `ItemsSource` 后，执行 `DataGrid.SelectedItem = DataGrid.ItemsSource.Cast<object>().ElementAt(rowIndex)`（或等效方式）和 `DataGrid.CurrentColumn = DataGrid.Columns[columnIndex]`。
3. 增量刷新策略下（`ReplaceRow`），因不替换 `ItemsSource`，可保留原有行对象引用恢复策略。文档应区分两种刷新路径的恢复方式。

---

### 问题 6：`CubeEditor` 平铺表格的排序责任未在契约层明确，影响 z 切片分组和高亮假设

**问题描述**：需求第 129 行明确要求"表格按 z 轴值分组排序，每组内部按 y 轴值排序"。`ZSliceActivationTracker` 和 z 切片级高亮策略均隐含假设"相同 z 值的行在 `Rows` 集合中连续出现"。但 `ITableDataSource<TRow>` 接口及 `CubeTableAdapter` 的职责描述中均未明确排序责任。如果实现方未按 z/y 排序生成行集合，z 切片分组线的视觉呈现将断裂（分组线无法准确标记切片边界），`ZSliceActivationTracker` 的索引逻辑虽然仍能工作，但用户体验（如 z 列的视觉分组）将严重受损。

**所在位置**：
- 「核心抽象」→「`ITableDataSource<TRow>`（接口）」小节
- 「核心抽象」→「`CubeRowData` / `MapRowData`」小节
- 需求 `requirement.md` 第 129 行

**严重程度**：一般

**改进建议**：在 `ITableDataSource<TRow>` 的职责描述中明确增加排序契约：`Rows` 集合必须按原始数据的多维坐标顺序排列（`CubeRowData` 按 `ZIndex` 升序分组，组内按 `YIndex` 升序排列；`MapRowData` 按 `YIndex` 升序排列）。并在 `CubeTableAdapter`/`MapTableAdapter` 的职责中明确排序是适配器的强制职责，不可省略。

---

### 问题 7：`CubeRowData`/`MapRowData` 行头值（`ZValue`/`YValue`）的类型未定义，导致格式化职责归属不清

**问题描述**：文档描述 `CubeRowData` "承载 `ZValue` 和 `YValue` 文本（用于 z/y 双列行头展示）"，`MapRowData` "承载 `YValue` 文本（用于 y 轴行头展示）"。但文档未明确这些字段是 `string`（已由适配器格式化）还是 `double`（原始值，由 DataGrid 的 CellTemplate/Binding 负责格式化）。类型歧义直接影响：
1. `DataGrid` 行头列的 `CellTemplate` 设计（若 `ZValue` 为 `double`，模板内需额外格式化逻辑）。
2. 精度策略的落地位置（若在适配器中格式化，则 `FormatString`/`DisplayPrecision` 由适配器消费；若为 `double`，则格式化逻辑需下沉到列模板或转换器）。

**所在位置**：
- 「核心抽象」→「`MapRowData` / `CubeRowData`（`readonly struct`）」小节

**严重程度**：一般

**改进建议**：明确 `ZValue` 和 `YValue` 的类型。推荐定义为 `string` 类型，由 `CubeTableAdapter`/`MapTableAdapter` 在生成行数据时按 `ICalibrationData` 的精度策略（`FormatString` → `DisplayPrecision` → `G6`）预先格式化。这样行头列的 `CellTemplate` 可直接绑定到 `ZValue`/`YValue` 并以纯文本展示，无需在模板层重复处理精度逻辑。若坚持定义为 `double`，则需在「关键行为契约」中补充行头列的格式化路径。

---

### 问题 8：动态列生成时，`FormatString`/`DisplayPrecision` 向 `DataGridTextColumn.Binding.StringFormat` 的传递机制缺失

**问题描述**：`MapEditor`/`CubeEditor` 在变量切换时需要动态生成 `DataGrid` 的数据列（x 轴列）。`DataGridTextColumn` 的 `Binding` 可设置 `StringFormat` 属性以控制数值显示格式。但 `ITableDataSource<TRow>` 仅提供 `ColumnHeaders`（列头文本）和 `DataColumnCount`，未暴露 `FormatString` 或 `DisplayPrecision`。设计文档也未说明 `CalibrationEditorBase<T>` 或 `MapEditor`/`CubeEditor` 如何将 `ICalibrationData` 的精度信息传递到动态列的 `Binding.StringFormat`。此遗漏将导致不同实现者采用不同的精度传递方式，无法保证表格与图表的数值展示一致性。

**所在位置**：
- 「核心抽象」→「`ITableDataSource<TRow>`（接口）」小节
- 「关键行为契约」→「精度展示策略」小节

**严重程度**：一般

**改进建议**：在 `ITableDataSource<TRow>` 中新增精度信息暴露点，例如：
- `string? FormatString { get; }`
- `int DisplayPrecision { get; }`

或统一暴露格式化后的列绑定：`IReadOnlyList<BindingBase> ColumnBindings { get; }`，由适配器内部为每列创建已设置 `StringFormat` 的 `Binding` 实例。Editor 在动态生成列时直接消费适配器提供的绑定定义，无需自行拼接 `Values[index]` 和格式字符串。

---

### 问题 9：行级高亮驱动源与 Avalonia DataGrid 实际 API 不匹配，`CurrentItem` 不可外部访问

**问题描述**：文档描述行级高亮"由 DataGrid 的 `CurrentItem` / 当前活动单元格所在行驱动"。但 Avalonia DataGrid 的 `CurrentItem` 是 `protected` 只读属性，控件外部（如 `CubeEditor`）无法直接访问或订阅其变更。行级高亮的实际实现应依赖 `SelectedItem`（public，可读写）或 `CurrentColumn` 的变更事件。文档使用 WPF 的 `CurrentItem` 概念来描述 Avalonia 场景，会误导实现者尝试访问不可见的 protected 成员。

**所在位置**：
- 「关键行为契约」→「高亮叠加策略」小节（"行级高亮 → DataGrid `CurrentItem` / `CurrentCell` 状态"）

**严重程度**：轻微

**改进建议**：将行级高亮的驱动源修正为 Avalonia 实际可用的 API：
- 驱动源改为 `DataGrid.SelectedItem`（或结合 `CurrentColumn`），通过订阅 `SelectionChanged` 事件或 `PropertyChanged`（`SelectedItemProperty`）实现。
- 行级高亮样式通过 `DataGridRow` 的 `:selected` 伪类或基于 `SelectedItem` 匹配的样式选择器实现，而非 `CurrentItem`。
- 同时修正「z 切片级高亮状态与 DataGrid 虚拟化联动」中 `DataContextChanged` 的使用方式，确保与 Avalonia 的 `DataGridRow.DataContext` 变更机制兼容。

---

## 整体质量评价

文档在接口契约的完备性、行为契约的精确性、错误处理的覆盖度等方面已达到较高水平，前 4 轮迭代中的绝大多数反馈已得到妥善响应。剩余的 9 项问题主要集中在 **Avalonia 实际 API 与 WPF 概念混用**（`CurrentCell`、`CurrentItem`）和 **接口能力与上层功能需求不匹配**（脏标记、轴标签、排序契约、类型定义）两个类别。这些问题若不修正，将直接阻塞编码落地或导致运行时异常。建议在下一轮迭代中优先修复 3 项严重问题，随后处理一般和轻微问题。
