# 09 Basics: How To Understand Linux Soft Interrupt?

In the last issue, I used a case of an uninterruptible process to take you to
learn the analysis method when iowait (that is, the CPU usage rate waiting for I/O)
increases. Here you have to remember that the uninterruptible state of the process
is a protection mechanism of the system, which can ensure that the hardware
interaction process will not be interrupted accidentally. Therefore, a short
period of uninterruptible state is normal.

However, when the process is in an uninterruptible state for a long time, you
have to be careful. At this time, you can use `dstat`, `pidstat` and other tools
to confirm whether it is a disk I/O problem, and then troubleshoot related
processes and disk devices. Regarding the performance of disk I/O, you don’t
need to memorize it for the time being. I will introduce it in detail in the
subsequent I/O section, and remember it when you understand it.

In fact, in addition to iowait, soft interrupt (softirq) CPU usage is also the
most common performance problem. In the next two lessons, we will learn the
content of soft interrupts. I will also take you to analyze this situation with
the most common case of reverse proxy server Nginx.

## Think of interruptions as "takeaways"

Speaking of interrupts, I briefly talked about the meaning of interrupts in the
previous 🔗[article on "context switching"](03-Basicc-WhatIsTheMeaningOfCPUContextSwitchingThatIsOftenSaid-Part1.md). Let's review it first.
Interrupt is a mechanism used by the system to respond to hardware device requests.
It interrupts the normal scheduling and execution of processes, and then calls
the interrupt handler in the kernel to respond to device requests.

You may have to ask, why is there an interruption? I can give an example from my
life to let you feel the charm of interruption.

For example, you ordered a takeaway, but you are not sure when it will be delivered,
and there is no other way to know the progress of the takeaway. However, The
courier will not wait for you. If the courier gets to your home and noone comes
to take the food, the courier will just go away. So you can only wait hard,
go to the door from time to time to see if the takeaway is delivered, and you
can't do other things.

However, if you make an appointment with the courier when ordering food delivery
and ask him to call you after delivery, then you don’t have to wait so hard, you
can go to other things until the phone rings, answer the phone and pick up takeaway.

The "call" here is actually an interruption. When you don't receive a call, you
can do other things; only when you receive a call (that is, an interruption occurs),
you have to perform another action: pick up takeaway.

From this example, you can find that **interrupt is actually an asynchronous
event processing mechanism, which can improve the concurrent processing capability
of the system**.

Since the interrupt handler will interrupt the operation of other processes, **in
order to reduce the impact on the normal process operation scheduling, the
interrupt handler needs to run as fast as possible**. If the interrupt itself does
not have many things to do, then it will not be too much of a problem to handle;
but if the interrupt has a lot to do, the interrupt service routine may take a
long time to run.

In particular, interrupt handlers also temporarily disable interrupts while
responding to interrupts. This will cause a possibility that other interrupts cannot
be responded before the last interrupt processing is completed, which means that
the interrupt may be lost.

So let’s take takeaway as an example. Suppose you ordered 2 takeaways, a main
dish and a drink, and they were delivered by 2 different couriers. You don't
have to wait all the time this time, the two takeaways have agreed on the way to
pick up the takeaway by phone. However, here comes a new problem.

When the first delivery arrives, the courier makes a long phone call to discuss
what to do with the invoice. Meanwhile, a second courier arrives and wants to
call you too.

But obviously, because the phone is busy (that is, the interrupt response is
turned off), the second courier person's phone cannot be connected. So, the second
courier will probably walk away after a few tries (i.e. lose an interrupt).

## Soft Interrupt

If you figure out the "take-away" mode, it's easy to understand the system's
interruption mechanism. In fact, in order to solve the problem of excessive
execution of interrupt handlers and interrupt loss, Linux divides the interrupt
processing process into two stages, namely **the first half and the second half**:

- **The first part is used to quickly handle interrupts**. It runs in interrupt-disabled mode and mainly handles hardware-related or time-sensitive work.
- **The second half is used to delay processing the unfinished work of the first half, and usually runs as a kernel thread**.

For example, in the previous example of taking food delivery, the first half is
for you to answer the phone, tell the courier that you already know, and we can
talk about other things when we meet, and then you can hang up the phone; the
second half is the action of taking food delivery, and the action of meeting then
discuss the handle of invoice.

In this way, the first courier will not take up too much of your time, and when
the second courier comes, he can still get through your phone normally.

In addition to taking out food, let me give you an example of the most common
network card receiving data packets to give you a better understanding.

After the network card receives the data packet, it will notify the kernel that
new data has arrived through a **hardware interrupt**. At this time, the kernel
should call the interrupt handler to respond to it. You can think about it for
yourself, what are the first and second parts responsible for in this case?

