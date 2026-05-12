
## 迭代第 1 轮

1. **问题描述**：行级高亮驱动逻辑与需求定义矛盾——需求定义行级高亮为"当前选中单元格所在的整行"，但设计方案将其与z切片级高亮混为一谈，均通过`ZSliceActivationTracker`驱动，导致当前激活z切片内的所有行都获得行级高亮。
   - 所在位置：「关键行为契约」→「高亮叠加策略」小节
   - 严重程度：严重
   - 改进建议**：明确行级高亮由DataGrid的`CurrentItem`/`SelectedItem`或当前活动单元格所在行驱动；`ZSliceActivationTracker`仅负责z切片级高亮。补充三者的独立驱动来源说明。

2. **问题描述**：z列首行显示方案在虚拟化滚动下丢失z值标识——当某z切片的首行滚出可视区域后，该切片剩余可见行在z列中将没有任何z值标识，严重破坏三维数据的层次可读性。
   - 所在位置：「设计决策 2」小节
   - 严重程度：严重
   - 改进建议：补充滚动场景下的z值可见性保障策略，如粘性行头/悬浮标签、z值在每行重复显示、或增加独立的z切片导航面板作为辅助标识。

3. **问题描述**：`ICurveData`/`IMapData`/`ICubeData`接口缺少具体方法签名，无法指导后端实现。
   - 所在位置：「核心抽象」→「`ICurveData`/`IMapData`/`ICubeData`（接口）」小节
   - 严重程度：一般
   - 改进建议：补充三个接口的最小方法签名集，包括各轴序列访问、二维/三维索引读写方法、轴维度查询属性等。

4. **问题描述**：`IChartPresenter`缺少关键契约定义——仅描述高层职责，未给出方法、属性或事件签名，阻塞实现。
   - 所在位置：「核心抽象」→「`IChartPresenter`（接口）」小节
   - 严重程度：一般
   - 改进建议：明确最小接口签名，包括数据加载方法（考虑拆分为`ILineChartPresenter`和`ISurfaceChartPresenter`）、渲染区域关联方法、显式重绘触发方法、渲染失败降级通知机制。

5. **问题描述**：`ICurveTablePresenter`缺少接口方法签名。
   - 所在位置：「核心抽象」→「`ICurveTablePresenter`（接口）」小节
   - 严重程度：一般
   - 改进建议：补充最小接口定义，包括数据加载方法、选中单元格变更事件、单元格编辑完成事件（参数需包含列索引、行标识、新数值）、视觉树根元素属性。

6. **问题描述**：数值精度和小数位数一致性展示需求未响应。
   - 所在位置：需求`requirement.md`第178行；设计文档全文未涉及
   - 严重程度：一般
   - 改进建议：在数据契约层或表格适配层补充精度展示契约，明确精度信息的流转路径。

7. **问题描述**：列宽/行高自适应或手动调整需求未响应。
   - 所在位置：需求`requirement.md`第179行；设计文档全文未涉及
   - 严重程度：一般
   - 改进建议：在「关键行为契约」或「设计决策」中补充列宽/行高调整策略，明确DataGrid和CurveEditor横向布局的调整能力。

8. **问题描述**：数据编辑后的变更反馈机制未明确——即时同步还是显式保存、是否需要脏数据标记、保存按钮位置等核心产品决策缺失。
   - 所在位置：需求`requirement.md`第180行；设计文档「单元格编辑与数据回写」小节
   - 严重程度：一般
   - 改进建议：在「关键行为契约」中明确编辑数据回写模式及变更反馈的具体形式。

9. **问题描述**：变量选择下拉框显示格式缺少数据契约支撑——需求要求显示`<曲线变量组> AllMkn_n_AirMax`，但`ICalibrationData`缺少变量组属性。
   - 所在位置：需求`requirement.md`第41行；设计文档「核心抽象」→「`ICalibrationData`（接口）」小节
   - 严重程度：轻微
   - 改进建议：在`ICalibrationData`中补充`GroupName`属性或`DisplayName`属性，明确下拉框`DisplayMemberBinding`的绑定路径。

10. **问题描述**：顶部信息栏单位/轴信息展示格式未定义。
    - 所在位置：需求`requirement.md`第43行；设计文档「核心抽象」→「`CalibrationEditorBase<T>`（抽象类）」小节
    - 严重程度：轻微
    - 改进建议：在`CalibrationEditorBase<T>`的职责中补充信息栏文本的格式化模板，明确各轴信息的分隔符及可选性。

