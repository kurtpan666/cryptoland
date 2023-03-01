---
tags:
- ECDSA签名
- SNARKs

title: 使用 SNARKs 批处理 ECDSA 签名
date: 2022-08-01 00:00:00
#published: false
sidebar:
    nav: cryptoland
---

> - 原文：[Batch ECDSA in SNARKs](https://0xparc.org/blog/batch-ecdsa)
> - 作者：John Guibas, Uma Roy
> - 译者：Kurt Pan

{: .info}
我们带来了 [circom-batch-ECDSA](https://github.com/puma314/batch-ecdsa)，一个基于[circom-ECDSA](https://github.com/0xPARC/circom-ecdsa)（由0xPARC 社区中其他人之前完成的工作）之上的概念验证实现，其灵感来自[halo2-batch-ECDSA](https://github.com/privacy-scaling-explorations/halo2wrong/pull/22)，它允许在单个 SNARK 中显著更快地验证一批 ECDSA 签名。

<!--more-->
## 简介

ECDSA 签名是一种广泛使用的密码学原语，可使人确信一条消息只能来自拥有特定公钥的人。在区块链场景中，ECDSA 签名通常用于验证提交的交易是否来自拥有发出交易的账户的人。

0xPARC 的 [circom-ECDSA 项目](https://0xparc.org/blog/zk-ecdsa-1)在一个zkSNARK中实现了 ECDSA 签名验证。这个原语造成了一波应用的小规模爆炸式增长，这些应用利用了在 SNARK 中的 ECDSA 验证来匿名地验证你控制一组地址中的一个以太坊地址。此类项目包括[cabal](https://www.cabal.xyz/)、[zkmessage](https://zkmessage.xyz/)和[heyanon](https://heyanon.xyz/)。

`circom-batch-ECDSA` 实现了一个可以对*批量*的ECDSA 签名进行优化验证的原语。与 `circom-ECDSA` 一样，`circom-batch-ECDSA` 具有广阔的应用场景，从加速 ZK rollups到支持新的身份原语。批处理 ECDSA 未来还可能用于降低optimistic rollups等应用中的`calldata`开销——根据当前的gas标准，粗略的基准测试表明用 SNARK 证明替换 ECDSA 签名可以将`calldata`开销降低多达 18%。

## 动机：降低Optimistic Rollups的交易费？
本节中我们会详述批处理 ECDSA 验证最引人注目的潜在用例之一——帮助 L2 rollups节省 gas 开销！我们还没有用我们的电路实现这个，但是这个用例是实现这个原语背后的一个很大的动机。

Optimistic rollups，例如[Optimism](https://www.optimism.io/)或[Arbitrum](https://arbitrum.io/)，使用“定序器”离线处理批量交易，然后定期将状态根更新发布到以太坊主网。作为更新的一部分，他们将这批中的所有交易发布到 `calldata`，以便所有用户都可以使用此信息（并且可以供希望通过发布欺诈证明来挑战状态更新的人使用）。您可以看到分别为[Optimism](https://etherscan.io/address/0x4bf681894abec828b212c906082b444ceb2f6cf6)和[Arbitrum](https://etherscan.io/address/0x9685e7281fb1507b6f141758d80b08752faf0c43#code)实现此功能的合约。*降低optimistic rollups开销的关键就是尽可能地压缩这些数据。*

我们的主要洞察在于 ECDSA 签名是“不可伪造的”数据。如果我们可以证明某人知道对一个消息的有效签名，则可以替换签名本身。我们的想法是用单个批处理 ECDSA SNARK 替换一批交易上的所有签名，该SNARK证明了定序器知道所有这些交易的有效签名。除了 SNARK 证明，我们还必须包括每笔交易的发送者地址，因为`ecrecover`通常依赖于交易的存在来恢复发送者。

ECDSA 签名每个是 64 字节，而 groth16 证明是 131 字节。如果我们能够用 1 个批处理 ECDSA SNARK 和 50 个地址（每个 22 字节）替换 50 个 ECDSA 签名，那么我们就将3200 字节替换为1231字节，从而显着节省了成本。一般来说，我们可以替换掉带有$N$个22 字节地址的$N$个64 字节的签名，通过引入 131 字节的固定开销（SNARK 证明的大小）。

在对CTC的每个承诺中，Optimism 都会放置大约 200 个 ECDSA 签名，这需要花费 $200000=16\cdot64\cdot200$ gas。如果我们将其替换为单个 SNARK 证明（131 字节）和发送者地址（每笔交易 22 字节），则该gas开销降低到 $66 ,000$ gas。以每5分钟一个CTC块和30gwei 的 gas 开销计算，每天可以节省约 1.1 ETH！

尽管无论电路的复杂性如何，SNARK 证明的大小和验证开销始终相同，但优化电路有助于满足rollups的延迟要求，因为必须为每个块生成 SNARK 证明。

## 数学回顾

下面给出对 **ECDSA 签名生成和验证过程**的快速数学回顾（主要遵循[Wikipedia 文章](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm)中的术语）。这是去了解为什么 `circom-batch-ECDSA` 是比将 `circom-ECDSA` 电路包在一个 for 循环中更有效的优化方法的重要背景。

固定$n$阶椭圆曲线群，生成元为$G$ 。令私钥为$d$ ，相关公钥为 $Q=d \cdot G$ 。

给定消息$m$，以如下方式生成由公钥$Q$签署的对消息$m$的签名：

- 计算消息$m$的哈希$h=H(m)$ , 假设 $H(m) \in$ $[1, n)$
- 随机选择 $k \in[1, n)$
- 计算曲线点 $\left(x_1, y_1\right)=k \cdot G$
- 计算$r \equiv x_1 \bmod n$
- 计算$s \equiv k^{-1}(h+r \cdot d) \bmod n$

要验证对公钥$Q$和消息$m$的签名$(r, s)$，需进行如下检验：
- $Q$ 是一个有效公钥
- 是否$(x, y)=\left(H(m) s^{-1}\right) \cdot G+\left(r s^{-1}\right) \cdot Q$ 
- 是否 $(x, y) \neq O$ (即不是无穷远点) 
- 是否$r \equiv x \bmod n$.


对于批处理 ECDSA，我们还需要熟悉一种称为 **ECDSA\*** 的 ECDSA 变体，其中用户除了提供$r,s$，还提供一个$r'$ 使得$R=( r ,r')=( x_1,y_1)$，其中验证条件如下：

-   $Q$ 是一个有效公钥
-   验证$R=\left(H(m) s^{-1}\right) \cdot G+\left(r s^{-1}\right) \cdot Q$

在 ECDSA* 中，用户通过提供$r$是$x$坐标的椭圆曲线点的$y$坐标来简化验证过程。给定一个常规的 ECDSA 签名（只有r), 可以由椭圆曲线方程推导出$r'$。现在起，我们令$R$表示完整的椭圆曲线点，因为我们需要它来进行批量验证。注意如果签名有效，则组成$R=( r ,r')$的$r'$一直存在，所以$R$的存在性得以保证。

## 批处理 ECDSA 验证

### 朴素方法

朴素的ECDSA批处理验证会输入一批（$B$个）ECDSA签名 $\left(r_i, s_i\right)$ ，分别验证所有的签名对消息$m_i$和公钥$Q_i$都是有效的。一种朴素的批处理验证方法就是简单地遍历这一批中的签名，通过计算上述方程单独验证每个签名，如果所有签名都得以验证，则返回 `true` 。使用 `circom-ecdsa` 库，我们可以朴素地在 SNARK 中实现批处理 ECDSA 验证，只需遍历所有$B$个签名并将 ECDSA 验证原语依次应用于每个签名。

### 能做得更好吗

事实证明我们可以！如该[笔记](https://eprint.iacr.org/2012/582.pdf)所述，可能在保持正确的安全保证下，将对$B$个ECDSA签名的验证约简为单个代数方程。

给定一个随机域元素$t$，批处理验证签名等价于检验下列椭圆曲线点等式：（方程（1））

$$\sum_i t^i \cdot R_i=\left(\sum_i t^i\left(h_i s_i^{-1}\right)\right) \cdot G+\left(\sum_i t^i\left(r_i s_i^{-1}\right)\right) \cdot Q_i$$

$R_i$ 指$\left(r_i, r_i^{\prime}\right)$，即这里是ECDSA\*签名。

为什么这等价于单独验证所有$B$个签名？考虑下列多项式方程（方程（2））：

$$
\sum_i x^i \cdot R_i=\sum_i x^i\left(\left(h_i s_i^{-1}\right) \cdot G+\left(r_i s_i^{-1}\right) \cdot Q_i\right)
$$

注意到这个多项式方程成立当且仅当所有$B$个签名是有效的（因为左右两边$x^i$ 的系数正是ECDSA\*验证方程中的左右两边）。但是根据[Schwartz Zippel 引理](https://en.wikipedia.org/wiki/Schwartz%E2%80%93Zippel_lemma)，如果我们在某个随机点$t$处对两个多项式求值且两个值相等，这意味着这两个多项式实际上以非常高的概率是相同的。可以注意到方程 (1) 只是对方程 (2)在随机点$t$处的求值结果（以及带有一些系数分布）。因此根据 Schwartz Zippel 引理，如果我们选择一个随机的$t$并证明方程（1）中的等式，则等价于验证所有$B$个签名。

Schwartz Zippel 引理要想成立$t$必须是“随机”的！在 SNARK 电路中，没有对随机性的（像 CPU 上一样的）“原生”访问，所以我们通过将所有输入哈希到SNARK中，构造了一个确定性伪随机的$t$。

### 窗口法和平摊倍点

在SNARKs中实现这些密码学协议时，标量乘法和椭圆曲线点的加法会非常昂贵，这是因为在以太坊中用于ECDSA签名的椭圆曲线（例如[secp256k1](https://en.bitcoin.it/wiki/Secp256k1)）对于SNARKs中的算术来说是“域外”的存在。

计算方程 (1) 中的代数表达式可能看起来很昂贵，但对于计算椭圆曲线点的线性组合，有几个相当高效的优化方法。为了高效计算椭圆曲线点$P$的标量$d$倍数，我们可以使用**窗口法**来高效地计算标量倍数。

在窗口法中，选择一个窗口大小$w$ 并对$d=0, \ldots, 2^{w-1}$计算全部$2^w$ 个$dP$值。算法使用$d$ 在基$2^w$ 下的表示$\left(d_0, \ldots, d_m\right)$来高效地计算$dP$：

```python
Q = 0
for i in reversed(range(m+1)):
  Q = point_double_repeat(Q, w)
  if d[i] > 0:
      Q = point_add(Q, d[i]*P) # 使用的d[i]*P的预计算值
return Q
```

`point_double_repeat`通过迭代倍点计算$2^w \cdot Q$ 。

我们可以修改这个算法来计算椭圆曲线点的线性组合$d^{(1)} P_1+\ldots+d^{(B)} P_B$ 同时只保留$m$次`point_double_repeat`调用：

```python
Q = 0
for i in reversed(range(m+1)):
    Q = point_double_repeat(Q, w)
    for j in range(B):
        if d[i,j] > 0:
            Q = point_add(Q, d[i,j]*P) # 使用的d[i][j]*P的预计算值
return Q
```

### 直观理解

对上述算法中哪些操作是“昂贵”的有一个直观的了解是有帮助的。实际上`point_add`和`point_double_repeat`都是昂贵的，后者比`point_add`更昂贵$w$倍。`point_add`使用$X$个约束，`point_double_repeat`就需要$wX$个约束，因为它需要将椭圆曲线点翻倍$w$次。

如果我们要实现一个朴素方法来单独验证每个签名，对每个签名我们必须做$m$次加和$m$次倍点，这意味着总共需要$m B$ 次加和 $m \cdot w \cdot B$ 次倍点。但是使用这个优化，因为加法发生在内部循环中，我们可以避开对$B$个点进行先行组合的$m \cdot w$ 个倍点操作。这导致了$(B-1)\cdot m \cdot w$个倍点的节省，这是显著的！

我们的基准测试表明，与给`circom-ecdsa`简单的for循环包裹相比，这种优化可以节省高达 68% 的电路大小（约束数量）。

## 代码
- 核心电路代码在这里：[https://github.com/puma314/batch-ecdsa/blob/master/circuits/batch_ecdsa.circom](https://github.com/puma314/batch-ecdsa/blob/master/circuits/batch_ecdsa.circom)

## 基准测试

下表比较了SNARK 中用于批量验证$B$个ECDSA 签名的我们优化的批处理ECDSA 电路和串行的 `circom-ECDSA` 验证。`circom-ecdsa`需要约150 万个约束来验证 1 个签名，因此朴素地对$B$个签名使用 `circom-EDSA`会导致约$150\cdot B$万个约束。

要注意的相关行是**约束**、**见证生成时间**和**证明时间**。因为生成 zkSNARK 证明在计算上非常昂贵（并且与约束数量和信号总数成线性关系），减少约束数量意味着生成证明所需时间更少。我们还显式地测量了证明生成时间（见证生成时间和证明时间的总和）。如下表所示，随着批处理大小$B$增加，加速程度（通常）也会增加[^1]。

|  | verify2 | verify4 | verify8 | verify16 | verify32 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 约束 | 1.9M | 2.8M | 4.6M | 8.1M | 15.3M |
| 见证生成 | 44s | 68s | 130s | 211s | 436s |
| 证明时间 | 4s | 7s | 14s | 40s | 86s |
| 总证明生成时间 | 48s | 75s | 144s | 251s | 522s |
| 证明验证时间 | 1s | 1s | 1s | 1s | 1s |
| 朴素见证生成 | 83s | 167s | 328s | 663s | 1360s |
| 朴素证明时间 | 35s | 42s | 110s | 211s | 332s |
| 总朴素证明生成时间 | 118s | 209s | 438s | 874s | 1692s |
| 加速比 | 2.45x | 2.78x | 3.04x | 3.48x | 3.24x |

### 电路元数据统计
在下表中，我们提供了一些有关编译电路和生成证明和验证密钥所用时间的信息。

|  | verify2 | verify4 | verify8 | verify16 | verify32 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 电路编译 | 162s | 186s | 345s | 579s | 1101s |
| 可信设置阶段2密钥生成 | 641s | 923s | 1485s | 2715s | 5352s |
| 可信设置阶段2贡献 | 120s | 196s | 366s | 498s | 987s |
| 证明密钥大小 | 1.1GB | 1.8GB | 2.9GB | 4.8GB | 9.0GB |
| 证明密钥验证 | 709s | 1050s | 1769s | 3198s | 6450s |




[^1]: 注意该模式仅在一定程度上成立：`verify32` 的加速比比 `verify16` 的加速比略低（3.24x 与 3.48x）。 随着电路变大，生成证明所需的内存量也在增加，假设在分析机器上溢出到交换空间中对此起了作用。

