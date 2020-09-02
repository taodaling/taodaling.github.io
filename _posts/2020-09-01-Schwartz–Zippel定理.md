---
categories: algorithm
layout: post
---

- Table
{:toc}

# Schwartz–Zippel定理

**定理：在有限域$\mathbb{F}$上给定一个总度数(total degree)为$k$的$n$元多项式$P$，且多项式非$0$，对于$\mathbb{F}$的某个子集$S$，我们从中均匀随机挑选$n$个数值$x_1,\ldots,x_n$，那么有：**

$$
\mathtt{Pr}[P(x_1,\ldots,x_n)=0]\leq \frac{k}{|S|}
$$

# 判断多项式是否相同

**题目1：给定$m$个$n$元多项式$P_1,\ldots,P_n$，其中每个多项式的总度数均为$1$。接下来要求回答$q$个请求，每个请求给定$(l,a,x)$，要求回答$P_{l}\times P_{l+1}\times \ldots \times P_{l+x-1}$是否与$P_{a}\times P_{a+1}\times \ldots \times P_{a+x-1}$是否相等。所有的计算都在模$10^9+7$的意义下计算。其中$1\leq n,m, q\leq 10^5$，且每个多项式$P_i$最多只有$10$个变量的系数非$0$。**

我们可以直接用Schwartz–Zippel定理来判断。为每个变量提前选定值，这样每个多项式都对应一个整数。我们只需要用线段树计算区间中所有整数的乘积即可。

为了保证概率足够大，我们可以判定两次，两次都通过才算相同。


# 参考资料

- [https://en.wikipedia.org/wiki/Schwartz%E2%80%93Zippel_lemma](https://en.wikipedia.org/wiki/Schwartz%E2%80%93Zippel_lemma)