# 公链 Rollup 技术调研整理

## 1. Rollup 基本概念

Rollup 是 Ethereum 等公链生态中常见的 Layer 2 扩容方案。它的核心思想是：将大量交易放到 L2 链下执行，再把批次数据、状态承诺或证明提交到 L1，由 L1 提供结算、安全锚定和争议裁决。

可以把 Rollup 理解为：

```text
L2 负责执行大量交易
L1 负责数据发布、证明验证、资产桥和最终结算
```

Rollup 的目标是降低交易成本、提高吞吐量，同时尽量继承 L1 的安全性。

## 2. 当前公链 Rollup 的主要方案

### 2.1 Optimistic Rollup

Optimistic Rollup 的核心逻辑是“默认交易批次正确”。L2 在链下执行交易，并把交易数据、状态根等信息提交到 L1。如果有人发现错误，可以在挑战期内提交 fraud proof 或 fault proof。

典型代表：

- Arbitrum
- OP Mainnet
- Base

基本流程：

```text
用户提交交易
 -> L2 sequencer 排序
 -> L2 执行交易
 -> 批量提交数据和状态承诺到 L1
 -> 默认接受结果
 -> 如有错误，挑战者提交证明
```

优点：

- EVM 兼容性较好；
- 工程成熟度较高；
- 对现有 Solidity 应用迁移友好；
- 不需要每个批次都生成复杂 ZK proof。

缺点：

- 通常有挑战期；
- 提现到 L1 可能较慢；
- 需要 watcher / challenger 持续监控；
- 早期阶段 sequencer 可能比较中心化。

参考：

- https://ethereum.org/en/developers/docs/scaling/optimistic-rollups/

### 2.2 ZK Rollup / Validity Rollup

ZK Rollup 更准确地说是 validity rollup。它将交易放到链下执行，然后生成一个有效性证明，提交到 L1 验证。L1 不需要重新执行所有交易，只需要验证 proof。

典型代表：

- Starknet
- zkSync Era
- Scroll
- Linea
- Polygon zkEVM

基本流程：

```text
用户提交交易
 -> L2 sequencer 排序
 -> L2 执行交易
 -> prover 生成 validity proof
 -> L1 verifier 合约验证 proof
 -> L1 接受新的 L2 state root
```

优点：

- 不依赖传统挑战期；
- L1 可以通过 proof 验证 L2 状态转换正确；
- 最终性通常比 Optimistic Rollup 更快；
- 适合批量验证大量状态变更。

缺点：

- prover 系统复杂；
- zkVM / zkEVM 工程难度高；
- 证明生成成本高；
- EVM 等价性和证明效率之间存在取舍。

参考：

- https://ethereum.org/en/developers/docs/scaling/zk-rollups/

### 2.3 Validium / Volition / Alt-DA Rollup

Validium 也使用有效性证明，但交易数据不完整发布到 Ethereum L1，而是放到链下数据可用性层、数据委员会或其他 DA 系统中。

基本逻辑：

```text
L2 执行交易
 -> 生成 validity proof
 -> L1 验证 proof
 -> 交易数据由链下 DA 层保存
```

优点：

- 成本更低；
- 吞吐量更高；
- 数据不必全部公开在 L1。

缺点：

- 数据可用性信任假设更强；
- 如果链下 DA 层扣留数据，用户可能无法重建状态；
- 安全性不如标准 Rollup 直接。

参考：

- https://ethereum.org/en/developers/docs/scaling/validium/

### 2.4 Rollup Stack / App-specific Rollup

现在很多 Rollup 项目不只运行一条链，而是提供“发链框架”。这些框架把执行、排序、证明、桥接、数据可用性等模块拆开，使开发者可以构建专用 L2 或应用链。

代表：

- OP Stack
- Arbitrum Orbit
- ZK Stack
- Polygon CDK

OP Stack 代表了 Optimistic Rollup 的模块化路线。ZK Stack 和 Polygon CDK 则更偏向 ZK / Validium / 多链互操作方向。

参考：

- https://docs.optimism.io/op-stack/protocol/overview
- https://docs.zksync.io/zk-stack
- https://docs.polygon.technology/chain-development/cdk

### 2.5 排序器方案

Rollup 一般需要 sequencer 对交易排序。当前多数 L2 仍然依赖项目方或少数实体运行 sequencer。

常见排序方式：

- 中心化 sequencer：用户体验好，但存在审查和停机风险；
- 去中心化 sequencer：多个节点共同排序，去中心化程度更高；
- based rollup：排序权交给 L1 proposer / builder，继承 L1 的活性和抗审查性。

参考：

