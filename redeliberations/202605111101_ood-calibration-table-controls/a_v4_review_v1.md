# OOD 设计方案审查报告（v6）

## 审查结果

APPROVED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 类型形态选择均与 C# 类型系统能力匹配：
- `ICalibrationData` 接口继承 `INotifyPropertyChanged`，接口多继承合法
- `ICurveData`/`IMapData`/`ICubeData` 继承 `ICalibrationData`，接口层级关系清晰
- `CalibrationEditorBase<T>` 泛型抽象类（`T : ICalibrationData`）继承 `TemplatedControl`，泛型约束与单继承兼容
- `CurveEditor`/`MapEditor`/`CubeEditor` 密封类继承泛型基类，符合 C# 密封类型语义
- `ITableDataSource<TRow> where TRow : struct` 泛型接口约束正确
- `MapRowData`/`CubeRowData` 为 `readonly struct`，值类型语义精确
- `ZSliceActivationTracker` 独立类，不依赖 Avalonia 视觉类型，可测试性良好
- `IAsyncSaveCapable` 独立于 `ICalibrationData` 继承树，保持向后兼容性

**[轻微]** `MapRowData`/`CubeRowData` 作为 `readonly struct` 包含 `IReadOnlyList<double>` 引用类型字段。当 `struct` 被默认构造时（如 `new MapRowData()` 而不初始化 `Values`），`Values` 为 `null`，DataGrid 绑定到 `Values[0]` 会抛出 `NullReferenceException`。表格适配器在生成行数据时必须确保 `Values` 始终被初始化。建议在实现规格中明确 `Values` 的非空约束。

### 2. 标准库与生态覆盖

**[通过]** 设计中需要的能力均在 C# 标准库或 Avalonia 框架覆盖范围内：
- `INotifyPropertyChanged`、`INotifyCollectionChanged`、`IReadOnlyList<T>`、`TimeProvider`（.NET 8+）、`Task`、`EventArgs` 均为标准库类型
- `Dispatcher.UIThread`、`DataGrid`、`DataGridRow`、`TemplatedControl`、`AvaloniaProperty` 等均为 Avalonia 框架核心类型
- `DataGrid` 的 `LoadingRow`/`UnloadingRow` 事件在 Avalonia 11.3.12 和 V12 中均存在，经 Avalonia API 文档确认
- `DataGridCellEditEndingEventArgs` 的属性（`Column`、`Row`、`EditingElement`、`EditAction`、`Cancel`）经 Avalonia API 文档确认存在

**[轻微]** 设计文档关于 `IReadOnlyList<double> Values` 与 Avalonia V12 CompiledBinding 兼容性的描述略有误导。动态生成 DataGrid 列时，绑定无法在 XAML 中静态声明 `x:DataType`，因此无论使用索引器还是属性访问器，均无法在动态列场景中使用 XAML 编译器生成的 CompiledBinding，而需要在代码中使用 `ReflectionBinding` 或 `CompiledBinding.Create(...)` 动态创建。`Values[0]` 相对于自定义结构体索引器的优势主要体现在反射绑定场景下（属性路径解析更稳定、更易被 Avalonia 绑定系统识别），而非 CompiledBinding 兼容性。此描述不准确但不会导致设计不可行，建议在实现阶段的技术验证项中明确动态列绑定使用 `ReflectionBinding` 或代码级 `CompiledBinding` 创建。

### 3. 语言特性可行性

**[通过]** 语言特性与 C# / Avalonia 能力匹配：
- 错误处理采用「就地反馈」策略，通过 `CellEditEnding` 事件拦截 + 手动验证 + 模型写接口回写，在 C# / Avalonia 中完全可行
- 并发设计通过 `Dispatcher.UIThread.Post` 保证 UI 线程亲和性，符合 Avalonia 调度模型
- 资源管理通过 `IAvaloniaChartPresenter.Detach()` 释放图表宿主关联，符合 IDisposable 模式的精神
- 模块结构分层清晰，符合 NuGet 多项目组织方式
- Avalonia 控件通过 `TemplatedControl` 派生、依赖属性注册、ControlTemplate 部件约定，符合 Avalonia 控件组织规范
- MVVM 支持通过数据绑定接口（`INotifyPropertyChanged`）和命令模式（`ICommand` 可用于保存按钮）实现

