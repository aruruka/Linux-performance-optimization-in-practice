# 10 Cases: The CPU usage of soft interrupts in the system increases, what should I do?

In the last issue, I told you the basic principles of soft interrupts. Let's
briefly review them first.

Interrupt is an asynchronous event processing mechanism, which is used to improve
the concurrent processing capability of the system. When an interrupt event occurs,
it will trigger the execution of the interrupt handler, and the interrupt handler
is divided into two parts, the upper half and the lower half.

- The first half corresponds to hard interrupts, which are used to quickly process interrupts;
- The second half corresponds to the soft interrupt, which is used to asynchronously process the unfinished work of the upper part.

Softirqs in Linux include various types such as network sending and receiving,
timing, scheduling, and RCU locks. We can check <ins>/proc/softirqs</ins> in the
proc file system to observe the running status of softirqs.

In Linux, each CPU corresponds to a soft interrupt kernel thread, the name is
ksoftirqd/CPU number. When the frequency of soft interrupt events is too high,
the kernel thread will also cause soft interrupt processing not in time due to
high CPU usage, which will cause performance problems such as network sending
and receiving delays and slow scheduling.

Softirqs High CPU usage is also one of the most common performance issues. Today,
I will use the most common case of reverse proxy server Nginx to teach you how to
analyze this situation.

## Case

### Your preparation

The following examples are based on Ubuntu 18.04 and are also applicable to other
Linux systems. The case environment I'm using is this:

- Machine configuration: 2 CPUs, 8 GB memory.
- Pre-install docker, sysstat, sar, hping3, tcpdump and other tools, such as `apt-get install docker.io sysstat hping3 tcpdump`.

Here I have used three new tools, `sar`, `hping3` and `tcpdump`, a brief introduction:

- `sar` is a system activity reporting tool, which can not only view the current activity of the system in real time, but also can be configured to save and report historical statistics.
- `hping3` is a tool that can construct TCP/IP protocol packets, and can perform security audits and firewall tests on the system.
- `tcpdump` is a commonly used network packet capture tool, which is often used to analyze various network problems.

In this case, two virtual machines are used, and I drew a picture to show their relationship.

<center><a href="https://ibb.co/XbkBgKG">
    <img src="https://i.ibb.co/VtpyZzk/10-Case-Analysis-Vm-Role-Diagram.png" alt="10-Case-Analysis-Vm-Role-Diagram" border="0">
</a></center>

You can see that one of the virtual machines runs Nginx to simulate the web server
to be analyzed; the other acts as a client of the web server to add pressure requests
to Nginx. The purpose of using two virtual machines is to isolate each other and
avoid "cross-infection".

Next, we open two terminals, log in to the two machines with SSH, and install
the tools mentioned above.

As in the previous case, all the following commands are run as the root user by
default. If you log in to the system as a normal user, please run the `sudo su root`
command to switch to the root user.

If you have any problems during the installation process, you are also encouraged
to search and solve them yourself first. If you can’t solve them, you can ask me
questions in the message area. If you have already installed it before, you can
ignore this.

### Operation and analysis

After the installation is complete, we first execute the following command on
the first terminal to run the case, which is the most basic Nginx application:

```shell
# 运行Nginx服务并对外开放80端口
$ docker run -itd --name=nginx -p 80:80 nginx
```

Then, in the second terminal, use `curl` to access the port where Nginx is listening,
and confirm that Nginx starts normally. Assuming that 192.168.153.130 is the IP
address of the virtual machine where Nginx is located, after running the `curl`
command, you should see the following output interface:

