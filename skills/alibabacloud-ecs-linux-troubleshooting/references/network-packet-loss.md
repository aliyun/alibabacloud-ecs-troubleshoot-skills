# 网络收发过程中出现丢包排查

## 确定是否属于 GuestOS 问题

1. 通过 `aliyun.ecs.DescribeInstances` 获取实例规格，通过 `aliyun.ecs.DescribeInstanceTypes` 获取共享型规格列表，判断当前实例规格是否为共享型规格。
   - 是则无需排查，共享型规格不承诺性能 SLA。
   - 否：继续。
2. 向用户确认数据包是否经过 GuestOS 网络栈
   - 未到达：不属于 GuestOS 问题。
   - 到达：继续。
3. 通过 `aliyun.ecs.DescribeSecurityGroupAttribute` 查询安全组规则，并向用户确认安全组规则是否配置正确
   - 配置错误：可能导致包被拦截，不属于 GuestOS 问题。
   - 配置正确：继续。

## GuestOS 内问题排查流程

### 相关组件

- 防火墙（iptables/nftables、firewalld/ufw）
- 软中断负载
- sysctl 参数配置
- 第三方网络驱动

### 问题定位

首先，向用户确认丢包现象是否需要手动复现，还是会持续丢包。需要复现则要求用户给出复现命令，并执行验证确定可复现。无法复现则告知用户并直接结束流程。**注意**：复现命令可能会阻塞 shell，需正确处理该场景。

然后，开始排查问题：

1. 运行 `dmesg -T` 确定是否有网络队列 hang 的 call trace 或其它网络异常
   - 有则按线索进一步排查
   - 无则继续
2. 运行 `cat /proc/net/udp` 观察最后一列 drops 是否增高？
   - 有则存在 UDP 丢包
   - 无则继续
3. 运行 `tc qdisc show` 确认是否有模拟丢包规则
   - 有则给出结论
   - 无则继续
4. 运行 `ip xfrm policy show` 和 `ip xfrm state show` 查看是否有网络安全策略拦截？
   - 有则给出结论
   - 无则继续
