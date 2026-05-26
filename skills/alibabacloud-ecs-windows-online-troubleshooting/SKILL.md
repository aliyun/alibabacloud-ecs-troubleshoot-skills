---
name: alibabacloud-ecs-windows-online-troubleshooting
description: Use this skill whenever the user reports a problem with a Windows system, Windows Server, or Windows ECS instance, or asks to check, diagnose, inspect, or troubleshoot anything Windows-related. Covers network issues (ping, DNS, DHCP, firewall, SMB), RDP/remote desktop (connection failures, authentication, black screen, lag), storage/disk, system activation, Windows Update, performance slowdowns, user accounts/permissions, drivers, app crashes, security/certificate/TLS, scheduled tasks, or vague requests like 'check the system' or 'something is wrong.' Load this skill even if the user does not explicitly mention Windows but the context implies a Windows environment.
---

# Windows ECS 在线诊断 Skill

## 何时使用本技能

当用户报告 Windows 实例的故障现象，或要求对 Windows 实例进行问题排查、状态检查时，加载本技能。覆盖范围包括：

- 网络问题：ping、DNS、DHCP、防火墙、SMB
- RDP/远程桌面：连接失败、认证、黑屏、卡顿
- 存储/磁盘、系统激活、Windows Update、性能问题
- 用户账户/权限、驱动、应用崩溃、安全/证书/TLS、计划任务
- 模糊描述如“检查一下系统”或“系统有问题”

诊断能力清单和问题路由表存放在 `references/REFERENCE.md` 中，在路径规划阶段加载。

---

## 诊断执行流程

### 用户呈现通用规则

执行过程中向用户呈现进度、排查路径、当前步骤等信息时，**禁止暴露 Skill 内部标记**，必须用自然语言描述该步骤所对应的诊断功能。需隐藏的内部标记包括但不限于：

- **内部文件名 / 路径**：如 `references/REFERENCE.md`、`references/rdp-service.md`、`references/networking-tcpip.md` 等
- **内部术语**：如「路由表」「reference」「推测性排查」「动态规划」「step 集合」「事件日志预诊断」 等 Skill 设计概念
- **内部编号 / 标签**：如 `Step 1`、`Step 2` 等机械编号，以及 `Direct/Contributing/Unrelated`、`Critical/Warning/Info`、`MUST` 等词汇的原始字样
- **工具调用详情**：如「调用 ReadReference / 加载 xxx 文件」之类的实现描述

**示例**：
- 不要说：「现在执行 rdp-service.md 的步骤」「本问题为 Direct + Critical」
- 改为说：「现在检查 TermService 服务状态与 RDP 监听端口配置」「此问题是直接根因且严重影响连接」

内部标记仅用于工具调用、日志与模型内部推理，**不进入**对话面向用户的文本。

### 问题理解

1. 提取用户问题中的核心现象（错误代码、提示信息、用户操作）
2. 确认关键上下文：系统能否启动？当前执行环境（控制台/RDP/云助手）？
3. **信息不足时 MUST 先向用户追问**（具体现象、错误提示、操作时机等），收集到足够信息后再进入路径规划阶段
4. 记录原始问题描述，后续所有分析 MUST 围绕此问题

### 路径规划

1. **加载诊断能力与路由表**：读取 `references/REFERENCE.md`，获取诊断能力清单和问题路由表
2. **事件日志预诊断**（可选）：
   - 采集用户报告时间段内的关键事件日志（System、Application、Security）
   - 筛选 Error/Critical 级别事件，提取 Event ID 和来源
   - 根据事件日志特征初步界定问题方向（如网络类、驱动类、服务类、安全类）
   - 注意：仅采集和预分析，不执行具体诊断步骤
3. **路由表匹配**（可使用专有知识库和世界知识辅助判断）：
   - 精确匹配 → 使用既定序列
   - 模糊匹配 → 使用最接近序列，但标注为「推测性排查」
4. **动态规划**（未命中路由表时）：
   - 结合用户问题描述 + 事件日志预诊断结果，从诊断能力清单中选择相关 reference
   - 组合排查序列，并向用户说明：「此为动态规划的排查路径，非预设场景」
