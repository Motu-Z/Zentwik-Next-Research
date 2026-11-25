# IPC 契约与错误模型

- 规范名称：UI 与服务通信协议（IPC Contract）
- 英文名称：Inter-Process Communication Protocol
- 缩写：IPC、DACL
- 领域：`architecture`
- 核验状态：Verified（本仓库 ADR-004 + 实现记录）
- 权威来源：本仓库 ADR-004、research/03 §4、M2/M3 交接
- 相关文档：[security-service.md](../windows/security-service.md)、[lifecycle.md](lifecycle.md)、[../data/serialization-persistence.md](../data/serialization-persistence.md)

## 1. 传输格式（Accepted，ADR-004）

- 命名管道，帧格式：4 字节小端长度 + UTF-8 JSON 正文（M1/M2 现状）。
- 请求带自增 Id，响应按 Id 配对；事件帧 Id=0。
- 命令字 `域名.动作`（`telemetry.subscribe`）。
- 错误返回稳定错误码 + 可选 detail，客户端不解析文案。

## 2. 背压与关联（Verified）

- 遥测有积压时丢弃最新帧并计数；响应永不丢；状态事件 latest-wins；日志拉模式。
- 长度前缀需处理半包（已用 `ReadAtLeastAsync`）。
- 演进方向：单连接多路复用、二进制列式遥测（前提是实测证明 JSON 有成本，当前 4–10 Hz 单帧 <1 KB 暂无证据）。

## 3. 错误模型（Verified 设计原则）

- 稳定错误码按域前缀：`protocol.*`、`request.*`、`plan.*`、`scene.*`、`ledger.*`、`hardware.*`。
- 通道状态是一等字段，不可压成 bool；区分"查询成功但目标不可用"与"查询失败"（`ok` + `available:false` vs 失败）。
- 退出码契约（CLI）：0 成功 /1 代理不可用 /2 用法错误 /3 操作失败 /4 能力不可用 /5 超时 /6 部分成功。

## 4. 版本协商（演进方向）

- 旧版单一整数版本（v23），任一端不匹配即整机不可用。
- 演进：主版本 + 能力位协商；未知方法返回 `METHOD_UNKNOWN`，绝不静默成功。

## 5. 安全边界

见 [security-service.md](../windows/security-service.md)：调用方校验、DACL、防抢占、授权矩阵。

## 6. 常见误解

- "错误用中文字符串即可" → 文案不可作为机器判据，须稳定错误码。
- "JSON 套 JSON 没问题" → 双重转义、无 schema、AOT 风险，已改为结构化载荷。

## 7. 核验状态

帧格式与错误码 = Verified（已实现并测试）；二进制列式遥测 = Proposed（未实施）。
