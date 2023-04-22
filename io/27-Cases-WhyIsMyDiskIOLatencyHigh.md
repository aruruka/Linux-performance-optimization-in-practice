# Cases: Why is my disk I/O latency high?

In the last section, we looked at a case where wildly typed logs caused I/O performance problems, so let's start with a brief review.

Logging, the most common and effective tool for understanding the inner workings of an application. Logs are generally categorized into debugging, information, warnings, errors, and many other different levels.

Typically, production environments only use logs with the warning level turned on, which generally does not cause I/O problems. However, occasionally, when troubleshooting problems, we may need to turn on debug logging. After debugging, it is likely that we forget to turn back the logging level. At that point, a large amount of debug logging can cause I/O performance problems.

You can use `iostat` to confirm if there is an I/O performance bottleneck. Then use `strace` and `lsof` to locate the application and the path to the log file it is writing to. Finally, the I/O problem is perfectly solved by adjusting the logging level through the application's interface.

However, if the application does not have the ability to dynamically adjust the log level, you will also need to modify the application configuration and restart the application in order for the configuration to take effect.

Today, let's look at a new case. This time it's a web application based on the Python Flask framework, which provides an API for querying word hotness, but the API's response time is not satisfactory.

Many thanks to Guoxing Dong, a senior backend engineer in Ctrip's system development department, for helping to provide today's case.

## Case preparation

This case is also based on Ubuntu 18.04, which is also applicable to other Linux systems. The case environment I used is shown below:

- Machine configuration: 2 CPU, 8GB RAM
- Pre-install `docker`, `sysstat` and other tools like `apt install docker.io sysstat`

To make it easier for you to run today's case, I've packaged it as a Docker image. This way, you just need to run the Docker command to start it.

Today's case requires two virtual machines, one of which is the target machine for the case analysis, running the Flask application, which has an IP address of 192.168.0.10; and the other serves as the client, requesting the word hotness. I drew a diagram to represent their relationship, as follows:

<center><a href="https://ibb.co/Chrc5kX">
    <img src="https://i.ibb.co/3BJ3yD8/27-case-diagram.png" alt="27-case-diagram" border="0">
</a></center>

Next, open two terminals, SSH into each of the two virtual machines, and in the first virtual machine, install the above tools.

As before, all commands in this case are run as root by default. If you are logged in as a normal user, run `sudo su root` to switch to the root user.

Here, the preparation work is done. Next, let's get into the action.

> Tip: The core logic of the Python application in this case is relatively simple, so you may be able to see the problem at a glance, but the source code in a real production environment is much more complex.
> So, I still recommend that you don't look at the source code before operating to avoid preconceptions, but rather analyze it as a black box.
> This way, you can get a better grasp of how to analyze the application where the bottleneck is located and the approximate location of the bottleneck in the application, starting from the system's resource usage problem.

## Case analysis

First, we run the target application to be analyzed in this case by executing the following command in the first terminal:

```shell
$ docker run --name=app -p 10000:80 -itd feisky/word-pop 
```

Then, run the curl command in a second terminal, visit 🔗<http://192.168.0.10:1000/>, and confirm that the case starts properly. You should see a hello world in the output screen of `curl` as follows:

```shell
$ curl http://192.168.0.10:10000/ 
hello world 
```

Next, in the second terminal, access the word heat interface of the case application, which is 🔗<http://192.168.0.10:1000/popularity/word>.

```shell
$ curl http://192.168.0.10:1000/popularity/word 
```

Wait a little while, you will find that the interface actually did not respond for so long, what is going on? We go back to the terminal first to analyze it.

We try in the first terminal, execute a random command, such as the execution of the `df` command, check the file system usage. Strangely enough, such a simple command, but also take a long time to wait for the output.

```shell
$ df 
Filesystem     1K-blocks    Used Available Use% Mounted on 
udev             4073376       0   4073376   0% /dev 
tmpfs             816932    1188    815744   1% /run 
/dev/sda1       30308240 8713640  21578216  29% / 
```

By `df` we know that the system still has enough disk space. So why is the response slowing down? It seems to be necessary to observe the system's resource usage, such as CPU, memory and disk I/O.

The idea here is actually similar to the previous case, we can first use `top` to observe the CPU and memory usage, and then use `iostat` to observe the disk I/O situation.

In order to avoid the `curl` request to end abruptly during the analysis, we go back to Terminal 2 and press Ctrl+C to stop the application; then, we put the `curl` command into a loop; this time, we also add a `time` command to observe the execution time of each time:

