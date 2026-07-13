# 原生 Rollup：来自 L1 执行的超能力（Native rollups—superpowers from L1 execution）

> **作者**：JustinDrake（正文明确声明功劳属于更广泛的以太坊研发社区，本文只是"把一个大想法拼成一套连贯设计"的尽力之作）｜**发表日期**：2025-01-20（讨论持续到 2025-07）｜**原文链接**：https://ethresear.ch/t/native-rollups-superpowers-from-l1-execution/21517
> **一句话结论**：给 L1 加一个 `EXECUTE` 预编译，把「验证一段 EVM 状态转移」这件事本身变成 L1 的原生能力，从而让 EVM 等价的 rollup 彻底扔掉欺诈证明博弈、SNARK 验证器和安全委员会——代价是 rollup 必须放弃它们今天赖以生存的那些"非标准 EVM"定制，而这正是整个讨论区吵得最凶的地方。

> **阅读定位**：本篇只做原始提案帖的精读 + 一手争论梳理。综述性内容（路线图背景、native rollup 与 L1 扩容的关系）见姊妹篇 `04-l2beat-native-rollups.md`。

---

## 提案的核心：EXECUTE 预编译

原帖分两部分：Part 1 讲预编译，Part 2 讲基于它构建的 native rollup。核心构造只有几行：

`EXECUTE` 接受四个输入 —— `pre_state_root`、`post_state_root`、`trace`、`gas_used`，当且仅当以下三条同时成立时返回 `true`：

1. `trace` 是一段格式合法的执行轨迹（例如"一批 L2 交易 + 对应的状态访问证明"）；
2. 从 `pre_state_root` 出发，对 `trace` 做**无状态执行（stateless execution）**，终点恰好是 `post_state_root`；
3. 这次无状态执行恰好消耗 `gas_used` 的 gas。

关键点在于「谁来保证这个 `true` 是对的」：不是某个链上验证器合约，而是**以太坊验证者本身**。验证者在处理这个区块时必须自己确认这次 `EXECUTE` 调用的正确性。于是 rollup 的状态转移函数不再是 rollup 自己实现的东西，而是 L1 共识规则的一部分。

原帖给的定义链条是：

- **native rollup（原生 rollup）**：用 `EXECUTE` 来验证用户交易批次的 EVM 状态转移的 rollup。可以理解为「可编程的执行分片」——把预编译包在一个**推导函数（derivation function）**里，用来处理 EVM 之外的系统逻辑：排序、跨链桥、强制包含、治理。
- **trustless rollup（无信任 rollup）**：完全继承以太坊安全性的 rollup。原帖直接下了一个很强的断言：想要真正做到这一点，某种形式的「EVM 自省（EVM introspection）」是**必要**的。

作者反复强调的一句话（引用）："With `EXECUTE` one can deploy minimal native and based rollups in just a few lines of Solidity code."

## 它要解决的痛点

原帖列了五条收益，每条对应一个今天 rollup 的真实伤口：

- **简洁性**：今天 EVM 等价的 optimistic / zk rollup，为了实现欺诈证明博弈或 SNARK 验证器，要写几千行代码；用了 `EXECUTE`，这几千行可以坍缩成一行。连带着不再需要证明网络、看门狗（watchtowers）、安全委员会这些配套设施。
- **安全性**：这是全文最尖锐的一句判断（引用）："Every optimistic and zk EVM rollup most likely has critical vulnerabilities today in their EVM state transition function." 正因为怕被人构造恶意区块打穿，rollup 才普遍**把中心化排序器当拐杖**（"centralised sequencing is often used as a crutch"）。有了原生执行，才敢安全地开放无许可排序。
- **EVM 等价性**：今天 rollup 想跟上 L1 的 EVM 规则，唯一办法是靠治理（安全委员会 / 治理代币）去镜像 L1 的硬分叉。治理既是攻击面，本身也意味着**严格来说它永远不是真正的 EVM 等价**。native rollup 可以随 L1 硬分叉同步升级，无需治理。
- **SNARK gas 成本**：链上验证 SNARK 很贵，所以很多 zk rollup 拉长结算间隔。`EXECUTE` 因为**不在链上验证证明**，反而可以把验证成本压低。
- **同步可组合性（synchronous composability）**：今天想和 L1 同步组合，需要同 slot 的实时证明（~100ms 级别），工程上极难；有了「延迟一个 slot 的状态根」，证明时延可以放宽到一整个 slot。（**注意：这一条在讨论区被打得最惨，见下文 cwgoes 部分。**）

