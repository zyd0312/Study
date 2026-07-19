# 公链 Rollup 深入调研阅读清单：交互计算与证明细节

整理日期：2026-07-14

这份清单重点关注 Rollup 内部“每一步干了什么、算了什么”：交易如何从 L1/L2 输入推导出 L2 状态，Optimistic Rollup 如何通过交互式争议游戏定位错误步骤，ZK Rollup 如何把执行轨迹变成约束和证明。

## 1. 推荐阅读顺序

建议按下面顺序读，不然很容易被证明系统的细节拐进雾里。

1. 先读 Rollup 的交易推导和状态承诺。
2. 再读 Optimistic Rollup 的 fault proof / fraud proof，尤其是二分争议游戏。
3. 然后读 fault proof VM 如何验证单步计算。
4. 用 OP Succinct 对比“原来 optimistic challenge 怎样确认状态，现在 ZK validity proof 怎样确认状态”。
5. 再读 ZK Rollup 的执行轨迹、约束系统、prover 和 verifier。
6. 最后读论文和源码。

## 2. Optimistic Rollup：交互式 Fault Proof / Fraud Proof

这一部分最贴近“交互计算细节”。核心问题是：

```text
两个参与者对某个 L2 状态根有分歧
 -> 他们如何交互式缩小分歧范围
 -> 如何定位到某个 L2 block 或某条 VM 指令
 -> L1 合约如何只验证一个最小计算步骤
```

### 2.1 OP Stack Fault Proof 总入口

链接：

- https://specs.optimism.io/fault-proof/index.html

重点看：

- fault proof 的三个组件：Program、VM、Interactive Dispute Game；
- Program 如何根据 L1 数据无状态重放 L2 状态转换；
- VM 如何跟踪执行 trace；
- Interactive Dispute Game 如何二分到单条指令；
- Pre-image Oracle 如何给 VM 提供外部数据。

它回答的问题：

```text
争议中真正被验证的计算是什么？
程序如何从 L1 输入推导 L2 状态？
为什么最后只需要在 L1 上验证一个 VM step？
```

### 2.2 OP Stack Fault Dispute Game

链接：

- https://specs.optimism.io/fault-proof/stage-one/fault-dispute-game.html

重点看：

- Claim；
- Attack / Defend；
- Game Tree；
- DAG；
- Position / generalized index；
- Execution Trace；
- Step；
- PreimageOracle Interaction；
- Game Clock；
- Resolution。

它是理解 OP fault proof 的核心文档。里面会解释：

```text
root claim 声称某个 L2 output root 正确
challenger 对 claim 发起 attack
defender 对某些 claim 发起 defend
双方不断提交更细粒度的 trace commitment
最终定位到单个 FPVM 状态转换
L1 调用 step() 验证单步计算是否正确
```

重点关注 `step()`：

```solidity
function step(
    uint256 _claimIndex,
    bool _isAttack,
    bytes calldata _stateData,
    bytes calldata _proof
) external;
```

这里的 `_stateData` 是 VM 状态数据，`_proof` 是访问 VM Merkle memory / state 的证明。最终 L1 不需要重放整个 L2 batch，只验证某一个 VM 指令步骤。

### 2.3 OP Stack Cannon Fault Proof VM

链接：

- https://specs.optimism.io/fault-proof/cannon-fault-proof-vm.html
- https://github.com/ethereum-optimism/cannon

重点看：

- Cannon / MTCannon 如何把 fault proof program 放进 MIPS64 VM；
- VM state 包含哪些字段；
- state hash 怎么算；
- memory 如何用 Merkle tree 表示；
- 单步 `f(S_pre) -> S_post` 到底更新了什么；
- syscall / read / write / preimage communication 如何建模；
- 多线程 Cannon 如何处理 thread state。

核心计算模型：

```text
FPVM state S_pre
 -> 读取当前 pc 对应的一条 MIPS64 指令
 -> 根据寄存器、memory root、syscall/preimage 状态执行一步
 -> 得到 S_post
 -> 对 S_post 重新 hash
 -> 与 claim 中的 post-state commitment 比较
```

这篇适合精读，因为它几乎就是“交互计算细节”本体。

### 2.4 OP Stack Derivation

链接：

- https://specs.optimism.io/protocol/derivation.html

重点看：

- L1 block、batcher transaction、deposit transaction 如何推导成 L2 block；
- L2 chain 如何从 L1 数据确定性派生；
- channel、frame、batch、payload attributes 的处理过程；
- rollup node 和 execution engine 如何协作。

它回答：

```text
L2 状态不是凭空出现的，它从哪些 L1 数据推导出来？
如果要在 fault proof program 中重放 L2，输入到底是什么？
```

### 2.5 OP Stack Proposals / Output Root

