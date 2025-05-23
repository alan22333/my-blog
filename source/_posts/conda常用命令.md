---
title: conda常用命令
description: 'Anaconda 是一个流行的 Python 数据科学平台，主要用于管理虚拟环境和包。'
tags: ['科研', 'conda', 'python']
toc: false
date: 2025-05-20 18:51:23
categories:
---

Anaconda 是一个流行的 Python 数据科学平台，主要用于管理虚拟环境和包。以下是常用的 Anaconda 命令分类整理：

---

**1. 环境管理**
• 查看所有环境  

  ```bash
  conda env list
  ```
• 创建新环境  

  ```bash
  conda create --name myenv python=3.9  # 指定 Python 版本
  ```
• 激活/退出环境  

  ```bash
  conda activate myenv    # 激活环境
  conda deactivate        # 退出当前环境
  ```
• 删除环境  

  ```bash
  conda env remove --name myenv
  ```
• 导出/导入环境配置  

  ```bash
  conda env export > environment.yml    # 导出
  conda env create -f environment.yml   # 导入
  ```

---

**2. 包管理**
• 安装包  

  ```bash
  conda install numpy        # 安装指定包
  conda install numpy=1.21    # 指定版本
  pip install package_name    # 使用 pip 安装（conda 不支持的包）
  ```
• 卸载包  

  ```bash
  conda uninstall numpy
  ```
• 更新包  

  ```bash
  conda update numpy         # 更新单个包
  conda update --all         # 更新所有包
  ```
• 搜索包  

  ```bash
  conda search numpy
  ```
• 列出已安装包  

  ```bash
  conda list
  ```

---

**3. Anaconda 自身管理**
• 更新 Conda  

  ```bash
  conda update conda
  ```
• 更新 Anaconda  

  ```bash
  conda update anaconda
  ```
• 清理缓存/无用包  

  ```bash
  conda clean --all
  ```

---

**4. Jupyter Notebook 相关**
• 在特定环境中安装 Jupyter  

  ```bash
  conda install nb_conda       # 支持环境切换
  ```
• 启动 Jupyter Notebook  

  ```bash
  jupyter notebook
  ```
• 生成 Jupyter 内核（针对虚拟环境）  

  ```bash
  python -m ipykernel install --user --name myenv --display-name "Python (myenv)"
  ```

---

**5. 其他实用命令**
• 查看 Conda 信息  

  ```bash
  conda info
  ```
• 检查 Conda 版本  

  ```bash
  conda --version
  ```
• 从 requirements.txt 安装包（pip 格式）  

  ```bash
  pip install -r requirements.txt
  ```

---

**常见问题解决**
1. Conda 命令慢/卡顿  
   • 更换国内镜像源（如清华源）：  

     ```bash
     conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
     conda config --set show_channel_urls yes
     ```
   • 恢复默认源：  

     ```bash
     conda config --remove-key channels
     ```
---