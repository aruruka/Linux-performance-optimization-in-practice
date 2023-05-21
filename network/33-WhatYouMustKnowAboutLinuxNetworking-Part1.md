# 33 What you must know about Linux networking(part-1)

In the previous sections, we learned together how the file system and disk I/O work, and the corresponding performance analysis and optimization methods. Next, we will move on to the next important module - the network subsystem of Linux.

Since the network process is the most complex and is inextricably linked to the process scheduling, interrupt handling, memory management, and I/O we talked about earlier, I will cover the network module as the last resource module.

Like CPU, memory, and I/O, networking is also the core function of a Linux system. Networking is a technology that connects different computers or network devices together. It is essentially a way of inter-process communication, especially inter-process communication across systems, which must be done over the network. With the popularity of technologies such as high concurrency, distributed, cloud computing, and microservices, the performance of the network is becoming more and more important.

So, how does a Linux network work? And what are the metrics to measure the performance of the network? In the next two articles, I'll take you through the workings and performance metrics of Linux networks.

## Network model

When it comes to networking, I'm sure you often mention Layer 7 load balancing, Layer 4 load balancing, or Layer 3 devices, Layer 2 devices, and so on. So, what do we mean here by Layer 2, Layer 3, Layer 4, and Layer 7?

In fact, these layers are from the International Organization for Standardization developed **Open System Interconnection Reference Model**, referred to as the OSI network model.

In order to solve the compatibility problem of heterogeneous devices in network interconnection and decouple the complex network packet processing process, the OSI model divides the framework of network interconnection into seven layers, such as application layer, representation layer, session layer, transport layer, network layer, data link layer and physical layer, each layer is responsible for different functions. Among them, the

- The application layer, which is responsible for providing a uniform interface to the application.
- The representation layer, which is responsible for converting data into a format compatible with the receiving system.
- The session layer, which is responsible for maintaining communication connections between computers.
- The transport layer, which is responsible for adding a transport header to the data to form packets.
- The network layer, which is responsible for routing and forwarding data.
- The data link layer, which is responsible for MAC addressing, error detection, and error correction.
- The physical layer, which is responsible for transmitting data frames across the physical network.

But the OSI model is still too complex and doesn't provide an implementable approach. So, in Linux, we actually use another more practical four-layer model, the TCP/IP network model.

The TCP/IP model, which divides the framework of network interconnection into four layers, including the application layer, the transport layer, the network layer, and the network interface layer, where the

- The application layer, which is responsible for providing a set of applications to users, such as HTTP, FTP, DNS, etc.
- The transport layer, responsible for end-to-end communication, such as TCP, UDP, etc.
- Network layer, responsible for encapsulation, addressing and routing of network packets, such as IP, ICMP, etc.
- The network interface layer, which is responsible for the transmission of network packets across the physical network, such as MAC addressing, error detection, and transmission of network frames through the NIC.

To help you understand the relationship between TCP/IP and the OSI model more visually, I have drawn a diagram as follows:

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/zRjz1Yz/Tcp-Ip-And-Osi.png" alt="Tcp-Ip-And-Osi" border="0">
</a></center>

Of course, although Linux actually implements a network protocol stack according to the TCP/IP model, we are used to describing it in the OSI seven-layer model in our usual learning exchanges. For example, when it comes to seven-layer and four-layer load balancing, they correspond to the application and transport layers of the OSI model (while they correspond to the TCP/IP model, which is actually four and three layers).

The TCP/IP model includes a large number of network protocols, and the principles of these protocols are the core fundamentals that each of us must master. If you are not too proficient, we recommend you to learn Volume I and Volume II of "TCP/IP in Detail" (W. Richard Stevens), or study the column "[Interesting Talk on Network Protocols](https://time.geekbang.org/column/intro/100007101)" produced by Geek Time.

## Linux network stack

With the TCP/IP model, when network transmission is performed, packets are processed layer by layer according to the protocol stack, with the data sent from the previous layer; then the protocol header of that layer is encapsulated and sent to the next layer.

Of course, the processing logic of network packets at each layer depends on the network protocol used at each layer. For example, at the application layer, an application that provides a REST API can use the HTTP protocol, encapsulate the JSON data it needs to transmit into the HTTP protocol, and then pass it down to the TCP layer.

What encapsulation does is simple: it adds fixed-format metadata before and after the original load, and the original load data is not modified.

For example, using a network packet communicating over the TCP protocol as an example, we can see the encapsulation format of the application data at each layer with the following diagram.

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/SmtbCqc/Network-Packets-For-Tcp-Protocol-Communication.png" alt="Network-Packets-For-Tcp-Protocol-Communication" border="0">
</a></center>

Among them:

- The transport layer adds a TCP header before the application data;
- The network layer adds an IP header before the TCP packet;
- The network interface layer, in turn, adds a frame header and frame tail before and after the IP packet.

The headers and tails of these additions are populated according to a specific protocol format, and to understand the exact format, you can check the protocol's documentation. For example, you can look at 🔗 [here](https://zh.wikipedia.org/wiki/%E4%BC%A0%E8%BE%93%E6%8E%A7%E5%88%B6%E5%8D%8F%E8%AE%AE#%E5%B0%81%E5%8C%85%E7%B5%90%E6%A7%8B) for the format of TCP headers.

These added headers and tails increase the size of network packets, but we all know that packets of arbitrary size cannot be transmitted on a physical link. The maximum transmission unit (MTU) configured for the network interface specifies the maximum IP packet size. In our most common use of Ethernet, the MTU defaults to 1500 (which is also the default for Linux).