链接：

- https://specs.optimism.io/protocol/proposals.html

重点看：

- output root 是什么；
- proposer 提交的 claim 到底承诺了哪些状态；
- withdrawal root、state root、block hash 如何进入 output root；
- fault proof 最终要验证的对象是什么。

### 2.6 OP Succinct：从 Optimistic Challenge 到 ZK Validity Proof

这一组文档非常适合作为 L1/L2 Rollup 交互细节的精读案例。它不是把 OP Stack 的执行层全部重写成 zkEVM，而是把 L1 上确认 L2 output root 的机制，从“先提交、等挑战”改成“提交时附带 validity proof”。

需要先校准一个说法：

```text
不准确说法：
Optimism 已经把内核整体换成 ZK

更准确说法：
Optimism 正在把 OP Stack 的状态确认机制升级为 ZK validity proof 路线。
L2 执行和数据流大体保留，替换的是 L1 上确认 output root 的证明/结算机制。
```

#### 2.6.1 官方公告与背景

链接：

- https://optimism.io/blog/optimism-partners-with-succinct-as-preferred-provider-to-accelerate-zk-proving-on-the-superchain
- https://blog.succinct.xyz/op-succinct/

重点看：

- Succinct 成为 Optimism 首个 preferred provider for ZK proving；
- validity proofs 会成为 OP Stack 的 canonical 证明方向；
- OP Succinct 的目标是消除 optimistic challenge period；
- 目标效果是更快的 withdrawal finality；
- 这是 Superchain multi-proof architecture 的一部分。

需要注意：Optimism 官方博客是战略公告，不是完整技术规范。它说明方向，但要理解“怎么替换”，需要继续读 OP Succinct 文档和源码。

#### 2.6.2 OP Succinct 文档入口

链接：

- https://succinctlabs.github.io/op-succinct/introduction.html
- https://succinctlabs.github.io/op-succinct/architecture.html
- https://succinctlabs.github.io/op-succinct/validity/quick-start.html
- https://succinctlabs.github.io/op-succinct/validity/contracts/intro.html
- https://github.com/succinctlabs/op-succinct

重点看：

- OP Succinct 有两种模式：
  - ZK fault proofs / OP Succinct Lite：只有争议时生成 ZK proof；
  - Validity proofs / OP Succinct：每个交易范围都生成 ZK proof，完全消除 dispute。
- 标准 OP Stack 有 `op-geth`、`op-batcher`、`op-node`、`op-proposer`；
- OP Succinct 主要替换链上合约和 `op-proposer`；
- `op-geth`、`op-batcher`、`op-node`、indexer 等组件大体不变；
- `OPSuccinctL2OutputOracle.sol` 用 validity proof 验证 proposed state root；
- `OPSuccinctDisputeGame.sol` 实现 `IDisputeGame` 接口，用于 validity proof verification。

#### 2.6.3 原 OP Stack 怎样确认状态

原来的标准 OP Stack 流程：

```text
用户交易
 -> sequencer 排序并生成 L2 block
 -> op-batcher 把 L2 batch 数据提交到 L1
 -> op-node 从 L1 batch 数据推导 L2 chain
 -> op-geth 执行交易，得到 L2 state root
 -> op-proposer 把 output root 提交到 L1
 -> 等待挑战期
 -> 如果没人挑战，withdrawal 可以 finalize
```

如果有人挑战：

```text
challenger 质疑 output root
 -> FaultDisputeGame 开始
 -> 双方围绕执行 trace 交互式二分
 -> 定位到某个 VM 单步执行
 -> Cannon / FPVM 在 L1 上验证这一小步
 -> 判断 proposer 或 challenger 谁错
```

原来的安全逻辑：

```text
proposer 可以先提交 output root
但是必须给 challenger 一个窗口来证明它错了
```

这里真正要验证的计算是：

```text
L1 数据 + rollup 配置 + 前一个 L2 状态
 -> derivation
 -> execution
 -> 新 L2 状态
 -> output root
```

#### 2.6.4 OP Succinct 现在怎样确认状态

OP Succinct validity proof 模式：

```text
用户交易
 -> sequencer 排序
 -> op-batcher 提交 batch 到 L1
 -> op-node / op-geth 仍按 OP Stack 规则运行
 -> OP Succinct proposer 监听 L1 和 L2
 -> 用 Kona 重放 OP Stack state transition
 -> 把这个执行过程放进 SP1 zkVM 证明
 -> 对一段 block range 生成 range proof
 -> 多个 range proof 聚合成 aggregation proof
 -> 提交到 L1 的 ZK verifier / OPSuccinctL2OutputOracle
 -> L1 验证 proof 后更新 verified state root
 -> withdrawal 可基于已验证 state root finalize
```

原来的：

