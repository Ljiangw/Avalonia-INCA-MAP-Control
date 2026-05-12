# OOD 设计方案审查报告 v1

## 审查对象

- **用户需求**：`C:\Users\jiangwei\Documents\C#\INCA_MAP_Control\redeliberations\202605111101_ood-calibration-table-controls\requirement.md`
- **待审查产出**：`C:\Users\jiangwei\Documents\C#\INCA_MAP_Control\redeliberations\202605111101_ood-calibration-table-controls\a_v1_design_v2.md`
- **审查轮次**：第 1 次
- **审查视角**：侧重需求响应充分度、整体深度和完整性、可落地性（内部审议已覆盖的技术可行性维度不再重复验证）

---

## 审查发现

### 问题 1：行级高亮驱动逻辑与需求定义矛盾

- **问题描述**：需求定义「行级高亮」为**当前选中单元格所在的整行**附加轻微背景色，用于辅助跨列浏览定位。但设计文档在「高亮叠加策略」中将「行级高亮」与「z 切片级高亮」一并描述为"通过 `ZSliceActivationTracker` 输出的状态值驱动附加样式类"。`ZSliceActivationTracker` 输出的是**激活 z 切片的 ZIndex**，若用它来驱动行级高亮，则会导致当前激活 z 切片内的**所有行**都获得行级高亮，与需求中"当前选中单元格所在的整行"这一定义严重矛盾。行级高亮应跟随 DataGrid 的当前单元格/选中项，而非 z 切片激活状态。
- **所在位置**：「关键行为契约」→「高亮叠加策略」小节
- **严重程度**：严重
- **改进建议**：明确行级高亮由 DataGrid 的 `CurrentItem` / `SelectedItem` 或当前活动单元格所在行驱动，可通过 DataGrid 内置的选中行样式或基于 `CurrentCell` 状态的样式选择器实现；`ZSliceActivationTracker` 仅负责 z 切片级高亮。在「高亮叠加策略」中补充三者的独立驱动来源说明。

---

### 问题 2：z 列首行显示方案在滚动时丢失 z 值标识

- **问题描述**：设计文档决定用"z 值首行显示 + 视觉分组线"替代原生跨行合并单元格。具体方案是：通过 `CubeRowData.IsFirstRowInZSlice` 控制，只在每组的**第一个可见行**渲染 z 值文本，其余行留空。该方案在虚拟化滚动下存在致命缺陷：当某 z 切片的首行滚出可视区域后，该切片剩余的所有可见行在 z 列中将**没有任何 z 值标识**。用户滚动到切片中间位置时，无法从视觉上判断当前行属于哪个 z 切片，严重破坏三维数据的层次可读性。需求中合并单元格的原始意图正是为了确保 z 标识在切片的全部行范围内始终可见，替代方案未能等效满足这一需求。
- **所在位置**：「设计决策 2」小节
- **严重程度**：严重
- **改进建议**：补充滚动场景下的 z 值可见性保障策略。可选方向：(1) 在 z 列使用粘性行头（sticky header）或悬浮标签，使 z 值在首行滚出后仍锚定显示；(2) 允许 z 值在切片的每一行重复显示（牺牲部分视觉简洁性换取信息完整性）；(3) 在表格顶部或侧边增加独立的 z 切片导航面板作为辅助标识。无论采用哪种方向，都需要在设计文档中明确，否则实现阶段将出现需求缺口。

---

### 问题 3：维度特化数据接口缺少具体方法签名，无法指导后端实现

