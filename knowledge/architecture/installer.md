# 安装器：选型、结构与实测

产品脚本：`scripts/installer/zentwik-next.iss` + `scripts/build-installer.ps1`。本文记录为什么这么做，以及验证过什么。

## 1. 选型：Inno Setup 6，不是 WiX

`research/03-ui-ipc-deployment.md` §5 的建议是 WiX v5 + Burn（服务、可选组件、事务升级、可回滚）。本轮**没有**照此执行，理由是当前形态还不需要 MSI 的那部分能力：

| 需求 | Inno 能否满足 | 说明 |
|---|---|---|
| 装/升级/卸载服务 | 能 | 调 `Zentwik.Agent.exe --install-service`，逻辑在 C# 里 |
| 可选组件勾选 | 能 | `[Tasks]` + `Flags: unchecked` |
| 运行时前置检查 | 能 | `[Code]` 里读注册表，旧版已用过 |
| 事务回滚 / 修复安装 | **不能** | 装坏只能重装；当前单机工具可接受 |
| 企业分发（GPO/Intune） | 弱 | 现阶段无此需求 |

**决定性理由是"服务逻辑只留一份"**：服务创建/升级/删除已经由 `ServiceInstaller.cs` 用 P/Invoke 实现并真机验证过。MSI 的 `<ServiceInstall>` 是另一套声明式实现，采用它等于把同一件事写两遍，还要维护两套行为一致性。Inno 只做文件搬运 + 调那条命令，打包器可替换而不影响服务行为。

**许可证**：Inno Setup 6.7 起对商业使用收取许可费（官网 *"All commercial users of Inno Setup are requested to purchase a commercial license"*，2026-11-27 访问），非商业免费；ISCC 编译时会打印 `Non-commercial use only`。本项目当前非商业，免费使用；若将来商业化，需购买许可或改用 WiX（MIT）。

## 2. 结构

```
scripts/build-installer.ps1     发布 Agent + CLI 到 pkg\，读版本号，调 ISCC
scripts/installer/zentwik-next.iss
scripts/installer/ChineseSimplified.isl   Inno 官方语言目录不含简中，需随脚本带
```

安装布局：`%ProgramFiles%\Zentwik`（平铺，不设 `Agent` 子目录），服务名 `ZentwikAgent`，数据目录 `%ProgramData%\ZentwikNext`。

四条关键行为：

- **升级**：`PrepareToInstall` 用旧 exe 执行 `--stop-service`（否则服务锁住 exe，`[Files]` 覆盖失败）→ 复制 → `ssPostInstall` 调 `--install-service`。`Install` 已改为**幂等**：服务存在时停服务、`ChangeServiceConfig` 更新启动命令行、刷新描述与恢复动作，不再报"已存在请先卸载"。
- **可选组件**：勾选功耗墙 = 安装时传 `--enable-module power-limits`。该参数在服务启动前写 `state.json`，避开与运行中服务争状态文件。**不勾选不关闭已启用模块**——语义是"启用"，不是"设为该状态"，升级时保留用户选择。
- **PawnIO 非必需**：只查 `HKLM\SYSTEM\CurrentControlSet\Services\PawnIO`，缺失弹一次提示，不阻断安装。
- **卸载**：`[UninstallRun]` 先 `--uninstall-service`（停 + 删）再删文件；数据目录在 `usPostUninstall` 询问后删，默认否。

## 3. 实测（2026-11-27，装在本机）

| 场景 | 结果 |
|---|---|
| 干净安装，不勾功耗墙 | 服务 RUNNING；`power-limits` 显示 `[可选·未启用]` |
| 升级安装（服务在跑） | 文件覆盖成功、服务重启、`Installation process succeeded` |
| 勾选功耗墙安装 | `power-limits` 变 `ready [可选·已启用]`，`powerlimit get` 读到 FAST 100 / SLOW 68 / STAPM 56 W |
| 卸载 | 服务删除（`sc` 1060）、`%ProgramFiles%\Zentwik` 清空、`%ProgramData%\ZentwikNext` 保留 |
| 旧布局迁移 | 预置 `{app}\Agent\legacy.marker`，安装后该目录被 `[InstallDelete]` 清掉 |

包体 2.3 MB（framework-dependent，要求 .NET 10 运行时；**未签名**，首次运行触发 SmartScreen）。

## 4. 踩过的坑

- **脚本必须 UTF-8 with BOM**：PowerShell 5.1 按 ANSI 读无 BOM 文件，中文注释把引号吃掉，报一堆 `意外的标记`。`.iss` 同理——Inno 认 UTF-8 BOM 才正确显示中文。Edit 工具改写后要复查前三字节是否为 `EF BB BF`。
- **pdb 会进包**：`dotnet publish` 默认产出 pdb，`[Files]` 用 `Excludes: "*.pdb"` 排除，否则包体翻倍。
- **合并发布可行**：Agent（`net10.0-windows`）与 CLI（`net10.0`）发布到同一目录不冲突——Contracts/Domain/Localization 都是单 TFM `net10.0`，同名 dll 内容一致。已实测两个入口都能启动。
- **`& $setup` 不等待**：PowerShell 对 GUI 子系统程序是异步的，必须 `Wait-Process -Name <安装器名>` 或 `Start-Process -Wait`（后者在本环境被安全策略拦）。用 `&` 后立刻查状态会读到安装前的值。
- **PowerShell 5.1 抓原生命令输出会乱码**：CLI 输出跟随控制台代码页（cmd 936 与 `chcp 65001` 下都正常），但 PowerShell 管道用 `[Console]::OutputEncoding` 解码，看到的是乱码。不是产品问题，用 cmd 或设 `$OutputEncoding` 验证。

## 5. 未做

- 数字签名（无证书）。企业环境会拦；分发给他人前需要 Authenticode。
- 自动更新（updater + 灰度）。当前靠重跑安装包升级。
- 首次安装的"组件页"以外的设置项（端口、数据目录位置）。
- 安装前检测旧版 `Zentwik`（AppId 不同，允许并存，未做冲突提示）。