## 技术细节

### gas 计量与定价

- 有一套 **EIP-1559 式**机制，对一个 L1 区块内所有 `EXECUTE` 调用的**累计** gas 计量和定价：设有累计上限 `EXECUTE_CUMULATIVE_GAS_LIMIT` 和累计目标 `EXECUTE_CUMULATIVE_GAS_TARGET`。等到 L1 EVM 本身可被验证者无状态验证时，这套累计限额可以与 L1 的 EIP-1559 合并。
- 调用成本 = 固定的 `EXECUTE_GAS_COST` + `gas_used * gas_price`。**即使返回 `false` 也要全额预付**（防止用垃圾 trace 白嫖验证者算力）。
- `trace` 必须指向以太坊上**可得的数据**：calldata、blob、状态、或内存。

### 两种执行路径

**(a) 重执行（re-execution）—— 过渡阶段**

如果 `EXECUTE_CUMULATIVE_GAS_LIMIT` 设得足够小，验证者可以直接朴素地重执行 `trace`。作者把它类比为 proto-danksharding 之于 full danksharding：先用笨办法上线，再演进。好处是重执行**不增加状态膨胀、不增加带宽负担**，且执行开销可以在 CPU 多核上并行。

一个重要约束：验证者必须**显式持有 trace 的副本**才能重执行，因此**不能**用指向 blob 的指针（blob 只是被 DAS 采样，不是被下载）。

但作者留了个口子：optimistic 型的 native rollup 仍然可以把数据发在 blob 里、只在欺诈证明博弈中回退到 calldata；而且它们的 gas limit 可以**远超** `EXECUTE_CUMULATIVE_GAS_LIMIT`——因为 `EXECUTE` 只需在争议裁决时对一小段 EVM 执行调用一次。

历史注脚：2017 年 Vitalik 提过一个几乎同构的「EVM inside EVM」预编译，叫 `EXECTX`（EIPs issue #726）。

**(b) SNARK 强制 —— 目标状态**

要把累计上限做大，就让验证者可选地去验 SNARK。这里假设了**延迟一个 slot 的执行（one-slot delayed execution）**：无效区块（或无效交易）被当作 no-op。由此得到整整一个 slot 的证明时间，同时避免了 MEV 驱动的"抢证明"竞赛（那会是中心化风险）。

激励设计（假设在 APS / 出块者-证明者分离、执行 slot 与共识 slot 交替的语境下）：

- 证明者只有在区块 `n` 的证明可得时，才为执行区块 `n+1` 投票（建议在 p2p 层把 `n+1` 区块和 `n` 的证明打包在一起 gossip）。
- 跳过证明的执行提议者会**错过自己的 slot**，损失手续费和 MEV；此外再加一笔固定罚金（例如 **1 ETH**），确保总是高于证明成本。
- 为了让轻客户端也能及时读到链头状态，额外依赖**利他少数证明者假设（altruistic-minority prover assumption）**：一个利他证明者就够了；大多数可以待机，只在 1 个 slot 内没等到证明时才顶上，最坏情况 2 slot 时延。
- 因此 `EXECUTE_CUMULATIVE_GAS_LIMIT` 必须设得足够低，让「利他证明者假设」可信。保守策略：单 slot 证明能在**高端 MacBook Pro 上跑完**；激进策略：瞄准一小簇 GPU，未来或许是商品化的 SNARK ASIC。

