# 36 Routines: How to evaluate the network performance of the system?

In the previous section, we reviewed the classic C10K and C1000K problems. As a brief recap, C10K is the problem of how to handle 10,000 requests (10,000 concurrent connections) on a single machine, while C1000K is the problem of handling 1 million requests (1 million concurrent connections) on a single machine.

The best solution to the C10K problem is the optimization of the I/O model. epoll, introduced in Linux 2.6, solved the C10K problem perfectly and has been used ever since. Many of today's high-performance networking solutions are still based on epoll.

Naturally, the proliferation of Internet technology has led to higher performance requirements. From C10K to C100K, we only need to increase the physical resources of the system to meet the requirements; but from C100K to C1000K, increasing the physical resources alone is not enough.

At this point, the hardware and software of the system should be optimized in a unified way, from the interrupt handling of the hardware, to the number of file descriptors of the network protocol stack, connection status tracking, cache queue, to the whole network link of the working model of the application, etc., all need to be optimized in depth.

Taking it a step further, implementing C10M is not a problem that can be solved by increasing physical resources and tuning the kernel and applications. This is where the lengthy network protocol stack in the kernel becomes the biggest burden.

- The XDP approach is needed to process network packets before the kernel stack.
- Or, based on DPDK, it can skip the network stack and process it in user space by polling.

Among them, DPDK is the most mainstream high-performance network solution, but it requires a DPDK-capable NIC to work with.

Of course, in practice, in most scenarios, we do not need 10 million concurrent requests on a single machine. It is simpler and more scalable to distribute requests to multiple servers in parallel by adjusting the system architecture.

However, this scenario requires us to evaluate the network performance of the system in order to examine the processing capacity of the system and to provide benchmark data for capacity planning.

So, how exactly do you evaluate the performance of your network? Today, I'll take a look at this question.

## Performance metrics review

Before we evaluate network performance, let's review the metrics that measure network performance. In the Linux Networking Basics chapter, we talked about bandwidth, throughput, latency, PPS, etc., which are the most commonly used network performance metrics. Do you remember what they mean exactly? You can think about it for a while before continuing below.

First, **Bandwidth**, indicates the maximum transmission rate of the link in b/s (bits/sec). Bandwidth is the core reference indicator when you are shopping for a NIC for your server. Commonly used bandwidths are 1000M, 10G, 40G, 100G, etc.

Second, **throughput**, which indicates the maximum data transfer rate without packet loss, usually in b/s (bits/s) or B/s (bytes/s). Throughput is limited by bandwidth, and throughput/bandwidth is the utilization rate of that network link.

Third, **latency**, indicates the time delay from the time a network request is sent until a response is received from the far end. This metric may have different meanings in different scenarios. It can indicate the time required to establish a connection (e.g., TCP handshake delay), or the time required for a packet round trip (e.g., RTT).

Finally, **PPS**, which stands for Packet Per Second, indicates the transmission rate in terms of network packets.PPS is often used to evaluate the forwarding capacity of a network, which, based on a Linux server, is easily affected by the size of network packets (switches are usually not affected too much, i.e., they can forward linearly).

Of these four metrics, bandwidth is directly related to the physical NIC configuration. Generally, once the NIC is determined, the bandwidth is also determined (of course, the actual bandwidth will be limited by the smallest module in the entire network link).

In addition, you may have heard of "network bandwidth tests" in many places, but this is not actually a test of bandwidth, but of network throughput, which is generally smaller than bandwidth for Linux servers, and closer to bandwidth for specialized network devices such as switches.

The final PPS, which is the network transmission rate in network packets, is typically used in scenarios where a lot of forwarding is required. And for TCP or Web services, metrics such as concurrent connections and requests per second (QPS, Query per Second), which are more reflective of actual application performance, will be used.

## Network benchmarking

Having familiarized yourself with the performance metrics of the network, let's next look at how to benchmark these metrics through performance testing.

