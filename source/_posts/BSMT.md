---
title: Binary Sparse Merkle Tree (BSMT)
description: ''
tags: ['科研','区块链']
toc: false
date: 2025-11-21 14:02:48
categories:
    - 科研
    - 区块链
---

Binary Sparse Merkle Tree (BSMT) 不仅仅是一种数据结构，它是解决区块链 "状态爆炸" 和 "高效验证" 矛盾的关键技术方案。它结合了 Merkle Tree 的验证能力和 Trie（前缀树）的寻址能力。

> 本文结构：通俗讲解->运行机制->代码实现

<!--more-->

## A 什么事是BSMT？

为了让你一听就懂，我们可以把 **Binary Sparse Merkle Tree (BSMT)** 想象成一个 **“带有魔法的无限大仓库”** 。

我们可以把它的三个名字拆开来理解：

### 1. Binary（二叉）：这是“地图”
想象你要去仓库取东西，路标只有两种：**“向左”**（代表0）和 **“向右”**（代表1）。
所有的门牌号（Key）都是由0和1组成的。比如你要找门牌号 `011` 的柜子，你就走：左 -> 右 -> 右。
*   **通俗点说：** 只要知道门牌号（二进制），你就能顺着唯一的路径找到那个位置。

### 2. Sparse（稀疏）：这是“魔法”
这是 BSMT 最核心的技术。
这个仓库理论上**大到无穷大**（比如有 $2^{256}$ 个柜子，比宇宙里的原子还多）。
如果我们要真的造这么多柜子，地球都放不下。

**魔法在于：**
*   在这个无限大的仓库里，**99.9999% 的柜子都是空的** 。
*   对于空的柜子，我们 **不造它，也不存储它** 。
*   我们预先算好一个“空柜子的指纹”（Hash），凡是没东西的地方，直接用这个“空指纹”代替。
*   **通俗点说：** 这是一个 **“懒人仓库”** 。只有当你真的往某个柜子放东西时，我们才会在那一瞬间把去往那个柜子的路和柜子造出来。其他的空间都是虚拟的，不占地方。

### 3. Merkle Tree（默克尔树）：这是“防伪锁”
仓库大门上有一把总锁（Root Hash）。
*   如果你偷偷改了哪怕最深处一个柜子里的东西，这把总锁的“指纹”就会立刻改变。
*   **通俗点说：** 它可以极快地证明“某样东西确实在这个仓库里”，而且没人动过手脚。

---

### 总而言之

它就是一个**看起来有无限容量，但实际上只占用你存了多少东西的空间，且能随时极速验证数据真伪的超级数据库**。

**它解决了什么问题？**
*   **空间小：** 哪怕树深达256层，如果你只存了3个数据，它占用的空间就只和这3个数据相关，而不是整个宇宙。
*   **速度快：** 验证数据是否存在非常快（顺着0和1走就行）。
*   **甚至能证明“不存在”：** 你顺着路走到某个位置，发现那是“空指纹”，就能立刻证明“这个数据确实不存在”（Non-inclusion proof），这在区块链里非常有用。

## B 运行机制

好，我们继续用“魔法仓库”的比喻，把 **BSMT (Binary Sparse Merkle Tree)** 的具体运作流程拆解开来看。

首先你要记住一个核心规则：**在这个树里，位置（Key）就是导航图。**
比如你的 Key 是 `101`，那就代表：第一层向右走，第二层向左走，第三层向右走。

---

### 1. 存储：它是怎么做到“省空间”的？

普通的树是把所有节点都存进数据库，但 BSMT 是个“极简主义者”。

*   **预计算空节点（Zero Hashes）：**
    系统启动前，就已经算好了一套“空值表”。
    *   第256层（最底层）如果为空，Hash是 $H_0$。
    *   第255层如果左右孩子都为空（即孩子都是 $H_0$），那这一层的Hash就是 $H_1 = Hash(H_0, H_0)$。
    *   ...以此类推，算到根节点。
    **这些“空指纹”是永远不变的，不需要存进数据库，写在代码常量里就行。**

