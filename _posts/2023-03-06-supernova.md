---
tags:
- 折叠方案
- 同态承诺
- 增量可验证计算
- 递归零知识证明

title: SuperNova
date: 2023-03-06 00:00:00
#published: false
sidebar:
    nav: cryptoland
---

> - 原文：[Champagne SuperNova, incrementally verifiable computation](https://www.notamonadtutorial.com/champagne-supernova-incrementally-verifiable-computation-2/)
> - 作者：Not a Monad Tutorial
> - 译者：Kurt Pan

## 引言
增量证明系统相比于传统证明系统具有如下优势：

- 不需要循环迭代的静态边界，更适合具有动态控制流的程序。
- 需要的内存开销极小，因为证明者只需要与执行一步所需空间成比例的空间，而不是存储整个计算迹。
- 非常适合证明生成的分布式和并行化。
证明者可以运行程序，追踪输入和输出变量以及状态变化，然后使用 CPU 或 GPU 为计算的每个步骤并行生成证明。 更好的是，证明可以方便地聚合成一个，验证者可以去检查。

[增量可验证计算](https://cryptography.land/2023/03/05/nova.html) (IVC) 提供了一种证明机器执行完整性的方法。 要使用 IVC，我们需要设计一个可以执行任何机器支持指令的通用电路。 在每一步，我们都必须调用这个电路。 这很不方便，因为证明每一步的开销都与通用电路的大小成正比，即使程序仅以低得多的开销执行其中一个受支持的指令时也是如此。 解决该缺点的一种方法是通过构造具有最小指令集的虚拟机来限制通用电路的大小。


[SuperNova](https://eprint.iacr.org/2022/1758) 提供了一个基于虚拟机的密码证明系统（包括证明者和验证者）和设计用于在该虚拟机上运行的程序，满足以下性质：

- 简洁性：证明的大小和验证所述证明的时间至多为程序执行时间的多项式对数。
- 零知识：除了问题的正确执行之外，证明没有泄漏任何其他内容。
- 小开销：证明程序每一步的成本与表示该指令的电路的大小成正比。
- 增量证明生成：证明者可以为程序执行的每个步骤独立生成一个证明，之后在不增加证明大小的情况下将这些证明合并成一个单一的证明。

SuperNova 利用折叠方案（之前[Nova](https://github.com/microsoft/Nova) 使用的密码原语），使用[松弛承诺的 R1CS](https://cryptography.land/2023/03/05/nova.html) 来实现非均匀 IVC。 SuperNova 是 Nova 的扩展，支持具有丰富指令集的机器（Nova 仅限于一条指令）。 以下部分中，我们将分别叙述 SuperNova 所需的不同组件以及如何实现非均匀 IVC。



## 对向量的承诺方案
一个对向量的[承诺方案](https://www.notamonadtutorial.com/the-hunting-of-the-zk-snark/)是三个有效算法的集合：
- 参数生成 $\operatorname{Gen}\left(1^\lambda\right)=p p$ ：给定安全级别参数 $\lambda$ ，算法输出公共参数 $p p$。
- 承诺，$\operatorname{commit}(p p, x, r)=\mathrm{cm}$：给定公共参数、一个向量和随机性 $r$，输出承诺 $\mathrm{cm}$ 。
- 打开， $\operatorname{open}(p p, \mathrm{~cm}, x, r)=0,1$ ：给定承诺、向量、随机性和公共参数，算法验证给定的承诺是否对应向量 $x$。

承诺方案必须满足以下性质：
- 绑定性：给定承诺 $\mathrm{cm}$，不可能找到两个 $x_1, x_2$ 使得 $\operatorname{commit}\left(p p, x_1, r\right)=\operatorname{commit} \left(p p, x_2, r\right)$。 简言之，承诺将我们绑定到原始值 $x$。
- 隐藏性：承诺不会泄露任何 $x$ 的信息。

一些承诺方案还满足下面两个性质，例如 Pedersen 方案，在我们的场景中很有用：

- 加法同态性：给定 $x_1, x_2$ ，承诺是加法同态的，如果

$$
\operatorname{commit}\left(p p, x_1+x_2, r_1+r_2\right)=\operatorname{commit}\left(p p, x_1, r_1\right)+\operatorname{commit}\left(p p, x_2, r_2\right)
$$

- 简洁性：承诺的大小远小于相应的向量（例如，$\operatorname{commit}(p p, x, r)=\mathcal{O}(\log (x))$）。

SuperNova 可以用任何满足上述四个性质的承诺方案来实例化，例如 Pedersen 、KZG 或 Dory。


## 非均匀IVC（NIVC）的计算模型

我们可以将程序视为 $n+1$ 个非确定性多项式时间可计算函数的集合，$f_1, f_2, \ldots, f_n, \phi$，其中每个函数$f_j$接收$k$个输入变量，产生 $k$个输出变量；每个 $f_j$ 也会额外有非确定性输入。 函数 $\phi$ 可以接收$k$个输入和非确定性输入，输出元素 $j=\phi(z=(x, w))$，起到选择 $f_i$ 之一的作用。 每个函数都表示为二次秩一约束系统 (R1CS)，这是一个 NP 完全问题。 

在 IVC 中，证明者第$k$步的输入为$\left(k, x_0, x\right)$以及证明见证$\left(w_0, w_1, \ldots, w_{k-1}\right)$知识的证明$\Pi_k$，使得对所有$j=0,1, \ldots, k$，（ $x=x_{k+1}$），

$$
x_{j+1}=F\left(x_j, w_j\right)
$$


换句话说，给定一个表明前一步计算正确的证明和当前状态 $x_k$，我们得到下一个状态 $x_{k+1}$ 和一个表明我们正确计算了步骤 $k$的证明 $\Pi_{k+1}$ 。

在 NIVC 场景中，用$\phi$ 来选择我们要使用的函数，

$$
x_{j+1}=F_{\phi\left(x_j, w_j\right)}\left(x_j, w_j\right)
$$

每一步中，SuperNova 折叠一个代表程序执行的上一步的R1CS实例及其见证，将其变成一个正在运行的实例（将两个 $N$ 大小的 NP 实例变成单个 $N$ 大小的 NP 实例）。 证明者使用包含验证者电路和与正在执行的函数 $f_j$ 对应的电路的增强电路。 验证者电路包括非交互式折叠方案和用于计算 $\phi$ 的电路。 我们将增强函数表示为 $f_j^{\prime}$。

折叠方案的一个问题是我们会有多个指令，每个指令都有其 R1CS 表示。 可以走通用电路的方式，但这会使我们为许多廉价指令也付出高昂的开销。 Nova中避开了这个问题，因为只有一种指令。 
为处理多条指令，SuperNova 使用 $n$ 个运行实例 $U_i$，其中 $U_i(j)$ 证明 $f_j^{\prime}$ 的所有先前执行，直至第 $i-1$ 步。 
因此，检查所有 $U_i$ 等价于检查所有 $i-1$ 步。 
每个 $f_j^{\prime}$ ，对应于第$i$步 ，$u_i$ 作为输入，并将其折叠到相应的 $U_i$ 实例。 
可以理解为是查看我们要执行的函数，并与之前执行的相关函数进行实例折叠。 通过这样做，我们只在每条指令被使用时才支付开销，代价是维持更多运行实例和相应更新。

$f_j^{\prime}$对应的验证者电路做以下步骤：
1. 检查 $U_i$ 和 $p c_i=\phi\left(x_{i-1}, w_{i-1}\right)$（先前执行的函数的索引）是否包含在实例 $u_i$的公共输出中。 这确保上一步生成了 $U_i$ 和 $p c_i$。
2. 运行折叠方案的验证者以折叠实例并更新运行实例，$U_{i+1}$。
3. 调用$\phi\left(x_i, w_i\right)=p c_{i+1}$ 得到后面要调用的函数的索引。


## 总结
IVC 是一种强大的密码原语，它使我们能够以增量方式证明计算的完整性。 该策略非常适合虚拟机执行和具有动态控制流的通用程序。
 当然可以通过使用通用（universal）电路来实现这一点，但代价是每条指令（无论多快）都会有相当大的开销。
Nova 引入了折叠方案，允许为单个指令实现 IVC。 SuperNova 通过添加选择每一步要执行指令的选择函数$\phi$将 Nova 扩展为多条指令。
 为了支持多条指令，SuperNova需要为每个函数的执行维护单独的记录。 该构造有许多令人兴奋的应用，因为我们可以在不需要昂贵的任意电路的情况下实现 IVC。