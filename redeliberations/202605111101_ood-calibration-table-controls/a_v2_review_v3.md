# OOD 设计方案审查报告（v4）

## 审查结果

REJECTED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 类型形态选择合理且与 C# 类型系统能力匹配。

- `ICalibrationData` 作为接口明确继承 `INotifyPropertyChanged`，类型形态正确。
- `ICurveData` / `IMapData` / `ICubeData` 采用接口继承 `ICalibrationData`，避免了后端领域模型可能面临的单继承限制。
- `CalibrationEditorBase<T>` 作为泛型抽象类（`T : ICalibrationData`）继承 `TemplatedControl`，可在 Avalonia 类型系统中注册公共依赖属性，形态恰当。
- `CurveEditor` / `MapEditor` / `CubeEditor` 使用密封类，作为领域终态控件无进一步派生需求，设计合理。
- `ITableDataSource<TRow> where TRow : struct` 的泛型约束在 C# 泛型系统能力范围内。
- `ILineChartPresenter` / `ISurfaceChartPresenter` 继承 `IChartPresenter` 的接口层次清晰，拆分依据（数据加载方式与交互反馈差异）充分。
- `ZSliceActivationTracker` 作为独立类，不依赖 Avalonia 视觉类型，仅消费 `CubeRowData` 行数据，类型交互模式可在 C# 中干净实现。

### 2. 标准库与生态覆盖

**[一般]** 设计方案依赖 `DataGrid.UnloadingRow` 事件在行回收前清理 z 切片高亮样式类，但 **Avalonia 的 `DataGrid` 不存在 `UnloadingRow` 事件**。Avalonia DataGrid 仅提供 `LoadingRow` 事件，没有对应行卸载事件。若缺少清理机制，被回收的 `DataGridRow` 容器残留的 `active-z-slice` 样式类可能在重用于其他行数据时导致错误的视觉状态，破坏 z 切片高亮的正确性。

**[轻微]** 设计方案声称 `readonly struct` 行数据可"彻底消除接口装箱带来的 GC 压力"，但 `DataGrid` 在将行数据赋给 `DataGridRow.DataContext`（`object` 类型属性）时，值类型必然发生 boxing。虽然避免了接口引用传递中的额外装箱，但无法避免 DataGrid 自身的 DataContext boxing 机制。表述需要修正以避免实现阶段对 GC 收益的错误预期。

### 3. 语言特性可行性

**[通过]** 语言特性使用与 C# 及 Avalonia 能力匹配。

- `CellEditEnding` 事件拦截策略在 Avalonia DataGrid 中可行，事件参数支持 `Cancel` 属性取消默认提交。
- `Dispatcher.UIThread.Post` 的线程调度方式符合 Avalonia 的 UI 线程亲和性模型。
- `DataGrid.Styles` 中通过选择器 `DataGridRow` 设置 `MinHeight`/`MaxHeight` 是 Avalonia 支持的样式应用方式。
- `CanUserResizeColumns` 是 Avalonia DataGrid 的内置属性，列宽手动调整能力存在。
- 即时同步编辑回写模式在事件驱动架构中完全可行。
- 图表渲染异常内部捕获并通过事件通知降级的策略符合 C# 异常处理模式。

### 4. 设计一致性

**[一般]** `readonly struct` 行数据的值变更自动刷新路径不完整。设计文档在「单元格编辑与数据回写」和「错误处理策略」中多次声称"模型触发 `PropertyChanged` → 绑定系统自动刷新关联单元格"，但 `MapRowData` / `CubeRowData` 作为 `readonly struct`：
1. 不支持 `INotifyPropertyChanged`，绑定系统无法监听其属性变更；
2. 若值以内联方式存储，已生成的行数据实例是原始数据的值副本，外部模型变更后副本不会自动更新；
3. 若值通过索引器动态读取原始数据，绑定系统监听的 boxing 后的 `DataContext` 对象仍不会转发原始模型的 `PropertyChanged` 事件。

