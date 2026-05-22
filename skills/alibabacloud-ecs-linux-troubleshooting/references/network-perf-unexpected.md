# 网络收发性能不符合预期

## 确定是否属于 GuestOS 问题

1. 通过 `aliyun.ecs.DescribeInstances` 获取实例规格，通过 `aliyun.ecs.DescribeInstanceTypes` 获取共享型规格列表，判断当前实例规格是否为共享型规格。
   - 是则无需排查，共享型规格不承诺性能 SLA。
   - 否：继续。
2. 向用户确认测试方法是否使用[阿里云官方推荐方法](https://help.aliyun.com/zh/ecs/user-guide/best-practices-for-testing-network-performance)
   - 未使用官方推荐方法：按帮助中心网络性能测试最佳实践重测。
   - 已使用：继续。
3. 新建同规格同镜像测试机，验证是否确实不符合预期
   - 新实例符合预期：多为客户环境或配置问题。
   - 新实例仍不符合预期：继续。
4. 确认是否为 AMD 实例更新内核后性能下降的已知问题：[AMD 实例内核更新性能说明](https://help.aliyun.com/zh/ecs/user-guide/performance-may-degrade-after-the-guest-operating-system-kernel-of-an-amd-instance-is-updated#f664bab6fejjj)
   - 是已知问题：按文档说明处理。
   - 否：继续。

## GuestOS 内问题排查流程

### 相关组件

- sysctl 网络参数
- irqbalance
- ecs_mq
- 中断亲和性配置
- TCP 内存压力

### 问题定位

1. 执行 `sysctl -a` 命令与同镜像新实例对比，是否有客户自定义网络配置
   - 有则清除后重新测试。
   - 无则继续下一步。
2. 执行 `ps aux | grep irqbalance` 或 `systemctl status ecs_mq` 检查 irqbalance/ecs_mq 是否开启；执行 `cat /proc/irq/<NIC_IRQ_NUM>/smp_affinity` 命令看中断亲和性是否合理
   - 不合理则按同镜像新实例设置后重新测试。
   - 合理则继续下一步。
3. 执行 `dmesg -T` 命令查看是否有 TCP bucket 满或 TCP memory 满日志
   - 有：可能内存压力或 Socket 泄露，建议增大 tcp_mem、tcp_rmem、tcp_wmem 等参数后重新测试。
   - 无：提工单。
