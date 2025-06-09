---
title: go-blockchain学习笔记01
description: '使用 Golang 一步步构建一个基本的区块链系统的学习笔记'
tags: ['go','区块链']
toc: false
date: 2025-06-09 20:01:03
categories:
---

前言：近日看到一篇非常棒的中文文章，讲解使用 Golang 一步步构建一个基本的区块链系统，鼠鼠直接双厨狂喜，一口气看了小一半，受益匪浅。随着不断地更新代码，鼠鼠感觉有必要写一个总结记录一下这个很棒的项目，本文无 AI 总结，观点来自鼠鼠个人，请自行斟酌

### 基本认识

首先，区块链顾名思义就是由区块构成的链表，而区块则作为区块链的基本元素。区块链的核心特性是**去中心化、不可篡改和透明可追溯**。那么该如何实现以上三个特性呢？具体到数据结构上来说可以总结为：

- **去中心化：**

  - **分布式账本（Distributed Ledger）**

  - **共识机制（Consensus Mechanism）**

- **不可篡改：**

  - **区块哈希（Hash Block）**

  - **默克尔树（Merkle Tree）**

- **透明可追溯：**

  - **区块头（Block Header）**

  - **交易记录（Transaction Records）**

  - **地址（Addresses）**

其中关于分布式系统、默克尔树以及地址相关的内容后面再讨论，我们先设计一个最基本的区块链结构，首先是区块结构：

```Go
// block.go
type Block struct {
	Timestamp int64
	Hash      []byte
	PrevHash  []byte
	Target    []byte
	Nonce     int64
	// Data      []byte
	Transactions []*transaction.Transaction // real data
}
```

区块中的关键元素：区块头（时间戳、区块哈希、前一块哈希、挖矿目标、随机数）+交易记录

时间戳比较好理解，时间戳在区块链中是确保数据按时间顺序排列、防范双重支付、维护共识安全以及提供历史证据的关键数字印记。

区块中的“哈希指针”是通过将前一区块的哈希值包含在当前区块中，形成了环环相扣的链式结构。任何对历史区块数据的微小改动都会导致其哈希值发生变化，进而导致后续所有区块的哈希值失效，从而被网络轻易检测到。

之于挖矿目标和随机数后文涉及到再进行解释。

然后我们定义区块链：

```Go
// blockchain.go
type BlockChain struct {
	Blocks []*Block
}
```

现在我们已经有了区块链系统的基本元素，下面就是让他动起来，赋予他们一些方法

### 哈希机制

哈希是区块之间逻辑上的连接手段，哈希结果也包含了区块的所有信息，所以哈希也可以作为区块的唯一表示（ID），这个思想也用在交易这个概念上，这个后面会提到。下面我们来具体看一下怎么创建区块和计算区块哈希的：

```Go
// block.go
// package txs
func (b *Block) BackTrasactionSummary() []byte {
	txIDs := make([][]byte, 0)
	for _, tx := range b.Transactions {
		txIDs = append(txIDs, tx.ID)
	}
	summary := bytes.Join(txIDs, []byte{})
	return summary
}

// compute and set block hash
func (b *Block) SetHash() {
	information := bytes.Join(
		[][]byte{
			utils.ToHexInt(b.Timestamp),
			b.PrevHash,
			utils.ToHexInt(b.Nonce),
			b.Target,
			b.BackTrasactionSummary(),
		},
		[]byte{},
	)
	hash := sha256.Sum256(information)
	b.Hash = hash[:]
}
```

其实就是将信息打包，然后使用 sha256 算法进行哈希运算，其中 SHA-256 算法是一种**广泛应用于加密货币和信息安全的哈希函数**，它能将任意长度的输入数据转换成一个固定长度的 256 位（32 字节）哈希值，且该过程是单向的，微小的输入变化都会导致输出哈希值发生巨大改变

### 创建区块

首先是普通区块的创建，需要前置哈希和交易信息，另外就是 POW 相关的两个参数，后面也会专门来介绍：

