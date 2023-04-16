# 18 Cases: A memory leak, how should I locate and deal with it?

Through the previous sections on memory fundamentals, I'm sure you have an initial
understanding of how Linux memory works.

For ordinary processes, what can be seen is actually the virtual memory provided
by the kernel, which still needs to be mapped by the system to physical memory
through the page table.

When a process requests virtual memory via `malloc()`, the system does not
immediately allocate physical memory for it, but instead allocates memory in the
kernel only when it is first accessed via a missing page exception.

To reconcile the performance difference between CPU and disk, Linux also uses
Cache and Buffer to cache data read and written to file and disk, respectively,
into memory.

The allocation and reclamation of dynamic memory is a core and complex logical
module for applications. The process of managing memory is also prone to a variety
of "accidents", such as

- The allocated memory was not reclaimed correctly, resulting in a leak.
- Accessed an address outside the boundaries of allocated memory, causing the program to exit abnormally, etc.

Today I'll take you through how memory leaks actually happen, and how to troubleshoot
and locate them after they occur.

When it comes to memory leaks, it starts with memory allocation and reclamation.

## Memory allocation and reclamation

First of all, do you remember what methods are used to allocate memory in the
application? And how to release it back to the system when you are done with it?

As I mentioned earlier when talking about the process memory space, user space
memory consists of several different memory segments, such as read-only segments,
data segments, heap, stack, and file-mapped segments. These memory segments are
the basic way that applications use memory.

For example, if you define a local variable in your program, such as an array of
integers `int data[64]`, you define a segment of memory that can store 64 integers.
Since this is a local variable, it will allocate memory from the stack of the
memory space.

The stack memory is automatically allocated and managed by the system. Once the
program runs beyond the scope of this local variable, the stack memory will be
automatically reclaimed by the system, so no memory leak will happen.

Then again, many times we don't know the size of the data in advance, so you have
to use the standard library function `malloc() _,_` to dynamically allocate memory
in the program. This is when the system allocates memory from the heap of the
memory space.

Heap memory is allocated and managed by the application itself. Unless the program
exits, this heap memory is not automatically freed by the system, but requires
the application to explicitly call the library function `free()` to free it. If
the application does not free heap memory properly, it can cause a memory leak.

These are two examples of stacks and heaps, so can other memory segments also
cause memory leaks? After what we learned earlier, this is not a difficult question
to answer.

- Read-only segments, which include the program's code and constants, also do not generate memory leaks because they are read-only and do not go back to allocate new memory.
- The data segment, which includes global variables and static variables, are sized at the time of definition, so they also do not generate memory leaks.
- The last memory mapping segment, which includes dynamic link libraries and shared memory, where the shared memory is dynamically allocated and managed by the program. So, if the program forgets to reclaim it after allocation, it can lead to similar leakage problems as heap memory.

**Memory leaks are so harmful that not only can these forgotten freed memories
not be accessed by the application itself, but also the system cannot reallocate
them to other applications.** Memory leaks accumulate and can even run out of
system memory.

Although the system can eventually kill the process through the OOM (Out of Memory)
mechanism, the process may have triggered a cascade of reactions before the OOM,
leading to serious performance problems.

For example, other processes that need memory may not be able to allocate new memory;
insufficient memory will trigger the system's cache recycling and SWAP mechanism,
which will further lead to I/O performance problems, etc.

Memory leaks are so harmful, so how should we detect such problems? In particular,
if you have already found a memory leak, how to locate and deal with it.

Next, we'll use a case study of computing the Fibonacci sequence to see how to
locate and handle memory leaks.

Fibonacci series is a series of such numbers: 0, 1, 1, 2, 3, 5, 8..., that is,
except for the first two numbers are 0 and 1, all other numbers are obtained by
adding the first two numbers, expressed by the mathematical formula is
$F(n)=F(n-1)+F(n-2), (n>=2) , F(0)=0, F(1)=1$.

## Case

Today's case is based on Ubuntu 18.04, but of course, it is also applicable to
other Linux systems.

- Machine configuration: 2 CPU, 8GB RAM
- Pre-installed `sysstat`, Docker and `bcc` packages, e.g:

```shell
# install sysstat docker
sudo apt-get install -y sysstat docker.io

# Install bcc
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 4052245BD4284CDD
echo "deb https://repo.iovisor.org/apt/bionic bionic main" | sudo tee /etc/apt/sources.list.d/iovisor.list
sudo apt-get update
sudo apt-get install -y bcc-tools libbcc-examples linux-headers-$(uname -r)
```

We are already familiar with `sysstat` and Docker. `vmstat`, which is part of the
`sysstat` package, allows you to observe changes in memory, while Docker allows
you to run case programs.