因此，当原始数据模型在非编辑路径下变更值并触发 `PropertyChanged` 时，DataGrid 不会自动刷新对应单元格。设计文档未明确 `MapEditor` / `CubeEditor` 在此场景下的刷新策略（如重新生成 `Rows` 集合并重新设置 `ItemsSource`、调用显式刷新方法等）。此外，`ITableDataSource<TRow>` 缺少与 `ICurveTablePresenter.Refresh()` 对等的刷新契约，导致 `MapEditor` / `CubeEditor` 的数据刷新策略在适配层层面未闭环。

**[轻微]** `ZSliceActivationTracker` 的防抖策略（100–200ms）被描述为"纯 .NET"逻辑，但实现该防抖需要某种定时器机制（如 `Task.Delay` 或 `CancellationTokenSource`）。若使用 `Task.Delay`，事件触发可能不在 UI 线程；虽然 `CubeEditor` 的订阅者通过 `Dispatcher.UIThread.Post` 调度，但事件源线程与调度假设之间的关系未在设计中明确，存在实现歧义。

### 5. 设计质量

**[轻微]** `ISurfaceChartPresenter` 声明了两个 `LoadData` 重载，参数差异为类型不同（`IMapData` vs `double[,] + IReadOnlyList<double> + IReadOnlyList<double>`）而非数量递进。重载基于参数数量区分，调用方的意图表达不够显式。建议将两个重载拆分为语义明确的独立方法名（如 `LoadMapData(IMapData)` 和 `LoadSliceData(double[,], ...)`），提升接口可读性和调用安全性。

**[轻微]** 脏标记视觉反馈（橙色边框）和状态栏未保存提示的设计假设后端存在异步保存机制。若后端为即时持久化（无存盘按钮/无异步保存概念），此反馈机制将永久不触发。建议在设计中明确该机制的启用条件（如后端实现 `IAsyncSaveCapable` 标记接口），避免实现阶段困惑。

## 修改要求（REJECTED 时存在）

### 问题1：`DataGrid.UnloadingRow` 事件不存在

- **问题**： Avalonia DataGrid 无 `UnloadingRow` 事件，设计方案依赖该事件清理行回收前的样式类残留。
- **原因**： 缺少清理机制会导致虚拟化行回收后，残留的 `active-z-slice` 样式类被错误地应用到后续重用的行上，造成 z 切片高亮状态错乱。
- **建议方向**：
  1. 在 `LoadingRow` 事件处理器中，**先无条件清理所有已附加的 z 切片样式类，再根据当前行数据判断是否附加**，替代依赖 `UnloadingRow` 的清理逻辑。
  2. 或者利用 `DataGridRow` 的 `DataContextChanged` 事件（Avalonia 中 `Control` 基类支持），在 `DataContext` 变更时清理样式类。
  3. 删除审查报告中所有引用 `UnloadingRow` 的描述，更新「z 切片级高亮状态与 DataGrid 虚拟化联动」小节的实现机制。

### 问题2：`readonly struct` 行数据值变更后的自动刷新路径缺失

- **问题**： 设计方案声称 `PropertyChanged` 可自动刷新 DataGrid 单元格，但 `readonly struct` 行数据不支持属性变更通知，绑定系统无法自动感知原始数据模型的值变更。
- **原因**： 该路径缺失会导致非编辑路径下的数据变更（如后端标定数据从 ECU 更新后）无法正确反映到表格 UI，造成表格显示与底层数据不一致。
- **建议方向**：
  1. 在 `ITableDataSource<TRow>` 接口中补充 `void Refresh()` 方法（与 `ICurveTablePresenter.Refresh()` 对齐），明确 `MapEditor` / `CubeEditor` 在接收到原始数据 `PropertyChanged` 后的调用契约。
  2. 在「关键行为契约」中明确数据变更刷新路径：外部数据变更 → `Editor` 检测到 `PropertyChanged` → 调用 `ITableDataSource.Refresh()` 重新生成 `Rows` → 重新设置 `DataGrid.ItemsSource`（或调用 Avalonia 等效的集合刷新机制）。
  3. 或者，若希望避免全量重新生成行集合，考虑将 `MapRowData` / `CubeRowData` 中的数据值属性改为通过索引器动态读取原始数据模型（持有 `IMapData`/`ICubeData` 引用），并在 `Editor` 层订阅原始模型的 `PropertyChanged` 后手动触发对应 `DataGridRow` 的刷新。此方向需要在设计中明确索引器的实现契约。
