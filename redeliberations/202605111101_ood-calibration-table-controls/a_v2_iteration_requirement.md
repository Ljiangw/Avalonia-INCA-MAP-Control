根据以下审查结果，迭代上一轮的产出，形成新版的文件，从而更好地满足用户需求。

## 当前审查结果

审查报告共识别出 14 项质量问题，按严重程度分布如下：

### 严重（2项）

**问题1：行级高亮驱动逻辑与需求定义矛盾**
需求定义「行级高亮」为当前选中单元格所在的整行附加轻微背景色，但设计方案在「高亮叠加策略」中将其与「z 切片级高亮」混为一谈，均通过 `ZSliceActivationTracker` 输出的状态值驱动。`ZSliceActivationTracker` 输出的是激活 z 切片的 ZIndex，若用它来驱动行级高亮，会导致当前激活 z 切片内的**所有行**都获得行级高亮，与需求定义严重矛盾。行级高亮应跟随 DataGrid 的 `CurrentItem` / `SelectedItem` 或当前活动单元格所在行驱动。
- **所在位置**：「关键行为契约」→「高亮叠加策略」小节
- **改进建议**：明确行级高亮由 DataGrid 的 `CurrentItem` / `SelectedItem` 或当前活动单元格所在行驱动，可通过 DataGrid 内置的选中行样式或基于 `CurrentCell` 状态的样式选择器实现；`ZSliceActivationTracker` 仅负责 z 切片级高亮。补充三者的独立驱动来源说明。

**问题2：z 列首行显示方案在滚动时丢失 z 值标识**
设计方案采用「z 值首行显示 + 视觉分组线」替代原生跨行合并单元格，通过 `CubeRowData.IsFirstRowInZSlice` 控制只在每组的第一个可见行渲染 z 值文本。该方案在虚拟化滚动下存在致命缺陷：当某 z 切片的首行滚出可视区域后，该切片剩余的所有可见行在 z 列中将**没有任何 z 值标识**，用户无法判断当前行属于哪个 z 切片，严重破坏三维数据的层次可读性。
- **所在位置**：「设计决策 2」小节
- **改进建议**：补充滚动场景下的 z 值可见性保障策略。可选方向：(1) 在 z 列使用粘性行头或悬浮标签，使 z 值在首行滚出后仍锚定显示；(2) 允许 z 值在切片的每一行重复显示（牺牲部分视觉简洁性换取信息完整性）；(3) 在表格顶部或侧边增加独立的 z 切片导航面板作为辅助标识。无论采用哪种方向，都需要在设计文档中明确。

### 中等（6项）

**问题3：`ICurveData` / `IMapData` / `ICubeData` 缺少具体方法签名**
三个接口被描述为"维度特化的数据契约"，但未给出任何属性或方法签名。例如 `ICurveData` 应如何暴露一维 x/z 序列？`IMapData` 如何读写二维矩阵的某个单元格？`ICubeData` 如何读写三维张量的某个单元格？缺少这些签名，后端实现方和控件开发者均无法确定契约边界。
- **所在位置**：「核心抽象」→「`ICurveData` / `IMapData` / `ICubeData`（接口）」小节
- **改进建议**：补充三个接口的最小方法签名集。例如 `ICurveData` 至少声明 `IReadOnlyList<double> XValues { get; }`、`IReadOnlyList<double> ZValues { get; }`、`void SetZValue(int index, double value)`；`IMapData` 至少声明二维索引读写方法（如 `double GetValue(int xIndex, int yIndex)` / `void SetValue(...)`）；`ICubeData` 同理需要三维索引读写。同时明确各轴维度的查询属性（如 `XAxisLength`、`YAxisLength`、`ZAxisLength`）。

**问题4：`IChartPresenter` 缺少关键契约定义**
`IChartPresenter` 被定义为"图表渲染的抽象契约"，但仅描述了高层职责，未给出任何方法、属性或事件签名。后续编码实现时无法确定：数据通过什么方法传递？渲染目标区域如何指定？图表何时重绘？3D 曲面图和折线图是否共享同一个接口？
- **所在位置**：「核心抽象」→「`IChartPresenter`（接口）」小节
- **改进建议**：明确 `IChartPresenter` 的最小接口签名。至少需要：(1) 数据加载方法（考虑是否需要拆分为 `ILineChartPresenter` 和 `ISurfaceChartPresenter`）；(2) 渲染区域/宿主关联方法（如 `AttachTo(Control host)`）；(3) 显式重绘触发方法（如 `Refresh()`）；(4) 渲染失败时的降级通知机制。

