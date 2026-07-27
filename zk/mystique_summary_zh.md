# Mystique 论文中文总结

> 论文: "Mystique: Efficient Conversions for Zero-Knowledge Proofs with Applications to Machine Learning"  
> 作者: Chenkai Weng, Kang Yang, Xiang Xie, Jonathan Katz, Xiao Wang  
> 会议: USENIX Security 2021  
> 本文档用途: 作为后续阅读和问答的主索引。若后续提问没有特别说明，我会优先依据这份总结回答。

## 0. 一句话概括

Mystique 是一个基于 sVOLE 的交互式零知识证明系统，核心贡献不是单个通用证明协议，而是一组高效的"转换"能力: 算术值和布尔值互转、公开承诺值和私有认证值互转、定点数和浮点数互转，再加上面向神经网络推理的矩阵乘法优化和 TensorFlow/Rosetta 集成，使得对 ResNet-50/ResNet-101 这类大型模型做 ZK 推理证明成为可运行系统。

## 1. 论文要解决的问题

零知识证明可以让一方证明自己知道某个秘密 witness，且不泄露 witness 本身。论文关注机器学习场景，尤其是神经网络推理，包括:

- **规避攻击证明**: 证明者知道两个非常接近的输入，却能让公开模型输出不同分类，从而证明模型存在漏洞，但不泄露具体攻击样本。
- **正确推理证明**: 模型服务方把模型参数保持私有，但向用户证明自己确实用承诺过的模型对用户输入进行了推理。
- **私有 benchmark 证明**: 测试数据方把私有测试集公开承诺，模型训练者提交模型后，测试数据方证明自己确实在承诺数据上评估了该模型。

现有方案的主要不足:

- zk-SNARK 证明短、验证快，但 prover 内存通常与 statement size 成比例。论文提到十亿级约束可能需要约 640 GB prover 内存。
- sVOLE、privacy-free garbled circuits、MPC-in-the-head 等方案在时间和内存上更可扩展，但对公开承诺数据、混合算术/布尔计算、大规模 ML 推理支持不足，通信也较大。
- 递归 SNARK 理论上能组合，但当时具体性能仍不理想。

Mystique 的目标是让 ZK 系统同时具备:

- 可扩展到大型神经网络。
- 能处理公开承诺数据和私有 witness。
- 能在算术电路、布尔电路、浮点/定点表示之间高效切换。
- 尽量保持 ML 推理精度。

## 2. 核心贡献

论文贡献可以归为五类:

1. **算术/布尔值转换**  
   设计 ZK-friendly edaBits，也就是 zk-edaBits，用于在 sVOLE ZK 的认证值体系中把域元素和 bit decomposition 高效关联起来，从而实现 A2B 和 B2A。

2. **公开承诺值到私有认证值的转换**  
   设计 commitment-authentication conversion。公开承诺适合放到 bulletin board、网站或区块链上，私有认证值适合 sVOLE ZK 内部计算。该协议把两者连接起来。

3. **定点/浮点转换**  
   神经网络线性层适合在有限域上的定点数中计算，非线性层如 ReLU、Sigmoid、SoftMax、BatchNorm 更适合 IEEE-754 单精度浮点。论文实现二者转换来兼顾效率和精度。

4. **矩阵乘法 ZK 优化**  
   基于 Freivalds 思想，用随机线性组合证明 `A * B = C`，使 ZK 中需要证明的私有乘法数量从朴素的 `O(n^3)` 降到近似矩阵维度级别。与当时最优矩阵乘法 ZK 证明相比，执行时间提升约 7 倍。

5. **系统实现与实验**  
   把 Mystique 集成到 Rosetta，一个基于 TensorFlow 的隐私保护计算框架中。实验支持 LeNet-5、ResNet-50、ResNet-101，在 CIFAR-10 上几乎无精度损失。

## 3. 背景机制

### 3.1 IT-MAC 认证值

