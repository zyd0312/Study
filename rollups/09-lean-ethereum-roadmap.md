# Lean Ethereum 路线图

> **来源**：leanroadmap.org 官方站点，页面标题《Lean Consensus R&D Progress》
> **时间**：页面自标日期 **2026 年 4 月**（抓取于 2026-07）
> **原文链接**：https://leanroadmap.org/
> **一句话结论**：这是一个**共识层（consensus layer）研发进度追踪站**——围绕「**Lean Consensus**：以太坊共识层的彻底重设计」展开，核心工程主线是**后量子（post-quantum）哈希签名 + 签名聚合 + 秒级最终性**，通过一系列 `pq-devnet-N` 测试网推进。**它通篇没有提到 rollup 或 L2。**

---

## 这是什么

严格按原文：这是 **Lean Consensus 的研发进展页**，包含五个部分——概览（Key Ideas）、性能基准（Benchmarks）、测试网（Devnets）、研究方向（Research Tracks）、客户端实现（Client Implementations），外加学习资源和历次 leanConsensus 电话会议记录。

原文给出四条 **Key Ideas**（注意：原文称之为 "Key Ideas"，**并未**称之为「三大支柱」或"pillars"）：

| 原文名称 | 中文 | 原文描述 |
| --- | --- | --- |
| **Lean Consensus** | 精简共识 | 以太坊共识层的**完整重设计**——为安全性、去中心化、**秒级最终性（finality in seconds）**而强化 |
| **Lean Cryptography** | 精简密码学 | **哈希签名（hash-based signatures）**，同时对 SNARK 友好、对量子计算机安全——「一块简单积木撑起一切」 |
| **Lean Governance** | 精简治理 | **战略性地打包协议升级**——让互补的技术改进能一次性高效交付 |
| **Lean Craft** | 精简工艺 | 极简主义（minimalism）、模块化、**形式化验证（formal verification）**——「能多做一步就多做一步」 |

> **批注**：如果一定要归纳「支柱」，从原文的研究方向标签看，实际的四个分类是 **Consensus / Post-Quantum Signatures / Scaling / Security**。**原文没有「隐私（privacy）」这一项**——整页找不到 privacy 相关的研究方向、里程碑或术语。任何把 Lean Ethereum 概括为「抗量子 + 隐私 + 可扩展性三大支柱」的说法，在这个页面上**找不到依据**。

「beam chain」这个名字在原文中只以**旧称**出现：所有会议都标为 "leanConsensus (fka beam chain)"，即 **beam chain 是 Lean Consensus 的前身叫法**，现已改名。

---

## 核心支柱与时间线

### Devnet 时间线（原文唯一明确的时间轴）

| 测试网 | 状态 | 时间 | 目标 | 结果 |
| --- | --- | --- | --- | --- |
| **pq-devnet-0** | 已完成 | 2025-09 | 建立 leanSpec 框架；多客户端协同（**尚无 PQ 签名**）；4 秒 slot、QUIC、Gossipsub v1.0 互操作试验；采用改良版 **3SF-mini** 作为共识机制 | 初版客户端规范确立；Ream / Zeam / Qlean 三客户端在改良 3SF-mini 下达成互操作 |
| **pq-devnet-1** | 已完成 | 2025-11 | 集成 **leanSig** 签名与验证；基础签名聚合（**靠简单拼接**）；建立 PQ 签名性能基线 | leanSig 已集成进客户端；带 PQ 签名/验证的客户端互操作达成。客户端增至 5 个（+Lantern、Grandine） |
| **pq-devnet-2** | 已完成 | 2025-12 | 集成 **leanMultisig** 聚合；建立 PQ 签名聚合性能基线 | 原文写 "In progress..."（结果尚未填写）。客户端 6 个（+ethlambda） |
| **pq-devnet-3** | 已完成 | 2026-01 | 通过独立的 **aggregator 角色**把聚合与出块解耦；建立聚合签名传播协议；为探索不同聚合/传播策略打基础 | 原文写 "In progress..." |
| **pq-devnet-4** | **Active（进行中）** | 目标 2026-02 | 用 **leanVm** 实现**递归 PQ 签名聚合**；把同一消息的多个聚合合并为一个最终聚合；确保每个区块每条消息只含一个聚合 | 进行中。客户端 8 个（+gean、Peam） |
| **pq-devnet-5** | 计划中 | 未给日期 | 产出**一个块级聚合证明**覆盖多条 attestation 消息；利用**证明可分解性**使单条消息的证明仍可从块级证明中恢复；**用「PQ heartbeat」替换 3SF-mini**——在 **Goldfish fork-choice** 下的委员会投票 | — |

