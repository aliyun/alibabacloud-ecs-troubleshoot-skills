# CPU 使用率异常升高

## 确定是否属于 GuestOS 问题

1. 通过 `aliyun.ecs.DescribeInstances` 获取实例规格，通过 `aliyun.ecs.DescribeInstanceTypes` 获取共享型规格列表，判断当前实例规格是否为共享型规格。
   - 是则无需排查，共享型规格不承诺性能 SLA。
   - 否：继续。

## GuestOS 内问题排查流程

### 相关组件

- CPU 压力
- 进程调度器
- 业务进程
- perf 工具
- 硬中断
- 软中断

### 问题定位

1. 建议使用云监控定位异常时间是否与业务高峰一致。
   - 有则属于业务高峰正常现象。
   - 无则继续排查。
2. 运行 `top -b -n 1` 命令查看第三行 %Cpu 哪项高。
   - user 高：用户态，运行 `perf top -p <pid>` 命令定位。
   - system 高：运行 `perf record -g && perf report` 命令定位内核热点。
   - hardirq 高：运行 `cat /proc/interrupts` 命令查看硬中断情况。
   - softirq 高：运行 `cat /proc/softirqs` 命令查看软中断情况。
   - wait 高：运行 `ps aux | awk '$8 ~ /D/'` 命令查看 D 状态进程、运行 `strace -p <pid>` 命令查看系统调用情况、运行 `cat /proc/<pid>/stack` 命令查看调用栈。
3. 观察 `top -b -n 1` 命令输出：CPU 利用率不高但 Tasks 大量 running。可能为频繁 execve/fork，运行 `pidstat 1 5` 命令观测。
