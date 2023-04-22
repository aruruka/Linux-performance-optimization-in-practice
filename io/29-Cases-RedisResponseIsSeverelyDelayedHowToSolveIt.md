# 29 Cases: Redis response is severely delayed, how to solve it?

In the previous section, we analyzed a MySQL-based product search case together, so let's review it first. When accessing the product search interface, we found that the response of the interface was particularly slow. By analyzing the system CPU, memory and disk I/O usage, we found that there was a disk I/O bottleneck, and it was caused by the case application.

Then, with the help of `pidstat`, we found that the culprit was the mysqld process. We then used `strace` and `lsof` to find out which file mysqld was reading. Based on the names and paths of the files, we found out which database and data tables mysqld was operating on. Combining this information, we guessed that this was a slow query problem caused by not using indexes.

To verify our guess, we went to the MySQL command line terminal and used the database analysis tool to find that the fields accessed by the case application were indeed not indexed. Since the guess was correct, the problem was naturally solved by adding indexes.

As you can see from this case, MySQL's MyISAM engine relies heavily on the system cache to speed up disk I/O access. But if there are other applications running in the system at the same time, it is difficult for the MyISAM engine to take full advantage of the system cache. The cache may be taken up by other applications or even cleaned up.

So, in general, I don't recommend basing your application's performance optimization entirely on the system cache. It is better to allocate memory within the application and build a fully autonomous cache, or use a third-party caching application, such as Memcached, Redis, etc.

Redis is one of the most commonly used key-value storage systems, often used as a database, cache, and message queue proxy, etc. Redis stores data based on memory, however, in order to ensure that data is not lost in case of server exceptions, in many cases we have to configure persistence for it, and this can cause performance issues with disk I/O.

Today, I'm going to take you through a case study of using Redis as a cache. This is also a Python Flask-based application that provides an interface to query the cache, but the response time of the interface is long and does not meet the requirements of the online system.

Many thanks to Guoxing Dong, a senior backend engineer in Ctrip's system development department, for helping to provide today's case study.

## Case preparation

This case is also based on Ubuntu 18.04, which is also applicable to other Linux systems. The environment I used for the case is shown below:

- Machine configuration: 2 CPU, 8GB RAM
- Pre-installed docker, sysstat, git, make and other tools, such as `apt install docker.io sysstat`

Today's case consists of a Python application + Redis. The Python application is a Flask-based application that uses Redis to manage the application's cache and provides three HTTP interfaces to the outside world:

- <ins>/</ins>: returns hello redis;
- <ins>/init/</ins>: inserts the specified amount of cached data, the default is 5000 if no amount is specified;
- the cache key format is uuid:
- the cache value is one of good, bad, or normal
- <ins>/get_cache /</ins>: queries the cached data for the specified value and returns the processing time. Where the type_name parameter only supports good, bad and normal (i.e. finding a list of keys with the same value).

