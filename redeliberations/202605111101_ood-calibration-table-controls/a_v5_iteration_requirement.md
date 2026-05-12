根据以下审查结果，迭代上一轮的产出，形成新版的文件，从而更好地满足用户需求。

## 当前审查结果

组件B质量审查报告（`b_v4_diag_v1.md`）针对 `a_v4_design_v1.md`（内部版本 v6）识别出以下质量问题，按严重程度排列：

### 严重问题（经质询调整为"一般"）

**问题1：CubeEditor z列展示方案与需求存在偏差，未明确标注为需求折中**
- 需求 `requirement.md` v3 第119–125行明确要求z轴列"相同z值的连续行采用合并单元格形式展示"。设计文档「设计决策2」采用了"z值每行重复显示+视觉分组线"的替代方案，但未在显著位置明确声明这是与需求原始意图的有意识折中，也未说明未来升级路径。
- **改进建议**：在「设计决策2」开头增加显式声明，说明因Avalonia DataGrid V12原生不支持RowSpan式跨行单元格，本方案将"合并单元格"降级为"每行重复显示+视觉分组线"，属于技术折中；补充未来升级路径（若后续Avalonia版本支持跨行合并或改用自定义表格组件，可重新评估）。

### 一般问题

**问题2：`ICurveTablePresenter`接口契约与职责描述不一致（列宽调整能力缺失）**
- 职责描述声明"暴露列宽调整能力"，但接口关键成员列表中无任何列宽调整相关成员（如列宽变更事件、最小/最大列宽属性等）。
- **改进建议**：在接口中补充列宽调整相关契约（如 `double MinColumnWidth { get; set; }`、`event EventHandler<int, double>? ColumnWidthChanged;`），或删除职责描述中的相关声明。

**问题3：`CalibrationEditorBase<T>`模板部件名称未正式定义**
- 文档提到"约定公共ControlTemplate的部件名称（如`PART_VariableSelector`、`PART_InfoPanel`）"，但未正式定义部件名称、预期类型和角色，也未说明缺失时的降级行为。
- **改进建议**：在「核心抽象」中补充模板部件正式定义表格（部件名称、预期类型、角色），并说明模板中未找到对应部件时的降级行为（静默跳过或抛出异常）。

**问题4：`IMapData.GetValueMatrix()`/`ICubeData.GetSliceMatrix()`返回数组的可变性语义未定义**
- 方法说明"返回二维值矩阵的视图或拷贝"，但未约定调用方对返回数组的修改权限，可能导致数据污染。
- **改进建议**：明确约定返回的`double[,]`是否为只读；若允许修改，需说明不会影响后端数据模型；或考虑更安全的返回类型。

**问题5：编辑模式下方向键行为未定义**
- 「键盘导航行为」定义了浏览模式下的方向键行为，但未定义编辑模式下（TextBox获得焦点时）按方向键的行为。
- **改进建议**：补充编辑模式方向键行为：方向键在TextBox内部移动光标（不退出编辑）；Tab/Enter确认并退出编辑；Esc取消编辑。

**问题6：空状态默认视觉呈现未定义**
- 定义了空状态触发条件和虚方法`OnEnterEmptyState()`/`OnExitEmptyState()`，但未描述基类默认实现行为。
- **改进建议**：补充基类默认空状态行为（如在主内容区中央显示"无有效数据"占位文本，灰色调，居中），子类可覆盖。

**问题7：数值输入解析策略缺失**
- 需求要求"输入的数据应能正确解析为数值类型"，但设计文档未定义具体解析规则（科学计数法、千分位、本地化小数点等）。
- **改进建议**：统一使用`CultureInfo.InvariantCulture`+`NumberStyles.Float`解析；千分位和本地化小数点不纳入支持；定义`double`溢出输入的处理方式。

**问题8：`IAsyncSaveCapable.SaveAsync()`异常语义缺失**
- 未约定保存失败时抛出异常还是返回值、异常类型约定、控件层捕获策略、`UnsavedChangesChanged`触发语义等。
- **改进建议**：明确保存失败时抛出异常；控件层try/catch捕获后显示错误提示但不阻断编辑；明确`UnsavedChangesChanged`仅在`HasUnsavedChanges`/`UnsavedChangeCount`实际值变化时触发。

**问题9：键盘Tab导航在CubeEditor跨z切片边界时的行为未定义**
- 当用户位于某个z切片最后一个数据单元格按Tab时，焦点移动策略未明确。
- **改进建议**：明确采用"自然延续到下一z切片第一行"策略，与DataGrid默认行为一致。

