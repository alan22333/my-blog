---
title: Solidity中的自定义错误
description: 'Solidity 0.8.4 引入了自定义错误(Custom Errors)功能，这是一种比传统 require 语句更高效的错误处理方式，可以显著节省 Gas 消耗'
date: 2025-05-03 21:26:17
tags: ['Solidity']
categories: Solidity学习
toc: true
---


Solidity 0.8.4 引入了自定义错误(Custom Errors)功能，这是一种比传统 `require` 语句更高效的错误处理方式，可以显著节省 Gas 消耗。

## 为什么自定义错误能节省 Gas

1. **更小的字节码**：自定义错误不需要存储错误字符串

2. **更低的运行时成本**：回滚时只需传递 4 字节的选择器，而不是完整的字符串

3. **更少的存储开销**：字符串错误信息需要存储在合约字节码中

## 基本语法

```Solidity
// 定义自定义错误
error InsufficientBalance(uint256 available, uint256 required);

contract MyContract {
    function withdraw(uint256 amount) public {
        uint256 balance = address(this).balance;
        if (balance < amount) {
            revert InsufficientBalance(balance, amount);
        }
        // 其他逻辑...
    }
}
```


## 最佳实践

### 1. 为常见错误条件定义明确的错误

```Solidity
error Unauthorized();
error InvalidAddress();
error ValueTooLow(uint256 min, uint256 actual);
error DeadlineExpired(uint256 deadline);
```


### 2. 在复杂合约中组织错误

```Solidity
library Errors {
    error Unauthorized();
    error InsufficientBalance(uint256 available, uint256 required);
}

contract MyContract {
    using Errors for *;
    
    function withdraw(uint256 amount) public {
        if (msg.sender != owner) revert Errors.Unauthorized();
        // ...
    }
}
```


### 3. 提供有用的上下文信息

```Solidity
error TransferFailed(address from, address to, uint256 amount);

function transfer(address to, uint256 amount) public {
    bool success = _transfer(msg.sender, to, amount);
    if (!success) revert TransferFailed(msg.sender, to, amount);
}
```


### 4. 与 require 对比

```Solidity
// 传统方式 - 消耗更多 Gas
require(balance >= amount, "Insufficient balance");

// 自定义错误方式 - 更高效
if (balance < amount) revert InsufficientBalance(balance, amount);
```


### 5. 在接口中定义错误

```Solidity
interface IERC20 {
    error InsufficientBalance();
    error InsufficientAllowance();
    
    function transfer(address to, uint256 amount) external;
}
```


## Gas 节省示例

假设一个简单的转账合约：

```Solidity
// 使用 require
function transferWithRequire(address to, uint256 amount) public {
    require(balances[msg.sender] >= amount, "Insufficient balance");
    balances[msg.sender] -= amount;
    balances[to] += amount;
}

// 使用自定义错误
error InsufficientBalance(uint256 available, uint256 required);

function transferWithError(address to, uint256 amount) public {
    uint256 available = balances[msg.sender];
    if (available < amount) revert InsufficientBalance(available, amount);
    balances[msg.sender] = available - amount;
    balances[to] += amount;
}
```


**Gas 消耗比较**：
• `require` 版本：约 22,000 Gas（成功时），失败时更高
• 自定义错误版本：约 21,500 Gas（成功时），失败时显著更低

失败时的 Gas 节省更为明显，因为不需要存储和传递错误字符串。

## 结论

自定义错误是 Solidity 中优化 Gas 消耗的重要工具，特别是在频繁可能失败的函数中。通过遵循这些最佳实践，你可以编写出更高效、更清晰的智能合约代码。



