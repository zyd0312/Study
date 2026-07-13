# L2BEAT Stages 框架：如何评估一个 Rollup 的成熟度

> **来源**：L2BEAT 论坛，《The Stages Framework》，作者 donnoh，分类 Methodology & Framework
> **首发**：2024 年 6 月 6 日（更早的引入文见 L2BEAT 的 Medium 文章 "Introducing Stages"）｜**最近更新**：2026 年 4 月 30 日（见修订历史）
> **原文链接**：https://forum.l2beat.com/t/the-stages-framework/291
> **一句话结论**：Stages 框架把「这条 rollup 到底有多去信任化」这个模糊问题，拆成一组**可逐条核查的工程条件**——链上数据可用性、可用的证明系统、Security Council 的权限边界、用户强制退出窗口——并按满足程度把 rollup 归入 Stage 0 / 1 / 2。

原文明确声明：**这篇帖子是 Stages 框架最新版本的「真理之源」（source of truth）**，文末附有 changelog。

---

## 为什么需要这个框架

Rollup 在宣传上都自称「继承以太坊安全性」，但实际形态差异极大：有的连状态根都不上 L1，有的证明系统根本没启用，有的多签可以在几秒内改掉合约把用户资金搬走。

Stages 框架的作用是：**把「信任假设」变成一张清单**。它不问「你用的是 optimistic 还是 ZK」，只问一串二元问题——你把数据放 L1 了吗？证明系统真的在跑吗？谁能阻止我提款？我想跑路时有多少天窗口？

比喻上，L2BEAT 用「训练轮（training wheels）」来描述 Stage 0/1 阶段仍然存在的中心化后门：项目方为了在早期能修 bug、能救急，保留了升级权和 Security Council 的干预权。Stages 就是在衡量**训练轮还剩几个、什么时候能拆掉**。

> **批注**：这个框架的隐含立场是——早期保留训练轮不是罪，**隐瞒还带着训练轮才是罪**。所以框架的核心产出不是「打分」，而是「公开披露」。

---

## Stage 0 / 1 / 2 的定义与准入条件

### 总览表

| 维度 | Stage 0 | Stage 1 | Stage 2 |
| --- | --- | --- | --- |
| 自称 rollup | 必须（且不能是「不一定跟随以太坊共识」的链） | 继承 Stage 0 | 继承 |
| 状态根发布到 L1 | 必须 | 继承 | 继承 |
| 数据可用性（DA）在 L1 | 必须（重建 L2 状态所需的全部数据） | 继承 | 继承 |
| 节点软件源码可得（source available） | 必须（任何人可重建状态、独立校验状态根） | 继承 | 继承 |
| 「正经的」证明系统（proper proof system） | 必须 | 继承 | 继承 |
| 至少 5 个外部欺诈证明提交者 | 必须（2025-06-20 起从 Stage 1 下放到 Stage 0） | 继承 | 继承 |
| 欺诈证明系统是否**无许可** | 不要求 | 不要求 | **必须**：任何人都能提交，不限白名单 |
| 阻止/伪造 L2→L1 消息的唯一途径 | — | **必须是攻陷 ≥75% 的 Security Council**（bug 除外） | — |
| Security Council 配置 | — | **必须**：≥8 名参与者、75% 门槛、成员多元（不同公司/司法辖区）、身份或化名公开 | 权力被严格限制（见下） |
| Security Council 之外发起的升级 | — | 允许，但**必须给 ≥5 天退出窗口（exit window）** | 需给用户 **≥30 天**退出时间（含 DAO 发起的升级） |
| Optimistic rollup 的挑战期 | — | **≥5 天**（人为的 grace period / air gap / execution delay 都要计入这 5 天） | — |
| Security Council 能否随意干预 | — | 可以（作为证明系统的兜底） | **只能针对链上可裁定（adjudicable onchain）的 bug 出手** |

> 注：原文以问答式条目组织，并无正式表格；上表是我按原文条目整理的（**批注**：表格结构为整理所加，条件文字忠于原文）。

### Stage 0：这确实是一条 rollup

Stage 0 回答的是「**它在结构上算不算 rollup**」，六个问题：

