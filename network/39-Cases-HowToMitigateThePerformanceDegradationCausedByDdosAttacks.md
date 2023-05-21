# 39 Cases: How to mitigate the performance degradation caused by DDoS attacks?

In the last section, I took you through the use of `tcpdump` and `Wireshark`, and through several cases, I took you through the actual analysis of the network sending and receiving process with these two tools. When you encounter network performance problems, don't forget that you can use `tcpdump` and `Wireshark` to capture the actual transmitted network packets and troubleshoot potential performance problems.

Today, let's look at another problem, how to mitigate the performance degradation brought by DDoS (Distributed Denial of Service).

## DDoS Introduction

The predecessor of DDoS is DoS (Denial of Service), a denial of service attack that uses a large number of reasonable requests to occupy too many target resources, thus making the target service unable to respond to normal requests.

DDoS (Distributed Denial of Service) is a distributed architecture based on DoS that uses multiple hosts to attack the target host at the same time. In this way, even if the target service deploys network defense equipment, it is still unable to respond to a large number of network requests.

For example, the largest known traffic attack was the 🔗[DDoS attack](https://github.blog/2018-03-01-ddos-incident-report/) on Github last year, which reached a peak traffic of 1.35Tbps and a PPS of over (126.9 million).

In terms of the principle of the attack, DDoS can be divided into the following types.

The first one is to exhaust the bandwidth. Whether it is a server, router, switch or other network equipment, there is a fixed bandwidth limit. When the bandwidth is exhausted, network congestion will occur, thus preventing the transmission of other normal network messages.

The second, depleting the resources of the operating system. Network services require certain system resources, such as CPU, memory and other physical resources, as well as software resources such as connection tables, to operate properly. Once the resources are exhausted, the system cannot handle other normal network connections.

The third one is to consume the running resources of the application. Applications usually need to interact with other resources or systems to run. If the application stays busy processing invalid requests, it can also cause normal requests to be processed slower, or even unresponsive.

For example, constructing a large number of different domain names to attack a DNS server can cause the DNS server to keep performing iterative queries and updating its cache. This can greatly consume the resources of the DNS server and slow down the DNS response.

Regardless of the type of DDoS, the damage is huge. So, how can you find out that your system is under DDoS attack and how to deal with it? Next, let's take a look at these issues through a case study.

## Case preparation

The following example is still based on Ubuntu 18.04 and is applicable to other Linux systems as well. The case environment I used is the following:

- Machine configuration: 2 CPU, 8GB RAM.
- Pre-install `docker`, `sar`, `hping3`, `tcpdump`, `curl` and other tools, such as `apt-get install docker.io hping3 tcpdump curl`.

You should be familiar with all of these tools. One of them, `hping3`, which is described in 🔗[A case of elevated CPU usage in a system with soft interrupts](https://time.geekbang.org/column/article/72147), can construct TCP/IP protocol packets, perform security audits, firewall tests, DoS attack tests, etc. on your system. It can construct TCP/IP protocol packets, perform security audit, firewall test, DoS attack test, etc.

This case uses three virtual machines, and I drew a diagram to represent the relationship between them.

<center><a href="https://ibb.co/MkxDVdb">
    <img src="https://i.ibb.co/hFJRW0k/39-Case-Vm-Role-Figure.png" alt="39-Case-Vm-Role-Figure" border="0">
</a></center>

You can see that one of the virtual machines is running Nginx, which is used to simulate the web server to be analyzed, while the other two are used as clients of the web server, one of which is used for DoS attacks and the other is a normal client. The purpose of using multiple virtual machines is, naturally, to isolate each other and avoid "cross-infection".

> Since only one machine is used as the source of the attack, the attack here is actually a traditional DoS, not a DDoS.

Next, we open three terminals, SSH to each of the three machines (in the following steps, we assume that the terminal number matches the VM number shown in the diagram), and install the tools mentioned above.

As in the previous case, all commands below are run as root by default. If you are logged in as a normal user, please run `sudo su root` to switch to the root user.

Next, let's move on to the case study.

## Case analysis

First, in Terminal 1, run the following command to run the case, i.e., start a minimal Nginx application:

```shell
# 运行Nginx服务并对外开放80端口
# --network=host表示使用主机网络（这是为了方便后面排查问题）
$ docker run -itd --name=nginx --network=host nginx
```

Then, in both Terminal 2 and Terminal 3, use `curl` to access the port Nginx is listening on and confirm that Nginx is starting up properly. Assuming that 192.168.0.30 is the IP address of the virtual machine where Nginx is hosted, you should see the following output after running the `curl` command:

```shell
# -w表示只输出HTTP状态码及总时间，-o表示将响应重定向到/dev/null
$ curl -s -w 'Http code: %{http_code}\nTotal time:%{time_total}s\n' -o /dev/null http://192.168.0.30/
...
Http code: 200
Total time:0.002s
```

As you can see here, under normal circumstances, it takes only 2ms (0.002s) to access Nginx.

Next, in Terminal 2, run the `hping3` command to simulate a DoS attack:

```shell
# -S参数表示设置TCP协议的SYN（同步序列号），-p表示目的端口为80
# -i u10表示每隔10微秒发送一个网络帧
$ hping3 -S -p 80 -i u10 192.168.0.30
```

Now, go back to terminal one, and you will see that whatever command you execute is now much slower. However, be careful when practicing:

- If your phenomenon is not so obvious, then try turning down u10 in the parameter (e.g. to u1) or adding the -flood option;
- If your terminal one is not responding at all, then turn up u10 appropriately (e.g. to u30), otherwise you will not be able to operate VM1 via SSH later.

Then, go to Terminal 3 and execute the following command to simulate a normal client connection:

```shell
# --connect-timeout表示连接超时时间
$ curl -w 'Http code: %{http_code}\nTotal time:%{time_total}s\n' -o /dev/null --connect-timeout 10 http://192.168.0.30
...
Http code: 000
Total time:10.001s
curl: (28) Connection timed out after 10000 milliseconds
```

You can notice that in Terminal 3, the normal client connection times out and does not receive a response from the Nginx service.

What's wrong with this? Let's go back to Terminal 1 and check the network conditions. You will remember that we have used `sar` many times to look at both PPS (messages sent and received per second) and BPS (bytes sent and received per second).

We can go back to Terminal 1 and execute the following command:

```shell
$ sar -n DEV 1
08:55:49        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
08:55:50      docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
08:55:50         eth0  22274.00    629.00   1174.64     37.78      0.00      0.00      0.00      0.02
08:55:50           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```

The meaning of the columns in the `sar` output is described in my previous article on Linux networking basics, which you can see by clicking 🔗[here](https://time.geekbang.org/column/article/81057), or by running `man sar` to query the manual.

From the output of this `sar`, you can see that the network has received more than 20,000 PPS, but only 1174 kB of BPS, which makes the size of each packet only 54B ($1174*1024/22274=54$).

This is obviously a small packet, but what kind of packet is it? Let's use `tcpdump` to grab the packets and see.

In Terminal 1, execute the following `tcpdump` command:

```shell
# -i eth0 只抓取eth0网卡，-n不解析协议名和主机名
# tcp port 80表示只抓取tcp协议并且端口号为80的网络帧
$ tcpdump -i eth0 -n tcp port 80
09:15:48.287047 IP 192.168.0.2.27095 > 192.168.0.30: Flags [S], seq 1288268370, win 512, length 0
09:15:48.287050 IP 192.168.0.2.27131 > 192.168.0.30: Flags [S], seq 2084255254, win 512, length 0
09:15:48.287052 IP 192.168.0.2.27116 > 192.168.0.30: Flags [S], seq 677393791, win 512, length 0
09:15:48.287055 IP 192.168.0.2.27141 > 192.168.0.30: Flags [S], seq 1276451587, win 512, length 0
09:15:48.287068 IP 192.168.0.2.27154 > 192.168.0.30: Flags [S], seq 1851495339, win 512, length 0
...
```

In this output, Flags [S] indicates that this is a SYN packet. The large number of SYN packets indicates that this is a SYN Flood attack. If you look at it with `Wireshark`, which you talked about in the previous section, you can see the SYN Flood process more visually:

<center><a href="https://ibb.co/bgpTZ7G">
    <img src="https://i.ibb.co/VWXRZ3k/39-Case-Wireshark.png" alt="39-Case-Wireshark" border="0">
</a></center>

In fact, SYN Flood is exactly the most classic form of DDoS attack in the Internet. You can also see how it works from this diagram above:

- That is, the client constructs a large number of SYN packets requesting a TCP connection;
- and when the server receives the packets, it sends a SYN+ACK message to the source IP and waits for the last ACK message of the three handshakes until it times out.

This wait-state TCP connection is also commonly referred to as a half-open connection. Due to the limited size of the connection table, a large number of half-open connections will cause the connection table to fill up quickly, making it impossible to establish new TCP connections.

Referring to the following TCP state diagram, you can see that the TCP connection on the server side is in the SYN_RECEIVED state:

<center><a href="https://ibb.co/wcjNck4">
    <img src="https://i.ibb.co/fDLYDgH/Tcp-Status-Figure.png" alt="Tcp-Status-Figure" border="0">
</a></center>

(Image from 🔗[Wikipedia](https://en.wikipedia.org/wiki/File:Tcp_state_diagram.png))

This actually hints at a way to see TCP half-open connections, the key being connections with SYN_RECEIVED status. We can use `netstat` to see the status of all connections, but note that the SYN_REVEIVED status is often abbreviated to SYN_RECV.

Let's continue in Terminal 1 and execute the following netstat command:

```shell
# -n表示不解析名字，-p表示显示连接所属进程
$ netstat -n -p | grep SYN_REC
tcp        0      0 192.168.0.30:80          192.168.0.2:12503      SYN_RECV    -
tcp        0      0 192.168.0.30:80          192.168.0.2:13502      SYN_RECV    -
tcp        0      0 192.168.0.30:80          192.168.0.2:15256      SYN_RECV    -
tcp        0      0 192.168.0.30:80          192.168.0.2:18117      SYN_RECV    -
...
```

From the results, you can find a large number of connections with SYN_RECV status and a source IP address of 192.168.0.2.

Further, we can also use the `wc` tool to count the number of connections in all SYN_RECV states:

```shell
$ netstat -n -p | grep SYN_REC | wc -l
193
```

After finding the source IP, to resolve the SYN attack, just drop the relevant packets. This is where `iptables` can help you with this task. You can execute the following `iptables` command in Terminal 1:

```shell
$ iptables -I INPUT -s 192.168.0.2 -p tcp -j REJECT
```

Then go back to Terminal 3 and execute the `curl` command again to see what happens when a normal user accesses Nginx:

```shell
$ curl -w 'Http code: %{http_code}\nTotal time:%{time_total}s\n' -o /dev/null --connect-timeout 10 http://192.168.0.30
Http code: 200
Total time:1.572171s
```

Now, you can find that normal users can access Nginx, but the response is slower, from 2ms to 1.5s.

However, in general, the source IP in a SYN Flood attack is not fixed. For example, you can add the --rand-source option to the `hping3` command to randomize the source IP, but then the method just described will not work.

Luckily, there are many other ways to achieve similar goals. For example, you can use the following two methods to limit the rate of syn packets:

```shell
# 限制syn并发数为每秒1次
$ iptables -A INPUT -p tcp --syn -m limit --limit 1/s -j ACCEPT

# 限制单个IP在60秒新建立的连接数为10
$ iptables -I INPUT -p tcp --dport 80 --syn -m recent --name SYN_FLOOD --update --seconds 60 --hitcount 10 -j REJECT
```

At this point, we have initially limited the SYN Flood attack. But this is not enough, because our case is still only a single source of attack.

If multiple machines are sending SYN Flood at the same time, this approach may be outright ineffective. Because you will probably not be able to SSH to the machine (SSH is also TCP-based), let alone execute all the above troubleshooting commands.

So, this also requires you to do some TCP optimization on your system beforehand.

For example, SYN Flood can cause a sharp increase in the number of connections in the SYN_RECV state. In the `netstat` command above, you can also see more than 190 connections in the half-open state.

However, there is a limit to the number of connections in the half-open state, and by running the following command, you can see that the default half-connection capacity is only 256:

```shell
$ sysctl net.ipv4.tcp_max_syn_backlog
net.ipv4.tcp_max_syn_backlog = 256
```

In other words, a slightly larger number of SYN packets will not allow you to SSH to the machine. So, you should also increase the size of the semi-connections, for example, to 1024 with the following command:

```shell
$ sysctl -w net.ipv4.tcp_max_syn_backlog=1024
net.ipv4.tcp_max_syn_backlog = 1024
```

In addition, the kernel will automatically retry each SYN_RECV connection if it fails, and the default number of retries is 5. You can reduce this to 1 by executing the following command:

```shell
$ sysctl -w net.ipv4.tcp_synack_retries=1
net.ipv4.tcp_synack_retries = 1
```

In addition, TCP SYN Cookies are a specialized defense against SYN Flood attacks. SYN Cookies calculate a hash (SHA1) based on connection information (including source address, source port, destination address, destination port, etc.) and a cryptographic seed (such as system startup time), which is called a cookie.

This cookie is then used as the sequence number to answer the SYN+ACK packet and release the connection state. After the client sends the last ACK of the three handshakes, the server calculates this hash again and confirms that it is the return packet of the last returned SYN+ACK before entering the TCP connection state.

Thus, with SYN cookies enabled, there is no need to maintain a half-open connection state, and thus no limit on the number of half-connections.

> Note that the kernel option net.ipv4.tcp_max_syn_backlog is also disabled when TCP syncookies are enabled.

You can turn on TCP SYN cookies by using the following command:

```shell
$ sysctl -w net.ipv4.tcp_syncookies=1
net.ipv4.tcp_syncookies = 1
```

Note that the configurations modified by the `sysctl` command above are temporary, and they will be lost after a reboot. So, to ensure configuration persistence, you should also write these configurations, to the <ins>/etc/sysctl.conf</ins> file. For example:

```shell
$ cat /etc/sysctl.conf
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_max_syn_backlog = 1024
```

Remember, however, that the configuration written to <ins>/etc/sysctl.conf</ins> needs to be executed with the sysctl -p command before it will take effect dynamically.

Of course, at the end of the case, don't forget to run `docker rm -f nginx` to clean up the Nginx application started at the beginning of the case.

### DDoS How to defend in the end

Here, today's case is over. However, you must still have questions. You should have noticed that today's topic is "mitigation", not "resolution" of DDoS problems.

Why not solve DDoS, but only mitigation? And the approach in today's case only stops the Nginx service from timing out, but the access latency is still much larger than the 2ms it was at the beginning.

In fact, when DDoS messages arrive at the server, the mechanisms provided by Linux can only mitigate, but not completely solve. Even a small packet attack like SYN Flood, with its huge PPS, can cause the Linux kernel to consume a lot of resources, which in turn can slow down the processing of other network messages.

While you can tweak kernel parameters to mitigate the performance issues caused by DDoS, it will not completely solve it, as in this case.

As I mentioned in the previous 🔗[C10K, C100K article](https://time.geekbang.org/column/article/81268), the lengthy protocol stack in the Linux kernel is a huge burden when the PPS is very large. The same holds true for DDoS attacks.

So, the C10M approach mentioned at the time is equally appropriate here. For example, you can build a DDoS scheme based on XDP or DPDK to identify and discard DDoS messages before the kernel network stack or skip the kernel stack to avoid the consumption of other system resources by DDoS.

However, for traffic-based DDoS, when the server's bandwidth is exhausted, there is nothing that can be done to handle it inside the server. At this point, the only way to identify and block the traffic is in the network equipment outside the server (provided, of course, that the network equipment can withstand the traffic attack). For example, the purchase of professional intrusion detection and prevention equipment, configure traffic cleaning equipment to block malicious traffic, etc..

Since DDoS is so difficult to defend, does this mean that Linux servers do not pay attention to this point internally, but all to the professional network equipment to deal with it?

Of course not, because DDoS is not always due to large traffic or large PPS, sometimes slow requests can also bring huge performance degradation (this situation is called slow DDoS).

For example, many attacks against applications disguise themselves as normal users to request resources. In this case, the request traffic itself may not be large, but the response traffic may be large, and it is also likely to take a lot of resources to process within the application.

At this point, the application needs to consider identifying and rejecting this malicious traffic as early as possible, such as by making reasonable use of caching, increasing WAF (Web Application Firewall), using CDN, etc.

## Summary

Today, we learned about mitigation methods in the event of a distributed denial of service (DDoS), which uses a large number of forged requests to cause the target service to consume a large amount of resources to process these invalid requests, and thus not respond properly to normal user requests.

Due to the distributed, high-volume, hard-to-track nature of DDoS, there is no way to completely defend against the problems caused by DDoS, only to try to mitigate this effect.

For example, you can purchase professional traffic cleaning equipment and network firewall to block malicious traffic at the entrance of the network and keep only normal traffic into the servers in the data center.

In Linux servers, you can use kernel tuning, DPDK, XDP, and other methods to increase the server's resistance to attacks and reduce the impact of DDoS on normal services. And in applications, you can use caching at all levels, WAF, CDN, etc., to mitigate the impact of DDoS on applications.
