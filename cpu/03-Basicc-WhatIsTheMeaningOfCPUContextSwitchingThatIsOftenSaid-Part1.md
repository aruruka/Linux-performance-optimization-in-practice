# 03 Basics: What is the meaning of CPU context switching that is often said? (Part 1)

In the last section, I told you how to understand the load average (Load Average),
and used three cases to show the analysis method of the average load increase in
different scenarios. Among them, multiple processes competing for CPU is a problem
that is often overlooked by us.

I think you must be wondering why the process is not actually running when it is
competing for the CPU, so why does it cause the load on the system to increase?
Seeing today's topic, you should have guessed that CPU context switching is the culprit.

We all know that Linux is a multitasking operating system, which supports tasks
that are far greater than the number of CPUs to run at the same time. Of course,
these tasks are not actually running at the same time, but because the system
allocates CPUs to them in turn in a short period of time, creating the illusion
that multiple tasks are running at the same time.

Before each task runs, the CPU needs to know where the task is loaded and started
to run, that is to say, the system needs to set the **CPU register** and **program counter**
(Program Counter, PC) for it in advance.

The CPU register is a small-capacity but extremely fast memory built into the CPU.
The program counter is used to store the position of the instruction being executed
by the CPU, or the position of the next instruction to be executed. They are all
dependent environments that the CPU must run before running any tasks, so they
are also called **CPU contexts**.

<a href="https://imgbb.com/"><img src="https://i.ibb.co/1RbBqMN/Cpu-Register-Program-Counter.png" alt="Cpu-Register-Program-Counter" border="0"></a>

Knowing what a CPU context is, I think you can easily understand **CPU context switching**.
CPU context switching is to save the CPU context (that is, CPU register and program counter)
of the previous task, then load the context of the new task to this register
and program counter, and finally jump to the new location pointed by the program counter,
to run the new task.

These saved contexts will be stored in the system kernel and loaded again when
the task is rescheduled for execution. In this way, the original state of the
task will not be affected, and the task will appear to be running continuously.

I guess some people will say that CPU context switching is nothing more than
updating the value of CPU registers, but these registers are designed to run tasks
quickly. Why would it affect the CPU performance of the system?

Before answering this question, I wonder if you have ever thought about what are
these "tasks" managed by the operating system?

Maybe you will say that a task is a process, or a task is a thread.
Yes, processes and threads are exactly the most common tasks. But other than that,
are there any other tasks?

Don't forget that it is also a common task for the hardware to trigger the signal,
which will cause the call of the interrupt handler.

Therefore, **<span style="color:orange;">according to different tasks</span>**,
CPU context switching can be divided into several different scenarios, that is,
**process context switching**, **thread context switching**, and **interrupt context switching**.

In this lesson, I will take you to see how to understand these different context
switches and why they cause CPU performance-related problems.

## Process context switch

According to the privilege level, Linux divides the running space of the process
into kernel space and user space, corresponding to Ring 0 and Ring 3 of the CPU
privilege level in the figure below.

- Kernel space (Ring 0) has the highest authority and can directly access all resources;
- User space (Ring 3) can only access restricted resources, and cannot directly access hardware devices such as memory. It must be trapped in the kernel through system calls to access these privileges resource.

<a href="https://imgbb.com/"><img src="https://i.ibb.co/pK96dZM/Linux-Process-Privilege-Level.png" alt="Linux-Process-Privilege-Level" border="0"></a>

From another perspective, that is to say, a process can run both in user space
and in kernel space. When a process runs in user space, it is called the user mode
of the process, and when it falls into the kernel space, it is called the kernel
mode of the process.

The transition from user mode to kernel mode needs to be done through **system calls**.
For example, when we view the content of a file, we need multiple system calls
to complete: first call `open()` to open the file, then call `read()` to read the
file content, and call `write()` to write the content to standard output, and
finally call `close()` to close the file.

So, has there been a CPU context switch in the process of the system call?
The answer is of course yes.

The original user mode instruction position in the CPU register needs to be saved first.
Next, in order to execute kernel-mode code, the CPU registers need to be updated
with the new locations of kernel-mode instructions. The last thing is to jump to
the kernel mode to run the kernel task.

After the system call ends, the CPU registers need to **restore** the original
saved user state, and then switch to the user space to continue running the process.
**<span style="color:orange;">Therefore, in the process of a system call, there
are actually two CPU context switches</span>**.

However, it should be noted that during the system call process, resources in
process user mode such as virtual memory will not be involved, and processes will
not be switched. This is different from what we usually call process context switching:

- Process context switching refers to switching from one process to another process.
- The same process is always running during the system call.

Therefore, **the system call process is usually called a privileged mode switch,
not a context switch**. But in fact, during the system call process, the context
switch of the CPU is still unavoidable.

So, what is the difference between a process context switch and a system call?

First of all, you need to know that processes are managed and scheduled by the kernel,
and process switching can only occur in kernel mode. Therefore, the context of
a process includes not only user space resources such as virtual memory, stacks,
and global variables, but also the state of kernel space such as kernel stacks
and registers.

Therefore, the context switch of the process is one more step than the system call:
before saving the kernel state and CPU registers of the current process,
the virtual memory, stack, etc. of the process need to be saved; and the kernel
state of the next process is loaded, after that, the virtual memory and user
stack of the process need to be refreshed.

