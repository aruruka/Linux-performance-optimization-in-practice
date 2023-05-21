# 34 What you must know about Linux networking(part-2)

In the last section, I took you through the fundamentals of Linux networking. To briefly review, Linux networks build their network stack based on the TCP/IP model, which consists of four layers: the application layer, the transport layer, the network layer, and the network interface layer, which are the core components of the Linux network stack.

When the application sends packets through the socket interface, they are first processed from top to bottom in the network protocol stack before they are finally sent out to the network card; when they are received, they are also processed from bottom to top in the network stack before they are finally sent to the application.

Once you understand the basics of Linux networking and the sending and receiving process, you'll want to know how to observe the performance of your network. Specifically, what metrics can be used to measure the performance of a Linux network?

## Profiling metrics

In fact, we usually measure the performance of a network with metrics such as bandwidth, throughput, latency, and PPS (Packet Per Second).

- **Bandwidth**, which indicates the maximum transmission rate of the link, usually in b/s (bits/sec).
- **Throughput**, which indicates the amount of data successfully transferred per unit time, usually in b/s (bits/s) or B/s (bytes/s). Throughput is limited by bandwidth, and throughput/bandwidth, which is the utilization rate of the network.
- **Delay**, which indicates the time delay from the time a network request is made until a response is received from the far end. In different scenarios, this metric may have different meanings. For example, it may indicate the time it takes to establish a connection (e.g., TCP handshake delay), or the time it takes for a packet to make a round trip (e.g., RTT).
- **PPS**, which stands for Packet Per Second, indicates the transmission rate in terms of network packets. pps is often used to evaluate the forwarding capability of a network, such as a hardware switch, which can usually achieve linear forwarding (i.e., pps can be at or near its theoretical maximum). Linux server-based forwarding, on the other hand, is susceptible to the effects of network packet size.

In addition to these metrics, **network availability** (whether the network can communicate properly), **concurrent connections** (number of TCP connections), **packet loss rate** (percentage of packets lost), **retransmission rate** (percentage of network packets retransmitted), etc. are also common performance metrics.

Next, please open a terminal, SSH to the server, and join me to explore and observe these performance metrics.

## Network configuration

The first step in analyzing a network problem is usually to look at the configuration and status of the network interface. You can use the `ifconfig` or `ip` commands to see the configuration of the network. I personally recommend using the `ip` tool because it provides a richer feature set and an easier-to-use interface.

> `ifconfig` and `ip` are part of the packages net-tools and iproute2, respectively. iproute2 is the next generation of net-tools.
> They are normally installed by default in the distribution. However, if you cannot find the `ifconfig` or `ip` commands, you can install both packages.

Using network interface eth0 as an example, you can run the following two commands to view its configuration and status:

```shell
$ ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST> mtu 1500
      inet 10.240.0.30 netmask 255.240.0.0 broadcast 10.255.255.255
      inet6 fe80::20d:3aff:fe07:cf2a prefixlen 64 scopeid 0x20<link>
      ether 78:0d:3a:07:cf:3a txqueuelen 1000 (Ethernet)
      RX packets 40809142 bytes 9542369803 (9.5 GB)
      RX errors 0 dropped 0 overruns 0 frame 0
      TX packets 32637401 bytes 4815573306 (4.8 GB)
      TX errors 0 dropped 0 overruns 0 carrier 0 collisions 0

$ ip -s addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
  link/ether 78:0d:3a:07:cf:3a brd ff:ff:ff:ff:ff:ff
  inet 10.240.0.30/12 brd 10.255.255.255 scope global eth0
      valid_lft forever preferred_lft forever
  inet6 fe80::20d:3aff:fe07:cf2a/64 scope link
      valid_lft forever preferred_lft forever
  RX: bytes packets errors dropped overrun mcast
   9542432350 40809397 0       0       0       193
  TX: bytes packets errors dropped carrier collsns
   4815625265 32637658 0       0       0       0
```

As you can see, the `ifconfig` and `ip` commands output basically the same metrics, with only a slightly different display format. For example, they both include network interface status flags, MTU size, IP, subnet, MAC address, and statistics on network packets sent and received.

