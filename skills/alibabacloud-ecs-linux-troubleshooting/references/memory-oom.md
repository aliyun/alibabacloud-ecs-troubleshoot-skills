# 内存使用率高

## 确定是否属于 GuestOS 问题

1. 通过 `aliyun.ecs.DescribeInstances` 获取实例规格，通过 `aliyun.ecs.DescribeInstanceTypes` 获取共享型规格列表，判断当前实例规格是否为共享型规格。
   - 是则无需排查，共享型规格不承诺性能 SLA。
   - 否：继续。

## GuestOS 内问题排查流程

### 相关组件

- 内存压力
- OOM 机制

### 问题定位

1. **无异常现场**：建议通过云监控定位内存异常时间段并检查业务进程
2. **有异常现场**：运行 `free -h` 命令查看内存概况；运行 `vmstat` 或 `cat /proc/meminfo` 命令查看内存详情；运行 `top -b -n 1` 命令查看高内存进程。
   - 已能定位：给出结论与修复建议。
   - 若仍无法定位：建议尝试 `procrank` 工具。