```text
提交 output root -> 等挑战期 -> 有错就挑战
```

现在变成：

```text
提交 output root + ZK proof -> L1 验证 proof -> 通过后 final
```

换掉的不是 L2 执行本身，而是这部分：

```text
L1 上如何相信 L2 output root
```

没有主要替换的部分：

```text
sequencer
op-batcher
op-node
op-geth
L2 交易格式
L2 合约执行语义
EVM 兼容性
```

#### 2.6.5 原来与现在的对比

| 位置 | 原 OP Stack | OP Succinct validity 模式 |
| --- | --- | --- |
| L2 执行 | `op-geth` 执行交易 | 基本不变 |
| L2 推导 | `op-node` 从 L1 batch 推导 L2 | 基本不变 |
| batch 提交 | `op-batcher` 提交数据到 L1 | 基本不变 |
| 状态提交 | `op-proposer` 提交 output root | OP Succinct proposer 提交 proof + state root |
| L1 验证 | 等挑战期，错误时 fraud/fault proof | L1 verifier 直接验证 validity proof |
| 争议机制 | FaultDisputeGame + Cannon FPVM | validity 模式下不需要 dispute |
| 提现终局性 | 依赖挑战期，通常约 7 天 | 依赖 proof 生成和链上验证 |
| 关键证明 | “如果错，会有人挑战” | “提交时已经证明正确” |

#### 2.6.6 为什么这个场景适合研究 L1/L2 交互

OP Succinct 正好把 Rollup 的核心问题剖开：

```text
L2 算了什么？
L1 看到了什么？
L1 为什么相信 L2？
原来靠谁挑战？
现在靠谁证明？
```

沿着 OP -> OP Succinct 这条线读，会自然摸到这些细节：

- L1 batch 数据如何被 `op-node` 推导成 L2 block；
- `op-geth` 如何执行 L2 transaction 并更新 state root；
- `op-proposer` 原本提交的 output root 到底承诺了什么；
- FaultDisputeGame 如何把争议二分到 FPVM 单步计算；
- Cannon 如何在 L1 上验证单条 MIPS 指令状态转换；
- Kona 如何作为 OP Stack state transition function 的 Rust 实现；
- SP1 zkVM 如何证明 Kona 执行得到的 output root 正确；
- L1 verifier 如何从“等待挑战”变成“验证 proof”。

这条阅读线很适合深入理解：

```text
L1 数据 -> L2 状态推导
L2 执行 -> output root
output root -> fault proof / validity proof
挑战期 finality -> proof finality
```

## 3. Arbitrum：Nitro、BoLD 与 One-Step Proof

Arbitrum 的技术路线也属于 Optimistic Rollup，但它的术语和 OP Stack 不一样。读 Arbitrum 时要抓住这条线：

```text
Sequencer 排序
 -> Nitro STF 确定性执行
 -> validator 提交 assertion
 -> 出现分歧时进入 BoLD challenge
 -> bisection 缩小争议
 -> one-step proof 在 Ethereum 上裁决
```

### 3.1 Inside Arbitrum Nitro

链接：

- https://docs.arbitrum.io/how-arbitrum-works/inside-arbitrum-nitro

重点看：

- transaction lifecycle；
- Sequencer 如何排序和发布 batch；
- ArbOS + Geth 如何执行；
- state transition function 如何处理交易；
- assertion 与 validation；
- BoLD 如何进入 dispute resolution。

### 3.2 Arbitrum Nitro Whitepaper

链接：

- https://docs.arbitrum.io/nitro-whitepaper.pdf
- https://github.com/OffchainLabs/nitro/blob/master/docs/Nitro-whitepaper.pdf

重点看：

- sequencing followed by deterministic execution；
- Geth at the core；
- separate execution from proving；
- interactive fraud proof；
- assertion protocol；
- challenge sub-protocol。

这篇比网页文档更适合做理论整理，因为它解释了 Nitro 为什么要把“普通执行”和“证明执行”分开：

```text
日常节点：native execution，追求快
争议证明：WASM / WAVM proving execution，追求可验证和机器无关
```

### 3.3 BoLD Technical Deep Dive

链接：

- https://docs.arbitrum.io/how-arbitrum-works/bold/bold-technical-deep-dive

重点看：

- assertion；
- challenge manager；
- edge；
- rival edge；
- timer；
- bonding；
- one-step prover；
- all-vs-all dispute；
- permissionless validation。

它回答：

```text
多个挑战者/验证者同时参与时，怎么避免被恶意挑战拖住？
如何保证只有正确 assertion 最终确认？
为什么 BoLD 可以 bounded delay？
```

### 3.4 BoLD Bisection

链接：

- https://docs.arbitrum.io/how-arbitrum-works/bold/how-bold-bisection-works

