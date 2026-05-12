# 质量审查报告 — OOD 设计文档 v8

## 前5轮迭代历史问题修复确认

### 迭代第1轮（14项）
| # | 问题摘要 | 修复状态 | 验证说明 |
|---|---------|---------|---------|
| 1 | 行级高亮驱动逻辑与需求矛盾 | ✅ 已修复 | 「高亮叠加策略」已明确四种高亮的独立驱动源，行级高亮由 `SelectedItem` 驱动 |
| 2 | z列首行虚拟化滚动下丢失z值标识 | ✅ 已修复 | 「设计决策2」降级为「每行重复显示+视觉分组线」方案 |
| 3 | `ICurveData`/`IMapData`/`ICubeData` 缺少方法签名 | ✅ 已修复 | v8「核心抽象」中已补充完整的方法签名集 |
| 4 | `IChartPresenter` 缺少契约定义 | ✅ 已修复 | 已拆分为 `IChartPresenter`/`IAvaloniaChartPresenter`/`ILineChartPresenter`/`ISurfaceChartPresenter` |
| 5 | `ICurveTablePresenter` 缺少方法签名 | ✅ 已修复 | 已补充 `LoadData`、`SelectCell`、`ClearSelection` 等完整接口 |
| 6 | 数值精度展示需求未响应 | ✅ 已修复 | 「精度展示策略」完整定义了精度流转路径和后备规则 |
| 7 | 列宽/行高调整需求未响应 | ✅ 已修复 | 「列宽/行高调整策略」已定义 DataGrid 和 CurveEditor 的调整能力 |
| 8 | 变更反馈机制未明确 | ✅ 已修复 | 「变更反馈机制」明确即时同步模式，降级为全局未保存提示 |
| 9 | 下拉框显示格式缺少数据契约 | ✅ 已修复 | `ICalibrationData` 已补充 `GroupName`/`DisplayName` |
| 10 | 信息栏格式未定义 | ✅ 已修复 | `CalibrationEditorBase<T>` 已定义默认格式及动态构建方式 |
| 11 | 空状态边界处理不完整 | ✅ 已修复 | 「变量切换流程」和「错误处理策略」覆盖全部空状态场景 |
| 12 | 图表交互深度未确定 | ✅ 已修复 | 「设计决策5」明确分阶段策略（MVP静态展示→阶段2交互） |
| 13 | `readonly struct` 验证反馈路径 | ✅ 已修复 | 「设计决策9」充分分析，采用手动 `CellEditEnding` 拦截策略 |
| 14 | `ITableDataSource` 泛型关系 | ✅ 已修复 | 已明确 `ITableDataSource<TRow> where TRow : struct` |

### 迭代第2轮（8项）
| # | 问题摘要 | 修复状态 | 验证说明 |
|---|---------|---------|---------|
| 1 | `readonly struct` 根因分析不准确 | ✅ 已修复 | 「设计决策9」已修正为 boxing 副本问题是根本原因 |
| 2 | 全量刷新未处理选中状态丢失 | ✅ 已修复 | 「单元格编辑与数据回写」已补充选中状态保持步骤，区分全量/增量路径 |
| 3 | 元素级写方法 `PropertyChanged` 约定缺失 | ✅ 已修复 | `ICalibrationData` 中明确约定属性名使用 `string.Empty` |
| 4 | `ISurfaceChartPresenter` 参数风格不一致 | ✅ 已修复 | 两个加载方法均统一为接收 `double[,]` 原始数组 |
| 5 | `ZSliceActivationTracker` 事件签名缺失 | ✅ 已修复 | 已定义 `ActiveSliceChanged` 及 `ActiveZSliceChangedEventArgs` |
| 6 | `ICurveTablePresenter` 缺少程序化选中 | ✅ 已修复 | 已补充 `SelectCell`/`ClearSelection` 方法 |
| 7 | `IAsyncSaveCapable` 未定义 | ✅ 已修复 | 「核心抽象」中已完整定义接口成员 |
| 8 | z/y列行头文字激活样式 | ✅ 已修复 | 「高亮叠加策略」第4条已补充行头文字样式变化规则 |

