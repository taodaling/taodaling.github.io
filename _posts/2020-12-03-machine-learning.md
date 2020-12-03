---
categories: notebook
layout: post
---

- Table
{:toc}

# 简介

机器学习是指给定目标，程序通过从经验中学习，最后提高它的性能指标。

根据目标的取值范围，我们可以将问题分类为：

- 回归问题：连续的值，比如所有的实数
- 分类问题：输出离散的值，比如0,1,2

根据我们是否对样本进行标记，机器学习分为：

- 监督学习：对样本进行标记，比如图像识别
- 无监督学习：没有对样本进行标记，比如聚类问题。

# 监督学习

监督学习的基本流程为：

给定训练集合$x, y$，其中$x$是输入，$y$是正确的输出，记$x^{(i)},y^{(i)}$表示第$i$个样本的输入和输出，记$m$为样本数量，$n$为特征数量。

之后我们需要通过训练集合训练出一个假说函数$h$，这个假说函数拥有最小的惩罚，其中惩罚为$J(h)$。这里我们认为$h$由参数向量$\theta$确定。

最后我们利用$h$预测新数据的输出。

## 线性回归模型

线性回归算法的假说函数为$h(x)=\theta_1^Tx+\theta_0$，其中$\theta_1$是一个长度为$n$的向量，$\theta_0$为常数。或者我们可以为每个$x$后面补充一个$1$，这时候可以简化为$h(x)=\theta^T x$。

我们的目标是最小化下面的惩罚函数$J$：

$$
J(\theta)=\frac{1}{2m}\sum_{i=1}^m(\theta^Tx^{(i)}-y^{(i)})^2
$$

计算导数如下：

$$
\begin{aligned}
\frac{\partial}{\partial \theta_j}J(\theta)&=\frac{1}{m}\sum_{i=1}^m(\theta^Tx^{(i)}-y^{(i)})x^{(i)}_j
\end{aligned}
$$

线性回归模型的性质：

- 只有唯一的极值点，同时这个点也是全局极小值点。（因此很适合使用梯度下降算法）

# 梯度下降算法

我们可以把惩罚函数$J$绘制成图形。

![https://raw.githubusercontent.com/taodaling/taodaling.github.io/master/assets/images/machine-learning/gradient-descending.png](https://raw.githubusercontent.com/taodaling/taodaling.github.io/master/assets/images/machine-learning/gradient-descending.png)

梯度下降算法实际上是选择一条下坡的道路（沿梯度的逆向）。当不存在这样的道路的时候，就意味着我们到了一个局部极小值点。

梯度下降分为多次迭代，每次迭代我们都通过下面方式修正参数$\theta$。

$$
\theta := \theta-\alpha \frac{\mathrm{d}}{\mathrm{d} \theta}J(\theta)
$$

其中$\alpha$是一个称为学习率的参数。

梯度下降法的性质：

- 如果迭代点抵达某个极值点，那么迭代点不会再发生变化。
- $\theta$越是接近极值点，梯度下降的步长就会越小，因此不必特意不断减少$\alpha$。