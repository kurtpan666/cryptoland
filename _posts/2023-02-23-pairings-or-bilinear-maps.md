---
tags:
- 双线性映射
- BLS曲线
- BLS签名
- 身份基加密

title: 配对或双线性映射
date: 2023-02-22 00:00:00
#published: false
sidebar:
    nav: cryptoland
---

> - 原文：[Pairings or bilinear maps](https://alinush.github.io/2022/12/31/pairings-or-bilinear-maps.html)
> - 作者：Alin Tomescu
> - 译者：Kurt Pan

<!-- TODO: Add example of pairing (insecure). -->

{: .info}
**摘要：** *配对*，或者*双线性映射*，是对密码学来说非常强大的一个数学工具。配对给我们带来了最简洁的零知识证明[^GGPR12e]$^,$[^PGHR13e]$^,$[^Grot16]，最高效的门限签名[^BLS01]，第一个可用的身份基加密（IBE）方案[^BF01] ，以及其它很多高效的密码系统[^KZG10]。本文中，我将介绍一点配对的性质，其密码学应用和令人着迷的历史。事实上，读完此文后，[你可能会想要去监狱里待上个一两年](#历史)。

<!--more-->


{: .warning}
**关于推文的订正：**我在发布这篇文章的[原始推文](https://twitter.com/alinush407/status/1612518576862408705)中说到"没有【配对】，**S**NARKs就不可能"，这里加粗了 **S** 以强调这些SNARKs的“简洁性”。然而，感谢[推特上的一些大佬](#致谢)，我意识到这并**不**完全正确，而要依赖于一个人说“简洁”时他到底是什么意思。具体来说，Gentry和Wichs[^GW10]的*多项式对数证明大小*意义上的”简洁“SNARKs，存在于很多假设之上，包括离散对数[^BCCplus16]和随机谕言[^Mica98]。此外，$O(1)$群元素证明大小意义上的”简洁“SNARKs，也存在于RSA假设之上[^LM18]。当前，配对给予我们的，是具有最小的具体证明大小（即按照字节数计算）的SNARKs。

<p hidden>$$
\def\idt{\mathbb{1}_{\Gr_T}}
\def\msk{\mathsf{msk}}
\def\dsk{\mathsf{dsk}}
\def\mpk{\mathsf{mpk}}
$$</p>

## 预备知识

 - 熟悉素数阶循环群（例如椭圆曲线）
 - 令 $$\idt$$ 表示群 $\Gr_T$ 的单位元
 - 令 $x \randget S$ 表示从集合 $S$ 中随机抽取一个元素 $x$
 - 回顾 $\langle g \rangle = \Gr$ 表示 $g$ 是群 $\Gr$ 的生成元

## 配对的定义
*配对*，也称为*双线性映射*，是一个在素数阶 $p$ 的三个群 $\Gr_1、\Gr_2$ 和 $\Gr_T$ 之间的函数 $e : \Gr_1 \times \Gr_2 \rightarrow \Gr_T$ ，其中生成元 $g_1 = \langle \Gr_1 \rangle，g_2 = \langle \Gr_2 \rangle$ ， $g_T = \langle \Gr_T \rangle$。

当 $\Gr_1 = \Gr_2$ 时，称为 **对称**配对。 否则，是**非对称**配对。

最重要的是，配对有两个对密码学有用的性质：*双线性*和*非退化性*。

### 双线性性

*双线性性*要求，对于所有 $u\in\Gr_1$、$v\in\Gr_2$ 和 $a,b\in\Zp$，有：
$$e(u^a, v^b) = e(u, v)^{ab}$$

{: .warning}
考虑密码学，这是**最酷**的一个性质。 例如，这正是使[三方Diffie-Hellman](#三方diffie-hellman) 等有用应用成为可能的原因。

### 非退化性
*非退化性*要求：
$$e(g_1, g_2) \ne \idt$$

{: .info}
**为什么有这个性质？** 我们需要非退化性是因为没有它的话，定义一个（退化的）双线性映射是非常简单的（但没有用）。对于每个输入，返回 $\idt$即可。 这样的映射将满足双线性，但完全没有用。

### 高效性
*高效性*要求存在群元素的大小（即 $\lambda = \log_2{p}$ 中）上的多项式时间算法，可用于求出任意输入上的配对 $e$。

<details>
<summary><b>为什么要有这个要求</b> 排除了平凡但计算上困难的配对。 <i>（点击展开。）</i></summary>
<p markdown="1" style="margin-left: .3em; border-left: .15em solid black; padding-left: .5em;">
例如，假设 $r$ 是 $\Gr_T$ 中的一个随机元素。
首先，将配对定义为 $e(g_1, g_2) = r$。
这样，配对满足*非退化性*。
<br /><br />

其次，给定 $(u,v)\in \Gr\_1 \times \Gr\_2$，算法可以花费指数时间 $O(2^\lambda)$ 来计算离散对数 $x = \log\_ {g\_1}{(u)}$ 和 $y = \log\_{g\_2}{(v)}$ 并返回 $e(u, v) = e(g_1^x, g_2^y) = r^{xy}$。
这样，配对满足*双线性性*因为：
<br /><br />

\begin{align}
e(u^a, v^b)
    &= e\left((g_1^x)^a, (g_2^y)^b\right)\\\\\
    &= e\left(g_1^{(ax)}, g_2^{(by)}\right)\\\\\
    &= r^{(ax)\cdot (by)}\\\\\
    &= \left(r^{xy}\right)^{ab}\\\\\
    &= e(u, v)^{ab}
\end{align}
</p>
</details>

## 历史
{: .warning}
这是我对配对历史的有限理解，主要来自 [此视频中 Dan Boneh 的叙述](https://www.youtube.com/watch?v=1RwkqZ6JNeo) 以及我自己对相关文献的研究。 如果你知道更多历史，请给我发电子邮件，我可以尝试将其合并。

### 监狱里的数学家
（密码学）配对的历史始于一位名叫 **André Weil**[^Wiki22Weil] 的数学家，他在二战期间因拒绝在法国军队服役而入狱。
在那里，Weil，“设法说服了思想开明的监狱长将 [他] 关在一个单独的牢房里，[他] 被允许保留 [..] 笔、墨水和纸。”

Weil 使用这些他新获得的工具定义出了跨两个椭圆曲线群的配对。
**然而**，当时看来可能**非常奇怪**的是，Weil 付出了很多努力来确保他对配对的定义是*可计算的*。
这种额外的努力使今天基于配对的密码学成为了可能[^danboneh-shimuranote]。

### 去监狱，不去大学？
有趣的是，韦尔在监狱里的时间是如此高效，以至于他开始考虑是否应该每年在那里度过几个月。
更绝的是，Weil 考虑他是否应该**向有关当局建议每个数学家都要在监狱里度过一段时间。**
Weil 写道：
 
  > 我开始认为没有什么比监狱更有利于抽象科学了。
  >
  > [...]
  >
  > 我的数学工作进展出乎我的意料，我甚至有点担心——如果我只是在监狱里学得这么好，我是否每年都要安排关押两三个月？
  >
  >与此同时，我正在考虑向有关当局写一份报告，如下：“致科学研究主任：最近经由个人经验发现了一个可以对纯粹和无私的研究提供相当大的优势的方法，即留在监狱系统的设施中，我冒昧地请求……。”

你可以在他引人入胜的自传中读到所有这些以及更多的内容，该自传是从他作为数学家的角度写成的[^Weil92]。

### 从破解密码学到构建密码学

Weil 的工作是基础。
但是，基于配对的密码学的兴起还需要三个进展。

#### 第一个进展：Miller算法
1985 年，**Victor Miller** 撰写了一份手稿，表明实际上可以在多项式时间内有效地计算 Weil 配对（其本身实际上涉及指数阶多项式求值） [^Mill86Short]。

1984 年 12 月，Miller在 IBM 发表了关于椭圆曲线密码学的演讲，他声称椭圆曲线离散对数比有限域上的普通离散对数更难计算 [^miller-talk]。
Miller 受到了在场的 Manuel Blum 的挑战，要求他通过[归约](https://en.wikipedia.org/wiki/Reduction_(complexity)) 来支持这一说法：即，表明用于求解椭圆曲线上的离散对数的算法$B$可以有效地转换为另一种用于求解有限域中的离散对数的算法$A$。
这种归约意味着$B$解决的问题（即计算椭圆曲线离散对数）至少与$A$解决的问题（即计算有限域离散对数）一样难，如果不是更难的话。

Miller 着手通过思考唯一能将椭圆曲线群和有限域关联起来的事物--Weil配对，来试图找到归约。
有趣的是，他很快意识到，虽然 Weil 配对给出了一个归约，但是在相反的方向上的：即事实上，在 Weil 配对的帮助下，有限域中离散对数的算法 $A$ 可以有效地转化为椭圆曲线中的离散对数的算法 $B$ 。
这种“不想要的”的归约可以很容易看出来。
由于 $e(g^a, g) = e(g,g)^a$，求解椭圆曲线元素 $g_a\in \Gr$ 上的离散对数只要求解 $e(g,g )^a\in \Gr_T$，它实际上是有限域的乘法子群（参见 [配对到底是怎么做的？](#配对到底是怎么做的)）。

这几乎与 Miller 试图证明的相反，可能会使得整个椭圆曲线密码学崩溃，但幸运的是，Weil 配对映射到的扩域的阶太大，使得这种“不想要的”归约效率低下，因此也就根本不是一个归约。

整个事件让 Miller 对是否可以高效地计算 Weil 配对产生了兴趣，这导致了他算法的发现。
有趣的是，他将这篇手稿投给了顶级理论计算机科学会议 FOCS，但这篇论文被拒了，直到很久以后才发表在JoC上（根据 Miller 的说法）[^alin-where]。


#### 第二个进展：MOV攻击
1991 年，**Menezes、Vanstone 和 Okamoto**[^MVO91] 利用 Miller 的高效算法来求 Weil 配对，攻破了特定椭圆曲线上**在亚指数时间内**的离散对数假设。
这是非常惊人的，因为当时还没有已知的用于椭圆曲线的亚指数时间算法。

{: .info}
他们的攻击称为*MOV 攻击*，将椭圆曲线离散对数挑战 $g^a\in\Gr$ 使用配对映射到**目标群** $e(g^a, g)=e(g,g)^ a \in \Gr_T$ 。
由于目标群是有限域 $\mathbb{F}_q^{k}$ 的子群，这就可以使用更快的亚指数时间算法来计算 $e(g,g)^a$ 上的离散对数。


#### 第三个进展：Joux的三方Diffie-Hellman
到目前为止，配对似乎只对**密码分析**有用。
没有人知道如何使用它们来构造（而不是破解）密码学。

这在 2000 年发生了变化，那时 **Joux**[^Joux00] 使用配对在三方之间实现了单轮密钥交换协议，即 [三方Diffie-Hellman](#三方diffie-hellman) 
以前，已知的这种单轮协议仅在两方之间，而三方需要 2 轮。

从那时开始，大量新的、高效的密码学开始涌现：

  - BLS（短）签名[^BLS01]
  - 基于身份的加密[^BF01]
  - 支持一次乘法的加法同态加密[^BGN05]
  - 简洁零知识证明[^GGPR12e]

{: .info}
请注意这里有趣的模式：配对如何从用于破解密码系统的*密码分析工具*演变为用于构造密码系统的**建设性工具**的。
有趣的是，同样的模式也出现在了基于格的密码学的发展中。


## 配对的算术技巧
在密码系统的正确性或安全性证明中处理配对时，密码学家经常会使用一些技巧。

最明显的技巧，**“指数上相乘”**，来自双线性性。

\begin{align}
e(u^a, v^b) = e(u, v)^{ab}
\end{align}

双线性性也蕴含着下述技巧：
\begin{align}
e(u, v^b) = e(u, v)^b
\end{align}
或者：
\begin{align}
e(u^a, v) = e(u, v)^a
\end{align}

另一个技巧如下，这只是定义双线性性的一种类似方式：
\begin{align}
e(u, v\cdot w) = e(u, v)\cdot e(u, w)
\end{align}

{: .info}
**为什么这是对的？** 令 $y,z$ 分别表示 $v$ 和 $w$ 的离散对数 (关于 $g_2$的)。
然后，我们有：
\begin{align}
e(u, v\cdot w) 
    &= e(u, g_2^y \cdot g_2^z)\\\\\
    &= e(u, g_2^{y + z})\\\\\
    &= e(u, g_2)^{y + z}\\\\\
    &= e(u, g_2)^y \cdot e(u, g_2)^z\\\\\
    &= e(u, g_2^y) \cdot e(u, g_2^z)\\\\\
    &= e(u, v)\cdot e(u, w)
\end{align}

或者：
\begin{align}
e(u, v / w) = \frac{e(u, v)}{e(u, w)}
\end{align}

## 配对的应用

### 三方Diffie-Hellman

该协议由 Joux 在 2000 年 [^Joux00] 引入，使用**对称配对**：即，其中 $$\Gr_1 = \Gr_2 = \langle g\rangle \stackrel{\mathsf{def}}{=} \Gr$$。

我们有三个参与方 Alice、Bob 和 Charles，他们分别拥有私钥 $a、b$ 和 $c$。
他们互相发送他们的公钥 $g^a, g^b, g^c$ 并得到共享密钥 $k = e(g, g)^{abc}$。[^dhe]

怎么做到？

考虑Alice的视角。
她从 Bob 和 Charles 那里得到 $g^b$ 和 $g^c$。
首先，她可以使用她的秘密 $a$ 来计算 $g^{ab}$。
其次，她可以使用配对来计算 $e(g^{ab}, g^c) = e(g, g)^{abc} = k$。

通过对称性，所有其他参与者都可以做同样的事情并就相同的 $k$ 达成一致。

{: .info}
该协议也可以推广到 [**非**对称配对](#配对的定义)，其中 $\Gr_1 \neq \Gr_2$。

### BLS 签名

Boneh、Lynn 和 Shacham 使用配对[^BLS01] 给出了一个非常短的签名方案，其工作原理如下：

- 假设 $\Gr\_2 = \langle g_2 \rangle$ 且存在一个哈希函数 $H : \\{0,1\\}^\* \rightarrow \Gr\_1$ ，建模为随机谕言。
- 私钥是 $s \in \Zp$ 而公钥是 $\pk = g\_2^s \in \Gr\_2$。
- 要对消息 $m$ 签名，签名者计算 $\sigma = H(m)^s\in \Gr\_1$。
- 要在公钥 $\pk$ 下验证 $m$ 上的签名 $\sigma$，检查 $e(\sigma, g_2) \stackrel{?}{=} e(H(m), \pk) $ 是否成立。

请注意，正确计算的签名将始终可以通过验证，因为：
\begin{align}
e(\sigma, g_2) \stackrel{?}{=} e(H(m), \pk) \Leftrightarrow\\\\\
e(H(m)^s, g_2) \stackrel{?}{=} e(H(m), g_2^s) \Leftrightarrow\\\\\
e(H(m), g_2)^s \stackrel{?}{=} e(H(m), g_2)^s \Leftrightarrow\\\\\
e(H(m), g_2) = e(H(m), g_2)
\end{align}

请参阅 BLS 论文 [^BLS01]，关于如何证明没有攻击者可以在访问 $\pk$ 和签名谕言的情况下伪造 BLS 签名。

#### BLS签名的酷炫性质
BLS 签名非常棒：

1. 如果可以访问椭圆曲线库，它是实现**最简单**的方案之一。
1. 可以**聚合**同一消息 $m$ 上来自不同公钥的许多签名到一个单一的*多重签名*中，继续仅使用 2 个配对验证。
1. 甚至可以将不同消息中的此类签名**聚合**为一个*聚合签名*。
     + 然而，这样的聚合签名需要 $n+1$ 个配对来验证。
1. 可以轻松高效地[^TCZplus20]构造**门限**BLS签名方案，其中$n$个签名者中$\ge t$的任意子集可以协作签署消息$m$，但没有少于 $t$ 的子集可以产生有效的签名。
     + 更好的是，BLS 门限签名是**确定性的**，从而支持*门限可验证随机函数 (VRF)*，这对于在链上生成随机性很有用。
1. 可以定义非常高效的 BLS 签名的 **盲化** 变体，其中签名者可以在不知道消息 $m$ 的情况下签署消息 $m$。
1. BLS签名在实践中非常**高效**。
     - 据我所知，是 (1) 多重签名、(2) 聚合签名和 (3) 门限签名的最高效的方案
     - 对于单签名者 BLS，在非配对友好曲线上比 Schnorr 签名慢

{: .warning}
如果你对多重签名、聚合签名和门限签名的各种概念感到困惑，请参阅[我的幻灯片](https://docs.google.com/presentation/d/1G4XGqrBLwqMyDQce_xpPQUEMOK4lFrneuvGYU3MVDsI/edit?usp=sharing)。

### 身份基加密 (IBE)

在 IBE 方案中，可以直接用用户友好的电子邮件地址（或电话号码）来加密，而不是用难以记住或正确输入的繁琐公钥。

Boneh 和 Franklin 基于配给出了一个非常有效的 IBE 方案 [^BF01]。

IBE正常运作必须引入一个称为**私钥生成者 (PKG)** 的可信第三方 (TTP)，该第三方将根据用户的电子邮件地址向用户颁发密钥。
这个 PKG 有一个 **主密钥 (MSK)** $\msk \in \Zp$ 和一个关联的 **主公钥 (MPK)** $\mpk = g_2^s$，其中 $\langle g_2 \rangle = \Gr_2$。

$\mpk$ 是公开的，可用于给任意用户（给定电子邮件地址）加密消息。
至关重要的是，PKG 必须对 $\msk$ 保密。
否则，窃取它的攻击者可以导出任何用户的密钥并解密每个人的消息。

{: .warning}
如你所知，PKG 是一个中心故障点：盗窃 $\msk$ 会危及每个人的机密。
为了缓解这种情况，可以将 PKG 分散到多个权威机构中，这样就必须攻破一定数量的权威机构才能窃取 $\msk$。

令 $H_1 : \\{0,1\\}^\* \rightarrow \Gr_1^\*$ 和 $H_T : \Gr_T \rightarrow \\{0,1\\}^n$ 是两个哈希函数，建模为随机谕言。
要加密发送给电子邮件地址为 $id$ 的用户的 $n$ 比特消息 $m$，需要计算：
\begin{align}
    g_{id} &= e(H_1(id), \mpk) \in \Gr_T\\\\\
    r &\randget \Zp\\\\\
    \label{eq:ibe-ctxt}
    c &= \left(g_2^r, m \xor H_T\left(\left(g_{id}\right)^r\right)\right) \in \Gr_2\times \\{0,1\\}^n
\end{align}

要解密，电子邮件地址为 $id$ 的用户必须首先从 PKG 获取他们的**解密密钥** $\dsk_{id}$。
为此，我们假设 PKG 有一种方法可以在将密钥交给用户之前对用户进行身份验证。
例如，这可以通过电子邮件完成。

PKG 将用户的解密密钥计算为：
\begin{align}
    \dsk_{id} = H_1(id)^s \in \Gr_1
\end{align}

现在用户有了他们的解密密钥，他们可以将方程 $\ref{eq:ibe-ctxt}$ 中的密文 $c = (u, v)$ 解密为：
\begin{align}
    m &= v \xor H_T(e(\dsk_{id}, u))
\end{align}

你可以看到为什么正确加密的密文将成功解密，因为：
\begin{align}
v \xor H_T(e(\dsk_{id}, u))
    &= \left(m \xor       H_T\left((g_{id})^r            \right)\right) \xor H_T\left(e(H_1(id)^s, g_2^r)     \right)\\\\\
    &= \left(m \xor       H_T\left(e(H_1(id), \mpk )^r   \right)\right) \xor H_T\left(e(H_1(id),   g_2  )^{rs}\right)\\\\\
    &=       m \xor \left(H_T\left(e(H_1(id), g_2^s)^r   \right)        \xor H_T\left(e(H_1(id),   g_2  )^{rs}\right)\right)\\\\\
    &=       m \xor \left(H_T\left(e(H_1(id), g_2  )^{rs}\right)        \xor H_T\left(e(H_1(id),   g_2  )^{rs}\right)\right)\\\\\
    &= m
\end{align}

要了解为什么该方案在选择明文攻击下是安全的，请参阅原始论文[^BF01]。


## 配对到底是怎么做的？
大多数情况下，我也完全不知道。
怎么会呢？
好吧，我真的不需要知道。
这正是配对的美妙之处：人们可以以一种黑盒方式使用它们，而对其内部结构的了解为零。

不过，让我们来看一下这个黑盒子的内部。
让我们考虑流行的配对友好曲线 *BLS12-381* [^Edgi22]，它来自以 Barreto、Lynn 和 Scott 命名的 BLS 曲线家族 [^BLS02e]。

{: .warning}
**公共服务声明：**
你们中的一些人可能听说过*Boneh-Lynn-Shacham (BLS)* 签名。 请注意，这与 *Barretto-Lynn-Scott* 曲线中的 BLS 不同。 令人困惑的是，这两个首字母缩略词都有一个共同的作者，Ben Lynn。 （如果这还不够令人困惑，等你必须在 BLS12-381 曲线上使用 BLS 签名时就知道了。）

对于BLS12-381，涉及到的三个群$\Gr\_1, \Gr\_2, \Gr\_T$分别是：

  - 群 $\Gr_1$ 是椭圆曲线 $E(\F_q) = \left\\{(x, y) \in (\F\_q)^2\ \vert\ y^2 = x ^3 + 4 \right\\}$的子群， 其中 $\vert\Gr_1\vert = p$
  - 群 $\Gr_2$ 是一个不同的椭圆曲线$E'(\F_{q^2}) = \left\\{(x, y) \in (\F\_{q^2} )^2\ \vert\ y^2 = x^3 + 4(1+i) \right\\}$的子群，其中 $i$ 是 $-1$ 的平方根且 $\vert\Gr_2\vert = p $。
  - 群 $\Gr_T$ 是 $\F_{q^{k}}$ 中的所有的第 $p$ 个单位根，其中 $k=12$ 称为*嵌入度*

那么这三个群的配对映射是如何工作的？ 配对 $e(\cdot,\cdot)$ 可以展开为如下内容：
\begin{align}
\label{eq:pairing-def}
e(u, v) = f_{p, u}(v)^{(q^k - 1)/p}
\end{align}

知道计算配对包括两个步骤是很有用的：

1. 求出基 $f_{p, u}(v)$，也称为 **Miller 循环**，以纪念 [Victor Miller 的工作](#历史)
2. 基于这个基上求常数的 $(q^k - 1)/p$作为指数，也称为**最终指数**。
     - 这一步比第一步昂贵几倍

有关内部结构的更多信息，请参阅其他资源 [^Cost12]$^,$[^GPS08]$^,$[^Mene05]。

## 基于配对密码学的实现
本节讨论从业者可以用来加速实现的各种实现级细节。

### 使用非对称配对！

BLS12-381 上的配对是**非对称**的：即，$\Gr_1\ne\Gr_2$ 是两个**不同的**群（相同的阶 $p$）。 但是，也存在**对称**配对，其中 $\Gr_1 = \Gr_2$ 是同一个群。

不幸的是，“这种对称配对只存在于超奇异曲线上，这对协议的效率和安全性都会产生严重限制”[^BCMplus15e]。
换句话说，这种超奇异曲线在相同安全级别上不如**非**对称配对中使用的曲线高效。

因此，据我所知，今天的从业者完全依赖**非**对称配对，因为它们在安全级别保持不变时效率更高。

### BLS12-381 性能

我将为在 Filecoin 的 ([blstrs](https://github.com/filecoin-project/blstrs) 中实现的 BLS12-381 曲线提供一些关键性能的数据，blstrs是流行的[blst](https://github.com/supranational/blst)库的一个Rust包裹。

这些微基准测试使用`cargo bench`，在 10 核 2021 Apple M1 Max 上运行。

#### 配对计算时间

<!--
	alinush@MacBook [~/repos/blstrs] (master %) $ cargo +nightly bench -- pairing_
	running 4 tests
	test bls12_381::bench_pairing_final_exponentiation     ... bench:     276,809 ns/iter (+/- 1,911)
	test bls12_381::bench_pairing_full                     ... bench:     484,718 ns/iter (+/- 2,510)
	test bls12_381::bench_pairing_g2_preparation           ... bench:      62,395 ns/iter (+/- 4,161)
	test bls12_381::bench_pairing_miller_loop              ... bench:     148,534 ns/iter (+/- 1,203)
-->

正如式\ref{eq:pairing-def}中所解释的那样，配对涉及两个步骤：

- Miller循环计算
     - 210 微秒
- 最终指数
     - 276 微秒

因此，一次配对大约需要 486 微秒（即两者之和）。

#### 求指数时间

{: .warning}
$\Gr_T$ 微基准测试是通过稍微修改`blstrs`的基准测试代码 （[此处](https://github.com/filecoin-project/blstrs/blob/e70aff6505fb6f87f9a13e409c080995bd0f244e/benches/bls12_381/ec.rs#L10)） 完成的 。
（有关这些修改，请参阅本页的 HTML 注释。）

<!--
	alinush@MacBook [~/repos/blstrs] (master *%) $ git diff
	diff --git a/benches/bls12_381/ec.rs b/benches/bls12_381/ec.rs
	index 639bcad..8dcec20 100644
	--- a/benches/bls12_381/ec.rs
	+++ b/benches/bls12_381/ec.rs
	@@ -167,3 +167,34 @@ mod g2 {
			 });
		 }
	 }
	+
	+mod gt {
	+    use rand_core::SeedableRng;
	+    use rand_xorshift::XorShiftRng;
	+
	+    use blstrs::*;
	+    use ff::Field;
	+    use group::Group;
	+
	+    #[bench]
	+    fn bench_gt_mul_assign(b: &mut ::test::Bencher) {
	+        const SAMPLES: usize = 1000;
	+
	+        let mut rng = XorShiftRng::from_seed([
	+            0x59, 0x62, 0xbe, 0x5d, 0x76, 0x3d, 0x31, 0x8d, 0x17, 0xdb, 0x37, 0x32, 0x54, 0x06,
	+            0xbc, 0xe5,
	+        ]);
	+
	+        let v: Vec<(Gt, Scalar)> = (0..SAMPLES)
	+            .map(|_| (Gt::random(&mut rng), Scalar::random(&mut rng)))
	+            .collect();
	+
	+        let mut count = 0;
	+        b.iter(|| {
	+            let mut tmp = v[count].0;
	+            tmp *= v[count].1;
	+            count = (count + 1) % SAMPLES;
	+            tmp
	+        });
	+    }
	+}
	alinush@MacBook [~/repos/blstrs] (master *%) $ cargo +nightly bench -- mul_assign
	   Compiling blstrs v0.6.1 (/Users/alinush/repos/blstrs)
		Finished bench [optimized] target(s) in 0.75s
		 Running unittests src/lib.rs (target/release/deps/blstrs-349120dc60ef3711)

	running 2 tests
	test fp::tests::test_fp_mul_assign ... ignored
	test scalar::tests::test_scalar_mul_assign ... ignored

	test result: ok. 0 passed; 0 failed; 2 ignored; 0 measured; 115 filtered out; finished in 0.00s

		 Running benches/blstrs_benches.rs (target/release/deps/blstrs_benches-a6732e3e4e5c6a4d)

	running 4 tests
	test bls12_381::ec::g1::bench_g1_mul_assign            ... bench:      72,167 ns/iter (+/- 1,682)
	test bls12_381::ec::g2::bench_g2_mul_assign            ... bench:     136,184 ns/iter (+/- 1,300)
	test bls12_381::ec::gt::bench_gt_mul_assign            ... bench:     497,330 ns/iter (+/- 7,802)
	test bls12_381::scalar::bench_scalar_mul_assign        ... bench:          14 ns/iter (+/- 0)

	test result: ok. 0 passed; 0 failed; 0 ignored; 4 measured; 21 filtered out; finished in 5.30s
-->

- $\Gr_1$ 求指数是最快的
    + 72 微秒
- $\Gr_2$ 求指数大约慢2倍
    + 136 微秒
- $\Gr_T$ 求指数比 $\Gr_2$ 慢大约3.5倍
    + 500 微秒

{: .info}
**注意：**这些基准测试随机选择取指数操作的基，并且**不**对其执行任何预计算，预计算会将这些时间加快 2-4 倍。

#### 多指数
这是一个众所周知的优化，为了完整起见，我将其包括在内。

具体来说，许多库可以使得计算$k$ 次指数运算的乘积 $\prod_{0 < i < k} \left(g_i\right)^{x_i}$ 比单独计算 $k$ 次指数再聚合它们的乘积要快很多。

例如，[blstrs](https://github.com/filecoin-project/blstrs) 在这方面似乎快得令人难以置信：

<!--
running 4 tests
test bls12_381::bench_g1_multi_exp                     ... bench:     760,554 ns/iter (+/- 47,355)
test bls12_381::bench_g1_multi_exp_naive               ... bench:  18,575,716 ns/iter (+/- 42,688)
test bls12_381::bench_g2_multi_exp                     ... bench:   1,876,416 ns/iter (+/- 58,743)
test bls12_381::bench_g2_multi_exp_naive               ... bench:  35,272,720 ns/iter (+/- 266,279)
-->

- $\Gr_1$ 中的大小为 256 的多指数运算
     + 总共需要 760 微秒，或者每次指数需要 3 微秒！
     - 朴素方法共需要 18.5 毫秒，慢24倍
- $\Gr_2$ 中的大小为 256 的多指数运算
     - 总共需要 1.88 毫秒，或者每次指数需要 7.33 微秒！
     - 朴素方法共需要 35.3 毫秒，慢18.8倍

#### 群元素大小
- $\Gr_1$群元素是最小的
	+ 例如，BLS12-381 为 48 字节， BN254 曲线为 32 字节[^BN06Pair]
- $\Gr_2$ 群元素大2倍
    + 例如，BLS12-381 为 96 字节
- $\Gr_T$ 元素大 12 倍
    + 通常，对于具有*嵌入度* $k$ 的配对友好曲线，大 $k$ 倍

#### 交换$\Gr_1$ 和 $\Gr_2$
在设计基于配对的密码协议时，需要仔细考虑选择使用 $\Gr_1$ 和使用 $\Gr_2$ 的目的。

例如，在 BLS 签名中，如果想要更小的签名，那么应该计算签名 $\sigma = H(m)^s \in \Gr_1$ 并在 $\Gr_2$ 中设置稍大的公钥。
另一方面，如果想要最小化公钥大小，那么可以将它放在 $\Gr_1$ 中，同时花费较长的时间来计算 $\Gr_2$ 中的签名。

{: .warning}
其他因素也会影响使用 $\Gr_1$ 和 $\Gr_2$ 的方式，例如同构 $\phi : \Gr_2 \rightarrow \Gr_1$ 的存在或均匀哈希到这些群中的能力。
事实上，这种同构的存在将非对称配对进一步分为两种类型：类型 2 和类型 3（有关不同类型配对的更多信息，请参阅 *Galbraith 等人*[^GPS08]）

#### 和非配对友好椭圆曲线的比较
与不支持配对的椭圆曲线相比，配对友好的椭圆曲线要慢两倍左右。

例如，流行的素数阶椭圆曲线群 [Ristretto255](https://ristretto.group/) 提供：

<!--
ristretto255/basepoint_mul
                        time:   [10.259 µs 10.263 µs 10.267 µs]

ristretto255/point_mul  time:   [40.163 µs 40.187 µs 40.212 µs]
-->

- 求指数 40 微秒，快$\approx 2\times$ 
	+ 当基数固定时使用预计算可以加速到 10 微秒
- 群元素大小32 字节

### 多配对
如果你还记得配对的实际工作方式（参见式 $\ref{eq:pairing-def}$），你会注意到以下优化：

每当我们必须计算 $n$ 个配对的乘积时，我们可以先计算 $n$ 次Miller循环并进行一次最终指数而不是 $n$ 次。
这大大减少了许多应用中的配对计算时间。

\begin{align}
\prod_i e(u_i, v_i)
    &= \prod_i \left(f_{p, u_i}(v_i)^{(q^k - 1)/p}\right)\\\\\
    &= \left(\prod_i f_{p, u_i}(v_i)\right)^{(q^k - 1)/p}
\end{align}


## 结论
本文本来应该只是[配对的三个性质](#配对的定义) 的一个简短总结：双线性性、非退化性和高效性。

不幸的是，我觉得很有必要去讨论下其[迷人的历史](#历史)。
而且我不能让你在没有看到一些强大的 [配对的密码学应用](#配对的应用) 的情况下离开。

之后，我意识到实现基于配对的密码系统的从业者可能会受益于了解其[内部工作机制](#how-do-pairings-actually-work)，因为可以利用其中一些细节来加速[实现](#implementation-details)。

## 致谢

我要感谢 Dan Boneh 帮助我澄清和了解 Weil 相关的历史，以及 [他在 2015 年 Simons 的演讲](https://www.youtube.com/watch?v=1RwkqZ6JNeo)，这启发了我 做更多的研究并写下了这个历史记录。

非常感谢：

  - [Lúcás Meier](https://twitter.com/cronokirby)、[Pratyush Mishra](https://twitter.com/zkproofs)、[Ariel Gabizon](https://twitter.com/rel_zeta_tech) 和 [ Dario Fiore](https://twitter.com/dariofiore0)，感谢他们关于“简洁”(S) 在 **S**NARKs[^GW10] 中代表什么的启发性观点，并提醒我带有$O(1)$群元素证明大小的 SNARKs 其实存在于 RSA 假设的 [^LM18]。
  - [Sergey Vasilyev](https://twitter.com/swasilyev) 指出 BLS12-381 椭圆曲线定义中的拼写错误。
  - [@BlakeMScurr](https://twitter.com/BlakeMScurr) 指出对 Joux 作品的错误引用[^Joux00]。
  - [Conrado Guovea](https://twitter.com/conradoplg) 向我指出了 Victor Miller 关于他如何开发用于求 Weil 配对的算法的说明（[此处](#first-development-millers-algorithm) 进行了讨论） 。
  - [Chris Peikert](https://twitter.com/ChrisPeikert) 指出有很多不依赖配对的快速 IBE 方案 [^DLP14e]。


---

[^dhe]: 通常，会用一些密钥导出函数 $\mathsf{KDF}$ 用于导出密钥 $k = \mathsf{KDF}(e(g,g)^{abc})$。

[^danboneh-shimuranote]: 感谢 Dan Boneh，他将 Weil 的定义与 Shimura 在他关于模形式的经典著作中的不同定义进行了对比。 虽然 Shimura 的定义使得证明配对的所有性质变得容易得多，但它将 $n$ 阶配对定义为 $n$个$n^2$阶的点的求和。 这使得它无可救药地不可计算。 另一方面，Weil 的定义涉及对一个非常具体的函数的求值——没有指数大小的求和——但用来证明所有配对的性质需要做更多的工作。

[^miller-talk]: 2010 年 10 月 10 日，Miller在 [微软研究院的演讲](https://www.youtube.com/watch?v=yK5fYfn6HJg&t=2901s) 中亲自讲述了这个故事。

[^alin-where]: 除了 Boneh 在 [^Mill86Short] 中发表的手稿之外，我找不到任何 Miller 在这方面发表的作品的踪迹。 任何提示我将不胜感激。

{% include refs.md %}
