系统级性能优化通常包括两个阶段：性能剖析（performance profiling）和代码优化。
性能剖析的目标是寻找性能瓶颈，查找引发性能问题的原因及热点代码。

代码优化的目标是针对具体性能问题而优化代码或编译选项，以改善软件性能。
性能剖析阶段，需要借助于现有的profiling工具，如perf等。在代码优化阶段往往需要借助开发者的经验，编写简洁高效的代码，甚至在汇编级别合理使用各种指令，合理安排各种指令的执行顺序。

perf是一款Linux性能分析工具。Linux性能计数器是一个新的基于内核的子系统，它提供一个性能分析框架，比如硬件（CPU、PMU(Performance Monitoring Unit)）功能和软件(软件计数器、tracepoint)功能。
通过perf，应用程序可以利用PMU、tracepoint和内核中的计数器来进行性能统计。它不但可以分析制定应用程序的性能问题（per thread），也可以用来分析内核的性能问题，当然也可以同事分析应用程序和内核，从而全面理解应用程序中的性能瓶颈。

使用perf，可以分析程序运行期间发生的硬件事件，比如instructions retired、processor clock cycles等；也可以分析软件时间，比如page fault和进程切换。

https://www.cnblogs.com/arnoldlu/p/6241297.html