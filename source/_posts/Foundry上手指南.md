---
title: Foundry上手指南
description: 'Foundry 是一个开源的以太坊智能合约开发工具包，旨在提供快速、便携、模块化的开发体验。Foundry 完全用 Rust 编写，避免了繁重的 Node.js 依赖，性能暴打Hardhat'
tags: ['Foundry','web3']
toc: true
date: 2025-05-05 19:03:16
categories: Cyfrin Updraft
---

## Foundry 是什么

Foundry 是一个开源的以太坊智能合约开发工具包，由 Paradigm 等团队开发，旨在提供**快速、便携、模块化**的开发体验。Foundry 完全用 Rust 编写，避免了繁重的 Node.js 依赖，支持多种 Solidity 版本。它主要包含四个组件：

- **Forge**：测试框架（类似于 Truffle/Hardhat），可直接用 Solidity 编写测试合约。

- **Cast**：多功能命令行工具，可执行以太坊 RPC 调用、查询链上数据和发送交易。

- **Anvil**：本地 EVM 节点（类似于 Ganache 或 Hardhat Network），支持快速启动和主网状态分叉（fork）测试。

- **Chisel**：Solidity 解释器（REPL），方便交互式调试和尝试合约代码。

总体而言，Foundry 适合专注于智能合约本身的开发场景，尤其是需要高效测试、模糊测试以及本地链调试的项目。

## 核心特性

- **原生支持 Solidity 编译与测试**：Foundry 的工具链直接调用本地 `solc` 编译合约，并允许在 Solidity 中编写测试，无需 JavaScript/TypeScript 环境。

- **极快的编译和测试速度**：基于 Rust 的二进制程序，编译和运行测试速度远超传统的 Hardhat/Truffle 等 JS 工具。对于大型项目或频繁迭代而言，这带来了明显效率提升。

- **内置模糊测试（Fuzz Testing）**：支持属性（property-based）测试，只要测试函数带有参数（如 `function testFuzz_XXX(uint256 x)`），Forge 会自动生成随机输入进行测试。如果找到失败的输入，还会给出最小化的反例，帮助发现边界问题。

- **丰富的 Cheatcodes**：提供多种“作弊码”用于修改测试时的区块链状态。例如，可以使用 `vm.prank(address)` 伪造交易发送者、`vm.warp(uint)` 修改区块时间戳、`vm.roll(uint)` 改变区块高度、`vm.deal(address, amount)` 设定地址余额、`vm.expectRevert(...)` 预期下一次调用回退等。这些功能可以大幅简化测试场景，如设置账户余额、时间推进等操作无需真实链交互。

- **与 EVM 的高度一致性**：Anvil 作为本地节点，与官方客户端行为高度相符，支持如 Hardhat 那样的主网分叉。测试时可直接以当前或指定区块高度的主网状态作为基础，更真实地模拟生产环境。

## 与 Hardhat 的区别和迁移考量

Foundry 和 Hardhat 都是主流的以太坊开发框架，但在设计理念和生态上有明显差别：

- **语言和依赖**：Hardhat 以 JavaScript/TypeScript 为主，依赖大量 Node.js 包；而 Foundry 是用 Rust 编写的 CLI 工具，不依赖 JS 环境。这意味着 Foundry 安装包更轻量、跨平台性好，但如果项目需要使用 JS 工具链（如前端脚本或第三方 JS 库），可能需要额外集成。

- **测试框架**：Hardhat 测试基于 Mocha/Chai 和 ethers.js，需要编写 JS 脚本来部署合约、发送交易和断言结果；Foundry 测试在 Solidity 内部进行，使用 `DSTest` 或 `forge-std` 标准库中的断言函数，测试代码更贴近合约本身。这对于纯 Solidity 团队来说，上手更快、避免了语言上下文切换。

- **性能**：Foundry 的编译和测试速度显著高于 Hardhat。如果项目测试规模大、迭代频繁，Foundry 可以节省大量时间。

- **生态和插件**：Hardhat 拥有成熟的插件生态，可与各种工具（如 Gas 报告、安全检查、前端框架等）无缝集成；Foundry 生态相对新兴，但也支持部分集成（例如官方提供了 Hardhat 插件，可在同一个项目中同时使用两者）。选择时应考虑团队技术栈：如果以 JS/TS 开发前端和自动化脚本为主，Hardhat 更方便；如果团队主要关心合约逻辑和高效测试，Foundry 更具吸引力。

- **迁移注意**：从 Hardhat 迁移到 Foundry 时，需要将测试代码从 JS 重写为 Solidity 脚本，并将配置文件从 `hardhat.config.js` 转换为 `foundry.toml`。依赖管理也有所不同，Foundry 使用 `forge install` 将依赖以 Git 子模块形式安装到 `lib/` 目录。

## 安装和初始化项目

