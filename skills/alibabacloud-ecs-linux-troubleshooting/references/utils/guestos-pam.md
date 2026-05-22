
# PAM 模块及配置排查

本文描述 GuestOS 内 PAM（Pluggable Authentication Modules）模块及配置的排查步骤

## 排查步骤

1. 对比 `/etc/pam.d/`下文件与同镜像正常实例，是否有差异？
2. 对比 `/usr/lib64/security/`（RHEL/SUSE）或 `/usr/lib/x86_64-linux-gnu/security/`（Debian）下 PAM 模块动态库的文件大小、修改时间或哈希与正常实例，是否有差异？
3. 对上述模块使用 `ldd <模块路径>` 查看依赖动态库，是否有缺失？