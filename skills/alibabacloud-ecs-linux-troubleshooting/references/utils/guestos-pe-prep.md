# 离线排查环境准备

本文描述将**问题实例的系统盘卸载并挂载到救援实例**完成离线排查的标准流程。以下操作包含写操作，进行操作前，**必须**告知用户将要执行的操作、可能的风险，并征得用户同意。

## 前置约束

- 需要用户提供**救援实例**，且**救援实例**与问题实例必须位于**同一可用区**（云盘仅可挂载到同 AZ 实例）。
- 问题实例的系统盘必须 `Portable=true`（`cloud_essd` / `cloud_auto` / `cloud_ssd` 等弹性云盘默认满足）。
- 问题实例必须先停机至 `Status=Stopped` 才能卸载系统盘。
- 推荐**救援实例**与问题实例同发行版同大版本（避免 chroot 后命令/库不兼容），且救援实例处于 `Running` 状态。
- 救援实例**云助手必须在线**（`CloudAssistantStatus=true`），否则后续 `RunCommand` 下发挂载脚本将失败。若为 `false`，需告知用户。

> ⚠️ 系统盘换挂在部分账号/region 受策略限制，若 `AttachDisk` 报 `InvalidDisk.SystemDiskAttach` 等错误，则恢复环境并终止后续流程。

## 准备步骤

1. **停机问题实例**，并轮询直到 `Status=Stopped`：
   ```bash
   # --StoppedMode=StopCharging：节省停机不收费；若实例规格不支持会自动 fallback 到 KeepCharging，不影响后续流程
   # --ForceStop=true：GuestOS 无响应时强制断电，防止卡停机
   aliyun ecs StopInstance --RegionId <region> --InstanceId <bad-i-xxx> --StoppedMode StopCharging --ForceStop true
   aliyun ecs DescribeInstances --RegionId <region> --InstanceIds '["<bad-i-xxx>"]' \
     | jq -r '.Instances.Instance[0].Status'
   ```

2. **定位问题实例系统盘 ID 与可用区**，并校验救援实例同 AZ：
   ```bash
   # 拿到系统盘 DiskId / ZoneId / Portable
   aliyun ecs DescribeDisks --RegionId <region> --InstanceId <bad-i-xxx> \
     | jq '.Disks.Disk[] | select(.Type=="system") | {DiskId, ZoneId, Category, Portable}'

   # 校验救援实例 ZoneId 一致
   aliyun ecs DescribeInstances --RegionId <region> --InstanceIds '["<rescue-i-xxx>"]' \
     | jq -r '.Instances.Instance[0].ZoneId'

   # 校验救援实例云助手在线（CloudAssistantStatus 必须为 true）
   aliyun ecs DescribeCloudAssistantStatus --RegionId <region> --InstanceId.1 <rescue-i-xxx> \
     | jq '.InstanceCloudAssistantStatusSet.InstanceCloudAssistantStatus[0] | {CloudAssistantStatus, CloudAssistantVersion}'
   ```

3. **从问题实例卸载系统盘**，轮询直到 `Disk.Status=Available`：
   ```bash
   aliyun ecs DetachDisk --InstanceId <bad-i-xxx> --DiskId <system-disk-id>
   aliyun ecs DescribeDisks --RegionId <region> --DiskIds '["<system-disk-id>"]' \
     | jq -r '.Disks.Disk[0].Status'
   ```

4. **挂载到救援实例**（救援实例可保持 Running，作为热挂载数据盘），轮询直到 `Disk.Status=In_use` 且 `Device` 字段已分配：
   ```bash
   aliyun ecs AttachDisk --InstanceId <rescue-i-xxx> --DiskId <system-disk-id>
   aliyun ecs DescribeDisks --RegionId <region> --DiskIds '["<system-disk-id>"]' \
     | jq '.Disks.Disk[0] | {Status, Type, Device, InstanceId}'
   ```

   挂上救援实例后，该盘 `Type` 会从 `system` 变为 `data`（属预期行为，回挂回原实例后会恢复为 `system`）。

