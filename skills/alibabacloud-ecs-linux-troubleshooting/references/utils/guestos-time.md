
# 时间配置排查

本文描述 GuestOS 内时区与 NTP/chrony 时间同步的排查步骤。

## 排查步骤

1. 运行 `ls -la /etc/localtime` 命令确认是否指向 `/usr/share/zoneinfo/` 下某文件的软链；不是则建议改为对应软链。
2. 检查 NTP/chrony 配置中的 ntp server 是否为阿里云 NTP 服务器。参考：[配置 NTP 服务确保实例时间准确](https://help.aliyun.com/zh/ecs/user-guide/alibaba-cloud-ntp-server#1d2319ae414lc)
3. 确认 NTP/chrony 服务是否已启动。
4. 运行 `journalctl -u ntp` 或 `journalctl -u chronyd` 命令查看日志，是否有明显报错。
5. 运行 `ls -la /etc/localtime` 命令查看时区是否与 `/etc/timezone` 文件中配置的时区一致。不一致则建议用户修改 `/etc/timezone` 使其与 `/etc/localtime` 一致。