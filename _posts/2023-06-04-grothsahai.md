---
tags:
- 配对

title: Groth-Sahai 证明并没那么可怕 （未完成且还有2）
date: 2023-06-04 00:00:00
published: false
sidebar:
    nav: cryptoland
---

> - 原文：[Groth-Sahai Proofs Are Not That Scary](https://crypto.ethereum.org/blog/groth-sahai-blogpost)
> - 作者：Mikhail Volkhov, Dimitris Kolonelos, Dmitry Khovratovich, Mary Maller
> - 译者：Kurt Pan


![](https://crypto.ethereum.org/images/posts/groth-sahai-explainer/armchairs.png)

{: .info}
Groth-Sahai (GS) 证明是一种零知识证明技术，理解起来似乎令人望而生畏。 这是因为许多文献都试图概括所有可以使用 GS 技术证明的可能事物，而不是因为证明过于复杂。 事实上，GS 证明是最简单的零知识结构之一。 对于基于配对群中的群元素的陈述，它们是理想的，因为 NP 约束系统没有大量减少，这使得证明者非常快。 在安全方面，它们也比 SNARK 依赖更多的标准假设，因此更有可能是安全的。
本文中我们将完整地过一个 Groth-Sahai 证明的例子，让一般的密码学读者都能够理解。 
具体来说，我们讨论如何证明 ElGamal 的密文包含 0 或 1.
我们的例子包括 Escala 和 Groth 的改进。 
前置知识包括要知道什么是零知识论证以及什么是III型配对（但不用知道它们的构造方式）。



<!--more-->

## 配对乘积等式中的 ElGamal
## 符号和配对
回顾配对的标准性质：$e\left(g^a, \widehat{h}^b\right)=e(g, \widehat{h})^{a b}$ 。用 $ \mathbb{G}_1$ 表示第一源群， $\mathbb{G}_2$ 为第二源群。 $\mathbb{G}_2$ 中的元素上面用宽帽符表示，如 $\widehat{E}$。 尝试在可能的情况下使用小写字母作为相应大写字母元素的指数： $\Theta=g^\theta$ 和 $\widehat{D}=\widehat{h}^d$ 

配对允许我们在参数的对数中定义二次方程，例如：

$$
\begin{array}{ccc}
e(X, \widehat{h}) e(g, \widehat{Y})=1 & \Longleftrightarrow & \log _g X+\log _{\widehat{h}} \widehat{Y}=0 \\
e(X, \widehat{Y})=e\left(X^{\prime}, \widehat{Y^{\prime}}\right) & \Longleftrightarrow & \log _g X \cdot \log _{\widehat{h}} \widehat{Y}=\log _g X^{\prime} \cdot \log _{\widehat{h}} \widehat{Y}^{\prime}
\end{array}
$$


如果你没有用过多基配对，请参阅以下说明：
## 配对等式中的 ElGamal 验证

![](https://crypto.ethereum.org/images/posts/groth-sahai-explainer/figure-1.png)

## 一般证明结构
### 透明设置
### 对见证的承诺
### 证明者和验证者

## 推导等式$E_2$
### 总体策略
### 承诺的中间配对乘积等式
### 添加零知识：获得足够的随机化器

## 安全证明
### 完备性
### 可靠性
### 零知识

## 额外：推导$E_4$证明结构