11. **问题描述**：空状态与边界条件处理不完整——未覆盖`ItemsSource`为null、空列表、`SelectedVariable`初始值、变量切换过渡状态等边界。
    - 所在位置：「错误处理策略」→「数据绑定不匹配错误」小节
    - 严重程度：轻微
    - 改进建议：在「关键行为契约」中补充变量切换流程的完整状态机，定义`OnEnterEmptyState`/`OnExitEmptyState`虚方法。

12. **问题描述**：图表交互深度需求未在设计中确定——未明确是否支持缩放、平移、旋转，影响`IChartPresenter`接口设计。
    - 所在位置：需求`requirement.md`第214行；设计文档「设计决策 5」小节
    - 严重程度：轻微
    - 改进建议：明确图表交互深度（如MVP仅静态展示，后续支持拖拽旋转和滚轮缩放），若支持交互则补充交互事件契约。

13. **问题描述**：`readonly struct`编辑验证反馈路径未充分考虑——拦截`CellEditEnding`事件后，DataGrid内置验证模板可能无法正常工作。
    - 所在位置：「核心抽象」→「`MapRowData`/`CubeRowData`」小节；「错误处理策略」→「输入验证错误」小节
    - 严重程度：轻微
    - 改进建议：补充`readonly struct`场景下的验证反馈具体实现路径，明确是手动实现还是改为可变`class`。

14. **问题描述**：`ITableDataSource`与不同行数据类型的泛型关系未明确。
    - 所在位置：「核心抽象」→「`ITableDataSource`（接口）」小节
    - 严重程度：轻微
    - 改进建议：明确`ITableDataSource`的泛型定义，如`ITableDataSource<TRow> where TRow : struct`。

## 迭代第 2 轮

1. **问题描述**：`readonly struct` 与 DataGrid 标准编辑机制不兼容的根因分析不准确，将 struct 值类型的 boxing 副本问题误述为 readonly 的不可变性问题，可能误导实现者认为改为可变 struct 即可启用标准编辑。
   - 所在位置：「设计决策 9」→「`readonly struct` vs `class` 的权衡分析」
   - 严重程度：严重
   - 改进建议：修正根因说明，明确指出值类型的 boxing 副本问题是根本原因，`readonly` 只是额外禁止 setter 调用；明确说明只有降级为 `class` 才能利用 DataGrid 内置编辑提交；补充 `class` 方案的 GC 影响和坐标索引内联存储损失。

2. **问题描述**：单元格编辑后采用全量刷新策略（`e.Cancel = true` + `Refresh()` + 重新设置 `ItemsSource`），但未处理替换 `ItemsSource` 后的选中状态丢失问题，也未评估全量刷新的性能影响。
   - 所在位置：「关键行为契约」→「单元格编辑与数据回写」→「MapEditor / CubeEditor（DataGrid 纵向行模型）」
   - 严重程度：严重
   - 改进建议：在编辑流程中补充「选中状态保持」步骤（编辑前保存 `CurrentCell` 坐标，刷新后恢复）；评估增量刷新可行性（如 `ObservableCollection<TRow>` 单元素替换）；大数据量场景下补充异步/延迟刷新兜底方案。

3. **问题描述**：`ICalibrationData` 的元素级写方法（`SetXValue`、`SetValue` 等）触发 `PropertyChanged` 的约定缺失，包括属性名使用什么、如何区分数据值变更与元信息变更。
   - 所在位置：「核心抽象」→「`ICurveData` / `IMapData` / `ICubeData`」；「关键行为契约」→「单元格编辑与数据回写」
   - 严重程度：一般
   - 改进建议：明确约定元素级写方法触发 `PropertyChanged` 时属性名建议为 `string.Empty`，或增加独立的 `ValuesChanged` 事件；说明控件层收到通知后的刷新策略。

4. **问题描述**：`ISurfaceChartPresenter` 两个加载方法参数风格不一致，`LoadMapData` 接收 `IMapData`，`LoadSliceData` 接收原始数组，不对称性未说明理由。
   - 所在位置：「核心抽象」→「`ILineChartPresenter` / `ISurfaceChartPresenter`」
   - 严重程度：一般
   - 改进建议：统一参数风格（均接收原始数组），或在「设计决策 5」中补充说明不对称性的原因。