```shell
$ curl http://192.168.153.130/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

Next, still in the second terminal, we run the `hping3` command to simulate Nginx
client requests:

```shell
# -S参数表示设置TCP协议的SYN（同步序列号），-p表示目的端口为80
# -i u100表示每隔100微秒发送一个网络帧
# 注：如果你在实践过程中现象不明显，可以尝试把100调小，比如调成10甚至1
$ hping3 -S -p 80 -i u100 192.168.153.130
```

Now let's go back to the first terminal, you should find the exception. Do you
feel that the system response is obviously slowed down, even if you just hit enter
in the terminal a few times, it takes a long time to get a response? What should
we do at this time?

Although I told you when running the `hping3` command that this is a SYN FLOOD
attack, you must also think of starting from the network to analyze this problem.
However, in an actual production environment, no one tells you why directly.

Therefore, I hope you temporarily forget the operation of `hping3` simulating
SYN FLOOD, and then start from the observed problems again, analyze the resource
usage of the system, and find out the root of the problem step by step.

So, where to start? Just now we found that simple SHELL commands are significantly
slower. It should be a good idea to check the overall resource usage of the system.
For example, run `top` to see if there is a CPU bottleneck. We run the top command
in the first terminal to see the resource usage of the system as a whole.

```shell
# top运行后按数字1切换到显示所有CPU
$ top
top - 10:50:58 up 1 days, 22:10,  1 user,  load average: 0.00, 0.00, 0.00
Tasks: 122 total,   1 running,  71 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.0 us,  0.0 sy,  0.0 ni, 96.7 id,  0.0 wa,  0.0 hi,  3.3 si,  0.0 st
%Cpu1  :  0.0 us,  0.0 sy,  0.0 ni, 95.6 id,  0.0 wa,  0.0 hi,  4.4 si,  0.0 st
...

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    7 root      20   0       0      0      0 S   0.3  0.0   0:01.64 ksoftirqd/0
   16 root      20   0       0      0      0 S   0.3  0.0   0:01.97 ksoftirqd/1
 2663 root      20   0  923480  28292  13996 S   0.3  0.3   4:58.66 docker-containe
 3699 root      20   0       0      0      0 I   0.3  0.0   0:00.13 kworker/u4:0
 3708 root      20   0   44572   4176   3512 R   0.3  0.1   0:00.07 top
    1 root      20   0  225384   9136   6724 S   0.0  0.1   0:23.25 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.03 kthreadd
...
```

Have you noticed anything unusual here? Let's start with the first line and look
at it one by one:

- The average load is all 0, and there is only one process (1 running) in the ready queue.
- The usage rate of each CPU is quite low, and the usage rate of the highest CPU1 is only 4.4%, which is not high.
- Looking at the process list, the process with the highest CPU usage is only 0.3%, which is not high.

So why is the system's response slowed down? Since the value of each indicator
is not large, let's take a look at the more specific meanings of these indicators.
After all, even if the same indicator is used in different parts and scenarios of
the system, it may correspond to different performance problems.

Looking carefully at the output of `top`, although the usage rates of the two CPUs
are only 3.3% and 4.4% respectively, they are both used in soft interrupts; and
from the process list, we can also see that the soft interrupt process ksoftirqd
has the highest CPU usage. It seems that soft interrupts are a bit suspicious.

According to the content of the previous issue, since there may be problems with
soft interrupts, you must first know which type of soft interrupts is the problem.
Stop and think about it, what method did we use in the previous section to judge
the type of soft interrupt? That's right, it's still the proc file system. By
observing the contents of the <ins>/proc/softirqs</ins> file, you can know the
number of various softirq types.

However, the number of various soft interrupts here, and the number of times in
what time period? It is the **cumulative number of interruptions** since the
system was running. So we directly check the content of the file, and what we get
is only the cumulative number of interruptions, which has no direct reference
significance for the problem here. Because, **the rate of change of the number
of these interruptions** is what we need to pay attention to.

So what tool can observe the changes in the output of the command? I think you
should remember, the `watch` command used in the previous case, you can run a
command regularly to view the output; if you add the -d parameter, you can also
highlight the changed part, from the highlighted part we can intuitively see
which content changes faster.

For example, still in the first terminal, we run the following command:

```shell
$ watch -d cat /proc/softirqs
                    CPU0       CPU1
          HI:          0          0
       TIMER:    1083906    2368646
      NET_TX:         53          9
      NET_RX:    1550643    1916776
       BLOCK:          0          0
    IRQ_POLL:          0          0
     TASKLET:     333637       3930
       SCHED:     963675    2293171
     HRTIMER:          0          0
         RCU:    1542111    1590625
