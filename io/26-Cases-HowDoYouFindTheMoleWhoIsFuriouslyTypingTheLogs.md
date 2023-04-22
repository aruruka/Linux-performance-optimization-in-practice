# 26 Cases: How do you find the "mole" who is furiously typing the logs?

In the first two sections, we learned about file systems and disk I/O principles, so let me take you through a review.

A file system, is a mechanism for organizing and managing the files on a storage device. To support all kinds of different file systems, Linux abstracts a layer of virtual file system, VFS, on top of the various file systems.

It defines a set of data structures and standard interfaces that are supported by all file systems. This way, applications and other subsystems in the kernel only need to interact with the uniform interface provided by VFS.

At the lower level of the file system, to support different types of storage devices, Linux abstracts a common block layer on top of the various storage devices.

The generic block layer, provides a standard interface for file systems and applications to access block devices; at the same time, it provides a unified framework for drivers for various block devices. In addition, the generic block layer queues I/O requests sent by file systems and applications and improves the efficiency of disk read and write by reordering and request merging.

The next layer of the common block layer is, naturally, the device layer, which includes drivers for various block devices as well as physical storage devices.

The file system, the common block layer, and the device layer make up the Linux storage I/O stack. The I/O of the storage system is usually the slowest part of the entire system. Therefore, Linux uses various caching mechanisms to optimize the efficiency of I/O, for example

- To optimize the performance of file access, various caching mechanisms such as page caching, index node caching, and directory item caching are used to reduce the direct calls to lower-level block devices.
- Similarly, to optimize the access efficiency of block devices, buffers are used to cache data from block devices.

However, when you encounter I/O problems with file systems and disks, how exactly should you locate and analyze them? Today, I'll take you through this situation with one of the most common cases of applications logging large amounts of logs.

## Case preparation

This case is also based on Ubuntu 18.04, which is also applicable to other Linux systems. The environment I used for the case is shown below:

- Machine configuration: 2 CPU, 8GB memory
- Pre-installed docker, sysstat and other tools, such as `apt install docker.io sysstat`

Here I would like to thank Yang Xiangyi, a senior operation and maintenance engineer at Vipshop, for helping to share today's case. This case, a small application developed in Python, I packaged it into a Docker image to make it easier to run. This way, you can start it by simply running the Docker command.

Next, open a terminal, SSH into the machine used in the case, and install the above tools. As before, all commands in the case are run by default as the root user. If you are logged in as a normal user, run the `sudo su root` command to switch to the root user.

At this point, you are done with the preparation. Next, let's get into the action.

> Note: The core logic of the Python application in this case is relatively simple, so you may be able to see the problem at a glance, but the source code in a real production environment is much more complex.
> So, I still recommend that you don't look at the source code before you operate to avoid preconceptions, but treat it as a black box to analyze.
> This way, you can get a better grasp of how to analyze the application where the bottleneck is located and the approximate location of the bottleneck in the application, starting from the system's resource usage problem.

## Case study

First, let's run today's target application by executing the following command in the terminal:

```shell
$ docker run -v /tmp:/tmp --name=app -itd feisky/logapp
```

Then, run the `ps` command in the terminal to make sure the case application starts properly. If this is done correctly, you should see an app.py process in the output of `ps`: `ps`:

```shell
$ ps -ef | grep /app.py 
root     18940 18921 73 14:41 pts/0    00:00:02 python /app.py 
```

Next, let's see if there are any performance issues with the system. What are the performance indicators to observe? In the previous article, we knew that system resources such as CPU, memory, and disk I/O are prone to resource bottlenecks, and that's where we'll look. Let's observe the usage of these resources.

Of course, before you do it you should think about which tools to use to do it, and what is the order of using the tools again. You can recall the previous cases and ideas, think about it yourself, and then continue with the following steps.

My idea is that we can first use `top` to see CPU and memory usage, and then use `iostat` to see disk I/O.

So, next, you can run the `top` command in a terminal to see how much CPU and memory is being used:

```shell
# 按1切换到每个CPU的使用情况 
$ top 
top - 14:43:43 up 1 day,  1:39,  2 users,  load average: 2.48, 1.09, 0.63 
Tasks: 130 total,   2 running,  74 sleeping,   0 stopped,   0 zombie 
%Cpu0  :  0.7 us,  6.0 sy,  0.0 ni,  0.7 id, 92.7 wa,  0.0 hi,  0.0 si,  0.0 st 
%Cpu1  :  0.0 us,  0.3 sy,  0.0 ni, 92.3 id,  7.3 wa,  0.0 hi,  0.0 si,  0.0 st 
KiB Mem :  8169308 total,   747684 free,   741336 used,  6680288 buff/cache 
KiB Swap:        0 total,        0 free,        0 used.  7113124 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND 
18940 root      20   0  656108 355740   5236 R   6.3  4.4   0:12.56 python 
1312 root      20   0  236532  24116   9648 S   0.3  0.3   9:29.80 python3
```

Looking at the output of `top`, you will see that CPU0 is very heavily used, with a system CPU usage (sys%) of 6% and an iowait of over 90%. This indicates that I/O-intensive processes may be running on CPU0. But what exactly is the cause? Let's keep this doubt for now and continue to finish reading.