```Go
// block.go
func CreateBlock(prevhash []byte, txs []*transaction.Transaction) *Block {
	block := Block{
		Timestamp:    time.Now().Unix(),
		Hash:         []byte{},
		PrevHash:     prevhash,
		Target:       []byte{},
		Nonce:        0,
		Transactions: txs,
	}
	// compute target and hash
	block.Target = block.GetTarget()
	block.Nonce = block.FindNonce()
	block.SetHash()
	return &block
}
```

而区块链中的第一个区块叫做“创世区块”，他是没有前驱的，我们来构建一个创世区块（先不用管交易相关）：

```Go
// block.go
func GenesisBlock() *Block {
	// make a base-tx: generate init coin for the God
	tx := transaction.BaseTx([]byte("alan"))
	return CreateBlock([]byte{}, []*transaction.Transaction{tx})
}

```

### 共识机制（POW 工作量证明）

网络中所有节点都可以构造区块然后添加到链上，但是这是不可能，因为区块链是一条链表而不是树形结构，所以就需要一种机制来保证每次只能有一个区块产生并添加到链上，并且要让其他所有的节点都认可这个新产生的区块，这就是共识机制。这里采用的是最最最最经典的比特币的共识机制，即**工作量证明（Proof Of Work）**。

采用原文的比喻就是“要在相对公平的条件下让想要添加自己的候选区块进区块链的节点内卷，通过竞争选择出一个大家公认的节点来添加它的区块进入区块链。整个共识机制被分为两部分，首先是竞争，然后是共识”。

游戏的规则是这样的：所有的网络节点（矿工）都做同一种工作，像是一种竞赛（挖矿），率先完成的节点得到记账权，也就是打包发布区块的权利。既然是 game 就一定有奖励，而奖励就是发布区块的节点可以得到“出块奖励”，例如比特币，这也是大家争相内卷的原因。

具体到工作的细节，这样一件工作非常的公平，比拼的因素只有一个就是算力，谁的算力高，谁就更有机会成为最终卷王，那么既然要求算力，计算过程究竟是什么呢？其实非常的简单，就是找一个随机数（nonce），把他打包到自己的候选区块中区进行哈希运算，转换成数之后和一个目标（target）对比，如果小于这个目标则视为计算成功，而这个目标则可以通过难度系数（difficulty）进行控制。

**这个“区块寻找”游戏设计得非常精妙：**

- 首先，每个节点需要找到的 **nonce**只对它自己提议的区块有效，这杜绝了“抄袭”的可能，确保了每个节点都必须独立完成任务。

- 其次，寻找这个 **nonce** 的过程是纯粹随机的，没有任何捷径可言；找到 **nonce** 的时间主要取决于网络设定的难度目标和节点自身的计算能力，但即使是性能较弱的节点，也有公平的机会赢得胜利。

- 最后，虽然寻找 **nonce** 可能需要投入大量的时间和计算资源，但验证一个节点是否真的找到了正确的 **nonce** 却非常迅速且几乎不消耗资源。可以说，这个被找到的 **nonce** 就是该节点“实力”的直接证明。

说完了基本的规则，我们来看一下如何实现：

首先我们设定一个难度值：

```Go
// constcoe.go

const (
	Difficulty = 12
	Initcoin = 1_000
)
```

然后我们就可以计算目标了：

```Go
// proofofwork.go

// difficulty larger => target smaller
func (b *Block) GetTarget() []byte {
	target := big.NewInt(1)
	target.Lsh(target, uint(256-constcoe.Difficulty))
	return target.Bytes()
}
```

有了目标之后，我们就可以填入不同的 nonce 来进行“挖矿”竞争卷王了：

```Go
// proofofwork.go

// join with nonce
func (b *Block) GetBase4Nonce(nonce int64) []byte{
	data := bytes.Join(
		[][]byte{
			utils.ToHexInt(b.Timestamp),
			b.PrevHash,
			utils.ToHexInt(int64(nonce)),
			b.Target,
			b.BackTrasactionSummary(),
		},
		[]byte{},
	)
	return data
}

// find right nonce (mine)
func (b *Block)FindNonce() int64{
	var intHash big.Int
	var intTarget big.Int
	var hash [32]byte
	var nonce int64 = 0

	intTarget.SetBytes(b.Target)
	for nonce < math.MaxInt64 {
		data := b.GetBase4Nonce(nonce)
		hash = sha256.Sum256(data)
		intHash.SetBytes(hash[:])
		if intHash.Cmp(&intTarget) == -1 {
			break
		}else {
			nonce ++
		}
	}
	return nonce
}
```

