---
title: Caliper在Fabric网络上的入门示例
description: '联盟链fabric和测试工具caliper的基本使用'
tags: ['密码学','科研']
toc: false
date: 2025-10-28 10:08:18
categories:
    - 科研
    - 密码学
---

# Hyperledger Caliper 在 Fabric 上进行性能测试入门教程 (Go 语言链码)

## 概述

Hyperledger Caliper 是一个区块链性能基准测试工具，支持多种区块链平台。本教程将指导您使用 Caliper 在 Hyperledger Fabric 上运行性能测试案例。

本教程基于 Fabric Samples 的 test-network 和 asset-transfer-basic 的 **Go 语言链码**版本，演示如何测试区块链上的基本资产创建和读取操作。

## 安装和设置

### 1. 安装 Fabric Samples

```bash
git clone https://github.com/hyperledger/fabric-samples.git
cd fabric-samples
```

### 2. 启动 Fabric 网络

```bash
cd test-network
./network.sh up createChannel -ca -s couchdb
./network.sh deployCC -ccp ../asset-transfer-basic/chaincode-go -ccn basic -ccl go
```

**命令参数解释**:

- `up createChannel`: 启动网络并创建应用通道
- `-ca`: 使用证书颁发机构 (CA) 生成证书
- `-s couchdb`: 使用 CouchDB 作为状态数据库 (而不是默认的 LevelDB)
- `deployCC`: 部署链码 (智能合约)
- `-ccp`: 指定链码路径 (chaincode path) - 这里使用 Go 语言链码
- `-ccn`: 指定链码名称 (chaincode name)
- `-ccl`: 指定链码语言 (chaincode language) - 这里使用 go

这将启动一个包含 2 个组织的 Fabric 网络，并部署 `basic` Go 语言链码。

### 3. 创建 Caliper 工作区

```bash
cd ..
mkdir caliper-workspace
cd caliper-workspace
```

### 4. 安装 Caliper

```bash
npm install --only=prod @hyperledger/caliper-cli@0.6.0
```

**命令参数解释**:

- `install`: 安装 npm 包
- `--only=prod`: 只安装生产依赖，不安装开发依赖
- `@hyperledger/caliper-cli@0.6.0`: 指定安装 Caliper CLI 版本 0.6.0

### 5. 绑定 Fabric SDK

```bash
npx caliper bind --caliper-bind-sut fabric:fabric-gateway
```

**命令参数解释**:

- `bind`: 绑定 Caliper 到指定的区块链平台和 SDK
- `--caliper-bind-sut`: 指定系统类型 (fabric:fabric-gateway 表示使用 Fabric Gateway SDK)

## 关键配置文件

### 网络配置文件 (network.yaml)

创建 `network.yaml` 文件：

```yaml
version: '2.0.0'

caliper:
  blockchain: fabric

organizations:
  - mspid: Org1MSP
    identities:
      certificates:
      - name: admin
        admin: true
        clientPrivateKey:
          path: ../test-network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/priv_sk
        clientSignedCert:
          path: ../test-network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts/Admin@org1.example.com-cert.pem
    connectionProfile:
      path: ../test-network/organizations/peerOrganizations/org1.example.com/connection-org1.yaml

channels:
  - channelName: mychannel
    contracts:
    - id: basic
```

**关键配置说明**:

- `version: '2.0.0'`: 配置文件版本
- `organizations`: 定义参与测试的组织及其身份
- `channels`: 指定测试的通道和链码
- 路径必须相对于工作区正确

### 基准测试配置文件 (benchmark.yaml)

创建 `benchmark.yaml` 文件：

```yaml
test:
  name: basic-asset-benchmark
  description: A benchmark for the asset-transfer-basic chaincode
  workers:
    number: 1
  rounds:
    - label: createAsset
      description: Create a new asset
      txNumber: 100
      rateControl:
        type: fixed-rate
        opts:
          tps: 20
      workload:
        module: workloads/createAsset.js
        arguments:
          contractId: basic
          assetSize: 64

    - label: readAsset
      description: Read an existing asset
      txNumber: 100
      rateControl:
        type: fixed-rate
        opts:
          tps: 20
      workload:
        module: workloads/readAsset.js
        arguments:
          contractId: basic
```

**关键配置说明**:

- `workers.number`: 工作节点数量
- `rounds`: 测试轮次，每个轮次定义交易数量、速率控制和工作负载
- `rateControl`: 控制交易发送速率
- `workload`: 指定工作负载模块和参数

### 工作负载模块

创建 `workloads/` 目录，并创建工作负载文件。

#### createAsset.js

```javascript
'use strict';

const { WorkloadModuleBase } = require('@hyperledger/caliper-core');

class CreateAssetWorkload extends WorkloadModuleBase {
    constructor() {
        super();
        this.txIndex = 0;
    }

    async submitTransaction() {
        this.txIndex++;
        const assetID = `asset_${this.workerIndex}_${this.txIndex}`;
        
        const args = {
            contractId: this.roundArguments.contractId,
            contractFunction: 'CreateAsset',
            contractArguments: [assetID, 'blue', '20', 'Alan', '150'],
            readOnly: false
        };

        await this.sutAdapter.sendRequests(args);
    }
}

function createWorkloadModule() {
    return new CreateAssetWorkload();
}

module.exports.createWorkloadModule = createWorkloadModule;
```

#### readAsset.js