5. **问题描述**：`ZSliceActivationTracker` 被定义为提供激活切片变更通知的核心类，但未定义公开事件签名（事件名称、参数类型、触发语义）。
   - 所在位置：「核心抽象」→「`ZSliceActivationTracker`」
   - 严重程度：一般
   - 改进建议：补充 `ActiveSliceChanged` 事件及 `ActiveZSliceChangedEventArgs` 定义；明确事件在防抖后触发还是每次潜在变更都触发。

6. **问题描述**：`ICurveTablePresenter` 缺少输入方向的程序化选中设置方法，无法在变量切换或图表联动时反向控制表格选中状态。
   - 所在位置：「核心抽象」→「`ICurveTablePresenter`」
   - 严重程度：一般
   - 改进建议：补充 `SelectCell(int columnIndex, CurveTableRow row)` 和 `ClearSelection()` 方法，其中 `CurveTableRow` 为 `X`/`Z` 枚举。

7. **问题描述**：`IAsyncSaveCapable` 标记接口在设计文档中被引用，但未在任何位置定义其命名空间、成员和控件层检测逻辑。
   - 所在位置：「关键行为契约」→「单元格编辑与数据回写」→「变更反馈机制」
   - 严重程度：一般
   - 改进建议：在「核心抽象」中补充 `IAsyncSaveCapable` 的完整定义（纯标记接口或含 `HasUnsavedChanges`/`SaveAsync`），并说明控件层的检测和消费逻辑。

8. **问题描述**：需求 v3 第 169 行明确要求 z 列和 y 列行头文字在激活状态下加粗或颜色变化，但 v4 设计文档完全未涉及此需求。
   - 所在位置：需求 `requirement.md` 第 169 行；设计文档「高亮叠加策略」
   - 严重程度：一般
   - 改进建议：在「高亮叠加策略」中补充 z 列/y 列行头文字的样式变化规则，明确通过样式选择器或 `LoadingRow` 事件实现。

## 迭代第 3 轮

1. **问题描述**：Avalonia DataGrid `UnloadingRow` 事件存在，设计文档错误声明其不存在，并基于错误声明构建了虚拟化状态清理策略
   - 所在位置：「z 切片级高亮状态与 DataGrid 虚拟化联动（CubeEditor）」小节第 412 行
   - 严重程度：严重
   - 改进建议：修正事实声明，将虚拟化状态清理策略改为标准的 `LoadingRow`/`UnloadingRow` 配对模式

2. **问题描述**：`ITableDataSource<TRow>.Rows` 返回 `IReadOnlyList<TRow>` 与增量刷新机制矛盾，文档描述的 `Rows[index] = newRow` 增量刷新路径在接口层面被阻断
   - 所在位置：「核心抽象」→「`ITableDataSource<TRow>`（接口）」；「关键行为契约」→「单元格编辑与数据回写」→ 增量刷新策略
   - 严重程度：严重
   - 改进建议：保留 `IReadOnlyList<TRow>` 作为只读视图，同时新增 `void ReplaceRow(int index, TRow newRow)` 方法作为增量刷新的显式契约入口

3. **问题描述**：`IAsyncSaveCapable` 的 `bool HasUnsavedChanges` 无法支撑"未保存数量"显示需求，接口设计与上层需求之间存在不可调和的矛盾
   - 所在位置：「核心抽象」→「`IAsyncSaveCapable`（接口）」；「`CalibrationEditorBase<T>`（抽象类）」脏标记状态栏提示职责
   - 严重程度：严重
   - 改进建议：在 `IAsyncSaveCapable` 中补充 `int UnsavedChangeCount { get; }`，或降级 `CalibrationEditorBase<T>` 的职责描述为布尔状态提示

4. **问题描述**：`ZSliceActivationTracker.OnZColumnClicked` 的"选中第一个数据单元格"通知机制缺失，`ActiveSliceChanged` 事件无法区分触发源
   - 所在位置：「核心抽象」→「`ZSliceActivationTracker`（类）」职责描述；「z 切片激活与 3D 视图联动（CubeEditor）」
   - 严重程度：一般
   - 改进建议：推荐将"选中第一个数据单元格"行为从 Tracker 职责中移除，改为 `CubeEditor` 自行处理选中逻辑

