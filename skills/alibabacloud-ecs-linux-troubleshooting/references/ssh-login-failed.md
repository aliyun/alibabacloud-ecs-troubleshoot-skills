# 无法通过 SSH 远程登录实例

## 确定是否属于 GuestOS 问题

1. **通过 `aliyun.ecs.DescribeCloudAssistantStatus` 查看云助手状态，确认 GuestOS 是否已正常启动？**
   - 否：先按 [guestos-not-running](guestos-not-running.md) 排查。
   - 是：继续。
2. **通过 `aliyun.ecs.DescribeSecurityGroups` 确认安全组是否放行入方向 SSH？**
   - 否：先配置规则。
   - 是：继续。

## GuestOS 内问题排查流程

### 相关组件

- 网卡 IP 与路由
- 防火墙
- sshd 及配置
- PAM
- Shell 及依赖库

### 问题定位

1. **`ssh` 登录错误是否匹配 [已知问题](https://help.aliyun.com/zh/ecs/support/troubleshooting-guidelines-when-you-cannot-remotely-log-on-to-a-linux-instance-through-ssh#0b2ba7509557s)？**
   - 是：按该文档。
   - 否：继续。
2. **云助手会话管理能登录 GuestOS 吗？**
   - 能：多半 PAM 问题，按 [guestos-pam](utils/guestos-pam.md)。
   - 否：继续。
3. **云助手能在 GuestOS 内执行命令吗？**
   - 不能或报 `SystemDefaultShellNotFound` 错误：多半 Shell 问题，先按 [guestos-pe-prep](utils/guestos-pe-prep.md) 进入 chroot 环境后，按 [guestos-shell](utils/guestos-shell.md) 排查。
   - 能：按 [guestos-shell](utils/guestos-shell.md) 排查。
