#Dapper，大规模分布式系统的跟踪系统

------
感谢网友[bigbully ](https://github.com/bigbully/Dapper-translation)的翻译，造福我等英盲人。以下是我摘抄笔记。


------
[TOC]

###概述
>* 当代的互联网服务是由结构复杂的大规模分布式集群实现
>* 需要可以帮助理解系统行为，分析性能的工具，google中发明了Drapper--一种大规模的分布式跟踪系统
>* Drapper它是如何满足一个**低损耗、应用透明的、大范围部署**这三个需求的
>* Dapper是怎么构建和部署的
>* 介绍一些使用Dapper搭建的分析工具，分享一下这些工具在google内部使用的统计数据
>* 展现一些使用场景，最后会讨论一下我们迄今为止从Dapper收获了些什么
###介绍

>* 为了能在分布式系统中帮助开发者理解横跨不同APP，服务器之间的关联动作，Drapper收集更多的复杂分布式系统的行为信息

#### Universal Search Example
>* 前端服务--》上百个独立index的查询服务器--》各自发送到多个子系统--》分别处理广告，拼写检查，查找图片、视频或新闻。---》子系统筛选结果，得到结果。期间可能调用了上千个服务器，工程师只能知道这个查询耗时过长，但无从知晓问题到底是由于哪个服务调用导致。
>* 系统任何时间可能被修改上线过，不可苛求工程师对这次universal search了如指掌(我发现系统越来越大后，新人做事愈发捻手捻脚，何况我等系统已经足够小了)
>* 对Drapper的要求结论：无所不在的部署，持续的监控。
####Design goals
>* 1.低消耗：跟踪系统对在线服务的影响应该做到足够小。在一些高度优化过的服务，即使一点点损耗也会很容易察觉到，而且有可能迫使在线服务的部署团队不得不将跟踪系统关停。（哈哈哈。。。。）
>* 2.应用级的透明：对于应用的程序员来说，是不需要知道有跟踪系统这回事的。如果一个跟踪系统想生效，就必须需要依赖应用的开发者主动配合，那么这个跟踪系统也太脆弱了，往往由于跟踪系统在应用中植入代码的bug或疏忽导致应用出问题，这样才是无法满足对跟踪系统“无所不在的部署”这个需求。面对当下想Google这样的快节奏的开发环境来说，尤其重要。（我司层内部trace显著入侵业务代码，令我印象非常深刻）
>* 3.延展性：Google至少在未来几年的服务和集群的规模，监控系统都应该能完全把控住。
一个额外的设计目标是为跟踪数据产生之后，进行分析的速度要快，理想情况是数据存入跟踪仓库后一分钟内就能统计出来。尽管跟踪系统对一小时前的旧数据进行统计也是相当有价值的，但如果跟踪系统能提供足够快的信息反馈，就可以对生产环境下的异常状况做出快速反应。

做到真正的应用级别的透明，这应该是当下面临的最挑战性的设计目标，我们把核心跟踪代码做的很轻巧，然后把它植入到那些无所不在的公共组件种，比如**线程调用、控制流以及RPC库**。使用**自适应的采样率**可以使跟踪系统变得可伸缩，并降低性能损耗，这些内容将在第4.4节中提及。
###1.1 Summary of contributions
###2. Dapper的分布式跟踪
![tup](https://raw.githubusercontent.com/mars00772/write/master/drapper/server_call.jpg)




>* 为服务器上每一次你发送和接收动作来收集跟踪标识符和时间戳
>* 为了将所有记录条目与一个给定的发起者（例如，图1中的RequestX）关联上并记录所有信息，现在有两种解决方案，
>* 黑盒(black-box)方案：使用统计回归技术来推断两者之间的关系。需要更多历史数据，以获得足够的精度，依赖于统计推论.
>* 基于标注(annotation-based)的监控方案:应用程序或中间件明确地标记一个全局ID，从而连接每一条记录和发起者的请求。（分配方式可以简单地用机器IP_进程pid_线程id_ts)
>* Dapper跟踪模型使用的树形结构，Span以及Annotation。




#### 2.1 跟踪树和span
在Dapper跟踪树结构中，树节点是整个架构的基本单元，而每一个节点又是对span的引用。节点之间的连线表示的span和它的父span直接的关系。虽然span在日志文件中只是简单的代表span的开始和结束时间，
 
 
图2：5个span在Dapper跟踪树种短暂的关联关系
在图2中说明了span在一个大的跟踪过程中是什么样的。

Dapper记录了span名称，以及每个span的ID和父ID，以重建在一次追踪过程中不同span之间的关系。如果一个span没有父ID被称为root span。所有span都挂在一个特定的跟踪上，也共用一个跟踪id

（在图中未示出）。

所有这些ID用全局唯一的64位整数标示。在一个典型的Dapper跟踪中，我们希望为每一个RPC对应到一个单一的span上，而且每一个额外的组件层都对应一个跟踪树型结构的层级。


 
图3：在图2中所示的一个单独的span的细节图
图3给出了一个更详细的典型的Dapper跟踪span的记录点的视图。在图2中这种某个span表述了两个“Helper.Call”的RPC(分别为server端和client端)。span的开始时间和结束时间，以及任何RPC的时间信息都通过Dapper在RPC组件库的植入记录下来。如果应用程序开发者选择在跟踪中增加他们自己的注释（如图中“foo”的注释）(业务数据)，这些信息也会和其他span信息一样记录下来。
记住，任何一个span可以包含来自不同的主机信息，这些也要记录下来。事实上，每一个RPC span可以包含客户端和服务器两个过程的注释，使得链接两个主机的span会成为模型中所说的span。由于客户端和服务器上的时间戳来自不同的主机，我们必须考虑到时间偏差。在我们的分析工具，我们利用了这个事实：RPC客户端发送一个请求之后，服务器端才能接收到，对于响应也是一样的（服务器先响应，然后客户端才能接收到这个响应）。这样一来，服务器端的RPC就有一个时间戳的一个上限和下限。
2.2 植入点
Dapper可以以对应用开发者近乎零浸入的成本对分布式控制路径进行跟踪，几乎完全依赖于基于少量通用组件库的改造。如下：
•	当一个线程在处理跟踪控制路径的过程中，Dapper把这次跟踪的上下文的在ThreadLocal中进行存储。追踪上下文是一个小而且容易复制的容器，其中承载了Scan的属性比如跟踪ID和span ID。
•	当计算过程是延迟调用的或是异步的，大多数Google开发者通过线程池或其他执行器，使用一个通用的控制流库来回调。Dapper确保所有这样的回调可以存储这次跟踪的上下文，而当回调函数被触发时，这次跟踪的上下文会与适当的线程关联上。在这种方式下，Dapper可以使用trace ID和span ID来辅助构建异步调用的路径。
•	几乎所有的Google的进程间通信是建立在一个用C++和Java开发的RPC框架上。我们把跟踪植入该框架来定义RPC中所有的span。span的ID和跟踪的ID会从客户端发送到服务端。像那样的基于RPC的系统被广泛使用在Google中，这是一个重要的植入点。当那些非RPC通信框架发展成熟并找到了自己的用户群之后，我们会计划对RPC通信框架进行植入。
Dapper的跟踪数据是独立于语言的，很多在生产环境中的跟踪结合了用C++和Java写的进程的数据。在3.2节中，我们讨论应用程序的透明度时我们会把这些理论的是如何实践的进行讨论。


> Written with [StackEdit](https://stackedit.io/).