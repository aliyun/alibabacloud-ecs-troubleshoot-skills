# 系统夯机

## 确定是否属于 GuestOS 问题

1. 本域无前置边界判定，直接进入 GuestOS 内排查。

## GuestOS 内问题排查流程

### 相关组件

- 无固定 GuestOS 组件清单。

### 问题定位

1. 检查 ECS 控制台事件通知中是否存在夯机事件
   - 有则参考事件详情。
   - 无则继续。
2. 检查系统日志中是否存在夯机日志，或/var/crash/ 下是否有 coredump 文件
   - 有则参考 [如何收集操作系统宕机后的内核转储信息](https://help.aliyun.com/zh/ecs/collect-kdump-information-after-an-instance-experiences-an-operating-system-failure)，上传 coredump 文件并提工单。
   - 无则直接提工单。
