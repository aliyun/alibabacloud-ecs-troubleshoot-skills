# cloud-init 与 User-Data 排查

GuestOS 内 cloud-init 及 User-Data 执行相关排查步骤。

## 1. 访问 meta-server

运行 `curl 100.100.100.200` 检查是否无法访问。无法访问参考 [无法访问 Metaserver](https://help.aliyun.com/zh/ecs/support/a-linux-instance-cannot-access-the-metaserver)。

## 2. User-Data 是否下发

运行 `curl http://100.100.100.200/latest/user-data` 检查是否存在内容。不存在则说明 User-Data 未配置。

## 3. User-Data 执行频率与信号量

User-Data 一般为 per-instance，执行后在 `/var/lib/cloud/instances/i-xxxx/sem/` 生成信号量，下次启动有信号量则不再执行。如果实例是创建后第二次启动则 User-Data 不会执行。

## 4. User-Data 内容格式

1. 首行需以 `#!` 开头，符合 User-Data 标准，参考文档：[使用自定义数据进行实例初始化](https://help.aliyun.com/zh/ecs/user-guide/overview-of-ecs-instance-user-data)。
2. 查看 `/var/lib/cloud/instance/scripts/` 目录下是否存在 `part-001` 等脚本文件，如果不存在，则说明 user-data 格式不正确。
3. 如果 `/var/log/cloud-init.log` 文件出现 `Unhandled non-multipart` 类似内容则表示格式错误。

## 5. cloud-init 配置与 fw_cfg 文件

1. 检查 cloud-init 是否安装并开机启动？不存在或没正确开机启动会导致 User-Data 功能异常。
2. 检查 `/sys/firmware/qemu_fw_cfg/by_name/etc/cloud-init/vendor-data/raw` 文件是否存在，检查 `/etc/cloud/cloud.cfg.d/aliyun_cloud.cfg` 文件是否软链到上述 raw 文件？不存在或没正确软链会导致 cloud-init 功能异常。

## 6. 脚本执行返回值

如果 `/var/log/cloud-init.log` 文件出现 `Failed running .../part-001 [1]` 类似内容，`[1]` 为脚本退出码，非 0 表示脚本本身问题，需用户自行检查脚本内容。
