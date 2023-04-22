# 32 Q&A(4): Difference and connection between blocking and non-blocking I/O, synchronous and asynchronous I/O

The column has been updated so far, and we have finished the third module of the four basic modules - File System and Disk I/O. I'm glad you haven't fallen behind and are still actively learning to think and practice, and enthusiastically leaving comments and discussions.

Today is the fourth installment of Performance Optimization. As usual, I have extracted some typical questions from the I/O module comments as today's Q&A, and will focus on replies. Again, for your convenience, they are not in strict article order.

For each question, I have included a screenshot of the question asked in the comments section. If you need to review the original content, you can scan the QR code at the bottom right of each question to view it.

## Question 1: The difference and connection between blocking and non-blocking I/O, synchronous and asynchronous I/O

In 🔗[How File Systems Work](https://time.geekbang.org/column/article/76876), I introduced the meaning of blocking and non-blocking I/O as well as synchronous and asynchronous I/O, so let's briefly review them here.

First, let's look at blocking and non-blocking I/O. I/O can be classified as blocking or non-blocking depending on whether the application is blocking itself or not.

- Blocking I/O means that if the application does not get a response after executing an I/O operation, it blocks the current thread and cannot perform other tasks.
- Non-blocking I/O means that the application does not block the current thread after performing I/O operations and can continue to perform other tasks.

File I/O can be divided into synchronous and asynchronous I/O depending on how the I/O response is notified.

- Synchronous I/O means that after receiving an I/O request, the system will not respond to the application immediately; when the processing is completed, the system will tell the application the I/O result by means of a system call.
- Asynchronous I/O means that after receiving an I/O request, the system will first tell the application that the I/O request has been received and then process it asynchronously; when the processing is complete, the system will tell the application the result by means of an event notification.

As you can see, blocking/non-blocking and synchronous/asynchronous are really two different perspectives on how I/O is classified. They also describe different objects, blocking/non-blocking for the I/O caller (i.e., the application) and synchronous/asynchronous for the I/O executor (i.e., the system).

Let me give you an example to explain further. For example, in a Linux I/O call, the

- The system call read is a synchronous read, so the read does not respond to the application until the disk data is available.
- In contrast, aio_read is an asynchronous read, where the system receives an AIO read request and returns it without waiting for processing, and then notifies the application of the exact read result asynchronously via a callback.

Again, in the interface of a network socket, the

- When sending data directly to a socket using `send()`, if the socket does not have the O_NONBLOCK flag set, then the `send()` operation will keep blocking and the current thread will not be able to do anything else.
- Of course, if you use `epoll`, the system will tell you the status of this socket, and that can be used in a non-blocking way. When this socket is not writable, you can go do other things, such as reading and writing other sockets.

## Question 2: "File System" post-lesson reflection

At the end of 🔗[File System Principles article](https://time.geekbang.org/column/article/76876), I left you with a question to ponder: Does executing the `find` command cause the system's cache to be elevated? And if so, what type of cache is raised?

The answer to this question from Baihua and coyang is quite accurate. By learning the principles of the Linux file system, we know that file names, and the directory relationships between files, are placed in the directory entry cache. And this is a memory-based data structure that is built dynamically as needed. So, when looking for files, Linux dynamically builds the structure of directory entries that are not in the cache, resulting in a higher dentry cache.

In fact, in addition to the increase in the directory entry cache, Buffer usage will also increase. If you look at it with `vmstat`, you will see that both Buffer and Cache are growing:

```shell
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  1      0 7563744   6024 225944    0    0  3736     0  574 3249  3  5 89  3  0
 1  0      0 7542792  14736 236856    0    0  8708     0 13494 32335  8 19 66  7  0
 0  1      0 7494452  27280 272284    0    0 12544     0 4550 17084  5 15 68 13  0
 0  1      0 7475084  42380 276320    0    0 15096     0 2541 14253  2  6 78 13  0
 0  1      0 7455728  57600 280436    0    0 15220     0 2025 14518  2  6 70 22  0
```

Here, the Buffer grows because the metadata needed to build the directory item cache (e.g., file names, index nodes, etc.), needs to be read from the file system.

## Question 3: "Disk I/O Latency" post-lesson reflection

At the end of 🔗[The Case of Disk I/O Latency](https://time.geekbang.org/column/article/78409), I left you with a thought-provoking question.

We used `iostat` to confirm that disk I/O has become a performance bottleneck, and we also used `pidstat` to find a large number of processes with disk I/O. But then, using `strace` to trace the process, we couldn't find any write system calls. Why is this?

Many of the comments from students have answered this question accurately. For example, both strace and jeff's messages point out that in this scenario, we need to add the -f option in order to track multi-process and multi-threaded system calls.

You see, just the wrong option can cause the performance tool to "make a mistake" and present this seemingly illogical result. It's great to see that so many of you have gotten to the heart of the idea of using performance tools - figuring out how the tools themselves work and what the problems are.

## Question 4: Post-lesson reflection on "MySQL Cases"

At the end of 🔗[MySQL Case](https://time.geekbang.org/column/article/78633), I left you with a thought-provoking question. Why is it that after the DataService application is stopped, MySQL queries are much faster and the disk I/O bottleneck disappears, even though there are still no indexes?

ninuxer's message basically explains the problem, but it's not perfect. In fact, when you see that DataService is modifying <ins>/proc/sys/vm/drop_caches</ins>, you should think about what you learned earlier about the role of Cache.

We know that the data tables accessed by the case application are based on the MyISAM engine, and one of the features of MyISAM is that it only caches indexes in memory and does not cache data. So, when the query statement cannot use indexes, it requires the data table to be read from the database file into memory and then processed.

So, if you use the `vmstat` tool and watch the trend of cache and I/O, you will see the following result:

```shell
$ vmstat 1

procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st

# 备注： DataService正在运行
0  1      0 7293416    132 366704    0    0 32516    12   36  546  1  3 49 48  0
 0  1      0 7260772    132 399256    0    0 32640     0   37  463  1  1 49 48  0
 0  1      0 7228088    132 432088    0    0 32640     0   30  477  0  1 49 49  0
 0  0      0 7306560    132 353084    0    0 20572     4   90  574  1  4 69 27  0
 0  2      0 7282300    132 368536    0    0 15468     0   32  304  0  0 79 20  0

# 备注：DataService从这里开始停止
 0  0      0 7241852   1360 424164    0    0   864   320  133 1266  1  1 94  5  0
 0  1      0 7228956   1368 437400    0    0 13328     0   45  366  0  0 83 17  0
 0  1      0 7196320   1368 470148    0    0 32640     0   33  413  1  1 50 49  0
...
 0  0      0 6747540   1368 918576    0    0 29056     0   42  568  0  0 56 44  0
 0  0      0 6747540   1368 918576    0    0     0     0   40  141  1  0 100  0  0
```

Before the DataService stops, the cache grows three times in a row and then drops back down, precisely because the DataService clears the page cache every 3 seconds. After the DataService stops, the cache keeps growing until it reaches 918576 and then stops changing.

At this point, the disk read (bi) decreases to 0, and at the same time, the iowait (wa) decreases to 0, which means that all the data is already in the system's cache at this point. We know that cache is a part of memory which is much faster to access than disk, and this explains why MySQL queries become much faster.

From this case, you will see that MySQL's MyISAM engine, itself, does not cache data and relies on the system cache to speed up disk I/O accesses. Once there are other applications running in the system at the same time, it is difficult for the MyISAM engine to take full advantage of the system cache. This is because the system cache can be taken up by other applications or even just cleaned up.

So, in general, I don't recommend that you base your application's performance optimization entirely on the system cache. It is better to allocate memory within the application and build a fully autonomous cache, such as MySQL's InnoDB engine, which caches both indexes and data, or use a third-party caching application such as Memcached, Redis, etc.

I'll keep answering these questions today, and you are welcome to continue writing your questions and thoughts in the comments section. I hope that with each Q&A, we can work together with you to internalize the knowledge of the article into your capabilities, and we will not only practice in the real world, but also progress in the communication.