- https://ethresear.ch/t/based-rollups-superpowers-from-l1-sequencing/15016

## 3. Ethereum L2 是否由不同组织维护

Ethereum 的 L2 区块链通常不是由 Ethereum 基金会统一维护，而是由不同团队、公司、DAO 或生态组织独立建设和运营。

示例：

| L2 项目 | 主要维护方 |
| --- | --- |
| Arbitrum | Offchain Labs、Arbitrum DAO、验证者生态 |
| OP Mainnet | Optimism Collective、OP Labs、生态节点 |
| Base | Coinbase、Base 团队、OP Stack 生态 |
| zkSync Era | Matter Labs、ZKsync 生态 |
| Starknet | StarkWare、Starknet Foundation、生态开发者 |
| Scroll | Scroll 团队和生态 |
| Linea | Consensys / Linea 团队 |

L2 自己负责：

- 排序交易；
- 构造 L2 区块；
- 批量提交交易数据；
- 维护桥合约；
- 运行证明系统或挑战系统；
- 升级自身协议和客户端。

Ethereum L1 提供：

- 结算层；
- 数据发布层；
- 资产桥接基础；
- proof 验证；
- 争议裁决；
- 安全锚定。

因此，Ethereum L2 可以理解为：

```text
Ethereum L1 = 公共安全层 / 数据发布层 / 结算层
不同 L2 = 各组织建设和维护的扩容链
```

需要注意的是，“继承 Ethereum 安全性”不是绝对的。不同 L2 的去中心化程度、sequencer 控制权、合约升级权限、证明系统成熟度差别很大。

## 4. L2 与 L1 上存储的数据区别

L2 和 L1 上存的数据有明显区别。

### 4.1 L1 上通常保存的数据

L1 不会保存完整 L2 状态。它主要保存：

| L1 数据 | 作用 |
| --- | --- |
| 交易批次数据 / calldata / blob data | 让其他节点可以重放或恢复 L2 状态 |
| 状态根 / output root / state root | 承诺某一时刻的 L2 状态 |
| 存款、提现、跨链消息 | 连接 L1 和 L2 |
| fraud proof / validity proof 相关数据 | 验证或挑战 L2 状态 |
| Rollup 合约状态 | 管理桥、挑战、输出根和最终化 |

L1 更像是：

```text
L2 的数据公告栏 + 结算法院 + 资产桥
```

### 4.2 L2 上通常保存的数据

L2 是一条完整执行链，保存：

| L2 数据 | 作用 |
| --- | --- |
| L2 区块 | L2 自己的区块历史 |
| L2 交易 | 用户在 L2 上提交的交易 |
| 账户余额 | L2 内部账户资产状态 |
| 智能合约代码 | 部署在 L2 的合约 |
| 合约存储 | 合约变量和业务数据 |
| receipts / logs | 事件和交易结果 |
| 当前 state trie / state tree | 当前世界状态 |

### 4.3 标准 Rollup 与 Validium 的数据差异

标准 Rollup：

```text
L2 执行交易
 -> L2 把交易数据或状态差异发布到 L1
 -> 任何人可以从 L1 数据重建 L2 状态
```

Validium：

```text
L2 执行交易
 -> L2 把状态根和证明提交到 L1
 -> 交易数据放在链下 DA 层或数据委员会
```

核心区别：

```text
L2 存业务状态和执行结果
L1 存可验证承诺、批次数据、证明和跨链结算信息
```

## 5. Arbitrum

Arbitrum 是 Optimistic Rollup 的代表项目之一，核心架构是 Nitro。

Arbitrum 的交易流程：

```text
用户交易
 -> Sequencer 排序
 -> 批量压缩
 -> 发布到 Ethereum
 -> Arbitrum 节点按规则执行
 -> 状态断言提交到 L1
 -> 如有错误，进入挑战 / 争议流程
```

Arbitrum Nitro 不是简单复制一个 EVM。它使用 Geth 处理 EVM 兼容执行，同时通过 ArbOS 处理 L2 特有逻辑，例如：

- 跨链消息；
- 手续费；
- 存取款；
- gas 定价；
- L1/L2 通信。

Arbitrum 的争议机制重点是 BoLD。争议发生时，系统不会让 Ethereum 重新执行所有交易，而是把争议逐步缩小到单个执行步骤，最后由 L1 合约验证 one-step proof。

Arbitrum 的特点：

- Optimistic Rollup 路线；
- Nitro 架构成熟；
- EVM 兼容度较高；
- 使用 ArbOS 处理 L2 特有逻辑；
- 支持 fraud proof / challenge 机制；
- sequencer 提供快速确认；
- 支持通过 Delayed Inbox 绕过 sequencer 审查。