1. **项目自称 rollup 吗？** 用来把 rollup 和那些「明确社会性地表示自己不一定跟随以太坊共识、可能最终偏离」的链区分开。（这条在 2025-08-05 被改写过。）
2. **L2 状态根发布到 L1 了吗？** 这是提款（withdrawal）能成立的前提；不发状态根，就缺了「桥接式 rollup」的根本部件。
3. **数据可用性（Data Availability, DA）在 L1 吗？** 重建 L2 状态所需的**全部**数据必须在 L1 可得。
4. **是否有源码可得（source available）的、能从 L1 数据重建状态的节点软件？** 让任何人可以审计、运行、并独立校验被提交的状态根。
5. **是否使用了「正经的」证明系统（proof system）？** 证明系统用来裁定被提交的状态根是否正确：欺诈证明用来**拒绝**无效根；ZK rollup 则是**必须有证明才能接受**状态根。如果 DA 用的是 state diffs，证明系统还必须保证**所有状态变更都被包含在 diff 里**。
6. **至少有 5 个外部主体能提交欺诈证明吗？** 欺诈证明系统只需要「一个诚实的人」来核验并挑战，但框架要求这个集合至少开放给 5 个外部主体。

### Stage 1：训练轮还在，但已经拧紧了

Stage 1 有一条**高层指导原则**，原文用箭头单独标出：

> 除了 bug 之外，**唯一**能无限期阻止一条 L2→L1 消息（例如一笔提款）、或推送一条无效 L2→L1 消息（例如一笔无效提款）的方式，是**攻陷 ≥75% 的 Security Council**。

附带的**假设**（原文以 warning 标注）：如果 proposer 集合对任何有足够资源的人开放，则假定任何时刻至少有一个活跃的 proposer（即 **1-of-N 假设，N 无上界**）；但**不假设它不审查（non-censoring）**。

具体条件：

- **升级路径**：由 Security Council 之外的主体发起的升级是允许的，前提是提供**至少 5 天的退出窗口（exit window）**。
- **挑战期**：Optimistic rollup 的挑战期必须 **≥5 天**；任何人为设置的「宽限期（grace period）」「air gap」「执行延迟（execution delay）」都算进这 5 天里（也就是说不能靠叠加各种延迟来凑，也不能靠它们来变相缩短）。
- **Security Council 是否配置妥当？** 它是系统的安全兜底，在证明系统出 bug 或出问题时介入。要求：
  - 多签（multisig）形式，**≥8 名参与者**，**75% 共识门槛**；
  - 成员足够去中心化和多元，最好来自**不同公司、不同司法辖区**；
  - **同一实体的多个地址会被合并计为一个**；实体门槛按「达到签名者门槛所需的最少实体数」计算；
  - 出于透明与问责，成员的**身份或化名必须公开披露**。

原文另外指向两篇补充帖：Stage 1 高层原则的详细解释与示例（forum 帖 338），以及 7 天→5 天最短挑战期缩减的理由（forum 帖 425）。

### Stage 2：训练轮拆掉，代码是最终权威

1. **欺诈证明系统是否无许可（permissionless）？** 必须完全去中心化、对所有人开放，不能只有白名单主体能提交欺诈证明——目的是让系统接受整个社区的集体审视，而不是被有限几个实体掌控。
2. **用户在遭遇不想要的升级时，是否有至少 30 天可以退出？** 必须给 **≥30 天**，**包括 DAO 发起的升级**。唯一的例外：如果存在**链上 bug 检测系统**（例如「同一批次能产生两个互相矛盾的有效 ZK 证明」这种可裁定情形），那么针对**已检测到的 bug** 允许即时升级。
3. **Security Council 是否被限制为只能因链上检测到的错误而行动？** 在最终阶段，Security Council（如果还存在）的权力应被**高度限制**：只能在**链上可裁定的 bug（adjudicable onchain bugs）**出现时迅速介入。原文给的例子是 Polygon zkEVM 合约：当同一批 batch 能提交出两个不同的有效证明时，rollup 进入「紧急模式（Emergency Mode）」。

> 原文对 Stage 2 的定性：这让系统更去中心化，对 Security Council 的信任被压到最低，rollup 进一步逼近**信任最小化（trust minimization）的理想——代码本身是最终权威**。

---

## 关键设计考量

**1. 条件全都指向「资金能不能被夺走 / 被冻住」。** 六条 Stage 0 条件里，DA 在 L1 + 源码可得 + 状态根上链，合起来保证「**任何人都能独立算出正确的 L2 状态**」；证明系统保证「**错误的状态根会被拒绝**」。这是安全性的最小闭环。

**2. Security Council 是被明确承认、而不是被藏起来的后门。** 框架没有天真地要求「零信任立刻实现」，而是把这个后门**形式化**：谁在里面（身份公开）、需要多少人（≥8 人 / 75%）、能干什么（Stage 1 可兜底证明系统；Stage 2 只能对链上可裁定 bug 出手）。「同一实体多地址合并计数」这条细则尤其关键——它防止了「8 个签名者其实都是同一家公司的 8 台服务器」这类伪去中心化。