**问题5：`ICurveTablePresenter` 缺少接口方法签名**
职责描述详尽，但缺少具体的方法、属性和事件签名。未定义事件参数类型（如是否携带数据点索引、行标识、解析后的数值），也未定义如何接收 `ICurveData`、如何构建视觉树、如何响应外部刷新请求。
- **所在位置**：「核心抽象」→「`ICurveTablePresenter`（接口）」小节
- **改进建议**：补充最小接口定义。至少包括：(1) 数据加载方法（如 `LoadData(ICurveData data)`）；(2) 选中单元格变更事件（如 `EventHandler<CurveCellSelectedEventArgs> SelectedCellChanged`）；(3) 单元格编辑完成事件（如 `EventHandler<CurveCellEditCompletedEventArgs> CellEditCompleted`，参数需包含列索引、行标识[X/Z]、新数值）；(4) 视觉树根元素属性（如 `Control VisualRoot { get; }`）。

**问题6：数值精度和小数位数一致性展示需求未响应**
需求明确要求"需考虑数值精度和小数位数的一致性展示"，但设计文档完全未涉及。未说明精度信息由谁提供、由谁消费、如何在表格和图表中统一应用。
- **所在位置**：需求 `requirement.md` 第 178 行；设计文档全文未涉及
- **改进建议**：在数据契约层或表格适配层补充精度展示契约。可选方向：(1) `ICalibrationData` 增加 `int DisplayPrecision { get; }` 或 `string FormatString { get; }` 属性；(2) 表格适配层在生成行数据时应用格式字符串；(3) `ICurveTablePresenter` 和 `IChartPresenter` 的轴标签/数据标签均消费同一格式策略。明确精度信息的流转路径。

**问题7：列宽/行高自适应或手动调整需求未响应**
需求明确要求"表格应支持列宽/行高的自适应或手动调整"，但设计文档完全未涉及。对于 `MapEditor`/`CubeEditor` 的 DataGrid，x 轴列头可能数量众多且动态生成，需要明确是否允许用户拖拽调整、是否有最小/最大列宽约束。对于 `CurveEditor` 的横向自定义布局，列宽调整完全没有设计支撑。
- **所在位置**：需求 `requirement.md` 第 179 行；设计文档全文未涉及
- **改进建议**：在「关键行为契约」或「设计决策」中补充列宽/行高调整策略。对于 DataGrid 方案，明确 `CanUserResizeColumns`、`CanUserResizeRows` 的启用范围、动态生成列的宽度初始化策略。对于 `CurveEditor` 的横向布局，明确 `ICurveTablePresenter` 是否暴露列宽调整能力。

**问题8：数据编辑后的变更反馈机制未明确**
需求要求"数据编辑后应有适当的变更反馈机制（设计阶段确定具体形式，如即时同步、显式保存等）"。设计文档在「单元格编辑与数据回写」中描述了"调用原始数据模型的写方法更新对应位置"，暗示了即时同步模式，但从未明确说明。如果采用即时同步，是否需要脏数据标记？如果采用显式保存，保存按钮在哪里、状态如何反馈？
- **所在位置**：需求 `requirement.md` 第 180 行；设计文档「单元格编辑与数据回写」小节
- **改进建议**：在「关键行为契约」中明确编辑数据回写模式：(1) 若采用即时同步，说明编辑完成后立即调用数据模型写接口，并补充是否需要脏标记/未保存状态指示；(2) 若采用显式保存，说明保存触发的时机和保存失败的处理策略。同时明确"变更反馈"的具体形式（如单元格边框变色提示已修改、底部状态栏提示未保存数量等）。

### 轻微（6项）

**问题9：变量选择下拉框显示格式缺少数据契约支撑**
需求要求下拉框显示项格式为 `<曲线变量组> AllMkn_n_AirMax`，但 `ICalibrationData` 仅声明了"变量名称"属性，未定义"变量组"属性或"显示名称格式化"机制。
- **所在位置**：需求 `requirement.md` 第 41 行；设计文档「核心抽象」→「`ICalibrationData`（接口）」小节
- **改进建议**：在 `ICalibrationData` 中补充 `GroupName`（或 `CategoryName`）属性，或增加 `string DisplayName { get; }` 属性让后端直接提供格式化后的显示文本。同时明确 `CalibrationEditorBase<T>` 中下拉框 `DisplayMemberBinding` 的绑定路径。

**问题10：顶部信息栏单位/轴信息展示格式未定义**
需求给出了示例格式 `[mgpl] x: AllMkn_n_AirMax [rpm]`，但未说明 `CalibrationEditorBase<T>` 如何将这些字段组合成最终的展示字符串。当变量有多个轴时，格式模板是什么？分隔符是否固定？
- **所在位置**：需求 `requirement.md` 第 43 行；设计文档「核心抽象」→「`CalibrationEditorBase<T>`（抽象类）」小节
- **改进建议**：在 `CalibrationEditorBase<T>` 的职责中补充信息栏文本的格式化模板。例如定义默认格式为 `[{Unit}] x: {XAxisName} [{XAxisUnit}] y: {YAxisName} [{YAxisUnit}]`，或允许子类/模板覆盖。明确哪些轴信息字段为可选。

