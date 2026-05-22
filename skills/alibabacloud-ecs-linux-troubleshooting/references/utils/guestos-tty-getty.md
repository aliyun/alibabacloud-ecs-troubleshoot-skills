
# tty 终端与 getty 排查

GuestOS 内 tty 及 getty/mintty 的排查步骤。

## 排查步骤

1. 在 VNC 上尝试 Ctrl+Alt+F2…F6 切换终端并登录，能登录吗？
2. 运行 `ps -ef` 命令查看登录失败的 tty 对应进程，是否有 getty/mintty 绑定该终端？
3. 对比异常实例与正常实例的 getty/mintty 配置，检查是否有差异。
4. 运行 `systemctl status getty@tty<终端号>.service` 命令查看对应 getty 状态，是否明显失败？
5. 运行 `journalctl -xe -u getty@tty<终端号>.service` 命令查看日志，是否有明显报错？
6. 若 tty 被多进程抢占：回溯进程树观察根部进程特征，给出结论与修复建议。
