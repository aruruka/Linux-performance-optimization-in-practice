# 37 Cases: What should I do if DNS resolution is somtimes fast and sometimes slow?

In the last section, I took you through the network performance evaluation methodology. To briefly recap, Linux networks are built on the TCP/IP stack, and at different layers of the stack, we are concerned with different network performance.

At the application layer, we are concerned with the number of concurrent connections, requests per second, processing latency, errors, etc. of the application, and can use tools like `wrk`, `JMeter`, etc. to simulate the user load and get the desired test results.

As for the transport layer, we are concerned about the working status of transport layer protocols such as TCP, UDP, etc., such as the number of TCP connections, TCP retransmissions, TCP errors, etc. At this point, you can use `iperf`, `netperf`, etc. to test the performance of TCP or UDP.

Moving down to the network layer, we are concerned about the network packet processing capability, i.e. PPS. `pktgen`, which comes with the Linux kernel, can help you test this metric.

Since the lower layer protocols are the basis for the higher layer protocols, in general, what we call network optimization actually includes optimization of all layers of the entire network protocol stack. Of course, performance requirements vary, and the specific locations and targets to be optimized are not exactly the same.

When evaluating network performance (e.g., HTTP performance), we specified the IP address of the network service in the test tool. Each IP address consists of two parts: a host number and a network number. Hosts with the same network number form a subnet; different subnets are then connected by routers to form a large network.

However, while IP addresses facilitate the communication of machines, they place a heavy burden of memory on the people who access these services. I don't think many people can remember the IP address of GitHub because it's a string of characters that doesn't make much sense to the human brain and doesn't fit our memory logic.

However, that doesn't stop us from using the service a lot. Why? Because there is a simpler and more convenient way, of course. Instead of having to rely on a specific IP address, we can access it via the domain name <ins>github.com</ins>, which is actually where the Domain Name System DNS comes from.

DNS (Domain Name System), or Domain Name System, is one of the most basic services in the Internet, providing a query service for mapping relationships between domain names and IP addresses.

DNS not only facilitates access to different Internet services, but also provides a dynamic service discovery and Global Server Load Balance (GSLB) mechanism for many applications. In this way, DNS can select the nearest IP to the user to provide the service. Even if the IP address of the back-end service changes, the user can still access it with the same domain name.

DNS is obviously a fundamental and important part of our work. So, how do you analyze and troubleshoot DNS problems? Today, I'll take a look at this issue.

## Domain names and DNS resolution

A domain name, which we are all familiar with in itself, consists of a string of characters separated by dots and is used as the name of a computer or a group of computers in the Internet for the purpose of easy identification, the location of the hosts providing various services in the Internet.

It is important to note that domain names are globally unique and need to be applied for registration only through a specialized domain name registrar. To organize the many computers in the global Internet, domain names are likewise separated by dots, forming a hierarchical structure. Each string separated by dots constitutes a hierarchy in the domain name, and the further back the position, the higher the hierarchy.

Let's take the geek time website <ins>time.geekbang.org</ins> as an example to understand the meaning of the domain name. In this string, the org at the end is the top-level domain, the geekbang in the middle is the second-level domain, and the time on the far left is the third-level domain.

As shown in the figure below, note that the dot (.) is the root of all domain names, which means that all domain names are suffixed with dot, and it can also be understood that all domain names end with dot in the domain name resolution process.

<center><a href="https://ibb.co/mHsbV37">
    <img src="https://i.ibb.co/2j2W1TB/Domain-Name-System-Example.png" alt="Domain-Name-System-Example" border="0">
</a></center>

By understanding these concepts, you can see that domain names are mainly for the convenience of being remembered, while IP addresses are the real mechanism for communication between machines. The service that converts domain names into IP addresses is what we mentioned at the beginning, the Domain Name Resolution Service (DNS), and the corresponding server is the domain name server, and the network protocol is the DNS protocol.

Note here that the DNS protocol is part of the application layer in the TCP/IP stack, but the actual transport is based on UDP or TCP protocols (UDP mostly), and the domain name servers generally listen on port 53.

Since domain names are managed in a hierarchical structure, domain name resolution is in fact recursive (starting from the top level and so on) and is sent to each level of the name servers until the resolution result is obtained.

But don't worry, the recursive query process doesn't require you to do it yourself, the DNS server will do it for you, all you have to do is to pre-configure an available DNS server and you're done.