```shell
$ while true; do time curl http://192.168.0.10:10000/popularity/word; sleep 1; done
```

Let's go back to Terminal 1 to analyze the performance. Let's run the `top` command in Terminal 1 to observe CPU and memory usage:

```shell
$ top 
top - 14:27:02 up 10:30,  1 user,  load average: 1.82, 1.26, 0.76 
Tasks: 129 total,   1 running,  74 sleeping,   0 stopped,   0 zombie 
%Cpu0  :  3.5 us,  2.1 sy,  0.0 ni,  0.0 id, 94.4 wa,  0.0 hi,  0.0 si,  0.0 st 
%Cpu1  :  2.4 us,  0.7 sy,  0.0 ni, 70.4 id, 26.5 wa,  0.0 hi,  0.0 si,  0.0 st 
KiB Mem :  8169300 total,  3323248 free,   436748 used,  4409304 buff/cache 
KiB Swap:        0 total,        0 free,        0 used.  7412556 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND 
12280 root      20   0  103304  28824   7276 S  14.0  0.4   0:08.77 python 
   16 root      20   0       0      0      0 S   0.3  0.0   0:09.22 ksoftirqd/1 
1549 root      20   0  236712  24480   9864 S   0.3  0.3   3:31.38 python3
```

Looking at the output of `top`, we can see that the iowait is very high for both CPUs. CPU0 in particular has an iowait of 94%, and with 3GB of memory remaining, it looks like it has plenty.

Looking further down, the process section has a python process with a slightly higher CPU usage of 14%. While 14% is not a performance bottleneck, it is a bit suspicious - it may be related to the increase in iowait.

So is this python process, with PID 12280, our case application or not?

In our first terminal, we stop the `top` command by pressing Ctrl+C; then we execute the following `ps` command to find the PID number of our case app.py:

```shell
$ ps aux | grep app.py 
root     12222  0.4  0.2  96064 23452 pts/0    Ss+  14:37   0:00 python /app.py 
root     12280 13.9  0.3 102424 27904 pts/0    Sl+  14:37   0:09 /usr/local/bin/python /app.py 
```

From the output of `ps`, you can see that the process with the high CPU usage is our case application. But don't rush to analyze the CPU problem, after all, the iowait is already 94%, and the I/O problem is the first thing we need to solve.

Next, in Terminal 1, we run the following `iostat` command, where the .

- The -d option shows the I/O performance metrics;
- the -x option shows the extended statistics (i.e., all I/O metrics are shown).

```shell
$ iostat -d -x 1
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util 
loop0            0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sda              0.00   71.00      0.00  32912.00     0.00     0.00   0.00   0.00    0.00 18118.31 241.89     0.00   463.55  13.86  98.40
```

When you see the output of `iostat` again, do you remember the meaning of the performance metrics in this interface? If you can't remember, make sure to check the previous section or use `man iostat` to find out.

Once you understand what the metrics mean, look at the output of `iostat`. You can see that the I/O usage of disk sda has reached 98%, which is close to saturation. Moreover, the response time for write requests is up to 18 seconds, and the write data is 32 MB per second, so it is clear that the write disk has hit a bottleneck.

How do you know which processes are responsible for these I/O requests? I think you already remember the `pidstat` we used in the previous section.

In Terminal 1, run the following `pidstat` command to see the I/O status of the processes:

```shell
$ pidstat -d 1 
14:39:14      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command 
14:39:15        0     12280      0.00 335716.00      0.00       0  python
```

From the output of `pidstat`, we again see the result of PID number 12280. This means that it is the case application that is causing the I/O performance bottleneck.

At this point, you probably think that the next step is very simple, didn't you just learn it in the last case? Just use `strace` to check if it is writing a file, and then use `lsof` to find the file descriptor corresponding to it.

Is this the case or not? Let's try it. Again, in Terminal 1, execute the following `strace` command:

```shell
$ strace -p 12280 
strace: Process 12280 attached 
select(0, NULL, NULL, NULL, {tv_sec=0, tv_usec=567708}) = 0 (Timeout) 
stat("/usr/local/lib/python3.7/importlib/_bootstrap.py", {st_mode=S_IFREG|0644, st_size=39278, ...}) = 0 
stat("/usr/local/lib/python3.7/importlib/_bootstrap.py", {st_mode=S_IFREG|0644, st_size=39278, ...}) = 0
```

From `strace`, you can see a lot of stat system calls, mostly python files, but note that there aren't any write system calls.

Since `strace` has a lot of output, we can use `grep`, to filter the write, e.g:

```shell
$ strace -p 12280 2>&1 | grep write 
```