- **问题描述**：`ICurveData` / `IMapData` / `ICubeData` 被描述为"维度特化的数据契约"，但设计文档未给出任何属性或方法签名。例如，`ICurveData` 应如何暴露一维 x/z 序列？是 `IReadOnlyList<double> XValues { get; }` 还是索引器？`IMapData` 如何读写二维矩阵的某个单元格？`ICubeData` 如何读写三维张量的某个单元格？缺少这些签名，后端数据模型的实现方不知道该实现什么，控件开发者也不知道该如何调用。作为一份"可直接指导编码实现"的 OOD 设计，接口的核心契约必须明确。
- **所在位置**：「核心抽象」→「`ICurveData` / `IMapData` / `ICubeData`（接口）」小节
- **严重程度**：中等
- **改进建议**：补充三个接口的最小方法签名集。例如 `ICurveData` 至少应声明 `IReadOnlyList<double> XValues { get; }`、`IReadOnlyList<double> ZValues { get; }`、`void SetZValue(int index, double value)`；`IMapData` 至少应声明二维索引读写方法（如 `double GetValue(int xIndex, int yIndex)` / `void SetValue(int xIndex, int yIndex, double value)`）；`ICubeData` 同理需要三维索引读写。同时明确各轴维度的查询属性（如 `XAxisLength`、`YAxisLength`、`ZAxisLength`）。

---

### 问题 4：图表抽象层 `IChartPresenter` 缺少关键契约定义

- **问题描述**：`IChartPresenter` 被定义为"图表渲染的抽象契约"，但设计文档仅描述了其高层职责，未给出任何方法、属性或事件签名。后续编码实现时无法确定：数据通过什么方法传递给图表（如 `LoadData(ICurveData data)` 还是泛型方法）？渲染目标区域如何指定（宿主控件引用还是 `DrawingContext`）？图表何时重绘（由外部调用 `Invalidate()` 还是监听数据变更事件）？3D 曲面图和折线图是否共享同一个接口（如果是，`IChartPresenter` 需要同时承载两种差异巨大的渲染契约；如果不是，应拆分为两个接口）？这些 ambiguity 会阻塞实现。
- **所在位置**：「核心抽象」→「`IChartPresenter`（接口）」小节
- **严重程度**：中等
- **改进建议**：明确 `IChartPresenter` 的最小接口签名。至少需要：(1) 数据加载方法（考虑是否需要拆分为 `ILineChartPresenter` 和 `ISurfaceChartPresenter`，因为 1D 折线图和 3D 曲面图的参数差异很大）；(2) 渲染区域/宿主关联方法（如 `AttachTo(Control host)`）；(3) 显式重绘触发方法（如 `Refresh()`）；(4) 渲染失败时的降级通知机制（如事件或返回值）。

---

### 问题 5：`ICurveTablePresenter` 缺少接口方法签名

- **问题描述**：与 `IChartPresenter` 类似，`ICurveTablePresenter` 的职责描述详尽，但缺少具体的方法、属性和事件签名。设计文档提到它应"提供单元格编辑完成事件"和"提供选中单元格变更通知"，但未定义事件参数类型（如是否携带数据点索引、行标识、解析后的数值）。同样，它如何接收 `ICurveData`、如何构建视觉树、如何响应外部刷新请求，均未定义。
- **所在位置**：「核心抽象」→「`ICurveTablePresenter`（接口）」小节
- **严重程度**：中等
- **改进建议**：补充 `ICurveTablePresenter` 的最小接口定义。至少包括：(1) 数据加载方法（如 `LoadData(ICurveData data)`）；(2) 选中单元格变更事件（如 `EventHandler<CurveCellSelectedEventArgs> SelectedCellChanged`）；(3) 单元格编辑完成事件（如 `EventHandler<CurveCellEditCompletedEventArgs> CellEditCompleted`，参数需包含列索引、行标识[X/Z]、新数值）；(4) 视觉树根元素属性（如 `Control VisualRoot { get; }`），供 `CurveEditor` 将其嵌入模板。

---

### 问题 6：数值精度和小数位数一致性展示需求未响应

