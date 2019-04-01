---
categories: algorithm
layout: post
---

要判断一个数$n$是否是素数，最简单的方法是遍历$2$到$\sqrt{n}$，如果其中存在$n$的因子，就意味着$n$是合数，否则就是素数。这个算法的时间复杂度为$O(\sqrt{n})$，如果$n$特别大，则不是什么好主意。

由费马小定理，可知，如果$n$是素数，对于一切$1$到$n-1$的数值$x$，都应该满足$x^{n-1}=1$。利用这个性质，我们可以过滤掉很多合数。比如$2^5=2(mod\phantom{1}6)$，因此可以断定$6$是合数。

但是$x^{n-1}=1$是$n$是素数的必要条件，而非充分条件。比如$8^8=1(mod\phantom{1}9)$，但是你不能就因此说9是素数。实际上，在$2$到$n-1$之间选择多个$x$进行测试可以大大增强结果的可信度，比如$2^8=4(mod\phantom{1}9)$。不幸的是，存在一类特殊的合数$c$，对于任意$1$到$c-1$之间的数$x$，都满足$x^{c-1}=1(mod\phantom{1}c)$，这种合数称为Carmichael数，其中的一个例子就是$561$。

要处理这类Carmichael数，我们需要引入Miller Rabin算法。其原理非常简单：


$$
x^2=1\rightarrow (x-1)(x+1)=0\rightarrow x=1\or x=n-1(mod\phantom{1}n)
$$


因此由于n-1为偶数，因此我们能要求x^{\frac{n-1}{2}}的值为1或n-1。如果x^{\frac{n-1}{2}}=1且\frac{n-1}{2}是偶数，我们可以继续使用上面过程。比如对于n=561。


$$
2^{560}\mod 561=1\\
2^{280}\mod 561=1\\
2^{140}\mod 561=67
$$


因此在第三次判断时，发现$561$不满足素数的要求。这样我们就解决了Carmichael数。

如果你随机从$1$到$n-1$中抽取$x$来印证一个奇数$n$是素数，重复上面过程$s$次，那么Miller Rabin出错的概率至多为$2^{-s}$，详情见《算法导论》。

```java
boolean mr(int n, int s)
{
	if(n == 2)
    {
        return true;
    }
    if(n % 2 == 0)
    {
        return false;
    }
    for(int i = 0; i < s; i++)
    {
        int x = RANDOM(1, n - 1);
        if(!mr0(x, p))
        {
            return false;
        }
    }
    return true;
}

boolean mr0(int x, int n)
{
    int exp = n - 1;
    while(true){
        int y = pow(x, exp) % n;
        if(y != 1 && y != n - 1)
        {
            return false;
        }
        if(y != 1 || exp % 2 == 1)
        {
            break;
        }
        exp = exp / 2;
    }
    return true;
}
```



