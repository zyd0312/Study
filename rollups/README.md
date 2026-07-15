# Rollup 演进：从 2021 的"乐观 vs ZK"到 2026 的收敛

> 这是一份精读笔记合集。起点是 Deribit Insights 2021 年 5 月那篇《Making Sense of Rollups, Part One: Optimistic vs. Zero Knowledge》，终点是 2026 年的现状。
> 所有笔记均为**中文总结**（非全文翻译），基于**一手原文抓取**，每篇都标注了原文链接。
> **整理日期**：2026-07-13

---

## 一句话总结

**2021 年那场"乐观 vs 零知识"的路线之争已经结束了——ZK 赢了。但宣布结局的不是 ZK 阵营，是 Optimistic 阵营自己：Optimism 官方已宣布 ZK 有效性证明将成为 OP Stack 的规范方案，直接消除挑战期。**

不过这个"结局"有三层复杂性，是本合集真正想讲的东西：

1. **ZK 赢，靠的不是它 2021 年被吹捧的优点。** 当年 ZK 的卖点是"密码学更安全"，而实际让它赢的是**提款延迟**——一个被机构资金效率倒逼出来的、纯商业的理由。
2. **二分法本身正在瓦解。** "乐观（欺诈证明）vs ZK（有效性证明）"已不再是干净的两分。**ZK 欺诈证明**这类混合方案正在出现（虽然争议很大，见 [ZK 笔记 07](../zk/07-ethresearch-zk-fraud-proofs.md)）。
3. **战场已经转移。** 今天的前沿问题不是"哪种 rollup 更好"，而是"**几十条 rollup 之间怎么才像一条链**"，以及"**rollup 还该不该自己造证明系统**"（native rollups）。

---

## 阅读顺序

如果时间有限，**读 01 → 02** 就能拿到完整故事线（约 20 分钟）。ZK 相关深度内容已移入 [ZK 专题](../zk/README.md)。

| # | 笔记 | 一句话 | 优先级 |
|---|---|---|---|
| [01](01-deribit-2021-optimistic-vs-zk.md) | **Deribit 2021：乐观 vs 零知识** | 起点。分析框架至今有效，技术结论已被推翻 | ⭐ 基线 |
| [02](02-vitalik-2025-scaling-l1-l2.md) | **Vitalik：2025 及以后的 L1/L2 扩容** | 官方路线图。全文**没有再争论**乐观 vs ZK——这本身就是答案 | ⭐ 必读 |
| [04](04-l2beat-native-rollups.md) | **L2BEAT：Native Rollups 综述** | 下一个五年的主线。质量最高的一篇 | ⭐ 推荐 |
| [05](05-ethresearch-native-rollups.md) | **ethresear.ch：Native Rollups 原始提案** | Justin Drake 的原帖 + 讨论区的一手争论 | 深入 |
| [06](06-eip-8079.md) | **EIP-8079 规范速查** | ⚠️ Draft 状态，大量 `TBD`，别当成定案 | 速查 |
| [08](08-l2beat-stages-framework.md) | **L2BEAT Stages 框架** | 安全性辩论如何从"打嘴仗"变成"可核查标准" | ⭐ 推荐 |
| [09](09-lean-ethereum-roadmap.md) | **Lean Ethereum 路线图** | ⚠️ 与 rollup 的关系被媒体严重误传，见下文 | 参考 |

> 📂 **ZK 专题**（独立目录）: [Optimism × Succinct 结局公告](../zk/03-optimism-succinct-zk.md) · [ZK 欺诈证明争议](../zk/07-ethresearch-zk-fraud-proofs.md) · [ZK-Rollup 核心机制](../zk/10-zk-proving-circuit-deep-dive.md) · [SNARK 数学基础](../zk/01-vitalik-2021-snarks-explained.md)

---

## 主线一：那场争论是怎么结束的

### 2021 年的赌局

Benjamin Simon 在 Deribit 那篇里的核心判断是：

- ZK 理论上更漂亮，**但当时做不出通用 EVM 智能合约**，只能做转账和撮合。
- Optimistic 能让 DeFi **无痛迁移**，所以是唯一能救急的方案。
- 安全性上**两边各有软肋**：ZK 依赖单一 relayer、活性弱（甚至可能冻结资金勒索用户）；Optimistic 有 1–2 周挑战期，但"n 中有 1 个诚实"的假设其实很弱——因为**告发者能拿走作恶者被没收的质押金**，所以真正需要的只是"**一个贪婪的人**"。
- Optimistic 的生死取决于**快速提款桥**（Connext / Hop / MakerDAO 桥）能否做成。

### 五年后的结算

