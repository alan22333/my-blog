---
title: Hardhat上手指南
description: 'Hardhat 是一个用于开发、测试和部署智能合约的开发环境。它提供了强大的工具和插件，帮助开发者更高效地构建以太坊和其他区块链应用'
tags: ['hardhat', 'web3']
toc: false
date: 2025-05-04 20:11:14
categories:
---

Hardhat 是一个用于开发、测试和部署智能合约的开发环境。它提供了强大的工具和插件，帮助开发者更高效地构建以太坊和其他区块链应用。以下是 Hardhat 的基本使用指南：

### 1. 安装 Hardhat

首先，确保你已经安装了 Node.js 和 npm。然后，可以通过以下命令全局安装 Hardhat：

```Shell
npm install -g hardhat
```


或者在项目目录中局部安装：

```Shell
npm install --save-dev hardhat
```


### 2. 初始化项目

在项目目录中运行以下命令来初始化一个新的 Hardhat 项目：

```Shell
npx hardhat
```


这将启动一个交互式命令行界面，你可以选择创建一个新的 Hardhat 项目或使用示例项目。

### 3. 项目结构

初始化后，项目目录结构通常如下：

```Plain Text
my-project/
├── contracts/           # 存放智能合约
├── scripts/             # 存放部署脚本
├── test/                # 存放测试脚本
├── hardhat.config.js    # Hardhat 配置文件
└── package.json         # Node.js 项目配置文件
```


### 4. 编写智能合约

