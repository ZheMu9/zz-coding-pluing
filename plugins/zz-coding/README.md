# 游戏开发设计模式插件

存放游戏活动开发相关的设计模式和代码模板，供团队统一使用。

## 包含内容

### Skills

| Skill | 描述 | 适用场景 |
| ----- | ---- | -------- |
| data-presentation-separation | 数据与表现分离 | TV动画展示、战斗表现等 |
| activity-development | 既有 Activity 框架下的标准活动开发 | 新增活动、补数据管理器、入口按钮等 |
| activity-architecture-refactoring | 旧活动系统迁移到新 Activity 架构 | 活动重构、框架迁移 |
| unity-activity-development | Unity 活动功能开发方法论草稿 | 活动面板、活动数据、表现联动 |
| git-workflow | Git 分支、提交、PR 规范 | commit、branch、PR、团队协作 |
| pdf | PDF 处理能力（来自 anthropics/skills） | 提取、合并、拆分、填表、OCR |

## 安装方式

```bash
# 方式1: 克隆到本地插件目录
git clone <仓库地址> C:/Users/<用户名>/.claude/plugins/cache/zz-coding-plugins

# 方式2: 使用符号链接
mklink /D C:\Users\<用户名>\.claude\plugins\cache\zz-coding-plugins <仓库本地路径>
```

## 使用方式

当讨论相关话题时，skill 会自动激活，例如：

- "数据与表现分离"
- "TV动画展示"
- "活动表现数据"
- "活动设计模式"
- "活动开发"
- "EventDataManager"
- "branch 怎么命名"
- "帮我写 commit"
- "处理 pdf"

## 贡献指南

详见 [CONTRIBUTING.md](./CONTRIBUTING.md)

## 版本历史

- v1.0.0 - 初始版本，包含数据与表现分离模式

## 目录结构

```text
zz-coding-plugins/
├── .claude-plugin/
│   └── plugin.json          # 插件清单
├── skills/
│   ├── activity-architecture-refactoring/
│   │   └── SKILL.md
│   ├── activity-development/
│   │   ├── SKILL.md
│   │   └── references/
│   ├── data-presentation-separation/
│   │   └── SKILL.md
│   ├── git-workflow/
│   │   └── SKILL.md
│   ├── pdf/
│   │   ├── SKILL.md
│   │   ├── reference.md
│   │   ├── forms.md
│   │   └── scripts/
│   └── unity-activity-development/
│       ├── SKILL.md
│       └── references/
├── examples/                # 示例项目（待添加）
├── README.md
└── CONTRIBUTING.md
```
