# 质量挑战报告

## 逐条审查

### 问题1: Avalonia DataGrid 不存在 `CurrentCell` 属性
- **质量审查结论**: 严重。设计文档在多处基于 WPF 概念引用 `DataGrid.CurrentCell`，但 Avalonia DataGrid（含 V12）并不提供 `CurrentCell` 属性。Avalonia DataGrid 仅暴露 `CurrentColumn`（public，可读写）和 `CurrentItem`（protected，只读）。
- **挑战结论**: AGREE
- **理由**: 经核查，Avalonia V12 DataGrid 确实不存在 WPF 中的 `CurrentCell` 属性，仅提供 `CurrentColumn`（public，read/write）和 `CurrentItem`（protected，read-only）。设计文档在「单元格编辑与数据回写」「CubeEditor 跨 z 切片导航」「z 切片激活与 3D 视图联动」等关键实现路径中反复引用 `CurrentCell`，按文档字面意思编码将导致编译失败。质量审查指出的 API 事实错误成立，严重程度「严重」评估准确。补充一点：全量刷新后 `DataGrid.Columns` 通常会被重新生成，此时不仅 `CurrentCell` 不存在，保存 `CurrentColumn` 对象引用也可能因列对象重建而失效，应优先使用 `DisplayIndex` 或列索引进行状态恢复。

### 问题2: 单元格级脏标记与 `IAsyncSaveCapable` 接口矛盾
- **质量审查结论**: 严重。`IAsyncSaveCapable` 仅提供全局级别 `HasUnsavedChanges`（布尔）和 `UnsavedChangeCount`（整数总量），未暴露逐单元格脏标记查询能力，与「单元格级脏标记」需求形成不可调和矛盾。
- **挑战结论**: AGREE
- **理由**: 设计文档明确承诺「已编辑但尚未持久化的单元格以边框变色（如橙色细边框）提示已修改」，但 `IAsyncSaveCapable` 契约仅包含全局布尔状态 `HasUnsavedChanges` 和总量计数 `UnsavedChangeCount`，没有任何逐坐标或逐单元格的脏状态查询能力。在即时同步模式下，控件调用 `SetValue` 后数据已写入内存模型，模型是否完成持久化对控件透明；控件若不自行维护编辑历史坐标集合，则根本无法定位「哪个单元格」触发了未保存状态。这是一个真实存在的逻辑断层，会阻塞该功能的编码落地。必须在「扩展接口以支持逐点脏标记」与「降级为全局提示」之间做出明确决策。

### 问题3: `ActiveZSliceChangedEventArgs.OldZValue` 数组越界风险
- **质量审查结论**: 严重。当 `OldZIndex == -1`（空状态默认值）时，`ICubeData.ZValues[-1]` 将抛出 `IndexOutOfRangeException`。文档未对 `OldZIndex < 0` 时的取值做任何防护。
- **挑战结论**: AGREE
- **理由**: 设计文档规定 `ZAxisLength == 0` 时 `ActiveZIndex = -1`，同时定义 `OldZValue` 由 `ICubeData.ZValues[OldZIndex]` 提供。当变量从空状态切换到第一个有效变量时，`OldZIndex = -1` 必然导致数组越界。这是一个可预见的运行时异常路径，严重程度「严重」评估准确。需补充说明：不仅 `OldZValue` 需要防护（`OldZIndex < 0` 时返回 `double.NaN`），`NewZValue` 同样需要（`NewZIndex < 0` 时返回 `double.NaN`）。

### 问题4: `ILineChartPresenter` 缺少折线图轴标签设置契约
- **质量审查结论**: 一般。`ISurfaceChartPresenter` 提供显式 `SetAxisLabels`，`ILineChartPresenter` 缺少对应契约，不对称性使得审查和验证困难。
- **挑战结论**: PARTIAL
- **理由**: 质量审查指出的不对称确实存在，但设计选择具有合理性：`ILineChartPresenter.LoadData(ICurveData data)` 已接收包含完整轴元数据（`XAxisName`、`XAxisUnit`、`VariableName`、`Unit`）的接口实例，实现方可在 `LoadData` 内部解析并设置轴标签；而 `ISurfaceChartPresenter.LoadMapData` 接收的是原始 `double[,]` 数组，不含任何元数据，因此必须通过独立的 `SetAxisLabels` 传递轴信息。这种不对称是由数据源类型差异（富数据接口 vs 原始数值数组）决定的，并非设计缺陷。但质量审查关于「审查时难以验证实现是否满足需求」的担忧有道理——建议在设计文档中显式说明 `ILineChartPresenter` 的实现方应在 `LoadData` 中负责设置轴标签，以消除歧义。严重程度「一般」基本恰当，但更接近文档完善需求而非设计缺陷。