Mystique 依赖信息论 MAC。验证者持有全局秘密 MAC key `Delta`，证明者知道值 `x` 和 tag `M`，验证者持有 local key `K`，满足:

```text
M = K + Delta * x
```

记作认证值 `[x] = (x, M, K)`:

- 证明者持有 `(x, M)`。
- 验证者持有 `K`。
- `Delta` 只有验证者知道。

认证值有加法同态性，所以线性组合可以本地完成，基本不需要通信。非线性乘法则需要一致性检查。

### 3.2 sVOLE-based ZK 的基本流程

论文使用近期 sVOLE-based ZK 方案作为底层。可理解为 commit-and-prove:

1. **预处理**  
   证明者和验证者通过 sVOLE 生成大量随机认证值 `[r]`，用于后续承诺 witness wires 和乘法门输出。

2. **承诺线值**  
   对真实 wire value `x`，证明者发送 `d = x - r`。双方计算 `[x] = [r] + d`。因为 `r` 随机，`d` 不泄露 `x`。

3. **批量检查乘法门**  
   对乘法门 `z = x * y`，协议把正确性转化为关于验证者秘密 `Delta` 的等式。若证明者作弊，等式成立概率很低。所有乘法门通过随机线性组合批量检查，可用 Fiat-Shamir 变成非交互形式。

### 3.3 功能接口

论文定义了理想功能 `F_authZK`，支持:

- 初始化全局 MAC key。
- 输入 witness 并生成认证值。
- 输出认证值。
- 随机认证值。
- 线性组合。
- 乘法。
- 算术到布尔转换 `convertA2B`。
- 布尔到算术转换 `convertB2A`。
- 公开承诺值到认证值转换 `convertC2A`。

## 4. 算术/布尔转换

### 4.1 为什么需要转换

神经网络和很多程序同时包含:

- 算术友好的部分: 矩阵乘法、卷积、线性层。
- 布尔友好的部分: 比较、条件、位操作、IEEE-754 浮点逻辑。

如果只用算术电路模拟 bit decomposition，开销高；如果只用布尔电路做大规模线性代数，也很慢。Mystique 因此需要在两种表示之间频繁转换。

### 4.2 zk-edaBits

一个 zk-edaBit 包含:

```text
([r_0]_2, ..., [r_{m-1}]_2, [r]_p)
```

其中:

```text
r = sum_i r_i * 2^i mod p
```

含义是同一个随机数 `r` 同时以布尔 bit 形式和算术域元素形式被认证。

构造思路:

- 先生成可能有问题的候选 zk-edaBits。
- 用 cut-and-bucketing 技术抽查和组合。
- 每个 bucket 中选择一个保留，其余用于 combine-and-open 检查。
- 用 `AdderModp` 布尔电路检查 bit 表示和域表示的一致性。

定理 1 给出安全性: 协议在 `F_authZK` hybrid model 中 UC-realizes `F_zk-edaBits`，统计误差由 bucket 大小 `B`、牺牲数量 `c` 和目标数量 `N` 控制。论文例子: `N = 10^6` 时可选 `B = 3, c = 2`，达到至少 40-bit 统计安全。

### 4.3 A2B: arithmetic to Boolean

输入: 算术认证值 `[x]_p`。  
输出: bit 认证值 `([x_0]_2, ..., [x_{m-1}]_2)`。

协议:

1. 取一个随机 zk-edaBit `([r_i]_2, [r]_p)`。
2. 计算并打开 masked value:

```text
z = x - r mod p
```

3. 把公开的 `z` 分解成 bits `z_i`。
4. 用布尔电路 `AdderModp` 计算:

```text
x_bits = z_bits + r_bits mod p
```

由于 `r` 随机，公开 `z` 不泄露 `x`。

定理 2: A2B 协议在 `(F_zk-edaBits, F_authZK)` hybrid model 中实现 `convertA2B`，统计误差为 `1 / p^k`。

### 4.4 B2A: Boolean to arithmetic

