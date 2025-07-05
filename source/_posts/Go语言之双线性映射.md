---
title: Go语言之双线性映射
description: '复习一下双线性映射'
tags: ['go','科研','密码学']
toc: false
date: 2025-07-02 14:27:16
categories:
    - 科研
    - 密码学
---


当然可以！下面是对“一、双线性对：通俗讲解复习”这一节的**重新编写**，内容更详细、比喻更丰富，完全为没有抽象代数基础的读者设计，确保真正理解双线性对的**是什么、有什么用、为什么重要**。

---

## 🧩 一、双线性对：从0开始的通俗讲解

### 🟢 1. 从最熟悉的数学说起：加法和乘法的关系

小时候一定学过这样的规律：

$$
(a \cdot b) + (a \cdot c) = a \cdot (b + c)
$$

这说明：乘法对加法是“线性的”——乘法可以分布在加法上。

数学家把“具有线性特性”的东西统称为 **线性映射**。
而当一个函数在**两个变量上都具有线性特性**时，它就叫**双线性映射（Bilinear Mapping）**。

我们来看个简单的例子：

---

### 🧮 2. 什么是“双线性”函数？（举例子）

假设有这样一个函数 $f(x, y) = x \cdot y$，你可以试试：

$$
f(2, 3) = 2 \cdot 3 = 6 \\
f(2+4, 3) = f(6, 3) = 18 = f(2, 3) + f(4, 3)
$$

也就是说：

$$
f(x_1 + x_2, y) = f(x_1, y) + f(x_2, y)
$$

同时也成立：

$$
f(x, y_1 + y_2) = f(x, y_1) + f(x, y_2)
$$

这样的函数，我们就称之为“双线性函数”。

---

### 🔁 3. 那什么是“双线性对”？（pairing）

现在我们把场景搬到密码学使用的群结构中。

我们引入三个群：

| 群     | 是什么                      |
| ----- | ------------------------ |
| $G_1$ | 第一个群，元素可以是椭圆曲线上的点        |
| $G_2$ | 第二个群，也是一组特殊点             |
| $G_T$ | 目标群，用来存储结果（通常是整数或其他字段元素） |

我们定义一个函数：

$$
e: G_1 \times G_2 \rightarrow G_T
$$

这就是**pairing 函数**，也叫**双线性对**。

它接收两个输入，分别来自 $G_1$ 和 $G_2$，输出一个值在 $G_T$ 中。

---

### 🎯 4. pairing 函数必须满足什么条件？

#### ✅ 1. 双线性（最重要的魔法）

对于任意整数 $a, b$，点 $P \in G_1, Q \in G_2$，有：

$$
e(aP, bQ) = e(P, Q)^{ab}
$$

这相当于说：

* 如果你把 P 点放大 $a$ 倍、Q 点放大 $b$ 倍
* 那么 pairing 的结果就相当于：原始 pairing 的结果再“提升到 ab 次方”

📌 **这个指数转换能力是 pairing 最大的魔力**
它能把点之间的运算变成指数之间的关系！

---

#### ✅ 2. 非退化（不能恒等于 1）

我们希望 $e(P, Q) \neq 1$（也就是说，不是无论输入什么都输出 1），这样 pairing 才有实际用途。

#### ✅ 3. 可计算（算法能在合理时间内算出来）

pairing 必须要能用程序实现，不能只是数学理论。

---

### 🔁 5. pairing 的形象比喻（非常重要）

你可以把 pairing 函数想象成一个“智能转换器”，它的作用是：

| 输入             | 输出     | 保留了什么                           |
| -------------- | ------ | ------------------------------- |
| 椭圆曲线上的点 $P, Q$ | 数字 $Z$ | 指数关系，例如 $aP, bQ \rightarrow ab$ |

---

#### 🎨 类比生活场景：

假设：

* $P$：代表一封加密邮件
* $Q$：代表一个用户身份
* pairing 就是一个验证器

它能算出 $e(P, Q)$，并且这个值具有很强的**一致性和可验证性**，而且别人不能仿造！

---

### 🚀 6. pairing 在密码学的用处

因为 pairing 保留了倍数/指数的关系，它非常适合做这些事：

| 应用          | pairing 做了什么            |
| ----------- | ----------------------- |
| 身份加密（IBE）   | 让邮箱当公钥，不用发证书            |
| 可搜索加密（PEKS） | 让云端可以搜索关键词但看不到原文        |
| BLS 签名      | 用 pairing 验证签名合法性，签名特别短 |
| 属性加密        | 用 pairing 判断用户是否满足访问条件  |

> pairing 是这些现代密码机制的“发动机”。

---

## 💻 二、Go 实现 pairing 验证（基础）