**问题11：空状态与边界条件处理不完整**
设计文档提到 `ICubeData` 中 z 维度长度为 0 时进入"空状态"，但未覆盖：(1) `ItemsSource` 为 `null` 时控件的行为；(2) `ItemsSource` 为空列表时 ComboBox 和主内容区的状态；(3) `SelectedVariable` 的初始值；(4) 变量切换过程中的过渡状态。
- **所在位置**：「错误处理策略」→「数据绑定不匹配错误」小节
- **改进建议**：在「关键行为契约」中补充「变量切换流程」的完整状态机，明确 `ItemsSource` 为 null/空列表、`SelectedVariable` 为 null 时的空状态展示。为 `CalibrationEditorBase<T>` 定义受保护的虚方法如 `OnEnterEmptyState` / `OnExitEmptyState`。

**问题12：图表交互深度需求未在设计中确定**
需求在「推断与待确认事项」中指出"折线图和 3D 曲面图是否支持缩放、平移、旋转等交互操作，由设计阶段确定"。但设计文档仅说明了通过 `IChartPresenter` 接口抽象图表渲染，未确定图表的交互深度。
- **所在位置**：需求 `requirement.md` 第 214 行；设计文档「设计决策 5」小节
- **改进建议**：在「设计决策 5」或单独的设计决策中明确图表交互深度。例如：阶段 1（MVP）仅支持默认视角静态展示；阶段 2 支持鼠标拖拽旋转（3D）和滚轮缩放。若确定支持交互，`IChartPresenter` 需要补充交互事件契约。

**问题13：`readonly struct` 编辑验证反馈路径未充分考虑**
设计文档使用 `readonly struct` 作为 DataGrid 行数据源以避免 GC 压力，但 DataGrid 的标准单元格编辑流程会尝试将编辑值自动写回绑定的数据源——对于不可变的 struct，此自动写回必然失败。设计文档说明通过 `CellEditEnding` 事件拦截实现回写，但遗漏了输入验证错误的就地反馈路径。在 `readonly struct` 且回写被事件拦截时，DataGrid 的验证模板（`DataErrorValidationRule`）可能无法正常工作，红色边框反馈可能需要完全在事件处理器中手动实现。
- **所在位置**：「核心抽象」→「`MapRowData` / `CubeRowData`」小节；「错误处理策略」→「输入验证错误」小节
- **改进建议**：在「错误处理策略」中补充 `readonly struct` 场景下的验证反馈具体实现路径。明确：是否放弃 DataGrid 内置验证模板而完全采用手动事件拦截+编辑模板状态控制？或考虑将行数据类型改为可变的 `class`？至少应在设计决策中记录此权衡。

**问题14：`ITableDataSource` 与不同行数据类型的泛型关系未明确**
`MapEditor` 和 `CubeEditor` 分别使用 `MapRowData` 和 `CubeRowData` 两种行数据类型，但两者共享 `ITableDataSource` 接口。设计文档未说明 `ITableDataSource` 是否泛型化，还是通过非泛型接口返回 `object` 或基类型。
- **所在位置**：「核心抽象」→「`ITableDataSource`（接口）」小节
- **改进建议**：明确 `ITableDataSource` 的泛型定义，如 `ITableDataSource<TRow> where TRow : struct`，并分别实例化为 `ITableDataSource<MapRowData>` 和 `ITableDataSource<CubeRowData>`。同时声明返回类型为 `IReadOnlyList<TRow>`。

---

质询报告结论：**LOCATED**（诊断结论被确认，所有 14 个问题证据充分、逻辑自洽、覆盖完备，无事实矛盾）。

## 历史迭代回顾

当前为第 2 轮迭代。历史迭代反馈（第 1 轮）中记录的全部 14 个问题在本轮审查报告中均被重新识别，不存在"已解决的问题"。本轮审查与历史反馈完全一致，所有问题均为**持续存在的问题**，需在本轮迭代中重点解决。建议优先处理 2 项严重问题和 6 项中等问题，再处理 6 项轻微问题。

## 上一轮产出路径

C:\Users\jiangwei\Documents\C#\INCA_MAP_Control\redeliberations\202605111101_ood-calibration-table-controls\a_v1_design_v2.md

## 用户需求

C:\Users\jiangwei\Documents\C#\INCA_MAP_Control\redeliberations\202605111101_ood-calibration-table-controls\requirement.md
