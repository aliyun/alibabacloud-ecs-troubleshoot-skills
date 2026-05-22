
# 内核与 Initrd 阶段排查

## 内核加载阶段

结合 dmesg 或运行 `aliyun.ecs.GetInstanceConsoleOutput`获取串口日志以判断内核阶段（如从 “Linux version ...” 到 “Unpacking initramfs...”）是否正常。内核问题多与规格、内核版本、底层虚拟化相关，建议提交阿里云工单协助排查。

## Initrd 阶段

从 initramfs 解压到 rootfs remount 为 initrd 阶段，失败常进入 emergency.target，`chroot /mnt` 后运行 `journalctl -b 0` 命令查看启动过程报错以定位异常。常见异常：initrd 缺 virtio/nvme 磁盘驱动；fstab 挂载点设备/目录不存在；initrd 内 `/lib/systemd/system/` 下 systemd 服务异常等。