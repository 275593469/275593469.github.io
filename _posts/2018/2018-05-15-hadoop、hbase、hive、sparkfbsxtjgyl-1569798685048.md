---
layout: post
title:  "hadoop、hbase、hive、spark分布式系统架构原理"
categories: 大数据
tags: hadoop hbase hive spark
author: jiangc
excerpt: hadoop、hbase、hive、spark分布式系统架构原理
---
**hadoop、hbase、hive、spark分布式系统架构原理**

2018年05月15日 11:22:50 [数据架构师](https://me.csdn.net/luanpeng825485697) 阅读数：11461更多

所属专栏： [微服务架构](https://blog.csdn.net/column/details/19467.html)

 版权声明：本文为博主原创文章，转载请注明来源。开发合作联系[email protected] https://blog.csdn.net/luanpeng825485697/article/details/80319552

**全栈工程师开发手册 （作者：栾鹏）**

[**架构系列文章**](https://blog.csdn.net/luanpeng825485697/article/details/83830968)

机器学习、数据挖掘等各种大数据处理都离不开各种开源分布式系统，hadoop用户分布式存储和map-reduce计算，spark用于分布式机器学习，hive是分布式数据库，hbase是分布式kv系统，看似互不相关的他们却都是基于相同的hdfs存储和yarn资源管理，

**hadoop、spark、Hbase、Hive、hdfs简介**

**Hbase** ：是一个nosql数据库，和mongodb类似

**hdfs** ：hadoop distribut file system，hadoop的分布式文件系统

**Hive** ：hive是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件（或者非结构化的数据）映射为一张数据库表，并提供简单的sql查询功能，可以将sql语句转换为MapReduce任务进行运行。 其优点是学习成本低，可以通过类SQL语句快速实现简单的MapReduce统计，不必开发专门的MapReduce应用，十分适合数据仓库的统计分析。

**使用Hive，就不用去写MapReduce，而是写sql语句就行了。**

**sqoop** ：sqoop是和Hive一起使用的。Sqoop(发音：skup)是一款开源的工具，主要用于在Hadoop(Hive)与传统的数据库(mysql、postgresql…)间进行数据的传递，可以将一个关系型数据库（例如 ： MySQL ,Oracle ,Postgres等）中的数据导进到Hadoop的HDFS中，也可以将HDFS的数据导进到关系型数据库中。

使用sqoop导入数据至hive常用语句 ：

直接导入hive表

     sqoop import --connect jdbc:postgresql://ip/db\_name--username user\_name  --table table\_name  --hive-import -m 5

1. 1

内部执行实际分三部，1.将数据导入hdfs（可在hdfs上找到相应目录），2.创建hive表名相同的表，3，将hdfs上数据传入hive表中

[![image](/images/2018\05\dsj\1569798685049.jpg "image")]

**分布式hadoop架构**

hadoop分为几大部分：yarn负责资源和任务管理、hdfs负责分布式存储、map-reduce负责分布式计算

**YARN资源任务调度**

YARN总体上仍然是master/slave（主从）结构

ResourceManager是Master上一个独立运行的进程，负责集群统一的资源管理、调度、分配等等；NodeManager是Slave上一个独立运行的进程，负责上报节点的状态；App Master和Container是运行在Slave上的组件，负责应用程序相关事务，比如任务调度、任务监控和容错等，Container是yarn中分配资源的一个单位，包涵内存、CPU等等资源，yarn以Container为单位分配资源。

YARN的基本架构 ，YARN的架构设计使其越来越像是一个云操作系统，数据处理操作系统。

[![image](/images/2018\05\dsj\1569798685055.jpg "image")]

从YARN的架构来看，它主要由ResourceManager、 NodeManager、ApplicationMaster 和 Container组成

**（1）ResourceManager（RM）**

RM是一个全局的资源管理器，负责整个系统的 资源管理和分配。它主要由两个组件构成：调度器（Schedule）和应用程序管理器（Application Manager， ASM）

YARN分层结构的本质是ResourceManager。这个实体控制整个集群并管理应用程序向基础计算资源的分配。ResourceManager将各个资源部分（计算、内存、带宽等）精心安排给基础NodeManager（YARN的每节点代理）。ResourceManager还与ApplicationMaster一起分配每个应用程序内每个任务所需的资源，与NodeManager一起启动和监视它们的基础应用程序。在此上下文中，ApplicationMaster承担了以前的TaskTracker的一些角色，ResourceManager承担了JobTracker的角色。

a）调度器（Scheduler）

调度器根据容量、队列等限制条件（如每个队列分配一定的资源，最多执行一定数量的作业等），将系统中的资源分配给各个正在运行的应用程序。该调度器是一个&quot;纯调度器&quot;，它不再从事任何与具体应用程序相关的工作。，比如不负责监控或者跟踪应用的执行状态等，也不负责重新启动因应用执行失败或者硬件故障而产生的失败任务，这些均交由应用程序相关的ApplicationMaster完成。调度器仅根据各个应用程序的资源需求进行资源分配，而资源分配单位用一个抽象概念&quot;资源容器&quot;（Resource Container，简称Container）表示，Container是一个动态资源分配单位，它将内存、CPU、磁盘、网络等资源封装在一起，从而限定每个任务使用的资源量。此外，该调度器是一个可插拔的组件，用户可根据自己的需要设计新的调度器，YARN提供了多种直接可用的调度器，比如Fair Scheduler和Capacity Scheduler等。

b）应用程序管理器（Application Manager，ASM）

应用程序管理器负责管理整个系统中所有的应用程序，包括应用程序提交、调度协调资源以启动ApplicationMaster、监控ApplicationMaster运行状态并在失败时重新启动它。

**（2）ApplicationMaster（AM）**

ApplicationMaster管理一个在YARN内运行的应用程序的每个实例。

ApplicationMaster负责申请而获得的来自ResourceManager的资源，并通过NodeManager监视容器的执行和资源的使用（cpu、内存等资源分配）。

请注意，尽管目前的资源更加传统（CPU核心、内存），但未来会带来基于手头任务的新资源类型（比如图形处理单元，或专用处理设备）。从YARN角度来讲，ApplicationMaster使用户代码因此存在潜在安全问题。YARN假设ApplicationMaster存在错误或者甚至是恶意的，因此将它们当做无特权的代码对待。

AM功能：数据切分、为应用程序申请资源并进一步分配给内部任务、任务监控与容错

**（3）NodeManager（NM）**

NodeManager管理一个YARN集群中的每个节点。NodeManager提供针对集群中每个节点的服务，从监督对一个容器的终身管理到监视资源和跟踪节点健康。MRv1通过插槽管理Map和Reduce任务执行，而NodeManager管理抽象容器，这些容器代表着可供一个特定应用程序使用的针对每个节点的资源。YARN继续使用HDFS层。它的主要NameNode主要用于元数据服务，而DataNode用于分散在一个集群中的复制存储服务。

NM是每个节点上的资源和任务管理器。一方面，它会定时地向RM汇报本节点上的资源使用情况和各个Container运行状态；另一方面，它接收并处理来自AM的 Container 启动/停止等各种请求。

功能：单个节点上的资源管理和任务。处理来自于resourcemanager的命令。处理来自域ApplicationMaster的命令。

**（4）Container**

Container是YARN中的资源抽象，它封装了某个节点上的多维度资源，如内存，CPU，磁盘，网络等。当AM向RM申请资源时，RM为AM返回的资源便是用Container表示的。YARN会为每个任务分配一个Container，且该任务只能用该Container中描述的资源。（ **注意：ApplicationMaster获得的资源不一定是当前主机节点上的** ）

**应用程序在yarn上的调度流程**

Client向ResourceManager提交的每一个应用程序都必须有一个Application Master，它经过ResourceManager分配资源后，运行于某一个Slave节点的Container中，每个应用程序包含多个任务task，每个任务同样也运行在某一个Slave节点的Container容器中（不一定和Application Master在同一个Slave节点中）。RM，NM，AM乃至普通的Container之间的通信，都是用RPC机制。

所以说：一个应用程序所需的Container分为两大类，如下：

（1） 运行ApplicationMaster的Container：这是由ResourceManager（向内部的资源调度器）申请和启动的，用户提交应用程序时，可指定唯一的ApplicationMaster所需的资源；

（2） 运行各类任务的Container：这是由ApplicationMaster向ResourceManager申请的，并由ApplicationMaster与NodeManager通信以启动之。

以上两类Container可能在任意节点上，它们的位置通常而言是随机的，即ApplicationMaster可能不与它管理的任务运行在一个节点上。

所以ResourceManager接收到一个应用程序的客户请求后，协商一个容器的必要资源，启动一个ApplicationMaster来表示已经提交的应用程序。ApplicationMaster开始管理该应用程序的执行，也就是该应用程序内多个任务的执行。ApplicationMaster向ResourceManager请求每个任务所需的资源，ResourceManager分配处于多个节点撒上的资源后，ApplicationMaster协商每个节点上供该应程序使用的资源容器。执行应用程序时，ApplicationMaster监视资源容器，直到完成。当应用程序完成时，ApplicationMaster从ResourceManager注销其容器，执行周期就完成了。

YARN的工作原理

[![image](/images/2018\05\dsj\1569798685062.jpg "image")]

**来总结以下yarn的功能：**

yarn的两个部分：资源管理、任务调度。

资源管理需要一个全局的ResourceManager(RM)和分布在每台机器上的NodeManager协同工作，RM负责资源的仲裁，NodeManager负责每个节点的资源监控、状态汇报和Container的管理

任务调度也需要ResourceManager负责任务的接受和调度，在任务调度中，在Container中启动的ApplicationMaster(AM)负责这个任务的管理，当任务需要资源时，会向RM申请，分配到的Container资源用来做任务，然后AM和这些Container做通信，管理任务的运行，AM和具体执行的任务都是在Container中执行的。

一个应用程序的运行过程如下：

**步骤1** ：用户将应用程序提交到ResourceManager上；

**步骤2** ：ResourceManager并与某个NodeManager通信，在节点的container中启动负责该应用程序的ApplicationMaster；

**步骤3** ：ApplicationMaster与ResourceManager通信，为内部要执行的任务（一个应用程序包含多个任务）申请资源，一旦得到资源后，将与NodeManager通信，以启动对应的任务。

**步骤4** ：所有任务运行完成后，ApplicationMaster向ResourceManager注销，整个应用程序运行结束。

ApplicationMaster当向ResourceManager申请资源，需向它发送一个ResourceRequest列表，其中，每个ResourceRequest描述了一个资源单元的详细需求，而ResourceManager则为之返回分配到的资源描述Container。每个ResourceRequest可看做一个可序列化Java对象，包含的字段信息（直接给出了Protocol Buffers定义）如下：

    message ResourceRequestProto {

    optional PriorityProto priority = 1; // 资源优先级

    optional string resource\_name = 2; // 资源名称（期望资源所在的host、rack名称等）

    optional ResourceProto capability = 3; // 资源量（仅支持CPU和内存两种资源）

    optional int32 num\_containers = 4; // 满足以上条件的资源个数

    optional bool relax\_locality = 5 [default = true];  //是否支持本地性松弛（2.1.0-beta之后的版本新增加的，具体参考我的这篇文章：Hadoop新特性、改进、优化和Bug分析系列3：YARN-392）

    }

1. 1
2. 2
3. 3
4. 4
5. 5
6. 6
7. 7
8. 8
9. 9
10. 10
11. 11
12. 12
13. 13

通过上面的信息也看出了，资源不一定在当前主机上。可以为应用程序申请任意大小的资源量（CPU和内存），且默认情况下资源是本地性松弛的，即申请优先级为10，资源名称为&quot;node11&quot;，资源量为\&lt;2GB, 1cpu\&gt;的5份资源时，如果节点node11上没有满足要求的资源，则优先找node11同一机架上其他节点上满足要求的资源，如果仍找不到，则找其他机架上的资源。而如果你一定要node11上的节点，则将relax\_locality置为false。

发出资源请求后，资源调度器并不会立马为它返回满足要求的资源，而需要应用程序的ApplicationMaster不断与ResourceManager通信，探测分配到的资源，并拉取过来使用。一旦分配到资源后，ApplicatioMaster可从资源调度器那获取以Container表示的资源，Container可看做一个可序列化Java对象，包含的字段信息（直接给出了Protocol Buffers定义）如下：

    message ContainerProto {

    optional ContainerIdProto id = 1; //container id

    optional NodeIdProto nodeId = 2; //container（资源）所在节点

    optional string node\_http\_address = 3;

    optional ResourceProto resource = 4; //container资源量

    optional PriorityProto priority = 5; //container优先级

    optional hadoop.common.TokenProto container\_token = 6; //container token，用于安全认证

    }

1. 1
2. 2
3. 3
4. 4
5. 5
6. 6
7. 7
8. 8
9. 9
10. 10
11. 11
12. 12
13. 13
14. 14
15. 15

一般而言，每个Container可用于运行一个任务。ApplicationMaster收到一个或多个Container后，再次将该Container进一步分配给内部的某个任务，一旦确定该任务后，ApplicationMaster需将该任务运行环境（包含运行命令、环境变量、依赖的外部文件等）连同Container中的资源信息封装到ContainerLaunchContext对象中，进而与对应的NodeManager通信，以启动该任务。ContainerLaunchContext包含的字段信息（直接给出了Protocol Buffers定义）如下：

    message ContainerLaunchContextProto {

    repeated StringLocalResourceMapProto localResources = 1; //Container启动以来的外部资源

    optional bytes tokens = 2;

    repeated StringBytesMapProto service\_data = 3;

    repeated StringStringMapProto environment = 4; //Container启动所需的环境变量

    repeated string command = 5; //Container内部运行的任务启动命令，如果是MapReduce的话，Map/Reduce Task启动命令就在该字段中

    repeated ApplicationACLMapProto application\_ACLs = 6;

    }

1. 1
2. 2
3. 3
4. 4
5. 5
6. 6
7. 7
8. 8
9. 9
10. 10
11. 11
12. 12
13. 13
14. 14
15. 15

每个ContainerLaunchContext和对应的Container信息（被封装到了ContainerToken中）将再次被封装到StartContainerRequest中，也就是说，ApplicationMaster最终发送给NodeManager的是StartContainerRequest，每个StartContainerRequest对应一个Container和任务。

**hdfs分布式存储架构**

HDFS即Hadoop Distributed File System分布式文件系统，它的设计目标是把超大数据集存储到分布在网络中的多台普通商用计算机上，并且能够提供高可靠性和高吞吐量的服务。分布式文件系统要比普通磁盘文件系统复杂，因为它要引入网络编程，分布式文件系统要容忍节点故障也是一个很大的挑战。

**设计前提和目标**

专为存储超大文件而设计：hdfs应该能够支持GB级别大小的文件；它应该能够提供很大的数据带宽并且能够在集群中拓展到成百上千个节点；它的一个实例应该能够支持千万数量级别的文件。
适用于流式的数据访问：hdfs适用于批处理的情况而不是交互式处理；它的重点是保证高吞吐量而不是低延迟的用户响应
容错性：完善的冗余备份机制
支持简单的一致性模型：HDFS需要支持一次写入多次读取的模型，而且写入过程文件不会经常变化
移动计算优于移动数据：HDFS提供了使应用计算移动到离它最近数据位置的接口
兼容各种硬件和软件平台

1. 1
2. 2
3. 3
4. 4
5. 5
6. 6

**不适合的场景**

大量小文件：文件的元数据都存储在NameNode内存中，大量小文件会占用大量内存。
低延迟数据访问：hdfs是专门针对高数据吞吐量而设计的
多用户写入，任意修改文件

1. 1
2. 2
3. 3

**hdfs架构设计**

我们看一下hdfs的架构：hdfs部分由NameNode、SecondaryNameNode和DataNode组成。DataNode是真正的在每个存储节点上管理数据的模块，NameNode是对全局数据的名字信息做管理的模块，SecondaryNameNode是它的从节点，以防挂掉。HSFS是以master/slave模式运行的，其中NameNode、SecondaryNameNode 运行在master节点，DataNode运行slave节点。

**数据块**

磁盘数据块是磁盘读写的基本单位，与普通文件系统类似，hdfs也会把文件分块来存储。hdfs默认数据块大小为64MB，磁盘块一般为512B，hdfs块为何如此之大呢？块增大可以减少寻址时间与文件传输时间的比例，若寻址时间为10ms，磁盘传输速率为100MB/s，那么寻址与传输比仅为1%。当然，磁盘块太大也不好，因为一个MapReduce通常以一个块作为输入，块过大会导致整体任务数量过小，降低作业处理速度。

数据块是存储在DataNode中的，为了能够容错数据块是以多个副本的形式分布在集群中的，副本数量默认为3，后面会专门介绍数据块的复制机制。

hdfs按块存储还有如下好处：

文件可以任意大，也不用担心单个结点磁盘容量小于文件的情况
简化了文件子系统的设计，子系统只存储文件块数据，而文件元数据则交由其它系统（NameNode）管理
有利于备份和提高系统可用性，因为可以以块为单位进行备份，hdfs默认备份数量为3。
有利于负载均衡

1. 1
2. 2
3. 3
4. 4

**NameNode**

当一个客户端请求一个文件或者存储一个文件时，它需要先知道具体到哪个DataNode上存取，获得这些信息后，客户端再直接和这个DataNode进行交互，而这些信息的维护者就是NameNode。

NameNode管理着文件系统命名空间，它维护着文件系统树及树中的所有文件和目录。NameNode也负责维护所有这些文件或目录的打开、关闭、移动、重命名等操作。对于实际文件数据的保存与操作，都是由DataNode负责。当一个客户端请求数据时，它仅仅是从NameNode中获取文件的元信息，而具体的数据传输不需要经过NameNode，是由客户端直接与相应的DataNode进行交互。

NameNode保存元信息的种类有：

文件名目录名及它们之间的层级关系
文件目录的所有者及其权限
每个文件块的名及文件有哪些块组成

1. 1
2. 2
3. 3

需要注意的是，NameNode元信息并不包含每个块的位置信息，这些信息会在NameNode启动时从各个DataNode获取并保存在内存中，因为这些信息会在系统启动时由数据节点重建。把块位置信息放在内存中，在读取数据时会减少查询时间，增加读取效率。NameNode也会实时通过心跳机制和DataNode进行交互，实时检查文件系统是否运行正常。不过NameNode元信息会保存各个块的名称及文件由哪些块组成。

一般来说，一条元信息记录会占用200byte内存空间。假设块大小为64MB，备份数量是3 ，那么一个1GB大小的文件将占用16_3=48个文件块。如果现在有1000个1MB大小的文件，则会占用1000_3=3000个文件块（多个文件不能放到一个块中）。我们可以发现，如果文件越小，存储同等大小文件所需要的元信息就越多，所以，Hadoop更喜欢大文件。

**元信息的持久化**

在NameNode中存放元信息的文件是 fsimage。在系统运行期间所有对元信息的操作都保存在内存中并被持久化到另一个文件edits中。并且edits文件和fsimage文件会被SecondaryNameNode周期性的合并

**其它问题**

运行NameNode会占用大量内存和I/O资源，一般NameNode不会存储用户数据或执行MapReduce任务。

为了简化系统的设计，Hadoop只有一个NameNode，这也就导致了hadoop集群的单点故障问题。因此，对NameNode节点的容错尤其重要，hadoop提供了如下两种机制来解决：

1. 将hadoop元数据写入到本地文件系统的同时再实时同步到一个远程挂载的网络文件系统（NFS）。
2. 运行一个secondary NameNode，它的作用是与NameNode进行交互，定期通过编辑日志文件合并命名空间镜像，当NameNode发生故障时它会通过自己合并的命名空间镜像副本来恢复。需要注意的是secondaryNameNode保存的状态总是滞后于NameNode，所以这种方式难免会导致丢失部分数据（后面会详细介绍）。

**DataNode**

DataNode是hdfs中的worker节点，它负责存储数据块，也负责为系统客户端提供数据块的读写服务，同时还会根据NameNode的指示来进行创建、删除、和复制等操作。此外，它还会通过心跳定期向NameNode发送所存储文件块列表信息。当对hdfs文件系统进行读写时，NameNode告知客户端每个数据驻留在哪个DataNode，客户端直接与DataNode进行通信，DataNode还会与其它DataNode通信，复制这些块以实现冗余。

NameNode和DataNode架构图

[![image](/images/2018\05\dsj\1569798685067.jpg "image")]

**SecondaryNameNode**

需要注意，SecondaryNameNode并不是NameNode的备份。我们从前面的介绍已经知道，所有HDFS文件的元信息都保存在NameNode的内存中。在NameNode启动时，它首先会加载fsimage到内存中，在系统运行期间，所有对NameNode的操作也都保存在了内存中，同时为了防止数据丢失，这些操作又会不断被持久化到本地edits文件中。

Edits文件存在的目的是为了提高系统的操作效率，NameNode在更新内存中的元信息之前都会先将操作写入edits文件。在NameNode重启的过程中，edits会和fsimage合并到一起，但是合并的过程会影响到Hadoop重启的速度，SecondaryNameNode就是为了解决这个问题而诞生的。

SecondaryNameNode的角色就是定期的合并edits和fsimage文件，我们来看一下合并的步骤：

1. 合并之前告知NameNode把所有的操作写到新的edites文件并将其命名为edits.new。
2. SecondaryNameNode从NameNode请求fsimage和edits文件
3. SecondaryNameNode把fsimage和edits文件合并成新的fsimage文件
4. NameNode从SecondaryNameNode获取合并好的新的fsimage并将旧的替换掉，并把edits用第一步创建的edits.new文件替换掉
5. 更新fstime文件中的检查点

最后再总结一下整个过程中涉及到NameNode中的相关文件

1. fsimage ：保存的是上个检查点的HDFS的元信息
2. edits ：保存的是从上个检查点开始发生的HDFS元信息状态改变信息
3. fstime：保存了最后一个检查点的时间戳

**MapReduce分布式计算架构**

**MapReduce特点：**

1. 易于编程，用户通常情况下只需要编写Mapper和Reducer程序即可。
2. 良好的扩展性，即可以很容易的增加节点
3. 高容错性，一个Job默认情况下会尝试启动两次，一个mapper或者reducer默认会尝试4次，如果一个节点挂了，可以向系统申请新的节点来执行这个mapper或者reducer
4. 适合PB级别的数据的离线处理

**MapReduce框架的缺点**

1. 不擅长实时计算，像MySQL一样能够立即返回结果
2. MapReduce的设计本身决定了处理的数据必须是离线数据，因为涉及到数据切分等等。
3. 不擅长DAG（有向图）计算，需要一个Job执行完成之后，另一个Job才能使用他的输出。

**MapReduce编程模型：**

一种分布式计算模型框架，解决海量数据的计算问题

MapReduce将整个并行计算过程抽象到两个函数

1. Map（映射）：对一些独立元素组成的列表的每一个元素进行指定的操作，可以高度并行。
2. Reduce（化简）：对一个列表的元素进行合并。

例如wordcount功能：

Map阶段：首先将输入数据进行分片，然后对每一片数据执行Mapper程序，计算出每个词的个数，之后对计算结果进行分组，每一组由一个Reducer程序进行处理，到此Map阶段完成。

Reduce阶段：每个Reduce程序从Map的结果中拉取自己要处理的分组（叫做Shuffling过程），进行汇总和排序（桶排序），对排序后的结果运行Reducer程序，最后所有的Reducer结果进行规约写入HDFS。

一个简单的MapReduce程序只需要指定map()、reduce()、input和output，剩下的事由架构完成。

input()——\&gt;map()——\&gt;reduce()——\&gt;output()

每个应用程序称为一个作业（Job），每个Job由一系列的Mappers和Reducers来完成。 每个Mapper处理一个分片（Split），处理过程如下：

Map阶段：

1. 输入数据的解析：InputFormat
2. 输入数据处理：Mapper
3. 输入分组：Partitioner
4. 本节点的规约：Combiner ，

Reduce阶段：

1. Shuffling阶段拉取数据
2. 桶排序，是一个hash过程，使得相同的Key可以排在一堆
3. 数据规约：Reducer
4. 数据输出格式： OutputFormat

**MapReduce2.0 架构**

[![image](/images/2018\05\dsj\1569798685069.jpg "image")]

MapReduce2.0运行在YARN之上。YARN由ResourceManager（RM） 和NodeManager（NM）两大块组成。

**MapReduce2 架构设计：**

1:用户向YARN中提交应用程序，其中包括ApplicationMaster程序、启动ApplicationMaster的命令、用户程序等。

2:ResourceManager为该应用程序分配第一个Container，并与对应的Node-Manager通信，要求它在这个Container中启动应用程序的ApplicationMaster。

3:ApplicationMaster首先向ResourceManager注册，这样用户可以直接通过ResourceManage查看应用程序的运行状态，然后它将为各个任务申请资源，并监控它的运行状态，直到运行结束，即重复步骤4~7。

4:ApplicationMaster采用轮询的方式通过RPC协议向ResourceManager申请和领取资源。

5:一旦ApplicationMaster申请到资源后，便与对应的NodeManager通信，要求它启动任务。

6:NodeManager为任务设置好运行环境（包括环境变量、JAR包、二进制程序等）后，将任务启动命令写到一个脚本中，并通过运行该脚本启动任务。

7:各个任务通过某个RPC协议向ApplicationMaster汇报自己的状态和进度，以让ApplicationMaster随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务。在应用程序运行过程中，用户可随时通过RPC向ApplicationMaster查询应用程序的当前运行状态。

8:应用程序运行完成后，ApplicationMaster向ResourceManager注销并关闭自己。

**MapReduce1 的架构设计：**

Client: 客户端

JobTracker : 主要负责 资源监控管理和作业调度。

a.监控所有TaskTracker 与job的健康状况,一旦发现失败,就将相应的任务转移到其他节点;

b.同时JobTracker会跟踪任务的执行进度、资源使用量等信息,并将这些信息告诉任务调度器,而调度器会在资源出现空闲时,选择合适的任务使用这些资源.

TaskTracker: :是JobTracker与Task之前的桥梁

a.从JobTracker接收并执行各种命令:运行任务、提交任务、Kill任务、重新初始化任务;

b.周期性地通过心跳机制,将节点健康情况和资源使用情况、各个任务的进度和状态等汇报给JobTracker

Task Scheduler: 任务调度器(默认 FIFO,先按照作业的优先级高低,再按照到达时间的先后选择被执行的作业)

Map Task: 映射任务

Reduce Task: 归约任务

**MapReduce实例——WordCount**

问题：

有一批文件（规模为TB级或者PB级），如何统计这些文件中所有单词出现的次数。

方案：

首先，分别统计每个文件中单词出现的次数。

然后，累加不同文件中同一个单词出现的次数。

**MapReduce WordCount实例运行**

在dfs中创建input目录

[email protected] data]# hadoop fs -mkdir /wc/input

1. 1

将data中的.data文件拷贝到dfs中的input

[email protected] data]# hadoop fs -put ./\*.data /wc/input

1. 1

查看

[email protected] data]# hadoop fs -ls /wc/input