### 问题5: 全量刷新后的选中状态恢复策略存在歧义
- **质量审查结论**: 一般。文档描述「编辑前保存 `CurrentCell` 的列索引和行对象引用（或行索引），刷新后根据新 `ItemsSource` 重新定位...」，但全量刷新后 `CubeRowData`/`MapRowData` 为全新 struct 实例，行对象引用失效。
- **挑战结论**: PARTIAL
- **理由**: 质量审查的问题描述准确：全量刷新后 `ITableDataSource<TRow>.Refresh()` 重建完整 `Rows` 集合，`CubeRowData`/`MapRowData` 为全新分配的 struct 实例，旧的行对象引用（包括 `SelectedItem` 中 boxing 的副本）在新集合中无法可靠匹配（因为 `Values` 引用已变更，默认 struct 相等性比较会失败）。但设计文档已用「（或行索引）」暗示了替代方案，并非完全未考虑该问题。核心缺陷在于文档未明确区分两种刷新路径的差异：增量刷新（`ReplaceRow`）下不替换 `ItemsSource`，保留对象引用可行；全量刷新下必须使用「行索引 + 列索引」恢复。建议将模糊表述「（或行索引）」改为明确的分支策略说明。

### 问题6: `CubeEditor` 平铺表格的排序责任未在契约层明确
- **质量审查结论**: 一般。需求要求按 z 分组、组内按 y 排序，但 `ITableDataSource<TRow>` 接口及适配器职责中均未明确排序契约。
- **挑战结论**: AGREE
- **理由**: 需求第129行明确要求「表格按 z 轴值分组排序，每组内部按 y 轴值排序」。z 切片分组线、z 列视觉分组、`ZSliceActivationTracker` 的交互体验均隐含「相同 z 值的行在 `Rows` 集合中连续出现」的假设。若 `CubeTableAdapter` 未按 z 升序分组、组内按 y 升序生成行集合，z 切片级高亮的视觉呈现将严重受损（分组线位置错误、同一切片行不连续、z 列视觉分组断裂）。`ITableDataSource<TRow>` 接口及其适配器职责中确实缺少排序契约，可能导致不同实现者行为不一致。严重程度「一般」评估准确。

### 问题7: `CubeRowData`/`MapRowData` 行头值类型未定义
- **质量审查结论**: 一般。`ZValue`/`YValue` 是 `string`（已由适配器格式化）还是 `double`（原始值，由模板负责格式化）未明确，导致格式化职责归属不清。
- **挑战结论**: AGREE
- **理由**: 设计文档用「承载...文本」描述 `ZValue`/`YValue`，暗示 `string` 类型，但未在结构体定义中显式声明字段类型。类型歧义直接影响 `CellTemplate` 设计（是否需要额外格式化逻辑）和精度策略的职责边界（适配器预格式化 vs 模板层/转换器后格式化）。推荐明确为 `string` 并由适配器在生成行数据时按 `FormatString` → `DisplayPrecision` → `G6` 策略预格式化，此方案与 `CubeTableAdapter`/`MapTableAdapter` 已承担的精度策略消费职责一致。严重程度「一般」评估准确。

### 问题8: 动态列生成时 `FormatString`/`DisplayPrecision` 传递机制缺失
- **质量审查结论**: 一般。`ITableDataSource<TRow>` 仅提供 `ColumnHeaders` 和 `DataColumnCount`，未暴露精度信息；设计文档未说明动态列的 `Binding.StringFormat` 如何构造。
- **挑战结论**: AGREE
- **理由**: 设计文档「精度展示策略」声称适配器「在生成行数据时按 `FormatString`...格式化数值文本」，但 `MapRowData`/`CubeRowData` 的 `Values` 属性类型为 `IReadOnlyList<double>`（非 `string`），说明数据单元格的值并未在适配器层预格式化。因此数据列的数值格式化必须通过 `DataGridTextColumn.Binding.StringFormat` 在动态列生成时设置。虽然 `CalibrationEditorBase<T>` 可通过泛型约束访问 `SelectedVariable.FormatString`/`DisplayPrecision`，但设计文档未明确说明此路径，存在不同实现者采用不一致方案的风险。质量审查建议在 `ITableDataSource<TRow>` 中暴露精度信息（或由适配器直接提供绑定定义）是更集中、更清晰的方案。严重程度「一般」评估准确。