Unfortunately, there is still no output here.

Are there no more performance problems at this point? Re-execute the `top` and `iostat` commands just now and you will unfortunately find that the performance problem still exists.

We have to combine the results of `strace`, `pidstat` and `iostat` to analyze them. Obviously, you should have noticed a contradiction here: `iostat` has proven that there is a performance bottleneck in disk I/O, and `pidstat` has proven that this bottleneck is caused by process 12280, but `strace` traces this process and does not find any write system calls.

That's strange. Is it because the case uses Python, which is an interpreted language, so it can't be found? Or is it because the case is running in Docker? Here's an open question, think about it.

If the file is written, there should be a corresponding write system call, but you can't find any trace of it with the existing tools, so it's time to think about changing tools. How can I know where the file is being written?

Here I'll introduce you to a new tool, `filetop`. It is part of the bcc package and is based on the Linux kernel's eBPF (extended Berkeley Packet Filters) mechanism, which tracks the reading and writing of files in the kernel and outputs the thread ID (TID), read and write sizes, read and write types, and file names.

The eBPF works, you don't have to look deeper for now, we will gradually get to the content later, just use it first.

As for the installation of the old friend bcc, you can refer to its Github site https://github.com/iovisor/bcc. For example, on Ubuntu 16 and above, you can run the following command to install it:

```shell
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 4052245BD4284CDD 
echo "deb https://repo.iovisor.org/apt/$(lsb_release -cs) $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/iovisor.list 
sudo apt-get update 
sudo apt-get install bcc-tools libbcc-examples linux-headers-$(uname -r)
```

Once installed, all the tools provided by bcc are installed in the <ins>/usr/share/bcc/tools</ins> directory. Next, we will use this tool to observe how the files are read and written.

First, run the following command in Terminal 1:

```shell
# 切换到工具目录 
$ cd /usr/share/bcc/tools 

# -C 选项表示输出新内容时不清空屏幕 
$ ./filetop -C 

TID    COMM             READS  WRITES R_Kb    W_Kb    T FILE 
514    python           0      1      0       2832    R 669.txt 
514    python           0      1      0       2490    R 667.txt 
514    python           0      1      0       2685    R 671.txt 
514    python           0      1      0       2392    R 670.txt 
514    python           0      1      0       2050    R 672.txt 

...

TID    COMM             READS  WRITES R_Kb    W_Kb    T FILE 
514    python           2      0      5957    0       R 651.txt 
514    python           2      0      5371    0       R 112.txt 
514    python           2      0      4785    0       R 861.txt 
514    python           2      0      4736    0       R 213.txt 
514    python           2      0      4443    0       R 45.txt 
```

You will see that `filetop` outputs 8 columns, which are the thread ID, the thread command line, the number of reads and writes, the size (in KB) of the reads and writes, the file type, and the name of the file read and written.

You'll probably see a lot of dynamic link libraries in these, but that's not our focus, so we'll ignore them for now. Our focus is on a python application, so pay special attention to the python related content.

If you look around for a while, you'll see that every once in a while, the python application with thread number 514 will first write a lot of txt files, and then read a lot of them.

Which process does the thread number 514 belong to? We can check it with the `ps` command. First, in Terminal 1, press Ctrl+C to stop `filetop`; then, run the following `ps` command. The second column of this output is the process number we want to know:

```shell
$ ps -efT | grep 514
root     12280  514 14626 33 14:47 pts/0    00:00:05 /usr/local/bin/python /app.py 
```

As we can see, this thread is the same as the case application 12280. Finally, we can breathe a sigh of relief, but it's not over yet, `filetop` only gives the file name, but not the file path, so we still have to keep looking.

I'll introduce another good tool, `opensnoop`. It is part of the bcc package and can dynamically track open system calls in the kernel. This way, we can find the paths to these txt files.

Next, in Terminal 1, run the following `opensnoop` command:

```shell
$ opensnoop 
12280  python              6   0 /tmp/9046db9e-fe25-11e8-b13f-0242ac110002/650.txt 
12280  python              6   0 /tmp/9046db9e-fe25-11e8-b13f-0242ac110002/651.txt 
12280  python              6   0 /tmp/9046db9e-fe25-11e8-b13f-0242ac110002/652.txt 
```

This time, with the output of `opensnoop`, you can see that the txt paths are located in the <ins>/tmp</ins> directory. You can also see that it opens a number of files, numbered numerically, from 0.txt to 999.txt, which is much more than what you saw earlier with `filetop`.