5. 输出：有序的 reference 列表（通常 2-5 个，避免过多）+ 规划理由
   - 向用户展示排查路径时，禁止使用 "Step 1、Step 2" 等无意义编号，必须描述每一步要检查的具体内容（如「检查 TermService 服务状态及监听端口配置」）

### 逐步执行

按排查序列逐个加载 reference 执行诊断。

**单个 reference 执行规则**：
1. 读取 reference 文件（`references/{file}.md`）
2. 根据用户问题从步骤集合中选取相关子集
3. 严格逐步骤执行：每个步骤必须完成「数据采集 → 分析判定 → 正常/异常结论」全流程后，才能进入下一步。禁止一次性批量采集多个步骤的数据再集中分析
   - 每个异常判定 MUST 有对应采集数据作为证据
   - 每个步骤 MUST 给出明确的正常/异常二分结论
   - 发现的异常 MUST 关联到具体根因
   - 采集脚本可根据实际情况调整，但应参考 reference 文件中提供的脚本作为起点
4. 遇到交叉引用 → 优先处理跳转目标，完成后回到主序列
5. 重复引用优化：同一个 reference 被多次引用时只需执行一次，后续直接复用已有结果

**序列控制逻辑**：
- **提前终止**：发现的根因已能完全解释用户所有症状 → 终止后续 reference
- **继续执行**：仅解释部分症状 → 继续执行后续 reference
- **方向修正**：执行中发现问题方向与初始规划不符 → 回退到路径规划阶段，基于新线索重新规划

### 因果链分析

1. 汇总所有 reference 的发现
2. 构建因果关系链（区分直接原因和间接原因）
3. 按与用户问题的相关性排列优先级：**relation（Direct > Contributing > Unrelated）× severity（Critical > Warning > Info）双维度排序**；Unrelated 问题单独列为"其他发现"，不混入主要修复方案
4. 无发现时：尝试扩展排查范围；仍无发现 → 如实告知用户，给出建议的排查方向

### 修复方案

1. 每个根因提供可执行的修复脚本，MUST 附带验证方法和预期结果
2. 按优先级顺序呈现
3. **展示完整修复内容给用户，等待明确确认 — 禁止自动执行任何修复命令**
4. 同题聚合：同一根因的多个修改合并为一个脚本
5. 异题分离：不同根因的修复严格分开，逐个确认
6. 涉及系统修改的操作 MUST 标注风险

---

## 因果链输出规范

### 因果链示例

```
[用户问题] 远程桌面连不上
    │
    ├── [Direct] TermService 服务已停止
    │       └── [Contributing] 依赖服务 RpcSs 异常
    │
    └── [Direct] 防火墙阻止 3389 端口
            └── [Contributing] 公共网络配置文件生效
```

### 修复脚本模板

```powershell
#requires -RunAsAdministrator
# 修复：{根因名称}
# 风险：{风险说明}
# 验证：执行后运行 {验证命令} 确认修复结果

# --- 修复操作 ---
{修复命令}

# --- 验证 ---
{验证命令}
```

### 诊断结论输出模板

> **诊断结论**
>
> **用户问题**：{原始问题描述}
>
> 共发现 {N} 个问题，按修复优先级排列：
>
> ---
>
> **🔴 问题 1（Direct | Critical）：{root_cause}**
>
> **证据**：{采集到的异常数据}
>
> **分析**：{为什么这个问题直接导致了用户看到的现象}
>
> **因果链**：{用户问题} ← {直接原因} ← {间接原因（如有）}
>
> **修复方案**：
> ```powershell
> {修复脚本}
> ```
>
> **验证**：
> ```powershell
> {验证命令}
> ```
> 预期结果：{正常状态}
>
> ---
>
> **🟡 问题 2（Contributing | Warning）：{root_cause}**
> ...

---

## Gotchas（易错点）

### 诊断阶段

- `Get-WmiObject` 在部分系统上被弃用，优先使用 `Get-CimInstance`
- 网络连通性检查需区分 ICMP 被阻止和真正的网络不可达
- **命令输出格式简化原则**：使用命令采集信息时，输出格式 MUST 尽量简单，减少大模型理解成本。优先使用 `Select-Object` 仅输出关键字段，避免冗长的完整对象输出。示例：
  - 推荐：`Get-Service TermService | Select-Object Name, Status, StartType`
  - 不推荐：`Get-Service TermService`（输出包含大量无关字段）
  - 推荐：`Get-Process | Sort-Object CPU -Descending | Select-Object -First 5 Id, Name, CPU`
  - 不推荐：`Get-Process | Sort-Object CPU -Descending | Select-Object -First 5`（输出 20+ 字段）
