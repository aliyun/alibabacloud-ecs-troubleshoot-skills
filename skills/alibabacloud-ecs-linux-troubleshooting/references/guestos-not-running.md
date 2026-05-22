# 实例已经处于「运行中」状态，但 GuestOS 没有正常启动

## 确定是否属于 GuestOS 问题

1. 通过 `aliyun.ecs.DescribeInstances` 和 `aliyun.ecs.DescribeImages` 获取实例规格与镜像信息，确定规格启动模式、镜像启动模式与实例启动模式是否匹配：[UEFI/BIOS 启动模式](https://help.aliyun.com/zh/ecs/user-guide/best-practices-for-using-the-uefi-boot-mode-and-bios-boot-mode#d3b42dc870vzq)
   - 不匹配：会导致启动失败。
   - 匹配：继续。
2. 如果是 FreeBSD 镜像（通过 `aliyun.ecs.DescribeInstances`确认镜像类型）：需要满足在阿里云运行条件：[FreeBSD 操作系统兼容性](https://help.aliyun.com/zh/ecs/user-guide/the-freebsd-operating-system-compatibility?spm=a2c4g.11186623.help-menu-25365.d_4_2_15_2_5.288eb11ewWYENa)
   - 不满足：会导致启动失败。
   - 满足：继续。
3. 如果是机密实例（通过 `aliyun.ecs.DescribeInstanceTypes` 确认实例规格规格和 TDX/异构机密计算实例规格列表）：镜像是否支持 TDX/异构机密环境：[构建 TDX 机密计算环境及远程证明服务](https://help.aliyun.com/zh/ecs/user-guide/build-a-tdx-confidential-computing-environment)、[构建异构机密计算环境](https://help.aliyun.com/zh/ecs/user-guide/build-a-heterogeneous-confidential-computing-environment)
   - 不支持：会导致启动失败。
   - 支持：继续。
4. 通过 `aliyun.ecs.DescribeImage` 和 `aliyun.ecs.DescribeInstances` 获取的规格与 OS 信息，校验实例规格、操作系统与 AMD/Intel 的兼容性：[不同代系的 AMD 实例规格与操作系统兼容性说明](https://help.aliyun.com/zh/ecs/user-guide/compatibility-between-amd-instance-types-and-operating-systems)、[Intel 实例规格与操作系统兼容性说明](https://help.aliyun.com/zh/ecs/user-guide/intel-instance-specifications-and-operating-system-compatibility)
   - 不兼容：会导致启动失败。
   - 兼容：继续。

## GuestOS 内问题排查流程

### 相关组件

- GRUB 等引导程序及配置
- 内核及配置
- init 进程及配置
- 文件系统挂载及配置
- 主要系统服务及配置

### 问题定位

1. 确认 GuestOS 是否已正常启动，如果正常启动则表示不属于此现象域问题，直接返回结论给用户
   - 通过 `aliyun.ecs.GetInstanceScreenshot` 查看 VNC 截屏或`aliyun.ecs.GetInstanceConsoleOutput` 查看串口日志：到 login 界面可认为已启动。
   - 通过 `aliyun.ecs.DescribeCloudAssistantStatus` 查看云助手状态；云助手正常通常表示系统已启动。
2. 如果 GuestOS 未正常启动，参考 [guestos-pe-prep](utils/guestos-pe-prep.md) 准备离线排查环境（PE），然后在 PE 内按 Linux 启动各阶段组件排查:
   1. [guestos-boot](utils/guestos-boot.md) — 引导阶段排查  
   2. [guestos-grub](utils/guestos-grub.md) — GRUB 引导阶段排查  
   3. [guestos-kernel-initrd](utils/guestos-kernel-initrd.md) — 内核与 Initrd 阶段排查  
   4. [guestos-systemd](utils/guestos-systemd.md) — systemd 及系统服务阶段排查