重点看：

- 如何把争议拆成 block、bigstep、smallstep；
- 如何从大范围历史承诺逐步缩小到单个执行步骤；
- 最后如何触发 one-step proof。

核心模型：

```text
双方对最终状态不同意
 -> 先比较中间状态承诺
 -> 找到第一个分歧区间
 -> 继续二分
 -> 定位到单步计算
 -> L1 合约验证该步
```

### 3.5 Arbitrum Assertions

链接：

- https://docs.arbitrum.io/how-arbitrum-works/deep-dives/assertions

重点看：

- assertion 包含什么；
- assertion 与 batch / inbox position 的关系；
- history commitment 如何承诺中间状态；
- assertion 如何与挑战期、提现最终性关联。

### 3.6 Arbitrum STF Inputs

链接：

- https://docs.arbitrum.io/how-arbitrum-works/reference/stf-inputs

重点看：

- state transition function 的输入有哪些；
- sequencer message、delayed inbox message、L1/L2 消息如何进入 STF；
- 重放状态时到底要喂给 STF 什么数据。

### 3.7 BoLD 论文与源码

链接：

- https://arxiv.org/abs/2404.10491
- https://github.com/OffchainLabs/bold
- https://github.com/OffchainLabs/nitro

重点看：

- BoLD 的 bounded delay 安全论证；
- dispute game 的经济模型；
- 合约和 validator 客户端如何配合。

## 4. ZK Rollup：Trace、Circuit、Prover、Verifier

ZK Rollup 不像 Optimistic Rollup 那样进行多轮挑战。它的细节重点是：

```text
执行产生 trace
 -> trace 被展开成约束系统
 -> prover 生成 proof
 -> L1 verifier 验证 proof
 -> L1 接受新的 state root
```

### 4.1 Ethereum ZK Rollup 基础

链接：

- https://ethereum.org/en/developers/docs/scaling/zk-rollups/

重点看：

- validity proof；
- state root；
- on-chain verifier；
- data availability；
- ZK Rollup 与 Optimistic Rollup 的区别。

### 4.2 Vitalik：ZK-SNARK 入门

链接：

- https://vitalik.eth.limo/general/2021/01/26/snarks.html

重点看：

- 为什么不能逐步检查大型计算；
- 如何用多项式编码大量计算步骤；
- polynomial commitment 是什么；
- FRI 如何通过折叠和随机抽样做低度测试；
- Fiat-Shamir 如何把交互式证明变成非交互式证明。

这篇适合理解：

```text
为什么 verifier 不需要重新执行计算？
为什么一个小 proof 能代表大量计算步骤？
```

### 4.3 Vitalik：PLONK

链接：

- https://vitalik.eth.limo/general/2019/09/22/plonk.html

重点看：

- PLONK 的约束思想；
- gate、wire、permutation argument；
- 复制约束如何表达“这些位置的值相同”；
- 为什么 zkEVM 常用 PLONKish 约束。

### 4.4 Vitalik：ZK-EVM 类型

链接：

- https://vitalik.eth.limo/general/2022/08/04/zkevm.html

重点看：

- Type 1 / Type 2 / Type 3 / Type 4 zkEVM；
- EVM 等价性与证明效率的取舍；
- 为什么完全等价 Ethereum 很难证明；
- hash、state tree、precompile、memory、gas 规则为什么会影响证明成本。

## 5. Scroll：zkEVM Rollup 的 Commit / Prove / Finalize

Scroll 的文档很适合看 ZK Rollup 工程流程，因为它明确拆成三阶段。

链接：

- https://docs.scroll.io/en/technology/chain/rollup/
- https://docs.scroll.io/en/technology
- https://docs.scroll.io/en/technology/zkevm/intro-to-zkevm

重点看：

- Phase 1: Transaction Execution；
- Phase 2: Batching and Data Commitment；
- Phase 3: Proof Generation and Finalization；
- chunk / batch 如何组织；
- commit transaction 提交什么；
- finalize transaction 验证什么。

关键公式：

```text
batch.dataHash := keccak(chunk[0].dataHash || ... || chunk[n-1].dataHash)

publicInputHash := keccak(
    chainId ||
    prevStateRoot ||
    postStateRoot ||
    withdrawRoot ||
    batch.dataHash
)
```

这个文档要重点理解：

```text
commit 阶段提交数据承诺
prove 阶段为执行 trace 生成证明
finalize 阶段用 proof + public input 更新 L1 记录的 state root
```

## 6. Linea：Trace Expansion、Circuit、gnark Prover

Linea 的文档适合看“trace 如何进入 circuit / prover”。

### 6.1 Linea Protocol Overview

链接：

- https://docs.linea.build/protocol/overview

重点看：

