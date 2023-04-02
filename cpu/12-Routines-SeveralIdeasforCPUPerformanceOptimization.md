# 12 Routines: Several ideas for CPU performance tuning

In the previous section, we reviewed common CPU performance indicators, sorted
out the core CPU performance profiling tools, and finally summarized the ideas
for quickly analyzing CPU performance problems. Although there are many CPU
performance indicators and corresponding performance analysis tools, after
understanding the meaning of various indicators, you will find that they are
actually related to a certain extent.

Follow these relationships to understand, and you will find that it is not difficult
to master these commonly used bottleneck analysis routines.

After finding the performance bottleneck of the CPU, the next step is to optimize,
that is, to find out ways to make full use of the CPU so that more work can be done.

Today, I will talk about the ideas and precautions for optimizing CPU performance.

## Performance Optimization Methodology

After we have gone through untold hardships and used various performance analysis
methods to finally find the bottleneck that caused the performance problem, should
we start optimizing immediately? Don't worry, before you start, you can take a
look at the following three questions.

- First of all, since performance optimization is to be done, how to judge whether it is effective? Especially after optimization, how much performance can be improved?
- Second, performance problems are usually not independent. If multiple performance problems occur at the same time, which one should you optimize first?
- Third, the method to improve performance is not the only one. When there are multiple methods to choose from, which one would you choose? Is it always enough to choose the method that maximizes performance?

If you can easily answer these three questions, then you can start optimizing
without saying a word.

For example, in the previous case of an uninterruptible process, through performance
analysis, we found that the iowait was as high as 90% because of the **direct I/O**
of a process. Can it be optimized immediately by using the method of "**replacing
direct I/O with cached I/O**"?

According to the above, you can first think about the three questions yourself.
If you're not sure, let's take a look.

- For the first question, replacing direct I/O with cached I/O can reduce iowait from 90% to close to 0, and the performance improvement is obvious.
- The second problem, we did not find other performance problems, direct I/O is the only performance bottleneck, so there is no need to pick and optimize objects.
- The third question, caching I/O is the simplest optimization method we have used so far, and this optimization will not affect the function of the application.

OK, these three questions are easy to answer, so there's no problem with
immediately optimizing.

However, many real situations are not as simple as the examples I gave.
Performance evaluation may have multiple indicators, and multiple performance
problems may occur at the same time. Moreover, optimizing the performance of a
certain indicator may cause the performance of other indicators to decline.

So, in the face of this complex situation, what should we do?

Next, we will analyze these three questions in depth.

### How to evaluate the effect of performance optimization?

First of all, let's look at the first question, how to evaluate the effect of
performance optimization.

Our purpose of solving performance problems is naturally to obtain a performance
improvement effect. In order to evaluate this effect, we need to quantify the
performance indicators of the system, and test the performance indicators before
and after optimization, and use the changes of the indicators before and after to
compare the presentation effect. I call this method the performance evaluation
"three steps".

1. Determine the quantitative indicators of performance.
2. Test the performance indicators before optimization.
3. Test the optimized performance indicators.

Let’s look at the first step first. There are many quantitative indicators of
performance, such as CPU usage, application throughput, and client request latency,
etc., which can be used to evaluate performance. So what indicator should we choose
to evaluate?

My suggestion is not to limit yourself to single-dimension indicators. You should
choose different indicators from at least two dimensions: application and system
resources. For example, take the web application as an example:

- The dimension of the application, we can use the throughput and request latency to evaluate the performance of the application.
- The dimension of system resources, we can use CPU usage to evaluate the CPU usage of the system.

The reason for choosing metrics from these two different dimensions is mainly
because of the complementary relationship between application and system resources.

- A good application is the ultimate goal and result of performance optimization, and system optimization always serves the application. Therefore, it is necessary to use application metrics to evaluate the overall effect of performance optimization.
- System resource usage is the root cause of application performance. Therefore, it is necessary to use indicators of system resources to observe and analyze the source of bottlenecks.

As for the next two steps, the main purpose is to compare the performance before
and after optimization and present the effect more intuitively. If your first step
is to select multiple indicators from two different dimensions, then you need to
obtain the specific values of these indicators during performance testing.

Still take the web application just now as an example. Corresponding to the indicators
mentioned above, we can choose tools such as `ab` to test the number of concurrent
requests and response delay of the web application. While testing, you can also
use performance tools such as `vmstat` and `pidstat` to observe the CPU usage of
the system and processes. In this way, we obtain the indicator values of the two
dimensions of application and system resources at the same time.