**问题10：`ISurfaceChartPresenter`缺少轴标签（名称+单位）设置契约**
- `LoadMapData`/`LoadSliceData`仅接收原始数组和轴值序列，未传递轴名称和单位信息。
- **改进建议**：补充`SetAxisLabels`方法，或让加载方法接收包含轴标签信息的参数。

### 轻微问题

**问题11：文档版本号与文件名不一致**
- 产出文件名为`a_v4_design_v1.md`，但文档标题为"v6"，修订说明章节也为"v6"。
- **改进建议**：统一文件名与文档内部版本号。

**问题12：`ActiveZSliceChangedEventArgs`缺少z轴实际值**
- 事件参数仅携带`OldZIndex`和`NewZIndex`，未携带对应z轴实际数值。
- **改进建议**：补充`double OldZValue { get; }`和`double NewZValue { get; }`。

**问题13：`ZSliceActivationTracker`防抖延迟未参数化**
- 文档提到"100–200ms"防抖延迟，但构造函数未提供可配置参数。
- **改进建议**：构造函数增加`TimeSpan debounceDelay`参数，提供合理默认值（如150ms）。

**问题14：精度展示策略编辑模式后备格式未细化（经质询确认基础定义已存在）**
- 经质询核实，设计文档第518行已定义"单元格进入编辑模式时TextBox初始文本按相同格式策略格式化"。但`FormatString`为空且`DisplayPrecision`为0时的后备格式（如`G6`）以及编辑完成后重新格式化的精确规则未覆盖。
- **改进建议**：在现有精度策略基础上补充后备格式规则（`G6`作为默认）和编辑完成后重新格式化的精确规则。

## 历史迭代回顾

### 已解决的问题
- **迭代第1轮全部13项问题**：行级高亮与z切片级高亮驱动逻辑混淆、z列首行虚拟化滚动丢失z值标识、`ICurveData`/`IMapData`/`ICubeData`缺少方法签名、`IChartPresenter`缺少契约、`ICurveTablePresenter`缺少签名、精度展示未响应、列宽/行高调整未响应、变更反馈机制未明确、下拉框显示格式缺少`GroupName`、信息栏格式未定义、空状态边界不完整、图表交互深度未确定、`readonly struct`编辑验证路径未明确、`ITableDataSource`泛型关系未明确。以上问题在v6中已通过修订说明中的10项修改措施全部解决。
- **迭代第2轮全部8项问题**：`readonly struct`根因分析不准确、全量刷新选中状态丢失、`PropertyChanged`约定缺失、`ISurfaceChartPresenter`参数不对称、`ZSliceActivationTracker`事件未定义、`ICurveTablePresenter`缺少程序化选中、`IAsyncSaveCapable`未定义、z/y列行头文字样式未涉及。以上问题在v6中已系统性修复。
- **迭代第3轮全部12项问题**：`UnloadingRow`事件事实错误、`Rows`与增量刷新矛盾、`HasUnsavedChanges`与数量显示矛盾、Tracker选中通知缺失、`SupportsIncrementalRefresh`关联机制未定义、轴属性违反ISP、`double[,]`转换开销、CompiledBinding兼容性风险、零维度空状态未覆盖、事件参数未定义、键盘导航未定义、`OnSelectionChanged(null)`语义缺失。以上问题在v6中已通过修订说明全部修复。

### 持续存在的问题
- 无。前三轮反馈的核心问题在v6中已得到系统性修复，本轮反馈的问题均为v6中新增识别或此前未充分覆盖的落地细节。

### 新发现的问题
- 本轮（迭代第4轮→第5轮）审查报告识别的14项问题均为新发现，主要集中在：**接口契约完整性**（问题2、4、8、10）、**编码落地细节**（问题3、5、6、7、9）、**需求折中透明度**（问题1）、**文档一致性**（问题11、12、13、14）。
- 特别说明：问题1的严重程度经质询报告调整为"一般"；问题14经质询确认基础定义已存在，焦点调整为"后备格式细化"而非"完全缺失"。

## 上一轮产出路径
C:\Users\jiangwei\Documents\C#\INCA_MAP_Control\redeliberations\202605111101_ood-calibration-table-controls\a_v4_design_v1.md

## 用户需求
C:\Users\jiangwei\Documents\C#\INCA_MAP_Control\redeliberations\202605111101_ood-calibration-table-controls\requirement.md
