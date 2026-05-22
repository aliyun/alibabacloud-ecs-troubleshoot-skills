# 添加或移除云盘不生效

## 确定是否属于 GuestOS 问题

1. **通过 `aliyun.ecs.DescribeDisks` 确认已挂载的磁盘状态，并确认 GuestOS 内能否看到对应设备路径？**
   - 不能：先排查「无法看到新添加的云盘」。
   - 能：继续。

## GuestOS 内问题排查流程

### 相关组件

- kconfig 配置
- udev

### 问题定位

1. 按照 [guestos-hotplug-disk](utils/guestos-hotplug-disk.md) 排查热插拔与磁盘设备识别。