> **批注**：页面自标日期为 2026 年 4 月，但 pq-devnet-4 的目标是 2026 年 2 月且仍标为 Active，devnet-2/3 标为 "Completed" 而结果栏仍是 "In progress..."。说明**页面更新有滞后**，devnet 进度较原计划有延后。原文未解释原因。

### 研究方向进度（原文给出的百分比）

| 研究方向 | 进度 | 分类标签 | 负责人 |
| --- | --- | --- | --- |
| Hash-Based Multi-Signatures（基于哈希的多重签名，用 Winternitz XMSS 替代 BLS） | 70% | PQ 签名 / 安全 | Benedikt Wagner |
| Poseidon Cryptanalysis Initiative（Poseidon 哈希函数密码分析计划） | 50% | PQ 签名 / 安全 | Dmitry Khovratovich |
| Post-Quantum Signature Aggregation with zkVMs（用极简 zkVM 做 PQ 签名聚合） | 50% | PQ 签名 | Thomas Coratger |
| Faster Finality（更快最终性：从约 15 分钟降到秒级，用 **3SF** 三槽最终性） | 50% | 共识 / 扩展 | Barnabé Monnot |
| Formal Verification（用 Lean 4 形式化验证 FRI / STIR / WHIR 等证明系统） | 40% | PQ 签名 | Alex Hicks |
| P2P Networking（Gossipsub v2.0、集合对账，支撑 4 秒出块与质押门槛 **32 ETH → 1 ETH**） | 30% | 安全 | Raúl Kripalani |
| Attester-Proposer Separation（证明者-提议者分离 APS） | 20% | 共识 / 扩展 | TBD |

### 性能基准（Benchmarks）

原文有两组基准图表：**leanSig**（哈希签名方案）和 **leanMultisig**（聚合签名方案）。

- **leanSig**：签名时间目标 500 µs 量级；在 MacBook Pro M1 单核上，**签名时间约为目标的 107%**（略超），**验证时间约为目标的 39%**（大幅优于目标，原文标了 🎉）。密钥生成基于 10 核 M1、8 年密钥寿命。公钥结构为「root 8 个元素 + randomiser 5 个元素」。
- **leanMultisig**：目标 **1000 XMSS 聚合/秒**。M4 Max（efficient 分支）达到目标的 **97%**，M4 Max（lean-vm-simple 分支）**82%**，Intel i9-12900H **38%**。**聚合体积**是主要短板：efficient 分支为目标的 **313–391%**，simple 分支 **234%**（即体积比目标大 2–4 倍）。

> **批注**：页面上的绝对数值（µs、KiB、hours）由前端 JS 动态渲染，抓取到的静态文本里全部显示为 0，**因此绝对数字无法从本次抓取中获得**；上面引用的**百分比**是原文明确写出的。

---

## 涉及的关键技术

