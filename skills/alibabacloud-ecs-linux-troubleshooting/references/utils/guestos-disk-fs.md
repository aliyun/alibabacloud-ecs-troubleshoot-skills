
# 磁盘分区与文件系统排查

GuestOS 内磁盘分区表、分区、文件系统及挂载配置的排查步骤。

## 1. 盘符与 udev 规则

如果是重启后盘符变化问题，检查 udev 规则是否导致盘符重命名。
   - 是则注释 udev 规则并重启。
   - 否则多 PCI bus 下盘符可能异步分配，建议在 `/etc/fstab` 中使用 [永久块设备命名](https://wiki.archlinux.org/title/Persistent_block_device_naming)。

## 2. 分区表与分区

运行 `fdisk -l <设备>` 或 `parted <设备> print` 命令查看分区表与分区是否正常且符合预期。

## 3. 文件系统信息

运行 `blkid | grep <设备>` 命令查看文件系统信息是否符合预期。

## 4. 挂载信息

运行 `grep <设备> /proc/mounts` 命令查看挂载是否符合预期。