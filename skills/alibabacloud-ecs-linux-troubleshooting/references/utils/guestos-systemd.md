
# systemd 及系统服务阶段排查

以 systemd 为例。initramfs 中 init 通常软链接到 systemd，拉起 ramfs 内服务后由 initrd-switch-root 切到 rootfs，再执行根目录下的 systemd。从串口 “Switch Root” 或 “Welcome to ...” 到出现 login 提示为本阶段。

若此间卡住，串口日志和 VNC 屏幕常停在某一行（如因某 systemd 服务异常卡住）。

## 排查步骤

1. `chroot /mnt` 后通过 `journalctl` 命令分析启动日志。
2. 暂时禁用可疑服务（如重命名 `xxx.service` 为 `.bak`）。
3. 在 `/etc/systemd/system.conf` 中设置 `LogLevel=debug`，重启后查看详细日志。
4. 如 systemd 日志中发现服务异常，再针对具体的异常服务排查其 unit 与执行逻辑。
