---
title: ethers和viem快速入门
description: '通过两段示例，对比ethers和viem的使用方法和设计思路'
tags: ['ethers', 'viem', 'web3']
toc: false
date: 2025-05-04 18:10:43
categories:
---

### Ethers
```Solidity
import { ethers, Wallet } from "ethers";
import { abi, bytecode } from "./erc20";

async function main() {

    // 基本使用： 获取当前块高，钱包地址，钱包余额，发送交易等等
    const url = "http://localhost:8545"
    const provider = new ethers.JsonRpcProvider(url);
    const blockNumber = await provider.getBlockNumber();
    console.log(`Current block number: ${blockNumber}`);

    const privateKey = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80";
    const wallet = new Wallet(privateKey, provider);
    const walletAddress = await wallet.getAddress();
    console.log(`Wallet address: ${walletAddress}`);

    const balance = await provider.getBalance(walletAddress);
    console.log(`Wallet balance: ${ethers.formatEther(balance)} ETH`);

    const transaction = {
        to: "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
        value: "1000000000000000000"
    }
    const tx = await wallet.sendTransaction(transaction);
    await tx.wait()
    console.log(`Transaction hash: ${tx.hash}`);

    const newBalance = await provider.getBalance(walletAddress);
    console.log(`New wallet balance: ${ethers.formatEther(newBalance)} ETH`);

    // 合约交互： 部署合约，调用合约方法等等
    const contract = new ethers.ContractFactory(abi,bytecode,wallet);
    const contractInstance = await contract.deploy("name","symbol",18,100000);
    await contractInstance.waitForDeployment();// 等待合约部署完成(打包到区块链)
    const contractAddress = contractInstance.target.toString();
    console.log(`Contract address: ${contractInstance.target}`);

    const readContract = new ethers.Contract(contractAddress,abi,provider);
    const totalSupply = await readContract.totalSupply();
    console.log(`Total supply: ${totalSupply}`);

    const writeContract = new ethers.Contract(contractAddress,abi,wallet);
    const contractTransaction = await writeContract.transfer("0x70997970C51812dc3A010C7d01b50e0d17dc79C8",12345);
    await contractTransaction.wait();
    const erc20Balance = await readContract.balanceOf("0x70997970C51812dc3A010C7d01b50e0d17dc79C8");
    console.log(`New balance: ${erc20Balance}`);

    // 事件监听： 监听合约事件等等
    provider.on("block", async (blockNumber) => {
        console.log(`New block number: ${blockNumber}`);
    });

}

main()
```

### Viem
```Solidity
import { createPublicClient, createWalletClient, defineChain, getContract, hexToBigInt, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { abi, bytecode } from "./erc20";

export const localChain = (url: string) => defineChain({
    id: 31337,
    name: "testnet",
    network: "testnet",
    nativeCurrency: {
        decimals: 18,
        name: "Ether",
        symbol: "ETH",
    },
    rpcUrls: {
        default: {
            http: [url],
        },
        public: {
            http: [url],
        },
    },
    testnet: true,
});

async function main() {
    const url = "http://127.0.0.1:8545";
    const publicClient = createPublicClient({
        chain: localChain(url),
        transport: http(),
    });

    // Public client
    const blockNumber = await publicClient.getBlockNumber();
    console.log(`Current block number: ${blockNumber}`);

    const pk = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80";
    const wallet = privateKeyToAccount(pk);
    const address = wallet.address;
    console.log(`Wallet address: ${address}`);
    const balance = await publicClient.getBalance({ address });
    console.log(`Wallet balance: ${balance}`);

    // Wallet client
    const walletClient = createWalletClient({
        account: wallet,
        chain: localChain(url),
        transport: http(),
    });

    const tx = await walletClient.sendTransaction({
        to: "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
        value: hexToBigInt("0x100"),
    });
    console.log(`Transaction hash: ${tx}`);

    // Deploy contract (return hash)
    const formattedBytecode = `0x${bytecode}`;
    const contract = await walletClient.deployContract({
        abi,
        bytecode: formattedBytecode,
        account: wallet,
        args: ["Test Token", "TST", 18, BigInt(10000)],
    });

    // Get contract address
    const receipt = await publicClient.waitForTransactionReceipt({ hash: contract });
    if (!receipt.contractAddress) {
        throw new Error("Contract deployment failed or no contract address returned.");
    }
    const contractAddress = receipt.contractAddress;
    console.log(`Contract address: ${contractAddress}`);

    // Call contract
    const totalSupply = await publicClient.readContract({
        address: contractAddress, // TypeScript now ensures this is non-null
        abi,
        functionName: "totalSupply",
    });
    console.log(`Total supply: ${totalSupply}`);

    // Transfer tokens
    const writeContract = getContract({
        address: contractAddress,
        abi: abi,
        client: walletClient,
    })
    const tx2 = await writeContract.write.transfer([
        "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
        321,
    ]);
    const erc20balance = await publicClient.readContract({
        address: contractAddress,
        abi: abi,
        functionName: "balanceOf",
        args: ["0x70997970C51812dc3A010C7d01b50e0d17dc79C8"],
    });
    console.log(`ERC20 balance: ${erc20balance}`);

    // Watch block
    publicClient.watchBlockNumber({
        onBlockNumber: (blockNumber) => {
            console.log(`Current block number: ${blockNumber}`);
        },
    });
}

main().catch((error) => {
    console.error("An error occurred:", error);
});
```
