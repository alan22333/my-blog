---
title: 简单DEX合约案例
description: '简化的流动性合约'
tags: ['web3','uniswap']
toc: false
date: 2025-06-02 19:38:40
categories:
    - uniswap系列
---

# 简单流动性合约 (Simple Liquidity Pool) 学习文档

## 1. 引言

本文档旨在介绍一个简化的流动性合约，其设计思想来源于 UniswapV2 的 Pair 合约。通过学习这个简单的合约，你将了解去中心化交易平台中流动性池子的基本运作原理。

## 2. 核心概念

在深入代码之前，我们需要了解几个核心概念：

- **流动性提供者 (Liquidity Provider, LP)**：向流动性池子中存入两种代币的用户。
- **流动性池子 (Liquidity Pool)**：一个存储了两种或多种代币的智能合约，用于支持交易。
- **流动性代币 (LP Tokens)**：当流动性提供者存入代币时，合约会铸造代表其在池子中份额的 LP 代币给他们。
- **储备量 (Reserves)**：池子中每种代币的存量。
- **恒定乘积公式 ($x \times y = k$)**: 这是 UniswapV2 等 AMM (Automated Market Maker) 的核心。其中，$x$ 和 $y$ 分别代表池子中两种代币的储备量，$k$ 是一个（在没有交易费用时）保持不变的常数。这个公式决定了交易的价格。
- **滑点 (Slippage)**：由于交易会改变池子中的储备量，导致执行价格与预期价格之间的差异。较大的交易通常会有更高的滑点。

## 3. 合约结构概览

我们实现的 `SimpleLiquidityPool` 合约主要包含以下几个部分：

- **状态变量**: 存储合约的关键信息，如代币地址、储备量、LP 代币总供应量等。
- **事件**: 用于记录合约中发生的关键操作，方便在链下追踪。
- **构造函数**: 在合约部署时初始化。
- **内部函数**: 执行一些底层操作，例如更新储备量、铸造和销毁 LP 代币、安全地转移代币。
- **外部函数**: 允许用户与合约交互的主要接口，例如提供流动性 (`mint`)、移除流动性 (`burn`)、进行代币交换 (`swap`)。
- **修饰器 (Modifier)**: 用于修改函数的行为，例如这里的 `lock` 用于简单的重入保护。

## 4. 代码详解
### 4.1. 导入

```solidity
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";
```

- `IERC20.sol`: 导入 ERC-20 代币的标准接口，使得我们的合约可以与任何符合 ERC-20 标准的代币进行交互。
- `SafeMath.sol`: 导入 SafeMath 库，用于执行安全的算术运算，防止溢出。

### 4.2. 合约声明和状态变量

```solidity
contract SimpleLiquidityPool {
    using SafeMath for uint256;

    IERC20 public token0;
    IERC20 public token1;
    uint256 public reserve0;
    uint256 public reserve1;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;

    uint256 constant MINIMUM_LIQUIDITY = 10**3;
```

- `token0` 和 `token1`: 存储这个流动性池子支持的两种 ERC-20 代币的合约地址。
- `reserve0` 和 `reserve1`: 记录当前池子中 `token0` 和 `token1` 的储备数量。
- `totalSupply`: 记录流动性代币（LP tokens）的总发行量。
- `balanceOf`: 一个映射，记录每个地址拥有的 LP 代币数量。
- `MINIMUM_LIQUIDITY`: 在首次提供流动性时，会有一小部分 LP 代币发送到零地址并永久锁定，这是为了防止 `totalSupply` 在初始状态为零时可能导致的一些除零错误。

### 4.3. 事件

```solidity
    event Mint(address indexed sender, uint256 amount0, uint256 amount1);
    event Burn(address indexed sender, uint256 amount0, uint256 amount1, address indexed to);
    event Swap(
        address indexed sender,
        uint256 amount0In,
        uint256 amount1In,
        uint256 amount0Out,
        uint256 amount1Out,
        address indexed to
    );
    event Sync(uint256 reserve0, uint256 reserve1);
```

这些事件在合约执行关键操作时发出，例如提供流动性 (`Mint`)、移除流动性 (`Burn`)、代币交换 (`Swap`) 和同步储备量 (`Sync`)。这对于在区块链外部追踪合约的状态非常有用。

