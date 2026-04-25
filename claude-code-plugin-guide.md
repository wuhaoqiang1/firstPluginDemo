# Claude Code 插件开发指南 -- 从 0 到 1 创建可安装的插件

> 基于对 Claude Code 已安装插件的逆向分析，总结出的完整规范和实战指南
> 文档版本：v1.0 | 更新日期：2026-04-24

---

## 一、核心概念

### 1.1 两个层级：Marketplace 与 Plugin

Claude Code 的插件系统分两个层级：

| 层级 | 定义 | 对应文件 | 安装命令 |
|------|------|----------|----------|
| **Marketplace** | 插件市场/目录，包含一个或多个插件的注册信息 | `.claude-plugin/marketplace.json` | `/plugin marketplace add <URL>` |
| **Plugin** | 单个功能插件，提供具体的 commands/agents/skills/hooks/MCP | `.claude-plugin/plugin.json` | `/plugin install <marketplace>/<plugin>` |

**两种安装路径：**
1. **先添加 Marketplace，再安装 Plugin** -- 适合包含多个插件的仓库
2. **直接安装 Plugin** -- 适合单一插件的仓库（Marketplace 即 Plugin 自身）

### 1.2 五种插件能力组件

| 组件 | 调用方式 | 目录 | 核心文件 |
|------|----------|------|----------|
| **Commands** | 用户通过 `/command-name` 主动调用 | `commands/*.md` | 每个 .md 文件 |
| **Agents** | Claude 根据上下文自动 spawn 子进程 | `agents/*.md` | 每个 .md 文件 |
| **Skills** | Claude 根据任务上下文自主激活 | `skills/<name>/SKILL.md` | SKILL.md |
| **Hooks** | 在特定事件（工具调用前后、停止时等）自动触发 | `hooks/hooks.json` | hooks.json + 脚本 |
| **MCP Servers** | 为 Claude 提供外部工具调用能力 | `.mcp.json` | .mcp.json |

### 1.3 本机存储结构

```
~/.claude/plugins/
  installed_plugins.json       # 已安装插件注册表（版本、路径、git SHA）
  known_marketplaces.json      # 已添加的 Marketplace 注册表
  marketplaces/                # Marketplace 仓库的 git clone
    claude-plugins-official/   # Anthropic 官方市场
    karpathy-skills/           # Karpathy 技能市场
  cache/                       # 插件的实际运行文件副本
    claude-plugins-official/
      code-review/2cd88e7947b7/  # 版本号 = git commit short SHA 或 semver
      code-simplifier/1.0.0/
      github/2cd88e7947b7/
    karpathy-skills/
      andrej-karpathy-skills/1.0.0/
```

---

## 二、规范标注说明

> **本节非常重要！** 以下文档中，所有规范项用标注区分：

| 标注 | 含义 | 违反后果 |
|------|------|----------|
| **[必须]** | Claude Code 强制要求的字段/结构 | 缺失会导致安装失败或功能不生效 |
| **[推荐]** | 最佳实践，缺失不影响安装但影响体验 | 无 |
| **[可选]** | 按需使用，完全自由 | 无 |
| **[禁止]** | 不允许的做法 | 会导致报错或行为异常 |

---

## 三、Marketplace 规范

### 3.1 目录结构 **[必须]**

Marketplace 仓库根目录**必须**包含：

```
your-marketplace-repo/
  .claude-plugin/
    marketplace.json              **[必须]** 市场清单文件
```

> **[必须]** `.claude-plugin/marketplace.json` 是整个插件系统的入口。Claude Code 执行 `/plugin marketplace add` 或 `/plugin install` 时，第一步就是在仓库根目录查找此文件。找不到就报 "Marketplace not found"。

### 3.2 marketplace.json 规范

#### 最小可用版本 **[必须]**

```json
{
  "name": "your-marketplace-id",
  "plugins": [
    {
      "name": "your-plugin-name",
      "source": "./"
    }
  ]
}
```

| 字段 | 级别 | 说明 |
|------|------|------|
| `name` | **[必须]** | Marketplace 唯一标识符 |
| `plugins` | **[必须]** | 插件列表数组 |
| `plugins[].name` | **[必须]** | 插件名称 |
| `plugins[].source` | **[必须]** | 插件代码来源（见下文 source 规范） |

