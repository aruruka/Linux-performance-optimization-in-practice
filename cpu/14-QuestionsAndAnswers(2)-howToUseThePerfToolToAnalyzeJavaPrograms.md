# 14 Q&A(2): How to use the perf tool to analyze Java programs?

Today is our second Q&A session. The topic of this Q&A session is the `perf` tool
that we have used many times. The content mainly includes various questions about
how to use `perf` in the previous cases.

`perf` is very effective in performance analysis and is a core tool that each of
us needs to master. There are also many ways to use perf, but don’t worry, you
only need to know how to use `perf record` and `perf report`. As for some kernel
symbols in the call stack displayed by `perf`, it doesn’t matter if you don’t
understand them, you can skip them temporarily and it will not affect our analysis.

Similarly, in order to make it easier for you to learn and understand, they are
not strictly arranged in the order of the articles. If you need to review the
original text of the content, you can scan the QR code at the bottom right of
each question to view it.

## Problem 1: When using the `perf` tool, I see the hexadecimal address instead of the function name

> Teacher, I have a question, I follow your document, I use perf top -g -p process ID,
> why can’t I check the call relationship of phpfpm, can’t find out which function
> uses a higher CPU? The following is my display result, please help me See why it shows 0x7fd...

This is also a problem with a lot of comments. In the CentOS system, the function
name cannot be seen using the `perf` tool, but only some function addresses in hexadecimal format.

In fact, as long as you look at the bottom line of the `perf` interface, you will
find a warning message:

```plaintext
Failed to open /opt/bitnami/php/lib/php/extensions/opcache.so, continuing without symbols
```

This shows that perf cannot find the library that the process to be analyzed
depends on. Of course, in this case, many dependent libraries cannot be found,
but the perf tool itself only displays warning information on the last line, so
you can only see this warning.

This problem is actually a problem we often encounter when analyzing Docker
container applications, because the libraries that container applications depend
on are all in the image.

In response to this situation, I have summarized the following **four solutions**.

**The first method is to build dependent libraries with the same path outside the
container.** This method is feasible in principle, but I do not recommend it. On
the one hand, it is troublesome to find out these dependent libraries. More
importantly, building these paths will pollute the environment of the container
host.

**The second method, run perf inside the container**. However, this requires the
container to run in a privileged mode, but actual applications often only run as
ordinary containers. Therefore, there is generally no permission to perform `perf`
analysis inside the container.

For example, if you run `perf record` inside a normal container, you will see
this error message:

```shell
$ perf_4.9 record -a -g
perf_event_open(..., PERF_FLAG_FD_CLOEXEC) failed with unexpected error 1 (Operation not permitted)
perf_event_open(..., 0) failed unexpectedly with error 1 (Operation not permitted)
```

Of course, you can also allow unprivileged users to perform `perf` event analysis
by configuring <ins>/proc/sys/kernel/perf_event_paranoid</ins> (for example, to -1).

But again, for the sake of safety, I don't recommend this method.

**The third method specifies the symbol path as the path of the container file system**.
For example, for the application of Lecture 05, you can execute the following command:

```shell
$ mkdir /tmp/foo
$ PID=$(docker inspect --format {{.State.Pid}} phpfpm)
$ bindfs /proc/$PID/root /tmp/foo
$ perf report --symfs /tmp/foo

# 使用完成后不要忘记解除绑定
$ umount /tmp/foo/
```

However, it should be noted here that the `bindfs` tool requires you to install
it additionally. The basic function of `bindfs` is to implement directory binding
(similar to `mount --bind`), here you need to install version 1.13.10 (this is
also its latest release).

