# 07 Cases: What should I do if there are a large number of uninterruptible processes and zombie processes in the system? Part-1

In the last section, I used a case of Nginx + PHP to tell you about the analysis
and coping methods of high server CPU usage. Here you must remember that when
you encounter unexplained CPU usage problems, you must first check whether it is
a short-term application that is doing the trick.

The running time of short-term applications is relatively short, and it is
difficult to find them in tools such as `top` or `ps` that display system
profiles and process snapshots. You need to use tools that record events for
diagnosis, such as `execsnoop` or `perf top`.

You don't need to memorize these ideas deliberately, you can use them flexibly
if you practice a few times and think more in operation.

Also, we covered the types of CPU usage. In addition to the user CPU mentioned
in the previous section, it also includes the system CPU (such as context switching),
the CPU waiting for I/O (such as waiting for the response of the disk), and the
interrupt CPU (including soft interrupts and hard interrupts).

We have already analyzed the problem of high system CPU usage in the context
switching article, and the remaining CPU usage waiting for I/O (hereinafter
referred to as iowait) increases, which is also the most common server
performance problem. Today we will look at a case of multi-process I/O and analyze
this situation.

## Process status

When iowait is high, the process is likely to be in an uninterruptible state for
a long time because it cannot get a response from the hardware. From the output
of the `ps` or `top` command, you can find that they are all in the D state,
which is the uninterruptible state (Uninterruptible Sleep). Now that the state
of the process is mentioned, do you remember what states the process has?
Let's review first.

`top` and `ps` are the most commonly used tools to view process status, we will
start with the output of `top`. The following is an example of the output of the
`top` command, the S column (that is, the Status column) indicates the status of
the process. From this example, you can see several states such as R, D, Z, S, I,
etc. What do they mean?

```bash
$ top
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
28961 root      20   0   43816   3148   4040 R   3.2  0.0   0:00.01 top
  620 root      20   0   37280  33676    908 D   0.3  0.4   0:00.01 app
    1 root      20   0  160072   9416   6752 S   0.0  0.1   0:37.64 systemd
 1896 root      20   0       0      0      0 Z   0.0  0.0   0:00.00 devapp
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.10 kthreadd
    4 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 kworker/0:0H
    6 root       0 -20       0      0      0 I   0.0  0.0   0:00.00 mm_percpu_wq
    7 root      20   0       0      0      0 S   0.0  0.0   0:06.37 ksoftirqd/0
```

Let's take a look at them one by one:

- R is the abbreviation of Running or Runnable, indicating that the process is in the CPU's ready queue, running or waiting to run.
- D is the abbreviation of Disk Sleep, that is, Uninterruptible Sleep, which generally means that the process is interacting with the hardware, and the interaction process is not allowed to be interrupted by other processes or interrupts.
- Z is an abbreviation for Zombie, if you played the game "Plants vs Zombies", you should know what it means. It means a zombie process, that is, the process has actually ended, but the parent process has not reclaimed its resources (such as the process descriptor, PID, etc.).
- S is the abbreviation of Interruptible Sleep, which means that the process is suspended by the system because it is waiting for an event. When the event the process is waiting for occurs, it wakes up and enters the R state.
- I is the abbreviation of Idle, used on kernel threads that cannot interrupt sleep. As mentioned earlier, the uninterruptible process caused by hardware interaction is represented by D, but for some kernel threads, they may not actually have any load, and Idle is used to distinguish this situation. Note that processes in the D state will increase the load average, while processes in the I state will not.

Of course, the above example does not include all states of the process.
In addition to the above 5 states, the process also includes the following 2 states.

The first one is **T or t**, which is the abbreviation of Stopped or Traced,
indicating that the process is in a suspended or traced state.

Send a SIGSTOP signal to a process, and it will become suspended (Stopped) in
response to this signal; then send a SIGCONT signal to it, and the process will
resume running (if the process is started directly in the terminal, you need to
use the `fg` command , to resume running in the foreground).

And when you use a debugger (such as `gdb`) to debug a process, after using a
breakpoint to interrupt the process, the process will become a tracking state,
which is actually a special pause state, but you can use the debugger to track
and control the operation of the process as needed.

