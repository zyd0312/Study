# Optimism 与 Succinct 合作，把 ZK 证明带上 Superchain

> **来源**：Optimism 官方博客｜**原文链接**：https://optimism.io/blog/optimism-partners-with-succinct-as-preferred-provider-to-accelerate-zk-proving-on-the-superchain
> **一句话结论**：Optimistic Rollup 的旗舰项目**自己宣布转投 ZK**——零知识有效性证明将成为 OP Stack 的规范（canonical）方案，**挑战期将被彻底消除**。

> 📌 **为什么这篇最短的文章却最重要**：它是 2021 年那场"乐观 vs ZK"辩论的**结局公告**。而且宣布结局的不是 ZK 阵营，是 Optimistic 阵营自己。

## 公告说了什么

Optimism 宣布 **Succinct** 成为其**首个官方 ZK 证明合作方**（first preferred provider for ZK proving）。核心内容：

- **零知识有效性证明（validity proofs）将成为 OP Stack 的规范方案**（canonical）。
- 落地产品是 **OP Succinct**——用有效性证明对交易做密码学确认，**消除 optimistic rollup 目前必需的挑战期（eliminating the challenge period）**。
- 最终目标：**即时、实时的提款**（instant, real-time withdrawals）。
- 路径：先在 **OP 主网**上线 ZK 证明，之后**现有的 OP Stack 链可以无缝升级**到 ZK 有效性证明。
- 定位为**多证明架构（multi-proof architecture）**的一部分——将来各条链可以自行选择最适合的证明者网络（prover network）。

## 动机：为什么是现在

公告给出的理由**完全是机构资金效率**，不是意识形态：

> "交易本身是亚秒级的快，但把资金安全地从 L2 下车到 L1 却要花好几天。"

具体痛点：

| 谁 | 痛在哪 |
|---|---|
| **托管服务商**（custody providers） | 转移客户资产要锁仓一周 |
| **做市商**（market makers） | 跨 L2 头寸的资金效率被拖累 |
| **财资操作**（treasury operations） | 提款终局性达不到传统金融基础设施的预期 |

Optimism 把这次合作明确定位为"为上链机构打造模块化、企业级基础设施"的一步。

> 💡 **值得玩味**：2021 年 Deribit 那篇文章把 1–2 周提款延迟称为 Optimistic 的"阿喀琉斯之踵"，但当时设想的解法是**快速提款桥**（Connext / Hop / MakerDAO 桥）——即用金融手段绕过技术缺陷。五年后 Optimism 给出的答案是：**不绕了，直接把证明机制换掉。**

## 两句关键表态

Succinct CEO **Uma Roy**（引用）：

> "随着 rollup 向 ZK 收敛（As rollups consolidate around ZK），Succinct 正在构建整个生态可以依赖的证明基础设施。"

Optimism 联合创始人 **Karl Floersch**（引用）：

> "我们很兴奋能把有效性证明带到 Superchain，聚焦于为 Optimism 及合作伙伴在 **2026 年及以后**实现快速、低成本的扩容。"

> 📌 **"rollups consolidate around ZK"（rollup 正在向 ZK 收敛）——这半句话就是本合集要讲的整个故事。** 而且它出现在 Optimism 自家的官方博客上，这比任何第三方分析都更有分量。

## 关键术语（中英对照）

| 术语 | 英文 | 含义 |
|---|---|---|
| 有效性证明 | validity proof | 证明状态转移正确的密码学证明；ZK Rollup 的核心 |
| 挑战期 | challenge period | Optimistic Rollup 等待欺诈证明的窗口（传统上 7 天），本次要消除的对象 |
| OP Stack | OP Stack | Optimism 的开源 rollup 技术栈，Superchain 中众多链共用 |
| Superchain | Superchain | 基于 OP Stack 构建的一组共享标准的 L2 网络 |
| 多证明架构 | multi-proof architecture | 一条链可接入多个证明系统/证明者网络，避免单点依赖 |
| 证明者网络 | prover network | 提供 ZK 证明生成服务的去中心化算力网络（如 Succinct） |
| 提款终局性 | withdrawal finality | 资金从 L2 回到 L1 后不可撤销所需的时间 |

## 与 2021 年那篇的关系

Deribit 那篇文章的核心押注是：**"至少在近期，Optimistic Rollup 的成败在很大程度上取决于这些'快速提款'协议能否成功。"**

结局是：**押错了赛道，但押对了痛点。** 提款延迟确实是生死问题——重要到 Optimism 宁可把自己赖以得名的"乐观"机制整个换掉。

同时它也印证了 Deribit 那篇的长期预测：**"各方案在成本、速度、体验上的差距会趋近于零。"** 只不过收敛的方式不是双方各自优化，而是**一方直接采用了另一方的技术**。

## 值得记住的要点

- **Optimistic vs ZK 的路线之争，实质上已经结束了**——由 Optimistic 阵营的旗舰亲自宣布。
- **驱动力是机构资金效率，不是技术优雅**。一周的提款锁仓对托管方和做市商是不可接受的成本。
- **"消除挑战期"意味着 optimistic rollup 将不再 optimistic**——名字会留下，机制会消失。
- ⚠️ **但别急着下定论**：这是一份**公司公关公告**，不是技术规范，也没有给出时间表和成本数据。"最终目标是实时提款"是愿景措辞。真实的工程约束（证明成本、证明延迟）请看笔记 07——那篇给出了不那么乐观的量化分析。