### 链下证明（offchain proofs）——最反直觉的一条设计

**共识层不内置（enshrine）任何证明系统或电路**，`EXECUTE` 甚至**不接受证明作为输入**。证明不上链，而是**在链下传播**；每个质押运营者自选自己信任的 zkEL 验证客户端——就像今天自选 EL 客户端一样。这个思路作者归功于 Vitalik（"enshrined zk-EVM" 笔记）。

好处四条：**多样性**（一个 zkEL 客户端的 bug 不该拖垮以太坊）、**中立性**（zkVM 市场竞争激烈，共识层不该钦点 Risc0 / Succinct 或其他任一家）、**简洁性**（共识层只需内置「状态访问证明的格式」，不需内置具体验证器实现）、**灵活性**（发现 bug 或优化，客户端自己升级即可，无需硬分叉）。

代价也讲得很实在：

- **证明者负载 + p2p 分裂**：没有唯一的规范证明，每个 zkEL 客户端（乃至每次版本升级、每次换 zkVM）都要一份不同的证明，会加重证明负担并分裂 gossip 网络。
- **少数派 zkEL 难激励**：理性的执行提议者只会生成"够到超级多数证明者"的那几种证明。缓解办法是社会性地鼓励运营者并行跑多个 zkEL（类似今天的 Vouch 运营者），`k`-of-`n` 配置还能额外对冲**可靠性 bug（soundness bug）**——即攻击者能为任意 `EXECUTE` 调用伪造证明的那种灾难。

对**实时结算**的 L2 还有三条硬性代价：

- **不能用替代 DA**：因为 `trace` 必须对 L1 验证者可得，实时结算的 L2 必须消耗 L1 DA，即必须是 rollup，不能是 validium。（延迟结算的 optimistic L2 不受此限。）
- **状态访问开销**：`trace` 必须可无状态执行，所以要带上读写到的状态树叶子，比典型 L2 区块多一点 DA 开销。
- **不能做状态差分（state diffing）**：因为要保证「给定 trace 就能无许可证明」，就不能只发状态差分。

### 与现有 rollup 的兼容性 / 命名

- **based 与 native 是正交的**：based 讲的是 L1 排序，native 讲的是 L1 执行。两者兼得的 rollup 被戏称为 "ultra sound rollup"。
- **native ≠ 执行分片**：执行分片是不可编程的（不能自定义治理、排序、gas 代币），且通常固定数量（64 / 1024 个）。native rollup 是"可编程的执行分片"。
- 旧称是 "enshrined rollup"；改叫 "native" 是为了表达「现存的 EVM 等价 rollup 有**升级成为** native 的选项」。"native" 一词由 Dan Robinson 和一位匿名 Lido 贡献者于 2022 年 11 月独立提出。

**批注**：命名那一节看着像文字游戏，其实是政治动作——"enshrined"（内置）听起来像 L1 要吞并 L2，"native"（原生）听起来像 L2 自愿升级。原帖花整整一节澄清，说明作者很清楚这个提案会被读成"以太坊基金会要收编 rollup"。讨论区后面果然就这么吵起来了。

## 讨论区的关键争论

**首先一个必须点明的事实：JustinDrake 在这个帖子里只发了 1 楼（原帖），全程 42 楼、跨度半年，他一次都没有回复。** 所以严格地说，本帖不存在"作者的回应"。为提案辩护、并给出补丁方案的主要是 **thegaram33**（Scroll 团队）。作者本人的进一步回应散落在别处：帖内被引用的 Native Rollups Call #0（视频）、以及原帖底部链接的一串后续研究帖（Native Rollup for 3SF、Fee structure for EXECUTE-precompile、Enshrined Native L2s and Stateless Block Building 等）。

### 争论一：现实中的 L2 根本不跑"纯 EVM"（最核心、最未解决的质疑）