#### 完整版本（含推荐和可选字段）

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",   **[可选]**
  "name": "your-marketplace-id",                                              **[必须]**
  "id": "your-marketplace-id",                                                **[可选]** 与 name 一致时可不写
  "description": "插件市场的简短描述",                                          **[推荐]**
  "owner": {                                                                  **[推荐]**
    "name": "你的名字或组织名",                                                  **[推荐]**
    "email": "contact@example.com"                                             **[可选]**
  },
  "metadata": {                                                               **[可选]**
    "description": "更详细的描述",                                               **[可选]**
    "version": "1.0.0"                                                         **[可选]**
  },
  "plugins": [                                                                **[必须]**
    {
      "name": "your-plugin-name",                                              **[必须]**
      "source": "./",                                                          **[必须]**
      "description": "插件功能描述",                                             **[推荐]**
      "version": "1.0.0",                                                      **[可选]**
      "author": { "name": "Author" },                                          **[可选]**
      "keywords": ["tag1", "tag2"],                                            **[可选]**
      "category": "development",                                               **[可选]**
      "tags": ["community-managed"],                                           **[可选]**
      "homepage": "https://...",                                               **[可选]**
      "strict": true,                                                          **[可选]** 严格模式
      "lspServers": { ... }                                                    **[可选]** 仅 LSP 插件使用
    }
  ]
}
```

### 3.3 source 字段规范 **[必须]**

source 决定插件代码的位置，有三种格式：

#### 格式一：相对路径（插件代码在本仓库内）**[推荐]**

```json
// 单一插件市场，source 指向仓库根目录自身
"source": "./"

// 多插件市场，source 指向子目录
"source": "./plugins/your-plugin-name"

// 外部插件封装，source 指向 external_plugins 下的子目录
"source": "./external_plugins/your-plugin-name"
```

#### 格式二：Git URL 对象（插件代码在独立仓库）**[可选]**

```json
"source": {
  "source": "url",
  "url": "https://github.com/owner/repo.git"
}
```

#### 格式三：Git URL + 分支指定 **[可选]**

```json
"source": {
  "source": "url",
  "url": "https://github.com/owner/repo.git",
  "ref": "dev"
}
```

> **[重要]** 当 source 是 URL 对象时，Claude Code 会单独 clone 该仓库到 cache 目录。默认 clone 的分支是远程 HEAD 指向的分支（通常是 `main`）。如果你的仓库默认分支是 `master`，需要用 `ref` 字段指定。

### 3.4 多插件市场 vs 单一插件市场

#### 多插件市场目录结构 **[可选]**

```
multi-plugin-marketplace/
  .claude-plugin/
    marketplace.json              **[必须]** 包含多个 plugin 条目
  plugins/                        第一方插件（代码在本仓库）
    plugin-a/
      .claude-plugin/plugin.json  **[必须]**
      commands/...
      skills/...
    plugin-b/
      .claude-plugin/plugin.json  **[必须]**
      agents/...
  external_plugins/               外部插件封装（MCP wrapper 等）
    service-x/
      .claude-plugin/plugin.json  **[必须]**
      .mcp.json
```

marketplace.json 中：

```json
"plugins": [
  { "name": "plugin-a", "source": "./plugins/plugin-a" },
  { "name": "plugin-b", "source": "./plugins/plugin-b" },
  { "name": "service-x", "source": "./external_plugins/service-x" }
]
```

#### 单一插件市场目录结构 **[推荐]**（适合只提供一个插件的场景）

```
single-plugin-marketplace/
  .claude-plugin/
    marketplace.json              **[必须]** plugins 数组只有一条，source 为 "./"
    plugin.json                   **[必须]** 插件自身清单
  skills/...
  commands/...
  agents/...
```

marketplace.json 中：

```json
"plugins": [
  { "name": "my-plugin", "source": "./" }
]
```

> **[必须]** 即使是单一插件市场，marketplace.json 仍然是必需的。plugin.json 也是必需的。

---

## 四、Plugin 规范

### 4.1 目录结构 **[必须]**

```
your-plugin/
  .claude-plugin/
    plugin.json                   **[必须]** 插件清单文件
  commands/                       **[可选]** 斜杠命令
    command-name.md
  agents/                         **[可选]** 子代理定义
    agent-name.md
  skills/                         **[可选]** 技能定义
    skill-name/
      SKILL.md                    **[必须]** 每个 skill 必须有 SKILL.md
  hooks/                          **[可选]** 事件钩子
    hooks.json
    *.sh / *.py                   **[可选]** 钩子脚本
  .mcp.json                       **[可选]** MCP 服务器配置
  CLAUDE.md                       **[可选]** 注入为项目指令
  README.md                       **[可选]** 文档
