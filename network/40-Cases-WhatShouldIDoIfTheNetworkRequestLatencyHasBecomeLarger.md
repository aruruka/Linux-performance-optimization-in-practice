# 40 Cases: What should I do if the network request latency has become larger?

In the previous section, we learned about mitigation methods for encountering distributed denial of service (DDoS). As a brief recap, DDoS utilizes a large number of forged requests, causing the target service to consume a large number of resources to process these invalid requests, and thus cannot respond properly to normal user requests.

Due to the distributed, high traffic, difficult to track characteristics of DDoS, there is really no way to completely defend against the problems caused by DDoS, we can only try to mitigate the impact of DDoS.

For example, you can purchase professional traffic cleaning equipment and network firewall to block malicious traffic at the entrance of the network and only keep normal traffic into the servers in the data center.

In Linux servers, you can increase the server's resistance to attacks and reduce the impact of DDoS on normal services through various methods such as kernel tuning, DPDK, XDP, etc. And in applications, you can use caching at all levels, WAF, CDN, etc. to mitigate the impact of DDoS on applications.

However, note that if the DDoS traffic, has reached the Linux server, then, even if the application layer has done various optimizations, the latency of network services will generally still be much larger than normal.

Therefore, in practice, we usually have to let the Linux server, with professional traffic cleaning and network firewall equipment, together to alleviate this problem.

In addition to the increased network latency caused by DDoS, I'm sure you've seen many other causes of network latency, such as

- Slow network transmission, resulting in latency;
- Slow processing of Linux kernel stack messages, resulting in latency;
- Slow processing of application data, resulting in latency, etc.

So, what should we do when we encounter these causes of latency? How to locate the root cause of network latency? Today, I will take you through a case study to look at these problems together.

## Network latency

I'm sure that when referring to **network latency**, you may easily recall what it means - the time spent in network data transmission. Note, however, that this time may be one-way, meaning the one-way time from the source address to the destination address, or it may be two-way, meaning the time spent on the entire round trip from the source address to the destination address and then back to the response from the destination address.

Usually, we more commonly use the round-trip communication delay in both directions, such as the result of the `ping` test, which is the round-trip delay RTT (Round-Trip Time).

In addition to network latency, another common metric is **Application Latency**, which is the time taken from the time the application receives a request to the time it sends back a response. Usually, application latency is also referred to as round-trip latency, which is the sum of network data transfer time plus data processing time.

In addition to network latency, another common metric is application latency, which is the time taken from the time the application receives a request to the time it sends back a response. Often, application latency is also referred to as round-trip latency, and is the sum of network data transfer time plus data processing time.

