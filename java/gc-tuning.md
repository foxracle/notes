GC调优

>翻译原文 => [plumbr Java GC handbook](https://plumbr.eu/handbook/gc-tuning)

前文参见：
>[Java垃圾回收手册(一)：初识垃圾回收](http://www.jianshu.com/p/80fd72f48267)
>[Java垃圾回收手册(二)：Java中的垃圾回收](http://www.jianshu.com/p/4d8be1f3ad51)
>[Java垃圾回收手册(三)：垃圾回收算法基础](http://www.jianshu.com/p/c18526b589d2)
>[Java垃圾回收手册(四)：垃圾回收算法具体实现](http://www.jianshu.com/p/74dd0ffd4386)

GC调优和其他性能调优没有什么区别。一般很容易掉入调优陷阱：无头绪的随机调整200个GC相关的JVM参数或者随机修改程序代码。其实，遵循一个简单的流程会确保你一直在向正确的目标接近，并同时对进展了然于心：

1. 列出性能目标
2. 测试
3. 衡量结果
4. 把结果和目标进行比较
5. 如果没有达到目标，做出修改，然后重新进行测试

因此，对于GC调优来说，第一步我们需要设定一个清晰的性能目标。对于性能监控和管理来说，一般把目标分成三类：

- 延迟（Latency）
- 吞吐量（Throughput）
- 容量（Capacity）

简单介绍基本概念之后，我们再看看如何在GC调优中使用这些目标。如果你已经对延迟，吞吐量和容量的概念非常了解，你可以决定略过下一章节。

#核心概念

我们先来观察一个工厂的装配流水线的生产流程。流水线通过半成品来组装自行车，自行车的组装是线性的。通过观察，我们发现从第一个车架零件进入流水线到组装好的自行车从另一头离开流水线总共需要4小时。

继续观察我们发现，每一分钟就有一辆自行车离开流水线，一天24小时，日复一日。不考虑维护窗口期，我们认为任何一小时，这条流水线能组装60辆自行车。

有了这样两个衡量指标，我们从延迟和吞吐量两个方面掌握了当前流水线的准确性能信息：

- 装配线的延迟：4小时
- 装配线的吞吐量：60辆/时

注意，工作的延迟用时间单位进行度量，系统的吞吐量用单位时间内完成操作数进行度量。这个例子的时间单位是小时，操作是组装好的自行车。

有了明确的关于延迟和吞吐量的定义，我们可以尝试在这条装配线进行性能调优。比如由于自行车订单翻倍，之前是每天60*24=1440已不再能满足客户需求，需要对装配线进行调优以提高生产效率。如果不对延迟进行调整，最简单的办法就是弄两条一模一样的装配线，即扩大容量，就可以满足新的需求。当然我们也可以把延迟降到2小时，这样当前的装配线就能满足新的需求。这里面有一个很重要的概念，对于调优我们经常会有两个方案可选，或者升级硬件或者花时间优化代码。

#Latency（延迟）

GC的延迟目标一般都来自于延迟需求。而延迟需求的描述方式一般的形式如下：

- 所有用户交易必须在10秒内得到响应
- 90%的订单付款操作必须在3秒以内处理完成
- 推荐商品必须在 100 ms 内展示到用户面前

面对这类性能指标时, 需要确保在交易过程中, GC暂停不能占用太多时间，否则就满足不了性能需求。“太多” 需要视具体应用而定, 还要考虑到其他导致延迟的因素, 比如外部数据源的交互时间, 锁竞争, 以及其他的安全点等等。

假设性能需求为: 90%的交易要在1000ms以内完成，每次交易最长不能超过10秒。根据延迟要求, 我们假设GC暂停时间对延迟的贡献不能超过10%。也就是说90%的GC暂停必须在100ms内结束, 也不能有超过1000ms的GC暂停。为简单起见, 我们忽略在同一次交易过程中发生多次GC停顿的可能性。

对需求进行形式化之后,下一步就是检查暂停时间。有许多工具可以使用，在本节中我们通过查看GC日志, 检查一下GC暂停的时间。相关的信息散落在不同的日志片段中, 看下面的数据:
`2015-06-04T13:34:16.974-0200: 2.578: [Full GC (Ergonomics) [PSYoungGen: 93677K->70109K(254976K)] [ParOldGen: 499597K->511230K(761856K)] 593275K->581339K(1016832K), [Metaspace: 2936K->2936K(1056768K)], 0.0713174 secs] [Times: user=0.21 sys=0.02, real=0.07 secs`
这表示一次GC暂停, 在 2015-06-04T13:34:16 这个时刻触发. 对应于JVM启动之后的2,578 ms。

此事件将应用线程暂停了0.0713174秒。虽然花费的总时间为210ms, 但因为是多核CPU机器, 所以最重要的数字是应用线程被暂停的总时间, 这里使用的是并行GC, 所以暂停时间大约为70ms。这次GC的暂停时间小于100ms的阈值，满足需求。

继续从所有GC日志中提取出暂停相关的数据, 汇总之后就可以得知是否GC是否满足设定的延迟需求。

#Throughput（吞吐量）

吞吐量和延迟指标有很大区别。当然两者都是根据一般吞吐量需求而得出的。一般吞吐量需求类似这样:

- 解决方案每天必须处理 100万个订单
- 解决方案必须支持1000个登录用户,同时在5-10秒内执行某个操作: A、B或C
- 每周对所有客户进行统计, 时间不能超过6小时，时间窗口为每周日晚12点到次日6点之间。

可以看出,吞吐量需求不是针对单个操作的, 而是在给定的时间内, 系统必须完成多少个操作。和延迟需求类似, GC调优也需要确定GC行为所消耗的总时间。每个系统能接受的时间不同, 一般来说, GC占用的总时间比不能超过 10%。

现在假设需求为: 每分钟处理 1000 笔交易。同时, 每分钟GC暂停的总时间不能超过6秒(即10%)。

有了正式的需求, 下一步就是获取相关的信息。依然是从GC日志中提取数据, 可以看到类似这样的信息:
`2015-06-04T13:34:16.974-0200: 2.578: [Full GC (Ergonomics) [PSYoungGen: 93677K->70109K(254976K)] [ParOldGen: 499597K->511230K(761856K)] 593275K->581339K(1016832K), [Metaspace: 2936K->2936K(1056768K)], 0.0713174 secs] [Times: user=0.21 sys=0.02, real=0.07 secs`
此时我们对 用户耗时(user)和系统耗时(sys)感兴趣, 而不关心实际耗时(real)。在这里, 我们关心的时间为 0.23s(user + sys = 0.21 + 0.02 s), 这段时间内, GC暂停占用了 cpu 资源。 重要的是, 系统运行在多核机器上, 转换为实际的停顿时间(stop-the-world)为 0.0713174秒, 下面的计算会用到这个数字。

提取出有用的信息后, 剩下要做的就是统计每分钟内GC暂停的总时间。看看是否满足需求: 每分钟内总的暂停时间不得超过6000毫秒(6秒)。

#Capacity（系统容量）

系统容量需求,是在达成吞吐量和延迟指标的情况下,对硬件环境的额外约束。这类需求大多是用计算资源或者直接用预算来进行描述。例如:

- 系统必须能部署到小于512 MB内存的Android设备上
- 系统必须部署在Amazon EC2实例上, 配置不得超过 c3.xlarge(4核8GB)。
- 每月的 Amazon EC2 账单不得超过 $12,000

因此, 在满足了延迟和吞吐量的需求之后，系统容量也需要纳入考虑之中了。可以说, 假若有无限的计算资源可供挥霍, 那么任何 延迟和吞吐量指标 都不成问题, 但现实情况是, 预算(budget)和其他约束限制了可用的资源。

#相关示例

介绍完性能调优的三个维度后, 我们来进行实际的操作以达成GC性能指标。

请看下面的代码:
```
//imports skipped for brevity
public class Producer implements Runnable {

  private static ScheduledExecutorService executorService = Executors.newScheduledThreadPool(2);

  private Deque<byte[]> deque;
  private int objectSize;
  private int queueSize;

  public Producer(int objectSize, int ttl) {
    this.deque = new ArrayDeque<byte[]>();
    this.objectSize = objectSize;
    this.queueSize = ttl * 1000;
  }

  @Override
  public void run() {
    for (int i = 0; i < 100; i++) { deque.add(new byte[objectSize]); if (deque.size() > queueSize) {
        deque.poll();
      }
    }
  }

  public static void main(String[] args) throws InterruptedException {
    executorService.scheduleAtFixedRate(new Producer(200 * 1024 * 1024 / 1000, 5), 0, 100, TimeUnit.MILLISECONDS);
    executorService.scheduleAtFixedRate(new Producer(50 * 1024 * 1024 / 1000, 120), 0, 100, TimeUnit.MILLISECONDS);
    TimeUnit.MINUTES.sleep(10);
    executorService.shutdownNow();
  }
}
```
这段程序代码, 每 100毫秒 提交两个作业来。每个作业都模拟特定的生命周期: 创建对象, 然后在预定的时间释放, 接着就不管了, 由GC来自动回收占用的内存。

在运行这个示例程序时，通过以下JVM参数打开GC日志记录:
`-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps`

在日志文件中可以看到GC的行为, 类似下面这样:
```
2015-06-04T13:34:16.119-0200: 1.723: [GC (Allocation Failure) [PSYoungGen: 114016K->73191K(234496K)] 421540K->421269K(745984K), 0.0858176 secs] [Times: user=0.04 sys=0.06, real=0.09 secs] 
2015-06-04T13:34:16.738-0200: 2.342: [GC (Allocation Failure) [PSYoungGen: 234462K->93677K(254976K)] 582540K->593275K(766464K), 0.2357086 secs] [Times: user=0.11 sys=0.14, real=0.24 secs] 
2015-06-04T13:34:16.974-0200: 2.578: [Full GC (Ergonomics) [PSYoungGen: 93677K->70109K(254976K)] [ParOldGen: 499597K->511230K(761856K)] 593275K->581339K(1016832K), [Metaspace: 2936K->2936K(1056768K)], 0.0713174 secs] [Times: user=0.21 sys=0.02, real=0.07 secs]
```
基于日志中的信息, 可以通过三个优化目标来提升性能:

1. 确保最坏情况下,GC暂停时间不超过预定阀值
2. 确保线程暂停的总时间不超过预定阀值
3. 在确保达到延迟和吞吐量指标的情况下, 降低硬件配置以及成本。

为此, 用三种不同的配置, 将代码运行10分钟, 得到了三种不同的结果, 汇总如下:

| 堆内存大小 | GC算法 | 有效工作占比 | 最长停顿时间 |
|----|------|----|---|
| -Xmx12g | -XX:+UseConcMarkSweepGC | 89.8% | 560 ms |
| -Xmx12g | -XX:+UseParallelGC | 91.5% | 1,104 ms |
| -Xmx8g | -XX:+UseConcMarkSweepGC | 66.3% | 1,610 ms |

使用不同的GC算法，不同的内存配置，运行相同的代码，以测量GC暂停时间与 延迟、吞吐量的关系。实验的细节和结果在后面章节详细介绍。

注意, 为了尽量简单, 示例中只改变了很少的输入参数, 此实验也没有在不同CPU数量或者不同的堆布局下进行测试。

##延迟调优

假设有一个需求, 每次作业必须在 1000ms 内处理完成。我们知道, 实际的作业处理只需要100 ms，简化后， 两者相减就可以算出对 GC暂停的延迟要求。现在需求变成: GC暂停不能超过900ms。这个问题很容易找到答案, 只需要解析GC日志文件, 并找出GC暂停中最大的那个暂停时间即可。

从前面的结果图表可以看到,其中有一个配置达到了要求。运行的参数为:
`java -Xmx12g -XX:+UseConcMarkSweepGC Producer`
对应的GC日志中,暂停时间最大为 560 ms, 这达到了延迟指标 900 ms 的要求。如果同时满足吞吐量和系统容量目标的话,就可以说成功达成了GC调优目标, 调优结束。

##吞吐量调优

假定吞吐量指标为: 每小时完成 1300万次操作处理。同样是上面的配置, 其中有一种配置满足了需求:

此配置对应的命令行参数为:
`java -Xmx12g -XX:+UseParallelGC Producer`

可以看到,GC占用了 8.5%的CPU时间,剩下的 91.5% 是有效的计算时间。为简单起见, 忽略示例中的其他安全点。现在需要考虑:

- 每个CPU核心处理一次作业需要耗时 100ms
- 因此, 一分钟内每个核心可以执行 60,000 次操作(每个job完成100次操作)
- 一小时内, 一个核心可以执行 360万次操作
- 有四个CPU内核, 则每小时可以执行: 4 x 3.6M = 1440万次操作

根据这些理论，通过简单的计算就可以得出结论, 每小时可以执行的操作数为: 14.4 M * 91.5% = 13,176,000 次, 满足需求。

值得一提的是, 假若还要满足延迟指标, 那就有问题了, 最坏情况下, GC暂停时间为 1,104 ms, 最大延迟时间是前一种配置的两倍。

##系统容量调优

假设需要将软件部署到商用服务器上, 配置为 4核10G。这样的话, 系统容量的要求就变成: 最大的堆内存空间不能超过 8GB。有了这个需求, 我们需要调整为第三套配置进行测试:
程序可以通过如下参数执行:
`java -Xmx8g -XX:+UseConcMarkSweepGC Producer`

测试结果显示延迟大幅增长, 吞吐量同样大幅降低:

- 现在,GC占用了更多的CPU资源, 这个配置只有 66.3% 的有效CPU时间。因此,这个配置让吞吐量从最好的情况 13,176,000 操作/小时 下降到 不足 9,547,200次操作/小时.
- 最坏情况下的延迟变成了 1,610 ms, 而不再是 560ms。

通过对这三个维度的介绍, 你应该了解, 不是简单的进行性能优化, 而是需要从三种不同的维度来进行考虑, 测量, 并调优延迟和吞吐量, 同时还需要考虑系统容量的约束。