| 技术 | 在原文中的角色 |
| --- | --- |
| **leanSig** | 后量子哈希签名方案的参考实现（Rust）；基于 **Winternitz XMSS**，用来替代 **BLS 签名** |
| **leanMultisig** | 后量子签名**聚合**方案参考实现 |
| **leanVm / zkVM** | 用极简零知识虚拟机做签名聚合；原文点名探索过 Binius M3、SP1、KRU、STU、Jolt、OpenVM |
| **SNARK / 哈希证明系统** | 研究涉及 **FRI、STIR、WHIR**、Plonky3、STwo、Binius、Hashcaster、GKR 类 prover、folding 方法 |
| **Poseidon / Poseidon2** | 算术友好哈希函数；原文列出一整套针对它的密码分析计划（含 66k 美元赏金、多篇 2025 年攻击论文） |
| **3SF（3-Slot Finality，三槽最终性）** | 把最终性从约 15 分钟降到秒级；原文称其为单槽最终性（SSF）的务实替代；与 ePBS、FOCIL、PeerDAS 集成 |
| **3SF-mini / PQ heartbeat / Goldfish** | devnet 早期用改良 3SF-mini；pq-devnet-5 计划改用 Goldfish fork-choice 下的委员会投票（"PQ heartbeat"） |
| **Gossipsub v2.0 / 无速率集合对账** | P2P 层改造，支撑 4 秒出块和**质押门槛从 32 ETH 降到 1 ETH**后暴增的验证者数量 |
| **APS（证明者-提议者分离）** | 降低中心化压力、改善 MEV 处理 |
| **Lean 4 形式化验证** | 数学化证明 FRI / STIR / WHIR 的安全性质，并验证 zkEVM 实现正确性 |
| **客户端多样性** | 8 个 Lean Consensus 客户端：Ream(Rust)、Zeam(Zig)、Qlean-mini(C++)、Lantern(C)、Lighthouse fork(Rust)、ethlambda(Rust)、gean(Go)、Peam(Rust) |

---

## 对 rollup / L2 路线意味着什么

**必须先说清楚一个事实：原文（leanroadmap.org 抓取内容）中，"rollup"、"L2"、"layer 2"、"data availability"、"blob" 这些词一次都没有出现。**

这个页面的范围**完全局限于共识层（consensus layer）**：签名方案、聚合、最终性、fork-choice、P2P 网络、客户端实现。它既没有讨论执行层，也没有讨论 L1 扩容 vs L2 扩容的路线取舍。

### 关于「以 rollup 为中心的路线图终结了」这一说法

> **结论：原文并未如此表述。这个说法在 leanroadmap.org 上没有一手依据。**

具体来说：

1. 原文**没有任何一句话**说以太坊放弃或降级 rollup-centric roadmap。
2. 原文**没有任何一句话**把 L1 扩容与 L2 扩容对立起来。
3. 原文中唯一与「扩展性」有关的标签是 **Scaling**，其下只有两项：**Faster Finality（3SF）** 和 **Attester-Proposer Separation（APS）**——两者都是**共识层**改进，与 rollup 的存废无关。
4. 唯一可能被引申的一句是 P2P 方向里提到的「质押门槛从 32 ETH 降到 1 ETH」——这是关于**验证者去中心化**，不是关于 rollup。

**批注（我的推测，不是原文内容）**：媒体的「rollup-centric roadmap 终结」叙事，更可能来自 Vitalik 或 Justin Drake 在其他场合（如 Devconnect 演讲、博客、播客）关于「L1 也要扩容 / gas limit 大幅提高 / 用 SNARK 做 L1 实时证明」的表述，而**不是**来自这个路线图页面。如果要评估那个说法，必须去看那些一手材料（原文的 Learning Resources 里列了 "Next 10 Years of Ethereum by Fede"、"Ethereum (Roadmap) in 30min by Vitalik Buterin"、Justin Drake 的 "lean Ethereum" 博客等），**不能把它挂在 leanroadmap.org 头上**。

### 那么，Lean Consensus 对 L2 的真实（间接）影响是什么？

以下是**基于原文内容的合理推导，标为批注**：

> **批注**：
> - **秒级最终性（3SF）**：L2 提款、跨 L2 桥、CEX 存款确认，全都依赖 L1 最终性。把 15 分钟压到秒级，会直接改善所有依赖 L1 结算的 L2 用户体验。
> - **后量子签名**：如果 L1 共识层换成哈希签名，那么依赖 L1 验证的 L2 桥合约、以及 L2 自己的签名体系，长期都会面临同样的迁移压力。
> - **SNARK 友好**：Lean Cryptography 明确强调「同时对 SNARK 和量子计算机友好」。共识层如果本身可被 SNARK 高效证明，L1 就更容易被「实时证明」，这确实**可能**改变 L1 与 L2 的分工——但**原文没有把这层含义写出来**。
> - 以上都是**推论**，原文只字未提 L2。