### 4.4. 构造函数

```solidity
    constructor(IERC20 _token0, IERC20 _token1) {
        token0 = _token0;
        token1 = _token1;
    }
```

构造函数在合约部署时被调用，用于初始化 `token0` 和 `token1` 的地址。

### 4.5. 内部函数

- `_update(uint256 balance0, uint256 balance1)`: 更新 `reserve0` 和 `reserve1` 的值，并触发 `Sync` 事件。
- `_mint(address to, uint256 value)`: 增加 `totalSupply` 并将 `value` 数量的 LP 代币分配给地址 `to`。
- `_burn(address from, uint256 value) returns (uint256 amount0, uint256 amount1)`: 销毁地址 `from` 的 `value` 数量的 LP 代币，并根据其在总供应量中的比例计算出应返还的 `token0` 和 `token1` 的数量。
- `_safeTransfer(IERC20 token, address to, uint256 value)`: 安全地转移 ERC-20 代币，如果转移失败会触发 `require` 错误。

### 4.6. 外部函数

- **`mint(address to)` (提供流动性)**:

  - 计算用户存入的两种代币数量。
  - 如果是首次提供流动性（`totalSupply` 为 0），则根据几何平均值计算铸造的 LP 代币数量，并永久锁定 `MINIMUM_LIQUIDITY`。
  - 否则，根据用户提供的代币相对于现有储备的比例来计算铸造的 LP 代币数量。
  - 将计算出的 LP 代币铸造给流动性提供者。
  - 更新储备量。
  - 触发 `Mint` 事件。

- **`burn(address to) returns (uint256 amount0, uint256 amount1)` (移除流动性)**:

  - 获取调用者拥有的 LP 代币数量。
  - 根据其持有的 LP 代币占总供应量的比例，计算出应返还的两种代币数量。
  - 销毁调用者的 LP 代币。
  - 将计算出的两种代币返还给调用者。
  - 更新储备量。
  - 触发 `Burn` 事件。

- **`swap(uint256 amount0Out, uint256 amount1Out, address to)` (代币交换)**:

  - 要求用户指定想要输出的一种代币的数量 (`amount0Out` 或 `amount1Out`)，另一个输出量必须为 0。
  - 根据恒定乘积公式 ($x \times y = k$) 计算用户需要输入的另一种代币的数量。例如，如果用户想要输出 `amount0Out` 的 `token0`，则需要计算出需要输入的 `token1` 的数量，以保持（近似）$reserve0 \times reserve1$ 的恒定。
  - 将用户输入的代币转移到合约。
  - 更新储备量。
  - 将输出的代币转移给接收者。
  - 触发 `Swap` 事件。

- **`skim(address to)`**: 允许在合约实际持有的代币数量与记录的储备量不一致时，将多余的代币发送到指定地址。
- **`sync()`**: 允许外部强制更新合约的储备量，使其与合约实际持有的代币数量一致。

### 4.7. 修饰器

- **`lock()`**: 一个简单的修饰器，用于防止潜在的重入攻击。

## 5. 如何使用

1.  **部署 ERC-20 代币**: 首先需要部署你想要在这个流动性池中交易的两种 ERC-20 代币合约。
2.  **部署流动性池合约**: 部署 `SimpleLiquidityPool` 合约，并将上述两个 ERC-20 代币合约的地址作为构造函数的参数传入。
3.  **批准代币转移**: 用户需要调用他们持有的 ERC-20 代币合约的 `approve` 函数，授权给 `SimpleLiquidityPool` 合约可以从他们的账户转移一定数量的代币。
4.  **提供流动性**: 调用 `mint` 函数，将一定数量的两种代币存入池子。你会收到相应的 LP 代币作为回报。
5.  **移除流动性**: 调用 `burn` 函数，发送你的 LP 代币到合约，合约会销毁这些 LP 代币，并返还相应的两种代币给你。
6.  **进行交易**: 调用 `swap` 函数，指定你想要输出的代币数量，合约会计算并发送给你相应的另一种代币。

## 6. 完整代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol"; // 导入 ERC-20 接口
import "@openzeppelin/contracts/utils/math/SafeMath.sol"; // 导入 SafeMath 库以进行安全的算术运算

