# 质量挑战报告

## 逐条审查

### 问题1：`DisplayPrecision = 0` 与后备格式 `G6` 存在语义矛盾
- **质量审查结论**：一般 — `int` 默认值 0 无法区分"未设置"与"显式 0 位小数"，建议改为 `int?`
- **挑战结论**：AGREE
- **理由**：经核对，`ICalibrationData.DisplayPrecision` 声明为 `int`（不可空），`ITableDataSource<TRow>.DisplayPrecision` 同样为 `int`。在 C# 中 `int` 默认值就是 0，后端实现方无法通过返回 `0` 来表达"我想显示整数"（`F0`）还是"我没设置精度"（应 fallback 到 `G6`）。文档的后备规则将 `DisplayPrecision == 0` 统一视为 fallback 到 `G6`，确实剥夺了后端表达"强制整数显示"的能力。此问题真实存在，严重程度"一般"评估合理。建议的 `int?` 改进方案或"明确声明语义"方案均可行。

### 问题2：全量刷新后选中状态恢复策略存在实现可靠性风险
- **质量审查结论**：一般 — 包含列索引类型歧义和行选中 boxing 比较风险两个子问题
- **挑战结论**：AGREE
- **理由**：
  - **子问题A（列索引类型歧义）**：文档允许编辑前保存 `CurrentColumn.DisplayIndex` 或 `Columns.IndexOf(CurrentColumn)`，但恢复时统一使用 `DataGrid.Columns[columnIndex]`。若实现者选择保存 `DisplayIndex`，而用户此前拖拽过列顺序，则 `Columns[displayIndex]` 会定位到视觉顺序不同的列。此问题真实存在，改进建议（统一使用同一种索引类型）合理。
  - **子问题B（boxing 比较风险）**：文档使用 `ItemsSource.Cast<object>().ElementAt(rowIndex)` 恢复 `SelectedItem`。`MapRowData`/`CubeRowData` 为 `readonly struct`，`ElementAt` 返回的 boxing 对象与 `DataGrid` 内部 `Items` 中的 boxing 对象引用不同。`DataGrid` 的 `SelectedItem` 设置逻辑内部依赖对象匹配，默认结构体 `Equals` 会逐字段比较，其中 `Values` 字段为 `IReadOnlyList<double>` 类型（引用比较），若 `Refresh()` 后 `Values` 引用新列表实例，即使数值内容相同，`Equals` 也会返回 `false`，导致选中恢复失败。改进建议改用 `SelectedIndex = rowIndex` 完全正确。此问题真实存在，严重程度"一般"评估合理。

### 问题3：`DataGridRow.DataContextChanged` 事件订阅清理策略缺失
- **质量审查结论**：一般 — `LoadingRow` 中订阅 `DataContextChanged` 但 `UnloadingRow` 未提及取消订阅
- **挑战结论**：AGREE
- **理由**：文档在「z 切片级高亮状态与 DataGrid 虚拟化联动」中明确说明 `CubeEditor` 在 `LoadingRow` 中为 `DataGridRow` 订阅 `DataContextChanged` 事件作为后备机制，在 `UnloadingRow` 中仅提及"移除该行的 z 切片高亮样式类"，确实未提及取消订阅 `DataContextChanged`。在 Avalonia `DataGrid` 行虚拟化机制下，行容器实例被回收重用，重复订阅而不取消将导致事件处理器累积，可能引发重复处理和内存泄漏。此问题真实存在，严重程度"一般"评估合理。

### 问题4：`ICalibrationData.FormatString` 与 `ITableDataSource<TRow>.FormatString` 可空性不一致
- **质量审查结论**：轻微 — `ICalibrationData` 声明为 `string`，`ITableDataSource` 声明为 `string?`
- **挑战结论**：AGREE
- **理由**：经核对原文，`ICalibrationData.FormatString` 确实声明为不可空 `string`，而 `ITableDataSource<TRow>.FormatString` 声明为可空 `string?`。若 `ICalibrationData` 端不可空，则理论上不会返回 `null`，`ITableDataSource` 端的 `string?` 永远不会为 `null`，造成接口语义不一致和实现者困惑。文档的后备规则同时检查 `null` 和空字符串（"非空且非 null"），也侧面印证了这种不一致。此问题真实存在，严重程度"轻微"评估合理。

