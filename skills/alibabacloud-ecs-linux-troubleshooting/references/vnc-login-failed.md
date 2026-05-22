# 无法通过 VNC 登录实例

## 确定是否属于 GuestOS 问题

1. **如果 VNC 页面无法打开**
   - 通过 `aliyun.ecs.DescribeInstances` 获取实例规格，确认是否支持 VNC
     - 参考[实例规格族概述](https://help.aliyun.com/zh/ecs/user-guide/overview-of-instance-families)
     - GPU 规格不支持 VNC
     - TDX 规格不支持 VNC
2. **如果问题是 VNC 桌面环境出现双指针**
   - 按[通过VNC远程连接实例的问题-阿里云帮助中心](https://help.aliyun.com/zh/ecs/support/through-vnc-or-workbench-instance-remote-connection-problems)处理

## GuestOS 内问题排查流程

### 相关组件

- tty 及 getty
- PAM
- Shell 及依赖库

### 问题定位

1. **询问用户能否从 VNC 以外通道（如 SSH，不含云助手）登录**
   - 能：多半 tty 问题，从 SSH 登录后按 [guestos-tty-getty](utils/guestos-tty-getty.md)。
   - 否：继续。
2. **询问用户能否从云助手会话管理登录**
   - 能：多半 PAM 问题，按 [guestos-pam](utils/guestos-pam.md)。
   - 否：继续。
3. **能否通过云助手在 GuestOS 内执行 `ls` 命令**
   - 能：多半 tty/pty 特性问题。
   - 否：多半 Shell 损坏，先按 [guestos-pe-prep](utils/guestos-pe-prep.md) 准备离线排查环境，然后在 chroot 环境里按 [guestos-shell](utils/guestos-shell.md) 排查。