contract SimpleLiquidityPool {
    using SafeMath for uint256; // 使用 SafeMath 库处理 uint256 类型的运算

    IERC20 public token0; // 第一个 ERC-20 代币的接口
    IERC20 public token1; // 第二个 ERC-20 代币的接口
    uint256 public reserve0; // 池子中 token0 的储备量
    uint256 public reserve1; // 池子中 token1 的储备量
    uint256 public totalSupply; // 流动性代币（LP tokens）的总发行量
    mapping(address => uint256) public balanceOf; // 记录每个地址拥有的 LP 代币数量

    uint256 constant MINIMUM_LIQUIDITY = 10**3; // 首次提供流动性时，发送给零地址的最小流动性数量，用于防止 totalSupply 为零

    // 定义事件，方便在链下监听合约状态变化
    event Mint(address indexed sender, uint256 amount0, uint256 amount1); // 当提供流动性时触发
    event Burn(address indexed sender, uint256 amount0, uint256 amount1, address indexed to); // 当移除流动性时触发
    event Swap(
        address indexed sender,
        uint256 amount0In,
        uint256 amount1In,
        uint256 amount0Out,
        uint256 amount1Out,
        address indexed to
    ); // 当进行代币交换时触发
    event Sync(uint256 reserve0, uint256 reserve1); // 当储备量更新时触发

    // 构造函数，在合约部署时被调用
    constructor(IERC20 _token0, IERC20 _token1) {
        token0 = _token0; // 设置 token0 的合约地址
        token1 = _token1; // 设置 token1 的合约地址
    }

    // 获取当前的储备量
    function getReserves() public view returns (uint256 _reserve0, uint256 _reserve1) {
        return (reserve0, reserve1);
    }

    // 内部函数：更新储备量并触发 Sync 事件
    function _update(uint256 balance0, uint256 balance1) internal {
        reserve0 = balance0;
        reserve1 = balance1;
        emit Sync(reserve0, reserve1);
    }

    // 内部函数：铸造流动性代币
    function _mint(address to, uint256 value) internal {
        totalSupply = totalSupply.add(value);
        balanceOf[to] = balanceOf[to].add(value);
    }

    // 内部函数：销毁流动性代币，并计算应返还的两种代币数量
    function _burn(address from, uint256 value) internal returns (uint256 amount0, uint256 amount1) {
        require(balanceOf[from] >= value, "INSUFFICIENT_LIQUIDITY");
        balanceOf[from] = balanceOf[from].sub(value);
        totalSupply = totalSupply.sub(value, "INSUFFICIENT_TOTAL_SUPPLY");
        amount0 = value.mul(reserve0).div(totalSupply); // 根据持有的 LP 比例计算应得的 token0
        amount1 = value.mul(reserve1).div(totalSupply); // 根据持有的 LP 比例计算应得的 token1
        _safeTransfer(token0, from, amount0); // 安全地转移 token0 给用户
        _safeTransfer(token1, from, amount1); // 安全地转移 token1 给用户
    }

    // 内部函数：安全地转移 ERC-20 代币
    function _safeTransfer(IERC20 token, address to, uint256 value) internal {
        bool success = token.transfer(to, value);
        require(success, "TRANSFER_FAILED");
    }

    // 外部函数：提供流动性
    function mint(address to) external lock {
        uint256 balance0 = token0.balanceOf(address(this)); // 获取合约中当前的 token0 余额
        uint256 balance1 = token1.balanceOf(address(this)); // 获取合约中当前的 token1 余额
        uint256 amount0 = balance0.sub(reserve0); // 用户存入的 token0 数量
        uint256 amount1 = balance1.sub(reserve1); // 用户存入的 token1 数量

        // 在首次提供流动性时，需要特殊处理来初始化 totalSupply
        uint256 liquidity;
        if (totalSupply == 0) {
            liquidity = SafeMath.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
            _mint(address(0), MINIMUM_LIQUIDITY); // 永久锁定一部分 LP 代币
        } else {
            // 根据已有的储备量和提供的代币比例计算应铸造的 LP 代币数量
            liquidity = SafeMath.min(amount0.mul(totalSupply) / reserve0, amount1.mul(totalSupply) / reserve1);
        }
        require(liquidity > 0, "INSUFFICIENT_LIQUIDITY_MINTED");
        _mint(to, liquidity); // 将计算出的 LP 代币铸造给提供者
        _update(balance0, balance1); // 更新储备量
        emit Mint(to, amount0, amount1); // 触发 Mint 事件
    }

    // 外部函数：移除流动性
    function burn(address to) external lock returns (uint256 amount0, uint256 amount1) {
        uint256 liquidity = balanceOf[msg.sender]; // 获取调用者拥有的 LP 代币数量
        require(liquidity > 0, "BURN_ZERO_LIQUIDITY");

        amount0 = liquidity.mul(reserve0).div(totalSupply); // 计算应返还的 token0 数量
        amount1 = liquidity.mul(reserve1).div(totalSupply); // 计算应返还的 token1 数量

        _burn(msg.sender, liquidity); // 销毁调用者的 LP 代币
        _safeTransfer(token0, to, amount0); // 将 token0 返还给调用者
        _safeTransfer(token1, to, amount1); // 将 token1 返还给调用者
        emit Burn(msg.sender, amount0, amount1, to); // 触发 Burn 事件
        _update(token0.balanceOf(address(this)), token1.balanceOf(address(this))); // 更新储备量
        return (amount0, amount1); // 返回返还的代币数量
    }

    // 外部函数：进行代币交换
    function swap(uint256 amount0Out, uint256 amount1Out, address to) external lock {
        require(amount0Out > 0 ^ amount1Out > 0, "INVALID_OUTPUT_AMOUNT"); // 确保只输出一种代币
        (uint112 _reserve0, uint112 _reserve1,) = (uint112(reserve0), uint112(reserve1), uint112(block.timestamp)); // gas 优化，读取储备量

        require(amount0Out < _reserve0 && amount1Out < _reserve1, "INSUFFICIENT_LIQUIDITY"); // 确保输出量小于储备量

        uint256 amount0In;
        uint256 amount1In;

        // 根据输出量计算输入量（遵循 x * y = k 原则）
        if (amount0Out > 0) { // 用户想要输出 token0
            uint256 numerator = _reserve0.mul(amount1Out);
            uint256 denominator = _reserve1.sub(amount1Out);
            amount0In = (numerator.div(denominator)).add(1); // +1 避免精度损失
            require(amount0In > 0 && token0.balanceOf(msg.sender) >= amount0In, "INSUFFICIENT_INPUT_AMOUNT");
            _safeTransfer(token0, address(this), amount0In); // 将输入的 token0 转入合约
        } else { // 用户想要输出 token1
            uint256 numerator = _reserve1.mul(amount0Out);
            uint256 denominator = _reserve0.sub(amount0Out);
            amount1In = (numerator.div(denominator)).add(1); // +1 避免精度损失
            require(amount1In > 0 && token1.balanceOf(msg.sender) >= amount1In, "INSUFFICIENT_INPUT_AMOUNT");
            _safeTransfer(token1, address(this), amount1In); // 将输入的 token1 转入合约
        }
        uint256 balance0 = token0.balanceOf(address(this));
        uint256 balance1 = token1.balanceOf(address(this));

        _update(balance0, balance1); // 更新储备量
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to); // 触发 Swap 事件
        if (amount0Out > 0) _safeTransfer(token0, to, amount0Out); // 将输出的 token0 转给接收者
        if (amount1Out > 0) _safeTransfer(token1, to, amount1Out); // 将输出的 token1 转给接收者
    }

    // 允许外部合约强制同步余额
    function skim(address to) external lock {
        uint256 balance0 = token0.balanceOf(address(this));
        uint256 balance1 = token1.balanceOf(address(this));
        _safeTransfer(token0, to, balance0.sub(reserve0, "SKIM_FAILED")); // 将多余的 token0 发送到指定地址
        _safeTransfer(token1, to, balance1.sub(reserve1, "SKIM_FAILED")); // 将多余的 token1 发送到指定地址
    }

    // 允许外部合约强制更新储备量
    function sync() external lock {
        _update(token0.balanceOf(address(this)), token1.balanceOf(address(this)));
    }

    // 一个简单的修饰器，用于防止重入攻击
    modifier lock() {
        _;
    }
}
```