As shown in the figure below, the process of saving and restoring context is not
"free" and requires the kernel to run on the CPU to complete.

<a href="https://ibb.co/R6VyvHf"><img src="https://i.ibb.co/n15Cgk9/Process-Context-Switching.png" alt="Process-Context-Switching" border="0"></a>

According to 🔗<span style="color:orange;">[Tsuna](https://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html)</span>'s test report,
each context switch requires tens of nanoseconds to several microseconds of CPU time.
This time is still quite considerable, especially in the case of a large number
of process context switches, it is easy to cause the CPU to spend a lot of time
on saving and restoring resources such as registers, kernel stacks, and virtual
memory, which greatly shortens the actual execution time. The time of the process.
This is exactly what we said in the previous section, an important factor that
leads to an increase in the average load.

In addition, we know that Linux manages the mapping relationship between virtual
memory and physical memory through TLB (Translation Lookaside Buffer).
When the virtual memory is updated, the TLB also needs to be refreshed, and
memory access will also slow down. Especially on a multi-processor system, the
cache is shared by multiple processors. Refreshing the cache will not only affect
the process of the current processor, but also affect the processes of other
processors that share the cache.

After knowing the potential performance problems of process context switching,
let's look at when the process context will be switched.

Obviously, the context needs to be switched only when the process is switched.
In other words, the context needs to be switched only when the process is scheduled.
Linux maintains a ready queue for each CPU, sorts active processes (that is,
processes that are running and waiting for the CPU) according to priority and
waiting time for the CPU, and then selects the process that needs the most CPU,
that is, the process with the highest priority and Wait for the process with the
longest CPU time to run.

So, when will the process be scheduled to run on the CPU?

The easiest time to think of is when the process finishes executing and terminates,
the CPU it used before will be released, and at this time, a new process will be
taken from the ready queue to run. <span style="color:orange;">In fact, there are many other scenarios that
will also trigger process scheduling. Here I will sort out them one by one for you.</span>

First, in order to ensure that all processes can be scheduled fairly, the CPU
time is divided into time slices, and these time slices are allocated to each
process in turn. In this way, when the time slice of a certain process is exhausted,
it will be suspended by the system and switched to run by other processes waiting
for the CPU.

Second, when the system resources are insufficient (such as insufficient memory),
the process cannot run until the resources are satisfied. At this time, the process
will also be suspended, and the system will schedule other processes to run.

Third, when a process actively suspends itself through methods such as the `sleep`
function, it will naturally be rescheduled.

Fourth, when a process with a higher priority is running, in order to ensure the
running of the high-priority process, the current process will be suspended and
run by the high-priority process.

The last one, when a hardware interrupt occurs, the process on the CPU will be
suspended by the interrupt and then execute the interrupt service routine in the kernel.

It is very necessary to understand these scenarios, because once the performance
problems of context switching occur, they are the murderers behind the scenes.

## Thread context switch

After talking about the context switching of the process, let's look at the thread-related issues.

The biggest difference between a thread and a process is that **a thread is the
basic unit of scheduling**, while **a process is the basic unit of resource ownership**.
To put it bluntly, the so-called task scheduling in the kernel actually schedules threads;
the process only provides resources such as virtual memory and global variables
for threads. So, for threads and processes, we can understand it like this:

- When a process has only one thread, it can be considered that the process is equal to the thread.
- When a process has multiple threads, these threads share the same resources such as virtual memory and global variables. These resources do not need to be modified during context switches.
- In addition, threads also have their own private data, such as stacks and registers, which also need to be saved during context switching.

In this way, thread context switching can actually be divided into two situations:

In the first type, the two threads before and after belong to different processes.
At this time, because resources are not shared, the switching process is the same as process context switching.

Second, the two threads before and after belong to the same process. At this time,
because the virtual memory is shared, resources such as virtual memory remain
unchanged during switching, and only data that is not shared, such as private data
and registers of threads, need to be switched.

At this point, you should also find that although it is both context switching,
thread switching within the same process consumes less resources than switching
between multiple processes, and this is also an advantage of multi-threading
instead of multi-process.

## Interrupt context switch

In addition to the previous two context switches, there is another scenario that
also switches CPU contexts, that is, interrupts.

In order to quickly respond to hardware events, **interrupt processing will interrupt
the normal scheduling and execution of the process**, and instead call the interrupt handler
to respond to device events. When interrupting other processes, it is necessary
to save the current state of the process, so that after the interruption ends,
the process can still resume operation from the original state.

Unlike process context, interrupt context switching does not involve the user
mode of the process. Therefore, even if the interrupt process interrupts a process
in user mode, there is no need to save and restore user mode resources such as
virtual memory and global variables of this process. The interrupt context
actually only includes the state necessary for the execution of the kernel mode
interrupt service program, including CPU registers, kernel stack, hardware interrupt
parameters, etc.

## Summary

To sum up, no matter what kind of scenario causes the context switch, you should know:

1. CPU context switch is one of the core functions to ensure the normal operation of the Linux system. Generally, we don’t need to pay special attention to it.
2. However, excessive context switching will consume CPU time on saving and restoring data such as registers, kernel stacks, and virtual memory, thereby shortening the real running time of the process and causing a significant drop in the overall performance of the system.

Today I will mainly introduce the working principles of these types of context switching.
In the next section, I will continue with the actual case and talk about the analysis method of the context switching problem.
