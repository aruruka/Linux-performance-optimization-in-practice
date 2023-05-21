# 44 Rouatines: A few ideas for network performance optimization(part 2)

In the last section, we learned a few ideas for network performance optimization, so let me take you through a brief review.

When optimizing the performance of your network, you can combine the network protocol stack and network send/receive flow of your Linux system, and then optimize each layer from application, socket, transport layer, network layer and then link layer, layer by layer. In the last issue, we focused on optimization ideas for applications and sockets, such as

- in the application, mainly optimizing the I/O model, the working model, and the network protocols at the application layer;
- in the socket layer, mainly optimizing the buffer size of the sockets.

Today, let's follow the TCP/IP network model and continue down to see how to optimize Linux network performance from the transport layer, network layer, and link layer.

## Network performance optimization

### Transport layer

The most important transport layer is the TCP and UDP protocols, so the optimization here is actually mainly for these two protocols.

Let's first look at the optimization of the TCP protocol.

TCP provides a reliable connection-oriented transport service. To optimize TCP, we first need to understand the basic principles of the TCP protocol, such as flow control, slow start, congestion avoidance, delayed acknowledgement, and state flow diagrams (as shown in the figure below).

<center><a href="https://ibb.co/wcjNck4">
    <img src="https://i.ibb.co/fDLYDgH/Tcp-Status-Figure.png" alt="Tcp-Status-Figure" border="0">
</a></center>

I will not expand on the details of these principles. If you haven't fully mastered them, I suggest you learn these basic principles before optimizing them, instead of copying and trying them out in a lumpy manner.

Once you have mastered these principles, you can optimize TCP without disrupting its normal operation. In the following, I will elaborate in several categories.

In the first category, in scenarios where the number of requests is relatively high, you may see a large number of connections in the TIME_WAIT state, which take up a lot of memory and port resources. In this case, we can optimize the kernel options associated with the TIME_WAIT state, for example by taking the following steps.

- Increase the number of connections in the TIME_WAIT state net.ipv4.tcp_max_tw_buckets and increase the size of the connection tracking table net.netfilter.nf_conntrack_max.
- Decrease net.ipv4.tcp_fin_timeout and net. netfilter.nf_conntrack_tcp_timeout_time_wait to allow the system to release their resources as soon as possible.
- Enable port reuse net.ipv4.tcp_tw_reuse so that the ports occupied by the TIME_WAIT state can also be used for new connections.
- Increase the range of local ports. net.ipv4.ip_local_port_range. This allows more connections to be supported, increasing the overall concurrency.
- Increase the maximum number of file descriptors. You can use fs.nr_open and fs.file-max to increase the maximum number of file descriptors for the process and system, respectively, or configure LimitNOFILE in the application's systemd configuration file to set the maximum number of file descriptors for the application.

In the second category, to mitigate performance issues caused by attacks such as SYN FLOOD that exploit TCP protocol characteristics, you can consider optimizing kernel options related to SYN state, for example by taking the following steps.

- Increase the maximum number of TCP semi-connections net.ipv4.tcp_max_syn_backlog or enable TCP SYN cookies net.ipv4.tcp_syncookies to get around the problem of limiting the number of semi-connections (note that these two options cannot be used at the same time).
- Reduce the number of SYN+ACK packet retransmissions for connections in SYN_RECV state net.ipv4.tcp_synack_retries.

In the third category, in long connection scenarios, Keepalive is usually used to detect the status of TCP connections so that they can be automatically reclaimed when the peer connection is disconnected. However, the default Keepalive detection interval and retry count are generally not sufficient to meet the performance requirements of the application. So, this is when you need to optimize the kernel options associated with Keepalive, such as

- shorten the interval between the last packet and the Keepalive probe packet net.ipv4.tcp_keepalive_time;
- shorten the interval between sending a Keepalive probe packet net.ipv4.tcp_keepalive_intvl;
- reduce the number of retries after a failed Keepalive probe until the application is notified of the reduce the number of retries after a failed Keepalive probe until the application is notified net.ipv4.tcp_keepalive_probes.

After talking about so many TCP optimization methods, I have also organized them into a table for your easy reference when you need them (the values are for reference only, and the specific configuration has to be adjusted with your actual scenario)::

