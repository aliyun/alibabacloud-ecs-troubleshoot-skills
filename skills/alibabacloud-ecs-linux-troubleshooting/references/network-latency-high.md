# 网络收发包延迟高

## 确定是否属于 GuestOS 问题

1. 通过 `aliyun.ecs.DescribeInstances` 获取实例规格，通过 `aliyun.ecs.DescribeInstanceTypes` 获取共享型规格列表，判断当前实例规格是否为共享型规格。
   - 是则无需排查，共享型规格不承诺性能 SLA。
   - 否：继续。

## GuestOS 内问题排查流程

### 相关组件

- 网络协议栈
- sysctl 网络参数
- CPU 压力
- 软硬中断负载（硬中断：网卡/VirtIO；软中断：NET_RX/NET_TX 等）
- 内存压力
- slab 压力

### 问题定位

1. 运行 `dmesg -T` 命令查看是否有 TCP bucket 满或 TCP memory 满日志
   - 有则建议增大 tcp_mem、tcp_rmem、tcp_wmem 等 sysctl参数后重新测试。
   - 无则继续下一步。
2. 运行 `top -b -n 1` 命令查看 CPU 是否过高导致包处理慢
   - 高则建议升配或降负载后重新测试。
   - 不高则继续下一步。
3. 查看软硬中断负载是否过高导致收包/协议栈处理拥塞
   - 运行 `top -b -n 1` 关注 `%hi`（硬中断）、`%si`（软中断）；或 `mpstat -P ALL 1 1` 查看各 CPU 的 `%irq`、`%soft`。
   - 运行 `cat /proc/softirqs` 查看 NET_RX、NET_TX 等计数增长是否异常、是否长期集中在少数 CPU。
   - 运行 `cat /proc/interrupts` 查看网卡/VirtIO 相关 IRQ 是否集中在少数 CPU。
   - 偏高则建议：结合 `ethtool -l <NIC_NAME>`/`ethtool -S <NIC_NAME>` 与队列、RPS/RSS/XPS、中断亲和与 `irqbalance`、GRO 等合并/卸载参数做均衡或降中断，必要时升配或降低包率后重测。
   - 不高则继续下一步。
4. 运行 `free -h` 命令查看 slab 是否过大导致协议栈申请内存慢
   - 过大则建议执行 `echo 2 > /proc/sys/vm/drop_caches` 命令释放 reclaimable slab 尝试缓解。
   - 不大则继续下一步。