```

Through the changes in the contents of the <ins>/proc/softirqs</ins> file, you
can find that TIMER (timer interrupt), NET_RX (network reception), SCHED
(kernel scheduling), RCU (RCU lock) and other soft interrupts are constantly
changing.

Among them, NET_RX, that is, the change rate of the network data packet receiving
soft interrupt is the fastest. Several other types of softirqs are necessary to
ensure that Linux scheduling, clocks, and critical section protection work properly,
so it is normal for them to have some changes.

Then, we will start with the soft interrupt received by the network and continue
the analysis. Since it is a soft interrupt for network reception, the first step
should be to observe the network reception of the system. You may think of many
network tools here, but I recommend today's protagonist tool `sar`.

`sar` can be used to view the network sending and receiving of the system. Another
advantage is that not only the throughput of the network sending and receiving
(BPS, the number of bytes sent and received per second), but also the PPS of the
network sent and received, that is, the number of packets sent and received per
second can be observed, which is network frames count.

We run the `sar` command in the first terminal, and add the "-n DEV" parameter
to display a report of network traffic:

```shell
# -n DEV 表示显示网络收发的报告，间隔1秒输出一组数据
$ sar -n DEV 1
15:03:46        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
15:03:47         eth0  12607.00   6304.00    664.86    358.11      0.00      0.00      0.00      0.01
15:03:47      docker0   6302.00  12604.00    270.79    664.66      0.00      0.00      0.00      0.00
15:03:47           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
15:03:47    veth9f6bbcd   6302.00  12604.00    356.95    664.66      0.00      0.00      0.00      0.05
```

For the output interface of `sar`, let me briefly introduce it first, from left to right:

- First column: Indicates the time of the report.
- Second column: IFACE indicates the network card.
- The third and fourth columns: rxpck/s and txpck/s indicate the number of network frames received and sent per second, that is, PPS.
- The fifth and sixth columns: rxkB/s and txkB/s respectively represent the number of kilobytes received and sent per second, that is, BPS.
- The other parameters behind are basically close to 0, obviously not directly related to today's problem, you can ignore it first.

Let's look at the output in detail, you can find:

- For the network card eth0, the number of network frames received per second is relatively large, reaching 12607, while the number of network frames sent is relatively small, only 6304; the number of kilobytes received per second is only 664 KB, while the number of kilobytes sent The section count is smaller, only 358 KB.
- The data of docker0 and veth9f6bbcd is basically the same as that of eth0, except that the sending and receiving are opposite, the sent data is larger and the received data is smaller. This is caused by the forwarding of the Linux internal bridge. You don’t need to delve into it for now, as long as you know that the system forwards the packets received by eth0 to the Nginx service. The specific working principle will be introduced in detail in the network part later.

From these data, did you find anything unusual?

Since it is suspected that the network reception is interrupted, we still focus
on eth0: the received PPS is relatively large, reaching 12607, while the received
BPS is very small, only 664 KB. Intuitively, the network frames should be relatively
small. Let’s calculate a little bit, $664*1024/12607 = 54$ bytes, indicating that
each network frame is only 54 bytes on average, which is obviously a very small
network frame. This is what we usually call the "small packet" problem.

Next, we run the `tcpdump` command in the first terminal, specify the network
card eth0 through the "-i eth0" option, and specify "port 80" of the TCP protocol
through the tcp port 80 option:

```shell
# -i eth0 只抓取eth0网卡，-n不解析协议名和主机名
# tcp port 80表示只抓取tcp协议并且端口号为80的网络帧
$ tcpdump -i eth0 -n tcp port 80
15:11:32.678966 IP 192.168.153.131.18238 > 192.168.153.130.80: Flags [S], seq 458303614, win 512, length 0
...
```

From the output of tcpdump, you can find

- 192.168.153.131.18238 > 192.168.153.130.80, which means that the network frame is sent from port 18238 of 192.168.153.131 to port 80 of 192.168.153.130, that is, the network frame is sent from port 18238 of the machine running hping3, and the destination is the machine where Nginx is located port 80.
- Flags [S] indicate that this is a SYN packet.

In addition to the fact that the PPS exceeds 12,000 found by `sar`, we can now
confirm that this is a SYN FLOOD attack sent from the address 192.168.153.131.

So far, we have done a full set of performance diagnosis and analysis. Starting
from the high usage rate of softirqs in the system, by observing the changes in
the <ins>/proc/softirqs</ins> file, it can be judged that the type of softirq is
a network receiving interrupt; and then through `sar` and `tcpdump`, it is
confirmed that this is a SYN FLOOD problem.

The easiest solution to the SYN FLOOD problem is to block the source IP from the
switch or hardware firewall, so that the SYN FLOOD network frame will not be sent
to the server.

As for the principle of SYN FLOOD and more solutions, you don’t need to pay too
much attention for now, we will learn about it in the following network chapters.

After the case is over, don’t forget to close it. Remember to stop the Nginx
service and `hping3` command that were started at the beginning.

In the first terminal, run the following command to stop Nginx:

```shell
# 停止 Nginx 服务
$ docker rm -f nginx
```

Then go to the second terminal and press Ctrl+C to stop `hping3`.

## Summary

Increased softirq CPU usage is a very common performance problem. Although there
are many types of soft interrupts, in actual production, most of the performance
bottlenecks we encounter are network sending and receiving soft interrupts,
especially network receiving soft interrupts.

When encountering such problems, you can use tools such as `sar` and `tcpdump`
for further analysis. Don't be afraid of network performance, I will teach you
more analysis methods later.