- **安装 Foundryup**：Foundry 官方提供了安装脚本。执行以下命令即可安装 Foundryup 管理工具：

  ```Shell
curl -L https://foundry.paradigm.xyz | bash
```


  按提示完成安装后，即可在终端使用 `foundryup` 命令。运行 `foundryup` 将自动下载并安装最新稳定版本的 Forge、Cast、Anvil、Chisel 等组件。若要安装最新夜间版，可使用 `foundryup --install nightly`。**注意**：在 Windows 上应通过 Git Bash 或 WSL 环境执行此脚本。

- **初始化新项目**：安装完成后，可以使用 `forge init` 创建新项目。比如：

  ```Shell
forge init hello_foundry
```


  上述命令会在当前目录下创建 `hello_foundry` 文件夹，并根据模板生成项目结构。默认情况下，它还会初始化一个 Git 仓库并安装所需子模块。若不需要 Git 仓库，可加上 `--no-git` 参数。

## 项目结构说明

默认模板创建的 Foundry 项目结构如下：

```Solidity
.
├── foundry.toml       // Foundry 配置文件
├── src                // 智能合约源代码目录
├── test               // 测试合约目录
├── lib                // 依赖库目录（git 子模块）
└── script             // Solidity 脚本目录
```


- **foundry.toml**：项目配置文件，可在其中指定编译器版本、文件路径、编译参数等。