<center><a href="https://ibb.co/q11L6Mg">
    <img src="https://i.ibb.co/yssm2nk/Tcp-Optimization.png" alt="Tcp-Optimization" border="0">
</a></center>

When optimizing TCP performance, you should also be aware that conflicts may arise if different optimization methods are used at the same time.

For example, as we have analyzed in the case of network request latency, it is easy to increase network latency if Nagle algorithm is enabled on the server side and delayed acknowledgement mechanism is enabled on the client side.

In addition, if net.ipv4.tcp_tw_recycle is enabled on a server using NAT, it can easily lead to various connection failures. In fact, this option has been removed in kernel version 4.1 due to too many pitfalls.

After TCP, let's look at the UDP optimization.

UDP provides a datagram-oriented network protocol, which does not require a network connection and does not provide reliability guarantees. Therefore, UDP optimization, compared to TCP, is much simpler. Here I also summarize a few common optimization options.

- Increase the socket buffer size and UDP buffer range as mentioned in the previous section on sockets;
- Increase the range of local port numbers as mentioned in the previous section on TCP;
- Adjust the size of UDP packets according to the MTU size to reduce or avoid the occurrence of fragmentation. 

### Network layer

接下来，我们再来看网络层的优化。

网络层，负责网络包的封装、寻址和路由，包括 IP、ICMP 等常见协议。在网络层，最主要的优化，其实就是对路由、 IP 分片以及 ICMP 等进行调优。

第一种，从路由和转发的角度出发，你可以调整下面的内核选项。

- 在需要转发的服务器中，比如用作 NAT 网关的服务器或者使用 Docker 容器时，开启 IP 转发，即设置 net.ipv4.ip_forward = 1。
- 调整数据包的生存周期 TTL，比如设置 net.ipv4.ip_default_ttl = 64。注意，增大该值会降低系统性能。
- 开启数据包的反向地址校验，比如设置 net.ipv4.conf.eth0.rp_filter = 1。这样可以防止 IP 欺骗，并减少伪造 IP 带来的 DDoS 问题。

第二种，从分片的角度出发，最主要的是调整 MTU（Maximum Transmission Unit）的大小。

通常，MTU 的大小应该根据以太网的标准来设置。以太网标准规定，一个网络帧最大为 1518B，那么去掉以太网头部的 18B 后，剩余的 1500 就是以太网 MTU 的大小。

在使用 VXLAN、GRE 等叠加网络技术时，要注意，网络叠加会使原来的网络包变大，导致 MTU 也需要调整。

比如，就以 VXLAN 为例，它在原来报文的基础上，增加了 14B 的以太网头部、 8B 的 VXLAN 头部、8B 的 UDP 头部以及 20B 的 IP 头部。换句话说，每个包比原来增大了 50B。

所以，我们就需要把交换机、路由器等的 MTU，增大到 1550， 或者把 VXLAN 封包前（比如虚拟化环境中的虚拟网卡）的 MTU 减小为 1450。

另外，现在很多网络设备都支持巨帧，如果是这种环境，你还可以把 MTU 调大为 9000，以提高网络吞吐量。

第三种，从 ICMP 的角度出发，为了避免 ICMP 主机探测、ICMP Flood 等各种网络问题，你可以通过内核选项，来限制 ICMP 的行为。

- 比如，你可以禁止 ICMP 协议，即设置 net.ipv4.icmp_echo_ignore_all = 1。这样，外部主机就无法通过 ICMP 来探测主机。
- 或者，你还可以禁止广播 ICMP，即设置 net.ipv4.icmp_echo_ignore_broadcasts = 1。

### Data Link Layer

Below the network layer is the link layer, so finally, let's look at the link layer optimization approach.

The link layer is responsible for the transport of network packets across the physical network, such as MAC addressing, error detection, and the transmission of network frames through the NIC. Naturally, the optimization of the link layer revolves around these basic functions. Next, let's look at each of the different aspects.

Since the interrupt handlers (especially soft interrupts) called by the NIC after packet reception consume a lot of CPU, scheduling these interrupt handlers to a different CPU can significantly improve network throughput. This can usually be done in the following two ways.