**[轻微]** Avalonia 中存在个别 GitHub issue 讨论 `DataGridCellEditEndingEventArgs.Cancel = true` 在某些场景下不能完全阻止事件传播到 DataGrid 内部状态（issue #12769）。但设计文档采用的是「取消默认提交 + 后续手动调用 `Refresh()` 并重置 `ItemsSource`」的组合策略，即使 `Cancel` 不完全阻止内部行为，后续的 `Refresh()` 和 `ItemsSource` 替换仍会恢复正确的 UI 状态。此策略在实现层面可行，但实现阶段建议对 Avalonia V12 的具体行为进行验证。

### 4. 设计一致性

**[通过]** 设计一致性良好：
- 各抽象职责描述清晰无歧义，模块职责表明确
- 协作关系形成闭环：数据变更 → 模型 `PropertyChanged` → Editor 调用 `Refresh()`/`ReplaceRow()` → UI 刷新 → 用户编辑 → `CellEditEnding` 拦截 → 模型写接口 → 数据变更
- 行为契约完整：变量切换流程、单元格编辑回写、键盘导航、z 切片激活联动、虚拟化高亮同步均有详细定义
- 模块间依赖方向合理：数据契约层（无 outward 依赖）→ 适配层/图表抽象层 → 激活追踪层 → Views 层，无循环依赖

**[轻微]** `CubeEditor` 中 z 列点击后的「选中第一个数据单元格」行为由 `CubeEditor` 在 `ActiveSliceChanged` 事件处理器中执行。由于 `ZSliceActivationTracker` 的防抖机制（100–200ms），`ActiveSliceChanged` 会在用户停止操作后才触发，这意味着「选中第一个数据单元格」也会延迟执行。用户点击 z 列后可能需要等待防抖结束才能看到单元格选中变化，这与即时的视觉反馈期望存在轻微偏差。建议明确：若 z 列点击需要即时选中反馈，可在调用 `OnZColumnClicked` 的同时由 `CubeEditor` 立即执行选中逻辑（不等待 `ActiveSliceChanged`），而 `ActiveSliceChanged` 仅负责 3D 图表刷新和 z 切片高亮更新。

### 5. 设计质量

**[通过]** 设计质量良好：
- 职责划分遵循单一职责原则：`ZSliceActivationTracker` 专注 z 切片状态管理、`ITableDataSource` 专注数据投影、`IChartPresenter` 专注图表渲染
- 抽象层次恰当：架构级设计未过度侵入实现细节（如未规定具体字段类型、XAML 模板结构），同时提供了足够的行为契约指导后续实现
- 便于详细设计和实现：各接口的成员签名、事件参数类型、状态流转流程均已定义
- 便于单元测试：`ZSliceActivationTracker` 不依赖 Avalonia 视觉类型（可注入 `FakeTimeProvider`）、`IChartPresenter` 基接口无 UI 依赖、`ITableDataSource` 可 mock

**[轻微]** `ITableDataSource<TRow>.ReplaceRow` 方法约定「由实现方内部执行集合替换并触发 `INotifyCollectionChanged`」。但如果实现方选择使用 `List<TRow>` 等不原生支持变更通知的集合作为 `Rows` 的底层存储，实现方需要自行触发 `CollectionChanged` 事件（通过实现自定义的 `INotifyCollectionChanged` 包装器或切换到 `ObservableCollection`）。设计文档未明确约束底层集合类型，但 `CollectionChangedNotifier` 属性的存在已暗示实现方需要提供通知能力。建议在实现规格中明确：支持增量刷新的实现方应使用 `ObservableCollection<TRow>` 或等效的变更通知集合作为 `Rows` 的底层存储。

## 修改要求（REJECTED 时存在）

无。本审查未识别严重或一般级别问题。
