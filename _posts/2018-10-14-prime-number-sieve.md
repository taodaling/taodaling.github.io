---
categories: algorithm
layout: post
---



{:toc}

# 简介

假如我们要筛选出[1,N]之间的所有素数，我们该如何实现？下面列举几种实现方式

# 暴力法

```java
prime = []
for(i = 2; i <= N; i++)
    boolean flag = true
    for(j = 2, until = sqrt(i); j <= until; j++)
        if(i % j == 0)
        {
            flag = false;
            break;
        }
	if(flag)
        prime.add(i)
```

对于每个可能的素数i，由于如果i是一个合数，则必定可以写作i=ab，其中a或b至少一者不超过sqrt(i)。因此对于每个素数的判断时间复杂度最多为O(sqrt(N))，总的时间复杂度上界确认为O(N*sqrt(N))。

# 筛法

```java
prime = []
isComposite = [1,N] filled with false 
for(i = 2; i <= N; i++)
    if(isComposite[i])
        continue
    prime.add(i)
    for(j = i + i; j <= N; j += i)
        isComposite[j] = true
```

由于对于合数i，其必定能写作i=ab，其中b为某个素数且b<i。因此我们对于每个找到的素数都将以其作为因子的合数打上标记，就可以过滤出所有素数。

总的时间复杂度为O(N(1/2+1/3+1/4+...+1/N))=O(Nlog2N)。

# 欧几里得筛法

尽管普通的筛法已经拥有非常不错的时间复杂度，但是在筛选范围大的情况下依旧是会有点疲软。

我们观察筛法，发现其时间复杂度高的一个根本原因是对于一个合数，其被多次打上了标记，比如12，其可以表示为2\*6，也可以表示为3\*4，因此其被标记了两次。实际上一个合数被标记的次数为其所有不同素因子的个数。下面我们优化这一过程，保证每个合数仅被标记一次。

```java
prime = []
isComposite = [1,N] filled with false 
for(i = 2; i <= N; i++)
    if(!isComposite[i])
        prime.add(i)
    for(j = 0; j < prime.length() && i * prime[j] <= N; j++)
        isComposite[i * prime[j]] = true
        if(i % prime[j] == 0)
            break
```

下面证明这个算法确实是正确的且每个合数仅被标记一次。

每个合数k都由一个最小素数p，很显然x=k/p是k的最大非平凡因子，而且x的最小素因子不可能小于p。而由于当i=x时，所有小于p的素数均不是x的因子，故在prime[j]小于p时均不会导致循环跳出，故isComposite[x*p]会被执行而导致k被标记为合数。

上面证明了每个合数都被正确标记，下面证明确实仅标记一次。假设当i=y时k被再次标记了，由于p'=k/y>p，而p不是p'的因子，故p一定是y的因子。当prime[j]为p时循环将跳出，故不存在设置isComposite[y*p']的情况。

由于每个合数仅被设置一次，而算法的时间复杂度等同于isComposite被设置的次数，故总的时间复杂度时O(N)。