However, when performing performance testing, there are two particularly important
things you need to pay attention to.

First, avoid performance testing tools interfering with application performance.
Typically, for web applications, the performance testing tool runs on a different
machine from the target application.

For example, in the previous Nginx case, I always emphasized the need to use two
virtual machines, one of which runs the Nginx service, and the other runs a tool
that simulates the client, so to avoid the benchmark tool impacting the application.

Second, avoid changes in the external environment affecting the evaluation of
performance indicators. This requires that the applications before and after
optimization run on machines with the same configuration, and their external
dependencies should be exactly the same.

For example, if you still take Nginx as an example, you can run it on the same
machine and use client tools with the same parameters for performance testing.

### Multiple performance problems exist at the same time, how to choose?

Let’s look at the second question again. As we said in the opening remarks, the
performance of the system always affects the whole body, so performance problems
usually do not exist independently. When multiple performance problems occur at
the same time, which one should be optimized first?

In the field of performance testing, a widely circulated saying is the "2-8 principle",
which means that 80% of the problems are caused by 20% of the code. Just by
finding those 20% you can optimize 80% of performance. So, what I want to express
is that **not all performance problems are worth optimizing**.

My suggestion is to use your brain before optimizing, analyze all these performance
problems first, find out the most important problem that can improve performance
to the greatest extent, and start optimizing from it. The advantage of this is
that not only does the performance improvement benefit the most, but it is also
possible that other issues can be met without optimization.

The key lies in how to judge which performance problem is the most important.
In fact, this is still the core problem to be solved in our performance analysis,
but the object to be analyzed here has changed from one problem to multiple problems,
and the idea is actually the same.

Therefore, you can still use the method I mentioned earlier to analyze one by one
to find out their bottlenecks. After analyzing all the problems, according to the
relationship between cause and effect, eliminate the performance problems with
causal relationship. Finally, optimize the remaining performance issues.

If there are still several remaining problems, you have to perform performance
tests separately. After comparing different optimization effects, choose the
problem that can significantly improve performance to fix. This process usually
takes a lot of time. Here, I recommend two approaches that can simplify this process.

First, if it is found that system resources have reached a bottleneck, such as
CPU usage reaching 100%, then the first thing to optimize must be the use of system
resources. After completing the optimization of system resource bottlenecks, we
then consider other issues.

Second, for different types of indicators, first optimize those problems that are
caused by bottlenecks and whose performance indicators vary the most. For example,
after a bottleneck occurs, the user CPU usage increases by 10%, while the system
CPU usage increases by 50%. At this time, the system CPU usage should be optimized
first.

### When there are many optimization approaches, how to choose?

Then look at the third question, which one should be chosen when multiple methods
are available? Is it the best way to improve performance?

Under normal circumstances, of course we want to choose the method that can improve
performance the most, which is actually the goal of performance optimization.

However, it should be noted that the factors to be considered in reality are not
so simple. Most intuitively, **performance optimization is not without cost**.
Performance optimization usually brings about an increase in complexity, reduces
the maintainability of the program, and may cause exceptions in other indicators
when optimizing one indicator. That is, it is quite possible that you optimize one
metric and the performance of another metric deteriorates.

A very typical example is the DPDK (Data Plane Development Kit) that I will talk
about in the networking section. DPDK is a method to optimize network processing
speed. It improves the processing capacity of the network by bypassing the kernel
network protocol stack.

However, it has a very typical requirement, which is to monopolize a CPU and a
certain number of large pages of memory, and always run at 100% CPU utilization.
So, if you have a small number of CPU cores, it's not worth the candle.

Therefore, when considering which performance optimization method to choose, you
have to integrate many factors. Remember, don't try to solve all the problems at
once by thinking of "one step to the sky"; and don't just use the optimization
methods of other applications without any thinking and analysis.

### CPU optimization

After clarifying the three most basic questions of performance optimization,
let's take a look at how to reduce CPU usage and improve CPU parallel processing
capabilities from the perspective of applications and systems.

#### Application optimization

First of all, from the application point of view, the best way to reduce CPU
usage is of course to exclude all unnecessary work and keep only the core logic.
For example, reduce the level of loops, reduce recursion, reduce dynamic memory
allocation, and so on.

In addition, application performance optimization also includes many methods, I
have listed the most common ones here, you can write them down.