- Lineth protocol；
- batch；
- trace blocks；
- validity proofs；
- finalization layer。

### 6.2 Linea Prover

链接：

- https://docs.linea.build/protocol/architecture/prover

重点看：

- execution proofs；
- compression proofs；
- aggregation proofs；
- inner proof / outer proof；
- Corset + gnark 的分工。

### 6.3 Linea Circuit Building

链接：

- https://docs.linea.build/protocol/architecture/prover/trace-expansion

重点看：

- Corset 如何生成约束系统；
- trace expansion 是什么；
- 运行时 trace 如何被展开成 gnark 可处理的数据；
- constraint system 如何对应 zkEVM 规则。

### 6.4 Linea Circuit Execution and Runtime

链接：

- https://docs.linea.build/protocol/architecture/prover/proving

重点看：

- gnark frontend / backend；
- circuit 如何生成；
- expanded trace 如何进入 backend；
- proof 如何返回 coordinator。

### 6.5 Linea RISC-V Proving Architecture

链接：

- https://docs.linea.build/protocol/architecture/prover/risc-v-overview

重点看：

- guest program；
- RISC-V 作为证明目标；
- 为什么从 bespoke circuit 演进到 zkVM / guest program 模型；
- 这种路线如何让 EVM 升级更容易适配。

## 7. ZKsync Era：EraVM、Circuits、State Diff

ZKsync 适合看“不是完全照搬 EVM，而是为证明效率设计 VM”的路线。

链接：

- https://docs.zksync.io/zksync-protocol/rollup
- https://docs.zksync.io/zksync-protocol/era-vm/vm
- https://docs.zksync.io/zksync-protocol/era-vm/circuits
- https://docs.zksync.io/zksync-protocol/rollup/data-availability

重点看：

- Node Implementation；
- ZK Circuits；
- Prover；
- Smart Contracts；
- EraVM 指令和状态模型；
- circuits 如何覆盖 VM 执行、opcode、storage、precompile；
- state diff 如何作为数据可用性提交。

核心阅读问题：

```text
EraVM 执行一笔交易时状态如何变化？
这些状态变化如何变成 circuit witness？
哪些 circuit 负责 VM、RAM、storage、log sorting、code decommitment？
为什么 ZKsync 更关注 state diff 而非完整交易重放？
```

## 8. Starknet：Cairo、AIR、STARK、S-two

Starknet 适合看“为证明系统重新设计执行环境”的路线。它不是 zkEVM，而是 Cairo VM + STARK。

### 8.1 Starknet Protocol Intro

链接：

- https://docs.starknet.io/learn/protocol/intro

重点看：

- validity rollup；
- sequencer；
- SNOS；
- SHARP；
- compressed state diffs；
- Ethereum 上的验证。

### 8.2 Starknet Blocks

链接：

- https://docs.starknet.io/learn/protocol/blocks

重点看：

- block header；
- global state root；
- state diff commitment；
- transactions commitment；
- events commitment；
- receipts commitment；
- Poseidon hash；
- state diff commitment 的具体输入。

它适合研究：

```text
一个 Starknet block 到底承诺了哪些数据？
这些 commitment 是如何算出来的？
```

### 8.3 Starknet Transactions

链接：

- https://docs.starknet.io/learn/protocol/transactions

重点看：

- transaction types；
- invoke / declare / deploy_account；
- validate / execute；
- transaction lifecycle；
- transaction hash 和 receipt。

### 8.4 S-two Book

链接：

- https://docs.starknet.io/learn/S-two-book/introduction
- https://docs.starknet.io/learn/S-two-book/why-use-a-proof-system
- https://docs.starknet.io/learn/S-two-book/why-stwo

重点看：

- AIR；
- Circle STARK；
- Mersenne31 prime field；
- frontend / backend；
- 为什么 STARK 适合重复性 VM 执行；
- 为什么 S-two 不一定提供 zero-knowledge 隐私，但适合 validity proof。

### 8.5 Cairo AIR 论文

链接：

- https://arxiv.org/abs/2109.14534

重点看：

- Cairo 程序执行如何表示成代数约束；
- execution trace 如何被 AIR 捕获；
- AIR 满足性如何推出 Cairo 计算正确。

## 9. Polygon zkEVM / CDK / Agglayer

Polygon zkEVM 的早期技术路线是 EVM-compatible zkEVM。当前 Polygon 官方资料更多转向 CDK 和 Agglayer。

链接：

- https://docs.polygon.technology/chain-development/cdk
- https://l2beat.com/scaling/projects/polygonzkevm
- https://vitalik.eth.limo/general/2022/08/04/zkevm.html

重点看：

- zkEVM 类型；
- CDK 中 rollup / validium / sovereign chain 的区别；
- Agglayer / pessimistic proof 的方向；
- Polygon 早期 zkEVM 与当前 CDK 叙事的区别。

