# 无法在线扩容云盘、盘中分区或文件系统

## 确定是否属于 GuestOS 问题

1. **通过 `aliyun.ecs.DescribeCloudAssistantStatus` 云助手状态，确认 GuestOS 已正常启动**
   - 否则先按 [guestos-not-running](guestos-not-running.md) 排查。
   - 是：继续。
2. **通过 `aliyun.ecs.DescribeDisks` 确认云盘是否扩容成功**
   - 没扩容成功：提工单。
   - 扩容成功：继续。

## GuestOS 内问题排查流程

### 相关组件

- virtio_blk / nvme_core 驱动
- growpart
- resize2fs / xfs_growfs 等

### 问题定位

分区与文件系统排查可参考 [guestos-disk-fs](utils/guestos-disk-fs.md)。

1. GuestOS 内是否识别到云盘整体为新大小？
   - 不能且内核≤3.6：不支持在线扩容需重启。
   - 不能且内核>3.6：提工单。
   - 能：继续。
2. 可能是分区扩容失败，建议用户手动执行 `growpart` 命令尝试分区扩容。
3. 可能是文件系统扩容失败，建议用户手动执行 `resize2fs` 或 `xfs_growfs` 命令尝试文件系统扩容。
