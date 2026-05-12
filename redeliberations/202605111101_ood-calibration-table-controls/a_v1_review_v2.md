# OOD 设计方案审查报告（v2）

## 审查结果

APPROVED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 类型形态选择与 C# 类型系统完全匹配：
- `ICalibrationData` 作为公共数据契约接口，并继承 `INotifyPropertyChanged`，利用 C# 接口多继承能力确保后端模型具备变更通知能力，设计合理。
- 维度特化接口 `ICurveData`/`IMapData`/`ICubeData` 继承 `ICalibrationData`，避免后端模型因单继承限制而无法复用领域基类，符合 C# 单继承约束。
- `CalibrationEditorBase<T>` 采用泛型抽象类（`T : ICalibrationData`）继承自 Avalonia `TemplatedControl`，可封装公共依赖属性注册和模板部件约定，接口无法承载此类实现代码，选择恰当。
- `CurveEditor`/`MapEditor`/`CubeEditor` 使用 `sealed` 终态类，明确禁止下游派生，符合控件作为领域终态产品的定位。
- `MapRowData`/`CubeRowData` 使用 `readonly struct` 值类型承载行坐标索引，避免堆分配和接口装箱，与 DataGrid 行虚拟化场景的性能诉求一致。
- `ZSliceActivationTracker` 为独立类，纯 .NET 事件输出，不依赖 Avalonia 视觉类型，类型形态合理。

**[轻微]** `readonly struct` 作为 `DataGrid` 的 `ItemsSource` 行数据时，其属性的不可变性（无 setter）与 `DataGrid` 默认编辑模板的双向绑定提交机制存在语义张力。虽然设计方案已通过 `CellEditEnding` 事件拦截手动回写原始数据模型来规避，但实现阶段需确保 `DataGrid` 的编辑模板和绑定模式配置正确（如使用 OneWay 绑定或自定义编辑模板），否则可能触发绑定异常。建议在详细设计阶段明确 `DataGrid` 列的 `Binding.Mode` 和 `CellTemplate`/`CellEditingTemplate` 配置策略。

### 2. 标准库与生态覆盖

**[通过]** 设计中依赖的能力均在 C# 标准库和 Avalonia 生态覆盖范围内：
- Avalonia `DataGrid`（`Avalonia.Controls.DataGrid`）支持 `LoadingRow`/`UnloadingRow`/`RowEditEnding`/`RowEditEnded`/`PreparingCellForEdit`/`SelectionChanged` 等事件，设计方案中的虚拟化行视觉元素联动策略在 Avalonia 中可实现。
- `IDataErrorInfo`/`INotifyDataErrorInfo` 为 .NET 标准数据验证接口，Avalonia `DataGrid` 内置支持数据验证模板（如错误边框、工具提示），满足就地反馈需求。
- `Dispatcher.UIThread` 为 Avalonia 标准线程调度 API，满足 UI 线程亲和性要求。
- `TemplatedControl`/`AvaloniaProperty.Register` 为 Avalonia 自定义控件的标准基础设施。
- Avalonia V12 标准控件集中不包含 3D 曲面图控件，设计方案通过 `IChartPresenter` 接口隔离具体图表库，是最低耦合的合理抽象。社区可选方案（OxyPlot、LiveCharts、ScottPlot、自定义 Skia 渲染）在 Avalonia V12 下的兼容性可在实现阶段评估。

**[轻微]** Avalonia V12 当前处于预览/RC 阶段，部分 API（尤其是主题系统、某些控件行为细节）可能存在 breaking changes。设计方案中使用的基础 API（`TemplatedControl`、`DataGrid` 核心事件、`Dispatcher`）属于 Avalonia 核心基础设施，变更风险较低，但建议在实现启动前对照 Avalonia 12 正式发布版本的 breaking changes 清单做一次快速复核。

### 3. 语言特性可行性

**[通过]** 语言特性使用与 C# / Avalonia 能力匹配：
- 错误处理策略采用「就地反馈 + 空状态降级 + 渲染降级」三级策略，与 Avalonia 的数据验证模板和控件空状态展示能力匹配，无需异常驱动控制流。
- 并发设计明确 UI 线程亲和性，后端数据变更通过 `Dispatcher.UIThread.Invoke` 调度到 UI 线程，符合 Avalonia 的调度模型。
- `ZSliceActivationTracker` 的防抖策略（100–200ms）可通过标准 .NET `Timer` 或 `System.Reactive` 实现，无语言层面障碍。
- 资源管理方面无显式非托管资源需求，无需额外 `IDisposable` 设计。
- 模块依赖方向清晰：数据契约层无 outward 依赖，适配层仅依赖数据契约，Views 层向下依赖所有下层，无反向依赖，符合 NuGet 项目组织方式。
- Avalonia 控件组织方式符合规范：抽象基类承载公共依赖属性 + 模板约定，子类密封实现，支持 MVVM 数据绑定。

