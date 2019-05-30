title: java并发计数器分析
tags: [java并发系统学习番外篇]
top: 201
comments: true
abbrlink: 51589
categories: 技术
date: 2019-05-22 19:04:00
---
![](http://prl6c63q7.bkt.clouddn.com/p12.jpg)
#### 前言

    一提到线程安全的并发计数器，AtomicLong 必然是第一个被联想到的工具。Atomic* 一系列的原子类以及它们背后的 CAS 无锁算法，常常是高性能，高并发的代名词。本文将会阐释，在并发场景下，使用 AtomicLong 来充当并发计数器将会是一个糟糕的设计，实际上存在不少 AtomicLong 之外的计数器方案。近期我研究了一些 Jdk1.8 以及 JCTools 的优化方案，并将它们的对比与实现细节整理于此。
<!-- more -->
    
#### AtomicLong 的前世今生
    在 Java 中，Atomic* 是高效的，这得益于 sun.misc.Unsafe 提供的一系列底层 API，使得 Java 这样的高级语言能够直接和硬件层面的 CPU 指令打交道。并且在 Jdk1.7 中，这样的底层指令可以配合 CAS 操作，达到 Lock-Free。
    
    在 Jdk1.7 中，AtomicLong 的关键代码如下：
    
```
public final long getAndIncrement() {
    while (true) {
        long current = get();
        long next = current + 1;
        if (compareAndSet(current, next))
            return current;
    }
}

public final boolean compareAndSet(long expect, long update) {
    return unsafe.compareAndSwapLong(this, valueOffset, expect, update);
}
```
    1. get() 方法 volatile 读当前 long 值
    2. 自增
    3. 自旋判断新值与当前值
    4. 自旋成功，返回；否则返回 1
    我们特别留意到 Jdk1.7 中 unsafe 使用的方法是 compareAndSwapLong，它与       x86 CPU 上的 LOCK CMPXCHG 指令对应，并且在应用层使用 while(true) 完成自     旋，这个细节在 Jdk1.8 中发生了变化。

    在 Jdk1.8 中，AtomicLong 的关键代码如下：
```
    public final long getAndIncrement() {
    	return unsafe.getAndAddLong(this, valueOffset, 1L);
    }
    ```
Jdk1.7 的 CAS 操作已经不复存在了，转而使用了 getAndAddLong 方法，它与 x86 CPU 上的 LOCK XADD 指令对应，以原子方式返回当前值并递增（fetch and add）。

``` 当问及 Atomic* 高效的原因，回答 CAS 是不够全面且不够严谨的，Jdk1.7 的 unsafe.compareAndSwapLong 以及 Jdk1.8 的 unsafe.getAndAddLong 才是关键，且 Jdk1.8 中不存在 CAS。 ```

Jdk1.8 AtomicLong 相比 Jdk1.7 AtomicLong 的表现是要优秀的，这点我们将在后续的测评中见证。

#### Jdk1.8 AtomicLong 相比 Jdk1.7 AtomicLong 的表现是要优秀的，这点我们将在后续的测评中见证。

无论在 Jdk1.7 还是 Jdk1.8 中，Atomic* 的开销都是很大的，主要体现在：
```
1. 高并发下，CAS 操作可能会频繁失败，真正更新成功的线程占少数。(Jdk1.7 独有的问题)

2. 我之前的文章中介绍过“伪共享” (false sharing) 问题，但在 CAS 中，问题则表现的更为直接，这是“真共享”，与”伪共享“存在相同的问题：缓存行失效，缓存一致性开销变大。

3. 底层指令的开销不见得很低，无论是 LOCK XADD 还是 LOCK CMPXCHG，想深究的朋友可以参考 instruction_tables ，（这一点可能有点钻牛角尖，但不失为一个角度去分析高并发下可行的优化）

4. Atomic* 所做的，比我们的诉求可能更大，有时候我们只需要计数器具备线程安全地递增这样的特性，但 Atomic* 的相关操作每一次都伴随着值的返回。他是个带返回值的方法，而不是 void 方法，而多做了活大概率意味着额外的开销。 ```

抛开上述导致 AtomicLong 慢的原因，AtomicLong 仍然具备优势：

```

1. 上述的第 4 点换一个角度也是 AtomicLong 的有点，相比下面要介绍的其他计数器方案，AtomicLong 能够保证每次操作都精确的返回真实的递增值。你可以借助 AtomicLong 来做并发场景下的递增序列号方案，注意，本文主要讨论的是计数器方案，而不是序列号方案。

2. 实现简单，回到那句话：“简单的架构通常性能不高，高性能的架构通常复杂度很高”，AtomicLong 属于性能相对较高，但实现极其简单的那种方案，因为大部分的复杂性，由 JMM 和 JNI 方法屏蔽了。相比下面要介绍的其他计数器实现，AtomicLong 真的太“简易”了。 ```

![upload successful](http://prl6c63q7.bkt.clouddn.com/AtomicLongSpeet.png)

```
横向对比，写的性能相比读的性能要差很多，在 20 个线程下写性能比读性能差距了 4~5 倍。

纵向对比，主要关注并发写，线程竞争激烈的情况下，单次自增耗时从 22 ns 增长为了 488 ns，有明显的性能下降。

实际场景中，我们需要统计系统的 qps、接口调用次数，都需要使用到计数的功能，写才是关键，并不是每时每刻都需要关注自增后的返回值，而 AtomicLong 恰恰在核心的写性能上有所欠缺。由此引出其他计数器方案。 ```

#### 认识 LongAdder
Doug Lea 在 JDK1.8 中找到了一个上述问题的解决方案，他实现了一个 LongAdder 类。
```
@since 1.8
@author Doug Lea
public class LongAdder extends Striped64 implements Serializable {}
```
LongAdder 的 API 如下

![upload successful](http://prl6c63q7.bkt.clouddn.com/longAddr.png)

你应当发现，LongAdder 和 AtomicLong 明显的区别在于，increment 是一个 void 方法。直接来看看 LongAdder 的性能表现如何。(LA = LongAdder, AL = AtomicLong, 单位 ns/op)

![upload successful](http://prl6c63q7.bkt.clouddn.com/longAddrSpett.png)

我们从中可以发现一些有意思的现象，网上不少很多文章没有从读写上对比二者，直接宣称 LongAdder 性能优于 AtomicLong，其实不太严谨。在单线程下，并发问题没有暴露，两者没有体现出差距；随着并发量加大，LongAdder 的 increment 操作更加优秀，而 AtomicLong 的 get 操作则更加优秀。鉴于在计数器场景下的特点—写多读少，所以写性能更高的 LongAdder 更加适合。

#### LongAdder 写速度快的背后

网上分析 LongAdder 源码的文章并不少，我不打算详细分析源码，而是挑选了一些必要的细节以及多数文章没有提及但我认为值得分析的内容。

![upload successful](http://prl6c63q7.bkt.clouddn.com/cell.png)
```
1. Cell 设计减少并发修改时的冲突
在 LongAdder 的父类 Striped64 中存在一个 volatile Cell[] cells; 数组，其长度是 2 的幂次方，每个 Cell 都填充了一个 @Contended 的 Long 字段，为了避免伪共享问题。 ```

``` @sun.misc.Contended static final class Cell {
    volatile long value;
    Cell(long x) { value = x; }
    // ... ignore
} ```

```LongAdder 通过一系列算法，将计数结果分散在了多个 Cell 中，Cell 会随着并发量升高时发生扩容，最坏情况下 Cell == CPU core 的数量。Cell 也是 LongAdder 高效的关键，它将计数的总值分散在了各个 Cell 中，例如 5 = 3 + 2，下一刻，某个线程完成了 3 + (2 + 1) = 6 的操作，而不是在 5 的基础上完成直接相加操作。通过 LongAdder 的 sum() 方法可以直观的感受到这一点（LongAdder 不存在 get 方法）```
```
public long sum() {
    Cell[] as = cells; Cell a;
    long sum = base;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}```

```这种惰性求值的思想，在 ConcurrentHashMap 中的 size() 中也存在，毕竟他们的作者都是 Doug Lea。```

```
2. 并发场景下高效获取随机数

LongAdder 内部算法需要获取随机数，而 Random 类在并发场景下也是可以优化的。```

```
ThreadLocalRandom random =  ThreadLocalRandom.current();
random.nextInt(5);```

```使用 ThreadLocalRandom 替代 Random，同样出现在了 LongAdder 的代码中。```

```
3. longAccumulate
longAccumulate 方法是 LongAdder 的核心方法，内部存在大量的分支判断。首先和 Jdk1.7 的 AtomicLong 一样，它使用的是 UNSAFE.compareAndSwapLong 来完成自旋，不同之处在于，其在初次 cas 方式失败的情况下(说明多个线程同时想更新这个值)，尝试将这个值分隔成多个 Cell，让这些竞争的线程只负责更新自己所属的 Cell，这样将竞争压力分散开。 ```

#### LongAdder 的前世今生
```
其实在 Jdk1.7 时代，LongAdder 还未诞生时，就有一些人想着自己去实现一个高性能的计数器了，比如一款 Java 性能监控框架 dropwizard/metrics 就做了这样事，在早期版本中，其优化手段并没有 Jdk1.8 的 LongAdder 丰富，而在 metrics 的最新版本中，其已经使用 Jdk1.8 的 LongAdder 替换掉了自己的轮子。在最后的测评中，我们将 metrics 版本的 LongAdder 也作为一个参考对象。```

#### JCTools 中的 ConcurrentAutoTable
```
并非只有 LongAdder 考虑到了并发场景下计数器的优化，大名鼎鼎的并发容器框架 JCTool 中也提供了和今天主题相关的实现，虽然其名称和 Counter 看似没有关系，但通过其 Java 文档和 API ，可以发现其设计意图考虑到了计数器的场景。 ```

![upload successful](http://prl6c63q7.bkt.clouddn.com/ConcurrentAutoTable%20.png)
```
在最后的测评中，我们将 JCTools 的 ConcurrentAutoTable 也作为一个参考对象。```

#### 最终测评
```
Jdk1.7 的 AtomicLong，Jdk1.8 的 AtomicLong，Jdk 1.8 的 LongAdder，Metrics 的 LongAdder，JCTools 的 ConcurrentAutoTable，我对这五种类型的计数器使用 JMH 进行基准测试。```

```
public interface Counter {
    void inc();
    long get();
}
```
```
将 5 个类都适配成 Counter 接口的实现类，采用 @State(Scope.Group)，@Group 将各组测试用例进行隔离，尽可能地排除了互相之间的干扰，由于计数器场景的特性，我安排了 20 个线程进行并发写，1 个线程与之前的写线程共存，进行并发读。Mode=avgt 代表测试的是方法的耗时，越低代表性能越高。```

![](http://prl6c63q7.bkt.clouddn.com/test.png)

```
如果我们只关注 inc 即写性能，可以发现 jdk1.8 的 LongAdder 表现的最为优秀，ConcurrentAutoTable 以及两个版本的 LongAdder 在一个数量级之上；1.8 的 AtomicLong 相比 1.7 的 AtomicLong 优秀很多，可以得出这样的结论，1.7 的 CAS+LOCK CMPXCHG 方案的确不如 1.8 的 LOCK XADD 来的优秀，但如果与特地优化过的其他计数器方案来进行比较，便相形见绌了。

如果关注 get 性能，虽然这意义不大，但可以见得，AtomicLong 的 get 性能在高并发下表现依旧优秀，而 LongAdder 组合求值的特性，导致其性能必然存在一定下降，位列第二梯队，而 ConcurrentAutoTable 的并发读性能最差。

关注整体性能，CounterBenchmark.rw 是对一组场景的整合打分，可以发现，在我们模拟的高并发计数器场景下，1.8 的 LongAdder 获得整体最低的延迟 98 ns，相比性能最差的 Jdk1.7 AtomicLong 实现，高了整整 10 倍有余，并且，随着并发度提升，这个数值还会增大。```

#### AtomicLong 可以被废弃吗？
既然 LongAdder 的性能高出 AtomicLong 这么多，我们还有理由使用 AtomicLong 吗？

```
本文重点讨论的角度还是比较局限的：单机场景下并发计数器的高效实现。AtomicLong 依然在很多场景下有其存在的价值，例如一个内存中的序列号生成器，AtomicLong 可以满足每次递增之后都精准的返回其递增值，而 LongAdder 并不具备这样的特性。LongAdder 为了性能而丧失了一部分功能，这体现了计算机的哲学，无处不在的 trade off。```

#### 高性能计数器总结
- AtomicLong ：并发场景下读性能优秀，写性能急剧下降，不适合作为高性能的计数器方案。内存需求量少。

- LongAdder ：并发场景下写性能优秀，读性能由于组合求值的原因，不如直接读值的方案，但由于计数器场景写多读少的缘故，整体性能在几个方案中最优，是高性能计数器的首选方案。由于 Cells 数组以及缓存行填充的缘故，占用内存较大。

- ConcurrentAutoTable ：拥有和 LongAdder 相近的写入性能，读性能则更加不如 LongAdder。它的使用需要引入 JCTools 依赖，相比 Jdk 自带的 LongAdder 并没有优势。但额外说明一点，ConcurrentAutoTable 的使用并非局限于计数器场景，其仍然存在很大的价值。

***在前面提到的性能监控框架 Metrics，以及著名的熔断框架 Hystrix 中，都存在 LongAdder 的使用场景，有兴趣的朋友快去实践一下 LongAdder 吧。***

```
本文所有的 JMH 测试代码，均可在我的 github 中获得：https://github.com/lexburner/JMH-samples.git

微信不支持外部超链接，文中相关仓库附录：

Netflix/Hystrix : https://github.com/Netflix/Hystrix

Metrics : https://github.com/dropwizard/metrics

JCTools : https://github.com/JCTools/JCTools

instruction_tables : https://www.agner.org/optimize/instruction_tables.pdf

本文转载于：https://mp.weixin.qq.com/s/yAvJFZWxfKb38IDMjQd5zg?spm=a2c4e.11153940.blogcont651530.11.303e7bebu6FTOM 
```



    