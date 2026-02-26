# 游戏开发设计模式插件

存放游戏活动开发相关的设计模式和代码模板，供团队统一使用。

## 包含内容

### Skills

| Skill | 描述 | 适用场景 |
|-------|------|---------|
| data-presentation-separation | 数据与表现分离 | TV动画展示、战斗表现等 |

## 安装方式

```bash
# 方式1: 克隆到本地插件目录
git clone <仓库地址> C:/Users/<用户名>/.claude/plugins/cache/zz-coding-plugins

# 方式2: 使用符号链接
mklink /D C:\Users\<用户名>\.claude\plugins\cache\zz-coding-plugins <仓库本地路径>
```

## 使用方式

当讨论相关话题时，skill 会自动激活：
- "数据与表现分离"
- "TV动画展示"
- "活动表现数据"
- "活动设计模式"

## 贡献指南

详见 [CONTRIBUTING.md](./CONTRIBUTING.md)

## 版本历史

- v1.0.0 - 初始版本，包含数据与表现分离模式

## 目录结构

```
zz-coding-plugins/
├── .claude-plugin/
│   └── plugin.json          # 插件清单
├── skills/
│   └── data-presentation-separation/
│       └── SKILL.md         # 数据与表现分离 skill
├── examples/                # 示例项目（待添加）
├── README.md
└── CONTRIBUTING.md
```