Of course, we know that, in general, each level of DNS server will have a cache of recently resolved records. When the cache hits, you can just answer directly with the records in the cache. If the cache is out of date or does not exist, then you need to query it recursively as mentioned above.

Therefore, when configuring the network of a Linux system, the system administrator needs to configure the DNS server in addition to the IP address, so that it can access external services via the domain name.

For example, my system is configured with the name server 114.114.114.114. You can execute the following command to query your system configuration:

```shell
$ cat /etc/resolv.conf
nameserver 114.114.114.114
```

In addition, the DNS service manages all data by means of resource records, which support various types of records such as A, CNAME, MX, NS, PTR, etc. For example:

- A record, which is used to convert a domain name into an IP address;
- CNAME record, which is used to create an alias;
- and NS record, which indicates the address of the domain name server corresponding to the domain name.

Simply put, when we access a URL, we need to look up the IP address corresponding to the domain name through the A record of DNS, and then access the Web service through that IP.

For example, let's take Geek Time's website <ins>time.geekbang.org</ins> as an example, and execute the following `nslookup` command to look up the A record of the domain name, and we can see that its IP address is <ins>39.106.233.176</ins>:

```shell
$ nslookup time.geekbang.org
# 域名服务器及端口信息
Server:    114.114.114.114
Address:  114.114.114.114#53

# 非权威查询结果
Non-authoritative answer:
Name:  time.geekbang.org
Address: 39.106.233.17
```

Note here that since 114.114.114.114 is not the name server that directly manages <ins>time.geekbang.org</ins>, the query results are non-authoritative. Using the above command, you will only get the results of the 114.114.114.114 query.

As mentioned earlier, DNS queries are actually a recursive process if there is no hit cache, so is there any way to know the execution of the entire recursive query?

In fact, besides `nslookup`, another common DNS resolution tool, `dig`, provides a `trace` function that can show the whole process of recursive queries. For example, you can execute the following command to get the query result:

```shell
# +trace表示开启跟踪查询
# +nodnssec表示禁止DNS安全扩展
$ dig +trace +nodnssec time.geekbang.org

; <<>> DiG 9.11.3-1ubuntu1.3-Ubuntu <<>> +trace +nodnssec time.geekbang.org
;; global options: +cmd
.      322086  IN  NS  m.root-servers.net.
.      322086  IN  NS  a.root-servers.net.
.      322086  IN  NS  i.root-servers.net.
.      322086  IN  NS  d.root-servers.net.
.      322086  IN  NS  g.root-servers.net.
.      322086  IN  NS  l.root-servers.net.
.      322086  IN  NS  c.root-servers.net.
.      322086  IN  NS  b.root-servers.net.
.      322086  IN  NS  h.root-servers.net.
.      322086  IN  NS  e.root-servers.net.
.      322086  IN  NS  k.root-servers.net.
.      322086  IN  NS  j.root-servers.net.
.      322086  IN  NS  f.root-servers.net.
;; Received 239 bytes from 114.114.114.114#53(114.114.114.114) in 1340 ms

org.      172800  IN  NS  a0.org.afilias-nst.info.
org.      172800  IN  NS  a2.org.afilias-nst.info.
org.      172800  IN  NS  b0.org.afilias-nst.org.
org.      172800  IN  NS  b2.org.afilias-nst.org.
org.      172800  IN  NS  c0.org.afilias-nst.info.
org.      172800  IN  NS  d0.org.afilias-nst.org.
;; Received 448 bytes from 198.97.190.53#53(h.root-servers.net) in 708 ms

geekbang.org.    86400  IN  NS  dns9.hichina.com.
geekbang.org.    86400  IN  NS  dns10.hichina.com.
;; Received 96 bytes from 199.19.54.1#53(b0.org.afilias-nst.org) in 1833 ms

time.geekbang.org.  600  IN  A  39.106.233.176
;; Received 62 bytes from 140.205.41.16#53(dns10.hichina.com) in 4 ms
```

The output of `dig trace` consists of four main parts.

- In the first part, some root name servers (.) are found from 114.114.114.114 of NS records.
- In the second part, we select one of the NS record results (h.root-servers.net) and query the NS record of the top-level domain org.
- The third part is to select one of the NS records of org. (b0.org.afilias-nst.org) and query the NS servers of the second-level domain geekbang.org.
- The last part is to query the A record of the final host time.geekbang.org. from the NS server of geekbang.org. (dns10.hichina.com).