**edfelten**（9 / 21 / 31 楼；**批注**：即 Ed Felten，Arbitrum 背后 Offchain Labs 的联合创始人——所以这是来自最大 optimistic rollup 阵营的正面回应）给出了全帖分量最重的反对意见，列了四类"生产环境 L2 与 L1 STF 的偏离"：

1. **跨链桥**：无信任的存取款必须由 L2 的状态转移函数支持；
2. **交易类型**：L2 有 L1 合约发起的、**没有签名**的交易类型，L1 EVM 没有；
3. **gas 记账**：L2 要为 DA 加收附加费，可变出块时间的 L2 还要按**时间戳**而非区块号来调价——这都得改 STF 才能做到无信任；
4. **专有预编译**：读 L1 状态、查 DA gas 价格等，而且这些预编译是在**其他交易执行过程中被内部调用**的，不只是顶层调用。

当 thegaram33（20 楼）提出「双 STF」折中方案——rollup 合约同时暴露 `submitEVMBlob()`（走 `EXECUTE`）和 `submitCustomBlob()`（走自定义验证器）——edfelten（21 楼）一击致命：

> "in practice vanilla EVM functionalities and extension functionalities need to be composable, so that a single transaction can use some of both... So in practice a chain needs to have a single STF."（合约可以任意嵌套调用彼此是核心原则，把 STF 一劈为二就破坏了它——一条链只能有一个 STF。）

thegaram33 自己也承认，双 STF 方案"当然算不上 trustless rollup，因为扩展逻辑仍然由 rollup 控制"。

后续（29 楼）thegaram33 提出补丁：一个 `L1ContextPrecompile`，把任意数据从 L1 的 rollup 合约传给 L2 EVM（对应他另一篇《Native Rollup Deposits by Passing L1 Context》），声称能覆盖 edfelten 除 gas 记账外的大部分清单。edfelten（31 楼）不买账："让 L2 看到 L1 状态"只解决了 L1→L2 消息传递，**远远不足以**覆盖今天生产 rollup 已有的定制范围。

**批注**：这是全帖最重要的悬案，到 7 月最后一楼都没被解决。它也解释了为什么后来的 native rollup 讨论（含 EIP-8079）不得不认真处理 derivation / 系统交易这一层。

### 争论二：设计取的是不是"次优点"（pepyakin、maxgillett、OdysLam）

**pepyakin**（15 楼）的论证很漂亮：提案为了去治理，把太多东西"锁死"了（trie 格式、gas 表、交易类型、推导函数）。他反问：既然治理是坏东西、L1 该接管，那为什么不把跨链桥和提款也一并接管？那样用户就不需要查 L2BEAT 了。他的结论是提案落在了**设计空间中的次优点**，应该二选一：

1. **管得更多**：往执行分片方向走（但保留自定义排序等今天才懂的东西）；
2. **管得更少**：放松 `EXECUTE` 的语义——不再执行一个带存储/交易的 EVM 环境，而是执行一个「吃输入、吐输出」的**通用程序**。这样失去 EVM 等价，但换回灵活性，且 rollup 开发者仍然不用自己写欺诈证明或 SNARK。他认为这条路应该拥抱 RISC-V，直接复用 Rust / reth 工具链。

**maxgillett**（18 楼）给了一个具体的技术化版本：给 `EXECUTE` 增加 `vm` 和 `state` 两个参数，各指向一个 L1 合约地址——`vm` 合约是自定义 VM 的**无状态解释器**（在 EVM 里模拟），`state` 合约描述状态如何存取。为空则退化为默认 L1 EVM + MPT。他还处理了模拟开销问题：可以给每种已部署的 VM 设单独的 gas 限额，由共识提议者投票上下调。这样 RISC-V / WASM / Cairo VM / Move VM 都能作为 native rollup 接入，且无需硬分叉。

