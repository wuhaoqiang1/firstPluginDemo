# Demo Plugin for Claude Code

这是一个简单的Claude Code插件示例，用于学习插件开发。

## 功能特性

- **Skills**: 提供演示技能，在用户询问相关问题时自动触发
- **Commands**: 提供 `/demo-command` 命令，显示插件信息
- **Hooks**: 在编辑文件前提供提醒

## 安装方法

### 方法一：直接安装

```bash
/plugin install https://github.com/your-username/demo-plugin.git
```

### 方法二：先添加市场，再安装插件

```bash
/plugin marketplace add https://github.com/your-username/demo-plugin.git
/plugin install demo-plugin-marketplace/demo-plugin
```

## 使用方法

1. 使用 `/demo-command` 查看插件信息
2. 询问关于"demo"或"example"的问题，会自动触发demo-skill
3. 正常使用Claude Code时，编辑文件前会有提醒

## 开发说明

这个插件展示了Claude Code插件的基本结构：

- `.claude-plugin/marketplace.json` - 市场配置文件
- `.claude-plugin/plugin.json` - 插件配置文件
- `skills/` - 技能目录
- `commands/` - 命令目录
- `hooks/` - 钩子配置目录

## 贡献

欢迎提交Issue和Pull Request来改进这个示例插件。