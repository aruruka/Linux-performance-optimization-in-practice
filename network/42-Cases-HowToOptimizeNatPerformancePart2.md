# 42 Cases: How to Optimize NAT Performance? (part 2)

In the last section, we learned how NAT works and understood how to manage NAT rules in Linux. Let's start with a brief refresher.

NAT technology can rewrite the source or destination IP of IP packets, so it is commonly used to solve the problem of public IP address shortage. It allows multiple hosts in a network to access external resources by sharing the same public IP address. At the same time, since NAT blocks the intranet network, it also serves as a security isolation for the machines in the LAN.

NAT in Linux is implemented based on the kernel's connection tracking module. So, while it maintains the state of each connection, it also has some impact on network performance. So, what can we do when we encounter NAT performance problems?

Next, I will take you through a case study to learn how to analyze NAT performance issues.

## Case preparation

The following example is still based on Ubuntu 18.04 and is applicable to other Linux systems as well. The case environment I used is the following:

- Machine configuration: 2 CPU, 8GB RAM.
- Pre-installed `docker`, `tcpdump`, `curl`, `ab`, `SystemTap` and other tools, such as

```shell
# Ubuntu
$ apt-get install -y docker.io tcpdump curl apache2-utils

# CentOS
$ curl -fsSL https://get.docker.com | sh
$ yum install -y tcpdump curl httpd-tools
```

Most of the tools you should be familiar with, here I'll briefly introduce `SystemTap`.

