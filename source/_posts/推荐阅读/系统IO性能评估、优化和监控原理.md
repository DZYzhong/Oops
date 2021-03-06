---
title: 系统IO性能评估、优化和监控原理
date: 2020-04-30 11:23:00
categories: 推荐阅读
---
生产中经常遇到一些IO延时长导致的系统吞吐量下降、响应时间慢等问题，例如交换机故障、网线老化导致的丢包重传；存储阵列条带宽度不足、缓存不足、QoS限制、RAID级别设置不当等引起的IO延时。

**一、评估 IO 能力的前提**

评估一个系统IO能力的前提是需要搞清楚这个系统的IO模型是怎么样的。那么IO模型是什么，为什么要提炼IO模型呢？

**(一) IO模型**

在实际的业务处理过程中，一般来说IO比较混杂，比如说读写比例、IO尺寸等等，都是有波动的。所以我们提炼IO模型的时候，一般是针对某一个特定的场景来建立模型，用于IO容量规划以及问题分析。

最基本的模型包括：IOPS，带宽和IO的大小。如果是磁盘IO，那么还需要关注：

* 磁盘IO分别在哪些盘
* 读IO和写IO的比例
* 读IO是顺序的还是随机的
* 写IO是顺序的还是随机的

**(二)为什么要提炼IO模型**

不同模型下，同一台存储，或者说同一个LUN，能够提供的IOPS、带宽（MBPS）、响应时间3大指标的最大值是不一样的。

当存储中提到IOPS最大能力的时候，一般采用随机小IO进行测试，此时占用的带宽是非常低的，响应时间也会比顺序的IO要长很多。如果将随机小IO改为顺序小IO，那么IOPS还会更大。当测试顺序大IO时，此时带宽占用非常高，但IOPS却很低。

因此，做IO的容量规划、性能调优需要分析业务的IO模型是什么。

**二、评估工具**

**(一)磁盘IO评估工具**

磁盘IO能力的评估工具有很多，例如**orion、iometer，dd、xdd、iorate，iozone，postmark**，不同的工具支持的操作系统平台有所差异，应用场景上也各具特色。

有的工具可以模拟应用场景，比如orion是oracle出品，模拟Oracle数据库IO负载(采用与Oracle相同的IO软件栈)。

模拟Oracle应用对文件或磁盘分区进行读写(可指定读写比例、IO size，顺序或随机)这里就需要提前知道自己的IO模型。如果不知道，可以采用自动模式，让orion自动的跑一遍，可以得出不同进程的并发读写下，最高的IOPS、MBPS，以及对应的响应时间。

比对DD，仅仅是对文件进行读写，没有模拟应用、业务、场景的效果。Postmark可以实现文件读写、创建、删除这样的操作。适合小文件应用场景的测试。

**(二)网络IO评估工具**

* **Ping：**最基本的，可以指定包的大小。
* **IPerf、ttcp：**测试tcp、udp协议最大的带宽、延时、丢包。
* **衡量windows平台下的带宽能力，工具比较多：****NTttcp、LANBench、pcattcp、LAN Speed Test (Lite)、NETIO、NetStress。**

**三、主要监控指标和常用监控工具**

**(一)磁盘IO**

对于存储IO：Unix、Linux平台，Nmon、IOstat是比较好的工具。Nmon用于事后分析，IOstat可用于实时查看，也可以采用脚本记录下来事后分析。

**1\. IOPS**

* 总IOPS：Nmon DISK\_SUMM Sheet：IO/Sec
* 每个盘对应的读IOPS ：Nmon DISKRIO Sheet
* 每个盘对应的写IOPS ：Nmon DISKWIO Sheet
* 总IOPS：命令行iostat -Dl：tps
* 每个盘对应的读IOPS ：命令行iostat -Dl：rps
* 每个盘对应的写IOPS ：命令行iostat -Dl：wps

**2.带宽**

* 总带宽：Nmon DISK\_SUMM Sheet：Disk Read KB/s，Disk Write KB/s
* 每个盘对应的读带宽：Nmon DISKREAD Sheet
* 每个盘对应的写带宽：Nmon DISKWRITE Sheet
* 总带宽：命令行iostat -Dl：bps
* 每个盘对应的读带宽：命令行iostat -Dl：bread
* 每个盘对应的写带宽：命令行iostat -Dl：bwrtn

**3.响应时间**

* 每个盘对应的读响应时间：命令行iostat -Dl：read - avg serv，max serv
* 每个盘对应的写响应时间：命令行iostat -Dl：write - avg serv，max serv

**4.其他**

磁盘繁忙程度、队列深度、每秒队列满的次数等等。

**(二)网络IO**

**1.带宽**

* 最好在网络设备处直接查看流量（比较准），如果在业务的服务器也可以查看
* Nmon：NET Sheet
* 命令行topas：Network：BPS、B-In、B-Out

