
# DNS 配置排查

本文描述 GuestOS 内 DNS 配置（如 `/etc/resolv.conf`）的排查步骤。

## 排查步骤

1. 使用 `ls -hal /etc/resolv.conf` 查看该文件是普通文件还是符号链接。
  - **普通文件**： 将 `/etc/resolv.conf` 文件中的 `nameserver ...` 配置为阿里云 DNS（如 `100.100.2.136`、`100.100.2.138`）。
  - **符号链接**：说明实际 DNS 配置不在 `/etc/resolv.conf` 文件中，与所用网络配置服务相关，尝试按网络配置服务相关文档排查。