---

## 关键术语（中英对照）

| 中文 | English | 说明 |
| --- | --- | --- |
| 精简共识 | Lean Consensus | 以太坊共识层的完整重设计；前身名为 beam chain |
| 光束链（旧称） | beam chain (fka) | Lean Consensus 的前身叫法，已弃用 |
| 后量子 | post-quantum (PQ) | 抵抗量子计算机攻击 |
| 哈希签名 | hash-based signatures | 只依赖哈希函数安全性的签名，天然抗量子 |
| 温特尼茨 XMSS | Winternitz XMSS | 具体的哈希签名方案，用来替代 BLS |
| 签名聚合 | signature aggregation | 把大量签名压缩成一个；leanMultisig 的目标 |
| 递归聚合 | recursive aggregation | pq-devnet-4 的目标：用 leanVm 把多个聚合再合并 |
| 三槽最终性 | 3-Slot Finality (3SF) | 把最终性从约 15 分钟降到秒级 |
| 单槽最终性 | Single-Slot Finality (SSF) | 3SF 是其务实替代 |
| 分叉选择 | fork-choice | pq-devnet-5 计划采用 Goldfish |
| 证明者-提议者分离 | Attester-Proposer Separation (APS) | 降低中心化压力、改善 MEV 处理 |
| 零知识虚拟机 | zkVM | 用来证明签名聚合的极简虚拟机 |
| 形式化验证 | formal verification | 用 Lean 4 数学化证明协议安全性质 |
| 集合对账 | set reconciliation | P2P 层优化技术 |
| 测试网 | devnet | pq-devnet-0 至 5 |

---

## 值得记住的要点

1. **这是共识层的事，不是 L2 的事。** leanroadmap.org 通篇没有 rollup / L2 字样。
2. **主线是抗量子**：用 Winternitz XMSS 哈希签名替代 BLS，并解决它带来的**聚合体积**问题（当前是目标的 2–4 倍，是最大短板）。
3. **副线是秒级最终性**：3SF 把约 15 分钟压到秒级。
4. **beam chain 已改名 Lean Consensus**——看到 beam chain 字样按新名理解。
5. **进度**：devnet-0 到 devnet-3 已完成（2025-09 至 2026-01），devnet-4 进行中（递归聚合），devnet-5 计划把共识机制换成 Goldfish 下的委员会投票。
6. **没有隐私支柱**。「抗量子 + 隐私 + 可扩展性」这种三支柱概括在本页面无依据。
7. **「rollup-centric roadmap 终结了」在本页面无依据**——不要因为媒体标题就把这个论断归给 Lean Ethereum 路线图。要论证这个命题，必须另找 Vitalik / Justin Drake 的一手表述。

---

## 材料局限

**这份材料的效力级别很低，必须清楚：**

1. **它是一个进度追踪网站，不是协议规范。** 真正的规范在 GitHub 上的 `leanEthereum/leanSpec`、`leanSig`、`leanMultisig`、`leanMetrics` 仓库里；本页只是索引和进度看板。
2. **它不是 EIP，也没有经过 All Core Devs 的采纳流程。** 页面上的任何内容都**不构成对以太坊主网的承诺**。研究方向进度条（20%~70%）是自评，没有客观标准。
3. **没有主网时间线。** 全文只有 devnet 时间点，**没有任何关于何时上主网、走哪次硬分叉的表述**。
4. **页面本身有滞后和空缺**：devnet-2/3 标为 Completed 但结果栏仍写 "In progress..."；devnet-4 目标是 2026-02 而页面日期是 2026-04 仍显示 Active。
5. **关键数字缺失**：基准图表的绝对数值由 JS 渲染，静态抓取拿不到；只有相对目标的百分比可用。
6. **多处标为 "Exploratory research"**（如 APS、Faster Finality 的里程碑），说明相当一部分内容仍处于早期探索，随时可能改变或被放弃。

> **一句话**：把它当作「以太坊共识层研究团队 2026 年在干什么」的**可靠索引**；不要把它当作「以太坊未来协议长什么样」的**权威规范**，更不要从它推导出关于 L2 路线的结论——它根本没谈这个话题。