**
**

**2.响应时间**

简单的方法，可采用ping命令查看ping的延时是否在合理范围，是否有丢包现象。

有些交换机对ping命令设置了较低的优先级，可能在回复、转发ping包的时候有延迟，因此ping的结果不一定能反映真实情况。如果需要更为精确的测量可以探针捕获从某服务器建立TCP连接时发送的SYN包后开始计时起，到其收到对端发回的TCP SYNACK后的时间差。

更为准确、利于后期分析的方法是采用专业的网络设备在网络设备的端口处进行报文捕获和计算分析。

**四、性能定位与优化**

**(一)对磁盘IO争用的调优思路有哪些？**

**典型问题：**针对主要争用是IO相关的场景下，调优的思路有哪些？主要的技术或者方法是什么？

一、首先要搞清楚IO争用是因为应用等层面的IO量过大导致，还是系统层面不能承载这些IO量。

如果应用层面有过多不必要的读写，首先解决应用问题。

**举例1：**数据库里面用于sort的buffer过小，当做sort的时候，有大量的内存与磁盘之间的数据交换，那么这类IO可以通过扩大sort buffer的内存来减少或避免。

**举例2：**从应用的角度，一些日志根本不重要，不需要写，那么可以把日志级别调低、甚至不记录日志，数据库层面可以加hint “no logging”。

二、存储问题的分析思路

存储IO问题可能出现在IO链路的各个环节，分析IO瓶颈是主机/网络/存储中的哪个环节导致的。

IO从应用-\>内存缓存-\>块设备层-\>HBA卡-\>驱动-\>交换网络-\>存储前端-\>存储cache-\>RAID组-\>磁盘，经过了一个很长的链条，分析思路：

* **1、主机侧：**应用-\>内存缓存-\>块设备层→HBA卡-\>驱动
* **2、网络侧：**交换网络
* **3、存储侧：**存储前端-\>存储cache-\>RAID组-\>磁盘

**1、主机侧**

当主机侧观察到的时延很大，存储侧的时延较小，则可能是主机侧或网络存在问题。

主机是I/O的发起端，I/O特性首先由主机的业务软件和操作系统软件和硬件配置等决定。例如，在“服务队列满”这一章节介绍的I/O 队列长度参数(queue\_depth)，当然，还有许多其他的参数(如： **Driver 可以向存储发的最大的 I/O、光纤卡DMA Memor区域大小、块设备并发数、HBA卡并发数**)。

若排查完成，性能问题还是存在，则需要对组网及链路、存储侧进行性能问题排查。

**2、网络侧**

当主机侧观察到的时延很大，存储侧的时延较小，且排查主机侧无问题时，则性能问题可能出现在链路上。

可能的问题有**：带宽达到瓶颈、交换机配置不当、交换机故障、多路径选路错误、线路的电磁干扰、光纤线有损、接口松动等。带宽达到瓶颈、交换机配置不当、多路径选路错误、线路的电磁干扰等**。

**3、存储侧**

如果主机侧时延与存储侧时延都很大且相差较小，说明问题可能出现在存储上。首先需要了解当前存储侧所承载的IO模型、存储资源配置，并从存储侧收集性能数据，按照I/O路径进行性能问题的定位。

常见原因如**硬盘性能达到上限、镜像带宽达到上限、存储规划(如条带过小)、硬盘域和存储池划分（例如划分了低速的磁盘）、thin LUN还是thick LUN、LUN对应的存储的缓存设置**(缓存大小、缓存类型，内存还是SSD)。

IO的Qos限制的磁盘IO的带宽、LUN优先级设置、存储接口模块数量过小、RAID划分(比如RAID10 \> RAID5 \> RAID6)、条带宽度、条带深度、配置快照、克隆、远程复制等增值功能拖慢了性能、是否有重构、Balancing等操作正在进行、存储控制器的CPU利用率过高、LUN未格式化完成引起短时的性能问题、cache刷入磁盘的参数(高低水位设置)，甚至数据在盘片的中心还是边缘等等。

具体每个环节 都有一些具体的方法、命令、工具来查看性能表现，这里不再赘述。

**(二)关于低延迟事务、高速交易的应用在IO方面可以有哪些调优思路和建议？**

**典型问题：**关于近期在一些证券行业碰到的低延迟事务、高速交易的应用需求，在IO模型路径方面可以有哪些可以调优的思路和建议？

对于低延迟事务，可以分析一下业务是否有持久化保存日志的需要，或者说保存的安全程度有多高，以此来决定采用什么样的IO。

**1.从业务角度**

比如说业务上不需要保存日志，那就不用写IO。或者保存级别不高，那就可以只写一份数据，对于保存级别较高的日志，一般要双写、或多写。

**2.从存储介质角度**

