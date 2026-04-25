# 插件安装问题排查指南

## 问题现象
```
Marketplace "https://github.com/wuhaoqiang1/firstPluginDemo.git" not found
```

## 可能原因和解决方案

### 1. Claude Code缓存问题
**解决方案：**
```bash
# 清除Claude Code的插件缓存
rm -rf ~/.claude/plugins/cache/firstPluginDemo/

# 然后重新安装
/plugin install https://github.com/wuhaoqiang1/firstPluginDemo.git
```

### 2. 使用添加Marketplace的方式
```bash
# 先添加Marketplace
/plugin marketplace add https://github.com/wuhaoqiang1/firstPluginDemo.git

# 再安装插件
/plugin install demo-plugin-marketplace/demo-plugin
```

### 3. 检查仓库结构
确保你的仓库结构正确：
```
firstPluginDemo/
├── .claude-plugin/
│   ├── marketplace.json    ✓ 已上传
│   └── plugin.json         ✓ 已上传
├── commands/
│   └── demo-command.md     ✓ 已上传
├── hooks/
│   └── hooks.json          ✓ 已上传
├── skills/
│   └── demo-skill/
│       └── SKILL.md       ✓ 已上传
├── README.md              ✓ 已上传
└── ...
```

### 4. 确认文件内容
确保marketplace.json内容正确：
```json
{
  "name": "demo-plugin-marketplace",
  "id": "demo-plugin-marketplace",
  "owner": {
    "name": "Your Name"
  },
  "plugins": [
    {
      "name": "demo-plugin",
      "source": "./",
      "description": "一个简单的Claude Code插件示例",
      "version": "1.0.0",
      "author": {
        "name": "Your Name"
      },
      "keywords": ["demo", "example", "learning"],
      "category": "development"
    }
  ]
}
```

### 5. 网络问题
如果GitHub访问慢，可以尝试：
- 使用VPN
- 稍后重试
- 使用GitHub镜像（如果可用）

### 6. 检查已安装的插件
```bash
/plugins list
```

## 如果还是不行

### 手动安装方式（临时方案）
1. 手动clone仓库：
```bash
git clone https://github.com/wuhaoqiang1/firstPluginDemo.git
cd firstPluginDemo
```

2. 将插件复制到Claude Code插件目录：
```bash
cp -r . ~/.claude/plugins/cache/firstPluginDemo/demo-plugin/0.0.1/
```

3. 在settings.json中启用插件：
```json
{
  "enabledPlugins": {
    "demo-plugin": true
  }
}
```

## 联系支持
如果以上方法都不行，请提供：
- Claude Code版本
- 操作系统
- 错误完整信息
- 你的GitHub仓库URL