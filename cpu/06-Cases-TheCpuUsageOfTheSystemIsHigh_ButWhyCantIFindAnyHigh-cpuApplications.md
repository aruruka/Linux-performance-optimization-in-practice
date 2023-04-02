# 06 Cases: The Cpu Usage Of The System Is High, But Why Can't I Find Any High-cpu Applications?

In the previous section, I talked about what CPU usage is, and through a case,
I taught you to use tools such as `top`, `vmstat`, and `pidstat` to troubleshoot
processes with high CPU usage, and then use the `perf top` tool to locate problems
with internal functions of the application. However, someone left a message,
saying that it seems that the problem of high CPU usage is quite easy to troubleshoot.

So can all problems with high CPU usage be analyzed in this way?
I think your answer should be no.

Looking back at the previous content, we know that the CPU usage of the system
includes not only the running of the process user space and kernel space, but also
interrupt processing, waiting for I/O, and kernel threads.
Therefore, **when you find that the CPU usage of the system is high, you may not
always be able to find the corresponding high CPU usage process**.

Today, I will use a case of Nginx + PHP Web service to take you to analyze this situation.

## Case analysis

### Your preparation

Today, we are still exploring the high CPU usage of the system, so the preparation
for this experiment is basically the same as that of the previous lesson, the
difference is that the Docker image used in the case is different.

This case is still based on Ubuntu 22.04, and it is also applicable to other Linux systems.
The case environment I'm using looks like this:

- Machine spec: 2 CPU, 8GB memory
- Pre-install `docker`, `sysstat`, `perf`, `ab` and other tools, such as `apt install docker.io sysstat linux-tools-common apache2-utils`

