# Storage Disk and Volume 诊断

## 目录

- [Storage Disk and Volume 诊断](#storage-disk-and-volume-诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: 磁盘状态与属性检查](#step-1-磁盘状态与属性检查)
    - [Step 2: 分区表与空间分配检查](#step-2-分区表与空间分配检查)
    - [Step 3: 分区状态与挂载检查](#step-3-分区状态与挂载检查)
    - [Step 4: 卷容量与 Cluster Size 限制检查](#step-4-卷容量与-cluster-size-限制检查)
    - [Step 5: 卷空间使用率检查](#step-5-卷空间使用率检查)
    - [Step 6: 文件系统健康检查](#step-6-文件系统健康检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：磁盘离线](#根因磁盘离线)
    - [根因：磁盘不可管理（动态/外部磁盘）](#根因磁盘不可管理动态外部磁盘)
    - [根因：磁盘 UniqueID 重复](#根因磁盘-uniqueid-重复)
    - [根因：磁盘大小与预期不符](#根因磁盘大小与预期不符)
    - [根因：磁盘未分区](#根因磁盘未分区)
    - [根因：磁盘存在未分配空间](#根因磁盘存在未分配空间)
    - [根因：MBR 磁盘超过 2TB 限制](#根因mbr-磁盘超过-2tb-限制)
    - [根因：分区类型无法识别](#根因分区类型无法识别)
    - [根因：分区未格式化](#根因分区未格式化)
    - [根因：分区未挂载（无盘符）](#根因分区未挂载无盘符)
    - [根因：最后一个分区无法扩展](#根因最后一个分区无法扩展)
    - [根因：Volume Cluster Size 限制扩容](#根因volume-cluster-size-限制扩容)
    - [根因：系统盘剩余空间不足（≤1GB）](#根因系统盘剩余空间不足1gb)
    - [根因：NTFS MFT 过大（≥10GB）](#根因ntfs-mft-过大10gb)
    - [根因：CHKDSK 报告文件系统错误](#根因chkdsk-报告文件系统错误)

## 功能说明

诊断 Windows 存储栈中从物理磁盘到逻辑卷的完整链路问题。覆盖磁盘在线/离线状态、SAN 策略、动态/外部磁盘、UniqueID 冲突、磁盘容量一致性、分区表类型与 2TB 限制、未分区/未分配空间、分区类型识别、格式化与挂载状态、分区扩展可行性、Cluster Size 容量限制、卷空间使用率、文件系统健康（MFT 膨胀、CHKDSK 错误）。覆盖 15 个已知问题项。


**输入**：用户问题描述（必选）、错误代码/事件ID/截图（可选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 新挂载的数据盘不可见、磁盘显示"脱机" | Step 1 (磁盘状态与属性) → Step 2 (分区表与空间分配) |
| 磁盘标记为"动态"或"外部" | Step 1 (磁盘状态与属性) |
| 多块云盘挂载后数据异常 | Step 1 (磁盘状态与属性) |
| 扩容后系统内容量未变 | Step 1 (磁盘状态与属性) → Step 2 (分区表与空间分配) |
| MBR 磁盘超过 2TB 无法使用 | Step 2 (分区表与空间分配) |
| 分区显示"未知"或提示需格式化 | Step 3 (分区状态与挂载) |
| 分区无盘符、在"我的电脑"中不可见 | Step 3 (分区状态与挂载) |
| 扩展卷失败 | Step 3 (分区状态与挂载) → Step 4 (卷容量与 Cluster Size 限制) |
| 系统盘空间不足、运行缓慢 | Step 5 (卷空间使用率) |
| 磁盘性能下降、MFT 过大 | Step 6 (文件系统健康) |
| 文件读写异常、文件丢失 | Step 6 (文件系统健康) |

## 诊断步骤

### Step 1: 磁盘状态与属性检查

**数据采集**：

> 采集目标：获取所有磁盘的在线/离线状态、健康状态、分区样式、UniqueID、容量、只读属性，以及当前 SAN 策略配置。**重要**：`Get-Disk` 基于 Storage Management API，可能隐藏动态磁盘（Dynamic Disk）或状态异常的磁盘，因此必须同时使用 `diskpart list disk` 获取底层物理磁盘全貌，并对两者做差异对比。

```powershell
# === Part A: PowerShell 视角 (可能遗漏动态/Foreign/脱机磁盘) ===
Get-Disk | Select-Object Number, FriendlyName, SerialNumber, UniqueId, OperationalStatus, HealthStatus, PartitionStyle, Size, AllocatedSize, NumberOfPartitions, IsOffline, IsReadOnly, IsBoot, IsSystem | Format-Table -AutoSize

# SAN policy (determines whether new disks come online automatically)
(Get-StorageSetting).NewDiskPolicy

# === Part B: Diskpart 视角 (底层物理磁盘全貌，含 Dyn 标记) ===
$dpOutput = "list disk" | diskpart
Write-Host "=== Diskpart Raw Output ==="
$dpOutput

# === Part C: 双重校验 - 找出 Diskpart 有但 Get-Disk 没有的磁盘 (漏网之鱼) ===
$psDisks = Get-Disk | Select-Object -ExpandProperty Number
$dpDiskNumbers = [regex]::Matches($dpOutput, 'Disk\s+(\d+)') | ForEach-Object { [int]$_.Groups[1].Value }
$missingDisks = $dpDiskNumbers | Where-Object { $psDisks -notcontains $_ }

if ($missingDisks) {
    Write-Host "!!! WARNING: Disks found in Diskpart but MISSING in Get-Disk (Likely Dynamic/Foreign/Offline): $missingDisks !!!"
    # 针对漏网之鱼，通过 WMI 获取底层信息
    foreach ($d in $missingDisks) {
        Get-CimInstance Win32_DiskDrive | Where-Object { $_.Index -eq $d } |
            Select-Object Index, Caption, Status, @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}}
    }
    # 获取这些磁盘上的卷信息
    Get-CimInstance Win32_Volume | Where-Object { $_.DriveLetter } |
        Select-Object DriveLetter, FileSystem, @{N='SizeGB';E={[math]::Round($_.Capacity/1GB,2)}}, @{N='FreeGB';E={[math]::Round($_.FreeSpace/1GB,2)}} | Format-Table -AutoSize
} else {
    Write-Host "INFO: No discrepancy between Diskpart and Get-Disk."
}

# === Part D: 检测 Diskpart 输出中是否存在 Dyn 标记 (动态磁盘) ===
$dynDisks = [regex]::Matches($dpOutput, 'Disk\s+(\d+).*?\bDyn\b') | ForEach-Object { [int]$_.Groups[1].Value }
if ($dynDisks) {
    Write-Host "!!! DYNAMIC DISKS DETECTED: $dynDisks !!!"
}
```

**分析思路**：

1. **双重校验对比（Part B/C vs Part A）**：
   - 正常：`diskpart list disk` 与 `Get-Disk` 列出的磁盘编号完全一致
   - 异常：`diskpart` 中存在但 `Get-Disk` 中缺失的磁盘 → 这些磁盘很可能为**动态磁盘（Dynamic Disk）**且处于 Foreign/脱机/元数据异常状态
   - 异常：`diskpart` 输出中某磁盘标记为 **Dyn**，但在 `Get-Disk` 中未显示或显示为 RAW/Unknown → **根因**：磁盘为动态磁盘且可能存在元数据异常/Foreign 状态，**严重程度**：Warning
   - 处理策略：对于 `Get-Disk` 找不到的磁盘，不可标记为"不存在"，应通过 Part C 中 `Win32_DiskDrive` 和 `Win32_Volume` 继续追踪其卷状态

2. 检查磁盘在线状态：
   - 正常：所有磁盘处于 Online 状态且未离线
   - 异常：磁盘处于 Offline 状态 → **根因**：磁盘离线，**严重程度**：Warning
   - 补充检查 SAN 策略：如果策略不是 OnlineAll，新挂载的磁盘会默认离线

3. 检查磁盘是否可管理：
   - 正常：分区样式为 GPT 或 MBR（基本磁盘）
   - 异常：磁盘类型标记为动态磁盘或外部磁盘 → **根因**：磁盘不可管理（动态/外部磁盘），**严重程度**：Warning

4. 检查 UniqueID 是否重复：
   - 正常：所有磁盘的 UniqueID 互不相同
   - 异常：两块或更多磁盘共享相同的 UniqueID → **根因**：磁盘 UniqueID 重复，**严重程度**：Critical
   - 说明：UniqueID 重复在阿里云 ECS 场景中常见于旧版 VirtIO 驱动（驱动未正确透传磁盘序列号），会导致 Windows 在磁盘枚举时产生冲突，严重时可能引发数据覆盖

5. 检查磁盘大小是否与预期一致（需用户提供云平台侧的磁盘规格作为参考）：
   - 正常：系统内识别的磁盘大小与云平台控制台一致
   - 异常：系统内大小小于云平台大小 → **根因**：磁盘大小与预期不符，**严重程度**：Warning
   - 说明：扩容后系统未自动感知新大小时，需通过 `Update-Disk` 或 diskpart `rescan` 刷新

如果磁盘完全无法识别或设备管理器中存储控制器有异常标记，参见 → [storage-hardware.md](references/storage-hardware.md)

### Step 2: 分区表与空间分配检查

**数据采集**：

> 采集目标：获取各磁盘的分区样式、已分配空间、未分配空间，以及全部分区的基本信息

```powershell
# Disk-level space overview
Get-Disk | Select-Object Number, SerialNumber, PartitionStyle, Size, AllocatedSize, NumberOfPartitions, @{N='UnallocatedGB';E={[math]::Round(($_.Size - $_.AllocatedSize)/1GB, 2)}} | Format-Table -AutoSize

# Partition details
Get-Partition | Select-Object DiskNumber, PartitionNumber, Type, Size, Offset, DriveLetter, IsActive, IsBoot, IsSystem, GptType, MbrType | Format-Table -AutoSize
```

**分析思路**：

1. 检查磁盘是否已分区：
   - 正常：磁盘至少有一个分区
   - 异常：分区数量为 0，且分区样式为 RAW → **根因**：磁盘未分区，**严重程度**：Warning
   - 说明：新挂载的数据盘需要先初始化（选择 GPT 或 MBR）再创建分区

2. 检查是否存在大量未分配空间：
   - 正常：已分配空间接近磁盘总大小（差值小于 1GB）
   - 异常：存在超过 1GB 的未分配空间 → **根因**：磁盘存在未分配空间，**严重程度**：Warning
   - 说明：常见于云盘扩容后未扩展分区，或创建分区时未使用全部空间

3. 检查 MBR 磁盘大小限制：
   - 正常：MBR 磁盘总大小小于 2TB（2199023255552 字节）
   - 异常：MBR 磁盘大于或等于 2TB → **根因**：MBR 磁盘超过 2TB 限制，**严重程度**：Warning
   - 说明：MBR 分区表使用 32 位 LBA 寻址，最大支持 2TB。超出部分无法被分区使用，需转换为 GPT

### Step 3: 分区状态与挂载检查

**数据采集**：

> 采集目标：获取所有分区的类型识别状态、文件系统格式、盘符挂载情况、每个分区的最大可扩展大小，以及系统的自动挂载（automount）状态

```powershell
# Partition and corresponding volume information
Get-Partition | ForEach-Object {
    $vol = $_ | Get-Volume -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        DiskNumber      = $_.DiskNumber
        PartitionNumber = $_.PartitionNumber
        Type            = $_.Type
        Size            = $_.Size
        Offset          = $_.Offset
        DriveLetter     = $_.DriveLetter
        FileSystem      = if ($vol) { $vol.FileSystem } else { 'N/A' }
        FileSystemLabel = if ($vol) { $vol.FileSystemLabel } else { 'N/A' }
        AllocationUnitSize = if ($vol) { $vol.AllocationUnitSize } else { 0 }
    }
}

# Expansion capability of the last partition on each disk
Get-Disk | ForEach-Object {
    $disk = $_
    $lastPart = Get-Partition -DiskNumber $disk.Number -ErrorAction SilentlyContinue | Sort-Object Offset | Select-Object -Last 1
    if ($lastPart) {
        $supported = $lastPart | Get-PartitionSupportedSize -ErrorAction SilentlyContinue
        [PSCustomObject]@{
            DiskNumber      = $disk.Number
            PartitionNumber = $lastPart.PartitionNumber
            CurrentSizeGB   = [math]::Round($lastPart.Size/1GB, 2)
            MaxSizeGB       = if ($supported) { [math]::Round($supported.SizeMax/1GB, 2) } else { 'N/A' }
            Expandable      = if ($supported) { $supported.SizeMax -gt $lastPart.Size } else { $false }
        }
    }
}

# Auto-mount status (when automount is disabled, new volumes will not be assigned drive letters automatically)
"automount" | diskpart
```

> **Localization note**: diskpart 输出语言取决于操作系统语言环境。中文系统输出 "已启用"/"已禁用"，英文系统输出 "Enabled"/"Disabled"。分析时需根据实际系统语言来解释输出。

**分析思路**：

1. 检查分区类型是否可识别：
   - 正常：分区类型为 Basic、IFS、系统保留等已知类型
   - 异常：分区类型为 Unknown → **根因**：分区类型无法识别，**严重程度**：Warning
   - 说明：常见原因是分区表损坏，或分区使用了 Windows 不支持的文件系统格式（如 Linux ext4/xfs），后者属于正常行为

2. 检查分区是否已格式化：
   - 正常：分区关联的卷具有 NTFS/ReFS/FAT32 等有效文件系统
   - 异常：无文件系统信息 → **根因**：分区未格式化，**严重程度**：Warning

3. 检查数据分区是否已挂载盘符：
   - 正常：非系统保留分区（非 MSR、非 EFI、非 Recovery）具有盘符
   - 异常：数据分区无盘符 → **根因**：分区未挂载（无盘符），**严重程度**：Warning
   - 说明：系统保留分区、MSR 分区、EFI 分区无盘符是正常设计
   - 如果无盘符且 automount 显示已禁用，盘符丢失的根因是自动挂载被关闭，每次重启后都会复现
   - 如果盘符存在但用户在文件资源管理器中看不到，可能是组策略隐藏了驱动器，参见 → [system-gpo.md](references/system-gpo.md)

4. 检查最后一个分区是否可扩展：
   - 正常：最大可扩展大小大于当前大小
   - 异常：不可扩展 → **根因**：最后一个分区无法扩展，**严重程度**：Warning
   - 常见原因：分区后方存在恢复分区（Recovery Partition）阻挡、文件系统不支持在线扩展（如 FAT32）、磁盘为动态磁盘

### Step 4: 卷容量与 Cluster Size 限制检查

**数据采集**：

> 采集目标：获取所有已格式化卷的 Cluster Size（分配单元大小）和当前容量，评估是否存在因 Cluster Size 导致的扩容上限

```powershell
Get-Volume | Where-Object { $_.DriveLetter -and $_.FileSystem } | Select-Object DriveLetter, FileSystem, AllocationUnitSize, Size, SizeRemaining, @{N='SizeGB';E={[math]::Round($_.Size/1GB, 2)}}, @{N='ClusterSizeKB';E={$_.AllocationUnitSize/1KB}}
```

**分析思路**：

1. 检查 NTFS 卷是否接近 Cluster Size 对应的最大容量限制：
   - NTFS 最大卷容量 = Cluster Size × 2³²（即 Cluster Size × 4,294,967,296）。例如默认 4KB Cluster Size 对应最大卷容量 16TB

   - 正常：当前卷大小远小于 Cluster Size 对应的最大容量
   - 异常：当前卷大小已接近或达到上限，继续扩容将失败 → **根因**：Volume Cluster Size 限制扩容，**严重程度**：Warning
   - 说明：Windows 格式化时默认使用 4KB Cluster Size，对于大容量云盘（>16TB）扩容场景需要注意此限制。Cluster Size 无法在线修改，需备份数据后重新格式化

### Step 5: 卷空间使用率检查

**数据采集**：

> 采集目标：获取所有已挂载卷的容量、剩余空间和使用率，重点关注系统盘

```powershell
Get-Volume | Where-Object { $_.DriveLetter } | Select-Object DriveLetter, FileSystem, FileSystemLabel, @{N='SizeGB';E={[math]::Round($_.Size/1GB, 2)}}, @{N='FreeGB';E={[math]::Round($_.SizeRemaining/1GB, 2)}}, @{N='UsedPercent';E={if($_.Size -gt 0){[math]::Round(($_.Size - $_.SizeRemaining)/$_.Size * 100, 1)}else{0}}}
```

**分析思路**：

1. 检查系统盘剩余空间：
   - 正常：系统盘（通常为 C:）剩余空间大于 1GB
   - 异常：剩余空间小于等于 1GB → **根因**：系统盘剩余空间不足（≤1GB），**严重程度**：Warning
   - 说明：系统盘空间不足会导致更新安装失败、临时文件无法写入、系统日志停止记录，严重时可能导致系统无法正常运行

### Step 6: 文件系统健康检查

**数据采集**：

> 采集目标：获取所有 NTFS 卷的 MFT（Master File Table）大小，以及最近 7 天内的 CHKDSK/Wininit 相关事件日志

```powershell
# NTFS volume filesystem details (including MFT size, field names vary by system language)
Get-Volume | Where-Object { $_.FileSystem -eq 'NTFS' -and $_.DriveLetter } | ForEach-Object {
    Write-Output "=== $($_.DriveLetter): ==="
    fsutil fsinfo ntfsinfo "$($_.DriveLetter):"
}

# CHKDSK / Wininit events in the last 7 days
Get-WinEvent -FilterHashtable @{LogName='Application'; StartTime=(Get-Date).AddDays(-7)} -MaxEvents 20 -ErrorAction SilentlyContinue | Where-Object { $_.ProviderName -in 'Chkdsk','Wininit' } | Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize

# Ntfs warning/error events in the last 7 days (filesystem corruption indicator)
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2,3; StartTime=(Get-Date).AddDays(-7)} -MaxEvents 20 -ErrorAction SilentlyContinue | Where-Object { $_.ProviderName -eq 'Ntfs' } | Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查 MFT 大小：
   - 正常：MFT 大小小于 10GB
   - 异常：MFT 大小大于等于 10GB → **根因**：NTFS MFT 过大（≥10GB），**严重程度**：Warning
   - 说明：MFT 是 NTFS 的核心元数据结构，每个文件/目录对应一条 MFT 记录。当卷上曾存在大量小文件时 MFT 会持续增长，文件删除后 MFT 空间不会自动回收。表现为磁盘可用空间统计异常（文件总大小远小于已用空间）

2. 检查 CHKDSK 和 Ntfs 事件日志：
   - 正常：无 CHKDSK 错误事件，无 Ntfs 警告/错误事件
   - 异常：存在文件系统错误事件 → **根因**：CHKDSK 报告文件系统错误，**严重程度**：Warning
   - 说明：Ntfs 事件日志中的 Event ID 55（文件系统结构损坏）、Event ID 98（卷需要联机扫描）是常见的文件系统损坏指标，需要根据事件详情判断是否需要执行 `chkdsk /f` 修复

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 1 发现磁盘完全不可识别或存储控制器异常 | → [storage-hardware.md](references/storage-hardware.md) |
| 条件跳转 | Step 1 UniqueID 重复且怀疑 VirtIO 驱动版本问题 | → [cloud-driver.md](references/cloud-driver.md) |
| 条件跳转 | Step 3 分区扩展失败且磁盘为动态磁盘或驱动异常 | → [storage-hardware.md](references/storage-hardware.md) |
| 链式后继 | 本文件所有步骤执行完毕，未确认根因 | → [storage-hardware.md](references/storage-hardware.md) |

## 修复建议

### 根因：磁盘离线

**修复操作**：

```powershell
# Set offline disk to online (replace $diskNum with actual disk number)
$diskNum = <DiskNumber>
Set-Disk -Number $diskNum -IsOffline $false

# If disk is read-only, also clear read-only flag
Set-Disk -Number $diskNum -IsReadOnly $false

# Change SAN policy to OnlineAll to prevent newly attached disks from staying offline
Set-StorageSetting -NewDiskPolicy OnlineAll
```

**验证方法**：

```powershell
Get-Disk -Number $diskNum | Select-Object Number, OperationalStatus, IsOffline, IsReadOnly | Format-Table -AutoSize
(Get-StorageSetting).NewDiskPolicy
```

预期结果：OperationalStatus = Online，IsOffline = False，NewDiskPolicy = OnlineAll

**风险说明**：如果多块磁盘存在 UniqueID 重复，同时上线可能导致 Windows 将不同磁盘识别为同一设备，存在数据损坏风险，需先解决 UniqueID 问题

---

### 根因：磁盘不可管理（动态/外部磁盘）

**说明**：

动态磁盘或外部磁盘无法通过标准 PowerShell 命令（`Get-Disk`/`Set-Disk`）管理，需通过磁盘管理器（diskmgmt.msc）或 diskpart 操作：
- **动态磁盘**：如果不需要动态磁盘功能（跨区卷、镜像卷），建议转换为基本磁盘（需先删除所有卷，会丢失数据）
- **外部磁盘（Foreign）**：在磁盘管理器中右键选择"导入外部磁盘"，或使用 diskpart 导入：

```powershell
# 导入外部动态磁盘 (replace X with actual disk number)
"select disk X", "online", "import" | diskpart
```

- **验证导入结果**：导入后重新执行 Step 1 的双重校验，确认 `Get-Disk` 已能识别该磁盘


---

### 根因：磁盘 UniqueID 重复

**修复操作**：

```powershell
# Confirm UniqueID duplication
Get-Disk | Select-Object Number, SerialNumber, UniqueId | Group-Object UniqueId | Where-Object { $_.Count -gt 1 }
```

根因通常是旧版 VirtIO 驱动未正确透传磁盘序列号。修复方式是更新 VirtIO 驱动到最新版本。

**验证方法**：

```powershell
# Verify after driver update and reboot
Get-Disk | Select-Object Number, UniqueId | Group-Object UniqueId | Where-Object { $_.Count -gt 1 }
```

预期结果：无输出（无重复 UniqueID）

**风险说明**：更新 VirtIO 驱动需要重启实例，建议在维护窗口操作


如果需要检查 VirtIO 驱动版本和状态，参见 → [cloud-driver.md](references/cloud-driver.md)

---

### 根因：磁盘大小与预期不符

**修复操作**：

```powershell
# Rescan disk to refresh disk size information
$diskNum = <DiskNumber>
Update-Disk -Number $diskNum

# Verify refresh result
Get-Disk -Number $diskNum | Select-Object Number, @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}}
```

如果 `Update-Disk` 无效，可尝试 diskpart 方式：

```powershell
"rescan" | diskpart
```

**验证方法**：

```powershell
Get-Disk -Number $diskNum | Select-Object Number, Size, @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}}
```

预期结果：磁盘大小与云平台控制台一致

**风险说明**：重新扫描为只读操作，不影响已有数据和分区


---

### 根因：磁盘未分区

**修复操作**：

```powershell
# Initialize disk and create partition (GPT partition style recommended)
$diskNum = <DiskNumber>
Initialize-Disk -Number $diskNum -PartitionStyle GPT

# Create partition and format as NTFS
New-Partition -DiskNumber $diskNum -UseMaximumSize -AssignDriveLetter | Format-Volume -FileSystem NTFS -Confirm:$false
```

**验证方法**：

```powershell
Get-Partition -DiskNumber $diskNum | Select-Object PartitionNumber, DriveLetter, @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}}
```

预期结果：分区已创建，且具有盘符

**风险说明**：初始化磁盘会清除磁盘上所有数据，操作前务必确认磁盘是新盘或无重要数据


---

### 根因：磁盘存在未分配空间

**修复操作**：

```powershell
# Expand the last partition to maximum available space
$diskNum = <DiskNumber>
$partNum = <PartitionNumber>
$maxSize = (Get-PartitionSupportedSize -DiskNumber $diskNum -PartitionNumber $partNum).SizeMax
Resize-Partition -DiskNumber $diskNum -PartitionNumber $partNum -Size $maxSize
```

**验证方法**：

```powershell
Get-Disk -Number $diskNum | Select-Object Number, Size, AllocatedSize, @{N='UnallocatedGB';E={[math]::Round(($_.Size - $_.AllocatedSize)/1GB, 2)}}
```

预期结果：未分配空间接近 0

**风险说明**：扩展分区为在线操作，通常安全，但建议提前创建快照备份


---

### 根因：MBR 磁盘超过 2TB 限制

**修复操作**：

```powershell
# MBR to GPT conversion (only data disks support online conversion, system disk does not)
# Windows Server 2019+ can use mbr2gpt
mbr2gpt /convert /disk:<DiskNumber> /allowFullOS
```

**验证方法**：

```powershell
Get-Disk -Number <磁盘编号> | Select-Object Number, PartitionStyle, @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}}
```

预期结果：PartitionStyle = GPT

**风险说明**：MBR 转 GPT 操作存在数据丢失风险，操作前必须备份数据。系统盘转换需要特殊流程且有额外限制条件


---

### 根因：分区类型无法识别

**说明**：

分区类型无法识别的常见原因：
- **Linux 分区**：Windows 不支持原生读写 ext4/xfs/btrfs 等 Linux 文件系统，分区显示为 Unknown 是正常行为
- **分区表损坏**：如果原本的 Windows 分区变成 Unknown，可能是分区表元数据损坏，需要使用数据恢复工具


---

### 根因：分区未格式化

**修复操作**：

```powershell
# Format partition (will erase all data on the partition)
Format-Volume -DriveLetter <DriveLetter> -FileSystem NTFS -Confirm:$false
```

**验证方法**：

```powershell
Get-Volume -DriveLetter <盘符> | Select-Object DriveLetter, FileSystem, @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}}
```

预期结果：FileSystem = NTFS

**风险说明**：格式化会清除分区上所有数据，操作前需确认分区内无重要数据

---

### 根因：分区未挂载（无盘符）

**修复操作**：

```powershell
# 1. Assign drive letter to partition
$diskNum = <DiskNumber>
$partNum = <PartitionNumber>
$letter = '<DriveLetter>'
Set-Partition -DiskNumber $diskNum -PartitionNumber $partNum -NewDriveLetter $letter

# 2. If automount is disabled, enable auto-mount (prevent drive letter loss after reboot)
"automount enable" | diskpart
```

**验证方法**：

```powershell
Get-Partition -DiskNumber $diskNum -PartitionNumber $partNum | Select-Object DriveLetter
"automount" | diskpart
```

预期结果：DriveLetter 为指定盘符，automount 显示已启用

**风险说明**：分配盘符和启用 automount 均无风险，但需确保目标盘符未被占用


---

### 根因：最后一个分区无法扩展

**说明**：

最后一个分区无法扩展的常见原因及处理方式：

- **分区后方有恢复分区**：恢复分区（Recovery Partition）位于磁盘末尾，阻止了数据分区向后扩展。需要先删除或移动恢复分区，再扩展目标分区
- **文件系统不支持在线扩展**：FAT32 不支持在线扩展，需先转换为 NTFS（`convert <盘符>: /fs:ntfs`）
- **磁盘为动态磁盘**：动态磁盘的卷扩展行为不同，需通过磁盘管理器操作

```powershell
# View partition layout to confirm if recovery partition is blocking expansion
Get-Partition -DiskNumber <DiskNumber> | Sort-Object Offset | Select-Object PartitionNumber, Type, @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}}, Offset
```


---

### 根因：Volume Cluster Size 限制扩容

**说明**：

NTFS 卷的最大容量受 Cluster Size（分配单元大小）限制：最大卷容量 = Cluster Size × 2³²。Windows 格式化时默认使用 4KB Cluster Size，对应最大卷容量 16TB。

当云盘扩容后总容量超过 Cluster Size 对应的上限时，扩展卷操作会失败。

```powershell
# Confirm current volume Cluster Size
Get-Volume -DriveLetter <DriveLetter> | Select-Object DriveLetter, FileSystem, AllocationUnitSize, @{N='ClusterSizeKB';E={$_.AllocationUnitSize/1KB}}, @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}}
```

**解决方案**：Cluster Size 无法在线修改。需要备份数据 → 重新格式化为更大的 Cluster Size → 恢复数据。

**风险说明**：更大的 Cluster Size 会增加小文件的存储空间浪费（每个文件至少占用一个 Cluster）


---

### 根因：系统盘剩余空间不足（≤1GB）

**修复操作**：

```powershell
# 1. Check temporary file usage
Get-ChildItem -Path C:\Windows\Temp -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum
Get-ChildItem -Path $env:TEMP -Recurse -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum

# 2. Clean Windows Update cache
Stop-Service -Name wuauserv -Force -ErrorAction SilentlyContinue
Remove-Item -Path "C:\Windows\SoftwareDistribution\Download\*" -Recurse -Force -ErrorAction SilentlyContinue
Start-Service -Name wuauserv

# 3. Run disk cleanup (interactive)
cleanmgr /d C
```

**验证方法**：

```powershell
Get-Volume -DriveLetter C | Select-Object DriveLetter, @{N='FreeGB';E={[math]::Round($_.SizeRemaining/1GB, 2)}}
```

预期结果：剩余空间大于 1GB

**风险说明**：清理临时文件通常安全。清理 Windows Update 缓存会导致已下载但未安装的更新需要重新下载


---

### 根因：NTFS MFT 过大（≥10GB）

**说明**：

MFT（Master File Table）是 NTFS 文件系统的核心元数据结构，每个文件和目录在 MFT 中对应一条记录。当卷上曾经创建大量小文件时，MFT 会持续增长。即使文件被删除，MFT 占用的磁盘空间也不会自动回收。

MFT 过大的影响：
- 磁盘空间统计异常（实际文件总大小远小于卷已用空间）
- 文件操作性能下降

目前没有安全的在线 MFT 收缩方法。可选方案：
1. 备份数据 → 格式化卷 → 恢复数据（MFT 会以合理大小重建）
2. 如果对业务影响不大，可以忽略此告警


---

### 根因：CHKDSK 报告文件系统错误

**修复操作**：

```powershell
# Step 1: Read-only scan to assess filesystem state (does not modify any data)
chkdsk <DriveLetter>: /scan

# Step 2: If read-only scan confirms errors, perform repair (requires exclusive volume access)
# Data disks can be repaired directly; system disk repair will be scheduled at next reboot
# chkdsk <DriveLetter>: /f
```

**验证方法**：

```powershell
Get-WinEvent -FilterHashtable @{LogName='Application'; StartTime=(Get-Date).AddDays(-1)} -MaxEvents 5 -ErrorAction SilentlyContinue | Where-Object { $_.ProviderName -in 'Chkdsk','Wininit' } | Select-Object TimeCreated, Id, Message
```

预期结果：CHKDSK 完成事件中报告无错误

**风险说明**：`chkdsk /scan` 为只读操作，安全无风险。`chkdsk /f` 需要独占卷访问权，系统盘会在下次重启时执行；修复过程中如果断电可能导致数据丢失

