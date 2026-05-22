# 磁盘分区表、分区和文件系统损坏或者不符合预期

## 确定是否属于 GuestOS 问题

1. 无前置边界判定，直接进入 GuestOS 内排查。

## GuestOS 内问题排查流程

### 相关组件

- 磁盘分区表
- 磁盘分区
- 文件系统
- /etc/fstab

### 问题定位

1. 按顺序定位并详见 [guestos-disk-fs](utils/guestos-disk-fs.md)。
