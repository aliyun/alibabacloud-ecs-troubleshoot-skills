# 网络重传率高

## 确定是否属于 GuestOS 问题

1. 排查相关业务进程是否有异常日志。
2. TCP 丢包会导致重传；若确认存在丢包，先按 [network-packet-loss](network-packet-loss.md) 排查。

## GuestOS 内问题排查流程

### 相关组件

- TCP 网络栈
- 业务进程

### 问题定位

1. `tcpdump` 抓包并分析重传原因
   - 发现异常：给出结论与修复建议。
   - 未发现：继续排查其他可能原因。