```

> **[必须]** 所有能力组件目录（commands/、agents/、skills/、hooks/）**必须放在插件根目录**，不能放在 `.claude-plugin/` 内部。只有 `plugin.json` 放在 `.claude-plugin/` 下。

> **[禁止]** 不要把 commands/、agents/、skills/ 等目录放在 `.claude-plugin/` 内部。

### 4.2 plugin.json 规范

#### 最小可用版本 **[必须]**

```json
{
  "name": "your-plugin-name"
}
```

> **[必须]** `name` 是唯一必需字段。用 kebab-case 命名（如 `code-review`、`my-awesome-plugin`）。

#### 完整版本

```json
{
  "name": "your-plugin-name",                        **[必须]**
  "description": "插件功能描述",                       **[推荐]**
  "version": "1.0.0",                                **[可选]** semver 格式
  "author": {                                        **[可选]**
    "name": "Author Name",
    "email": "author@example.com",
    "url": "https://example.com"
  },
  "homepage": "https://docs.example.com",            **[可选]**
  "repository": "https://github.com/...",            **[可选]**
  "license": "MIT",                                  **[可选]**
  "keywords": ["tag1", "tag2"],                      **[可选]**

  "commands": "./commands",                          **[可选]** 默认自动发现
  "agents": ["./agents", "./specialized"],           **[可选]** 默认自动发现
  "skills": ["./skills/xxx"],                        **[可选]** 默认自动发现
  "hooks": "./hooks/hooks.json",                     **[可选]** 默认自动发现
  "mcpServers": "./.mcp.json"                        **[可选]** 默认自动发现
}
```

> **[重要]** commands/agents/skills/hooks/mcpServers 这些字段是**补充性的，不是替代性的**。Claude Code 默认会自动发现对应目录下的组件。只有当你需要自定义路径或引入额外的目录时，才需要显式声明。

---

## 五、能力组件规范

### 5.1 Commands（斜杠命令）

#### 目录规范 **[必须]**

```
commands/
  command-name.md                 **[必须]** 文件名即命令名（去掉 .md）
```

> **[必须]** 文件名决定命令名。例如 `code-review.md` 对应 `/code-review` 命令。

> **[禁止]** 文件名不能包含空格或特殊字符，用 kebab-case。

#### 文件格式规范

每个命令文件是 Markdown + YAML frontmatter：

```markdown
---
description: 命令的简短描述，显示在 /help 中          **[必须]**
argument-hint: <required-arg> [optional-arg]         **[可选]** 参数提示
allowed-tools: [Read, Glob, Grep, Bash(gh:*)]       **[可选]** 预授权工具列表
model: sonnet                                       **[可选]** 覆盖模型：haiku/sonnet/opus
disable-model-invocation: false                     **[可选]** 禁止程序化调用
---

# 命令标题

命令的详细指令内容...

## 可用变量

- `$ARGUMENTS` -- 用户传入的所有参数（单字符串）
- `$1`, `$2`, `$3` -- 位置参数
- `@$1` -- 引用文件内容
- `` !`command` `` -- 执行 bash 命令获取上下文
- `${CLAUDE_PLUGIN_ROOT}` -- 插件根目录路径
```

#### frontmatter 字段规范

| 字段 | 级别 | 类型 | 说明 |
|------|------|------|------|
| `description` | **[必须]** | string | /help 中显示的描述 |
| `argument-hint` | **[可选]** | string | 参数格式提示 |
| `allowed-tools` | **[可选]** | string/array | 预授权工具，减少权限弹窗 |
| `model` | **[可选]** | string | `haiku` / `sonnet` / `opus` |
| `disable-model-invocation` | **[可选]** | boolean | 防止其他代码程序化调用此命令 |

#### allowed-tools 格式示例

```yaml
allowed-tools: [Read, Glob, Grep, Bash(gh issue view:*), Bash(gh pr list:*)]
```

> **[推荐]** 为命令配置 allowed-tools，这样用户调用命令时不会频繁被权限弹窗打断。

---

### 5.2 Agents（子代理）

#### 目录规范 **[必须]**

```
agents/
  agent-name.md                   **[必须]** 文件名即代理名（去掉 .md）
```

#### 文件格式规范

```markdown
---
name: agent-identifier            **[必须]** 3-50字符，小写+连字符，首尾必须字母数字
description: Use this agent when [触发条件]. Examples:  **[必须]** 10-5000字符

<example>
Context: [场景描述]
user: "[用户请求]"
assistant: "[Claude 如何响应并使用此 agent]"
<commentary>
[为什么应该触发此 agent]
</commentary>
</example>

model: inherit                    **[必须]** inherit / sonnet / opus / haiku
color: blue                       **[必须]** blue / cyan / green / yellow / magenta / red
tools: ["Read", "Grep", "Glob"]   **[可选]** 限制可用工具，默认全部
---

你是一个专门负责 [领域] 的代理...

