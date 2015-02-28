#Dapper，大规模分布式系统的跟踪系统

------
Distributed Systems Tracing Infrastructure是极好的，此paper与我有交集是在2014年初，有公司高层发起分布式trace项目（此前，我领导单挑完成了类Dapper项目，受益良多，我只是对其提过想加入的合适的features(/ □ \)我就是只擅长提需求而已），入坑前被嘱咐先读Dapper论文，遗憾的是，后来公司级别的分布式trace大量入侵业务层代码，心力不济后不了了之，没有在开发、调试、运维上对开发者有任何帮助，读了此paper，特别是Dapper creative 应用，你是否也为曾嘲笑过内部分布式trace不给力而颤抖？



感谢网友[bigbully ](https://github.com/bigbully/Dapper-translation)的翻译。以下是我摘抄笔记。


------
[TOC]

###概述
>* 当代的互联网服务是由结构复杂的大规模分布式集群实现
>* 需要可以帮助理解系统行为，分析性能的工具，google中发明了Dapper--一种大规模的分布式跟踪系统
>* Dapper它是如何满足一个**低损耗、应用透明的、大范围部署**这三个需求的
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
###2. Distributed Tracing in Dapper
![tup](https://raw.githubusercontent.com/mars00772/write/master/drapper/server_call.jpg)




>* 为服务器上每一次你发送和接收动作来收集跟踪标识符和时间戳
>* 为了将所有记录条目与一个给定的发起者（例如，图1中的RequestX）关联上并记录所有信息，现在有两种解决方案，
>* 黑盒(black-box)方案：使用统计回归技术来推断两者之间的关系。需要更多历史数据，以获得足够的精度，依赖于统计推论.
>* 基于标注(annotation-based)的监控方案:应用程序或中间件明确地标记一个全局ID，从而连接每一条记录和发起者的请求。（分配方式可以简单地用机器IP_进程pid_线程id_ts)
>* Dapper跟踪模型使用的树形结构，Span以及Annotation。

#### 2.1 Trace trees and spans
span是跟踪树的节点，有span name，span id，span parent id等属性。span在日志文件中只是简单的代表span的开始和结束时间，
   span在一个大的跟踪过程中的表达，待插图。所有span都挂在一个特定的跟踪上，也共用一个跟踪id，在一个典型的Dapper跟踪中，我们希望为每一个RPC对应到一个单一的span上，而且每一个额外的组件层都对应一个跟踪树型结构的层级。

#### 2.2 Instrumentation points
Dapper可以以对应用开发者近乎零浸入的成本对分布式控制路径进行跟踪，几乎完全依赖于基于少量通用组件库的改造。如下：
>* 当一个线程在处理跟踪控制路径的过程中，Dapper把这次跟踪的上下文的在Thread Local中进行存储。追踪上下文是一个小而且容易复制的容器，其中承载了Scan的属性比如跟踪ID和span ID。
>* 延迟或异步调用时，回调函数会与适当的线程关联，Dapper可以使用trace ID和span ID来辅助构建异步调用的路径。
####2.3 Annotation
>*  Dapper可以在跟踪的过程中添加额外的信息，以监控更高级别的系统行为，或帮助调试问题。这点其实很重要而且好用
>* Dapper也支持的key-value映射的 Annotation，提供给开发人员更强的跟踪能力，如持续的计数器，二进制消息记录和在一个进程上跑着的任意的用户数据。键值对的Annotation方式用来在分布式追踪的上下文中定义某个特定应用程序的相关类型。

####2.4 Sampling
![tu1](https://raw.githubusercontent.com/mars00772/write/master/drapper/tracedata.jpg)
####2.5 Trace collection
Dapper的跟踪记录和收集管道的过程分为三个阶段，如上图，
>* span数据写入（1）本地日志文件中
>* Dapper的守护进程和收集组件把这些数据从生产环境的主机中拉出来（2）
>* 最终写到（3）Dapper的Bigtable仓库中。

一次跟踪被设计成Bigtable中的一行，每一列相当于一个span。Bigtable的支持稀疏表格布局正适合这种情况，因为每一次跟踪可以有任意多个span。跟踪数据收集的平均延迟--即从应用中的二进制数据传输到中央仓库（Bigtable）所花费的时间--不多于15秒。

#### 2.5.1 Out-of-band trace collection
tip1:带外数据:传输层协议使用带外数据(out-of-band，OOB)来发送一些重要的数据,如果通信一方有重要的数据需要通知对方时,协议能够将这些数据快速地发送到对方。为了发送这些数据，协议一般不使用与普通数据相同的通道,而是使用另外的通道。
tip2:这里指的in-band策略是把跟踪数据随着调用链进行传送，out-of-band是通过其他的链路进行跟踪数据的收集，Dapper的写日志然后进行日志采集的方式就属于out-of-band策略
note:不明白，是说
The Dapper system as described performs trace logging and collection out-of-band with the request tree itself.(说Dapper的日志和收集日志用out-of-band collection方式)原因是
>* 首先，带内收集方案--这里跟踪数据会以RPC响应头的形式被返回--会影响应用程序网络动态。
>* 其次，带内收集方案要求所有的RPC是完美嵌套的，但实际中有不是这样，带内收集无法解释这种非嵌套的执行模式（但又没说带外怎么解决了这个问题）
#### 2.5.2 Security and privacy considerations
（只能说很机智）
###3. Dapper Deployment Status
#### 3.1 Dapper runtime library
也许Dapper代码中中最关键的部分，就是对基础RPC、线程控制和流程控制的组件库的植入，其中包括span的创建，采样率的设置，以及把日志写入本地磁盘。除了做到轻量级，植入的代码更需要稳定和健壮，因为它与海量的应用对接，维护和bug修复变得困难。植入的核心代码是由未超过1000行的C++和不超过800行Java代码组成。为了支持键值对的Annotation还添加了额外的500行代码。

####3.2 Production coverage
####3.4  Use of trace annotations
Java应用的作用域往往是更接近最终用户(C++偏底层);这些类型的应用程序经常处理更广泛的请求组合，因此具有比较复杂的控制路径。所以更多是用annotations.

###4. Managing Tracing Overhead
####4.1 Trace generation overhead
>* 生成跟踪的开销是Dapper性能影响中最关键的部分，因为收集和分析可以更容易在紧急情况下被关闭。
>* Dapper运行库中最重要的跟踪生成消耗在于创建和销毁span和annotation，以及logging
>* 根span的创建和销毁需要损耗平均204纳秒的时间，而同样的操作在其他span上需要消耗176纳秒，因为前者要分配全局唯一trace id（为何销毁也比较久）？
> * Dapper运行期对Thread Local查找操作构成，这平均只消耗9纳秒。
>* 在Dapper运行期写入到本地磁盘是最昂贵的操作，但是他们的可见损耗大大减少，因为写入日志文件和操作相对于被跟踪的应用系统来说都是异步的。不过，日志写入的操作如果在大流量的情况，尤其是每一个请求都被跟踪的情况下就会变得可以察觉到。
####4.2 Trace collection overhead
####4.3 Effect on production workloads
#### 4.4 Adaptive sampling
>* 任何给定进程的Dapper的消耗和每个进程单位时间的跟踪的采样率成正比。Dapper的第一个生产版本在Google内部的所有进程上使用统一的采样率，为1/1024。
>* 这样一来，低流量低负载自动提高采样率，而在高流量高负载的情况下会降低采样率，使损耗一直保持在控制之下。

#### 4.5 Coping with aggressive sampling

#### 4.6 Additional sampling during collection
>* Dapper的团队还需要控制写入中央资料库的数据的总规模，因此为达到这个目的，我们结合了二级采样。
>* 为了维持物质资源的需求和渐增的Bigtable的吞吐之间的灵活性，我们在收集系统自身上增加了额外的采样率的支持。
###6 Experiences
我觉得他们脑洞大开的Drapper有趣使用方式是良好启发。
####6.1 Using Dapper during development
当轮到从头重新设计一个广告审查服务时，这个团队迭代的从第一个系统原型开始使用Dapper，并且，最终用Dapper一直维护着他们的系统。Dapper帮助他们从以下几个方面改进了他们的服务：
>* 性能：开发人员针对请求延迟的目标进行跟踪，并对容易优化的地方进行定位。
>* 正确性：广告审查服务围绕大型数据库系统搭建。系统同时具有只读副本策略（数据访问廉价）和读写的主策略（访问代价高）。Dapper被用来在很多种情况中确定，哪些查询是无需通过主策略访问而可以采用副本策略访问。Dapper现在可以负责监控哪些主策略被直接访问，并对重要的系统常量进行保障。
>* 理解性：广告审查查询跨越了各种类型的系统，包括BigTable—之前提到的那个数据库，多维索引服务，以及其他各种C++和Java后端服务。Dapper的跟踪用来评估总查询成本，促进重新对业务的设计，用以在他们的系统依赖上减少负载。
>* 测试：新的代码版本会经过一个使用Dapper进行跟踪的QA过程，用来验证正确的系统行为和性能。在跑测试的过程中能发现很多问题，这些问题来自广告审查系统自身的代码或是他的依赖包。



####6.2 Integration with exception monitoring

####6.3 Addressing long tail latency

####6.4 Inferring service dependencies

####6.5 Network usage of different services

####6.6 Layered and Shared Storage Systems

###7 Firefighting with Dapper

###8 Other Lessons Learned

###9 Related Work

###10 Conclusions
> Written with [StackEdit](https://stackedit.io/).