5. **问题描述**：`SupportsIncrementalRefresh` 与 `INotifyCollectionChanged` 的关联机制未定义，运行时只能通过类型强制转换检测
   - 所在位置：「核心抽象」→「`ITableDataSource<TRow>`（接口）」
   - 严重程度：一般
   - 改进建议：在接口中新增 `INotifyCollectionChanged? CollectionChangedNotifier { get; }` 属性

6. **问题描述**：`ICalibrationData` 轴属性对所有维度可见，一维数据模型被迫暴露无关属性，违反接口隔离原则
   - 所在位置：「核心抽象」→「`ICalibrationData`（接口）」
   - 严重程度：一般
   - 改进建议：将轴信息属性从 `ICalibrationData` 中移除，下沉到各维度子接口

7. **问题描述**：`ISurfaceChartPresenter` 的 `double[,]` 参数与 `IMapData` 逐点访问接口之间的转换开销未考虑
   - 所在位置：「核心抽象」→「`ILineChartPresenter` / `ISurfaceChartPresenter`」
   - 严重程度：一般
   - 改进建议：在 `IMapData` 中增加 `double[,] GetValueMatrix()` 方法，或在 `ISurfaceChartPresenter` 中增加逐点数据填充的替代方法

8. **问题描述**：动态列绑定 `readonly struct` 索引器与 Avalonia V12 CompiledBinding 的兼容性风险未经验证
   - 所在位置：「核心抽象」→「`MapRowData` / `CubeRowData`」；「设计决策 8」
   - 严重程度：一般
   - 改进建议：在设计决策中补充 CompiledBinding + 动态索引器绑定的技术验证项，或改为通过 `IReadOnlyList<double> Values` 暴露数据

9. **问题描述**：空状态未覆盖数据内容为零维度的场景，可能导致运行时异常或控件白屏
   - 所在位置：「错误处理策略」→「数据绑定不匹配错误」；「核心抽象」→「`ZSliceActivationTracker`（类）」
   - 严重程度：一般
   - 改进建议：在变量切换流程中增加对数据内容维度的检查，在 `ZSliceActivationTracker` 初始化逻辑中增加 `ZAxisLength == 0` 的防护

10. **问题描述**：多个事件参数类型未定义（`CurveCellSelectedEventArgs`、`CurveCellEditCompletedEventArgs`、`ChartRenderFailedEventArgs` 等）
    - 所在位置：「核心抽象」→「`ICurveTablePresenter`（接口）」；「核心抽象」→「`IChartPresenter`（接口）」
    - 严重程度：一般
    - 改进建议：在「核心抽象」中补充相关事件参数类型的完整结构定义

11. **问题描述**：键盘导航行为未在设计中充分定义，方向键、Tab 切换、跨 z 切片导航等行为缺失
    - 所在位置：需求 `requirement.md` 第 160 行、第 221 行；设计文档「关键行为契约」→「单元格编辑与数据回写」
    - 严重程度：一般
    - 改进建议：在「关键行为契约」中补充键盘导航的完整行为定义及与 `ZSliceActivationTracker` 防抖策略的交互细节

12. **问题描述**：`ZSliceActivationTracker.OnSelectionChanged(null)` 行为未定义，Tracker 对无选中项时的处理语义缺失
    - 所在位置：「核心抽象」→「`ZSliceActivationTracker`（类）」
    - 严重程度：一般
    - 改进建议：明确 `OnSelectionChanged(null)` 的语义为"保持当前激活切片不变"，若需要重置行为应通过独立方法显式触发


## ������ 4 ��

1. **��������**��ICurveTablePresenter �ӿ���Լ��ְ��������һ�£��п���������ȱʧ
   - ����λ�ã������ĳ��󡹡���ICurveTablePresenter���ӿڣ���
   - ���س̶ȣ�һ��
   - �Ľ�����**�������п����������Լ���� MinColumnWidth��ColumnWidthChanged �¼�������ɾ��ְ�������е��п���������

2. **��������**��CalibrationEditorBase<T> ģ�岿������δ��ʽ����
   - ����λ�ã������ĳ��󡹡���CalibrationEditorBase<T>�������ࣩ��
   - ���س̶ȣ�һ��
   - �Ľ�����**���Ա�����ʽ��ʽ���� PART_VariableSelector��PART_InfoPanel ��ģ�岿�������ơ�Ԥ�����ͺͽ�ɫ����˵��δ�ҵ�ʱ�Ľ�����Ϊ

