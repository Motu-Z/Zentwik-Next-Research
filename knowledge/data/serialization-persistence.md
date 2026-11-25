# 序列化、持久化与兼容性

- 规范名称：数据持久化与序列化契约
- 英文名称：Serialization, Persistence & Compatibility
- 缩写：SchemaVersion
- 领域：`data`
- 核验状态：Verified（项目决策 ADR-006 + 实测）
- 权威来源：本仓库 ADR-006、research/01 §4.7、research/03 §6、M2/M3 交接
- 相关文档：[numeric-semantics.md](numeric-semantics.md)、[../domain/state-ownership.md](../domain/state-ownership.md)

## 1. 存储结构（ADR-006）

- 设置、方案/场景、账本**分文件**存放，避免无关写入互相串行化。
- 写入方式：写临时文件 → 原子替换（`File.Move(overwrite: true)`）。
- 文档带 `SchemaVersion`；版本高于当前支持时**拒绝加载并保留原文件**，不做"尽力解析"。
- 方案用 `planType` 判别字段存多态，不用"万能方案对象 + 可空字段"。

## 2. 数值与格式（Verified）

- 协议与持久化一律使用 invariant culture（research/03 §6）；`CurrentCulture` 只管 UI 显示。
- IPC 帧小端；长度前缀。
- 源生成 JSON（AOT 安全）；反射重载在 NativeAOT 下会运行期抛异常。
- 用户自建名称不随语言切换改变；身份比较 `Ordinal`，显示排序用 `CurrentCulture`。

## 3. 兼容性与演进

- 增量字段需兼容旧读者；未知方法/字段不得静默成功。
- 破坏性变更才升主版本；数据迁移只向前，迁移前备份。
- 跨主版本拒绝加载并提示导出。
- 遥测列式布局演进须带 `streamSchemaVersion` 并入协商。

## 4. 已知风险（Verified 记录）

- `File.Move` 原子性依赖同卷；跨卷数据目录待验证。
- 多文件无跨文件事务；一致性需协调。
- 服务被强杀时文件完整性待验证。
- 配置原子写但不加跨进程锁，最后写入者胜出（旧版教训）。

## 5. 数值语义联动

持久化值必须保留单位信息或强类型；不得依赖代码版本"记得"单位。见 [numeric-semantics.md](numeric-semantics.md) §8。

## 6. 常见误解

- "高版本尽力解析" → 应拒绝加载保留原文件，供抢救。
- "一个万能方案对象" → 用判别字段多态。
- "落盘即生效" → 落盘与硬件生效是两件事，须回读。

## 7. 核验状态

存储决策与实测 = Verified；跨卷/强杀场景 = Unverified。
