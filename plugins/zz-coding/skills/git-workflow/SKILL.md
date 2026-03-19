---
name: git-workflow
description: 为 Git 提交、分支命名、PR 描述和常见协作流程提供统一规范。适用于用户提到 commit、提交信息、branch、分支命名、pull request、PR、代码提交流程、Git 工作流时。
version: 0.1.0
metadata:
  source: "based on the public summary of agno-agi/git-workflow"
---

# Git Workflow

## 概述

该 skill 用于在日常开发中统一 Git 工作流，重点覆盖：

- 分支命名
- 提交信息格式
- Pull Request 标题与描述
- 常见协作命令顺序

如果仓库本身已有更严格规范，以仓库规范优先。

## 分支命名

默认格式：

```text
<type>/<ticket-id>-<short-description>
```

常见示例：

```text
feature/AUTH-123-oauth-login
fix/BUG-456-null-reference
chore/TECH-789-update-deps
```

### type 建议

| type | 使用场景 |
| ---- | -------- |
| `feature` | 新功能开发 |
| `fix` | 缺陷修复 |
| `refactor` | 重构但不改外部行为 |
| `chore` | 工具、配置、依赖维护 |
| `docs` | 文档更新 |
| `test` | 测试补充或重写 |

## 提交信息格式

默认采用 Conventional Commits：

```text
<type>(<scope>): <subject>
```

示例：

```text
feat(event1v1): add slap reward preview flow
fix(tv-panel): prevent delayed animation from using mutated runtime data
refactor(activity): extract panel refresh into isolated presenter logic
```

## 提交正文建议

当变更不小或跨多个文件时，补 1 到 2 段正文，说明：

1. 为什么做这次修改
2. 风险点或兼容性影响
3. 如何验证

示例：

```text
fix(tv-panel): prevent delayed animation from using mutated runtime data

Snapshot the display payload before async animation starts so the panel no
longer reads post-update runtime state.

Verified by checking delayed TV playback and reward settlement sequence.
```

## PR 标题与描述

PR 标题可直接沿用 commit 风格：

```text
fix(tv-panel): avoid stale activity state during delayed playback
```

PR 描述建议使用下面模板：

```markdown
## Summary
- 做了什么
- 为什么要做

## Changes
- 主要改动点 1
- 主要改动点 2

## Testing
- [ ] 本地编译
- [ ] 关键流程验证
- [ ] 回归影响确认
```

## 推荐工作流

### 开始开发

1. 从最新基线拉取代码
2. 创建语义化分支
3. 小步提交，避免一次 commit 混太多主题

### 提交前

1. 检查变更范围是否单一
2. 先运行必要验证
3. 确认不提交临时文件、日志、凭证和本地配置

### 发起 PR 前

1. Rebase 或同步目标分支最新代码
2. 自查 diff 是否干净
3. 在 PR 描述里写清测试与风险

## 常用命令

```bash
git checkout main
git pull
git checkout -b feature/AUTH-123-oauth-login

git status
git add <files>
git commit -m "feat(auth): implement oauth login"

git fetch origin
git rebase origin/main
git push -u origin HEAD
```

## 使用准则

- 优先拆分成语义清晰的小提交
- 一个提交只表达一个主题
- 标题写“意图”，不要只写“改了什么”
- 若仓库已有 commit message/branch 命名规范，优先遵循仓库规范

## 何时使用该 skill

- 用户说“帮我写 commit”
- 用户问“branch 怎么命名”
- 用户需要整理 PR 标题或描述
- 用户想统一团队 Git 提交流程
