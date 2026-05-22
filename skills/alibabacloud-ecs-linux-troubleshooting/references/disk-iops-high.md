# 磁盘 IOPS 异常升高

## 确定是否属于 GuestOS 问题

1. 通过 `aliyun.ecs.DescribeInstances` 获取实例规格，通过 `aliyun.ecs.DescribeInstanceTypes` 获取共享型规格列表，判断当前实例规格是否为共享型规格。
   - 是则无需排查，共享型规格不承诺性能 SLA。
   - 否：继续。

## GuestOS 内问题排查流程

### 相关组件

- 磁盘设备与块层
- 文件系统与 IO 调度
- 业务进程 IO 行为

### 问题定位

1. 运行 `iostat 1` 命令观测磁盘读写次数。
   - 过高则给出结论与修复建议
   - 不高则继续下一步
2. 运行 `pidstat -d 1` 命令观测进程磁盘读写。
   - 过高则给出结论与修复建议
   - 不高则继续下一步
