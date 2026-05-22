# alibabacloud-ecs-troubleshoot-skills

面向 AI Agent 的阿里云 ECS 问题排查技能集（Agent Skills）。

## 概述

本仓库提供一组可被 AI Agent 直接加载使用的 Agent Skill，用于对**阿里云 ECS 实例**进行结构化、可复现的异常问题排查。

## Skills 列表

| Skill | 路径 | 适用场景 |
| --- | --- | --- |
| `alibabacloud-ecs-linux-troubleshooting` | [`skills/alibabacloud-ecs-linux-troubleshooting/SKILL.md`](skills/alibabacloud-ecs-linux-troubleshooting/SKILL.md) | 阿里云 ECS Linux 实例的异常问题排查：无法启动、SSH/VNC 登录失败、网络不通/丢包/延迟、磁盘与文件系统异常、性能异常、宕机夯机、时钟跳变、配置不生效等。 |

## 依赖

- [aliyun CLI](https://github.com/aliyun/aliyun-cli)：所有 Skill 均依赖 aliyun CLI 完成对目标 ECS 实例的远程诊断与数据采集。请先安装并完成凭证配置（`aliyun configure`），确保具备目标实例所在账号下的相应只读/诊断权限。

## 安装

1. 安装 aliyun CLI：参考 [aliyun/aliyun-cli](https://github.com/aliyun/aliyun-cli) 的安装文档，安装完成后执行：

   ```bash
   aliyun configure
   ```

   按提示填入 AccessKey ID、AccessKey Secret 和默认 Region。

2. 安装 Skill

### 通过 npx 安装

参考 [skills.sh](https://skills.sh) 的文档进行安装。

```bash
npx skills add aliyun/alibabacloud-ecs-troubleshoot-skills -g
```

### 手动安装

```bash
# 克隆本仓库到本地
git clone https://github.com/aliyun/alibabacloud-ecs-troubleshoot-skills.git
# 复制 Skill 到 Agent Skills 全局目录
mkdir -p ~/.agents/skills
cp -r alibabacloud-ecs-troubleshoot-skills/skills/* ~/.agents/skills/
```

## 贡献指南

欢迎通过 Issue 和 Pull Request 贡献新的 Skill 或改进现有排查文档。

- **新增 Skill**：遵循 [Agent Skills 规范](https://agentskills.io/)
- **提交规范**：建议使用 [Conventional Commits](https://www.conventionalcommits.org/)
- **质量要求**：新增/修改的排查步骤需基于真实排查案例，给出可复现的命令与判定标准，避免引入未经验证的命令。
- **文档语言**：与现有文档保持一致，使用中文撰写。

## 相关链接

如需在控制台侧进行可视化诊断，可访问阿里云 ECS 自助问题排查页面：

- <https://ecs.console.aliyun.com/troubleshooting>

## 许可证

本项目采用 [Apache License 2.0](LICENSE)。