- **问题描述**：需求在「隐含但必要的功能」中明确要求"需考虑数值精度和小数位数的一致性展示"。标定数据的数值精度对 ECU 标定工具是核心体验（例如所有单元格统一显示 4 位小数，或根据数据模型动态确定）。设计文档完全没有涉及这一需求点，未说明精度信息由谁提供、由谁消费、如何在表格和图表中统一应用。
- **所在位置**：需求 `requirement.md` 第 178 行；设计文档全文未涉及
- **严重程度**：中等
- **改进建议**：在数据契约层或表格适配层补充精度展示契约。可选方向：(1) `ICalibrationData` 增加 `int DisplayPrecision { get; }` 或 `string FormatString { get; }` 属性，由后端数据模型提供；(2) 表格适配层在生成行数据时应用格式字符串；(3) `ICurveTablePresenter` 和 `IChartPresenter` 的轴标签/数据标签均消费同一格式策略。设计文档需明确精度信息的流转路径。

---

### 问题 7：列宽/行高自适应或手动调整需求未响应

- **问题描述**：需求在「隐含但必要的功能」中明确要求"表格应支持列宽/行高的自适应或手动调整，以保证不同数据量下的可读性"。设计文档完全没有涉及这一需求点。对于 `MapEditor`/`CubeEditor` 的 DataGrid，列宽调整可通过 `DataGridColumn.CanUserResize` 实现，但 x 轴列头可能数量众多且动态生成，需要明确是否允许用户拖拽调整、是否有最小/最大列宽约束。对于 `CurveEditor` 的横向自定义布局，列宽调整完全没有设计支撑。
- **所在位置**：需求 `requirement.md` 第 179 行；设计文档全文未涉及
- **严重程度**：中等
- **改进建议**：在「关键行为契约」或「设计决策」中补充列宽/行高调整策略。对于 DataGrid 方案，明确 `CanUserResizeColumns`、`CanUserResizeRows` 的启用范围、动态生成列的宽度初始化策略（如基于内容自适应或固定初始宽度）。对于 `CurveEditor` 的横向布局，明确 `ICurveTablePresenter` 是否暴露列宽调整能力（如通过拖拽列分隔线），或在接口层面预留 `ColumnWidth` 相关属性。

---

### 问题 8：数据编辑后的变更反馈机制未明确

- **问题描述**：需求在「隐含但必要的功能」中要求"数据编辑后应有适当的变更反馈机制（设计阶段确定具体形式，如即时同步、显式保存等）"。这是标定工具的核心交互决策——编辑是即时写回后端模型（即时同步），还是需要用户点击保存按钮（显式保存）？设计文档在「单元格编辑与数据回写」中描述了"调用原始数据模型的写方法更新对应位置"，这暗示了即时同步模式，但从未明确说明。如果采用即时同步，是否需要脏数据标记？如果采用显式保存，保存按钮在哪里、状态如何反馈？这些关键产品决策的缺失会导致实现阶段的方向分歧。
- **所在位置**：需求 `requirement.md` 第 180 行；设计文档「单元格编辑与数据回写」小节
- **严重程度**：中等
- **改进建议**：在「关键行为契约」中明确编辑数据回写模式：(1) 若采用即时同步，说明编辑完成后立即调用数据模型写接口，并补充是否需要脏标记/未保存状态指示；(2) 若采用显式保存，说明保存触发的时机（按钮、快捷键）和保存失败的处理策略。同时明确需求中"变更反馈"的具体形式（如单元格边框变色提示已修改、底部状态栏提示未保存数量等）。

---

### 问题 9：变量选择下拉框显示格式缺少数据契约支撑

- **问题描述**：需求明确要求下拉框显示项的文本格式为 `<曲线变量组> AllMkn_n_AirMax`（即包含"所属变量组"和"变量名"）。但 `ICalibrationData` 接口仅声明了"变量名称（用于下拉框显示）"这一属性，未定义"变量组"属性或"显示名称格式化"机制。如果只有 `Name` 属性，实现方无法构造出需求要求的显示格式。
- **所在位置**：需求 `requirement.md` 第 41 行；设计文档「核心抽象」→「`ICalibrationData`（接口）」小节
- **严重程度**：轻
- **改进建议**：在 `ICalibrationData` 中补充 `GroupName`（或 `CategoryName`）属性，或增加 `string DisplayName { get; }` 属性让后端直接提供格式化后的显示文本。同时明确 `CalibrationEditorBase<T>` 中下拉框 `DisplayMemberBinding` 的绑定路径。

