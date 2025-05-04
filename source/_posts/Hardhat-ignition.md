---
title: Hardhat-ignition
description: 'Hardhat Ignition 是 Hardhat 的一个插件，用于管理和自动化智能合约的部署流程'
tags: ['hardhat','web3']
toc: false
date: 2025-05-04 20:10:35
categories:
---
**Hardhat Ignition** 是 Hardhat 的一个插件，用于管理和自动化智能合约的部署流程。它特别适合复杂的部署场景，例如需要部署多个合约、处理合约依赖关系或按特定顺序部署的场景。以下是 Hardhat Ignition 的常见用法和详细介绍：

---

## **1. 安装 Hardhat Ignition**

首先，确保你已经安装了 Hardhat 和 Hardhat Ignition：

```Shell
npm install --save-dev @nomicfoundation/hardhat-ignition
```


然后在 `hardhat.config.js` 中引入插件：

```JavaScript
require("@nomicfoundation/hardhat-ignition");
```


---

## **2. 定义部署模块**

Hardhat Ignition 的核心概念是 **模块（Module）**。一个模块是一个独立的部署单元，可以包含多个合约部署和配置。

### **基本模块定义**

```JavaScript
const { buildModule } = require("@nomicfoundation/hardhat-ignition/modules");

module.exports = buildModule("MyModule", (m) => {
  const myContract = m.contract("MyContract");

  return { myContract };
});
```


- `buildModule`：定义一个模块。

- `"MyModule"`：模块的名称。

- `m.contract("MyContract")`：部署名为 `MyContract` 的合约。

- `return { myContract }`：返回部署后的合约实例。

---

### **带参数的模块**

你可以通过 `m.getParameter` 动态设置部署参数：

```JavaScript
module.exports = buildModule("MyModule", (m) => {
  const initialValue = m.getParameter("initialValue", 100); // 默认值为 100
  const myContract = m.contract("MyContract", [initialValue]);

  return { myContract };
});
```


---

### **部署多个合约**

在一个模块中可以部署多个合约，并处理它们之间的依赖关系：

```JavaScript
module.exports = buildModule("MyModule", (m) => {
  const contractA = m.contract("ContractA");
  const contractB = m.contract("ContractB", [contractA.address]);

  return { contractA, contractB };
});
```


- `contractB` 依赖于 `contractA` 的地址，Hardhat Ignition 会自动按正确顺序部署。

---

### **调用合约函数**

在部署过程中，你可以调用合约的函数：

```JavaScript
module.exports = buildModule("MyModule", (m) => {
  const myContract = m.contract("MyContract");
  m.call(myContract, "initialize", [42]); // 调用 initialize 函数

  return { myContract };
});
```


---

### **使用库合约**

如果合约依赖于库合约，可以先部署库合约，然后将其链接到主合约：

```JavaScript
module.exports = buildModule("MyModule", (m) => {
  const myLibrary = m.library("MyLibrary");
  const myContract = m.contract("MyContract", [], {
    libraries: {
      MyLibrary: myLibrary.address,
    },
  });

  return { myContract };
});
```


---

## **3. 运行部署**

使用 `ignition.deploy` 运行部署模块：

```JavaScript
const { ignition } = require("hardhat");

async function main() {
  const { myContract } = await ignition.deploy("./path/to/MyModule");
  console.log("MyContract deployed to:", myContract.address);
}

main();
```


运行部署脚本：

```Shell
npx hardhat run scripts/deploy.js --network localhost
```


---

## **4. 高级功能**

### **分阶段部署**

Hardhat Ignition 支持分阶段部署，例如先部署合约，然后初始化：

```JavaScript
module.exports = buildModule("MyModule", (m) => {
  const myContract = m.contract("MyContract");

  m.stage("Initialize", async () => {
    await myContract.initialize(42);
  });

  return { myContract };
});
```


---

### **条件部署**

你可以根据条件决定是否部署某个合约：

```JavaScript
module.exports = buildModule("MyModule", (m) => {
  const shouldDeploy = m.getParameter("shouldDeploy", true);

  if (shouldDeploy) {
    const myContract = m.contract("MyContract");
    return { myContract };
  }

  return {};
});
```


---

### **使用现有合约**

如果某个合约已经部署，可以直接使用其地址，而不重新部署：

```JavaScript
module.exports = buildModule("MyModule", (m) => {
  const existingContract = m.getParameter("existingContract", "0x...");
  const myContract = m.contractAt("MyContract", existingContract);

  return { myContract };
});
```


---

### **多网络支持**

Hardhat Ignition 支持多网络配置。你可以在 `hardhat.config.js` 中定义不同的网络，并在部署时指定网络：

```JavaScript
module.exports = {
  networks: {
    localhost: {
      url: "http://localhost:8545",
    },
    mainnet: {
      url: "https://mainnet.infura.io/v3/YOUR_INFURA_PROJECT_ID",
      accounts: ["0x..."], // 私钥
    },
  },
};
```


运行部署时指定网络：

```Shell
npx hardhat run scripts/deploy.js --network mainnet
```


---

## **5. 使用场景**

### **复杂部署流程**

- 部署多个合约并处理依赖关系。

- 按特定顺序初始化合约。

### **多环境部署**

- 在测试网和主网之间切换部署配置。

- 动态调整部署参数（如初始值、管理员地址等）。

### **自动化测试**

- 在测试中自动部署合约并初始化状态。

### **升级合约**

- 部署新版本的合约并迁移数据。

---