输入: bit 认证值 `[x_0]_2, ..., [x_{m-1}]_2`。  
输出: 算术认证值 `[x]_p`。

协议:

1. 取随机 zk-edaBit `([r_i]_2, [r]_p)`。
2. 用 `AdderModp` 在布尔域中计算:

```text
z_bits = x_bits + r_bits mod p
```

3. 打开 `z_bits` 得到公开 `z`。
4. 输出:

```text
[x]_p = z - [r]_p
```

定理 3: B2A 协议实现 `convertB2A`。

### 4.5 circuit-based zk-edaBits 优化

论文还提出一种替代方案: 在某些场景中可以通过调用 `F_authZK` 的 input 命令直接生成 circuit-based zk-edaBits，再用 cut-and-bucketing 检查一致性。它能降低总体开销，但会增加 online phase 成本，约增加 `B - 1` 倍。

## 5. 公开承诺值到私有认证值

### 5.1 场景

有些数据需要先公开承诺，例如:

- 模型参数 commitment 发布到网站或区块链。
- 测试集 commitment 发布到公告板。

但 sVOLE ZK 内部需要的是一对证明者/验证者之间的认证值 `[x]`。因此需要把公开承诺数据转换成私有认证数据，并证明二者一致。

### 5.2 混合承诺方案

直接用 SHA-256 之类 hash 承诺大量数据会导致 ZK 电路很大。论文使用 hybrid commitment:

1. 证明者选随机 key:

```text
sk <- {0,1}^lambda
```

2. 用慢承诺只承诺 key:

```text
com_0 = H(sk, r)
```

论文指出由于 `sk` 熵足够高，实际也可去掉 `r`，直接用 `H(sk)`。

3. 对每个数据块 `x_i`，用 PRF 掩码:

```text
c_i = PRF(sk, i) + x_i
```

如果是 bit 串可理解为 xor；在一般域 `F_q` 中是域加法。

4. 对所有 `c_i` 建 Merkle tree，公开:

```text
(com_0, com_1)
```

其中 `com_1` 是 Merkle root。

### 5.3 转换协议 C2A

对某个 committed block `x_i`:

1. 证明者发送 `c_i` 和 Merkle path，验证者检查 `c_i` 确实属于 root `com_1`。
2. 双方通过 `F_authZK` 得到 `sk`、随机数等的认证 bit。
3. 证明者在 ZK 中证明 `com_0 = H(sk, r)`。
4. 双方计算:

```text
[x_i] = c_i - PRF([sk], i)
```

这样公开承诺数据被转成私有认证值，同时证明它确实来自同一 commitment。

定理 4: 在随机预言机模型中，若 `H` 是 random oracle，`PRF` 是伪随机函数，则协议实现 `convertC2A`。

### 5.4 PRF 实例化和性能

论文用 LowMC 实例化 PRF，原因是 LowMC 的 AND gate 数量少，适合 ZK 电路。优化点:

- 同一个 `sk` 会被用于很多 block，可预计算 key 相关矩阵乘法。
- 选择 64-bit block size，减少 XOR 操作。
- 使用约 `2^30` block 的 data complexity，足够承诺约 8 GB 数据；更大数据可换新 PRF key。

性能对比表:

| 方案 | This work | SHA-256 | LowMC-256 |
|---|---:|---:|---:|
| 时间 | 55 us | 395 us | >= 1000 us |
| 通信 | 62 bits | 705 bits | 49 bits |

该协议约能每秒转换 18,000 个 64-bit committed values，即约 144 KB/s。

## 6. 面向 ML 的优化

### 6.1 矩阵乘法证明

朴素证明 `A * B = C` 需要证明大量乘法。Mystique 使用 Freivalds 风格随机线性组合。

设:

```text
A in F_q^{n*m}
B in F_q^{m*l}
C in F_q^{n*l}
```

验证者随机取:

```text
u in F_{q^k}^n
v in F_{q^k}^l
```

把矩阵等式压缩成标量等式:

```text
u^T * A * B * v = u^T * C * v
```

双方本地计算:

```text
[x]^T = u^T * [A]
[y]   = [B] * v
[z]   = u^T * [C] * v
```

然后只需证明:

```text
[x]^T * [y] = [z]
```

定理 5: 该矩阵乘法协议实现标准 ZK 功能，soundness error 为 `3 / q^k`。

进一步优化: 用最新 ZK 协议证明多变量多项式 `f(x,y,z) = x^T y - z = 0`，通信降到 `O(k log q)`，与中间维度 `m` 无关。

### 6.2 定点数和浮点数

线性层使用有限域定点数，非线性层使用 IEEE-754 单精度浮点。

定点编码:

```text
Encode(x in Z) = x mod p
Decode(a in F_p) =
  a,     if a <= (p - 1) / 2
  a - p, if a >  (p - 1) / 2
```

实数 `x` 用精度参数 `s` 编码为:

```text
Encode(floor(2^s * x))
```

实现参数:

- 使用 Mersenne prime `p = 2^61 - 1`。
- 精度参数 `s = 16`。
- 把定点数限制在 30-bit range 内，为矩阵乘法留出 overflow slack。

浮点/定点转换:

- 使用 EMP 中符合 IEEE-754 的单精度电路。
- 主要成本来自 private logical shift。
- 论文指出，未经优化时 fixed-to-float 会比 float-to-fix 慢约 3 倍，因为相关 private logical shift 的位宽更大。
- 优化思路: 让证明者把转换后的浮点数作为 extended witness 提供，只证明 float-to-fix 方向。最终 benchmark 中两个方向都约为 46 us。

### 6.3 TensorFlow/Rosetta 集成

Mystique 被集成到 Rosetta，后端 C++ 实现，前端保留 TensorFlow/Python 接口。

集成分两步:

- **Static pass**: TensorFlow graph 被转换为 secure graph。普通边/节点变成携带隐私类型和协议信息的 secure edges/operators。
- **Dynamic pass**: 图执行时，把 string-type 数据转换成 ZK-friendly authenticated values，调用底层 ZK operator，再把输出转换回框架可传递的数据形式。

已实现/支持的 ML 操作包括:

- Matrix multiplication / convolution 类线性层。
- ReLU。
- Sigmoid。
- Max Pooling。
- SoftMax。
- Batch Normalization。

论文强调其它浮点非线性操作也可相对容易添加。

## 7. 实验设置

模型:

| 模型 | 层数 | 参数量 |
|---|---:|---:|
| LeNet-5 | 5 | 62,000 |
| ResNet-50 | 50 | 23.5 million |
| ResNet-101 | 101 | 42.5 million |

环境:

- 两台 Amazon EC2 `m5.2xlarge`。
- 每台 32 GB 内存。
- 使用所有 CPU 资源。
- ResNet-101 最大实验使用约 12 GB 内存。
- 底层 ZK 使用最新 sVOLE-based 协议 Quicksilver/Wolverine 系列中的相关方案。
- 计算安全参数 `lambda = 128`。
- 统计安全参数 `rho >= 40`。

## 8. 基础组件性能

表 2 汇总了不同带宽下基础组件性能。

### 8.1 转换

| 转换 | 50 Mbps | 200 Mbps | 500 Mbps | 1 Gbps |
|---|---:|---:|---:|---:|
| A2B | 107 us | 45 us | 34 us | 29 us |
| B2A | 109 us | 49 us | 38 us | 33 us |
| C2A | 56 us | 55 us | 55 us | 55 us |
| Fix2Float | 50 us | 46 us | 46 us | 46 us |
| Float2Fix | 49 us | 46 us | 46 us | 46 us |

观察:

- A2B/B2A 对带宽较敏感，因为涉及 zk-edaBits 和打开 masked values。
- C2A 主要受 PRF 电路本地计算影响，在 50 Mbps 以上几乎不随带宽变化。
- 定点/浮点转换约 46 us/次。

