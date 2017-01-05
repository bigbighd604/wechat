# GSLB是什么
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
2. 从运行上
3. 从功能上

由于篇幅较长，整篇文章会分为三个部分，本文是第一部分，从原理入手解释为什么很难。

## 从原理上看很难
说到GSLB，很难不谈到DNS，所有的GSLB实现都离不开DNS，不夸张的说很多GSLB的负载均衡就是通过DNS实现的。

下面这张图是一个最简单的GSLB架构图

![image](https://github.com/bigbighd604/wechat/blob/master/GSLBIsHard/images/GSLBArchSimplified.png)

我们以用户访问**www.example.com**为例来解释上图，图中有两个很重要的信息流:
1. local balancer上报数据给global balancer，global balancer汇总local balancer上报的数据，并将汇总后的数据发给权威DNS，权威DNS会根据global balancer上报的数据调整返回给用户的IP地址
2. 用户打开浏览器，通过ISP DNS递归查询网址www.example.com对应的IP地址，并发送请求到IP地址对应的服务器集群

### DNS based GSLB is bad in many ways
 
图一是张很简单的GSLB架构图，图中NYC和LON用户各自访问距离最近的服务，这是理想状态，在现实中要维持这种状态是有很多问题需要解决的。
#### 1. 如何返回最优IP
一般情况下，example.com的DNS权威服务器可以根据源请求IP返回最近的Google集群IP，对应到上图中就是NYC的用户得到的example.com集群IP实际是根据本地ISP DNS的IP返回的。这就涉及了一个问题，如果本地ISP没有DNS服务器而是使用其他上级ISP的DNS服务器，NYC的客户端就有可能得到距离很远的集群IP（假如本地ISP使用了多伦多的ISP进行DNS递归查询）。

当然了，现在我们有edns-client-subnet了，如果本地ISP DNS支持，那么客户端就可以得到距离最近的集群IP来提供服务。edns-client-subnet同样适用于用户手动设置DNS服务器的情况。

可以通过以下链接查看支持edns-client-subnet的大型DNS服务提供者：
http://www.afasterinternet.com/participants.htm。
这个列表肯定是不全的，不是所有公司都会更新这个名单。 

#### 2. 如何准确分流
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

另外一种可行的方案是，服务器端在收到用户请求时判断用户是否在访问最优的集群，如不是最优集群就重定向用户 到距离最近/最优的集群。判断方法也比较简单，比如服务器端进行RTT的对比，如果RTT大于一定阈值就向自己的权威服务器提交客户IP查询更优结果，然后创建一个基于FQDN的重定向。就像当年Akamai做的那样：
https://blogs.akamai.com/2013/03/intelligent-user-mapping-in-the-cloud.html
这种方法对某些业务形态比较有效，比如长时间的视频流或下载业务，初始的那个额外重定向和DNS查询时间占比很小，但是性能改进很大；如果是短暂的HTTP请求，效果就不明显了。

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

如果不能部署anycast呢？来看看Dyn的DNS服务提供的anycast接入点你就明白了

![image](https://github.com/bigbighd604/wechat/blob/master/GSLBIsHard/images/DynAnycastNetworkMap.png)

还有另外一种方法是通过BGP的route policy在不同的集群对外通告同一IP地址段的路由，这些路由当中只有一个是最优路径，DNS权威服务器根据用户所在区域和集群容量信息返回某一集群VIP，正常请下对该VIP的数据包会被路由到本集群，当该集群故障时，其他集群对外通告的路由会生效，从而流量会被路由到其他健康的集群,同样会对有状态服务造成影响。这属于从网络层弥补应用层负载均衡的缺陷。《需要改正》

### 延迟控制系统
当你的GSLB系统具备了内部集群间的流量调度功能时，形成了一个反馈闭环，整个GSLB系统就具备了控制系统的特征，而且还是具有延迟的控制系统。

![image](https://github.com/bigbighd604/wechat/blob/master/GSLBIsHard/images/GSLBControlSystem.png)

上图去掉了互联网部分，只关注GSLB本身的主要构成，以4个集群为例。

信息流动过程为：
1. 本地集群的各个服务向local balancer汇报当前负载
2. 向global balancer汇报所有本地服务的负载和容量（服务owner提前配置）
3. global balancer收集到所有local balancer的上报之后计算出如何分配集群间的流量
4. global balancer下发流量调配指示；更新权威DNS服务器
5. local balancer调整流量分配
6. 回到步骤1
 
然而整个过程不是瞬间完成的，如图中所示：本地服务每10秒钟上报一次负载，本地balancer每5秒钟上报一次，本地balancer可以在上报的RPC请求结果中拿到流量分配指示，由于global balancer不可能实时计算所有集群所有服务的流量分配情况，所以本地balancer拿到的分配指示和本次上报是没有任何关系的，流量分配指示是根据历史数据计算出来的，WTF？！

#### 一个全球系统
你虽然可以通过调整上报时间间隔和global balancer的计算频率来缩小差距，但是考虑到正是一个全球性的系统，为了正确调度global balancer需要所有集群的数据，有几个问题是无法避免的
* 数据中心间的网络延迟
* 频繁上报、下发导致的跨网流量
* 大量上报数据对计算能力和实时性的要求

基本上就是一个权衡各种因素，最终选择一个合适的这种配置，就将就着用旧点的数据做流量调整吧！

#### 不能处理短暂的峰值
不能避免延迟，一个最大的问题就是
> **GSLB在面对短暂的流量峰值时是无能为力的。**
> 
> 所以，一般的GSLB系统有一个假设：服务的流量在很短时间内不会出现剧烈变化。

以上图为例，如果NYC集群突然受到超过集群容量的请求（比如短暂的DDoS），但是只持续了5秒钟的时间，这个峰值很可能都不会被local balancer观测到。但是，如果你的服务由于过载导致进程重启，但是需要较长时间恢复，从而NYC集群容量减少，十几秒钟之后流量被分配到其他集群，如果还有多次的短暂峰值会将你的服务推向雪崩或者进入全球超负荷运行状态。

上面的结论是基于一个前提条件：local balancer并不在请求路径上。如果local balancer在请求路径上呢，那么先挂掉的可能是local balancer，随之而来的是服务的全球超负荷运行。

你可能会说，难道没有DDoS防护措施吧，当然有了，但是DDoS的介入也是需要满足一定条件的，这种瞬时的流量峰值很难触发。（我对DDoS防护手段研究不深，但是根据经验觉得可能不会触发，因为触发DDoS保护也需要历史和当前的流量信息，而这些信息很可能就是GSLB系统提供的，如果GSLB系统本身检测不到又怎么传递给DDoS系统呢。）