The meaning of these specific indicators is explained in detail in the documentation, but here are a few indicators that are closely related to network performance and require your special attention.

First, the status flag of the network interface. RUNNING in the `ifconfig` output, or LOWER_UP in the `ip` output, both indicate that the physical network is connected, i.e. the NIC is connected to the switch or router. If you do not see them, it usually means that the network cable is unplugged.

Second, the size of the MTU, which is 1500 by default and may need to be adjusted up or down depending on the network architecture (e.g., whether you are using an overlay network such as VXLAN).

Third, the IP address, subnet, and MAC address of the network interface. These are necessary for proper network function, and you need to make sure they are configured correctly.

Fourth, the number of bytes, packets, errors, and packet loss sent and received by the network. In particular, errors, dropped, overruns, carriers, and collisions in the TX and RX sections are usually indicative of network I/O problems when they are not zero. Among them:

- **errors** indicates the number of packets with errors, such as checksum errors, frame synchronization errors, etc.;
- **dropped** indicates the number of packets dropped, i.e., packets received in the Ring Buffer but dropped due to insufficient memory, etc.;
- **overruns** indicates the number of packets overrun, i.e., packets dropped due to excessive network I/O speed **carrier** indicates the number of packets with carrirer errors, such as duplex mode mismatch, physical cable problems, etc.;
- **collisions** indicates the number of collision packets.

### Socket information

`ifconfig` and `ip` only show statistics on packets sent and received by the network interface, but in the real world of performance problems, there are statistics in the network stack that we must also pay attention to. You can use `netstat` or `ss` to see the information of sockets, network stack, network interfaces and routing tables.

I personally recommend, using `ss` to query the network connection information, because it provides better performance (faster) than `netstat`.

For example, you can execute the following command to query the socket information:

```shell
# head -n 3 表示只显示前面3行
# -l 表示只显示监听套接字
# -n 表示显示数字地址和端口(而不是名字)
# -p 表示显示进程信息
$ netstat -nlp | head -n 3
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      840/systemd-resolve

# -l 表示只显示监听套接字
# -t 表示只显示 TCP 套接字
# -n 表示显示数字地址和端口(而不是名字)
# -p 表示显示进程信息
$ ss -ltnp | head -n 3
State    Recv-Q    Send-Q        Local Address:Port        Peer Address:Port
LISTEN   0         128           127.0.0.53%lo:53               0.0.0.0:*        users:(("systemd-resolve",pid=840,fd=13))
LISTEN   0         128                 0.0.0.0:22               0.0.0.0:*        users:(("sshd",pid=1459,fd=3))
```

The output of `netstat` and `ss` is similar, both show the status of the socket, the receive queue, the send queue, the local address, the remote address, the process PID and the process name.

Of these, the receive-queue (Recv-Q) and send-queue (Send-Q) need your special attention, they should usually be 0. When you find that they are not 0, it means that a network packet buildup is happening. Also note, of course, that they mean different things in different socket states.

When a socket is in the connected state (Established)

- Recv-Q indicates the number of bytes in the socket buffer that have not yet been fetched by the application (i.e., the length of the receive queue).
- Send-Q indicates the number of bytes that have not yet been acknowledged by the remote host (i.e., the length of the send queue).

When the socket is in the Listening state (Listening), the

- Recv-Q indicates the length of the full connection queue.
- And Send-Q indicates the maximum length of the full connection queue.

A full connection means that the server has received the ACK from the client, completed the TCP triple handshake, and then moved the connection to the full connection queue. The sockets in these full connections also need to be fetched by the `accept()` system call before the server can start actually processing the client's request.

In contrast to the full connection queue, there is a half connection queue. A half-connection is a connection that has not yet completed the three TCP handshakes; the connection is only halfway through. Once the server receives a SYN packet from the client, it puts the connection into the half-connection queue and then sends a SYN+ACK packet to the client.

### Protocol stack statistics

Similarly, using `netstat` or `ss`, you can also view information about the protocol stack: `netstat` or `ss`:

```shell
$ netstat -s
...
Tcp:
    3244906 active connection openings
    23143 passive connection openings
    115732 failed connection attempts
    2964 connection resets received
    1 connections established
    13025010 segments received
    17606946 segments sent out
    44438 segments retransmitted
    42 bad segments received
    5315 resets sent
    InCsumErrors: 42
...

$ ss -s
Total: 186 (kernel 1446)
TCP:   4 (estab 1, closed 0, orphaned 0, synrecv 0, timewait 0/0), ports 0

Transport Total     IP        IPv6
*    1446      -         -
RAW    2         1         1
UDP    2         2         0
TCP    4         3         1
...
```

The statistics for each of these protocol stacks are very intuitive. While `ss` only shows summary statistics such as connected, closed, orphaned sockets, etc., `netstat` provides more detailed information about the network protocol stack.

For example, the example output from `netstat` above shows various information such as active connections, passive connections, failed retries, number of segments sent and received, etc. for the TCP protocol.

### Network throughput and PPS

Next, let's see how to view the current network throughput and PPS of the system. here, I recommend using our old friend `sar`, which we have used several times in the previous CPU, memory and I/O modules.

Adding the -n parameter to `sar` will allow you to view network statistics such as network interfaces (DEV), network interface errors (EDEV), TCP, UDP, ICMP, and so on. You can get network interface statistics by executing the following command:

```shell
# 数字1表示每隔1秒输出一组数据
$ sar -n DEV 1
Linux 4.15.0-1035 (ubuntu)   01/06/19   _x86_64_  (2 CPU)

13:21:40        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
13:21:41         eth0     18.00     20.00      5.79      4.25      0.00      0.00      0.00      0.00
13:21:41      docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
13:21:41           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```

There are a lot of output indicators here, so I'll briefly explain what they mean.

- **rxpck/s** and **txpck/s** are the received and transmitted PPS in packets/sec, respectively.
- **rxkB/s** and **txkB/s** are the receive and transmit throughput in KB/sec, respectively.
- **rxcmp/s** and **txcmp/s** are the number of compressed packets received and sent, respectively, in packets/sec.
- **%ifutil** is the utilization of the network interface, i.e., (rxkB/s+txkB/s)/Bandwidth in half-duplex mode and max(rxkB/s, txkB/s)/Bandwidth in full-duplex mode.

One of them, Bandwidth, can be queried with `ethtool`, and it is usually in Gb/s or Mb/s, but note that here the lowercase b, stands for **bits** rather than **bytes**. We usually refer to Gigabit NICs, 10 Gigabit NICs, etc., and the units are also bits. As you can see below, my eth0 NIC is a Gigabit NIC:

```shell
$ ethtool eth0 | grep Speed
  Speed: 1000Mb/s
```

### Connectivity and latency

Finally, we usually use `ping` to test the connectivity and latency of a remote host, which is based on the ICMP protocol. For example, you can test the connectivity and latency of the local machine to the IP address 114.114.114.114 by executing the following command:

```shell
# -c3表示发送三次ICMP包后停止
$ ping -c3 114.114.114.114
PING 114.114.114.114 (114.114.114.114) 56(84) bytes of data.
64 bytes from 114.114.114.114: icmp_seq=1 ttl=54 time=244 ms
64 bytes from 114.114.114.114: icmp_seq=2 ttl=47 time=244 ms
64 bytes from 114.114.114.114: icmp_seq=3 ttl=67 time=244 ms

--- 114.114.114.114 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 244.023/244.070/244.105/0.034 ms
```

The output of ping can be divided into two parts.

- The first part, for each ICMP request, contains information about the ICMP sequence number (icmp_seq), the TTL (time to live, or hop count), and the round-trip delay.
- The second part, then, is a summary of the three ICMP requests.

For example, the example above shows that 3 network packets were sent and 3 responses were received with no packet loss, indicating that the test host is connected to 114.114.114.114; the average round-trip delay (RTT) is 244ms, i.e., 244ms from the time the ICMP is sent to the time the confirmation is received back from 114.114.114.114.

## Summary

We usually use bandwidth, throughput, latency, and other metrics to measure the performance of a network; accordingly, you can use tools like `ifconfig`, `netstat`, `ss`, `sar`, `ping`, and so on to check these network performance metrics.

In the next section, I will take you further into the workings of Linux networks with the classic C10K and C100K problems.
