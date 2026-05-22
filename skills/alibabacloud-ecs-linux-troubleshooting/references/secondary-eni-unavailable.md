# 无法使用新添加的辅助网卡

## 确定是否属于 GuestOS 问题

1. 通过 `aliyun.ecs.DescribeInstances` 获取实例规格，确认规格支持网卡热插拔：[弹性网卡 ENI](https://help.aliyun.com/zh/ecs/user-guide/eni-overview#7d95612e86k74)。否则，新添加的辅助网卡需要重启实例才能在 GuestOS 内看到

## GuestOS 内问题排查流程

### 相关组件

- 网卡驱动
- IP 与路由配置

### 问题定位

1. 执行 `ip addr` 或 `ls /sys/class/net/` 命令，查看用户告知的网卡设备是否存在：
   - 不存在：常见：不支持热插拔、内存不足、网卡驱动问题、中断冲突等，查看系统日志尝试排查。注意：系统日志中如果存在「pci xxx failed to assign xxx io/mem」日志，多为正常现象，非异常。
   - 存在：多半未对辅助网卡配置 IP，需客户手动配置或安装多网卡工具。参见 [创建与配置弹性网卡](https://help.aliyun.com/zh/ecs/user-guide/configure-a-secondary-eni#e0f99dc02bgz1)。