### 问题9: 行级高亮驱动源与 Avalonia DataGrid 实际 API 不匹配
- **质量审查结论**: 轻微。文档描述行级高亮「由 DataGrid 的 `CurrentItem` / 当前活动单元格所在行驱动」，但 Avalonia DataGrid 的 `CurrentItem` 是 protected 只读属性，控件外部无法访问。
- **挑战结论**: PARTIAL
- **理由**: 质量审查关于 `CurrentItem` 为 protected 的事实正确。但「高亮叠加策略」中对行级高亮驱动源的描述是概念性/视觉性的（「由 DataGrid `CurrentItem` / `CurrentCell` 状态」），并非要求控件代码后台直接访问 `CurrentItem` 属性。在 Avalonia 中，行级高亮的实际实现可通过 `DataGridRow :selected` 伪类响应 `SelectedItem` 变更来完成，这是标准做法，无需直接访问 protected 的 `CurrentItem`。文档使用 `CurrentItem` 属于 API 命名错误（WPF 概念残留），但不构成实现阻塞——实现者自然会转向 `SelectedItem`/`SelectionChanged` 完成行高亮。建议下一版中将描述修正为 `SelectedItem`，但严重程度「轻微」已恰当，不宜提升。

## 遗漏问题检查

经逐条对照迭代历史（第1~4轮全部反馈），**质量审查 Agent 未遗漏迭代历史中已记录但尚未修复的问题**。v7 版本的修订说明已完整覆盖迭代历史中的全部历史问题。

但质量审查 Agent 在以下方面存在补充空间：

1. **`NewZValue` 同样需要索引边界防护**：质量审查仅将问题标题和描述聚焦于 `OldZValue` 的越界风险，未在问题描述中同等强调 `NewZValue` 在 `NewZIndex < 0` 时存在完全相同的边界问题。虽然改进建议中「当 `NewZIndex < 0` 时，`NewZValue` 同样返回 `double.NaN」」已覆盖此点，但并列强调可避免读者遗漏。

2. **`CubeEditor` z 列点击后「选中第一个数据单元格」的边界条件**：设计文档规定 `CubeEditor` 在 `OnZColumnClicked` 后的 `ActiveSliceChanged` 事件中「自行处理选中该切片内第一个数据单元格」，但未考虑 `XAxisLength == 0`（无 x 数据列）的边界情况。此时不存在「第一个数据单元格」，若不加防护将引发无效坐标计算或异常。

3. **图表呈现器生命周期管理（`Detach()` 调用时机）**：设计文档定义了 `IAvaloniaChartPresenter.AttachTo` 和 `Detach`，但在 `CurveEditor`/`MapEditor`/`CubeEditor` 的职责描述及变量切换流程中均未说明 `Detach()` 的调用时机（如控件从视觉树卸载、变量切换导致旧图表丢弃时）。缺少生命周期约定存在图表资源（GPU 上下文、渲染目标）泄露风险。

## 总体结论

- **RETRY**: 设计存在严重问题，需要 Component A 修改。
- 核心理由：
  1. **问题1（`CurrentCell` 事实错误）**：Avalonia API 与 WPF 概念混用属于编译级阻塞错误，设计文档在多条关键实现路径中引用不存在的属性，必须修正为 `SelectedItem` + `CurrentColumn`/`DisplayIndex` 组合策略。
  2. **问题2（脏标记逻辑矛盾）**：单元格级脏标记功能在当前 `IAsyncSaveCapable` 契约下逻辑上无法实现，这是一个需求与接口能力之间的硬性矛盾，必须在「扩展接口以支持逐坐标脏查询」与「降级为仅全局提示」之间做出明确设计决策。
  3. **问题3（数组越界风险）**：`ActiveZIndex = -1` 的默认值与 `ZValues[-1]` 的取值定义之间存在可预见的运行时异常路径，需补充索引有效性防护（`OldZValue` 和 `NewZValue` 均需处理）。

其余问题（4~9）的评估基本准确，对设计完善具有积极价值，但不构成实现阻塞。建议 Component A 在下一迭代中优先修复上述 3 项严重问题（预计工作量较小），随后处理一般和轻微问题。

CHALLENGED