🔗[SystemTap](https://sourceware.org/systemtap/) is a dynamic tracing framework for Linux that takes user-supplied scripts and converts them into kernel modules for execution, which are used to monitor and track the behavior of the kernel. You don't need to dig deeper into its principles for now, as they will be covered later. All you need to know here is how to install it:

```shell
# Ubuntu
apt-get install -y systemtap-runtime systemtap
# Configure ddebs source
echo "deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-updates main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-proposed main restricted universe multiverse" | \
sudo tee -a /etc/apt/sources.list.d/ddebs.list
# Install dbgsym
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys F2EDC64DC5AEE1F6B9C621F0C8CAB6595FDFF622
apt-get update
apt install ubuntu-dbgsym-keyring
stap-prep
apt-get install linux-image-`uname -r`-dbgsym

# CentOS
yum install systemtap kernel-devel yum-utils kernel
stab-prep
```

This case is still our most common Nginx, and will use `ab` as its client for stress testing. There are two virtual machines used in this case, and I have drawn a diagram to represent their relationship.

<center><a href="https://ibb.co/rZ2VVf2">
    <img src="https://i.ibb.co/0fyxxVy/42-Vm-Role-Diagram.png" alt="42-Vm-Role-Diagram" border="0">
</a></center>

Next, we open two terminals, SSH to each machine (for the following steps, assume that the terminal number matches the VM number shown), and install these tools mentioned above. Note that `curl` and `ab` only need to be installed in the client VM (i.e. VM2).

As in previous cases, all commands below are run by default as the root user. If you are logged in as a normal user, run the `sudo su root` command to switch to the root user.

> If you have any problems with the installation process, you are encouraged to search and solve them by yourself first.
> If you have already installed it before, you can ignore this point.

Next, let's move on to the case study.

## Case analysis

To compare the performance issues associated with NAT, we first run an Nginx service without NAT and test its performance with `ab`.

In Terminal 1, execute the following command to start Nginx, noting the option **--network=host**, which indicates that the container uses Host network mode, i.e., no NAT:

```shell
$ docker run --name nginx-hostnet --privileged --network=host -itd feisky/nginx:80
```

Then go to Terminal 2 and execute the `curl` command to confirm that Nginx is starting up properly:

```shell
$ curl http://192.168.0.30/
...
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Go ahead and run the `ab` command in Terminal 2 to stress test Nginx. Before testing, however, note that Linux allows a relatively small number of open file descriptions by default; for example, on my machine, this value is only 1024: the

```shell
# open files
$ ulimit -n
1024
```

So, before executing `ab`, turn this option up, for example to 65536.

```shell
# 临时增大当前会话的最大文件描述符数
$ ulimit -n 65536
```

Next, go back and execute the `ab` command to perform the stress test:

```shell
# -c表示并发请求数为5000，-n表示总的请求数为10万
# -r表示套接字接收错误时仍然继续执行，-s表示设置每个请求的超时时间为2s
$ ab -c 5000 -n 100000 -r -s 2 http://192.168.0.30/
...
Requests per second:    6576.21 [#/sec] (mean)
Time per request:       760.317 [ms] (mean)
Time per request:       0.152 [ms] (mean, across all concurrent requests)
Transfer rate:          5390.19 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0  177 714.3      9    7338
Processing:     0   27  39.8     19     961
Waiting:        0   23  39.5     16     951
Total:          1  204 716.3     28    7349
...
```

I have already described the meaning of the `ab` output interface in the 🔗[How to evaluate the network performance of your system](https://time.geekbang.org/column/article/81497) article, so go ahead and review it yourself if you forget. From this interface, you can see that

- the average delay per request (Time per request) is 760ms;
- the average delay in establishing a connection (Connect) is 177ms.

Remember these values, which will be the baseline metrics for the next cases.

> Note that the results on your machine may not be the same as mine, but that's okay, it doesn't affect the idea of the next case study.

Next, go back to Terminal 1 and stop the Nginx application without using NAT:

```shell
$ docker rm -f nginx-hostnet
```

Then execute the following command to start today's case application. The case application listens on port 8080 and uses DNAT to implement a mapping relationship from port 8080 of the Host to port 8080 of the container:

```shell
$ docker run --name nginx --privileged -p 8080:8080 -itd feisky/nginx:nat
```

After Nginx starts, you can execute the `iptables` command to confirm that the DNAT rules have been created:

```shell
$ iptables -nL -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

...

Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:8080
```

You can see that in the PREROUTING chain, requests with a local destination go to the DOCKER chain, and in the DOCKER chain, tcp requests with a destination port of 8080 are DNAT to port 8080 on 172.17.0.2. Where 172.17.0.2 is the IP address of the Nginx container.

Next, we switch to Terminal 2 and execute the `curl` command to confirm that Nginx has started properly:

```shell
$ curl http://192.168.0.30:8080/
...
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Then, execute the above `ab` command again, but this time, change the requested port number to 8080:

```shell
# -c表示并发请求数为5000，-n表示总的请求数为10万
# -r表示套接字接收错误时仍然继续执行，-s表示设置每个请求的超时时间为2s
$ ab -c 5000 -n 100000 -r -s 2 http://192.168.0.30:8080/
...
apr_pollset_poll: The timeout specified has expired (70007)
Total of 5602 requests completed
```

Sure enough, `ab`, which was running fine earlier, is now failing and reporting a connection timeout error. The -s parameter when running `ab` sets the timeout to 2s per request, and as you can see from the output, only 5602 requests were completed this time.

Since we are trying to get the results of `ab`, we might as well extend the timeout to 30s, for example.

```shell
$ ab -c 5000 -n 10000 -r -s 30 http://192.168.0.30:8080/
...
Requests per second:    76.47 [#/sec] (mean)
Time per request:       65380.868 [ms] (mean)
Time per request:       13.076 [ms] (mean, across all concurrent requests)
Transfer rate:          44.79 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0 1300 5578.0      1   65184
Processing:     0 37916 59283.2      1  130682
Waiting:        0    2   8.7      1     414
Total:          1 39216 58711.6   1021  130682
...
```

Take a look at the output of `ab` again, this time the results show

- Requests per second (76);
- Latency per request (65s);
- Latency to establish a connection (Connect) (1300ms).

Obviously, each metric is much worse than the previous one.

So, what do you do when you encounter this problem? You can analyze it yourself based on the previous explanations, and then continue with what follows.

In the previous section, we used the `tcpdump` packet capture method to find out the root cause of the increased latency. So for today's case, we can still use a similar method to find the clues. However, let's change our thinking now, because today we already know the root cause of the problem in advance - that is, NAT.

Recall that in Netfilter, the flow of network packets and the principle of NAT, you will find that to ensure that NAT works properly, you need at least two steps:

- First, use the hook function (Hook) in Netfilter to modify the source or destination address.
- Second, use the connection tracking module conntrack to correlate requests and responses for the same connection.

Is there a problem in these two areas? Let's try it with `SystemTap`, the dynamic tracing tool mentioned earlier.

Since today's case is in a pressure testing scenario, the number of concurrent requests is much lower and we know full well that NAT is the culprit. So, it is reasonable to suspect that packet loss is occurring in the kernel.

We can go back to Terminal 1, create a script file <ins>dropwatch.stp</ins>, and write the following:

```shell
#! /usr/bin/env stap

############################################################
# Dropwatch.stp
# Author: Neil Horman <nhorman@redhat.com>
# An example script to mimic the behavior of the dropwatch utility
# http://fedorahosted.org/dropwatch
############################################################

# Array to hold the list of drop points we find
global locations

# Note when we turn the monitor on and off
probe begin { printf("Monitoring for dropped packets\n") }
probe end { printf("Stopping dropped packet monitor\n") }

# increment a drop counter for every location we drop at
probe kernel.trace("kfree_skb") { locations[$location] <<< 1 }

# Every 5 seconds report our drop locations
probe timer.sec(5)
{
  printf("\n")
  foreach (l in locations-) {
    printf("%d packets dropped at %s\n",
           @count(locations[l]), symname(l))
  }
  delete locations
}
```

This script, tracks calls to the kernel function `kfree_skb()` and counts the location of dropped packets. Once the file is saved, the packet loss tracking script can be run by executing the following `stap` command. Here `stap`, the command line tool for `SystemTap`, is:

```shell
$ stap --all-modules dropwatch.stp
Monitoring for dropped packets
```

When you see the probe begin output "Monitoring for dropped packets", it means that `SystemTap` has compiled the script into a kernel module and is up and running.

Next, we switch to Terminal 2 and execute the `ab` command again:

```shell
$ ab -c 5000 -n 10000 -r -s 30 http://192.168.0.30:8080/
```

Then, go back to Terminal 1 again and observe the output of the `stap` command:

```plaintext
10031 packets dropped at nf_hook_slow
676 packets dropped at tcp_v4_rcv

7284 packets dropped at nf_hook_slow
268 packets dropped at tcp_v4_rcv
```

You will notice that a lot of packet loss is happening in the nf_hook_slow position. When you see the name, you should be able to think that this is in the hook function of Netfilter Hook, and there is a packet loss problem. But it's not NAT, so we can't be sure yet. Next, we have to trace the execution of nf_hook_slow again, and this step can be done with `perf`.

We switch to terminal 2 and execute the `ab` command again:

```shell
$ ab -c 5000 -n 10000 -r -s 30 http://192.168.0.30:8080/
```

Then, switch back to Terminal 1 again and execute the `perf record` and `perf report` commands

```shell
# 记录一会（比如30s）后按Ctrl+C结束
$ perf record -a -g -- sleep 30

# 输出报告
$ perf report -g graph,0
```

In the `perf report` interface, enter the lookup command **/** Then, in the pop-up dialog box, enter **nf_hook_slow**; finally, expand the call stack to get the following call graph:

<center><a href="https://ibb.co/qgqbD80">
    <img src="https://i.ibb.co/c8fzy9b/42-Case-Perf.png" alt="42-Case-Perf" border="0">
</a></center>

From this diagram, we can see that nf_hook_slow is called most often in three places: ipv4_conntrack_in, br_nf_pre_routing, and iptable_nat_ipv4_in. In other words, nf_hook_slow is performing three main actions.

- First, when receiving a network packet, it looks up the connection in the connection tracking table and assigns a tracking object (Bucket) to the new connection.
- Second, it forwards packets in the Linux bridge. This is because the case Nginx is a Docker container, and the container's network is implemented through the bridge;
- Third, when receiving network packets, it performs a DNAT, which forwards the packets received on port 8080 to the container.

At this point, we have actually found three sources of performance degradation. These three sources are all Linux kernel mechanisms, so the next optimization is naturally to start with the kernel.

Based on the previous content of each resource module, we know that the Linux kernel provides the user with a large number of configurable options that can be viewed and modified via the **proc** file system, or the **sys** file system. In addition to this, you can also use the command line tool `sysctl` to view and modify the kernel configuration.

For example, our topic today is DNAT, and the basis of DNAT is conntrack, so we can start by looking at what conntrack configuration options the kernel provides.

In Terminal 1, we go ahead and execute the following command:

```shell
$ sysctl -a | grep conntrack
net.netfilter.nf_conntrack_count = 180
net.netfilter.nf_conntrack_max = 1000
net.netfilter.nf_conntrack_buckets = 65536
net.netfilter.nf_conntrack_tcp_timeout_syn_recv = 60
net.netfilter.nf_conntrack_tcp_timeout_syn_sent = 120
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
...
```

As you can see, the three most important metrics here are:

- net.netfilter.nf_conntrack_count, which indicates the current number of connection traces;
- net.netfilter.nf_conntrack_max, which indicates the maximum number of connection traces;
- net.netfilter.nf_conntrack_buckets, which indicates the connection trace table size.

So, this output tells us that the current number of connection traces is 180, the maximum number of connection traces is 1000, and the size of the connection tracking table is 65536.

If you think back to the `ab` command, the number of concurrent requests is 5000, and the number of requests is 100000, it is clear that setting the trace table to record only 1000 connections is not enough.

In fact, the kernel records exception information to the log when it works exceptionally. For example, in the previous `ab` test, the kernel already reported the error "nf_conntrack: table full" in the log. Executing the `dmesg` command, you can see that

```shell
$ dmesg | tail
[104235.156774] nf_conntrack: nf_conntrack: table full, dropping packet
[104243.800401] net_ratelimit: 3939 callbacks suppressed
[104243.800401] nf_conntrack: nf_conntrack: table full, dropping packet
[104262.962157] nf_conntrack: nf_conntrack: table full, dropping packet
```

Where net_ratelimit indicates that a large amount of logs are compressed, a kernel measure to prevent logging attacks. And when you see the error "nf_conntrack: table full", it means that nf_conntrack_max is too small.

So is it possible to just make the connection tracking table bigger? Before you adjust it, you have to understand that the connection tracking table is actually a hash table in memory. If the number of connection traces is too large, it will also consume a lot of memory.

In fact, the nf_conntrack_buckets we saw above is the size of the hash table. Each item in the hash table is a linked table (called Bucket), and the length of the linked table, then, is equal to nf_conntrack_max divided by nf_conntrack_buckets.

For example, we can estimate the size of the memory occupied by the connection tracking table configured above:

```plaintext
# The connection tracking object size is 376 and the chain table entry size is 16
nf_conntrack_max*"Connection Tracking Object Size"+nf_conntrack_buckets*"Link Table Item Size" 
= 1000*376+65536*16 B
= 1.4 MB
```

Next, let's change nf_conntrack_max to something larger, say 131072 (i.e. 2x nf_conntrack_buckets):

```shell
$ sysctl -w net.netfilter.nf_conntrack_max=131072
$ sysctl -w net.netfilter.nf_conntrack_buckets=65536
```

Then switch to Terminal 2 and execute the `ab` command again. Note that this time we also change the timeout back to the original 2s:

```shell
$ ab -c 5000 -n 100000 -r -s 2 http://192.168.0.30:8080/
...
Requests per second:    6315.99 [#/sec] (mean)
Time per request:       791.641 [ms] (mean)
Time per request:       0.158 [ms] (mean, across all concurrent requests)
Transfer rate:          4985.15 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0  355 793.7     29    7352
Processing:     8  311 855.9     51   14481
Waiting:        0  292 851.5     36   14481
Total:         15  666 1216.3    148   14645
```

Sure enough, now you can see that:

- Time per request is 791ms (760ms without NAT);
- Delay in establishing a connection is 355ms (177ms without NAT).

This result is already much better than the previous test and is very close to the initial benchmark results without NAT.

However, you may still be curious as to what exactly is included in the connection tracking table. And how is it refreshed here?

You can actually use the `conntrack` command line tool to see the contents of the connection tracking table. For example:

```shell
# -L表示列表，-o表示以扩展格式显示
$ conntrack -L -o extended | head
ipv4     2 tcp      6 7 TIME_WAIT src=192.168.0.2 dst=192.168.0.96 sport=51744 dport=8080 src=172.17.0.2 dst=192.168.0.2 sport=8080 dport=51744 [ASSURED] mark=0 use=1
ipv4     2 tcp      6 6 TIME_WAIT src=192.168.0.2 dst=192.168.0.96 sport=51524 dport=8080 src=172.17.0.2 dst=192.168.0.2 sport=8080 dport=51524 [ASSURED] mark=0 use=1
```

From here you can find the objects in the connection tracking table, including protocol, connection status, source IP, source port, destination IP, destination port, tracking status, etc. Since this format is fixed, we can use tools like `awk`, `sort`, etc., to analyze it statistically.

For example, let's take `ab` as an example. After starting the `ab` command in Terminal 2, go back to Terminal 1 and execute the following command:

```shell
# 统计总的连接跟踪数
$ conntrack -L -o extended | wc -l
14289

# 统计TCP协议各个状态的连接跟踪数
$ conntrack -L -o extended | awk '/^.*tcp.*$/ {sum[$6]++} END {for(i in sum) print i, sum[i]}'
SYN_RECV 4
CLOSE_WAIT 9
ESTABLISHED 2877
FIN_WAIT 3
SYN_SENT 2113
TIME_WAIT 9283

# 统计各个源IP的连接跟踪数
$ conntrack -L -o extended | awk '{print $7}' | cut -d "=" -f 2 | sort | uniq -c | sort -nr | head -n 10
  14116 192.168.0.2
    172 192.168.0.96
```

Here are the statistics for the total number of connection traces, the number of connection traces for each state of the TCP protocol, and the number of connection traces for each source IP. You can see that most of the TCP connection traces are in the TIME_WAIT state, and they mostly come from the IP address 192.168.0.2 (which is the VM2 where the `ab` command is running).

These connection traces that are in TIME_WAIT are cleaned up after a timeout, which by default is 120s, and you can check this by executing the following command:

```shell
$ sysctl net.netfilter.nf_conntrack_tcp_timeout_time_wait
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
```

So, if you have a very large number of connections, you should really consider, as well, reducing the timeout time appropriately.

In addition to these common configurations above, `conntrack` also contains many other configuration options, you can refer to nf_conntrack's 🔗[documentation](https://www.kernel.org/doc/Documentation/networking/) to configure them according to your actual needs nf_conntrack-sysctl.txt) to configure it.

## Summary

Today, I've taken you through how to troubleshoot and optimize performance issues caused by NAT.

Since NAT is implemented based on the connection tracking mechanism of the Linux kernel. So, when analyzing NAT performance problems, we can first analyze it from the `conntrack` perspective, such as using `systemtap`, `perf`, etc., to analyze the `conntrack` lines in the kernel; then, by adjusting the parameters of the netfilter kernel options, to optimize it.

In fact, this kind of NAT implemented by connection tracking mechanism in Linux is also often referred to as stateful NAT, and maintaining state, too, brings a high performance cost.

So, in addition to tuning the kernel behavior, in scenarios where stateful tracking is not required (e.g., only mapping by predefined IP and port, not dynamic mapping), we can also use stateless NAT (e.g., developed with tc or based on DPDK) to further improve performance.
