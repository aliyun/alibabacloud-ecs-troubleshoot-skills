
# Shell 程序、启动配置及依赖库排查

GuestOS 内 Shell（/bin/sh 及用户登录 Shell）、Shell 启动配置文件及其依赖库（动态链接器、glibc）的排查步骤。

## 排查步骤

1. 检查 Shell 启动配置文件（如 `/etc/profile`、`/etc/bashrc`、`~/.bash_profile`、`~/.bashrc`、`~/.profile`）是否包含异常命令。
    - 重点关注会阻断登录或卡住非交互执行的内容，例如：`exit`、`logout`、`read`、`stty`、无限循环、长时间阻塞的外部命令、明显不存在的命令，或仅适用于交互式终端却未加条件判断的命令。
    - 如果仅个别账户受影响，优先检查对应用户家目录下的启动配置文件。
2. 使用 `less /etc/passwd` 命令查看系统账户文件。以 `:` 分隔，最后一列为登录账户自行配置的 Shell 程序地址。将其作为目标 `<SHELL>` 继续排查。
3. 执行 `<SHELL>` 命令，如果不能成功拉起则根据报错重点检查：动态链接器 `/lib64/ld-linux-x86-64.so.2`（RHEL/SUSE）或 `/lib/x86_64-linux-gnu/ld-*.so`（Debian）、基础 libc 动态库 `libc.so.6`。若路径或版本与正常实例不一致，可能 glibc 版本错乱，可参考正常实例修复符号链接或使用正确库文件覆盖进行修复。