- **src/**：存放待部署的 Solidity 合约源文件，所有最终部署的合约都应放在此目录下。

- **test/**：存放测试合约，文件名通常以 `.t.sol` 结尾。Forge 会编译此目录中的合约，并运行所有函数名以 `test` 开头的函数。例如，`MyContract.t.sol` 中的 `function testFoo()` 会被识别为一个测试用例。

- **lib/**：存放项目依赖的库代码，以 Git 子模块方式管理。例如通过 `forge install` 安装 OpenZeppelin 库时，会将其放在 `lib/openzeppelin-contracts` 子目录中。

- **script/**：通常放置使用 Solidity 编写的脚本文件（后缀 `.s.sol`），用于与链交互（如部署合约或执行操作）。这些脚本可以通过 `forge script` 命令运行，将生成事务并发送到指定网络。

以上目录结构可以通过 `foundry.toml` 或命令行参数进行自定义。同时，Foundry 支持使用 `--hh` 参数来兼容 Hardhat 项目结构（自动设置 `--lib-paths node_modules --contracts contracts`）。

## 编写和运行测试

Foundry 允许直接用 Solidity 编写测试合约。通常在测试文件中导入 Forge 标准库，然后让测试合约继承 `Test`（或 `DSTest`）基类。下面是一个简单示例：

```Solidity
import "forge-std/Test.sol";
contract MyContractTest is Test {
    MyContract c;
    function setUp() public {
        c = new MyContract();
    }
    function testFunctionality() public {
        // 在这里进行断言
        assertTrue(c.value() == 42);
    }
}
```


上述代码在 `test/` 目录下，例如命名为 `MyContract.t.sol`。`setUp()` 函数会在每个测试运行前调用，用于初始化状态（类似于 JS 测试中的 `beforeEach`）。以 `test` 开头的函数（如 `testFunctionality`）会被 Forge 自动识别为测试。测试内部可使用 `assertEq`、`assertTrue`、`assertRevert` 等函数进行断言，无需手动捕捉异常。

运行测试非常简单：在项目根目录运行

```Shell
forge test
```


Forge 会自动编译 `src/` 和 `test/` 目录下的合约，并执行所有测试用例。测试输出会列出每个测试的执行结果及 Gas 消耗情况。如果测试失败，会展示失败信息和回溯调用栈。

与 Hardhat 不同，Foundry 测试过程不依赖 Mocha/Chai，也不需要启动额外节点，完全在本地进行，运行速度更快。而且测试代码与合约语言一致，Solidity 开发者无需切换语言即可编写和调试测试。

## 使用 Cheatcodes

Foundry 的测试框架内置了大量“作弊码”，用于在测试中修改链上状态或检查行为。这些作弊码通过一个预设地址（`0x7109...DD12D`）暴露，通常通过 `vm` 变量调用。例如：

- `vm.prank(address)`：将下一笔交易的发送者伪造为指定地址。

- `vm.startPrank(address)` / `vm.stopPrank()`：连续伪造后续多笔交易的发送者。

- `vm.warp(uint256 timestamp)`：设置区块的时间戳。

- `vm.roll(uint256 blockNumber)`：设置区块高度。

- `vm.deal(address who, uint256 amount)`：为指定地址设置以太币余额。

- `vm.load(address who, bytes32 slot)` / `vm.store(address who, bytes32 slot, bytes32 val)`：直接读写任意地址的存储槽。

- `vm.expectRevert() / vm.expectRevert(bytes4) / vm.expectRevert(string)`：预期下一次合约调用会回退（可指定错误选择子或错误信息）。

- `vm.expectEmit(bool,bool,bool,bool)`：预期下一笔交易会发出某个事件，并可部分匹配事件的索引参数。

例如，下面的测试片段使用 `expectRevert` 和 `prank` 来断言非合约所有者调用失败：

```Solidity
function test_RevertWhen_CallerIsNotOwner() public {
    vm.expectRevert(Unauthorized.selector);
    vm.prank(address(0));
    upOnly.increment();
}
```


通过这些 Cheatcodes，测试可以方便地模拟各种链上操作（例如快速推进时间、设置账户余额、模拟外部合约调用等），从而覆盖更多场景。

## 使用 Fuzz Testing 和属性测试

Foundry 支持“属性测试”（Property-Based Testing），即模糊测试。如果测试函数带有至少一个参数，Forge 会将其视为模糊测试，并自动生成随机输入执行多次测试。例如：

```Solidity
function testFuzz_Withdraw(uint256 amount) public {
    payable(address(safe)).transfer(amount);
    uint256 pre = address(this).balance;
    safe.withdraw();
    uint256 post = address(this).balance;
    assertEq(pre + amount, post);
}
```


在上例中，`testFuzz_Withdraw` 会被 Forge 多次调用，每次传入不同的随机 `amount` 值。Forge 会寻找导致测试失败的输入，并给出经过“缩小”处理（minimal counterexample）的反例。这种方法有助于自动发现边界情况和潜在错误，使得测试覆盖更全面。测试函数名可以以 `testFuzz_` 开头（或者任何 `test` 前缀带参数的函数都会自动 fuzz），也可配合分支条件等，以明确表示这是一个模糊测试场景。

## 合约部署和与前端交互方式简介

Foundry 提供了 `forge` 和 `cast` 两个命令行工具来进行部署和与链交互：

- **部署合约**：使用 `forge create` 命令。例如：

  ```Shell
forge create MyContract --private-key <YOUR_PRIVATE_KEY> --rpc-url <RPC_URL>
```


  这会编译 `MyContract` 合约并使用指定的私钥在指定网络（通过 `--rpc-url`）上部署该合约。可以通过 `--constructor-args` 传入构造函数参数，通过 `--verify` 启用 Etherscan 等服务的源码验证。

- **读取链上数据（只读调用）**：使用 `cast call`，无需私钥。例如，可以调用一个 ERC20 合约的 `balanceOf`：

  ```Shell
cast call 0x6B175474E89094C44Da98b954EedeAC495271d0F "balanceOf(address)(uint256)" --rpc-url <RPC_URL>
```


  该命令通过 ABI 编码并调用合约函数，然后输出返回结果。

- **发送交易**：使用 `cast send` 并提供私钥。比如向 ERC20 合约发送一笔 `transfer` 交易：

  ```Shell
cast send --private-key <YOUR_PRIVATE_KEY> 0xTOKEN_ADDRESS "transfer(address,uint256)" 0xRECIPIENT 100
```


  `cast send` 会使用指定私钥签名并发送交易，可通过 `--rpc-url` 指定节点。这些工具使得在脚本或命令行下即可完成链上交互，无需像 Hardhat 那样启动整个框架环境。

## 最佳实践

- **测试组织**：测试合约文件一般与源合约同名（`MyContract.sol` 对应 `MyContract.t.sol`）。可以为同一个合约创建多个测试合约或分组：一种方式是按功能拆分小的测试合约；另一种方式是每个待测合约对应一个测试合约，内含多个测试函数。无论采用哪种，都建议保持一致的结构便于维护。

- **命名规范**：统一的函数命名有助于管理测试。常见约定如 `testXXX`（一般测试）、`testFuzz_XXX`（模糊测试）、`test_RevertIfYYY`（预期发生回退）等。例如，可使用 `testFork_...` 前缀表示需要主网分叉的测试。遵循这些模式可以在运行 `forge test` 时方便地筛选测试用例。

- **代码格式和静态检查**：建议使用 `forge fmt` 格式化代码，保持代码风格统一。Foundry 还提供静态分析工具（如 `forge check`）检查可能的错误。此外，尽量编写清晰可读的合约和测试代码，例如使用有意义的变量名、避免冗余逻辑、将相关代码放在一起等。

- **持续集成（CI）**：在 CI 流水线中集成 Foundry 通常只需安装 Rust 环境并运行 `forge test`。官方文档提供了 GitHub Actions、Travis CI、GitLab CI 等示例配置。例如，在 GitHub Actions 中可在 push 时触发，安装 Foundry 后执行 `forge test` 验证所有测试通过即可。通过自动化 CI，可确保每次代码提交都经过 Foundry 测试套件的检查。



**参考资料：** Foundry 官方文档和示例等。 



