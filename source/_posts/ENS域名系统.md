---
title: ENS域名系统
description: 'ENS（Ethereum Name Service，以太坊域名服务）是以太坊上的去中心化域名解析系统'
tags: ['web3']
toc: false
date: 2025-05-04 20:05:20
categories:
---

### **ENS 域名简介**

ENS（Ethereum Name Service，以太坊域名服务）是以太坊上的去中心化域名解析系统，类似于传统互联网的DNS（域名系统），但它专门为区块链地址服务。ENS允许用户将复杂的以太坊钱包地址（如 `0x1234...abcd`）映射到易于记忆的**域名**（如 `alice.eth`）。

ENS不仅支持地址映射，还可以存储IPFS内容哈希、文本记录、加密钱包等信息，从而让区块链更具可读性和可用性。

---

### **ENS 的核心功能**

ENS 提供的主要功能包括：

1. **钱包地址解析**：将 `alice.eth` 解析为 `0x1234...abcd`。

2. **反向解析**：从钱包地址解析出 ENS 域名。

3. **文本记录存储**：可以存储社交媒体链接、IPFS哈希、电子邮件等信息。

4. **子域名管理**：可以创建如 `mail.alice.eth` 这样的子域名。

---

### **ENS 工作流程**

ENS 依赖智能合约运行在以太坊网络上，主要组件包括：

5. **Registry（注册表合约）**：存储域名的所有者、解析器、到期时间等信息。

6. **Resolver（解析器合约）**：处理域名的解析逻辑，将ENS名称转换为地址或其他记录。

7. **Registrar（注册管理器）**：处理ENS域名的购买、转移和续费。

---

### **如何在DApp中集成ENS域名解析**

为了在你的Web3Mail DApp中支持ENS域名，可以使用 `ethers.js` 库进行交互，ENS解析流程如下：

#### **1. 检查用户是否拥有ENS域名**

在用户登录时，我们可以使用 `ethers.js` 查询用户的ENS名称，如果没有，则显示其钱包地址。

```JavaScript
import { ethers } from "ethers";

// 连接以太坊钱包
const provider = new ethers.BrowserProvider(window.ethereum);

async function getENSName(address) {
  try {
    const ensName = await provider.lookupAddress(address);
    if (ensName) {
      console.log(`ENS 名称: ${ensName}`);
      return ensName;
    } else {
      console.log(`未找到 ENS 名称，地址: ${address}`);
      return address;
    }
  } catch (error) {
    console.error("ENS 查询失败:", error);
    return address;
  }
}

// 获取当前用户地址
async function fetchUserENS() {
  const signer = await provider.getSigner();
  const address = await signer.getAddress();
  return getENSName(address);
}

fetchUserENS().then(console.log);
```


---

#### **2. 发送邮件时解析 ENS 域名**

当用户输入接收者 ENS 域名（如 `bob.eth`）时，需解析为以太坊地址，并存储到智能合约。

```JavaScript
async function resolveENS(ensName) {
  try {
    const address = await provider.resolveName(ensName);
    if (address) {
      console.log(`ENS 解析结果: ${ensName} -> ${address}`);
      return address;
    } else {
      throw new Error("ENS 名称解析失败");
    }
  } catch (error) {
    console.error("解析 ENS 时出错:", error);
    throw error;
  }
}

// 示例: 发送邮件前解析 ENS 名称
async function sendMail(toENS, ipfsHash) {
  try {
    const recipientAddress = await resolveENS(toENS);
    // 这里调用智能合约，将 recipientAddress 和 ipfsHash 存储
  } catch (error) {
    console.error("发送邮件失败:", error);
  }
}
```


---

#### **3. 反向解析（地址到 ENS 名称）**

在收件箱列表中，你可以使用反向解析将以太坊地址转换为ENS名称。

```JavaScript
async function reverseLookup(address) {
  const ensName = await provider.lookupAddress(address);
  return ensName ? ensName : address;
}
```


---

### **如何在 Solidity 智能合约中集成 ENS**

ENS 提供了官方的智能合约接口，可以在合约中直接查询 ENS 名称：

#### **ENS 合约地址（Ethereum Mainnet）**

ENS 的注册表合约在以太坊主网上的地址为：
`0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e`

#### **在 Solidity 中解析 ENS**

```Solidity
pragma solidity ^0.8.0;

interface IENSResolver {
    function addr(bytes32 node) external view returns (address);
}

contract ENSHelper {
    IENSResolver ensResolver = IENSResolver(0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e);

    function resolveENS(bytes32 node) public view returns (address) {
        return ensResolver.addr(node);
    }
}
```


在合约中使用ENS解析时，ENS名称需要转换为 `keccak256` 哈希，例如：

```JavaScript
const ensNameHash = ethers.keccak256(ethers.toUtf8Bytes("alice.eth"));
```


---

### **部署 ENS 域名**

如果你想为你的 DApp 部署 ENS 域名，你可以使用 ENS 官方网站或脚本进行注册：

8. 打开 [ens.domains](https://ens.domains/) 官方网站。

9. 连接你的以太坊钱包（MetaMask等）。

10. 搜索你的域名（如 `web3mail.eth`）。

11. 按照提示完成注册并支付所需的GAS费用。

注册完成后，你可以将ENS域名指向你的DApp前端或合约地址。

---

### **ENS 费用**

ENS 域名注册通常涉及：

- **年费**：根据ENS域名的长度决定，3位字符的域名最贵，5位以上相对便宜。

- **Gas 费**：进行注册或更新时需要支付以太坊的交易费用。

- **子域名**：注册的ENS域名可以免费创建子域名，如 `inbox.alice.eth`。

---





