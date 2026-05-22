# 磁盘读写性能不符合预期

## 确定是否属于 GuestOS 问题

1. 通过 `aliyun.ecs.DescribeInstances` 获取实例规格，通过 `aliyun.ecs.DescribeInstanceTypes` 获取共享型规格列表，判断当前实例规格是否为共享型规格。
   - 是则无需排查，共享型规格不承诺性能 SLA。
   - 否：继续。
2. 通过 `aliyun.ecs.DescribeInstances` 获取实例规格，通过 `aliyun.ecs.DescribeInstanceTypes` 获取共享型规格列表。查看当前磁盘 IO 负载是否达到实例规格限速
   - 若已达：建议升配。
   - 若未达：继续。

## GuestOS 内问题排查流程

### 相关组件

- 块设备队列参数
- IO 调度器
- CPU 亲和性配置

### 问题定位

1. 运行 `cat /sys/block/<disk_name>/queue/rq_affinity` 命令查看磁盘请求队列的 CPU 亲和性策略，可适当调配。
2. 运行 `cat /sys/block/<disk_name>/mq/<queue_num>/cpu_list` 命令查看磁盘多队列的 CPU 亲和性策略，可适当调配。
3. 运行 `cat /sys/block/<disk_name>/queue/scheduler` 命令查看磁盘调度算法，默认应为 NOP，若有必要可适当调配。
