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


增量证明系统提供了一些优于传统证明系统的优势：

- 不需要循环迭代的静态边界，更适合具有动态控制流的程序。
- 需要的内存开销极小，因为证明者只需要与执行一步所需空间成比例的空间，而不是存储整个计算迹。
- 非常适合证明生成的分布式和并行化。
证明者可以运行程序，跟踪输入和输出变量以及状态变化，然后使用 CPU 或 GPU 为计算的每个步骤并行生成证明。 更好的是，证明可以方便地聚合成一个，验证者可以检查。



They are well suited for the distribution and parallelization of proof generation.
The prover can run the program, keeping track of the input and output variables and state changes, and then generate the proofs in parallel using CPU or GPU for each step of the computation. Better still, the proofs can be conveniently aggregated into a single one, which the verifier can check.


## 非均匀IVC（NIVC）的计算模型
We can think of the program as a collection of $n+1$ non-deterministic, polynomial time computable functions, $f_1, f_2, \ldots, f_n, \phi$, where each function receives $k$ input and $k$ output variables; each $f_j$ can also take non-deterministic input. The function $\phi$ can take non-deterministic input variables and output an element $j=\phi(z=(x, w))$, choosing one of the $f_i$. Each function is represented as a quadratic rank-one constraint system (R1CS), an NP-complete problem. In IVC, the prover takes as input at step $k\left(k, x_0, x\right)$ and a proof $\Pi_k$ that proves knowledge of witnesses $\left(w_0, w_1, \ldots, w_{k-1}\right)$ such that
$$
x_{j+1}=F\left(x_j, w_j\right)
$$
for all $j=0,1, \ldots, k$ we have $x=x_{k+1}$. In other words, given a proof that shows that the previous step has been computed correctly and the current state $x_k$, we get the next state $x_{k+1}$ and a proof $\Pi_{k+1}$ showing that we computed step $k$ correctly. In the NIVC setting, $\phi$ selects which function we are going to use,
$$
x_{j+1}=F_{\phi\left(x_j, w_j\right)}\left(x_j, w_j\right)
$$
At each step, SuperNova folds an $\mathrm{R} 1 \mathrm{CS}$ instance and its witness, representing the last step of the program's execution into a running instance (it takes two $N$-sized NP instances into a single $N$-sized NP instance). The prover uses an augmented circuit containing a verifier circuit and the circuit corresponding to the function $f_j$ being executed. The verifier circuit comprises the non-interactive folding scheme and a circuit for computing $\phi$. We will represent the augmented functions as $f_j^{\prime}$

One problem with the folding scheme is that we have multiple instructions, each having its R1CS representation. We could take the path of universal circuits, but this would make us pay a high cost for many cheap instructions. In Nova, we avoided the problem since we only had one type of instruction. To deal with multiple instructions, SuperNova works with $n$ running instances $U_i$, where $U_i(j)$ attests to all previous executions of $f_j^{\prime}$, up to step $i-1$. Therefore, checking all $U_i$ is equivalent to checking all $i-1$ steps. Each $f_j^{\prime}$ takes as input $u_i$, corresponding to step $i$, and folds it to the corresponding $U_i$ instance. We can think of it as looking at the function we want to execute and performing the instance folding with the one related to the previous executions. By doing so, we pay the cost for each instruction only when it is used, at the expense of keeping more running instances and updating accordingly.
The verifier circuit corresponding to $f_j^{\prime}$ does the following steps:
1. Checks that $U_i$ and $p c_i=\phi\left(x_{i-1}, w_{i-1}\right)$ (the index of the function executed previously) are contained in the public output of the instance $u_i$ . This enforces that the previous step produces both $U_i$ and $p c_i$.
2. Runs the folding scheme's verifier to fold an instance and updates the running instances, $U_{i+1}$.
3. Calls $\phi\left(x_i, w_i\right)=p c_{i+1}$ to obtain the index of the following function to invoke.


## 总结
IVC is a powerful cryptographic primitive which allows us to prove the integrity of computation in an incremental fashion. This strategy is well-suited for virtual machine executions and general programs with dynamic flow control. We could achieve this by using universal circuits, but at the expense of a considerable cost for each instruction, no matter how cheap it could be. Nova introduced folding schemes, allowing one to realize IVC for a single instruction. SuperNova generalizes Nova to multiple instructions by adding a selector function $\phi$, choosing the instruction to be executed at each step. To support several instructions, SuperNova needs to maintain separate bookkeeping for each function's execution. This construction has many exciting applications since we could realize IVC without requiring expensive arbitrary circuits.