5. **通过云助手在救援实例上挂载问题盘根文件系统**：阿里云 virtio-blk / NVMe 在 GuestOS 暴露的磁盘 `SERIAL` 字段即云盘 `DiskId` 去掉 `d-` 前缀（如 `d-bp19xdd4d6utncfv6r6f` ↔ SN `bp19xdd4d6utncfv6r6f`），据此可在救援实例内**精准定位新挂载的问题盘**，避免按 `/dev/vdb` 猜设备名。

   ```bash
   EXPECTED_SN=$(echo "<system-disk-id>" | sed 's/^d-//')

   CMD=$(cat <<'EOF' | sed "s/__EXPECTED_SN__/$EXPECTED_SN/"
   #!/bin/bash
   set -euo pipefail
   EXPECTED_SN='__EXPECTED_SN__'

   # 1) 定位问题盘的块设备：依次尝试 lsblk SERIAL → udevadm ID_SERIAL → nvme id-ctrl。
   DISK_DEV=""
   for d in $(lsblk -dn -o NAME); do
     sn=$(lsblk -dn -o NAME,SERIAL | awk -v n="$d" '$1==n && $2!="" {print $2; exit}')
     [ -z "$sn" ] && sn=$(udevadm info --query=property --name="/dev/$d" 2>/dev/null \
       | awk -F= '$1=="ID_SERIAL_SHORT"{print $2; exit}')
     [ -z "$sn" ] && [[ "$d" == nvme* ]] && command -v nvme >/dev/null 2>&1 \
       && sn=$(nvme id-ctrl "/dev/$d" 2>/dev/null | awk '/^sn[[:space:]]*:/{print $NF; exit}')
     if [ "$sn" = "$EXPECTED_SN" ]; then DISK_DEV="$d"; break; fi
   done
   [ -n "$DISK_DEV" ] || { echo "FATAL: no block device with serial=$EXPECTED_SN" >&2; exit 1; }
   echo "Matched block device: /dev/$DISK_DEV"

   # 2) 遍历该盘所有分区，三个标志文件任一存在即认定为根分区
   ROOT_PART=""
   PROBE=/tmp/_root_probe; mkdir -p "$PROBE"
   for p in $(lsblk -ln -o NAME,TYPE "/dev/$DISK_DEV" | awk '$2=="part"{print $1}'); do
     if mount -o ro "/dev/$p" "$PROBE" 2>/dev/null; then
       if [ -f "$PROBE/etc/fstab" ] || [ -f "$PROBE/etc/passwd" ] || [ -f "$PROBE/etc/shadow" ]; then
         ROOT_PART="$p"; umount "$PROBE"; break
       fi
       umount "$PROBE"
     fi
   done
   rmdir "$PROBE" 2>/dev/null || true
   [ -n "$ROOT_PART" ] || { echo "FATAL: no rootfs partition on /dev/$DISK_DEV" >&2; exit 1; }
   echo "Root partition: /dev/$ROOT_PART"

   # 3) 挂载根分区
   mkdir -p /mnt
   mount "/dev/$ROOT_PART" /mnt

   # 4) 解析 /mnt/etc/fstab，若存在 EFI 条目（挂载点为 /boot/efi 或类型为 vfat）则一并挂载。填选 EFI 条目可能是 UUID= / LABEL= / PARTUUID= / /dev/* 四种形式，需用 blkid 转换为设备路径
   EFI_LINE=$(awk '$1 !~ /^#/ && NF>=3 && ($2=="/boot/efi" || $2=="/boot/EFI" || $3=="vfat") {print; exit}' /mnt/etc/fstab || true)
   if [ -n "$EFI_LINE" ]; then
     EFI_SRC=$(echo "$EFI_LINE" | awk '{print $1}')
     EFI_MNT=$(echo "$EFI_LINE" | awk '{print $2}')
     case "$EFI_SRC" in
       UUID=*)     EFI_DEV=$(blkid -U "${EFI_SRC#UUID=}" || true) ;;
       LABEL=*)    EFI_DEV=$(blkid -L "${EFI_SRC#LABEL=}" || true) ;;
       PARTUUID=*) EFI_DEV=$(blkid -t "PARTUUID=${EFI_SRC#PARTUUID=}" -o device | head -n1 || true) ;;
       /dev/*)     EFI_DEV="$EFI_SRC" ;;
       *)          EFI_DEV="" ;;
     esac
     if [ -n "$EFI_DEV" ] && [ -b "$EFI_DEV" ]; then
       mkdir -p "/mnt$EFI_MNT"
       mount "$EFI_DEV" "/mnt$EFI_MNT"
       echo "Mounted EFI: $EFI_DEV -> /mnt$EFI_MNT"
     fi
   fi

   # 5) bind mount 运行时文件系统
   for d in dev dev/pts dev/shm proc sys run tmp; do
     mkdir -p "/mnt/$d"
     mount --bind "/$d" "/mnt/$d"
   done
   echo "OK: /dev/$ROOT_PART mounted at /mnt with bind mounts"
   EOF
   )

   INVOKE_ID=$(aliyun ecs RunCommand \
     --RegionId <region> \
     --Type RunShellScript \
     --InstanceId.1 <rescue-i-xxx> \
     --CommandContent "$CMD" \
     | jq -r '.InvokeId')

   # 轮询执行状态直到 InvokeRecordStatus=Finished
   aliyun ecs DescribeInvocations --RegionId <region> --InvokeId "$INVOKE_ID" \
     | jq '.Invocations.Invocation[0] | {InvokeStatus, InvocationStatus: .InvokeInstances.InvokeInstance[0].InvocationStatus}'

   # 拉取脚本输出（Output 为 base64，需解码）
   aliyun ecs DescribeInvocationResults --RegionId <region> --InvokeId "$INVOKE_ID" \
     | jq -r '.Invocation.InvocationResults.InvocationResult[0].Output' | base64 -d
   ```

   输出中出现 `OK: /dev/<root-part> mounted at /mnt with bind mounts` 即表示问题盘已成功挂载。