The 🔗[bcc](https://github.com/iovisor/bcc) package, also described earlier, provides a series of Linux
performance analysis tools that are often used to dynamically track the behavior
of processes and the kernel. You don't need to dig deeper into how it works,
we'll get to that later in the study. All you need to remember here is that after
following the steps above, all the tools it provides are located in the
<ins>/usr/share/bcc/tools</ins> directory.

> Note: bcc-tools requires kernel version 4.1 or higher. If you are using CentOS7,
> or other systems with older kernel versions, you will need to manually 🔗[upgradethe kernel version before installing it](https://github.com/iovisor/bcc/issues/462).

Open a terminal, SSH login to the machine and install the above tools.

As in previous cases, all the commands below are run as root by default. If you
are logged in as a normal user, please run `sudo su root` to switch to root.

If there are any problems with the installation process, again, you are encouraged
to search for solutions yourself first, and if you can't solve them, you can ask
me questions in the comments section. If you have already installed it before,
you can ignore this point.

Once the installation is complete, run the case by executing the following
command:

```shell
$ docker run --name=app -itd feisky/app:mem-leak
```

After the case has run successfully, you need to enter the following command to
confirm that the case application has started properly. If everything is working
properly, you should see the following screen:

```shell
$ docker logs app
2th => 1
3th => 2
4th => 3
5th => 5
6th => 8
7th => 13
```

From the output, we can see that the case outputs a series of values of the
Fibonacci series. In fact, these values are output once every 1 second.

Knowing this, how should we check the memory situation and determine if there are
any leaks occurring? The first thing that comes to your mind is probably the `top`
tool. However, although `top` can observe the memory usage of the system and processes,
it is not suitable for today's case. For memory leaks, we should focus more on
the trend of memory usage.

So, as I mentioned at the beginning, today's recommendation is for another old
acquaintance, the `vmstat` utility.

Run the following `vmstat` and wait for a while to see how the memory changes.
If you forget the meaning of each indicator in `vmstat`, remember to review the
previous section, or run `man vmstat` to query it.

```shell
# 每隔3秒输出一组数据
$ vmstat 3
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
0  0      0 6601824  97620 1098784    0    0     0     0   62  322  0  0 100  0  0
0  0      0 6601700  97620 1098788    0    0     0     0   57  251  0  0 100  0  0
0  0      0 6601320  97620 1098788    0    0     0     3   52  306  0  0 100  0  0
0  0      0 6601452  97628 1098788    0    0     0    27   63  326  0  0 100  0  0
2  0      0 6601328  97628 1098788    0    0     0    44   52  299  0  0 100  0  0
0  0      0 6601080  97628 1098792    0    0     0     0   56  285  0  0 100  0  0 
```

As you can see from the output, the free column of memory is constantly changing
and trending downward, while the buffer and cache remain essentially the same.

The fact that the unused memory is decreasing while the buffer and cache remain
essentially the same indicates that the memory used in the system has been increasing.
However, this does not indicate a memory leak, because the memory needed by the
application to run may also increase. For example, if a program uses a dynamically
growing array to cache the results of a computation, the memory footprint will
naturally grow.

So how do you determine if it's a memory leak? Or in other words, is there an easy
way to identify the process that is letting the memory grow and locate where the
growth is being used?

Based on the above, you should have thought of using `top` or `ps` to observe the
memory usage of the processes, then find out the processes whose memory usage keeps
growing, and finally check the memory distribution of the processes by `pmap`.

But this approach doesn't work very well, because to determine the memory changes,
you also need to write a script to process the output of `top` or `ps`.

Here, I introduce a tool specifically designed to detect memory leaks, `memleak`.
`memleak` can track memory allocation and release requests from the system or a
given process, and then periodically output a summary of unfree memory and the
corresponding call stack (default 5 seconds).

Of course, `memleak` is one of the tools in the `bcc` package that we installed
at the beginning, and we can run it by executing <ins>/usr/share/bcc/tools/memleak</ins>.
For example, we run the following command:

```shell
# -a 表示显示每个内存分配请求的大小以及地址
# -p 指定案例应用的PID号
$ /usr/share/bcc/tools/memleak -a -p $(pidof app)
WARNING: Couldn't find .text section in /app
WARNING: BCC can't handle sym look ups for /app
    addr = 7f8f704732b0 size = 8192
    addr = 7f8f704772d0 size = 8192
    addr = 7f8f704712a0 size = 8192
    addr = 7f8f704752c0 size = 8192
    32768 bytes in 4 allocations from stack
        [unknown] [app]
        [unknown] [app]
        start_thread+0xdb [libpthread-2.27.so]
```

As you can see from the output of `memleak`, the case application is constantly
allocating memory, and these allocated addresses are not being reclaimed.

There is a problem here, 💬"Couldn't find .text section in /app", so the call stack
is not output properly, and only the [unknown] flag is seen in the last call stack
section.

Why does this error occur? Actually, it is caused by the fact that the case application
is running in a container. `memleak` tools run outside the container and do not
have direct access to the process path <ins>/app</ins>.

Let's say you run the `ls` command directly in the terminal, you will find that
the path does not exist:

```shell
$ ls /app
ls: cannot access '/app': No such file or directory
```

I have mentioned several ideas to solve similar problems in the 🔗[perf usage](https://time.geekbang.org/column/article/73738)
in the CPU module. The easiest way is to build files with the same path and
dependency libraries outside the container. In this case there is only one binary,
so just putting the binary of the case application in the <ins>/app</ins> path
will fix the problem.

For example, you can run the following command to copy the app binary out of the
container, and then re-run the memleak utility:

```shell
$ docker cp app:/app /app
$ /usr/share/bcc/tools/memleak -p $(pidof app) -a
Attaching to pid 12512, Ctrl+C to quit.
[03:00:41] Top 10 stacks with outstanding allocations:
    addr = 7f8f70863220 size = 8192
    addr = 7f8f70861210 size = 8192
    addr = 7f8f7085b1e0 size = 8192
    addr = 7f8f7085f200 size = 8192
    addr = 7f8f7085d1f0 size = 8192
    40960 bytes in 5 allocations from stack
        fibonacci+0x1f [app]
        child+0x4f [app]
        start_thread+0xdb [libpthread-2.27.so] 
```

This time, we finally see the call stack for the memory allocation, and it turns
out that the memory allocated by the fibonacci() function was not freed.

Having located the source of the memory leak, the next natural step is to look at
the source code and figure out how to fix it. Let's look at the source code of
the case application together 🔗[app.c](https://github.com/feiskyer/linux-perf-examples/blob/master/mem-leak/app.c):

```shell
$ docker exec app cat /app.c
...
long long *fibonacci(long long *n0, long long *n1)
{
    //分配1024个长整数空间方便观测内存的变化情况
    long long *v = (long long *) calloc(1024, sizeof(long long));
    *v = *n0 + *n1;
    return v;
}


void *child(void *arg)
{
    long long n0 = 0;
    long long n1 = 1;
    long long *v = NULL;
    for (int n = 2; n > 0; n++) {
        v = fibonacci(&n0, &n1);
        n0 = n1;
        n1 = *v;
        printf("%dth => %lld\n", n, *v);
        sleep(1);
    }
}
...
```

You will notice that `child()` calls the `fibonacci()` function, but does not free
the memory returned by `fibonacci()`. So, to fix the leak, just add a "free" function
to `child()`, e.g:

```shell
void *child(void *arg)
{
    ...
    for (int n = 2; n > 0; n++) {
        v = fibonacci(&n0, &n1);
        n0 = n1;
        n1 = *v;
        printf("%dth => %lld\n", n, *v);
        free(v);    // 释放内存
        sleep(1);
    }
}
```

I put the repaired code into 🔗[app-fix.c](https://github.com/feiskyer/linux-perf-examples/blob/master/mem-leak/app-fix.c) and also packaged it into a Docker image.
You can run the following command to verify that the memory leak is fixed:

```shell
# 清理原来的案例应用
$ docker rm -f app

# 运行修复后的应用
$ docker run --name=app -itd feisky/app:mem-leak-fix

# 重新执行 memleak工具检查内存泄漏情况
$ /usr/share/bcc/tools/memleak -a -p $(pidof app)
Attaching to pid 18808, Ctrl+C to quit.
[10:23:18] Top 10 stacks with outstanding allocations:
[10:23:23] Top 10 stacks with outstanding allocations:
```

Now, we see that the case application has no more legacy memory, proving that our
fix was successfully completed.

## Summary

To summarize today's content.

The user memory space accessible to the application consists of read-only segments,
data segments, heap, stack, and file-mapped segments. The heap memory and file-mapped
segments require the application to dynamically manage the memory segments, so we
must handle them carefully. Not only should you use the standard library function
`malloc()` to dynamically allocate memory, but you should also remember to call
the library function `free()` to free them when you are done with them.

Today's case is relatively simple, just add a `free()` call to fix the memory leak.
However, real applications are much more complex. For example,

- `malloc()` and `free()` do not usually occur in pairs, but require you, instead, to free memory on each exception handling path and success path.
- In a multi-threaded program, memory allocated in one thread may be accessed and freed in another thread.
- To complicate matters further, implicitly allocated memory in a third-party library function may need to be freed explicitly by the application.

So, in order to avoid memory leaks, the most important point is to develop good
programming habits, for example, after allocating memory, make sure to write the
memory release code first before developing other logic. Still, it is not difficult
to borrow and return in order to run efficiently and borrow again.

Of course, if you have already completed the development task, you can also use
the `memleak` tool to check for memory leaks while the application is running.
If a memory leak is found, you can then locate where the memory was allocated
based on the application call stack output from `memleak`, and thus release the
memory that is no longer accessible.