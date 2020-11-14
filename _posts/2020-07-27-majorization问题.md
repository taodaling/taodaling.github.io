---
categories: algorithm
layout: post
---

- Table
{:toc}

# 定义

给定两个$n$维向量$a$和$b$，若将$a$中所有元素从大到小排序，$b$也从大到小排序，且$\sum_{i=1}^ka_i\geq \sum_{i=1}^kb_i$对任意$1\leq k\leq n$都成立，则称$a$弱主宰$b$，写作$a\succ_w b$。

对应的如果在上面的条件的基础上再加上$\sum_{i=1}^na_i=\sum_{i=1}^nb_i$，则陈$a$主宰$b$，写作$a\succ b$。

# 参考资料

- [https://en.wikipedia.org/wiki/Majorization](https://en.wikipedia.org/wiki/Majorization)