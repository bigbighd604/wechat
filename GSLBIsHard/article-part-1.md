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

我们以用户访问**www.example.com**为例来解释上图，图中有两个很重要的信息流:
1. local balancer上报数据给global balancer，global balancer汇总local balancer上报的数据，并将汇总后的数据发给权威DNS，权威DNS会根据global balancer上报的数据调整返回给用户的IP地址
2. 用户打开浏览器，通过ISP DNS递归查询网址www.example.com对应的IP地址，并发送请求到IP地址对应的服务器集群

### DNS based GSLB is bad in many ways
 
图一是张很简单的GSLB架构图，图中NYC和LON用户各自访问距离最近的服务，这是理想状态，在现实中要维持这种状态是有很多问题需要解决的。
#### 1. 如何返回离用户最近的集群IP
一般情况下，example.com的DNS权威服务器可以根据源请求IP返回最近的Google集群IP，对应到上图中就是NYC的用户得到的example.com集群IP实际是根据本地ISP DNS的IP返回的。这就涉及了一个问题，如果本地ISP没有DNS服务器而是使用其他上级ISP的DNS服务器，NYC的客户端就有可能得到距离很远的集群IP（假如本地ISP使用了多伦多的ISP进行DNS递归查询）。

当然了，现在我们有edns-client-subnet了，如果本地ISP DNS支持，那么客户端就可以得到距离最近的集群IP来提供服务。edns-client-subnet同样适用于用户手动设置DNS服务器的情况。

可以通过以下链接查看支持edns-client-subnet的大型DNS服务提供者：
http://www.afasterinternet.com/participants.htm。
这个列表肯定是不全的，不是所有公司都会更新这个名单。 

#### 2. 如何准确分流负载
简单基于DNS的负载均衡方案，有个很大的弊端是流量分配不均。

原因主要有下面2点：
* ISP DNS
  > DNS解析结果会被缓存，一个DNS请求可能对应到成百上千个服务请求。

* 缺乏edns-client-subnet支持
  > * 用户手动设置DNS服务器，得不到最优IP解析
  > * 小ISP使用其他异地ISP的DNS，得不到最优IP解析
  > 
  > 这不仅会导致流量的跨地区访问，还会严重影响用户体验。

由于服务提供者，比如Google，很难控制用户自己的DNS设置以及广大DNS服务提供者对edns-client-subnet的支持，如何调整流量就必须从服务端进行了。

![image](https://github.com/bigbighd604/wechat/blob/master/GSLBIsHard/images/GSLBArchWithFeedback.png)

如图二所示，服务端balancer信息流新增了一个环节
> global balancer根据每个集群的容量和已有请求进行**集群间**的流量迁移。

这对GSLB系统提出2个新的要求：
1. local balancer上报的数据有新的要求，不仅要上报本地集群的容量还要对收到的流量进行上报，这样global blancer就可以根据每个集群容量和负载进行集群间的流量调配了。
2. local balancer要能够转发远大于本地集群容量的请求，这样才能在满足本地集群负载的情况下，转发多余的请求到其他集群。

这其实是在服务器端构建了一个反馈回路，local/global balancer可以根据集群容量、负载进行流量调配，消除了由于DNS调度导致的热点集群问题。

有吃瓜群众会问了：HttpDNS可以解决这个问题吧？如果你的服务是通过可控客户端提供的，那么HttpDNS可以帮助你解决这个问题，因为你也修改客户端使用HttpDNS；但是如果你的服务是通过浏览器提供的，那么HttpDNS是帮不上忙的。

#### 3. 如何快速切流
事故是一定会发生的，如何在事故发生时快速切走流量，将用户的影响降到最低是至关重要的。

![image](https://github.com/bigbighd604/wechat/blob/master/GSLBIsHard/images/GSLBArchWithFailure.png)

然而，基于DNS的切流机制是不靠谱的
* 客户端的DNS缓存，i.e 浏览器、操作系统
* ISP DNS缓存
* 不遵循DNS TTL的客户端

即便是能在第一时间更新权威服务器，不再返回故障集群的IP，由于以上原因，还是会有一段时间（从几分钟到几小时不等）请求仍然发送到故障集群，导致请求失败。

要解决这个问题，最好的方案是通过anycast，每个集群通过eBGP对外通告同样的VIP，互联网上的路由器会最终根据通告的VIP距离判断路由的优先级。所有用户对www.example.com的DNS请求都会返回同样的VIP地址，但是不同区域用户对VIP的请求会被路由到各自最近的集群。

当某个集群出现故障时，该集群对外通告的VIP会被撤回，路由更新之后，之前该集群服务的用户请求会被自动路由到次优集群，这对用户来说都是透明的，但是如果提供的服务是有状态的，会导致状态丢失，比如youtube视频播放会中断。

如果不能部署anycast呢？你懂的
![image](https://github.com/bigbighd604/wechat/blob/master/GSLBIsHard/images/DynAnycastNetworkMap.png)

还有另外一种方法是通过BGP的route policy在不同的集群对外通告同一IP地址段的路由，这些路由当中只有一个是最优路径，DNS权威服务器根据用户所在区域和集群容量信息返回某一集群VIP，正常请下对该VIP的数据包会被路由到本集群，当该集群故障时，其他集群对外通告的路由会生效，从而流量会被路由到其他健康的集群。同样会对有状态服务造成影响。

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



