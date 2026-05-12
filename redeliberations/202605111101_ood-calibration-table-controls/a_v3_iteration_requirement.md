根据以下审查结果，迭代上一轮的产出，形成新版的文件，从而更好地满足用户需求。

## 当前审查结果

### 严重问题

**问题1：`readonly struct` 与 DataGrid 标准编辑机制不兼容的根因分析错误**
- 设计文档将 `readonly struct` 行数据无法使用 DataGrid 标准编辑提交机制归因于「不可变性」，实际根本原因是**值类型（`struct`）本身的 boxing 副本问题**——即使改为可变 `struct`，DataGrid 在 `DataContext` 中拿到的是已 boxing 的副本，标准编辑提交对该副本的修改不会反映回 `Rows` 集合中的原始实例。只有降级为 `class`（引用类型）才能让 DataGrid 的标准编辑机制直接修改原集合中的对象。
- **改进要求**：修正根因说明，明确指出值类型的 boxing 副本问题是根本原因，`readonly` 只是在此基础上额外禁止了 setter 调用；明确说明无论 `readonly struct` 还是可变 `struct` 都必须采用手动事件拦截，只有降级为 `class` 才能利用 DataGrid 内置编辑提交；补充 `class` 方案的 GC 影响和坐标索引内联存储损失，使权衡分析完整。

**问题2：单元格编辑后的全量刷新策略存在选中状态丢失和性能风险，但未给出应对方案**
- 设计文档规定编辑完成后采用 `e.Cancel = true` + `ITableDataSource<TRow>.Refresh()` + 重新设置 `DataGrid.ItemsSource` 的刷新路径，存在两个副作用：(1) 替换 `ItemsSource` 后 `CurrentItem`/`CurrentCell`/`SelectedItem` 被重置，焦点跳回表格顶部，破坏编辑连续性；(2) 对于 `CubeEditor`，每次编辑都执行 O(z×y) 的全量重建，大数据量时造成 UI 卡顿。
- **改进要求**（按优先级）：
  - **必选（高优先级）**：在编辑流程中补充「选中状态保持」步骤——编辑前保存 `CurrentCell` 坐标（行索引 + 列索引），`Refresh()` 后根据新 `ItemsSource` 重新定位并恢复选中。
  - **推荐（中优先级）**：评估增量刷新可行性——若将 `ITableDataSource<TRow>.Rows` 改为 `ObservableCollection<TRow>`，可通过 `Rows[index] = newRow` 实现单元素替换，DataGrid 仅刷新对应行而无需全量重建。
  - **兜底（低优先级）**：若坚持全量刷新或数据量极端大，在「并发设计」或「设计决策」中明确性能兜底方案（如异步刷新、延迟刷新、编辑后批量刷新等）。

### 中等问题

**问题3：数据模型元素级变更的 `PropertyChanged` 通知约定缺失**
- `ICalibrationData` 继承 `INotifyPropertyChanged`，但设计文档未定义元素级写方法（`SetXValue`、`SetValue` 等）触发 `PropertyChanged` 时使用的属性名，也未说明如何区分「仅数据值变更」与「元信息变更」。
- **改进要求**：在 `ICalibrationData` 或其子接口的文档中明确约定：元素级写方法触发 `PropertyChanged` 事件，属性名建议为 `string.Empty`（表示所有属性均可能变更），或增加一个专门的 `ValuesChanged` 事件；说明控件层收到通知后的刷新策略。

**问题4：`ISurfaceChartPresenter` 两个加载方法的参数风格不一致**
- `LoadMapData(IMapData data)` 接收数据模型接口，`LoadSliceData(double[,] sliceData, IReadOnlyList<double> xValues, IReadOnlyList<double> yValues)` 接收原始数组，参数风格不统一且未给出理由。
- **改进要求**：统一参数风格。推荐将 `LoadMapData` 也拆分为接收原始数组的形式，由调用方负责从数据模型提取数据；若保留 `IMapData` 参数以简化调用，应在「设计决策 5」中补充说明不对称性的原因。

**问题5：`ZSliceActivationTracker` 核心事件签名未定义**
- `ZSliceActivationTracker` 被定义为提供激活切片变更通知的核心类，但未给出公开事件的签名（事件名称、参数类型、触发语义）。
- **改进要求**：补充 `ActiveSliceChanged` 事件及 `ActiveZSliceChangedEventArgs` 定义；明确事件在防抖结束后触发，还是在每次潜在变更时都触发。

**问题6：`ICurveTablePresenter` 缺少程序化选中设置能力**
- 接口定义了 `SelectedCellChanged` 事件（输出方向）和 `CellEditCompleted` 事件，但没有输入方向的选中控制方法，导致 `CurveEditor` 无法在变量切换后重置选中，也无法在图表联动时反向选中表格单元格。
- **改进要求**：补充选中设置方法，如 `void SelectCell(int columnIndex, CurveTableRow row)` 和 `void ClearSelection()`，其中 `CurveTableRow` 为枚举（`X` / `Z`）。