**OdysLam**（24 楼）同调：让 `EXECUTE` 跑在一个 RISC-V/WASM 基底上、支持多种 STF（主网 EVM、Arbitrum WASM、OP EVM…），才是生态的超能力；强行要求所有 native rollup 都是主网 EVM"对生态没好处"。

### 争论三：同步可组合性的宣称站不住脚（cwgoes vs mteam88 / thogard785，37–43 楼）

半年后（2025-07）**cwgoes**（**批注**：Christopher Goes，Anoma）来算了一笔硬账，直接质疑原帖收益清单里的最后一条。他先把定义钉死：同步可组合性 = 跨合约调用中，一个合约的状态变更**可以依赖**对另一个合约的同步调用结果（否则没意义）。而 `EXECUTE` 的签名里 `post_state_root` 和 `trace` 是**固定输入**，于是只有三种可能：

1. 由事先不知道最终交易排序和结果的人（如用户）算出来 → 不可能有同步可组合性；
2. 在 EVM 内部算出来 → 等于在 EVM 里模拟 EVM，算力上不可行；
3. 在排序确定之后、下一批交易执行之前，在 EVM 之外算出来 → 这确实能实现同步可组合性，但**今天的 L1 执行层没有这个能力**；而且要让它可扩展（不被单个验证者的重执行卡住），需要的恰恰是**执行分片**——而这正是提案想避开的东西。

"What am I missing?"（我漏了什么？）

**mteam88**（Spire，38 楼）答：靠共享区块构建者做模拟（他们称之为 coordinated sequencing），且这基本只适用于 based rollup。**cwgoes**（39 楼）反驳：那这个共享构建者要么**就是** L1 提议者，要么需要执行级预确认——而后者要求 L1 提议者去做执行，那就**没有扩容收益**了；至于分布式区块构建，"不过是执行分片的又一次改名"。

**thogard785**（40 楼）换了个角度：区分「保证成功」和「要么全部回滚、要么正确互操作」两种保证，认为后者今天就能做到、且在 gas 抽象 UX 下用户感知不到差别。**cwgoes**（41 楼）顺势收网：如果你要的只是后者，那你**根本不需要 native rollup、based rollup 或任何特定排序构造**——事先模拟、失败就回滚即可。

**批注**：这一串交锋含金量极高。它说明原帖把「延迟一个 slot 的状态根 → 证明时延放宽 → 同步可组合性」这条推理链写得太轻了：放宽的只是**证明**的时延，而同步可组合性卡在**执行**必须在排序之后、在链上语义之内完成。这一条至今仍是 native rollup 叙事里最脆弱的部分。

### 争论四：一堆扎实的工程细节

