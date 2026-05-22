
# 网卡与路由配置排查

GuestOS 内网卡 IP 与路由配置的排查步骤。

## 排查步骤

1. 通过`aliyun.ecs.DescribeNetworkInterfaces`获取已绑定网卡列表及主私网 IP、MAC 地址。
2. 运行 `ip addr` 命令查看 GuestOS 内网卡数量与 MAC 是否与 1 中的一致。
   - 不一致：提工单。
   - 一致：继续。
3. 运行 `ip addr` 命令查看网卡是否配置了 IP。
   - 无 IP：是网络配置服务问题。
   - 有 IP：继续。
4. 各网卡 IP 是否与 1 中信息一致。
   - 存在不一致：该网卡 IP 配置有误。
   - 仅一个主私网 IP：建议参考 [辅助弹性网卡获取 IP](https://help.aliyun.com/zh/ecs/user-guide/configure-a-secondary-eni) 设 DHCP 或静态。主网卡不一致时需手写配置。
   - 有主私网 IP 与至少一个辅助私网 IP：建议参考 [分配辅助私网 IP](https://help.aliyun.com/zh/ecs/user-guide/assign-secondary-private-ip-addresses) 静态配置。网关不明时可先 DHCP 再改回静态。
5. 运行 `ip route` 命令查看路由是否与客户需求一致。
   > 常见：单主网卡单 IP：默认路由 `0.0.0.0/0 via <网关> dev <主网卡>`；主网卡+多辅助网卡多网段：主网卡默认路由，每辅助网卡一条对应网段路由（如 `172.16.136.0/24 via 172.16.136.253 dev eth1`）。
   - 不一致：给出结论与修复建议。
   - 一致：继续。
