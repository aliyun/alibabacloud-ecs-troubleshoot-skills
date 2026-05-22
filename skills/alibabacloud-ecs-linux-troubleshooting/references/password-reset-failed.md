# 无法重置系统中的用户密码

重置系统用户密码的方式有以下几种：
1. 登录实例后使用 chpasswd/passwd 命令修改密码。
2. 通过云助手修改实例登录密码
3. 在 ECS 控制台上，**在线重置密码**
4. 通过 ECS 控制台或者 ModifyInstanceAttribute 接口**离线重置密码**

## 确定是否属于 GuestOS 问题

1. 对于方式 2、3，确认通过云助手能执行其它命令，否则为云助手自身问题。

## GuestOS 内问题排查流程

### 相关组件

- Shell 及依赖
- `/etc/passwd` 与 `/etc/shadow` 权限与属性
- chpasswd/passwd 工具

对于方式 4，还可能涉及：

- ISO 阶段
- cloud-init
- fw_cfg

### 问题定位

1. 能登录 Shell 并执行简单命令吗？
   - 不能：先 [guestos-pe-prep](utils/guestos-pe-prep.md) 准备离线排查环境 ，再在 chroot 环境中按照 [guestos-shell](utils/guestos-shell.md)排查。
   - 能：继续。
2. 能否手动 chpasswd/passwd 改密？
   - 不能：多半命令或依赖损坏，先 [guestos-pe-prep](utils/guestos-pe-prep.md) 准备离线排查环境 ，再在 chroot 环境中按照 [guestos-shell](utils/guestos-shell.md)排查。
   - 能：提工单。
3. **仅对于方式 4，按以下步骤排查：**
   - 按 [guestos-cloud-init](utils/guestos-cloud-init.md) 排查。
     - 发现问题：给出结论与修复建议。
     - 未发现问题：提工单。