## 10. 论文与综述

### 10.1 Fraud and Data Availability Proofs

链接：

- https://arxiv.org/abs/1809.09044

重点看：

- fraud proof；
- data availability proof；
- light client security；
- dishonest majority 下如何维持可验证性。

### 10.2 BoLD: Fast and Cheap Dispute Resolution

链接：

- https://arxiv.org/abs/2404.10491

重点看：

- Arbitrum BoLD 的理论模型；
- delay attack；
- all-vs-all dispute；
- bounded confirmation delay；
- staking / bonding 成本。

### 10.3 Dynamic Fraud Proof

链接：

- https://arxiv.org/abs/2502.10321

重点看：

- 动态挑战期；
- optimistic execution；
- verifier 节点交互批准；
- fraud proof finality 改进方向。

### 10.4 Economic Censorship Games in Fraud Proofs

链接：

- https://arxiv.org/abs/2502.20334

重点看：

- fraud proof 挑战期和 L1 审查攻击；
- 攻击者如何通过贿赂 block proposer 影响挑战交易；
- 挑战期长度和参与者预算之间的关系。

### 10.5 Constraint-Level Design of zkEVMs

链接：

- https://arxiv.org/abs/2510.05376

重点看：

- zkEVM 约束层设计；
- R1CS、PLONKish、AIR 的取舍；
- Type 1-4 zkEVM 的约束复杂度；
- dispatch / selector / ROM 机制；
- production zkEVM 的设计差异。

## 11. 源码阅读入口

### OP Stack

- https://github.com/ethereum-optimism/optimism
- https://github.com/ethereum-optimism/cannon
- https://github.com/succinctlabs/op-succinct
- https://github.com/op-rs/kona
- https://specs.optimism.io/

建议先看：

- `op-program`；
- `op-node` derivation；
- `op-geth`；
- fault proof contracts；
- Cannon VM；
- OP Succinct `contracts`；
- OP Succinct `programs`；
- OP Succinct `validity` proposer；
- Kona 中的 OP Stack state transition function。

### Arbitrum

- https://github.com/OffchainLabs/nitro
- https://github.com/OffchainLabs/bold

建议先看：

- assertion 相关合约；
- ChallengeManager；
- OneStepProver；
- WAVM / proving 相关代码；
- validator client。

### ZKsync

- https://github.com/matter-labs/zksync-os-server
- https://github.com/matter-labs/zksync-protocol
- https://github.com/matter-labs/zksync-airbender

建议先看：

- VM；
- bootloader；
- circuits；
- prover；
- state diff / pubdata。

### Scroll

- https://github.com/scroll-tech
- https://docs.scroll.io/en/technology/chain/rollup/

建议先看：

- rollup contract；
- batch header；
- chunk / batch encoding；
- verifier；
- prover coordinator。

### Linea

- https://docs.linea.build/protocol/architecture/prover
- https://docs.linea.build/protocol/architecture/prover/trace-expansion
- https://docs.linea.build/protocol/architecture/prover/proving

建议先看：

- Corset；
- gnark；
- coordinator；
- trace expansion；
- proof aggregation。

## 12. 精读优先级

如果时间有限，建议先读这 12 个：

1. OP Fault Proof Overview  
   https://specs.optimism.io/fault-proof/index.html

2. OP Fault Dispute Game  
   https://specs.optimism.io/fault-proof/stage-one/fault-dispute-game.html

3. OP Cannon FPVM  
   https://specs.optimism.io/fault-proof/cannon-fault-proof-vm.html

4. OP Succinct Architecture  
   https://succinctlabs.github.io/op-succinct/architecture.html

5. OP Succinct Validity Quick Start  
   https://succinctlabs.github.io/op-succinct/validity/quick-start.html

6. OP Succinct Contract Management  
   https://succinctlabs.github.io/op-succinct/validity/contracts/intro.html

7. OP Succinct GitHub  
   https://github.com/succinctlabs/op-succinct

8. Arbitrum Nitro Whitepaper  
   https://docs.arbitrum.io/nitro-whitepaper.pdf

9. Arbitrum BoLD Technical Deep Dive  
   https://docs.arbitrum.io/how-arbitrum-works/bold/bold-technical-deep-dive

10. Scroll Rollup Process  
   https://docs.scroll.io/en/technology/chain/rollup/

11. Linea Circuit Building  
   https://docs.linea.build/protocol/architecture/prover/trace-expansion

12. Vitalik ZK-SNARKs Explained  
   https://vitalik.eth.limo/general/2021/01/26/snarks.html

这 12 篇合起来能覆盖：

