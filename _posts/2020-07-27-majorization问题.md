---
categories: algorithm
layout: post
---

- Table
{:toc}

# 定义

给定两个$n$维向量$a$和$b$，若将$a$中所有元素从大到小排序，$b$也从大到小排序，且$\sum_{i=1}^ka_i\geq \sum_{i=1}^kb_i$对任意$1\leq k\leq n$都成立，则称$a$弱主宰$b$，写作$a\succ_w b$。

对应的如果在上面的条件的基础上再加上$\sum_{i=1}^na_i=\sum_{i=1}^nb_i$，则陈$a$主宰$b$，写作$a\succ b$。

# 消除问题

**给定$n$种颜色的史莱姆，第$i$种颜色的史莱姆数量为$a_i$。现在我们每次可以选择两只不同颜色的史莱姆，将它们消灭。问重复上面操作，是否最后可以消灭所有史莱姆。**

直接给结论，记$m=\sum_{i}a_i$，则能完成目标当且仅当$m$是偶数，且$\forall 1\leq i\leq n(a_i\leq \frac{m}{2})$。

偶数的要求非常显然，而第二个要求的必要性也很显然，下面在$m$是偶数的前提下我们来证明第二个要求的充分性。

通过归纳法证明。当$m=0$的时候命题显然成立。现在假设当$m\leq 2k-2$的时候命题均成立，接着考虑$m=2k$的时候。我们可以将$a_i$从大到小排序，之后将$a_1$和$a_2$减少$1$。

新的情况共有$m-2$只史莱姆，很显然是偶数。同时有$a_1\leq \frac{m}{2}\Rightarrow a_1-1\leq \frac{m-2}{2}$，对于$a_2$也是同理。而对于$a_3$，一定有$a_3\leq \frac{m}{2}-1\leq \frac{m-2}{2}$，因此新的情况满足$\forall 1\leq i\leq n(a_i\leq \frac{m-2}{2})$。



# 参考资料

- [https://en.wikipedia.org/wiki/Majorization](https://en.wikipedia.org/wiki/Majorization)