The NS records for each level of domain name shown in this output are actually the addresses of the domain name servers at each level, so you can have a better understanding of the DNS resolution process. To help you understand the recursive query more intuitively, I've put together a flowchart of the process, which you can save to understand.

<center><a href="https://ibb.co/5crJ4FQ">
    <img src="https://i.ibb.co/F75d3VQ/DNS-Procedure-Example.png" alt="DNS-Procedure-Example" border="0">
</a></center>

Of course, it's not only services published to the Internet that need domain names, but many times we also want to be able to resolve domain names (i.e. intranet domain names, in most cases hostnames) for hosts inside the LAN. Linux supports this behavior as well.

So, you can write the mapping between hostnames and IP addresses to the local <ins>/etc/hosts</ins> file. This way, the specified hostname can find the target IP directly locally. for example, you can do this by executing the following command:

```shell
$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain
::1         localhost6 localhost6.localdomain6
192.168.0.100 domain.com
```

Alternatively, you can set up custom DNS servers on your intranet that are dedicated to resolving domain names on your intranet. For intranet DNS servers, you can also set up one or more upstream DNS servers to resolve domain names on the extranet.

Once you are clear about the basic principles of domain names and DNS resolution, I will take you through a few cases to analyze how to locate DNS resolution problems in practice.

## Case preparation

This case is also based on Ubuntu 18.04, which is also applicable to other Linux systems. The case environment I used is shown below:

- Machine configuration: 2 CPU, 8GB RAM.
- Pre-install tools like docker, such as `apt install docker.io`.

You can start by opening a terminal, SSH into the Ubuntu machine, and then run the following command to pull the Docker image used in the case:

```shell
$ docker pull feisky/dnsutils
Using default tag: latest
...
Status: Downloaded newer image for feisky/dnsutils:latest
```

Then, run the following command to view the host's currently configured DNS servers:

```shell
$ cat /etc/resolv.conf
nameserver 114.114.114.114
```

As you can see, the DNS server I configured for this host is 114.114.114.114.

Here, the preparation work is done. Next, let's officially enter the operation.

## Case analysis

### Case 1: DNS resolution failure

First, execute the following command to enter the first case of the day. If everything works, you will be able to see the following output:

```shell
# 进入案例环境的SHELL终端中
$ docker run -it --rm -v $(mktemp):/etc/resolv.conf feisky/dnsutils bash
root@7e9ed6ed4974:/#
```

Note that the 7e9ed6ed4974 after root is the ID prefix of the Docker-generated container, which is likely to be a different ID in your environment, so just ignore it.

> Note: In the following code snippet, the commands starting with "/#" all indicate commands run inside the container.

Then, continuing in the container terminal, we execute the DNS query command, still looking for the IP address of <ins>time.geekbang.org</ins>:

You can find that this command blocked for a long time, but still failed, reported connection timed out and no servers could be reached error.

When you see this, I guess your first thought is that the network is down. We use the ping tool to check to try. Execute the following command to test the local connectivity to 114.114.114.114:

```shell
/# ping -c3 114.114.114.114
PING 114.114.114.114 (114.114.114.114): 56 data bytes
64 bytes from 114.114.114.114: icmp_seq=0 ttl=56 time=31.116 ms
64 bytes from 114.114.114.114: icmp_seq=1 ttl=60 time=31.245 ms
64 bytes from 114.114.114.114: icmp_seq=2 ttl=68 time=31.128 ms
--- 114.114.114.114 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 31.116/31.163/31.245/0.058 ms
```

In this output, you can see that the network is working. Then how to know why the `nslookup` command failed? There are actually many ways to do this, and the easiest one is to turn on the debug output of `nslookup` and check the detailed steps of the query process to see if there are any abnormalities in it.

For example, we can continue in the container terminal and execute the following command:

```shell
/# nslookup -debug time.geekbang.org
;; Connection to 127.0.0.1#53(127.0.0.1) for time.geekbang.org failed: connection refused.
;; Connection to ::1#53(::1) for time.geekbang.org failed: address not available.
```

As you can see from the output this time, `nslookup` failed to connect to port 53 at the loopback address (127.0.0.1 and ::1). Here's the question: Why would it connect to the loopback address instead of the 114.114.114.114 we saw earlier?

You may have already figured out the crux of the problem - it could be because there is no DNS server configured in the container. Then let's run the following command to confirm:

```shell
/# cat /etc/resolv.conf
```

Sure enough, there is no output from this command, indicating that there is indeed no DNS server configured in the container. At this point, naturally, we know the solution. In the <ins>/etc/resolv.conf</ins> file, just configure the DNS server.

To achieve a successful resolution, execute the following command and rerun the `nslookup` command after configuring the DNS server. The resolution should now be successful.

```shell
/# echo "nameserver 114.114.114.114" > /etc/resolv.conf
/# nslookup time.geekbang.org
Server:    114.114.114.114
Address:  114.114.114.114#53

Non-authoritative answer:
Name:  time.geekbang.org
Address: 39.106.233.176
```

At this point, the first case is easily solved. Finally, execute the `exit` command in the terminal to exit the container, and Docker will automatically clean up the container you just ran.

### Case 2: Unstable DNS resolution

Next, let's look at the second case. Execute the following command to start a new container and enter it in the terminal:

```shell
$ docker run -it --rm --cap-add=NET_ADMIN --dns 8.8.8.8 feisky/dnsutils bash
root@0cd3ee0c8ecb:/#
```

Then, as in the previous case, you run the `nslookup` command to resolve the IP address of <ins>time.geekbang.org</ins>. But this time, add a `time` command to output the time taken to resolve. If everything works, you may see the following output:

```shell
/# time nslookup time.geekbang.org
Server:    8.8.8.8
Address:  8.8.8.8#53

Non-authoritative answer:
Name:  time.geekbang.org
Address: 39.106.233.176

real  0m10.349s
user  0m0.004s
sys  0m0.0
```

As you can see, the parsing was very slow, taking 10 seconds. If you run the above `nslookup` command several times, you may occasionally encounter the following error:

```shell
/# time nslookup time.geekbang.org
;; connection timed out; no servers could be reached

real  0m15.011s
user  0m0.006s
sys  0m0.006s
```

In other words, similar to the previous case, there will be a failure to resolve. Taken together, DNS resolution is now not only slower, but also results in timeout failures.

Why is that? What should I do if I encounter this problem?

In fact, according to the previous explanation, we know that DNS resolution is, frankly speaking, the process of interaction between the client and the server, and this process also uses the UDP protocol.

So, for the whole process, there are many possible cases of unstable resolution results. Let's say:

- the DNS server itself is faulty, slow and unstable;
- or, the network latency from the client to the DNS server is high;
- or, DNS requests or response packets are, in some cases, lost by network devices in the link.

According to the output of `nslookup` above, you can see that the client is now connected to DNS 8.8.8.8, which is the DNS service provided by Google. We still have confidence in Google, the probability of DNS server problems should be relatively small. Basically, the problem of DNS server has been ruled out, so is it possible that the delay from local machine to DNS server is bigger?

As mentioned before, `ping` can be used to test the latency of the server. For example, you can run the following command:

```shell
/# ping -c3 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=31 time=137.637 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=31 time=144.743 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=31 time=138.576 ms
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 137.637/140.319/144.743/3.152 ms
```

As you can see from the `ping` output, the latency here is already 140ms, which explains why the parsing is so slow. In fact, if you run the above `ping` test several times, you will also see occasional packet loss.

```shell
$ ping -c3 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=30 time=134.032 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=30 time=431.458 ms
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 2 packets received, 33% packet loss
round-trip min/avg/max/stddev = 134.032/282.745/431.458/148.713 ms
```

This further explains why `nslookup` occasionally fails, precisely because of packet loss in the network link.

What should I do if I encounter this problem? Obviously, since the latency is too high, switch to a DNS server with less latency, such as 114.114.114.114 provided by Telecom.

Before configuring it, we can test it with `ping` to see if its latency is really better than 8.8.8.8. You can see that it has a latency of 31ms by running the following command:

```shell
/# ping -c3 114.114.114.114
PING 114.114.114.114 (114.114.114.114): 56 data bytes
64 bytes from 114.114.114.114: icmp_seq=0 ttl=67 time=31.130 ms
64 bytes from 114.114.114.114: icmp_seq=1 ttl=56 time=31.302 ms
64 bytes from 114.114.114.114: icmp_seq=2 ttl=56 time=31.250 ms
--- 114.114.114.114 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 31.130/31.227/31.302/0.072 ms
```