If you installed an old version, you can download the source code from 🔗[GitHub](https://github.com/mpartel/bindfs),
then compile and install it.

**The fourth method is to save the analysis records outside the container, and
then go to the container to check the results**. In this way, the paths of libraries
and symbols are correct.

For example, you can do this. Run `perf record -g -p <pid>` first, and after a
while (for example, 15 seconds), press Ctrl+C to stop.

Then, copy the generated perf.data file into the container for analysis:

```shell
$ docker cp perf.data phpfpm:/tmp
$ docker exec -i -t phpfpm bash
```

Next, proceed to run the following command in the container's bash to install
`perf` and view the report with `perf report`:

```shell
$ cd /tmp/ 
$ apt-get update && apt-get install -y linux-tools linux-perf procps
$ perf_4.9 report
```

However, there are two points here that require your attention.

The first is the version of the perf tool. In the last step, the tool we run is
the version `perf_4.9` installed inside the container instead of the normal `perf`
command. This is because the `perf` command is actually a soft link that matches
the kernel version, but the `perf` version installed in the image may not be
consistent with the kernel version of the virtual machine.

In addition, the php-fpm image is based on the Debian system, so the command to
install the `perf` tool is not exactly the same as Ubuntu. For example, the
installation method on Ubuntu is as follows:

```shell
$ apt-get install -y linux-tools-common linux-tools-generic linux-tools-$(uname -r)）
```

And in the php-fpm container, you should execute the following command to install `perf`:

```shell
$ apt-get install -y linux-perf
```

After you follow the previous methods, you can see the `sqrt` stack inside the container:

<center><a href="https://ibb.co/17bdZnx">
    <img src="https://i.ibb.co/5G2nT9V/Use-Perf-Inside-Container.png" alt="Use-Perf-Inside-Container" border="0">
</a></center>

In fact, in our case aside, you might run into this problem even in non-containerized
applications. If your application uses `strip` to delete the symbol table of the
ELF binary file when compiling, then you can only see the address of the function.

The current disk space is actually large enough. Keeping these symbols, although
it will cause the compiled file to become larger, is not a big problem for the
entire disk space. So for the convenience of debugging, it is recommended that
you keep them.

By the way, the installation methods of various tools in the case can be regarded
as the basic skills of our column study, which I hope you can be familiar with
and master. Still the same sentence, if you don’t know how to install, check the
documentation first, if it still doesn’t work, search the Internet or leave a
message in the article to ask questions.

I also want to praise here, many students have already shared the methods they
have explored in the message. Recording and sharing is a good habit.

## Question 2: How to analyze Java program with perf tool

> I don't know if perl can locate the java function called by the java dynamic bytecode.
> It seems that the `jstack` is used to break the dynamic bytecode.

> Teacher, I am a java application here. I used topjstack to trace the code and
> found that the cpu level occurred in a class generated by dynamic bytecode.
> Then this piece has no source code, and it is caused by the call of a function
> that I don’t know. , and then use perf top -v to find this application, kernal_spin_lock,
> which takes up 80% of the cpu for a long time, I have nothing to do now, because
> I don’t know that program in the application uses a spin lock. Is there any
> way to track the entire call chain? Woolen cloth?

These two problems are actually an extension of the previous `perf` problem.
For applications like Java that run through the JVM, the running stack uses the
built-in functions and stack management of the JVM. Therefore, from the system
level, you can only see the function stack of the JVM, but not the stack of the
Java application directly.

perf_events actually supports JIT, but also needs a <ins>/tmp/perf-PID.map</ins>
file for symbol translation. Of course, the open source project 🔗[perf-map-agent](https://github.com/jvm-profiling-tools/perf-map-agent)
can help you generate this symbol table.

In addition, in order to generate the entire call stack, you also need to enable
the JDK option "-XX:+PreserveFramePointer". Because a lot of Java knowledge is
involved here, I won't expand it in detail. If your application happens to be
based on Java, then you can refer to Netflix's technical blog 🔗[Java in Flames](https://netflixtechblog.com/java-in-flames-e763b3d32166)
to view detailed usage steps.

Speaking of this, I also want to emphasize a problem, that is, when learning
performance optimization, don't limit yourself to a specific programming language
or performance tool from the beginning, and get entangled in the details of the
language or tool.

Mastering the overall analysis thinking is the first thing we need to do. Because
the principles and ideas of performance optimization are the same in any programming
language.

## Question 3: Why do many symbols not display the call stack in the perf report

Teacher, I used the `perf record -g` command to collect information. It has been
collected many times, and each time the command is listed as swapper with a plus
sign on the left, but for app or others, there is no plus sign. But you can see
sys_read, new_sync_read, blkdev_read_iter and other information in the symbol column.
My system also uses Ubuntu 18.04. I don't know why your implementation scenario
cannot be restored.

`perf report` is a tool to visualize perf.data. In the case of Lecture 08, I gave
the final result directly without detailing its parameters. It is estimated that
many students have encountered the same problem as passing students when their
machines are running, and what they see is the following interface.

<center><a href="https://ibb.co/4jjB1Yz">
    <img src="https://i.ibb.co/THHzWb6/Perf-Wrong-Screenshot.png" alt="Perf-Wrong-Screenshot" border="0">
</a></center>

This interface can clearly see that in the output of `perf report`, only the
swapper shows the call stack, and all other symbols cannot view the stack, including
the app application in our case.

We have encountered this situation before. What should you do when you find that
the output of performance tools cannot be understood? Of course, check the manual
of the tool. For example, you can execute the `man perf-report` command to find
the description of the "-g" parameter:

```plaintext
-g, --call-graph=<print_type,threshold[,print_limit],order,sort_key[,branch],value> 
           Display call chains using type, min percent threshold, print limit, call order, sort key, optional branch and value. Note that 
           ordering is not fixed so any parameter can be given in an arbitrary order. One exception is the print_limit which should be 
           preceded by threshold. 

               print_type can be either: 
               - flat: single column, linear exposure of call chains. 
               - graph: use a graph tree, displaying absolute overhead rates. (default) 
               - fractal: like graph, but displays relative rates. Each branch of 
                        the tree is considered as a new profiled object. 
               - folded: call chains are displayed in a line, separated by semicolons 
               - none: disable call chain display. 

               threshold is a percentage value which specifies a minimum percent to be 
               included in the output call graph.  Default is 0.5 (%). 

               print_limit is only applied when stdio interface is used.  It's to limit 
               number of call graph entries in a single hist entry.  Note that it needs 
               to be given after threshold (but not necessarily consecutive). 
               Default is 0 (unlimited). 

               order can be either: 
               - callee: callee based call graph. 
               - caller: inverted caller based call graph. 
               Default is 'caller' when --children is used, otherwise 'callee'. 

               sort_key can be: 
               - function: compare on functions (default) 
               - address: compare on individual code addresses 
               - srcline: compare on source filename and line number 

               branch can be: 
               - branch: include last branch information in callgraph when available. 
                         Usually more convenient to use --branch-history for this. 

               value can be: 
               - percent: diplay overhead percent (default) 
               - period: display event period 
               - count: display event count
```

Through this description, we can see that the -g option is equivalent to --call-graph,
and its parameters are the following options separated by commas, meaning output
type, minimum threshold, output limit, sorting method, sorting keywords, branches
and types of values.

We can see that the default parameters here are "graph, 0.5, caller, function,
and percent". The specific meanings are explained in detail in the documentation,
so I won’t repeat them here.

Now looking back at our problem, the stack is not fully displayed, and the relevant
parameter is of course the minimum threshold. Through the description of threshold
in the manual, we know that when the proportion of an event is higher than this
threshold, its call stack will be displayed.

The default value of threshold is 0.5%, that is to say, the call stack can only
be displayed when the proportion of events exceeds 0.5%. Looking at the event
ratio of our case application app, it is only 0.34%, which is lower than 0.5%,
so it is normal not to see the call stack of the app.

In this case, you only need to set a threshold of less than 0.34% for `perf report`
to display the call graph we want to see. For example, execute the following command:

```shell
$ perf report -g graph,0.3
```

You can get the following new output interface. After expanding the app, you can
see its call stack.

<center><a href="https://ibb.co/b70B5yV">
    <img src="https://i.ibb.co/SJjnw1g/Perf-Correct-Screenshot.png" alt="Perf-Correct-Screenshot" border="0">
</a></center>

## Question 4: How to understand the perf report report

> In the analysis details carried out by perf, there is a swapper process with
> more than 90% of children and self. If I see this, I will definitely start
> around this process first, and ignore the app process that occupies only 0.6%.
> In this case, is there any deviation in my understanding?

> Hello teacher, I have two questions about perf:
> 1. When adding -g to perf top, what do the first two columns of children andself indicate? What does the percentage after pressing the Enter key indicate? I don’t understand it after studying the man manual.
> 2. In addition, I am doing distributed storage. When using perf for performance analysis, do I need to consider the impact of perf itself on program performance?

Seeing this, I guess you have also muttered, why not use the `perf` tool to solve
it as soon as you come up, and execute so many other tools? This question actually
gives a good explanation.

In the `perf report` interface of question 3, you must have noticed that the
`swapper` ratio is as high as 99%. Intuitively, we should observe it directly,
why not?

In fact, when you understand the principle of `swapper`, it is easy to understand
why we can ignore it.

When you see swapper, the first thing you may think of is the SWAP partition.
In fact, swapper has nothing to do with SWAP, it only creates the `init` process
when the system is initialized, after that, it becomes a lowest priority idle task.
That is, the `swapper` is executed when no other tasks are running on the CPU.
So, you can call it an "idle task".

Back to our question, expand its call stack in the `perf report` interface, and
you will see that the `swapper` clock events are all spent on do_idle, that is,
executing idle tasks.

<center><a href="https://ibb.co/8mR7695">
    <img src="https://i.ibb.co/9pfrsWT/Perf-Swapper.png" alt="Perf-Swapper" border="0">
</a></center>

Therefore, when analyzing the case, we directly ignored the 99% symbol in the front,
and instead analyzed the app with only 0.3% in the back. In fact, you can also
understand from here why we didn't use `perf` analysis first.

Because in a multitasking system, events with a large number of events are not
necessarily performance bottlenecks. Therefore, only observing a large value does
not explain anything. Whether there is a bottleneck specifically, you need to
observe multiple indicators in multiple aspects to cross-validate. This is also
a point I keep emphasizing in the routine articles.

In addition, regarding the meaning of Children and Self, there are actually detailed
descriptions in the manual, and a friendly example is given to illustrate the
calculation method of their percentages. simply put,

- Self is the proportion of the symbol in the last column (which can be understood as a function);
- Children is the sum of proportions occupied by other symbols called by this symbol (which can be understood as sub-functions, including direct and indirect calls).

As the classmates asked in the message, many performance tools do have a certain
impact on system performance. Take `perf` as an example, it needs to track various
events of the kernel stack in the kernel, so it will inevitably bring a certain
performance loss. This, although for most applications, does not have much impact,
but for some specific applications (such as those that are particularly sensitive
to clock cycles), it may be a disaster.

Therefore, when using performance tools, you should really consider the impact
of the tool itself on system performance. In this case, you need to understand
the principles of these tools. for example,

- `perf`, a dynamic tracking tool, will bring a certain performance loss to the system.
- Tools such as `vmstat` and `pidstat` that directly read the proc file system to obtain indicators will not cause performance loss.

## Question 5: Performance optimization book and reference recommendations

> Teacher, do you provide some reference materials and books?

I am very happy to see that there is such a high enthusiasm for learning in the comments.
In fact, there are a lot of comments behind many articles. I hope I can recommend
books and learning materials. This is what I would like to see too. Column learning
must not be the whole of your performance optimization journey. I am very happy
to be able to get you started, help you solve practical problems, and even
stimulate your enthusiasm for learning.

In the article on 🔗[how to learn Linux performance optimization](https://time.geekbang.org/column/article/69346), I once
introduced Brendan Gregg, who is a well-deserved master of performance optimization.
You can basically see his performance tool map in various Linux performance
optimization articles.

Therefore, my favorite book about performance optimization is actually the book
"Systems Performance: Enterprise and the Cloud" written by him. This book has also
been published in Chinese, the name is 《性能之巅：洞悉系统、企业与云计算》.

Judging from the publication time, this book is indeed an old book, and the English
version was published in 2013. But the reason why classics become classics is
precisely because they will not become obsolete. The performance analysis ideas
and many performance tools in this book are still applicable today.

In addition, I also recommend you to pay attention to his personal website
🔗http://www.brendangregg.com/, especially the Linux Performance page, which
contains a lot of information about 🔗[Linux performance optimization](https://hpeindia.udemy.com/course/go-the-complete-developers-guide/learn/lecture/7797236#overview), such as:

- Linux performance tool map;
- Performance analysis reference materials;
- Lecture videos optimized for performance.

However, a lot of content here will involve a lot of kernel knowledge, which is
not friendly to beginners. However, if you want to become a master, hard work
and persistence are inevitable. Therefore, I hope that when you check these
materials, you will not retreat as soon as you encounter something you don’t
understand. It's normal to have something you don't understand in the first pass
of learning anything. Hold back the fear and don't give up. Keep going. Many
problems in the front may be solved together, and it will be easier to read it
the second or third time.

Still the same sentence, grasp the main line without wavering, start with the
most basic principles, master the ideas of performance analysis, and then gradually
deepen, explore the details, and don't try to eat a big fat man.

Finally, welcome to continue to write your questions in the message area, and I
will continue to answer them. My purpose remains the same. I hope that I can work
with you to turn the knowledge of the article into your ability. We not only
practice in actual combat, but also improve in communication.