1. 1

运行wordcount

[email protected] hadoop-2.7.3]# hadoop jar hadoop-examples-1.2.1.jar wordcount /wc/input /wc/output

1. 1

**MapReduce基本流程**

首先，将数据输入到HDFS，再交给map，map对每个单词进行统计

在map之后reduce之前进行排序

然后，将排好序的数据拷贝并进行合并，合并好的数据交给reduce，reduce再将完成的数据输出回HDFS

[![image](/images/2018\05\dsj\1569798685075.jpg "image")]

[![image](/images/2018\05\dsj\1569798685078.jpg "image")]

**MapReduce执行流程**

**Map任务处理**

1，读取输入文件内容，解析成key、value对。对输入文件的每一行，解析成key、value对。每一个键值对调用一次map函数。

2，写自己的逻辑，对输入的key、value处理，转换成新的key、value输出。

3，对输出的key、value进行分区

4、对不同分区的数据，按照key进行排序、分组。相同key的value放到一个集合中。

5、（可选）分组后的数据进行归约。

**Reduce任务处理**

1，对多个map任务的输出，按照不同的分区，通过网络copy到不同的reduce节点。

2，对多个map任务的输出进行合并、排序。写reduce函数自己的逻辑，对输入的key、value处理，转换成新的key、value输出。

