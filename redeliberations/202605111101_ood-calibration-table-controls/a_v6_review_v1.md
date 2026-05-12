# 验证报告

## 本轮修复检查

### 严重-问题1：Avalonia DataGrid `CurrentCell` 事实错误
**状态：✅ 已修复**
- 文档已将所有 `DataGrid.CurrentCell` 引用替换为 Avalonia 实际 API 组合：`SelectedItem` + `CurrentColumn` + `DisplayIndex` + `BeginEdit()`。
- 覆盖范围包括：编辑回写（第451行）、键盘导航（第478行）、z切片激活联动（第511行）、全量刷新后选中恢复等全部关键路径。
- 消除了 WPF 概念残留导致的编译阻塞风险。

### 严重-问题2：单元格级脏标记与 `IAsyncSaveCapable` 接口能力矛盾
**状态：✅ 已修复**
- 采用方案A（降级为全局提示）：明确移除「单元格级脏标记（橙色边框）」承诺。
- `CalibrationEditorBase<T>` 职责中明确声明"控件层不维护逐单元格脏状态"（第167行）。
- 变更反馈机制中明确仅提供全局级别的未保存提示（第469行）。
- `IAsyncSaveCapable` 的 `HasUnsavedChanges` / `UnsavedChangeCount` 仅支撑信息栏全局提示，接口能力与上层展示需求一致。

### 严重-问题3：`ActiveZSliceChangedEventArgs` 数组越界风险
**状态：✅ 已修复**
- 明确规定 `OldZIndex`/`NewZIndex < 0` 时，对应 `OldZValue`/`NewZValue` 返回 `double.NaN`。
- `ActiveSliceChanged` 事件触发条件增加约束：**仅在 `OldZIndex >= 0` 且 `NewZIndex >= 0` 同时成立时触发**（第368行）。
- 杜绝了 `ZValues[-1]` 运行时异常。

### 一般-问题4：`ILineChartPresenter` 缺少折线图轴标签契约
**状态：✅ 已修复**
- 在 `ILineChartPresenter` 中补充 `void SetAxisLabels(string xName, string xUnit, string zName, string zUnit)` 方法（第342行）。
- 与 `ISurfaceChartPresenter` 的 `SetAxisLabels` 保持对称。
- `CurveEditor` 变量切换流程中明确加载数据后调用 `SetAxisLabels`（第414行）。

### 一般-问题5：全量刷新后的选中状态恢复策略歧义
**状态：✅ 已修复**
- 明确区分两种刷新路径的选中状态恢复策略（第450-453行）：
  - **全量刷新路径**（`Refresh()` + 重新设置 `ItemsSource`）：只能使用「行索引 + 列索引」恢复选中，禁止使用行对象引用。
  - **增量刷新路径**（`ReplaceRow`）：因不替换 `ItemsSource`，可保留原有 `SelectedItem` 和 `CurrentColumn` 引用恢复。
- 两种路径均确保焦点不跳回表格顶部。

### 一般-问题6：`CubeEditor` 平铺表格排序责任未明确
**状态：✅ 已修复**
- 在 `ITableDataSource<TRow>` 中新增排序契约（第261-266行）：
  - `CubeRowData` 按 `ZIndex` 升序分组，组内按 `YIndex` 升序排列。
  - `MapRowData` 按 `YIndex` 升序排列。
- 明确"排序是适配器的强制职责，不可省略"。

### 一般-问题7：`CubeRowData`/`MapRowData` 行头值类型未定义
**状态：✅ 已修复**
- 明确 `ZValue` 和 `YValue` 为 `string` 类型（第288-290行）。
- 格式化职责归属清晰：由 `CubeTableAdapter`/`MapTableAdapter` 在生成行数据时按精度策略（`FormatString` → `DisplayPrecision` → `G6`）预先格式化。
- 行头列的 `CellTemplate` 直接绑定纯文本展示，无需在模板层重复处理精度逻辑。

### 一般-问题8：动态列生成时 `FormatString`/`DisplayPrecision` 传递机制缺失
**状态：✅ 已修复**
- 在 `ITableDataSource<TRow>` 中新增 `string? FormatString { get; }` 和 `int DisplayPrecision { get; }` 属性（第276-277行）。
- 由适配器从 `ICalibrationData` 透传，供 Editor 动态生成 `DataGrid` 数据列时设置 `Binding.StringFormat`。

### 轻微-问题9：行级高亮驱动源与 Avalonia API 不匹配
**状态：✅ 已修复**
- 将驱动源从不可外部访问的 `DataGrid.CurrentItem`（protected）修正为 `DataGrid.SelectedItem`（public）。
- 通过 `SelectionChanged` 事件或 `SelectedItemProperty` 变更实现行级高亮（第538行）。

