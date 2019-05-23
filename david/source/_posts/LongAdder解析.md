---
title: LongAdder解析
tags: [java并发系统学习番外篇]
top: 202
comments: true
date: 2019-05-23 08:01:56
categories: 技术
---                                             
### LongAdder解析

![输入图片说明](http://prl6c63q7.bkt.clouddn.com/p1.jpg "在这里输入图片标题")

```
摘要： 对`LongAdder`的最初了解是从Coolshell上的一篇文章中获得的，但是一直都没有深入的了解过其实现，只知道它相较于`AtomicLong`来说，更加适合写多读少的并发情景。今天，我们就研究一下`LongAdder`的原理，探究一下它如此高效的原因。
```
对LongAdder的最初了解是从Coolshell上的一篇文章中获得的，但是一直都没有深入的了解过其实现，只知道它相较于AtomicLong来说，更加适合写多读少的并发情景。今天，我们就研究一下LongAdder的原理，探究一下它如此高效的原因。

#### 基本原理和思想

```
Java有很多并发控制机制，比如说以AQS为基础的锁或者以CAS为原理的自旋锁。不了解AQS的朋友可以阅读我之前的AQS源码解析文章。一般来说，CAS适合轻量级的并发操作，也就是并发量并不多，而且等待时间不长的情况，否则就应该使用普通锁，进入阻塞状态，避免CPU空转。

 所以，如果你有一个Long类型的值会被多线程修改，那么使用CAS进行并发控制比较好，但是如果你是需要锁住一些资源，然后进行数据库操作，那么还是使用阻塞锁比较好。

 第一种情况下，我们一般都使用AtomicLong。AtomicLong是通过无限循环不停的采取CAS的方法去设置内部的value，直到成功为止。那么当并发数比较多或出现更新热点时，就会导致CAS的失败机率变高，重试次数更多，越多的线程重试，CAS失败的机率越高，形成恶性循环，从而降低了效率。

 而LongAdder的原理就是降低对value更新的并发数，也就是将对单一value的变更压力分散到多个value值上，降低单个value的“热度”。

 我们知道LongAdder的大致原理之后，再来详细的了解一下它的具体实现，其中也有很多值得借鉴的并发编程的技巧。
```

#### LongAdder的成员变量
LongAdder是Striped64 的子类，其有三个比较重要的成员函数，在之后的函数分析中需要使用到，这里先说明一下。

```
// CPU的数量
static final int NCPU = Runtime.getRuntime().availableProcessors();
// Cell对象的数组，长度一般是2的指数
transient volatile Cell[] cells;
// 基础value值，当并发较低时，只累加该值
transient volatile long base;
// 创建或者扩容Cells数组时使用的自旋锁变量
transient volatile int cellsBusy;
```
cells是LongAdder的父类Striped64中的Cell数组类型的成员变量。每个Cell对象中都包含一个value值，并提供对这个value值的CAS操作。
```
static final class Cell {
    volatile long value;
    Cell(long x) { value = x; }
    final boolean cas(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
    }
}
```
#### Add操作

我们首先来看一下LongAdder的add函数，其会多次尝试CAS操作将值进行累加，如果成功了就直接返回，失败则继续执行。代码比较复杂，而且涉及的情况比较多，我们就以梳理历次尝试CAS操作为主线，讲清楚这些CAS操作的前提条件和场景。

```
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    // 当cells数组为null时，会进行第一次cas操作尝试。
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null || 
            !(uncontended = a.cas(v = a.value, v + x)))
            // 当cells数组不为null，并且通过getProbe() & m
            // 定位的Cell对象不为null时进行第二次CAS操作。
            // 如果执行不成功，则进入longAccumulate函数。
            longAccumulate(x, null, uncontended); 
    }
}
```

当并发量较少时，cell数组尚未初始化，所以只调用casBase函数，对base变量进行CAS累加。

![输入图片说明](https://images.gitee.com/uploads/images/2019/0523/074538_6154a8cf_1728187.png "屏幕截图.png")

我们来看一下casBase函数相关的源码吧。我们可以认为变量base就是第一个value值，也是基础value变量。先调用casBase函数来cas一下base变量，如果成功了，就不需要在进行下面比较复杂的算法，

```
final boolean casBase(long cmp, long val) {
    return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
}
```

当并发量逐渐提高时，casBase函数会失败。如果cells数组为null或为空,就直接调用longAccumulate方法。因为cells为null或在为空，说明cells未

初始化，所以调用longAccumulate进行初始化。否则继续判断。
 如果cells中已经初始化，就继续进行后续判断。我们先来理解一下getProbe() & m的这个操作吧，可以把这个操作当作一次计算"hash"值，然后将cells中这个位置的Cell对象赋值给变量a。如果变量a不为null，那么就调用该对象的cas方法去设置其value值。如果a为null，或在cas赋值发生冲突，那么调用longAccumulate方法。

![输入图片说明](https://images.gitee.com/uploads/images/2019/0523/074837_29dc2078_1728187.png "屏幕截图.png")

#### LongAccumulate方法

longAccumulate函数比较复杂，带有我的注释的代码已经贴在了文章后边，这里我们就只讲一下其中比较关键的一些技巧和思想。

 首先，我们都知道只有当对base的cas操作失败之后，LongAdder才引入Cell数组．所以在longAccumulate中就是对Cell数组进行操作，分别涉及了数组的初始化，扩容和设置某个位置的Cell对象等操作。

 在这段代码中，关于cellBusy的cas操作构成了一个SpinLock，这就是经典的SpinLock的编程技巧，大家可以学习一下。

 我们先来看一下longAccumulate的主体代码，首先是一个无限for循环，然后根据cells数组的状态来判断是要进行cells数组的初始化，还是进行对象添加或者扩容。

```
final void longAccumulate(long x, LongBinaryOperator fn,
                             boolean wasUncontended) {
       int h;
       if ((h = getProbe()) == 0) { 
           //获取PROBE变量，探针变量，与当前运行的线程相关，不同线程不同
           ThreadLocalRandom.current(); 
       //初始化PROBE变量，和getProbe都使用Unsafe类提供的原子性操作。
           h = getProbe();
           wasUncontended = true;
       }
       boolean collide = false;
       for (;;) { //cas经典无限循环，不断尝试
           Cell[] as; Cell a; int n; long v;
           if ((as = cells) != null && (n = as.length) > 0) { 
           // cells不为null,并且数组size大于0,表示cells已经初始化了
           // 初始化Cell对象并设置到数组中或者进行数组扩容
           }
           else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
           //cells数组未初始化，获得cellsBusy lock,进行cells数组的初始化
           // cells数组初始化操作
           }
          //如果初始化数组失败了，那就再次尝试一下直接cas base变量，
          // 如果成功了就直接返回，这是最后一个进行CAS操作的地方。
           else if (casBase(v = base, ((fn == null) ? v + x :
                                       fn.applyAsLong(v, x))))
               break;
       }
   }
```

进行Cell数组代码如下所示，它首先调用casCellsBusy函数获取了cellsBusy‘锁’，然后进行数组的初始化操作，最后将cellBusy'锁'释放掉。

```
// 注意在进入这段代码之前已经casCellsBusy获得cellsBusy这个锁变量了。
boolean init = false;
try {
    if (cells == as) {
        Cell[] rs = new Cell[2];
        rs[h & 1] = new Cell(x); //设置x的值为cell对象的value值
        cells = rs;
        init = true;
    }
} finally {
    cellsBusy = 0;
}
if (init)
    break;
```

![输入图片说明](https://images.gitee.com/uploads/images/2019/0523/074946_a12a6605_1728187.png "屏幕截图.png")

如果Cell数组已经初始化过了，那么就进行Cell数组的设置或者扩容。这部分代码有一系列的if else的判断，如果前一个条件不成立，才会进入下一条判断。

 首先，当Cell数组中对应位置的cell对象为null时，表明该位置的Cell对象需要进行初始化，所以使用casCellsBusy函数获取'锁'，然后初始化Cell对象，并且设置进cells数组，最后释放掉'锁'。

 当Cell数组中对应位置的cell对象不为null，则直接调用其cas操作进行累加。

 当上述操作都失败后，认为多个线程在对同一个位置的Cell对象进行操作，这个Cell对象是一个“热点”，所以Cell数组需要进行扩容，将热点分散。

```

if ((a = as[(n - 1) & h]) == null) { //通过与操作计算出来需要操作的Cell对象的坐标
    if (cellsBusy == 0) { //volatile 变量，用来实现spinLock,来在初始化和resize cells数组时使用。
    //当cellsBusy为0时，表示当前可以对cells数组进行操作。 
        Cell r = new Cell(x);//将x值直接赋值给Cell对象
        if (cellsBusy == 0 && casCellsBusy()) {//如果这个时候cellsBusy还是0
        //就cas将其设置为非０，如果成功了就是获得了spinLock的锁．可以对cells数组进行操作．
        //如果失败了，就会再次执行一次循环
            boolean created = false;
            try {
                Cell[] rs; int m, j;
                //判断cells是否已经初始化，并且要操作的位置上没有cell对象．
                if ((rs = cells) != null &&
                    (m = rs.length) > 0 &&
                    rs[j = (m - 1) & h] == null) {
                    rs[j] = r;　//将之前创建的值为x的cell对象赋值到cells数组的响应位置．
                    created = true;
                }
            } finally {
                //经典的spinLock编程技巧，先获得锁，然后try finally将锁释放掉
                //将cellBusy设置为0就是释放锁．
                cellsBusy = 0;
            }
            if (created)
                break;　//如果创建成功了，就是使用x创建了新的cell对象，也就是新创建了一个分担热点的value
            continue; 
        }
    }
    collide = false; //未发生碰撞
}
else if (!wasUncontended)//是否已经发生过一次cas操作失败
    wasUncontended = true; //设置成true,以便第二次进入下一个else if 判断
else if (a.cas(v = a.value, ((fn == null) ? v + x :
                            fn.applyAsLong(v, x))))
    　//fn是操作类型，如果是空，就是相加，所以让a这个cell对象中的value值和x相加，然后在cas设置，如果成果
    //就直接返回
    break;
else if (n >= NCPU || cells != as)
　　//如果cells数组的大小大于系统的可获得处理器数量或在as不再和cells相等．
    collide = false;
else if (!collide)
    collide = true;
else if (cellsBusy == 0 && casCellsBusy()) {
　　//再次获得cellsBusy这个spinLock,对数组进行resize
    try {
        if (cells == as) {//要再次检测as是否等于cells以免其他线程已经对cells进行了操作．
            Cell[] rs = new Cell[n << 1]; //扩容一倍
            for (int i = 0; i < n; ++i)
                rs[i] = as[i];
            cells = rs;//赋予cells一个新的数组对象
        }
    } finally {
        cellsBusy = 0;
    }
    collide = false;
    continue;
}
h = advanceProbe(h);//由于使用当前探针变量无法操作成功，所以重新设置一个,再次尝试

```

![输入图片说明](https://images.gitee.com/uploads/images/2019/0523/075028_9cdebffc_1728187.png "屏幕截图.png")

后记
 本篇文章写的不是很好，我写完之后又看了一遍coolshell上的关于LongAdder的文章，感觉自己没有人家写的那么简洁明了。我对代码细节的注释和投入太多了。其实很多代码大家都可以看懂，并不需要大量的代码片段加注释。以后要注意一下。之后会接着研究一下JUC包中的其他类，希望大家多多关注。

```
文章来源： https://yq.aliyun.com/articles/688822
```
