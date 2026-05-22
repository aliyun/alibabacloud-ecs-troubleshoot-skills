# aliyun CLI 工具速查

通过 `aliyun ecs <SubCommand> --<Param> <Value>` 调用,下表列出排查流程常用子命令。完整参数以 `aliyun ecs <SubCommand> help` 输出为准。

## 实例与资源元数据

| 工具 | 用途 | 必需参数 |
| --- | --- | --- |
| `aliyun.ecs.DescribeInstances` | 查询实例信息（状态、规格、网络、付费类型等） | `RegionId`（`InstanceIds` JSON 数组过滤单实例） |
| `aliyun.ecs.DescribeInstanceAttribute` | 查询单实例详细属性 | `InstanceId` |
| `aliyun.ecs.DescribeDisks` | 查询实例系统盘/数据盘详情 | `RegionId`（`InstanceId` 过滤） |
| `aliyun.ecs.DescribeImages` | 查询镜像信息与标记 | `RegionId`（`ImageId` / `InstanceId` 过滤） |
| `aliyun.ecs.DescribeNetworkInterfaces` | 查询实例网卡（主网卡/辅助网卡）配置 | `RegionId`（`InstanceId` 过滤） |
| `aliyun.ecs.DescribeSecurityGroups` | 查询实例所属安全组列表 | `RegionId`（`InstanceId` 过滤） |
| `aliyun.ecs.DescribeSecurityGroupAttribute` | 查询安全组规则详情（入/出向） | `RegionId` + `SecurityGroupId` |
| `aliyun.ecs.DescribeUserData` | 查询实例的 UserData | `RegionId` + `InstanceId` |
| `aliyun.ecs.GetInstanceScreenshot` | 获取实例 VNC 控制台实时截屏 | `RegionId` + `InstanceId` |
| `aliyun.ecs.GetInstanceConsoleOutput` | 获取实例串口控制台输出 | `RegionId` + `InstanceId` |

## 规格与镜像目录

| 工具 | 用途 | 必需参数 |
| --- | --- | --- |
| `aliyun.ecs.DescribeInstanceTypes` | 查询实例规格目录 | 无（可用 `InstanceTypeFamily` / `InstanceTypes.n` 过滤） |
| `aliyun.ecs.DescribeInstanceTypeFamilies` | 查询可用实例规格族 | `RegionId` |
| `aliyun.ecs.DescribeImageSupportInstanceTypes` | 查询镜像兼容的实例规格 | `RegionId`（`ImageId` 过滤） |

## ECS 诊断报告

| 工具 | 用途 | 必需参数 |
| --- | --- | --- |
| `aliyun.ecs.CreateDiagnosticReport` | 触发资源诊断；不指定 `MetricSetId` 时使用默认集 `dms-instancedefault`，返回 `ReportId` | `RegionId` + `ResourceId`（实例 ID） |
| `aliyun.ecs.DescribeDiagnosticReportAttributes` | 轮询诊断报告详情，提取异常诊断项 | `RegionId` + `ReportId` |
| `aliyun.ecs.DescribeDiagnosticReports` | 查询历史诊断报告列表 | `RegionId` |

## GuestOS 内数据采集

| 工具 | 用途 | 必需参数 |
| --- | --- | --- |
| `aliyun.ecs.DescribeCloudAssistantStatus` | 查询云助手 Agent 是否在线（调用 `RunCommand` 前的前置检查） | `RegionId`（`InstanceId.n` 过滤） |
| `aliyun.ecs.RunCommand` | 在一台或多台 ECS 内执行 Shell/PowerShell/Bat 脚本 | `RegionId` + `Type`（`RunShellScript`） + `CommandContent` + `InstanceId.N` |
| `aliyun.ecs.DescribeInvocations` | 查询云助手命令执行列表与状态（`Running` / `Finished` / `Failed`） | `RegionId`（`InvokeId` / `InstanceId` 过滤） |
| `aliyun.ecs.DescribeInvocationResults` | 查询云助手命令实际执行输出（stdout/exit code） | `RegionId`（`InvokeId` / `InstanceId` 过滤） |
| `aliyun.ecs.StopInvocation` | 停止 `Running` 状态的云助手命令 | `RegionId` + `InvokeId` + `InstanceId.N` |
