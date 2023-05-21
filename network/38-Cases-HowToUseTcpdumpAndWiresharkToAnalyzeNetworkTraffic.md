# 38 Cases: How to use tcpdump and Wireshark to analyze network traffic?

In the previous section, we learned about the analysis and optimization methods for DNS performance issues. To briefly review, DNS can provide a mapping relationship between domain names and IP addresses, and is also a common global load balancing (GSLB) implementation method.

Usually, services that need to be exposed to the public network are bound to a domain name, which makes it easy to remember and also avoids the change of IP address of the backend service from affecting the users.

Be aware, however, that DNS resolution is affected by various network conditions and performance may be unstable. For example, increased latency on the public network, cache expiration causing a new request to the upstream server, or lack of performance of DNS servers during peak traffic can all lead to increased latency in DNS response.

In this case, you can use the debugging function of `nslookup` or `dig` to analyze the DNS resolution process and debug the DNS server latency with tools such as ping to locate the performance bottleneck. Usually, you can use caching, prefetching, HTTPDNS, etc. to optimize the performance of DNS.

`ping`, which we used in the previous section, is one of the most commonly used tools for testing service latency. In many cases, `ping` can help us locate latency problems, but sometimes, `ping` itself can cause unexpected problems. In this case, we need to capture the network packets sent and received during the execution of the `ping` command, and then analyze them to find the root cause of the problem.

`tcpdump` and `Wireshark` are the most commonly used network packet capture and analysis tools, and are essential for analyzing network performance.

- `tcpdump` only supports command line format, and is often used to capture and analyze network packets in servers.
- In addition to packet capture, `Wireshark` also provides a powerful graphical interface and summary analysis tools, which are particularly simple and useful when analyzing complex network scenarios.

Thus, it is a common approach to actually analyze network performance by first capturing packets with `tcpdump` and then analyzing them with `Wireshark`.

Today, I will take you through how to use `tcpdump` and `Wireshark` to analyze network performance issues.

## Case preparation

This case is also based on Ubuntu 18.04, and is applicable to other Linux systems as well. The case environment I used is the following:

- Machine configuration: 2 CPU, 8GB RAM.
- Pre-installed tcpdump, Wireshark and other tools such as

```shell
# Ubuntu
apt-get install tcpdump wireshark

# CentOS
yum install -y tcpdump wireshark
```

Since Wireshark has a graphical interface and is not available via SSH, I recommend that you install it on a local machine (e.g. Windows). You can download and install Wireshark at <ins>https://www.wireshark.org/</ins>.

> As before, all commands in this case are run by default as the root user (in Windows, except when running Wireshark). > If you are logged in as a normal user, run `sudo su root` to switch to the root user.

## Exploring ping again