在 `contracts/` 目录中编写你的智能合约。例如，创建一个简单的 `Greeter.sol` 合约：

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Greeter {
    string private greeting;

    constructor(string memory _greeting) {
        greeting = _greeting;
    }

    function greet() public view returns (string memory) {
        return greeting;
    }

    function setGreeting(string memory _greeting) public {
        greeting = _greeting;
    }
}
```


### 5. 编译合约

使用以下命令编译合约：

```Shell
npx hardhat compile
```


编译后的文件将存储在 `artifacts/` 目录中。

### 6. 编写测试

在 `test/` 目录中编写测试脚本。例如，创建一个 `Greeter.test.js` 文件：

```JavaScript
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Greeter", function () {
  it("Should return the new greeting once it's changed", async function () {
    const Greeter = await ethers.getContractFactory("Greeter");
    const greeter = await Greeter.deploy("Hello, world!");
    await greeter.deployed();

    expect(await greeter.greet()).to.equal("Hello, world!");

    const setGreetingTx = await greeter.setGreeting("Hola, mundo!");

    // wait until the transaction is mined
    await setGreetingTx.wait();

    expect(await greeter.greet()).to.equal("Hola, mundo!");
  });
});
```


### 7. 运行测试

使用以下命令运行测试：

```Shell
npx hardhat test
```


### 8. 部署合约

在 `scripts/` 目录中编写部署脚本。例如，创建一个 `deploy.js` 文件：

```JavaScript
async function main() {
  const Greeter = await ethers.getContractFactory("Greeter");
  const greeter = await Greeter.deploy("Hello, Hardhat!");

  await greeter.deployed();

  console.log("Greeter deployed to:", greeter.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```


然后使用以下命令部署合约：

```Shell
npx hardhat run scripts/deploy.js
```


### 9. 配置网络

在 `hardhat.config.js` 中配置网络。例如，配置本地网络和以太坊测试网络：

```JavaScript
require("@nomiclabs/hardhat-waffle");

module.exports = {
  solidity: "0.8.4",
  networks: {
    localhost: {
      url: "http://127.0.0.1:8545"
    },
    ropsten: {
      url: `https://ropsten.infura.io/v3/YOUR_INFURA_PROJECT_ID`,
      accounts: [`0x${YOUR_PRIVATE_KEY}`]
    }
  }
};
```


### 10. 使用插件

Hardhat 支持许多插件，例如：

- `@nomiclabs/hardhat-ethers`：用于与以太坊交互。

- `@nomiclabs/hardhat-waffle`：用于测试。

- `hardhat-gas-reporter`：用于报告 Gas 使用情况。

可以通过 npm 安装这些插件，并在 `hardhat.config.js` 中配置。

### 11. 其他命令

- `npx hardhat clean`：清理编译缓存。

- `npx hardhat node`：启动本地以太坊节点。

- `npx hardhat console`：启动 Hardhat 控制台。





Hardhat 的强大之处在于其丰富的插件生态系统，这些插件可以帮助开发者更高效地进行智能合约开发、测试和部署。以下是一些常用的 Hardhat 插件及其功能：

---

### 1. **@nomiclabs/hardhat-ethers**

- **功能**: 集成 `ethers.js` 库，用于与以太坊网络交互。

- **用途**: 部署合约、发送交易、调用合约方法等。

- **安装**:

  ```Shell
npm install @nomiclabs/hardhat-ethers ethers
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("@nomiclabs/hardhat-ethers");
```


---

### 2. **@nomiclabs/hardhat-waffle**

- **功能**: 集成 `Waffle` 测试框架，用于编写和运行智能合约测试。

- **用途**: 提供 Chai 匹配器和工具，方便测试智能合约。

- **安装**:

  ```Shell
npm install @nomiclabs/hardhat-waffle ethereum-waffle chai
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("@nomiclabs/hardhat-waffle");
```


---

### 3. **hardhat-deploy**

- **功能**: 提供更强大的合约部署功能，支持命名部署、依赖管理和部署脚本的组织。

- **用途**: 管理复杂的部署流程，支持多网络部署。

- **安装**:

  ```Shell
npm install hardhat-deploy
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("hardhat-deploy");
```


---

### 4. **hardhat-gas-reporter**

- **功能**: 在运行测试时报告 Gas 使用情况。

- **用途**: 优化合约的 Gas 消耗。

- **安装**:

  ```Shell
npm install hardhat-gas-reporter
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("hardhat-gas-reporter");
module.exports = {
  gasReporter: {
    currency: 'USD', // 设置货币单位
    gasPrice: 21,    // 设置 Gas 价格
  },
};
```


---

### 5. **solidity-coverage**

- **功能**: 生成智能合约的代码覆盖率报告。

- **用途**: 确保测试覆盖了合约的所有代码路径。

- **安装**:

  ```Shell
npm install solidity-coverage
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("solidity-coverage");
```


- **使用**:
运行以下命令生成覆盖率报告：

  ```Shell
npx hardhat coverage
```


---

### 6. **@nomicfoundation/hardhat-toolbox**

- **功能**: 集成了多个常用插件（如 `hardhat-ethers`、`hardhat-waffle`、`hardhat-gas-reporter` 等）。

- **用途**: 快速启动项目，避免手动安装多个插件。

- **安装**:

  ```Shell
npm install @nomicfoundation/hardhat-toolbox
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("@nomicfoundation/hardhat-toolbox");
```


---

### 7. **hardhat-etherscan**

- **功能**: 用于验证合约源码并发布到 Etherscan。

- **用途**: 在 Etherscan 上公开合约源码，方便用户查看和验证。

- **安装**:

  ```Shell
npm install @nomiclabs/hardhat-etherscan
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("@nomiclabs/hardhat-etherscan");
module.exports = {
  etherscan: {
    apiKey: "YOUR_ETHERSCAN_API_KEY", // 替换为你的 Etherscan API Key
  },
};
```


- **使用**:
运行以下命令验证合约：

  ```Shell
npx hardhat verify --network <network_name> <contract_address> <constructor_args>
```


---

### 8. **hardhat-abi-exporter**

- **功能**: 自动导出合约的 ABI 到指定目录。

- **用途**: 方便前端或其他应用使用合约的 ABI。

- **安装**:

  ```Shell
npm install hardhat-abi-exporter
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("hardhat-abi-exporter");
module.exports = {
  abiExporter: {
    path: './abi', // 导出目录
    clear: true,   // 每次编译前清空目录
  },
};
```


---

### 9. **hardhat-contract-sizer**

- **功能**: 分析合约的字节码大小。

- **用途**: 确保合约不超过以太坊的字节码大小限制。

- **安装**:

  ```Shell
npm install hardhat-contract-sizer
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("hardhat-contract-sizer");
```


---

### 10. **hardhat-ignition**

- **功能**: 提供更高级的部署管理功能，支持模块化部署。

- **用途**: 管理复杂的部署流程，支持依赖关系和条件部署。

- **安装**:

  ```Shell
npm install hardhat-ignition
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("hardhat-ignition");
```


---

### 11. **hardhat-typechain**

- **功能**: 自动生成 TypeScript 类型定义文件。

- **用途**: 在 TypeScript 项目中提供类型安全的合约交互。

- **安装**:

  ```Shell
npm install @typechain/hardhat typechain @typechain/ethers-v5
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("@typechain/hardhat");
module.exports = {
  typechain: {
    outDir: 'typechain', // 类型定义文件输出目录
    target: 'ethers-v5', // 目标框架
  },
};
```


---

### 12. **hardhat-storage-layout**

- **功能**: 导出合约的存储布局信息。

- **用途**: 分析合约的存储结构，方便升级和调试。

- **安装**:

  ```Shell
npm install hardhat-storage-layout
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("hardhat-storage-layout");
```


---

### 13. **hardhat-tracer**

- **功能**: 提供更详细的交易调试信息。

- **用途**: 调试复杂的交易和合约调用。

- **安装**:

  ```Shell
npm install hardhat-tracer
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("hardhat-tracer");
```


---

### 14. **hardhat-interface-generator**

- **功能**: 自动生成合约的接口文件。

- **用途**: 方便其他合约或应用与当前合约交互。

- **安装**:

  ```Shell
npm install hardhat-interface-generator
```


- **配置**:
在 `hardhat.config.js` 中添加：

  ```JavaScript
require("hardhat-interface-generator");
```


---

