---
title: Git合作开发流程
description: '使用Git进行团队协作开发可以高效管理代码'
tags: ['git']
toc: false
date: 2025-05-04 20:00:20
categories:
---

使用Git进行团队协作开发可以高效管理代码，以下是详细步骤和最佳实践：

---

### **1. 创建远程仓库**

- **选择平台**：在GitHub、GitLab或Gitee上创建远程仓库。

- **初始化仓库**：由一人创建空仓库，或推送现有项目：

  ```Shell
git init
git add .
git commit -m "Initial commit"
git remote add origin <远程仓库URL>
git push -u origin main
```


---

### **2. 克隆仓库到本地**

其他成员克隆仓库到本地：

```Shell
git clone <远程仓库URL>
```


---

### **3. 分支管理策略**

- **主分支（main/master）**：稳定版本，仅通过合并更新。

- **开发分支（develop）**（可选）：集成新功能，适合复杂项目。

- **功能分支（feature/xxx）**：每个新功能在独立分支开发。

**常用命令**：

- 创建并切换分支：

  ```Shell
git checkout -b feature/login
```


- 推送分支到远程：

  ```Shell
git push origin feature/login
```


---

### **4. 日常开发流程**

- **频繁提交**：小步提交，描述清晰：

  ```Shell
git add .
git commit -m "feat: 添加用户登录功能"
```


- **推送更改**：

  ```Shell
git push origin feature/login
```


---

### **5. 代码合并与审查**

- **发起Pull Request (PR)/Merge Request (MR)**：

  1. 在远程仓库页面选择分支发起PR。

  2. 团队成员审查代码，讨论修改。

  3. 确认无误后合并到主分支。

- **合并分支**（本地操作）：

  ```Shell
git checkout main
git pull origin main       # 更新主分支
git merge feature/login    # 合并功能分支
git push origin main       # 推送合并结果
```


---

### **6. 处理冲突**

- 当多人修改同一文件时，拉取最新代码并解决冲突：

  ```Shell
git pull origin main       # 拉取最新代码，发现冲突
# 手动编辑冲突文件，保留需要的代码
git add .
git commit -m "fix: 解决合并冲突"
git push origin feature/login
```


---

### **7. 同步更新**

- **定期拉取主分支**：

  ```Shell
git checkout main
git pull origin main
```


- **Rebase保持历史整洁**（可选）：

  ```Shell
git checkout feature/login
git rebase main           # 将主分支更新应用到当前分支
```


---

### **8. 其他最佳实践**

- **.gitignore文件**：排除临时文件、日志、编译产物。

- **Commit规范**：使用语义化标签（如`feat:`, `fix:`, `docs:`）。

- **分支清理**：合并后删除已无用的分支：

  ```Shell
git branch -d feature/login   # 删除本地分支
git push origin --delete feature/login  # 删除远程分支
```


---

### **9. 选择协作模型**

- **GitHub Flow**（简单）：

  - 主分支始终可部署。

  - 功能分支开发 → PR合并 → 立即部署。

- **Git Flow**（复杂）：

  - 包含`develop`、`release`、`hotfix`等分支，适合定期发布。

---