**3. 退出窗口是用户唯一真正的自卫武器。** Stage 1 给 5 天、Stage 2 给 30 天，本质是「**如果你不同意这次升级，你有时间跑**」。挑战期把 grace period / air gap 计入其中，是为了封死「表面 7 天，实际有效挑战窗口只剩几小时」的会计游戏。

**4. 训练轮（training wheels）的隐喻。** Stage 0 = 装着训练轮上路；Stage 1 = 训练轮还在，但被上了锁、有明确的拆卸计划和用户逃生通道；Stage 2 = 训练轮拆掉，只留下一个「车架断了才允许扶一把」的应急机制。

---

## 框架的演进（修订历史）

按原文 changelog 逐条（时间倒序）：

| 日期 | 变更 |
| --- | --- |
| **2026-04-30** | **Stage 1 最短挑战期从 7 天降到 5 天**（rationale 见 forum 帖 425 "Stage 1 update: minimum challenge period reduction from 7d to 5d"）。 |
| 2025-08-05 | 更新「项目是否自称 rollup」这一条，使其**区分「自称 rollup 的链」与「不一定打算跟随以太坊共识的链」**。 |
| 2025-07-31 | 按 6 个月前公布的**新定义**更新 Stage 1 要求（即前述「攻陷 ≥75% Security Council 是唯一途径」的高层指导原则，forum 帖 338）。 |
| 2025-06-20 | 按 "Recategorization"（重新分类）更新 Stage 0 要求：**「至少 5 个外部挑战者」从 Stage 1 下放到 Stage 0**。 |
| 2025-03-04 | 澄清 Stage 1 和 Stage 2 中 **Security Council 规模与门槛的评估方式**。 |
| 2023-12-07 | 更新 **Security Council 的要求**（rationale 为 Medium 文 "Stages update: Security Council requirements"）。 |
| 2023-08-25 | 把 "Optimistic chains" **改名为 Optimiums**（对应 l2beat PR #1823）。 |

> **注意**：正文里的挑战期数字（5 天）与 2026-04-30 那条 changelog 是**一致的**——原文正文已经是缩减后的版本。所以引用「Stage 1 需要 7 天挑战期」的说法在今天已经过时。

原文页面底部还列出了若干**正在讨论中、尚未并入本框架**的提案（**批注**：这些是「相关主题」，不是已生效条款）：
- Stages update proposal: decouple escrows from rollup stages（把托管合约与 rollup stage 解耦）
- Alt-DA Stages Framework - call for feedback（针对非以太坊 DA 的 Stages 框架）
- Cont: Stage 1 challenge period reduction discussion
- Stage 1 requirements update: Security Council walkaway test（Security Council「集体撂挑子」测试）
- The Risk Rosette Framework（风险玫瑰图，L2BEAT 的另一套风险披露框架）

---

## 与 2021 年「乐观 vs ZK」辩论的关系

**批注：这一整节是我的对比分析，不是原文内容。**

2021 年 Deribit Insights 那篇著名的 optimistic rollup vs ZK rollup 讨论（以及那一时期的普遍论调），安全性论证基本靠**定性直觉**：

- Optimistic 阵营：「**一个诚实的人就够了**（1-of-N honest）」——只要有一个人盯着，欺诈就会被挑战掉。
- ZK 阵营：「**我们有密码学保证**」——无效状态根根本上不了链，不需要指望任何人诚实。

这场辩论的问题在于：**它比较的是两种范式的理论极限，而不是两条链的现实状态**。而现实是——

| 2021 年的辩论问题 | Stages 框架给出的可核查版本 |
| --- | --- |
| 「一个诚实的人就够了」 | 那么**这个人存在吗**？→ Stage 0 要求「至少 5 个外部主体能提交欺诈证明」；Stage 2 要求欺诈证明**完全无许可**。 |
| 「他有时间挑战吗？」（默认没人问） | → Stage 1 要求挑战期 ≥5 天，且 grace period / air gap / execution delay 都要计入。 |
| 「ZK 有密码学保证」 | 那么**证明系统真的启用了吗**？→ Stage 0 的「proper proof system」条目；很多所谓 ZK rollup 在很长时间里根本没开验证器。 |
| 「密码学保证 ⇒ 无需信任」 | 那么**谁能升级验证合约**？→ 密码学保证只在合约不被瞬间换掉时成立。Stage 1 的 5 天退出窗口、Stage 2 的 30 天窗口，正是在给这个漏洞打补丁。 |
| （双方都没讨论） | **Security Council 是谁、能干什么** → 变成 ≥8 人 / 75% / 身份公开 / 同实体合并计数 / Stage 2 只能对链上可裁定 bug 出手。 |