**核心职责：**
1. [职责1]
2. [职责2]

**分析流程：**
[步骤1] → [步骤2] → [步骤3]

**输出格式：**
[返回什么内容]
```

#### frontmatter 字段规范

| 字段 | 级别 | 说明 |
|------|------|------|
| `name` | **[必须]** | 代理标识符，3-50字符，kebab-case |
| `description` | **[必须]** | 触发条件描述，**必须包含 `<example>` 块** |
| `model` | **[必须]** | `inherit` / `sonnet` / `opus` / `haiku` |
| `color` | **[必须]** | `blue` / `cyan` / `green` / `yellow` / `magenta` / `red` |
| `tools` | **[可选]** | 工具白名单数组，默认全部可用 |

> **[必须]** description 中**必须包含 `<example>` 块**，包含 `Context`、`user`、`assistant`、`<commentary>` 四个标签。这是 Claude 学习何时触发此代理的关键训练数据。

---

### 5.3 Skills（技能）

#### 目录规范 **[必须]**

```
skills/
  skill-name/
    SKILL.md                       **[必须]** 技能定义文件
    references/                    **[可选]** 参考材料
    examples/                      **[可选]** 示例文件
    scripts/                       **[可选]** 辅助脚本
    templates/                     **[可选]** 模板文件
```

> **[必须]** 每个 skill **必须**有一个子目录，且子目录中**必须**有 `SKILL.md`。不能直接把 SKILL.md 放在 skills/ 根目录下。

#### SKILL.md 文件格式规范

```markdown
---
name: skill-name                  **[必须]** 技能标识符
description: 触发条件描述          **[必须]** 描述 Claude 在何时应该自主激活此技能
version: 1.0.0                    **[可选]** 语义版本号
license: MIT                      **[可选]** 许可证信息
---

# 技能标题

技能的详细指令内容...

## 何时适用
- 条件1
- 条件2

## 指令
[详细步骤和指导]
```

#### frontmatter 字段规范

| 字段 | 级别 | 说明 |
|------|------|------|
| `name` | **[必须]** | 技能标识符 |
| `description` | **[必须]** | **触发条件描述** -- 这是最关键的字段，告诉 Claude 在什么情况下自主使用此技能 |
| `version` | **[可选]** | 语义版本号 |
| `license` | **[可选]** | 许可证 |

> **[必须]** `description` 字段要写得非常明确和具体，包含触发短语、关键词、适用场景。这是 Claude 自主决定是否激活此技能的唯一依据。

> **[推荐]** description 模式：`This skill should be used when the user asks to "具体短语", "另一个短语", mentions "关键词", or discusses 主题领域.`

---

### 5.4 Hooks（事件钩子）

#### 目录规范 **[必须]**

```
hooks/
  hooks.json                      **[必须]** 钩子配置
  security_reminder_hook.py       **[可选]** 钩子脚本文件
```

#### hooks.json 格式规范

**插件格式**（带 `hooks` 包装层）**[推荐]**：

```json
{
  "description": "钩子的简短描述",                    **[可选]**
  "hooks": {                                         **[必须]** 包装层
    "PreToolUse": [...],                             **[可选]**
    "PostToolUse": [...],                            **[可选]**
    "Stop": [...],                                   **[可选]**
    "SessionStart": [...]                            **[可选]**
  }
}
```

#### 可用事件类型

| 事件 | 触发时机 | 用途 |
|------|----------|------|
| `PreToolUse` | 工具调用前 | 校验、修改输入、审批/拒绝 |
| `PostToolUse` | 工具调用后 | 反馈、日志记录 |
| `Stop` | 主代理考虑停止时 | 完整性检查 |
| `SubagentStop` | 子代理考虑停止时 | 任务验证 |
| `SessionStart` | 会话开始时 | 上下文加载、环境初始化 |
| `SessionEnd` | 会话结束时 | 清理、日志 |
| `UserPromptSubmit` | 用户提交 prompt 时 | 上下文注入、校验 |
| `PreCompact` | 上下文压缩前 | 保留关键信息 |
| `Notification` | Claude 发送通知时 | 日志、反应 |

#### 钩子条目规范

每个事件类型下的条目结构：

```json
{
  "matcher": "Edit|Write|MultiEdit",              **[必须]** 正则匹配工具名，"*" 匹配所有
  "hooks": [
    {
      "type": "command",                           **[必须]** "command" 或 "prompt"
      "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/script.py",  **[必须]** 当 type=command 时
      "timeout": 30                                **[可选]** 默认 command=60s，prompt=30s
    },
    {
      "type": "prompt",                            **[必须]**
      "prompt": "评估此工具调用是否合适: $TOOL_INPUT",  **[必须]** 当 type=prompt 时
      "timeout": 30                                **[可选]**
    }
  ]
}
```

> **[必须]** `matcher` 用正则表达式匹配工具名。`"Edit|Write"` 匹配 Edit 和 Write 工具。

#### command 类型钩子的环境变量

| 变量 | 说明 |
|------|------|
| `$CLAUDE_PLUGIN_ROOT` | 插件目录路径 |
| `$CLAUDE_PROJECT_DIR` | 项目根路径 |

#### 钩子输入（stdin JSON）

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.txt",
  "cwd": "/current/working/dir",
  "hook_event_name": "PreToolUse",
  "tool_name": "Write",
  "tool_input": { ... },
  "tool_result": "...",
  "user_prompt": "..."
}
```