### 迭代第3轮（12项）
| # | 问题摘要 | 修复状态 | 验证说明 |
|---|---------|---------|---------|
| 1 | `UnloadingRow` 事件错误声明不存在 | ✅ 已修复 | v6修订说明已修正，采用 `LoadingRow`/`UnloadingRow` 配对模式 |
| 2 | `IReadOnlyList<TRow>` 与增量刷新矛盾 | ✅ 已修复 | 新增 `ReplaceRow(int index, TRow newRow)` 显式契约入口 |
| 3 | `HasUnsavedChanges` 无法支撑数量显示 | ✅ 已修复 | 补充 `int UnsavedChangeCount { get; }` |
| 4 | Tracker "选中第一个数据单元格"通知缺失 | ✅ 已修复 | 该行为从 Tracker 移除，改由 `CubeEditor` 自行处理 |
| 5 | `SupportsIncrementalRefresh` 与 `INotifyCollectionChanged` 关联未定义 | ✅ 已修复 | 新增 `INotifyCollectionChanged? CollectionChangedNotifier { get; }` |
| 6 | `ICalibrationData` 轴属性违反ISP | ✅ 已修复 | `YAxisName`/`YAxisUnit`/`ZAxisName`/`ZAxisUnit` 下沉到维度子接口 |
| 7 | `double[,]` 与逐点访问的转换开销 | ✅ 已修复 | 新增 `GetValueMatrix()`/`GetSliceMatrix()` 方法 |
| 8 | `readonly struct` 索引器与 CompiledBinding 兼容性 | ✅ 已修复 | 改为 `IReadOnlyList<double> Values { get; }` |
| 9 | 空状态未覆盖零维度场景 | ✅ 已修复 | 变量切换流程和 Tracker 初始化均增加零维度防护 |
| 10 | 事件参数类型未定义 | ✅ 已修复 | 已补充 `CurveCellSelectedEventArgs` 等三个事件参数定义 |
| 11 | 键盘导航行为未定义 | ✅ 已修复 | 新增「键盘导航行为」完整章节 |
| 12 | `OnSelectionChanged(null)` 语义未定义 | ✅ 已修复 | 明确语义为"保持当前激活切片不变" |

### 迭代第4轮（10项，从乱码及v7修订说明推断）
| # | 问题摘要 | 修复状态 | 验证说明 |
|---|---------|---------|---------|
| 1 | `ICurveTablePresenter` 列宽调整能力缺失 | ✅ 已修复 | 补充 `MinColumnWidth`/`ColumnWidthChanged`/`ColumnWidthChangedEventArgs` |
| 2 | `CalibrationEditorBase<T>` 模板部件未定义 | ✅ 已修复 | 新增「模板部件」表格定义三个 `PART_` 部件 |
| 3 | `GetValueMatrix`/`GetSliceMatrix` 可变性语义 | ✅ 已修复 | 明确约定返回只读视图或深拷贝 |
| 4 | 编辑模式下方向键行为 | ✅ 已修复 | 「键盘导航行为」中已补充 |
| 5 | 空状态默认视觉呈现 | ✅ 已修复 | 补充默认空状态行为及 `OnEnterEmptyState`/`OnExitEmptyState` |
| 6 | 数值输入解析策略 | ✅ 已修复 | 新增「数值输入解析策略」小节 |
| 7 | `SaveAsync()` 异常语义 | ✅ 已修复 | 明确保存失败抛出异常，控件层 try/catch 捕获 |
| 8 | Tab 跨 z 切片边界行为 | ✅ 已修复 | 「CubeEditor 跨 z 切片导航」中已补充 |
| 9 | `ISurfaceChartPresenter` 缺少轴标签契约 | ✅ 已修复 | 补充 `SetAxisLabels` 方法 |
| 10 | CubeEditor z 列展示未标注需求折中 | ✅ 已修复 | 「设计决策2」开头增加显式声明区块 |

