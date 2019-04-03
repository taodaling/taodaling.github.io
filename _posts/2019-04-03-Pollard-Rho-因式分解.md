---
categories: algorithm
layout: post
---

对于任意正整数$n$，由算术基本定理可知，$n=\prod_{i=1}^kp_i^{c_i}$，其中$p_i$为互不相同的素数。

而因式分解是指，将一个合数分解为两个非平凡因子的乘积，比如$6=2\cdot 3$。

如何对合数$n$进行因式分解呢？

最简单的方式是枚举它可能的因子，由于$n$的最小因子一定在$2$到$\sqrt{n}$之间，因此可以在时间复杂度$O(\sqrt{n})$内完成。

Pollard's Rho算法在1975年由John Pollard发明，对于分解合数$n$，它的期望时间复杂度仅为$O(n^{\frac{1}{4}})$。

首先我们考虑建立一组序列$X=(x_1,x_2,\ldots)$，其中$x_{i+1}=x_i^2+c(mod \phantom{1} n)$。由于是处于$n$的剩余类环中，因此序列中实际包含的不同值最多为$n$个。序列的图像应该如下：

![https://upload.wikimedia.org/wikipedia/commons/4/47/Pollard_rho_cycle.jpg](https://upload.wikimedia.org/wikipedia/commons/4/47/Pollard_rho_cycle.jpg)

由于图像酷似一个$\rho$，因此算法得名rho。

由生日悖论可知，在一年有$d$天的情况下，当班级里有$\sqrt{d}$个学生的时候，学生中期望至少有一对学生的生日相同，即发生碰撞。将生日悖论用在我们的序列中，可知我们序列中的不同值的数量约莫为$O(\sqrt{n})$。

设$m$为$n$的最小非平凡因子，易知$m\leq \sqrt{n}$。我们记$y_i=x_i(mod \phantom{1} m)$，可以推导：
$$
y_{i+1}=x_{i+1}\mod m=(x_i^2+c\mod n)\mod m\\
=(x_i^2+c)\mod m=((x_i\mod m)^2+c)\mod m\\
=y_i^2+c(mod\phantom{1} m)
$$
因此我们得到了新的序列$Y=(y_1,y_2,\ldots)$，同样根据生日悖论，可知序列中不同值的数量约为$O(\sqrt{m})\leq O(\sqrt[4]{n})$。

由于第一次碰撞等概率发生在序列上，因此序列$X$中的环的入口期望为第$\sqrt{n}/2$个元素，而$Y$中的环的入口期望为第$\sqrt{m}/2$个元素。因此$X$中环的长度$C_x$期望为$\sqrt{n}/2$，同理$Y$中的环的长度$C_y$期望为$\sqrt{m}/2$。由于环的大小期望不同，那么就应该存在一对数$i,j$,满足$x_i\neq x_j \land y_i=y_j$。这意味着$n\nmid \left| x_i - x_j \right| \land m \mid \left| x_i - x_j \right|$。因此我们可以通过$gcd(n,\left|x_i - x_j \right|)$获得一个$n$的非平凡因子。

那么我们该如何获得$i$和$j$呢。有两种方法，一个是Floyd判圈算法，另外一种是倍增法。当Floyd判圈法记录两个下标$i$,$j$，每次循环$i=i+1$，$j=j+2$。当$C_y \mid (j-i)$时，我们就能找到对应的$i$和$j$。而倍增法则通过枚举可能的$Y$序列中环上的某个值和环的长度上限实现，首先猜想$y_1$落在环上且环的长度不大于$1$，如果第$i$次猜测失败，下一次就猜测$y_{2^i}$落在环上且环的长度不大于$2^i$。两种算法都可以在$O(\sqrt(m))$的时间复杂度内完成。

如果你挑选的$x_1$和$c$不好的话，可能会导致最终没法找到$n$的任意因子。那么你可以在合适的时机，用新的随机数替代$x_1$和$c$重新进行启发式查找。

注意，这个算法是建立在$n$是合数的前提下的，如果$n$是素数，那么这个算法就永远无法找到因子而陷入死循环中。所以在执行Pollard's Rho算法之前，需要先测试$n$是否是合数，这里可以采用随机算法Miller Rabin进行，这里不过多讲。

```java
int pollard_rho(int x, int c, int n)
{
    int xi = x;
    int xj = x;
    int j = 2;
    int i = 1;
    while(i < n)
    {
        i++;
        xi = (xi * xi + c) % n;
        int g = gcd(n, abs(xi - xj));
        if(g != 1 && g != n)
        {
            return g;
        }
        if(i == j)
        {
            j = j << 1;
            xj = xi;
        }
    }
    return -1;
}
```

上面的是倍增法的代码。



[1] https://en.wikipedia.org/wiki/Pollard%27s_rho_algorithm