接自：http://www.importnew.com/2057.html  http://www.importnew.com/3146.html http://www.pianshen.com/article/2178378771/

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