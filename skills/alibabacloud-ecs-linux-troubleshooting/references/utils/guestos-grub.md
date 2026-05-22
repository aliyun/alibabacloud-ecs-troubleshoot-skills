
# GRUB 引导阶段排查

只针对 GRUB2 的 boot.img 及之后阶段，不考虑 GRUB Legacy。

## 常见 GRUB 失败类型

### 文件缺失导致 GRUB 运行失败

- **内核或 initrd/initramfs 缺失**：典型为内核或 initrd/initramfs 缺失导致引导失败或 initramfs panic。
- **/boot/grub2/i386-pc 相关文件缺失**：文件缺失会导致启动失败。建议重置系统盘或从同版本机器拷贝。
- **grub 配置文件缺失**：
  - **BIOS**：`chroot /mnt` 后运行 `grub2-install /dev/<SYSTEM_DISK>` 重新安装 grub。
  - **UEFI**：`chroot /mnt` 后运行 `grub2-mkconfig -o /boot/efi/EFI/<OS_NAME>/grub.cfg` 重新生成 grub 配置。

### core.img 损坏

现象：卡在 “Booting from Hard Disk...”。`chroot /mnt` 后运行 `grub2-install /dev/<SYSTEM_DISK>` 重新安装 grub。

### 系统 /boot 目录被删除

1. `chroot /mnt` 后通过 `grub2-install`重新安装 grub2
2. `chroot /mnt` 后通过包管理器重新安装内核
3. `chroot /mnt` 后通过 `grub2-mkconfig` 重新创建 grub2 配置文件