The other is `X`, short for Dead, which means the process is dead, so you won't
see it in `top` or `ps` commands.

With this in mind, let's go back to today's topic. First look at the
uninterruptible state, which is actually to ensure that the process data is
consistent with the hardware state, and under normal circumstances, the
uninterruptible state will end in a short time. Therefore, we can generally
ignore the short-term uninterruptible state process.

But if the system or hardware fails, the process may remain in the
uninterruptible state for a long time, and even cause a large number of
uninterruptible processes in the system. At this time, you have to pay attention
to whether the system has performance problems such as I/O.

Looking at zombie processes again, this is a problem that multi-process
applications can easily encounter. Under normal circumstances, when a process
creates a child process, it should wait for the end of the child process through
the system call `wait()` or `waitpid()`, and reclaim the resources of the child
process; when the child process ends, it will send to its parent process SIGCHLD
signal, therefore, the parent process can also register the callback processing
function of the SIGCHLD signal to reclaim resources asynchronously.

If the parent process does not do this, or the child process executes too fast,
the child process has already exited early before the parent process has time to
process the status of the child process, then the child process will become a
zombie process at this time. In other words, a father should always be responsible
for his son, start well and end well. If he does nothing or fails to keep up,
it will lead to the emergence of "problem boys".

Usually, the duration of the zombie process is relatively short, and it will die
after the parent process reclaims its resources; or after the parent process exits,
it will also die after being reclaimed by the init process.

Once the parent process does not handle the termination of the child process and
keeps running, the child process will remain in the zombie state. A large number
of zombie processes will run out of PID process numbers, resulting in the inability
to create new processes, so this situation must be avoided.
## Case analysis

