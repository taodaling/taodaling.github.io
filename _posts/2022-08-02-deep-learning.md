---
categories: technology
layout: post
---

# PAC

定义：

- $C$: 概念集合，每个概念$c:X\rightarrow Y$是一个映射
- $X$：输入集合
- $Y$：输出集合
- $S$：样本集合，$(x,y)$形式
- $H$：假说集合
- $D$：输入集合的概率分布

考虑binary classify，其对应的概念为$c$，这时候$Y=\left\{0,1\right\}$。我们的目标是从$H$中找到与$c$误差最小的假说$h$。定义泛化误差为：

$$
R(h)=\mathop{\mathrm{P}}\limits_{x\sim D}[h(x)\neq c(x)]=\mathop{\mathrm{E}}\limits_{x\sim D}[1_{h(x)\neq c(x)}]
$$

定义经验误差为：

$$
\widehat{R}_S(h)=\frac{1}{m}\sum_{i=1}^m1_{h(x_i)\neq c(x_i)}
$$

始终满足

$$
\mathop{\mathrm{E}}\limits_{x\sim D}[\widehat{R}_S(h)]=R(h)
$$

记$n$代表表示任意一个输入空间中元素所需的空间大小。记$size(c)$表示概念类$C$中需要最多空间表示的概念的需要的空间。

我们称概念类$C$是PAC-learnable，如果存在一个算法$A$，和一个多项式$poly$，满足对于任意$\epsilon>0$以及$\delta>0$，对于任意一个$X$上的概率分布$D$，以及任意目标概念$c\in C$，只要样本数量$m\geq poly(1/\epsilon,1/\delta,n,size(c))$，就有：

$$
\mathop{\mathrm{P}}\limits_{S\sim D^m}[R(h_S)\leq \epsilon]\geq 1-\delta
$$

进一步的如果$A$能在$poly(1/\epsilon,1/\delta,n,size(c))$时间复杂度内结束，那么认为$C$是可以被有效PAC-learnable的，并认为算法$A$是$C$的PAC-learnable算法。

## 有限假说集合一致情况

对于有限大小的假说集$H$，目标概念$c\in H$。且算法$A$能找到一个假说$h_S$满足$\widehat{R}_S(h_S)=0$。那么对于任意$\delta>0$，一定满足$\mathrm{P}_{S\sim D^m}\[R(h_S)\leq \epsilon\]\geq 1-\delta$，只要

$$
m\geq \frac{1}{\epsilon}\ln \frac{|H|}{\delta}
$$

## 有限假说集合不一致情况

之前讨论的$h$是在$S$上是完全一致的（$0$误差），但是实际上很多时候由于目标概念超出了我们假说的范畴，因此会存在一定的经验误差。

对于任意假说$h:X\rightarrow \left\{0,1\right\}$，由Hoeffding's inequality可以直接推出

$$
\mathop{\mathrm{P}}\limits_{S\sim D^m}[|\widehat{R}_S(h)-R(h)|\geq \epsilon]\leq 2\exp(-2m\epsilon^2)
$$

继而可以推出对于任意$\delta>0$，下面不等式拥有至少$1-\delta$的置信度

$$
R(h)\leq \widehat{R}_S(h)+\sqrt{\frac{1}{2m}\log \frac{2}{\delta}}
$$


对于任意有限假说集合$H$，以及任意$\delta>0$，都至少有$1-\delta$置信度，使得不等式满足：

$$
\forall h\in H, R(h)\leq \widehat{R}_S(h)+\sqrt{\frac{1}{2m}\ln \frac{2|H|}{\delta}}
$$

## 不可知PAC

上面提到的样本是$X\times Y$的子集，但是有一种场景是一个样本的标签是概率分布而不是精确值。比如给定身高和体重，标签是性别。PAC学习在这种场景下的自然扩展称作不可知（agnostic）PAC。

泛化误差为：

$$
R(h)=\mathop{\mathrm{P}}\limits_{(x,y)\sim D}[h(x)\neq y]=\mathop{\mathrm{E}}\limits_{(x,y)\sim D}[h(x)\neq y]
$$

称$A$是不可知PAC学习算法，如果存在一个多项式$poly$，对于任意$\epsilon>0$以及$\delta>0$，对于所有$X\times Y$上的概率分布$D$，只要$m\geq poly(1/\epsilon,1/\delta,n,size(c))$，就有：

$$
\mathop{\mathrm{P}}\limits_{S\sim D^m}[R(h_S)-\mathop{\min}\limits_{h\in H}R(h)\leq \epsilon]\geq 1-\delta
$$

如果$A$能在$poly(1/\epsilon,1/\delta,n,size(c))$实际复杂度内结束，那么称$A$是一个有效的不可知PAC学习算法。

定义贝叶斯误差为：

$$
R^*=\mathop{\inf}\limits_{h} R(h)
$$

对于假说$h$，如果满足$R(h)=R^*$，那么称$h$为贝叶斯假说或贝叶斯分类器。

在确定的情况（只允许一个标签）下，满足$R^*=0$，但是在随机的情况（允许多个标签）下，$R^*\neq 0$，可以通过条件概率定义为：

$$
\forall x\in X, h_{Bayers}(x)=\mathop{\argmax}\limits_{y\in \{0,1\}} \mathrm{P}[y|x]
$$

对于给定的$X\times Y$上的概率分布，定义在点$x$处的噪音为

$$
noise(x)=\min\{\mathrm{P}[0|x],\mathrm{P}[1|x]\}
$$

满足

$$
\mathrm{E}[noise(x)]=R^*
$$

对于点$x$，如果$noise(x)$接近$\frac{1}{2}$，那么这样的点称为噪音点，也是精确预判所会遇到的挑战。

