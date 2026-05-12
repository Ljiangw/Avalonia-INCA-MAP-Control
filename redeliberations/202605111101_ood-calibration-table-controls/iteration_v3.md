# 再审议判定报告（v3）

## 判定结果

RETRY

## 判定理由

组件B诊断报告共识别出12项问题，其中包含3项严重问题和9项一般问题。质询报告结论为LOCATED，表明审查结论经质询后被确认为有效。

依据判定标准，审查报告包含严重或一般等级的问题时，应判定为RETRY（重新运行组件A）。具体问题如下：

- 3项严重问题涉及事实错误（`UnloadingRow`事件存在性）、接口自洽性（`IReadOnlyList<T>`与增量刷新机制矛盾）、以及接口设计与需求矛盾（`bool HasUnsavedChanges`无法支撑未保存数量显示），这些问题会直接阻塞编码实现或导致需求无法满足。
- 9项一般问题涉及边界条件、事件契约、性能开销、异常场景和细节契约的完备性，修复后可显著提升设计的落地可靠性。

由于实际轮次（1）远小于最大轮次（12），且质询已确认问题存在，具备充分的重新运行空间以修复上述问题。

## 需要解决的问题

- **问题描述**：Avalonia DataGrid `UnloadingRow` 事件存在，设计文档错误声明其不存在，并基于错误声明构建了虚拟化状态清理策略
- **所在位置**：「z 切片级高亮状态与 DataGrid 虚拟化联动（CubeEditor）」小节第 412 行
- **严重程度**：严重
- **改进建议**：修正事实声明，将虚拟化状态清理策略改为标准的 `LoadingRow`/`UnloadingRow` 配对模式

- **问题描述**：`ITableDataSource<TRow>.Rows` 返回 `IReadOnlyList<TRow>` 与增量刷新机制矛盾，文档描述的 `Rows[index] = newRow` 增量刷新路径在接口层面被阻断
- **所在位置**：「核心抽象」→「`ITableDataSource<TRow>`（接口）」；「关键行为契约」→「单元格编辑与数据回写」→ 增量刷新策略
- **严重程度**：严重
- **改进建议**：保留 `IReadOnlyList<TRow>` 作为只读视图，同时新增 `void ReplaceRow(int index, TRow newRow)` 方法作为增量刷新的显式契约入口

- **问题描述**：`IAsyncSaveCapable` 的 `bool HasUnsavedChanges` 无法支撑"未保存数量"显示需求，接口设计与上层需求之间存在不可调和的矛盾
- **所在位置**：「核心抽象」→「`IAsyncSaveCapable`（接口）」；「`CalibrationEditorBase<T>`（抽象类）」脏标记状态栏提示职责
- **严重程度**：严重
- **改进建议**：在 `IAsyncSaveCapable` 中补充 `int UnsavedChangeCount { get; }`，或降级 `CalibrationEditorBase<T>` 的职责描述为布尔状态提示

- **问题描述**：`ZSliceActivationTracker.OnZColumnClicked` 的"选中第一个数据单元格"通知机制缺失，`ActiveSliceChanged` 事件无法区分触发源
- **所在位置**：「核心抽象」→「`ZSliceActivationTracker`（类）」职责描述；「z 切片激活与 3D 视图联动（CubeEditor）」
- **严重程度**：一般
- **改进建议**：推荐将"选中第一个数据单元格"行为从 Tracker 职责中移除，改为 `CubeEditor` 自行处理选中逻辑

- **问题描述**：`SupportsIncrementalRefresh` 与 `INotifyCollectionChanged` 的关联机制未定义，运行时只能通过类型强制转换检测
- **所在位置**：「核心抽象」→「`ITableDataSource<TRow>`（接口）」
- **严重程度**：一般
- **改进建议**：在接口中新增 `INotifyCollectionChanged? CollectionChangedNotifier { get; }` 属性

- **问题描述**：`ICalibrationData` 轴属性对所有维度可见，一维数据模型被迫暴露无关属性，违反接口隔离原则
- **所在位置**：「核心抽象」→「`ICalibrationData`（接口）」
- **严重程度**：一般
- **改进建议**：将轴信息属性从 `ICalibrationData` 中移除，下沉到各维度子接口

- **问题描述**：`ISurfaceChartPresenter` 的 `double[,]` 参数与 `IMapData` 逐点访问接口之间的转换开销未考虑
- **所在位置**：「核心抽象」→「`ILineChartPresenter` / `ISurfaceChartPresenter`」
- **严重程度**：一般
- **改进建议**：在 `IMapData` 中增加 `double[,] GetValueMatrix()` 方法，或在 `ISurfaceChartPresenter` 中增加逐点数据填充的替代方法

- **问题描述**：动态列绑定 `readonly struct` 索引器与 Avalonia V12 CompiledBinding 的兼容性风险未经验证
- **所在位置**：「核心抽象」→「`MapRowData` / `CubeRowData`」；「设计决策 8」
- **严重程度**：一般
- **改进建议**：在设计决策中补充 CompiledBinding + 动态索引器绑定的技术验证项，或改为通过 `IReadOnlyList<double> Values` 暴露数据

- **问题描述**：空状态未覆盖数据内容为零维度的场景，可能导致运行时异常或控件白屏
- **所在位置**：「错误处理策略」→「数据绑定不匹配错误」；「核心抽象」→「`ZSliceActivationTracker`（类）」
- **严重程度**：一般
- **改进建议**：在变量切换流程中增加对数据内容维度的检查，在 `ZSliceActivationTracker` 初始化逻辑中增加 `ZAxisLength == 0` 的防护

- **问题描述**：多个事件参数类型未定义（`CurveCellSelectedEventArgs`、`CurveCellEditCompletedEventArgs`、`ChartRenderFailedEventArgs` 等）
- **所在位置**：「核心抽象」→「`ICurveTablePresenter`（接口）」；「核心抽象」→「`IChartPresenter`（接口）」
- **严重程度**：一般
- **改进建议**：在「核心抽象」中补充相关事件参数类型的完整结构定义

- **问题描述**：键盘导航行为未在设计中充分定义，方向键、Tab 切换、跨 z 切片导航等行为缺失
- **所在位置**：需求 `requirement.md` 第 160 行、第 221 行；设计文档「关键行为契约」→「单元格编辑与数据回写」
- **严重程度**：一般
- **改进建议**：在「关键行为契约」中补充键盘导航的完整行为定义及与 `ZSliceActivationTracker` 防抖策略的交互细节

- **问题描述**：`ZSliceActivationTracker.OnSelectionChanged(null)` 行为未定义，Tracker 对无选中项时的处理语义缺失
- **所在位置**：「核心抽象」→「`ZSliceActivationTracker`（类）」
- **严重程度**：一般
- **改进建议**：明确 `OnSelectionChanged(null)` 的语义为"保持当前激活切片不变"，若需要重置行为应通过独立方法显式触发
