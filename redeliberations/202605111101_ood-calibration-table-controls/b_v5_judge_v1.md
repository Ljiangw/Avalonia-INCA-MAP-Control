# Judge 裁决报告

## 审查结论汇总

经对质量审查报告、质量挑战报告及迭代历史进行综合研判：

- **双方确认为严重的问题共 3 项**：问题1（`CurrentCell` API 事实错误）、问题2（脏标记逻辑矛盾）、问题3（数组越界风险）。质量审查 Agent 与挑战 Agent 均一致认同这三项问题会阻塞编码落地或导致运行时异常。
- **存在争议的问题共 3 项**：问题4（`ILineChartPresenter` 轴标签不对称性）、问题5（选中状态恢复策略歧义）、问题9（行级高亮 `CurrentItem` API 命名错误）。挑战 Agent 认为这些问题程度较轻或已有合理设计 rationale，但不否认问题本身的存在。
- **挑战 Agent 补充的遗漏问题共 3 项**：`NewZValue` 同样需要索引边界防护、`CubeEditor` z 列点击后「选中第一个数据单元格」的边界条件缺失（`XAxisLength == 0`）、图表呈现器生命周期管理（`Detach()` 调用时机未定义）。这些补充问题进一步暴露了设计在边界防护和生命周期约定上的不足。

文档经过 4 轮迭代后整体质量显著提升，接口契约、行为契约、错误处理等维度均有大幅完善。但剩余的 3 项严重问题均属于**实现阻塞级缺陷**（编译失败、逻辑不可实现、运行时异常），无法在后续编码阶段通过简单绕过解决，必须在本阶段修正。

---

## 问题清单

| 编号 | 问题描述 | 严重程度 | 裁定 |
|------|----------|----------|------|
| 1 | Avalonia DataGrid 不存在 `CurrentCell` 属性，文档多处引用该不存在的属性，按字面编码将导致编译失败 | 严重 | 必须修复 |
| 2 | 单元格级脏标记（橙色边框）与 `IAsyncSaveCapable` 接口能力矛盾：接口仅提供全局布尔/总数，无逐单元格脏状态查询能力，该功能在当前契约下逻辑不可实现 | 严重 | 必须修复 |
| 3 | `ActiveZSliceChangedEventArgs.OldZValue` 在 `OldZIndex == -1`（空状态默认值）时存在 `ICubeData.ZValues[-1]` 数组越界风险；`NewZValue` 同理 | 严重 | 必须修复 |
| 4 | `ILineChartPresenter` 缺少显式轴标签设置契约，与 `ISurfaceChartPresenter` 不对称，实现方可能遗漏轴标签功能 | 一般 | 建议修复 |
| 5 | 全量刷新后选中状态恢复策略存在歧义：行对象引用在 `ItemsSource` 替换后失效，文档未明确区分全量/增量刷新路径的恢复策略 | 一般 | 建议修复 |
| 6 | `CubeEditor` 平铺表格排序责任未在契约层明确，`ZSliceActivationTracker` 和 z 切片分组线均隐含「相同 z 值行连续出现」假设 | 一般 | 建议修复 |
| 7 | `CubeRowData`/`MapRowData` 行头值（`ZValue`/`YValue`）类型未定义（`string` 预格式化还是 `double` 原始值），导致格式化职责归属不清 | 一般 | 建议修复 |
| 8 | 动态列生成时 `FormatString`/`DisplayPrecision` 向 `DataGridTextColumn.Binding.StringFormat` 的传递机制缺失 | 一般 | 建议修复 |
| 9 | 行级高亮驱动源描述使用 WPF 的 `CurrentItem` 概念，但 Avalonia DataGrid 中 `CurrentItem` 为 protected 只读属性，属于 API 命名错误 | 轻微 | 建议修复 |
| 10 | `CubeEditor` z 列点击后「选中第一个数据单元格」未考虑 `XAxisLength == 0` 边界情况，此时无数据单元格可选 | 一般 | 建议修复 |
| 11 | 图表呈现器生命周期管理缺失：`Detach()` 调用时机（控件卸载、变量切换时）未在设计中约定，存在 GPU/渲染资源泄露风险 | 一般 | 建议修复 |

---

## 最终裁决

**裁决结果**: RETRY

**理由**:

1. **3 项严重问题均获双方一致确认，且均为实现阻塞级缺陷**：
   - 问题1（`CurrentCell` 事实错误）：Avalonia DataGrid V12 确实不存在 `CurrentCell` 属性，文档在「单元格编辑与数据回写」「CubeEditor 跨 z 切片导航」「z 切片激活与 3D 视图联动」等关键路径中反复引用该属性，按文档字面编码将无法通过编译。必须统一替换为 `SelectedItem` + `CurrentColumn` / `DisplayIndex` 的 Avalonia 实际 API 组合策略。
   - 问题2（脏标记逻辑矛盾）：文档明确承诺「已编辑但尚未持久化的单元格以边框变色提示已修改」，但 `IAsyncSaveCapable` 契约仅提供全局 `HasUnsavedChanges` 和 `UnsavedChangeCount`，无任何逐坐标或逐单元格脏状态查询能力。在即时同步模式下，控件层若不自行维护编辑历史坐标集合，则根本无法定位「哪个单元格」触发了未保存状态。这是需求与接口能力之间的硬性矛盾，必须在「扩展接口以支持逐点脏查询」与「降级为仅全局提示」之间做出明确设计决策。
   - 问题3（数组越界风险）：`ZSliceActivationTracker` 规定 `ZAxisLength == 0` 时 `ActiveZIndex = -1`，同时定义 `OldZValue` 由 `ICubeData.ZValues[OldZIndex]` 提供。当变量从空状态切换到有效变量时，`OldZIndex = -1` 必然导致 `IndexOutOfRangeException`。`NewZValue` 在 `NewZIndex < 0` 时同样存在此风险。必须补充索引有效性防护（如 `< 0` 时返回 `double.NaN`）。

2. **挑战 Agent 补充的遗漏问题进一步暴露设计不完备**：`CubeEditor` z 列点击的边界条件缺失和图表呈现器生命周期管理缺失，均属于编码落地时不可忽视的边界场景。这些问题的发现说明设计文档在「边缘情况覆盖」维度仍有提升空间。

3. **其余一般/轻微问题虽不构成实现阻塞，但对编码指导价值有实质影响**：如排序契约缺失、行头值类型未定义、精度传递机制缺失等，若不修复将导致不同实现者行为不一致。

综上，本设计方案在核心接口契约和关键行为契约上仍存在 3 项严重缺陷，必须退回 Component A 重新设计。建议 Component A 在下一轮迭代中：
- **优先修复** 3 项严重问题（预计工作量较小）；
- **随后处理** 一般和轻微问题，特别是挑战 Agent 补充的边界条件和生命周期问题；
- 完成后重新提交 Judge Agent 审议。

RETRY
