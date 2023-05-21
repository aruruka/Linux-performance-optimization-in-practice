# 43 Rouatines: A few ideas for network performance optimization (part 1)

In the last section, we learned about the principles of NAT (Network Address Translation), learned how to troubleshoot performance problems caused by NAT, and finally summarized the basic ideas for NAT performance optimization. Let me take you through a brief review.

NAT is based on the Linux kernel's connection tracking mechanism, which implements IP address and port number rewriting and is mainly used to solve the problem of public IP address shortage.

When analyzing NAT performance issues, you can start with the kernel connection tracking module `conntrack`, such as using tools like `systemtap`, `perf`, `netstat`, and the kernel options in the `proc` file system to analyze the behavior of the network stack; then, through kernel options tuning, switching to stateless NAT, and Use DPDK, etc., to perform the actual optimization.

As you should have learned earlier, network problems are more complex than the CPU, memory, or disk I/O we learned earlier. Both the various I/O models at the application layer, the lengthy network stack and numerous kernel options, and the various complex network environments all add to the complexity of networking.

However, don't worry too much, as long as you master the fundamentals of Linux networking and the workflow of common network protocols, and analyze them in conjunction with the performance metrics of each network layer, you will find that locating network bottlenecks is not difficult.

After finding the network performance bottleneck, the next step is to optimize it, that is, how to reduce the network latency and improve the network throughput. After learning the principles and cases, I'll talk about the ideas and some considerations for optimizing network performance problems.

Since there are more network optimization ideas, we will learn them in two sections, and today we will start with the previous one.

## Define optimization goals

As with CPU and I/O performance optimization, before I optimize, I ask myself, what is the goal of network performance optimization? In other words, what are the appropriate network performance metrics to observe?

In fact, while the overall goal of network performance optimization is to reduce network latency (e.g., RTT) and increase throughput (e.g., BPS and PPS), the optimization criteria for each metric may vary from application to application, and the order of priority may be quite different.

Take the NAT gateway mentioned in the previous section. Since it directly affects the network access performance of the entire data center, NAT gateways usually need to achieve or approach linear forwarding, which means that PPS is the primary performance target.

Then again, for systems such as databases and caches, rapid network send and receive, i.e., low latency, is the primary performance goal.

For Web services, which we access frequently, both throughput and latency are required.

Therefore, in order to evaluate the optimization effect more objectively and reasonably, we should first clarify the criteria for optimization, i.e., we should benchmark the system and application to get the benchmark performance of each layer of the network protocol stack.