### Judge 补充-问题10：`CubeEditor` z 列点击未考虑 `XAxisLength == 0`
**状态：✅ 已修复**
- 在「z 切片激活与 3D 视图联动」中补充边界条件（第513行）：若 `XAxisLength == 0`，不存在数据单元格，跳过"选中第一个数据单元格"步骤，仅更新 3D 曲面图和高亮样式。

### Judge 补充-问题11：图表呈现器生命周期管理缺失
**状态：✅ 已修复**
- 在三个 Editor 的职责描述中明确 `IAvaloniaChartPresenter.Detach()` 的调用时机（第194-199行）：
  - 变量切换导致旧图表丢弃时
  - 控件从视觉树卸载时（`OnDetachedFromVisualTree`）
  - 控件销毁时
- 变量切换流程中补充旧呈现器 `Detach()` 步骤（第414-416行）。

---

## 需求一致性检查

| 需求项 | 状态 | 说明 |
|--------|------|------|
| 3 个独立控件（Curve/Map/Cube） | ✅ | CurveEditor、MapEditor、CubeEditor 均已定义 |
| 顶部信息栏 + 主内容区 | ✅ | `CalibrationEditorBase<T>` 统一承载 |
| 变量选择下拉框（List 数据源） | ✅ | `PART_VariableSelector` + `ItemsSource` 属性 |
| CurveEditor：折线图 + 两行横向表格 | ✅ | `ICurveTablePresenter` 处理横向布局 |
| MapEditor：3D 曲面图 + 2 维表格 | ✅ | `ITableDataSource<MapRowData>` + `ISurfaceChartPresenter` |
| CubeEditor：3D 曲面图 + 平铺表格（z/y 双列行头） | ✅ | `ITableDataSource<CubeRowData>` + `ZSliceActivationTracker` |
| z 列合并单元格 | ⚠️ | Avalonia DataGrid V12 原生不支持，**已明确标注为有意识技术折中**，降级为"每行重复显示 + 视觉分组线"，并提供未来升级路径 |
| 单元格编辑（双击触发） | ✅ | 已在「单元格编辑与数据回写」中定义 |
| 选中高亮（单元格/行/z 切片） | ✅ | 四种高亮独立驱动，层级叠加策略清晰 |
| 行头文字加粗/颜色变化 | ✅ | 「高亮叠加策略」第4项已定义 |
| 数值精度一致性 | ✅ | `FormatString` → `DisplayPrecision` → `G6` 三级后备策略 |
| 列宽/行高调整 | ✅ | `CanUserResizeColumns` + `MinColumnWidth` + `MinHeight`/`MaxHeight` |
| 变更反馈机制 | ✅ | 即时同步 + 全局未保存提示 |
| 数据绑定（INotifyPropertyChanged） | ✅ | `ICalibrationData` 明确继承 `INotifyPropertyChanged` |
| 空状态处理 | ✅ | 覆盖 null/空列表/零维度等全部边界 |
| 键盘导航 | ✅ | 方向键/Tab/Enter/Esc 行为完整定义 |
| 图表交互深度 | ✅ | MVP 静态展示，阶段2支持旋转/缩放 |

---

## 新增问题

**无阻塞性新增问题。** 以下事项在实现阶段需注意，但不影响设计方案通过：

1. **`DisplayIndex` 与集合索引的一致性**：全量刷新后选中恢复时，保存列索引可使用 `CurrentColumn.DisplayIndex` 或 `Columns.IndexOf(CurrentColumn)` 两种方式，恢复时需确保使用同一种索引语义，避免列宽拖拽后索引错位。

2. **z 列点击后的选中时机**：文档第511行表述为"在事件处理器中（或直接在其后的同步代码中）自行处理"，存在轻微歧义。由于 `OnZColumnClicked` 会触发防抖，建议在 `ActiveSliceChanged` 事件处理器中统一处理，与 3D 曲面图刷新同步，避免防抖期间视觉状态不一致。

3. **`ActiveZSliceChangedEventArgs` 的 z 值计算主体**：文档定义 `OldZValue`/`NewZValue` "由 `ICubeData.ZValues[OldZIndex]` 提供"，但未明确是由 `ZSliceActivationTracker` 在构造事件参数时计算并传入，还是事件参数自行持有 `ICubeData` 引用。建议实现时由 Tracker 计算后传入，保持事件参数的无状态 DTO 特性。

---

## 结论

本轮 v8 方案对迭代第 5 轮提出的 9 项问题及 Judge 补充的 2 项问题均已完成恰当修复，设计文档与需求 v3 保持一致，未发现新的严重矛盾或 Avalonia API 使用错误。z 列合并单元格的技术折中已明确标注为有意识的限制，并提供了合理的替代方案和未来升级路径。

APPROVED
