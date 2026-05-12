# 再审议判定报告（v2）

## 判定结果

RETRY

## 判定理由

组件B经2轮审议后，质询报告结论为 **LOCATED**，即审查报告中的问题均经核实成立，审查质量可信。终止原因为审查意见被确认，非循环耗尽（实际轮次 2 < 最大轮次 12）。

诊断报告共识别出 **2 个严重问题、6 个中等问题、4 个轻微问题**。根据判定标准，审查报告包含严重或一般等级的问题时应判定为 RETRY。具体问题分布如下：

- **严重问题**：`readonly struct` 与 DataGrid 编辑机制不兼容的根因分析不准确（将 struct boxing 问题误述为 readonly 问题）；单元格编辑全量刷新策略存在选中状态丢失和性能风险但未给出应对方案。
- **中等问题**：元素级变更的 `PropertyChanged` 通知约定缺失；`ISurfaceChartPresenter` 加载方法参数风格不一致；`ZSliceActivationTracker` 核心事件签名未定义；`ICurveTablePresenter` 缺少程序化选中设置能力；`IAsyncSaveCapable` 接口未定义但被引用；z/y 行头文字激活状态样式需求未响应。
- **轻微问题**：`CubeEditor` 默认 z 切片语义未定义；`GetXIndexFromColumn` 列索引语义不明确；键盘跨 z 切片导航时表格与 3D 视图视觉不一致未处理；3D 曲面图 z 切片切换动画未给出决策结论。

以上问题若不在下一轮迭代中修正，将导致实现者产生困惑或做出错误的技术决策，因此需要重新运行组件A进行修订。

## 需要解决的问题

- **问题描述**：`readonly struct` 与 DataGrid 标准编辑机制不兼容的根因分析不准确，将 struct 值类型的 boxing 副本问题误述为 readonly 的不可变性问题，可能误导实现者认为改为可变 struct 即可启用标准编辑。
- **所在位置**：「设计决策 9」→「`readonly struct` vs `class` 的权衡分析」
- **严重程度**：严重
- **改进建议**：修正根因说明，明确指出值类型的 boxing 副本问题是根本原因，`readonly` 只是额外禁止 setter 调用；明确说明只有降级为 `class` 才能利用 DataGrid 内置编辑提交；补充 `class` 方案的 GC 影响和坐标索引内联存储损失。

- **问题描述**：单元格编辑后采用全量刷新策略（`e.Cancel = true` + `Refresh()` + 重新设置 `ItemsSource`），但未处理替换 `ItemsSource` 后的选中状态丢失问题，也未评估全量刷新的性能影响。
- **所在位置**：「关键行为契约」→「单元格编辑与数据回写」→「MapEditor / CubeEditor（DataGrid 纵向行模型）」
- **严重程度**：严重
- **改进建议**：在编辑流程中补充「选中状态保持」步骤（编辑前保存 `CurrentCell` 坐标，刷新后恢复）；评估增量刷新可行性（如 `ObservableCollection<TRow>` 单元素替换）；大数据量场景下补充异步/延迟刷新兜底方案。

- **问题描述**：`ICalibrationData` 的元素级写方法（`SetXValue`、`SetValue` 等）触发 `PropertyChanged` 的约定缺失，包括属性名使用什么、如何区分数据值变更与元信息变更。
- **所在位置**：「核心抽象」→「`ICurveData` / `IMapData` / `ICubeData`」；「关键行为契约」→「单元格编辑与数据回写」
- **严重程度**：一般
- **改进建议**：明确约定元素级写方法触发 `PropertyChanged` 时属性名建议为 `string.Empty`，或增加独立的 `ValuesChanged` 事件；说明控件层收到通知后的刷新策略。

- **问题描述**：`ISurfaceChartPresenter` 两个加载方法参数风格不一致，`LoadMapData` 接收 `IMapData`，`LoadSliceData` 接收原始数组，不对称性未说明理由。
- **所在位置**：「核心抽象」→「`ILineChartPresenter` / `ISurfaceChartPresenter`」
- **严重程度**：一般
- **改进建议**：统一参数风格（均接收原始数组），或在「设计决策 5」中补充说明不对称性的原因。

- **问题描述**：`ZSliceActivationTracker` 被定义为提供激活切片变更通知的核心类，但未定义公开事件签名（事件名称、参数类型、触发语义）。
- **所在位置**：「核心抽象」→「`ZSliceActivationTracker`」
- **严重程度**：一般
- **改进建议**：补充 `ActiveSliceChanged` 事件及 `ActiveZSliceChangedEventArgs` 定义；明确事件在防抖后触发还是每次潜在变更都触发。

- **问题描述**：`ICurveTablePresenter` 缺少输入方向的程序化选中设置方法，无法在变量切换或图表联动时反向控制表格选中状态。
- **所在位置**：「核心抽象」→「`ICurveTablePresenter`」
- **严重程度**：一般
- **改进建议**：补充 `SelectCell(int columnIndex, CurveTableRow row)` 和 `ClearSelection()` 方法，其中 `CurveTableRow` 为 `X`/`Z` 枚举。

- **问题描述**：`IAsyncSaveCapable` 标记接口在设计文档中被引用，但未在任何位置定义其命名空间、成员和控件层检测逻辑。
- **所在位置**：「关键行为契约」→「单元格编辑与数据回写」→「变更反馈机制」
- **严重程度**：一般
- **改进建议**：在「核心抽象」中补充 `IAsyncSaveCapable` 的完整定义（纯标记接口或含 `HasUnsavedChanges`/`SaveAsync`），并说明控件层的检测和消费逻辑。

- **问题描述**：需求 v3 第 169 行明确要求 z 列和 y 列行头文字在激活状态下加粗或颜色变化，但 v4 设计文档完全未涉及此需求。
- **所在位置**：需求 `requirement.md` 第 169 行；设计文档「高亮叠加策略」
- **严重程度**：一般
- **改进建议**：在「高亮叠加策略」中补充 z 列/y 列行头文字的样式变化规则，明确通过样式选择器或 `LoadingRow` 事件实现。
