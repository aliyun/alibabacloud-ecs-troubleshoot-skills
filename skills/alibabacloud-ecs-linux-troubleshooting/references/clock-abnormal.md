# 时钟异常跳变

## 确定是否属于 GuestOS 问题

1. 无前置边界判定，直接进入 GuestOS 内排查。

## GuestOS 内问题排查流程

### 相关组件

- NTP/chrony 服务
- 时区配置

### 问题定位

1. 通过 `aliyun.ecs.DescribeImages` 检查镜像 `standardizedTimeZone` 标记与实例内 `/etc/adjtime` 第三行行为是否一致。
   - 不一致：建议修改 `/etc/adjtime` 使其与标记行为一致。
   - 一致：继续下一步。
2. 运行 `timedatectl status` 命令检查系统时间配置是否符合预期。
   - 不符合：按 [guestos-time](utils/guestos-time.md) 排查。
   - 符合：继续下一步。
3. 检查 RTC 时间是否符合预期。
   - 不符合：若用户曾手动改过时间或时区，建议用户执行 `hwclock -w` 或重启实例以刷新 RTC。
   - 符合：继续下一步。
4. 检查是否存在异常进程修改时间。

如果 auditd 服务正在运行： 通过 audit 跟踪下 `clock_adjtime`、`clock_settime` 系统调用，是否有异常进程改时间
```bash
auditctl -a always,exit -F arch=b64 -S adjtimex -S settimeofday -S clock_settime -S clock_adjtime -k time-change
auditctl -a always,exit -F arch=b32 -S adjtimex -S settimeofday -S clock_settime -S clock_adjtime -k time-change
```

如果 auditd 服务不在运行：使用 [tracepoint 工具](utils/tracepoint-perf-tools.md)检查 `syscalls:sys_enter_clock_adjtime`、`syscalls:sys_enter_clock_settime` 系统调用 hook，是否有异常进程改时间。

   - 有：给出结论与修复建议。
   - 无：继续排查其他可能原因。