For the first part, since it is fast processing, it is actually to read the data
of the network card into the memory, then update the status of the hardware
register (indicating that the data has been read), and finally send a soft
interrupt signal to notify the second half for further processing.

After the second half is awakened by the soft interrupt signal, it needs to find
the network data from the memory, and then analyze and process the data layer by
layer according to the network protocol stack until it is sent to the application
program.

Therefore, you can also understand these two stages as follows:

- The first half directly processes hardware requests, which is what we often call hard interrupts, and is characterized by fast execution;
- The second half is triggered by the kernel, which is what we often call soft interrupts, which is characterized by delayed execution.

In effect, the first half interrupts the task the CPU is executing, and immediately
executes the interrupt handler. The second half is executed as a kernel thread,
and each CPU corresponds to a softirq kernel thread named "ksoftirqd/CPU number".
For example, the name of the softirq kernel thread corresponding to CPU No. 0 is
ksoftirqd/0.

However, it should be noted that soft interrupts not only include the lower half
of the hardware device interrupt handler just mentioned, some kernel-defined events
also belong to soft interrupts, such as kernel scheduling and RCU locks
(abbreviation for Read-Copy Update, RCU is one of the most commonly used locks
in the Linux kernel), etc.

So how do you know which soft interrupts are in your system?

### View softirqs and kernel threads

I don’t know if you still remember, the proc file system mentioned earlier. It
is a mechanism for communicating between kernel space and user space, which can
be used to view the data structure of the kernel, or to dynamically modify the
configuration of the kernel. in:

- <ins>/proc/softirqs</ins> provides the running status of softirqs;
- <ins>/proc/interrupts</ins> provides the operation of hard interrupts.

Run the following command to view the contents of the <ins>/proc/softirqs</ins>
file, and you can see the cumulative running times of various types of softirqs
on different CPUs:

```shell
$ cat /proc/softirqs
                    CPU0       CPU1
          HI:          0          0
       TIMER:     811613    1972736
      NET_TX:         49          7
      NET_RX:    1136736    1506885
       BLOCK:          0          0
    IRQ_POLL:          0          0
     TASKLET:     304787       3691
       SCHED:     689718    1897539
     HRTIMER:          0          0
         RCU:    1330771    1354737
```

When viewing the contents of the <ins>/proc/softirqs</ins> file, you should pay
special attention to the following two points.

First, pay attention to the type of soft interrupt, which is the content of the
first column in this interface. From the first column, you can see that soft
interrupts include 10 categories, corresponding to different types of work. For
example, NET_RX indicates the interruption of the network reception, and NET_TX
indicates the interruption of network transmission.

Second, pay attention to the distribution of the same soft interrupt on different
CPUs, that is, the content of the same line. Under normal circumstances, the
cumulative number of interrupts of the same type on different CPUs should be about
the same. For example, in this interface, the number of interrupts of NET_RX on
CPU0 and CPU1 is basically the same order of magnitude, with little difference.

However, you may find that TASKLET is not evenly distributed on different CPUs.
TASKLET is the most commonly used soft interrupt implementation mechanism. Each
TASKLET will end after running only once, and it will only run on the CPU where
the function calling it is located.

Therefore, it is very convenient to use TASKLET. Of course, there are some problems,
such as unbalanced scheduling caused by running on only one CPU, and performance
limitations caused by not being able to run in parallel on multiple CPUs.

In addition, as mentioned just now, softirqs actually run as kernel threads, and
each CPU corresponds to a softirq kernel thread, and this softirq kernel thread
is called ksoftirqd/CPU number. So how do you check the running status of these
threads?

In fact, it can be done with the `ps` command, for example, execute the following
command:

```shell
$ ps aux | grep softirq
root         7  0.0  0.0      0     0 ?        S    Oct10   0:01 [ksoftirqd/0]
root        16  0.0  0.0      0     0 ?        S    Oct10   0:01 [ksoftirqd/1]
```

Note that the names of these threads have square brackets around them, which
means that `ps` cannot get their command line parameters (cmline). Generally
speaking, in the output of `ps`, the names enclosed in square brackets are
generally kernel threads.

## Summary

Interrupt handlers in Linux are divided into first and second halves:

- The first half corresponds to hardware interrupts and is used to quickly handle interrupts.
- The second half corresponds to the soft interrupt, which is used to asynchronously process the unfinished work of the first half.

Softirqs in Linux include various types such as network sending and receiving,
timing, scheduling, and RCU locks. You can observe the running status of softirqs
by viewing <ins>/proc/softirqs</ins>.