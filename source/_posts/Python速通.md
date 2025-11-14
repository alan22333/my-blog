---
title: Python速通
description: '速通一下Python，以备不时之需'
tags: ['科研','python']
toc: false
date: 2025-09-20 19:48:11
categories:
---

# Python 实用速成教程（科研导向，完整版）

> 面向已经掌握其他编程语言的程序员，帮助快速掌握 Python 的科研常用技能。强调**实用性、完整性和清晰表达**，避免炫技写法。

---

## 1. 环境准备

推荐方式：使用虚拟环境管理依赖。

```bash
# 创建虚拟环境
python3 -m venv venv
source venv/bin/activate   # Linux / Mac
venv\Scripts\activate      # Windows

# 安装科学计算常用包
pip install numpy scipy pandas matplotlib jupyter
```

启动 Jupyter Notebook：

```bash
jupyter notebook
```

---

## 2. 基本语法

### 2.1 变量与数据类型

```python
x = 10           # 整数
pi = 3.14        # 浮点数
name = "Alan"    # 字符串
flag = True      # 布尔值
```

### 2.2 条件语句

```python
if x > 0:
    print("正数")
elif x == 0:
    print("零")
else:
    print("负数")
```

### 2.3 循环

```python
for i in range(5):
    print(i)

count = 0
while count < 3:
    print("循环", count)
    count += 1
```

---

## 3. 常用数据结构

### 3.1 列表

```python
nums = [1, 2, 3]
nums.append(4)
print(nums[0])
```

### 3.2 字典

```python
person = {"name": "Alan", "age": 25}
print(person["name"])

# 安全访问
print(person.get("gender", "未知"))
```

### 3.3 集合

```python
s = {1, 2, 3}
s.add(2)
print(s)
```

---

## 4. 函数与模块

```python
def greet(name):
    return f"Hello, {name}"

print(greet("Alan"))
```

导入模块：

```python
import math
print(math.sqrt(16))
```

---

## 5. 文件与数据读写

### 5.1 文本文件

```python
with open("data.txt", "r") as f:
    for line in f:
        print(line.strip())
```

### 5.2 CSV 文件

```python
import csv

with open("data.csv", newline="") as f:
    reader = csv.reader(f)
    for row in reader:
        print(row)
```

### 5.3 JSON 文件

```python
import json

with open("config.json") as f:
    data = json.load(f)
print(data["param"])
```

---

## 6. NumPy 基础

### 6.1 创建数组

```python
import numpy as np

arr = np.array([1, 2, 3, 4])
print(arr)
```

### 6.2 数值运算

```python
print(arr + 10)
print(arr.mean())
print(arr.max())
```

### 6.3 矩阵运算

```python
A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])

print(A + B)
print(A @ B)  # 矩阵乘法
```

---

## 7. pandas 数据分析

```python
import pandas as pd

# 读取 CSV 文件
df = pd.read_csv("data.csv")

# 查看前几行
print(df.head())

# 基本统计
print(df.describe())

# 筛选数据
print(df[df["value"] > 10])

# 分组统计
print(df.groupby("group")["value"].mean())
```

---

## 8. Matplotlib 绘图

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 10, 100)
y = np.sin(x)

plt.plot(x, y, label="sin(x)")
plt.xlabel("x")
plt.ylabel("sin(x)")
plt.title("正弦函数")
plt.legend()
plt.show()
```

---

## 9. SciPy 常用功能

### 9.1 优化

```python
from scipy.optimize import minimize

res = minimize(lambda x: (x-3)**2, x0=0)
print(res.x)
```

### 9.2 统计检验

```python
from scipy import stats

t, p = stats.ttest_ind([1,2,3], [1.1,2.1,3.1])
print("t=", t, "p=", p)
```

---

## 10. 并行与性能

### 10.1 多进程

```python
from multiprocessing import Pool

def square(x):
    return x*x

with Pool(4) as p:
    results = p.map(square, range(10))
print(results)
```

### 10.2 向量化替代循环

```python
# 慢：for 循环
out = [np.sin(v) for v in arr]

# 快：向量化
out = np.sin(arr)
```

---

## 11. 实用技巧与常见陷阱

* 使用 `f"..."` 进行格式化输出。
* 文件操作时总用 `with open`，避免忘记关闭。
* 默认参数不要用可变对象：

```python
def f(x, lst=None):
    if lst is None:
        lst = []
    lst.append(x)
    return lst
```

* 浮点数比较时使用 `math.isclose()`。

---

## 12. 实践案例：科研数据分析流程

### 步骤：读取 → 清洗 → 分析 → 可视化 → 保存结果

```python
import pandas as pd
import matplotlib.pyplot as plt

# 1. 读取数据
df = pd.read_csv("experiment.csv")

# 2. 数据清洗
df = df.dropna()  # 去掉缺失值

# 3. 基本统计
print(df.describe())

# 4. 分组统计
summary = df.groupby("group")["value"].mean()
print(summary)

# 5. 可视化
summary.plot(kind="bar")
plt.xlabel("Group")
plt.ylabel("Mean Value")
plt.title("实验结果对比")
plt.show()

# 6. 保存结果
summary.to_csv("summary.csv")
```

---

## 13. 推荐资源

* [Python 官方教程](https://docs.python.org/3/tutorial/)
* [NumPy 用户指南](https://numpy.org/doc/stable/user/index.html)
* [pandas 文档](https://pandas.pydata.org/docs/)
* [Matplotlib 教程](https://matplotlib.org/stable/tutorials/index.html)
* 书籍：《Python for Data Analysis》

---

## 总结

* Python 语法简单，重点是熟悉科学计算库。
* 科研最常用的工具链：**NumPy + pandas + Matplotlib + SciPy**。
* 建议从实际任务出发，边做边学。

---
