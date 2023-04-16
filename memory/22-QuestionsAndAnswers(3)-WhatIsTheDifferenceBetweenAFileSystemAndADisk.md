# 22 Q&A(3): What is the difference between a file system and a disk?

The column has been updated so far, and we have finished learning the second of the four basic modules - the memory performance chapter. I'm glad you haven't fallen behind and are still actively learning and practicing the operations, and enthusiastically leaving comments and discussions.

I'm very happy to see that many of you have used the case ideas you learned to solve real-world performance problems. I am also very grateful to espzest, Big Beet, Smile and other active thinkers for pointing out some inappropriate or uncritical parts of the article. In addition, there are also I came to also, JohnT3e, Bai Hua and other students, actively in the message area to discuss the learning and practice of the problem, also shared valuable experience, here also thank you very much.

Today is the third installment of performance optimization. As usual, I have extracted some typical questions from the messages of the memory module as today's Q&A and focused on replying. To make it easier for you to learn and understand, they are not strictly in the order of the article.

For each question, I've included a screenshot of the question asked in the comments section. If you need to review the original content, you can scan the QR code at the bottom right of each question to view it.

## Problem 1: Memory Recovery and OOM

This problem, in fact, consists of four sub-problems, namely.

- How to understand LRU memory recovery?
- Where does the recycled memory go again?
- Is OOM scored according to virtual memory or actual memory?
- How to estimate the minimum memory for an application?

