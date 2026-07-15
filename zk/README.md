# 零知识证明 (ZK) 深度研读

> 集中存放与零知识证明（ZK、SNARK、STARK）相关的精读笔记和技术解析。
> **整理日期**：2026-07-14

---

## 目录

| # | 笔记 | 一句话 | 类别 |
|---|------|--------|------|
| [01](01-vitalik-2021-snarks-explained.md) | **Vitalik：zk-SNARKs 如何成为可能** | 从多项式自比较 → 多项式承诺 → 随机抽查，逐层拆解 SNARK 的数学基础。含 $S=\{0,1,2,3,4\}$ 完整数值示例 | 📐 基础理论 |
| [03](03-optimism-succinct-zk.md) | **Optimism × Succinct：ZK 证明上 Superchain** | 2021 年"乐观 vs ZK"路线之争的结局公告——Optimism 亲自转投 ZK | 🏭 产业落地 |
| [07](07-ethresearch-zk-fraud-proofs.md) | **ZK 欺诈证明：两全其美？** | 对 Kailua / OP Succinct Lite 等混合方案的审慎评估。标题的问号，答案是"不是" | 🔬 前沿争议 |
| [10](10-zk-proving-circuit-deep-dive.md) | **ZK-Rollup 核心机制深度解析** | 证明电路如何工作、L1 为何只需验证一次、恶意交易在哪被拦截 | 🔧 工程机制 |

---

## 阅读建议

- **想理解 SNARK 数学基础** → 从 [01](01-vitalik-2021-snarks-explained.md) 开始
- **想了解产业落地现状** → 读 [03](03-optimism-succinct-zk.md)
- **想跟踪前沿技术争议** → 读 [07](07-ethresearch-zk-fraud-proofs.md)
- **想弄清 zkEVM 工程细节** → 读 [10](10-zk-proving-circuit-deep-dive.md)

---

## 关键概念速查

| 概念 | 一句话 | 详见 |
|------|--------|------|
| 多项式自比较 | 用 $P(x)=Z(x)\cdot H(x)$ 替代逐点验证 | 笔记 01 §一 |
| 多项式承诺 (KZG) | 多项式在秘密点 $\tau$ 上的椭圆曲线编码 + 配对验证 | 笔记 01 §三 |
| Schwartz-Zippel 引理 | 随机抽查一个点即可代表整个多项式 | 笔记 01 §3.3 |
| Fiat-Shamir 变换 | 用哈希模拟随机数，变交互为非交互 | 笔记 01 §3.5 |
| zkEVM | 能执行 EVM 智能合约的 ZK 证明系统 | 笔记 10 |
| ZK 欺诈证明 | 用 ZK 做非交互式欺诈证明（第三条路） | 笔记 07 |