* 1）可以全部采用SSD
* 2）或者采用SSD作为存储的二级缓存(一级缓存是内存）
* 3）或者存储服务器里面采用存储分级(将热点数据迁移到SSD、SAS等性能较好的硬盘上）
* 4）可以采用RAMDISK（内存作为磁盘用）
* 5）增加LUN所对应的存储服务器的缓存

**3.从配置的角度**

普通磁盘存储的LUN，可以设置合理的RAID模式（比如RAID10）去适应你的业务场景。

分条的深度大于等于一个IO的大小、有足够的宽度支持并发写。

**4.IO路径的角度**

采用高速的组网技术，而不用iSCSI之类的低速方式。

**(三) 网络IO问题定位思路和方法**

与磁盘IO类似，网络IO同样需要分段查找和分析。通过网络抓包和分析的工具，诊断网络的延时、丢包等异常情况出现在哪一段，然后具体分析。

同时，抓主机端的IPtrace有助诊断不少的网络问题，参考文章http://www.aixchina.net/Article/177921

**(四)误判为IO问题的案例**

很多时候，应用响应时间很慢，看似是IO问题，实则不然，这里举两个例子。

**1.案例分享：**

Oracle buffer等待占总时间的大头，在一个场景中，Oracle的awr报告top10事件的第一名是：buffer busy waits。

Buffer busy waits是个比较General的等待，是session等待某个buffer引起的，但具体是什么buffer并不清楚，比如log sync等待也会引起Buffer busy wait。

这是个连带指标，分析是暂且不管，需要看看他临近的问题事件是什么。

AWR报告top10事件的第二名是enq：TX - index contention。这里的临近事件就是enq：TX - index contention， index contention常由大量并发INSERT 造成的 index split 引起，也就是说不断更新索引的过程中，二叉树不断长大。需要分裂，分裂的时候，其他Session就需要等着(这里的分析需要些数据库知识)。

之后的调优过程中，将索引分区，避免竞争。调整后重新测试，Index contention、Bufferbusy wait双双从top10事件中消失了。

这类数据库相关的等待事件非常常见，看似是等待IO，实际上是数据库的规划设计有问题。

**2.案例分享：**

Ping延时间歇性暴增。某业务系统的响应时间很不稳定，该系统有两类服务器构成，可以简单理解为A和B，A为客户端，B为服务端，A处业务的响应时间非常不稳定。

**第一步：**从各类资源(CPU、内存、网络IO、磁盘IO)中追查原因。最终发现A与B直接的网络延时非常不稳定。A ping B，在局域网环境，按理说延时应该是0ms-1ms之间，而我们在业务高峰时发现，隔一小段时间就有100-200ms的延时出现。即使在没有业务的情况下，Ping也30-40ms的延时。

**第二步：**开始排查网路。换A的物理端口、换交换机、换网线、换对端的物理端口等等一系列措施之后，发现问题依然存在。

**第三步：**采用网络探测设备，从交换机两侧端口抓包，分析一个tcp连接的建立过程时间消耗在哪里。分析后发现，200ms的延时，都是在B测。即一个tcp连接建立过程在A侧和交换机侧几乎没有什么时间消耗。

**第四步：**B侧多台分区共用一个物理机。猜测是否是分区过多导致。当只有一个LPAR启动的时候，没有ping的延时，当启动一部分LPAR时候，延时较小，当所有LPAR均启动，Ping 延时较大。

**问题根本原因：**此时，问题水落石出，原来是由于分区过多导致了B回复A的ping有了延时。那么为什么会出现这种情况呢？一个物理机上CPU资源是有限的（本环境中是3颗），即使只有一个LPAR，其上面的N个进程也会去轮流使用CPU，何况此时是M台LPAR，MN个进程去轮流使用这三个CPU，当然调度算法并不是这么简单，这里仅仅是从理论上做个说明。

假设每个CPU时间片是10ms，那么极端情况下，一个进程要等到CPU需要等待(MN-1)\*10(ms)/3。况且，这么多LPAR的进程轮询一遍CPU，CPU里面的cache 数据估计早就被挤走了，重新加载是比较耗时的。

**应对方法：**之前LPAR也设置了保障的CPU(MIPS数量的保障)，但只有数量没有质量(上述提到的CPU cache问题，即亲和性问题)。

将重要的LPAR分配dedicated CPU，保证CPU资源的质量，保证轮询CPU的客户尽量少，这样CPU cache中的数据尽量不被清走。经验证，Ping延时基本消失，方法有效。本案例是一起看似是网络问题，但实际是资源调度方式的问题。

顺便提一句，很多情况下，客户端的响应时间不稳定都是由服务器端的服务能力不稳定造成的。一般情况下都是应用、数据库的问题造成。而本案例是操作系统层面答复Ping出现间歇性延时，很容易误导我们的分析判断。