Next, let's look at the CPU usage of the process part. You will see that the python process has reached a CPU usage of 6%, while the rest of the processes have a relatively low CPU usage of no more than 0.3%. It looks like python is a suspicious process. Write down the PID number 18940 of the python process, and we will analyze it later.

Finally, looking at the memory usage, the total memory is 8G and the remaining memory is only 730 MB, while Buffer/Cache occupies as much as 6GB of memory, which means that the memory is mainly occupied by cache. Although most of the cache can be recycled, we still need to understand where the cache goes and make sure that it is used reasonably.

At this point, you can basically tell that iowait is a potential bottleneck in CPU usage, and that the memory portion of the cache accounts for a relatively large amount, so what about disk I/O?

Let's stop the `top` command by pressing Ctrl+C in the terminal and run the `iostat` command to observe the I/O usage:

```shell
# -d表示显示I/O性能指标，-x表示显示扩展统计（即所有I/O指标） 
$ iostat -x -d 1 
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util 
loop0            0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sdb              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sda              0.00   64.00      0.00  32768.00     0.00     0.00   0.00   0.00    0.00 7270.44 1102.18     0.00   512.00  15.50  99.20
```

Do you remember what these performance metrics mean? First, recall them yourself, but if you can't, check the previous section, or use `man iostat` to look them up.

Looking at the last column of `iostat`, you will see that disk sda is already at 99% I/O usage and is probably approaching I/O saturation.

The ultra-slow response times and exceptionally long request queue lengths further validate the suspicion that I/O is saturated. At this point, the sda disk has hit a serious performance bottleneck.

At this point, it is understandable why the iowait seen earlier is as high as 90%, which is the result of the I/O bottleneck of disk sda. The next step is to analyze the root cause of the I/O performance bottleneck. How do we know which processes are involved in these I/O requests?

I don't know if you remember, but in the last section I mentioned that you can use `pidstat` or `iotop` to look at the I/O of a process. Here, I'll use `pidstat` to take a look.

Using `pidstat` with the -d parameter, you can display the I/O of each process. So, you can run the following command in the terminal to see:

```shell
$ pidstat -d 1 

15:08:35      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command 
15:08:36        0     18940      0.00  45816.00      0.00      96  python 

15:08:36      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command 
15:08:37        0       354      0.00      0.00      0.00     350  jbd2/sda1-8 
15:08:37        0     18940      0.00  46000.00      0.00      96  python 
15:08:37        0     20065      0.00      0.00      0.00    1503  kworker/u4:2
```

From the output of `pidstat`, you can see that only the python process is writing more, and is writing more than 45 MB of data per second, which is larger than the 32 MB result found by `iostat` above. Clearly, it is the python process that is causing the I/O bottleneck.

Look further down at the iodelay entry. Although only python is writing a lot of data, you should notice that two processes (kworker and jbd2) are actually delayed much more than the python process.

In this case, kworker is a kernel thread, and jbd2 is a kernel thread used to ensure data integrity in the ext4 file system. They are both kernel threads that guarantee the basic functionality of the file system, so let's not worry about the details for now, we just need to understand that the root cause of their latency is still the large amount of I/O.

The combined output of `pidstat` shows that the python process is the most suspicious. Next, let's analyze what the python process is actually writing.

First, take a look at the PID number of the python process, 18940. do you feel familiar with 18940? In fact, when we used `top` earlier, we recorded the process with the highest CPU usage, and that was it. However, although the highest usage rate in `top` is only 6%, which is not that high. So, the I/O problem is still the right direction to analyze.

Once we know the PID number of the process, how do we check the write situation?

In fact, as I said in the system call case, reading and writing files must be done through system calls. By looking at the system calls, you can see what files are being written by the process. Think of `strace`, the tool we use most often to analyze system calls.

Next, we run the `strace` command in the terminal and specify the PID number of the python process with -p 18940:

```shell
$ strace -p 18940 
strace: Process 18940 attached 
...
mmap(NULL, 314576896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f0f7aee9000 
mmap(NULL, 314576896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f0f682e8000 
write(3, "2018-12-05 15:23:01,709 - __main"..., 314572844 
) = 314572844 
munmap(0x7f0f682e8000, 314576896)       = 0 
write(3, "\n", 1)                       = 1 
munmap(0x7f0f7aee9000, 314576896)       = 0 
close(3)                                = 0 
stat("/tmp/logtest.txt.1", {st_mode=S_IFREG|0644, st_size=943718535, ...}) = 0 
```

From the `write()` system call, we can see that the process has written 300MB of data to the file with file descriptor number 3. It seems that it is the file we are looking for. However, the `write()` call only shows the descriptor number of the file, the file name and path are still unknown.

Looking at the later `stat()` call, you can see that it is getting the status of <ins>/tmp/logtest.txt.1</ins>. This "dot + number format" file is very common in log rollbacks. We can guess that this is the first log rollback file and that the path to the log file being written is <ins>/tmp/logtest.txt</ins>.