### 迭代第5轮（9项）
| # | 问题摘要 | 修复状态 | 验证说明 |
|---|---------|---------|---------|
| 1 | `DataGrid.CurrentCell` 不存在 | ✅ 已修复 | 统一替换为 `SelectedItem`+`CurrentColumn`/`DisplayIndex`+`BeginEdit()` |
| 2 | 单元格级脏标记与接口矛盾 | ✅ 已修复 | 采用方案A降级为全局未保存提示 |
| 3 | `ActiveZSliceChangedEventArgs` 数组越界 | ✅ 已修复 | 索引<0时返回 `double.NaN`，事件仅在有效索引间触发 |
| 4 | `ILineChartPresenter` 缺少轴标签 | ✅ 已修复 | 补充 `SetAxisLabels(string xName, string xUnit, string zName, string zUnit)` |
| 5 | 全量刷新选中恢复策略歧义 | ✅ 已修复 | 明确区分全量/增量两种路径的恢复方式 |
| 6 | CubeEditor 排序责任未明确 | ✅ 已修复 | `ITableDataSource<TRow>` 中明确排序契约 |
| 7 | 行头值类型未定义 | ✅ 已修复 | 明确 `ZValue`/`YValue` 为 `string` 类型 |
| 8 | `FormatString`/`DisplayPrecision` 传递机制缺失 | ✅ 已修复 | `ITableDataSource<TRow>` 中新增同名属性 |
| 9 | 行级高亮驱动源 `CurrentItem` 不可访问 | ✅ 已修复 | 修正为 `DataGrid.SelectedItem` |

**结论**：前5轮迭代历史中的全部 **53 项问题** 均已在当前 v8 设计文档中得到修复，修复措施恰当。

---

## 发现的问题

### 问题 1：`DisplayPrecision = 0` 与后备格式 `G6` 存在语义矛盾
**问题描述**：`DisplayPrecision` 被定义为"小数位数"，按语义 `0` 应表示整数格式（`F0`）。但后备规则将 `FormatString` 为空且 `DisplayPrecision == 0` 的情况视为"未提供精度信息"而使用 `G6`。由于 C# 中 `int` 默认值为 0，后端无法区分"未设置 `DisplayPrecision`"和"显式设置为 0 位小数"。若后端意图显示整数（设置 `DisplayPrecision = 0`），控件层将错误地显示 `G6` 格式（可能带小数位）。
**所在位置**：「核心抽象」→「`ICalibrationData`（接口）」`DisplayPrecision` 属性；「关键行为契约」→「精度展示策略」→「后备格式规则」
**严重程度**：一般
**改进建议**：将 `ICalibrationData.DisplayPrecision` 和 `ITableDataSource<TRow>.DisplayPrecision` 统一改为可空 `int?` 类型，`null` 表示未设置（使用 `G6` 后备），`0` 表示显式 0 位小数（使用 `F0`）。若保持 `int` 不可空，需在文档中明确声明 `DisplayPrecision = 0` 的语义为"未提供精度信息"，整数显示必须通过 `FormatString = "F0"` 实现。

### 问题 2：全量刷新后选中状态恢复策略存在实现可靠性风险
**问题描述**：该问题包含两个子问题：
- **列索引类型歧义**：文档允许编辑前保存 `CurrentColumn.DisplayIndex` 或 `Columns.IndexOf(CurrentColumn)`，但恢复时统一使用 `DataGrid.Columns[columnIndex]`。若实现者选择保存 `DisplayIndex` 而用户拖拽过列顺序，`Columns[displayIndex]` 会定位到错误列（`Columns` 集合索引与 `DisplayIndex` 在用户拖拽后不再一致）。
- **行选中 boxing 比较风险**：文档使用 `SelectedItem = ItemsSource.Cast<object>().ElementAt(rowIndex)` 恢复行选中。由于 `TRow` 为值类型，`ElementAt` 返回的 boxing 对象与 `DataGrid` 内部 `Items` 中对应位置的 boxing 对象引用不同。`DataGrid` 查找匹配项时依赖 `Equals` 比较，若 `Refresh()` 后 `Values` 字段引用全新列表实例（即使数值内容相同），默认结构体 `Equals` 可能因引用不匹配而返回 false，导致选中恢复失败。
**所在位置**：「关键行为契约」→「单元格编辑与数据回写」→「MapEditor / CubeEditor」→「选中状态保持」
**严重程度**：一般
**改进建议**：明确统一保存和恢复时使用同一种列索引（推荐保存 `Columns.IndexOf(CurrentColumn)` 并用 `Columns[index]` 恢复，或保存 `DisplayIndex` 并用 `ColumnFromDisplayIndex(displayIndex)` 恢复）；将行恢复方式改为 `DataGrid.SelectedIndex = rowIndex`，直接通过索引定位，避免值类型 boxing 后的对象比较不确定性。

