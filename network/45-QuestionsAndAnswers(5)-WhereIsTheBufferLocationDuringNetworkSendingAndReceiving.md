# 45 Q&A(5): Where is the buffer location during network sending and receiving?

The column has been updated so far, the last of the four basic modules - the network chapter, we have finished learning. We are very happy that you have not fallen behind and are still actively learning and thinking and practicing, leaving comments and interacting enthusiastically. Many students also shared their analysis ideas and optimization methods for various performance problems they encountered in the actual production environment, so thank you for that.

Today is the fifth session of performance optimization Q&A. As usual, I have extracted some typical questions from the comments of the web module as the content of today's Q&A, and will focus on replying. Again, for your convenience, they are not in strict article order.

For each question, I have included a screenshot of the question asked in the comments section. If you need to review the original content, you can scan the QR code at the bottom right of each question to view it.

## Question 1: The location of the buffer during network sending and receiving

Written on 2019/02/26

> When a network frame arrives at the NIC, the NIC puts the network packet into the packet receipt queue by DMA;
> then it tells the interrupt handler that the network packet has been received by a hard interrupt.
> Next, the NIC interrupt handler allocates a kernel data structure (sk_buff) for the network frame and copies it to the sk_buff buffer;
> then it notifies the kernel of the new network frame via a soft interrupt.
> Next, the kernel stack retrieves the network frame from the buffer and processes it through the network stack, layer by layer, from bottom to top.

Q:  
> Hello teacher, I have some questions about the above paragraph.
> 
> The packet queue belongs to where the storage space, is it part of the physical memory, or the storage space in the network card, through the dma way to put the data into the packet queue, I guess this packet queue is the space in the physical memory. The packet queue is managed by the kernel, right, that is, it has nothing to do with the user space address of a process?
> 
> Then where is the sk_buf buffer storage space, and why is the packet queue copied to this buffer, which is maintained by the protocol stack? It also belongs to the kernel, and the process of user space address has a relationship? Is the receive-send buffer of the socket mapped to the user space address of the process? Or is it a buffer maintained by the stack for each socket in the kernel? And how do these buffers mentioned above relate to cache and buf? Will they be recycled? Is the kernel stack run by means of a kernel thread? Is it possible to see the name of this thread?

The first point is the question of sending and receiving queues and buffer locations during network sending and receiving.

