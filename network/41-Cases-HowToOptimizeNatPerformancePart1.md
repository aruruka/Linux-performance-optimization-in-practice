# 41 Cases: How to Optimize NAT Performance? (part 1)

In the previous section, we explored the analysis of network latency increase problems and mastered how to troubleshoot and locate the root cause of the problem with a case study using tools such as `hping3`, `tcpdump`, `Wireshark`, `strace`, and so on.

To briefly review, network latency is the most core network performance metric. Network latency is inevitable due to various factors such as network transmission and network packet processing. However, excessive network latency will directly affect the user experience.

Therefore, after finding out the increase of network latency, you can first analyze the network latency from various layers, such as routing, network packet sending and receiving, network packet processing, and then application, etc. Wait until you find out the source layer of network latency, and then locate the bottleneck in depth.

Today, I'll take a look at another factor that can cause network latency, namely Network Address Translation (NAT), for short.

Next, let's learn how NAT works and figure out how to optimize for potential performance issues caused by NAT.

## Principles of NAT

NAT technology rewrites the source or destination IP of IP packets and is commonly used to solve the problem of public IP address shortage. Its main principle is that multiple hosts in a network share the same public IP address to access external network resources. At the same time, since NAT blocks the intranet network, it naturally provides security isolation for the machines in the LAN.

You can configure NAT either in a router that supports network address translation (called a NAT gateway) or in a Linux server, which actually acts as a "soft" router if the second approach is used.

The main purpose of NAT is to enable address translation. Depending on how it is implemented, NAT can be divided into three categories:

- Static NAT, where the intranet IPs are permanently mapped to the public IPs in a one-to-one relationship;
- Dynamic NAT, where the intranet IPs are dynamically selected from the public IP pool for mapping;
- Network Address and Port Translation (NAPT), where the intranet IPs are mapped to different ports of the public IPs, allowing Multiple intranet IPs can share the same public IP address.

NAPT is currently the most popular type of NAT, and the NAT we configure in Linux is also of this type. Depending on the conversion method, we can divide NAPT into three categories.

The first type is Source Address Translation SNAT, where the destination address remains unchanged and only the source IP or source port is replaced, SNAT is mainly used when multiple intranet IPs share the same public IP to access external resources.

The second type is Destination Address Translation DNAT, where the source IP remains unchanged and only the destination IP or destination port is replaced. DNAT is mainly used to access various services on the intranet through different port numbers of public IPs, while hiding the real IP address of the backend server.

The third type is bidirectional address translation, which uses both SNAT and DNAT. when a network packet is received, DNAT is executed to convert the destination IP to an intranet IP, while when a network packet is sent, SNAT is executed to replace the source IP with an external IP.

Two-way address translation is actually a one-to-one mapping relationship between the external IP and the internal IP, so it is commonly used in virtualized environments to assign floating public IP addresses to virtual machines.

To help you understand NAPT, I've drawn a diagram. Let's assume that

- The internal IP address of the local server is 192.168.0.2;
- The public IP address in the NAT gateway is 100.100.100.100;
- The destination server <ins>baidu.com</ins> to be accessed is 123.125.115.110.

The process of SNAT and DNAT, then, is shown in the following diagram:

<center><a href="https://ibb.co/5h1v782">
    <img src="https://i.ibb.co/d4f6h27/NAPT-Two-Way-Snat-Dnat.png" alt="NAPT-Two-Way-Snat-Dnat" border="0">
</a></center>

From the diagram, you can see that

- When the server accesses <ins>baidu.com</ins>, the NAT gateway replaces the source address, from the server's intranet IP 192.168.0.2, with the public IP address 100.100.100.100 before sending it to <ins>baidu.com</ins>;
- When <ins>baidu.com</ins> sends back the response packet, the NAT gateway replaces the destination address, from the server's intranet IP 100.100.100.100, with the server's intranet IP 192.168.100.100. ins> sends back a response packet, the NAT gateway replaces the destination address from the public IP address 100.100.100.100 with the server's intranet IP 192.168.0.2, and then sends it to the server on the intranet.

After understanding the principle of NAT, let's take a look at how to implement the NAT function in Linux.

## iptables and NAT

The Linux kernel provides a Netfilter framework that allows network packet modification (e.g. NAT) and filtering (e.g. firewall). On top of this, tools such as `iptables`, `ip6tables`, `ebtables`, etc., provide easier-to-use command line interfaces for system administrators to configure and manage NAT and firewall rules.