- **PowerShell 格式化输出延迟**：`Select-Object` 返回的对象进入延迟格式化队列，后续 `Write-Host` 输出可能先于表格到达控制台，导致输出顺序错乱。采集脚本编写时必须在 `Select-Object` 后追加 `| Format-Table` 或 `| Format-List` 强制同步渲染，从源头保证输出顺序正确
- **避免解析 cmd 命令行的文本输出**：cmd 命令输出的字段名随系统语言变化，解析会在中/英文系统上产生不同结果。应优先使用 PowerShell cmdlet 替代；无法替代时直传原始输出交给 LLM 分析。常用替换：
  | cmd 命令 | PowerShell 替代 |
  |-----------|-------------------|
  | `net user` | `Get-LocalUser` / `Get-CimInstance Win32_UserAccount` |
  | `net localgroup` | `Get-LocalGroupMember` / `Get-CimInstance Win32_GroupUser` |
  | `netstat` | `Get-NetTCPConnection` / `Get-NetUDPEndpoint` |
  | `ipconfig` | `Get-NetIPConfiguration` / `Get-NetIPAddress` |
  | `tasklist` | `Get-Process` / `Get-CimInstance Win32_Process` |
  | `sc query` | `Get-Service` / `Get-CimInstance Win32_Service` |
  | `net accounts` | 无直接替代，直传原始输出 |
  | `netsh` | 无直接替代，直传原始输出 |
  | `fsutil` | 无直接替代，直传原始输出 |
  | `bcdedit` | 无直接替代，直传原始输出 |
- **禁止直接调用 GUI 弹窗命令**：`slmgr`、`winver` 等命令默认会弹出图形化对话框，在无人值守或非交互式执行环境（如云助手、远程脚本）中会导致挂起。必须使用 `cscript //Nologo` 前缀调用对应的 `.vbs` 脚本，将输出重定向到控制台：例如 `cscript //Nologo C:\windows\system32\slmgr.vbs /dli` 替代 `slmgr /dli`
- **编码规范**：编码问题分为输入侧（读取外部命令输出）和输出侧（PowerShell 自身输出）两个方面：

  **输入侧 — CMD/EXE 命令输出捕获**：以下命令使用系统默认代码页（简体中文系统为 GBK/936），与 PowerShell 的 UTF8 编码不一致，直接执行会导致中文乱码。执行这些命令时，MUST 通过 `ProcessStartInfo` 显式指定 `StandardOutputEncoding = [System.Text.Encoding]::GetEncoding(936)` 来正确捕获输出：
  - `w32tm` — Windows 时间服务命令
  - `cscript` — 脚本宿主（如 slmgr.vbs）
  - `netsh` — 网络配置命令
  - `ipconfig` — IP 配置命令
  - `net accounts` — 账户策略命令
  - `pnputil` — 设备驱动工具
  - `bcdedit` — 启动配置命令
  - `fsutil` — 文件系统工具
  - `icacls` — 权限管理命令
  - `wusa` — Windows 更新独立安装程序
  - 其他非 PowerShell 原生的 CMD/EXE 命令

  ProcessStartInfo 包装模板：
  ```powershell
  $psi = New-Object System.Diagnostics.ProcessStartInfo "<executable>", "<arguments>"
  $psi.RedirectStandardOutput = $true; $psi.UseShellExecute = $false
  $psi.StandardOutputEncoding = [System.Text.Encoding]::GetEncoding(936)
  $p = [System.Diagnostics.Process]::Start($psi)
  $p.StandardOutput.ReadToEnd(); $p.WaitForExit()
  ```
  reference 文件中可直接写原始命令（如 `w32tm /query /source`），agent 在实际执行时根据本规则自动应用 ProcessStartInfo 包装

  **输入侧**：以上规则用于正确捕获 cmd 命令的中文输出，避免乱码