In 🔗[What you must know about Linux networking](https://time.geekbang.org/column/article/80898), I have described the send/receive process for Linux networks. This process involves several queues and buffers, including

- A **circular buffer** that is interacted with through DMA when the network card sends or receives network packets;
- The **sk_buff buffer**, which is a kernel data structure used by the interrupt handler of the network card to allocate network frames;
- The **socket buffer** used when an application interacts with the network protocol stack through the socket interface.

Accordingly, though, there are two problems.

First, where are these buffers located? Is it in the NIC hardware, or is it in memory? This question is actually easy to understand when you think about it - these buffers are in kernel-managed memory.

Among them, the **circular buffer**, since it requires DMA to interact with the NIC, should rightfully be in the scope of the NIC device driver.

The **sk_buff buffer**, is a bi-directional chain table that maintains the network frame structure; each element of the chain table is a network frame (Packet). Although the TCP/IP stack is divided into several layers, passing between different layers above and below actually requires only manipulating the pointers in this data structure without data replication.

The **Socket Buffer** allows the application to configure each socket with a different size receive or send buffer. When an application sends data, it actually writes it to the buffer; when it receives data, it actually reads it from the buffer. Further processing of the data in the buffers is done by the TCP or UDP protocols at the transport layer.

Second, what is the connection between these buffers and the Buffer and Cache mentioned in the memory section?

This is not a difficult question to answer. As I mentioned in the memory module, the Buffer mentioned in memory is directly related to the block device; the others are Cache.

In fact, sk_buff, socket buffers, connection tracking, etc. are all managed through the slab allocator. You can see how much memory they occupy directly via <ins>/proc/slabinfo</ins>.

## Question 2: Is the kernel stack, run by way of a kernel thread?

The second question is, does the kernel stack run as a kernel thread? And how is the network stack executed in the kernel?

When it comes to network sending and receiving, as I mentioned in the interrupt handling article, there is a dedicated kernel thread ksoftirqd for soft interrupt handling. each CPU is bound to a ksoftirqd kernel thread, for example, for 2 CPUs, there are two kernel threads ksoftirqd/0 and ksoftirqd/1.

Note, however, that not all networking functions are handled in the soft interrupt kernel thread. There are many other mechanisms in the kernel (e.g. hard interrupts, kworker, slab, etc.) that work together to ensure the proper operation of the entire network stack.

There will be a separate article on how the network stack works in the kernel and how to dynamically track the execution flow of the kernel. If you are interested in this part, you can try to analyze it with the `perf`, `systemtap`, `bcc-tools`, etc. that we have mentioned.

## Question 3: Is the maximum number of connections limited to 65535 ports?

We know that both TCP and UDP, the port number only occupies 16 bits, which means that the maximum value is only 65535, does that mean that if you use TCP protocol, the maximum number of concurrent connections is only 65535 for a single machine and a single IP address?

For this question, first of all, you need to know that the Linux protocol stack marks a connection by a five-tuple (i.e., protocol, source IP, source port, destination IP, destination port).

Understand this, the problem actually has an idea. We should analyze it in two scenarios, client-side and server-side.

For the client, each time a TCP connection request is initiated, a free local port needs to be allocated to connect to the remote server. Since this local port is exclusive, the client can only initiate a maximum of 65535 connections.

For the server side, it usually listens on a fixed port (e.g., port 80) and waits for the client to connect. Based on the five-tuple structure, we know that the client's IP and port are both variable. Without considering IP address classification and resource constraints, the theoretical maximum number of server-side connections, which can be 2 to the 48th power (32-bit IP and 16-bit port), is much larger than 65535.

So, all things considered, the maximum number of connections supported on the client side is 65535, while the number of connections supported on the server side is massive. Of course, due to the performance of the Linux stack itself, as well as various physical and software resource limitations, such a large number of connections is still far from being possible (in fact, C10M is already difficult).

## Question 4: "How to optimize NAT performance" Post-lesson reflection

At the end of 🔗[How to Optimize NAT Performance](https://time.geekbang.org/column/article/83189), I left you with two thought-provoking questions.

MASQUERADE is one of the most common SNAT rules used to provide shared egress IPs for multiple intranet IP addresses. suppose you have a Linux server that uses MASQUERADE to provide egress access to all IPs on the intranet. Then

- Does MASQUERADE work properly when multiple intranet IP addresses have the same port number?
- Are there any potential problems with this usage when the number of intranet IP addresses or requests is high?

For these two thought-provoking questions, students like "我来也" and "ninuxer" have given good answers:

我来也  
wrote on 2019/02/25

[Day 41 clocking in]

> I've never seen NAT in my project experience. Instead, I've seen nat-related options in local virtual machine environments or on routers.
> 
> Question 1: Does MASQUERADE still work when multiple intranet IP addresses have the same port number?
> 
> I think it works, otherwise it would not allow setting ip address segments. [pure speculation ha] in the router to do port mapping, an external port can only correspond to an internal IP. But the opposite direction, nat in the conversion of the source address, it should record the original connection information, otherwise the received packet should be sent to whom?
> 
> Question 2: If the number of intranet IP addresses or the number of requests is relatively large, is there any pitfalls in this way? According to previous experience, when the number of requests is too many, it will lead to a rise in CPU soft interruptions. I've seen that the conntrack table of iptables is full, which causes slow access to websites. [https://my.oschina.net/jean/blog/189935]
> 
> ```
> The kernel uses the ip_conntrack module to record the status of iptables network packets and save it to a table (which is in memory), which will gradually take up the available space if the network is busy, such as high connections, high concurrent connections, etc.
> ```
> 
> Optimize Linux NAT gateway [https://tech.youzan.com/linu x_nat/]
> 
> ```
> net.netfilter.nfconntrackbuckets This parameter, by default, is a bit small, and when the number of connections increases, it will inevitably cause "hash conflicts" to increase and "hash processing" performance to decrease. (Is that so? )
> ```

___

ninuxer  
wrote on 2019/02/27

[Day 43 clocking in]

> My work scenario is not used to nat, basically based on layer-4 or layer-7 of the reverse proxy.  
> For the first problem, it is possible. The second problem is not possible, I think there is a connection tracking table, the number of files, the number of ports limit.

Let's look at the first point first. When multiple intranet IP addresses have the same port number, MASQUERADE will of course still work fine. However, you must have heard that after configuring MASQUERADE, you need to configure each application to change the port number manually.

In fact, MASQUERADE records the information of each connection through the conntrack mechanism. As I mentioned in the third question, it takes five tuples to flag a connection, and as long as the five tuples are not identical, the network connection will work.

Looking at the second point, this is not a big problem when the number of intranet IP addresses and connections is small. However, in the case of a particularly large number of IP addresses or concurrent connections, you may encounter a variety of resource constraints.

For example, since MASQUERADE converts multiple internal IPs into the same external IP (i.e., SNAT), the source port of the original network packet may also be reassigned to ensure that the source port sent out is not duplicated. In this case, the port number of the converted external IP becomes an important factor in limiting the number of connections.

In addition to this, connection tracking, network bandwidth of MASQUERADE machines, etc., are potential bottlenecks, and there is also the problem of single point of access. All these situations need special attention in our practical use.

Today mainly answer these questions, but also welcome you to continue to write down questions and feelings in the comments section, I will continue to answer. I hope that with each question and answer, you can work with you to internalize the knowledge of the article into your ability, we not only in the practical exercises, but also in the exchange of progress.
