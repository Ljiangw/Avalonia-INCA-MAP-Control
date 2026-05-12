# OOD 设计方案审查报告（v4）

## 审查结果

REJECTED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 所有类型形态选择与 C# 类型系统完全兼容：
- `ICalibrationData` 继承 `INotifyPropertyChanged`（接口继承接口）合法
- `CalibrationEditorBase<T>` 作为泛型抽象类继承 `TemplatedControl`，约束 `T : ICalibrationData` 合法
- `CurveEditor`/`MapEditor`/`CubeEditor` 使用 `sealed class` 继承泛型抽象类合法
- `ITableDataSource<TRow> where TRow : struct` 泛型约束合法
- `MapRowData`/`CubeRowData` 作为 `readonly struct` 合法
- `IChartPresenter` 接口抽象与 `ZSliceActivationTracker` 独立类的类型形态合理

### 2. 标准库与生态覆盖

**[一般]** Avalonia `DataGrid` 不存在 `MinRowHeight`/`MaxRowHeight` 属性 — 设计方案在「列宽/行高调整策略」中提出"通过 `MinRowHeight` / `MaxRowHeight` 限制范围"，但 Avalonia `DataGrid` API 仅提供 `RowHeight` 属性，不存在 `MinRowHeight` 和 `MaxRowHeight` 属性（此为 WPF DataGrid 的属性）。该策略无法按描述直接实现。

**[轻微]** `DataErrorValidationRule` 是 WPF 验证类，Avalonia 中不存在 — 方案在「错误处理策略」中以此类作为被放弃的内置验证模板示例，但 Avalonia 的数据验证机制基于 `DataValidationErrors` 附加属性与 `INotifyDataErrorInfo`/`IDataErrorInfo`，不沿用 WPF 的 `ValidationRule` 体系。术语引用不准确，但不影响最终的手动验证决策。

**[轻微]** `DataGridLength.SizeToHeader` 需确认 Avalonia V12 等价物 — 方案建议 x 轴数据列采用 `DataGridLength.SizeToHeader` 初始化宽度，该值在 WPF DataGrid 中存在，但 Avalonia 的 `DataGridLength` 构造方式可能不同。实际实现中可通过 `DataGridLength.Auto` 或固定像素值替代，影响可控。

### 3. 语言特性可行性

**[通过]** 错误处理策略与 C# / Avalonia 能力匹配：
- `CellEditEnding` 事件在 Avalonia DataGrid 中存在，且 `DataGridCellEditEndingEventArgs` 支持 `Cancel` 属性，与方案描述一致
- `Dispatcher.UIThread.Post` 在 Avalonia 中存在，用于 Tracker 到 UI 线程的调度正确
- `LoadingRow` / `UnloadingRow` 事件在 Avalonia DataGrid 中存在，z 切片高亮联动机制可行
- `readonly struct` 作为 `DataGrid.ItemsSource` 可行，`DataGridRow.DataContext` 会自动装箱，但不影响功能

### 4. 设计一致性

**[一般]** `ZSliceActivationTracker` 职责中保留"z 列分组首行点击"逻辑，与"每行重复显示"策略矛盾 — 设计决策 2 已明确 z 列展示策略为「每行重复显示 + 视觉分组线」，不存在"首行"或"合并单元格"概念。但 `ZSliceActivationTracker` 的职责描述仍包含"处理 z 列分组首行的点击事件"，「关键行为契约」中也保留"点击 z 列分组区域"的描述。这使得实现者无法判断 z 列点击交互的实际语义：是仅首行可点击，还是每行均可点击？该矛盾可能导致实现与预期不符。

**[轻微]** `CubeRowData` 缺少 x 列索引映射定义 — 方案说明 `CubeRowData` "每行对应一个 (z, y) 组合下所有 x 列的数据值"，但编辑单元格时需要知道当前编辑的是第几列 x。「关键行为契约」中提到"结合该单元格所在的 x 列索引 `XIndex`"，但 `CubeRowData` 的职责定义中未说明 x 列索引的映射方式（如通过动态属性、`Dictionary<int, double>`、列 `Tag` 绑定等）。此遗漏对架构级设计影响有限，但需在详细设计阶段补全。

### 5. 设计质量

**[通过]** 整体职责划分清晰，遵循单一职责原则：
- `ZSliceActivationTracker` 将 z 切片激活逻辑从 `CubeEditor` 中剥离，可独立单元测试，设计良好
- `ICurveTablePresenter` 将 CurveEditor 的横向布局与 MapEditor/CubeEditor 的纵向 DataGrid 模型解耦，抽象层次恰当
- 数据契约层无 outward UI 依赖，依赖方向合理无循环

**[轻微]** `FormatString` 与 `DisplayPrecision` 的优先级关系未明确 — `ICalibrationData` 同时暴露两个精度控制属性，但未说明当两者同时存在时的消费优先级（如 `FormatString` 优先、`DisplayPrecision` 作为后备，或两者独立作用于不同场景）。这可能导致表格适配层和图表实现方对精度策略的理解不一致。建议明确两者的协作规则。

## 修改要求（REJECTED 时存在）

### 问题 1：Avalonia DataGrid 不存在 `MinRowHeight`/`MaxRowHeight` 属性

- **问题**：设计方案在「列宽/行高调整策略」中提出通过 `DataGrid.MinRowHeight` / `DataGrid.MaxRowHeight` 限制行高范围，但 Avalonia `DataGrid` 类没有这两个属性（仅有 `RowHeight`）。
- **原因**：Avalonia 的 `DataGrid` API 与 WPF 存在差异，该属性缺失意味着实现者按文档直接编码时会遇到编译错误，需要重新寻找替代方案。
- **建议方向**：将行高限制策略改为通过 `DataGridRow` 样式实现。在 `DataGrid.Styles` 中添加 `Style Selector="DataGridRow"`，在其中设置 `Setter Property="MinHeight" Value="..."` 和 `Setter Property="MaxHeight" Value="..."`。这是 Avalonia 中限制行高的可行方式。

### 问题 2：`ZSliceActivationTracker` 中"分组首行点击"逻辑与展示策略矛盾

- **问题**：设计决策 2 已将 z 列展示策略改为「每行重复显示」，不存在"分组首行"概念。但 `ZSliceActivationTracker` 的职责描述和「z 切片激活与 3D 视图联动」行为契约中仍保留"点击 z 列分组首行/分组区域"的交互逻辑。
- **原因**：设计文档内部存在矛盾。实现者阅读「设计决策 2」后认为 z 列每行都显示相同值，但阅读「关键行为契约」时又会看到"分组首行点击"的描述，无法确定实际交互规则。
- **建议方向**：统一修正 `ZSliceActivationTracker` 的职责描述和「关键行为契约」中的交互逻辑。由于 z 列每行都重复显示 z 值，用户点击 z 列的**任意单元格**均可将该 z 值设为激活切片。删除所有"分组首行""分组区域"的表述，改为"点击 z 列任意单元格"。
