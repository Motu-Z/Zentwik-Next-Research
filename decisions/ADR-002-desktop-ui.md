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
- **Avalonia 12**：保留为跨平台成为真实需求时的第一替代。
  - **漏洞版本归因勘误（2026-11-26，一手核实）**：原表述"传递依赖 `Tmds.DBus.Protocol` 0.21.2 有 high 级漏洞（Windows 运行时不走该路径）"**归因错误且低估范围**，已更正：
    - `0.21.2` 属于 **Avalonia 11.3.x**（`Avalonia.FreeDesktop/11.3.0 → Tmds.DBus.Protocol 0.21.2`，见 CommunityToolkit/Datasync#499 / PR#508 一手原文），**不是 Avalonia 12 的依赖**。
    - **Avalonia 12.0.0** 实际搭载 `Tmds.DBus.Protocol 0.90.3`（官方 Release 原文 "Bump Tmds.DBus.Protocol to version 0.90.3"，2026-04-07），而 `0.90.3` 落在 GHSA-xrw6-gwf8-vvr9 受影响范围 `>= 0.22.0, < 0.92.0` 内，**同样受影响**；**12.0.1** 的 Security 条目 "Linux – Bump Tmds.DBus.Protocol to 0.92.0"（2026-04-13）才修复该漏洞。
    - 漏洞：GHSA-xrw6-gwf8-vvr9 / CVE-2026-39959，High / CVSS 7.1，影响 `Tmds.DBus.Protocol < 0.21.3` 与 `>= 0.22.0, < 0.92.0`；修复版 `0.21.3`（11.x 回溯）与 `0.92.0`。来源：GitHub Advisory（2026-11-26 逐字核对）。
    - **"Windows 不走 D-Bus 路径"需限定**：运行时层面 Windows 不加载 Linux D-Bus 后端成立；但 NuGet 审计按目标框架（TFM）的依赖图判断，与运行时 OS 无关，漏洞包仍进入 Windows restore 依赖图。默认 `NuGetAuditMode=direct` 不审计传递依赖，故通常不报；若改为 `all` 会报 NU1903。因此不能把"Windows 不走该路径"当作"无漏洞风险"的通用理由。
  - 对决策的影响：WPF 选型结论不变；若未来回滚到 Avalonia，应指定 **12.0.1+** 并把 `NuGetAuditMode` 策略纳入评估，而非以 0.21.2/Windows 路径作为免除依据。
- **Tauri 2**：需 Rust + WebView2，双语言边界落在最危险的硬件封送层；托盘/服务/驱动安装仍要 Win32。
- **Qt 6**：无 SDK、LGPL 动态链接约束、C++ 维护成本；图表与原生性能优势在 10 Hz 下无意义。

## 后果

正面：零额外运行时依赖、可访问性与主题机制成熟、调试工具链完整。
负面：无 AOT、内存约 120 MB、渲染停留在 DirectX 9 代、60 Hz 需 vsync 回调而非 `DispatcherTimer`。

## 验证与回滚

- 必做：UIA 树实测 + Narrator 走查（0.5 天）；真实数据面 10 Hz × 8 序列 + 300 行表格压测（1 天）。
- 回滚条件：可访问性实测不达标且无法用 `AutomationProperties` 补齐，或需要跨平台发布。回滚目标 Avalonia 12，成本估计 2–3 周（XAML 语法近似，ViewModel 不动）。
