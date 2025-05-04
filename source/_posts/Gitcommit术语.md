---
title: Git Commit术语
description: '在使用 git commit -m 提交代码时，通常会在提交信息中使用一些常见的前缀来描述提交的类型和目的'
date: 2025-05-03 21:34:37
tags: git
---

在使用 `git commit -m` 提交代码时，通常会在提交信息中使用一些常见的前缀来描述提交的类型和目的。这些前缀有助于团队更好地理解提交的内容，尤其是在查看提交历史或生成变更日志时。以下是一些常用的前缀及其含义：

---

### 1. **`fix`**

表示修复了一个 bug 或问题。

```Shell
git commit -m "fix: 修复登录页面无法加载的问题"
```


---

### 2. **`feat`**

表示新增了一个功能。

```Shell
git commit -m "feat: 添加用户注册功能"
```


---

### 3. **`docs`**

表示文档相关的更改（如 README、注释等）。

```Shell
git commit -m "docs: 更新项目 README 文件"
```


---

### 4. **`style`**

表示代码风格的更改（如格式化、缩进等），不涉及功能或逻辑的修改。

```Shell
git commit -m "style: 格式化代码缩进"
```


---

### 5. **`refactor`**

表示代码重构，既不修复 bug 也不添加新功能。

```Shell
git commit -m "refactor: 重构用户模块的代码结构"
```


---

### 6. **`test`**

表示测试相关的更改（如添加或修改测试用例）。

```Shell
git commit -m "test: 添加用户登录功能的单元测试"
```


---

### 7. **`chore`**

表示日常维护或工具相关的更改（如依赖更新、构建配置等）。

```Shell
git commit -m "chore: 更新项目依赖包"
```


---

### 8. **`perf`**

表示性能优化相关的更改。

```Shell
git commit -m "perf: 优化数据库查询性能"
```


---

### 9. **`ci`**

表示持续集成（CI）相关的更改（如 GitHub Actions、Travis CI 等）。

```Shell
git commit -m "ci: 添加 GitHub Actions 自动化测试"
```


---

### 10. **`revert`**

表示回滚之前的提交。

```Shell
git commit -m "revert: 回滚错误的用户注册功能提交"
```


---

### 11. **`build`**

表示构建系统或外部依赖的更改（如 Webpack、Gulp 等）。

```Shell
git commit -m "build: 更新 Webpack 配置"
```


---

### 12. **`wip`**

表示正在进行中的工作（Work In Progress），通常用于临时提交。

```Shell
git commit -m "wip: 用户模块开发中"
```


---

### 13. **`hotfix`**

表示紧急修复生产环境中的问题。

```Shell
git commit -m "hotfix: 紧急修复支付接口崩溃问题"
```


---

### 14. **`init`**

表示项目初始化或首次提交。

```Shell
git commit -m "init: 初始化项目"
```


---

### 15. **`merge`**

表示合并分支。

```Shell
git commit -m "merge: 合并 feature/login 分支到 main"
```


---

### 提交信息格式建议

通常推荐使用以下格式：

```Shell
<类型>: <描述>
```


例如：

```Shell
git commit -m "feat: 添加用户登录功能"
```


如果需要更详细的描述，可以使用多行提交信息：

```Shell
git commit -m "fix: 修复登录页面无法加载的问题

- 修复了登录页面因 API 接口错误导致的加载失败问题
- 优化了错误提示信息"
```


---

### 总结

- 使用清晰的前缀（如 `fix`、`feat`、`docs` 等）来描述提交的类型。

- 提交信息应简洁明了，便于团队协作和代码审查。

- 如果需要更详细的描述，可以使用多行提交信息。

这些规范可以帮助团队更好地管理提交历史，并生成更有意义的变更日志。