Among them, `iptables` is one of the most commonly used configuration tools. To master the principle and usage of `iptables`, the core thing is to figure out the work flow of network packets through Netfilter, and the following diagram shows the process.

<center><a href="https://ibb.co/ncXKy49">
    <img src="https://i.ibb.co/C7jSy4g/Packet-Flow-In-Netfilter-And-General-Networking.png" alt="Packet-Flow-In-Netfilter-And-General-Networking" border="0">
</a></center>

(Image from 🔗[Wikipedia](https://en.wikipedia.org/wiki/Iptables))

Linux supports four types of tables, including filter (for filtering), nat (for NAT), mangle (for modifying packet data), and raw (for raw packets).

The white background box with the table represents the chain, which is used to manage specific iptables rules. Each table can contain multiple chains, e.g:

- filter table with built-in INPUT, OUTPUT and FORWARD chains;
- nat table with built-in PREROUTING, POSTROUTING, OUTPUT, etc.

Of course, you can also create your own chains, as needed.

The gray conntrack, indicates the connection tracking module. It records the state of network connections through connection tracking tables (aka hash tables) in the kernel, and is the basis for the implementation of `iptables` state filtering (-m state) and NAT.

All the rules of `iptables` are then put into these tables and chains and are executed in the order of the diagram and the priority order of the rules.

For today's topic, to implement the NAT functionality, the main operations are performed in the nat table. And the nat table has three chains built in:

- PREROUTING, for rules performed before routing determination, for example, DNAT for incoming packets.
- POSTROUTING, for rules performed after routing determination, for example, SNAT or MASQUERADE for packets sent or forwarded.
- OUTPUT, similar to PREROUTING, but only processes packets sent out from the local machine.

Once you are familiar with the tables and chains in `iptables`, the corresponding NAT rules are relatively simple. Let's also take the three classifications of NAPT as an example to explain them in detail.

### SNAT

Based on what we just read, we know that SNAT needs to be configured in the POSTROUTING chain of the nat table. There are two common ways to configure it.

The first way is to configure SNAT uniformly for a subnet and have Linux choose the default egress IP. this is actually often referred to as MASQUERADE:

```shell
$ iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -j MASQUERADE
```

The second method, which configures SNAT for a specific IP address and specifies the converted source address, is

```shell
$ iptables -t nat -A POSTROUTING -s 192.168.0.2 -j SNAT --to-source 100.100.100.100
```

### DNAT

Looking at DNAT, obviously DNAT needs to be configured in the PREROUTING or OUTPUT chain of the nat table, where the PREROUTING chain is more commonly used (as it can also be used for forwarded packets).

```shell
$ iptables -t nat -A PREROUTING -d 100.100.100.100 -j DNAT --to-destination 192.168.0.2
```

### Bidirectional(Two-Way/Inbound) address translation

Two-way address translation, which means adding both SNAT and DNAT rules to achieve a one-to-one mapping relationship for public IPs and intranet IPs, i.e:

```shell
$ iptables -t nat -A POSTROUTING -s 192.168.0.2 -j SNAT --to-source 100.100.100.100
$ iptables -t nat -A PREROUTING -d 100.100.100.100 -j DNAT --to-destination 192.168.0.2
```

When using `iptables` to configure NAT rules, Linux needs to forward network packets from other IPs, so you should never forget to turn on the IP forwarding feature of Linux.

You can execute the following command to see if this feature is enabled. If the output is `1`, then IP forwarding is enabled:

```shell
$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
```

If it is not already on, you can manually turn it on by executing the following command:

```shell
$ sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
```

Of course, to avoid losing the configuration after a reboot, don't forget to write the configuration to the <ins>/etc/sysctl.conf</ins> file at

```shell
$ cat /etc/sysctl.conf | grep ip_forward
net.ipv4.ip_forward=1
```

## Summary

Today, we learned together the principles of Linux Network Address Translation NAT.

NAT technology can rewrite the source or destination IP of IP packets, so it is commonly used to solve the problem of public IP address shortage. It allows multiple hosts in a network to access external network resources by sharing the same public IP address. At the same time, since NAT blocks the intranet network, it also serves as a security isolation for the machines in the LAN.

NAT in Linux is implemented based on the kernel's connection tracking module. Therefore, it maintains the state of each connection while incurring a high performance cost. We will continue to learn how to analyze specific NAT performance issues in the next lesson.
