# OOD 设计方案审查报告（v1）

## 审查结果

REJECTED

## 逐维度审查

### 1. 类型系统可行性

**[通过]** 设计中的类型形态选择与 C# 类型系统完全匹配。`ICalibrationData` 继承 `INotifyPropertyChanged` 符合接口可继承接口的规则；`CalibrationEditorBase<T>` 作为泛型抽象类继承 `TemplatedControl` 并约束 `T : ICalibrationData`，在 C# 泛型系统中完全可行；三个 Editor 使用 `sealed class` 是终态控件的合理选择；`ITableDataSource<TRow> where TRow : struct` 的泛型约束合法；`readonly struct` 作为值类型行数据源在类型系统中无阻碍；`ZSliceActivationTracker` 作为独立类与控件解耦，类型关系清晰。接口之间的继承（`ICurveData`/`IMapData`/`ICubeData` → `ICalibrationData`）和实现关系无冲突，无循环依赖。

### 2. 标准库与生态覆盖

**[一般]** `CanUserResizeRows` 在 Avalonia DataGrid 中不存在——设计方案在「列宽/行高调整策略」中明确声明对 `MapEditor`/`CubeEditor` 的 DataGrid "启用 `CanUserResizeRows`，允许用户调整行高"。但 Avalonia（包括 V12）的 `DataGrid` 控件从未提供 `CanUserResizeRows` 属性或任何内置的用户拖拽调整行高机制。DataGrid 仅支持列宽调整（`CanUserResizeColumns`）。行高由 `RowHeight`、`MinRowHeight`、`MaxRowHeight` 属性控制，或通过行内容与 `DataGridLength.Auto` 自动计算，不支持用户交互式拖拽调整。该假设在 Avalonia 框架层面不成立，设计文档中需移除此声明并调整相关交互预期。

**[一般]** `UnloadingRow` 事件在 Avalonia DataGrid 中不存在——设计方案在「z 切片级高亮状态与 DataGrid 虚拟化联动」中依赖 `DataGrid` 的 `UnloadingRow` 事件在行视觉元素回收前清理附加的样式类。但 Avalonia 的 `DataGrid`（包括 V12）不提供 `UnloadingRow` 事件，WPF 的 DataGrid 同样没有此事件。在虚拟化和行回收场景下，行容器被回收后复用于新数据行时，可能携带残留的 `active-z-slice` 等样式类，导致视觉状态错误。需要补充替代的样式清理策略。

**[轻微]** `IChartPresenter.AttachTo(Control host)` 引入了 Avalonia `Control` 类型依赖，与模块职责描述中「图表抽象层...自身无 UI 框架依赖」存在轻微矛盾。虽然图表渲染最终必须关联到 Avalonia 视觉树，但模块描述与接口签名之间的表述不一致建议统一。

### 3. 语言特性可行性

**[通过]** 即时同步回写模式在 C# 事件驱动模型中完全可行；`ZSliceActivationTracker` 的键盘防抖策略可通过 `DispatcherTimer` 或 `Task.Delay` 实现；`Dispatcher.UIThread.Post` 的 UI 线程调度符合 Avalonia 的线程亲和模型；`readonly struct` 的验证反馈通过 `CellEditEnding` 事件拦截 + 编辑模板手动状态控制虽然增加了实现复杂度，但在 Avalonia 的样式系统下可行；数据模型写异常通过 try-catch 降级处理的策略符合 C# 错误处理惯例；资源管理方面无特殊需求，无 IDisposable 泄漏风险。

### 4. 设计一致性

**[通过]** 各抽象的职责描述清晰无歧义。高亮叠加策略中三种高亮的驱动源已完全分离（单元格级/行级由 DataGrid 选中状态驱动，z 切片级由 `ZSliceActivationTracker` 驱动），解决了上一轮审查中的严重问题；z 列展示策略从「首行显示」调整为「每行重复显示」，逻辑自洽且兼容虚拟化；变量切换流程形成完整闭环（ComboBox 变更 → `SelectedVariable` 更新 → `OnSelectedVariableChanged` → 子类响应 → 空状态处理）；精度信息从 `ICalibrationData` → 表格适配层 → 图表层的流转路径完整；模块间依赖方向合理（数据契约层为最底层，无 outward 依赖；Views 层向下依赖所有下层模块），无循环依赖。

### 5. 设计质量

**[通过]** 职责划分遵循单一职责原则：`CalibrationEditorBase<T>` 负责公共 UI 结构和变量切换；`ZSliceActivationTracker` 独立管理 z 切片激活语义；表格适配层负责多维到二维的投影；图表抽象层隔离具体渲染库。抽象层次恰当，未过度设计（如折叠/展开机制留待实现阶段评估，不预引入抽象），也未设计不足（核心接口均补充了最小契约成员）。`ZSliceActivationTracker` 不依赖 Avalonia 视觉类型，仅消费 `CubeRowData` 并通过纯 .NET 事件输出状态，具备良好的单元测试隔离性；`IChartPresenter` 的接口抽象使图表库可替换，便于 mock 测试。

## 修改要求（REJECTED 时存在）

### 问题 1：`CanUserResizeRows` 在 Avalonia DataGrid 中不存在

- **问题**：设计方案对 `MapEditor`/`CubeEditor` 的 DataGrid 启用了 `CanUserResizeRows`，但 Avalonia（含 V12）的 DataGrid 控件不提供此属性或任何用户拖拽调整行高的能力。
- **原因**：该假设直接导致设计文档中声明的一项用户交互功能在目标框架层面无法实现，属于与 Avalonia 控件能力不匹配的设计缺陷。
- **建议方向**：
  1. 移除 `MapEditor`/`CubeEditor` 策略中关于 `CanUserResizeRows` 的全部声明；
  2. 行高调整策略改为：通过 `DataGrid.RowHeight` 或 `MinRowHeight`/`MaxRowHeight` 控制默认行高，或通过行内容自适应（`DataGridLength.Auto`）保证可读性；
  3. 如需支持行高调整，可考虑在控件外层提供独立的缩放/字号调整机制，而非依赖 DataGrid 原生行高拖拽。

### 问题 2：`UnloadingRow` 事件在 Avalonia DataGrid 中不存在

- **问题**：z 切片高亮样式清理机制依赖 `DataGrid` 的 `UnloadingRow` 事件，但 Avalonia（含 V12）的 DataGrid 不提供此事件，导致虚拟化行回收时的样式残留风险。
- **原因**：行容器被虚拟化回收后复用时，若未清理已附加的样式类（如 `active-z-slice`），新进入可视区域的行可能错误地显示不属于它的视觉状态，破坏高亮逻辑的正确性。
- **建议方向**：
  1. **首选方案**：在 `LoadingRow` 事件处理器中，总是先清除该行的 `active-z-slice` 样式类，再根据当前激活 `ZIndex` 判断是否重新附加。这利用了 `LoadingRow` 在行数据变化或新行进入时必定触发的特性，天然覆盖回收复用场景；
  2. **备选方案**：订阅 `DataGridRow` 的 `Unloaded` 事件进行清理，但需注意 `Unloaded` 在行数据更新时可能不会触发（仅视觉树移除时触发）；
  3. 移除设计文档中对 `UnloadingRow` 事件的依赖描述，统一采用上述 `LoadingRow` 中重置的方案。
