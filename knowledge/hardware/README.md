# 硬件知识领域

- 领域：`hardware`
- 说明：CPU / GPU / 内存 / 传感器相关的长期技术知识与访问规范。

## 目录

- [cpu/](cpu/)：AMD SMU、Curve Optimizer、Intel OC Mailbox、CPU Sets 访问链路
- [gpu/](gpu/)：GPU 技术（待研究入口）
- [memory/](memory/)：内存技术（待研究入口）
- [sensors/](sensors/)：传感器与采样率（部分 Verified）

## 阅读顺序

1. 先读 [../data/numeric-semantics.md](../data/numeric-semantics.md)：数值语义是所有硬件规范的前置。
2. 再读 [cpu/amd-curve-optimizer.md](cpu/amd-curve-optimizer.md) 与 [cpu/intel-oc-mailbox.md](cpu/intel-oc-mailbox.md)：两份重点规范明确区分了"无单位步进"与"电压定点编码"。
3. 涉及 Windows API 时读 [../windows/](../windows/)。

## 一手证据位置

原始研究、实测数据与论证保留在 `research/02-hardware-access.md`，本目录只维护结论性规范与交叉引用。