```text
L1 数据 -> L2 状态推导
状态承诺 -> output root / assertion
交互争议 -> 二分 -> 单步证明
optimistic challenge -> ZK validity proof
执行 trace -> 约束系统 -> ZK proof
```

---

## 13. Fault Proof 核心疑点 Q&A

以下 Q&A 基于 OP Stack Fault Proof 规范的深入讨论，覆盖交互式争议游戏中最容易产生困惑的概念。

### Q1: 文档中提到的 VM Instruction / VM Step 是什么意思？

**答**：在 OP Stack Fault Proof 的上下文中，一个 VM Step 指的是**被模拟 CPU 中单条机器指令的执行**。当前 OP Stack Cannon / MTCannon 路线主要使用的是 MIPS64 指令集；RISC-V 更多出现在其他 zkVM / proving 系统语境里，不能直接和 Cannon 混为一谈。

整个 Fault Proof Program（如 `op-program`）被编译成 MIPS 二进制文件，在 VM 中逐条指令执行。每一步就是一条 CPU 指令，比如 `ADD`（加法）、`LOAD`（从内存加载）、`STORE`（写入内存）、`JUMP`（跳转）等。

```
程序执行过程:
┌─────────────────────────────────────────────────┐
│ Step 0  →  Step 1  →  Step 2  →  ...  →  Step N │
│ (指令1)    (指令2)    (指令3)           (指令N)  │
└─────────────────────────────────────────────────┘
```

整个 Fault Proof 系统的精妙之处在于：通过链上的 dispute game 提交逐层缩小的状态承诺，将"验证整个 Rollup 状态转换"这个巨大的问题，简化为在链上执行并验证**仅仅一条 CPU 指令**是否正确。

### Q2: VM 在 OP Stack 中的位置和角色是什么？

**答**：Fault Proof VM 在 OP Stack 中扮演的是一个**可裁决的执行环境**角色——它不参与正常 L2 出块流程，只在**出现争议时进入证明路径**。

在正常情况下，L2 由 `op-node` / `op-geth` 等常规节点软件执行，不需要走 FPVM 证明路径。出现争议时，参与者会在链下运行 FPVM 来生成执行轨迹和证明材料，L1 上的合约只在最终 `step()` 阶段验证某个单步状态转换。

VM 与 `op-program` 的关系：

```
op-program（Go 写的 Fault Proof Program）
     │ Go 交叉编译
     ▼
MIPS64 二进制文件（一大串机器指令）
     │ 在 Cannon VM 中模拟执行
     ▼
Cannon VM（Fault Proof VM）
  - 模拟 MIPS CPU，逐条执行指令
  - 每一步都有可验证的状态转换
  - 可通过 Pre-image Oracle 获取外部数据
```

VM 不直接理解"Rollup"或"区块链"——它只理解 CPU 指令。它模拟执行的是 `op-program` 编译后的 MIPS 二进制，`op-program` 才包含 Rollup 的业务逻辑（派生 L2 状态、验证 Output Root 等）。

**一句话概括**：VM 是 OP Stack 安全模型的**可验证执行底座**——它让 Optimistic Rollup 的"乐观"假设有了可执行的裁决机制。即使只有一个诚实的人愿意挑战，错误的状态也可以被压缩到一个 L1 可验证的单步计算上。

### Q3: VM 只是为了在 L1 上执行一个程序吗？这个程序的目的是什么？

**答**：这里有一个非常关键的澄清——**VM 不是在 L1 上执行整个程序**，而是在**链下执行整个程序**，仅在**链上验证单条指令**。

```
链下（Off-Chain）：完整执行 op-program（数百万条 MIPS 指令）
  → 生成每一步的状态根（Merkle Root）
  → 双方各自运行，各自得到状态根序列
  → 为链上的二分争议提供每一步的状态承诺和证明材料

链上（On-Chain / L1）：记录 dispute game 中的 claim / attack / defend，并在最终阶段执行和验证 1 条有争议的 MIPS 指令
  → 输入：Step K 的状态根 + 该位置的指令
  → 输出：Step K+1 的状态根
  → 验证：计算结果是否匹配主张方声称的状态根
```

`op-program` 的核心目的就一个：**用纯函数的方式，无状态地验证"L2 状态转换是否正确"**。

具体来说，它做三件事：
1. **Prologue（序言）**：加载输入（如 `l1_head`、agreed / disputed output、L2 block number、rollup 配置）
2. **Main Content（主内容）**：从 L1 Calldata/Blob 中派生 L2 交易，逐笔执行，得到 L2 状态
3. **Epilogue（尾声）**：计算 Output Root，与 Proposer 声称的值对比，输出 `exit(0)`（正确）或 `exit(1)`（错误）