- **状态树的选择（ihagopian，4 / 13 楼；批注：Ignacio Hagopian，EF）**：`EXECUTE` 用的树最自然应当与 L1 相同，否则 EL 客户端将长期背负多种树的证明。麻烦在于 L1 自己还要迁移（Verkle 或二叉树）。他给了两条路：要么等 L1 换完树再上 native rollup（引入依赖，且逼迫 L1 把树的选择**一次定死**）；要么让 native rollup **抢跑到最终形态的树**上，反过来给 L1 当"演武场"。
- **重执行阶段的带宽（perama-v，5 楼）**：一份实打实的餐巾纸计算。trace = L2 区块 + L2 前置状态；15M gas 区块的前置状态实测约 **3MB**（去重 + ssz + snappy 后）。折算约 **127 KB/s per Mgas per rollup**。若 5 个 native rollup 各开 2M gas，额外带宽约 1.2 MB/s——他认为**可接受**，是通往 zk-EXECUTE 的合理中间步骤。
- **欺诈证明二分博弈碎了（i-norden，19 / 25 楼）**：如果 `EXECUTE` 的前后状态是 MPT 状态根，二分博弈就**不能细分到 opcode 级**（有些 opcode 根本不改状态），最细只能到**单笔交易**。而传统欺诈证明是二分到单条 MIPS 指令、在 VM 帧上比对的，而且二分的对象不只是交易执行，还包括 **L1→L2 的推导逻辑**。原帖没有解答这一层怎么办。（他还补了一刀：ZK 欺诈证明这条替代路线，在朴素重执行版的 `EXECUTE` 上跑不通。）
- **RISC-V 不是免费午餐（mratsim，22 楼）**：直接反驳原帖结尾"RISC-V 原生执行"的设想。RISC-V（以及 WASM、MIPS）对大整数和椭圆曲线运算是**糟糕的 ISA**——模拟一次带进位加法要多花约 5 倍指令。正因如此，所有 RISC-V zkVM 实际上都在用带 uint256 / 椭圆曲线扩展的**方言**。结论：这份复杂度应该留在 zkVM 里，不要抬进共识层。
- **已有原型（acolytec3，26 楼）**：他在 ethereumjs 里实现了最朴素版的 `EXECUTE` 预编译（PR #3865），状态树用 EIP-7864 的二叉 Merkle 树，从前置状态证明构造稀疏状态树后重执行交易，trace 以 SSZ 序列化存放。**批注**：提案发出不到两个月就有客户端原型，这一点常被忽略。
- **gas 记账能不能标准化（The-CTra1n 35 楼 / thegaram33 36 楼）**：正面回应 edfelten 的第 3 条。The-CTra1n 认为在 blob 共享、blob 数量上升之后可以假设 blob 总是满的、每字节 DA 成本一致，于是 DA 费可按交易摊进 blob；执行成本因为大家调同一个执行函数而天然标准化；证明成本则内含在执行成本里。他坦承真正的输家是**跑专属排序器、提供预确认的 rollup**，并给了一句很狠的备选项："Don't be native."（那就别做 native。）thegaram33 补充：难点在于把 DA / 证明成本**正确归因到具体的 L2 交易**上（数据重的交易 vs 计算重的交易），现行 L1 费用机制不够用，EIP-7623 算是一步。
- **另一条路线（levs57，3 楼）**：他同期发了自己的方案——`EXECUTE` 只作为**逃生舱**：当 rollup 的证明系统被检测到出错（证出两个互相矛盾的命题）时，才把执行搬回 L1。这样先让所有 zk rollup 摆脱安全委员会，之后再给足够可信的证明系统叠加 native 特性。

### 争论五：这到底解决了谁的问题（怀疑派）

- **kladkogex**（27 / 32 楼）：措辞很冲，但值得记下。他质疑"与主网同等安全"的说法（**批注**：他的论据是状态被中心化存储、可被扣留——这一点其实站不住，因为原帖明确要求 trace 数据必须在 L1 上可得），并直指整个提案是 Paul Graham 意义上的 **"a solution in search of a problem"**（一个在找问题的解法）：用户要的是更快的 L1，rollup 用户已经在用 Base 了。
- **LampAdmin**（28 / 34 楼）：写了一整张表格系统复述 edfelten 的批评（可组合性、桥、gas、预编译），结论是"技术上有缺陷且不实用"，随后话题滑向 EF 与 Solana 的生态之争和价值捕获。**批注**：技术部分是 edfelten 的转述，原创性不高；但它引出了 thegaram33 的 `L1ContextPrecompile` 回应，是那条线索的起点。
- **GregTheGreek**（16 楼）：唯一认真谈**经济激励**的一楼。他主张排序器费用应主要归 L2（"We are not Apple and should not be the App Store."），并主张给 native rollup 在 blob 里设**优先队列和专属定价**——如果一个 rollup 把部分排序收入让渡给协议，它的数据就该在 DA 上有优先权。
- **未被回答的基础问题**：0xTariz（7 楼）问验证者到底在哪一步确认 `post_state_root`；otrsk（11 楼）问 "trustless rollup" 和 "native rollup" 到底是不是一回事；0xvon（14 楼）问在有了 `EXECUTE` 的前提下，究竟哪些部分**必须**改协议；Marchhill（10 楼）问"几乎 native"的 rollup 可不可行（自答：为那点差异单独做证明器，就等于把治理请回来了）。这些问题在帖内**全部没有得到作者回复**。

