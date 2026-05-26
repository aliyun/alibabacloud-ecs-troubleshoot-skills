# Security BitLocker 诊断

## 功能说明

诊断 Windows BitLocker 加密状态和恢复密钥状态。覆盖 2 个已知问题项。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 系统启动时提示输入恢复密钥 | Step 1 (BitLocker 加密状态) |
| 无法访问加密磁盘数据 | Step 2 (恢复密钥状态) |

## 诊断步骤

### Step 1: 检查 BitLocker 加密状态

**数据采集**：

> 采集目标：获取所有卷的 BitLocker 加密状态、保护方式和解锁状态

```powershell
# Check BitLocker service status
$bdeService = Get-Service -Name BDESVC -ErrorAction SilentlyContinue
if ($bdeService) {
    Write-Host "BitLocker Service (BDESVC): Status=$($bdeService.Status), StartType=$($bdeService.StartType)"
} else {
    Write-Host "BitLocker Service (BDESVC): Not installed"
}
# Get BitLocker encryption status
try {
    $volumes = Get-CimInstance -Namespace 'Root\CIMV2\Security\MicrosoftVolumeEncryption' -ClassName 'Win32_EncryptableVolume' -ErrorAction Stop
    foreach ($vol in $volumes) {
        Write-Host "`nVolume: $($vol.DriveLetter)"
        Write-Host "  ProtectionStatus: $($vol.ProtectionStatus) (0=Off, 1=On, 2=Unknown)"
        Write-Host "  ConversionStatus: $($vol.ConversionStatus) (0=FullyDecrypted, 1=FullyEncrypted, 2=EncryptionInProgress, 3=DecryptionInProgress, 4=EncryptionPaused, 5=DecryptionPaused)"
        Write-Host "  EncryptionMethod: $($vol.EncryptionMethod)"
    }
} catch {
    Write-Host "BitLocker WMI namespace not available: $($_.Exception.Message)"
    Write-Host "BitLocker feature may not be installed on this edition"
}
```

**分析思路**：

1. 检查 BitLocker 服务是否存在：
   - 如果服务不存在，BitLocker 功能未安装，本步骤无需继续
2. 检查卷加密状态：
   - 正常：ProtectionStatus = 0（未加密）或 ProtectionStatus = 1 且 ConversionStatus = 1（已加密）
   - 异常：ConversionStatus = 2/4（加密进行中/暂停） → **根因**：BitLocker 加密未完成，磁盘 I/O 性能可能受影响，**严重程度**：Warning
   - 异常：ProtectionStatus = 1 且系统要求输入恢复密钥 → **根因**：BitLocker 触发恢复模式，系统无法正常启动，**严重程度**：Critical

### Step 2: 检查恢复密钥状态

**数据采集**：

> 采集目标：检查 BitLocker 恢复密钥保护器是否存在，确认恢复密钥可用性

```powershell
try {
    $volumes = Get-CimInstance -Namespace 'Root\CIMV2\Security\MicrosoftVolumeEncryption' -ClassName 'Win32_EncryptableVolume' -ErrorAction Stop
    foreach ($vol in $volumes) {
        if ($vol.ProtectionStatus -eq 1) {
            Write-Host "Volume $($vol.DriveLetter) is BitLocker encrypted"
            # Get key protector types
            $protectors = $vol | Invoke-CimMethod -MethodName 'GetKeyProtectors' -ErrorAction SilentlyContinue
            if ($protectors -and $protectors.VolumeKeyProtectorID) {
                Write-Host "  Key Protectors count: $($protectors.VolumeKeyProtectorID.Count)"
                foreach ($id in $protectors.VolumeKeyProtectorID) {
                    $type = ($vol | Invoke-CimMethod -MethodName 'GetKeyProtectorType' -Arguments @{VolumeKeyProtectorID=$id}).KeyProtectorType
                    $typeNames = @{0='Unknown';1='TPM';2='ExternalKey';3='NumericalPassword';4='TPMAndPIN';5='TPMAndStartupKey';6='TPMAndPINAndStartupKey';7='PublicKey';8='Passphrase';9='TPMCertificate';10='CryptoAPI_NextGen'}
                    $typeName = if ($typeNames.ContainsKey([int]$type)) { $typeNames[[int]$type] } else { "Type_$type" }
                    Write-Host "  Protector: ID=$id, Type=$typeName"
                }
            } else {
                Write-Host "  WARNING: No key protectors found"
            }
        }
    }
} catch {
    Write-Host "Cannot query BitLocker: $($_.Exception.Message)"
}
# Check BitLocker related events
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2,3} -MaxEvents 5 -ErrorAction SilentlyContinue |
    Where-Object { $_.ProviderName -eq 'Microsoft-Windows-BitLocker-Driver' } |
    Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查密钥保护器：
   - 正常：存在至少一个密钥保护器（如 TPM、NumericalPassword 等）
   - 异常：无密钥保护器 → **根因**：BitLocker 加密卷无恢复密钥，一旦触发恢复模式将无法解锁，**严重程度**：Critical
2. 检查 BitLocker 事件日志：
   - 关注错误和警告事件，特别是 TPM 通信失败、密钥保护器失效等
   - 发现此类事件 → **根因**：BitLocker 运行时错误，可能导致恢复模式触发，**严重程度**：Warning

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | BitLocker 导致磁盘 I/O 性能下降 | → [performance-slow.md](references/performance-slow.md) |
| 链式后继 | 本文件未确认根因 | → [storage-disk.md](references/storage-disk.md) |

## 修复建议

### 根因: BitLocker 触发恢复模式

**修复操作**：

```powershell
# If recovery key is available, use it to unlock
manage-bde -unlock C: -RecoveryPassword <48DigitRecoveryKey>
# Re-enable normal protection after unlock
manage-bde -protectors -enable C:
```

**验证方法**：

```powershell
manage-bde -status C:
```

预期结果：Protection Status 显示为 "Protection On"，Lock Status 为 "Unlocked"

**风险说明**：如果无法获取恢复密钥，将无法解锁加密卷，磁盘数据不可访问。建议提前备份恢复密钥到 AD 或微软账户。

### 根因: BitLocker 加密未完成

**修复操作**：

```powershell
# 恢复暂停的加密过程
manage-bde -resume C:
```

**验证方法**：

```powershell
manage-bde -status C:
```

预期结果：Conversion Status 显示 "Fully Encrypted" 或正在进行加密

**风险说明**：加密过程中磁盘 I/O 会有额外开销，建议在业务低峰期执行。
