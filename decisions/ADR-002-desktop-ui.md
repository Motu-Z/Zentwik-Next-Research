# ADR-002：桌面 UI 技术

- 状态：Accepted（暂定）
- 日期：2026-11-25
- 相关：`research/04-tech-stack-evaluation.md`、`research/03-ui-ipc-deployment.md`

## 背景

候选：WPF、WinUI 3、Avalonia 12、Tauri 2、Qt 6。约束：Windows 优先、需要可访问性、运行时中英切换、主题与强调色、高密度数据展示、单人维护。

## 决策

WPF + 自绘图表（`StreamGeometry` / `DrawingVisual`），不使用第三方图表库。视图层保持薄，ViewModel 不引用 WPF 类型。

## 证据

实测（8 序列 × 600 点，Release，JIT，10 s）：

| | 渲染 p50 | 进程 CPU（%单核） | RSS |
|---|---|---|---|
| WPF 10 Hz | 0.254 ms | 16.6% | 120 MB |
| Avalonia 10 Hz | 0.303 ms | 29.4% | 237 MB |

绘图成本两者相同（10 Hz 预算 100 ms），Avalonia 在本机反而更重。更关键的是：产品域（电源方案、CPU Sets、PawnIO、MSR）只有 Windows 实现，跨平台 UI 的优势被中和；可访问性上 WPF 是长期验证过的方案，Avalonia 12 有 automation peer 但未实测。

## 候选与否决理由

- **WinUI 3**：需 Windows App SDK 运行时与 MSIX/自包含打包；无内置托盘（产品需要）；数据密集表格口碑差；本机未验证构建。
- **Avalonia 12**：保留为跨平台成为真实需求时的第一替代。其传递依赖 `Tmds.DBus.Protocol` 0.21.2 有 high 级漏洞（Windows 运行时不走该路径）。
- **Tauri 2**：需 Rust + WebView2，双语言边界落在最危险的硬件封送层；托盘/服务/驱动安装仍要 Win32。
- **Qt 6**：无 SDK、LGPL 动态链接约束、C++ 维护成本；图表与原生性能优势在 10 Hz 下无意义。

## 后果

正面：零额外运行时依赖、可访问性与主题机制成熟、调试工具链完整。
负面：无 AOT、内存约 120 MB、渲染停留在 DirectX 9 代、60 Hz 需 vsync 回调而非 `DispatcherTimer`。

## 验证与回滚

- 必做：UIA 树实测 + Narrator 走查（0.5 天）；真实数据面 10 Hz × 8 序列 + 300 行表格压测（1 天）。
- 回滚条件：可访问性实测不达标且无法用 `AutomationProperties` 补齐，或需要跨平台发布。回滚目标 Avalonia 12，成本估计 2–3 周（XAML 语法近似，ViewModel 不动）。