You can start by thinking about a problem. We already know that Linux networks are based on the TCP/IP stack, and the behavior of the different protocol layers is obviously different. Before testing, then, you should figure out which layer of the protocol stack does the network performance you want to evaluate belong to? In other words, which layer of the protocol stack is your application based on?

Based on what you learned earlier about the principles of the TCP/IP protocol stack, this question should not be difficult to answer. For example:

- Web applications based on HTTP or HTTPS, which obviously belong to the application layer, require us to test the performance of HTTP/HTTPS;
- and for most game servers, in order to support a larger number of simultaneous online users, they are usually based on TCP or UDP and interact with clients, which requires us to test the performance of TCP/UDP;
- and of course, there are some Of course, there are scenarios where Linux is used as a soft switch or router. In this case, you are more concerned about the network packet processing capability (i.e. PPS), focusing on the forwarding performance of the network layer.

Next, I'll take you from the bottom up through the network performance testing methods for the different protocol layers. Note, however, that the lower layer protocols are the basis for the network protocols at the layers above them. Naturally, the performance of the lower layer protocols, in turn, determines the network performance of the higher layers.

Note that for all the following test methods, two Linux virtual machines are required. One, which can be used as the target machine to be tested, and the other, which can be used as a client running network services to run the test tools.

### Performance testing of each protocol layer

#### Forwarding performance

Let's look first at the network interface layer and the network layer, which are responsible for encapsulating, addressing, routing, and sending and receiving network packets. In these two network protocol layers, the number of network packets that can be processed per second, PPS, is the most important performance metric. In particular, the processing capability of 64B small packets deserves special attention. So, how to test the network packet processing capability?

When it comes to network packet-related tests, you may feel unfamiliar. However, in the CPU performance section at the beginning of this column, we have touched on a related tool, `hping3`, in the soft interrupt case.

In that case, `hping3` was used as a tool for SYN attacks. In fact, `hping3` is used more as a performance tool to test the network packet processing capability.