Since the application is quite large, I packaged them into two Docker images for your convenience and pushed them to 🔗[Github](https://github.com/feiskyer/linux-perf-examples/tree/master/redis-slow). This way you only need to run a few commands to get started.

Today's case requires two virtual machines, one of which is used as the target machine for the case study, running the Flask application, which has an IP address of 192.168.0.10, and the other as the client, requesting the cache query interface. I have drawn a diagram to represent their relationship.

<center><a href="https://ibb.co/w6RC6WT">
    <img src="https://i.ibb.co/j4TG4Md/29-case-diagram.png" alt="29-case-diagram" border="0">
</a></center>

Next, open two terminals, SSH into each of the two virtual machines, and install the above tools in the first virtual machine.

As before, all commands in this case are run as root by default. If you are logged in as a normal user, run the `sudo su root` command to switch to the root user.

Here, the preparation work is done. Next, let's get into the action.

## Case analysis

First, let's run the target application to be analyzed in this case by executing the following command in the first terminal. Under normal circumstances, you should see the following output:

```shell
# 注意下面的随机字符串是容器ID，每次运行均会不同，并且你不需要关注它
$ docker run --name=redis -itd -p 10000:80 feisky/redis-server
ec41cb9e4dd5cb7079e1d9f72b7cee7de67278dbd3bd0956b4c0846bff211803
$ docker run --name=app --network=container:redis -itd feisky/redis-app
2c54eb252d0552448320d9155a2618b799a1e71d7289ec7277a61e72a9de5fd0
```

Then, run the `docker ps` command again to confirm that both containers are running (Up):

```shell
$ docker ps
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                             NAMES
2c54eb252d05        feisky/redis-app      "python /app.py"         48 seconds ago      Up 47 seconds                                         app
ec41cb9e4dd5        feisky/redis-server   "docker-entrypoint.s…"   49 seconds ago      Up 48 seconds       6379/tcp, 0.0.0.0:10000->80/tcp   redis
```

Today's application is listening on port 10000, so you can access the three previously mentioned interfaces through 🔗<http://192.168.0.10:10000>.

For example, let's switch to the second terminal and use the `curl` utility to access the application home page. If you see the output of hello redis, the application started properly:

Next, go ahead and execute the following `curl` command in terminal two to call the <ins>/init</ins> interface of the application, initialize the Redis cache, and insert 5000 cached messages. This is a slow process, for example, it took me about 10 minutes on my machine. After waiting patiently for a while, you will see the following line of output:

```shell
# 案例插入5000条数据，在实践时可以根据磁盘的类型适当调整，比如使用SSD时可以调大，而HDD可以适当调小
$ curl http://192.168.0.10:10000/init/5000
{"elapsed_seconds":30.26814079284668,"keys_initialized":5000}
```

Go ahead and execute the next command to access the application's cache query interface. If everything works, you will see the following output:

```shell
$ curl http://192.168.0.10:10000/get_cache
{"count":1677,"data":["d97662fa-06ac-11e9-92c7-0242ac110002",...],"elapsed_seconds":10.545469760894775,"type":"good"}
```

We see that the interface call takes 10 seconds! Such a long response time obviously does not meet the actual application requirements.

What went wrong? We still have to use the performance tools and principles we learned earlier to find this bottleneck.

But don't worry, again to avoid the end of the client's request during the analysis, we have to put the `curl` command into a loop to execute it before doing the performance analysis. You can continue in Terminal 2 by executing the following command:

```shell
$ while true; do curl http://192.168.0.10:10000/get_cache; done
```

Next, go back to Terminal 1 and find the "cause" of the slow response of the interface.

In several recent cases, the response has been very slow, and in this case, we naturally first wonder if there is a bottleneck in the system resources. Therefore, it is not wrong to first observe the usage of CPU, memory, disk I/O, etc.

Let's run the `top` command in Terminal 1 to analyze the CPU usage of the system:

```shell
$ top
top - 12:46:18 up 11 days,  8:49,  1 user,  load average: 1.36, 1.36, 1.04
Tasks: 137 total,   1 running,  79 sleeping,   0 stopped,   0 zombie
%Cpu0  :  6.0 us,  2.7 sy,  0.0 ni,  5.7 id, 84.7 wa,  0.0 hi,  1.0 si,  0.0 st
%Cpu1  :  1.0 us,  3.0 sy,  0.0 ni, 94.7 id,  0.0 wa,  0.0 hi,  1.3 si,  0.0 st
KiB Mem :  8169300 total,  7342244 free,   432912 used,   394144 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7478748 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 9181 root      20   0  193004  27304   8716 S   8.6  0.3   0:07.15 python
 9085 systemd+  20   0   28352   9760   1860 D   5.0  0.1   0:04.34 redis-server
  368 root      20   0       0      0      0 D   1.0  0.0   0:33.88 jbd2/sda1-8
  149 root       0 -20       0      0      0 I   0.3  0.0   0:10.63 kworker/0:1H
 1549 root      20   0  236716  24576   9864 S   0.3  0.3  91:37.30 python3
```

Looking at the output of top, we can see that the iowait of CPU0 is relatively high, reaching 84%; while the CPU usage of each process is not too high, the highest being python and redis-server, which are only 8% and 5% respectively. If we look at the memory, we see 8GB of total memory and 7GB of remaining memory, so it's clear that there are no memory problems.

So, the next step is to continue analyzing whether it is an I/O problem.

In the first terminal, first press Ctrl+C to stop the `top` command; then, execute the following `iostat` command to see if there are any I/O performance problems:

```shell
$ iostat -d -x 1
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
...
sda              0.00  492.00      0.00   2672.00     0.00   176.00   0.00  26.35    0.00    1.76   0.00     0.00     5.43   0.00   0.00
```

Looking at the output of `iostat`, we see that disk sda is writing 2.5MB per second (wkB/s) and the I/O usage (%util) is 0. It seems that although there are some I/O operations, they are not causing an I/O bottleneck on the disk.

After a round of troubleshooting, the CPU and memory usage are fine, and there are no bottlenecks in I/O, so there seems to be no direction for analysis?

When you encounter this situation, as always, reflect on whether you have missed any useful clues again. You can start by thinking on your own, working in three directions: the object of analysis (the case application), the system principles, and the performance tools, recalling their characteristics, looking for anomalies in the phenomenon, and then moving on to the next step.

Recall that today's case problem is slow querying data from the Redis cache. For queries, the corresponding I/O should be a read operation on disk, but what we just saw with `iostat` was a write operation. While I/O itself is not a performance bottleneck, the disk writes here are rather odd. Why would there be disk writes? Then we need to know exactly which process is writing to disk.

To know which processes the I/O requests are coming from, it's still up to our old friend `pidstat`. Run the following `pidstat` command in Terminal 1 to observe the I/O of the processes:

```shell
$ pidstat -d 1
12:49:35      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
12:49:36        0       368      0.00     16.00      0.00      86  jbd2/sda1-8
12:49:36      100      9085      0.00    636.00      0.00       1  redis-server
```

From the pidstat output, we see that the process with the most I/O is redis-server with PID 9085, and it also happens to be writing to disk. This means that it is indeed redis-server that is doing the disk writes.

Of course, it's not enough to find the processes that read and write to disk, we need to use the `strace`+`lsof` combination to see what redis-server is actually writing.

Next, again in Terminal 1, execute the `strace` command and specify redis-server process number 9085:

```shell
# -f表示跟踪子进程和子线程，-T表示显示系统调用的时长，-tt表示显示跟踪时间
$ strace -f -T -tt -p 9085
[pid  9085] 14:20:16.826131 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 65, NULL, 8) = 1 <0.000055>
[pid  9085] 14:20:16.826301 read(8, "*2\r\n$3\r\nGET\r\n$41\r\nuuid:5b2e76cc-"..., 16384) = 61 <0.000071>
[pid  9085] 14:20:16.826477 read(3, 0x7fff366a5747, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000063>
[pid  9085] 14:20:16.826645 write(8, "$3\r\nbad\r\n", 9) = 9 <0.000173>
[pid  9085] 14:20:16.826907 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 65, NULL, 8) = 1 <0.000032>
[pid  9085] 14:20:16.827030 read(8, "*2\r\n$3\r\nGET\r\n$41\r\nuuid:55862ada-"..., 16384) = 61 <0.000044>
[pid  9085] 14:20:16.827149 read(3, 0x7fff366a5747, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000043>
[pid  9085] 14:20:16.827285 write(8, "$3\r\nbad\r\n", 9) = 9 <0.000141>
[pid  9085] 14:20:16.827514 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 64, NULL, 8) = 1 <0.000049>
[pid  9085] 14:20:16.827641 read(8, "*2\r\n$3\r\nGET\r\n$41\r\nuuid:53522908-"..., 16384) = 61 <0.000043>
[pid  9085] 14:20:16.827784 read(3, 0x7fff366a5747, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000034>
[pid  9085] 14:20:16.827945 write(8, "$4\r\ngood\r\n", 10) = 10 <0.000288>
[pid  9085] 14:20:16.828339 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 63, NULL, 8) = 1 <0.000057>
[pid  9085] 14:20:16.828486 read(8, "*3\r\n$4\r\nSADD\r\n$4\r\ngood\r\n$36\r\n535"..., 16384) = 67 <0.000040>
[pid  9085] 14:20:16.828623 read(3, 0x7fff366a5747, 1) = -1 EAGAIN (Resource temporarily unavailable) <0.000052>
[pid  9085] 14:20:16.828760 write(7, "*3\r\n$4\r\nSADD\r\n$4\r\ngood\r\n$36\r\n535"..., 67) = 67 <0.000060>
[pid  9085] 14:20:16.828970 fdatasync(7) = 0 <0.005415>
[pid  9085] 14:20:16.834493 write(8, ":1\r\n", 4) = 4 <0.000250>
```
    
Observe for a while, do you see anything interesting?

In fact, from the system calls, `epoll_pwait`, `read`, `write`, and `fdatasync` are all relatively frequent. So, the write to disk that we just observed should be caused by `write` or `fdatasync`.

Next, let's run the `lsof` command to find out who these system calls operate on:

```shell
$ lsof -p 9085
redis-ser 9085 systemd-network    3r     FIFO   0,12      0t0 15447970 pipe
redis-ser 9085 systemd-network    4w     FIFO   0,12      0t0 15447970 pipe
redis-ser 9085 systemd-network    5u  a_inode   0,13        0    10179 [eventpoll]
redis-ser 9085 systemd-network    6u     sock    0,9      0t0 15447972 protocol: TCP
redis-ser 9085 systemd-network    7w      REG    8,1  8830146  2838532 /data/appendonly.aof
redis-ser 9085 systemd-network    8u     sock    0,9      0t0 15448709 protocol: TCP
```

Now you will see that descriptor number 3 is a pipe pipe, number 5 is an eventpoll, number 7 is a normal file, and number 8 is a TCP socket.

Combined with the phenomenon of disk writes, we know that only the common file #7 generates a disk write, and the file path it operates on is <ins>/data/appendonly.aof</ins>, and the corresponding system calls include `write` and `fdatasync`.

If you are familiar with the Redis persistence configuration, you should be able to see this file path and the fdatasync system call, which corresponds to the appendonly and appendfsync options in the Redis persistence configuration. It is likely that they are not configured properly, resulting in more disk writes.

To test this suspicion, we can query the configuration of these two options through the Redis command line tool.

Go ahead and run the following command in Terminal 1 to query the configuration of appendonly and appendfsync:

```shell
$ docker exec -it redis redis-cli config get 'append*'
1) "appendfsync"
2) "always"
3) "appendonly"
4) "yes"
```

As you can see from the results, appendfsync is always configured, and appendonly is yes. The details of these two options can be found in the 🔗[Redis Persistence](https://redis.io/topics/persistence), which I'll briefly describe here.

Redis provides two ways of persisting data, snapshots and appending files.

The **snapshot method** generates a snapshot of the data at a specified interval and saves it to a disk file. To avoid blocking the main process, Redis also forks a sub-process to take care of the snapshot saving. This approach is much better than appending files, both for backup and recovery.

However, it also has obvious drawbacks. When the data volume is large, the fork subprocess requires a large amount of memory, and saving the data is time consuming. So you need to set a longer time interval to cope with it, for example at least 5 minutes. That way, if a failure occurs, all you lose is a few minutes of data.

**Append File**, on the other hand, uses appending records at the end of the file to persist the data written by Redis, in order, so it is also more secure in its persistence.

It also provides a policy for setting fsync with the appendfsync option to ensure that writes are dropped to disk, including always, everysec, no, etc.

- everysec means that fsync is invoked once every second, which ensures that even in the worst case, only 1 second of data is lost; - and no means that it is left to the operating system.

Recall that the configuration we just saw, appendfsync is configured to always, which means that fsync will be called once every time data is written, causing a relatively large disk I/O strain.

Of course, you can also use `strace` and watch the execution of this system call. For example, after specifying fdatasync with the -e option, you will get the following result:

```shell
$ strace -f -p 9085 -T -tt -e fdatasync
strace: Process 9085 attached with 4 threads
[pid  9085] 14:22:52.013547 fdatasync(7) = 0 <0.007112>
[pid  9085] 14:22:52.022467 fdatasync(7) = 0 <0.008572>
[pid  9085] 14:22:52.032223 fdatasync(7) = 0 <0.006769>
...
[pid  9085] 14:22:52.139629 fdatasync(7) = 0 <0.008183>
```

As you can see here, there is an fdatasync call every 10ms or so, and each call itself consumes 7~8ms. Either way, it verifies our suspicion that the configuration does not make sense. Thus, we have identified the file that Redis is writing to, and we know the reason for the large amount of I/O generated. But, back to the original question, why are there disk writes when querying? Shouldn't there be only data reads? This requires us to review the results of `strace -f -T -tt -p 9085` again.

```c
read(8, "*2\r\n$3\r\nGET\r\n$41\r\nuuid:53522908-"..., 16384)
write(8, "$4\r\ngood\r\n", 10)
read(8, "*3\r\n$4\r\nSADD\r\n$4\r\ngood\r\n$36\r\n535"..., 16384)
write(7, "*3\r\n$4\r\nSADD\r\n$4\r\ngood\r\n$36\r\n535"..., 67)
write(8, ":1\r\n", 4)
```

If you're careful, you'll recall that according to `lsof`, file descriptor number 7 is a normal file <ins>/data/appendonly.aof</ins>, while number 8 is a TCP socket, and looking at the above, number 8 corresponds to a TCP read/write in a standard "request-response" format. response" format, i.e:

- Read GET uuid:53522908-... from the socket and respond good;
- Read SADD good 535... from the socket and respond 1.

For Redis, SADD is a write operation, so Redis also saves it to the <ins>appendonly.aof</ins> file for persistence.

Looking at more `strace` results, you see that every time GET returns good, there is a subsequent SADD operation, which results in a lot of on-disk writes to Redis for what is clearly a query interface.

At this point, we've identified the reason Redis writes to disk. However, before we draw any final conclusions, we need to make sure that the Redis client corresponding to TCP socket number 8 is our case application or not.

We can add the -i option to the `lsof` command to find out the TCP connection information corresponding to the TCP socket. However, since both Redis and Python applications run in containers, we need to get inside the container's network namespace to see the full TCP connections.

> Note: The `nsenter` tool, used in the following command, allows access to the container namespace. If your system does not have it installed, run the following command to install nsenter:
> `docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter`

Still in Terminal 1, run the following command:

```shell
# 由于这两个容器共享同一个网络命名空间，所以我们只需要进入app的网络命名空间即可
$ PID=$(docker inspect --format {{.State.Pid}} app)
# -i表示显示网络套接字信息
$ nsenter --target $PID --net -- lsof -i
COMMAND    PID            USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
redis-ser 9085 systemd-network    6u  IPv4 15447972      0t0  TCP localhost:6379 (LISTEN)
redis-ser 9085 systemd-network    8u  IPv4 15448709      0t0  TCP localhost:6379->localhost:32996 (ESTABLISHED)
python    9181            root    3u  IPv4 15448677      0t0  TCP *:http (LISTEN)
python    9181            root    5u  IPv4 15449632      0t0  TCP localhost:32996->localhost:6379 (ESTABLISHED)
```

This time we can see that the file descriptor 8 of redis-server corresponds to the TCP connection localhost:6379->localhost:32996, where localhost:6379 is redis-server's own listening port, and naturally localhost:32996 is the localhost:32996 is the client of redis. Looking at the last line, localhost:32996 corresponds to our Python application (process number 9181).

After all the twists and turns, we finally figured out the potential cause of Redis response latency. To summarize, we found two problems.

The first problem is that Redis is configured to always appendfsync, which causes every write to Redis to trigger the fdatasync system call. In today's case, there is no need to use such a high frequency of synchronous writes; using the default 1s interval is sufficient.

The second problem is that the Python application calls the Redis SADD command in the query interface, which is probably caused by unreasonable use of the cache.

For the first configuration problem, we can execute the following command to change appendfsync to everysec:

```shell
$ docker exec -it redis redis-cli config set appendfsync everysec
OK
```

After the change, switch to Terminal 2 and you will see that the request time has now been reduced to 0.9s:

```json
{..., "elapsed_seconds":0.9368953704833984,"type":"good"}
```

And for the second question, you have to check the source code of the application. By clicking on 🔗[Github](https://github.com/feiskyer/linux-perf-examples/blob/master/redis-slow/app.py) , you can view the source code of the case application at

```python
def get_cache(type_name):
    '''handler for /get_cache'''
    for key in redis_client.scan_iter("uuid:*"):
        value = redis_client.get(key)
        if value == type_name:
            redis_client.sadd(type_name, key[5:])
    data = list(redis_client.smembers(type_name))
    redis_client.delete(type_name)
    return jsonify({"type": type_name, 'count': len(data), 'data': data})
```

Sure enough, the Python application uses Redis as a temporary space to store the data found during the query. But we know that this data is fine in memory, and there's absolutely no need to store it in Redis via network calls.

With this in mind, I've pushed the modified code into the same source file, which you can access through the 🔗<http://192.168.0.10:10000/get_cache_data> interface.

Let's switch to Terminal 2 and press Ctrl+C to stop the previous `curl` command; then execute the following `curl` command to call the 🔗<http://192.168.0.10:10000/get_cache_data> new interface:

```shell
$ while true; do curl http://192.168.0.10:10000/get_cache_data; done
{...,"elapsed_seconds":0.16034674644470215,"type":"good"}
```

You can find that after solving the second problem, the performance of the new interface is further improved, from 0.9s just now, to less than 0.2s again.

Of course, at the end of the case, don't forget to clean up the case application. You can do this by switching to Terminal 1 and executing the following command:

```shell
$ docker rm -f app redis
```

## Summary

Today I took you through a case study of Redis caching.

We first used `top` and `iostat` to analyze the CPU, memory and disk usage of the system, but found that there is no bottleneck in the system resources. At this point, if you want to analyze further, which direction should you start from?

As you will see in today's case, in order to analyze further, you need to have a certain understanding of how the system and the application work.

For example, in today's case, although there is no bottleneck in disk I/O, the principle of Redis is that there should not be a large number of disk I/O writes when querying the cache.

Following this idea, we continued to use a series of tools such as `pidstat`, `strace`, `lsof`, `nsenter`, and so on, to identify two potential problems, one is the unreasonable configuration of Redis, and the other is the abuse of Redis by Python applications. Once the bottlenecks were found, the corresponding optimization work was naturally easier.