#### 钩子输出（stdout JSON）

```json
{
  "continue": true,
  "suppressOutput": false,
  "systemMessage": "给 Claude 的消息",
  "hookSpecificOutput": {
    "permissionDecision": "allow|deny|ask",
    "updatedInput": { "field": "modified_value" }
  },
  "decision": "approve|block",
  "reason": "解释"
}
```

#### 退出码规范 **[必须]**

| 退出码 | 含义 |
|--------|------|
| `0` | 成功（stdout 内容会显示在 transcript） |
| `2` | 阻塞错误（stderr 反馈给 Claude） |
| 其他 | 非阻塞错误 |

---

### 5.5 MCP Servers（外部工具服务）

#### 文件位置规范 **[可选]**

两种方式：
1. **独立文件** `.mcp.json` -- 放在插件根目录 **[推荐]**
2. **内嵌在 plugin.json** -- 作为 `mcpServers` 字段 **[可选]**

#### .mcp.json 格式规范

支持四种服务器类型：

**stdio（本地进程）** -- **[推荐]** 本地服务：

```json
{
  "server-name": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-filesystem"],
    "env": {
      "LOG_LEVEL": "info"
    }
  }
}
```

**SSE（Server-Sent Events）** -- OAuth 认证场景：

```json
{
  "server-name": {
    "type": "sse",
    "url": "https://mcp.service.com/sse"
  }
}
```

**HTTP（REST API）** -- Token 认证场景：

```json
{
  "server-name": {
    "type": "http",
    "url": "https://api.service.com/mcp/",
    "headers": {
      "Authorization": "Bearer ${API_TOKEN}"
    }
  }
}
```

**WebSocket** -- 实时通信场景：

```json
{
  "server-name": {
    "type": "ws",
    "url": "wss://mcp.service.com/ws",
    "headers": {
      "Authorization": "Bearer ${TOKEN}"
    }
  }
}
```

> **[可选]** 环境变量 `${}` 在 MCP 配置中会自动展开。如 `${GITHUB_PERSONAL_ACCESS_TOKEN}` 会从用户环境变量读取。

> **[必须]** MCP 工具的命名规则为 `mcp__plugin_<plugin-name>_<server-name>__<tool-name>`。

---

## 六、实战：从 0 到 1 创建插件

### 场景 A：创建单一插件（直接可安装）

适合只有一个插件的场景，如技能包、工作流工具等。

#### Step 1：创建仓库目录结构

```bash
mkdir my-awesome-plugin
cd my-awesome-plugin
git init

# 创建必需的 manifest 文件
mkdir -p .claude-plugin
mkdir -p skills/my-skill
mkdir -p commands
mkdir -p hooks
```

#### Step 2：创建 marketplace.json **[必须]**

```bash
cat > .claude-plugin/marketplace.json << 'EOF'
{
  "name": "my-awesome-plugin-marketplace",
  "id": "my-awesome-plugin-marketplace",
  "owner": {
    "name": "Your Name"
  },
  "plugins": [
    {
      "name": "my-awesome-plugin",
      "source": "./",
      "description": "插件的简短描述",
      "version": "1.0.0",
      "author": {
        "name": "Your Name"
      },
      "keywords": ["keyword1", "keyword2"],
      "category": "development"
    }
  ]
}
EOF
```

> **[必须]** 即使是单一插件，marketplace.json 也是必需的。Claude Code 首先查找此文件。
> **[必须]** `source` 为 `"./"` 表示插件代码就在本仓库根目录。

#### Step 3：创建 plugin.json **[必须]**

```bash
cat > .claude-plugin/plugin.json << 'EOF'
{
  "name": "my-awesome-plugin",
  "description": "插件的详细功能描述",
  "version": "1.0.0",
  "author": {
    "name": "Your Name",
    "email": "you@example.com"
  },
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"]
}
EOF
```