This result shows that the latency is indeed much smaller. Let's continue with the following command, change the DNS server, and then, again, execute the `nslookup` resolution command:

```shell
/# echo nameserver 114.114.114.114 > /etc/resolv.conf
/# time nslookup time.geekbang.org
Server:    114.114.114.114
Address:  114.114.114.114#53

Non-authoritative answer:
Name:  time.geekbang.org
Address: 39.106.233.176

real    0m0.064s
user    0m0.007s
sys     0m0.006s
```

You can see that it now takes only 64ms to complete the parsing, which is much better than the 10s it took earlier.

At this point, the problem looks like it's solved. However, if you run the nslookup command multiple times, you may not get good results every time. For example, on my machine, it often takes 1s or more.

```shell
/# time nslookup time.geekbang.org
Server:    114.114.114.114
Address:  114.114.114.114#53

Non-authoritative answer:
Name:  time.geekbang.org
Address: 39.106.233.176

real  0m1.045s
user  0m0.007s
sys  0m0.004s
```

The 1s DNS resolution time is still too long and unacceptable for many applications. So, how to solve this problem? I'm sure you've already thought of it: use DNS caching. This way, only the first query needs to be requested from the DNS server, and subsequent queries can use the records in the cache as long as the DNS records do not expire.

Note, however, that the major Linux distributions we use do not automatically configure DNS caching, except for the latest versions of Ubuntu (such as 18.04 or newer).

So, to enable DNS caching for your system, you will need to do additional configuration. For example, the easiest way to do this is to use `dnsmasq`.

`dnsmasq` is one of the most commonly used DNS caching services, and is also often used as a DHCP service. It is relatively easy to install and configure, and its performance meets the needs of most applications for DNS caching.

Let's go ahead and start `dnsmasq` by executing the following command in the container terminal we just used:

```shell
/# /etc/init.d/dnsmasq start
 * Starting DNS forwarder and DHCP server dnsmasq                    [ OK ]
```

Then, modify <ins>/etc/resolv.conf</ins> to change the DNS server to the listening address of `dnsmasq`, in this case 127.0.0.1. Next, re-execute the `nslookup` command several times:

```shell
/# echo nameserver 127.0.0.1 > /etc/resolv.conf
/# time nslookup time.geekbang.org
Server:    127.0.0.1
Address:  127.0.0.1#53

Non-authoritative answer:
Name:  time.geekbang.org
Address: 39.106.233.176

real  0m0.492s
user  0m0.007s
sys  0m0.006s

/# time nslookup time.geekbang.org
Server:    127.0.0.1
Address:  127.0.0.1#53

Non-authoritative answer:
Name:  time.geekbang.org
Address: 39.106.233.176

real  0m0.011s
user  0m0.008s
sys  0m0.003s
```

Now we can see that only the first resolution is slow, taking 0.5s, and each subsequent resolution is fast, taking only 11ms, and each subsequent DNS resolution takes a stable amount of time.

At the end of the case, don't forget to execute exit to exit the container terminal and Docker will clean up the case container automatically.

## Summary

Today, I have taken you through the basic principles of DNS, and through several cases, I have taken you through the analysis and solution ideas when you find DNS resolution problems.

DNS is one of the most basic services in the Internet, providing a query service for mapping relationships between domain names and IP addresses. Many applications do not consider DNS resolution when they are initially developed, and it takes days to find out that the problem is caused by slow DNS resolution.

Imagine a Web service interface that takes 1s to wait for DNS resolution each time, then no matter how you optimize the application's internal logic, the interface's response is too slow for the user because the response time is always greater than 1 second.

Therefore, during application development, we must take into account the performance issues that DNS resolution may bring and master common optimization methods. Here, I have summarized several common DNS optimization methods.

- Cache the results of DNS resolution. Caching is the most efficient method, but be aware that once the cache expires, you still have to go to the DNS server to re-fetch new records. However, this is acceptable for most applications.
- Prefetching the results of DNS resolution. This is the most common method used in web applications such as browsers, which means that the browser automatically resolves the domain name in the background and caches the results without waiting for the user to click on a hyperlink on the page.
- Use HTTPDNS instead of regular DNS resolution. This is the method of choice for many mobile applications, especially since domain name hijacking is common today and can be avoided by using the HTTP protocol to bypass DNS servers in the link.
- DNS-based Global Load Balancing (GSLB). This not only provides load balancing and high availability for the service, but also returns the closest IP address based on the user's location.