**核心转变**：Stages 框架把安全性从「**你属于哪个技术阵营**」重新定义为「**你的信任假设是什么，且它可被外部核查吗**」。在这个框架下，一条挂着 ZK 招牌、但验证器未启用、多签可即时升级的链，评级会**低于**一条老老实实跑欺诈证明、有 5 天挑战期和公开 Security Council 的 optimistic rollup。这在 2021 年的语境里近乎异端，但它才是对用户真正有意义的问法。

换句话说：**2021 年问「哪种证明更强」，Stages 问「你的后门有几个、多深、我能不能在你用它之前跑掉」。**

---

## 关键术语（中英对照）

| 中文 | English | 说明 |
| --- | --- | --- |
| 阶段框架 | Stages Framework | L2BEAT 的 rollup 成熟度分级框架 |
| 数据可用性 | Data Availability (DA) | 重建 L2 状态所需的全部数据必须在 L1 可得 |
| 证明系统 | proof system | 裁定状态根是否正确的机制（欺诈证明或有效性证明） |
| 欺诈证明 | fraud proof | Optimistic rollup 用来拒绝无效状态根的机制 |
| 状态差分 | state diffs | 只发布状态变更而非全部交易数据的 DA 方案 |
| 源码可得 | source available | 节点软件公开可读、可运行、可审计（不必然是开源许可证） |
| 安全委员会 | Security Council | 系统的应急兜底多签；Stage 1 要求 ≥8 人 / 75% 门槛 |
| 多签 | multisig | 多重签名合约 |
| 退出窗口 | exit window | 升级生效前留给用户撤资的时间；Stage 1 ≥5 天，Stage 2 ≥30 天 |
| 挑战期 | challenge period | Optimistic rollup 中可提交欺诈证明的窗口；Stage 1 ≥5 天 |
| 宽限期 / air gap / 执行延迟 | grace period / air gap / execution delay | 各种人为延迟；均计入挑战期，不得用来注水 |
| 无许可 | permissionless | 无白名单，任何人可参与（Stage 2 对欺诈证明的要求） |
| 链上可裁定的 bug | adjudicable onchain bug | 能被合约自动判定的错误（如同批次两个互相矛盾的有效证明） |
| 紧急模式 | Emergency Mode | Polygon zkEVM 检测到矛盾证明时进入的状态 |
| 信任最小化 | trust minimization | Stage 2 追求的理想：代码是最终权威 |
| 训练轮 | training wheels | 早期保留的中心化后门的隐喻 |
| Optimium | Optimium | 使用欺诈证明但 DA 不在 L1 的链（2023-08-25 从 "Optimistic chains" 改名而来） |

---

## 值得记住的要点

1. **Stage 0 ≠ 安全，Stage 0 = 结构上是 rollup**：状态根上链 + DA 在 L1 + 源码可得 + 有证明系统 + ≥5 个外部挑战者。
2. **Stage 1 的一句话定义**：除 bug 外，**唯一**能无限期卡住你提款、或推送无效提款的方式，是攻陷 **≥75% 的 Security Council**。
3. **Stage 1 的硬指标**：Security Council ≥8 人 / 75% 门槛 / 身份公开 / 同实体地址合并计数；外部升级需 ≥5 天退出窗口；Optimistic 挑战期 ≥5 天（含所有人为延迟）。
4. **Stage 2 的硬指标**：欺诈证明**无许可**；任何升级（含 DAO）给用户 **≥30 天**退出时间；Security Council **只能**对链上可裁定 bug 出手。唯一例外是链上 bug 检测系统触发时允许即时升级。
5. **数字会变**：**2026-04-30 起，Stage 1 最短挑战期由 7 天降为 5 天**。看到任何还在说「Stage 1 需要 7 天挑战期」的材料，都是旧版本。
6. **框架本身是活的**：changelog 有 7 条修订，且论坛上还有 escrow 解耦、Alt-DA Stages、Security Council walkaway test 等提案在讨论中。引用时务必回到本帖（forum 帖 291）确认最新版本。
7. **这是从「阵营辩论」到「工程核查表」的范式转移**：不再问 optimistic 还是 ZK，只问信任假设是什么、后门有几个、用户逃生窗口有多长。
