# FlClash Dashboard 布局机制：实际是什么，能复用什么

**来源**：`D:\Projects\Reference Projects\FlClash-main`（只读）。
**结论一句话**：FlClash 的「自由组件布局」只存在于**窄屏（< 640 px）**，且只有**顺序 + 增删**；**没有尺寸调整**；宽屏用的是**手写固定栅格**。所以“拖拽排序 + 尺寸调整 + 布局持久化”这套说法里，第二项在 FlClash 里不存在。

## 1. 三种密度，只有一种可编辑

`lib/views/dashboard/layout.dart`：

```dart
const _minDesktopWidth = 640.0;
const _minFourColumnWidth = 860.0;

enum DashboardDensity { standard, compact, mobile;
  bool get isDesktop => this != mobile;
  static DashboardDensity of(Size size) { ... width < 640 → mobile ... }
}
```

`lib/views/dashboard/dashboard.dart`：

```dart
final editable = !DashboardDensity.of(size).isDesktop;
if (!editable) { _editing = false; }          // 宽屏强制退出编辑
...
layout.isDesktop ? _DesktopDashboard(...) : _MobileDashboard(...)
```

即：**窗口宽 ≥ 640 px 时，编辑入口整块不渲染**，宽屏布局是写死的。

## 2. 宽屏（desktop）：手写固定栅格，不可改

`_DesktopDashboard` 里所有卡片位置、高度、跨度都是常量，按列数分两种分支：

| 条件 | 布局 |
|---|---|
| 宽度 < 860 → 2 列 | 单列滚动，每行两张，`topHeight = 176/208`、`controlHeight = 56/64`、`middleHeight = 152/176` |
| 宽度 ≥ 860 → 4 列 | 三行固定（2+1+1、2+1+1、1+1+1+1），下方表格区按剩余高度决定 `tablesFit` / `fills` |

列宽公式：

```dart
final room = constraints.maxWidth - base.horizontal - gap * (columns - 1);
final column = math.min(room / columns, _maxColumnWidth * scale);   // 440 上限
final inset = (room - column * columns) / 2;                        // 居中留白
```

没有任何卡片带 `span`/`flex` 配置项，跨度是 `row([(2, card), (1, card), (1, card)])` 这种**字面量**写死的。

## 3. 窄屏（mobile）：可编辑，但只改顺序与增删

`lib/views/dashboard/edit.dart`：

- 容器：`SliverReorderableList` + `ReorderDragStartListener`（长按/拖拽把手），回调 `onReorderItem: _reorder`
- 重排：`_cards.copyAndReorder(from, to)`（`lib/common/iterable.dart`，`removeAt` + `insert`）
- 删除：右上角 `_RoundButton`，`removable` 为 false 的卡片（`currentOutbound`）不给删除按钮
- 添加：底部「可添加」列表，`_add` 追加到末尾
- 视觉：每张卡 `_Sway` 轻微摆动（3600 ms 循环，三档摆幅），拖起时 `_LiftedCard` 放大 3% + 阴影渐显
- 进出动画：`_Presence` 用 `heightFactor` 开合“房间”，`_appear` 做 0.92→1 的缩放淡入，删除时先淡出再收房间，避免卡片跳动

**没有 resize**：卡片是全宽行，高度由卡片自身决定，没有任何拖拽手柄或跨度选择器。

## 4. 持久化：一个枚举名数组

`lib/models/config.dart`：

```dart
@JsonKey(fromJson: dashboardWidgetsFromJson)
List<DashboardWidget> dashboardWidgets,
```

- 存储形态就是 `["currentOutbound", "networkSpeed", ...]`，**顺序即布局**
- 平台过滤在枚举上用注解式字段：`vpnButton(platforms: [SupportPlatform.Android])`、`networkTakeover(platforms: desktopPlatforms)`
- `dashboardWidgetsFromJson` 做三件兼容处理：缺失 → 平台默认；含 `currentOutbound`（不可删）→ 保留；全部都是已下线项 → 回落默认
- 编辑时保存要**保住其他平台的卡在原位**：`_keepingOtherPlatforms(saved, shown)` 把不属于当前平台的卡按原相对位置插回

## 5. 组件注册表模式（这部分值得照搬）

`lib/views/dashboard/widget_registry.dart` 用 extension 把「枚举值 → 卡片 widget / 图标 / 标题 / 描述 / 是否可删」聚在一处：

```dart
extension DashboardWidgetView on DashboardWidget {
  Widget get card => switch (this) { ... };
  Glyph get glyph => switch (this) { ... };
  String label(BuildContext context) => switch (this) { ... };
  bool get removable => this != DashboardWidget.currentOutbound;
}
```

好处很实在：新增一个卡片只动两处（枚举 + switch 分支），列表页、编辑器、添加区自动跟上，不需要三处同步。

另外 `dashboard.dart` 里有个值得注意的性能细节：`_OnScreen` 用 `TickerMode(enabled: _shown)` 把**滚出视口的卡片停帧**，注释写明「不可见的更新仍然要 Impeller 画整屏一帧」。这条对我们“最小化后低占用”的目标直接有用。

## 6. 对 Zentwik 的取舍

| FlClash 做法 | 我们是否照搬 | 理由 |
|---|---|---|
| 枚举 + extension 注册表 | **照搬** | 新增卡片只改两处，AI 写错的机会最小 |
| 顺序持久化为名字数组 | **照搬但要扩展** | 我们还要存跨度与可见性，形态改成 `[{id, span, visible}]`，并做缺失字段回落 |
| `SliverReorderableList` 拖拽重排 | **照搬** | 官方组件，行为稳定 |
| 进出场 `_Presence` | 照搬 | 增删卡片不跳动，成本低 |
| `TickerMode` 停视口外卡片 | **照搬** | 与最小化降载目标一致 |
| 摆动 `_Sway` 动画 | **不做** | 桌面工具的编辑态不需要晃动，且每帧重绘 |
| 只有窄屏可编辑 | **不照搬** | 我们主场景就是桌面宽屏，编辑必须能在宽屏用 |
| 无尺寸调整 | **必须补** | 用户明确要求“尺寸调整”；实现为跨列数 1/2/4 三档，不做自由像素 |

**结论**：可以复用它的**注册表 + 重排 + 进出场 + 停帧**四件事，但布局模型要自己写——因为它的布局模型是“顺序 + 平台过滤”，我们的是“顺序 + 跨列 + 可见”，且必须在宽屏可用。