In 🔗[How to evaluate the network performance of a system](https://time.geekbang.org/column/article/81497), I have already introduced, the method of network performance testing. To briefly review, the Linux network protocol stack, is the core principle that we need to master. It is based on the layered structure of the TCP/IP protocol family, and I'll represent this structure in a diagram.

<center><a href="https://ibb.co/FnJv6xx">
    <img src="https://i.ibb.co/JC2LnBB/Schematic-Of-The-Linux-Generic-Ip-Networking-Stack.png" alt="Schematic-Of-The-Linux-Generic-Ip-Networking-Stack" border="0">
</a></center>

Understanding this, when benchmarking, we can follow each layer of the protocol stack. Since the bottom layer is the foundation of the layers above it, the bottom layer performance also determines the top layer performance. So let's be clear that the bottom layer performance metrics, in fact, correspond to the extreme performance of the top layer. Let's understand this from the bottom to the top.

First is the network interface layer and the network layer, which are responsible for encapsulating, addressing, routing, and sending and receiving network packets. The number of network packets that can be processed per second, PPS, is their most important performance metric (especially in the case of small packets). You can use the kernel's own packet sending tool, `pktgen`, to test the performance of PPS.

Moving up to the transport layer, TCP and UDP are responsible for network transport. For them, throughput (BPS), connections, and latency are the most important performance metrics. You can use `iperf` or `netperf` to test the performance of the transport layer.

Note, however, that the size of the network packets directly affects the values of these metrics. So, typically, you will need to test the performance of a range of different sized network packets.

Finally, further up to the application layer, the most important metrics to focus on are throughput (BPS), requests per second, and latency. You can use tools like `wrk`, `ab`, etc. to test the performance of your application.

However, it is important to note here that the test scenario should try to simulate the production environment so that the test is more valuable. For example, you can go to the production environment, record the actual request situation, and then play it back in the test.

In short, based on these benchmark metrics, combined with the performance bottlenecks that have been observed, we can clarify the performance optimization goals.

## Network performance tools

As with the previous study, I recommend starting with two different dimensions, metrics and tools, to organize and memorize network-related performance tools.

The first dimension, starting with network performance metrics, makes it easier for you to correlate performance tools with how the system works and to have a macro view and grasp of performance issues. That way, when you want to look at a certain performance metric, you know exactly which tools are available.

Here, I have made a table of the tools that provide network performance metrics, so that you can easily sort out the relationships and understand the memory. You can save it and print it out to view at any time. Of course, you can also use it as a "metrics tools" guide.

<center><a href="https://ibb.co/zsF0fRR">
    <img src="https://i.ibb.co/0qKdJMM/Find-Tools-Based-On-Indicators-Network-Performance.png" alt="Find-Tools-Based-On-Indicators-Network-Performance" border="0">
</a></center>

Let's look at the second dimension, starting with the performance tools. This allows you to get up to speed with the tools more quickly and find out the performance metrics you want to observe quickly. Especially with limited tools, it is important to make the best use of every tool at hand and try to dig out a lot of information even with a small number of tools.

Likewise, I've put together a table of these common tools to make it easier for you to distinguish and understand them. Naturally, you can also use it as a "tool indicator" guide and check the table when you need it.

<center><a href="https://ibb.co/JCvBmYz">
    <img src="https://i.ibb.co/mS9DbnH/Check-Indicators-According-To-Tools-Network-Performance.png" alt="Check-Indicators-According-To-Tools-Network-Performance" border="0">
</a></center>

## Network performance optimization

In summary, the network benchmarking report is obtained first, and then the network performance bottlenecks are located through relevant performance tools. The next step is to optimize it.

Of course, as always, you can't optimize network performance without the help of the Linux network stack and network sending and receiving processes. You can recall this part of knowledge with the following diagram.

<center><a href="https://ibb.co/wSXGsdT">
    <img src="https://i.ibb.co/X7g08W9/Flow-Chart-Of-Receiving-Network-Packets.png" alt="Flow-Chart-Of-Receiving-Network-Packets" border="0">
</a></center>

Next, we can look at the basic ideas of network performance optimization from the application, socket, transport, network, and link layer perspectives, respectively.

### Applications

Applications, typically, operate over the network through a socket interface. Since network sending and receiving is usually time-consuming, the optimization of the application is mainly about optimizing the network I/O and the working model of the process itself.

We have actually learned about this in the 🔗[C10K and C1000K review](https://time.geekbang.org/column/article/81268) article. Let's briefly review it here.

From the perspective of network I/O, there are two main optimization ideas.

The first is epoll, the most commonly used I/O multiplexing technique, which is used to replace select and poll, and is actually the key to solving the C10K problem and the default mechanism used by many network applications.

The second is to use asynchronous I/O (AIO), which allows applications to initiate many I/O operations at the same time without waiting for them to complete. When the I/O completes, the system uses event notifications to tell the application the result. However, the use of AIO is more complex, and you need to be careful with many edge cases.

And in terms of the working model of the process, there are two different models used for optimization.

The first one is a master process + multiple worker sub-processes. In this case, the main process is responsible for managing network connections, while the child processes are responsible for the actual business processing. This is also the most common model.

The second model is a multi-process model that listens to the same port. In this model, all processes listen to the same interface and the SO_REUSEPORT option is turned on, and the kernel is responsible for load balancing the requests to these listening processes.

In addition to the network I/O and process working model, the optimization of the network protocol at the application layer is also a crucial point. I have summarized several common optimization methods.

- Using long connections instead of short ones can significantly reduce the cost of TCP connection establishment. This can be very effective when the number of requests per second is high.
- Using memory, for example, to cache infrequently changing data can reduce the number of network I/Os while speeding up the response time of the application.
- Using serialization, such as Protocol Buffer, to compress the amount of data on network I/O can improve application throughput.
- Reducing DNS resolution latency using DNS caching, prefetching, HTTPDNS, etc. can also improve the overall speed of network I/O.

### Socket

Sockets mask out the differences between different protocols in the Linux kernel and provide a uniform access interface for applications. For each socket, there is a read and write buffer.

- The read buffer, which caches data sent from the remote end. If the read buffer is full, no new data can be received.
- The write buffer caches the data to be sent out. If the write buffer is full, the application's write operation will be blocked.

So, to increase the throughput of the network, you usually need to resize these buffers. For example:

- increasing the per-socket buffer size net.core.optmem_max;
- increasing the socket receive buffer size net.core.rmem_max and the send buffer size net.core.wmem_max;
- increasing the TCP receive buffer size net.ipv4.tcp_rmem and the send buffer size net.ipv4.tcp_wmem.

As for the kernel options for sockets, I've organized them into a table for your easy reference if you need to:

<center><a href="https://ibb.co/W26djXj">
    <img src="https://i.ibb.co/CHzXDyD/List-Of-Socket-Kernel-Options.png" alt="List-Of-Socket-Kernel-Options" border="0">
</a></center>

But there are a few points you need to pay attention to.

- The three values of tcp_rmem and tcp_wmem are min, default, and max. The system will automatically adjust the TCP receive/send buffer size according to these settings.
- The three values for udp_mem are min, pressure, and max, and the system will automatically adjust the size of the UDP send buffer according to these settings.

Of course, the values in the table only provide reference values, and you need to determine the exact size according to the actual network conditions. For example, the ideal value for the send buffer size is throughput * latency, so that the maximum network utilization can be achieved.

In addition to this, the socket interface provides a number of configuration options to modify the behavior of the network connection:

- Setting TCP_NODELAY for TCP connections disables the Nagle algorithm;
- Enabling TCP_CORK for TCP connections allows small packets to be aggregated into larger packets before being sent (note that this blocks the sending of small packets);
- Using SO_SNDBUF and SO_RCVBUF, the size of the socket send buffer and receive buffer can be adjusted, respectively.

## Summary

Today, we've combed through common Linux network performance optimization methods together.

When optimizing network performance, you can combine the network protocol stack and network send/receive process of Linux system, and then optimize layer by layer from application, socket, transport layer, network layer to link layer, etc.

Of course, we actually analyze and locate network bottlenecks based on these as well. Once the performance bottleneck is located, optimization can be performed based on the protocol layer where the bottleneck is located. For example, today we learned the optimization ideas for applications and sockets:

- In the application, the main optimizations are the I/O model, the working model, and the network protocols at the application layer; - In the socket layer, the main optimizations are the buffer sizes of the sockets.

And the optimization methods for each other network layer, we suggest you think about them by yourself first, and in the next section, we will summarize them together.
