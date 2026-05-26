# RDP 证书诊断

## 目录

- [RDP 证书诊断](#rdp-证书诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: RDP 证书来源与状态检查](#step-1-rdp-证书来源与状态检查)
    - [Step 2: MachineKeys 与 TLS 私钥权限检查](#step-2-machinekeys-与-tls-私钥权限检查)
    - [Step 3: 系统盘根目录权限检查](#step-3-系统盘根目录权限检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：RDP 证书过期](#根因rdp-证书过期)
    - [根因：MachineKeys 权限异常](#根因machinekeys-权限异常)
    - [根因：RDP 证书丢失](#根因rdp-证书丢失)
    - [根因：TLS 私钥访问被拒绝](#根因tls-私钥访问被拒绝)
    - [根因：系统盘根目录权限异常](#根因系统盘根目录权限异常)

## 功能说明

诊断远程桌面证书相关的问题。覆盖 RDP 证书来源与状态（逐 WinStation 检查 SSLCertificateSHA1Hash / SelfSignedCertificate 回退、有效期、私钥可访问性）、MachineKeys 目录与 TLS 私钥文件权限（关联证书 HasPrivateKey 状态）、系统盘根目录权限等 3 个诊断步骤。

**输入**：用户问题描述（必选）、RDP 证书警告或 Internal error（可选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| RDP 证书警告、证书不受信任、证书过期 | Step 1 (证书来源与状态) |
| Internal error、证书无法更新 | Step 2 (MachineKeys 与 TLS 私钥权限) → Step 1 (证书来源与状态) |
| TLS 协商失败、私钥访问拒绝、HasPrivateKey = false | Step 1 (证书来源与状态) → Step 2 (MachineKeys 与 TLS 私钥权限) |
| RDP 连接失败且其他步骤未发现问题 | Step 3 (系统盘根目录权限) |

## 诊断步骤

### Step 1: RDP 证书来源与状态检查

**数据采集**：

> 采集目标：读取全局默认自签名证书定义，枚举所有 WinStation 的证书配置（SSLCertificateSHA1Hash / SelfSignedCertificate 回退），获取证书详情（有效期、颁发者、私钥状态）

```powershell
# 1. Read global self-signed certificate info (SelfSignedCertificate / SelfSignedCertStore under WinStations key)
$winStationsPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations"
$selfSignedThumb = $null
$wsProps = Get-ItemProperty -Path $winStationsPath -ErrorAction SilentlyContinue
$selfSignedStore = $wsProps.SelfSignedCertStore
if (-not $selfSignedStore) { $selfSignedStore = "Remote Desktop" }
$bytes = $wsProps.SelfSignedCertificate
if ($bytes) { $selfSignedThumb = ($bytes | ForEach-Object { $_.ToString("X2") }) -join "" }
Write-Host "=== Global Self-Signed Certificate ==="
Write-Host "  SelfSignedCertStore   : $selfSignedStore"
Write-Host "  SelfSignedCertificate : $(if ($selfSignedThumb) { $selfSignedThumb } else { '(not set)' })"

# 2. Check certificate propagation service
Write-Host "`n=== Certificate Propagation Service ==="
Get-Service -Name CertPropSvc -ErrorAction SilentlyContinue | Select-Object Name, Status, StartType | Format-Table -AutoSize

# 3. Check certificates per WinStation
Get-ChildItem -Path $winStationsPath -ErrorAction SilentlyContinue |
    Where-Object { $_.PSChildName -ne "Console" } |
    ForEach-Object {
        $stationName = $_.PSChildName
        Write-Host "`n=== WinStation: $stationName ==="

        # SSLCertificateSHA1Hash is REG_BINARY, convert to hex thumbprint
        $sslHashBytes = (Get-ItemProperty -Path $_.PSPath -Name "SSLCertificateSHA1Hash" -ErrorAction SilentlyContinue).SSLCertificateSHA1Hash
        if ($sslHashBytes) {
            $thumb = ($sslHashBytes | ForEach-Object { $_.ToString("X2") }) -join ""
            $storeName = "My"
            $source = "Custom"
        } else {
            $thumb = $selfSignedThumb
            $storeName = $selfSignedStore
            $source = "SelfSigned (default)"
        }

        Write-Host "  Source    : $source"
        Write-Host "  CertStore : LocalMachine\$storeName"
        Write-Host "  Thumbprint: $(if ($thumb) { $thumb } else { '(none)' })"

        # Query certificate details
        if ($thumb) {
            $cert = Get-ChildItem -Path "Cert:\LocalMachine\$storeName" -ErrorAction SilentlyContinue |
                    Where-Object { $_.Thumbprint -eq $thumb }
            if ($cert) {
                $cert | Select-Object Subject, Issuer, NotBefore, NotAfter, HasPrivateKey, Thumbprint | Format-List
            } else {
                Write-Host "  [Abnormal] Certificate not found (thumbprint $thumb does not exist in $storeName store)"
            }
        }
    }
```

**分析思路**：

1. 检查全局自签名证书定义：
   - 正常：WinStations 键下 SelfSignedCertificate 已设置，SelfSignedCertStore 为 "Remote Desktop" 或有明确值
   - 异常：SelfSignedCertificate 未设置且所有 WinStation 均无 SSLCertificateSHA1Hash → TermService 可能未正常初始化，**严重程度**：Warning

2. 检查证书属性服务（CertPropSvc）：
   - 正常：服务正在运行
   - 异常：服务未运行 → **根因**：证书属性服务未运行，域环境下证书可能无法自动更新，**严重程度**：Warning

3. 对每个 WinStation 识别证书来源：
   - Source = Custom → SSLCertificateSHA1Hash 已配置，使用自定义证书，存储位置为 `Cert:\LocalMachine\My`
   - Source = SelfSigned (default) → SSLCertificateSHA1Hash 未配置，回退到全局默认自签名证书，存储位置由 SelfSignedCertStore 指定（通常为 `Cert:\LocalMachine\Remote Desktop`）
   - 两种来源均为正常行为，不构成异常

4. 对每个 WinStation 验证解析后的证书：
   - 证书不存在（指纹在对应存储中未找到） → **根因**：RDP 证书丢失（标注 StationName 和 Source），连接时可能出现 Internal error，**严重程度**：Critical
   - NotAfter < 当前时间 → **根因**：RDP 证书过期，连接时会出现证书警告，**严重程度**：Warning
   - Subject == Issuer → 自签名证书，客户端会显示证书警告，**严重程度**：Info
   - HasPrivateKey 为 false → **根因**：证书私钥不可访问，TermService 无法完成 TLS 握手 → 必须跳转 **Step 2** 检查 MachineKeys 权限，**严重程度**：Critical

### Step 2: MachineKeys 与 TLS 私钥权限检查

**数据采集**：

> 采集目标：检查 MachineKeys 目录权限、ProgramData 路径配置和 TLS 私钥文件权限。当 Step 1 发现证书 HasPrivateKey = false 时本步骤为必检项；RDP 自签名证书的私钥存储在 MachineKeys 目录下 f686aace 前缀的文件中

```powershell
# 1. Get ProgramData path (via registry to handle non-default paths)
$programDataPath = (Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList" -Name "ProgramData" -ErrorAction SilentlyContinue).ProgramData
if (-not $programDataPath) { $programDataPath = $env:ProgramData }
Write-Host "ProgramData path: $programDataPath"

# 2. MachineKeys directory permission check
$machineKeysPath = "$programDataPath\Microsoft\Crypto\RSA\MachineKeys"
Write-Host "--- MachineKeys Directory ACL ---"
Get-Acl -Path $machineKeysPath -ErrorAction SilentlyContinue |
    Select-Object -ExpandProperty Access |
    Where-Object { $_.IdentityReference -match 'Everyone|Administrators|SYSTEM' } |
    Select-Object IdentityReference, FileSystemRights, AccessControlType | Format-Table -AutoSize

# 3. TLS private key file check (f686aace prefix)
Write-Host "--- TLS Private Key Files (f686aace*) ---"
try {
    $tlsKeys = Get-ChildItem -Path $machineKeysPath -Filter "f686aace*" -ErrorAction Stop
    if ($tlsKeys) {
        $tlsKeys | ForEach-Object {
            Write-Host "  File: $($_.Name)"
            $fileAcl = Get-Acl -Path $_.FullName -ErrorAction SilentlyContinue
            $fileAcl.Access | Where-Object {
                $_.IdentityReference -match 'NETWORK SERVICE|SYSTEM|Administrators'
            } | Select-Object IdentityReference, FileSystemRights, AccessControlType | Format-Table -AutoSize
        }
    } else {
        Write-Host "  No TLS private key files starting with f686aace found"
    }
} catch {
    Write-Host "  [Error] Unable to read MachineKeys directory content: $($_.Exception.Message)"
}
```

**分析思路**：

1. 检查 MachineKeys 目录是否存在：
   - 正常：目录存在
   - 异常：目录不存在 → **根因**：MachineKeys 目录缺失，证书无法创建或更新，**严重程度**：Critical

2. 检查关键账户权限：
   - 正常：Administrators 有完全控制权，Everyone 有读取和写入权限
   - 异常：Everyone 无读取权限 → **根因**：MachineKeys 权限异常，RDP 证书无法更新，**严重程度**：Critical
   - 异常：Administrators 无完全控制权 → **根因**：MachineKeys 权限被篡改，**严重程度**：Critical

3. 检查 TLS 私钥文件（f686aace 前缀，对应 RDP 自签名证书的私钥；与 Step 1 中 HasPrivateKey = false 直接关联）：
   - 前置条件：如果输出 `[错误] 无法读取 MachineKeys 目录内容`，说明目录权限异常导致无法枚举文件，此时私钥文件查询结果不可信，输出根因时需注明「因 MachineKeys 目录权限异常，私钥状态待目录权限修复后确认」
   - 正常：文件存在，NETWORK SERVICE 有读取权限，SYSTEM 有完全控制权
   - 异常：文件不存在 → **根因**：TLS 私钥文件缺失，RDP TLS 协商将失败（对应 Step 1 证书 HasPrivateKey = false），**严重程度**：Critical
   - 异常：NETWORK SERVICE 无读取权限 → **根因**：TLS 私钥访问被拒绝，TermService 无法读取私钥，**严重程度**：Critical
   - 异常：SYSTEM 无完全控制权 → **根因**：TLS 私钥访问被拒绝，**严重程度**：Critical

### Step 3: 系统盘根目录权限检查

**数据采集**：

> 采集目标：检查系统盘根目录(C:\\) 的关键账户权限，确保 TermService 能正常访问系统文件

```powershell
$systemDrive = $env:SystemDrive + "\\"
Write-Host "System drive root: $systemDrive"
(Get-Acl -Path $systemDrive -ErrorAction SilentlyContinue).Access | Where-Object {
    $_.IdentityReference -match 'BUILTIN\\Users|NT AUTHORITY\\SERVICE'
} | Select-Object IdentityReference, FileSystemRights, AccessControlType | Format-Table -AutoSize
```

**分析思路**：

1. 检查 BUILTIN\Users 权限：
   - 正常：BUILTIN\Users 具有读取和执行权限（ReadAndExecute）
   - 异常：BUILTIN\Users 缺少读取和执行权限 → **根因**：系统盘根目录权限异常，可能导致 RDP 连接失败，**严重程度**：Critical

2. 检查 NT AUTHORITY\SERVICE 权限：
   - 正常：NT AUTHORITY\SERVICE 具有读取权限（Read）
   - 异常：NT AUTHORITY\SERVICE 缺少读取权限 → **根因**：系统盘根目录权限异常，服务账户无法读取系统文件，**严重程度**：Critical

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 1 HasPrivateKey = false | → Step 2 (MachineKeys 与 TLS 私钥权限) |
| 条件跳转 | Step 2 MachineKeys 权限异常 | → [identity-permission.md](references/identity-permission.md) |
| 条件跳转 | Step 3 系统盘权限异常 | → [identity-permission.md](references/identity-permission.md) |
| 链式后继 | 本文件未确认根因，用户报告 RDP 证书问题 | → 无（证书问题通常在此文件内定位） |

## 修复建议

### 根因：RDP 证书过期

**修复操作**：

```powershell
# Method 1: Delete expired certificate and let system auto-generate new self-signed certificate
# Enumerate all WinStations, check and delete expired certificates
$winStationsPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations"
Get-ChildItem -Path $winStationsPath -ErrorAction SilentlyContinue |
    Where-Object { $_.PSChildName -ne "Console" } |
    ForEach-Object {
        $stationName = $_.PSChildName
        $rdpCertThumbprint = (Get-ItemProperty -Path $_.PSPath -Name "SSLCertificateSHA1Hash" -ErrorAction SilentlyContinue).SSLCertificateSHA1Hash
        if ($rdpCertThumbprint) {
            $expiredCert = Get-ChildItem -Path "Cert:\LocalMachine\My" | Where-Object {$_.Thumbprint -eq $rdpCertThumbprint}
            if ($expiredCert -and $expiredCert.NotAfter -lt (Get-Date)) {
                Remove-Item -Path "Cert:\LocalMachine\My\$($expiredCert.Thumbprint)" -Force
                Write-Host "[$stationName] Deleted expired certificate"
            }
        }
    }

# Restart TermService to generate new certificate
Restart-Service -Name TermService -Force
Write-Host "Restarted TermService"
```

**验证方法**：

```powershell
Get-ChildItem -Path "Cert:\LocalMachine\Remote Desktop" | Select-Object Subject, NotAfter, Thumbprint
```

预期结果：存在未过期的证书

**风险说明**：删除证书会短暂中断 RDP 连接，新证书为自签名证书，客户端会显示警告

---

### 根因：MachineKeys 权限异常

**修复操作**：

```powershell
# Fix MachineKeys directory permissions
$machineKeysPath = "$env:ProgramData\Microsoft\Crypto\RSA\MachineKeys"
if (Test-Path $machineKeysPath) {
    # Restore Everyone read and write permissions
    $acl = Get-Acl -Path $machineKeysPath
    $rule = New-Object System.Security.AccessControl.FileSystemAccessRule("Everyone", "Read,Write,Synchronize", "Allow")
    $acl.SetAccessRule($rule)
    Set-Acl -Path $machineKeysPath -AclObject $acl
    Write-Host "Fixed MachineKeys directory permissions"
}
```

**验证方法**：

```powershell
Get-Acl -Path "$env:ProgramData\Microsoft\Crypto\RSA\MachineKeys" | 
    Select-Object -ExpandProperty Access | 
    Where-Object {$_.IdentityReference -match "Everyone"}
```

预期结果：Everyone 有 Read,Write,Synchronize 权限

**风险说明**：修改 MachineKeys 权限影响所有系统证书，需谨慎操作

---

### 根因：RDP 证书丢失

**修复操作**：

```powershell
# Generate self-signed RDP certificate (requires administrator privileges)
$cert = New-SelfSignedCertificate -Type SSLServerAuthentication `
    -Subject "CN=RDP Self-Signed Certificate" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -NotAfter (Get-Date).AddYears(1) `
    -KeyExportPolicy Exportable `
    -KeyLength 2048

# Configure new certificate for specified WinStation (replace <StationName> with actual station name)
$thumbprint = $cert.Thumbprint
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\<StationName>" -Name "SSLCertificateSHA1Hash" -Value $thumbprint

Write-Host "Generated and configured new RDP self-signed certificate: $thumbprint"

# Restart service
Restart-Service -Name TermService -Force
```

**验证方法**：

```powershell
$winStationsPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations"
Get-ChildItem -Path $winStationsPath -ErrorAction SilentlyContinue |
    Where-Object { $_.PSChildName -ne "Console" } |
    ForEach-Object {
        $hash = (Get-ItemProperty -Path $_.PSPath -Name "SSLCertificateSHA1Hash" -ErrorAction SilentlyContinue).SSLCertificateSHA1Hash
        [PSCustomObject]@{ Station = $_.PSChildName; SSLCertificateSHA1Hash = $hash }
    }
```

预期结果：目标 WinStation 的 SSLCertificateSHA1Hash 包含新证书指纹

**风险说明**：自签名证书会导致客户端显示证书警告，生产环境建议使用 CA 颁发的证书

---

### 根因：TLS 私钥访问被拒绝

**修复操作**：

```powershell
# Find TLS private key files
$programDataPath = (Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList" -Name "ProgramData" -ErrorAction SilentlyContinue).ProgramData
if (-not $programDataPath) { $programDataPath = $env:ProgramData }
$machineKeysPath = "$programDataPath\Microsoft\Crypto\RSA\MachineKeys"
$tlsKeys = Get-ChildItem -Path $machineKeysPath -Filter "f686aace*" -ErrorAction SilentlyContinue

if ($tlsKeys) {
    $tlsKeys | ForEach-Object {
        $acl = Get-Acl -Path $_.FullName

        # Add NETWORK SERVICE read permission
        $nsRule = New-Object System.Security.AccessControl.FileSystemAccessRule("NT AUTHORITY\NETWORK SERVICE", "Read", "Allow")
        $acl.SetAccessRule($nsRule)

        # Ensure SYSTEM has full control
        $sysRule = New-Object System.Security.AccessControl.FileSystemAccessRule("NT AUTHORITY\SYSTEM", "FullControl", "Allow")
        $acl.SetAccessRule($sysRule)

        Set-Acl -Path $_.FullName -AclObject $acl
        Write-Host "Fixed TLS private key file permissions: $($_.Name)"
    }
} else {
    Write-Host "No TLS private key files starting with f686aace found, may need to regenerate certificate"
}

# Restart TermService
Restart-Service -Name TermService -Force
```

**验证方法**：

```powershell
$machineKeysPath = "$env:ProgramData\Microsoft\Crypto\RSA\MachineKeys"
Get-ChildItem -Path $machineKeysPath -Filter "f686aace*" -ErrorAction SilentlyContinue | ForEach-Object {
    Write-Host "File: $($_.Name)"
    (Get-Acl $_.FullName).Access | Where-Object { $_.IdentityReference -match 'NETWORK SERVICE|SYSTEM' } |
        Select-Object IdentityReference, FileSystemRights, AccessControlType | Format-Table -AutoSize
}
```

预期结果：NETWORK SERVICE 有 Read 权限，SYSTEM 有 FullControl 权限

**风险说明**：修改私钥文件权限后需重启 TermService 生效；如果私钥文件缺失，需重新生成证书

---

### 根因：系统盘根目录权限异常

**修复操作**：

```powershell
$systemDrive = $env:SystemDrive + "\\"
Write-Host "System drive root: $systemDrive"

$acl = Get-Acl -Path $systemDrive

# Add BUILTIN\Users read and execute permission
$usersRule = New-Object System.Security.AccessControl.FileSystemAccessRule("BUILTIN\Users", "ReadAndExecute", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.SetAccessRule($usersRule)

# Add NT AUTHORITY\SERVICE read permission
$serviceRule = New-Object System.Security.AccessControl.FileSystemAccessRule("NT AUTHORITY\SERVICE", "Read", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.SetAccessRule($serviceRule)

Set-Acl -Path $systemDrive -AclObject $acl
Write-Host "Fixed system drive root permissions"
```

**验证方法**：

```powershell
$systemDrive = $env:SystemDrive + "\\"
(Get-Acl -Path $systemDrive).Access | Where-Object {
    $_.IdentityReference -match 'BUILTIN\\Users|NT AUTHORITY\\SERVICE'
} | Select-Object IdentityReference, FileSystemRights, AccessControlType | Format-Table -AutoSize
```

预期结果：BUILTIN\Users 有 ReadAndExecute 权限，NT AUTHORITY\SERVICE 有 Read 权限

**风险说明**：修改系统盘根目录权限影响范围广泛，需谨慎操作；修改前建议备份当前 ACL