#### Step 4：创建技能文件（可选）

```bash
cat > skills/my-skill/SKILL.md << 'EOF'
---
name: my-skill
description: This skill should be used when the user asks to "do something specific", mentions "my-keyword", or discusses my-domain-area.
version: 1.0.0
license: MIT
---

# My Skill

## 何时适用
- 用户需要执行某类任务时
- 涉及特定领域时

## 指令
1. 步骤一
2. 步骤二
3. 步骤三
EOF
```

#### Step 5：创建命令文件（可选）

```bash
cat > commands/my-command.md << 'EOF'
---
description: 执行某个特定操作的斜杠命令
argument-hint: <target> [options]
allowed-tools: [Read, Grep, Glob, Bash(git:*)]
model: sonnet
---

# My Command

当用户调用 `/my-command target` 时：

1. 解析参数 $ARGUMENTS
2. 执行操作
3. 返回结果

可用变量：
- $ARGUMENTS -- 所有参数
- $1 -- 第一个参数
- ${CLAUDE_PLUGIN_ROOT} -- 插件根目录
EOF
```

#### Step 6：创建 Hooks（可选）

```bash
cat > hooks/hooks.json << 'EOF'
{
  "description": "在编辑文件时提供提醒",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/reminder.py"
          }
        ]
      }
    ]
  }
}
EOF

# 创建钩子脚本
cat > hooks/reminder.py << 'EOF'
import sys
import json

input_data = json.load(sys.stdin)
tool_name = input_data.get("tool_name", "")
tool_input = input_data.get("tool_input", {})

# 检查逻辑
file_path = tool_input.get("file_path", "")
if file_path and "敏感关键词" in file_path:
    result = {
        "continue": False,
        "systemMessage": "此文件可能包含敏感内容，请谨慎操作",
        "hookSpecificOutput": {
            "permissionDecision": "ask"
        }
    }
    print(json.dumps(result))
    sys.exit(0)

# 正常通过
print(json.dumps({"continue": True}))
sys.exit(0)
EOF
```

#### Step 7：创建 MCP 配置（可选）

```bash
cat > .mcp.json << 'EOF'
{
  "my-service": {
    "type": "http",
    "url": "https://api.my-service.com/mcp/",
    "headers": {
      "Authorization": "Bearer ${MY_SERVICE_TOKEN}"
    }
  }
}
EOF
```

#### Step 8：提交并推送

```bash
git add .
git commit -m "Initial plugin setup"
# 推送到你的 Git 平台
git remote add origin https://your-git-platform.com/your-org/my-awesome-plugin.git
git push -u origin master
```

> **[重要]** 确保推送的分支是 Git 平台上的**默认分支**。Claude Code clone 时会 checkout 远程 HEAD 指向的分支。如果你的 Git 平台默认分支是 `master`，确认远程 HEAD 指向 `master`。

#### Step 9：安装

**方式一：直接安装（Marketplace 即 Plugin）**

```
/plugin install https://your-git-platform.com/your-org/my-awesome-plugin.git
```

**方式二：先添加 Marketplace，再安装 Plugin**

```
/plugin marketplace add https://your-git-platform.com/your-org/my-awesome-plugin.git
/plugin install my-awesome-plugin-marketplace/my-awesome-plugin
```

---

### 场景 B：创建多插件 Marketplace

适合包含多个插件的团队或组织。

#### Step 1：创建仓库目录结构

```bash
mkdir my-team-plugins
cd my-team-plugins
git init

mkdir -p .claude-plugin
mkdir -p plugins/plugin-a/.claude-plugin
mkdir -p plugins/plugin-a/skills/skill-a
mkdir -p plugins/plugin-b/.claude-plugin
mkdir -p plugins/plugin-b/commands
```

#### Step 2：创建 marketplace.json **[必须]**

```json
{
  "name": "my-team-plugins",
  "description": "团队插件集合",
  "owner": {
    "name": "Team Name"
  },
  "plugins": [
    {
      "name": "plugin-a",
      "source": "./plugins/plugin-a",
      "description": "插件A的功能描述",
      "version": "1.0.0",
      "category": "development"
    },
    {
      "name": "plugin-b",
      "source": "./plugins/plugin-b",
      "description": "插件B的功能描述",
      "version": "1.0.0",
      "category": "productivity"
    }
  ]
}
```

#### Step 3：为每个插件创建 plugin.json **[必须]**

```bash
# plugin-a
cat > plugins/plugin-a/.claude-plugin/plugin.json << 'EOF'
{
  "name": "plugin-a",
  "description": "插件A的功能描述",
  "version": "1.0.0"
}
EOF

# plugin-b
cat > plugins/plugin-b/.claude-plugin/plugin.json << 'EOF'
{
  "name": "plugin-b",
  "description": "插件B的功能描述",
  "version": "1.0.0"
}
EOF
```