## 排查完成后的回滚

1. **通过云助手在救援实例上卸载问题盘的所有挂载点**。

   ```bash
   INVOKE_ID=$(aliyun ecs RunCommand \
     --RegionId <region> \
     --Type RunShellScript \
     --InstanceId.1 <rescue-i-xxx> \
     --CommandContent 'umount -R -l /mnt; ! findmnt /mnt' \
     --Timeout 60 \
     | jq -r '.InvokeId')

   aliyun ecs DescribeInvocationResults --RegionId <region> --InvokeId "$INVOKE_ID" \
     | jq -r '.Invocation.InvocationResults.InvocationResult[0].Output' | base64 -d
   ```

2. 从救援实例卸载该云盘，轮询直到 `Disk.Status=Available`：
   ```bash
   aliyun ecs DetachDisk --InstanceId <rescue-i-xxx> --DiskId <system-disk-id>
   aliyun ecs DescribeDisks --RegionId <region> --DiskIds '["<system-disk-id>"]' \
     | jq -r '.Disks.Disk[0].Status'
   ```

3. **重新挂回问题实例为系统盘**。API 限制：`AttachDisk` 挂到系统盘位 `/dev/xvda` 时必须同时提供 `--Password` 或 `--KeyPairName`（API 视为「重置系统盘」路径，文档层硬要求）；若**不指定 `--Device`** 则该盘会被自动挂为数据盘位（如 `/dev/xvdc`）、原实例启动时会因无系统盘报错。**两种凭据方式二选一，均需告知用户并征得同意**：

   - 方式 A：复用 region 内已有密钥对，公钥被注入盘内 `/root/.ssh/authorized_keys`，原密码仍可用：
     ```bash
     aliyun ecs AttachDisk --InstanceId <bad-i-xxx> --DiskId <system-disk-id> --Device /dev/xvda --KeyPairName <existing-keypair>
     ```
   - 方式 B：注入新密码，适用于原实例未绑定 KeyPair 且可接受覆写密码的场景。**密码必须用单引号包裹**，避免本地 shell 解析特殊字符（`&` `$` `*` `;` `!` 等；密码复杂度要求：8–30 位，包含大小写字母/数字/特殊字符中至少三类：
     ```bash
     aliyun ecs AttachDisk --InstanceId <bad-i-xxx> --DiskId <system-disk-id> --Device /dev/xvda --Password '<new-pwd>'
     ```

   轮询直到 `Disk.Status=In_use` 且 `Type=system` 、`Device=/dev/xvda`：
   ```bash
   aliyun ecs DescribeDisks --RegionId <region> --DiskIds '["<system-disk-id>"]' \
     | jq '.Disks.Disk[0] | {Status, Type, Device, InstanceId}'
   ```

4. 启动问题实例并校验实例是否已正常启动。：
   ```bash
   aliyun ecs StartInstance --RegionId <region> --InstanceId <bad-i-xxx>
   aliyun ecs DescribeInstances --RegionId <region> --InstanceIds '["<bad-i-xxx>"]' \
     | jq -r '.Instances.Instance[0].Status'
   ```

   **首选：以云助手心跳判定 GuestOS 已启动成功**。`DescribeCloudAssistantStatus` 返回 `CloudAssistantStatus=true` 即等价于 GuestOS 在线:
   ```bash
   aliyun ecs DescribeCloudAssistantStatus --RegionId <region> --InstanceId.1 <bad-i-xxx> \
     | jq '.InstanceCloudAssistantStatusSet.InstanceCloudAssistantStatus[0] | {CloudAssistantStatus, LastHeartbeatTime, LastInvokedTime}'
   ```

   如果实例仍未正常启动则告知用户。