In fact, as I mentioned in 🔗[the Linux Memory Principles](https://time.geekbang.org/column/article/74272) and 🔗[Swap Principles](https://time.geekbang.org/column/article/75797), once memory is found to be tight, the system will reclaim memory in three ways. Let's review these three ways:

- Reclaiming the cache based on the LRU (Least Recently Used) algorithm;
- Recycle infrequently accessed anonymous pages based on the Swap mechanism;
- Based on the OOM (Out of Memory) mechanism, kill processes that take up a lot of memory.

The first two approaches, cache recovery and swap recovery, are actually based on the LRU algorithm, which prioritizes the recovery of infrequently accessed memory. the LRU recovery algorithm, in fact, maintains two bidirectional chains, active and inactive, where:

- active records active memory pages;
- inactive Records inactive memory pages.

The closer to the end of the chain table, the less frequently the memory page is accessed. In this way, when reclaiming memory, the system can prioritize the inactive memory based on the level of activity.

Active and inactive memory pages are further divided into file pages and anonymous pages according to their types, which correspond to cache recycling and Swap recycling, respectively.

Of course, you can look up their sizes from <ins>/proc/meminfo</ins>, e.g:

```shell
# grep表示只保留包含active的指标（忽略大小写）
# sort表示按照字母顺序排序
$ cat /proc/meminfo | grep -i active | sort
Active(anon):     167976 kB
Active(file):     971488 kB
Active:          1139464 kB
Inactive(anon):      720 kB
Inactive(file):  2109536 kB
Inactive:        2110256 kB
```


In the third way, the OOM mechanism sorts the processes by oom_score. the larger the oom_score, the more likely the process is to be killed by the system.

When the system finds that there is not enough memory to allocate new memory requests, it tries 🔗[direct memory reclamation](https://time.geekbang.org/column/article/75797). In this case, if there is enough memory after reclaiming file pages and anonymous pages, of course all is happy and the reclaimed memory is allocated to the process. But if there is still not enough memory, OOM will come into play.

When OOM happens, you can see the Out of memory information in dmesg to know which processes are killed by OOM. For example, you can execute the following command to query the OOM log:

```shell
$ dmesg | grep -i "Out of memory"
Out of memory: Kill process 9329 (java) score 321 or sacrifice child
```

Of course, if you don't want your application to be killed by OOM, you can adjust the process's oom_score_adj to reduce the OOM score and thus reduce the probability of being killed. Alternatively, you can enable memory overcommit, which allows processes to request more virtual memory than physical memory (this actually assumes that the process will not run out of requested virtual memory).

With these three approaches, we are done reviewing. Next, we return to the beginning of the four questions, I believe you already have the answer yourself.

1. The principle of the LRU algorithm has been mentioned earlier and will not be repeated here.
2. After memory is reclaimed, it is put back into unused memory. This way, new processes can request and use them.
3. The timing of the OOM trigger is based on virtual memory. In other words, when a process requests memory, if the sum of the requested virtual memory plus the actual memory already used by the server is larger than the total physical memory, OOM is triggered.
4. The easiest way to determine the minimum memory for a process or container is to let it run and then check its memory usage by ps or smap. Note, however, that when a process is first started, it may not have started processing the actual business yet, and once it starts processing the actual business, it will take up more memory. So, remember to leave some margin for memory.

## Problem 2: Difference between file system and disk

The principles of the file system and disk, which I will cover in the next module, are also very closely related to memory. However, when learning the principles of Buffer and Cache, I mentioned that Buffer is used for disk and Cache is used for file. Therefore, there are many students who are confused, such as these two questions in JJ's message.

- How do you distinguish between reading and writing a file and reading and writing a disk?
- Can I read and write to disk without going through the file system?

If you have the same question, you are still not clear about the difference between disk and file. I replied briefly in the comments section of the article "🔗[How to understand Buffer and Cache in memory](https://time.geekbang.org/column/article/74633)", but I'm afraid some students didn't see it, so I'll re-tell it here.

A disk is a storage device (a block device, to be exact) that can be divided into different disk partitions. And on a disk or disk partition, a file system can be created again and mounted to one of the system's directories. In this way, the system can read and write files through this mounted directory.

In other words, the disk is a block device for storing data and a carrier for the file system. So, the file system does still have to go through the disk, to ensure persistent storage of data.

You will see this phrase in many places, everything is a file in Linux. In other words, you can access disks and files through the same file interface (e.g. open, read, write, close, etc.).

- When we say "file", we mean a normal file.
- A disk or partition, on the other hand, is a block device file.

You can see the difference by running "ls -l < path >". If you don't understand the meaning of ls output, don't forget to man it. Run the man ls command, and the info '(coreutils) ls invocation' command, and you will find it.

When reading or writing a normal file, the I/O request goes through the file system first, and then the file system is responsible for interacting with the disk. When reading or writing block device files, the file system is skipped and the interaction with the disk is done directly, which is also known as "bare I/O".

These two types of reads and writes naturally use different caches. The cache managed by the file system is actually part of the Cache. Bare disk caching, on the other hand, uses a Buffer.

For more information about the file system, disk, and I/O principles, don't worry, we'll get to that later.

## Problem 3: How to count the physical memory usage of all processes

This is actually an after-school reflection question on 🔗[how to understand Buffer and Cache in memory](https://time.geekbang.org/column/article/74633), and a few students, such as John T3e, have given some ideas.

For example, the method of the nameless pawn is to add up the RSS of all processes:

```shell
for i in $( ls /proc/ |grep "^[0-9]"|awk '$0 >100') ;do cmd="";[ -f /proc/$i/cmdline ] && cmd=`cat /proc/$i/cmdline`;[ "$cmd"X = ""X ] && cmd=$i;awk -v i="$cmd" '/Rss:/{a=a+$2}END{printf("%s:%d\n",i,a)}' /proc/$i/smaps 2>/dev/null; done | sort -t: -k2nr | head -10
```

This approach, in fact, leads to a lot of double counting. RSS indicates resident memory and includes the shared memory used by the process. So, direct accumulation will lead to double counting of shared memory and not getting an accurate answer.

Several students in the message have similar problems with their answers. You can recheck your method and figure out the definition and rationale of each indicator to prevent double counting.

Of course, there are students who have very correct ideas, such as JohnT3e mentioned, the key point of this question is to understand the meaning of PSS.

> The key point of the final reflection question is to understand the meaning of the PSS terms. Interested students can refer to this source:
> https://unix.stackexchange.com/questions/33381/getting-information-about-a-process-memory-usage-from-proc-pid-smaps

You can of course find the answer via the link on stackexchange, but I would recommend, moreover, looking directly at the 🔗[documentation](https://www.kernel.org/doc/Documentation/filesystems/proc.txt) for the proc file system:

> The “proportional set size” (PSS) of a process is the count of pages it has in memory, where each page is divided by the number of processes sharing it.
> So if a process has 1000 pages all to itself, and 1000 shared with one other process, its PSS will be 1500.

Let me briefly explain here that the PSS of each process is the sum of the shared memory divided equally among the processes, plus the non-shared memory size of the process itself.

Like this example in the documentation, a process has 1000 pages of non-shared memory, and it shares 1000 pages with another process, so its PSS = 1000/2 + 1000 = 1500 pages.

This way, you can just accumulate the PSS and not worry about double counting the shared memory.

For example, you can run the following command to calculate:

```shell
# 使用grep查找Pss指标后，再用awk计算累加值
$ grep Pss /proc/[1-9]*/smaps | awk '{total+=$2}; END {printf "%d kB\n", total }'
391266 kB
```