#### Step 4：为每个插件创建能力组件

（同场景 A 中 Step 4-7，在每个插件目录下操作）

#### Step 5：提交推送

```bash
git add .
git commit -m "Initial multi-plugin marketplace setup"
git remote add origin https://your-git-platform.com/your-org/my-team-plugins.git
git push -u origin master
```

#### Step 6：安装

```
# 先添加市场
/plugin marketplace add https://your-git-platform.com/your-org/my-team-plugins.git

# 再从市场中安装特定插件
/plugin install my-team-plugins/plugin-a
/plugin install my-team-plugins/plugin-b
```

---

### 场景 C：MCP 封装插件（接入第三方服务）

适合只需要为 Claude Code 接入某个外部 MCP 服务的场景，无需 commands/agents/skills。

#### 最小 MCP 封装插件结构

```
my-service-wrapper/
  .claude-plugin/
    marketplace.json              **[必须]**
  .claude-plugin/
    plugin.json                   **[必须]**
  .mcp.json                       **[必须]** MCP 配置
```

> **[重要]** 注意：单一插件市场中，marketplace.json 和 plugin.json **都在 `.claude-plugin/` 目录下**，这是特殊情况。在多插件市场中，各子插件的 plugin.json 在各自子目录的 `.claude-plugin/` 下。

marketplace.json：

```json
{
  "name": "my-service-marketplace",
  "plugins": [
    {
      "name": "my-service",
      "source": "./"
    }
  ]
}
```

plugin.json：

```json
{
  "name": "my-service",
  "description": "接入 My Service 的 MCP 服务"
}
```

.mcp.json：

```json
{
  "my-service": {
    "type": "http",
    "url": "https://api.my-service.com/mcp/",
    "headers": {
      "Authorization": "Bearer ${MY_SERVICE_API_KEY}"
    }
  }
}
```

---

## 七、安装命令详解

### 7.1 `/plugin marketplace add <URL>`

- 将 URL 对应的 Git 仓库 clone 到 `~/.claude/plugins/marketplaces/<marketplace-id>/`
- 读取仓库根目录的 `.claude-plugin/marketplace.json`
- 注册到 `known_marketplaces.json`

### 7.2 `/plugin install <marketplace>/<plugin-name>`

- 从已添加的 marketplace 中安装指定插件
- 将插件文件复制到 `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/`
- 注册到 `installed_plugins.json`
- 在 `settings.json` 的 `enabledPlugins` 中标记为 `true`

### 7.3 `/plugin install <URL>`

- 直接通过 URL 安装插件
- Claude Code 先 clone 仓库，查找 `.claude-plugin/marketplace.json`
- 从 marketplace.json 中找到插件列表，安装第一个（或让用户选择）

### 7.4 常见安装问题及解决

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| "Marketplace not found" | 仓库根目录缺少 `.claude-plugin/marketplace.json` | 在仓库中添加此文件 |
| "HTTP 401 error" | 仓库是私有的，HTTP 请求没有认证凭据 | 方案1：手动 git clone 到 marketplace 目录；方案2：在 Git 平台设为公开 |
| 安装了但功能不生效 | 组件目录位置错误（放在 `.claude-plugin/` 内） | 把 commands/agents/skills/hooks 移到插件根目录 |
| 安装的分支不对 | Git 平台默认分支与包含 `.claude-plugin/` 的分支不一致 | 确保默认分支正确，或在 marketplace.json 的 source 中用 `"ref": "分支名"` 指定 |
| 插件不自动激活 | skill 的 description 触发条件写得不够明确 | 优化 description，包含具体触发短语和关键词 |

---

## 八、规范速查表

### 文件级规范

| 文件 | 必须存在？ | 位置 | 说明 |
|------|-----------|------|------|
| `.claude-plugin/marketplace.json` | **[必须]** | 仓库根目录 | Marketplace 入口 |
| `.claude-plugin/plugin.json` | **[必须]** | 插件根目录 | Plugin 入口 |
| `.mcp.json` | **[可选]** | 插件根目录 | MCP 服务器配置 |
| `CLAUDE.md` | **[可选]** | 插件根目录 | 注入为项目指令 |
| `hooks/hooks.json` | **[可选]** | 插件根目录 | 钩子配置 |
| `commands/*.md` | **[可选]** | 插件根目录 | 斜杠命令 |
| `agents/*.md` | **[可选]** | 插件根目录 | 子代理定义 |
| `skills/*/SKILL.md` | **[可选]** | 插件根目录 | 技能定义 |