*   **只存有效路径：**
    当你要存一个数据时，BSMT **只会把从“根”到“叶子”这条路径上的非空节点** 存到数据库（通常是 KV 数据库，如 LevelDB）里。
    *   **如果一个节点的右边是空的？** 数据库里不存右边，只记录“右边是默认空指纹”。
    *   **效果：** 哪怕树有 256 层高，存一个数据只需要存 256 个节点（甚至经过优化压缩后更少），而不是存 $2^{256}$ 个节点。

---

### 2. 更新（插入/修改）：如何放入数据？

假设你要把 **“苹果”** 放入柜子 `10...0`（假设这是 Key）。

1.  **下沉（找位置）：**
    从根节点（Root）出发，根据 Key（1 -> 右，0 -> 左...）一路往下走。
2.  **放置：**
    走到最底层的叶子节点，把“苹果”放进去。
3.  **上浮（重算指纹）：**
    这是最关键的一步。你放了苹果，叶子的指纹变了。
    *   你需要算出新的叶子 Hash。
    *   回到父节点，父节点的 Hash = `Hash(左孩子Hash, 右孩子Hash)`。
    *   **注意：** 如果左孩子是你刚改的，右孩子是空的，那就取 `Hash(新左Hash, 预计算的空右Hash)`。
    *   一路向上计算，直到生成一个新的 **Root Hash**。

---

### 3. 证明存在（Merkle Proof）：怎么证明“苹果”在里面？

你要向别人证明：柜子 `10...0` 里确实有“苹果”。你不需要把整个仓库给别人看，只需要提供**一条“证据链”**。

*   **证据链（Proof）包含什么？**
    你需要提供从“苹果”所在位置，一直到树根的路径上，**所有“另一侧”的兄弟节点的 Hash**。
    *   比如：你要算你爸爸的 Hash，你需要你自己的 Hash + 你兄弟的 Hash。

*   **验证过程：**
    验证者手里只有一个 **Root Hash**（可信的总指纹）。
    1.  验证者拿着你给的“苹果”。
    2.  根据 Key（路径），配合你提供的“兄弟节点 Hash”，一层层往上算。
    3.  如果最后算出来的 Hash **等于** 他手里的 Root Hash，那就证明：**苹果确实在这个位置，且没有被篡改。**

---

### 4. 证明不存在（Non-Inclusion Proof）：怎么证明“没苹果”？

这是 BSMT 最厉害的地方。普通的 Merkle Tree 很难证明“没有”，通常需要把相邻的数据都拿出来由你来推断中间是空的，但 BSMT 很直接。

假设有人问：柜子 `11...1` 里有东西吗？

*   **情况 A：路断了（遇到空子树）**
    你顺着 `1`（向右）走，发现这里本该有个节点，但这棵子树全是空的（是一个预计算的空 Hash）。
    **证明：** 你直接把这个“空 Hash”对应的兄弟路径给验证者。验证者一算，发现根 Hash 对得上，且路径指向的是个“空地”，那就证明**该位置绝对没有数据**。

*   **情况 B：被占了（叶子节点不匹配）**
    你顺着路走到终点，发现里面确实有个东西，但它的 Key 是 `11...0`，而不是你要找的 `11...1`。
    **证明：** 展示这个错误的叶子节点。验证者一看：路径是对的，但这只有 `11...0` 的数据，说明 `11...1` 没地方站了（位置冲突），所以 **目标数据不存在**。

---

### 总结一下流程

| 动作 | 核心机制 | 通俗解释 |
| :--- | :--- | :--- |
| **存储** | **剪枝 (Pruning)** | 只有真的放了东西的柜子才会造出来，空的全部用“空气指纹”代替，不占硬盘。 |
| **更新** | **回溯重哈希** | 动了最下面的东西，要一层层往上汇报，直到大门上的总锁（Root）换成新的。 |
| **证明存在** | **兄弟路径** | “这是我和我每一级邻居的合影，你看最后能不能拼出那把总锁。” |
| **证明不存在** | **证明为空** | “你顺着路走，你看，那地方是一片荒地（默认空值），所以东西肯定不在那。” |