### 问题5：`ActiveZSliceChangedEventArgs` 缺少构造函数签名
- **质量审查结论**：轻微 — 定义了四个只读属性但未定义构造函数
- **挑战结论**：CHALLENGE
- **理由**：
  1. **OOD 文档的定位**：本方案为架构级 OOD 设计文档，关注的是模块职责、接口契约、交互行为和依赖关系。构造函数签名属于实现层面的细节，在 OOD 层面不做强制要求是正常的工程实践。
  2. **审查Agent的不一致性**：v8 文档中定义了多个事件参数类型（`CurveCellSelectedEventArgs`、`CurveCellEditCompletedEventArgs`、`ChartRenderFailedEventArgs`、`ColumnWidthChangedEventArgs`、`ActiveZSliceChangedEventArgs`），**全部均未定义构造函数**。审查Agent仅针对 `ActiveZSliceChangedEventArgs` 提此问题，对其他同样缺少构造函数的事件参数类型未做任何提及，存在明显的选择性审查不一致。
  3. **不影响架构正确性**：缺少构造函数签名不会导致架构层面的设计缺陷或实现阻塞——C# 中只读属性的初始化方式（构造函数参数、对象初始化器、工厂方法）是基本的编码常识。
  因此，此问题不应作为需要修复的设计缺陷列入。

### 问题6：`CalibrationEditorBase<T>` 未提及 Avalonia `StyleKey` 要求
- **质量审查结论**：轻微 — 继承 `TemplatedControl` 但未提及 `StyleKeyOverride`
- **挑战结论**：PARTIAL
- **理由**：
  1. **这不是设计缺陷**：`StyleKeyOverride` 是 Avalonia 框架中所有自定义 `TemplatedControl` 的通用编码要求，属于框架常识而非架构设计层面的关注点。将其列为"设计问题"有过度审查之嫌——类似于要求 OOD 文档提及 C# 的命名空间声明或 `using` 语句。
  2. **但有一定价值**：考虑到文档已经详细到模板部件（`PART_VariableSelector` 等）级别，补充 `StyleKeyOverride` 的说明可以提升文档对实现者的指导价值。不过，这属于"文档增强"而非"设计缺陷"。
  建议：不将其作为必须修复的问题，但可作为可选的文档完善项在进入实现阶段时顺手补充。

### 问题7：`CubeEditor` z 列点击后的选中处理时机表述存在歧义
- **质量审查结论**：轻微 — "事件处理器中（或直接在其后的同步代码中）"的"或"造成两种时机模糊
- **挑战结论**：AGREE
- **理由**：文档原文使用了"或"字："于 `ActiveSliceChanged` 事件处理器中（或直接在其后的同步代码中）自行处理"。两种时机的用户体验有本质差异：同步代码中即时执行可立即给用户反馈；放在 `ActiveSliceChanged` 事件处理器中则需等待 100–200ms 防抖延迟。模糊的"或"确实可能导致不同实现者行为不一致。改进建议（明确推荐同步代码中即时执行）合理。此问题真实存在，严重程度"轻微"评估合理。

### 问题8：动态列绑定兼容性论述不够精确
- **质量审查结论**：轻微 — 声称 `IReadOnlyList` 在 "CompiledBinding" 中兼容性更优，但动态列场景本就无法使用 CompiledBinding
- **挑战结论**：AGREE
- **理由**：文档原文声称 `IReadOnlyList<double>` 的属性绑定在 "CompiledBinding 中的兼容性更优"。但实际上，动态生成 `DataGrid` 列的场景（列数和数据点数量在运行时确定）根本**无法使用 CompiledBinding**——因为 `x:DataType` 必须在 XAML 中静态声明，而动态列的绑定路径（`Values[0]`、`Values[1]` 等）是在代码中运行时构造的。此场景无论使用何种数据暴露方式，都必须使用 Runtime Binding。文档的论述确实具有误导性，可能让读者误以为 `IReadOnlyList` 索引器可以在动态列场景下使用 CompiledBinding。改进建议（修正为 Runtime Binding 的兼容性论述）完全正确。此问题真实存在，严重程度"轻微"评估合理。

---

## 遗漏问题检查

### 审查Agent未发现的遗漏问题

经独立交叉验证，发现以下 **1 项** 被质量审查Agent遗漏的问题：