### 问题 3：`DataGridRow.DataContextChanged` 事件订阅清理策略缺失
**问题描述**：`CubeEditor` 在 `LoadingRow` 中为 `DataGridRow` 订阅 `DataContextChanged` 事件作为虚拟化后备机制，但 `UnloadingRow` 事件处理器中仅提及移除 z 切片高亮样式类，未提及取消订阅该事件。在 `DataGrid` 行虚拟化机制下，行容器实例被回收重用；若每次 `LoadingRow` 都重复订阅而 `UnloadingRow` 不取消订阅，同一 `DataGridRow` 实例在多次重用后将累积多个事件处理器，可能导致同一状态变更被多次处理，且旧的事件处理器委托引用可能阻碍行容器的正确回收。
**所在位置**：「关键行为契约」→「z 切片级高亮状态与 DataGrid 虚拟化联动（CubeEditor）」
**严重程度**：一般
**改进建议**：在 `UnloadingRow` 事件处理器中补充取消订阅 `DataContextChanged` 的说明；或改为仅在 `LoadingRow` 中首次订阅一次（通过检查事件处理器是否已附加），后续重用不重复订阅。若 `LoadingRow`/`UnloadingRow` 配对机制已足够覆盖虚拟化场景，可考虑移除 `DataContextChanged` 后备机制以简化设计。

### 问题 4：`ICalibrationData.FormatString` 与 `ITableDataSource<TRow>.FormatString` 可空性不一致
**问题描述**：`ICalibrationData.FormatString` 声明为不可空 `string`，而 `ITableDataSource<TRow>.FormatString` 声明为可空 `string?`。两者语义不统一：若 `ICalibrationData.FormatString` 在"未提供"时返回空字符串 `""`，则 `ITableDataSource.FormatString` 理论上不会为 `null`；但文档的后备规则同时检查 `null` 和空字符串（"非空且非 null"），暗示两者都可能出现，增加了实现者的困惑。
**所在位置**：「核心抽象」→「`ICalibrationData`（接口）」；「核心抽象」→「`ITableDataSource<TRow>`（接口）」
**严重程度**：轻微
**改进建议**：统一两种声明为同一种可空性（推荐均使用不可空 `string`，以空字符串 `""` 表示"未提供"），并简化后备规则中的条件判断为单一判断（如仅检查 `string.IsNullOrEmpty` 或仅检查 `""`）。

### 问题 5：`ActiveZSliceChangedEventArgs` 缺少构造函数签名
**问题描述**：文档定义了 `ActiveZSliceChangedEventArgs` 的四个只读属性（`OldZIndex`、`NewZIndex`、`OldZValue`、`NewZValue`），但未定义构造函数签名。`ZSliceActivationTracker` 触发 `ActiveSliceChanged` 事件时需要构造此实例，缺少构造函数使接口不够完整，实现者无法确定参数顺序和是否需要额外验证（如索引边界检查）。
**所在位置**：「核心抽象」→「`ActiveZSliceChangedEventArgs`（类）」
**严重程度**：轻微
**改进建议**：补充构造函数签名，如 `public ActiveZSliceChangedEventArgs(int oldZIndex, int newZIndex, double oldZValue, double newZValue)`，并说明参数需由调用方（Tracker）在触发事件前验证索引范围。

### 问题 6：`CalibrationEditorBase<T>` 未提及 Avalonia `StyleKey` 要求
**问题描述**：`CalibrationEditorBase<T>` 继承自 `TemplatedControl`，Avalonia 要求自定义 `TemplatedControl` 通过重写 `StyleKeyOverride` 属性指定样式键，以便样式系统定位对应的 ControlTemplate。文档未提及此要求，实现者在编码阶段可能遗漏，导致控件无法正确应用模板。
**所在位置**：「核心抽象」→「`CalibrationEditorBase<T>`（抽象类）」→「类型形态」
**严重程度**：轻微
**改进建议**：在「类型形态」中补充说明：作为 Avalonia `TemplatedControl` 的派生类，子类需要重写 `StyleKeyOverride` 属性以提供样式键（如 `public override Type StyleKeyOverride => typeof(CalibrationEditorBase<T>);` 或各自子类的类型）。

