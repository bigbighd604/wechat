# Definition of GSLB
GSLB一种常见的解释是Global Server Load Balancing的缩写，也可以认为是Global Software Load Balancing的缩写，因为在互联网公司Server很可能是一个Software。翻译过来就是：全局服务器/软件负载均衡。

根据维基百科：
> 负载均衡可以促进工作负载在多个计算资源之间的分配，比如：计算机之间、计算集群直接、网络链路、CPUs或者磁盘。目的是优化资源使用率，最大化吞吐量，最小化响应时间，并且避免任何单一资源的过载。

维基百科的定义偏重于负载分配以优化资源使用率，大型互联网公司应用GSLB还有另外2个很重要的目标：
* 提高系统的可用性（availability）
  > 当某个站点/集群整体不可用时，系统仍然可以通过其他站点/集群提供完整可用的服务
* 降低用户的访问延迟（latency）
  > 根据用户地理位置，将请求发送到最近的站点/集群提供服务，降低用户的请求延迟，改进用户体验

# Why it's hard
首先定义一下“hard”:
> 这里并不是指很难实现/搭建一套GSLB系统，而是指实现/搭建一套可以**达到开篇提到的三个目标**的GSLB系统很难。

为什么说很难呢，我尝试从三个大的方面去解释
1. 从原理上
2. 从实现上
3. 从运行上

由于篇幅较长，整篇文章会分为三个部分，本文是第一部分，从原理入手解释为什么很难。

## it's hard in principle
说到GSLB，很难不谈到DNS，所有的GSLB实现都离不开DNS，不夸张的说很多GSLB的负载均衡就是通过DNS实现的。
下面这张图是一个最简单的GSLB架构图

![image](https://github.com/bigbighd604/wechat/blob/master/GSLBIsHard/images/GSLBArchSimplified.png)

### DNS based GSLB is bad in many ways
Let's consider the following simplified GSLB 
> image here

problems: 
* Load distribution doesn't work well
    * ISP DNS cache: a single DNS query doesn't mean 1 request, it may mean millions
    * No feedback loop to shift traffic around
* Fail over doesn't work well
    * client side DNS cache
    * ISP DNS cache 
    * some clients doesn't obey DNS TTL

how to fix fail over problem?
* return multiple A records
* use anycast
* 

how to fix load distribution problem?
* introduce feedback loop, see next section for control system.
    * we cannot control where users send traffic from, but we can control how to route traffic when it enters our clusters. use 统一接入层 + 内部GSLB rebalancing traffic

### control system with time delay
### stateful system
### global system
### cannot handle short spike
### Overload handling

## it's hard in implementation detail
### Need geo and RTT information about clusters
### Avoid proxy mode
### Need generic enough
#### out of box SDK should support a basic working solution
#### customized SDK should support more advanced solution without any server side changes.

### Need support stateful service too

### Need consider hackend health checking

### Need consider automatic failover

### Need support transparent troubleshooting mechanism for clients

### scheduling algrithm
Simple algorithms include random choice or round robin. More sophisticated load balancers may take additional factors into account, such as a server's reported load, least response times, up/down status (determined by a monitoring poll of some kind), number of active connections, geographic location, capabilities, or how much traffic it has recently been assigned.

## it's hard to run
### GSLB as a self service
Not only register and use the service, but also provide tools to support shifting traffic.
### GSLB as a micro service
In order to support even larger number clients and services.