这就是为什么 BSMT 在区块链（如 CKB, Ethereum 2.0 的某些状态树）中很受欢迎：它**又大**（能存无限Key），**又小**（只占有效数据空间），还能**极速证明**“有”或者“没有”。

## C 代码实现（Go）

```Go
package main

import (
	"bytes"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

// 设定树的深度为 256 层 (对应 SHA256 的位宽)
const TreeDepth = 256

// SMT 定义
type SparseMerkleTree struct {
	root       []byte                // 当前树根 Hash
	db         map[string][]byte     // 模拟数据库 (NodeKey -> NodeHash)
	zeroHashes [TreeDepth + 1][]byte // 预计算的空节点 Hash 表（默认的，用作计算查询）
}

// NewSMT 初始化一棵空树
func NewSMT() *SparseMerkleTree {
	smt := &SparseMerkleTree{
		db: make(map[string][]byte),
	}

	// 1. 预计算空哈希 (魔法部分)
	// 最底层(第256层)的空 Hash 是全 0
	smt.zeroHashes[TreeDepth] = make([]byte, 32)

	// 从下往上计算每一层的空 Hash
	// Hash(Parent) = Hash(LeftChild + RightChild)
	for i := TreeDepth - 1; i >= 0; i-- {
		childHash := smt.zeroHashes[i+1]
		smt.zeroHashes[i] = hashNode(childHash, childHash)
	}

	// 初始状态下，树根就是第 0 层的空 Hash
	smt.root = smt.zeroHashes[0]
	return smt
}

// Set 插入或更新数据
// key: 32字节的大整数 (路径图), value: 要存的数据
func (t *SparseMerkleTree) Set(key []byte, value []byte) {
	valueHash := hashData(value)
	// 更新根节点，递归更新路径
	t.root = t.update(t.root, key, 0, valueHash)
}

// update 递归函数：下沉寻找位置，上浮重新计算 Hash
func (t *SparseMerkleTree) update(nodeHash []byte, key []byte, depth int, valueHash []byte) []byte {
	// 1. 到底了 (叶子节点)，直接返回数据的 Hash
	if depth == TreeDepth {
		return valueHash
	}

	// 2. 确定走左边还是右边 (查看 Key 在当前深度的位是 0 还是 1)
	// path: 0 -> Left, 1 -> Right
	bit := getBit(key, depth)

	// 获取当前节点的左右孩子
	// 如果当前节点是空的(ZeroHash)，那它的孩子肯定也是空的
	leftChild, rightChild := t.getChildren(nodeHash, depth)

	var newLeft, newRight []byte

	if bit == 0 {
		// 向左走，右边保持不变
		newLeft = t.update(leftChild, key, depth+1, valueHash)
		newRight = rightChild
	} else {
		// 向右走，左边保持不变
		newLeft = leftChild
		newRight = t.update(rightChild, key, depth+1, valueHash)
	}

	// 3. 上浮：计算当前节点的新 Hash
	newHash := hashNode(newLeft, newRight)

	// 4. 存储：把非空节点存入 DB (为了演示简单，这里我们存了所有经过的路径节点)
	// 实际 BSMT 优化中，如果 Hash 等于 ZeroHash，是不需要存 DB 的
	t.db[hex.EncodeToString(newHash)] = append(newLeft, newRight...)

	return newHash
}

// GenerateProof 生成默克尔证明 (Merkle Proof)
// 返回：兄弟节点 Hash 列表
func (t *SparseMerkleTree) GenerateProof(key []byte) [][]byte {
	var proof [][]byte
	currentNode := t.root

	for i := 0; i < TreeDepth; i++ {
		bit := getBit(key, i)
		left, right := t.getChildren(currentNode, i)

		if bit == 0 {
			// 我走左边，把右边的兄弟放入证据包
			proof = append(proof, right)
			currentNode = left
		} else {
			// 我走右边，把左边的兄弟放入证据包
			proof = append(proof, left)
			currentNode = right
		}
	}
	return proof
}

// VerifyProof 验证默克尔证明
func VerifyProof(root []byte, key []byte, value []byte, proof [][]byte) bool {
	currentHash := hashData(value)

	// 从底向上回推
	// 注意 proof 的顺序是 从上到下 还是 从下到上，这里生成时是从上到下，验证我们要反过来或者对应索引
	// 实际上循环最好还是从 Root 往下模拟，或者从 Leaf 往上计算。
	// 这里我们采用：从 Leaf (TreeDepth) 往上 (0) 计算

	if len(proof) != TreeDepth {
		return false
	}

	for i := TreeDepth - 1; i >= 0; i-- {
		sibling := proof[i] // 获取对应层的兄弟
		bit := getBit(key, i)

		if bit == 0 {
			// Key 是 0 (左)，兄弟在右： Hash(Me, Sibling)
			currentHash = hashNode(currentHash, sibling)
		} else {
			// Key 是 1 (右)，兄弟在左： Hash(Sibling, Me)
			currentHash = hashNode(sibling, currentHash)
		}
	}

	// 比较计算出的 Root 和 这里的 Root
	return bytes.Equal(currentHash, root)
}

// --- 辅助函数 ---

// getChildren 获取左右孩子，如果是空节点，返回预计算的 ZeroHash
func (t *SparseMerkleTree) getChildren(nodeHash []byte, depth int) ([]byte, []byte) {
	// 如果当前节点就是该层的“空哈希”，那孩子直接取下一层的空哈希
	if bytes.Equal(nodeHash, t.zeroHashes[depth]) {
		return t.zeroHashes[depth+1], t.zeroHashes[depth+1]
	}

	// 否则查 DB
	data, ok := t.db[hex.EncodeToString(nodeHash)]
	if !ok {
		// 理论上不应发生，除非数据丢失，但在 Sparse 逻辑里，找不到我们就当它是空的
		return t.zeroHashes[depth+1], t.zeroHashes[depth+1]
	}
	// 前32字节是左，后32字节是右
	return data[:32], data[32:]
}

// getBit 获取 Key 在第 depth 位的二进制值 (0 或 1)
func getBit(key []byte, depth int) int {
	byteIndex := depth / 8
	bitIndex := 7 - (depth % 8)
	if (key[byteIndex]>>bitIndex)&1 == 1 {
		return 1
	}
	return 0
}

// hashNode 计算中间节点 Hash = SHA256(Left + Right)
func hashNode(left, right []byte) []byte {
	hash := sha256.New()
	hash.Write(left)
	hash.Write(right)
	return hash.Sum(nil)
}

// hashData 计算叶子数据 Hash
func hashData(data []byte) []byte {
	hash := sha256.New()
	hash.Write(data)
	return hash.Sum(nil)
}

// --- 主程序 ---

func main() {
	// 1. 创建树
	smt := NewSMT()
	fmt.Printf("初始 Root: %x\n", smt.root)

	// 2. 准备数据
	// Key 需要是 32 字节 (256 bit)，这里简单构造一个
	key1 := make([]byte, 32)
	key1[31] = 0x01 // ...0001

	val1 := []byte("Hello BSMT")

	// 3. 存储数据
	fmt.Println("正在存储 'Hello BSMT' 到 key 1...")
	smt.Set(key1, val1)
	fmt.Printf("新 Root:   %x\n", smt.root)

	// 4. 生成证明
	fmt.Println("正在生成证明...")
	proof := smt.GenerateProof(key1)

	// 5. 验证证明 (模拟轻客户端验证)
	isValid := VerifyProof(smt.root, key1, val1, proof)
	fmt.Printf("验证结果: %v\n", isValid)

	// 6. 测试篡改验证
	fmt.Println("测试篡改数据...")
	fakeVal := []byte("Hacker")
	isValidFake := VerifyProof(smt.root, key1, fakeVal, proof)
	fmt.Printf("篡改验证结果: %v\n", isValidFake)
}
```