参考：

- https://docs.arbitrum.io/how-arbitrum-works/inside-arbitrum-nitro
- https://www.bankless.com/the-essential-guide-to-arbitrum

## 6. OP Mainnet

OP Mainnet 是 Optimism 生态的主网 L2，属于 Optimistic Rollup，底层基于 OP Stack / Bedrock。

OP Mainnet 的基本流程：

```text
Sequencer 收集交易并出 L2 block
 -> L2 block 数据压缩后提交到 Ethereum
 -> 节点从 Ethereum 数据中推导 L2 状态
 -> 状态承诺进入挑战窗口
 -> 错误可被 fault proof 挑战
```

OP Stack 将 Rollup 拆成多个模块：

- op-geth：执行客户端；
- op-node：从 L1 数据推导 L2 链；
- batcher：把 L2 数据批量提交到 L1；
- proposer：提交 L2 output root；
- challenger / fault proof 系统：处理错误状态挑战；
- L1/L2 bridge 合约：处理跨层消息和资产。

OP Mainnet 的特点：

- Optimistic Rollup 路线；
- 模块化程度高；
- 强调 OP Stack 复用；
- 是 Optimism Superchain 生态的重要组成部分；
- 对 EVM 和 Solidity 迁移友好；
- 提现到 Ethereum 需要等待挑战期。

参考：

- https://docs.optimism.io/op-stack/protocol/overview
- https://specs.optimism.io/protocol/overview.html

## 7. Base

Base 是 Coinbase 孵化的 Ethereum L2。它最初基于 OP Stack 构建，当前按照 Base Chain specification 演进。

Base 和 OP Mainnet 技术亲缘较近，但定位不同。OP Mainnet 更像 Optimism / Superchain 的标准样板链，Base 更偏向 Coinbase 生态中的交易、支付、消费级应用和大规模用户入口。

Base 的最终性可以分成多个阶段：

```text
~200ms: Flashblock inclusion，预确认
~2s: L2 block inclusion
~2m: L1 batch inclusion，批次提交到 Ethereum
~20m: L1 batch finality
```

普通 Base 内部交易不需要等待 7 天挑战期。标准 withdrawal 从 Base 提现到 Ethereum 时，仍需要遵守 optimistic rollup 的挑战期。

Base 的特点：

- Coinbase 孵化；
- 基于 OP Stack 路线；
- 强调低成本和用户体验；
- 有 Flashblocks 等低延迟确认机制；
- 和 Coinbase 产品生态联系紧密；
- 标准提现仍受挑战期影响。

参考：

- https://docs.base.org/base-chain/overview
- https://docs.base.org/base-chain/network-information/transaction-ordering
- https://docs.base.org/base-chain/network-information/transaction-finality
- https://www.base.org/

## 8. Arbitrum、OP Mainnet、Base 对比

| 方案 | 技术路线 | 核心栈 | 主要特点 |
| --- | --- | --- | --- |
| Arbitrum | Optimistic Rollup | Nitro + ArbOS + Geth + BoLD | 性能优化强，有独立 ArbOS 和 BoLD 争议协议 |
| OP Mainnet | Optimistic Rollup | OP Stack / Bedrock | 标准化、模块化，目标是 Superchain 多链生态 |
| Base | OP Stack 路线的 Ethereum L2 | Base Chain spec，原始基于 OP Stack | Coinbase 孵化，偏支付、交易、消费级应用 |

三者都属于 optimistic 路线，但侧重点不同：

- Arbitrum 更强调自有执行架构和争议协议；
- OP Mainnet 更强调标准化 Rollup 栈；
- Base 更强调应用化、低延迟和 Coinbase 生态入口。

## 9. ZK Rollup 详细说明

ZK Rollup 的核心是 validity proof。它不是把所有交易都放到 L1 执行，而是在 L2 执行交易，然后向 L1 证明执行结果正确。

完整流程：

```text
用户交易
 -> L2 sequencer 排序
 -> L2 执行交易
 -> 生成新区块 / batch
 -> prover 根据执行轨迹生成 validity proof
 -> L1 verifier 合约验证 proof
 -> L1 接受新的 L2 state root
```

ZK Rollup 的关键组件：

| 组件 | 作用 |
| --- | --- |
| Sequencer | 收集和排序交易，构造 L2 区块 |
| Executor / VM | 执行交易，得到新状态 |
| State tree | 记录账户、合约和存储状态 |
| Prover | 根据执行轨迹生成证明 |
| Verifier contract | 部署在 L1，验证 proof |
| Rollup contract | 管理 batch、state root、bridge 和 withdrawal |
| DA 机制 | 保证其他节点能重建 L2 状态 |

