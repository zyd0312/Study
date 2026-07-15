# ZK-Rollup 核心机制深度解析：证明电路、L1 验证与安全模型

> 基于 [ethereum.org 官方文档](https://ethereum.org/developers/docs/scaling/zk-rollups/#how-do-zk-rollups-work) 的深度解读。
> 整理日期：2026-07-14

---

## 核心问题速览

| 问题 | 答案 |
|---|---|
| ZK-Rollup 向 L1 证明什么？ | "存在一串合法交易，使状态从 pre-state root 正确过渡到 post-state root" |
| L1 需要核验什么？ | ① ZK 证明密码学有效性 ② pre-state root 与合约记录一致 |
| 恶意交易在哪被拦截？ | **L2 证明电路**——非法状态转换在数学上无法生成有效 proof |
| 为什么 L1 只需验证一次？ | SNARK/STARK 的 **简洁性（Succinctness）**——验证成本 O(1)，与交易数量无关 |

---

## 一、ZK-Rollup 向 L1 证明什么，L1 核验什么

### 1.1 整体流程

```
L2 链下：
  交易批次执行 → 生成 ZK Proof（有效性证明） → 提交 pre/post state root + proof

L1 链上（验证合约）：
  验证 proof 密码学正确性 + 检查 pre-state root 匹配 → 接受 post-state root
```

### 1.2 证明的内容

ZK-Rollup 的 operator 生成的有效性证明（Validity Proof）核心命题：

> **"存在一串合法的交易序列，使得 Rollup 的状态从旧状态根（pre-state root）正确地过渡到新状态根（post-state root）。"**

证明电路接收的输入：

| 输入 | 说明 |
|---|---|
| **Batch Root**（交易的 Merkle 树根） | 该批次所有交易构成的 Merkle 树根哈希 |
| **每笔交易的 Merkle 证明** | 证明某笔交易确实包含在该批次中 |
| **发送方/接收方的 Merkle 证明** | 证明这些账户确实存在于 Rollup 的状态树中 |
| **中间状态根序列** | 每处理一笔交易后更新得到的新状态根 |

### 1.3 L1 验证的内容

L1 上的**验证者合约（Verifier Contract）**只核验两件事：

| 校验项 | 说明 |
|---|---|
| ZK 证明密码学有效性 | 验证 SNARK/STARK proof 本身是否成立 |
| Pre-state root 匹配 | 与合约中当前存储的状态根对比，确保是合法续接 |

若两者通过 → 将 post-state root 设为新的权威状态根，batch root 存档以供用户日后做 Merkle 证明。

> **L1 不重新执行交易，不检查余额，不验证签名**——这些全部由 ZK 证明在数学上保证。

---

## 二、恶意交易在哪被拦截？

### 三道防线

```
恶意交易
   │
   ▼
L2 Operator ──→ 被拦截（余额不足/nonce错/签名无效等）
   │
   │ （若 operator 作恶强行打包）
   ▼
L2 证明电路 ──→ 无法生成有效 ZK Proof（数学上不可能）
   │
   │ （若 operator 伪造 proof）
   ▼
L1 验证合约 ──→ Proof 验证失败，整批交易被拒绝
```

### 关键结论

- **恶意交易在 L2 就被密码学机制拦截**，L1 不需要逐笔"查账"。
- 非法状态转换在数学上**根本无法产生有效证明**（不是"被发现作弊"，而是"做不出证明"）。
- 这与 Optimistic Rollup 形成根本差异：安全不依赖于"有人举报"，而由数学直接保证。

---

## 三、Proving Circuit（证明电路）详解

> 这是 ZK-Rollup 安全模型中最关键的机制。电路是一套硬编码的数学约束——它不"检查"交易是否合法，而是让任何非法状态转换在数学上都无法产生有效证明。

### 3.1 什么是 Proving Circuit？

证明电路本质上是一个**算术电路（Arithmetic Circuit）**，将"验证一批交易是否正确执行"的计算问题转化为可被零知识证明系统（SNARK/STARK）证明的数学约束系统。

```
输入: 旧状态根 + 批次交易数据 + Merkle证明
  │
  ▼
┌─────────────────────────────────────────┐
│          PROVING CIRCUIT                │
│                                         │
│  for each transaction in batch:         │
│    ├─ ① 验证发送方 ∈ 状态树 (Merkle proof) │
│    ├─ ② 验证余额 >= 转账金额              │
│    ├─ ③ 验证 nonce 正确                 │
│    ├─ ④ 更新发送方: balance-=amt, nonce++ │
│    ├─ ⑤ 重新哈希 → 产生中间状态根          │
│    ├─ ⑥ 验证接收方 ∈ 新状态树 (Merkle proof)│
│    ├─ ⑦ 更新接收方: balance+=amt         │
│    └─ ⑧ 重新哈希 → 产生下一个中间状态根     │
│                                         │
│  输出: 最终状态根 + ZK Proof             │
└─────────────────────────────────────────┘
  │
  ▼
提交到 L1 验证合约
```

### 3.2 电路的输入

| 类型 | 内容 | 是否公开 |
|---|---|---|
| **公开输入 (Public Inputs)** | Pre-state root、Post-state root、Batch root | ✅ L1 可见 |
| **隐私输入 (Witness)** | 每笔交易详情、Merkle proofs、中间状态根 | ❌ 不公开 |

> 这就是"零知识"的体现：L1 不需要看到每笔交易的具体内容，也能确信它们被正确执行了。

### 3.3 逐笔验证过程

电路不是简单地验证最终结果，而是**逐笔、逐步**地验证状态转换：

```
初始状态: state_root₀ (即 pre-state root)

交易1:
  sender₁ 在 state_root₀ 的树中? → Merkle proof ✓
  sender₁.balance ≥ amount₁?     → 约束检查 ✓
  → 更新 sender₁ → 新中间根 root_s₁
  receiver₁ 在 root_s₁ 的树中?    → Merkle proof ✓
  → 更新 receiver₁ → 新中间根 root_r₁

交易2:
  sender₂ 在 root_r₁ 的树中?     → Merkle proof ✓
  sender₂.balance ≥ amount₂?    → 约束检查 ✓
  → 更新 sender₂ → root_s₂
  receiver₂ 在 root_s₂ 的树中?   → Merkle proof ✓
  → 更新 receiver₂ → root_r₂

...重复 N 次...

最终: root_rₙ = post-state root
```

**关键洞察**：每一步的状态根更新都利用了 Merkle 证明的特性——只需重新哈希被修改的那条 Merkle 路径即可得到新根，无需重建整棵树。

### 3.4 为什么"无法作弊"？

电路约束是**硬编码的数学条件**：

```
约束1: 新余额 = 旧余额 - 转账金额   (不能凭空减少不同的金额)
约束2: 旧余额 >= 转账金额           (不能透支)
约束3: nonce_new = nonce_old + 1   (不能重放)
约束4: Merkle proof 必须验证通过    (账户必须真实存在)
约束5: 哈希计算必须正确             (状态根必须如实计算)
```

如果 operator 试图：

| 作弊行为 | 后果 |
|---|---|
| 伪造一笔不存在的交易 | Merkle proof 验证失败 → 电路不满足 |
| 篡改某人的余额 | 哈希不一致 → 电路不满足 |
| 重复一笔旧交易 | nonce 约束失败 → 电路不满足 |

> **电路不满足 = 数学上无法生成有效 proof。这是密码学强制约束，不是人为监管。**

---

## 四、为什么 L1 只需验证一次？

### 4.1 核心原理：SNARK/STARK 的简洁性（Succinctness）

```
传统方式: L1 逐笔重放 1000 笔交易 → O(1000) 的验证成本
ZK 方式:  电路把 1000 笔交易的约束全部编码 → 生成 1 个证明 → L1 只验证这个证明 → O(1) 的验证成本
```

电路把所有约束编织成一张巨大的数学约束网，然后通过密码学压缩成一个极小的证明。验证这个证明 = 同时验证所有约束。

### 4.2 多项式承诺的力量

电路约束被转化为多项式等式：

$$P(x) = Q(x) \cdot R(x)$$

验证者不需要计算整个多项式：
1. 验证者随机选一个点 $s$（通过随机预言机）
2. 证明者提供 $P(s), Q(s), R(s)$ 在该点的取值
3. 验证者只需检查 $P(s) \stackrel{?}{=} Q(s) \cdot R(s)$

根据 **Schwartz-Zippel 引理**：如果两个多项式不等价，它们在随机点相等的概率 ≤ $\frac{d}{|\mathbb{F}|}$（可忽略不计）。

> **检查 1 个点 = 检查整个多项式 = 检查全部约束。**

### 4.3 复杂度对比

| | 证明生成 | 证明验证 |
|---|---|---|
| SNARK | 线性 O(n) | **常数 O(1)** |
| STARK | O(n·log n) | O(log² n) |

无论批次里有 10 笔还是 10000 笔交易，L1 验证成本约 50 万 gas（恒定）。

### 4.4 递归证明的叠加效应

SNARK 可以验证 SNARK：

```
Block1 proof + Block2 proof + ... + BlockN proof
         │
         ▼
    递归聚合电路
         │
         ▼
    1 个最终 proof  →  L1 验证 1 次 = N 个区块全部最终确定
```

### 4.5 形式化总结

$$\underbrace{\text{Verify}(proof, pre\_root, post\_root)}_{\text{L1 只做这一次}} \Rightarrow 
\begin{cases}
\forall tx_i \in batch: \\
\quad sender_i \in StateTree(pre\_root) \\
\quad balance_i \ge amount_i \\
\quad nonce_i \text{ correct} \\
\quad \ldots
\end{cases}$$

> **一次验证 → 蕴含全部约束成立。这不是工程优化，是密码学定理保证的。**

---

## 五、权威参考文档

| 文档 | 链接 | 说明 |
|---|---|---|
| 以太坊官方文档 - ZK-Rollups | [ethereum.org](https://ethereum.org/developers/docs/scaling/zk-rollups/#how-do-zk-rollups-work) | 本文档的主要来源，Validity proofs 章节 |
| Vitalik: ZK-SNARKs Under the Hood | [vitalik.eth.limo](https://vitalik.eth.limo/general/2017/02/01/zk_snarks.html) | SNARK 电路原理深入解释 |
| Vitalik: How do SNARKs possible? | [vitalik.eth.limo](https://vitalik.eth.limo/general/2021/01/26/snarks.html) | 从零解释 SNARK 的数学构造 |
| Vitalik: Different types of zkEVM | [vitalik.eth.limo](https://vitalik.eth.limo/general/2022/08/04/zkevm.html) | zkEVM 分类与电路设计权衡 |
| ZKEVM Specs (EF 项目) | [GitHub](https://github.com/privacy-scaling-explorations/zkevm-specs) | 以太坊基金会 zkEVM 电路规范 |
| yezhang: Intro to zkEVM | [HackMD](https://hackmd.io/@yezhang/S1_KMMbGt) | zkEVM 电路设计全面介绍 |
| ZK-SNARK 学术论文 | [arXiv](https://arxiv.org/abs/2202.06877) | SNARK 形式化论文 |
| ZK-STARK 学术论文 | [IACR ePrint](https://eprint.iacr.org/2018/046) | STARK 形式化论文 |
| Schwartz-Zippel 引理 | [Wikipedia](https://en.wikipedia.org/wiki/Schwartz%E2%80%93Zippel_lemma) | 多项式恒等检验的数学基础 |
| KZG 多项式承诺 | [Dankrad Feist](https://dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html) | 多项式承诺机制详解 |