3. **��������**��IMapData.GetValueMatrix() / ICubeData.GetSliceMatrix() ��������Ŀɱ�������δ����
   - ����λ�ã������ĳ��󡹡���IMapData / ICubeData��������ƾ��� 15��
   - ���س̶ȣ�һ��
   - �Ľ�����**����ȷԼ�����ص� double[,] �Ƿ�Ϊֻ��������ʹ�ø���ȫ�ķ�������

4. **��������**���༭ģʽ�·������Ϊδ����
   - ����λ�ã����ؼ���Ϊ��Լ���������̵�����Ϊ������ͨ�õ�������
   - ���س̶ȣ�һ��
   - �Ľ�����**������༭ģʽ�·����������ƶ�/�˳��༭����Tab/Enter��ȷ�ϲ��ƶ����㣩��Esc��ȡ���༭������Ϊ����

5. **��������**����״̬Ĭ���Ӿ�����δ����
   - ����λ�ã������ĳ��󡹡���CalibrationEditorBase<T>�������ࣩ���������������ԡ��������ݰ󶨲�ƥ�����
   - ���س̶ȣ�һ��
   - �Ľ�����**���������Ĭ�Ͽ�״̬��Ϊ����������ʾ��ɫռλ�ı�����˵�������ͨ�������鷽���Զ���

6. **��������**����ֵ�����������ȱʧ
   - ����λ�ã����ؼ���Ϊ��Լ��������Ԫ��༭�����ݻ�д���������������ԡ�����������֤����
   - ���س̶ȣ�һ��
   - �Ľ�����**����ȷʹ�� CultureInfo.InvariantCulture + NumberStyles.Float����֧��ǧ��λ/���ػ�С���㣬�������������ʽ

7. **��������**��IAsyncSaveCapable.SaveAsync() �쳣����ȱʧ
   - ����λ�ã������ĳ��󡹡���IAsyncSaveCapable���ӿڣ���
   - ���س̶ȣ�һ��
   - �Ľ�����**����ȷ����ʧ��ʱ�׳��쳣���ؼ��� 	ry/catch �������ԡ�UnsavedChangesChanged ��������

8. **��������**������ Tab ������ CubeEditor �� z ��Ƭ�߽�ʱ����Ϊδ����
   - ����λ�ã����ؼ���Ϊ��Լ���������̵�����Ϊ������ͨ�õ�������
   - ���س̶ȣ�һ��
   - �Ľ�����**����ȷ Tab ������Ƭ�߽�ʱ��Ȼ��������һ z ��Ƭ��һ�У��� DataGrid Ĭ����Ϊ����һ��

9. **��������**��ISurfaceChartPresenter ȱ�����ǩ������ + ��λ��������Լ
   - ����λ�ã������ĳ��󡹡���ILineChartPresenter / ISurfaceChartPresenter��
   - ���س̶ȣ�һ��
   - �Ľ�����**������ SetAxisLabels ������������Լ��ʹͼ��ʵ�ַ��ܻ�ȡ�����ƺ͵�λ��Ϣ

10. **��������**��CubeEditor z ��չʾ����δ��ȷ��עΪ��������
   - ����λ�ã�����ƾ��� 2��С�ڣ���Ӧ���� equirement.md �� 119�C125 ��
   - ���س̶ȣ�һ��
   - �Ľ�����**���ڡ���ƾ��� 2����ͷ��ʽ���������������ʣ�������δ������·��

## 迭代第 5 轮

1. **问题描述**：Avalonia DataGrid 不存在 `CurrentCell` 属性，文档中反复引用该属性属于事实错误
   - 所在位置：「关键行为契约」→「单元格编辑与数据回写」→「MapEditor / CubeEditor」；「CubeEditor 跨 z 切片导航」；「z 切片激活与 3D 视图联动」
   - 严重程度：严重
   - 改进建议**：将 `CurrentCell` 统一替换为 Avalonia 实际 API：`SelectedItem` + `CurrentColumn`/`DisplayIndex`。编辑前保存行索引和列索引，恢复时设置 `SelectedItem = Rows[index]` + `CurrentColumn = column`。程序化设置单元格时通过 `SelectedItem` + `CurrentColumn` + `BeginEdit()` 实现。

