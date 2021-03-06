- 接自：http://www.importnew.com/2057.html  http://www.importnew.com/3146.html http://www.pianshen.com/article/2178378771/
- gceasy分析理论及在线GC日志分析工具：https://blog.gceasy.io/ https://blog.gceasy.io/2016/06/18/garbage-collection-log-analysis-api/  https://gceasy.io/

GC调优标准量化参考：https://blog.csdn.net/dabokele/article/category/6762404
https://blog.csdn.net/dabokele/article/details/59794040 

JVC调优思维导图：https://blog.csdn.net/u011683530/article/details/51013219
https://img-blog.csdn.net/20160330122115344?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast

### GC合理的标准（并非绝对）---适用于响应时间优先场景（即高并发）：
1. Minor GC执行的很快（小于50ms）---（G1收集器默认MinorGC 停顿时间是200ms，很多场景可考虑控制在100ms以内）
2. Minor GC执行的并不频繁（大概10秒一次）
3. Full GC执行的很快（小于1s）
4. Full GC执行的并不频繁（10分钟一次）
- 数字并不是绝对的；他们根据服务状态的不同而有所区别，某些服务可能满足于Full GC每次0.9秒的速度，但另一些可能不是；因此，针对不同的服务设定不同的值以决定是否进行GC优化。
- 查看GC状态的时候有件事你需要特别注意，那就是不要只关注Minor GC 和Full GC的执行时间。还要关注GC执行的次数,例如，当新生代空间较小时，Minor GC会过于频繁的执行（有时每秒超过1次）。另外，转移到老年代的对象数增多，则会导致Full GC执行次数增多。因此，别忘了加上–gccapacity参数来查看具体占用了多少空间。
- HPJMeter 是用于分析-verbosegc 日志的工具；易于使用和分析结果；通过HPJmeter你可以很轻易查看GC执行时间以及GC发生频率。

### JVM GC调优：关键性能指标
https://blog.csdn.net/xiaocszn/article/details/83108058 
https://blog.gceasy.io/2016/10/01/garbage-collection-kpi/
- 对java应用的内存和GC调优时，我们应该基于关键性能指标来做决定，但是指标有很多，哪些我们应该着重考虑呢？这篇文章将尝试讨论这个问题。哪些是我们应该考虑的指标？
1. 吞吐量
2. 延迟
3. CPU消耗

#### 1. 吞吐量
吞吐量是指单位时间内能完成的生产任务的量，首先我们得明确一下，什么是生产任务，什么是非生产任务？
- 生产任务：大部分时间在执行的业务任务
- 非生产任务：像GC等跟业务无关的任务
举个例子，假设你的应用跑了60分钟，其中2分钟在做GC操作，那么，应用有3.33%的时间在做GC(2/60)，吞吐率是96.67%(100-3.33)。
现在问题来了，吞吐率多少是可接受的呢？这取决于我们的需求，一般来说吞吐率应该至少达到95%
#### 2. 延迟
这个指标主要是指单次GC执行时长，应该从三个方面来考虑
a) 平均GC时长
b) 最大GC时长: 如果你负责的服务SLA(Service Level Agreements)是任何请求不超过10秒，那么你的最大GC停顿时长就不能超过10秒。因为在GC停顿时，整个JVM会暂停，任何业务代码都无法执行，因此所以最大GC停顿时间很重要。
c) GC时长分布
#### 3. CPU消耗
CPU消耗会随着GC算法的不同和内存设置的不同而不同。有些GC算法比较消耗CPU（比如Parallel, CMS），而另一些算法比较节省CPU（比如Serial）
根据内存调优准则，以上这三个优化指标，你最多只能三者取其二
1. 如果你想要比较好的吞吐量和延迟，那就得在CPU消耗上有所牺牲
2. 如果你想要比较好的吞吐量和CPU消耗，那就得在延迟上有所牺牲
3. 如果你想要比较好的延迟和CPU消耗，那就得在吞吐量上有所牺牲

http://www.sohu.com/a/255601517_355140
- 理解应用需求和问题，确定调优目标。假设，我们开发了一个应用服务，但发现偶尔会出现性能抖动，出现较长的服务停顿。评估用户可接受的响应时间和业务量，将目标简化为，希望 GC 暂停尽量控制在 200ms 以内，并且保证一定标准的吞吐量。
- 掌握 JVM 和 GC 的状态，定位具体的问题，确定真的有 GC 调优的必要。具体有很多方法，比如，通过 jstat 等工具查看 GC 等相关状态，可以开启 GC 日志，或者是利用操作系统提供的诊断工具等。例如，通过追踪 GC 日志，就可以查找是不是 GC 在特定时间发生了长时间的暂停，进而导致了应用响应不及时。
- 这里需要思考，选择的 GC 类型是否符合我们的应用特征，如果是，具体问题表现在哪里，是 Minor GC 过长，还是 Mixed GC 等出现异常停顿情况；如果不是，考虑切换到什么类型，如 CMS 和 G1 都是更侧重于低延迟的 GC 选项。
- 通过分析确定具体调整的参数或者软硬件配置。
- 验证是否达到调优目标，如果达到目标，即可以考虑结束调优；否则，重复完成分析、调整、验证这个过程。
