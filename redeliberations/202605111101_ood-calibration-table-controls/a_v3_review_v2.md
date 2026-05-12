# OOD 设计方案审查报告（v5）

## 审查结果

APPROVED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 类型形态选择合理：
- `ICalibrationData` 继承 `INotifyPropertyChanged`，接口多继承符合 C# 约束
- `ICurveData`/`IMapData`/`ICubeData` 继承 `ICalibrationData`，单继承链清晰
- `CalibrationEditorBase<T>` 作为泛型抽象类，`T : ICalibrationData` 约束恰当
- `ITableDataSource<TRow> where TRow : struct` 泛型约束合理使用
- `MapRowData`/`CubeRowData` 选用 `readonly struct` 是有效的设计决策
- `IChartPresenter` 拆分为基接口 + `IAvaloniaChartPresenter` 子接口，类型层次清晰
- `ZSliceActivationTracker` 作为独立类，与 Avalonia 视觉类型解耦，类型边界干净
- `ActiveZSliceChangedEventArgs` 密封类继承 `EventArgs`，事件参数模型标准
- `CurveTableRow` 枚举精确表达行类型语义

### 2. 标准库与生态覆盖

**[通过]** 设计充分利用 C# 标准库和 Avalonia 生态：
- `INotifyPropertyChanged`、`IReadOnlyList<T>`、`ObservableCollection<T>`、`INotifyCollectionChanged` 均为标准库类型
- `TimeProvider` / `FakeTimeProvider` 为 .NET 8+ 标准 API，测试注入路径清晰
- `Dispatcher.UIThread.Post`/`Invoke` 为 Avalonia 标准线程调度机制
- `CellEditEnding`、`LoadingRow`、`UnloadingRow`、`CurrentCellChanged`、`DataContextChanged` 等 DataGrid 事件在 Avalonia 中均存在（已核实 Avalonia API 文档）
- `DataGrid` 的行虚拟化（`EnableRowVirtualization`）为 Avalonia 内置能力
- Avalonia V12 的 `TemplatedControl` 和依赖属性系统支撑 `CalibrationEditorBase<T>` 的设计
- 图表抽象层通过接口隔离具体图表库（OxyPlot/LiveCharts/ScottPlot 等），生态兼容策略合理

### 3. 语言特性可行性

**[通过]** 整体可行：
- `readonly struct` + 手动 `CellEditEnding` 事件拦截策略在 Avalonia DataGrid 中可实现
- `string.Empty` 作为 `PropertyChanged` 属性名是标准做法，表示所有属性均可能变更
- 异步刷新通过 `Dispatcher.UIThread.Post` 回 UI 线程的方案符合 Avalonia 并发模型
- `TimeProvider` 驱动的防抖计时器在 .NET 8+ 中可行，测试中注入 `FakeTimeProvider` 路径明确
- `IAsyncSaveCapable` 作为独立接口（不强制继承树）保持向后兼容，符合 C# 接口设计惯例
- 错误处理策略（就地验证反馈 + 空状态降级 + 图表渲染失败降级）均在 C# 能力范围内

**[轻微]** `CurrentCell` 引用在 Avalonia DataGrid 中不存在 — Avalonia DataGrid 没有 `CurrentCell` 属性（WPF 有 `DataGrid.CurrentCell` 返回 `DataGridCellInfo`），但提供了等效替代组合：`CurrentColumn`（当前列）+ `CurrentItem`/`SelectedItem`（当前行数据对象）。建议设计文档中将涉及保存/恢复选中状态的 `CurrentCell` 引用调整为 `CurrentColumn` + `SelectedItem` 的组合。

**[轻微]** 关于 `UnloadingRow` 事件的描述与事实不符 — 设计文档声称 "Avalonia `DataGrid` 不存在 `UnloadingRow` 事件"，但 Avalonia API 文档明确显示 `UnloadingRow` 事件存在（`Occurs when a DataGridRow object becomes available for reuse`），且对应 `OnUnloadingRow` 方法。虽然此描述错误不影响设计可行性（`LoadingRow` + `DataContextChanged` 的双重机制仍然有效），但建议在文档中更正该事实性描述。

### 4. 设计一致性

**[通过]** 设计一致性良好：
- 模块职责清晰，依赖方向从 Views 层向下指向数据契约层，无循环依赖
- 协作关系形成闭环：变量切换 → 数据加载 → 表格/图表渲染 → 单元格编辑 → 模型回写 → 刷新通知 → UI 更新
- `ZSliceActivationTracker` 的输入（`OnSelectionChanged`/`OnZColumnClicked`）和输出（`ActiveSliceChanged` 事件）边界清晰
- 变更反馈机制（脏标记 + 状态栏 + 模型通知）三条路径互补，无遗漏
- 四种高亮（单元格级、行级、z 切片级、行头文字样式）的驱动来源完全独立，无冲突
- `IAsyncSaveCapable` 从标记接口扩展为含状态成员的契约接口后，脏标记跟踪具备完整数据支撑
- `ISurfaceChartPresenter` 两个加载方法统一为接收原始数组的形式，参数风格一致

### 5. 设计质量

**[通过]** 设计质量良好：
- **单一职责原则**：`ZSliceActivationTracker` 专门管理 z 切片激活状态，`ICurveTablePresenter` 专门处理横向布局，`ITableDataSource<TRow>` 专门负责多维到二维的投影，职责划分清晰
- **抽象层次恰当**：架构级设计聚焦模块划分和协作契约，不陷入实现细节；`IChartPresenter` 接口抽象足够高以支持多种图表库替换
- **便于实现**：各接口的最小成员定义完整，行为契约（编辑流程、变量切换流程、z 切片激活联动）描述到足以指导后续实现
- **便于测试**：`ZSliceActivationTracker` 不依赖 Avalonia 视觉类型，可注入 `FakeTimeProvider`，可在无头环境中单元测试；`IChartPresenter`/`ITableDataSource` 等接口便于 mock

## 修改要求（仅轻微问题，不阻塞通过）

- **问题**：`CurrentCell` 属性引用在 Avalonia DataGrid 中不存在
  - **原因**：Avalonia DataGrid API 与 WPF 存在差异，没有 `CurrentCell` 属性，但有 `CurrentColumn` + `CurrentItem`/`SelectedItem` 的组合可达到相同目的
  - **建议方向**：将编辑流程中"保存 `CurrentCell`"和"恢复 `CurrentCell`"的步骤描述调整为：保存 `CurrentColumn.DisplayIndex`（或列索引）和 `SelectedItem`（行数据对象引用），刷新后通过 `ScrollIntoView` 定位行并重新设置 `CurrentColumn` 和 `SelectedItem`

- **问题**：`UnloadingRow` 事件不存在的事实性描述错误
  - **原因**：Avalonia DataGrid 实际存在 `UnloadingRow` 事件，文档中的否定陈述不正确
  - **建议方向**：更正描述为"Avalonia `DataGrid` 提供 `LoadingRow` 和 `UnloadingRow` 事件；本方案选择使用 `LoadingRow` + `DataContextChanged` 的双重机制以确保状态一致性"，或删除该注释中关于不存在的断言