2. **问题描述**：单元格级脏标记（橙色边框）与 `IAsyncSaveCapable` 接口能力之间存在逻辑矛盾，该功能在当前契约下无法实现
   - 所在位置：「关键行为契约」→「单元格编辑与数据回写」→「变更反馈机制」；「核心抽象」→「`IAsyncSaveCapable`」
   - 严重程度：严重
   - 改进建议**：二选一：方案 A（降级）放弃单元格级脏标记，仅保留全局未保存提示；方案 B（扩展接口）在 `IAsyncSaveCapable` 中新增逐坐标脏查询能力如 `bool IsValueUnsaved(...)` 或 `IReadOnlyList<ChangedCoordinate> UnsavedChanges { get; }`。

3. **问题描述**：`ActiveZSliceChangedEventArgs.OldZValue` 在 `OldZIndex == -1` 时存在数组越界风险
   - 所在位置：「核心抽象」→「`ActiveZSliceChangedEventArgs`」；「`ZSliceActivationTracker`」
   - 严重程度：严重
   - 改进建议**：明确当索引 `< 0` 时 `OldZValue`/`NewZValue` 返回 `double.NaN`；或规定 Tracker 仅在 `OldZIndex >= 0 && NewZIndex >= 0` 时触发事件。

4. **问题描述**：`ILineChartPresenter` 缺少折线图轴标签的显式设置契约
   - 所在位置：「核心抽象」→「`ILineChartPresenter` / `ISurfaceChartPresenter`」；需求 requirement.md 第 68 行
   - 严重程度：一般
   - 改进建议**：在 `ILineChartPresenter` 中补充 `void SetAxisLabels(string xName, string xUnit, string zName, string zUnit)`，与 `ISurfaceChartPresenter` 保持对称。

5. **问题描述**：全量刷新后的选中状态恢复策略存在歧义，行对象引用在 `ItemsSource` 替换后失效
   - 所在位置：「关键行为契约」→「单元格编辑与数据回写」→「MapEditor / CubeEditor」
   - 严重程度：一般
   - 改进建议**：明确全量刷新后只能使用「行索引 + 列索引」恢复选中状态，禁止使用行对象引用；增量刷新策略下可保留行对象引用。文档应区分两种刷新路径的恢复方式。

6. **问题描述**：`CubeEditor` 平铺表格的排序责任未在契约层明确，影响 z 切片分组和高亮假设
   - 所在位置：「核心抽象」→「`ITableDataSource<TRow>`」；「`CubeRowData` / `MapRowData`」；需求 requirement.md 第 129 行
   - 严重程度：一般
   - 改进建议**：在 `ITableDataSource<TRow>` 职责中明确排序契约：`CubeRowData` 按 `ZIndex` 升序分组、组内按 `YIndex` 升序；`MapRowData` 按 `YIndex` 升序。排序是适配器的强制职责。

7. **问题描述**：`CubeRowData`/`MapRowData` 行头值（`ZValue`/`YValue`）的类型未定义，导致格式化职责归属不清
   - 所在位置：「核心抽象」→「`MapRowData` / `CubeRowData`」
   - 严重程度：一般
   - 改进建议**：明确 `ZValue`/`YValue` 为 `string` 类型，由适配器按精度策略预先格式化。行头列直接绑定纯文本展示。

8. **问题描述**：动态列生成时，`FormatString`/`DisplayPrecision` 向 `DataGridTextColumn.Binding.StringFormat` 的传递机制缺失
   - 所在位置：「核心抽象」→「`ITableDataSource<TRow>`」；「关键行为契约」→「精度展示策略」
   - 严重程度：一般
   - 改进建议**：在 `ITableDataSource<TRow>` 中新增 `string? FormatString { get; }` 和 `int DisplayPrecision { get; }`，或统一暴露 `IReadOnlyList<BindingBase> ColumnBindings { get; }`。

9. **问题描述**：行级高亮驱动源与 Avalonia DataGrid 实际 API 不匹配，`CurrentItem` 不可外部访问
   - 所在位置：「关键行为契约」→「高亮叠加策略」
   - 严重程度：轻微
   - 改进建议**：将驱动源修正为 `DataGrid.SelectedItem`（public），通过 `SelectionChanged` 事件或 `SelectedItemProperty` 变更实现行级高亮。