- **变量命名规避内置标识符**：PowerShell 采集脚本中的自定义变量名 MUST 避免使用内置关键字（如 `switch`、`foreach`、`function` 等）和自动变量（如 `$_`、`$input`、`$args`、`$error`、`$host`、`$pwd`、`$foreach`、`$switch`、`$null`、`$true`、`$false` 等）。误用内置标识符会导致脚本行为异常或变量值被覆盖，排查困难
- **Get-ItemProperty 输出过滤规则**：`Get-ItemProperty` 默认返回包含 `PSPath`、`PSParentPath`、`PSChildName`、`PSDrive`、`PSProvider` 等 PowerShell 元数据字段，干扰诊断输出。MUST 通过以下方式之一过滤这些字段：
  1. 管道 `| Select-Object <目标属性>` 明确选取需要的属性（推荐，适用于已知属性名的场景）：`Get-ItemProperty ... -Name ProxyEnable, ProxyServer | Select-Object ProxyEnable, ProxyServer`
  2. 管道 `| Select-Object -Property * -ExcludeProperty PSPath,PSParentPath,PSChildName,PSDrive,PSProvider`（适用于需要全部注册表值但排除元数据的场景）
  3. 变量赋值后通过 `[PSCustomObject]` 提取目标属性（适用于需要进一步处理的场景）
  禁止直接输出 `Get-ItemProperty` 的完整结果

### 修复阶段

- 修复防火墙规则时，MUST 保留现有规则，仅追加或修改目标规则
- 重启服务可能影响正在使用的 RDP 会话，需提醒用户
- 修改注册表前 MUST 建议用户先备份相关键值
- RDP 证书修复涉及 MachineKeys 权限，操作需谨慎

### 输出阶段

- 因果链中的“间接原因”不一定需要修复（可能是上游问题的连锁反应）
- 标注为 Unrelated 的问题不要混入主要修复方案，单独列出作为“其他发现”
- 动态规划的排查路径 MUST 标注为推测性，告知用户这不是预设的排查序列

### 命令不存在处理

- 当 cmd 工具或 PowerShell cmdlet 执行时报错「命令/模块不存在」（如 `CommandNotFoundException`、`'xxx' is not recognized as an internal or external command`）→ **直接跳过该检查项，不要重试**
- 原因：系统环境不支持该工具（如精简版系统、组件未安装、版本过低、功能未启用），重试不会产生结果
- 处理方式：记录「该检查项因工具不可用已跳过」，继续执行后续步骤

### PowerShell 与 cmd 重定向语法混用

- **禁止在 PowerShell 脚本中使用 cmd 重定向语法**（如 `>nul`、`2>nul`、`1>nul`），这会触发 `RedirectionFailed` 错误
- PowerShell 中丢弃输出的正确写法：使用 `$null` 替代 `nul`，如 `2>$null`、`>$null`
- 示例对照：
  - 错误：`schtasks /query /fo LIST /v 2>nul | Select-String "TermService"`
  - 正确：`schtasks /query /fo LIST /v 2>$null | Select-String "TermService"`
  - 推荐：优先使用 PowerShell cmdlet（如 `Get-ScheduledTask`）替代 cmd 命令

### 含花括号的命令参数 MUST 用引号包裹

- PowerShell 把裸 `{...}` 视为脚本块（ScriptBlock），传给原生命令（如 `bcdedit`、`reg`、`schtasks` 等）时会被错误展开为 `-encodedCommand` 参数，导致命令报错
- **典型现象**：`bcdedit /enum {default}` 报错 `Invalid command line switch: /encodedCommand` / `参数错误`
- **规则**：所有含花括号的字面量参数（BCD 标识符 `{default}` / `{bootmgr}` / `{current}` / `{globalsettings}` / GUID 形式 `{xxxxxxxx-xxxx-...}` 等）MUST 用 **双引号** 或 **单引号** 包裹后再传给原生命令
- 示例对照：
  - 错误：`bcdedit /enum {default} 2>&1`
  - 正确：`bcdedit /enum "{default}" 2>&1`
  - 错误：`bcdedit /set {default} bootstatuspolicy IgnoreAllFailures`
  - 正确：`bcdedit /set "{default}" bootstatuspolicy IgnoreAllFailures`
- 同样适用于 `reg`、`wmic`、`schtasks`、`takeown` 等所有原生 cmd/exe 工具调用