其实过程非常非常简单，就是一个个的尝试 nonce 的值进行计算和 target 来比较，另外我们也可以是设置一个方法来验证区块的有效性，如上文所说校验是非常简单的：

```Go
// proofofwork.go

// validate the block: compare with the target
func (b *Block) ValidatePoW() bool {
	var intHash big.Int
	var intTarget big.Int
	var hash [32]byte
	intTarget.SetBytes(b.Target)
	data := b.GetBase4Nonce(b.Nonce)
	hash = sha256.Sum256(data)
	intHash.SetBytes(hash[:])
	if intHash.Cmp(&intTarget) == -1 {
		return true
	}
	return false
}
```

至此，我们的区块链系统已经初具雏形了

### 交易机制（UTXO）

在上面的介绍中多次提到了交易，那么什么是交易？比特币系统中的交易又是什么样的呢？这一小节我来给大家介绍一下。

传统意义上的交易指的是用户之间的转账记录，例如 A 给 B 转了 1 个 bitcoin。但随着这个区块链领域的不断发展，即使在金融领域之外，我们还是习惯把区块里有用的数据条目称为“交易信息”（txs）。

一般来说，我们接触到的最常见的交易也许在银行，假设你有 100 ￥，你到银行转了 50 ￥给我，我去银行机器查看，发现余额多了 50 ￥，那么这样一次交易就完成了。但是，如果突然这个银行没了，那么如何证明我们之间的这笔交易存在呢？甚至该如何证明我们拥有资产？

问题的关键就是有一样东西消失了，在这个例子中就是银行，他是中心化设施的代表，也就是说我们所有的用户相信它，我们的资产才得以存在价值。银行记录了记录了我们的资产信息，可以轻松验证所有交易的有效性，只需要查看银行余额，因为我们相信这个余额。

那么，是否可以不要这样一个可信第三方，区块链就是旨在构建这样一个去中心化的分布式系统。而比特币系统就提出了这样一个模型：**UTXO 模型** 也就是“追溯历史交易”，要验证“A 转给 B 五块钱”是否有效，我们可以往前查找所有 A 作为接收方的交易记录，然后把这些交易中的金额加起来。如果总金额大于或等于 5，那么这笔转账就是有效的。

对于 UTXO 建议移步鼠鼠的另一篇文章了解，站内搜索即可快速访问，这里不在赘述。

我们直接用代码来构建“交易”，我会用大量的注释来代替讲解，聪明的你一定能看懂：

首先是 UTXO 模型中的输入输出的定义：

```Go
// inoutput.go
package transaction

import "bytes"

type TxOutput struct{
	Value int // output amount
	ToAddress []byte // output address
}

type TxInput struct{
	TxId []byte // pre-tx id
	OutIdx int // index of pre-tx output
	FromAddress []byte // input-from equals output-to
}


// utils check address
func (in *TxInput) FromAddressRight(address []byte) bool {
	return bytes.Equal(in.FromAddress, address)
}

func (out *TxOutput) ToAddressRight(address []byte) bool {
	return bytes.Equal(out.ToAddress, address)
}

```

然后我们定义交易：

```Go
// transaction.go

// Transaction 表示一笔交易，包含交易 ID、输入和输出
type Transaction struct {
	ID      []byte     // 交易 ID，是该交易内容的哈希值
	Inputs  []TxInput  // 输入数组（引用以前交易输出）
	Outputs []TxOutput // 输出数组（给哪些地址转了多少钱）
}
```

设置交易哈希：

```Go
// transaction.go
// TxHash 计算当前交易的哈希值（即交易 ID）
// 哈希值是对整个交易序列化后求 sha256，保证内容唯一性和不可篡改
func (tx *Transaction) TxHash() []byte {
	var encoded bytes.Buffer // 存储序列化后的交易数据
	var hash [32]byte        // 存储计算出的哈希结果

	// 使用 gob 对交易结构体进行序列化（编码）
	encoder := gob.NewEncoder(&encoded)
	err := encoder.Encode(tx)     // 将 tx 编码写入 encoded 缓冲区
	utils.Handle(err)             // 错误统一处理函数（例如 panic）

	// 对序列化后的字节进行 SHA-256 哈希
	hash = sha256.Sum256(encoded.Bytes())

	// 返回 hash 的切片（[:] 从数组变成切片）
	return hash[:]
}

func (tx *Transaction) SetID() {
	tx.ID = tx.TxHash()
}
```