Of course, this is just a guess on our part, and naturally needs to be verified. Here, I'll introduce you to a new tool, `lsof`. It is specifically designed to view the list of files opened by a process, but the "files" here are not just ordinary files, but also directories, block devices, dynamic libraries, network sockets, and so on.

Next, let's run the following `lsof` command in the terminal to see which files are opened by process 18940:

```shell
$ lsof -p 18940 
COMMAND   PID USER   FD   TYPE DEVICE  SIZE/OFF    NODE NAME 
python  18940 root  cwd    DIR   0,50      4096 1549389 / 
python  18940 root  rtd    DIR   0,50      4096 1549389 / 
… 
python  18940 root    2u   CHR  136,0       0t0       3 /dev/pts/0 
python  18940 root    3w   REG    8,1 117944320     303 /tmp/logtest.txt 
```

这个输出界面中，有几列我简单介绍一下，FD 表示文件描述符号，TYPE 表示文件类型，NAME 表示文件路径。这也是我们需要关注的重点。

再看最后一行，这说明，这个进程打开了文件 <ins>/tmp/logtest.txt</ins>，并且它的文件描述符是 3 号，而 3 后面的 w ，表示以写的方式打开。

这跟刚才 `strace` 完我们猜测的结果一致，看来这就是问题的根源：进程 18940 以每次 300MB 的速度，在“疯狂”写日志，而日志文件的路径是 <ins>/tmp/logtest.txt</ins>。

既然找出了问题根源，接下来按照惯例，就该查看源代码，然后分析为什么这个进程会狂打日志了。

你可以运行 `docker cp` 命令，把案例应用的源代码拷贝出来，然后查看它的内容。（你也可以点击🔗[这里](https://github.com/feiskyer/linux-perf-examples/tree/master/logging-app)查看案例应用的源码）：

```python
#拷贝案例应用源代码到当前目录
$ docker cp app:/app.py . 

#查看案例应用的源代码
$ cat app.py 

logger = logging.getLogger(__name__) 
logger.setLevel(level=logging.INFO) 
rHandler = RotatingFileHandler("/tmp/logtest.txt", maxBytes=1024 * 1024 * 1024, backupCount=1) 
rHandler.setLevel(logging.INFO) 

def write_log(size): 
  '''Write logs to file''' 
  message = get_message(size) 
  while True: 
    logger.info(message) 
    time.sleep(0.1) 

if __name__ == '__main__': 
  msg_size = 300 * 1024 * 1024 
  write_log(msg_size)
```

分析这个源码，我们发现，它的日志路径是 <ins>/tmp/logtest.txt</ins>，默认记录 INFO 级别以上的所有日志，而且每次写日志的大小是 300MB。这跟我们上面的分析结果是一致的。

```python
def set_logging_info(signal_num, frame): 
  '''Set loging level to INFO when receives SIGUSR1''' 
  logger.setLevel(logging.INFO) 

def set_logging_warning(signal_num, frame): 
  '''Set loging level to WARNING when receives SIGUSR2''' 
  logger.setLevel(logging.WARNING) 

signal.signal(signal.SIGUSR1, set_logging_info) 
signal.signal(signal.SIGUSR2, set_logging_warning)
```

According to the logging call `logger.info(message)` in the source code, we know that its logging is at INFO level, which is its default level. So, as long as the default level is raised to WARNING level, the logging problem should be solved.

Next, let's check if the analysis we just did is correct. Run the following kill command in the terminal to send a SIGUSR2 signal to process 18940:

```shell
$ kill -SIGUSR2 18940 
```

Then, execute `top` and `iostat` and observe:

```shell
$ top 
... 
%Cpu(s):  0.3 us,  0.2 sy,  0.0 ni, 99.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
```

```shell
$ iostat -d -x 1
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util 
loop0            0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sdb              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sda              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00
```

Looking at the output of `top` and `iostat`, you will see that after a while, `iowait` will become 0 and the I/O usage of the sda disk will gradually decrease to 0.

By this point, we have not only located the application that was hitting the logging frenzy, but also solved the I/O performance bottleneck perfectly by turning up the logging level.

At the end of the case, of course, don't forget to run the following command to stop the case application:

```shell
$ docker rm -f app
```

## Summary

Logging is the most common and effective tool for understanding the inner workings of an application. Both operating systems and applications record a large number of operational logs to view the history afterwards. These logs are typically turned on at different levels, for example, development environments usually turn on debug-level logs, while online environments only log warnings and errors.

When troubleshooting application problems, we may need to temporarily turn on the application debug logs in the online environment. Sometimes, we accidentally forget to turn it back on afterwards. Failure to turn up the online logs to the warning level may lead to a series of performance problems such as CPU usage, disk I/O, and even affect other applications running on the same server in severe cases.

In the future, when you encounter such a "logging frenzy" scenario, you can use tools such as `iostat`, `strace`, `lsof`, etc. to locate the process that is hitting the logs frenetically, find the corresponding log file, and then adjust the log level through the application's interface to solve the problem.

If the application cannot dynamically adjust the log level, you may also need to modify the application's configuration and restart the application for the configuration to take effect.