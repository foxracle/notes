垃圾回收算法具体实现

>翻译原文 => [plumbr Java GC handbook](https://plumbr.eu/handbook/garbage-collection-algorithms-implementations)

前文参见：
>[Java垃圾回收手册(一)：初识垃圾回收](http://www.jianshu.com/p/80fd72f48267)
>[Java垃圾回收手册(二)：Java中的垃圾回收](http://www.jianshu.com/p/4d8be1f3ad51)
>[Java垃圾回收手册(三)：垃圾回收算法基础](http://www.jianshu.com/p/c18526b589d2)

在熟悉GC算法背后的核心概念之后，我们来看看JVM提供的各种GC算法的具体实现。对于大部分JVM来说，GC算法主要分为两类，分别针对年轻代和老生代。

你可以选择JVM提供的各种GC算法。如果不显示指定的话，JVM会根据具体运行平台选择默认算法的。这篇文章我们来详细探讨一下这些算法的工作原理。

为了更直观一些，这里先列出来各种GC算法的可能组合（这些组合只针对Java 8，其他版本可能稍有不同）。


| 年轻代 | 老生代 | JVM参数|
|----|------|----|
| Incremental | Incremental | -Xincgc |
| **Serial** | **Serial** | **-XX:+UseSerialGC** |
| Parallel Scavenge | Serial | -XX:+UseParallelGC -XX:-UseParallelOldGC |
| Parallel New | Serial | N/A |
| Serial| Parallel Old | N/A |
| **Parallel Scavenge** | **Parallel Old** | **-XX:+UseParallelGC -XX:+UseParallelOldGC** |
| Parallel New | Parallel Old | N/A |
| Serial | CMS | -XX:-UseParNewGC -XX:+UseConcMarkSweepGC |
| Parallel Scavenge | CMS | N/A |
| **Parallel New** | **CMS** | **-XX:+UseParNewGC -XX:+UseConcMarkSweepGC** |
| **G1** | | **-XX:+UseG1GC** |


如果觉得这个表格看起来比较复杂，先不用担心。实际上常用的只有图中用粗线标出的4种组合，剩下要么已经被弃用，要么不再被支持或者说在实际应用环境中很少使用。所以我们接下来只讨论这几种组合的工作原理。

- Serial GC(年轻代+老生代)
- Parallel GC(年轻代+老生代)
- Parallel New(年轻代) + CMS(老生代)
- G1(本身不区分年轻代和老生代)

#Serial GC

这类垃圾回收器针对年轻代使用标记-拷贝算法，针对老生代使用标记-清除-整理算法。顾名思义，这些收集器是单线程，无法并行执行操作。他们也会产生让所有应用进程都停止的stop-the-world停顿。

这种垃圾收集器无法利用现代非常普遍的多核CPU，无论CPU有几个核，在JVM的垃圾回收过程中只能使用一个核。

使用以下参数可以针对年轻代和老生代开启该垃圾收集器：
>java -XX:+UseSerialGC com.mypackages.MyExecutableClass

该设置只有在CPU是单核，可用内存只有几百兆的JVM运行环境情况下才推荐使用。对于大部分的服务端部署来讲，很少使用这个组合。大部分服务端部署都是基于多核平台，选择Serial GC就人为限制了资源的使用，这肯定会导致资源浪费，而这些资源本来可以用来降低延迟或者提高吞吐量。

我们现在来看看在使用Serial GC算法时的GC日志格式，看从中我们能获得哪些信息。为此，我们需要打开JVM的GC日志功能。
>-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps

此时的GC日志格式如下：
```
2015-05-26T14:45:37.987-0200: 151.126: [GC (Allocation Failure) 151.126: [DefNew: 629119K->69888K(629120K), 0.0584157 secs] 1619346K->1273247K(2027264K), 0.0585007 secs] [Times: user=0.06 sys=0.00, real=0.06 secs]
2015-05-26T14:45:59.690-0200: 172.829: [GC (Allocation Failure) 172.829: [DefNew: 629120K->629120K(629120K), 0.0000372 secs]172.829: [Tenured: 1203359K->755802K(1398144K), 0.1855567 secs] 1832479K->755802K(2027264K), [Metaspace: 6741K->6741K(1056768K)], 0.1856954 secs] [Times: user=0.18 sys=0.00, real=0.18 secs]
```

GC日志片段暴露了很多信息，告诉我们JVM内部正在发生什么。事实上，这段日志表明有两个GC事件发生了：一个是针对年轻代的GC，一个是针对整个堆的。我们逐个来具体分析一下日志的具体含义。

##Minor GC

>2015-05-26T14:45:37.987-0200<sup>1</sup>:151.126<sup>2</sup>:[GC<sup>3</sup>(Allocation Failure<sup>4</sup>) 151.126: [DefNew<sup>5</sup>:629119K->69888K<sup>6</sup>(629120K)<sup>7</sup>, 0.0584157 secs]1619346K->1273247K<sup>8</sup>(2027264K)<sup>9</sup>,0.0585007 secs<sup>10</sup>][Times: user=0.06 sys=0.00, real=0.06 secs]<sup>11</sup>

1. 2015-05-26T14:45:37.987-0200：GC开始时间
2. 151.126：GC开始时间，相对JVM启动时间，单位是秒
3. GC：区分是Minor GC还是Full GC，这里表示Minor GC
4. Allocation Failure：产生GC的原因，这里是因为无法在年轻代为某个数据结构分配空间导致触发GC
5. DefNew：收集器名字：一个单线程，使用标记-拷贝算法，会产生的STW的垃圾收集器
6. 629119K->69888K：GC前后年轻代使用情况
7. (629120K)：年轻代总容量
8. 1619346K->1273247K：GC前后堆使用情况
9. (2027264K)：堆总容量
10. 0.0585007 secs：GC事件持续时间，单位是秒
11. [Times: user=0.06 sys=0.00, real=0.06 secs]：GC事件的时间开销，分三个类别：
 - user：整个过程GC耗费的全部CPU时间 **=> 垃圾收集器消耗的所有CPU执行时间之和，所谓的CPU time**
 - sys：系统耗费时间，包括系统调用或者等待系统事件
 - real：应用程序因GC被停止的时间。因为Serial GC是单线程的，real time等于user time + sys time  **=> 所谓的wall clock time，字面意思，墙上时间，该过程时钟走过的时间**

从上面的日志片段我们非常清楚的知道在GC发生过程中，JVM里面各个内存空间的使用情况。GC之前，堆内存总共使用了1619346K，这其中年轻代内存使用了629119K，从而可以推算出老生代的内存使用量为990,227K。

我们通过简单的计算还可以从这里面了解到更重要的信息：GC之后，年轻代内存使用量减少559,231K，但是整个堆的内存使用量只减少346,099K，从而我们可以推算出，在该GC过程中，有213,132K大小的对象从年轻代晋升到老生代。

用图来展示GC前和GC后内存使用变化如下：

![serial-gc-in-young-generation.png](http://upload-images.jianshu.io/upload_images/5475750-b5354cd7d2e81c30.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##Full GC

>2015-05-26T14:45:59.690-0200<sup>1</sup>: 172.829<sup>2</sup>:[GC (Allocation Failure) 172.829: [DefNew: 629120K->629120K(629120K), 0.0000372 secs<sup>3</sup>]172.829:[Tenured<sup>4</sup>: 1203359K->755802K<sup>5</sup>(1398144K)<sup>6</sup>,0.1855567 secs<sup>7</sup>] 1832479K->755802K<sup>8</sup>(2027264K)<sup>9</sup>,[Metaspace: 6741K->6741K(1056768K)]<sup>10</sup> [Times: user=0.18 sys=0.00, real=0.18 secs]<sup>11</sup>

1. 2015-05-26T14:45:59.690-0200：GC开始时间
2. 172.829：GC开始时间，相对JVM启动时间，单位是秒
3. [DefNew: 629120K->629120K(629120K), 0.0000372 secs：和上面的例子类似，因为内存分配失败导致了一次年轻代的GC，同样是一个叫做DefNew的收集器被运行了，GC后，年轻代内存使用从629120K降到0。这里日志里显示GC后依旧是629120K是JVM的bug
4. Tenured：老生代垃圾收集器名字：一个单线程，使用标记-清除-整理算法，会产生STW的垃圾回收器
5. 1203359K->755802K：GC前后老生代使用情况
6. (1398144K)：老生代总容量
7. 0.1855567 secs：老生代GC所耗时间
8. 1832479K->755802K：GC(年轻代+老生代)前后堆使用情况
9. (2027264K)：堆总容量
10. [Metaspace: 6741K->6741K(1056768K)]：关于元空间的类似信息
11. [Times: user=0.18 sys=0.00, real=0.18 secs]：GC事件的时间开销，分三个类别：
 - user：整个过程GC耗费的全部CPU时间
 - sys：系统耗费时间，包括系统调用或者等待系统事件
 - real：应用程序因GC被停止的时间。因为Serial GC是单线程的，real time等于user time + sys time

这个跟Minor GC的区别非常明显：除了年轻代，在这次GC过程中，老生代和元空间也被清除了。

用图来展示GC前和GC后内存使用变化如下：

![serial-gc-in-old-gen-java.png](http://upload-images.jianshu.io/upload_images/5475750-415ecdcce476dca2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#Parallel GC

这个组合包括在年轻代使用标记-复制算法的GC，在老生代使用标记-清除-整理算法的GC。年轻代和老生代的GC都会导致STW事件产生，所有应用在GC过程中都必须停下来。但在标记和复制/整理阶段是使用多线程，这也就是为什么称之为“并行”GC。使用这个算法，可以大大减少GC时间。

该垃圾收集器使用的线程数可以通过参数 *-XX:ParallelGCThreads=NNN* 来设置，默认为系统CPU核数。

使用以下参数之一可以开启该GC算法：

>java -XX:+UseParallelGC com.mypackages.MyExecutableClass
java -XX:+UseParallelOldGC com.mypackages.MyExecutableClass
java -XX:+UseParallelGC -XX:+UseParallelOldGC com.mypackages.MyExecutableClass

Parallel GC适合多核机器，并且你的主要目标是提高吞吐量。更高的吞吐量得益于更高效的使用系统资源。

- GC期间，所有核并行清除垃圾，从而可以获得更短的停顿时间
- 在GC周期之间，不消耗任何系统资源

另外，因为所有的收集阶段仍然是不允许被打断的，这种垃圾收集器仍有可能导致长时间停顿的发生。如果你的主要目标是延迟的话，你应该考虑下一节介绍的CMS。

我们现在来看看在使用Parallel GC算法时的GC日志格式，看从中我们能获得哪些信息。同样我们截取两个日志片段，一个是Minor GC的，一个是Major GC的。

```
2015-05-26T14:27:40.915-0200: 116.115: [GC (Allocation Failure) [PSYoungGen: 2694440K->1305132K(2796544K)] 9556775K->8438926K(11185152K), 0.2406675 secs] [Times: user=1.77 sys=0.01, real=0.24 secs]
2015-05-26T14:27:41.155-0200: 116.356: [Full GC (Ergonomics) [PSYoungGen: 1305132K->0K(2796544K)] [ParOldGen: 7133794K->6597672K(8388608K)] 8438926K->6597672K(11185152K), [Metaspace: 6745K->6745K(1056768K)], 0.9158801 secs] [Times: user=4.49 sys=0.64, real=0.92 secs]
```

##Minor GC

>2015-05-26T14:27:40.915-0200<sup>1</sup>: 116.115<sup>2</sup>:[GC<sup>3</sup>(Allocation Failure<sup>4</sup>)[PSYoungGen<sup>5</sup>: 2694440K->1305132K<sup>6</sup>(2796544K)<sup>7</sup>]9556775K->8438926K<sup>8</sup>(11185152K)<sup>9</sup>, 0.2406675 secs<sup>10</sup>][Times: user=1.77 sys=0.01, real=0.24 secs]<sup>11</sup>

1. 2015-05-26T14:27:40.915-0200：GC开始时间
2. 116.115：GC开始时间，相对JVM启动时间，单位是秒
3. GC：区分是Minor GC还是Full GC，这里表示Minor
4. Allocation Failure：产生GC的原因，这里是因为无法在年轻代为某个数据结构分配空间导致触发GC
5. PSYoungGen：年轻代垃圾收集器名字：一个并行的，使用标记-拷贝算法，会产生STW的垃圾回收器
6. 2694440K->1305132K：GC前后年轻代使用情况
7. (2796544K)：年轻代总容量
8. 9556775K->8438926K：GC前后堆使用情况
9. (11185152K)：堆总容量
10. 0.2406675 secs：GC事件持续时间，单位是秒
11. [Times: user=1.77 sys=0.01, real=0.24 secs]：GC事件的时间开销，分三个类别：
 - user：整个过程GC耗费的全部CPU时间
 - sys：系统耗费时间，包括系统调用或者等待系统事件
 - real：应用程序因GC被停止的时间。对于Parallel GC来说，real time接近于(user time + sys time)/GC线程数，这里使用了8个线程。因为某些活动不能并行，所以这个值会稍稍大一点。

简单来说，GC之前，堆使用量是9,556,775K，其中年轻代使用量是2,694,440K，可知老生代使用量为6,862,335K。GC之后，年轻代的使用量减少了1,389,308K，而整个堆的使用量只减少1,117,849K，可知该过程中有271,459K对象从年轻代晋升到老生代。如图：

![ParallelGC-in-Young-Generation-Java.png](http://upload-images.jianshu.io/upload_images/5475750-9b558bc4e0bf54a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##Full GC

>2015-05-26T14:27:41.155-0200<sup>1</sup>:116.356<sup>2</sup>:[Full GC<sup>3</sup> (Ergonomics<sup>4</sup>)[PSYoungGen: 1305132K->0K(2796544K)]<sup>5</sup>[ParOldGen<sup>6</sup>:7133794K->6597672K<sup>7</sup>(8388608K)<sup>8</sup>] 8438926K->6597672K<sup>9</sup>(11185152K)<sup>10</sup>, [Metaspace: 6745K->6745K(1056768K)] <sup>11</sup>, 0.9158801 secs<sup>12</sup>, [Times: user=4.49 sys=0.64, real=0.92 secs]<sup>13</sup>

1. 2015-05-26T14:27:41.155-0200：GC开始时间
2. 116.356：GC开始时间，相对JVM启动时间，单位是秒
3. Full GC：表示这是Full GC，垃圾清理包括年轻代和老生代
4. Ergonomics：产生GC的原因，这里表示JVM的内部工效逻辑判断目前是进行垃圾回收的好时机
5. [PSYoungGen: 1305132K->0K(2796544K)]：跟上面的类似，在年轻代使用了一个并行的，使用标记-拷贝算法，会产生STW的PSYoungGen垃圾回收器，回收之后年轻代被清空，这也是一次Full GC的典型结果
6. ParOldGen：老生代使用的垃圾收集器名字：一个并行的，使用标记-清除-整理算法，会产生STW的垃圾回收器
7. 7133794K->6597672K：GC前后老生代使用情况
8. (8388608K)：老生代总容量
9. 8438926K->6597672K：GC前后堆使用情况
10. (11185152K)：堆总容量
11. [Metaspace: 6745K->6745K(1056768K)]：关于元空间的类似信息
12. 0.9158801 secs：GC事件持续时间，单位是秒
13. [Times: user=4.49 sys=0.64, real=0.92 secs]：GC事件的时间开销，分三个类别：
 - user：整个过程GC耗费的全部CPU时间
 - sys：系统耗费时间，包括系统调用或者等待系统事件
 - real：应用程序因GC被停止的时间。对于Parallel GC来说，real time接近于(user time + sys time)/GC线程数，这里使用了8个线程。因为某些活动不能并行，所以这个值会稍稍大一点。

同样，这个跟Minor GC的区别非常明显：除了年轻代，在这次GC过程中，老生代和元空间也被清除了。

用图来展示GC前和GC后内存使用变化如下：

![Java-ParallelGC-in-Old-Generation.png](http://upload-images.jianshu.io/upload_images/5475750-7128ee7050db476a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#CMS

该垃圾收集器组合的官方名字是“Mostly Concurrent Mark and Sweep Garbage Collector”。它在年轻代使用并行的，使用标记-拷贝算法，会产生STW的GC，在老生代使用并发的，使用标记-清除算法的GC。

这个垃圾收集器设计初衷是在老生代垃圾收集中避免长时间停顿。它通过两个手段来实现该目标。一：不对老生代内存进行整理操作，而是使用空闲列表来管理那些可回收再利用的内存空间。二：在标记-清除阶段，和应用程序并发的做掉绝大分工作。这意味着垃圾回收并不会显示停止应用程序线程来执行这些操作。尽管如此，垃圾回收器线程还是会跟应用程序线程抢占CPU时间。默认情况下，这种GC算法使用的线程数量等于机器CPU核数的1/4。

可用通过以下参数启用该垃圾回收器：
>java -XX:+UseConcMarkSweepGC com.mypackages.MyExecutableClass

如果应用程序运行在多核机器上，且你的主要目标是降低延迟，CMS会是一个不错的选择。降低单次GC停顿时间直接影响终端用户对应用程序的感受，会让用户感觉应用程序响应更快。由于在大部分时间里，总有一些CPU资源被GC使用而不是用来执行你的应用程序代码，所以对于计算密集型应用来说，CMS在吞吐量上不如Parallel GC。

和之前的GC算法一样，让我们再次通过查看包含一次Minor GC和一次Major GC的GC日志来看看该算法是如何在实际中应用的。

```
2015-05-26T16:23:07.219-0200: 64.322: [GC (Allocation Failure) 64.322: [ParNew: 613404K->68068K(613440K), 0.1020465 secs] 10885349K->10880154K(12514816K), 0.1021309 secs] [Times: user=0.78 sys=0.01, real=0.11 secs]
2015-05-26T16:23:07.321-0200: 64.425: [GC (CMS Initial Mark) [1 CMS-initial-mark: 10812086K(11901376K)] 10887844K(12514816K), 0.0001997 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-mark: 0.035/0.035 secs] [Times: user=0.07 sys=0.00, real=0.03 secs]
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-preclean: 0.016/0.016 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-abortable-preclean-start]
2015-05-26T16:23:08.446-0200: 65.550: [CMS-concurrent-abortable-preclean: 0.167/1.074 secs] [Times: user=0.20 sys=0.00, real=1.07 secs]
2015-05-26T16:23:08.447-0200: 65.550: [GC (CMS Final Remark) [YG occupancy: 387920 K (613440 K)]65.550: [Rescan (parallel) , 0.0085125 secs]65.559: [weak refs processing, 0.0000243 secs]65.559: [class unloading, 0.0013120 secs]65.560: [scrub symbol table, 0.0008345 secs]65.561: [scrub string table, 0.0001759 secs][1 CMS-remark: 10812086K(11901376K)] 11200006K(12514816K), 0.0110730 secs] [Times: user=0.06 sys=0.00, real=0.01 secs]
2015-05-26T16:23:08.458-0200: 65.561: [CMS-concurrent-sweep-start]
2015-05-26T16:23:08.485-0200: 65.588: [CMS-concurrent-sweep: 0.027/0.027 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
2015-05-26T16:23:08.485-0200: 65.589: [CMS-concurrent-reset-start]
2015-05-26T16:23:08.497-0200: 65.601: [CMS-concurrent-reset: 0.012/0.012 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
```

##Minor GC

>2015-05-26T16:23:07.219-0200<sup>1</sup>: 64.322<sup>2</sup>:[GC<sup>3</sup>(Allocation Failure<sup>4</sup>) 64.322: [ParNew<sup>5</sup>: 613404K->68068K<sup>6</sup>(613440K) <sup>7</sup>, 0.1020465 secs<sup>8</sup>] 10885349K->10880154K <sup>9</sup>(12514816K)<sup>10</sup>, 0.1021309 secs<sup>11</sup>][Times: user=0.78 sys=0.01, real=0.11 secs]<sup>12</sup>

1. 2015-05-26T16:23:07.219-0200：GC开始时间
2. 64.322：GC开始时间，相对JVM启动时间，单位是秒
3. GC：区分是Minor GC还是Full GC，这里表示Minor
4. Allocation Failure：产生GC的原因，这里是因为无法在年轻代为某个数据结构分配空间导致触发GC
5. ParNew：年轻代垃圾收集器名字：一个并行的，使用标记-拷贝算法，会产生STW的垃圾回收器，该垃圾收集器是用来跟老生代的CMS配合使用的
6. 613404K->68068K：GC前后年轻代使用情况
7. (613440K)：年轻代总容量
8. 0.1020465 secs：GC事件持续时间
9. 10885349K->10880154K：GC前后堆使用情况
10. (12514816K)：堆总容量
11. 0.1021309 secs：GC标记和拷贝年轻代里面活对象耗费的时间，这里面包括和老生代CMS通讯开销，对象晋升到老年代的开销，GC结束前清理工作的开销
12. [Times: user=0.78 sys=0.01, real=0.11 secs]：GC事件的时间开销，分三个类别：
 - user：整个过程GC耗费的全部CPU时间
 - sys：系统耗费时间，包括系统调用或者等待系统事件
 - real：应用程序因GC被停止的时间。对于Parallel GC来说，real time接近于(user time + sys time)/GC线程数，这里使用了8个线程。因为某些活动不能并行，所以这个值会稍稍大一点。

从上面可以看出，GC之前，堆使用量是10,885,349K，其中年轻代使用量是613,404K，可知老生代使用量为10,271,945K。GC之后，年轻代的使用量减少了545,336K，而整个堆的使用量只减少5,195K，可知该过程中有540,141K对象从年轻代晋升到老生代。如图：

![ParallelGC-in-Young-Generation-Java.png](http://upload-images.jianshu.io/upload_images/5475750-16e1d5bf3da6bc88.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##Full GC

当你已经开始熟悉垃圾收集器的日志格式的时候，这一节将介绍一个格式完全不同的日志格式。下面的这个日志输出包含了CMS日志的所有阶段，为了更好的解释清楚，我们按照阶段来逐个分析各个阶段的日志含义。
```
2015-05-26T16:23:07.321-0200: 64.425: [GC (CMS Initial Mark) [1 CMS-initial-mark: 10812086K(11901376K)] 10887844K(12514816K), 0.0001997 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-mark: 0.035/0.035 secs] [Times: user=0.07 sys=0.00, real=0.03 secs]
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-preclean: 0.016/0.016 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-abortable-preclean-start]
2015-05-26T16:23:08.446-0200: 65.550: [CMS-concurrent-abortable-preclean: 0.167/1.074 secs] [Times: user=0.20 sys=0.00, real=1.07 secs]
2015-05-26T16:23:08.447-0200: 65.550: [GC (CMS Final Remark) [YG occupancy: 387920 K (613440 K)]65.550: [Rescan (parallel) , 0.0085125 secs]65.559: [weak refs processing, 0.0000243 secs]65.559: [class unloading, 0.0013120 secs]65.560: [scrub symbol table, 0.0008345 secs]65.561: [scrub string table, 0.0001759 secs][1 CMS-remark: 10812086K(11901376K)] 11200006K(12514816K), 0.0110730 secs] [Times: user=0.06 sys=0.00, real=0.01 secs]
2015-05-26T16:23:08.458-0200: 65.561: [CMS-concurrent-sweep-start]
2015-05-26T16:23:08.485-0200: 65.588: [CMS-concurrent-sweep: 0.027/0.027 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
2015-05-26T16:23:08.485-0200: 65.589: [CMS-concurrent-reset-start]
2015-05-26T16:23:08.497-0200: 65.601: [CMS-concurrent-reset: 0.012/0.012 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
```

请记住一点：在真实世界里，在对老生代进行并发垃圾回收的同时，年轻代的GC随时可能发生。这个时候，Minor GC和Full GC的日志就会穿插的出现在GC日志文件里面。

**Phase 1: Initial Mark。** 这是CMS GC过程中两次STW中的一次。这个阶段的目标是标记出老生代里面符合条件的对象，这些对象或者是从GC roots直接指向的，或者被年轻代活着对象指向的。后者非常重要，因为老生代是单独回收的。

![cms-initial-mark.png](http://upload-images.jianshu.io/upload_images/5475750-c0b6af26ce9e9852.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>2015-05-26T16:23:07.321-0200: 64.42<sup>1</sup>: [GC (CMS Initial Mark<sup>2</sup>[1 CMS-initial-mark: 10812086K<sup>3</sup>(11901376K)<sup>4</sup>] 10887844K<sup>5</sup>(12514816K)<sup>6</sup>, 0.0001997 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]<sup>7</sup>

1. 2015-05-26T16:23:07.321-0200: 64.42：GC开始时间
2. CMS Initial Mark：该阶段的名字
3. 10812086K：当前老生代使用量
4. (11901376K)：老生代总容量
5. 10887844K：当前堆使用量
6. (12514816K)：堆总容量
7. [Times: user=0.00 sys=0.00, real=0.00 secs]：该阶段的耗时

**Phase 2: Concurrent Mark。** 在这个阶段，垃圾收集器从上一个阶段找到的所有根节点开始遍历整个老生代，标记所有活着的对象。这个阶段是和应用程序并发执行的，不会停止应用程序进程。这里需要注意的是，因为是并发执行，程序可能在标记过程中修改引用，所以并不是所有的活着都会被标记出。

![cms-concurrent-mark.png](http://upload-images.jianshu.io/upload_images/5475750-38b24e6702508b56.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如图所示，在标记的过程中，“Current obj”指向另一个对象的引用被删除了。

>2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-mark<sup>1</sup>: 035/0.035 secs<sup>2</sup>] [Times: user=0.07 sys=0.00, real=0.03 secs]<sup>3</sup>

1. CMS-concurrent-mark：该阶段的名字
2. 035/0.035 secs：该阶段消耗的时间，分别展示了clock时间和cpu时间
3. [Times: user=0.07 sys=0.00, real=0.03 secs]：并发阶段的时间字段参考意义不大

**Phase 3: Concurrent Preclean。** 这又是一个并发阶段，和应用程序并发执行的，不会停止应用程序进程。在前一个阶段和应用程序并发执行的过程中，一些引用可能被修改，当修改发生时，JVM都会把包含该修改对象的堆区域(又被称为卡片)标记为“脏数据”（又被称为卡片标记）

![cms-concurrent-preclean.png](http://upload-images.jianshu.io/upload_images/5475750-920e3e9c52be94cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这个阶段，从脏数据到达的对象也会被标记成活对象，标记完成之后，卡片脏数据标记会被清除。

![cms-concurrent-preclean-2.png](http://upload-images.jianshu.io/upload_images/5475750-6c9d0cd9d4372784.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该阶段还会为Final Remark阶段做一些必要的整理记录和准备的工作。

>2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-preclean<sup>1</sup>: 0.016/0.016 secs<sup>2</sup>] [Times: user=0.02 sys=0.00, real=0.02 secs]<sup>3</sup>

1. CMS-concurrent-preclean：该阶段的名字
2. 0.016/0.016 secs：该阶段消耗的时间，分别展示了clock时间和cpu时间
3. [Times: user=0.02 sys=0.00, real=0.02 secs]：并发阶段的时间字段参考意义不大

**Phase 4: Concurrent Abortable Preclean。** 还是一个并发阶段，不会停止应用程序进程。这个阶段尝试着去承担STW的Final Remark阶段足够多的工作。这个阶段持续的时间依赖很多的因素，由于这个阶段是重复的做相同的事情直到某些中止条件被满足为止（中止条件包括：重复的次数、多少量的工作、累计执行时间等等）。

>2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-abortable-preclean-start]
2015-05-26T16:23:08.446-0200: 65.550: [CMS-concurrent-abortable-preclean<sup>1</sup>: 0.167/1.074 secs<sup>2</sup>] [Times: user=0.20 sys=0.00, real=1.07 secs]<sup>3</sup>

1. CMS-concurrent-abortable-preclean：该阶段的名字
2. 0.167/1.074 secs：该阶段消耗的时间，分别展示了clock时间和cpu时间。这里会发现user time要比clock time小得多。一般我们都是看到real time要比user time小，这是因为有些工作被并发执行了，所以elapsed clock time要比使用的CPU time少。这里我们仅有少量的工作，0.167秒的CPU时间，垃圾回收器线程花费了很多时间在等待上。也就是说，他们在尽量推迟STW的到来，默认情况下，这个阶段可能持续5秒
3. [Times: user=0.20 sys=0.00, real=1.07 secs]：并发阶段的时间字段参考意义不大

这个阶段可能对即将到来的STW阶段影响非常大，有很多重要的配置参数和失败方式。

**Phase 5: Final Remark。** 这是第二个也是最后一个STW阶段。这个阶段的目标就是最终标记老生代的所有活着的对象。因为之前的preclean是并发阶段，他们可能无法赶上应用程序的修改速度。所以需要一个STW暂停来完成整个标记过程。

通常CMS试着在年轻代尽可能空的情况下执行最终标记，试图减少多个STW阶段一个接着一个的产生的可能性。

这个阶段的GC日志比前面阶段要复杂一些。

>2015-05-26T16:23:08.447-0200: 65.550<sup>1</sup>: [GC (CMS Final Remark<sup>2</sup>) [YG occupancy: 387920 K (613440 K)<sup>3</sup>]65.550: [Rescan (parallel) , 0.0085125 secs]<sup>4</sup>65.559: [weak refs processing, 0.0000243 secs]65.559<sup>5</sup>: [class unloading, 0.0013120 secs]65.560<sup>6</sup>: [scrub string table, 0.0001759 secs<sup>7</sup>][1 CMS-remark: 10812086K(11901376K)<sup>8</sup>] 11200006K(12514816K) <sup>9</sup>, 0.0110730 secs<sup>10</sup>] [Times: user=0.06 sys=0.00, real=0.01 secs]<sup>11</sup>

1. 2015-05-26T16:23:08.447-0200: 65.550：GC开始时间
2. CMS Final Remark：该阶段的名字
3. YG occupancy: 387920 K (613440 K)：当前年轻代的使用量和总容量
4. [Rescan (parallel) , 0.0085125 secs]：应用程序停止过程中标记所有活着对象所耗的时间
5. [weak refs processing, 0.0000243 secs]65.559：第一个子阶段：处理弱引用所耗时间
6. [class unloading, 0.0013120 secs]65.560：第二个子阶段：卸载不用类所耗时间
7. [scrub string table, 0.0001759 secs：最后一个子阶段：清除字符表(存储类级别元数据)和字符串表(存储驻留字符串)所耗时间，停顿的clock time也被包括在里面
8. 10812086K(11901376K)：标记完之后老生代使用量和总容量
9. 11200006K(12514816K)：标记完之后堆使用量和总容量
10. 0.0110730 secs：该阶段所耗时间
11. [Times: user=0.06 sys=0.00, real=0.01 secs]：GC事件的时间开销，分三个类别

在经历五个标记阶段之后，老生代所有活着的对象都被标记完毕，现在垃圾收集器就准备通过清除老生代来回收所有不使用对象的存储空间。

**Phase 6: Concurrent Sweep。** 该阶段也是跟应用程序并发执行，不需要STW停顿。这个阶段的目标是清除那些不再使用的对象，并回收他们占用的内存空间以备后续使用。

![cms-concurrent-sweep.png](http://upload-images.jianshu.io/upload_images/5475750-ff22c3b5fdd470d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>2015-05-26T16:23:08.458-0200: 65.561: [CMS-concurrent-sweep-start] 2015-05-26T16:23:08.485-0200: 65.588: [CMS-concurrent-sweep<sup>1</sup>: 0.027/0.027 secs<sup>2</sup>] [Times: user=0.03 sys=0.00, real=0.03 secs] <sup>3</sup>

1. CMS-concurrent-sweep：该阶段名字
2. 0.027/0.027 secs：该阶段消耗的时间，分别展示了clock时间和cpu时间。
3. [Times: user=0.03 sys=0.00, real=0.03 secs]：并发阶段的时间字段参考意义不大


**Phase 7: Concurrent Reset。** 并发执行阶段，重置CMS算法的内部数据结构，为下一次执行做准备。

>2015-05-26T16:23:08.485-0200: 65.589: [CMS-concurrent-reset-start] 2015-05-26T16:23:08.497-0200: 65.601: [CMS-concurrent-reset<sup>1</sup>: 0.012/0.012 secs<sup>2</sup>] [Times: user=0.01 sys=0.00, real=0.01 secs]<sup>3</sup>

1. CMS-concurrent-reset：该阶段名字
2. 0.012/0.012 secs：该阶段消耗的时间，分别展示了clock时间和cpu时间。
3. [Times: user=0.01 sys=0.00, real=0.01 secs]：并发阶段的时间字段参考意义不大

总的来讲，CMS垃圾回收器通过把大量的工作放到不需要应用程序停顿的并发线程里面去做掉，大大减少了应用程序的停顿时间。但是，它也有自己的缺点，最明显的就是老生代内存碎片问题和某些情况下停顿时间的不可预测性，特别是在大内存堆的情况下

#G1

G1的一个重要的设计目标就是由GC导致的STW停顿的持续时间可预测，持续时间长短可配置。实际上，G1是一种软实时垃圾收集器，这意味着你可以给它设定具体的性能指标。你可以要求在任意给定Y毫秒范围内STW停顿持续时间不超过X毫秒，比如：在任意一秒钟内不超过5毫秒。G1垃圾收集器会尽自己最大努力尽可能满足这个目标设定(这个目标不一定能达成，否则就是硬实时了)。

为了达成这个设计目标，G1提供了几个新思路。首先，堆不是被划分成连续的年轻代和老生代，而是被划分成一些(典型是2048个)更小的堆区域块，这些区域块用来存储对象。任意一个区域块可能是一个伊甸区，也可能是一个存活区，还可能是一个老年区。逻辑上把所有的伊甸区和存活区合起来称作年轻代，所有的老年区合起来称作老生代。

![g1-01.png](http://upload-images.jianshu.io/upload_images/5475750-fa1c3d35370179a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样的话，GC可以采用增量的方式，每次回收一些区域块，而不是对整个堆进行垃圾回收。在每次停顿时，会对所有年轻代区域进行垃圾回收，某些老生代区域可能也被包含进来一起。

![g1-02.png](http://upload-images.jianshu.io/upload_images/5475750-38344f084096e0e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其次，G1在并发阶段它会计算每个区域块包含多少活对象。这个用来构建单次收集集合：包含最多垃圾的区域块最先进行垃圾收集。这也是它的名字由来：garbage-first。

启用G1垃圾收集器：
>java -XX:+UseG1GC com.mypackages.MyExecutableClass

##Evacuation Pause：全年轻代模式

在应用程序生命周期初期，G1还未执行并发标记阶段，对堆内存无任何额外信息。所以它最开始是运行于全年轻代模式。当整个年轻代被对象装满，应用程序线程被停止了，年轻代的活对象被拷贝到存活区，或者被拷贝到任何空闲区域块，这些空闲区域块也就变成了存活区块。


这个拷贝过程被称作Evacuation，它的工作方式跟之前介绍的年轻代垃圾收集器类似。整个evacuation pause阶段的日志非常大，为了简单化，我们把和全年轻代模式evacuation pause无关的日志省略掉了。在对并行阶段进行详细介绍之后我们再来解释这些被省略的日志。另外，考虑到日志记录的大小，这里把并行阶段和其他阶段的详细日志都剥离到独立的部分。

>0.134: [GC pause (G1 Evacuation Pause) (young), 0.0144119 secs]<sup>1</sup>
    [Parallel Time: 13.9 ms, GC Workers: 8]<sup>2</sup>
        …<sup>3</sup>
    [Code Root Fixup: 0.0 ms]<sup>4</sup>
    [Code Root Purge: 0.0 ms]<sup>5</sup>
    [Clear CT: 0.1 ms]
    [Other: 0.4 ms]<sup>6</sup>
        …<sup>7</sup>
    [Eden: 24.0M(24.0M)->0.0B(13.0M) <sup>8</sup>Survivors: 0.0B->3072.0K <sup>9</sup>Heap: 24.0M(256.0M)->21.9M(256.0M)]<sup>10</sup>
     [Times: user=0.04 sys=0.04, real=0.02 secs] <sup>11</sup>

1. 0.134: [GC pause (G1 Evacuation Pause) (young), 0.0144119 secs]：evacuation pause开始于JVM启动之后0.134秒，持续了0.0144119秒
2. [Parallel Time: 13.9 ms, GC Workers: 8]：下列活动用8个并发工作线程执行，总共花了13.9ms(clock time)
3. 这里为了简化，省略了具体的活动内容，具体见后面章节
4. [Code Root Fixup: 0.0 ms]：释放用来管理并行活动的数据结构，这个值一般都接近于0。这个操作是顺序的
5. [Code Root Purge: 0.0 ms]：清除更多的数据结构，这个操作也非常快，但不一定是0。这个操作也是顺序的
6. [Other: 0.4 ms]：其他各种活动耗时，很多都是并行的
7. 具体见后面章节
8. [Eden: 24.0M(24.0M)->0.0B(13.0M)：执行前后伊甸区的使用量和总容量
9. Survivors: 0.0B->3072.0K：执行前后存活区的使用量
10. Heap: 24.0M(256.0M)->21.9M(256.0M)]：执行前后堆的使用量和总容量
11. [Times: user=0.04 sys=0.04, real=0.02 secs]：GC事件的时间开销，分三个类别
 - user：整个过程GC耗费的全部CPU时间
 - sys：系统耗费时间，包括系统调用或者等待系统事件
 - real：应用程序因GC被停止的时间。由于GC的活动是并行的，real time接近于(user time + sys time)/GC线程数，这里使用了8个线程。因为某些活动不能并行，所以这个值会稍稍大一点。

多个专属GC工作线程负责执行大部分耗时操作，具体内容如下：
>[Parallel Time: 13.9 ms, GC Workers: 8]<sup>1</sup>
     [GC Worker Start (ms)<sup>2</sup>: Min: 134.0, Avg: 134.1, Max: 134.1, Diff: 0.1]
    [Ext Root Scanning (ms)<sup>3</sup>: Min: 0.1, Avg: 0.2, Max: 0.3, Diff: 0.2, Sum: 1.2]
    [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
        [Processed Buffers: Min: 0, Avg: 0.0, Max: 0, Diff: 0, Sum: 0]
    [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
    [Code Root Scanning (ms)<sup>4</sup>: Min: 0.0, Avg: 0.0, Max: 0.2, Diff: 0.2, Sum: 0.2]
    [Object Copy (ms)<sup>5</sup>: Min: 10.8, Avg: 12.1, Max: 12.6, Diff: 1.9, Sum: 96.5]
    [Termination (ms)<sup>6</sup>: Min: 0.8, Avg: 1.5, Max: 2.8, Diff: 1.9, Sum: 12.2]
        [Termination Attempts<sup>7</sup>: Min: 173, Avg: 293.2, Max: 362, Diff: 189, Sum: 2346]
    [GC Worker Other (ms)<sup>8</sup>: Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
    GC Worker Total (ms)<sup>9</sup>: Min: 13.7, Avg: 13.8, Max: 13.8, Diff: 0.1, Sum: 110.2]
    [GC Worker End (ms)<sup>10</sup>: Min: 147.8, Avg: 147.8, Max: 147.8, Diff: 0.0]

1. [Parallel Time: 13.9 ms, GC Workers: 8]：下列活动用8个并发工作线程执行，总共花了13.9ms(clock time)
2. [GC Worker Start (ms)：工作线程开始执行操作的时间，应该和停顿的开始时间一致。如果最小和最大差距太大，可能意味着使用了太多线程，或者机器上有其他进程和JVM的GC进程争抢CPU资源
3. [Ext Root Scanning (ms)：扫描非堆的GC roots所耗时间，包括类加载器，JNI引用，JVM系统根等，除了sum是cpu time，其他都是clock time
4. [Code Root Scanning (ms)：扫描来自实际代码的GC roots，比如本地变量等
5. [Object Copy (ms)：从收集区域块拷贝活对象所耗时间
6. [Termination (ms)：工作线程确信所有工作已经做完，可以安全停止所耗时间
7. [Termination Attempts：工作线程尝试中止的次数，如果还有未完成的工作，就是一次失败的中止尝试
8. [GC Worker Other (ms)：其他各种活动耗时
9. GC Worker Total (ms)：GC工作线程所耗时间总和
10. [GC Worker End (ms)：工作线程完成工作的时间，通常应该时间差不多，否则意味着太多线程堵塞或者有其他进程争抢CPU资源

另外，在该阶段还有一些其他活动在执行。这里只介绍部分内容，其他的放在后面章节。

>[Other: 0.4 ms]<sup>1</sup>
    [Choose CSet: 0.0 ms]
    [Ref Proc: 0.2 ms]<sup>2</sup>
    [Ref Enq: 0.0 ms]<sup>3</sup>
    [Redirty Cards: 0.1 ms]
    [Humongous Register: 0.0 ms]
    [Humongous Reclaim: 0.0 ms]
    [Free CSet: 0.0 ms]<sup>4</sup>

1. [Other: 0.4 ms]：其他各种活动耗时，很多都是并行的
2. [Ref Proc: 0.2 ms]：处理非强引用耗时：决定是清除它们还是无视它们
3. [Ref Enq: 0.0 ms]：待处理的非强引用入对应队列所耗时间
4. [Free CSet: 0.0 ms]：返回收集集合中已经释放区域块所耗时间

##并发标记

G1垃圾收集器借鉴了CMS的很多理念。所以在继续之前确保你对CMS的这些概念有很好的理解。尽管G1和CMS有很多不同，但是并发标记阶段的目的非常类似。G1并发标记使用SATB(Snapshot-At-The-Beginning)的方法，对所有在标记开始那一刻活着的对象进行标记，即使后来它变成垃圾。根据这些信息可以构建每个区域块的活对象状态信息，这样就可以很高效的确定收集集合。

这个信息也用于老生代的GC。如果通过标记发现某个区域块只包含垃圾，或者在对老生代(既包含垃圾也包含活对象)做evacuation pause的STW期间，并发标记可以完全并发执行。

当堆内存使用空间足够大就会触发并发标记。默认是45%，这个值可以通过*InitiatingHeapOccupancyPercent*进行调整。和CMS类似，G1中的并发标记也包含很多阶段，这些阶段有些是完全并发，有些需要停止应用程序。

**Phase 1: Initial Mark。** 标记所有可以从GC roots直达的对象。在CMS里，这个阶段需要单独的STW，但是在G1里，它只是evacuation pause的附带停顿，所以它的开销很小。在日志里面的表现就是在evacuation pause日志首行带有Initial Mark字样。

>1.631: [GC pause (G1 Evacuation Pause) (young) (initial-mark), 0.0062656 secs]

**Phase 2: Root Region Scan。** 标记所有可以从所谓的根区域块可达的活对象。例如，那些不是空。因为在并发标记过程中移动对象会有问题，这个过程必须在下一次evacuation pause开始之前完成。如果它要提前开始，就需要提前中止root region scan，并等到它结束。在目前的实现中，根区域块是存活区：他们是年轻代里面的对象，必定在下一次evacuation pause中被回收。

>1.362: [GC concurrent-root-region-scan-start]
1.364: [GC concurrent-root-region-scan-end, 0.0028513 secs]

**Phase 3：Concurrent Mark。** 跟CMS类似：遍历对象图，在一个特殊的位图里面标记访问的对象。为了确保满足SATB的要求，G1要求应用程序对对象图的并发更新要留下用来标记的旧引用。这是通过Pre-Write barriers实现的(注意不要跟下文提到的Post-Write barriers以及多线程编程里面的memory barriers混淆概念)：在G1并发标记执行过程中，对任何对象的写入，把旧引用存储到日志缓存里面，这些内容到时候会被并发标记线程处理的。

>1.364: [GC concurrent-mark-start]
1.645: [GC concurrent-mark-end, 0.2803470 secs]

**Phase 4：Remark。** 这个跟CMS的最终标记类似，需要STW停顿，用来终结整个标记阶段。G1停止应用程序，不让它们继续产生并发更新日志，然后处理完剩下的日志，标记所有在并发标记开始时刻是活着的所有未标记对象。该阶段还会处理一些额外的清除操作，比如引用处理（请参照前面的evacuation pause日志）或者类卸载。

>1.645: [GC remark 1.645: [Finalize Marking, 0.0009461 secs] 1.646: [GC ref-proc, 0.0000417 secs] 1.646: [Unloading, 0.0011301 secs], 0.0074056 secs]
[Times: user=0.01 sys=0.00, real=0.01 secs]

**Phase 5：Cleanup。** 并发标记的最终阶段，为即将到来的evacuation pause做准备：计算堆区域块所有活对象，根据期望的GC效益来排序这些区域块。为了下次并发标记，还需要做一些清理工作来维护内部数据结构的状态。

最后还有一件重要的事是，没有包含活对象的区域块会在这个阶段被回收。这个阶段有些部分是并发，比如空区域块回收，大部分的活性计算，但也需要一个短暂的STW停顿来结束整个过程，以防止应用程序干预。这个停顿的日志如下：

>1.652: [GC cleanup 1213M->1213M(1885M), 0.0030492 secs]
[Times: user=0.01 sys=0.00, real=0.00 secs]

当出现某些堆区域块只包含垃圾的时候，日志格式稍微有点不同：

>1.872: [GC cleanup 1357M->173M(1996M), 0.0015664 secs]
[Times: user=0.01 sys=0.00, real=0.01 secs]
1.874: [GC concurrent-cleanup-start]
1.876: [GC concurrent-cleanup-end, 0.0014846 secs]

##Evacuation Pause：混合模式

如果并发清除操作能够释放掉老生代的整个区域块那是最好不过了，但是经常不是这样。在并发标记结束之后，G1会在混合模式下工作，不仅仅是从年轻代区域收集垃圾，也会把一些老生代区域捎带放到收集集合中。

混合模式的evacuation pause一般并不会紧跟着并发标记阶段。是否要进行evacuation pause，受到很多规则和直觉影响。比如，如果在并发标记阶段已经释放了大部分老生代区域，就没必要立马进行evacuation pause了。

因此有可能在并发标记结束和混合模式evacuation pause之间，有很多次全年轻代模式evacuation pause。

被加入到收集集合中的老生代区域块个数，加入的顺序也是基于很多规则的。包括应用的软实时性能目标，并发标记过程中收集到的活性数据和gc效益数据，JVM参数配置等。混合模式的evacuation pause过程跟全年轻代模式的evacuation pause大部分一样，这里我们主要介绍一下remembered sets概念。

remembered sets使得对不同堆区域进行独立GC成为可能。比如，如果对区域A,B,C进行GC，我们需要知道是否有从区域D,E来的对象引用，这对影响到对一个对象是否是活着的判断。但是遍历整个堆对象图非常耗时，这也不符合增量回收的初衷，因此这里需要进行优化。类似于其他垃圾收集算法使用卡片表来实现对年轻代的独立回收，在G1这里就是remembered sets。

如下图所示，每一个区域块都有一个remembered set，里面记录了所有来自外部的引用，这些引用将被认为是GC roots的补充。注意在并发标记过程中被判定为垃圾的老生代区域的对象会被无视，即使有来自外部的引用指向这些对象，这个时候这些引用方也是垃圾。

![g1-03.png](http://upload-images.jianshu.io/upload_images/5475750-2d749476bab78fe6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来就跟其他垃圾收集器一样：多个并行GC线程找出来哪些对象是活的，哪些是垃圾。

![g1-04.png](http://upload-images.jianshu.io/upload_images/5475750-aad020a908a4e7de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后，活对象被拷贝到存活区，如果需要的话会创建一个新的存活区。这样空区域被释放了，可以用来存储新对象了。

![g1-05.png](http://upload-images.jianshu.io/upload_images/5475750-c8f7ce320ef1f6a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了维护这个remembered set，在应用程序运行过程中，当需要对某个字段进行写操作就会触发一个Post-Write Barrier。如果最终的引用是跨区的，比如从一个区域指向另外一个区域，这个时候在目标区域的remembered set里面就会相应追加一条记录。为了减少Write Barrier的开销，把这个卡片放入到remembered set里面的操作是异步的，并采取了一些其他的优化措施。基本来说分几个步骤：Write Barrier把脏卡片信息放入到一个本地缓存，一个专属GC线程读取它再把这个信息告诉给被引用区域的remembered set。

混合模式的日志里面会输出它自己独有的一些信息。

>[Update RS (ms)<sup>1</sup>: Min: 0.7, Avg: 0.8, Max: 0.9, Diff: 0.2, Sum: 6.1]
[Processed Buffers<sup>2</sup>: Min: 0, Avg: 2.2, Max: 5, Diff: 5, Sum: 18]
[Scan RS (ms)<sup>3</sup>: Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.8]
[Clear CT: 0.2 ms]<sup>4</sup>
[Redirty Cards: 0.1 ms]<sup>5</sup>

1. [Update RS (ms)：因为对RS的操作是并发的，我们必须确保在实际收集开始之前要处理完缓存中的卡片。如果这个数字很高的话，说明并发GC线程无法处理这个负荷，这可能是因为大量的字段修改导致也可能是因为CPU资源不够导致
2. [Processed Buffers：每个工作线程处理本地缓存次数
3. [Scan RS (ms)：扫描RS里面引用耗时
4. [Clear CT: 0.2 ms]：清除卡片表中卡片耗时，就是简单去除字段的脏数据标签
5. [Redirty Cards: 0.1 ms]：在卡片表中的适当位置标记脏数据耗时，这个位置是有GC对堆的修改决定的，比如在对引用进行入队列时

##总结

这节详细全面介绍了G1是如何工作的。当然还是有些实现细节是简要带过的，比如如何处理humongous objects，整体来讲，G1代表了HotSpot里面商用垃圾收集器的最前端技术成果。随着Java的版本的不断发布，对G1的优化也一直在进行。

从中可以看出，G1解决了CMS面临的很多问题，从停顿的可预见性到堆内存碎片。假如一个应用不受限于CPU利用率，但是对每个操作的延迟非常敏感，对HotSpot用户来说，G1就是最佳选择了，特别是当运行在最新版Java上的应用。但是，降低延迟并不是没有代价的：因为有额外的write barriers和更多活跃的后台线程，G1的吞吐量开销要更大一些。所以，如果应用程序更看重吞吐量或者已经耗尽了CPU资源，不太在意每次停顿时长，CMS或者并行算法的垃圾收集器会更加适合。

要想选择合适的GC算法和参数配置，唯一可行的方法就是不停的试错。但是我们会在下章给出一些基本的指导规则。

*注意：G1可能成为Java 9的默认GC：http://openjdk.java.net/jeps/248 *


#Shenandoah

我们已经介绍了HotSpot中所有可用的商用GC算法。还有一个正在开发中的，被称作“Ultra-Low-Pause-Time”垃圾收集器。它是专门为大型多核，大内存堆的服务器而设计的，其目标是在堆内存容量在100GB+的时候，停顿能控制在10ms以内。当然这个同样是以降低应用程序吞吐量为代价的：设计者的目标是用不超过10%的性能损失换取零GC停顿。

在该算法可用于生产环境之前，我们这里就不深入介绍具体的算法实现了，但是它仍然是基于这里讲的很多概念的，比如并发标记，增量回收等。当然它也有很多不同的实现，它不对堆内存进行分代，只有一个空间。对的，它不是分代垃圾回收器。这样它就没有card tables和remembered sets的概念。它也使用forwarding pointers和Brooks style read barrier来实现并发拷贝活对象，从而降低停顿的次数和持续时间。

更多关于Shenandoah算法进展可以关注这个[博客](https://rkennke.wordpress.com/)
