# Windows 知识领域

- 领域：`windows`
- 说明：系统 API、服务、安全、进程调度与电源管理的长期知识。

## 目录

- [process-scheduling.md](process-scheduling.md)：CPU Sets 与亲和性（Microsoft 原文 Verified）
- [power-management.md](power-management.md)：电源方案 API（Verified）
- [security-service.md](security-service.md)：服务、命名管道 ACL、调用方校验（Verified + 待验证项）

## 覆盖与缺口

- 已覆盖：电源方案、进程调度约束、服务与 IPC 安全边界。
- 待研究：注册表细节、WMI/ETW/PDH 完整语义、休眠唤醒（S3/S4）状态、VBS/HVCI 探测细节。

## 交叉引用

硬件写入链路见 [../hardware/](../hardware/)；数值语义见 [../data/numeric-semantics.md](../data/numeric-semantics.md)。