3、把reduce的输出保存到文件中。

**编写MapReduce程序**

基于MapReduce计算模型编写分布式并行程序非常简单，程序员的主要编码工作就是实现map和reduce函数。

MapReduce中，map和reduce函数遵循如下常规格式：

map：（K1，V1）——\&gt;list（K2，V2）
reduce：（K2，list（V2）） ——\&gt;list（K3，V3）

1. 1
2. 2

Mapper的接口：

protected void reduce(KEY key,Iterable\&lt;VALUE\&gt;values,Context context) throws IOException,interruptedException {

}

1. 1
2. 2
3. 3

Reduce的接口：

protected void reduce(KEY key,Iterable\&lt;VALUE\&gt;values,Context context) throws IOException,interruptedException {

}

1. 1
2. 2
3. 3

**Spark相对于MapReduce的优势**

**MapReduce存在的问题**

1. MapReduce框架局限性

1）仅支持Map和Reduce两种操作

2）处理效率低效。

a）Map中间结果写磁盘，Reduce写HDFS，多个MR之间通过HDFS交换数据; 任务调度和启动开销大;

b）无法充分利用内存

c）Map端和Reduce端均需要排序

3）不适合迭代计算(如机器学习、图计算等)，交互式处理(数据挖掘) 和流式处理(点击日志分析)