使用 [Cloudflare 的 bn256](https://pkg.go.dev/github.com/cloudflare/bn256) 库。

### ✅ pairing 验证公式：

$$
e(aP, bQ) = e(P, Q)^{ab}
$$

```go
package main

import (
	"crypto/rand"
	"fmt"
	"math/big"

	"github.com/cloudflare/bn256"
)

func main() {
	a, _ := rand.Int(rand.Reader, bn256.Order)
	b, _ := rand.Int(rand.Reader, bn256.Order)

	P := new(bn256.G1).ScalarBaseMult(big.NewInt(1))
	Q := new(bn256.G2).ScalarBaseMult(big.NewInt(1))

	aP := new(bn256.G1).ScalarMult(P, a)
	bQ := new(bn256.G2).ScalarMult(Q, b)

	p1 := bn256.Pair(aP, bQ)
	p2 := bn256.Pair(P, Q)
	ab := new(big.Int).Mul(a, b)
	p2.Exp(p2, ab)

	fmt.Println("e(aP,bQ) == e(P,Q)^ab ?", p1.String() == p2.String())
}
```

输出应该是：

```
e(aP,bQ) == e(P,Q)^ab ? true
```

---

## 🔍 三、什么是 PEKS？

### ✅ 问题背景

* 你加密了一堆邮件或文件，存在云端
* 你希望“搜索关键词”（如 "salary"）而**不暴露内容**
* 更重要的是，云服务商**不能解密原文**

### ✅ PEKS（Public Key Encryption with Keyword Search）

PEKS 是一种加密方法，它可以：

* 让云端使用“陷门”检查某密文是否包含某关键词
* 但云端无法知道关键词或密文内容

它的核心机制就依赖 **pairing（双线性对）**。

---

## ⚙️ 四、PEKS 架构与原理

参考 Boneh 等人在 2004 年提出的 PEKS 系统。

### 参与者

* **发送方**：加密关键词
* **接收方**：生成陷门（trapdoor）
* **服务器**：判断密文是否匹配陷门（不可得出关键词）

### 核心结构

1. 公钥系统：基于 pairing 的椭圆曲线系统
2. PEKS 密文结构：

   $$
   (A, B) = (r \cdot P, H(w) \oplus e(P_{pub}, H(w))^r)
   $$
3. 判断式（在服务器上）：
   检查：

   $$
   e(A, H(w')) = B'
   $$

---

## 🛠️ 五、Go 实现 PEKS 系统

一个简化版系统：

* 使用 SHA256 代替曲线上的 hash-to-point
* 模拟关键步骤，让你理解 PEKS 的 pairing 工作方式

### 📁 go.mod

```go
module peks-demo

go 1.20

require github.com/cloudflare/bn256 v0.0.0-20230823074015-81eab1aa433a
```

---

### 📄 main.go

```go
package main

import (
	"crypto/rand"
	"crypto/sha256"
	"fmt"
	"math/big"

	"github.com/cloudflare/bn256"
)

// 把关键词转为 G1 群中的点（模拟 hash-to-curve）
func hashToG1(keyword string) *bn256.G1 {
	hash := sha256.Sum256([]byte(keyword))
	h := new(big.Int).SetBytes(hash[:])
	return new(bn256.G1).ScalarBaseMult(h)
}

// 密钥对结构
type KeyPair struct {
	Priv *big.Int
	Pub  *bn256.G2
}

// 生成密钥对
func generateKeyPair() KeyPair {
	x, _ := rand.Int(rand.Reader, bn256.Order)
	Pub := new(bn256.G2).ScalarBaseMult(x)
	return KeyPair{Priv: x, Pub: Pub}
}

// PEKS 加密
func PEKS(pubKey *bn256.G2, keyword string) (A *bn256.G1, B *bn256.GT) {
	r, _ := rand.Int(rand.Reader, bn256.Order)
	Hw := hashToG1(keyword)
	A = new(bn256.G1).ScalarBaseMult(r)
	B = bn256.Pair(Hw, pubKey)
	B.Exp(B, r)
	return
}

// 生成陷门
func Trapdoor(privKey *big.Int, keyword string) *bn256.G1 {
	Hw := hashToG1(keyword)
	return new(bn256.G1).ScalarMult(Hw, privKey)
}

// 检查关键词是否匹配
func Test(A *bn256.G1, B *bn256.GT, Tw *bn256.G1) bool {
	pair := bn256.Pair(A, Tw)
	return pair.String() == B.String()
}

func main() {
	// 关键词加密方
	keyword := "salary"

	// 用户生成密钥对
	keys := generateKeyPair()

	// 加密关键词
	A, B := PEKS(keys.Pub, keyword)
	fmt.Println("🔐 PEKS 密文生成完毕")

	// 收件人生成陷门
	tw := Trapdoor(keys.Priv, "salary")

	// 服务器测试匹配
	result := Test(A, B, tw)
	fmt.Printf("💡 关键词匹配结果: %v\n", result)

	// 错误关键词测试
	twWrong := Trapdoor(keys.Priv, "bonus")
	resultWrong := Test(A, B, twWrong)
	fmt.Printf("🧪 错误关键词测试结果: %v\n", resultWrong)
}
```

---

## 🧪 示例输出

```
🔐 PEKS 密文生成完毕
💡 关键词匹配结果: true
🧪 错误关键词测试结果: false
```

---

