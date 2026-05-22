
# Linux 引导阶段排查

本文描述 VM OS 启动第一阶段（BIOS/Legacy 与 UEFI）的排查。

## 无法找到引导

### no bootable device（BIOS 启动但无法找到引导）

- **检查 MBR 或系统盘是否损坏**：在 PE 内运行 `fdisk -l` 能看见问题实例系统盘但 `blkid` 无对应分区则系统盘可能损坏。读首扇区确认 MBR 是否损坏：`dd if=/dev/<SYSTEM_DISK> of=/tmp/mbr.txt bs=512 count=1`、`hexdump -C /tmp/mbr.txt`。MBR 结构见 [主引导记录](https://zh.wikipedia.org/wiki/%E4%B8%BB%E5%BC%95%E5%AF%BC%E8%AE%B0%E5%BD%95)。

## 启动进入 UEFI Shell（无法找到 UEFI 引导）

- **检查引导扇区或镜像是否损坏**：在 PE 内运行 `fdisk -l` 能见问题实例系统盘但 `blkid` 无对应分区则可能损坏。读前 2 个扇区看 PART 标识（GPT）；检查 `/boot/efi/EFI/` 下文件是否齐全。上述均正常则大概率虚拟化 OVMF 或块存储问题。

## Booting from hard disk...

- 在 PE 内运行 `fdisk -l` 检查磁盘格式/label 是否与引导要求一致。（修复示例：误改为 gpt 则用 gdisk 改回 msdos、并运行`e2fsck` 同步文件系统、chroot /mnt 环境下重装 grub2 并核对根分区 UUID）