1. MapReduce编程不够灵活

1）尝试scala函数式编程语言

**Spark**

1. 高效(比MapReduce快10~100倍)

1）内存计算引擎，提供Cache机制来支持需要反复迭代计算或者多次数据共享，减少数据读取的IO开销

2）DAG引擎，减少多次计算之间中间结果写到HDFS的开销

3）使用多线程池模型来减少task启动开稍，shuffle过程中避免 不必要的sort操作以及减少磁盘IO操作

1. 易用

1）提供了丰富的API，支持Java，Scala，Python和R四种语言

2）代码量比MapReduce少2~5倍

1. 与Hadoop集成 读写HDFS/Hbase 与YARN集成

**spark应用执行机制分析**

目前Apache Spark支持三种分布式部署方式，分别是standalone、spark on mesos和 spark on YARN还有最新的spark on k8s，其中，第一种类似于MapReduce 1.0所采用的模式，内部实现了容错性和资源管理，后两种则是未来发展的趋势，部分容错性和资源管理交由统一的资源管理系统完成：让Spark运行在一个通用的资源管理系统之上，这样可以与其他计算框架，比如MapReduce，公用一个集群资源，最大的好处是降低运维成本和提高资源利用率（资源按需分配）。本文将介绍前三种部署方式，并比较其优缺点。 支持k8s原生的spark部署方式可以参考:[https://blog.csdn.net/luanpeng825485697/article/details/83651742](https://blog.csdn.net/luanpeng825485697/article/details/83651742)

**1. Standalone模式**

即独立模式，自带完整的服务，可单独部署到一个集群中，无需依赖任何其他资源管理系统。从一定程度上说，该模式是其他两种的基础。借鉴Spark开发模式，我们可以得到一种开发新型计算框架的一般思路：先设计出它的standalone模式，为了快速开发，起初不需要考虑服务（比如master/slave）的容错性，之后再开发相应的wrapper，将stanlone模式下的服务原封不动的部署到资源管理系统yarn或者mesos上，由资源管理系统负责服务本身的容错。目前Spark在standalone模式下是没有任何单点故障问题的，这是借助zookeeper实现的，思想类似于Hbase master单点故障解决方案。将Spark standalone与MapReduce比较，会发现它们两个在架构上是完全一致的：

1.
  1. 都是由master/slaves服务组成的，且起初master均存在单点故障，后来均通过zookeeper解决（Apache MRv1的JobTracker仍存在单点问题，但CDH版本得到了解决）；

1.
  1. 各个节点上的资源被抽象成粗粒度的slot，有多少slot就能同时运行多少task。不同的是，MapReduce将slot分为map slot和reduce slot，它们分别只能供Map Task和Reduce Task使用，而不能共享，这是MapReduce资源利率低效的原因之一，而Spark则更优化一些，它不区分slot类型，只有一种slot，可以供各种类型的Task使用，这种方式可以提高资源利用率，但是不够灵活，不能为不同类型的Task定制slot资源。总之，这两种方式各有优缺点。

流程：

1、使用SparkSubmit提交任务的时候(包括Eclipse或者其它开发工具使用new SparkConf()来运行任务的时候)，Driver运行在Client；使用SparkShell提交的任务的时候，Driver是运行在Master上

2、使用SparkSubmit提交任务的时候，使用本地的Client类的main函数来创建sparkcontext并初始化它；

3、SparkContext连接到Master，注册并申请资源（内核和内存）。

4、Master根据SC提出的申请，根据worker的心跳报告，来决定到底在那个worker上启动StandaloneExecutorBackend（executor）

5、executor向SC注册

6、SC将应用分配给executor，

7、SC解析应用，创建DAG图，提交给DAGScheduler进行分解成stage(当出发action操作的时候，就会产生job，每个job中包含一个或者多个stage，stage一般在获取外部数据或者shuffle之前产生)。然后stage（又称为Task Set）被发送到TaskScheduler。TaskScheduler负责将stage中的task分配到相应的worker上，并由executor来执行

8、executor创建Executor线程池，开始执行task，并向SC汇报

9、所有的task执行完成之后，SC向Master注销

[![image](/images/2018\05\dsj\1569798685084.jpg "image")]

**2. Spark On Mesos模式**

这是很多公司采用的模式，官方推荐这种模式（当然，原因之一是血缘关系）。正是由于Spark开发之初就考虑到支持Mesos，因此，目前而言，Spark运行在Mesos上会比运行在YARN上更加灵活，更加自然。目前在Spark On Mesos环境中，用户可选择两种调度模式之一运行自己的应用程序（可参考Andrew Xia的&quot;Mesos Scheduling Mode on Spark&quot;）：

1) 粗粒度模式（Coarse-grained Mode）：每个应用程序的运行环境由一个Dirver和若干个Executor组成，其中，每个Executor占用若干资源，内部可运行多个Task（对应多少个&quot;slot&quot;）。应用程序的各个任务正式运行之前，需要将运行环境中的资源全部申请好，且运行过程中要一直占用这些资源，即使不用，最后程序运行结束后，回收这些资源。举个例子，比如你提交应用程序时，指定使用5个executor运行你的应用程序，每个executor占用5GB内存和5个CPU，每个executor内部设置了5个slot，则Mesos需要先为executor分配资源并启动它们，之后开始调度任务。另外，在程序运行过程中，mesos的master和slave并不知道executor内部各个task的运行情况，executor直接将任务状态通过内部的通信机制汇报给Driver，从一定程度上可以认为，每个应用程序利用mesos搭建了一个虚拟集群自己使用。

