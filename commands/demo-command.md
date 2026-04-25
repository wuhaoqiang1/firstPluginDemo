---
description: 演示插件功能，显示插件信息
argument-hint: [action]
allowed-tools: [Read, Glob]
model: sonnet
---

# Demo Command

当用户调用 `/demo-command` 时：

1. 读取plugin.json获取插件信息
2. 显示插件的基本信息
3. 列出可用的skills和commands

可用变量：
- $ARGUMENTS -- 所有参数
- $1 -- 第一个参数
- ${CLAUDE_PLUGIN_ROOT} -- 插件根目录

对于不同的action参数：
- 不带参数：显示插件概览
- "skills"：显示所有技能列表
- "commands"：显示所有命令列表