Once a network packet exceeds the MTU size, it is sharded at the network layer to ensure that the sharded IP packet is no larger than the MTU value. Obviously, the larger the MTU, the less packets need to be sliced and, naturally, the better the network throughput capability.

After understanding the TCP/IP network model and the encapsulation principle of network packets, you can easily think that the network stack in the Linux kernel is actually similar to the four-layer structure of TCP/IP. A diagram of the Linux generic IP network stack is shown in the following figure:

<center><a href="https://ibb.co/FnJv6xx">
    <img src="https://i.ibb.co/JC2LnBB/Schematic-Of-The-Linux-Generic-Ip-Networking-Stack.png" alt="Schematic-Of-The-Linux-Generic-Ip-Networking-Stack" border="0">
</a></center>

(Image referenced in "Systems Performance" (Brendan Gregg), Figure 10.7 Generic IP Network Stack Drawing)

As we look at this network stack from top to bottom, you can see that

- At the top, the application needs to interact with the socket interface through system calls;
- Below the sockets are the transport, network and network interface layers we mentioned earlier;
- At the bottom, the NIC drivers and physical NIC devices.

Here I will briefly talk about the network card. The NIC is the basic device that sends and receives network packets. During the system boot process, the NIC is registered to the system through the NIC driver in the kernel. During network sending and receiving, the kernel interacts with the NIC through interrupts.

Combined with the Linux network stack mentioned earlier, it can be seen that the processing of network packets is very complex. Therefore, the NIC hard interrupts handle only the core NIC data reading or sending, while most of the logic in the protocol stack is put into soft interrupts.

### Linux Network Sending and Receiving Process

After understanding the Linux network stack, let's take a look at how Linux actually sends and receives network packets.

> Note that the following are examples of physical NICs.
> In fact, Linux also supports numerous virtual network devices, and their network sending and receiving processes can differ somewhat.

#### Flow of receiving network packets

Let's look at the process of receiving network packets.

When a network frame arrives at the NIC, the NIC puts this network packet into the receive queue by DMA; then, through a hard interrupt, it tells the interrupt handler that the network packet has been received.

Next, the NIC interrupt handler allocates a kernel data structure (sk_buff) for the network frame and copies it to the sk_buff buffer; then, via a soft interrupt, it notifies the kernel that a new network frame has been received.

Next, the kernel stack takes the network frame out of the buffer and processes it layer by layer, from bottom to top, through the network stack. For example.

- The legality of the packet is checked at the link layer to find out the type of the upper layer protocol (e.g. IPv4 or IPv6), then the header and tail of the frame are removed and handed over to the network layer.
- The network layer takes out the IP header and determines where the network packet will go next, such as whether to hand it off to the upper layer for processing or forwarding. Once the network layer confirms that the packet is to be sent to the local machine, it takes out the upper layer protocol type (e.g., TCP or UDP), removes the IP header, and passes it to the transport layer for processing.
- After the transport layer takes out the TCP header or UDP header, it finds out the corresponding socket based on the < source IP, source port, destination IP, destination port > quadruplet as the identifier and copies the data to the socket's receive cache.

Finally, the application can use the socket interface and read the newly received data.

To represent this flow more clearly, I drew a diagram. The left half of this diagram represents the receive flow, while the pink arrows in the diagram represent the network packet processing path.

<center><a href="https://ibb.co/wSXGsdT">
    <img src="https://i.ibb.co/X7g08W9/Flow-Chart-Of-Receiving-Network-Packets.png" alt="Flow-Chart-Of-Receiving-Network-Packets" border="0">
</a></center>

#### Flow of sending network packets

Once you understand the receiving flow of a network packet, it is easy to understand the sending flow of a network packet. The sending flow of a network packet is the right half of the diagram above, and it is easy to see that the sending direction of a network packet is exactly the opposite of the receiving direction.

First, the application calls the socket API (e.g. `sendmsg`) to send the network packet.

Since this is a system call, it gets caught in the socket layer of the kernel state. The socket layer puts the packet into the socket send buffer.

Next, the network stack takes the packet out of the socket send buffer; it then follows the TCP/IP stack and processes it layer by layer, from top to bottom. For example, the transport and network layers add TCP and IP headers respectively, perform route lookup to confirm the IP of the next hop, and slice according to MTU size.

The sliced network packet is then sent to the network interface layer for physical address addressing to find the MAC address of the next hop. Then frame headers and frame tails are added and placed in the outgoing packet queue. Once this is done, a soft interrupt notifies the driver that there are new network frames in the outgoing packet queue that need to be sent.

Finally, the driver reads the network frame from the packet queue via DMA and sends it out through the physical NIC.

## Summary

In today's article, I take you through the workings of a Linux network.

Multiple servers are connected together through network devices such as NICs, switches, and routers to form an interconnected network. Due to the heterogeneity of network devices and the complexity of network protocols, the International Organization for Standardization defined a seven-layer OSI network model, but this model is too complex and the de facto standard for practical work, is the more practical TCP/IP model.

The TCP/IP model, the framework for interconnecting networks, is divided into four layers: the application layer, the transport layer, the network layer, and the network interface layer, which are the most central components of the Linux network stack.

- The application sends packets through the socket interface, first going through layer-by-layer processing in the network stack from top to bottom before finally being sent to the NIC to be sent out.
- And when it is received, it also goes through the same layer-by-layer processing from bottom to top of the network stack before it is finally sent to the application.

After understanding the basics of Linux networking and the sending and receiving process, you'll want to know how to observe the performance of your network. So, specifically, what are the metrics to measure the performance of a Linux network? Don't worry, I'll explain it in detail in the next section.