2) 细粒度模式（Fine-grained Mode）：鉴于粗粒度模式会造成大量资源浪费，Spark On Mesos还提供了另外一种调度模式：细粒度模式，这种模式类似于现在的云计算，思想是按需分配。与粗粒度模式一样，应用程序启动时，先会启动executor，但每个executor占用资源仅仅是自己运行所需的资源，不需要考虑将来要运行的任务，之后，mesos会为每个executor动态分配资源，每分配一些，便可以运行一个新任务，单个Task运行完之后可以马上释放对应的资源。每个Task会汇报状态给Mesos slave和Mesos Master，便于更加细粒度管理和容错，这种调度模式类似于MapReduce调度模式，每个Task完全独立，优点是便于资源控制和隔离，但缺点也很明显，短作业运行延迟大。

**3. Spark On YARN模式**

这是一种很有前景的部署模式。但限于YARN自身的发展，目前仅支持粗粒度模式（Coarse-grained Mode）。这是由于YARN上的Container资源是不可以动态伸缩的，一旦Container启动之后，可使用的资源不能再发生变化，不过这个已经在YARN计划中了。

spark on yarn 的支持两种模式：

　　1) yarn-cluster：适用于生产环境；

　　2) yarn-client：适用于交互、调试，希望立即看到app的输出