需要注意：ZK Rollup 中的 ZK 通常不代表交易隐私。大多数公链 ZK Rollup 的交易数据仍然公开，ZK proof 的主要作用是证明执行正确性。

## 10. Starknet

Starknet 是 STARK-based validity rollup。它不是简单复制 EVM，而是使用 Cairo VM 和 Cairo 语言。

Starknet 的流程：

```text
用户提交交易
 -> Starknet sequencer 排序出块
 -> Cairo / Starknet OS 执行交易
 -> SHARP 聚合证明
 -> 证明提交到 Ethereum 验证
 -> 状态在 L1 上最终确认
```

Starknet 的特点：

- 使用 STARK 证明；
- 不依赖 trusted setup；
- 使用 Cairo VM；
- 原生账户抽象；
- 证明友好；
- 与 Solidity / EVM 生态不是完全原生兼容；
- 更像为 ZK 重新设计的智能合约链。

参考：

- https://docs.starknet.io/learn/protocol/intro

## 11. zkSync Era

zkSync Era 是 Matter Labs 推出的 ZK Rollup，目标是提供接近 Ethereum 的开发体验，同时优化 ZK 证明效率。

zkSync Era 的底层不是完全相同的 EVM，而是 EraVM。EraVM 是为 rollup 和证明效率优化的虚拟机。它主要运行 EraVM 原生字节码，同时支持 EVM bytecode interpreter。

zkSync Era 的关键组件：

- node implementation：接收交易、维护链下状态、聚合 batch；
- ZK circuits：定义可被证明的计算；
- prover：生成链下交易正确执行的证明；
- smart contracts：L1 上验证 proof，处理存取款和跨层消息。

zkSync Era 在数据可用性上强调 state diff，即提交状态变化，而不是逐笔完整交易数据，以降低 L1 数据成本。

zkSync Era 的特点：

- ZK Rollup 路线；
- 使用 EraVM；
- 偏 Ethereum 开发体验；
- 强调账户抽象、paymaster 和用户体验；
- 使用 state diff 和压缩优化成本；
- ZK Stack 支持部署多条 ZK 链。

参考：

- https://docs.zksync.io/zksync-protocol
- https://docs.zksync.io/zksync-protocol/era-vm/vm
- https://docs.zksync.io/zksync-protocol/rollup
- https://docs.zksync.io/zksync-protocol/rollup/data-availability

## 12. Scroll

Scroll 是 zkEVM Rollup，目标是尽量接近 Ethereum EVM 语义，让开发者可以较容易迁移 Solidity 合约。

Scroll 的整体架构可以分成三层：

| 层 | 作用 |
| --- | --- |
| Settlement Layer | Ethereum，负责数据可用性、proof 验证、桥接和最终性 |
| Sequencing Layer | 执行交易，生成 L2 block，提交 batch |
| Proving Layer | prover 池生成 ZK proof，coordinator 分配证明任务 |

Scroll 的 rollup 流程：

```text
交易进入 L2 sequencer
 -> sequencer 执行并出块
 -> rollup node 组成 chunk / batch
 -> relayer 提交 commit transaction 到 L1
 -> coordinator 分配证明任务
 -> prover 生成 chunk proof / batch proof
 -> relayer 提交 finalize transaction
 -> L1 verifier 验证 proof
```

Scroll 的特点：

- zkEVM Rollup；
- 强调 EVM 兼容；
- 使用 Ethereum 风格的状态承诺；
- commit / prove / finalize 流程清晰；
- 对 Solidity 合约迁移友好；
- 证明系统工程化程度较高。

参考：

- https://docs.scroll.io/en/technology
- https://docs.scroll.io/en/technology/zkevm/intro-to-zkevm
- https://docs.scroll.io/en/technology/chain/rollup/

## 13. Linea

Linea 是 Consensys 推出的 public zkEVM network。其技术栈称为 Lineth。

Linea / Lineth 的协议流程：

```text
执行交易
 -> 生成 batch 和 trace blocks
 -> 生成 validity proofs
 -> 提交 DA、验证和最终化信息到 finalization layer
```

Linea 的主要组件：

- Maru：共识客户端；
- Linea Besu：执行客户端，维护 EVM 状态；
- Sequencer：排序并构造区块；
- Coordinator：编排 batch、proof generation 和提交；
- Prover：生成 ZK proof；
- State manager：维护适合证明和恢复的状态表示；
- Tracer：生成证明所需执行轨迹。

Linea prover 生成三类证明：