Combining `filetop` and `opensnoop`, we can analyze further. We can hazard a guess that the case application writes 1000 txt files and then reads them into memory for processing. Let's check if there are really 1000 files in this directory:

```shell
$ ls /tmp/9046db9e-fe25-11e8-b13f-0242ac110002 | wc -l 
ls: cannot access '/tmp/9046db9e-fe25-11e8-b13f-0242ac110002': No such file or directory 
0
```

After the operation, I found that the directory does not exist anymore. What happened? Let's go back to `opensnoop` and observe for a while:

```shell
$ opensnoop 
12280  python              6   0 /tmp/defee970-fe25-11e8-b13f-0242ac110002/261.txt 
12280  python              6   0 /tmp/defee970-fe25-11e8-b13f-0242ac110002/840.txt 
12280  python              6   0 /tmp/defee970-fe25-11e8-b13f-0242ac110002/136.txt 
```

It turns out that the path has changed to another directory at this point. This shows that these directories are dynamically generated by the application and deleted when they are used up.

Combined with all the previous analysis, we can basically determine that the case application dynamically generates a batch of files for temporary data storage and deletes them when it runs out. But unfortunately, it is these file reads and writes that trigger the I/O performance bottleneck, causing the whole process to be very slow.

Of course, we still need to verify this conjecture. The old way, or looking at the application's source code, 🔗[app.py](https://github.com/feiskyer/linux-perf-examples/blob/master/io-latency/app.py)

```python
@app.route("/popularity/<word>") 
def word_popularity(word): 
  dir_path = '/tmp/{}'.format(uuid.uuid1()) 
  count = 0 
  sample_size = 1000 
   
  def save_to_file(file_name, content): 
    with open(file_name, 'w') as f: 
      f.write(content) 

  try: 
    # initial directory firstly 
    os.mkdir(dir_path) 

    # save article to files 
    for i in range(sample_size): 
        file_name = '{}/{}.txt'.format(dir_path, i) 
        article = generate_article() 
        save_to_file(file_name, article) 

    # count word popularity 
    for root, dirs, files in os.walk(dir_path): 
        for file_name in files: 
            with open('{}/{}'.format(dir_path, file_name)) as f: 
                if validate(word, f.read()): 
                    count += 1 
    finally: 
        # clean files 
        shutil.rmtree(dir_path, ignore_errors=True) 

    return jsonify({'popularity': count / sample_size * 100, 'word': word}) 
```

As you can see in the source code, this case application, during the processing of each request, generates a batch of temporary files, which are then read into memory for processing, and finally the entire directory is deleted.

This is a common technique of using disk space to process large amounts of data, however, the I/O requests in this case are too heavy, resulting in excessive disk I/O utilization.

To solve this, it's really a matter of algorithmic optimization. For example, when there is enough memory, you can put all the data into memory to process it, so that you can avoid I/O performance problems.

You can check this by visiting 🔗<http://192.168.0.10:10000/popularity/word> and 🔗<http://192.168.0.10:10000/popular/word> separately in Terminal 2 and comparing before and after effects:

```shell
$ time curl http://192.168.0.10:10000/popularity/word
{ 
  "popularity": 0.0, 
  "word": "word" 
} 
real    2m43.172s 
user    0m0.004s 
sys    0m0.007s
```

```shell
$ time curl http://192.168.0.10:10000/popular/word
{
  "popularity": 0.0,
  "word": "word"
}

real    0m8.810s
user    0m0.010s
sys    0m0.000s
```

The new interface takes only 8 seconds to return, which is significantly better than the 3 minutes it took at the beginning.

Of course, this is only the first step of optimization, and the method is not perfect, and further optimization can be done. However, in the actual system, we mostly do a similar approach, using the simplest method first, solving the online problem as early as possible, and then moving on to think of better optimization methods.

## Summary

Today, we analyzed a case of word hotness that was too slow to respond.

First, we used `top`, `iostat`, to analyze the CPU and disk usage of the system. We found the disk I/O bottleneck and also knew that this bottleneck was caused by the case application.

Then, we tried to copy the method from the previous section and used `strace` to observe the process system calls, but this time we were unlucky and did not find any write system calls.

We then used a new tool, `filetop` and `opensnoop` in the dynamic tracing toolkit bcc, to identify the problem in the case application and found that the root cause was a large number of read and write temporary files.

Once the problem was identified, the optimization method was relatively simple. If you have enough memory, the easiest way is to put all the data in faster memory so that there is no disk I/O bottleneck. Of course, you can go a step further and use various algorithms such as Trie trees to further optimize the efficiency of word processing.