yarn-cluster和yarn-client的区别在于yarn ApplicationMaster，每个yarn app实例有一个ApplicationMaster进程，是为app启动的第一个container；负责从ResourceManager请求资源，获取到资源后，告诉NodeManager为其启动container。yarn-cluster和yarn-client模式内部实现还是有很大的区别。如果你需要用于生产环境，那么请选择yarn-cluster；而如果你仅仅是Debug程序，可以选择yarn-client。

**Spark运行模式列表（一定要熟悉！）**

[![image](/images/2018\05\dsj\1569798685086.jpg "image")]

注意： Spark on Yarn 有 yarn client 和 yarn clusters 模式。

Spark on Standalone 也有 standalone client 和 standalone clusters 模式。

yarn client流程

1、spark-submit脚本提交，Driver在客户端本地运行；

2、Client向RM申请启动AM，同时在SC（client上）中创建DAGScheduler和TaskScheduler。

3、RM收到请求之后，查询NM并选择其中一个，分配container，并在container中开启AM

4、client中的SC初始化完成之后，与AM进行通信，向RM注册，根据任务信息向RM申请资源

5、AM申请到资源之后，与AM进行通信，要求在它申请的container中开启CoarseGrainedExecutorBackend(executor)。Executor在启动之后会向SC注册并申请task

6、SC分配task给executor，executor执行任务并向Driver（运行在client之上的）汇报，以便客户端可以随时监控任务的运行状态

7、任务运行完成之后，client的SC向RM注销自己并关闭自己

[![image](/images/2018\05\dsj\1569798685089.jpg "image")]

yarn cluster流程

1、spark-submit脚本提交，向yarn（RM）中提交ApplicationMaster程序、AM启动的命令和需要在Executor中运行的程序等

2、RM收到请求之后，选择一个NM，在其上开启一个container，在container中开启AM，并在AM中完成SC的初始化

3、SC向RM注册并请求资源，这样用户可以在RM中查看任务的运行情况。RM根据请求采用轮询的方式和RPC协议向各个NM申请资源并监控任务的运行状况直到结束

4、AM申请到资源之后，与对应的NM进行通信，要求在其上获取到的Container中开启CoarseGrainedExecutorBackend(executor),executor 开启之后，向AM中的SC注册并申请task

5、AM中的SC分配task给executor，executor运行task兵向AM中的SC汇报自己的状态和进度

6、应用程序完成之后（各个task都完成之后），AM向RM申请注销自己兵关闭自己

[![image](/images/2018\05\dsj\1569798685092.jpg "image")]

**HIVE和HBASE区别**

Hive中的表是纯逻辑表，就只是表的定义等，即表的元数据。Hive本身不存储数据，它完全依赖HDFS和MapReduce。这样就可以将结构化的数据文件映射为为一张数据库表，并提供完整的SQL查询功能，并将SQL语句最终转换为MapReduce任务进行运行。 而HBase表是物理表，适合存放非结构化的数据。

**1. 两者分别是什么？**

Apache Hive是数据仓库。通过Hive可以使用HQL语言查询存放在HDFS上的数据。HQL是一种类SQL语言，这种语言最终被转化为Map/Reduce. 虽然Hive提供了SQL查询功能，但是Hive不能够进行交互查询–因为它是基于MapReduce算法。

Apache Hbase Key/Value，基础单元是cell，它运行在HDFS之上。和Hive不一样，Hbase的能够在它的数据库上实时运行，而不是运行MapReduce任务，。

**2. 两者的特点**

Hive帮助熟悉SQL的人运行MapReduce任务。因为它是JDBC兼容的。运行Hive查询会花费很长时间，因为它会默认遍历表中所有的数据。但可以通过Hive的分区来控制。因为这样一来文件大小是固定的，就这么大一块存储空间，从固定空间里查数据是很快的。

HBase通过存储key/value来工作。注意版本的功能。

**3. 限制**

Hive目前不支持更新操作。另外，由于hive在hadoop上运行批量操作，它需要花费很长的时间，通常是几分钟到几个小时才可以获取到查询的结果。Hive必须提供预先定义好的schema将文件和目录映射到列，并且Hive与ACID不兼容。

HBase查询是通过特定的语言来编写的，这种语言需要重新学习。类SQL的功能可以通过Apache Phonenix实现，但这是以必须提供schema为代价的。另外，Hbase也并不是兼容所有的ACID特性，虽然它支持某些特性。最后但不是最重要的–为了运行Hbase，Zookeeper是必须的，zookeeper是一个用来进行分布式协调的服务，这些服务包括配置服务，维护元信息和命名空间服务。

**4. 应用场景**

Hive适合用来对一段时间内的数据进行分析查询，例如，用来计算趋势或者网站的日志。Hive不应该用来进行实时的查询。因为它需要很长时间才可以返回结果。

Hbase非常适合用来进行大数据的实时查询。Facebook用Hbase进行消息和实时的分析。它也可以用来统计Facebook的连接数。

**5. 总结**

Hive和Hbase是两种基于Hadoop的不同技术–Hive是一种类SQL的引擎，并且运行MapReduce任务，Hbase是一种在Hadoop之上的NoSQL 的Key/vale数据库。当然，这两种工具是可以同时使用的。就像用Google来搜索，用FaceBook进行社交一样，Hive可以用来进行统计查询，HBase可以用来进行实时查询，数据也可以从Hive写到Hbase，设置再从Hbase写回Hive。

**HBASE架构**

HBase由三个部分，如下

**1. HMaster**

对Region进行负载均衡，分配到合适的HRegionServer

**2. ZooKeeper**

选举HMaster，对HMaster，HRegionServer进行心跳检测（貌似是这些机器节点向ZooKeeper上报心跳）

**3. HRegionServer**

数据库的分片，HRegionServer上的组成部分如下:

Region：HBase中的数据都是按row-key进行排序的，对这些按row-key排序的数据进行水平切分，每一片称为一个Region，它有startkey和endkey，Region的大小可以配置，一台RegionServer中可以放多个Region

CF：列族。一个列族中的所有列存储在相同的HFile文件中

