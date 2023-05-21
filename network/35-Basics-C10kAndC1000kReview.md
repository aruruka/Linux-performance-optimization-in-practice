# 35 Basics: C10K and C1000K Review

In the previous content, we learned the basic principles of Linux networking and performance observation methods. To briefly review, Linux network is based on the TCP/IP model, which constructs its network protocol stack and divides the complicated network functions into four different layers, such as application layer, transport layer, network layer, and network interface layer, which not only solves the problem of device heterogeneity in the network environment, but also decouples the complexity of network protocols.

Based on the TCP/IP model, we also sort out the Linux network sending and receiving process and the corresponding performance metrics. When an application sends or receives network packets through a socket interface, these network packets are processed layer by layer through the protocol stack. We usually measure network performance in terms of bandwidth, throughput, latency, PPS, etc.

Today, we will focus on reviewing the classic C10K and C1000K issues to better understand how Linux networking works and further analyze how to achieve single machine support for C10M.

Note that the initial C in C10K and C1000K stands for Client, and C10K is the problem of handling 10,000 simultaneous requests (10,000 concurrent connections) on a single machine, while C1000K is the problem of handling 1 million requests (1 million concurrent connections) on a single machine.

## C10K

The 🔗[C10K problem](http://www.kegel.com/c10k.html) was first proposed by Dan Kegel in 1999. At that time, the server was a 32-bit system running Linux version 2.2 (later upgraded to 2.4 and 2.6, which only supported x86_64), with very little memory (2GB) and a gigabit NIC.

How can you support 10,000 concurrent requests on such a system?

In terms of resources, for a server with 2GB of memory and a Gigabit NIC to process 10,000 requests simultaneously, each request can be processed with less than 200KB (2GB/10000) of memory and 100Kbit (1000Mbit/10000) of network bandwidth. Therefore, the physical resources are sufficient, and the next issue is naturally the software, especially the I/O model of the network.

When it comes to the I/O model, I have introduced file I/O in the principle of file system, in fact, the network I/O model is similar. Before C10K, network processing in Linux was done in a synchronous blocking way, which meant that each request was assigned to a process or thread. When the number of requests was only 100, this approach was fine, but when it increased to 10,000 requests, the scheduling of 10,000 processes or threads, context switching, and even the memory they occupied, all became bottlenecks.

Since the one-thread-per-request approach is not suitable, there are two problems we need to solve in order to support 10,000 concurrent requests.

First, how to handle multiple requests in a single thread, that is, to respond to multiple network I/Os in a single thread; the previous synchronous blocking approach, where a thread can only handle one request, is no longer applicable here, can we use non-blocking I/O or asynchronous I/O to handle multiple network requests?

Second, how to handle client requests more resource-efficiently, i.e., to use fewer threads to serve these requests. Is it possible to continue to use 100 or fewer threads to serve the current 10,000 requests?

Of course, the fact is that the C10K problem has been solved for a long time now, so you can think about these two problems for yourself before moving on to the next section. Do you have an idea of how to solve the problem in the context of what you've learned earlier?

### I/O model optimization

What does I/O multiplexing mean?

Don't worry, before we get into the details, I'll talk about two types of I/O event notification: level-triggering and edge-triggering, which are commonly used in the file descriptors of socket interfaces.

- **Level-triggered**: Notification is triggered whenever a file descriptor can perform I/O non-blockingly. That is, the application can check the status of the file descriptor at any time and then, depending on the status, perform I/O operations.
- **Edge triggered**: Notifications are sent only once when the state of the file descriptor changes (that is, when an I/O request is reached). At this point, the application needs to execute as much I/O as possible until it can no longer read or write, then it can stop. If the I/O doesn't finish executing, or if for some reason it doesn't get to process it, then this notification is lost.

Next, let's go back and look at the I/O multiplexing approach. There are actually a number of implementations here, so I'll take you through them one by one.

**The first, using non-blocking I/O and level-triggered notifications, such as using `select` or `poll`.**

According to the principle of level-triggering just now, `select` and `poll` need to find out from the list of file descriptors which can perform I/O and then do the actual network I/O reading and writing. Since I/O is non-blocking, a batch of socket file descriptors can be monitored in one thread at the same time, thus achieving the purpose of single-threaded processing of multiple requests.

So, the biggest advantage of this approach is that it is application friendly and has a very simple API.

However, when applications use `select` and `poll`, they need to poll the list of file descriptors, which can be time consuming when the number of requests is high. Also, `select` and `poll` have some other limitations.

`select` uses a fixed-length bit vector to represent a collection of file descriptors, so there is a limit to the maximum number of descriptors. For example, on 32-bit systems, the default limit is 1024, and within `select`, checking the status of a socket is done by polling, which takes O(N) time in relation to the number of descriptors.

`poll` improves the representation of `select` by replacing it with an array with no fixed length, so that there is no limit to the maximum number of descriptors (although there is a system file descriptor limit). However, when the application uses `poll`, it also needs to poll the file descriptor list, so the processing time is O(N) with the number of descriptors.

In addition, each time the application calls `select` and `poll`, it needs to pass the set of file descriptors from user space into kernel space, which is modified by the kernel and then passed back out into user space. This switching back and forth between kernel space and user space also increases the processing cost.

Is there a better way to handle this? The answer is naturally yes.

**The second, using non-blocking I/O and edge-triggered notifications, such as `epoll`.**

Since `select` and `poll` have so many problems, they need to continue to be optimized, and `epoll` is a good solution to these problems.

- `epoll` uses a red-black tree to manage the collection of file descriptors in the kernel so that there is no need for the application to pass in and out of this collection on every operation.
- `epoll` uses an event-driven mechanism to focus only on file descriptors that have I/O events occurring and does not require polling to scan the entire collection.

Note, however, that epoll is a feature that was only added in Linux 2.6 (2.4 also had it, but it was not fully functional). Since edge triggering only notifies when a file descriptor readable or writable event occurs, then applications need to perform as much I/O as possible and have to handle more exception events.

**The third, using Asynchronous I/O (AIO for short).** I covered the difference between asynchronous I/O and synchronous I/O in the previous section on file system principles. Asynchronous I/O allows an application to initiate many I/O operations at the same time without having to wait for them to complete. Instead, after the I/O completes, the system will tell the application with an event notification (such as a signal or callback function). Only then does the application look up the results of the I/O operation.

Asynchronous I/O was not supported until Linux 2.6, and was in an imperfect state for a long time, for example, the asynchronous I/O library provided by `glibc` was criticized by the community. Also, because asynchronous I/O is not quite the same as our intuitive logic, it must be carefully designed to be difficult to use.

### Work model optimization

Once the I/O model is understood, the optimization of request processing is more intuitive. With I/O multiplexing, multiple requests can be processed in a single process or thread, of which there are two different working models.

**The first one, main process + multiple worker sub-processes, is the most common model.** A general working model for this approach is:

- After the master process executes `bind()` + `listen()`, multiple child processes are created;
- then, in each child process, the same sockets are processed via `accept()` or `epoll_wait()`.

For example, this is how Nginx, the most commonly used reverse proxy server, works. It is also composed of a master process and multiple worker processes. The master process is mainly used to initialize the sockets and manage the lifecycle of the child processes, while the worker processes, on the other hand, are responsible for the actual request processing. I've drawn a diagram to represent this relationship.

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/h77BqjN/Working-Model-Main-Process-Worker-Subprocess-Model.png" alt="Working-Model-Main-Process-Worker-Subprocess-Model" border="0">
</a></center>

Note here that with the `accept()` and `epoll_wait()` calls, there is also the problem of swarming. In other words, when a network I/O event occurs, multiple processes are woken up at the same time, but only one process actually responds to the event, and all the other woken processes are put back to sleep.

- The problem with `accept()` was solved in Linux 2.6;
- the problem with `epoll` was not solved until Linux 4.5 with EPOLLEXCLUSIVE.

To avoid swarming problems, Nginx adds a global lock (accept_mutex) to each worker process. These worker processes need to compete for the lock first, and only those that do will be added to epoll, ensuring that only one worker child is woken up.

However, as you may recall from studying the CPU module earlier, process management, scheduling, and context switching are very costly. So why does Nginx, which uses a multi-process model, have very good performance?

One of the main reasons is that these worker processes, which don't actually need to be created and destroyed very often, sleep when there is no task and wake up when there is. Only when the worker quits due to some exception does the main process need to create a new process to replace it.

Of course, you can also use threads instead of processes: the main thread is responsible for socket initialization and management of the state of the child threads, while the child threads are responsible for the actual request processing. Since threads are cheaper to schedule and switch, you can actually go a step further and put `epoll_wait()` all in the main thread, ensuring that only the main thread wakes up for each event, while the child threads are only responsible for the subsequent request processing.

**The second, multi-process model that listens to the same ports.** In this approach, all processes listen to the same interface and the SO_REUSEPORT option is turned on, and the kernel is responsible for load balancing the requests to these listening processes. This process is shown in the figure below.

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/XWNGvkr/Working-Model-Multi-Process-Listening-To-The-Same-Port-Model.png" alt="Working-Model-Multi-Process-Listening-To-The-Same-Port-Model" border="0">
</a></center>

Since the kernel ensures that only one process is woken up, there are no problems with swarming. For example, Nginx already supports this mode as of 1.9.1.

<center><a href="https://ibb.co/XpFTJD1">
    <img src="https://i.ibb.co/82PV8bq/Nginx-Multi-Process-Listening-To-The-Same-Port-Model.png" alt="Nginx-Multi-Process-Listening-To-The-Same-Port-Model" border="0">
</a></center>

(image from the 🔗[Nginx official blog](https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/))

Note, however, that to use the SO_REUSEPORT option, you need Linux 3.9 or higher.

## C1000K

Based on I/O multiplexing and request processing optimization, the C10K problem is easily solved. However, with the increase in server performance brought about by Moore's Law and the popularity of the Internet, it's not hard for you to imagine that emerging services will demand even higher performance.

Soon, the original C10K could no longer meet the demand, so there were C100K and C1000K, which meant that concurrency increased from 10,000 to 100,000, and even 1 million. The increase from 10,000 to 100,000 is still based on the theory of C10K, `epoll` with thread pooling, plus CPU, memory and network interface performance and capacity improvements. In most cases, C100K is a natural reach.

So, to take it a step further, can C1000K be easily achieved as well? It's not as simple as that.

First of all, in terms of physical resource usage, 1 million requests requires a lot of system resources. For example

- Assuming that each request requires 16KB of memory, then a total of about 15 GB of memory is required.
- And in terms of bandwidth, assuming only 20% active connections, even if each connection requires only 1KB/s of throughput, the total throughput would be 1.6 Gb/s. A Gigabit NIC is obviously not sufficient for such a large throughput, so a 10 Gigabit NIC, or a multi-NIC based Bonding, is required to carry a larger throughput.

Secondly, in terms of software resources, a large number of connections will also take up a lot of software resources, such as the number of file descriptors, connection status tracking (CONNTRACK), cache size of the network stack (e.g., socket read/write cache, TCP read/write cache), and so on.

Finally, the interruptions caused by the large number of requests can also result in very high processing costs. This requires various hardware and software optimizations such as multi-queued NICs, interrupt load balancing, CPU binding, RPS/RFS (soft interrupt load balancing to multiple CPU cores), and offloading (Offload) network packet processing to network devices (e.g. TSO/GSO, LRO/GRO, VXLAN OFFLOAD).

The C1000K solution is still essentially built on the `epoll` non-blocking I/O model. However, in addition to the I/O model, it requires deep optimization at all levels, from the application to the Linux kernel, to the CPU, memory, and network, and in particular, hardware to offload a large number of functions that were originally handled by software.

### C10M

Obviously, there is no end to the demand for performance. Taking it a step further, is it possible to handle 10 million requests simultaneously in a single machine? This is also the 🔗[C10M](http://c10m.robertgraham.com/p/blog-page.html) problem.

In fact, in the C1000K problem, various software and hardware optimizations are likely to have been done to the head. Especially after upgrading the hardware (e.g., enough memory, a network card with enough bandwidth, more network features offloaded, etc.), you may find that it is extremely difficult to achieve 10 million requests concurrently, no matter how you optimize the various network parameters in the application and the kernel.

The root cause is that the Linux kernel stack is doing too much work. Starting from the hard interrupt handler brought by the NIC interrupt, to the various layers of network protocol processing in the soft interrupt, and finally to the application, the path is just too long, which leads to optimized processing of network packets, and after a certain point, it is impossible to go any further.

To solve this problem, the most important thing is to skip the long path through the kernel protocol stack and send the network packets directly to the application to be processed. There are two common mechanisms here, DPDK and XDP.

The first mechanism, DPDK, is the standard for user-state networking. It skips the kernel stack and handles network reception directly by the user-state process by polling.

<center><a href="https://ibb.co/0qQM85N">
    <img src="https://i.ibb.co/rkv6T9P/The-Standard-Of-User-Mode-Network-Dpdk.png" alt="The-Standard-Of-User-Mode-Network-Dpdk" border="0">
</a></center>

(Image from 🔗<https://blog.selectel.com/introduction-dpdk-architecture-principles/>)

When it comes to polling, you will certainly subconsciously think it is a symbol of inefficiency, but further ask yourself, where is its inefficiency mainly reflected? It's when the query time is significantly more than the actual working time! Well, think about it another way, if there are new network packets to be processed every moment, the advantages of polling are obvious. For example:

- In a very high PPS scenario, the query time is much less than the actual working time, and most of the time is spent processing network packets;
- skipping the kernel stack eliminates the tedious layer-by-layer process of hard interrupts, soft interrupts, and then the Linux network stack, and the application can target and optimize the network packet processing logic for the actual application scenario without focusing on all the details.

In addition, DPDK optimizes network packet processing efficiency through various mechanisms such as large pages, CPU binding, memory alignment, pipelined concurrency, etc.

The second mechanism, XDP (eXpress Data Path), is a high-performance network data path provided by the Linux kernel. XDP is based on the Linux kernel's eBPF mechanism, like bcc-tools, which we used earlier.

The principle of XDP is shown in the following diagram:

<center><a href="https://ibb.co/VYwn2wz">
    <img src="https://i.ibb.co/GQHYcH8/XDP-Principles.png" alt="XDP-Principles" border="0">
</a></center>

(Image from 🔗<https://www.iovisor.org/technology/xdp>)

As you can see, XDP is more demanding on the kernel, requiring Linux 🔗 [version 4.8 or higher](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md#xdp), and it does not provide cache queues. XDP-based applications are usually dedicated network applications, commonly IDS (Intrusion Detection System), DDoS prevention, 🔗[cilium](https://github.com/cilium/cilium) container network plugins, etc.

## Summary

Today I take you through the classic C10K problem and extend it further to the C1000K and C10M problems.

The root cause of the C10K problem, on the one hand, lies in the limited resources of the system; on the other hand, and a more important factor, is the synchronous blocking I/O model and the polling socket interface, which limit the efficiency of processing network events. `epoll`, introduced in Linux 2.6, solved the C10K problem perfectly, and nowadays, high-performance network solutions are based on `epoll`.

From C10K to C100K, you may only need to increase the physical resources of the system to meet; but from C100K to C1000K, it is not only a problem that can be solved by increasing physical resources. At this point, optimization work is needed in many aspects, from hardware interrupt handling and network function offloading, to the number of file descriptors in the network protocol stack, connection state tracking, cache queueing and other kernel optimizations, to the optimization of the working model of the application, are the key considerations.

Taking it a step further, to implement C10M, it is not just a matter of adding physical resources or optimizing the kernel and application that can be solved. It is time to use XDP to process network packets before the kernel stack, or DPDK to skip the network stack and process network packets directly in user space by polling.

Of course, in reality, in most scenarios, we don't need 10 million concurrent requests on a single machine. It is often simpler and more scalable to distribute these requests across multiple servers by adjusting the system architecture to handle them.
