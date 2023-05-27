---
title: System.currentTimeMillis性能问题测试与处理   
date: 2019-07-10 10:21:52   
categories: "Java"  
tags: [性能 时间戳]    
description: Java_System.currentTimeMillis()性能问题测试与处理    

---

****背景****     
新项目打算做前后端分离，需要关注接口的响应时间，以便在出现问题时定位到是哪个接口，记得偶然看到获取多线程并发环境下该方法对性能可能会有影响，因此在项目前期做一个小测试。         




****过程****    

System.currentTimeMillis()在java中是最常用的获取系统时间的方法，它返回的是1970年1月1日0点到现在经过的毫秒数。    我们看以看到它在Java中的实现为：
`public static native long currentTimeMillis();` 

native关键字说明这个方法底层是由C语言实现的，我们稍后在讲，先做一个简单的测试。

这次测试电脑配置为 Intel(R) Core(TM) i7-8550 16内存。测试很简单10000次for循环调用和多线程10000次调用，直接看结果：   
单线程下10000次调用消耗时间为 6713300 ns    
多线程下10000次调用消耗时间为 4968593100 ns  

可以看到相差还是比较大的,在串行情况下这个api其实性能很好，但是在并发情况下回急剧下降，原因在于计时器在所有进程之间共享，并且其还一直在发生变化，当大量线程尝试同时去访问计时器的时候，就涉及到资源的竞争，于是也就出现并行效率远低于串行效率的现象了。所以在高并发场景下要慎重使用System.nanoTime()和System.currentTimeMillis()这两个API。  

在搜索过程中我找到的一篇大家都比较信服的文章来解释为什么会这样，连接如下:   
[http://pzemtsov.github.io/2017/07/23/the-slow-currenttimemillis.html](http://pzemtsov.github.io/2017/07/23/the-slow-currenttimemillis.html "The slow currentTimeMillis()")    

​


文章很长，讲的很详细，甚至从汇编语言的角度讲了为什么会这样，有几个比较重要的观点： 

>* 调用gettimeofday()需要从用户态切换到内核态；   
>* gettimeofday()的表现受Linux系统的计时器（时钟源）影响，在HPET计时器下性能尤其差；   
>* 系统只有一个全局时钟源，高并发或频繁访问会造成严重的争用


HPET计时器问题在处理器层面已经解决
处理器系列以不同方式增加时间戳计数器：

> 对于奔腾M处理器（系列[06H]，型号[09H，0DH]）；对于奔腾4处理器，英特尔至强处理器（系列[0FH]，型号[00H，01H或02H]）；对于P6系列处理器：时间戳记计数器会随着每个内部处理器时钟周期的增加而增加。内部处理器时钟周期由当前内核时钟与总线时钟之比确定。英特尔®SpeedStep®技术过渡也可能会影响处理器时钟。
> 
> 对于奔腾4处理器，英特尔至强处理器（系列[0FH]，型号[03H及更高版本]）；适用于Intel Core Solo和Intel Core Duo处理器（系列[06H]，型号[0EH]）；用于Intel Xeon处理器5100系列和Intel Core 2 Duo处理器（系列[06H]，型号[0FH]）；适用于Intel Core 2和Intel Xeon处理器（系列[06H]，DisplayModel [17H]）；对于Intel Atom处理器（系列[06H]，DisplayModel [1CH]）：时间戳记计数器以恒定速率递增。  

****解决思路****

网上的解决方法有很多：   
1. 维护一个全局缓存，使用单线程调度器器按毫秒更新时间戳，用到了 `ScheduledThreadPoolExecutor`，代码如下    
```
public class SystemClock {
    private static final SystemClock MILLIS_CLOCK = new SystemClock(1);
    private final long precision;
    private final AtomicLong now;

    private SystemClock(long precision) {
        this.precision = precision;
        now = new AtomicLong(System.currentTimeMillis());
        scheduleClockUpdating();
    }

    public static SystemClock millisClock() {
        return MILLIS_CLOCK;
    }

    private void scheduleClockUpdating() {
        ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor(runnable -> {
            Thread thread = new Thread(runnable, "system.clock");
            thread.setDaemon(true);
            return thread;
        });
        scheduler.scheduleAtFixedRate(() -> now.set(System.currentTimeMillis()), precision, precision, TimeUnit.MILLISECONDS);
    }

    public long now() {
        return now.get();
    }

```

​ 
​