# 客户现象 → 现象域路由

将用户/工单中的**自然语言现象**收敛到 **一个现象域**。所有的现象域与唯一标识以 [`phenomenon-domain.md`](phenomenon-domain.md) 为准；本文只做**分类决策**，不重复全表。

## 1. 先把现象写清楚（再分类）

至少区分下列维度（信息不足时先追问）：

| 维度 | 说明 |
| --- | --- |
| **实例状态** | 调用 `aliyun.ecs.DescribeInstances` 获取 |
| **范围** | 单实例 / 多实例；是否必现；起止时间或是否与变更、重启相关 |
| **通道** | 控制台状态（启动中/运行中）；能否 VNC；能否 SSH；能否云助手执行命令 |
| **网络方向** | 自外向实例业务端口、自实例向外网/元数据、仅内网互通等 |
| **表现** | 报错原文、监控指标（CPU/load/内存/磁盘/网络 BPS 与包级现象） |

## 2. 现象域大类

按最贴近的客户表述选大类，再进对应小节细分子域。

- **系统起不来 / 黑屏 / 卡在 grub·systemd·emergency** → 见 **§3 启动类问题**
- **连不上 / 登录失败 / 磁盘损坏 / 时钟异常 / 网络不通** → 见 **§4 使用类问题**
- **死机、panic、自动重启 / 卡住、soft lockup、不响应** → 见 **§5 宕机或夯机类问题**
- **CPU 高、load 高、内存高/OOM、磁盘 IO 高、网络慢/丢包/重传/延迟/带宽打满** → 见 **§6 性能类问题**
- **改配置不生效**（密码、密钥、user-data、辅助网卡、云盘挂载卸载、在线扩容）→ 见 **§7 控制台实例配置不生效类**

拿不准时：打开 [`phenomenon-domain.md`](phenomenon-domain.md) 参考 **典型现象** 进行分类。

## 3. 启动类问题

| 客户常说 | 主现象域文档 |
| --- | --- |
| 控制台实例状态已是 `Running`，但系统起不来 / 黑屏 / 卡在 grub·systemd·emergency | [`guestos-not-running.md`](guestos-not-running.md) |

## 4. 使用类问题

| 客户常说 | 主现象域文档 |
| --- | --- |
| 分区表损坏、挂载失败、文件系统错误、数据盘识别异常 | [`disk-fs-damaged.md`](disk-fs-damaged.md) |
| 时间跳变、NTP 异常 | [`clock-abnormal.md`](clock-abnormal.md) |
| 实例里上不了网 / curl 不了 100.100.100.200 | [`internal-network-failed.md`](internal-network-failed.md) |
| VNC 黑屏/连不上 | [`vnc-login-failed.md`](vnc-login-failed.md) |
| SSH 连不上 | [`ssh-login-failed.md`](ssh-login-failed.md) |

## 5. 宕机或夯机类问题

| 客户常说 | 主现象域文档 |
| --- | --- |
| 宕机、panic、自动重启 | [`system-crash.md`](system-crash.md) |
| 夯机、死机、卡住、soft lockup、不响应 | [`system-hang.md`](system-hang.md) |

## 6. 性能类问题

| 客户常说 | 主现象域文档 |
| --- | --- |
| CPU 占用高 | [`cpu-high.md`](cpu-high.md) |
| load 高 | [`load-high.md`](load-high.md) |
| 内存高、OOM | [`memory-oom.md`](memory-oom.md) |
| IOPS 异常高、磁盘使用率高 | [`disk-iops-high.md`](disk-iops-high.md) |
| 读写速率/IOPS 达不到规格预期 | [`disk-perf-unexpected.md`](disk-perf-unexpected.md) |
| 丢包 | [`network-packet-loss.md`](network-packet-loss.md) |
| 重传多 | [`retransmit-high.md`](retransmit-high.md) |
| 延迟大、RTT 高 | [`network-latency-high.md`](network-latency-high.md) |
| 带宽/流量打满或吞吐不达标 | [`network-perf-unexpected.md`](network-perf-unexpected.md) |

## 7. 控制台实例配置不生效类

| 客户常说 | 主现象域文档 |
| --- | --- |
| 控制台在线重置密码不生效 | [`password-reset-failed.md`](password-reset-failed.md) |
| user-data / 初始化脚本失败 | [`userdata-failed.md`](userdata-failed.md) |
| 绑定密钥对后仍用旧密钥 | [`ssh-keypair-not-applied.md`](ssh-keypair-not-applied.md) |
| 辅助网卡加了但实例里不通/无路由 | [`secondary-eni-unavailable.md`](secondary-eni-unavailable.md) |
| 挂载/卸载云盘后 OS 里看不到或卸不掉 | [`disk-attach-detach-failed.md`](disk-attach-detach-failed.md) |
| 控制台扩容后系统里容量没变 | [`disk-expand-failed.md`](disk-expand-failed.md) |

## 8. 不确定时

1. 在 [`phenomenon-domain.md`](phenomenon-domain.md) 中按 **典型现象**、**概念解释** 二次匹配。
