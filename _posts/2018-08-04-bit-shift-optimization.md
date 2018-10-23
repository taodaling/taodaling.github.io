---
layout: post
categories: algorithm
---

# 问题描述：
```
假设给定数正整数n。设x初始值为0，我们可以对x做如下两种操作任意次：
1. x += 2^i
2. x -= 2^i
其中i为任意非负整数。要求我们计算最少多少次操作后x等于n。
```

# 题解：
```
首先不加证明给出下面这个点：
最优解中每个操作中作为指数的i均不同。
每个操作中指定的指数i必定不低于x的最靠后的1。

由第二条性质，我们可以保证x最低位为1（如果不是则令x右移位）。

下面以二进制角度观察数n和x。重新解释两种操作：
1. 将x的某一位设置为1或0
2. 将x按位反转后加1

下面我们用动态规划来解决这个问题。
我们称N[i]表示n的第i位，而N[i..]表示n的二进制后i位，-N[i..]表示n的二进制后i位取反后加1。用P[i]表示从0到N[i..]所需最少的步骤数，而R[i]表示从0到-N[i..]所需最少的步骤数。

由于N[1..]=-N[1..]=1，因此P[i]=R[i]=1

对于i>1，考虑下面两种情况：
1. N[i]=0，那么有
P[i]=min({P[i-1], R[i-1]+1});
R[i]=min({P[i-1]+1, R[i-1]+1});

2.N[i]=1，那么有
P[i]=min({P[i-1]+1, R[i-1]+1});
R[i]=min({P[i-1]+1,R[i-1]});

假设n共有k位，则我们需要的结果为P[k]。总的时间复杂度为O(log2(n))。
```

# 代码
```java
public class BitShiftProblem {
    public int solve(int n) {
        if (n < 0) {
            n = -n;
        }
        if (n == 0) {
            return 0;
        }

        while ((n & 1) == 0) {
            n >>= 1;
        }

        n >>= 1;
        int p = 1;
        int r = 1;

        while (n != 0) {
            int lowest = n & 1;
            n >>= 1;

            int lastP = p;
            int lastR = r;
            if (lowest == 0) {
                p = Math.min(lastP, lastR + 1);
                r = Math.min(lastP + 1, lastR + 1);
            } else {
                p = Math.min(lastP + 1, lastR + 1);
                r = Math.min(lastP + 1, lastR);
            }
        }

        return p;
    }
}
```
