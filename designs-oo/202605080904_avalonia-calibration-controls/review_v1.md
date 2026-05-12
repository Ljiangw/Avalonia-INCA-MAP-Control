# OOD 设计方案审查报告（v1）

## 审查结果

REJECTED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 类型形态选择整体合理，与 C# 类型系统完全兼容。
- `ICalibrationData` 及维度特化子接口采用接口继承，充分利用了 C# 多接口实现能力，避免了后端模型可能已继承其他领域基类而导致的单继承冲突。
- `CalibrationEditorBase<T>` 作为泛型抽象类（`T : ICalibrationData`）继承自 `TemplatedControl`，类型约束合法，且三个非泛型密封控件继承泛型基类在 C# 和 Avalonia XAML 中均可正常实例化。
- `ITableDataSource`、`IChartPresenter` 等适配层抽象采用接口形态，耦合度恰当。
- `ZSliceActivationTracker` 作为独立类抽离状态管理，形态合理。

### 2. 标准库与生态覆盖

**[通过]** Avalonia V12 生态覆盖了设计方案的核心能力需求。
- `DataGrid`（含虚拟化、自定义 `CellTemplate`、样式选择器）、`TemplatedControl`、`Dispatcher.UIThread`、`INotifyPropertyChanged`、`Classes` 动态样式类机制均为 Avalonia V12 标准能力。
- 设计正确识别了 Avalonia `DataGrid` 不支持跨行单元格合并（`RowSpan`）的限制，并给出了兼容虚拟化的替代方案，判断准确。
- `ComboBox` 绑定 `IList<T>` 及选中项变更通知属于标准数据绑定模式，无生态缺口。
- 3D 曲面图确实无 Avalonia 内置方案，`IChartPresenter` 接口隔离策略合理。

### 3. 语言特性可行性

**[通过]** 设计方案中涉及的语言特性均在 C# / Avalonia 能力范围内。
- 依赖属性注册（`AvaloniaProperty.Register`）、ControlTemplate 部件约定、泛型依赖属性均受 Avalonia V12 支持。
- 单元格编辑的就地错误反馈、结构性错误的空状态降级、图表渲染异常降级三类错误处理策略均可在 Avalonia 中实现。
- UI 线程亲和性通过 `Dispatcher.UIThread.Invoke` 保障，防抖通过标准定时器机制实现，均无可行性障碍。
- 项目模块划分符合 NuGet 包组织惯例，控件继承链（`TemplatedControl` → 抽象基类 → 密封控件）符合 Avalonia 控件体系惯例。

### 4. 设计一致性

**[通过]** 整体协作关系清晰，行为契约闭环完整。
- 变量切换流程（ComboBox → 基类依赖属性 → 虚方法 → 子类重建适配器/图表）、单元格编辑回写流程（编辑确认 → 提取坐标 → 写模型 → PropertyChanged → UI 刷新）、z 切片激活联动流程（选中行 → Tracker → 激活 z 变更 → 图表刷新 + 样式更新）、高亮叠加策略（单元格/行/z 切片三级优先级）四条核心行为契约描述完整，各环节衔接无缺失。
- 依赖方向严格遵循单向分层：数据契约层（最底层）→ 适配层 / 图表抽象层 → Views 层，激活追踪层仅依赖表格适配层，无循环依赖。
- `ZSliceActivationTracker` 的「不依赖 Avalonia 视觉类型」与「由 CubeEditor 传递行数据」之间逻辑自洽，保持了状态管理逻辑的纯度和可测试性。

### 5. 设计质量

**[一般]** `ICellCoordinate` 采用「接口 + `readonly struct` 实现」的组合，但接口引用会导致值类型装箱，与其宣称的「避免堆分配压力」设计目标相矛盾。
- 设计方案明确将值类型选型理由定义为「表格数据量大时坐标对象数量巨大，值类型避免堆分配压力」。然而，`ITableDataSource` 以接口类型 `ICellCoordinate` 消费坐标时，每次赋值给接口变量/参数/属性均会发生装箱，在堆上生成包装对象。CubeEditor 场景下数据行数可达 `z 数 × y 数`，量级较大，该设计不仅无法避免堆分配，反而会在每行数据项之外额外制造大量装箱对象与 GC 压力，导致设计决策的技术依据失效。
- **建议方向**：取消 `ICellCoordinate` 接口抽象，将坐标索引直接内联到各维度行数据项类型中（如 `CurveRowData` 含 `XIndex`，`CubeRowData` 含 `XIndex/YIndex/ZIndex`），通过强类型行数据对象传递坐标信息，彻底消除接口装箱；或若必须保留统一抽象，将 `ITableDataSource` 改为泛型接口 `ITableDataSource<TCoord> where TCoord : struct, ICellCoordinate`，以泛型约束避免装箱。

**[轻微]** `ICalibrationData` 未明确继承 `INotifyPropertyChanged`。
- 需求明确要求数据模型「支持 Avalonia `INotifyPropertyChanged`」，且单元格编辑回写流程中明确依赖 `PropertyChanged` 事件驱动 UI 刷新。但 `ICalibrationData` 接口定义中仅泛称「数据变更通知能力」，未将 `INotifyPropertyChanged` 纳入继承列表。若后端实现方仅实现 `ICalibrationData` 而遗漏 `INotifyPropertyChanged`，数据绑定将无法感知变更。建议明确 `ICalibrationData : INotifyPropertyChanged`。

**[轻微]** `ZSliceActivationTracker` 的状态输出机制可进一步澄清线程语义。
- 该 Tracker 被定位为「不依赖 Avalonia 控件的具体视觉类型」，但其输出需要最终驱动 UI 样式变更。若内部实现防抖定时器，需明确其通知机制是纯 .NET 事件（由 CubeEditor 在事件处理器中调度 UI 线程）还是 Avalonia 可观察对象，以避免实现阶段出现跨线程更新 UI 的歧义。

## 修改要求（REJECTED 时存在）

- **问题**：`ICellCoordinate` 接口引用导致 `readonly struct` 实现发生装箱，违背「避免堆分配」的设计初衷
- **原因**：接口类型在 C# 中属于引用类型，值类型赋值给接口变量时会发生装箱。CubeEditor 大数据量场景下，每行数据项携带的坐标对象会在堆上生成额外包装实例，制造不必要的 GC 压力，使值类型选型失去意义
- **建议方向**：将坐标索引直接内联到各维度的行数据类型中，取消独立的坐标接口；或改用泛型约束 `where TCoord : struct, ICellCoordinate` 在 `ITableDataSource` 层面保持类型统一且避免装箱

- **问题**：`ICalibrationData` 未显式继承 `INotifyPropertyChanged`
- **原因**：数据绑定刷新机制依赖 `PropertyChanged` 事件，若契约层不强制要求，后端实现可能遗漏，导致编辑后 UI 不同步
- **建议方向**：将 `ICalibrationData` 的继承关系明确为 `ICalibrationData : INotifyPropertyChanged`
