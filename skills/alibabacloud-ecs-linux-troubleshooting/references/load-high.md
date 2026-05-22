# load1m/load5m/load15m 异常升高

## 确定是否属于 GuestOS 问题

1. 通过 `aliyun.ecs.DescribeInstances` 获取实例规格，通过 `aliyun.ecs.DescribeInstanceTypes` 获取共享型规格列表，判断当前实例规格是否为共享型规格。
   - 是则无需排查，共享型规格不承诺性能 SLA。
   - 否：继续。

## GuestOS 内问题排查流程

### 相关组件

- CPU 压力
- 进程调度器
- 进程与线程（含 D 状态、短时进程）

### 问题定位

1. `ps aux | awk '$8 ~ /R/'` 看 running 进程数量。
   - 过多：可能为大量短时进程。
   - 不多：继续下一步
2. 运行 `ps aux | awk '$8 ~ /D/'` 命令查看是否有 D 状态进程。
   - 有：建议运行 `cat /proc/<pid>/stack` 命令定位调用栈卡点。
   - 无：继续排查其他可能原因。
。
