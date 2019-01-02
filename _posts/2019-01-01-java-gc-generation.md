---
categories: java
layout: post
---

本文翻译于：[**Java Platform, Standard Edition HotSpot Virtual Machine Garbage Collection Tuning Guide**](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/generations.html)

一个对象被当成垃圾当且仅当它不能再通过运行中程序的的指针进行直接或间接引用。最直接的垃圾回收算法就是遍历所有可访问的对象，所有被留下的对象均被视作垃圾。这种方式下垃圾回收执行的时间将与存活的对象数目成正比，对于管理大量存活对象的大型应用来说是禁用的。

虚拟机通过分代回收的方式整合了多个不同的垃圾回收算法。原始的垃圾回收会检查堆中所有存活的对象，分代回收参考了数个通过经验观察得到的性质，用于减少回收垃圾对象所需的工作。其中最重要的观察得到的性质是弱分代假设（weak generational hypothesis），其声明了大部分的对象仅存活极短暂的时间。

下图中蓝色区域就是对象声明周期的典型分布，x轴表示以字节分配为单位的对象生命时间，而y轴表示在该生存时间内的对象所占用的总字节数。左边的高峰表示哪些能在创建后较短时间内回收的对象，比如说像迭代器这种在循环后就失去生命的对象。

![Description of "Figure 3-1 Typical Distribution for Lifetimes of Objects"](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/img/jsgct_dt_003_alc_vs_srvng.png)

一些对象活得较久，所以分布图延伸到了右边。比如说，总有一些对象会在初始化时被创建，并且一直存活到进程结束。在两个极端之间的对象的生存期为一些快速计算的期间，对应于上图中的高峰右边的一些块状。有一些应用会有着截然不同的分布，但是大部分的应用都遵循了图上的描述。通过关注大部分对象都在年幼的时候就死去这一事实，使得高效的回收变得可能。

为了在这个场景做出优化，内存被组织为数代（内存池持有不同年龄的对象）。当某一代被耗尽时，就会在该代发生垃圾回收。大部分对象在年轻代（young generation）中分配，并且其中绝大部分也在这里死去。当年轻代被耗尽时，将触发一次微回收（minor collect），仅清理年轻代，其它几代中的对象不会被回收。在弱分代假设始终成立并且年轻代中的大多数对象都是能够被回收的垃圾的前提下，微回收可以被优化。这类回收的成本，正比于存活的对象的数目，而一个充满死去对象的年轻代可以很快地被回收。典型的，在每次微清理期间年轻代中部分存活的对象会被移动到终生代（tenured generation）。最终，终生代也会被耗尽，这会导致一次主回收（major collection），主回收的垃圾回收范围是整个堆。由于涉及多得多的对象，主回收往往会比微回收要慢得多。

![**Figure 3-2 Default Arrangement of Generations, Except for Parallel Collector and G1**](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/img/jsgct_dt_001_armgnt_gn.png)

在初始时，一个最大地址空间被实际上保留下来，但是并不会分配给物理内存直到确实需要。完整的保留地址空间被切分为年轻代和终生代。

年轻代包含伊甸园（eden）和两个幸存者空间（survivor space）。大部分对象都是在伊甸园中分配。在任何时候至少一个幸存者空间为空，并且作为伊甸园中存活对象的目的地。而另外一个幸存者空间则作为下一次复制回收（copying collection）的目的地。对象在幸存者空间之间不断被拷贝，直到它们足够年老，可以进入终生代（拷贝到终生代中）。

