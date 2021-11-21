---
title: PCIe Posted Vs Non-Posted Transactions
date: 2021-11-21 19:10:36
categories: [体系结构]
tags: [virtio, pcie, 体系结构]
---

在[硬件virtio 0.95 reset问题](../virtio.0.95-reset.html)中提到了PCIe **posted transactions**和**non-posted transactions**两个概念，当时未做细致的分析，此处进行适量补充。学习输入也仅仅是些外网博客，讲得比较肤浅不够权威，但对于目前我的工作内容来说基本足够了。

#### Non-posted Transactions

当设备完成请求处理后，需要向requester发送一个completion Transaction Layer Packet (TLP) 。Completer可以在收到请求后的某个时刻发送TLP完成报文，可以不必立刻回复。

对于读操作，完成报文中包含了读取到的数据（如果成功）或者error status（如果失败）；对于写操作，完成报文中不会含有数据（如果成功）、但可能包含error status（如果失败）。

以CPU为requester为例：

![](/images/CPU_mem_rd.png)

&nbsp;

#### Posted Transactions

Requester不希望、也不会收到TLP完成报文。如果发生失败，requester将不会知道，但是completer可以触发失败消息通知到Root Complex。

以CPU为requester为例：

![](/images/CPU_mem_wr.png)

&nbsp;

#### 差异总结

If we compare the life cycle of a bus write operation with the one of a read, there’s an evident difference:

+ A **write** TLP operation is **fire-and-forget**. Once the packet has been formed and handed over to the Data Link Layer, there’s no need to worry about it anymore.
+ A **read** operation, on the other hand, requires the Requester to wait for a Completion. Until the Completion packet arrives, the Requester must retain information about what the Request was, and sometimes even hold the CPU’s bus: If the CPU’s bus started a read cycle, it must be held in wait states until the value of the desired read operation is available at the bus’ data lines. This can be a horrible slowdown of the bus, which is rightfully avoided in recent systems.

&nbsp;

#### 典型操作分类

| Non-posted Transcations                                      | Posted Transactions         |
| ------------------------------------------------------------ | --------------------------- |
| Memory Reads<br />Memory Read Lock<br />I/O Reads<br />I/O Writes<br />Configuration Reads (both Type 0 and Type 1)<br />Configuration Writes (both Type 0 and Type 1) | Memory Writes<br />Messages |

&nbsp;

#### 参考链接

1. <http://xillybus.com/tutorials/pci-express-tlp-pcie-primer-tutorial-guide-1>
2. <https://paritycheck.wordpress.com/2008/01/13/pcie-posted-vs-non-posted-transactions/>
3. 《*PCI* *Express* *System* *Architecture*》