### 字段级规范

#### marketplace.json 必须字段

| 字段 | 级别 |
|------|------|
| `name` | **[必须]** |
| `plugins` | **[必须]** |
| `plugins[].name` | **[必须]** |
| `plugins[].source` | **[必须]** |

#### plugin.json 必须字段

| 字段 | 级别 |
|------|------|
| `name` | **[必须]** |

#### command frontmatter 必须字段

| 字段 | 级别 |
|------|------|
| `description` | **[必须]** |

#### agent frontmatter 必须字段

| 字段 | 级别 |
|------|------|
| `name` | **[必须]** |
| `description`（含 `<example>` 块） | **[必须]** |
| `model` | **[必须]** |
| `color` | **[必须]** |

#### skill frontmatter 必须字段

| 字段 | 级别 |
|------|------|
| `name` | **[必须]** |
| `description` | **[必须]** |

#### hooks.json 必须字段

| 字段 | 级别 |
|------|------|
| `hooks.<event>.matcher` | **[必须]** |
| `hooks.<event>.hooks[].type` | **[必须]** |
| `hooks.<event>.hooks[].command`（type=command时） | **[必须]** |
| `hooks.<event>.hooks[].prompt`（type=prompt时） | **[必须]** |

---

## 九、命名规范

| 对象 | 规范 | 示例 |
|------|------|------|
| Marketplace name | kebab-case，全局唯一 | `claude-plugins-official` |
| Plugin name | kebab-case，市场内唯一 | `code-review` |
| Command 文件名 | kebab-case | `code-review.md` -> `/code-review` |
| Agent name | kebab-case，3-50字符 | `code-simplifier` |
| Skill 目录名 | kebab-case | `skills/karpathy-guidelines/` |
| Skill name | kebab-case | `karpathy-guidelines` |
| MCP server name | kebab-case | `github` |

---

## 十、默认分支注意事项

| Git 平台 | 默认分支 | 注意事项 |
|----------|----------|----------|
| GitHub | `main` | Claude Code clone 时 checkout 远程 HEAD，通常是 main |
| GitLab | `main` | 同上 |
| 自建 Git（如 devops.geely.com） | 通常 `master` | **[重要]** 如果你的仓库默认分支是 `master`，确认 Git 平台配置的远程 HEAD 指向 `master` |

> **[重要]** Claude Code 的 `/plugin install` 和 `/plugin marketplace add` 通过 git clone 访问仓库，会 checkout 远程 HEAD 指向的分支。如果你的 `.claude-plugin/` 目录不在该分支上，安装就会失败。

> 如果无法修改默认分支，可以在 marketplace.json 中用 URL source + ref 指定分支：
> ```json
> "source": {
>   "source": "url",
>   "url": "https://your-git-platform.com/org/repo.git",
>   "ref": "master"
> }
> ```

---

## 附录：已安装插件的真实案例

### 案例一：karpathy-skills（单一技能插件）

```
karpathy-skills/                    # Git 仓库根目录 = Marketplace 根目录 = Plugin 根目录
  .claude-plugin/
    marketplace.json                # name: "karpathy-skills", source: "./"
    plugin.json                     # name: "andrej-karpathy-skills"
  skills/
    karpathy-guidelines/
      SKILL.md                      # 单一技能
  CLAUDE.md                         # 项目指令
  README.md                         # 文档
```

### 案例二：code-review（命令型插件，来自官方市场）

```
claude-plugins-official/plugins/code-review/
  .claude-plugin/
    plugin.json                     # name: "code-review"
  commands/
    code-review.md                  # /code-review 命令
```

### 案例三：code-simplifier（代理型插件，来自官方市场）

```
claude-plugins-official/plugins/code-simplifier/
  .claude-plugin/
    plugin.json                     # name: "code-simplifier"
  agents/
    code-simplifier.md              # 单一代理
```

### 案例四：github（MCP 封装插件，来自官方市场 external_plugins）

```
claude-plugins-official/external_plugins/github/
  .claude-plugin/
    plugin.json                     # name: "github"
  .mcp.json                         # HTTP 类型 MCP 服务
```

### 案例五：security-guidance（Hooks 型插件，来自官方市场）

```
claude-plugins-official/plugins/security-guidance/
  .claude-plugin/
    plugin.json                     # name: "security-guidance"
  hooks/
    hooks.json                      # PreToolUse hook，matcher: "Edit|Write|MultiEdit"
    security_reminder_hook.py       # Python 钩子脚本
```

---

**文档版本**: v1.0
**创建日期**: 2026-04-24