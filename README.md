# ZCode Plugin Swarm

ZCode 插件蜂群发布包与开发资料。

本仓库用于集中存放 ZCode 桌面版插件相关文件，方便下载、安装、学习和二次开发。仓库包含插件压缩包、跨平台 Computer Use 后端参考代码、ZCode 插件开发手册，以及安全架构审查 Skill Prompt。
这些都是最初版本，因为zcode不开源，它还在不断更新，不能确保能一直适配zcode后续版本,因为可能出现后续zcode版本更新，架构调整，导致插件无法正常运行.

## 快速下载

- 下载整个仓库：[Download ZIP](https://github.com/ashelylinluo/zcode-plugin-swarm/archive/refs/heads/main.zip)
- Agent Teams 插件包：[zcode-agent-teams-plugin.zip](./zcode-agent-teams-plugin.zip)
- Computer Use 插件包：[zcode-computer-use-plugin.zip](./zcode-computer-use-plugin.zip)
- 开发手册：[ZCode开发手册.md](./ZCode%E5%BC%80%E5%8F%91%E6%89%8B%E5%86%8C.md)

## 文件说明

| 文件 | 说明 |
| --- | --- |
| `zcode-agent-teams-plugin.zip` | Agent Teams / Swarm 相关插件发布包，包含 commands、skills、agents、MCP server 源码与插件配置。 |
| `zcode-computer-use-plugin.zip` | Computer Use 插件发布包，包含截图、窗口、应用管理等桌面自动化能力相关源码与依赖。 |
| `ZCode开发手册.md` | ZCode 桌面版插件开发手册，覆盖插件体系、Skills、Commands、Agents、MCP Server 和 Hook。 |
| `security-architecture-review-skill.md` | 安全架构与回归验证审查 Skill Prompt。 |
| `darwin.ts` | macOS Computer Use 后端参考实现。 |
| `linux.ts` | Linux Computer Use 后端参考实现。 |
| `win32.ts` | Windows Computer Use 后端参考实现。 |
| `data_stat.xls` | 辅助数据表。 |

## 使用方式

1. 需要直接使用插件时，优先下载对应的 `.zip` 插件包。
2. 解压插件包后，阅读插件包内部的 `README.md` 和 `.zcode-plugin/plugin.json`。
3. 需要开发或修改插件时，先阅读 [ZCode开发手册.md](./ZCode%E5%BC%80%E5%8F%91%E6%89%8B%E5%86%8C.md)。
4. 需要参考 Computer Use 的平台适配逻辑时，查看 `darwin.ts`、`linux.ts`、`win32.ts`。

## 面向对象

- ZCode 桌面版插件开发者
- 想学习 ZCode Skills、Commands、Agents、MCP Server、Hook 扩展机制的用户
- 想下载并试用 Agent Teams 或 Computer Use 插件的用户

## 说明

本仓库当前主要用于文件分发和开发资料共享。若用于正式开源发布，建议后续补充 `LICENSE`、版本号、变更日志和更完整的安装说明。