**最简概括**：程序 = 一个**可验证的 L2 状态计算器**；VM = 让这个计算器的每一步都可被**追踪和挑战**的沙盒环境。两者配合，使得"在 L1 上直接证明 L2 状态错误"变得可行。

### Q4: 为什么非要让链上验证 1 条指令？不能在链下完成吗？

**答**：因为链下没有"终极裁判"——如果争议全程在链下解决：

```
Challenger: "Proposer 在某一步的指令执行错了！"
Proposer:   "不，我没错，是你在撒谎。"
Challenger: "不，你才撒谎。"
Proposer:   "你撒谎。"
... 永无止境 ...
```

两个人都声称对方错了，**谁来当裁判？** 这正是区块链存在的根本原因。

L1（以太坊主网）具备链下方案无法同时具备的三种属性：

| 属性 | 含义 |
|------|------|
| **中立性（Neutrality）** | 没有中心化机构控制，任何人都无法单方面修改规则 |
| **强制执行（Enforceability）** | 裁决结果自动执行：没收保证金、结算状态，不需要"请对方配合" |
| **不可篡改（Immutability）** | 一旦裁决，结果永久记录，无人能翻转 |

**为什么只验证 1 条而不是更多？** 这是一个成本优化问题：

| 做法 | L1 Gas 成本 | 可行性 |
|------|-------------|--------|
| 在 L1 上运行整个 op-program | 💸💸💸💸💸 天文数字 | ❌ 不可行 |
| 在 L1 上验证 100 条指令 | 💸💸 很贵 | ⚠️ 理论上可做，但没有必要 |
| 在 L1 上验证 1 条指令 | 💸 便宜 | ✅ 最优方案 |

二分法精妙之处：

```
争议范围: 1,000,000 条指令
   ↓ 双方在 L1 dispute game 中提交中点状态承诺
争议范围:   500,000 条指令
   ↓ 继续提交更细粒度的 claim / attack / defend
   ↓ ...
   ↓ 经过约 20 轮链上承诺提交
   ↓
争议范围: 1 条指令  ← 只有这一步需要链上执行计算！
```

这里最后一句需要更精确：不是“只有最后一步需要上链”，而是**二分过程中的 claim 提交在链上，真正昂贵的计算执行只发生在最后一步**。L1 不重放整段程序，只记录双方承诺并在最终 `step()` 中裁决单步计算。

**一句话**：链下负责完整运行程序和准备轨迹证明材料，链上负责记录争议承诺并最终判定谁对谁错。链上虽然昂贵，但它是双方唯一都信任的、能强制执行裁决的中立平台。

### Q5: 凭什么认为只会有一个分歧点？

**答**：分歧点可以有很多个，但只需要找到**第一个**。

```
双方一致同意的起点：
  Step 0: 0xAAA  ← 双方都认可

Proposer 的轨迹：                  Challenger 的轨迹：
  Step 1: 0xBBB  ✓ 一致              Step 1: 0xBBB  ✓
  Step 2: 0xCCC  ✓ 一致              Step 2: 0xCCC  ✓
  Step 3: 0xDDD  ✓ 一致              Step 3: 0xDDD  ✓
  Step 4: 0xEEE  ✗ 分歧！            Step 4: 0xFFF  ✗ 分歧！
  Step 5: 0xGGG  ✗                   Step 5: 0xHHH  ✗
  Step 6: 0xIII  ✗                   Step 6: 0xJJJ  ✗
  ...                                ...

从 Step 4 开始，之后每一步都不一样——分歧点有 N-3 个，不是只有 1 个。
```

二分法的目标是找到**第一个出现分歧的位置**（Step K 和 Step K+1）：

- Step K 状态一致（双方同意）
- Step K+1 状态不一致（第一个分歧）
- 有争议的只有 Step K → Step K+1 这条指令

**因为 CPU 指令是确定性的**（相同输入 + 相同指令 = 相同输出），第一个分歧点一旦定责，后面所有的分歧都自动归因于该方——后面的状态是"从错误状态继续执行"产生的，不需要再争议。

```
类比：两个人抄同一本书
原书: "The quick brown fox jumps over the lazy dog"

抄写员 A: "The quick brown fox jumps over the lazy dog"  ← 正确
抄写员 B: "The quick brawn fox jumps over the lazy dog"  ← 第4个词写错了

为什么只检查"第一个写错的地方"就够了？
- 第一个错误点之前：两人写的一样，不用查
- 第一个错误点：判定是 B 写错了 brawn ≠ brown
- 第一个错误点之后：B 基于错误的词继续抄，后面写对写错都不重要了
```

**一句话**：分歧可能有很多，但找到第一个分歧点后，谁对谁错就已经判定了——剩下所有的分歧都是"输方从错误状态继续计算"的必然结果，无须再验证。这就是确定性执行的核心特性：**一步错，步步错**。