| 2021 年的判断 | 结局 |
|---|---|
| ZK 做不出通用 EVM | ❌ **彻底推翻**。zkEVM 已成熟；native rollups 更进一步 |
| Optimistic 是唯一能救急的 | ❌ **反转**。Optimism 亲自转向 ZK 有效性证明 |
| 靠快速提款桥解决延迟 | ⚠️ **押对痛点，押错解法**。桥做出来了，但主流答案是**换掉证明机制本身** |
| 定性直觉不足以裁决安全性 | ✅ **被制度化印证**（L2BEAT Stages） |
| 长期只剩安全假设 + 网络效应 | ✅ **最有生命力的预测** |
| 用户不在乎安全、只追低费用 | ✅ 应验 |

### 最反直觉的一点

**赢的方式不是 ZK 阵营击败了 Optimistic 阵营，而是 Optimistic 阵营换上了 ZK 的引擎。**

而且驱动力**完全是商业的**。看 Optimism 公告里列的痛点——托管服务商要锁仓一周、做市商的跨 L2 资金效率、财资操作的终局性达不到传统金融预期。一句意识形态的话都没有。**"optimistic rollup"这个名字会留下，那个乐观的机制会消失。**

---

## 主线二：新战场（2021 年根本不存在的问题）

### 1. Rollup 的成功制造了更难的问题

Vitalik 2025 年那篇的重心已经完全不在"哪种 rollup 好"上了。他的问题是：

> **"用以太坊应该感觉像在用一个生态，而不是 34 条不同的区块链。"**（引用）

他给 L2 定的**及格线**是全文最锋利的判据：**"2016 年式原生分片本来会给你什么"**。凡是原生分片里不会存在的信任假设，今天一律不接受——所以他直接判了多签桥死刑（"An ecosystem relying on multisig bridges is NOT acceptable"）。

议题清单：链特定地址、无需额外信任的跨链消息标准、把提款压到分钟/单 slot、L2 同步读 L1、共享排序。**这些在 2021 年那篇里一个都不存在。**

### 2. Native Rollups：别各造轮子了

这是**下一个五年的主线**。核心是 `EXECUTE` 预编译（对应 EIP-8079）：**把 EVM 状态转移的验证做进 L1 协议本身**，L2 直接调用，不必自己造、自己维护、自己升级一套 zkEVM。

为什么这是对 2021 年框架的釜底抽薪：当年的整个辩论建立在"**每条 L2 各自造证明系统，然后我们比较谁的信任假设更好**"这个前提上。Native rollup 说的是——**别造了**。这实际上是让 L2 **回归它本来该是的东西：分片**。

⚠️ 但**别跟着炒作走**，一手材料里的保留意见很扎实：
- **EIP-8079 是 Draft，且预编译地址、输入输出编码、gas 计费、安全考量全部 `TBD`。**（笔记 06）
- 原提案帖里 **Justin Drake 是唯一作者，且发完原帖后从未回复过任何质疑**。为提案辩护的实际上是 Scroll 的 thegaram33。（笔记 05）
- 讨论区最有分量的质疑来自 **cwgoes：他系统性地质疑了"同步可组合性"这个核心收益到底值不值。**（笔记 05）
- L2BEAT 那篇点出的硬约束：**trace 型数据可得性的成本瓶颈**、**利他证明者假设**，以及**吞吐量与可组合性成反比**。（笔记 04）

### 3. 二分法正在瓦解：ZK 欺诈证明

这是本合集里**最该认真读的一篇反面材料**（[ZK 笔记 07](../zk/07-ethresearch-zk-fraud-proofs.md)）。

思路是走第三条路：既不是给每个区块生成有效性证明（成本高），也不是传统交互式欺诈证明（挑战期长），而是**用 ZK 做非交互式欺诈证明**。代表方案是 Kailua、OP Succinct Lite。

**但作者的结论是否定的**——标题《Best of Both Worlds?》的问号，答案是"不是"：

> 作者（GCdePaula & Augusto Teixeira, 2025-09）认为非交互式 ZK 欺诈证明**继承了两边的缺点**：挑战期仍然需要约 1 周，**且防守成本随攻击者投入的资金线性增长**。

⚠️ **两个必须知道的免责声明**（子代理核出来的，很重要）：
1. **这项研究由 Cartesi 资助，而文中作为对照标杆的 Dave 正是 Cartesi 自家的方案。** 这不是中立评测，看 Tally 对比表时要打折扣。
2. **讨论区有一处事实更正**：Kailua 团队的 rami 指出 **OP Succinct Lite 根本不是非交互式的**——每个坏提议都需要显式挑战，充其量算"一轮交互式协议"。所以把两者并列为"非交互式 ZK 欺诈证明"的代表本身就不准确。