Next, I will use a multi-process application case to take you to analyze the
problems of a large number of uninterruptible state and zombie state processes.
This application is developed based on C. Since its compiling and running steps
are cumbersome, I packaged it into a 🔗[Docker image](https://github.com/feiskyer/linux-perf-examples/tree/master/high-iowait-process). This way, you only
need to run a Docker container to get a simulated environment.

### Your preparation

The following cases are still based on Ubuntu 18.04 and are also applicable to
other Linux systems. The case environment I'm using looks like this:

- Machine configuration: 2 CPU, 8GB memory
- Pre-install docker, sysstat, dstat and other tools, such as `apt install docker.io dstat sysstat`

Here, `dstat` is a new performance tool, which absorbs the advantages of several
tools such as `vmstat`, `iostat`, and `ifstat`, and can observe the CPU, disk I/O,
network, and memory usage of the system at the same time.

Next, we open a terminal, SSH into the machine, and install the aforementioned tools.

Note that all the following commands are run as the root user by default.
If you log in to the system as a normal user, please run the `sudo su root`
command to switch to the root user.

If there is a problem in the installation process, you can search online to solve
it first. If you can’t solve it, remember to ask me in the message area.

> Tips: The core code logic of the case application is relatively simple, and
> you may be able to see the problem at a glance, but the source code in the
> actual production environment is much more complicated. Therefore, I still
> suggest that you don’t look at the source code before operating, to avoid
> preconceived ideas, but to analyze it as a black box, so that you can better
> analyze the problem based on the phenomenon. You treat it as a rehearsal in
> your work for the time being, so the effect is better.

### Operation and analysis

After the installation is complete, we first execute the following command to
run the case application:

```shell
$ docker run --privileged --name=app -itd feisky/app:iowait-new2
```

Then, enter the `ps` command to confirm that the sample application started normally.
If everything worked, you should see output that looks like this:

```shell
$ ps aux | grep /app
root      4009  0.0  0.0   4376  1008 pts/0    Ss+  05:51   0:00 /app
root      4287  0.6  0.4  37280 33660 pts/0    D+   05:54   0:00 /app
root      4288  0.6  0.4  37280 33668 pts/0    D+   05:54   0:00 /app
```

From this interface, we can find that multiple app processes have been started,
and their statuses are Ss+ and D+ respectively. Among them, S represents the
interruptible sleep state, and D represents the uninterruptible sleep state.
We just learned it before, so what do the s and + behind it mean? It doesn't
matter if you don't know, just check `man ps`. Now remember that the s means
that this process is the leader of a session, and the + means the foreground
process group.

There are two new concepts here, `process group` and `session`. They are used to
manage a group of interrelated processes, and the meaning is actually easy to understand.

- A process group represents a group of interrelated processes, for example, each child process is a member of the parent process's group;
- Whereas a session is a group of one or more processes sharing the same controlling terminal.

For example, when we log in to the server through SSH, a control terminal (TTY)
will be opened, and this control terminal corresponds to a session. The commands
we run in the terminal and their sub-processes form process groups one by one.
Among them, the commands running in the background form the background process
group; the commands running in the foreground form the foreground process group.

Knowing this, let's use `top` to look at the resource usage of the system:

```shell
# 按下数字 1 切换到所有 CPU 的使用情况，观察一会儿按 Ctrl+C 结束
$ top
top - 05:56:23 up 17 days, 16:45,  2 users,  load average: 2.00, 1.68, 1.39
Tasks: 247 total,   1 running,  79 sleeping,   0 stopped, 115 zombie
%Cpu0  :  0.0 us,  0.7 sy,  0.0 ni, 38.9 id, 60.5 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  0.7 sy,  0.0 ni,  4.7 id, 94.6 wa,  0.0 hi,  0.0 si,  0.0 st
...

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 4340 root      20   0   44676   4048   3432 R   0.3  0.0   0:00.05 top
 4345 root      20   0   37280  33624    860 D   0.3  0.0   0:00.01 app
 4344 root      20   0   37280  33624    860 D   0.3  0.4   0:00.01 app
    1 root      20   0  160072   9416   6752 S   0.0  0.1   0:38.59 systemd
...
```

Can you see what's the problem here? Be careful, observe line by line, don't miss
any place. If you forget the meaning of which line of parameters, you should go
back in time to review.

Ok, if you already have the answer, go ahead and see if the problem I am looking
for is the same. Here, I found four suspicious places.

- First look at the load average in the first line. The average load in the past 1 minute, 5 minutes, and 15 minutes has been decreasing in turn, indicating that the average load is increasing; and the average load in 1 minute has reached the CPU of the system. The number indicates that the system is likely to have a performance bottleneck.
- Looking at the Tasks in the second line, there is one running process, but there are many zombie processes, and the number keeps increasing, indicating that some child processes were not cleaned up when they exited.
- Next, look at the usage of the two CPUs. The user CPU and system CPU are not high, but iowait is 60.5% and 94.6% respectively, which seems a bit abnormal.
- Finally, look at the situation of each process. The process with the highest CPU usage is only 0.3%, which does not seem to be high; but there are two processes in the D state. They may be waiting for I/O, but this alone does not determine whether it is they cause iowait to rise.

If we sum up these four questions again, we can get two very clear points:

- The first point is that iowait is too high, causing the average load of the system to increase, even reaching the number of CPUs in the system.
- The second point is that the number of zombie processes is increasing, indicating that some programs failed to properly clean up the resources of the child processes.

So, what should you do if you encounter these two problems? Combined with our
previous analysis of the problem, you first think about it yourself, try it
yourself, and I will continue to "decompose" in the next class.

## Summary

Today we are mainly familiar with several necessary process states through
simple operations. Use our most familiar `ps` or `top` to view the status of the
process, which includes running (R), idle (I), uninterruptible sleep (D),
interruptible sleep (S), zombie (Z) and pause (T), etc.

Among them, the uninterruptible state and the zombie state are the focus of our
study today.

- The non-interruptible state means that the process is interacting with the hardware. In order to protect the consistency of process data and hardware, the system does not allow other processes or interrupts to interrupt the process. A process that is uninterruptible for an extended period of time usually indicates that the system has I/O performance problems.
- A zombie process means that the process has exited, but its parent process has not reclaimed the resources occupied by the child process. We generally don’t need to pay attention to the short-term zombie state, but if the process is in the zombie state for a long time, we should pay attention to it. Some applications may not handle the exit of the child process normally.