### 4. 设计一致性

**[通过]** 设计一致性强，协作关系形成闭环：
- **变量切换闭环**：`ComboBox` 选中变更 → `CalibrationEditorBase<T>` 更新 `SelectedVariable` → `OnSelectedVariableChanged` 虚方法 → 子类重建表格数据源 / 通知图表加载新数据 → `CubeEditor` 重置 `ZSliceActivationTracker` 激活状态。各环节职责清晰，无缺失。
- **编辑回写闭环**：单元格进入编辑 → 用户输入 → 编辑完成事件拦截 → Editor 从行数据对象（`MapRowData`/`CubeRowData`）提取坐标索引 → 调用原始数据模型写接口 → 模型触发 `PropertyChanged` → 绑定系统刷新 UI。`CubeEditor` 额外结合 x 列索引完成三维张量定位，逻辑完整。
- **z 切片激活闭环**：选中/导航变更 → `CubeEditor` 传递 `CubeRowData` 给 `ZSliceActivationTracker` → Tracker 判断 z 值变化 → 纯 .NET 事件通知 → `CubeEditor` 通过 `Dispatcher.UIThread.Post` 调度 UI 更新 → 图表刷新 + z 切片级高亮更新。Tracker 与视觉树解耦，状态同步机制明确。
- **虚拟化联动闭环**：激活 `ZIndex` 变化时遍历已加载 `DataGridRow` 更新样式类 + `LoadingRow` 事件为新进入可视区域的行设置高亮 + `UnloadingRow` 事件清理回收行状态。三层机制完整覆盖 DataGrid 虚拟化场景。
- 模块间无循环依赖：依赖方向始终从上（Views）向下（数据契约），底层模块无反向依赖。

**[轻微]** `CubeEditor` 的「行级高亮」（当前选中单元格所在整行）与「z 切片级高亮」（当前激活切片的所有行）在语义上存在强关联：根据需求，用户选中任意单元格即触发该单元格所属 z 切片为激活切片，因此行级高亮始终落在 z 切片级高亮的范围内。两种高亮在同一行上的叠加效果（轻微背景色 + 分组背景色/左侧边框）需在实现阶段验证视觉区分度，确保不冲突。此问题属于视觉设计细节，不影响架构可行性。

### 5. 设计质量

**[通过]** 设计质量良好：
- **单一职责**：各抽象职责划分清晰。`ICalibrationData` 及维度特化接口专注数据契约；`CalibrationEditorBase<T>` 专注公共 UI 结构和依赖属性；`ITableDataSource`/`ICurveTablePresenter` 分别负责纵向/横向表格投影；`IChartPresenter` 隔离图表渲染；`ZSliceActivationTracker` 独立管理 z 切片激活状态。无职责混杂。
- **抽象层次恰当**：未过早引入折叠/展开、分页、undo/redo 等超出当前需求范围的抽象；同时已覆盖必要的性能保障机制（行虚拟化、`readonly struct`、防抖策略）。
- **可测试性**：`ZSliceActivationTracker` 纯逻辑类，不依赖 Avalonia 视觉类型，可在无头环境中独立单元测试；`ICurveTablePresenter`、`IChartPresenter`、`ITableDataSource` 均为接口，实现方可被 mock；`MapRowData`/`CubeRowData` 为值类型，测试断言无引用相等性陷阱。
- **便于后续实现**：关键行为契约（变量切换、编辑回写、z 切片激活、高亮叠加）的描述完整到足以指导详细设计和编码。

**[轻微]** `CubeEditor` 中 z 切片级高亮状态更新涉及遍历 `DataGridRow` 容器并动态附加/移除样式类，这部分逻辑与 Avalonia 视觉树强耦合，难以在标准单元测试中覆盖。建议在实现阶段为这部分 UI 协调逻辑保留一个薄的可测试包装（如将样式类附加逻辑抽取为仅操作 `DataGridRow` 的静态方法或独立视觉状态协调器），以便通过 Avalonia 头less 测试或 UI 测试覆盖。

## 修改要求（REJECTED 时存在）

本次审查无严重或一般问题，无需修改要求。