- For example, you can configure CPU affinity (smp_affinity) for NIC hard interrupts, or enable the irqbalance service.
- For example, you can enable RPS (Receive Packet Steering) and RFS (Receive Flow Steering) to schedule application and soft interrupt processing to the same CPU, which increases CPU cache hit rate and reduces network latency.

In addition, NICs nowadays are very feature-rich, and functions that were originally handled by software in the kernel can be offloaded to the NIC and executed by hardware.

- TSO (TCP Segmentation Offload) and UFO (UDP Fragmentation Offload): Send large packets directly in TCP/UDP protocol; while the segmentation of TCP packets (segmentation according to MSS) and the fragmentation of UDP (fragmentation according to MTU) functions are performed by the NIC.
- GSO (Generic Segmentation Offload): When the NIC does not support TSO/UFO, the segmentation of TCP/UDP packets is delayed and executed before entering the NIC. This not only reduces CPU consumption, but also allows only the segmented packets to be retransmitted in case of packet loss.
- LRO (Large Receive Offload): When TCP segmented packets are received, they are assembled and merged by the NIC before being handed over to the upper layer network for processing. Note, however, that LRO cannot be enabled when IP forwarding is required, because LRO merging can cause network packet checksum errors if the header information of multiple packets is inconsistent.
- GRO (Generic Receive Offload): GRO fixes the flaws of LRO and is more generic, supporting both TCP and UDP.
- RSS (Receive Side Scaling): also known as multi-queue receive, it allocates network receive processes based on multiple receive queues in hardware, which allows multiple CPUs to process incoming network packets.
- VXLAN offload: This means that the NIC is allowed to perform the packet grouping function for VXLAN.

Finally, for the network interface itself, there are many ways to optimize the throughput of the network.

- For example, you can enable the multi-queue feature of the network interface. This way, each queue can be scheduled to execute on a different CPU with a different interrupt number, thus improving the network throughput.
- For example, you can increase the buffer size and queue length of the network interface to improve the throughput of network transfers (note that this may result in increased latency).
- You can also use the Traffic Control tool to configure QoS for different network traffic.

Up to this point, I have introduced the corresponding network performance optimization methods from application, socket, transport layer, network layer, and then link layer. With the optimization of these methods, the network performance can meet most of the scenarios.

Finally, don't forget an extreme scenario. Remember the C10M problem that we learned?

In a single-computer concurrent 10 million scenario, the various optimization strategies for the Linux network stack are basically ineffective. Because in this case, the lengthy process of the network stack is actually the main performance burden.

At this point, we can optimize it in two ways.

The first is to use the DPDK technique to skip the kernel stack and let the user-state process handle network requests directly by polling. This is combined with various mechanisms such as large pages, CPU binding, memory alignment, pipelined concurrency, etc. to optimize the efficiency of network packet processing.

The second, using the kernel's own XDP technology, processes network packets before they enter the kernel stack, which also achieves good performance.

## Summary

In these two lessons, we have combed through common Linux network performance optimization methods together.

When optimizing the performance of the network, we can combine the network protocol stack and network transceiver flow of the Linux system to optimize each layer layer by layer, from application, socket, transport layer, network layer to link layer, and so on.

In fact, we analyze and locate network bottlenecks based on these network layers as well. And after locating the network performance bottleneck, we can optimize based on the protocol layer where the bottleneck is located. Specifically:

- in the application layer, it is mainly about optimizing the I/O model, the working model, and the network protocols at the application layer;
- in the socket layer, it is mainly about optimizing the socket buffer size;
- in the transport layer, it is mainly about optimizing the TCP and UDP protocols;
- in the network layer, it is mainly about optimizing the routing, forwarding, sharding, and ICMP protocols;
- finally, in the link layer, it is mainly about optimizing the network packet sending and receiving, the Finally, in the link layer, the main focus is on optimizing network packet sending and receiving, network function offloading, and NIC options.

If these approaches still do not meet your requirements, then consider using a user-state approach such as DPDK to bypass the kernel stack or, alternatively, using XDP to process network packets before they enter the kernel stack.
