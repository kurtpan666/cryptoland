---
tags:
- SNARK
- Lookup Argument
title: Lookup奇点降临：Lasso 和 Jolt 简介
date: 2023-08-10 00:00:00
#published: false
---

> - 原文：[Approaching the “lookup singularity:” Introducing Lasso and Jolt](https://a16zcrypto.com/posts/article/introducing-lasso-and-jolt/) 
> - 作者：Justin Thaler
> - 译者：Kurt Pan


{: .info}
编者注：本系列中，我们将分享两项崭新的工作：Lasso 和 Jolt，它们可以显著加速 web3 中应用的扩展和构造。它们共同代表了一种本质上全新的 SNARK 设计方法，可将已广泛部署的工具链的性能提升一个数量级或更多；提供更好、更方便的开发者体验；并使得审计变得更加容易。
有关 SNARK 为何如此重要、设计现状、需要理解的关键概念以及给开发者和工程师的实现细节等更多信息，请阅读《[Building on Lasso and Jolt](https://a16zcrypto.com/posts/article/building-on-lasso-and-jolt/)》一文（其中还包括一个开源实现）。研究论文可以在这里找到([Lasso](https://people.cs.georgetown.edu/jthaler/Lasso-paper.pdf), [Jolt](https://people.cs.georgetown.edu/jthaler/Jolt-paper.pdf))。 还可以在这个 [YouTube 播放列表](https://www.youtube.com/playlist?list=PLjQ9HCQMu_8xjOEM_vh5p26ODtr-mmGxO) 中观看关于解释 Lasso 和 Jolt、lookup奇点以及思考 SNARK 设计的新方法的完整或简短视频。

<!--more-->

[SNARK](https://a16zcrypto.com/posts/tags/snarks/)（简洁非交互式知识论证）是一种密码协议，任何人都可以向不可信的验证者证明他们知道满足某些性质的“证据”。 web3 中的一个重要应用就是二层 (L2) 汇总向一层 (L1) 区块链证明其知道授权一系列交易的数字签名，使得签名本身不必由L1存储和验证，从而有助于可扩展性。

区块链之外的应用包括对快速但[不受信任的]((https://arxiv.org/pdf/1709.08501.pdf) )[硬件]((https://eprint.iacr.org/2015/1243.pdf))[设备]((https://eprint.iacr.org/2017/242) )，证明它们产生的所有输出的有效性，确保用户可以信任它们。个人可以以[零知识](https://a16zcrypto.com/posts/article/zero-knowledge-canon/)的方式证明是可信权威机构向他们颁发了[凭证](https://eprint.iacr.org/2022/878.pdf)，以证明比如他们的年龄足够访问有年龄限制的内容，而无需透露其出生日期。任何通过网络发送加密数据的人都可以向管理者证明该数据[符]((https://eprint.iacr.org/2021/1022.pdf))[合](https://eprint.iacr.org/2023/1022)网络政策，而无需透露更多细节。

虽然许多SNARK的 *验证者* 开销足够的低，SNARK 通常仍会在*证明者*计算中引入约六个数量级的开销。证明者的额外工作是高度并行化的，但数百万倍因子的开销严重限制了 SNARK 的应用范围。 （有关更多详细信息，请参阅我之前关于 SNARK [证明者性能](https://a16zcrypto.com/posts/article/measuring-snark-performance-frontends-backends-and-the-future/)、[安全性](https://a16zcrypto.com/posts/article/snark-security-and-performance/)、其发展[历史](https://a16zcrypto.com/posts/article/zero-knowledge-canon/#section--6)以及关于这种强大但复杂的技术的[常见误解](https://a16zcrypto.com/posts/article/17-misconceptions-about-snarks/)的文章。）

性能更好的 SNARK 将加速 L2，还可以允许builder去解锁出我们从未想过的应用。这就是为什么我们要引入两项新的相关技术——**Lasso**，一种新的lookup论证，可以显著降低证明者开销； **Jolt**，它使用 Lasso 提供了一个新的框架，用于为所谓的 zkVM 和更广泛的前端设计 SNARK。合在一起，它们共同提高了 SNARK 设计的性能、开发者体验和可审计性，进而提升了web3 中的开发。

我们对 Lasso 的初步实现已经表明，与流行的 SNARK 工具链 Halo2 中的lookup论证相比，速度提高了 10 倍以上。当 Lasso 代码库完全优化后，我们预计速度会提高约 40 倍。 Jolt 包含在 Lasso 之上的其他创新，我们预计它能够实现与现有 zkVM 类似或更好的加速。


## Lookup 论证，Lasso 和 Jolt

SNARK*前端*是将计算机程序转换为可以输入给SNARK后端的电路的编译器。电路是一种极其受限的计算模型，其中可用的“基础运算”只是加法和乘法。

现代 SNARK 设计中的一个关键工具是一种称为lookup论证的协议，可以让不受信任的证明者密码学地*承诺*一个大向量，然后证明该向量的每项都包含在某个预定义的表中。lookup论证可以通过高效地处理那些不能自然地由少量加法和乘法计算的操作来帮助保持较小的电路（请参阅[我们的配套文章](https://a16zcrypto.com/posts/article/building-on-lasso-and-jolt/)中讨论的位运算的例子）。


然而，正如以太坊基金会的 Barry Whitehat 去年在他称之为“[lookup奇点](https://zkresear.ch/t/lookup-singularity/65)”（即我们设计的电路只执行查找的时刻）的愿景中所概述的那样，“如果我们能够仅使用lookup论证来高效地定义电路，这将会得到更简单的工具和电路”。正如Barry所写，随着时间的推移，lookup论证“对于几乎所有事情都将变得比多项式约束更高效”。

这一愿景与如今的工作方式形成鲜明对比，如今的开发者通过使用特殊的领域特定语言（将程序编译为多项式约束）或直接手动编码约束来编写程序来部署 SNARK。使用这样的工具链需要大量费力工作，且为安全关键漏洞的蔓延提供了很大的空间。即使使用了复杂的工具和电路，SNARK 的性能[仍然很慢](https://a16zcrypto.com/posts/article/17-misconceptions-about-snarks/#section--20)，这限制了它们的适用性。

Lasso 和 Jolt 解决了所有的三个关键问题：性能、开发者体验和可审计性。合在一起，我相信它们实现了lookup奇点的愿景。 Lasso 和 Jolt 还迫使人们去重新思考 SNARK 设计中许多公认的认知。在介绍了一些必要的背景知识后，本文重新审视了有关 SNARK 性能的一些常见观点，并解释了如何根据 Lasso 和 Jolt 等创新来更新认知。

## SNARK 设计背景：为什么这么慢？
大多数 SNARK 后端都会让证明者使用[多项式承诺方案](https://cacr.uwaterloo.ca/techreports/2010/cacr2010-10.pdf)密码学地承诺电路中每个门的值。然后证明者去证明承诺值确实对应于证据检查程序的正确执行。

我将来自多项式承诺方案部分的证明者工作称为*承诺开销*。 SNARK 具有来自多项式承诺方案以外的额外证明者开销，但承诺开销往往是瓶颈所在， Lasso 和 Jolt 也是如此。如果所设计的 SNARK 中承诺开销不是主要的证明者开销，这并不意味着多项式承诺方案很便宜。相反，这意味着有其他开销高于应有的开销。

直观上，承诺的目的是通过密码学方法安全地给证明系统增加简洁性。当证明者承诺一些值的一个大向量时，证明者将整个向量发送给验证者，大致就像一个平凡证明系统将整个证据发送给验证者一样。承诺方案可以在*不*强迫验证者实际接收整个证据的情况下实现这一目标——这意味着 SNARK 设计中承诺方案的目的是控制验证者开销。

但这些密码学方法对于证明者来说非常昂贵，特别是去与 SNARK 中除了多项式承诺方案使用的[信息论](https://zkproof.org/2020/08/12/information-theoretic-proof-systems/)[方法](https://zkproof.org/2020/03/16/sum-checkprotocol/)相比。信息论方法仅依赖于有限域运算，且一个域操作比承诺任意域元素所需的时间快几个数量级。

根据使用的多项式承诺方案，计算承诺涉及多指数（也称多标量乘法，或 MSM）或  [FFTs](https://en.wikipedia.org/wiki/Fast_Fourier_transform) 和 Merkle 哈希。 Lasso 和 Jolt 可以使用任何多项式承诺方案，但在使用基于 MSM 的方案（例如 [IPA](https://eprint.iacr.org/2016/263)/[Bulletproofs](https://eprint.iacr.org/2017/1066.pdf), [KZG](https://www.iacr.org/archive/asiacrypt2010/6477178/6477178.pdf)/[PST](https://eprint.iacr.org/2011/587.pdf), [Hyrax](https://eprint.iacr.org/2017/1132), [Dory](https://eprint.iacr.org/2020/1274), 或 [Zeromorph](https://eprint.iacr.org/2023/917)）实例化时，具有特别低的开销。

## 为什么 Lasso 和 Jolt 很重要

Lasso 是一种新的lookup论证，相比之前工作证明者承诺更少且更小的值。根据上下文，这可以达到 20 倍或更多的加速，其中 2 到 4 倍的加速来自于承诺较少的值，另一个 10 倍的加速来自于 Lasso 中所有承诺的值都很小的事实。与许多先前的lookup论证不同，Lasso（和 Jolt）还避免了 FFT，FFT 占用大量空间且可能成为大型实例的瓶颈。

此外，Lasso 甚至适用于超大的表（比如大小为 $2^{128}$ ），只要这些表是（在精确的技术意义上）“结构化的”。这些表太大，任何人都无法显式实现，但 Lasso 只为其实际访问的表元素支付开销。特别地——与之前的lookup论证相比——如果表是结构化的，那么任何一方都不需要密码学地承诺表中的所有值。

 Lasso 利用两种不同的结构概念：*可分解性* 和 LDE 结构。 （LDE 是*低阶扩展*多项式的缩写。）可分解性大致意味着可以通过对更小的表执行一系列查找来回答对表的单次查找。这是比 LDE 结构更严格的要求，但 Lasso 在应用于可分解表时特别高效。

### Jolt
Jolt（*J*ust *O*ne *L*ookup *T*able）是一种新的前端，由 Lasso 使用超大的查找表的能力解锁。 Jolt 的目标是虚拟机/CPU 抽象，也称为[指令集架构]((https://en.wikipedia.org/wiki/Instruction_set_architecture)) (ISA)。支持这种抽象的 SNARK 称为 zkVM。比如说[RISC-Zero 项目](https://www.risczero.com/)针对的 [RISC-V 指令集](https://riscv.org/wp-content/uploads/2017/05/riscv-spec-v2.2.pdf)（包括[乘法扩展](https://en.wikichip.org/wiki/risc-v/standard_extensions)），这是由计算机体系结构社区开发的一种流行的开源 ISA，设计时*并没有*考虑到 SNARK。

对于每条 RISC-V 指令 $f_i$，Jolt 的主要思想是创建一个包含 $f_i$ 的整个求值表的查找表。因此，如果 $f_i$ 有两个 32 位的输入，则表将有 $2^{64}$ 项，其中第$(x, y)$项是$f_i(x, y)$。如果我们考虑 64 位而不是 32 位数据类型的指令，则每个指令的表的大小将是 $2^{128}$ 而不是 $2^{64}$ 。

对于基本上每条 RISC-V 指令，生成的查找表都是可分解的，可应用Lasso。 （一些指令是通过应用一小段其他指令的序列来处理的。例如，除法是通过让证明者承诺商和余数来处理的，并通过一次乘法和一次加法来检查提供的是否正确。）

然后可以按如下方式生成运行 RISC-V CPU $T$ 步的“电路”。对于 $T$ 个步骤中的每步，电路都包含一些简单的逻辑，用于确定该步要执行的原始 RISC-V 指令 $f_i$ 是什么，以及该指令的输入 $(x, y)$ 是什么。然后电路可以通过进行一次揭示相应表的第$(x,y)$项的查找来执行$f_i$。更准确地说，Jolt 包括由每条指令的表串联组成的单个表 - 这就是“仅一个查找表”名称的由来。


## 重新审视 SNARK 设计中广泛接受的认知
Lasso 和 Jolt 还颠覆了 SNARK 设计中的一些广泛接受的认知。每条广泛接受的认知都以粗体字显示，后面接着的是 Lasso 和 Jolt 如何改变了它们。

{: .warning}
**#1.大[域](https://en.wikipedia.org/wiki/Finite_field)上的 SNARK 是一种浪费。每个人都应该使用 [FRI](https://eccc.weizmann.ac.il/report/2017/134/)、[Ligero](https://eprint.iacr.org/2022/1608)、[Brakedown](https://eprint.iacr.org/2021/1043) 或其变体，因为它们避免了通常适用于大域的椭圆曲线技术。**

这里的域大小大致对应于 SNARK 证明中出现的数的大小。由于非常大的数的加法和乘法开销很高，因此大域上的 SNARK 很浪费的想法意味着我们应该努力设计永远不会出现大数的协议。基于 MSM 的承诺（定义如下）使用椭圆曲线，椭圆曲线通常在较大的域（大小约为 $2^{256}$ ）上定义，因此这些承诺得到了性能较差的名声。

我之前的文章[解释](https://a16zcrypto.com/posts/article/17-misconceptions-about-snarks/#section--17)到，使用小域是否有意义（即使在它们是一个选项时）在很大程度上取决于应用。 Lasso 和 Jolt 走得更远。它们表明，即使使用基于 MSM 的承诺方案，证明者的工作也几乎可以独立于域大小。这是因为，对于基于 MSM 的承诺，承诺值的大小比这些值所在域的大小更为重要。

**有关 MSM 的详细信息。** 大小为 $\mathrm{n}$ 的多指数（也称为 MSM）计算 $\prod_{i=1}^n g_i^{x_i}$ 形式的表达式, 其中每个 $\mathrm{g_i}$ 是乘法[群](https://en.wikipedia.org/wiki/Group_(mathematics))的一个元素。朴素多指数算法执行 $n$ 次群指数和 $\mathrm{n}$ 次群乘法。每次指数比乘法慢[约 400 倍](https://a16zcrypto.com/posts/article/17-misconceptions-about-snarks/#section--13)。

Pippenger的[多指数算法](https://ieeexplore.ieee.org/document/4567910)比朴素算法节省了大约 $\log (n)$ 的因子。在实际中, 这[可能](https://a16zcrypto.com/posts/article/17-misconceptions-about-snarks/#section--14)可以得到超过 10 倍的节省。此外, 如果所有的指数都很“小”(例如, 在 0 和 $2^{20}$ 之间）, 则可以在计算多指数的时间中节省*另一个*10 倍的因子。 这类似于计算 $g_i^{2^{16}}$ 比计算 $g_i^{2^{160}}$ 快 10 倍。前者包括16次平方, 后者160次。

如果所有承诺值都很小，Pippenger算法只需要（大约）一次群乘法来承诺一个值（一个不错的说明请参阅[这篇论文](https://eprint.iacr.org/2022/1400)第 4 节）。

**Lasso 和 Jolt 的新性质。** 现有的 SNARK 要求证明者承诺许多随机域元素。例如，在名为 [Plonk](https://eprint.iacr.org/2019/953) 的流行 SNARK 后端中的证明者对每个电路门大约承诺 7 个随机域元素（以及其他非随机域元素）。即使被证明的计算中出现的所有值都很小，这些随机域元素也会很大。

相比之下，Lasso 和 Jolt 仅要求证明者承诺较小的值（Lasso 的证明者承诺的值也比先前的lookup论证更少）。这极大地提高了基于 MSM 的方案的性能。至少，由Lasso 和 Jolt 应该废除这样的观念：在证明者性能很重要的情况下，社区应该放弃基于 MSM 的承诺。


{: .warning}
**#2：指令集越简单zkVM越快。**

只要每条指令的求值表是可分解的，Jolt 的（每条指令）复杂度就仅取决于指令的输入大小。 Jolt 令人惊讶地证实了复杂指令是可分解的，特别是所有的 RISC-V 指令。
 
这与人们普遍认为的观点相反，即更简单的虚拟机 (zkVM) 必然会导致（对VM 的每一步）更小的电路和相应的更快的证明者。这是特别简单的 zkVM（例如 [Cairo VM](https://eprint.iacr.org/2021/1063)）背后的指导动机，它们是专门为 SNARK 友好而设计的。

事实上，对于更简单的虚拟机，Jolt 为证明者实现了比之前的 SNARK 更低的承诺开销。例如，对于 Cairo VM 执行，[SNARK 证明者](https://eprint.iacr.org/2021/1063)在 VM 的每步中承诺 51 个域元素。 Cairo 现在部署的 SNARK 目前在 [251 比特的域](https://a16zcrypto.com/posts/article/17-misconceptions-about-snarks/#section--10)上工作（[63 比特](https://arxiv.org/pdf/2109.14534.pdf)是 SNARK 可以使用的域大小的硬下界）。 Jolt 的证明者对RISC-V CPU 的每步约承诺 60 个域元素（定义在 64 比特数据类型上）。在考虑承诺的域元素很小的事实后，Jolt 证明者的开销大致相当于承诺 6 个随机 256 比特域元素的开销。

{: .warning}
**#3：将大型计算分解为小片段不会造成性能损失。**

基于这种观点，当今的项目会将任何大电路分解成小块，分别证明每个块，并通过 SNARK 递归聚合结果。这些部署使用这种方法来缓解流行的 SNARK 中的性能瓶颈。例如，基于 FRI 的 SNARK 需要接近 100 GB 的证明者内存，即使对于小型电路也是如此。它们还需要 FFT，这是一种超线性运算，如果 SNARK“全部一次”应用于大型电路，则也可能成为瓶颈。

实际情况是，一些 SNARK（例如 Lasso 和 Jolt）会表现出规模经济（而不是当前部署的 SNARK 中展现的规模不经济）。意思是被证明的语句越大，相对于直接证据检查（即在不保证正确性的证据上求值电路所需的工作）的证明者开销越*小*。在技​​术层面，规模经济来自两个地方。

1. 对$n$大小的 MSM 的 Pippenger 加速：相对于朴素算法有 $\log (n)$ 因子的改进。 $n$越大，改进越大。
2. 在 Lasso 等lookup论证中，证明者支付“一次性”开销，该开销取决于查找表的大小，但与查找值的数量无关。一次性证明者的开销摊销在了对表的所有查找之中。更大的块意味着更多的查找，意味着更好的摊销。

当今处理大型电路的流行方法是将其分解成尽可能小的块。每个块大小的主要限制是它们不能太小以至于递归聚合证明成为证明者的瓶颈。

Lasso 和 Jolt 提出了一种本质上相反的方法。人们应该使用具有规模经济性的 SNARK。然后将大型计算分解为尽可能*大*的块，并对结果进行递归。每个块大小的主要限制是证明者的内存空间，这随着块变大而增长。


{: .warning}
**#4：高阶约束对高效SNARK是必要的。**

Jolt 使用 R1CS 作为其中间表示。在 Jolt 中使用“高阶”或“自定义”的约束没有任何好处。 Jolt 中的证明者开销大部分在lookup 论证 Lasso中，而不是证明所得到的想当然地处理查找的约束系统的可满足性。

Jolt 是通用的，不会失去一般性。这因此反驳了开发者对设计高阶约束（通常是手动指定的）以努力从当今流行的 SNARK 中挤出性能改进的强烈关注。

当然，某些情况仍然可能受益于高阶或自定义约束。一个重要的例子是 [Minroot VDF](https://eprint.iacr.org/2022/1626)，其 5 阶约束可以将承诺开销降低约 3 倍。

{: .warning}
**#5：稀疏多项式承诺方案成本高昂，应尽可能避免。**

这是针对最近引入的名为 [CCS](https://eprint.iacr.org/2023/552) 的约束系统和支持它的 SNARKs -- (Super-)[Spartan](https://eprint.iacr.org/2019/550) 和 (Super-)[Marlin](https://eprint.iacr.org/2019/1047)的主要批评。正如我在[上一篇文章]((https://a16zcrypto.com/posts/article/17-misconceptions-about-snarks/#section--6))中所讨论的，CCS 是当今实践中流行的所有约束系统的清晰的泛化。

这种批评是没有根据的。事实上，Lasso 和 Jolt 的技术核心就是 [Spartan](https://eprint.iacr.org/2019/550) 中的稀疏多项式承诺方案（参见第 7 节），称为 Spark。 Spark 本身是将*任何*（非稀疏）多项式承诺方案到支持稀疏多项式的方案的通用转换。

Lasso 优化并扩展了 Spark，以确保证明者只承诺“小”值，但即使没有这些优化，批评也是不合理的。事实上，Spartan 的证明者比避免稀疏多项式承诺的SNARK比如 [Plonk](https://eprint.iacr.org/2019/953)承诺更少的（随机）域元素。

正如我在[上一篇文章](https://a16zcrypto.com/posts/article/17-misconceptions-about-snarks/#section--6)中所讨论的，Spartan 的方法具有额外的性能优势，特别是对具有重复结构的电路。对于这些电路，加法门不会增加 Spartan 证明者的密码学工作。工作仅随给定约束系统中变量的数量而增长，而不是随着约束的数量而增长。与 Plonk 不同的是，Spartan 证明者无需承诺同一变量的多个不同“副本”。

---

我们乐观地认为 Lasso 和 Jolt 将极大地改变 SNARK 的设计方式，从而提高性能和可审计性。本系列的后续文章将更深入地探讨 Lasso 和 Jolt 的工作原理。我将特别解释他们对[和校验协议](https://people.cs.georgetown.edu/jthaler/blogpost.pdf)的使用，这是一种关键工具，具有神奇的能力，可以最大限度地降低证明者的承诺开销。随着我们继续优化 Lasso 和构建 Jolt，我们欢迎来自社区的反馈。你可以在[这里](https://twitter.com/SuccinctJT)联系我，我也会在那里宣布未来的工作。

---

致谢：Justin Thaler 与同事  [*Srinath Setty*](https://www.microsoft.com/en-us/research/people/srinath/)（微软研究院）、[*Riad Wahby*](https://wahby.net/)（卡内基梅隆大学）和 [*Arasu Arun*](https://arasua.run/)（纽约大学）一起开发了 Lasso 和 Jolt； Lasso 的实现是由 a16z 加密工程师  [*Sam Ragsdale*](https://twitter.com/samrags_) 和 [*Michael Zhu*](https://twitter.com/moodlezoup)完成的。