- execution proofs：证明 batch 内交易执行正确；
- compression proofs：证明 blob 压缩有效；
- aggregation proofs：聚合多个 proof。

Linea 的特点：

- Consensys 背景；
- public zkEVM network；
- 使用 Linea Besu 和 Lineth 技术栈；
- 支持 EIP-4844 blobs 作为数据可用性机制；
- 模块拆分清晰；
- 偏企业级工程栈。

参考：

- https://docs.linea.build/
- https://docs.linea.build/protocol/overview
- https://docs.linea.build/protocol/architecture
- https://docs.linea.build/protocol/architecture/prover

## 14. Polygon zkEVM

Polygon zkEVM 是 Polygon 早期推出的 EVM-compatible ZK L2，目标是通过 ZK 证明扩容 Ethereum，同时尽量兼容 EVM。

需要注意的是，截至 2026 年，Polygon 官方技术路线的重心已经明显转向 Polygon CDK 和 Agglayer，而不是单独围绕早期 Polygon zkEVM 主网叙事。

Polygon zkEVM 可以分两层理解：

1. 历史技术路线：EVM-compatible zkEVM，用 ZK 证明验证 L2 执行。
2. 当前 Polygon 路线：CDK + Agglayer + validium / private validium / sovereign chain。

Polygon zkEVM 的特点：

- 曾经是 zkEVM 代表项目之一；
- 强调 EVM 兼容；
- 后续路线逐渐转向 CDK 和 Agglayer；
- 当前更强调发链、互操作和多链聚合；
- CDK 支持 sovereign、validium、private validium 等部署模式。

参考：

- https://docs.polygon.technology/chain-development/cdk
- https://l2beat.com/scaling/projects/polygonzkevm

## 15. ZK Rollup 代表项目对比

| 项目 | 类型 | VM / 执行环境 | 证明路线 | 主要特点 |
| --- | --- | --- | --- | --- |
| Starknet | Validity Rollup | Cairo VM | STARK / SHARP | ZK 原生、证明友好、账户抽象强 |
| zkSync Era | ZK Rollup | EraVM + EVM interpreter | ZK circuits / prover | 用户体验好、状态差异压缩、多链 ZK Stack |
| Scroll | zkEVM Rollup | EVM / modified Geth / OpenVM prover | zkEVM proof | EVM 兼容强、工程流程清晰 |
| Linea | zkEVM Network | Linea Besu / Lineth | zk-SNARK / aggregation | Consensys 工程栈、模块化清楚 |
| Polygon zkEVM | 历史 zkEVM，当前转向 CDK / Agglayer | EVM-compatible / CDK 模式 | 当前更偏 Agglayer / pessimistic proof | 企业链、validium、互操作方向 |

## 16. 参考网址汇总

### Ethereum Rollup 基础

- https://ethereum.org/en/developers/docs/scaling/optimistic-rollups/
- https://ethereum.org/en/developers/docs/scaling/zk-rollups/
- https://ethereum.org/en/developers/docs/scaling/validium/
- https://ethereum.org/en/roadmap/danksharding/

### Optimistic Rollup 代表

- https://docs.arbitrum.io/how-arbitrum-works/inside-arbitrum-nitro
- https://www.bankless.com/the-essential-guide-to-arbitrum
- https://docs.optimism.io/op-stack/protocol/overview
- https://specs.optimism.io/protocol/overview.html
- https://docs.base.org/base-chain/overview
- https://docs.base.org/base-chain/network-information/transaction-ordering
- https://docs.base.org/base-chain/network-information/transaction-finality

### ZK Rollup 代表

- https://docs.starknet.io/learn/protocol/intro
- https://docs.zksync.io/zksync-protocol
- https://docs.zksync.io/zksync-protocol/era-vm/vm
- https://docs.zksync.io/zksync-protocol/rollup
- https://docs.zksync.io/zksync-protocol/rollup/data-availability
- https://docs.scroll.io/en/technology
- https://docs.scroll.io/en/technology/zkevm/intro-to-zkevm
- https://docs.scroll.io/en/technology/chain/rollup/
- https://docs.linea.build/
- https://docs.linea.build/protocol/overview
- https://docs.linea.build/protocol/architecture
- https://docs.linea.build/protocol/architecture/prover
- https://docs.polygon.technology/chain-development/cdk
- https://l2beat.com/scaling/projects/polygonzkevm

### 生态与风险分析

- https://l2beat.com/scaling/summary
- https://l2beat.com/scaling/risk
- https://ethresear.ch/t/based-rollups-superpowers-from-l1-sequencing/15016
- https://docs.celestia.org/learn/celestia-101/data-availability/
