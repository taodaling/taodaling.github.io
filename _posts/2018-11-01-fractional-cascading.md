---
categories: algorithm
layout: post
---

# 问题

给出k个有序的数组（最长的数组长度为n）。要求回答若干次下面的提问：

给出一个值x，求解每个数组中最大的不超过x的值。

# 解法

一般的解法非常简单，对每个数组都调用二分，每次回答提问所需的时间复杂度大致为O(klog2n)。

# Fractional cascading

Fractional cascading能够以O(log2n+k)的时间复杂度回答提问。为了避免问题的复杂化，我们将无穷小和无穷大加入每一个数组之中。

首先我们将数组标号为A[1],A[2],...,A[k]。

之后我们将每个数组进行扩展。对于数组A[i]，我们将其扩张为'A[i]。'A[i]为将A[i+1]中所有偶数序号元素加入到A[i]后重新排序得到的结果。同时我们为A\[i\]\[j\]增加两个属性，last和current,group,next。

其中A\[i\][j].last表示在'A[i-1]中处于其之前的来自A[i-1]中元素的current值，current记录j，group记录i，next表示在'A[i]中处于其之后的来自其A[i+1]中元素的current值。

很显然上面的预处理过程能在O(kn)的时空复杂度内完成。

对于一次提问x。我们首先利用二分查找法在O(log2n)的时间复杂度内找到'A[1]中最大的不超过x的元素v，之后令answer[v.group]=v。若v.group为2，则我们设置answer[1]=v.last。

对于组i，若我们找到了A[i-1]中小于等于x的最大值answer[i-1]。我们可以保证answer[i]的可能值一定为answer[i-1].last,answer[i-1].next,A[answer[i-1].last.current+1]之中，总共只有三个可能值。

可以发现上面算法除了第一组的结果需要二分查找得到外，其他的均最多只需要常量时间得到，因此后续每次回答提问所需的时间复杂度为O(log2n+k)。