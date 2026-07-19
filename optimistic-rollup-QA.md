# Optimistic Rollup 与 Optimism Fault Dispute Game 学习笔记

这份笔记整理 Optimistic Rollup 的基本模型，以及 Optimism Fault Dispute Game（FDG）的关键概念：L2 output root、anchor state、SPLIT_DEPTH、execution trace、challenge、step 和 L1 智能合约在其中的角色。

参考文档：[Optimism Fault Dispute Game](https://specs.optimism.io/fault-proof/stage-one/fault-dispute-game.html)

## 1. Rollup 想解决什么问题

以太坊 L1 安全性强，但直接在 L1 执行所有交易成本高、吞吐有限。

Rollup 的基本思路是：

```text
大量交易在 L2 执行
交易数据或数据承诺发布到 L1
L2 状态结果锚定到 L1
最终安全性依赖 L1
```

可以粗略理解为：

```text
L2 负责高频执行
L1 负责结算、安全和最终裁决
```

## 2. Optimistic Rollup 的核心假设

Optimistic Rollup 的 “optimistic” 意思是：默认相信提交的 L2 状态结果是正确的，但允许任何人在挑战期内提出挑战。

流程是：

```text
Proposer 提交 L2 output root 到 L1
  -> 系统进入挑战期
  -> 如果没人成功挑战，结果最终被接受
  -> 如果有人挑战并成功，错误结果被推翻
```

所以它的安全模型不是：

```text
L1 主动重放全部 L2 交易
```

而是：

```text
任何错误结果都可以被诚实挑战者发现并证明
```

这要求至少有一个诚实参与者在挑战期内监控并挑战错误状态。

## 3. L1 智能合约在 Rollup 中的角色

Rollup 不是只在 L2 跑一套链就够了。它还需要在以太坊 L1 上部署一组合约。

这些 L1 合约通常负责：

```text
接收 L2 批次数据或数据承诺
记录 L2 output root / state commitment
处理 L1 -> L2 存款
处理 L2 -> L1 提款
管理挑战期和 dispute game
托管桥接资产
分配保证金 / bond
控制升级和治理权限
```

在 Optimism/OP Stack 语境中，这些合约像是：

```text
结算层
桥
状态登记处
争议法庭
最终裁判
提款守门人
```

L1 合约不负责执行整个 L2 区块，但负责记录、管理和最终确认 L2 的状态声明。

## 4. 通过 L1 转账和通过 Rollup 转账的区别

### L1 直接转账

如果 A 在 Ethereum L1 给 B 转 ETH：

```text
A 构造 L1 交易
  -> A 签名
  -> 广播到以太坊网络
  -> 被打包进 L1 区块
  -> 所有 L1 节点执行
  -> A L1 余额减少
  -> B L1 余额增加
```

这是直接由以太坊 L1 共识和执行层确认的。

### 同一个 L2 内转账

如果 A 在 Optimism 上给 B 转 ETH：

```text
A 构造 L2 交易
  -> A 签名，chainId = Optimism
  -> 交易发送给 L2 sequencer
  -> sequencer 排序并放进 L2 区块
  -> L2 节点执行交易
  -> A L2 余额减少
  -> B L2 余额增加
  -> 批次数据后续发布到 L1
  -> output root 后续提交到 L1
```

L2 转账通常更快、更便宜，但最终安全性要通过 L1 数据可用性和 Rollup 证明/挑战机制来获得。

### L1 到 L2

这叫 deposit：

```text
A 调用 L1 Bridge / Portal 合约
  -> L1 资产被锁定
  -> L1->L2 消息被记录
  -> L2 根据消息给目标地址释放或铸造对应资产
```

### L2 到 L1

这叫 withdrawal：

```text
A 在 L2 发起提款
  -> L2 记录提款消息
  -> L2 output root 提交到 L1
  -> 等待挑战期
  -> 在 L1 证明提款消息包含在已确认 L2 状态里
  -> L1 Bridge / Portal 合约释放资产
```

Optimistic Rollup 的 L2->L1 提款通常较慢，因为需要等待挑战期。

## 5. 同一个地址，不同链上余额不同

同一个私钥通常可以控制 L1 和多个 L2 上的同一个地址。

但每条链都有自己的状态。

例如：

```text
地址 0xABC...

Ethereum L1: 1 ETH
Optimism L2: 0.2 ETH
Arbitrum L2: 0.5 ETH
Base L2: 0 ETH
```

同一个地址不代表同一份余额。

## 6. L2 output root 是什么

L2 output root 可以理解为某个 L2 区块高度下，L2 状态的摘要承诺。

它不是完整状态本身，而是一个哈希承诺，通常承诺了 L2 在某个 block 后的关键状态。

可以简单理解为：

```text
L2 output root = L2 在某个区块高度的状态快照哈希
```

当 proposer 提交：

```text
block N 的 output root 是 X
```

它其实是在声称：

```text
L2 执行到 block N 后，状态就是 X 所承诺的那个状态
```

## 7. 为什么 output root 可以被挑战

只要输入和规则确定，L2 状态转换结果就是确定的。

输入包括：

```text
前一个 L2 状态
L2 区块数据 / 交易
相关 L1 数据
链配置参数
协议执行规则
```

因此：

```text
同样输入 + 同样规则 = 唯一正确的 output root
```

如果 proposer 提交了错误的 output root，诚实挑战者本地重放后会得到不同结果，并可发起 dispute。

## 8. 如果没人 challenge 会怎样

在 Optimistic Rollup 中，如果错误 output root 被提交，但挑战期内没人成功挑战，它理论上会被接受。

这是 optimistic 模型的核心：

```text
L1 不主动完整执行 L2
L1 默认接受提交结果
挑战者负责在窗口期内指出错误
```

所以可以区分：

```text
提交到 L1：
只是记录 claim，不代表已经最终确认正确

最终化 / 可用于提款：
经过挑战期，且没有被成功挑战，才被系统当作有效状态
```

## 9. 恶意交易和错误状态转换不是一回事

Fault proof 检查的不是“交易是不是恶意”，而是：

```text
这个 L2 区块是否按照协议规则正确执行
```

例如一笔交易可能：

```text
调用失败
触发 revert
尝试攻击某个合约
消耗很多 gas
产生不友好的 MEV
```

这些可能是“恶意”或“不友好”的行为，但只要协议允许、执行正确，它就是合法状态转换。

真正会被 FDG 抓出来的是：

```text
按规则执行应该得到 output root A
proposer 却提交了 output root B
```

也就是执行结果违反协议规则。

## 10. Optimism Fault Dispute Game 是什么

FDG 用来验证某个 root claim 是否有效。

它通过交互式二分，把争议从大范围逐渐缩小到单步 VM 状态转换。

整体路径是：

```text
争议一个 L2 output root
  -> 在 output roots 之间二分
  -> 定位到某个 L2 block transition
  -> 对这个 block 的 FPVM execution trace 二分
  -> 定位到某一步 VM transition
  -> L1 上执行 step 验证
```

最终 L1 不需要执行整个 L2 区块，只需要验证最后那一步。

## 11. Claim 和 Move

Claim 是一个承诺，可能代表：

```text
某个 L2 output root
某个 FPVM 状态承诺
```

文档中说：

```text
A Move is a challenge against an existing claim and must include an alternate claim asserting a different trace.
```

意思是：

```text
Move 是对已有 claim 的挑战；
挑战者必须同时提交一个替代 claim，
表示自己认为正确的另一条状态执行路径或中间状态。
```

挑战者不能只说“你错了”。他必须说：

```text
我认为在这个位置，正确状态承诺应该是 Y，而不是你隐含的 X。
```

这样游戏才能继续二分。

## 12. Anchor State

Anchor state，也叫 anchor output root，是一个已经假定有效的历史 output root。

FDG 从 anchor state 开始，对 claimed output root 进行验证：

```text
anchor state -> claimed output root
```

最初的 anchor state 可以是 L2 genesis state。后续成功通过争议或最终化的 output root 也可能成为新的 anchor state，从而减少未来争议需要覆盖的历史窗口。

## 13. Game Tree、gindex 和图中节点 23

FDG 使用一棵 Game Tree 表示 claim 的位置。每个位置用 generalized index，也就是 `gindex` 表示。

图里的数字：

```text
1, 2, 3, ..., 23
```

是 Game Tree 的 position / gindex，不是区块号，也不是 anchor state。

例如图中高亮的 `23` 表示某个 claim 在 Game Tree 中的位置。它可以对应某个 L2 output root claim，并进一步指向一个 execution trace subgame。

它不是 anchor state。Anchor state 是整个争议窗口的历史起点。

## 14. 为什么需要 SPLIT_DEPTH 和 MAX_GAME_DEPTH

文档中有两个重要深度：

```text
SPLIT_DEPTH
MAX_GAME_DEPTH
```

它们对应两个不同粒度的争议。

### SPLIT_DEPTH

`SPLIT_DEPTH` 是从 output root 级别切换到 execution trace 级别的边界。

在此之前，争议在问：

```text
从 anchor state 到 claimed output root 之间，
哪一个 L2 block 的 output root 开始不一致？
```

当争议缩小到某个相邻区块转换：

```text
block n output root -> block n+1 output root
```

就进入更细粒度的 block 内执行争议。

### MAX_GAME_DEPTH

`MAX_GAME_DEPTH` 是整棵游戏树最大深度。

到这个深度时，claim 已经对应 FPVM execution trace 中的具体位置。此时不能再继续二分，只能调用 `step`，在 L1 上验证单步 VM 状态转换。

可以总结为：

```text
anchor state
  -> output root bisection
  -> SPLIT_DEPTH：定位到单个 L2 block transition
  -> execution trace bisection
  -> MAX_GAME_DEPTH：定位到单步 VM transition
  -> step：链上验证
```

## 15. Execution Trace 是什么

Execution trace 是 FPVM 执行过程中的状态序列。

例如：

```text
S0 -> S1 -> S2 -> S3 -> ... -> Sn
```

每一步都是 VM 执行一小步后的状态。

当争议已经缩小到某个 L2 block transition：

```text
block n -> block n+1
```

系统会把执行这个 block transition 的过程放到 FPVM 中。FPVM 读取区块数据、交易数据、L1 数据、状态 preimage 等，按 OP Stack 规则执行，并产生一串中间状态。

FDG 后半段二分的对象就是这些 FPVM 中间状态。

### 15.1 Cannon / op-program / op-challenger 在整个流程里的位置

在 OP Stack 里，Cannon 不是日常出块时运行的主执行引擎。正常路径是：

```text
sequencer 排序交易
  -> op-batcher 把 batch 数据提交到 L1
  -> op-node 从 L1 数据推导 L2 链
  -> op-geth 执行 L2 交易
  -> op-proposer 提交 L2 output root
```

这条路径里，真正执行交易的是 `op-geth`，负责 L1 到 L2 派生的是 `op-node`。Cannon 只在 output root 被争议时进入关键路径：

```text
proposer 提交 L2 output root
  -> challenger 发起 dispute
  -> FDG 二分争议
  -> 定位到某个 L2 block transition
  -> 再定位到某一步 FPVM transition
  -> L1 合约用 Cannon 的单步规则执行 step 裁决
```

几个概念的关系是：

```text
op-program：
Fault Proof Program 的 OP Stack 实现。它负责读取 L1 数据、batch 数据、状态 preimage 等，
按 OP Stack 规则重新推导并执行 L2，最后计算目标 output root。

Cannon / MTCannon：
Fault Proof VM。它不懂 Rollup 业务语义，只负责把 op-program 放进 MIPS64 VM 中执行，
并把执行 trace 拆成 L1 可验证的单步状态转换。

op-challenger：
参与 dispute 的链下程序。它监控 dispute game，本地运行 trace provider，例如 Cannon，
生成中间状态、Merkle proof 和最终 step 所需 witness。
```

因此，Cannon 与“区块中交易是否合规”的关系是间接的：

```text
交易合规性
  -> 体现在 OP Stack derivation + execution 规则里
  -> op-program 重新执行这些规则
  -> Cannon 证明 op-program 的执行 trace 没有被伪造
  -> L1 判断 claimed output root 是否正确
```

L1 最后不会直接检查“某笔交易余额是否足够”或“某个 EVM opcode 是否执行正确”。它只验证一条 MIPS 指令级别的 VM 状态转换。真正理解交易、EVM、deposit、batch、output root 的，是 `op-program` 中实现或复用的 OP Stack 逻辑。

### 15.2 absolutePrestate 与 op-program 的信任边界

`op-program` 的完整代码通常不直接存放在 L1 上。链上 dispute game 认的是一个 `absolutePrestate`：

```text
absolutePrestate = hash(Cannon VM 的初始状态)
```

这个初始状态承诺了：

```text
op-program 的 MIPS64 程序代码
初始内存
初始寄存器
运行时布局
其他 VM 初始状态
```

不同 dispute game 可以从同一个 `absolutePrestate` 开始，是因为它固定的是“同一个程序和同一台 VM 的初始状态”，不是固定某个具体区块。每个 game 要验证的区块、claim 和相关数据不同，这些差异通过 game 参数和 `PreimageOracle` 进入程序：

```text
同一个 absolutePrestate
  + game A 的 root claim / L2 block number / oracle 数据
  -> trace A

同一个 absolutePrestate
  + game B 的 root claim / L2 block number / oracle 数据
  -> trace B
```

这类似同一个 `sha256` 程序处理不同文件会得到不同输出。

链上通常保存的不是完整 prestate 文件，而是 `absolutePrestate` 的 hash / claim 值。完整 prestate 文件在链下分发，`op-challenger` 或 Cannon 本地用它恢复初始 VM：

```text
链上：
absolutePrestate hash

链下：
<absolutePrestate hash>.bin.gz
```

因此，每个 dispute game 逻辑上都从“已经包含 op-program 的初始 VM 内存承诺”开始；但这不是每次都把完整 `op-program` 上传到 L1，也不是链上重新加载程序。链下参与者从 prestate 文件恢复 VM，链上只认起始 hash，并在最后一步验证少量 Merkle witness。

L1 并不检查挑战者本地电脑上的 `op-program` 文件有没有被篡改。L1 只约束：

```text
dispute trace 必须从链上指定的 absolutePrestate 出发；
后续每一步必须符合 Cannon 的 VM 状态转换规则。
```

如果挑战者改了本地 `op-program`，它要么无法匹配链上的起始状态，要么会在后续 trace 中产生某一步非法状态转换。FDG 最终会二分到这一步，再由 L1 的 `step` 验证裁决。

不过需要注意一个边界：

```text
Cannon / FDG 能保证：
给定 absolutePrestate 后，这个程序是否被正确执行。

Cannon / FDG 不能保证：
这个 op-program 本身一定正确、合理、无 bug、无恶意。
```

`op-program` 本身的合理性来自 OP Stack 规范、开源实现、可复现构建、测试审计、治理批准的版本、链上配置的 `absolutePrestate`，以及 challenger / 节点运营者的独立验证。也就是说，fault proof 解决的是“执行可验证性”，不是“治理完全去中心化”。OP Stack 的安全模型是 trust-minimized，而不是完全没有治理信任假设。

### 15.3 为什么 FPVM 内存需要 Merkle tree

FDG 里的 claim 通常只提交 VM state hash，而不是完整 VM 内存。VM state 里包含 `memRoot`，它是整片 VM 内存的 Merkle root。

当 L1 最后验证一步时，它需要确认：

```text
pc 指向的位置里，真的是这条 MIPS 指令吗？
load 读出来的值，真的是 pre-state 内存里的值吗？
store 写入后得到的新 memRoot，真的是从旧 memRoot 正确更新来的值吗？
```

如果没有 Merkle proof，恶意参与者可以随口声称自己在某个地址读到了某个值，L1 没有完整内存，无法判断真假。

Merkle memory 的作用是让单步 witness 可以自证：

```text
pre-state memRoot
  + 被访问内存叶子
  + Merkle path
  -> L1 验证该 leaf 属于旧内存
  -> L1 执行一条 VM 指令
  -> L1 重新计算被修改路径
  -> 得到 post-state memRoot
```

是的，每次内存写入后，确实需要从叶子一路更新 hash 到 root。但这个成本是可控的：树深固定，一条指令通常只访问很少的内存；链下可以维护完整内存结构，链上只验证少量 Merkle path。这样换来的是 L1 不需要重放整段程序，也不需要保存完整 VM 内存。

FPVM 内存里存的是被模拟 MIPS64 程序运行时看到的内容，例如：

```text
op-program 的 MIPS 指令代码
只读数据和全局数据
堆
栈
运行时数据
preimage 交互缓冲
```

但并不是所有 VM 状态都在内存里。寄存器、pc / nextPC、线程状态、退出码、`preimageKey`、`preimageOffset`、`memRoot` 等属于 VM state 或 thread state，而不是普通内存内容。

## 16. 一个区块内为什么不是对交易 hash 二分

一个 L2 block 里可能包含交易列表、交易 hash、区块数据等。

但 FDG 不是直接对交易 hash 二分，而是对执行该 block transition 的 FPVM 状态轨迹二分。

关系是：

```text
L2 block data / txs / L1 data / state preimages
        |
        v
FPVM 中运行的 L2 derivation + execution 程序
        |
        v
S0 -> S1 -> S2 -> ... -> Sn
        |
        v
最终产生 block n+1 的 output root
```

交易数据是 FPVM 执行时读取的输入。真正被二分的是 VM 每一步执行后的状态承诺。

## 17. 为什么错误状态转换会被发现

因为 FPVM 是确定性的。

如果双方对同一个 pre-state 和输入数据执行同一套规则，正确的 trace 应该唯一。

如果 proposer 作恶：

```text
本该扣 Alice 10 ETH，却没扣
本该让交易 revert，却当成成功
本该写 storage slot A，却写了 B
本该处理某条 L1 deposit，却漏处理
本该得到 output root A，却提交了 B
```

诚实挑战者本地执行会得到另一条 trace。

FDG 通过二分找到第一处分歧：

```text
双方同意 Si
proposer 声称下一步是 Si+1_wrong
challenger 认为正确下一步是 Si+1_correct
```

最后 L1 执行：

```text
VM(Si, proof) = ?
```

如果链上算出的结果不是 proposer 承诺的状态，proposer 的 claim 就被反驳。

## 18. Attack、Defend 和 Step

FDG 中玩家可以通过 Move 挑战已有 claim。Move 分两种：

```text
Attack：
表示不同意某个 claim，提交一个攻击位置的新 claim。

Defend：
表示同意某个 claim 及其父 claim，但认为分歧在后半段，提交防守位置的新 claim。
```

每一次 move 都会把争议推进到更深层。

到 `MAX_GAME_DEPTH` 后，不能继续 move，只能 `step`。

`step` 是最终裁决动作：

```text
给定 pre-state
给定 proof
执行一步 VM
检查输出是否等于被 claim 的 post-state
```

## 19. PreimageOracle 的作用

FPVM 执行时可能需要外部数据，例如：

```text
L1 数据
L2 区块数据
交易数据
起始 output root
目标 output root
L2 block number
L2 chain id
状态访问所需 preimage
```

这些数据不一定全部直接放在 claim 中，而是通过 `PreimageOracle` 提供。

可以理解为：

```text
FPVM 执行程序时，如果需要某段外部数据，
就按 key 从 PreimageOracle 中读取。
```

## 20. 如果没有人 dispute，错误区块会被 L1 接受吗

在 Optimistic Rollup 模型下，答案是：可能会。

更精确地说：

```text
错误 output root 可以被提交到 L1
L1 不会立刻完整重放 L2 检查
如果挑战期内没人成功挑战
它会被视为有效并最终化
```

这就是为什么 optimistic rollup 需要诚实挑战者假设。

## 21. 其他 Rollup 也需要自己的智能合约吗

需要。

不同 Rollup 都需要在 L1 上部署自己的合约，用来处理：

```text
状态承诺
批次数据
桥接资产
提款
挑战或证明验证
治理和升级
```

区别在于证明方式：

```text
Optimistic Rollup：
默认接受结果，靠挑战期和 fraud proof 纠错。

ZK Rollup：
提交状态更新时附带 validity proof，L1 verifier 合约验证证明后接受。
```

所以 optimistic rollup 的重点是 dispute / fraud proof 合约；ZK rollup 的重点是 proof verifier 合约。

## 22. 一句话总结

Optimistic Rollup 把大量执行放在 L2，把状态声明和争议裁决放在 L1。L2 output root 是某个 L2 高度状态的承诺；如果它错了，诚实挑战者可以通过 FDG 把争议从 output root 二分到单个区块，再从区块执行二分到单步 FPVM 状态转换，最后由 L1 合约执行一步验证来裁决。L1 不主动执行全部 L2，但提供最终结算、挑战和资产安全。
