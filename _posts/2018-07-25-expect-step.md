---
layout: post
categories: math
description: Recently I'm thinking about a interesting problem...
---

考虑这样一个问题。我们不断投掷一枚均匀硬币，如果得到的硬币是正面的就继续投掷，直到硬币变成反面为止。求解硬币投掷次数的期望。

我们记`X`是投掷次数的随机变量，则

$$ pr\left( X=i \right) =\frac{1}{2^i} $$

故我们可以给出期望计算公式：

$$ E\left(x\right) =\sum ^{\infty }_{i=1}\dfrac {i}{2^{i}} $$

这个公式我们能很容易证明其是收敛的，接下来我们要预估尾项和。我们可以利用积分技术来预估：

$$ \sum_{i=n}^{\infty}{\frac{i}{2^i}}\le \frac{n}{2^n}+\int_n^{\infty}{\frac{i}{2^n}dx} $$

因此我们可以利用积分来获得尾项和的一个上界，即最大误差。下面让我们来解决这个积分计算。

$$ \left( x\cdot 2^{-x} \right) '=2^{-x}-\ln 2\cdot x\cdot 2^{-x} $$

将上式代入到积分公式中得到

$$\int_n^{\infty}{x\cdot 2^{-x}dx}=\frac{1}{\ln 2}\left( \int_n^{\infty}{2^{-x}dx}-\int_n^{\infty}{\left( x\cdot 2^{-x} \right) 'dx} \right) =\frac{1}{2^n\left( \ln 2 \right) ^2}+\frac{n}{2^n\ln 2}$$

最后我们可以得出：

$$ \sum_{i=n}^{\infty}{\frac{i}{2^i}}\le \frac{n}{2^n}+\int_n^{\infty}{\frac{i}{2^n}dx}=\frac{n}{2^n}\left( 1+\frac{1}{\ln 2}+\frac{1}{\left( \ln 2 \right) ^2} \right) $$

当我们代入n = 101的时候，可以得到误差上界大概为2e-28，几乎可以忽视。我们将前100项加总起来得到上式的一个近似值：2。
