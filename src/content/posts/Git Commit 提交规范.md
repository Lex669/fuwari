---
title: Git Commit 提交规范
published: 2024-05-01
updated: 2024-11-29
description: 'Git Commit 提交规范'
image: ''
tags: [Git, Tutorial]
category: 'Git'
draft: false 
---
# Git Commit 提交规范

## Conventional Commits（约定式提交）

格式：

```
<type>(<scope>): <subject>

<body>

<footer>
```

### 结构说明

| 部分 | 必填 | 说明 |
|------|------|------|
| type | 是 | 提交类型 |
| scope | 否 | 影响范围（模块/组件名） |
| subject | 是 | 简短描述（≤50 字符，中文/英文） |
| body | 否 | 详细描述（每行 ≤72 字符） |
| footer | 否 | 关联 issue、breaking change 等 |

### 常用 type

| type       | 含义                    |
| ---------- | --------------------- |
| `feat`     | 新功能                   |
| `fix`      | 修复 bug                |
| `docs`     | 文档变更                  |
| `style`    | 代码格式（空格、缩进、分号等，不影响逻辑） |
| `refactor` | 重构（既非新功能也非修复）         |
| `perf`     | 性能优化                  |
| `test`     | 增加或修改测试               |
| `chore`    | 构建过程或辅助工具的变动          |
| `ci`       | CI/CD 配置变更            |
| `build`    | 影响构建系统或外部依赖的变更        |
| `revert`   | 回退之前的提交               |

### 示例

```
feat(tgv): 添加阴影成像法 TGV 检测笔记

feat: add shadowgraphy-based TGV inspection system

fix(parser): 修复 Markdown 表格解析异常

docs: 更新 AGENTS.md 论文笔记规范

refactor(ros): 将 launch 文件迁移至 ROS2 格式
```

### Breaking Change（破坏性变更）

在 footer 添加 `BREAKING CHANGE:` 或在 type/scope 后加 `!`：

```
feat(api)!: 重构检测接口，不再兼容 v1

BREAKING CHANGE: 删除了 /api/v1/inspect 端点，请迁移至 /api/v2/measure
```

### Footer 关联 Issue

```
fix: 修复 TGV 直径计算偏差

Closes #42
Refs: #38
```

## 常见反模式

- 避免无意义提交信息：`fix bug`、`update code`、`.`、`WIP`
- 避免超长单行（subject 控制在 50 字符以内）
- 避免混合无关变更（一次提交只做一件事）
- 避免提交未完成的工作（如 `WIP: half done`）

## 本仓库习惯

本仓库（Obsidian vault）实践中使用简短 `update` 风格，符合个人笔记仓库的轻量需求。在协作项目或代码仓库中应使用 Conventional Commits。

## 参考

- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/zh-hans/v1.0.0/)
- [Angular Commit Guidelines](https://github.com/angular/angular/blob/main/CONTRIBUTING.md#commit)