**问题7：脏标记跟踪的 `IAsyncSaveCapable` 接口未定义但被引用**
- 设计文档在「变更反馈机制」中引入了 `IAsyncSaveCapable` 标记接口，但该接口在文档中没有任何定义，也未出现在「模块职责」或「核心抽象」中。
- **改进要求**：在「核心抽象」中补充 `IAsyncSaveCapable` 的完整定义（纯标记接口或含 `HasUnsavedChanges`/`SaveAsync`），并说明控件层的检测和消费逻辑。

**问题8：z/y 行头文字在激活状态下的加粗/颜色变化需求未响应**
- 需求文档 v3 第 169 行明确要求「z 列和 y 列的行头文字在激活状态下可采用加粗或颜色变化作为辅助标识」，但 v4 设计文档在详细定义三种高亮后完全未涉及此行头文字样式变化。
- **改进要求**：在「高亮叠加策略」中补充第四层视觉反馈：当前行被选中时 y 列文字加粗或变色；当前 z 切片激活时 z 列文字加粗或变色。明确通过 `DataGridCell` 样式选择器或 `LoadingRow` 事件中对行头单元格的样式类附加来实现。

### 轻微问题

**问题9：`CubeEditor` 初始默认 z 切片语义未定义**
- 设计文档说明变量切换时会「重置 `ZSliceActivationTracker` 为默认 z 切片」，但未定义「默认」的具体含义。
- **改进要求**：明确默认策略为 `ZIndex = 0`（第一个 z 切片），并说明后续可在实现阶段根据用户记忆偏好调整。

**问题10：`ITableDataSource.GetXIndexFromColumn` 的列索引语义不明确**
- `GetXIndexFromColumn(int columnIndex)` 的 `columnIndex` 参数是否包含行头列的偏移没有说明。
- **改进要求**：在接口文档中明确 `columnIndex` 为 DataGrid 的原始列索引（包含所有行头列），由 `ITableDataSource` 实现方内部处理偏移映射。

**问题11：键盘跨 z 切片导航时表格与 3D 视图的视觉不一致未处理**
- 键盘跨 z 切片导航时 DataGrid 当前行立即切换，但 3D 曲面图因防抖延迟刷新，导致约 100–200ms 的视觉不一致状态。
- **改进要求**：明确该不一致在标定工具场景下是否可接受；若需缓解，补充防抖期间 3D 视图区域的轻量过渡效果（如半透明遮罩或先清空再等待加载）。

**问题12：需求"推断与待确认事项"中 3D 曲面图数据源切换动画未给出决策结论**
- 需求 v3 第 217 行将「`CubeEditor` 切换激活 z 切片时 3D 曲面图是直接刷新还是有过渡动画」列为由设计阶段确定的事项，但设计文档「设计决策 5」完全未涉及此问题。
- **改进要求**：在「设计决策 5」或「关键行为契约」→「z 切片激活与 3D 视图联动」中补充明确决策：采用直接瞬时刷新（与表格即时响应保持一致），过渡动画作为可选增强在阶段 2 评估；或明确采用轻量淡入淡出过渡以提升视觉连贯性。

## 历史迭代回顾

- **已解决的问题（第 1 轮）**：行级高亮驱动逻辑与 z 切片级高亮混用（v4 已分离驱动源）、z 列首行显示在虚拟化下丢失 z 值标识（v4 改为每行重复显示 + 视觉分组线）、接口签名缺失（v4 补充了 ICurveData/IMapData/ICubeData/IChartPresenter/ICurveTablePresenter/ITableDataSource 的最小成员）、精度展示与列宽行高调整（v4 补充相关策略）、变更反馈机制（v4 明确即时同步模式）、下拉框显示格式与信息栏格式（v4 补充 GroupName/DisplayName 和 FormatInfoText）、空状态处理（v4 补充完整状态机）、图表交互深度（v4 设计决策 5 分阶段策略）、readonly struct 编辑验证路径（v4 明确手动事件拦截）、ITableDataSource 泛型关系（v4 明确 `ITableDataSource<TRow> where TRow : struct`）。
- **持续存在的问题（多轮反复）**：无新增持续问题。本轮所有问题均为第 2 轮审查新发现，尚未在任何历史版本中解决。
- **新发现的问题（本轮）**：上述 12 个问题全部为第 2 轮组件 B 审查新识别的问题，涵盖根因分析准确性、全量刷新副作用、接口契约完整性、需求响应遗漏、语义明确性等维度。

## 上一轮产出路径

C:\Users\jiangwei\Documents\C#\INCA_MAP_Control\redeliberations\202605111101_ood-calibration-table-controls\a_v2_design_v4.md

## 用户需求

C:\Users\jiangwei\Documents\C#\INCA_MAP_Control\redeliberations\202605111101_ood-calibration-table-controls\requirement.md
