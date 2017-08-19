# PCI Express transaction overview

PCI Express 事务可以分为四大类：
1. 存储事务(Memory)
2. I/O事务
3. 配置事务(Configuration)
4. 消息事务(Message)

事务定义：请求者和完成者之间完成一次信息传送需要完成的一系列数据包传输的过程。

上述事务又可分为Non-Posted transaction 和 Posted transaction。

## Posted 和 Non-Posted 传送方式
PCI总线规定了两类数据传送方式，分别是Posted和Non-Posted数据传送方式。其中使用Posted数据传送方式的总线事务也被称为Posted总线事务；而使用Non-Posted数据传送方式的总线事务也被称为Non-Posted总线事务。

其中Posted总线事务指PCI主设备向PCI目标设备进行数据传递时，当数据到达PCI桥后，即由PCI桥接管来自上游总线的总线事务，并将其转发到下游总线。采用这种数据传送方式，在数据还没有到达最终的目的地之前，PCI总线就可以结束当前总线事务，从而在一定程度上解决了PCI总线的拥塞。

而Non-Posted总线事务是指PCI主设备向PCI目标设备进行数据传递时，数据必须到达最终目的地之后，才能结束当前总线事务的一种数据传递方式。

显然采用Posted传送方式，当这个Posted总线事务通过某条PCI总线后，就可以释放PCI总线的资源；而采用Non-Posted传送方式，PCI总线在没有结束当前总线事务时必须等待。这种等待将严重阻塞当前PCI总线上的其他数据传送，因此PCI总线使用Delayed总线事务处理Non-Posted数据请求，使用Delayed总线事务可以相对缓解PCI总线的拥塞。

PCI总线规定只有存储器写请求(包括存储器写并无效请求)可以采用Posted总线事务，下文将Posted存储器写请求简称为PMW(Posted Memory Write)，而存储器读请求、I/O读写请求、配置读写请求只能采用Non-Posted总线事务。