**遗漏问题：`ZSliceActivationTracker.ResetToDefault()` 与防抖计时的交互未明确**
- **问题描述**：文档规定变量切换时 `CubeEditor` 调用 `ZSliceActivationTracker.ResetToDefault()` 重置激活切片。但文档未明确 `ResetToDefault()` 是否应取消正在进行的防抖计时。若用户在变量A上快速导航触发了防抖计时，在计时结束前切换变量B并调用 `ResetToDefault()`，旧的防抖计时到期后可能触发基于变量A的 `ActiveSliceChanged` 事件，而此时图表已加载变量B的数据，可能导致 3D 曲面图显示错误状态。
- **所在位置**：「核心抽象」→「`ZSliceActivationTracker`（类）」；「变量切换流程」
- **严重程度**：一般
- **说明**：此问题是否暴露取决于 `CubeEditor` 变量切换时是否重新创建 `ZSliceActivationTracker` 实例。文档措辞为"调用 `ResetToDefault()`"而非"创建新的 Tracker"，暗示是同一个实例。若实现者不额外处理防抖计时的取消，将存在运行时风险。

### 前5轮迭代历史问题修复确认

经逐条独立核对，前5轮迭代历史中的 **53 项问题** 确实均已在 v8 设计文档中得到恰当的修复，质量审查Agent的修复确认结论准确。关键验证点：
- `DataGrid.CurrentCell` 已全部替换为 `SelectedItem` + `CurrentColumn` + `BeginEdit()` ✅
- 单元格级脏标记已降级为全局未保存提示 ✅
- `ActiveZSliceChangedEventArgs` 数组越界已防护 ✅
- `UnloadingRow` 事件事实错误已修正 ✅
- `IReadOnlyList<TRow>` 与增量刷新矛盾已通过 `ReplaceRow` 解决 ✅
- `HasUnsavedChanges` 与数量显示矛盾已通过 `UnsavedChangeCount` 解决 ✅
- 轴属性下沉已完成 ✅
- 动态列绑定已从索引器改为 `IReadOnlyList<double> Values` ✅

---

## 总体结论

### 审查Agent发现问题统计（经挑战后）

| 问题 | 原评级 | 挑战结论 | 说明 |
|------|--------|---------|------|
| 问题1：`DisplayPrecision` 语义矛盾 | 一般 | AGREE | 真实问题 |
| 问题2：全量刷新选中恢复风险 | 一般 | AGREE | 真实问题，含两个有效子问题 |
| 问题3：`DataContextChanged` 未取消订阅 | 一般 | AGREE | 真实问题 |
| 问题4：`FormatString` 可空性不一致 | 轻微 | AGREE | 真实问题 |
| 问题5：`ActiveZSliceChangedEventArgs` 缺构造函数 | 轻微 | **CHALLENGE** | OOD层面非必须，且审查Agent选择性审查 |
| 问题6：未提及 `StyleKeyOverride` | 轻微 | **PARTIAL** | 编码常识，非设计缺陷 |
| 问题7：z 列点击选中时机歧义 | 轻微 | AGREE | 真实问题 |
| 问题8：动态列绑定论述不精确 | 轻微 | AGREE | 真实问题 |

### 遗漏问题

经独立验证，发现 **1 项遗漏问题**（`ResetToDefault()` 与防抖计时交互，一般级别）。

### 综合评估

- **架构层面**：v8 设计的模块划分、接口契约、依赖方向、交互行为均无重大缺陷。
- **Avalonia API 使用**：`CurrentCell` 事实错误已彻底消除，`SelectedItem`/`CurrentColumn`/`BeginEdit()` 组合覆盖全部关键路径；`LoadingRow`/`UnloadingRow` 虚拟化策略正确。
- **需求覆盖度**：v3 需求的核心功能（三种维度控件、图表联动、单元格编辑、选中高亮、变量切换、精度展示）均已覆盖。
- **边界条件**：空状态、零维度、数组越界、保存异常、渲染失败、数值解析等场景均有明确处理。
- **发现的问题**：本次审查和交叉验证共发现 **7 项有效问题**（问题5被挑战排除，问题6降级为可选增强），加上 **1 项遗漏问题**，全部为"一般"或"轻微"级别，不涉及架构层面的重大缺陷。

**总体评级**：**B+（良好，可进入实现阶段）**

所有发现的有效问题均为实现层面的语义明确性和可靠性指导问题，可以在进入实现阶段后逐一处理，不需要阻塞设计文档定稿。

CHALLENGED