HFile：HFile就是Hadoop磁盘文件，一个列族中的数据保存在一个或多个HFile中，这些HFile是对列族的数据进行水平切分后得到的。

MemStore：HFile在内存中的体现。当我们update/delete/create时，会先写MemStore，写完后就给客户端response了，当Memstore达到一定大小后，会将其写入磁盘，保存为一个新的HFile。HBase后台会对多个HFile文件进行merge，合并成一个大的HFile

**Hbase 架构的组件**

1. Region Server：提供数据的读写服务，当客户端访问数据时，直接和Region Server通信。
2. HBase Master:Region的分配,.DDL操作（创建表,删除表）
3. Zookeeper:分布式管理工具，维护一个活跃的集群状态

Hadoop DataNode存储着Region Server 管理的数据，所有的Hbase数据存储在HDFS文件系统中，Region Servers在HDFS DataNode中是可配置的，并使数据存储靠近在它所需要的地方，就近服务，当王HBASE写数据时是Local的，但是当一个region 被移动之后，Hbase的数据就不是Local的，除非做了压缩（compaction）操作。NameNode维护物理数据块的元数据信息。

[![image](/images/2018\05\dsj\1569798685095.jpg "image")]

**Regions**

HBase Tables 通过行健的范围（row key range）被水平切分成多个Region, 一个Region包含了所有的，在Region开始键和结束之内的行，Regions被分配到集群的节点上，成为 Region Servers,提供数据的读写服务，一个region server可以服务1000 个Region。

[![image](/images/2018\05\dsj\1569798685104.jpg "image")]

**三.HBase HMaster**

分配Region,DDL操作（创建表， 删除表）

协调各个Reion Server ：

    -在启动时分配Region、在恢复或是负载均衡时重新分配Region。

    -监控所有集群当中的Region Server实例，从ZooKeeper中监听通知。

1. 1
2. 2
3. 3

管理功能：

    -提供创建、删除、更新表的接口。

1. 1

[![image](/images/2018\05\dsj\1569798685111.jpg "image")]

**ZooKeeper：协调器**

Hbase使用Zookeeper作为分布式协调服务，来维护集群中的Server状态，ZooKeeper维护着哪些Server是活跃或是可用的。提供Server 失败时的通知。Zookeeper使用一致性机制来保证公共的共享状态，注意，需要使用奇数的三台或是五台机器，保证一致。

[![image](/images/2018\05\dsj\1569798685114.jpg "image")]

**组件之间如何工作**

Zookeeper一般在分布式系统中的成员之间协调共享的状态信息，Region Server和活跃的HMaster通过会话连接到Zookeeper，ZooKeeper维护短暂的阶段，通过心跳机制用于活跃的会话。

[![image](/images/2018\05\dsj\1569798685118.jpg "image")]

每个Region Server创建一个短暂的节点，HMaster监控这些节点发现可用的Region Server，同时HMaster 也监控这些节点的服务器故障。HMaster 通过撞见一个临时的节点，Zookeeper决定其中一个HMaster作为活跃的。活跃的HMaster 给ZooKeeper发送心跳信息，不活跃的HMaster在活跃的HMaster出现故障时，接受通知。

如果一个Region Server或是一个活跃的HMaster在发送心跳信息时失败或是出现了故障，则会话过期，相应的临时节点将被删除，监听器将因这些删除的节点更新通知信息，活跃的HMaster将监听Region Server，并且将会恢复出现故障的Region Server，不活跃的HMaster 监听活跃的HMaster故障，如果一个活跃的HMaster出现故障，则不活跃的HMaster将会变得活跃。

**Hbase META表**

有一个特殊的Hbase 目录表叫做Meta表，它拥有Region 在集群中的位置信息，ZooKeeper存储着Meta表的位置。

表结构

[![image](/images/2018\05\dsj\1569798685122.jpg "image")]

我们来仔细分析一下这个结构，每条Row记录了一个Region的信息。

首先是RowKey，RowKey由三部分组成：TableName, StartKey 和 TimeStamp。RowKey存储的内容我们又称之为Region的Name。将组成RowKey的三个部分用逗号连接就构成了整个RowKey，这里TimeStamp使用十进制的数字字符串来表示的.

然后是表中最主要的Family：info，info里面包含三个Column：regioninfo, server, serverstartcode。其中regioninfo就是Region的详细信息，包括StartKey, EndKey 以及每个Family的信息等等。server存储的就是管理这个Region的RegionServer的地址。

所以当Region被拆分、合并或者重新分配的时候，都需要来修改这张表的内容。

META 表包含集群中所有Region的列表

.META. 表像是一个B树

.META. 表结构为：

1. Key: region start key,region id
2. Values: Region 和 RegionServer

[![image](/images/2018\05\dsj\1569798685124.jpg "image")]

**Hbase 的首次读与写**

如下就是客户端首次读写Hbase 所发生的事情：

现在假设我们要从Table2里面插寻一条RowKey是RK10000的数据。那么我们应该遵循以下步骤：

1.客户端从Zookeeper查询到meta表的位置,然后在Meta表中查询哪个Region包含这条数据, 进而获取管理这个Region的RegionServer地址。(每个Region Server管理着不同的Region),然后和Region Server进行通信

2.客户端将查询 .META.服务器，获取它想访问的相对应的Region Server的行健。客户端将缓存这些信息以及META 表的位置。

3.客户端将从相应的Region Server获取行。

如果再次读取，客户端将使用缓存来获取META 的位置及之前的行健。这样时间久了，客户端不需要查询META表，除非Region 移动所导致的丢失，这样的话，则将会重新查询更新缓存。

[![image](/images/2018\05\dsj\1569798685128.jpg "image")]

**Region Server 的组件**

Region Server 运行在HDFS DataNode上，并有如下组件：

WAL:Write Ahead Log 提前写日志是一个分布式文件系统上的文件，WAL存储没有持久化的新数据，用于故障恢复，类似Oracle 的Redo Log。

BlockCache：读缓存，它把频繁读取的数据放入内存中，采用LRU

MemStore：写缓存，存储来没有来得及写入磁盘的新数据，每一个region的每一个列族有一个MemStore

Hfiles ：存储行，作为键值对，在硬盘上。

[![image](/images/2018\05\dsj\1569798685131.jpg "image")]

Hbase 写步骤1：

当客户端提交一个Put 请求，第一步是把数据写入WAL：

-编辑到在磁盘上的WAL的文件，添加到WAL文件的末尾

-WAL用于宕机恢复

[![image](/images/2018\05\dsj\1569798685134.jpg "image")]

Hbase 写步骤2

一旦数据写入WAL，将会把它放到MemStore里，然后将返回一个ACk给客户端

[![image](/images/2018\05\dsj\1569798685137.jpg "image")]

MemStore

MemStore 存储以键值对的方式更新内存，和存储在HFile是一样的。每一个列族就有一个MemStore ，以每个列族顺序的更新。

[![image](/images/2018\05\dsj\1569798685139.jpg "image")]

HBase Region 刷新（Flush）

当MemStore 积累到足够的数据，则整个排序后的集合被写到HDFS的新的HFile中，每个列族使用多个HFiles，列族包含真实的单元格，或者是键值对的实例，随着KeyValue键值对在MemStores中编辑排序后，作为文件刷新到磁盘上。

