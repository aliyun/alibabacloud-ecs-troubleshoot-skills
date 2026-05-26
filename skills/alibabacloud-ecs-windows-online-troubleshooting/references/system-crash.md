# System Crash 诊断

## 功能说明

诊断 Windows BugCheck 蓝屏事件和 Crash Dump 配置。覆盖 2 个已知问题项。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 蓝屏、BSOD、BugCheck | Step 1 (BugCheck 蓝屏事件) → Step 2 (Crash Dump 配置) |
| 崩溃重启 | Step 1 → Step 2 |

## 诊断步骤

### Step 1: BugCheck 蓝屏事件检查

**数据采集**：

> 采集目标：查询系统事件日志中 BugCheck 蓝屏记录（EventID 1001，Source "BugCheck"）

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; Id=1001} -MaxEvents 10 -ErrorAction SilentlyContinue |
  Where-Object { $_.ProviderName -eq 'BugCheck' } |
  Select-Object TimeCreated, Id, Message |
  Format-Table -AutoSize -Wrap
```

**分析思路**：

1. 检查 BugCheck 蓝屏事件：
   - 正常：无 BugCheck 记录 → 近期无蓝屏事件
   - 异常：存在 EventID 1001 BugCheck 记录 → **根因**：近期发生过蓝屏崩溃（CriticalEvent），**严重程度**：Critical

> BugCheck 事件的 Message 字段包含 BugCheck Code（如 0x0000009F、0x000000D1），可用于定位崩溃原因。

### Step 2: Crash Dump 配置检查

**数据采集**：

> 采集目标：获取 CrashControl 注册表配置，确认 Crash Dump 是否启用

```powershell
Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\CrashControl' -ErrorAction SilentlyContinue |
  Select-Object CrashDumpEnabled, DumpFile, MinidumpDir, AutoReboot, LogEvent |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查 CrashDumpEnabled 配置：
   - 0：禁用 → **根因**：Crash Dump 未启用，蓝屏后无法生成转储文件（CrashDumpNotEnabled），**严重程度**：Warning
   - 1（Complete）或 2（Kernel）或 3（Small 64KB）或 7（Automatic，推荐）：**正常**

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | BugCheck Code 指向驱动问题（0xD1/0xCE 等） | → [cloud-driver.md](references/cloud-driver.md)（驱动版本检查） |
| 链式后继 | 本文件未确认根因 | → [performance-lifecycle.md](references/performance-lifecycle.md) |

## 修复建议

### Fix 1: 启用 Crash Dump（CrashDumpNotEnabled）

**适用根因**：CrashDumpNotEnabled

```powershell
# Enable Automatic Memory Dump (recommended)
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\CrashControl' -Name 'CrashDumpEnabled' -Value 7 -Type DWord
# Ensure auto-reboot is enabled
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\CrashControl' -Name 'AutoReboot' -Value 1 -Type DWord
```

**验证方法**：

```powershell
Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\CrashControl' -Name CrashDumpEnabled, AutoReboot | Select-Object CrashDumpEnabled, AutoReboot
```

预期结果：`CrashDumpEnabled` 值为 `7`，`AutoReboot` 值为 `1`

> 启用后，下次蓝屏将在 %SystemRoot%\MEMORY.DMP 生成转储文件，可用 WinDbg 分析根因。

### Fix 2: 分析 BugCheck 蓝屏转储（CriticalEvent — BugCheck）

**适用根因**：CriticalEvent（BugCheck 类型）

1. 确认 MEMORY.DMP 或 Minidump 文件存在：
   ```powershell
   Get-ChildItem -Path "$env:SystemRoot\MEMORY.DMP" -ErrorAction SilentlyContinue
   Get-ChildItem -Path "$env:SystemRoot\Minidump\*.dmp" -ErrorAction SilentlyContinue | Sort-Object LastWriteTime -Descending | Select-Object -First 5
   ```
2. 使用 WinDbg 打开转储文件，执行 `!analyze -v` 获取详细崩溃分析
3. 根据 BugCheck Code 查阅微软 BugCheck Code Reference 文档

**验证方法**：

```powershell
Get-ChildItem -Path "$env:SystemRoot\MEMORY.DMP" -ErrorAction SilentlyContinue
Get-ChildItem -Path "$env:SystemRoot\Minidump\*.dmp" -ErrorAction SilentlyContinue | Sort-Object LastWriteTime -Descending | Select-Object -First 1
```

预期结果：MEMORY.DMP 或 Minidump 目录下存在 .dmp 文件，且修改时间为最近