---

### 问题 10：顶部信息栏单位/轴信息展示格式未定义

- **问题描述**：需求给出了顶部信息栏右侧的示例格式 `[mgpl] x: AllMkn_n_AirMax [rpm]`，并说明需要展示"当前变量的单位信息以及各轴变量名和单位"。设计文档提到 `ICalibrationData` 包含"变量单位"和"各轴变量名及单位"，但未说明 `CalibrationEditorBase<T>` 如何将这些字段组合成最终的展示字符串。例如：当变量有多个轴时，格式模板是什么？括号、冒号、空格等分隔符是否固定？这部分格式化逻辑属于公共基类职责，应在设计中明确。
- **所在位置**：需求 `requirement.md` 第 43 行；设计文档「核心抽象」→「`CalibrationEditorBase<T>`（抽象类）」小节
- **严重程度**：轻
- **改进建议**：在 `CalibrationEditorBase<T>` 的职责中补充信息栏文本的格式化模板。例如定义默认格式为 `[{Unit}] x: {XAxisName} [{XAxisUnit}] y: {YAxisName} [{YAxisUnit}]`，或允许子类/模板覆盖。明确哪些轴信息字段为可选（`CurveEditor` 只有 x 轴，`CubeEditor` 有 x/y/z 三轴）。

---

### 问题 11：空状态与边界条件处理不完整

- **问题描述**：设计文档在「错误处理策略」中提到当 `ICubeData` 中 z 维度长度为 0 时进入"空状态"，但未覆盖以下常见边界：(1) `ItemsSource` 为 `null` 时控件的行为（空状态还是异常？）；(2) `ItemsSource` 为空列表（0 个变量）时 ComboBox 和主内容区的状态；(3) `SelectedVariable` 的初始值（当 `ItemsSource` 非空时是否默认选中第一项？）；(4) 变量切换过程中（旧数据已卸载、新数据尚未加载）的过渡状态。这些边界直接影响控件的鲁棒性。
- **所在位置**：「错误处理策略」→「数据绑定不匹配错误」小节
- **严重程度**：轻
- **改进建议**：在「关键行为契约」中补充「变量切换流程」的完整状态机，明确 `ItemsSource` 为 null/空列表、`SelectedVariable` 为 null 时的空状态展示。为 `CalibrationEditorBase<T>` 定义受保护的虚方法如 `OnEnterEmptyState` / `OnExitEmptyState`，供子类统一覆盖。

---

### 问题 12：图表交互深度需求未在设计中确定

- **问题描述**：需求在「推断与待确认事项」中指出"折线图和 3D 曲面图是否支持缩放、平移、旋转等交互操作，由设计阶段确定"。设计文档在「设计决策 5」中仅说明了通过 `IChartPresenter` 接口抽象图表渲染，但**未确定图表的交互深度**（即是否支持缩放、平移、旋转）。这会导致图表实现方缺乏明确的交互需求输入，也可能影响 `IChartPresenter` 的接口设计（如果支持交互，接口需要暴露交互事件）。
- **所在位置**：需求 `requirement.md` 第 214 行；设计文档「设计决策 5」小节
- **严重程度**：轻
- **改进建议**：在「设计决策 5」或单独的设计决策中明确图表交互深度。例如：阶段 1（MVP）仅支持默认视角静态展示；阶段 2 支持鼠标拖拽旋转（3D）和滚轮缩放。若确定支持交互，`IChartPresenter` 需要补充交互事件契约（如 `ViewChanged` 事件携带相机参数）。