In 🔗[Linux Networking Basics](https://time.geekbang.org/column/article/81057), I described that you can use `ping` to test network latency. `ping` is based on the ICMP protocol, and it gets the round-trip latency by calculating the time difference between the ICMP return response message and the ICMP return request message. This process does not require special authentication and is often exploited by many network attacks, such as the port scanning tool `nmap`, the packet grouping tool `hping3`, and so on.

So, to avoid these problems, many network services disable ICMP, which makes it impossible to use `ping` to test the availability and round-trip latency of network services. In this case, you can use `traceroute` or `hping3` in TCP and UDP mode to get the network latency.

For example, using <ins>baidu.com</ins> as an example, you can execute the following `hping3` command to test the network latency from your machine to the Baidu search server:

```shell
# -c表示发送3次请求，-S表示设置TCP SYN，-p表示端口号为80
$ hping3 -c 3 -S -p 80 baidu.com
HPING baidu.com (eth0 123.125.115.110): S set, 40 headers + 0 data bytes
len=46 ip=123.125.115.110 ttl=51 id=47908 sport=80 flags=SA seq=0 win=8192 rtt=20.9 ms
len=46 ip=123.125.115.110 ttl=51 id=6788  sport=80 flags=SA seq=1 win=8192 rtt=20.9 ms
len=46 ip=123.125.115.110 ttl=51 id=37699 sport=80 flags=SA seq=2 win=8192 rtt=20.9 ms

--- baidu.com hping statistic ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 20.9/20.9/20.9 ms
```

From the results of `hping3`, you can see that the round-trip latency RTT is 20.9ms.

Of course, we can get similar results with `traceroute`: `traceroute`:

```shell
# --tcp表示使用TCP协议，-p表示端口号，-n表示不对结果中的IP地址执行反向域名解析
$ traceroute --tcp -p 80 -n baidu.com
traceroute to baidu.com (123.125.115.110), 30 hops max, 60 byte packets
 1  * * *
 2  * * *
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  * * *
 9  * * *
10  * * *
11  * * *
12  * * *
13  * * *
14  123.125.115.110  20.684 ms *  20.798 ms
```

`traceroute` sends three packets per hop of the route and outputs the round-trip delay when a response is received. If there is no response or the response times out (default 5s), an asterisk will be output.

After knowing how to test network service latency based on TCP, next, let's learn the idea of analyzing network latency when it is elevated by a case study.

## Case preparation

The following example is still based on Ubuntu 18.04 and is applicable to other Linux systems as well. The case environment I used is the following:

- Machine configuration: 2 CPU, 8GB RAM.
- Pre-install `docker`, `hping3`, `tcpdump`, `curl`, `wrk`, `Wireshark` and other tools, such as `apt-get install docker.io hping3 tcpdump curl`.

You should be familiar with all the tools here, including the installation and use of `wrk`, which was described in 🔗[How to evaluate the network performance of your system](https://time.geekbang.org/column/article/81497). If you have not already done so, install it by executing the following command:

```shell
$ https://github.com/wg/wrk
$ cd wrk
$ apt-get install build-essential -y
$ make
$ sudo cp wrk /usr/local/bin/
```

Since `Wireshark` requires a graphical interface, if your virtual machine does not have one, you can install `Wireshark` into another machine (e.g. a Windows laptop).

This case uses two virtual machines, and I have drawn a diagram to represent their relationship.

<center><a href="https://ibb.co/G5HGVYZ">
    <img src="https://i.ibb.co/sVWcFxT/40-Case-Vm-Role-Diagram.png" alt="40-Case-Vm-Role-Diagram" border="0">
</a></center>

Next, we open two terminals, SSH to each machine (for the following steps, assume that the terminal number matches the VM number shown), and install these tools mentioned above. Note that `curl` and `wrk` only need to be installed in the client VM (i.e. VM2).

As in previous cases, all commands below are run as root by default. If you are logged in as a normal user, run `sudo su root` to switch to the root user.

> If you have any problems with the installation process, you are encouraged to search and solve them by yourself first.
> If you have already installed it before, you can ignore this.

Next, let's move on to the case study.

## Case analysis

To compare the impact of increased latency, let's first run the simplest version of Nginx, which is to start a container with the official Nginx image. In Terminal 1, run the following command to run the official Nginx, which will listen on port 80:

```shell
$ docker run --network=host --name=good -itd nginx
fb4ed7cb9177d10e270f8320a7fb64717eac3451114c9fab3c50e02be2e88ba2
```

Continuing in Terminal 1, run the case application by executing the following command, which will listen on port 8080:

```shell
$ docker run --name nginx --network=host -itd feisky/nginx:latency
b99bd136dcfd907747d9c803fdc0255e578bad6d66f4e9c32b826d75b6812724
```

Then, execute the `curl` command in Terminal 2 to verify that both containers have started properly. If everything is fine, you will see the following output:

```shell
# 80端口正常
$ curl http://192.168.0.30
<!DOCTYPE html>
<html>
...
<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# 8080端口正常
$ curl http://192.168.0.30:8080
...
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Next, let's use `hping3`, as mentioned above, to test their latency and see what the difference is. Still in terminal 2, execute the following command to test the latency of port 80 and port 8080 on the case machine respectively:

```shell
# 测试80端口延迟
$ hping3 -c 3 -S -p 80 192.168.0.30
HPING 192.168.0.30 (eth0 192.168.0.30): S set, 40 headers + 0 data bytes
len=44 ip=192.168.0.30 ttl=64 DF id=0 sport=80 flags=SA seq=0 win=29200 rtt=7.8 ms
len=44 ip=192.168.0.30 ttl=64 DF id=0 sport=80 flags=SA seq=1 win=29200 rtt=7.7 ms
len=44 ip=192.168.0.30 ttl=64 DF id=0 sport=80 flags=SA seq=2 win=29200 rtt=7.6 ms

--- 192.168.0.30 hping statistic ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 7.6/7.7/7.8 ms
```

```shell
# 测试8080端口延迟
$ hping3 -c 3 -S -p 8080 192.168.0.30
HPING 192.168.0.30 (eth0 192.168.0.30): S set, 40 headers + 0 data bytes
len=44 ip=192.168.0.30 ttl=64 DF id=0 sport=8080 flags=SA seq=0 win=29200 rtt=7.7 ms
len=44 ip=192.168.0.30 ttl=64 DF id=0 sport=8080 flags=SA seq=1 win=29200 rtt=7.6 ms
len=44 ip=192.168.0.30 ttl=64 DF id=0 sport=8080 flags=SA seq=2 win=29200 rtt=7.3 ms

--- 192.168.0.30 hping statistic ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 7.3/7.6/7.7 ms
```

As you can see from this output, the latency is about the same for both ports, 7ms, but this is only for a single request. What about concurrent requests? Next, let's try it with `wrk`.

This time, in Terminal 2, run the following new command to test the performance of port 80 and port 8080 when the case machine is concurrently 100:

```shell
# 测试80端口性能
$ wrk --latency -c 100 -t 2 --timeout 2 http://192.168.0.30/
Running 10s test @ http://192.168.0.30/
  2 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     9.19ms   12.32ms 319.61ms   97.80%
    Req/Sec     6.20k   426.80     8.25k    85.50%
  Latency Distribution
     50%    7.78ms
     75%    8.22ms
     90%    9.14ms
     99%   50.53ms
  123558 requests in 10.01s, 100.15MB read
Requests/sec:  12340.91
Transfer/sec:     10.00MB
```

```shell
# 测试8080端口性能
$ wrk --latency -c 100 -t 2 --timeout 2 http://192.168.0.30:8080/
Running 10s test @ http://192.168.0.30:8080/
  2 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    43.60ms    6.41ms  56.58ms   97.06%
    Req/Sec     1.15k   120.29     1.92k    88.50%
  Latency Distribution
     50%   44.02ms
     75%   44.33ms
     90%   47.62ms
     99%   48.88ms
  22853 requests in 10.01s, 18.55MB read
Requests/sec:   2283.31
Transfer/sec:      1.85MB
```

As you can see from the above two outputs, the average latency of the official Nginx (listening on port 80) is 9.19 ms, while the average latency of the case Nginx (listening on port 8080) is 43.6 ms. In terms of latency distribution, 90% of the official Nginx requests can be completed within 9 ms, while 50% of the case Nginx requests already reach 44 ms.

Combining this with the output of `hping3` above, it is easy to see that the latency of Nginx in the case is much higher for concurrent requests.

I think you have already thought of the analysis method you learned in the last lesson, using `tcpdump` to capture the incoming and outgoing network packets and analyze the network sending and receiving process to see if there are any problems.

Next, let's execute the following `tcpdump` command in Terminal 1 to capture the packets sent and received on port 8080 and save them to the <ins>nginx.pcap</ins> file: `tcpdump`:

```shell
$ tcpdump -nn tcp port 8080 -w nginx.pcap
```

Then switch to Terminal 2 and re-execute the `wrk` command:

```shell
# 测试8080端口性能
$ wrk --latency -c 100 -t 2 --timeout 2 http://192.168.0.30:8080/
```

When the `wrk` command finishes, switch back to Terminal 1 again and press Ctrl+C to end the `tcpdump` command. Then, copy the captured <ins>nginx.pcap</ins> to the machine with `Wireshark` (if VM1 already comes with a GUI, you can skip the copy step) and open it with `Wireshark`.

Since there are a lot of network packages, we can filter them first. For example, after selecting a packet, you can right-click and select `Follow` -> `TCP Stream`, as shown below:

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/1Rx8C7y/40-Case-Wireshark-Filter-Packet.png" alt="40-Case-Wireshark-Filter-Packet" border="0">
</a></center>

Then, close the pop-up dialog box and go back to the main `Wireshark` window. At this point, you will see that `Wireshark` has automatically set a filter expression `tcp.stream eq 24` for you, as shown in the following figure (source and destination IP addresses are omitted in the figure):

<center><a href="https://ibb.co/wQtxNwm">
    <img src="https://i.ibb.co/7ph6JRm/40-Case-Wireshark-Filter-Packet-2.png" alt="40-Case-Wireshark-Filter-Packet-2" border="0">
</a></center>

From here, you can see every request and response for this TCP connection from three handshakes onwards. Of course, this may not be intuitive enough, you can go ahead and click on Statics -> Flow Graph in the menu bar, check "Limit to display filter" and set the Flow type to "TCP Flows ":

<center><a href="https://ibb.co/68227rX">
    <img src="https://i.ibb.co/cwPPSkY/40-Case-Wireshark-Filter-Packet-3.png" alt="40-Case-Wireshark-Filter-Packet-3" border="0">
</a></center>

Note that the left side of this diagram is the client, while the right side is the Nginx server. As you can see from this diagram, the first three handshakes, as well as the first HTTP request and response, are still quite fast, but the second HTTP request is slower, especially when the client receives the first packet from the server and sends an ACK response only 40ms later (blue line in the diagram).

When you see the value 40ms, does anything come to mind? Actually, this is the minimum timeout for TCP Delayed ACK.

Let me explain Delayed Acknowledgement here. This is an optimization mechanism for TCP ACKs, which means that instead of sending an ACK for every request, you wait for a while (say 40ms) to see if there are any "tailwinds". If there is another packet to be sent during this time, the ACK will be sent along with it. Of course, if there is no other packet, then send the ACK alone after the timeout.

Since the 40ms in this case occurs on the client, we have reason to suspect that the client has enabled the delayed acknowledgement mechanism. The client here is actually the `wrk` running earlier.

Query the TCP documentation (execute `man tcp`) and you will see that only TCP sockets with TCP_QUICKACK specifically set will have fast acknowledgement mode enabled; otherwise, by default, the delayed acknowledgement mechanism is used: the

```plaintext
TCP_QUICKACK (since Linux 2.4.4)
              Enable  quickack mode if set or disable quickack mode if cleared.  In quickack mode, acks are sent imme‐
              diately, rather than delayed if needed in accordance to normal TCP operation.  This flag is  not  perma‐
              nent,  it only enables a switch to or from quickack mode.  Subsequent operation of the TCP protocol will
              once again enter/leave quickack mode depending on internal  protocol  processing  and  factors  such  as
              delayed ack timeouts occurring and data transfer.  This option should not be used in code intended to be
              portable.
```

To verify our suspicions and confirm the behavior of `wrk`, we can use `strace` to see which TCP options `wrk` sets for the socket.

For example, you can switch to Terminal 2 and execute the following command:

```shell
$ strace -f wrk --latency -c 100 -t 2 --timeout 2 http://192.168.0.30:8080/
...
setsockopt(52, SOL_TCP, TCP_NODELAY, [1], 4) = 0
...
```

Thus, you can see that `wrk` only has the TCP_NODELAY option set, but not TCP_QUICKACK. This means that `wrk` is using exactly delayed acknowledgement, which explains the 40ms problem above.

However, don't forget that this is only client-side behavior, and by definition, the Nginx server should not be affected by this behavior. So is there a clue that we missed when analyzing the network packets? Let's go back to `Wireshark` and take a fresh look.

<center><a href="https://ibb.co/rHP8BwP">
    <img src="https://i.ibb.co/HqvwcHv/40-Case-Wireshark-Filter-Packet-4.png" alt="40-Case-Wireshark-Filter-Packet-4" border="0">
</a></center>

Look closely at the `Wireshark` interface, where packet 1173 is the delayed ACK packet mentioned earlier, and the next line, 1175, is the second packet sent by Nginx, which is combined with packet 697 to form a complete HTTP response (both ACK numbers are 85).

The second packet is not sent along with the previous packet (#697), but instead waits until after the client ACKs the first packet (#1173), which looks a bit like a delayed acknowledgement, except that instead of ACKs, the data is sent.

Seeing this, I guess you remember something - Nagle's algorithm (Nagle's algorithm). Before I analyze the case further, let me briefly introduce this algorithm.

Nagle algorithm, an optimization algorithm used in TCP protocol to reduce the number of small packets sent in order to improve the actual bandwidth utilization.

For example, when the payload is only 1 byte, plus 20 bytes for the TCP header and 20 bytes for the IP header, the entire network packet is 41 bytes, so the actual bandwidth utilization is only 2.4% (1/41). On the larger side, if the entire network bandwidth is filled with such small packets, the effective utilization of the entire network is too low.

The Nagle algorithm is designed to solve this problem. It improves network bandwidth utilization by combining TCP packets. nagle algorithm specifies that on a TCP connection, there can be at most one unacknowledged outstanding packet; no other packet is sent until the ACK for this packet is received. These small packets are combined and sent out with the same packet after the ACK is received.

Obviously, the Nagle algorithm itself is a pretty good idea, but knowing Linux's default delayed acknowledgement mechanism, you should not think so. Because when they are used together, the network latency will be noticeable. As shown in the figure below:

<center><a href="https://ibb.co/Gcyd2Xm">
    <img src="https://i.ibb.co/D5h1CNd/Tcp-Delayed-Ack-And-Nagle.png" alt="Tcp-Delayed-Ack-And-Nagle" border="0">
</a></center>

- When Sever sends the first packet, it will wait for 40ms before replying ACK because Client has delayed acknowledgement enabled.
- At the same time, Server will also wait here because Nagle is enabled on Server side, and the ACK of the first packet is not received at this time.
- The Client will not reply to the ACK until after the 40ms timeout, and then the Server will continue to send the second packet.

Since Nagle may be the problem, how do you know if Nagle is enabled in the case of Nginx?

If you check the tcp documentation, you will know that the Nagle algorithm is disabled only when TCP_NODELAY is set. So, we just need to check the tcp_nodelay option of Nginx.

```plaintext
TCP_NODELAY
              If set, disable the Nagle algorithm.  This means that segments are always sent as soon as possible, even
              if there is only a small amount of data.  When not set, data is buffered until  there  is  a  sufficient
              amount  to  send out, thereby avoiding the frequent sending of small packets, which results in poor uti‐
              lization of the network.  This option is overridden by TCP_CORK; however, setting this option forces  an
              explicit flush of pending output, even if TCP_CORK is currently set.
```

Let's go back to Terminal 1 and execute the following command to view the configuration of the case Nginx.

```shell
$ docker exec nginx cat /etc/nginx/nginx.conf | grep tcp_nodelay
    tcp_nodelay    off;
```

Sure enough, you can see that the tcp_nodelay is off in the case Nginx, so set it to on and you should be able to fix it.

After changing it, is the problem solved? Naturally, we need to verify it. I have packaged the modified application in a Docker image, and you can start it by executing the following command in Terminal 1:

```shell
# 删除案例应用
$ docker rm -f nginx

# 启动优化后的应用
$ docker run --name nginx --network=host -itd feisky/nginx:nodelay
```

Next, switch to Terminal 2 and re-execute `wrk` to test the latency:

```shell
$ wrk --latency -c 100 -t 2 --timeout 2 http://192.168.0.30:8080/
Running 10s test @ http://192.168.0.30:8080/
  2 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     9.58ms   14.98ms 350.08ms   97.91%
    Req/Sec     6.22k   282.13     6.93k    68.50%
  Latency Distribution
     50%    7.78ms
     75%    8.20ms
     90%    9.02ms
     99%   73.14ms
  123990 requests in 10.01s, 100.50MB read
Requests/sec:  12384.04
Transfer/sec:     10.04MB
```

Sure enough, the latency is now reduced to 9ms, which is the same as the official Nginx image we tested (Nginx has tcp_nodelay turned on by default).

For comparison, let's use `tcpdump` to grab the optimized network packets (here we are actually grabbing port 80, where the official Nginx listens). You can get the following results:

<center><a href="https://ibb.co/pztVFhP">
    <img src="https://i.ibb.co/wW9GmcL/40-Case-Wireshark-Filter-Packet-5.png" alt="40-Case-Wireshark-Filter-Packet-5" border="0">
</a></center>

As you can see from the diagram, since Nginx no longer has to wait for the ACK, the 536 and 540 packets are sent consecutively; and the client, although it still has delayed acknowledgement turned on, receives two packets that need to reply to the ACK, so it does not have to wait 40ms and can directly merge and reply to the ACK.

At the end of the case, don't forget to stop the two container applications. In Terminal 1, execute the following command to delete the case application:

```shell
$ docker rm -f nginx good
```

## Summary

Today, we learned how to analyze network latency after it increases. Network latency, is the most core network performance indicator. Network latency is inevitable due to various factors such as network transmission and network packet processing. However, excessive network latency will directly affect the user experience.

Therefore, after finding an increase in network latency, you can use various tools such as `traceroute`, `hping3`, `tcpdump`, `Wireshark`, `strace`, etc. to locate potential problems in the network. For example.

- Use tools such as `hping3` and `wrk` to confirm that the network latency is correct for single requests and concurrent request cases.
- Use `traceroute` to confirm that the route is correct and to see the latency of the gateway for each hop in the route.
- Use `tcpdump` and `Wireshark` to confirm that network packets are being sent and received properly.
- Use `strace`, etc., to see if the application calls to the network sockets are working properly.

This way, you can sequentially troubleshoot from routing, to sending and receiving network packets, to applications, etc., layer by layer until you locate the root cause of the problem.
