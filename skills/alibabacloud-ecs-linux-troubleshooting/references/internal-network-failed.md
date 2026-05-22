# 实例无法连通网络

## 确定是否属于 GuestOS 问题

1. 通过采集实例内网络信息，确定实例内部是否已存在网卡设备
   - 不存在：提工单。
   - 存在：继续。

## GuestOS 内问题排查流程

### 相关组件

- 网络配置服务
- 网卡 IP、路由、DNS

### 问题定位

1. 运行 `ip addr` 命令查看网卡 IP 是否符合预期？
   - 否：按 [guestos-nic-route](utils/guestos-nic-route.md) 排查。
   - 是：继续。
2. 运行 `ip route` 命令查看路由是否符合预期？
   - 否：按 [guestos-nic-route](utils/guestos-nic-route.md) 路由部分。
   - 是：继续。
3. 运行 `cat /etc/resolv.conf` 命令查看 DNS 是否符合预期？
   - 否：按 [guestos-dns](utils/guestos-dns.md)。
   - 是：继续。
4. 按 [guestos-net-sysctl](utils/guestos-net-sysctl.md) 排查。