- **Compiler optimization**: Many compilers provide optimization options. If you enable them appropriately, you can get help from the compiler during the compilation phase to improve performance. For example, gcc provides the optimization option -O2, which will automatically optimize the code of the application after it is turned on.
- **Algorithm optimization**: Using less complex algorithms can significantly speed up processing. For example, in the case of relatively large data, O(nlogn) sorting algorithms (such as quick sort, merge sort, etc.) can be used instead of O(n^2) sorting algorithms (such as bubble, insertion sort, etc.).
- **Asynchronous processing**: Using asynchronous processing can prevent the program from being blocked because it is waiting for a certain resource, thereby improving the concurrent processing capability of the program. For example, replacing polling with event notifications can avoid the problem of polling consuming CPU.
- **Multi-thread instead of multi-process**: As mentioned earlier, compared to process context switching, thread context switching does not switch process address space, so the cost of context switching can be reduced.
- **Make good use of cache**: Frequently accessed data or steps in the calculation process can be cached in memory, so that they can be directly obtained from memory the next time they are used, speeding up the processing speed of the program.

#### System optimization

From the perspective of the system, to optimize the operation of the CPU, on the
one hand, it is necessary to make full use of the locality of the CPU cache to
speed up cache access; on the other hand, it is to control the CPU usage of the
process and reduce the mutual influence between processes.

Specifically, there are many CPU optimization methods at the system level. Here
I also list some of the most common methods for your convenience in memory and use.

- **CPU binding**: Binding a process to one or more CPUs can improve the CPU cache hit rate and reduce context switching problems caused by cross-CPU scheduling.
- **CPU exclusive**: Similar to CPU binding, CPUs are further grouped and processes are assigned to them through the CPU affinity mechanism. In this way, these CPUs are exclusively occupied by the specified process, in other words, no other process is allowed to use these CPUs.
- **Priority adjustment**: use nice to adjust the priority of the process, a positive value lowers the priority, and a negative value increases the priority. We mentioned the numerical meaning of the priority earlier, if you forget it, review it in time. Here, appropriately reducing the priority of non-core applications and increasing the priority of core applications can ensure that core applications are processed first.
- **Set resource limits for processes**: Use Linux cgroups to set the upper limit of the CPU usage of the process, which can prevent system resources from being exhausted due to an application's own problems.
- **NUMA (Non-Uniform Memory Access) optimization**: A processor that supports NUMA will be divided into multiple nodes, and each node has its own local memory space. NUMA optimization is actually to let the CPU only access local memory as much as possible.
- **Interrupt load balancing**: Regardless of whether it is a soft interrupt or a hard interrupt, their interrupt handlers may consume a lot of CPU. Enable the irqbalance service or configure smp_affinity to automatically load balance the interrupt processing process to multiple CPUs.

## Do not optimize prematurely

After mastering the above optimization methods, I estimate that even if many
people don't find performance bottlenecks, they can't help but bring various
optimization methods into actual development.

However, I think you must have heard the famous saying of Knuth(Donald Ervin Knuth),
"Premature optimization is the root of all evil", I also very much agree with this,
premature optimization is not advisable.

Because, on the one hand, optimization will increase complexity and reduce
maintainability; on the other hand, requirements are not static. Optimizing for
the current situation may not adapt to the rapidly changing new requirements.
In this way, when new requirements arise, these complex optimizations may hinder
the development of new functions.

Therefore, performance optimization is best done gradually and dynamically, not
in one step, but first to ensure that the current performance requirements can
be met. When it is found that the performance does not meet the requirements or
a performance bottleneck occurs, then according to the results of the performance
evaluation, select the most important performance issues for optimization.

## Summary

Today, I will take you through common CPU performance optimization ideas and
optimization methods. After discovering performance problems, do not rush to
optimize them, but first find out the most important problems that can achieve
the greatest performance improvement, and then start optimizing from the two
aspects of the application program and the system.

In this way, not only can the maximum performance improvement be obtained, but
also the performance requirements are likely to be met without optimizing other problems.

But remember, be sure to resist the urge to "optimize CPU performance to the extreme",
because CPU is not the only performance factor. In subsequent articles, I will
introduce more performance issues, such as memory, network, I/O and even
architectural design issues.

If you do not conduct all-round analysis and testing, simply improving a certain
indicator to the extreme may not necessarily bring overall benefits.