注意列族是有数量限制的，每一个列族有一个MemStore，当MemStore满了，则进行刷新。它也会保持最后一次写的序列号，这让系统知道直到现在都有什么已经被持久化了。

最高的序列号作为一个meta field 存储在HFile中，来显示持久化在哪里结束，在哪里继续。当一个region 启动后，读取序列号，最高的则作为新编辑的序列号。

[![image](/images/2018\05\dsj\1569798685142.jpg "image")]

HBase HFile

数据存储在HFile，HFile 存储键值，当MemStore 积累到足够的数据，整个排序的键值集合会写入到HDFS中新的HFile 中。这是一个顺序的写，非常快，能避免移动磁头。

[![image](/images/2018\05\dsj\1569798685145.jpg "image")]

HFile 的结构

HFile 包含一个多层的索引，这样不必读取整个文件就能查找到数据，多层索引像一个B+树。

1. 键值对以升序存储
2. 在64K的块中，索引通过行健指向键值对的数据。
3. 每个块有自己的叶子索引
4. 每个块的最后的键被放入到一个中间索引中。
5. 根索引指向中间索引。

trailer (追踪器)指向 meta的块，并在持久化到文件的最后时被写入。trailer 拥有 bloom过滤器的信息以及时间范围（time range）的信息。Bloom 过滤器帮助跳过那些不含行健的文件，时间范围（time range）则跳过那些不包含在时间范围内的文件。

[![image](/images/2018\05\dsj\1569798685148.jpg "image")]

HFile Index

索引是在HFile 打开并放入内存中时被加载的，这允许在单个磁盘上执行查找。

[![image](/images/2018\05\dsj\1569798685150.jpg "image")]

HBase 读合并

一个行的键值单元格可以被存储在很多地方，行单元格已经被存储到HFile中、在MemStore最近被更新的单元格、在Block cache最佳被读取的单元格，所以当你读取一行数据时，系统怎么能把相对应的单元格内容返回呢？一次读把block cache, MemStore, and HFiles中的键值合并的步骤如下：

1、首先，扫描器（scanner ）在读缓存的Block cache寻找行单元格，最近读取的键值缓存在Block cache中，当内存需要时刚使用过的（Least Recently Used ）将会被丢弃。

2、接下来，扫描器（scanner）将在MemStore中查找，以及在内存中最近被写入的写缓存。

3、如果扫描器（scanner）在MemStore 和Block Cache没有找到所有的数据，则HBase 将使用 Block Cache的索引以及bloom过滤器把含有目标的行单元格所在的HFiles 加载到内存中。

[![image](/images/2018\05\dsj\1569798685153.jpg "image")]

每个MemStore有许多HFiles 文件，这样对一个读取操作来说，多个文件将不得不被多次检查，势必会影响性能，这种现象叫做读放大（read amplification）。

[![image](/images/2018\05\dsj\1569798685156.jpg "image")]

HBase 辅压缩（minor compaction）

HBase将会自动把小HFiles 文件重写为大的HFiles 文件，这个过程叫做minor compaction。

辅助压缩减少文件的数量，并执行合并排序。

[![image](/images/2018\05\dsj\1569798685221.jpg "image")]

HBase 主压缩（Major Compaction）

主压缩将会合并和重写一个region 的所有HFile 文件，根据每个列族写一个HFile 文件，并在这个过程中，删除deleted 和expired 的单元格，这将提高读性能。

然而因为主压缩重写了所有的文件，这个过程中将会导致大量的磁盘IO操作以及网络拥堵。我们把这个过程叫做写放大（write amplification）。

[![image](/images/2018\05\dsj\1569798685224.jpg "image")]

Region = 临近的键

1. 一个表将被水平分割为一个或多个Region，一个Region包含相邻的起始键和结束键之间的行的排序后的区域。
2. 每个region默认1GB
3. 一个region的表通过Region Server 向客户端提供服务
4. 一个region server可以服务1000 个region

[![image](/images/2018\05\dsj\1569798685226.jpg "image")]

Region 分裂

初始时一个table在一个region 中，当一个region 变大之后，将会被分裂为2个子region，每个子Region 代表一半的原始Region，在一个相同的 Region server中并行打开。

然后把分裂报告给HMaster。因为需要负载均衡的缘故，HMaster 可能会调度新的Region移动到其他的Server上。

[![image](/images/2018\05\dsj\1569798685229.jpg "image")]

读负载均衡（Read Load Balancing）

分裂一开始发生在相同的region server上，但是由于负载均衡的原因。HMaster 可能会调度新的Region被移动到其他的服务器上。

导致的结果是新的Region Server 提供数据的服务需要读取远端的HDFS 节点。直到主压缩把数据文件移动到Regions server本地节点上，Hbase数据当写入时是本地的，

但是当一个region 移动（诸如负载均衡或是恢复操作等），它将不会是本地的，直到做了主压缩的操作（major compaction.）

[![image](/images/2018\05\dsj\1569798685231.jpg "image")]

HDFS数据复制

所有的读写操作发生在主节点上，HDFS 复制WAL和HFile 块，HFile复制是自动发生的，HBase 依赖HDFS提供数据的安全，

当数据写入HDFS，本地化地写入一个拷贝，然后复制到第二个节点，然后复制到第三个节点。

WAL 文件和 HFile文件通过磁盘和复制进行持久化，那么HBase怎么恢复还没来得及进行持久化到HFile中的MemStore更新呢？

[![image](/images/2018\05\dsj\1569798685234.jpg "image")]

[![image](/images/2018\05\dsj\1569798685237.jpg "image")]

HBase 故障恢复

当一个RegionServer 挂掉了，坏掉的Region 不可用直到发现和恢复的步骤发生。Zookeeper 决定节点的失败，然后失去region server的心跳。

然后HMaster 将会被通知Region Server已经挂掉了。

当HMaster检查到region server已经挂掉后，HMaster 将会把故障Server上的Region重写分配到活跃的Region servers上。

为了恢复宕掉的region server，memstore 将不会刷新到磁盘上，HMaster 分裂属于region server 的WAL 到单独的文件，

然后存储到新的region servers的数据节点上，每个Region Server从单独的分裂的WAL回放WAL。来重建坏掉的Region的MemStore。

[![image](/images/2018\05\dsj\1569798685240.jpg "image")]

数据恢复

WAL 文件包含编辑列表，一个编辑代表一个单独的put 、delete.Edits 是按时间的前后顺序排列地写入，为了持久化，增加的文件将会Append到WAL 文件的末尾。

当数据在内存中而没有持久化到磁盘上时失败了将会发生什么？通过读取WAL将WAL 文件回放，

添加和排序包含的edits到当前的MemStore，最后MemStore 刷新将改变写入到HFile中。

[![image](/images/2018\05\dsj\1569798685243.jpg "image")]

**HBase架构的优点**

一致模型：当写操作返回时，所有的读将看到一样的结果

自动扩展：Regions 随着数据变大将分裂；使用HDFS传播和复制数据

内建的恢复机制：使用WAL

和Hadoop的集成：直接使用mapreduce

**HBase架构的缺点**

WAL回放较慢

故障恢复较慢

主压缩导致IO瓶颈。
