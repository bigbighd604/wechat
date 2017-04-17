# 数据中心开启微秒时代，你准备好了吗


## 现状
很多同学都熟悉Jeff Dean大神提出的：每个程序员都应该知道的延迟数字。
如果你不清楚，请自行Google搜索“Latency Numbers Every Programmer Should Know”。

### 定义
* 微秒（µs）：10^-6^秒
* 毫秒（ms）：10^-3^秒
* 纳秒（ns）：10^-9^秒

在普通的程序开发和运维时，我们日常打交道最多的延迟（latency）是毫秒，比如衡量服务的性能我们会说响应时间在XX毫秒以内，磁盘访问时间在YY毫秒等等。在内核和硬件系统开发时，更多的则是面对纳秒级别的延迟。

现在的硬件和软件开发目标都是在试图降低纳秒和毫秒级别事件的延迟:
> 比如纳秒级别的多级内存访问架构以及对应的同步编程模型

毫秒级别的优化主要是软件层面的，比如:
> 操作系统的上下文切换（Context Switch），程序A通过系统调用访问磁盘，操作系统在调度了I/O操作后切换程序B使用CPU，当I/O操作结束后，操作系统恢复程序A的执行。

由于磁盘的访问实在毫秒级别，而上下文切换时间实在微秒级别，所以尽管有2次上下文切换的代价，在程序A等待磁盘期间，其他程序依然可以获得可观的运行时间。

这在过去几十年内工作的很好，以至于微秒渐渐淡出了人们的视野。下图是利用Google ngram绘制的，从1800到2012年书中提及的纳秒、微秒和毫秒的趋势图。

![image](https://github.com/bigbighd604/wechat/blob/master/EraOfMicrosecond/images/ngram.png)


## 趋势
### 对低延迟的需求
如果摩尔定律依然有效，那么CPU处理性能会是今天实际处理性能的20倍。但是互联网公司和云计算，需要大量的计算能力，既然但CPU性能停止了增长，自然的为提供更好的服务，大公司会使用更多的服务器来扩展服务能力。比如，一个单一的Google搜索请求会产生上千个RPC请求，通过上千台服务器计算才能返回结果。这催生了对存储和通信更低的延迟要求。
### 高速网络的普及
互联网巨头数据中心的网络已经开始从10G向25G、40G演进，在不久的将来会达到100G。
在40G网络下，一个TCP数据包传输只需要不到0.5微秒的时间。在一个直径2，3百米的数据中心里，一个数据包端到端的传输也就需要1微秒左右的时间。

### 高速硬件的出现
高速存储设备提升了I/O访问性能，从机械磁盘的几十毫秒到flash设备的几十微秒，以及新一代的Non-Volatile Memory技术，比如Intel's Xpoint 3D memory可以达到几微秒的访问延迟。还有就是现在随着机器学习和人工智能的兴起，可以达到微秒级别的GPU和各种加速器（accelerators）也越来越多。

下表是高速网络和硬件的事件延迟对比：

![image](https://github.com/bigbighd604/wechat/blob/master/EraOfMicrosecond/images/latencies.png)

在数据中心领域，传统的毫秒级别的I/O设备，正在被微秒级别的硬件所取代。

## 挑战
在数据中心拥抱高速网络和存储设备的同时，也面临着严峻的挑战。针对纳秒和毫秒级别事件所做的优化在微秒级别失效了！
> 纳秒级的乱序执行、分支预测、超线程等技术无法应用于微秒级别。
> 应用于毫秒级别的优化，比如：上下文切换，无法应用于微秒级别，因为额外的延迟往往大于I/O设备的访问延迟

体现在具体性能上则是对数据中心网络和CPU的效率的影响。

### 对网络的影响
下图描述的是一个延迟2微秒(µs)的数据中心光纤网络，如何在经过一系列的软件开销，变成一个100微秒的光纤网络！
![image](https://github.com/bigbighd604/wechat/blob/master/EraOfMicrosecond/images/overhead.png)

> 延迟测量的是平均往返延迟（从软件开始），没有队列导致的延迟。


### 对CPU效率的影响
如果我们通过CPU处理的QPS来衡量CPU效率，通过简单的排队论可以知道：
> 服务时间(service time)的增加会导致吞吐量(throughput)的降低

在有很多微秒级别事件时，当服务时间里包含的开销过大时，吞吐量急剧下降。
如下图所示，Y轴表示的是CPU使用效率，X轴表示的是服务时间。不同的曲线表示的是不同微秒级开销下CPU的效率。（提供了开销为：1微秒，16微秒，以及没有开销三种对比）
![image](https://github.com/bigbighd604/wechat/blob/master/EraOfMicrosecond/images/throughput.png)

从上图可以看出，当服务时间小于5微秒时，即便是1微秒的额外开销都会导致CPU的效率大幅降低。

下表列出了一个Google的生产环境下测量的搜索服务的服务时间，服务时间通过衡量I/O时间之间的CPU instructions计算。
![image](https://github.com/bigbighd604/wechat/blob/master/EraOfMicrosecond/images/instructions.png)

随着数据中心大规模使用快速闪存或者新一代的NVM内存，大量的服务时间会落在0.5-10微秒之间，微秒级别的开销会大大降低CPU的使用效率。

## 结论
现在的硬件和系统软件在支撑微秒级别硬件上存在严重的不足，系统设计者们不能忽视微妙级事件对系统效率的影响，因为大规模的数据中心开始进入微秒级硬件时代。

短期内，微秒级别设备带来的收益会被现在的软件系统稀释，尤其是没有经过任何针对微秒级事件优化的软件。

长期内，计算机科学家们需要设计针对微秒级别事件的系统，以及更底层的优化：比如减少lock contention、同步、以及更低开销的中断处理、任务调度等，总之专门针对微秒级别的优化。

这些针对微秒级别的优化设计，以及这些高速I/O会引发新一轮的软件和编程模型进化，通过更好的利用这些超低延迟的通信机制，可以显著提高数据中心的有效计算能力！

On the hardware side, new ideas “ideas are needed to enable context switching across a large number of threads (tens to hundreds per processor, though finding the sweet spot is an open question) at extremely fast latencies (tens of nanoseconds). … System designers need new hardware optimizations to extend the use of synchronous blocking mechanisms and thread-level parallelism to the micro-second range.”