**教训**：这个领域正在快速演化，连"某方案属于哪一类"都还有争议。**不要把任何二分法当成稳定的事实。**

### 4. 安全性辩论的制度化

2021 年 Simon 只能靠**定性直觉**辩论安全性——"一个贪婪的人就够了" vs "ZK 有密码学保证"。今天有了 **L2BEAT Stages 框架**（笔记 08）：Stage 0 / 1 / 2，衡量"训练轮"（training wheels）拆掉了多少，Stage 2 = 完全由代码治理。

两个容易记错的现状（子代理从原文核出）：
- **Stage 1 的最短挑战期已经从 7 天降到 5 天**（2026-04-30 生效）。**任何还在说"Stage 1 需要 7 天挑战期"的材料都过时了。**
- 2025-06-20 的重新分类把"**≥5 个外部挑战者**"这条要求从 Stage 1 **下放到了 Stage 0**——这点很容易记反。

---

## ⚠️ 一处必须澄清的媒体误传

网上（包括中文圈）大量在传"**Vitalik 的 Lean Ethereum 路线图宣告了以 rollup 为中心的路线图终结**"，还有"抗量子 + 隐私 + 可扩展性三大支柱"的概括。

**我们核对了 leanroadmap.org 的一手原文，这两个说法都站不住**：

- **原文中 "rollup" / "L2" / "layer 2" 一次都没有出现。** "rollup 路线图终结"在这份材料里**没有任何一手依据**。
- **"三大支柱"的概括也是错的。** 原文的四条 Key Ideas 是 **Lean Consensus / Cryptography / Governance / Craft**，研究方向标签是 Consensus / PQ Signatures / Scaling / Security——**根本没有"隐私"这一项**。
- 原文的实际主线是：**后量子哈希签名**（用 Winternitz XMSS 替代 BLS）+ 签名聚合 + **3SF 秒级最终性**，通过 pq-devnet-0 到 5 逐步推进。（"beam chain" 是 Lean Consensus 的旧称。）

那个"rollup 终结"的叙事**可能**来自 Vitalik 或 Justin Drake 在其他场合（X、播客）的发言，但**需要另找一手材料才能评估**。详见笔记 09。

**这件事本身就是个方法论教训**：这个领域的中文和英文二手内容质量都极差，标题党严重。**凡是重要判断，回一手材料。**

---

## 已排除的材料（以及为什么）

搜索时出现了大量 Binance Academy、Coinbase Learn、Changelly、Nervos、eco.com 这类"ZK vs Optimistic 区别"的科普文，标题挂着 2026，**但内容停留在 2022 年的认知**（还在说"ZK 不支持通用智能合约"）。**读了会被误导，因此全部排除。**

---

## 如果只记住四句话

1. **路线之争已结束，ZK 赢了**——但赢的方式是 Optimistic 阵营自己换了引擎，且驱动力是机构的资金效率，不是密码学的优雅。
2. **"需要一个诚实的人" ≈ "需要一个贪婪的人"**——Deribit 那篇最有价值的洞察，是理解一切欺诈证明系统的钥匙，今天依然成立。
3. **活性（liveness）和有效性（validity）是两个独立的安全维度**——ZK 保证后者却牺牲前者，这个 2021 年的观察至今是评估任何证明系统的必备视角。
4. **下一个问题不是"哪种 rollup"，而是"rollup 还该不该自己造证明系统"**（native rollups），以及"几十条 rollup 怎么才像一条链"。

---

## 附：一手来源清单

| 笔记 | 原文 |
|---|---|
| 01 | https://insights.deribit.com/market-research/making-sense-of-rollups-part-one-optimistic-vs-zero-knowledge/ |
| 02 | https://vitalik.eth.limo/general/2025/01/23/l1l2future.html |
| 03 | https://optimism.io/blog/optimism-partners-with-succinct-as-preferred-provider-to-accelerate-zk-proving-on-the-superchain |
| 04 | https://medium.com/l2beat/native-rollups-where-they-are-and-where-they-are-going-cb21eb103d46 |
| 05 | https://ethresear.ch/t/native-rollups-superpowers-from-l1-execution/21517 |
| 06 | https://eips.ethereum.org/EIPS/eip-8079 |
| 07 | https://ethresear.ch/t/best-of-both-worlds-a-measured-review-of-non-interactive-zk-fraud-proofs/23132 |
| 08 | https://forum.l2beat.com/t/the-stages-framework/291 |
| 09 | https://leanroadmap.org/ |
| 10 | https://ethereum.org/developers/docs/scaling/zk-rollups/#how-do-zk-rollups-work |