```javascript
'use strict';

const { WorkloadModuleBase } = require('@hyperledger/caliper-core');

class ReadAssetWorkload extends WorkloadModuleBase {
    constructor() {
        super();
        this.txIndex = 0;
    }

    async submitTransaction() {
        this.txIndex++;
        const assetID = `asset_${this.workerIndex}_${this.txIndex}`;

        const args = {
            contractId: this.roundArguments.contractId,
            contractFunction: 'ReadAsset',
            contractArguments: [assetID],
            readOnly: true
        };

        await this.sutAdapter.sendRequests(args);
    }
}

function createWorkloadModule() {
    return new ReadAssetWorkload();
}

module.exports.createWorkloadModule = createWorkloadModule;
```

## 运行测试

```bash
./node_modules/.bin/caliper launch manager --caliper-workspace . --caliper-benchconfig benchmark.yaml --caliper-networkconfig network.yaml
```

**命令参数解释**:

- `launch manager`: 启动 Caliper 管理器进程来协调测试
- `--caliper-workspace .`: 指定工作区目录 (当前目录)
- `--caliper-benchconfig benchmark.yaml`: 指定基准测试配置文件
- `--caliper-networkconfig network.yaml`: 指定网络配置文件

## 结果分析

测试完成后，会生成 `report.html` 文件，包含：

- 交易成功/失败统计
- 发送速率 (TPS)
- 延迟统计 (最大、最小、平均)
- 吞吐量 (TPS)
- 图表可视化

## 注意点

1. **路径配置**: 所有文件路径必须相对于工作区根目录正确。注意相对路径的解析。

2. **版本兼容性**: 确保 Fabric 版本与 Caliper 绑定的 SDK 版本兼容。

3. **网络状态**: 在运行测试前，确保 Fabric 网络正常运行且链码已部署。

4. **资源管理**: 根据测试规模调整 Docker 资源限制。

5. **安全配置**: 在生产环境中，妥善管理私钥和证书。

6. **工作负载设计**: 工作负载应模拟真实的业务场景，避免过度简化。

## 常见问题解答

### 1. 工作负载模块一定要用 JavaScript 写吗？

**不一定**，但 JavaScript 是最常用和推荐的选择。

**支持的语言选项**：

- **JavaScript/Node.js** (推荐): 最成熟，社区支持最好
- **TypeScript**: 提供类型安全，编译为 JavaScript
- **其他语言**: 理论上支持，但需要额外配置

**为什么推荐 JavaScript**：

- Caliper 核心基于 Node.js
- 丰富的 Fabric SDK 支持
- 易于与现有工作负载示例集成

**如果要使用其他语言**：

```javascript
// 在 benchmark.yaml 中指定
workload:
  module: workloads/myWorkload.ts  // TypeScript 文件
  // 或
  module: workloads/myWorkload.py  // Python (需要额外配置)
```

### 2. 链码具体怎么写？规范是什么？

**不是简单改造**，需要完全重写以符合 Fabric 链码规范。

#### Fabric 链码规范要求

**核心接口**：
```go
// 必须实现的接口
type ContractInterface interface {
    // 链码初始化
    InitLedger(ctx contractapi.TransactionContextInterface) error
    
    // 业务函数
    // 您的函数将在这里定义
}

// 标准链码结构
type SmartContract struct {
    contractapi.Contract
}

// 构造函数
func (s *SmartContract) InitLedger(ctx contractapi.TransactionContextInterface) error {
    // 初始化逻辑
    return nil
}
```

**关键规范**：

1. **状态管理**: 使用 `ctx.GetStub().PutState()` 和 `GetState()` 替代内存变量
2. **确定性**: 所有操作必须是确定性的，不能使用随机数或时间
3. **错误处理**: 必须返回错误，不能 panic
4. **序列化**: 复杂对象需要 JSON 序列化存储
5. **背书策略**: 需要配置适当的背书策略

#### 从您的 blockchain.go 迁移的映射关系

| 原代码 | Fabric 链码改造 |
|--------|------------------|
| `BlockChain` 结构体 | `SmartContract` 结构体 |
| `RegisterCSP()` 方法 | `RegisterCSP()` 链码函数 |
| `Verify()` 方法 | `VerifyConsistency()` 链码函数 |
| 内存 `RegisteredCSPs` map | 账本状态 `csp_registry` |
| 内存证明存储 | 账本状态 `proofs_{dataID}` |

### 3. 工作负载模块的每个代码就是模拟算法中的每一个步骤吗？

**是的**，工作负载模块模拟算法执行流程的各个步骤。

#### 工作负载设计原则

**每个工作负载模块代表一个业务操作类型**：

- `registerCSP.js`: 模拟 CSP 注册流程
- `uploadData.js`: 模拟数据上传和元数据记录
- `auditRequest.js`: 模拟 DO 发起审计请求
- `submitProof.js`: 模拟 CSP 生成和提交证明
- `verifyConsistency.js`: 模拟区块链验证两个 CSP 的数据一致性

**工作负载 ≠ 算法步骤**，而是**用户行为模拟**：

- 一个工作负载模块 = 一个用户操作类型
- 内部可以包含多个算法步骤
- 重点是模拟真实用户的调用模式和频率

#### 工作负载执行流程

```
用户行为 → 工作负载模块 → 链码函数调用 → 算法执行 → 账本更新
     ↓
性能收集 → Caliper 报告
```

**示例**：`verifyConsistency.js` 工作负载

```javascript
// 这个工作负载模拟：
// 1. 选择要验证的两个 CSP (算法中的"选择验证对"步骤)
// 2. 调用链码验证函数 (算法中的"执行验证"步骤)
// 3. 收集验证结果 (性能测试需要)
```
