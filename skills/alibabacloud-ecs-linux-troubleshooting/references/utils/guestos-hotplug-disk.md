
# 热插拔与磁盘设备识别排查

GuestOS 内磁盘热插拔内核支持及在线挂载后设备识别的排查步骤。

## 热插拔内核参数

1. 运行 `cat /boot/$(uname -r)` 或类似命令检查 `CONFIG_HOTPLUG_PCI_PCIE`、`CONFIG_HOTPLUG_PCI`、`CONFIG_HOTPLUG_PCI_ACPI` 是否均为 `y`。
   - 均为 y 表示支持。
   - `is not set` 需重编内核。
   - `m` 需加载对应模块。

## 磁盘设备识别

1. 通过 `aliyun.ecs.DescribeDisks` 查询新增盘的序列号（SN，为 d-xxx 中的 xxx 部分），参考[查询云盘序列号](https://help.aliyun.com/zh/ecs/user-guide/query-the-serial-number-of-a-disk)。
2. 运行 `ls -l /dev/disk/by-id` 命令查看是否有对应 `virtio-{SN}`。不存在则属于底层虚拟化问题；存在则表示磁盘识别成功。
3. 运行 `fdisk -l` 命令确认是否未格式化、未分区；新增盘为裸盘需分区与格式化后使用。参考：[挂载数据盘](https://help.aliyun.com/zh/ecs/user-guide/attach-a-data-disk)、[卸载与重新挂载数据盘](https://help.aliyun.com/zh/ecs/user-guide/detach-a-data-disk)