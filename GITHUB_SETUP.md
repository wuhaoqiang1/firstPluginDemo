# GitHub仓库创建和插件安装指南

## 创建GitHub仓库

### 方法一：通过GitHub网站创建

1. 访问 [GitHub](https://github.com)
2. 点击右上角的 "+" 号，选择 "New repository"
3. 填写仓库信息：
   - **Repository name**: `demo-plugin`（或其他你喜欢的名字）
   - **Description**: `A simple Claude Code plugin example for learning`
   - **Public/Private**: 选择 Public（便于访问）
   - **Initialize this repository with**: 不要勾选任何选项（因为我们已经有代码了）
4. 点击 "Create repository"

### 方法二：通过GitHub CLI（如果已安装）

```bash
# 创建并推送仓库
gh repo create demo-plugin --public --source=. --remote=origin --push
```

## 推送代码到GitHub

创建完GitHub仓库后，在本地执行以下命令：

```bash
# 添加远程仓库（替换为你的GitHub仓库URL）
git remote add origin https://github.com/YOUR_USERNAME/demo-plugin.git

# 推送代码到main分支
git push -u origin main
```

> 注意：如果你的GitHub仓库默认分支是 `master`，请使用：
> ```bash
> git push -u origin master
> ```

## 安装插件

### 方法一：直接安装（推荐）

```bash
/plugin install https://github.com/YOUR_USERNAME/demo-plugin.git
```

### 方法二：先添加市场，再安装插件

```bash
# 添加市场
/plugin marketplace add https://github.com/YOUR_USERNAME/demo-plugin.git

# 安装插件
/plugin install demo-plugin-marketplace/demo-plugin
```

## 测试插件

安装成功后，你可以测试插件功能：

1. **使用命令**：
   ```
   /demo-command
   ```

2. **触发技能**：
   - 询问："请演示一下插件的功能"
   - 或者说："给我一个example"
   - 或者说："关于Claude Code插件你想说什么"

3. **测试Hook**：
   - 尝试编辑任何文件，会在操作前看到提醒

## 常见问题

### 1. "Marketplace not found" 错误
确保仓库根目录有 `.claude-plugin/marketplace.json` 文件。

### 2. HTTP 401错误
如果仓库是私有的，需要先设为公开，或者使用个人访问令牌。

### 3. 插件不生效
检查：
- 插件是否正确安装（使用 `/plugins list` 查看）
- commands/agents/skills目录是否在插件根目录（不在`.claude-plugin/`内）
- 文件格式是否正确

## 下一步

1. 修改插件配置文件，个性化你的插件
2. 添加更多commands、agents或skills
3. 根据需要配置hooks或MCP服务器
4. 测试并完善你的插件功能