## 关键术语（中英对照）

| 中文 | English | 说明 |
|---|---|---|
| 原生 rollup | native rollup | 用 `EXECUTE` 验证 EVM 状态转移的 rollup；旧称 enshrined rollup |
| 无信任 rollup | trustless rollup | 完全继承以太坊 L1 安全性的 rollup（原帖专门造的词） |
| 执行轨迹 | trace | L2 交易列表 + 状态访问证明；必须在 L1 上可得 |
| 无状态执行 | stateless execution | 只凭 trace 内附的状态叶子即可执行，不需要完整状态库 |
| 状态转移函数 | STF (State Transition Function) | 争论焦点：native rollup 只能用 L1 的那一个 |
| 链推导函数 | CDF (Chain Derivation Function) | 从 blob 原始数据还原 L2 链（解压、强制包含等）；`EXECUTE` 管不到这一层 |
| 延迟一个 slot 的执行 | one-slot delayed execution | 无效区块视为 no-op；为证明腾出整整一个 slot |
| 出块者-证明者分离 | APS (Attester-Proposer Separation) | 提案假设的激励语境，执行 slot 与共识 slot 交替 |
| 利他少数证明者假设 | altruistic-minority prover assumption | 只要有一个利他证明者就能保证轻客户端及时读到状态 |
| 链下证明 | offchain proofs | 证明不上链、不入共识；验证者自选 zkEL 客户端 |
| 可靠性 bug | soundness bug | 能为任意 `EXECUTE` 调用伪造证明的灾难性漏洞；用 k-of-n 对冲 |
| 状态差分 | state diffing | 只发状态变化量的压缩手段；native rollup **不能**用 |
| 超音速 rollup | ultra sound rollup | 戏称：同时是 based 又是 native 的 rollup |
| 执行分片 | execution shards | 不可编程、数量固定的 L1 EVM 副本；与 native rollup 相关但不同 |

## 值得记住的要点

1. **`EXECUTE` 本身极小**：四个输入、一个布尔输出、不接受证明。全部复杂度被推到共识层的"验证者如何确信这次调用是对的"上，而这一步刻意**不入链、不内置证明系统**。
2. **"证明不上链"是整个设计的枢纽**，也是它与"内置 zkEVM 验证器合约"路线的分水岭。它换来客户端多样性和技术中立，代价是证明负载翻倍、p2p 分裂、少数派 zkEL 无人证明。
3. **原帖对现有 rollup 的杀伤性判断**：几乎所有 EVM rollup 今天的 STF 里"很可能都有关键漏洞"，中心化排序器是掩盖这一点的拐杖。这句话是整个提案的道德正当性来源。
4. **最硬的未解难题不是密码学，是"L2 不是纯 EVM"**。Ed Felten 的"一条链只能有一个 STF"是全帖最强的论点，双 STF 折中被证明会破坏可组合性，`L1ContextPrecompile` 只补上了 L1→L2 消息这一角。
5. **"同步可组合性"这条收益被 cwgoes 严重削弱**：延迟 slot 放宽的是证明时延，而同步可组合性卡的是执行必须在排序之后完成——绕过去就要执行分片，而那正是提案想避开的。
6. **作者全程零回复**（42 楼、跨度半年）。把这个帖子当"提案 + 一手异议清单"读，而不是"提案 + 作者答辩"读。真正的回应发生在后续帖子和社区通话里。
7. **重执行版是真能落地的**：perama-v 的带宽估算（~127 KB/s per Mgas per rollup）和 acolytec3 的 ethereumjs 原型（PR #3865，EIP-7864 二叉树）说明，"先上朴素重执行、后上 SNARK"的路径不是纸上谈兵。