同样的，创世块里也打包了一个初始交易，他没有输入而凭空产生输出：

```Go
// transaction.go
// tx in gensis-block, mine init-coin for the god alan
func BaseTx(ToAddress []byte) *Transaction{
	txIn := TxInput{
		TxId: []byte{},
		OutIdx: -1,
		FromAddress: []byte{},
	}
	txOut := TxOutput{
		Value: constcoe.Initcoin,
		ToAddress: ToAddress,
	}
	tx := &Transaction{
		ID: []byte("This is base tx!"),
		Inputs: []TxInput{txIn},
		Outputs: []TxOutput{txOut},
	}
	return tx
}


func (tx *Transaction) IsBase()bool{
	return len(tx.Inputs) == 1 && tx.Inputs[0].OutIdx == -1
}
```

这样一来，再回头看之前的区块的相关定义，所有的信息都对上了。

### 构建交易

万事俱备，现在我们来看如何构建一个交易并把它打包发布到区块链上。

我们先明确一个思路：

- 我们直观上的一笔交易构成：发送者、接收者、数量

- 而实际 UTXO 系统内部的数据结构构成：交易哈希（ID）、若干输入（TxInput）、若干输出（TxOutput）

- 其中的**输入是过往的一些交易的输出（这点是理解 UTXO 的关键）**，并且是没有被花掉的输出，在 UTXO 的系统中我们没有余额的概念，我们在交易中使用的是当前地址没有使用过的某些输出。

- 并且在 UTXO 中花费就意味着全部使用，例如 A 有一个 100 的输入，但是本次交易只需要 50，那么交易的输出则有两个：50 输出给目标 B，50 作为找零再输出给 A，而原本的 100 则视为用过，不在有效。

下面直接上代码，通过详细的注释，我想大家一定能弄懂：

创建交易相关：