As we mentioned earlier, `ab` (apache bench) is a commonly used HTTP service
performance testing tool, which is also used here to simulate the Nginx client.
Since the configuration of Nginx and PHP is cumbersome, I packaged them into
two 🔗[Docker images](https://github.com/feiskyer/linux-perf-examples/tree/master/nginx-short-process), so that only two containers need to be run to get a simulated environment.

Note that this case uses two virtual machines, as shown in the following figure:

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/fYV2vdb/05-Case-Analysis-Vm-Role-Diagram.png" alt="05-Case-Analysis-Vm-Role-Diagram" border="0">
</a></center>

You can see that one is used as a web server to simulate performance issues
and the other is used as a client of the web server to stress the web service
with requests. The use of two virtual machines is to isolate each other and
avoid "cross-infection".

Next, we open two terminals, log in to the two machines with SSH, and install
the above tools.

Also note that all the following commands are run as the root user by default.
If you log in to the system as a normal user, please run the `sudo su root`
command to switch to the root user.

> Tips: The core logic of the PHP application in the case is relatively simple,
> and you may be able to see the problem at a glance, but the source code in
> the actual production environment is much more complicated. Therefore, I still
> suggest that you should not look at the source code before operating it,
> avoid preconceived ideas, but analyze it as a black box. In this way, you can
> better grasp how to analyze the application where the bottleneck is located
> and the approximate location of the bottleneck in the application based on
> the resource usage of the system.

### Operation and Analysis

First, in the first terminal, execute the following command to run Nginx and PHP applications:

```shell
$ docker run --privileged --name nginx -p 10000:80 -itd feisky/nginx:sp
$ docker run --privileged --name phpfpm -itd --network container:nginx feisky/php-fpm:sp
```

Then, in the second terminal, use `curl` to access http://[VM1's IP]:10000 to
confirm that Nginx has started normally. You should see the "It works!" response.

```shell
# 192.168.153.130是第一台虚拟机的IP地址
$ curl http://192.168.153.130:10000
It works!
```

Next, let's test the performance of this Nginx service. Run the `ab` command
below in the second terminal. It should be noted that, unlike the last operation,
this time we need to test Nginx performance with 100 concurrent requests, and
a total of 1000 requests are tested.

```shell
# 并发100个请求测试Nginx性能，总共测试1000个请求
$ ab -c 100 -n 1000 http://192.168.153.130:10000/
This is ApacheBench, Version 2.3 <$Revision: 1879490 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
...
Requests per second:    178.48 [#/sec] (mean)
Time per request:       560.291 [ms] (mean)
...
```

From the output of `ab`, we can see that the average number of requests per second
that Nginx can withstand is only a little more than 178.
Do you feel that its performance is a bit poor? So, what went wrong?
Let's use `top` and `pidstat` to observe again.

This time, in the second terminal, we changed the number of concurrent requests
tested to 5, and set the request duration to 10 minutes (-t 600).
This way, when you use the profiling tool on the first terminal, the pressure on Nginx continues.

Go ahead and run the `ab` command in the second terminal:

```shell
$ ab -c 5 -t 600 http://192.168.153.130:10000/
```

Then, we run the `top` command in the first terminal to observe the CPU usage of the system:

```shell
$ top
...
%Cpu(s): 67.4 us, 24.4 sy,  0.0 ni,  2.8 id,  0.0 wa,  0.0 hi,  5.4 si,  0.0 st
...

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   5027 systemd+  20   0   33140   3576   2244 S   5.0   0.0   0:08.40 nginx
   4904 root      20   0  720500   9108   7496 S   4.0   0.1   0:03.46 containerd-shim
  19711 daemon    20   0  336700  16992   9312 S   2.7   0.2   0:03.80 php-fpm
  19715 daemon    20   0  336700  15952   8276 S   2.7   0.2   0:03.85 php-fpm
  19723 daemon    20   0  336700  15952   8276 S   2.7   0.2   0:03.76 php-fpm
  19727 daemon    20   0  336700  15956   8280 S   2.7   0.2   0:03.78 php-fpm
   1080 root      20   0 1390728  76196  52556 S   2.3   0.9   0:03.85 dockerd
  19736 daemon    20   0  336700  15956   8280 S   2.3   0.2   0:03.76 php-fpm
     23 root      20   0       0      0      0 S   1.3   0.0   0:02.38 ksoftirqd/1
     14 root      20   0       0      0      0 S   0.3   0.0   0:00.39 ksoftirqd/0
     15 root      20   0       0      0      0 I   0.3   0.0   0:00.53 rcu_preempt
    297 root      20   0       0      0      0 I   0.3   0.0   0:00.38 kworker/u256:29-events_unbound
```

Observing the process list output by `top`, it can be found that the process
with the highest CPU usage is only 5.0%, which does not seem to be high.
However, if you look at the line of system CPU usage (%Cpu), you will find that
the overall CPU usage of the system is relatively high: the user CPU usage (`us`)
has reached 67.4%, the system CPU is 24.4%, and the idle CPU (id) is only 2.8%.

Why is the user CPU usage so high? Let's re-analyze the process list to see if
there are any suspicious processes:

- the docker-containerd process is used to run containers, 2.3% CPU usage seems normal;
- Nginx and php-fpm run web services, it is not surprising that they will take up some CPU, and the CPU usage of 5% and 2.7% is not high;
- Looking further down, the following process, only 0.3% CPU usage, doesn't look like it will cause the user CPU usage to reach 80%.

That's strange, obviously the user's CPU usage rate is 67.4%, but we analyzed
the process list one by one, but still couldn't find the process with high CPU
usage rate. It seems that `top` is not working, so is there any other tool to
check the CPU usage of the process? Don't know if you remember our old friend
`pidstat`, it can be used to analyze the CPU usage of the process.

```shell
# 间隔1秒输出一组数据（按Ctrl+C结束）
$ pidstat 1
...
01时30分01秒   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
01时30分02秒     0        23    0.00    1.96    0.00    0.00    1.96     1  ksoftirqd/1
01时30分02秒     0        48    0.00    0.98    0.00    0.00    0.98     1  kworker/1:1H-kblockd
01时30分02秒     0      1080    1.96    0.00    0.00    0.00    1.96     1  dockerd
01时30分02秒     0      4904    0.98    1.96    0.00    0.00    2.94     0  containerd-shim
01时30分02秒   101      5026    0.00    0.98    0.00    0.00    0.98     1  nginx
01时30分02秒   101      5027    0.00    6.86    0.00    1.96    6.86     1  nginx
01时30分02秒  1000    355630    0.98    0.00    0.00    0.00    0.98     0  sshd
01时30分02秒     1    362608    0.00    1.96    0.00    3.92    1.96     0  php-fpm
01时30分02秒     1    362612    0.98    1.96    0.00    2.94    2.94     1  php-fpm
01时30分02秒     1    362620    0.00    1.96    0.00    2.94    1.96     0  php-fpm
01时30分02秒     1    362623    0.00    1.96    0.00    1.96    1.96     1  php-fpm
01时30分02秒     1    362636    0.00    1.96    0.00    2.94    1.96     0  php-fpm
```

Observe for a while, do you find that the CPU usage of all processes is not high,
the highest Nginx and containerd-shim are only 6.86% and 2.94%, even if the CPU
usage of all processes is added up, it is only 27.44%. It's still far from 67.4%!

At the earliest time, I was completely stunned when I encountered this kind of
problem: the CPU usage rate of the user had already reached 67.4%, but I couldn't
find which process was the problem. At this point, you can also think about it,
have you also encountered this situation? Can we do further analysis?

Later, I found out that this happened probably because some key information was
missed in the previous analysis. You can pause for a while, scroll up by yourself,
and re-operate and check again. Or, let's go back and analyze the output of top
to see if we can find anything new.

Now, let's go back to the first terminal, re-run the top command, and watch for a while:

```shell
$ top
top - 17:50:04 up  3:42,  0 users,  load average: 4.78, 4.15, 2.70
Tasks:  20 total,   8 running,  14 sleeping,   0 stopped,   0 zombie
%Cpu(s): 68.3 us, 21.1 sy,  0.0 ni,  5.0 id,  0.0 wa,  0.0 hi,  5.5 si,  0.0 st
KiB Mem :  8104760 total,  5191884 free,  1292636 used,  1620240 buff/cache
KiB Swap:  2097148 total,  2097148 free,        0 used.  6493988 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 450522 daemon    20   0  336700  15956   8280 S   2.7  0.2   0:06.33 php-fpm
 450529 daemon    20   0  336700  15956   8280 S   2.7  0.2   0:06.20 php-fpm
 450543 daemon    20   0  336700  15956   8280 S   2.7  0.2   0:06.12 php-fpm
 450518 daemon    20   0  336700  15956   8280 S   2.3  0.2   0:06.23 php-fpm
 450532 daemon    20   0  336700  15956   8280 S   2.3  0.2   0:06.17 php-fpm
 838984 root      20   0   18196   3320   2784 S   0.0  0.0   0:00.00 bash
 995397 root      20   0   43176   3280   2696 R   0.0  0.0   0:00.00 top
 589544 daemon    20   0    7288    872    788 S   0.0  0.0   0:00.00 stress
 589545 daemon    20   0    4296    712    640 S   0.0  0.0   0:00.00 sh
 589546 daemon    20   0    8188    832    536 R   0.0  0.0   0:00.00 stress
 589547 daemon    20   0     336      4      0 R   0.0  0.0   0:00.00 stress
 589548 daemon    20   0    4296    704    632 S   0.0  0.0   0:00.00 sh
 589549 daemon    20   0    7288    880    796 S   0.0  0.0   0:00.00 stress
 589550 daemon    20   0    7288    872    792 S   0.0  0.0   0:00.00 stress
 589551 daemon    20   0    7288    100      0 R   0.0  0.0   0:00.00 stress
 589552 daemon    20   0     436      4      0 R   0.0  0.0   0:00.00 sh
 589553 daemon    20   0    7288     96      0 R   0.0  0.0   0:00.00 stress
...
```

Look at each line of top output from the beginning this time, huh?
The line of Tasks looks a bit strange. There are actually 8 running processes
in the ready queue. Is it a bit too much?

Recall the parameters of the `ab` test, the number of concurrent requests is 5.
Looking at the process list, the number of php-fpm is 6, plus Nginx and Dockerd,
it seems that there are 8 processes at the same time, which is not surprising.
But is that really the case?

Look carefully at the process list again, this time mainly look at the processes
in the Running (R) state. Have you noticed that Nginx and all php-fpm are in the
Sleep (S) state, but there are several stress processes in the Running (R) state.
These stress processes are quite strange, and we need to do further analysis.

We still use `pidstat` to analyze these processes, and use the `-p` option to
specify the PID of the process. First, find the PIDs of these processes from the
top results above. For example, first find a random 589544, and then use the
`pidstat` command to see its CPU usage:

```shell
$ pidstat -p 589544
Linux 5.19.0-35-generic (7c6dc68f53dd)  03/24/23        _x86_64_        (2 CPU)

03:40:41      UID       PID    %usr %system  %guest    %CPU   CPU  Command
```

Strange, there is no output. Could it be something wrong with the `pidstat` command?
As I said before, it is best to cross-check with other tools before suspecting
a problem with a performance tool. What tools do you use? `ps` should be the
easiest to use. We run the following command in the terminal to see the status
of the 589544 process:

```shell
$ ps aux | grep 589544
root      914730  0.0  0.0  11120   972 pts/1    S+   03:42   0:00 grep 589544
```

Still no output. Now I finally found the problem. It turns out that this process
no longer exists, so `pidstat` has no output. Now that the process is gone,
the performance problem should also be gone. Let's confirm again with the top command:

```shell
$ top
...
%Cpu(s): 66.7 us, 22.7 sy,  0.0 ni,  4.7 id,  0.0 wa,  0.0 hi,  5.9 si,  0.0 st
...

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 450522 daemon    20   0  336700  15956   8280 S   2.7  0.2   0:06.33 php-fpm
 450529 daemon    20   0  336700  15956   8280 S   2.7  0.2   0:06.20 php-fpm
 450543 daemon    20   0  336700  15956   8280 S   2.7  0.2   0:06.12 php-fpm
 450518 daemon    20   0  336700  15956   8280 S   2.3  0.2   0:06.23 php-fpm
 450532 daemon    20   0  336700  15956   8280 S   2.3  0.2   0:06.17 php-fpm
 838984 root      20   0   18196   3320   2784 S   0.0  0.0   0:00.00 bash
 995397 root      20   0   43176   3280   2696 R   0.0  0.0   0:00.00 top
1001907 daemon    20   0    4296    744    672 S   0.0  0.0   0:00.00 sh
1001908 daemon    20   0    4296    756    680 S   0.0  0.0   0:00.00 sh
1001910 daemon    20   0    4296    700    628 S   0.0  0.0   0:00.00 sh
1001912 daemon    20   0    7288    968    884 S   0.0  0.0   0:00.00 stress
1001914 daemon    20   0    7288    864    780 S   0.0  0.0   0:00.00 stress
1001915 daemon    20   0    7288    908    828 S   0.0  0.0   0:00.00 stress
1001917 daemon    20   0    8192   1096    536 R   0.0  0.0   0:00.00 stress
1001918 daemon    20   0    8192   1352    532 R   0.0  0.0   0:00.00 stress
1001919 daemon    20   0    8188     96      0 R   0.0  0.0   0:00.00 stress
1001920 daemon    20   0  336700   7680      4 R   0.0  0.1   0:00.00 php-fpm
```

However, just now we saw that the `stress` process does not exist, why is it
still running now? Take a closer look at the output of `top`. It turns out that
the PID of the stress process is different from the previous one.
The original PID 589544 is gone, and now it is 1001912.

The PID of the process is changing, what does this mean? In my opinion, either
these processes are constantly restarting, or they are brand new processes, and
this is nothing more than two reasons:

- The first reason is that the process keeps crashing and restarting, for example, due to segmentation faults, configuration errors, etc. At this time, the process may be automatically restarted by the monitoring system after exiting.
- The second reason is that these processes are short-lived processes, that is, external commands called by `exec` inside other applications. These commands generally only run for a short time before ending, and it is difficult for you to find out with a tool such as `top` with a relatively long interval (the above case, we happened to find out).

As for `stress`, we mentioned earlier that it is a commonly used stress testing tool.
Its PID is constantly changing, and it looks like a short-lived process being
called by another process. To continue the analysis, you have to find their parent process.

How to find the parent process of a process? That's right, `pstree` can be used
to display the relationship between all processes in tree form:

```shell
$ pstree | grep stress
        |-containerd-shim-+-php-fpm-+-php-fpm---sh---stress
        |                 |         |-2*[php-fpm---sh---stress---stress]
```

As you can see from here, `stress` is a subprocess called by `php-fpm`,
and there are more than one process (here there are 2). After finding the parent
process, we can enter the internal analysis of the app.

First of all, of course, you should look at its source code. Run the following
command to copy the source code of the case application to the <ins>app</ins>
directory, and then execute `grep` to find out whether there is code calling the 
`tress` command:

```shell
# 拷贝源码到本地
$ docker cp phpfpm:/app .

# grep 查找看看是不是有代码在调用stress命令
$ grep stress -r app
app/index.php:// fake I/O with stress (via write()/unlink()).
app/index.php:$result = exec("/usr/local/bin/stress -t 1 -d 1 2>&1", $output, $status);
```

Found it, it turned out that the `stress` command was directly called in the
<ins>app/index.php</ins> file.

Let's take a look at the source code of 🔗<ins>app/index.php</ins>:

```php
$ cat app/index.php
<?php
// fake I/O with stress (via write()/unlink()).
$result = exec("/usr/local/bin/stress -t 1 -d 1 2>&1", $output, $status);
if (isset($_GET["verbose"]) && $_GET["verbose"]==1 && $status != 0) {
  echo "Server internal error: ";
  print_r($output);
} else {
  echo "It works!";
}
?>
```

It can be seen that a `stress` command is called for each request in the source
code to simulate I/O pressure. From the comments, stress will test the I/O
process through `write()` and `unlink()`. It seems that this should be the root
cause of the increased CPU usage of the system.

However, stress simulates I/O pressure, and what I saw in the output of `top`
before is that the user CPU and system CPU have always increased, and iowait
has not seen an increase. What's the matter? Is `stress` realy the cause of the
increased CPU usage?

We have to keep going down. As you can see from the code, after adding the
verbose=1 parameter to the request, you can view the output of `stress`.
Try it first, run in the second terminal:

```shell
$ curl http://192.168.153.130:10000?verbose=1
Server internal error: Array
(
    [0] => stress: info: [1289829] dispatching hogs: 0 cpu, 0 io, 0 vm, 1 hdd
    [1] => stress: FAIL: [1289830] (563) mkstemp failed: Permission denied
    [2] => stress: FAIL: [1289829] (394) <-- worker 1289830 returned error 1
    [3] => stress: WARN: [1289829] (396) now reaping child worker processes
    [4] => stress: FAIL: [1289829] (400) kill error: No such process
    [5] => stress: FAIL: [1289829] (451) failed run completed in 0s
)
```

See the error message **mkstemp failed: Permission denied**, and **failed run
completed in 0s**. It turns out that the `stress` command didn't succeed, it
failed and exited due to a permission issue. Apparently, we found a bug in PHP
calling an external `stress` command: no permission to create temporary files.

From here we can guess that it is precisely because of permission errors that
a large number of `stress` processes fail to initialize at startup, which leads
to an increase in user CPU usage.

After analyzing the source of the problem, is the next step to start optimization?
Of course not! Since it is just a guess, it needs to be confirmed again, whether
this guess is correct, and whether there are really a large number of `stress`
processes. What tools or indicators should I use?

We have used tools such as `top`, `pidstat`, and `pstree` before, but we did 
not find a large number of `stress` processes. So, are there any other tools that can be used?

Remember the `perf` mentioned in the last issue? It can be used to analyze CPU
performance events, which is very suitable here. Still run the `perf record -g`
command in the first terminal, wait for a while (say 15 seconds) and press Ctrl+C to exit.
Then run `perf report` to view the report:

```shell
# 记录性能事件，等待大约15秒后按 Ctrl+C 退出
$ perf record -g

# 查看报告
$ perf report
```

In this way, you can see the performance report as shown below:

![perf record -g](https://static001.geekbang.org/resource/image/c9/33/c99445b401301147fa41cb2b5739e833.png?wh=720*527 "report of 'perf record -g'")

You see, `stress` accounts for 77% of all CPU clock events, and the highest
proportion of the stress call stack is the random number generator function
`random()`. It seems that it is indeed the culprit of the increased CPU usage.
Subsequent optimization is very simple, as long as the permission problem is fixed,
and the call of `stress` is reduced or deleted, the CPU pressure of the system can be reduced.

Of course, the problems in the actual production environment are generally more
complicated than this case. After you find the command line that triggers the
bottleneck, you may find that the calling process of this external command is
part of the core logic of the application and cannot be easily reduced or deleted.

At this time, you have to continue to investigate why the command being called
will cause problems such as increased CPU usage or increased I/O. I will analyze
the cases of these complex scenarios in detail in the comprehensive actual
practice later.

Finally, at the end of the example, don't forget to clean up the environment
and execute the following Docker command to stop the Nginx process used in the example:

```shell
$ docker rm -f nginx phpfpm
```

## execsnoop

In this case, we used tools such as `top`, `pidstat`, and `pstree` to analyze
the problem of high CPU usage in the system, and found that the increase in CPU
was caused by short-term process `stress`, but the entire analysis process is
still relatively complicated. For this kind of problem, is there a better way to monitor it?

🔗[execsnoop](https://github.com/brendangregg/perf-tools/blob/master/execsnoop) is a tool designed for short-lived processes. It monitors the
`exec()` behavior of the process in real time through `ftrace`, and outputs the
basic information of the short-term process, including process PID, parent
process PID, command line parameters and execution results.

For example, if you use `execsnoop` to monitor the above case, you can directly
get the PID of the parent process of the `stress` process and its command line
parameters, and you can find that a large number of stress processes are
constantly starting:

```shell
# 按 Ctrl+C 结束
$ execsnoop
PCOMM            PID    PPID   RET ARGS
sh               30394  30393    0
stress           30396  30394    0 /usr/local/bin/stress -t 1 -d 1
sh               30398  30393    0
stress           30399  30398    0 /usr/local/bin/stress -t 1 -d 1
sh               30402  30400    0
stress           30403  30402    0 /usr/local/bin/stress -t 1 -d 1
sh               30405  30393    0
stress           30407  30405    0 /usr/local/bin/stress -t 1 -d 1
...
```

The `ftrace` used by `execsnoop` is a commonly used dynamic tracking technology,
which is generally used to analyze the runtime behavior of the Linux kernel.
I will introduce it in detail and take you to use it in the following courses.

## Summary

When you encounter a CPU usage situation that cannot be explained by conventional
problems, you must first think that the problem may be caused by a short-term
application, such as the following two situations.

- First, other binary programs are directly called in the application. These programs usually run for a short time and are not easy to find through tools such as top
- Second, the application itself is constantly crashing and restarting, and the resource initialization during the startup process is likely to occupy a considerable amount of CPU.

For such processes, we can use `pstree` or `execsnoop` to find their parent process,
and then start with the application where the parent process is located to
troubleshoot the root cause of the problem.