As mentioned before, `ping` is one of the most commonly used network tools to probe the connectivity and latency between network hosts. The principle and usage of `ping` has been briefly described in the previous 🔗[Linux Networking Basics](https://time.geekbang.org/column/article/81057), and `ping` has been used several times in the case of slow DNS to test DNS server latency (RTT).

However, although `ping` is relatively simple, sometimes you may find that the `ping` tool itself can be anomalous, such as running slowly, but the actual network latency is not very large.

Next, let's open a terminal, SSH into the case machine, and execute the following command to test the connectivity and latency between the case machine and the Geekbang Technologies website. If everything works, you will see this output below:

```shell
# ping 3 次（默认每次发送间隔1秒）
# 假设DNS服务器还是上一期配置的114.114.114.114
$ ping -c3 geektime.org
PING geektime.org (35.190.27.188) 56(84) bytes of data.
64 bytes from 35.190.27.188 (35.190.27.188): icmp_seq=1 ttl=43 time=36.8 ms
64 bytes from 35.190.27.188 (35.190.27.188): icmp_seq=2 ttl=43 time=31.1 ms
64 bytes from 35.190.27.188 (35.190.27.188): icmp_seq=3 ttl=43 time=31.2 ms

--- geektime.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 11049ms
rtt min/avg/max/mdev = 31.146/33.074/36.809/2.649 ms
```

The output screen of `ping`, which we have already learned in 🔗[Linux Networking Fundamentals](https://time.geekbang.org/column/article/81057), can be reviewed so that you can interpret and analyze the output yourself.

However, be aware that if you find that `ping` ends quickly when you run it, then execute the following command and try again. We will explain the meaning of this command later.

```shell
# 禁止接收从DNS服务器发送过来并包含googleusercontent的包
$ iptables -I INPUT -p udp --sport 53 -m string --string googleusercontent --algo bm -j DROP
```

Based on the `ping` output, you can see that <ins>geektime.org</ins> resolves to an IP address of 35.190.27.188, and the next three `ping` requests all get a response, with a latency (RTT) of a little over 30ms.

But the summary is a bit more interesting. 3 sends, 3 responses, no packet loss, but the total time between the three sends and receives is actually more than 11s (11049ms), which is a bit unbelievable, right?

Remembering the DNS resolution issue from the previous section, you might suspect that this could be a slow DNS resolution issue. But is it?

Going back to the `ping` output, the IP address is used in all three `ping` requests, which means that `ping` only needs to resolve the IP once at the beginning of the run, and then it can just use the IP later.

Let's try it again with `nslookup`. Execute the following `nslookup` command in the terminal, and note that this time we also added the `time` command to output the execution time of `nslookup`: `nslookup`:

```shell
$ time nslookup geektime.org
Server:    114.114.114.114
Address:  114.114.114.114#53

Non-authoritative answer:
Name:  geektime.org
Address: 35.190.27.188


real  0m0.044s
user  0m0.006s
sys  0m0.003s
```

As you can see, the domain name resolution is still very fast, only 44ms, which is obviously much shorter than 11s.

Here, how to analyze it further? Actually, at this point you can use `tcpdump` to capture packets and see which network packets are being sent and received by `ping`.

Let's open another terminal (terminal 2), SSH into the case machine and execute the following command:

```shell
$ tcpdump -nn udp port 53 or host 35.190.27.188
```

Of course, you could just use `tcpdump` without any parameters to grab the packets, but then you might grab a lot of irrelevant packets. Since we have already executed the `ping` command, we know that the IP address of <ins>geekbang.org</ins> is 35.190.27.188, and we know that the `ping` command performs DNS lookups. So this command above is filtering based on this rule.

Let me explain this command in detail.

- **-nn** , indicates that the domain name (i.e., no reverse resolution), protocol, and port number of the captured packet are not resolved.
- **udp port 53**, which means that only packets with port number 53 (including source and destination ports) of the UDP protocol are displayed.
- **host 35.190.27.188**, which means only packets with IP address (both source and destination) of 35.190.27.188 will be displayed.
- The "**or**" in the middle of these two filter conditions indicates an or relationship, that is, as long as either of the above two conditions is met, it can be displayed.

接下来，回到终端一，执行相同的 `ping` 命令：

```shell
$ ping -c3 geektime.org
...
--- geektime.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 11095ms
rtt min/avg/max/mdev = 81.473/81.572/81.757/0.130 ms
```

命令结束后，再回到终端二中，查看 `tcpdump` 的输出：

```shell
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
14:02:31.100564 IP 172.16.3.4.56669 > 114.114.114.114.53: 36909+ A? geektime.org. (30)
14:02:31.507699 IP 114.114.114.114.53 > 172.16.3.4.56669: 36909 1/0/0 A 35.190.27.188 (46)
14:02:31.508164 IP 172.16.3.4 > 35.190.27.188: ICMP echo request, id 4356, seq 1, length 64
14:02:31.539667 IP 35.190.27.188 > 172.16.3.4: ICMP echo reply, id 4356, seq 1, length 64
14:02:31.539995 IP 172.16.3.4.60254 > 114.114.114.114.53: 49932+ PTR? 188.27.190.35.in-addr.arpa. (44)
14:02:36.545104 IP 172.16.3.4.60254 > 114.114.114.114.53: 49932+ PTR? 188.27.190.35.in-addr.arpa. (44)
14:02:41.551284 IP 172.16.3.4 > 35.190.27.188: ICMP echo request, id 4356, seq 2, length 64
14:02:41.582363 IP 35.190.27.188 > 172.16.3.4: ICMP echo reply, id 4356, seq 2, length 64
14:02:42.552506 IP 172.16.3.4 > 35.190.27.188: ICMP echo request, id 4356, seq 3, length 64
14:02:42.583646 IP 35.190.27.188 > 172.16.3.4: ICMP echo reply, id 4356, seq 3, length 64
```

In this output, the first two lines represent the options of `tcpdump` and the basic information of the interface; from the third line onwards, it is the output of the network packets captured. The format of this output is Timestamp Protocol Source Address. Source Port > Destination Address. Destination Port Network Packet Details (this is the most basic format, additional fields can be added through options).

The previous fields are relatively easy to understand. But the network packet details, themselves, vary depending on the protocol. So, to understand the detailed meaning of these network packets, you need to have a basic understanding of the basic format of common network protocols, as well as the principles of interaction.

Of course, in practice, these are documented in the 🔗[RFC](https://www.rfc-editor.org/rfc/index) (Request for Comment) issued by the IETF (Internet Engineering Task Force).

For example, the first record indicates that the A record query request sent from the local IP to 114.114.114.114 is documented in RFC1035, and you can refer to the message format by clicking 🔗[here](https://www.ietf.org/rfc/rfc1035.txt). Among this output of `tcpdump`, we can see the following fields:

- 36909+ indicates the query identifier value, which also appears in the response, and the plus sign indicates that a recursive query is enabled.
- A? indicates a query for an A record.
- geektime.org. indicates the domain name to be queried.
- 30 indicates the length of the message.

The next entry is the DNS response sent back from 114.114.114.114 - the A record for the domain geektime.org. has a value of 35.190.27.188.

The third and fourth, the ICMP echo request and ICMP echo reply, have the response packet timestamp 14:02:31.539667, minus the request packet timestamp 14:02:31.508164, which gives us 30ms for this ICMP, which doesn't seem to be a problem.

However, the next two reverse address resolution PTR requests are more suspicious. Because we only see the request packet, but no answer packet. If you look closely at their timing, you will see that both records are sent 5s before the next network packet appears, and the two PTR records consume 10s.

Looking further down, the last four packets are two normal ICMP requests and responses, and their latency is calculated based on the timestamp, which is also 30ms.

The purpose of PTR reverse address resolution is to backtrack the domain name from the IP address, but in fact, not all IP addresses define PTR records, so PTR queries are likely to fail.

So, when you use ping, if you find that the delay in the result is not big, but the ping command itself is slow, don't panic, it is possible that the PTR behind it is messing up.

Once you know the problem, it's easier to solve it, just disable PTR. As usual, execute the `man ping` command and consult the manual to find out how to disable name resolution by adding the -n option. For example, we can execute the following command in the terminal:

```shell
$ ping -n -c3 geektime.org
PING geektime.org (35.190.27.188) 56(84) bytes of data.
64 bytes from 35.190.27.188: icmp_seq=1 ttl=43 time=33.5 ms
64 bytes from 35.190.27.188: icmp_seq=2 ttl=43 time=39.0 ms
64 bytes from 35.190.27.188: icmp_seq=3 ttl=43 time=32.8 ms

--- geektime.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 32.879/35.160/39.030/2.755 ms
```

As you can see, it now takes only 2s to finish, which is much faster than the 11s it took earlier.

Here, I've taken you along with `tcpdump` to solve one of the most common problems with `ping` working slowly.

At the end of the case, if you executed the `iptables` command at the beginning, don't forget to delete it:

```shell
$ iptables -D INPUT -p udp --sport 53 -m string --string googleusercontent --algo bm -j DROP
```

However, after the deletion you must still have questions, obviously our case has nothing to do with Google, why should we filter the packets based on googleusercontent, an unrelated string?

In fact, if you change the DNS server, you can use PTR to backtrack to the domain name corresponding to 35.190.27.188:

```shell
$ nslookup -type=PTR 35.190.27.188 8.8.8.8
Server:  8.8.8.8
Address:  8.8.8.8#53
Non-authoritative answer:
188.27.190.35.in-addr.arpa  name = 188.27.190.35.bc.googleusercontent.com.
Authoritative answers can be found from:
```

You see, although the PTR record is checked, the result is not <ins>geekbang.org</ins>, but <ins>188.27.190.35.bc.googleusercontent.com</ins>. In fact, that's why, after dropping the one containing googleusercontent at the beginning of the case, `ping` is slow. This is because `iptables`, which actually drops the PTR response, causes the PTR request to time out.

`tcpdump` is arguably the most effective tool for network performance analysis. Next, I'll take you through more uses of `tcpdump`.

## tcpdump

As we know, `tcpdump` is also one of the most commonly used network analysis tools. It is based on 🔗[libpcap](https://www.tcpdump.org/) and uses the AF_PACKET socket in the kernel to grab the network packets transmitted in the network interface; and provides powerful filtering rules to help you pick out the information you want to focus on most from the large number of network packets.

`tcpdump` shows you the detailed details of each network packet, which requires that you have a basic understanding of the network protocol before using it. And to understand the detailed design and implementation details of network protocols, 🔗[RFC](https://www.rfc-editor.org/rfc-index.html) is, of course, the most authoritative source.

The contents of the RFC, however, may not be beginner-friendly. If you don't know much about network protocols yet, we recommend you to learn "TCP/IP in Detail", especially the TCP/IP protocol family in Volume 1. This is a core fundamental knowledge that every programmer should master.

Going back to the `tcpdump` tool itself, its basic usage, or rather simple, is `tcpdump [options] [filter expressions]`. Of course, both options and expressions are enclosed in brackets to indicate that they are optional.

> Tip: In Linux tools, if you see in the documentation that the options are in brackets, it means that this is an optional option.
> This is the time to look out for options that have default values.

Check out the 🔗[manual](https://www.tcpdump.org/manpages/tcpdump.1.html) for `tcpdump` and the [manual](https://www.tcpdump.org/manpages) for pcap-filter /pcap-filter.7.html), you will find that `tcpdump` provides a large number of options and a wide variety of filter expressions. But don't worry, you only need to master a few common options and filter expressions to satisfy most scenarios.

In order to help you get started with `tcpdump`, I have also organized some of the most common uses for you here, and made a table for you to refer to.

First, let's take a look at some of the common options. In the `ping` case above, we used the **-nn** option, which means that we don't need to resolve the IP address and port number by name. For other common options, I'll use this table below to explain them.

<center><a href="https://ibb.co/64k2q4v">
    <img src="https://i.ibb.co/sFZhXF5/Use-Tcpdump-Options.png" alt="Use-Tcpdump-Options" border="0">
</a></center>

Next, let's look at the common filtering expressions. The one I just used is udp port 53 or host 35.190.27.188, which indicates that it catches request and response packets for the DNS protocol, as well as packets with a source or destination address of 35.190.27.188.

Other common filtering options, I have also organized into the following table.

<center><a href="https://ibb.co/Q6bpD93">
    <img src="https://i.ibb.co/wK0BcyF/Use-Tcpdump-Filter-Expressions.png" alt="Use-Tcpdump-Filter-Expressions" border="0">
</a></center>

Finally, to re-emphasize the output format of `tcpdump`, whose basic format I have described earlier:

```plaintext
Timestamp Protocol Source Address. Source Port > Destination Address. Destination Port Network Packet Details
```

Among other things, the network packet details depend on the protocol, and the format displayed differs from protocol to protocol. So, for more detailed usage, you still need to check the 🔗[man](https://www.tcpdump.org/manpages/tcpdump.1.html) manual of `tcpdump` (you can also get it by executing `man tcpdump`).

However, after all this, you should have noticed that. `tcpdump` is powerful, but the output format is not intuitive. In particular, when the number of network packets in the system is relatively large (e.g. PPS over several thousand), it is not easy to analyze the problems from the network packets captured by `tcpdump`.

In contrast, `Wireshark` provides a more user-friendly interface through a graphical interface and a series of summary analysis tools, allowing you to solve network performance problems with a faster speed. Next, let's take a look at it in detail.

## Wireshark

`Wireshark` is also one of the most popular network analysis tools, and its biggest benefit is that it provides a cross-platform graphical interface. Similar to `tcpdump`, `Wireshark` also provides powerful filtering rule expressions, and a series of built-in summary analysis tools.

For example, in the `ping` case, you can execute the following command to save the captured network packets to <ins>ping.pcap</ins> file:

```shell
$ tcpdump -nn udp port 53 or host 35.190.27.188 -w ping.pcap
```

Next, copy it to the machine where you have `Wireshark` installed, e.g. you can copy it locally using `scp` to:

```shell
$ scp host-ip/path/ping.pcap .
```

Then, open it with `Wireshark`. Once opened, you will see the following screen:

<center><a href="https://ibb.co/pPGYzcL">
    <img src="https://i.ibb.co/W3qmyZ2/Wireshark-Ping.png" alt="Wireshark-Ping" border="0">
</a></center>

From the `Wireshark` interface, you can see that it not only shows the header information of each network packet in a more organized format; it also shows the two different protocols, DNS and ICMP, in different colors. You can also see at a glance that the two middle PTR queries do not have response packets.

Then, after selecting a network packet in the network packet list, you can also see the details of the packet at each layer of the protocol stack in the network packet details below it. For example, take the PTR packet numbered 5 as an example:

<center><a href="https://ibb.co/qyF24ZQ">
    <img src="https://i.ibb.co/ZL1bwZD/Wireshark-Packet-Protocol-Stack-Details.png" alt="Wireshark-Packet-Protocol-Stack-Details" border="0">
</a></center>

You can see the summary information for the source and destination addresses for the IP layer (Internet Protocol), the UDP protocol (User Datagram Protocol) for the transport layer, and the DNS protocol (Domain Name System) for the application layer.

Continue clicking on the arrows to the left of each layer to see all the information in the protocol header for that layer. For example, when you click on DNS, you can see the values and meanings of the fields of the DNS protocol such as Transaction ID, Flags, Queries, etc.

Of course, `Wireshark`s capabilities go far beyond that. Next, I'll take you through an HTTP example and understand how TCP's three handshakes and four waves work.

In this case, we are going to access <http://example.com/>. Go to Terminal 1 and execute the following command to find out the IP of <ins>example.com</ins>. Then, execute the `tcpdump` command to filter the IP address and save the result to <ins>web.pcap</ins>.

```shell
$ dig +short example.com
93.184.216.34
$ tcpdump -nn host 93.184.216.34 -w web.pcap
```

> In fact, you can use the domain name directly in the host expression, `tcpdump -nn host example.com -w web.pcap`.

Next, switch to Terminal 2 and execute the following `curl` command to access <http://example.com>:

```shell
$ curl http://example.com
```

Finally, go back to Terminal 1, press Ctrl+C to stop `tcpdump` and copy the resulting <ins>web.pcap</ins>.

After opening <ins>web.pcap</ins> with `Wireshark`, you will see the following screen in `Wireshark`:

<center><a href="https://ibb.co/GWnZvJt">
    <img src="https://i.ibb.co/LxYTzPJ/Wireshark-Http-Example-Com.png" alt="Wireshark-Http-Example-Com" border="0">
</a></center>

Since HTTP is based on TCP, the first three packets you see are the TCP triple handshake packets. Next, the middle ones are the HTTP request and response packets, and the last three packets are the TCP three wave packets when the TCP connection is broken.

From the menu bar, click Statistics -> Flow Graph, and then, in the pop-up interface, select TCP Flows for the Flow type, you can see more clearly how the TCP flow is executed during the whole process:

<center><a href="https://ibb.co/vwWTL7F">
    <img src="https://i.ibb.co/mDK2hLj/Wire-Shark-Tcp-Statistic-Flow.png" alt="Wire-Shark-Tcp-Statistic-Flow" border="0">
</a></center>

This is actually very similar to the three TCP handshakes and four TCP waves that are talked about in various tutorials, and as a comparison, the process you usually see for TCP three handshakes and four waves is basically like this:

<center><a href="https://ibb.co/DbpWNZC">
    <img src="https://i.ibb.co/4fdtcHT/Tcp-Handshakes-Waves.png" alt="Tcp-Handshakes-Waves" border="0">
</a></center>

(Image from 🔗[CoolShell](https://coolshell.cn/articles/11564.html))

However, comparing these two pictures, you will find that the packets caught here are not exactly the same as the four waves above, and the actual waving process only has three packets instead of four.

In fact, the reason why there are three packets is that after the server side receives the FIN from the client, the server side also has to close the connection at the same time, so that the ACK and FIN can be merged together and sent, saving one packet and turning it into "three waves".

Usually, when the server receives the FIN from the client, it is likely that it has not finished sending data yet, so it will reply to the client with an ACK packet first. Wait for a while, after all the packets are sent, the FIN packet will be sent. That's four waves of the hand.

After capturing the packet, the following screen will be displayed in `Wireshark` (the original network packet is from the Wireshark TCP 4-times close example, which you can download by clicking 🔗[here](https://wiki.wireshark.org/TCP%204-times%20close) to download):

<center><a href="https://ibb.co/cyddRFC">
    <img src="https://i.ibb.co/6y55GP1/Tcp-4-Way-Handshake-Termination.png" alt="Tcp-4-Way-Handshake-Termination" border="0">
</a></center>

Of course, this is not the only way to use `Wireshark`, for more information, please refer to 🔗[Official Document](https://www.wireshark.org/docs/) and 🔗[WIKI](https://wiki. wireshark.org/Home).

## Summary

Today, we learned how to use `tcpdump` and `Wireshark` together, and through several cases, we learned how to use these two tools to analyze the network sending and receiving process, and identify potential performance problems.

DNS resolution includes not only resolving A record requests for IP addresses from domain names, but also performance tools that help you "smartly" backtrack PTR requests for domain names from IP addresses. In fact, **discretionary** is based on the IP address.

In fact, **backtracking domain names based on IP addresses and protocol names based on port numbers is the default behavior of many network tools, and this often causes performance tools to work slowly.** So, typically, network performance tools provide an option (such as -n or -nn) to disable name resolution.

At work, when you run into network performance problems, don't forget about `tcpdump` and `Wireshark`, two big killers. You can use them to grab the actual network packets transmitted and then troubleshoot for potential performance issues.
