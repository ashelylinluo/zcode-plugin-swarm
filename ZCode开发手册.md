# ZCode 桌面版插件开发手册

> **面向 ZCode 桌面版插件开发者。覆盖插件体系、Skills、Commands、Agents、MCP Server 和 Hook 的全部扩展机制。**
>
> 最后更新：2026-07-01

---

## 目录

- [第一部分：插件体系概述](#第一部分插件体系概述)
  - [1.0 本机目录结构](#10-本机目录结构必读)
  - [1.1 ZCode 插件是什么](#11-zcode-插件是什么)
  - [1.2 plugin.json 完整 Schema](#12-pluginjson-完整-schema)
  - [1.3 插件目录结构](#13-插件目录结构)
  - [1.4 插件注册与加载](#14-插件注册与加载)
  - [1.5 本机开发 vs 市场分发](#15-本机开发-vs-市场分发)
  - [1.6 环境变量](#16-环境变量)
  - [1.7 关键目录速查](#17-关键目录速查)
- [第二部分：Skills 开发](#第二部分skills-开发)
  - [2.1 Skill 是什么](#21-skill-是什么)
  - [2.2 SKILL.md 格式规范](#22-skillmd-格式规范)
  - [2.3 Skill 的发现与加载机制](#23-skill-的发现与加载机制)
  - [2.4 Skill 的分类体系](#24-skill-的分类体系)
  - [2.5 如何写一个好的 Skill](#25-如何写一个好的-skill)
  - [2.6 官方 Superpowers 技能体系分析](#26-官方-superpowers-技能体系分析)
  - [2.7 高级主题](#27-高级主题)
  - [2.8 实战：从零写一个 Skill](#28-实战从零写一个-skill)
- [第三部分：Commands、Agents 与 MCP Server](#第三部分commandsagents-与-mcp-server)
  - [3.1 Commands（自定义命令）](#31-commands自定义命令)
  - [3.2 Agents（自定义 Agent 类型）](#32-agents自定义-agent-类型)
  - [3.3 MCP Server 集成](#33-mcp-server-集成)
- [第四部分：Hook 开发](#第四部分hook-开发)
  - [4.1 Hook 架构概览与核心引擎](#41-hook-架构概览与核心引擎)
  - [4.2 Hook 事件完整参考](#42-hook-事件完整参考)
  - [4.3 Hook 配置方式与加载机制](#43-hook-配置方式与加载机制)
  - [4.4 子 Agent 的 Hook 继承与行为](#44-子-agent-的-hook-继承与行为)
  - [4.5 实战示例与高级模式](#45-实战示例与高级模式)
  - [4.6 调试与故障排查](#46-调试与故障排查)
- [第五部分：插件扩展机制对比与选型](#第五部分插件扩展机制对比与选型)
- [附录](#附录)

---

## 第一部分：插件体系概述

> 面向 ZCode 桌面版插件开发者。本章覆盖插件系统架构、plugin.json 完整参考、目录结构、注册加载机制及开发 vs 市场分发模式。

---

### 1.0 本机目录结构（必读）

ZCode 在本机涉及**两个核心目录**，角色完全不同：

```
/Applications/ZCode.app/          ← 应用本体（只读，引擎源码在此）
    └── Contents/Resources/glm/zcode.cjs   ← Hook 引擎实现

~/.zcode/                         ← 用户数据目录（读写，插件开发者操作这里）
    ├── cli/
    │   ├── config.json           ← 插件注册表
    │   └── plugins/
    │       ├── cache/            ← 已安装的官方插件（含 hooks/hooks.json）
    │       └── marketplaces/     ← 插件市场源文件
    ├── skills/                   ← 手动安装的技能
    ├── teams/                    ← Swarm 团队配置
    └── agents/                   ← agent 产物目录
```

| 目录 | 谁在用 | 可写？ | 内容 |
|------|--------|--------|------|
| `/Applications/ZCode.app/` | ZCode 自身 | ❌ 只读 | Electron 壳 + 内核 CJS（hook 引擎源码在此） |
| `~/.zcode/` | **插件开发者** | ✅ 读写 | 插件配置、hooks 定义、skills、本地开发插件 |

**插件开发者的操作全在 `~/.zcode/` 下**：写 `plugin.json`、建 `hooks/hooks.json`、调 `config.json` 注册表等。`/Applications/ZCode.app/` 只是分析引擎行为时参考的源码。

> 关于 `CLAUDE_` 环境变量前缀：ZCode 复用了 Claude Code 的插件运行时协议以保持生态兼容，所以环境变量沿用 `CLAUDE_` 前缀。这**不代表** ZCode 依赖或调用 Claude Code——它有自己的引擎。同时 ZCode 也支持等价的 `ZCODE_` 前缀（`${ZCODE_PLUGIN_ROOT}` 等）。

---


### 1.1 ZCode 插件是什么

**插件是 ZCode 的扩展单元**。通过插件，你可以为 ZCode 添加以下能力：

| 扩展类型 | 说明 | 示例 |
|---------|------|------|
| **Skills** | 注入 LLM 对话的专业知识/行为指令 | `superpowers:brainstorming`、`computer-use` |
| **Commands** | 自定义 slash 命令（/xxx） | `/coordinator`、`/swarm`、`/computer-use` |
| **Agents** | 自定义子 agent 类型 | `teammate`、`worker` |
| **Hooks** | 拦截工具调用、注入上下文的事件钩子 | PreToolUse、PostToolUse、SessionStart |
| **MCP Servers** | 通过 MCP 协议暴露自定义工具 | computer-use MCP、swarm MCP |

#### 两种清单格式

ZCode 支持两种插件清单格式：

| 格式 | 路径 | 说明 |
|------|------|------|
| **`.zcode-plugin/plugin.json`** | `<插件根>/.zcode-plugin/plugin.json` | ZCode 原生格式，**推荐使用** |
| **`.claude-plugin/plugin.json`** | `<插件根>/.claude-plugin/plugin.json` | 兼容格式，为与 Claude Code 生态互通而保留 |

ZCode 启动时会同时扫描两种格式。如果两者都存在，ZCode 原生格式优先。

---

### 1.2 plugin.json 完整 Schema

`plugin.json` 是插件的入口清单文件，定义插件的元数据、扩展点路径和配置。

#### 1.2.1 基础元数据字段

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `name` | `string` | ✅ 是 | 插件唯一标识符。建议使用 kebab-case，如 `"zcode-agent-teams"` |
| `version` | `string` | ✅ 是 | 语义化版本号（SemVer），如 `"0.1.0"` |
| `description` | `string` | 推荐 | 插件功能简述，会在插件列表中展示 |
| `author` | `object` | 否 | 作者信息：`{ "name": "...", "email": "...", "url": "..." }` |
| `license` | `string` | 否 | 许可证类型，如 `"MIT"`、`"Proprietary"` |
| `homepage` | `string` | 否 | 项目主页 URL |
| `repository` | `string` | 否 | 源码仓库 URL |

#### 1.2.2 扩展点路径字段

这些字段的值是**相对于插件根目录的目录路径**，告诉 ZCode 去哪里找对应扩展资源：

| 字段 | 类型 | 说明 |
|------|------|------|
| `skills` | `string` | Skills 目录，包含 `SKILL.md` 文件。如 `"skills"` |
| `commands` | `string` | Commands 目录，包含 `.md` 命令文件。如 `"commands"` |
| `agents` | `string` | Agents 目录，包含 agent 配置文件。如 `"agents"` |
| `hooks` | `string` | Hooks 目录，包含 `hooks.json` 及 hook 脚本。如 `"hooks"` |

**省略某个字段 = 该插件不提供此类扩展**。例如，如果一个插件只提供 skills，可以只写 `"skills": "skills"`。

#### 1.2.3 MCP Server 配置 (`mcpServers`)

`mcpServers` 是一个对象，key 为 server 名称，value 为 MCP server 启动配置：

```json
{
  "mcpServers": {
    "computer-use": {
      "command": "/Applications/ZCode.app/Contents/Frameworks/ZCode Helper.app/Contents/MacOS/ZCode Helper",
      "args": [
        "/Applications/ZCode.app/Contents/Resources/glm/zcode.cjs",
        "__zcode-plugin-host",
        "/Users/chenjing/zcode-computer-use-plugin/dist/server.js"
      ],
      "env": {
        "ZCODE_COMPUTER_USE_ALLOWLIST": "${user_config.allowlist_path}",
        "ELECTRON_RUN_AS_NODE": "1"
      }
    }
  }
}
```

**MCP Server 配置字段：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `command` | `string` | 启动 MCP server 的可执行文件路径 |
| `args` | `string[]` | 命令行参数数组 |
| `env` | `object` | 环境变量 key-value，支持变量插值 |

**特殊参数 `__zcode-plugin-host`：** 当 command 指向 ZCode Helper 且 args 包含 `__zcode-plugin-host` 时，ZCode 以插件宿主模式启动——由 ZCode 进程管理 MCP server 的生命周期。

**环境变量插值：** `env` 值支持两种变量语法：

| 变量 | 解析为 |
|------|--------|
| `${ZCODE_PLUGIN_ROOT}` / `${CLAUDE_PLUGIN_ROOT}` | 插件根目录绝对路径 |
| `${ZCODE_PROJECT_DIR}` / `${CLAUDE_PROJECT_DIR}` | 当前项目工作目录 |
| `${user_config.<key>}` | 用户在 `userConfig` 中配置的值 |

#### 1.2.4 用户配置 (`userConfig`)

让用户在 `~/.zcode/cli/config.json` 的插件条目中自定义参数：

```json
{
  "userConfig": {
    "allowlist_path": {
      "type": "string",
      "default": "",
      "description": "白名单文件路径。为空时用 ~/.zcode/computer-use/allowlist.json"
    }
  }
}
```

每个 userConfig key 是一个配置项，包含：
- `type`: 值类型（`"string"`、`"boolean"`、`"number"`）
- `default`: 默认值
- `description`: 说明文字

#### 1.2.5 完整示例

**示例 1：简单 Skills 插件（Superpowers）**

```json
{
  "name": "superpowers",
  "version": "5.1.0",
  "description": "Planning, TDD, debugging, and delivery workflows for coding agents.",
  "author": { "name": "Jesse Vincent", "email": "jesse@fsck.com", "url": "https://github.com/obra" },
  "homepage": "https://github.com/obra/superpowers",
  "repository": "https://github.com/obra/superpowers",
  "license": "MIT",
  "skills": "skills"
}
```

**示例 2：带 Commands、Agents、MCP 的完整插件（agent-teams）**

```json
{
  "name": "zcode-agent-teams",
  "version": "0.1.0",
  "description": "ZCode 多 Agent 编排插件。提供 /coordinator 和 /swarm 两种并行模式。",
  "author": { "name": "chenjing" },
  "license": "MIT",
  "skills": "skills",
  "commands": "commands",
  "agents": "agents",
  "mcpServers": {
    "swarm": {
      "command": "/Applications/ZCode.app/Contents/Frameworks/ZCode Helper.app/Contents/MacOS/ZCode Helper",
      "args": [
        "/Applications/ZCode.app/Contents/Resources/glm/zcode.cjs",
        "__zcode-plugin-host",
        "${ZCODE_PLUGIN_ROOT}/dist/server.cjs"
      ],
      "env": { "ELECTRON_RUN_AS_NODE": "1" }
    }
  }
}
```

**示例 3：带 userConfig 的插件（computer-use）**

```json
{
  "name": "zcode_computer_use",
  "version": "0.1.0",
  "description": "ZCode 桌面自动化插件。通过截图+鼠标键盘控制本机 macOS 桌面应用。",
  "author": { "name": "chenjing" },
  "license": "MIT",
  "skills": "skills",
  "commands": "commands",
  "mcpServers": {
    "computer-use": {
      "command": "/Applications/ZCode.app/Contents/Frameworks/ZCode Helper.app/Contents/MacOS/ZCode Helper",
      "args": [
        "/Applications/ZCode.app/Contents/Resources/glm/zcode.cjs",
        "__zcode-plugin-host",
        "/Users/chenjing/zcode-computer-use-plugin/dist/server.js"
      ],
      "env": {
        "ZCODE_COMPUTER_USE_ALLOWLIST": "${user_config.allowlist_path}",
        "ELECTRON_RUN_AS_NODE": "1"
      }
    }
  },
  "userConfig": {
    "allowlist_path": {
      "type": "string",
      "default": "",
      "description": "白名单文件路径。为空时用 ~/.zcode/computer-use/allowlist.json"
    }
  }
}
```

---

### 1.3 插件目录结构

#### 1.3.1 标准布局

```
my-zcode-plugin/
├── .zcode-plugin/
│   └── plugin.json              ← 插件清单（必需）
├── skills/                      ← Skills：每个子目录一个 SKILL.md
│   └── my-skill/
│       └── SKILL.md
├── commands/                    ← Commands：每个 .md 定义一个 slash 命令
│   └── my_command.md
├── agents/                      ← Agents：每个 .md 定义一个 agent 类型
│   └── my_agent.md
├── hooks/                       ← Hooks：hooks.json + 脚本
│   ├── hooks.json
│   └── my-hook.sh
├── dist/                        ← 编译产物（TypeScript 插件）
│   └── server.js
├── src/                         ← TypeScript 源码
├── package.json                 ← npm 依赖声明
└── tsconfig.json                ← TypeScript 配置
```

#### 1.3.2 各子目录说明

| 目录 | 内容 | 加载方式 |
|------|------|---------|
| `.zcode-plugin/` | `plugin.json` 清单 | 启动时读取，解析扩展点 |
| `skills/` | `SKILL.md` 文件（每个子目录一个） | 自动发现，按需触发 |
| `commands/` | `.md` 文件 | 注册为 slash 命令 |
| `agents/` | `.md` 文件 | 注册为子 agent 类型 |
| `hooks/` | `hooks.json` + 可执行脚本 | 事件发生时由 hook 引擎调用 |
| `dist/` | 编译后的 JS/CJS | MCP server 入口等 |

---

### 1.4 插件注册与加载

#### 1.4.1 配置注册表：`~/.zcode/cli/config.json`

插件系统的所有配置集中在 `~/.zcode/cli/config.json` 的 `plugins` 字段：

```json
{
  "plugins": {
    "enabledPlugins": {
      "ios-simulator@zcode-plugins-official": false,
      "superpowers@zcode-plugins-official": true
    },
    "dirs": [
      "/Users/chenjing/projects/claude-code/zcodeplugin",
      "/Users/chenjing/zcode-computer-use-plugin",
      "/Users/chenjing/zcode-agent-teams-plugin"
    ]
  }
}
```

**两个注册维度：**

| 字段 | 用途 | 格式 |
|------|------|------|
| `plugins.dirs` | 本地开发插件的目录列表 | 字符串数组，每个是插件根目录的绝对路径 |
| `plugins.enabledPlugins` | 市场插件的启用/禁用开关 | key 为 `"<plugin>@<marketplace>"`，value 为 boolean |

#### 1.4.2 加载时机与生命周期

```
ZCode 启动
    │
    ├─ 1. 读取 ~/.zcode/cli/config.json
    │
    ├─ 2. 扫描 plugins.dirs 中的每个目录
    │     ├─ 寻找 .zcode-plugin/plugin.json
    │     ├─ 寻找 .claude-plugin/plugin.json（fallback）
    │     └─ 解析扩展点（skills/commands/agents/hooks/mcpServers）
    │
    ├─ 3. 扫描 plugins/cache/<marketplace>/<plugin>/<version>/
    │     └─ 读取 enabledPlugins 状态决定是否加载
    │
    └─ 4. 构建完整的扩展注册表（一次性，无热重载）
```

**关键特性：**

- **一次性扫描**：插件在 ZCode **启动时**一次性扫描和加载。启动后新增/修改插件需要重启 ZCode 生效。
- **无热重载**：运行中不会重新扫描插件目录。这是有意为之——保证会话期间插件行为一致。
- **目录深度 1**：`plugins.dirs` 中的路径必须是插件根目录（直接包含 `.zcode-plugin/` 子目录的目录），ZCode 不会递归扫描。

#### 1.4.3 插件启用/禁用机制

**本地插件：** 从 `plugins.dirs` 中移除路径即可禁用。没有单独的 enable/disable 开关。

**市场插件：** 通过 `enabledPlugins` 控制。设为 `false` 后，即使插件文件仍在 cache 目录中，也不会被加载。

```json
{
  "enabledPlugins": {
    "superpowers@zcode-plugins-official": true,   // ✅ 启用
    "ios-simulator@zcode-plugins-official": false  // ❌ 禁用
  }
}
```

---

### 1.5 本机开发 vs 市场分发

#### 1.5.1 本机开发模式

**适用场景：** 正在开发自己的插件，需要频繁修改和测试。

**操作步骤：**

1. 创建插件目录结构（含 `.zcode-plugin/plugin.json`）
2. 在 `~/.zcode/cli/config.json` 的 `plugins.dirs` 中添加插件根目录路径：

```json
{
  "plugins": {
    "dirs": [
      "/Users/chenjing/my-cool-plugin"
    ]
  }
}
```

3. 重启 ZCode 使插件生效

**优点：**
- 即时修改源文件，重启即可测试
- 无需打包、版本号管理
- 适合快速迭代

#### 1.5.2 市场分发模式

**适用场景：** 从插件市场安装，或发布自己的插件供他人使用。

**安装路径：**

```
~/.zcode/cli/plugins/cache/
└── <marketplace>/              ← 市场标识，如 "zcode-plugins-official"
    └── <plugin>/               ← 插件名
        └── <version>/          ← 版本号
            ├── .zcode-plugin/
            │   └── plugin.json
            ├── .zcode-plugin-seed.json   ← 安装记录
            └── skills/...
```

**`.zcode-plugin-seed.json` 记录结构：**

```json
{
  "hash": "562250757a6d32ffe69eeb9a720ba32b40964b0d9d198980ed802edc6273abb9",
  "marketplace": "zcode-plugins-official",
  "plugin": "document-skills",
  "pluginVersion": "0.1.0",
  "source": "filesystem",
  "version": 1
}
```

| 字段 | 说明 |
|------|------|
| `hash` | 插件包完整性校验哈希 |
| `marketplace` | 来源市场名称 |
| `plugin` | 插件标识符 |
| `pluginVersion` | 安装的版本号 |
| `source` | 来源类型（`"filesystem"` 等） |
| `version` | seed 格式版本 |

安装后，插件默认启用。用户可通过 `enabledPlugins` 手动禁用。

---

### 1.6 环境变量

ZCode 支持两套环境变量前缀，功能等价。**推荐使用 `ZCODE_` 前缀**（语义更准确）：

| 推荐写法 | 兼容写法 | 含义 | 可用范围 |
|---------|---------|------|---------|
| `${ZCODE_PLUGIN_ROOT}` | `${CLAUDE_PLUGIN_ROOT}` | 插件根目录绝对路径 | mcpServers.env、hook 脚本 |
| `${ZCODE_PROJECT_DIR}` | `${CLAUDE_PROJECT_DIR}` | 当前项目工作目录 | hook 脚本 |
| `${ZCODE_SESSION_ID}` | `${CLAUDE_SESSION_ID}` | 当前会话 ID | hook 脚本 |
| — | `${CLAUDE_ENV_FILE}` | SessionStart hook 可写入以持久化环境变量 | SessionStart hook |

> **关于 `CLAUDE_` 前缀：** ZCode 复用了 Claude Code 的插件运行时协议以保持生态兼容，所以环境变量沿用 `CLAUDE_` 前缀。这**不代表** ZCode 依赖或调用 Claude Code——ZCode 的 Hook 引擎是完全独立实现的。

---

### 1.7 关键目录速查

| 路径 | 角色 | 操作 |
|------|------|------|
| `/Applications/ZCode.app/Contents/Resources/glm/zcode.cjs` | Hook 引擎源码（只读） | 分析参考 |
| `~/.zcode/cli/config.json` | 插件注册表 | **编辑**：添加本地插件、开关市场插件 |
| `~/.zcode/cli/plugins/cache/` | 已安装的市场插件 | 只读（由市场安装流程管理） |
| `~/.zcode/skills/` | 手动安装的技能 | 直接添加 SKILL.md |
| `<你的插件根>/.zcode-plugin/plugin.json` | 插件清单 | **编辑**：定义扩展点 |

## 第二部分：Skills 开发


Skills 是 ZCode 插件体系中最核心的 LLM 行为注入机制。一个 Skill 就是一份 Markdown 文件，在合适的时机被加载到模型上下文中，改变模型的决策逻辑、工作流程或知识边界。

---

### 2.1 Skill 是什么

一个 Skill 是一个自包含的 **Markdown 指令文件**（`SKILL.md`），它为 LLM 提供：

- **专业知识**：某个领域的参考信息（API 文档、工具用法、协议规范）
- **行为约束**：强制流程规则（"修改前必须先读文件"、"不改代码就写测试"）
- **工作流指导**：结构化步骤（"先 brainstorm → 再写 plan → 再实现 → 再 review"）

Skill 与普通 system prompt 的关键区别：

| 维度 | System Prompt | Skill |
|------|--------------|-------|
| 触发方式 | 始终加载 | 按需 / 关键词 / 上下文自动加载 |
| 作用域 | 全局 | 可限定项目/用户/插件 |
| 可组合性 | 单一 | 多个 skill 可叠加 |
| 管理方式 | 硬编码 | 文件系统，可启用/禁用 |
| 热更新 | 需重启 | 编辑 SKILL.md 即可（下次加载生效） |

#### ZCode 如何"使用"一个 Skill

当 ZCode 判断某个 Skill 应该被激活时，它会将该 Skill 的 `SKILL.md` 全文注入到当前对话的上下文中。模型收到这份指令后，将其作为最高优先级的行为规则执行——直到用户显式指令覆盖它。

> **优先级链**：用户显式指令 > Skill 指令 > 默认 system prompt

---

### 2.2 SKILL.md 格式规范

#### 2.2.1 文件结构

```
skills/
  <skill-name>/
    SKILL.md              ← 必须存在，主指令文件
    辅助文件.md             ← 可选：被 SKILL.md 引用
    references/            ← 可选：参考文档
    scripts/               ← 可选：辅助脚本
    examples/              ← 可选：示例
```

`SKILL.md` 是唯一的必需文件。目录名即 skill 的标识名（`name`）。

#### 2.2.2 Frontmatter

每个 SKILL.md 以 YAML frontmatter 开头（三横线包围）：

```yaml
---
name: <skill-name>                          # 必须：技能标识名
description: <触发描述>                       # 必须：何时触发此技能的自然语言描述
---
```

真实示例——来自 `superpowers:using-superpowers`：

```yaml
---
name: using-superpowers
description: Use when starting any conversation - establishes how to find and use skills, requiring Skill tool invocation before ANY response including clarifying questions
---
```

来自 `zcode_computer_use:computer-use`：

```yaml
---
name: computer-use
description: 通过 computer-use MCP 工具控制本机 macOS 桌面应用（截图、鼠标、键盘），搭配 ZCode 内置 analyze_image 工具理解屏幕。
---
```

来自 `zcode-agent-teams:swarm`：

```yaml
---
name: swarm
description: 进入 Swarm Mode 时激活。指导主 agent 作为 Team Lead，组建团队、拆分任务到共享列表、并行派发 teammate 子 agent、通过 mailbox 收报告并用 SendMessage 续传新任务给空闲 teammate。
---
```

##### Frontmatter 字段说明

| 字段 | 必须 | 类型 | 说明 |
|------|------|------|------|
| `name` | **是** | string | Skill 唯一标识。插件内的 skill 会自动获得 `插件名:` 前缀成为 qualified name（如 `superpowers:brainstorming`） |
| `description` | **是** | string | 自然语言描述，用于 ZCode 判断"此 skill 是否适用于当前用户请求"。写得好坏直接影响 skill 的触发准确率 |

> **注意**：ZCode 的 SKILL.md 格式是 Claude Code 的简化版。它只支持 `name` 和 `description` 两个 frontmatter 字段，不支持 `model`、`tools`、`mcpServers` 等 Claude Code 扩展字段。

#### 2.2.3 Body 内容

Frontmatter 之后是 Markdown body。这是注入到 LLM 上下文的实际指令。根据 Skill 的类型不同，body 的编写方式也不同。

##### 流程控制型 Skill

使用明确的步骤、检查清单和强制门禁。典型模式：

```markdown
## Skill Name

### Overview
一句话说明此 Skill 做什么。

### When to Use
明确触发场景列表。

### The Iron Law
```
核心规则，不可违反
```

### Phase 1: ...
步骤说明（带 checkbox checklist 最佳）

### Checklist
- [ ] 步骤 1
- [ ] 步骤 2
```

示例：`superpowers:test-driven-development` 中定义了 RED-GREEN-REFACTOR 三步循环，每一步都有明确的"必须做"和"禁止做"。

##### 知识注入型 Skill

提供领域知识、工具参考或 API 文档。典型模式：

```markdown
## Skill Name

### MCP Tool Names
列出可用的工具及其调用方式

### Core Rules
关键使用规则

### Tool Reference
逐个工具说明参数和返回值

### 默认工作流
推荐的调用顺序
```

示例：`zcode_computer_use:computer-use` 列出了所有 computer-use MCP 工具的用法和坐标系统。

##### 行为约束型 Skill

强制模型遵守特定行为模式。典型模式：

```markdown
---
name: <name>
description: Use when <触发场景>
---

<EXTREMELY-IMPORTANT>
如果存在 1% 的可能性某 skill 适用，你必须调用它。
这不是可协商的。
</EXTREMELY-IMPORTANT>

### Red Flags
这些想法意味着 STOP：
| 想法 | 现实 |
|------|------|
| "这太简单了" | 简单的事也会出错 |
```

#### 2.2.4 支持文件与子目录

Skill 目录下可以包含任意辅助文件：

| 路径模式 | 用途 | 示例 |
|----------|------|------|
| `references/*.md` | 跨平台工具映射 | `using-superpowers/references/copilot-tools.md` |
| `scripts/*` | 辅助脚本（shell/python） | `brainstorming/scripts/server.cjs` |
| `*-prompt.md` | 子 agent prompt 模板 | `subagent-driven-development/implementer-prompt.md` |
| `examples/*` | 示例 | `writing-skills/examples/CLAUDE_MD_TESTING.md` |
| `*.md`（同级） | 补充文档（被 SKILL.md 引用） | `test-driven-development/testing-anti-patterns.md` |

支持文件的引用方式：SKILL.md 中用相对路径引用，如 `references/copilot-tools.md`，ZCode 不会自动加载它们——只在 Skill 内容中明确指示模型去读时才访问。

---

### 2.3 Skill 的发现与加载机制

#### 2.3.1 发现路径（优先级从高到低）

ZCode 从以下位置发现 Skill：

| 优先级 | 路径 | 说明 |
|--------|------|------|
| 1 | `<项目>/.zcode/skills/<name>/SKILL.md` | 项目级覆盖（优先级最高） |
| 2 | `<项目>/.agents/skills/<name>/SKILL.md` | 项目级 Agent Skills |
| 3 | `~/.zcode/skills/<name>/SKILL.md` | 用户级 ZCode Skills |
| 4 | `~/.agents/skills/<name>/SKILL.md` | 用户级 Agent Skills |
| 5 | 已启用插件的 `<plugin>/skills/<name>/SKILL.md` | 插件提供的 Skills |

同一 `name` 出现在多个位置时，高优先级的覆盖低优先级。本地 `~/.zcode/skills/` 下的同名 skill 会覆盖插件提供的版本——可用于个性化调整官方 skill。

#### 2.3.2 注册方式

##### 方式一：通过插件注册（推荐用于分发）

在 `plugin.json` 中指定 `skills` 字段，指向 skills 目录：

```json
{
  "name": "superpowers",
  "version": "5.1.0",
  "skills": "skills"
}
```

ZCode 启动时扫描 `skills/` 下的所有子目录，每个含 `SKILL.md` 的子目录注册为一个 Skill。Skill 的 qualified name 为 `插件名:目录名`，如 `superpowers:brainstorming`。

##### 方式二：手动安装（用于个人使用）

直接将 Skill 目录放到 `~/.zcode/skills/` 或项目的 `.zcode/skills/` 下：

```bash
## 用户级安装
mkdir -p ~/.zcode/skills/my-custom-skill
cp SKILL.md ~/.zcode/skills/my-custom-skill/

## 项目级安装
mkdir -p .zcode/skills/project-specific-skill
cp SKILL.md .zcode/skills/project-specific-skill/
```

也可以通过符号链接指向外部目录（本机 `~/.zcode/skills/` 下就是符号链接）：

```bash
ls -la ~/.zcode/skills/
## ask-user-question -> /Users/<user>/.claude/skills/ask-user-question
## banana-2 -> /Users/<user>/.claude/skills/banana-2
```

##### 方式三：通过市场安装

通过 ZCode 插件市场安装的插件会自动缓存到 `~/.zcode/cli/plugins/cache/zcode-plugins-official/<插件名>/<版本>/`，其中的 `skills/` 目录会被自动发现。

#### 2.3.3 启用与禁用

在 `~/.zcode/cli/config.json` 中，可以对单个 Skill 进行细粒度控制：

**禁用单个 Skill：**
```json
{
  "skills": {
    "/Users/<user>/.agents/skills/engineering-skills/a11y-audit/SKILL.md": {
      "enable": false
    }
  }
}
```

**插件级别的启用/禁用：**
```json
{
  "plugins": {
    "enabledPlugins": {
      "superpowers@zcode-plugins-official": true,
      "ios-simulator@zcode-plugins-official": false
    }
  }
}
```

#### 2.3.4 Skill 的触发方式

ZCode 中 Skill 有三种触发方式：

##### 1. 用户显式调用（`/skill-name` 或 `/插件名:skill-name`）

用户在输入中以 `/` 开头引用 skill 名称，ZCode 会在发送前提示用户加载该 skill。例如：

```
/computer-use 帮我打开浏览器搜索 ZCode 最新文档
```

ZCode 匹配到 `computer-use` skill 后，调用 `Skill` 工具将 SKILL.md 内容注入上下文。

##### 2. 模型自主调用（Skill 工具）

模型在对话中，当判断某个 Skill 可能对当前任务有帮助时，可以主动调用 `Skill` 工具：

```
Skill({ skill: "superpowers:brainstorming" })
```

`using-superpowers` skill 的核心规则就是："即使只有 1% 的可能性某 skill 适用，你也必须调用它。"

##### 3. SessionStart Hook 注入（自动加载）

插件可以通过 `SessionStart` Hook 在每次会话开始时自动注入 Skill 内容。这是 Superpowers 插件的核心机制：

**hooks.json：**
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```

**session-start 脚本核心逻辑：**
```bash
using_superpowers_content=$(cat "${PLUGIN_ROOT}/skills/using-superpowers/SKILL.md")
## 将内容作为 additionalContext 注入
printf '{"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": "%s"}}' "$session_context"
```

这样，每次会话开始时，`using-superpowers` skill 的完整内容就被注入到上下文中，确保模型始终知道如何发现和使用 Skills。

#### 2.3.5 加载时机总结

| 触发方式 | 时机 | 适用场景 |
|----------|------|----------|
| SessionStart Hook | 会话开始时 | 全局行为规则、Skill 发现框架（如 using-superpowers） |
| 用户 `/` 命令 | 用户显式请求时 | 领域专业技能（如 computer-use） |
| 模型 Skill 工具 | 模型自主判断时 | 流程型 Skill（如 brainstorming、TDD、debugging） |

---

### 2.4 Skill 的分类体系

#### 2.4.1 按内容类型分类

来自 `superpowers:writing-skills` 的官方分类：

| 类型 | 说明 | 示例 |
|------|------|------|
| **Technique（技术）** | 有具体步骤的方法论 | `systematic-debugging`（4 阶段调试法） |
| **Pattern（模式）** | 思考问题的方式 | `brainstorming`（先设计再实现） |
| **Reference（参考）** | API 文档、工具用法 | `computer-use`（MCP 工具参考） |

#### 2.4.2 按执行刚性分类

来自 `superpowers:using-superpowers` 的分类：

| 类型 | 特征 | 示例 |
|------|------|------|
| **Rigid（刚性）** | 严格按步骤，不可跳过，不可"适应" | `test-driven-development`、`systematic-debugging` |
| **Flexible（柔性）** | 吸收原则，根据上下文灵活应用 | `writing-plans`（模式指南） |

刚性 Skill 的特点：
- 包含 "Iron Law"（铁律）——不可违反的核心规则
- 包含 "Red Flags"（红旗）——列出模型可能产生的合理化借口
- 包含 "Anti-Patterns"（反模式）——常见错误做法
- 使用 `<EXTREMELY-IMPORTANT>` 等强调标签

柔性 Skill 的特点：
- 提供原则和指南，不强制具体步骤
- 允许根据项目上下文调整
- 更多"推荐"而非"必须"

---

### 2.5 如何写一个好的 Skill

#### 2.5.1 描述（description）的写法

`description` 是 ZCode 判断"是否加载此 skill"的关键依据。写作要点：

**好的 description：**
```yaml
description: Use when encountering any bug, test failure, or unexpected behavior, before proposing fixes
```

- 以 "Use when" 开头
- 描述具体的触发场景和触发时机
- 包含关键词（bug、test failure、unexpected behavior）

**不好的 description：**
```yaml
description: A debugging helper
```

- 太模糊，模型无法判断何时适用
- 缺少触发条件

**中文 description 示例：**
```yaml
description: 通过 computer-use MCP 工具控制本机 macOS 桌面应用（截图、鼠标、键盘），搭配 ZCode 内置 analyze_image 工具理解屏幕。
```

#### 2.5.2 Body 编写最佳实践

1. **开门见山**：第一段就说明"你是什么"和"你该什么时候用"
2. **使用强调标签**：`<EXTREMELY-IMPORTANT>`、`<HARD-GATE>`、`<Good>`/`<Bad>` 对比块
3. **提供检查清单（Checklist）**：用 `- [ ]` 让模型逐项确认
4. **列出反模式（Anti-Patterns）**：预先堵住模型可能走的捷径
5. **给出红旗（Red Flags）**：列举模型可能产生的合理化借口，配上"现实"列
6. **画决策流程图**：用 Graphviz DOT 或 ASCII 流程图明确决策路径
7. **防止跳过**：刚性 Skill 要声明 "This is not negotiable. This is not optional."
8. **给出具体示例**：Good vs Bad 对比能让模型更准确理解意图

#### 2.5.3 Skill 开发流程

来自 `superpowers:writing-skills` 的 TDD 方法：

```
1. 确定需求：这个 Skill 要解决什么问题？
2. 建立基线：在没有 Skill 的情况下，模型会怎么错？
3. 写 SKILL.md 初稿：针对性地堵住那些错误
4. 测试：用真实 prompt 测试，看模型是否遵守
5. 迭代：发现新的绕过方式 → 补充规则 → 重新测试
```

核心原则：**如果你没见过模型在缺少此 Skill 时犯错，你就不知道这个 Skill 是否真的有效。**

#### 2.5.4 常见陷阱

| 陷阱 | 后果 | 解决方案 |
|------|------|----------|
| description 太宽泛 | Skill 被频繁误触发 | 加入精确的触发关键词 |
| description 太窄 | Skill 不被触发 | 覆盖更多同义表达 |
| 规则太多 | 模型产生"规则疲劳"，选择性忽略 | 分层：铁律（必须）→ 指南（推荐） |
| 没有 Red Flags | 模型找到合理化借口跳过规则 | 预测模型可能的抵抗，提前堵住 |
| 只有步骤没有检查清单 | 模型跳过步骤 | 用 `- [ ]` 强制逐项确认 |
| 过长的示例 | 占用过多上下文预算 | 精简到最小必要示例 |

---

### 2.6 官方 Superpowers 技能体系分析

Superpowers 是 ZCode 最重要的官方技能集（版本 5.1.0），包含 15 个 Skill，覆盖完整的开发工作流。

#### 2.6.1 完整技能列表

| 技能 | 类型 | 刚性 | 用途 |
|------|------|------|------|
| `using-superpowers` | Reference | 刚性 | 教导模型如何发现和使用 Skills。SessionStart 自动注入 |
| `brainstorming` | Pattern | 刚性 | 在写任何代码之前先做设计：探索上下文 → 提问 → 提出方案 → 获得批准 |
| `writing-plans` | Pattern | 柔性 | 将设计转化为详细的可执行实现计划 |
| `subagent-driven-development` | Technique | 刚性 | 用独立子 agent 逐个执行计划任务 |
| `executing-plans` | Technique | 刚性 | 在同一个会话中批量执行计划任务 |
| `test-driven-development` | Technique | 刚性 | RED-GREEN-REFACTOR 循环，严格要求先写测试 |
| `systematic-debugging` | Technique | 刚性 | 4 阶段调试：根因调查 → 模式分析 → 修复 → 预防 |
| `verification-before-completion` | Technique | 刚性 | 声称"完成"前必须运行验证命令并提供证据 |
| `requesting-code-review` | Technique | 柔性 | 完成任务后请求代码审查 |
| `receiving-code-review` | Technique | 刚性 | 收到审查反馈后的处理流程 |
| `finishing-a-development-branch` | Technique | 柔性 | 实现完成后的合并/PR/清理决策 |
| `dispatching-parallel-agents` | Technique | 柔性 | 并行派发多个独立子 agent |
| `using-git-worktrees` | Technique | 刚性 | 用 git worktree 隔离特性开发 |
| `writing-skills` | Technique | 刚性 | 用 TDD 方法创建和迭代 Skills |
| `skill-creator` (来自 skill-creator 插件) | Technique | 柔性 | 简化的 Skill 创建助手 |

#### 2.6.2 设计模式总结

Superpowers 技能集体现了以下设计原则：

1. **技能链（Skill Chain）**：技能之间有明确的上下游关系
   - `brainstorming` → `writing-plans` → `subagent-driven-development` / `executing-plans`
   - 每个技能的终态是调用下一个技能

2. **入口技能（Entry Skill）**：`using-superpowers` 是整个体系的入口，SessionStart 自动注入，确保模型始终知道如何访问其他技能

3. **刚性守卫（Rigid Guard）**：`test-driven-development` 和 `systematic-debugging` 使用 `<HARD-GATE>` 标签和 "Iron Law" 阻止模型走捷径

4. **红旗机制（Red Flags）**：预测模型可能产生的合理化借口并预先驳斥。TDD skill 中有 13 条 "Common Rationalizations"

5. **检查清单驱动（Checklist-Driven）**：几乎所有流程型 Skill 都使用 `- [ ]` checklist 保证步骤完整

6. **平台适配层**：`using-superpowers` 的 `references/` 目录为不同平台（Claude Code、Copilot CLI、Codex）提供工具名映射

---

### 2.7 高级主题

#### 2.7.1 Skill 与 Hook 的组合

Skill 和 Hook 是最佳拍档：

- **Hook** 负责"何时注入"
- **Skill** 负责"注入什么"

典型模式：`SessionStart` Hook + `using-superpowers` Skill。Hook 在每次会话启动时触发，将 `using-superpowers` 的 SKILL.md 内容作为 `additionalContext` 注入，确保 Skills 体系始终可用。

#### 2.7.2 Skill 优先级与覆写

同名 Skill 的优先级：项目 `.zcode/skills/` > 项目 `.agents/skills/` > 用户 `~/.zcode/skills/` > 用户 `~/.agents/skills/` > 插件 `skills/`

这意味着可以：
- 在项目中覆盖插件的 Skill（放置同名的 SKILL.md）
- 为个人调整官方 Skill（放置到 `~/.zcode/skills/`）
- 通过 `config.json` 的 `skills` 字段禁用特定 Skill

#### 2.7.3 上下文预算管理

每个 Skill 注入的 Markdown 会消耗模型的上下文窗口。ZCode 提供了 `skills.metadataBudget` 配置（默认 8000 字节）来控制 Skill 元数据的上下文占用。

编写 Skill 时注意：
- 精简但不失完整性
- 长示例放到支持文件中（按需引用）
- 使用 `<Good>` / `<Bad>` 对比代替冗长解释
- 每句话都应该有明确的目的——如果删掉不影响行为，就删掉

#### 2.7.4 Skill 的 qualified name

插件提供的 Skill 自动获得 `<插件名>:<skill名>` 的 qualified name：

- 插件 `superpowers` 中的 `brainstorming` → `superpowers:brainstorming`
- 插件 `zcode_computer_use` 中的 `computer-use` → `zcode_computer_use:computer-use`
- 插件 `zcode-agent-teams` 中的 `swarm` → `zcode-agent-teams:swarm`

手动安装的 Skill（`~/.zcode/skills/` 或 `.zcode/skills/`）不自动添加插件前缀，直接使用目录名。

用户调用时可以用短名（如果唯一）或全限定名。模型调用 `Skill` 工具时使用全限定名最可靠。

---

### 2.8 实战：从零写一个 Skill

以下是一个最小完整的 Skill 模板：

```markdown
---
name: my-code-review-checklist
description: Use when reviewing code changes, before approving or merging any PR
---

## Code Review Checklist

### Overview

A systematic checklist for reviewing code changes. Use before approving any PR or merge request.

### When to Use

- Before clicking "Approve" on a PR
- After reading through the diff
- When asked "can you review this?"

### The Checklist

Before approving, verify ALL of the following:

- [ ] **Correctness**: Does the code do what it claims to do?
- [ ] **Tests**: Are there tests for the new behavior? Do they pass?
- [ ] **Error Handling**: Are edge cases and error states handled?
- [ ] **Naming**: Are variables, functions, and files clearly named?
- [ ] **No Dead Code**: Are there no commented-out blocks or unused imports?
- [ ] **Security**: No secrets, no SQL injection, no unsafe deserialization?

### Red Flags

| Thought | Reality |
|---------|---------|
| "It's a small change, I can skim it" | Small changes cause the worst bugs |
| "The tests pass, it's fine" | Tests passing ≠ code is good |
| "I trust this developer" | Everyone makes mistakes. Review anyway. |

### Output Format

When review is complete, output:

```
### Code Review Result

**Status:** [APPROVED / CHANGES REQUESTED]

**Summary:** [1-2 sentence summary]

**Issues Found:**
- [ ] Issue 1 (severity: high/medium/low)
- [ ] Issue 2 (severity: high/medium/low)

**Suggestions:**
- Suggestion 1
- Suggestion 2
```
```

将此文件保存为 `~/.zcode/skills/my-code-review-checklist/SKILL.md`，重启 ZCode 后即可在对话中使用 `/my-code-review-checklist` 触发。


## 第三部分：Commands、Agents 与 MCP Server



---

### 3.1 Commands（自定义命令）

**Commands** 是插件注册的 **slash 命令**（如 `/coordinator`、`/swarm`、`/computer-use`）。用户输入 `/命令名` 后，ZCode 将命令对应的内容注入到 LLM 对话中。

#### 3.1.1 注册方式

在 `plugin.json` 中声明 `commands` 字段，指向命令文件所在目录：

```json
{
  "commands": "commands"
}
```

然后在 `commands/` 目录下为每个命令创建一个 `.md` 文件。文件名（不含扩展名）即命令名称。

**示例目录结构：**

```
commands/
├── coordinator.md    → 注册为 /coordinator
├── swarm.md          → 注册为 /swarm
└── computer_use.md   → 注册为 /computer-use
```

#### 3.1.2 命令文件格式

每个命令 `.md` 文件包含 **YAML frontmatter** + **Markdown 正文**：

```markdown
---
description: 进入蜂群（Swarm）模式，组建多 agent 团队并行处理一批任务。
argument-hint: "[要并行处理的任务批次描述]"
skills: swarm
---

<system-reminder>
你现在处于 **Swarm Mode（蜂群模式）**，角色是 **Team Lead（团队负责人）**。

你的职责是组建团队、拆分任务、调度多个 teammate 子 agent 并行执行、综合结果。
teammate 之间通过文件邮箱通信，通过共享任务列表竞争认领任务。

立即调用 `swarm` skill，按其中的 team-lead 流程处理用户的请求：

$ARGUMENTS
</system-reminder>
```

#### 3.1.3 Frontmatter 字段说明

| 字段 | 必需 | 说明 |
|------|------|------|
| `description` | ✅ 是 | 命令简介，在 `/help` 和命令补全中展示 |
| `argument-hint` | 推荐 | 参数占位提示，如 `"[任务描述]"` |
| `skills` | 否 | 触发命令时自动加载的 skill 名称（多个用逗号分隔） |

#### 3.1.4 变量替换

命令正文中可使用特殊变量：

| 变量 | 说明 |
|------|------|
| `$ARGUMENTS` | 用户在命令后输入的参数文本（如 `/swarm 编译项目` → 值为 `编译项目`） |

#### 3.1.5 命令执行流程

```
用户输入 /swarm 修复 3 个 bug
    │
    ├─ 1. ZCode 查找 commands/swarm.md
    ├─ 2. 若 frontmatter 中指定了 skills，自动加载对应 SKILL.md
    ├─ 3. $ARGUMENTS 替换为 "修复 3 个 bug"
    ├─ 4. 命令正文作为 system-reminder 注入 LLM 对话
    └─ 5. LLM 按命令指令执行
```

#### 3.1.6 完整示例：computer-use 命令

```markdown
---
description: 启动桌面自动化（computer use）工作流。
argument-hint: "[任务描述]"
skills: computer-use
---

Use the `computer-use` skill to accomplish this desktop task:

$ARGUMENTS

Start with `request_access` to get app permission (the skill routes confirmation
through `AskUserQuestion`), then `screenshot` to see the screen.
```

> **设计原则：** 命令文件应简短、精确。复杂的操作流程应委托给 `skills` 字段加载的 SKILL.md 文件，保持命令文件作为"入口+参数传递"的薄层。

---

### 3.2 Agents（自定义 Agent 类型）

**Agents** 允许插件定义**可被派发的子 agent 类型**。通过 `Agent` 工具（`subagent_type` 参数）启动对应类型的子 agent，用于并行/分布式任务执行。

#### 3.2.1 注册方式

在 `plugin.json` 中声明 `agents` 字段：

```json
{
  "agents": "agents"
}
```

然后在 `agents/` 目录下为每种 agent 类型创建 `.md` 文件：

```
agents/
├── teammate.md   → 注册为 subagent_type: "teammate"
└── worker.md     → 注册为 subagent_type: "worker"
```

#### 3.2.2 Agent 文件格式

每个 agent `.md` 文件包含 **YAML frontmatter** + **Markdown 正文（prompt 模板）**：

```markdown
---
name: teammate
description: Swarm 团队的 teammate 子 agent。
model: inherit
color: cyan
tools: ["*"]
mcpServers: ["plugin:zcode-agent-teams:swarm"]
---

You are a teammate in an agent swarm. You work alongside other teammates,
coordinated through a shared task list and a file-based mailbox...

# Agent Teammate Communication

**IMPORTANT:** You are running as an agent in a team...
```

#### 3.2.3 Frontmatter 字段说明

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | ✅ 是 | Agent 类型名，即 `Agent(subagent_type:"<name>")` 使用的值 |
| `description` | ✅ 是 | Agent 用途简述，在 agent 选择列表中展示 |
| `model` | 否 | 指定模型 ID，或 `"inherit"`（继承父 agent 的模型）。默认 `"inherit"` |
| `color` | 否 | 在 UI 中区分不同 agent 类型的颜色标识（如 `"cyan"`、`"green"`） |
| `tools` | 否 | 可用工具列表。`["*"]` 表示全部工具；也可指定子集如 `["Read", "Bash", "Write"]` |
| `mcpServers` | 否 | 该 agent 可用的 MCP server 列表。格式：`["plugin:<插件名>:<server名>"]` |

#### 3.2.4 Prompt 模板

正文部分即该类型 agent 的**系统 prompt 模板**。ZCode 在启动子 agent 时，将此内容作为系统指令注入。

**编写要点：**
- 明确子 agent 的职责边界
- 给出执行流程和决策规范
- 指定输出格式和回报标准
- 说明与其他 agent 的协作协议（如适用）

#### 3.2.5 示例对比：worker vs teammate

| 特性 | worker | teammate |
|------|--------|----------|
| 场景 | Coordinator 模式（星型编排） | Swarm 模式（团队+任务列表） |
| 通信 | 完成后自动回报 coordinator | 通过 mailbox 工具与 lead/teammate 通信 |
| 任务分配 | Coordinator 通过 prompt 直接派发 | 从共享 task list 竞争认领 |
| 工具集 | 标准工具（Read/Write/Bash/...） | 标准工具 + Swarm MCP（mailbox/task/team） |
| 自主性 | 单次任务，完成即返回 | 多轮循环认领任务，直到任务耗尽 |

#### 3.2.6 与 Swarm/Coordinator 模式的关系

```
Coordinator 模式                 Swarm 模式
─────────────────                ──────────
主 Agent (coordinator)           主 Agent (team-lead)
    │                                │
    ├─ Agent(type:"worker")          ├─ team_create()
    ├─ Agent(type:"worker")          ├─ task_create() × N
    └─ Agent(type:"worker")          ├─ Agent(type:"teammate")
                                     ├─ Agent(type:"teammate")
                                     └─ Agent(type:"teammate")
                                         │
                                         └─ 竞争认领 task_list
                                            mailbox 通信
```

两种模式都依赖插件提供的 agent 类型定义。`worker` agent 由 `zcode-agent-teams` 插件的 `agents/worker.md` 定义，`teammate` agent 由 `agents/teammate.md` 定义。

---

### 3.3 MCP Server 集成

**MCP（Model Context Protocol）Server** 是插件暴露自定义工具的标准方式。ZCode 启动 MCP server 进程后，server 通过 MCP 协议将工具注册到 LLM 可用工具列表中。

#### 3.3.1 注册方式

在 `plugin.json` 中声明 `mcpServers` 字段：

```json
{
  "mcpServers": {
    "swarm": {
      "command": "/Applications/ZCode.app/Contents/Frameworks/ZCode Helper.app/Contents/MacOS/ZCode Helper",
      "args": [
        "/Applications/ZCode.app/Contents/Resources/glm/zcode.cjs",
        "__zcode-plugin-host",
        "${ZCODE_PLUGIN_ROOT}/dist/server.cjs"
      ],
      "env": {
        "ELECTRON_RUN_AS_NODE": "1"
      }
    }
  }
}
```

#### 3.3.2 配置字段详解

| 字段 | 类型 | 说明 |
|------|------|------|
| `command` | `string` | 启动 MCP server 进程的可执行文件 |
| `args` | `string[]` | 命令行参数。支持 `${ZCODE_PLUGIN_ROOT}` 等变量插值 |
| `env` | `object` | 环境变量 key-value。支持变量插值和 `${user_config.<key>}` |

#### 3.3.3 插件宿主模式（`__zcode-plugin-host`）

当 `args` 数组包含 `__zcode-plugin-host` 参数时，ZCode 进入**插件宿主模式**：

1. ZCode Helper 进程启动
2. ZCode 加载 `zcode.cjs` 内核
3. 内核以 `__zcode-plugin-host` 模式运行，加载 `args` 中指定的 JS 入口文件
4. MCP server 在 ZCode 进程内运行，生命周期由 ZCode 管理

**为什么用插件宿主模式？**
- 复用 ZCode 已有的 MCP 基础设施（stdio transport、工具注册等）
- 生命周期与 ZCode 绑定（ZCode 退出时自动清理）
- 减少独立进程管理的复杂度

#### 3.3.4 环境变量插值

`env` 和 `args` 中支持以下变量：

| 变量 | 解析为 | 示例 |
|------|--------|------|
| `${ZCODE_PLUGIN_ROOT}` | 插件根目录绝对路径 | `/Users/chenjing/zcode-agent-teams-plugin` |
| `${CLAUDE_PLUGIN_ROOT}` | 同上（兼容写法） | 同上 |
| `${ZCODE_PROJECT_DIR}` | 当前项目工作目录 | `/Users/chenjing/ZCodeProject` |
| `${user_config.<key>}` | 用户在 `userConfig` 中配置的值 | `${user_config.allowlist_path}` |

#### 3.3.5 工具可见性

MCP server 注册的工具默认对**所有 agent 可见**。通过 agent 文件的 `mcpServers` 字段可精确控制：

```yaml
# agents/teammate.md frontmatter
mcpServers: ["plugin:zcode-agent-teams:swarm"]
```

格式：`"plugin:<插件名>:<mcpServers 中的 key>"`。

未在 agent 中声明 `mcpServers` 时，该 agent 可访问所有 MCP 工具。

#### 3.3.6 实战分析

**场景 A：computer-use MCP（独立工具服务器）**

```json
{
  "mcpServers": {
    "computer-use": {
      "command": ".../ZCode Helper",
      "args": [".../zcode.cjs", "__zcode-plugin-host", ".../dist/server.js"],
      "env": {
        "ZCODE_COMPUTER_USE_ALLOWLIST": "${user_config.allowlist_path}",
        "ELECTRON_RUN_AS_NODE": "1"
      }
    }
  }
}
```

- 提供截图、鼠标、键盘等桌面自动化工具
- 通过 `userConfig` 暴露白名单配置给用户
- `ELECTRON_RUN_AS_NODE=1` 允许在 Electron 环境中运行 Node.js

**场景 B：swarm MCP（团队协作工具）**

```json
{
  "mcpServers": {
    "swarm": {
      "command": ".../ZCode Helper",
      "args": [".../zcode.cjs", "__zcode-plugin-host", "${ZCODE_PLUGIN_ROOT}/dist/server.cjs"],
      "env": { "ELECTRON_RUN_AS_NODE": "1" }
    }
  }
}
```

- 提供 `team_*`、`task_*`、`mailbox_*` 等团队协作工具
- 使用 `${ZCODE_PLUGIN_ROOT}` 使路径可移植
- 工具仅对 `teammate` agent 可见（通过 agent frontmatter 限制）

---


## 第四部分：Hook 开发

> 本章完整保留原「ZCode 桌面版插件 Hook 开发指南」的全部内容，章节号重新编排为 4.1 ~ 4.6。

---



---

### 4.1 Hook 架构概览与核心引擎

#### 1.1 核心结论

**ZCode 拥有自己完整的独立 Hook 引擎**，引擎源码位于 `/Applications/ZCode.app/Contents/Resources/glm/zcode.cjs`（应用本体，只读）。核心类是 **`H4`（InMemoryHookRunner）**。

插件开发者不需要改这个文件——所有 hook 配置都在 `~/.zcode/` 或你的插件目录里完成。

#### 1.2 核心架构：H4（InMemoryHookRunner）

```javascript
// 还原后的核心结构
class H4 {  // InMemoryHookRunner
  hooks: HookEntry[];           // 注册的 hook 列表
  defaultTimeoutMs: number;     // 默认超时 60000ms
  emitEvent;                    // 事件发射器
  logger;

  register(t) { this.hooks.push(t); }     // 注册 hook

  async run(hookInput, opts) {
    // 1. 按 event + matcher 过滤匹配的 hook
    let matched = this.hooks.filter(h =>
      h.event === hookInput.hookEventName &&
      regexMatch(opts.matchValues, h.matcher)
    );

    let result = { additionalContexts: [] };

    // 2. 串行执行（for...of — 非并行！）
    for (let hook of matched) {
      let hookResult = await this.runCallbackWithTimeout(hook, hookInput, ...);
      mergeResult(result, hookResult);
      if (blocked) break;  // 一旦 block，短路停止
    }
    return result;
  }
}
```

**关键设计特点：**

- **串行执行**：同一事件上的多个 hook **按注册顺序串行执行**，不是并行
- **短路阻断**：一旦某个 hook 返回 `exit code 2`（block/deny），后续 hook 不再执行
- **结果合并**：所有 hook 的 `additionalContext` 会被拼接合并

#### 1.3 7 种 Hook 事件

ZCode 支持 **7 种** hook 事件（内部枚举 `Br`）：

```
SessionStart → UserPromptSubmit → PreToolUse → PermissionRequest
→ [工具执行] → PostToolUse | PostToolUseFailure → Stop
```

**不存在的事件（⚠️ 常见误区）：**

| 误以为存在的 | 真相 |
|-------------|------|
| SessionEnd | ❌ 不存在 |
| PreCompact | ❌ 不存在 |
| Notification | ❌ 不存在 |
| SubagentStop | ❌ 不存在（SubagentStopped 是 session event，不经过 hook 引擎） |

#### 1.4 工具调用中的 Hook 完整流水线

```
工具调用进入
    │
    ├─ 1. PreToolUse hook
    │     ├─ permissionDecision: "deny"  → 拒绝执行
    │     ├─ permissionDecision: "ask"   → 升级权限确认
    │     └─ updatedInput               → 替换工具输入参数
    │
    ├─ 2. 权限检查 → 需要升级？
    │     └─ PermissionRequest hook
    │           ├─ decision: "deny"  → 拒绝执行
    │           └─ decision: "allow" → 允许（可选 modifiedInput）
    │
    ├─ 3. 执行工具
    │
    ├─ 4. 成功 → PostToolUse hook
    │     └─ additionalContext → 注入 LLM 上下文
    │
    └─ 5. 失败 → PostToolUseFailure hook
          └─ additionalContext → 注入 LLM 上下文
```

#### 1.5 上下文注入机制

Hook 产出通过 **`hook_context`** 附件注入 LLM 消息历史。系统提示中包含：

> "Hooks may intercept tool calls; treat hook output as user feedback."
> "Hook feedback is system-provided context from ZCode runtime hooks."

#### 1.6 环境变量双前缀（不是依赖 Claude Code）

ZCode 同时支持 `CLAUDE_` 和 `ZCODE_` 两种前缀。`CLAUDE_` 是**历史兼容**——ZCode 复用了 Claude Code 的插件运行时协议，变量名沿用以确保生态互通。这**不代表** ZCode 依赖或调用 Claude Code；ZCode 的 hook 引擎是独立实现的。

**推荐插件开发者用 `ZCODE_` 前缀**（语义更准确），两种均可：

| 变量（推荐） | 兼容写法 | 含义 |
|-------------|---------|------|
| `${ZCODE_PLUGIN_ROOT}` | `${CLAUDE_PLUGIN_ROOT}` | 插件根目录绝对路径 |
| `${ZCODE_PROJECT_DIR}` | `${CLAUDE_PROJECT_DIR}` | 当前项目/工作目录 |
| `${ZCODE_SESSION_ID}` | `${CLAUDE_SESSION_ID}` | 当前会话 ID |
| — | `${CLAUDE_ENV_FILE}` | SessionStart 中可写入以持久化环境变量 |

---

### 4.2 Hook 事件完整参考

#### 2.1 通用协议

**所有 hook 通过 stdin 接收 JSON 输入**，公共字段：

```json
{
  "session_id": "sess_abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/dir",
  "permission_mode": "ask",
  "hook_event_name": "PreToolUse"
}
```

**通用输出格式（Pkt Schema）**：

```json
{
  "continue": true,
  "suppressOutput": false,
  "systemMessage": "显示给用户的 TUI 消息",
  "additionalContext": "注入 LLM 上下文",
  "decision": "block",
  "reason": "block 的原因",
  "hookSpecificOutput": {
    // 根据 hookEventName 不同而不同
  }
}
```

**Exit Code 规范**：

| 退出码 | 含义 | 行为 |
|--------|------|------|
| `0` | 成功/放行 | stdout JSON 生效 |
| `2` | 阻塞/拒绝 | 操作被阻止；stderr 反馈给 agent |
| 其他 | 非阻塞错误 | 操作继续，错误被记录 |

#### 2.2 事件速查表

| 事件 | 触发时机 | 能否阻止 | 支持 matcher | 主要输出 |
|------|---------|---------|-------------|---------|
| **SessionStart** | 会话启动 | ❌ | ❌ | additionalContext |
| **UserPromptSubmit** | 用户提交 prompt | ❌ | ❌ | additionalContext |
| **PreToolUse** | 工具执行前 | ✅ deny/ask | ✅ | permissionDecision, updatedInput |
| **PermissionRequest** | 权限升级时 | ✅ deny | ❌ | decision |
| **PostToolUse** | 工具执行成功后 | ❌ | ✅ | additionalContext |
| **PostToolUseFailure** | 工具执行失败后 | ❌ | ✅ | additionalContext |
| **Stop** | agent 想退出时 | ✅ block | ❌ | decision, reason |

#### 2.3 SessionStart

**触发时机**：ZCode 会话初始化时，在所有其他事件之前。

**stdin 特有字段**：无额外字段（仅有公共字段）。

**输出**：
```json
{
  "hookSpecificOutput": {
    "additionalContext": "上下文内容，注入会话首条消息"
  },
  "systemMessage": "TUI 消息"
}
```

**环境变量持久化**：通过 `${CLAUDE_ENV_FILE}` 路径写入 `export KEY=value`，变量将在整个会话中可用。

**典型用途**：项目类型检测、开发约定注入、环境初始化。

---

#### 2.4 UserPromptSubmit

**触发时机**：用户每次提交 prompt 时。

**stdin 特有字段**：
```json
{
  "user_prompt": "用户输入的原始文本"
}
```

**输出**：
```json
{
  "hookSpecificOutput": {
    "additionalContext": "注入到用户 prompt 之前的上下文"
  }
}
```

**典型用途**：捕获 git baseline、注入静态分析上下文。

---

#### 2.5 PreToolUse（⭐ 最重要的事件）

**触发时机**：任何工具执行**之前**。

**stdin 特有字段**：
```json
{
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf /tmp/test",
    "description": "..."
  },
  "tool_use_id": "toolu_xxx"
}
```

**输出**：
```json
{
  "hookSpecificOutput": {
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "拒绝/询问的原因",
    "updatedInput": { "command": "修改后的安全命令" }
  },
  "systemMessage": "给用户的 TUI 消息"
}
```

- `permissionDecision: "deny"` + `exit 2` → 操作被阻止
- `permissionDecision: "ask"` → 升级为权限请求
- `updatedInput` → 修改工具参数（压缩代码不做额外校验，直接替换）

**支持 matcher**：✅ 可按工具名过滤（`Bash`, `Write|Edit`, `mcp__.*` 等）。

**典型用途**：危险命令拦截、文件路径验证、MCP 操作控制。

---

#### 2.6 PermissionRequest

**触发时机**：PreToolUse 返回 `"ask"` 或需要权限升级时。

**stdin 特有字段**：包含触发的工具信息。

**输出**：
```json
{
  "hookSpecificOutput": {
    "decision": "allow|deny",
    "behavior": "always|once"
  }
}
```

**注意**：此事件在社区文档中未提及，但在源码 Br 枚举中存在。

---

#### 2.7 PostToolUse

**触发时机**：工具执行**成功**后。

**stdin 特有字段**：
```json
{
  "tool_name": "Write",
  "tool_input": { "file_path": "...", "content": "..." },
  "tool_result": { ... }
}
```

**输出**：
```json
{
  "hookSpecificOutput": {
    "additionalContext": "注入 LLM 上下文的反馈信息"
  },
  "systemMessage": "TUI 消息"
}
```

**支持 matcher**：✅。

**典型用途**：敏感信息扫描、代码风格检查、自动审计。

---

#### 2.8 PostToolUseFailure

**触发时机**：工具执行**失败**后。

**输出**：同 PostToolUse（additionalContext）。

**支持 matcher**：✅。

**典型用途**：错误诊断、自动重试建议。

---

#### 2.9 Stop（质量门禁）

**触发时机**：主 agent 或子 agent 准备停止/退出时。

**stdin 特有字段**：
```json
{
  "reason": "agent 停止的原因",
  "transcript_path": "/tmp/transcript.jsonl"
}
```

**输出**：
```json
{
  "decision": "approve|block",
  "reason": "如果 block，告诉 agent 为什么和接下来要做什么",
  "systemMessage": "🔄 迭代 N | 退出方式: ..."
}
```

**重试循环**：ZCode 源码 `gpn = 3`，Stop hook 最多可触发 **3 轮**重试。返回 `{ continue: true, additionalContext: "..." }` 启动新一轮对话。

**典型用途**：确保测试通过、编译成功、安全审查完成。

---

### 4.3 Hook 配置方式与加载机制

#### 3.1 方式一：`hooks/hooks.json` 文件（推荐）

插件根目录下创建 `hooks/hooks.json`，ZCode 启动时自动发现。

**文件位置**：
```
my-plugin/
├── plugin.json
└── hooks/
    └── hooks.json   ← ZCode 自动发现
```

**格式规范（带 `"hooks"` 外层包装）**：
```json
{
  "description": "可选：描述这组 hook 的用途",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node ${ZCODE_PLUGIN_ROOT}/hooks/validate-bash.js",
            "timeout": 30000
          }
        ]
      },
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${ZCODE_PLUGIN_ROOT}/hooks/check-secrets.py"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash ${ZCODE_PLUGIN_ROOT}/hooks/audit-file.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash ${ZCODE_PLUGIN_ROOT}/hooks/quality-gate.sh",
            "timeout": 120000
          }
        ]
      }
    ]
  }
}
```

| 约定 | 说明 |
|------|------|
| 顶层 `hooks` 键 | **必须**。ZCode 用此键提取实际 hook 定义 |
| `description` | 可选，用于文档说明，不影响行为 |
| 事件名 | 必须是 7 种合法事件之一（见第 2 章），否则静默忽略 |

#### 3.2 方式二：`plugin.json` 内联 `hooks` 字段（⭐ 源码关键发现）

ZCode 的 `plugin.json` 中的 `hooks` 字段远比文档描述的灵活——支持**字符串**、**对象**或**数组**三种值类型。

#### 字符串（文件路径引用）

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "hooks": "./custom-hooks.json"
}
```

#### 对象（内联定义，不需要外部文件！）

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node ${ZCODE_PLUGIN_ROOT}/hooks/bash-guard.js",
            "timeout": 10000
          }
        ]
      }
    ]
  }
}
```

#### 数组（混合，合并所有来源）

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "hooks": [
    "./base-hooks.json",
    {
      "SessionStart": [
        {
          "hooks": [
            {
              "type": "command",
              "command": "bash ${ZCODE_PLUGIN_ROOT}/hooks/inject-context.sh"
            }
          ]
        }
      ]
    }
  ]
}
```

| 类型 | 行为 |
|------|------|
| **字符串** | 当作文件路径，相对于 `plugin.json` 所在目录解析并加载 |
| **对象** | 当作 **内联 rawHooks** 直接使用，不需要外部文件 |
| **数组** | 混合配置，每项可以是字符串或对象，**合并**所有来源 |

**⚠️ 重要**：内联对象格式**不使用** `"hooks"` 外层包装——直接写事件名即可。这与 `hooks.json` 的格式不同！

```json
// ✅ plugin.json 内联格式（无 "hooks" 包装）
"hooks": { "PreToolUse": [...], "Stop": [...] }

// ❌ 不要在内联中再加 "hooks" 包装
"hooks": { "hooks": { "PreToolUse": [...] } }  // 错误！
```

#### 3.3 方式三：项目 `.claude/settings.json`

用户在项目根目录的 `.claude/settings.json` 中也可以定义 hook。

```json
{
  "PreToolUse": [
    {
      "matcher": "Bash",
      "hooks": [
        {
          "type": "command",
          "command": "./scripts/project-policy-check.sh"
        }
      ]
    }
  ],
  "Stop": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "./scripts/pre-exit-check.sh"
        }
      ]
    }
  ]
}
```

| 特性 | `hooks/hooks.json`（插件） | `.claude/settings.json`（项目） |
|------|---------------------------|-------------------------------|
| 外层 `hooks` 包装 | **需要** | **不需要** |
| 作用范围 | 插件全局 | 仅当前项目 |

#### 3.4 完整加载流程

```
┌─────────────────────────────────────────────────┐
│  1. i0t() — 从所有已安装插件收集 hook 源         │
│     • 读取 hooks/hooks.json（如存在）            │
│     • 读取 plugin.json → manifest.hooks 字段     │
│     • 字符串 → 文件路径解析                      │
│     • 对象 → 直接作为 rawHooks                   │
│     • 数组 → 遍历每项并合并                      │
├─────────────────────────────────────────────────┤
│  2. $Rr() — 白名单校验                           │
│     • 用 __o 对照 Br 枚举（7 种合法事件）        │
│     • 不合法事件名 → 静默丢弃                    │
├─────────────────────────────────────────────────┤
│  3. lvo() — 合并到 runtimeConfig                 │
│     • 插件 hook → 全局 runtimeConfig             │
│     • 有插件 hook → auto-enable hooks 功能       │
│     • 项目 .claude/settings.json 也参与合并      │
├─────────────────────────────────────────────────┤
│  4. cV() — 工厂函数创建 InMemoryHookRunner       │
│     • 输入：{ config: runtimeConfig.hooks }      │
│     • 输出：H4 (InMemoryHookRunner) 实例         │
├─────────────────────────────────────────────────┤
│  5. 运行时                                       │
│     • 启动时一次性加载                           │
│     • 不支持热重载（修改配置需重启 ZCode）        │
│     • hook 执行顺序 = 注册顺序（for...of 串行）   │
└─────────────────────────────────────────────────┘
```

#### 3.5 Matcher 匹配逻辑

Matcher 是 hook 系统的过滤机制，决定 hook 是否对当前工具调用生效。

| 情况 | 行为 |
|------|------|
| 无 matcher | **总是匹配**，对所有工具调用触发 |
| matcher 为字符串 | 作为正则表达式匹配工具名 |
| matcher 含 `\|` | 多值 OR，任一匹配即通过 |

**工具名别名（硬编码）**：

| 原名 | 别名 | 说明 |
|------|------|------|
| `Agent` | `Task` | 互相关联 |
| `ApplyPatch` | `Write`, `Edit` | 补丁应用 |

这意味着 `matcher: "Write"` 也会匹配 `ApplyPatch`，反之亦然。

**多值 OR**：
```json
{ "matcher": "Bash|Write|Edit|Read" }
// 匹配 Bash、Write、Edit 或 Read 中任意一个
```

#### 3.6 Hook 条目完整 Schema

```json
{
  "matcher": "Bash|Write",       // 可选：工具名匹配正则
  "hooks": [
    {
      "type": "command",         // 固定值
      "command": "node ...",     // 要执行的命令（支持环境变量模板）
      "timeout": 30000           // 可选：超时毫秒（默认 60000，最大 600000）
    }
  ]
}
```

| 字段 | 必需 | 说明 |
|------|------|------|
| `matcher` | 否 | 工具名正则过滤。省略表示匹配所有 |
| `hooks` | 是 | hook 执行项列表 |
| `hooks[].type` | 是 | 固定 `"command"` |
| `hooks[].command` | 是 | shell 命令，支持 `${ZCODE_PLUGIN_ROOT}` 等模板 |
| `hooks[].timeout` | 否 | 超时毫秒，默认 60000，最大 600000 |

#### 3.7 配置方式选择指南

| 维度 | hooks/hooks.json | plugin.json 内联 | .claude/settings.json |
|------|-----------------|------------------|-----------------------|
| 适用场景 | 复杂 hook 集 | 简单插件 | 项目特定策略 |
| 可移植性 | 高（随插件分发） | 高（随插件分发） | 低（绑定项目） |
| 版本管理 | 独立文件，易 diff | 混在 plugin.json | 项目内 |
| **推荐** | ⭐⭐⭐ 首选 | ⭐⭐ 便捷 | ⭐ 特定场景 |

---

### 4.4 子 Agent 的 Hook 继承与行为

#### 4.1 继承机制（源码证据）

子 agent 创建时**展开父 session 的完整 runtimeConfig**：

```javascript
// 子 agent 创建代码（还原）
return new Od(childSessionId, {
  ...parentSession.runtimeConfig,  // 展开父配置，包含 hooks！
  agentName: "zcode-workflow",
  mode: "yolo",
  subagents: { enabled: false },   // 孙 agent 硬编码禁用
}, ...);
```

子 agent（Od 实例）使用展开后的 runtimeConfig 创建**自己的独立 HookRunner**：

```javascript
// Od 构造函数内
let hookRunner = opts.hookRunner ?? (
  runtimeConfig.hooks?.enabled && opts.executionPort
    ? cV({ config: runtimeConfig.hooks })
    : undefined
);
this.hookRunner = hookRunner;
```

#### 4.2 生效对照表

| 事件 | 主 Agent | 子 Agent（worker/teammate） | 孙 Agent |
|------|---------|---------------------------|---------|
| PreToolUse | ✅ | ✅ | ❌ 被禁用 |
| PostToolUse | ✅ | ✅ | ❌ 被禁用 |
| PostToolUseFailure | ✅ | ✅ | ❌ 被禁用 |
| Stop | ✅ | ✅ | ❌ 被禁用 |
| SessionStart | ✅ | ❌ 子 agent 无此事件 | ❌ |
| UserPromptSubmit | ✅ | ❌ 子 agent 不直接接收用户 prompt | ❌ |
| PermissionRequest | ✅ | ✅ | ❌ 被禁用 |

#### 4.3 关键限制

| 限制 | 详情 |
|------|------|
| **孙 agent 禁用** | `subagents: { enabled: false }` 硬编码在子 agent 创建代码中 |
| **SubagentStop 不存在** | 子 agent 的停止事件是 Stop，没有独立的 SubagentStop |
| **两层最大深度** | 主 Agent → 子 Agent，孙 Agent 不可用 |
| **独立实例** | 每个 agent 有独立的 HookRunner 实例，但执行**相同规则** |

#### 4.4 实际影响

```
主 Agent（你）
  ├── PreToolUse hook ✅ → 工具调用被拦截
  ├── 派发 worker-A（子 agent）
  │     ├── PreToolUse hook ✅ → 同样被拦截（继承）
  │     ├── 派发 ??? → ❌ 不允许（孙 agent 禁用）
  │     └── Stop hook ✅ → 退出时被检查
  └── 派发 worker-B（子 agent）
        ├── PreToolUse hook ✅ → 同样被拦截
        └── Stop hook ✅
```

**这意味着**：插件中定义的 PreToolUse 安全校验 hook 会**自动保护所有 Coordinator worker 和 Swarm teammate**——一次定义，全局生效。

---

### 4.5 实战示例与高级模式

#### 5.1 示例 1：PreToolUse 阻止危险 Bash 命令

**Hook 配置**（`plugin.json` 内联）：
```json
{
  "name": "security-guard",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node ${ZCODE_PLUGIN_ROOT}/hooks/bash-guard.js",
            "timeout": 10000
          }
        ]
      }
    ]
  }
}
```

**Hook 脚本 `hooks/bash-guard.js`**：
```javascript
#!/usr/bin/env node
const chunks = [];
process.stdin.on('data', c => chunks.push(c));
process.stdin.on('end', () => {
  const input = JSON.parse(Buffer.concat(chunks).toString('utf8'));
  const cmd = input.tool_input?.command || '';

  const dangerous = [
    { re: /rm\s+(-[a-z]*r[a-z]*f[a-z]*)\s+\//i,  msg: 'rm -rf on root' },
    { re: />\s*\/dev\/sd[a-z]/i,                   msg: 'write to block device' },
    { re: /mkfs\./,                                 msg: 'filesystem creation' },
    { re: /curl.*\|\s*(ba)?sh/i,                   msg: 'curl pipe to shell' },
    { re: /chmod\s+(-R\s+)?777\s+\//i,             msg: 'chmod 777 on root' },
    { re: /:\s*\(\s*\)\s*\{/,                      msg: 'fork bomb' },
  ];

  for (const { re, msg } of dangerous) {
    if (re.test(cmd)) {
      console.log(JSON.stringify({
        hookSpecificOutput: {
          permissionDecision: 'deny',
          permissionDecisionReason: `🚫 ${msg}\nCommand: ${cmd.substring(0, 200)}`
        },
        systemMessage: `⛔ Hook: blocked dangerous command — ${msg}`
      }));
      process.exit(2);
    }
  }

  console.log(JSON.stringify({ hookSpecificOutput: { permissionDecision: 'allow' } }));
  process.exit(0);
});
```

---

#### 5.2 示例 2：PostToolUse 敏感信息扫描

**Hook 配置**：
```json
{
  "PostToolUse": [
    {
      "matcher": "Write|Edit",
      "hooks": [
        {
          "type": "command",
          "command": "node ${ZCODE_PLUGIN_ROOT}/hooks/secret-scanner.js",
          "timeout": 30000
        }
      ]
    }
  ]
}
```

**Hook 脚本 `hooks/secret-scanner.js`**（关键逻辑）：
```javascript
const rules = [
  { re: /sk-[a-zA-Z0-9]{32,}/,              label: 'OpenAI API Key' },
  { re: /ghp_[a-zA-Z0-9]{36,}/,             label: 'GitHub Token' },
  { re: /AKIA[0-9A-Z]{16}/,                 label: 'AWS Access Key' },
  { re: /-----BEGIN\s+PRIVATE\s+KEY-----/,   label: 'Private Key' },
  { re: /(password|secret)\s*[:=]\s*['"][^'"]+['"]/i, label: 'Hardcoded Password' },
];

const findings = rules
  .filter(r => r.re.test(content))
  .map(r => `- **${r.label}**: \`${content.match(r.re)[0].substring(0, 80)}\``);

if (findings.length > 0) {
  console.log(JSON.stringify({
    hookSpecificOutput: {
      additionalContext: `⚠️ **Security Hook**: found ${findings.length} potential secrets:\n${findings.join('\n')}\n\nPlease remove and rotate if exposed.`
    },
    systemMessage: `🔍 Hook: ${findings.length} potential secrets in ${filePath}`
  }));
}
process.exit(0);  // exit 0 = warn only, don't block
```

---

#### 5.3 示例 3：Stop 质量门禁

**Hook 配置**：
```json
{
  "Stop": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "node ${ZCODE_PLUGIN_ROOT}/hooks/quality-gate.js",
          "timeout": 120000
        }
      ]
    }
  ]
}
```

**Hook 脚本关键逻辑**：
```javascript
const failures = [];

// Check 1: TypeScript compilation
try { execSync('npx tsc --noEmit', { timeout: 60000 }); }
catch (e) { failures.push('TypeScript compilation failed'); }

// Check 2: Tests
try { execSync('npm test -- --passWithNoTests', { timeout: 120000 }); }
catch (e) { failures.push('Tests failed'); }

if (failures.length === 0) { process.exit(0); }

console.log(JSON.stringify({
  decision: 'block',
  reason: `## 🚧 Quality gate failed — ${failures.length} issues:\n${failures.join('\n')}\n\nFix all issues and try to exit again.`,
  systemMessage: `⛔ Hook: Quality gate blocked exit — ${failures.length} checks failed`
}));
process.exit(2);
```

> **注意**：ZCode 最多重试 3 轮（源码 `gpn = 3`），之后即使 hook 返回 block 也会强制退出。

---

#### 5.4 示例 4：SessionStart 项目上下文注入

**Hook 配置**：
```json
{
  "SessionStart": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "node ${ZCODE_PLUGIN_ROOT}/hooks/project-context.js",
          "timeout": 15000
        }
      ]
    }
  ]
}
```

**关键逻辑**：
```javascript
const contexts = [];

// Detect Node.js
if (fs.existsSync('package.json')) {
  const pkg = JSON.parse(fs.readFileSync('package.json', 'utf8'));
  contexts.push(`Node.js project: ${pkg.name} v${pkg.version}`);
}

// Detect Go
if (fs.existsSync('go.mod')) {
  const mod = fs.readFileSync('go.mod', 'utf8');
  const name = mod.match(/^module\s+(.+)$/m)?.[1];
  contexts.push(`Go project: ${name}`);
}

// Detect Python
if (fs.existsSync('pyproject.toml') || fs.existsSync('requirements.txt')) {
  contexts.push('Python project');
}

console.log(JSON.stringify({
  hookSpecificOutput: {
    additionalContext: `## Project Context (auto-detected)\n\n${contexts.join('\n')}`
  },
  systemMessage: `📋 Hook: injected project context (${contexts.length} items)`
}));
process.exit(0);
```

---

#### 5.5 示例 5：PermissionRequest 权限升级控制

```json
{
  "PermissionRequest": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "node ${ZCODE_PLUGIN_ROOT}/hooks/permission-policy.js"
        }
      ]
    }
  ]
}
```

**脚本**：
```javascript
const toolName = input.tool_name;
// 对特定工具始终拒绝（不允许 agent 自行批准）
const alwaysDeny = ['Bash', 'WebSearch', 'WebFetch'];
if (alwaysDeny.includes(toolName)) {
  console.log(JSON.stringify({
    hookSpecificOutput: { decision: 'deny' },
    systemMessage: `⛔ Auto-denied permission for ${toolName}`
  }));
  process.exit(2);
}
process.exit(0);
```

---

#### 5.6 示例 6：完整的插件 `plugin.json`（内联 + 文件混合）

```json
{
  "name": "my-security-plugin",
  "version": "1.0.0",
  "description": "Security hooks for ZCode",
  "hooks": [
    "./hooks/quality-gate.json",
    {
      "PreToolUse": [
        {
          "matcher": "Bash",
          "hooks": [
            {
              "type": "command",
              "command": "node ${ZCODE_PLUGIN_ROOT}/hooks/bash-guard.js",
              "timeout": 10000
            }
          ]
        }
      ],
      "PostToolUse": [
        {
          "matcher": "Write|Edit",
          "hooks": [
            {
              "type": "command",
              "command": "node ${ZCODE_PLUGIN_ROOT}/hooks/secret-scanner.js",
              "timeout": 30000
            }
          ]
        }
      ]
    }
  ]
}
```

---

### 4.6 调试与故障排查

#### 6.1 调试方法

| 方法 | 说明 |
|------|------|
| **stderr 输出** | exit code 2 时 stderr 自动反馈给 agent；exit 0 时 stderr 被记录到 transcript |
| **systemMessage** | 在 TUI 中可见（终端），是最直接的 hook 触发确认方式 |
| **additionalContext** | 注入到 LLM 上下文中，可以在 AI 回复中观察到是否生效 |
| **外部日志文件** | 在 hook 脚本中写入 `/tmp/zcode-hook-debug.log`，方便离线排查 |
| **echo 到 stderr** | `echo "DEBUG: matched $cmd" >&2` 可在 transcript 中看到 |

**通用调试脚本模板**：
```bash
#!/bin/bash
INPUT=$(cat)
EVENT=$(echo "$INPUT" | jq -r '.hook_event_name')

# 记录调试日志
echo "[$(date)] EVENT=$EVENT INPUT=$INPUT" >> /tmp/zcode-hook-debug.log

# 正常处理逻辑...
echo '{"continue":true}'
```

#### 6.2 常见问题诊断

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| **hook 完全不触发** | 事件名不在 Br 白名单中 | 检查是否用了 SessionEnd/PreCompact/Notification/SubagentStop —— 这些都不存在！只用 7 种合法事件 |
| **matcher 不匹配** | 正则语法错误或工具名不兼容 | 用简单 `"*"` 先验证 hook 能触发，再逐步收紧 matcher |
| **JSON 输出不生效** | stdout 输出了非 JSON 内容 | 确保 stdout **只**输出一行合法 JSON。调试信息写到 stderr |
| **子 agent hook 行为不一致** | 子 agent 继承 hook 但孙 agent 被禁用 | 这是设计行为：最多两层 agent 深度 |
| **修改不生效** | hooks.json 在启动时一次性加载 | **必须重启 ZCode** |
| **hook 执行顺序** | 串行 for...of，按注册顺序 | 先注册的先执行；一个 block 后停止后续 |

#### 6.3 已知限制清单

| 限制 | 影响 |
|------|------|
| 不支持运行时动态注册 hook | 修改配置后必须重启 ZCode |
| 孙 agent 硬编码禁用 | 子 agent 的子 agent 不触发任何 hook |
| 没有 SessionEnd 事件 | 无法在会话结束时执行清理 |
| 没有 PreCompact 事件 | 无法在上下文压缩前保存关键信息 |
| 没有 Notification 事件 | 无法自定义通知行为 |
| 没有 SubagentStop 事件 | 无法为子 agent 写专用的停止校验（统一用 Stop） |
| hook 输出最大 32768 字节 | `maxOutputBytes` 硬限制 |
| 默认超时 60000ms | command hook 默认 60 秒超时 |
| 只支持 command 类型 | 源码中 `Kan()` 函数支持 `command`（shell）和 `process`（argv），但后者未暴露 |

#### 6.4 调试检查清单

1. ✅ 事件名是 7 种合法事件之一吗？
2. ✅ `hooks.json` 有外层 `"hooks"` 包装吗？`plugin.json` 内联没有额外包装吗？
3. ✅ matcher 正则语法正确吗？注意工具名别名（Agent↔Task）
4. ✅ hook 脚本的 stdout 输出了合法 JSON 吗？
5. ✅ exit code 正确吗？（0=通过，2=block，不要用 1）
6. ✅ 修改后重启 ZCode 了吗？
7. ✅ 子 agent 确实继承 hook（孙 agent 被禁用是预期行为）
8. ✅ timeout 设置合理吗？（默认 60s，最大 600s）
9. ✅ 路径用了 `${ZCODE_PLUGIN_ROOT}` 而不是硬编码吗？
10. ✅ hook 脚本设置了 `set -euo pipefail`（bash）或等价错误处理吗？

---



## 第五部分：插件扩展机制对比与选型

> 综合对比 ZCode 五种扩展机制：Skills、Commands、Agents、MCP Servers 和 Hooks。帮助开发者在不同场景下做出正确的技术选择。

---

### 5.1 五种机制全景对比

| 维度 | Skills | Commands | Agents | MCP Servers | Hooks |
|------|--------|----------|--------|-------------|-------|
| **触发方式** | 自动（关键词/上下文） | 用户 `/命令` | `Agent` 工具派发 | 工具调用时自动 | 事件驱动（7 种事件） |
| **作用范围** | 当前对话 | 当前对话 | 子会话（独立上下文） | 工具调用 | 全局横切 |
| **主要用途** | 注入知识、行为约束、工作流引导 | 快捷启动复杂工作流 | 并行/分布式任务执行 | 暴露外部能力（API、系统操作） | 拦截、审计、上下文注入 |
| **文件格式** | `SKILL.md` | `.md` | `.md` | JS/TS + plugin.json | `hooks.json` + 脚本 |
| **能否阻止操作** | ❌ | ❌ | ❌ | ❌ | ✅（PreToolUse/Stop） |
| **能否注入上下文** | ✅ 直接 | ✅ 间接（通过 Skill） | ✅ prompt 模板 | ❌ | ✅ additionalContext |
| **状态** | 无状态 | 无状态 | 有状态（独立会话） | 取决于实现 | 无状态（每次调用） |
| **复杂度** | ⭐ 低 | ⭐⭐ 中 | ⭐⭐⭐ 高 | ⭐⭐⭐ 高 | ⭐⭐⭐ 高 |
| **开发成本** | 写一个 md 文件 | 写一个 md 文件 | 写 prompt + 可能需 MCP | 写 JS/TS + 配置 | 写脚本 + hooks.json |

### 5.2 决策指南：何时用哪个

```
你的需求                                         → 选择
────────────────────────────────────────────────────────────────
给 LLM 灌输某个领域的专业知识                       → Skill
定义一个可复用的操作流程                             → Skill / Command
让用户能一键启动某个工作流                           → Command（配合 Skill）
需要并行执行多个独立子任务                           → Agent（通过 Coordinator/Swarm）
需要访问外部 API 或系统功能                         → MCP Server
Agent 之间需要协调通信                               → MCP Server（提供通信工具）
需要在每次工具调用前后做安全检查                     → Hook（PreToolUse / PostToolUse）
需要确保 agent 退出前满足质量门禁                    → Hook（Stop）
需要在会话启动时注入项目上下文                       → Hook（SessionStart）
需要阻止模型执行危险操作                             → Hook（唯一方案！）
```

### 5.3 五种机制的典型组合

**组合 1：Command + Skill（最轻量）**

```
用户 /computer-use 截图 → Command 加载 Skill → Skill 指导模型操作
```

**组合 2：Hook + Skill（自动化上下文）**

```
SessionStart Hook → 自动注入 using-superpowers Skill → 模型始终知道如何发现 Skills
```

**组合 3：Agent + MCP + Hook（全栈）**

```
Team Lead
  ├─ Hook（PreToolUse）→ 监控所有 agent 的工具调用安全
  ├─ Agent(type:"teammate") → 使用 Swarm MCP 工具协作
  └─ Hook（Stop）→ 质量门禁
```

**组合 4：Hook 作为横切层（全局安全网）**

```
所有工具调用
  │
  ├─ [PreToolUse Hook] → 安全检查（阻止危险命令）
  ├─ [工具执行]
  ├─ [PostToolUse Hook] → 敏感信息扫描
  └─ [PostToolUseFailure Hook] → 错误诊断
```

### 5.4 Hook 的独特价值

Hook 是五种机制中唯一能做到以下事情的能力：

| 能力 | 说明 |
|------|------|
| **阻止操作** | PreToolUse 返回 deny + exit 2 可阻止任何工具调用 |
| **修改输入** | PreToolUse 的 `updatedInput` 可在执行前修改参数 |
| **强制重试** | Stop hook 可 block agent 退出，强制最多 3 轮重试 |
| **跨 Agent 生效** | 子 agent 自动继承父 agent 的全部 Hook |
| **横切所有扩展** | Hook 可拦截 Skills、Commands、Agents、MCP 触发的任何工具调用 |

> **原则**：Skills/Commands/Agents/MCP 负责"扩展能力"，Hook 负责"安全保障"。两者互补，缺一不可。

---


## 附录

### A. 事件 × 输出字段矩阵

| 事件 | permissionDecision | decision | additionalContext | updatedInput |
|------|:--:|:--:|:--:|:--:|
| SessionStart | — | — | ✅ | — |
| UserPromptSubmit | — | — | ✅ | — |
| PreToolUse | ✅ | — | — | ✅ |
| PermissionRequest | — | ✅ | — | — |
| PostToolUse | — | — | ✅ | — |
| PostToolUseFailure | — | — | ✅ | — |
| Stop | — | ✅ | ✅ | — |

### B. Exit Code 速记

```
0  → 成功/放行
2  → 阻塞/拒绝
!0 && !2 → 非阻塞错误（忽略，继续）
```

### C. 最小可用 plugin.json 模板

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "我的 ZCode 插件",
  "author": { "name": "Your Name" },
  "license": "MIT",
  "skills": "skills",
  "commands": "commands",
  "agents": "agents",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node ${ZCODE_PLUGIN_ROOT}/hook.js",
            "timeout": 10000
          }
        ]
      }
    ]
  }
}
```

### D. 最小可用 SKILL.md 模板

```markdown
---
name: my-skill
description: Use when <触发场景描述>
---

# My Skill

## Overview
一句话说明此 Skill 做什么。

## When to Use
- 触发场景 1
- 触发场景 2

## Checklist
- [ ] 步骤 1
- [ ] 步骤 2
```

### E. 最小可用 Command 模板

```markdown
---
description: 命令简述。
argument-hint: "[参数提示]"
skills: 可选-skill-name
---

$ARGUMENTS
```

### F. 最小可用 Agent 模板

```markdown
---
name: my-agent
description: Agent 用途简述。
model: inherit
tools: ["Read", "Write", "Bash"]
---

You are a specialized agent. Your job is to...

## Core Responsibilities
1. ...
2. ...

## Output Format
返回时包含：...
```

### G. 关键路径速查

| 路径 | 用途 |
|------|------|
| `<插件根>/.zcode-plugin/plugin.json` | 插件清单 |
| `<插件根>/skills/<name>/SKILL.md` | Skill 定义 |
| `<插件根>/commands/<name>.md` | Command 定义 |
| `<插件根>/agents/<name>.md` | Agent 定义 |
| `<插件根>/hooks/hooks.json` | Hook 配置 |
| `~/.zcode/cli/config.json` | 插件注册表 |
| `~/.zcode/skills/` | 用户手动安装的 Skills |
| `~/.zcode/cli/plugins/cache/` | 市场插件缓存 |

### H. 已知限制清单

| 限制 | 影响 |
|------|------|
| 插件启动时一次性加载，无热重载 | 修改配置后必须重启 ZCode |
| 孙 agent 硬编码禁用 | 子 agent 的子 agent 不触发任何 hook |
| 没有 SessionEnd 事件 | 无法在会话结束时执行清理 |
| 没有 PreCompact 事件 | 无法在上下文压缩前保存信息 |
| hook 输出最大 32768 字节 | `maxOutputBytes` 硬限制 |
| 默认超时 60000ms | command hook 默认 60 秒超时 |

---

> **数据来源**：本文档基于 `/Applications/ZCode.app/Contents/Resources/glm/zcode.cjs` 源码逆向分析，以及本地插件 `/Users/chenjing/zcode-computer-use-plugin/`、`/Users/chenjing/zcode-agent-teams-plugin/` 的实际配置。关键类/函数：`H4` (InMemoryHookRunner), `Br` (事件枚举), `cV` (工厂函数), `i0t` (hook 源收集), `lvo` (合并), `Kan` (回调构建)。
