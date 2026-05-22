
# sshd 服务及配置排查

本文描述 GuestOS 内 sshd 服务及其配置的排查步骤。

## 排查步骤

1. 运行 `sudo systemctl status sshd.service`（RHEL/SUSE）或 `sudo systemctl status ssh.service`（Debian）命令查看 sshd 状态，是否有明显失败？
2. 运行 `sudo sshd -t` 命令检查 `/etc/ssh/sshd_config` 是否有明显报错？

- 参考 [Permission denied 等错误](https://help.aliyun.com/zh/ecs/support/what-do-i-do-if-the-permission-denied-please-try-again-error-message-appears-when-i-log-on-to-a-linux-instance-as-the-root-user-by-using-ssh)。