### 问题 7：`CubeEditor` z 列点击后的选中处理时机表述存在歧义
**问题描述**：文档说 `CubeEditor` 在 `ActiveSliceChanged` 事件处理器中"或直接在其后的同步代码中"处理"选中第一个数据单元格"。两种时机的用户体验有本质差异：若放在 `ActiveSliceChanged` 事件处理器中，选中操作会在防抖延迟（100–200ms）后执行，用户点击 z 列后需等待才能看到选中变化；若在同步代码中即时执行，用户可立即看到反馈。模糊的"或"可能导致不同实现者行为不一致。
**所在位置**：「关键行为契约」→「z 切片激活与 3D 视图联动（CubeEditor）」
**严重程度**：轻微
**改进建议**：明确推荐在调用 `OnZColumnClicked(row)` 后的同步代码中即时执行"选中第一个数据单元格"操作（避免用户等待防抖延迟），3D 曲面图刷新仍由 `ActiveSliceChanged` 事件驱动（受防抖控制）。文档中删除"或"的模糊表述，给出单一推荐策略。

### 问题 8：动态列绑定兼容性论述不够精确
**问题描述**：文档声称 `IReadOnlyList<double>` 的 `Values[0]` 在 CompiledBinding 中兼容性优于自定义结构体索引器。但实际上动态生成 `DataGrid` 列的场景本就无法使用 CompiledBinding——因为 `x:DataType` 无法在 XAML 中静态声明，列绑定路径（`Values[0]`、`Values[1]` 等）是运行时动态构造的，无论使用 `IReadOnlyList` 索引器还是自定义索引器，动态列场景都需要 Runtime Binding。文档的论述可能误导读者认为 `IReadOnlyList` 索引器可以在动态列场景下使用 CompiledBinding。
**所在位置**：「核心抽象」→「`MapRowData` / `CubeRowData`」；「设计决策 14」
**严重程度**：轻微
**改进建议**：修正论述为："动态列场景需使用 Runtime Binding（`Binding` 类在代码中动态构造）。`IReadOnlyList<double>` 的属性路径在 Avalonia Runtime Binding 引擎中的解析经过更充分验证，相比自定义结构体索引器降低了实现风险。"

---

## 整体质量评价

设计方案 v8 在修复了前5轮迭代历史中的全部 **53 项问题** 后，整体质量达到良好水平，具备进入实现阶段的条件。

**优势**：
1. **接口定义完整清晰**：从 `ICalibrationData` 到维度特化子接口、表格适配层、图表抽象层的分层合理，职责边界明确。
2. **Avalonia API 使用基本正确**：`CurrentCell` 事实错误已彻底消除，`SelectedItem`+`CurrentColumn`+`BeginEdit()` 的组合覆盖了全部关键路径；`LoadingRow`/`UnloadingRow` 虚拟化策略正确。
3. **边界条件覆盖全面**：空状态、零维度、数组越界、保存异常、渲染失败、数值解析失败等场景均有明确处理策略。
4. **性能与可测试性考量充分**：`readonly struct` 行数据、增量刷新可选路径、`TimeProvider` 注入、`ZSliceActivationTracker` 独立抽取等设计体现了性能意识和可测试性意识。
5. **需求覆盖度高**：v3 需求中的核心功能（三种维度控件、图表联动、单元格编辑、选中高亮、变量切换、精度展示）均已覆盖，合并单元格降级为重复显示+分组线的技术折中已经明确标注。

**待改进项**：
- 本次审查共发现 **8 项新问题**（0 严重 / 3 一般 / 5 轻微），均不涉及架构层面的重大缺陷，主要聚焦于接口语义一致性、实现可靠性指导和文档精确性。
- 其中 `DisplayPrecision = 0` 的语义矛盾（问题1）和全量刷新选中恢复的实现风险（问题2）建议在进入实现阶段前优先澄清，以避免后端实现者与控件层之间的理解偏差。

**总体评级**：**B+（良好，可进入实现阶段，建议先修复3项一般问题）**

DIAG_WRITTEN: 8