Today I will introduce another more commonly used tool, 🔗[pktgen](https://wiki.linuxfoundation.org/networking/pktgen), a high-performance network testing tool that comes with the Linux kernel. `pktgen` supports a rich set of customization options, making it easy for you to construct the network packets you need to more accurately test the performance of your target server.

However, you cannot find the `pktgen` command directly on Linux systems. Because `pktgen` runs as a kernel thread, you need to load the `pktgen` kernel module and then interact with it via the <ins>/proc</ins> file system. The following is the interaction file between the two kernel threads started by `pktgen` and the <ins>/proc</ins> filesystem:

```shell
$ modprobe pktgen
$ ps -ef | grep pktgen | grep -v grep
root     26384     2  0 06:17 ?        00:00:00 [kpktgend_0]
root     26385     2  0 06:17 ?        00:00:00 [kpktgend_1]
$ ls /proc/net/pktgen/
kpktgend_0  kpktgend_1  pgctrl
```

`pktgen` starts a kernel thread on each CPU and can interact with these threads via the file of the same name under <ins>/proc/net/pktgen</ins>, while pgctrl is mainly used to control the start and stop of this test.

> If the `modprobe` command fails, your kernel is not configured with the CONFIG_NET_PKTGEN option. > This requires you to configure the `pktgen` kernel module (i.e. CONFIG_NET_PKTGEN=m) and then recompile the kernel before it can be used.

When using `pktgen` to test network performance, you need to configure the `pktgen` option for each kernel thread, kpktgend_X, and the test NIC, before starting the test via `pgctrl`.

Take the packet sending test as an example, assuming that the sending machine uses eth0, and the target machine has an IP address of 192.168.0.30 and a MAC address of 11:11:11:11:11:11.

<center><a href="https://ibb.co/nPg5TgB">
    <img src="https://i.ibb.co/7kY6HYQ/Forwarding-Performance-pktgen.png" alt="Forwarding-Performance-pktgen" border="0">
</a></center>

Next, is an example of an outgoing package test.

```shell
# 定义一个工具函数，方便后面配置各种测试选项
function pgset() {
    local result
    echo $1 > $PGDEV

    result=`cat $PGDEV | fgrep "Result: OK:"`
    if [ "$result" = "" ]; then
         cat $PGDEV | fgrep Result:
    fi
}

# 为0号线程绑定eth0网卡
PGDEV=/proc/net/pktgen/kpktgend_0
pgset "rem_device_all"   # 清空网卡绑定
pgset "add_device eth0"  # 添加eth0网卡

# 配置eth0网卡的测试选项
PGDEV=/proc/net/pktgen/eth0
pgset "count 1000000"    # 总发包数量
pgset "delay 5000"       # 不同包之间的发送延迟(单位纳秒)
pgset "clone_skb 0"      # SKB包复制
pgset "pkt_size 64"      # 网络包大小
pgset "dst 192.168.0.30" # 目的IP
pgset "dst_mac 11:11:11:11:11:11"  # 目的MAC

# 启动测试
PGDEV=/proc/net/pktgen/pgctrl
pgset "start"
```

After a little while, the results are available from the <ins>/proc</ins> file system when the test is finished. We can view the test report we just ran by looking at the following code snippet:

```shell
$ cat /proc/net/pktgen/eth0
Params: count 1000000  min_pkt_size: 64  max_pkt_size: 64
     frags: 0  delay: 0  clone_skb: 0  ifname: eth0
     flows: 0 flowlen: 0
...
Current:
     pkts-sofar: 1000000  errors: 0
     started: 1534853256071us  stopped: 1534861576098us idle: 70673us
...
Result: OK: 8320027(c8249354+d70673) usec, 1000000 (64byte,0frags)
  120191pps 61Mb/sec (61537792bps) errors: 0
```

As you can see, the test report is divided into three main sections:

- Params in the first section are the test options;
- Current in the second section is the test progress, where packets so far (pkts-sofar) means that 1 million packets have been sent, which means that the test is complete.
- The third part, Result, is the test result, which contains the time spent on the test, the number of packets and slices, PPS, throughput, and the number of errors.

Based on the results above, we find that the PPS is 120,000 and the throughput is 61 Mb/s with no errors. So, is 120,000 PPS good?

As a comparison, you can calculate the PPS of a Gigabit switch. the switch can reach wire speed (full load with error-free forwarding), its PPS is 1000Mbit divided by the size of the Ethernet frame, which is $1000Mbps/((64+20)*8bit) = 1.5 Mpps$ (where 20B is the size of the Ethernet frame leading and frame spacing).

As you can see, even the PPS of a Gigabit switch can reach 1.5 million PPS, which is much more than the 120,000 we got from our test. So, you don't have to worry about this value, as multicore servers and 10GbE NICs are common nowadays and can reach millions of PPS with a little optimization, and, if you use DPDK or XDP, which we talked about in the last lesson, you can also reach tens of millions.

#### TCP/UDP performance

After mastering the PPS testing method, let's look at the TCP and UDP performance testing methods. When it comes to TCP and UDP testing, I think you are already familiar with them, and you may even be able to think of the corresponding testing tools, such as `iperf` or `netperf`, at once.

Especially in the era of cloud computing, the first thing you should do when you first get a batch of virtual machines is to use `iperf` and test whether the network performance meets the expectations.

Both `iperf` and `netperf` are the most commonly used network performance testing tools to test TCP and UDP throughput. They both test the average throughput over a period of time in terms of client and server communication.

Next, let's take a look at `iperf` as an example of how TCP performance is tested. Currently, the latest version of `iperf` is `iperf3`, which you can install by running the following command:

```shell
# Ubuntu
apt-get install iperf3
# CentOS
yum install iperf3
```

Then, start the `iperf` server on the target machine:

```shell
# -s表示启动服务端，-i表示汇报间隔，-p表示监听端口
$ iperf3 -s -i 1 -p 10000
```

Next, run the `iperf` client on another machine and run the test at

```shell
# -c表示启动客户端，192.168.0.30为目标服务器的IP
# -b表示目标带宽(单位是bits/s)
# -t表示测试时间
# -P表示并发数，-p表示目标服务器监听端口
$ iperf3 -c 192.168.0.30 -b 1G -t 15 -P 2 -p 10000
```

After waiting a little while (15 seconds) for the test to finish, go back to the target server and check the report of `iperf`:

```plaintext
[ ID] Interval           Transfer     Bandwidth
...
[SUM]   0.00-15.04  sec  0.00 Bytes  0.00 bits/sec                  sender
[SUM]   0.00-15.04  sec  1.51 GBytes   860 Mbits/sec                  receiver
```

The last SUM line is the summary result of the test, including test time, data transfer volume, and bandwidth. This section is divided into two rows, sender and receiver, according to sending and receiving.

As you can see from the test results, the bandwidth (throughput) of the TCP receiver on this machine is 860 Mb/s, which is still a bit short of the target of 1Gb/s.

#### HTTP performance

From the transport layer further up, to the application layer. There are applications that build services directly based on TCP or UDP. Of course, there are also a large number of applications that build services based on application layer protocols, and HTTP is one of the most common application layer protocols. For example, various Web services such as Apache and Nginx are commonly used and are based on HTTP.

To test the performance of HTTP, there are a large number of tools available, such as `ab`, `webbench`, etc., which are commonly used HTTP stress testing tools. Among them, `ab` is Apache's own HTTP stress test tool, which mainly tests the number of requests per second, request latency, throughput and the distribution of request latency of HTTP services.

You can install the `ab` tool by running the following command:

```shell
# Ubuntu
$ apt-get install -y apache2-utils
# CentOS
$ yum install -y httpd-tools
```

Next, on the target machine, start an Nginx service using Docker and test its performance with `ab`. First, run the following command on the target machine:

```shell
$ docker run -p 80:80 -itd nginx
```

On the other machine, run the `ab` command to test Nginx's performance:

```shell
# -c表示并发请求数为1000，-n表示总的请求数为10000
$ ab -c 1000 -n 10000 http://192.168.0.30/
...
Server Software:        nginx/1.15.8
Server Hostname:        192.168.0.30
Server Port:            80

...

Requests per second:    1078.54 [#/sec] (mean)
Time per request:       927.183 [ms] (mean)
Time per request:       0.927 [ms] (mean, across all concurrent requests)
Transfer rate:          890.00 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0   27 152.1      1    1038
Processing:     9  207 843.0     22    9242
Waiting:        8  207 843.0     22    9242
Total:         15  233 857.7     23    9268

Percentage of the requests served within a certain time (ms)
  50%     23
  66%     24
  75%     24
  80%     26
  90%    274
  95%   1195
  98%   2335
  99%   4663
 100%   9268 (longest request)
```

As you can see, the results of the `ab` test are divided into three parts: request summary, connection time summary, and request delay summary. Let's take the above results as an example and look at them in detail.

In the request summary section, you can see that

- Requests per second is 1074;
- Latency per request is divided into two rows, the first row of 927 ms represents the average latency, including the scheduling time of the threads running and the network request response time, while the next row of 0.927 ms represents the actual request response time;
- Transfer rate indicates a throughput (BPS) of 890 KB/s.

The connection time summary section shows the minimum, maximum, average, and median processing times for connection establishment, request, wait, and summary, respectively.

Finally, the request latency summary section gives the percentage of requests processed in different time periods, e.g., 90% of requests can be completed within 274ms.

### Application Load Performance

When you use testing tools like `iperf` or `ab` and get performance data for TCP, HTTP, etc., does that data indicate the actual performance of the application? I think your answer should be no.

For example, your application is based on the HTTP protocol and provides a Web service to the end user. At this point, using the `ab` tool, you can get the access performance of a certain page, but this result is likely to be inconsistent with the actual request from the user. This is because user requests often come with various kinds of payloads that affect the internal processing logic of the web application, thus affecting the final performance.

To get the actual performance of the application, then, requires that the performance tools themselves simulate the user request load, which tools like `iperf` and `ab` can't do. Fortunately, we can also use `wrk`, `TCPCopy`, `Jmeter` or `LoadRunner` to achieve this goal.

Take `wrk` for example, it is an HTTP performance testing tool with LuaJIT built in, which allows you to generate the required request load or customize the response handling according to your actual needs.

The `wrk` tool itself does not provide `yum` or `apt` installation methods, you need to install it by compiling the source code. For example, you can run the following command to compile and install `wrk`:

```shell
$ https://github.com/wg/wrk
$ cd wrk
$ apt-get install build-essential -y
$ make
$ sudo cp wrk /usr/local/bin/
```

The command-line arguments to `wrk` are relatively simple. For example, we can use `wrk` to re-measure the performance of a previously started Nginx.

```shell
# -c表示并发连接数1000，-t表示线程数为2
$ wrk -c 1000 -t 2 http://192.168.0.30/
Running 10s test @ http://192.168.0.30/
  2 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    65.83ms  174.06ms   1.99s    95.85%
    Req/Sec     4.87k   628.73     6.78k    69.00%
  96954 requests in 10.06s, 78.59MB read
  Socket errors: connect 0, read 0, write 0, timeout 179
Requests/sec:   9641.31
Transfer/sec:      7.82MB
```

The performance of Nginx was retested here using 2 threads and 1000 concurrent connections. As you can see, the number of requests per second is 9641, the throughput is 7.82MB, and the average latency is 65ms, which is much better than the results of the previous `ab` test.

This also shows that the performance of the performance tool itself, is also crucial for performance testing. An unsuitable performance tool does not accurately measure the best performance of an application.

Of course, the biggest advantage of `wrk` is its built-in LuaJIT, which can be used to implement performance tests for complex scenarios. When calling Lua scripts, `wrk` can divide HTTP requests into three phases, namely setup, running, and done, as shown in the following figure:

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/D9H54F2/wrk-Internal-Stages.png" alt="wrk-Internal-Stages" border="0">
</a></center>

(Image from 🔗 [NetEaseCloud blog](https://sq.sf.163.com/blog/article/200008406328934400))

For example, you can set authentication parameters for the request (from the `wrk` official 🔗[example](https://github.com/wg/wrk/blob/master/scripts/auth.lua)) during the setup phase:

```shell
-- example script that demonstrates response handling and
-- retrieving an authentication token to set on all future
-- requests

token = nil
path  = "/authenticate"

request = function()
   return wrk.format("GET", path)
end

response = function(status, headers, body)
   if not token and status == 200 then
      token = headers["X-Token"]
      path  = "/resource"
      wrk.headers["X-Token"] = token
   end
end
```

And when executing the test, with the -s option, the path to execute the script:

```shell
$ wrk -c 1000 -t 2 -s auth.lua http://192.168.0.30/
```

`wrk` requires you to use a Lua script to construct the request load. This is probably sufficient for most scenarios . However, it also has the exact disadvantage that everything needs to be constructed in code and the tool itself does not provide a GUI environment.

Products like Jmeter or LoadRunner (commercial products) provide richer features such as script recording, playback, GUI, etc. for complex scenarios, and are much easier to use.

## Summary

Today, I took you through a review of network performance metrics and learned how network performance is evaluated.

Performance evaluation is a prerequisite for optimizing network performance, and network performance optimization is only needed when you find network performance bottlenecks. According to the principle of TCP/IP protocol stack, different protocol layers focus on different performance priorities that are not exactly the same and correspond to different performance testing methods. For example.

- At the application layer, you can use `wrk`, `Jmeter`, etc. to simulate user load and test the number of requests per second, processing latency, errors, etc. of your application;
- at the transport layer, you can use tools like `iperf` to test TCP throughput;
- further down, you can also use `pktgen`, which comes with the Linux kernel, to test the PPS of your server.

Since lower layer protocols are the foundation of higher layer protocols. Therefore, in general, we need to perform performance tests for each protocol layer from top to bottom, and then, based on the results of the performance tests and the principles of the Linux network protocol stack, we can identify the root causes of performance bottlenecks and then optimize network performance.