### 8.2 ML 函数

| 函数 | 50 Mbps | 200 Mbps | 500 Mbps | 1 Gbps |
|---|---:|---:|---:|---:|
| Sigmoid | 2.1 ms | 1.6 ms | 1.6 ms | 1.6 ms |
| Max Pooling 2x2 | 1.6 ms | 0.5 ms | 0.4 ms | 0.4 ms |
| ReLU | 908 us | 262 us | 185 us | 188 us |
| SoftMax-10 | 209 ms | 157 ms | 161 ms | 171 ms |
| Batch Norm | 415 ms | 261 ms | 257 ms | 269 ms |

BatchNorm 和 SoftMax 明显更重，尤其 BatchNorm 是后续 ResNet 推理的最大瓶颈。

### 8.3 矩阵乘法

| 矩阵维度 | 50 Mbps | 200 Mbps | 500 Mbps | 1 Gbps |
|---|---:|---:|---:|---:|
| 512 | 361 ms | 186 ms | 185 ms | 185 ms |
| 1024 | 2.42 s | 1.48 s | 1.39 s | 1.37 s |
| 2048 | 15.19 s | 11.30 s | 10.63 s | 10.39 s |

论文称: 对 1024x1024 矩阵乘法，先前方案在 500 Mbps 下约 10 秒，Mystique 约 1.39 秒，提升约 7 倍。

## 9. ZK 神经网络推理性能

表 3 不包含模型或数据 commitment conversion 的成本，只看推理证明。

### 9.1 通信量

| 模型/输入设置 | LeNet-5 | ResNet-50 | ResNet-101 |
|---|---:|---:|---:|
| Private model, private image | 16.5 MB | 1.27 GB | 1.98 GB |
| Private model, public image | 16.5 MB | 1.27 GB | 1.98 GB |
| Public model, private image | 16.4 MB | 0.53 GB | 0.99 GB |

### 9.2 50 Mbps 网络执行时间

| 模型/输入设置 | LeNet-5 | ResNet-50 | ResNet-101 |
|---|---:|---:|---:|
| Private model, private image | 7.3 s | 465 s | 736 s |
| Private model, public image | 7.5 s | 463 s | 735 s |
| Public model, private image | 6.5 s | 210 s | 369 s |

### 9.3 200 Mbps 网络执行时间

| 模型/输入设置 | LeNet-5 | ResNet-50 | ResNet-101 |
|---|---:|---:|---:|
| Private model, private image | 5.9 s | 333 s | 535 s |
| Private model, public image | 5.5 s | 336 s | 541 s |
| Public model, private image | 4.9 s | 158 s | 262 s |

解读:

- LeNet-5 几秒完成。
- ResNet-50 在 200 Mbps 下约 2.6 到 5.6 分钟。
- ResNet-101 在 200 Mbps 下约 4.4 到 9 分钟。
- 模型私有时比模型公开更慢，因为模型参数相关计算也要在 ZK 中证明。

## 10. 准确率

实验使用 CIFAR-10:

- 10,000 张测试图片。
- 每张图片 `32 x 32 x 3 = 3072 bytes`。
- 10 个类别。

结论:

- ResNet-50 和 ResNet-101 上，ZK 推理相对明文推理的准确率下降仅 `0.02%`。
- 概率向量 `l2` 距离均值:
  - ResNet-50: `0.0011`。
  - ResNet-101: `0.0019`。
- ResNet-101 中 95% 的 case，`l2` 距离小于 `0.006`。
- LeNet-5 中 99.9% 的 case，`l2` 距离小于 `0.006`。

含义: 虽然定点/浮点转换和有限域编码会引入误差，但对 CIFAR-10 推理的最终分类影响极小。

## 11. 端到端应用性能

论文把三类应用连接到 commitment conversion。

额外 conversion 成本:

- 单张 CIFAR-10 图片 3072 bytes，从公开承诺转成私有认证值约 `2.6 ms`。
- 三个模型大小:
  - LeNet-5: `0.25 MB`，转换约 `1.7 s`。
  - ResNet-50: `94 MB`，转换约 `646 s`。
  - ResNet-101: `170 MB`，转换约 `1169 s`。

模型 commitment pull 成本很高，但如果同一 committed model 被多次私有推理复用，可以摊销。

端到端表 4，基于 200 Mbps 和两台 m5.2xlarge:

| 应用 | LeNet-5 | ResNet-50 | ResNet-101 |
|---|---:|---:|---:|
| ZK for evasion attacks | 9.8 s | 316 s | 524 s |
| ZK for genuine inference | 7.2 s | 16.4 min | 28 min |
| ZK for private benchmark | 8.2 min | 4.4 h | 7.3 h |

应用解释:

- **规避攻击证明**: 公共模型下，证明两个相近私有输入得到不同分类。主要成本是跑两次公共模型下的私有输入推理。
- **正确推理证明**: 私有模型被公开承诺，输入公开。主要成本是模型参数 C2A 转换，以及私有模型/公开输入下的正确分类证明。
- **私有 benchmark**: 测试集被公开承诺，模型公开。论文例子中测试集有 100 张图片，需要 100 次推理，因此成本最高。

## 12. 结论和局限

论文结论:

- Mystique 用一组高效转换协议，把 sVOLE-based ZK 扩展到复杂 ML 推理。
- 系统首次支持对超过 100 层的神经网络模型做 ZK 证明，并几乎没有精度损失。
- 在私有图片和私有/公开模型设置下，ResNet-101 推理证明已能在分钟级完成。

论文承认的局限:

- 协议一次只能证明给一个验证者。
- 相比 zk-SNARK 等 succinct ZK，通信成本仍然高。
- Batch Normalization 开销非常高，是重要优化方向。

## 13. 术语速查

- **sVOLE**: subfield vector oblivious linear evaluation，用于高效生成认证随机值。
- **IT-MAC**: 信息论 MAC，形式为 `M = K + Delta * x`。
- **Authenticated value `[x]`**: ZK 内部使用的私有认证值，证明者知道值，验证者能检查一致性。
- **Public commitment**: 公开可验证的承诺，例如 `(com_0, com_1)`。
- **C2A**: committed-to-authenticated，公开承诺值转私有认证值。
- **A2B**: arithmetic-to-Boolean，算术域认证值转 bit 认证值。
- **B2A**: Boolean-to-arithmetic，bit 认证值转算术域认证值。
- **zk-edaBit**: 同一随机数同时以算术形式和布尔 bit 分解形式被认证。
- **BatchCheck / CheckZero**: 批量打开和批量零检查，用随机线性组合降低通信。
- **Freivalds-style check**: 用随机向量把矩阵乘法等式压缩成标量等式。
- **Rosetta**: 基于 TensorFlow 的隐私保护框架，Mystique 集成于其中。

## 14. 后续阅读建议

如果按技术难度阅读，建议顺序:

1. 先读 Introduction，理解三个 ML 应用和为什么现有 ZK 不够。
2. 读 2.3 和 2.4，掌握 IT-MAC 与 sVOLE-based ZK。
3. 读 Section 3 技术总览，先建立全局结构。
4. 深入 Section 4 的 zk-edaBits、A2B、B2A。
5. 再读 Section 5 的公开承诺到认证值转换。
6. 最后读 Section 6 和 7，把协议映射到 ML operator、TensorFlow 和实验结果。

如果后续要提问，可以按这些主题问:

- "zk-edaBit 为什么能做 A2B/B2A?"
- "C2A 为什么需要 PRF + Merkle tree?"
- "矩阵乘法为什么只需随机向量检查?"
- "为什么 BatchNorm 这么慢?"
- "Mystique 和 zk-SNARK 的取舍是什么?"
- "论文里 ResNet-101 的 28 分钟包括什么成本?"