```Go
// blockchain.go

// Key: How To Create A Traction ?
// get user's all unspent txs（交易图回溯算法）
func (bc *BlockChain) FindUnspentTransactions(from []byte) []transaction.Transaction {
	var unSpentTxs []transaction.Transaction // 用于记录当前用户的未使用交易切片
	spentTxs := make(map[string][]int)       // 用于标记交易的输出已经被使用（仅仅标记已经被使用），交易ID => {输出的某个索引}

	// range blocks in the blockchain
	for idx := len(bc.Blocks) - 1; idx >= 0; idx-- { // 从新到旧地遍历区块，避免重复访问
		block := bc.Blocks[idx]
		// range txs in the block
		for _, tx := range block.Transactions {
			txID := hex.EncodeToString(tx.ID)

		IterOutputs:
			for outIdx, out := range tx.Outputs { // 检查每一个交易的输出（目标：将没有使用过的加入切片）
				if spentTxs[txID] != nil { // 检查已经使用的输出的前提是，存在已经使用的输出，否则直接到下一个if
					for _, spentOut := range spentTxs[txID] { // 检查当前交易中的每一个已经使用过的并标记过的输出（索引）
						if spentOut == outIdx { // 恰好为当前输出
							continue IterOutputs // 则不用添加，直接检查下一个输出即可，label语法用于跳出\跳过多层循环
						}
					}
				}
				if out.ToAddressRight(from) { // 检查是否是输出到当前用户，否则和当前用户无关
					unSpentTxs = append(unSpentTxs, *tx) // 添加到当前用户的未使用交易的切片中
				}
			}
			if !tx.IsBase() {
				for _, in := range tx.Inputs { // 检查每一个交易的输入（目标：将每一个源输出标记为已经使用过）
					if in.FromAddressRight(from) { // 前提是当前用户的源输出，否则和当前用户无关
						inTxID := hex.EncodeToString(in.TxId)
						spentTxs[inTxID] = append(spentTxs[inTxID], in.OutIdx) // 将过去那个输出标记为已经使用
					}
				}
			}
		}
	}
	return unSpentTxs
}

// get user's all unspent-outputs(UTXOs)
func (bc *BlockChain) FindUTXOs(address []byte) (int, map[string]int) {
	unspentOuts := make(map[string]int)               // 当前用户所有未使用的输出（交易ID+输出索引 确定一个输出）
	unspentTxs := bc.FindUnspentTransactions(address) // 当前用户所有未使用的交易
	accumulated := 0
Work:
	for _, tx := range unspentTxs {
		txID := hex.EncodeToString(tx.ID)
		for index, out := range tx.Outputs {
			if out.ToAddressRight(address) {
				accumulated += out.Value
				unspentOuts[txID] = index
				// one transaction can only have one output referred to adderss
				// so rediect to next tx ,then check its outputs
				// use lable to cross serveral for
				continue Work
			}
		}
	}
	return accumulated, unspentOuts
}

// get user's target unspent-outputs(UTXOs) for a tx-amount
func (bc *BlockChain) FindSpendableOutputs(address []byte, amount int) (int, map[string]int) {
	unspentOuts := make(map[string]int)
	unspentTxs := bc.FindUnspentTransactions(address)
	accumulated := 0
Work:
	for _, tx := range unspentTxs {
		txID := hex.EncodeToString(tx.ID)
		for index, out := range tx.Outputs {
			if out.ToAddressRight(address) {
				accumulated += out.Value
				unspentOuts[txID] = index
				// enough
				if accumulated >= amount {
					break Work
				}
				continue Work
			}
		}
	}
	return accumulated, unspentOuts
}

// create a new tx
func (bc *BlockChain) CreateTransaction(from, to []byte, amount int) (*transaction.Transaction, bool) {
	// 直观上的一笔交易构成：发送者、接收者、数量
	// 实际UTXO系统内部的数据结构构成：交易哈希（ID）、若干输入（TxInput）、若干输出（TxOutput）

	txInputs := make([]transaction.TxInput, 0)
	txOutputs := make([]transaction.TxOutput, 0)
	// 1. build TxInputs: inputs come from outputs(UTXOs)
	total, unspentOuts := bc.FindSpendableOutputs(from, amount)
	if total < amount {
		fmt.Println("Not enough coins!")
		// return nil,false (this is better actually)
		return &transaction.Transaction{}, false
	}
	for txId, outIndex := range unspentOuts {
		id, err := hex.DecodeString(txId)
		utils.Handle(err)
		txInput := transaction.TxInput{
			TxId:        id,
			OutIdx:      outIndex,
			FromAddress: from,
		}
		txInputs = append(txInputs, txInput)
	}
	// 2. build TxOutputs: output can divde into change-back and sent-amount
	txOutputs = append(txOutputs, transaction.TxOutput{
		Value:     amount,
		ToAddress: to,
	})
	if total > amount {
		txOutputs = append(txOutputs, transaction.TxOutput{
			Value:     total - amount,
			ToAddress: from,
		})
	}
	// 3. set hash
	tx := transaction.Transaction{
		// ID: []byte{},
		ID:      nil,
		Inputs:  txInputs,
		Outputs: txOutputs,
	}
	tx.SetID()
	return &tx, true
}

```

打包发布相关：

```Go
// blockchain.go

// add new block into blockchain(package txs)
func (bc *BlockChain) AddBlock(txs []*transaction.Transaction) {
	newBlock := CreateBlock(bc.Blocks[len(bc.Blocks)-1].Hash, txs)
	bc.Blocks = append(bc.Blocks, newBlock)
}
// simulate packaging and mining
func (bc *BlockChain) Mine(txs []*transaction.Transaction) {
	bc.AddBlock(txs)
}

```

也许查询 UTXO 的地方会比较难，如果是第一次接触的话确实不好理解，不过如果真的静下心来研究的话其实并不复杂。

至此，一个简易的区块链系统就完成了，主要实现了：**基本区块哈希结构的定义**、**POW 共识机制的实现**和**UTXO 交易机制的实现**

后面鼠鼠继续学习再更新，再探再报。。。
