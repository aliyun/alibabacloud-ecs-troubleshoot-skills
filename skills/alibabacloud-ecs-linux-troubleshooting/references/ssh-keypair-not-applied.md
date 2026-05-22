# 尝试配置的 SSH 密钥对不生效

## 确定是否属于 GuestOS 问题

- 本域无前置边界判定，直接进入 GuestOS 内排查。

## GuestOS 内问题排查流程

### 相关组件

- cloud-init
- fw_cfg
- Shell 及其依赖
- ssh 配置、权限与属性

### 问题定位

1. 按 [guestos-cloud-init](utils/guestos-cloud-init.md) 排查 cloud-init。无问题则继续。
2. 能通过云助手执行 `ls` 命令吗？
   - 不能：先 [guestos-pe-prep](utils/guestos-pe-prep.md) 挂载 PE 后按 [guestos-shell](utils/guestos-shell.md)排查。
   - 能：继续排查。