---

### 问题 13：`readonly struct` 编辑验证反馈路径未充分考虑

- **问题描述**：设计文档明确使用 `readonly struct`（`MapRowData` / `CubeRowData`）作为 DataGrid 行数据源，以避免 GC 压力。但 DataGrid 的标准单元格编辑流程会尝试将编辑值自动写回绑定的数据源——对于不可变的 struct，此自动写回必然失败。设计文档已说明通过 `CellEditEnding` 事件拦截实现回写，这是正确的方向，但遗漏了**输入验证错误的就地反馈路径**。需求要求"输入的数据应能正确解析为数值类型"并在错误时通过红色边框或工具提示标识。在 DataGrid 中，`IDataErrorInfo` / `INotifyDataErrorInfo` 验证通常依赖于绑定系统对数据源的写回和验证反馈循环。当数据源是 `readonly struct` 且回写被事件拦截时，DataGrid 的验证模板（`DataErrorValidationRule`）可能无法正常工作，因为绑定系统感知不到"写失败"。这意味着红色边框反馈可能需要完全在事件处理器中手动实现，实现复杂度高于设计文档的暗示。
- **所在位置**：「核心抽象」→「`MapRowData` / `CubeRowData`」小节；「错误处理策略」→「输入验证错误」小节
- **严重程度**：轻
- **改进建议**：在「错误处理策略」中补充 `readonly struct` 场景下的验证反馈具体实现路径。明确：是否放弃 DataGrid 内置验证模板而完全采用手动事件拦截+编辑模板状态控制（如手动设置 `TextBox` 的边框色和 `ToolTip`）？或考虑将行数据类型改为可变的 `class`（牺牲 GC 性能换取绑定兼容性）？至少应在设计决策中记录此权衡。

---

### 问题 14：`ITableDataSource` 与不同行数据类型的泛型关系未明确

- **问题描述**：`MapEditor` 和 `CubeEditor` 分别使用 `MapRowData` 和 `CubeRowData` 两种行数据类型，但两者共享 `ITableDataSource` 接口。设计文档未说明 `ITableDataSource` 是否泛型化（如 `ITableDataSource<TRow>`），还是通过非泛型接口返回 `object` 或基类型。如果 `ITableDataSource` 不是泛型的，则 `MapEditor`/`CubeEditor` 消费时需要进行类型转换，破坏类型安全；如果是泛型的，则应在设计中明确泛型参数。
- **所在位置**：「核心抽象」→「`ITableDataSource`（接口）」小节
- **严重程度**：轻
- **改进建议**：明确 `ITableDataSource` 的泛型定义，如 `ITableDataSource<TRow> where TRow : struct`，并分别实例化为 `ITableDataSource<MapRowData>` 和 `ITableDataSource<CubeRowData>`。同时声明返回类型为 `IReadOnlyList<TRow>` 或类似集合类型。

---

## 整体质量评价

本设计方案在模块划分、抽象层次和职责分离上思路清晰，`CalibrationEditorBase<T>` 抽象基类的设计合理封装了公共 UI 结构和依赖属性，`ZSliceActivationTracker` 的抽取体现了良好的关注点分离。v2 修订对 `CurveEditor` 横向布局与 `DataGrid` 纵向模型不匹配问题的处理是正确且必要的。

但产出在**可落地的接口契约完整性**和**需求响应充分度**方面存在明显不足：核心接口（`ICurveData`、`IMapData`、`ICubeData`、`IChartPresenter`、`ICurveTablePresenter`）均缺少方法签名，无法直接指导编码实现；多处需求点（数值精度、列宽调整、编辑反馈机制、变量组显示格式）在设计中完全缺失；z 列分组替代方案的滚动可见性问题和高亮驱动逻辑的混淆属于影响用户体验的关键缺陷，需要在进入详细设计前修正。

---

*审查完成*
