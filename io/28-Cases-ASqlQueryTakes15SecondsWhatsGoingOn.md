# 28 Cases: A SQL query takes 15 seconds, what's going on?

In the previous section, we analyzed a case of a word-hot application that was responding too slowly. After analyzing the system CPU and disk I/O usage with `top` and `iostat`, we found that the system had a disk I/O bottleneck and that it was the case application that was causing it.

Then, after using `strace` and not finding anything, I introduced you to two new tools, `filetop` and `opensnoop`, to analyze their traces of the system calls write() and open().

We found that the case application was reading and writing a large number of temporary files, thus creating a performance bottleneck. After identifying the bottleneck, we then solved the disk I/O performance problem by putting all the file data in memory.

Of course, you may say that in real-world applications, a large amount of data must be stored in a database and not directly in a text file. However, databases are not infallible. When a database has performance problems, and how to analyze and locate its bottlenecks?

Today we'll analyze a database case together. This is a Python Flask-based product search application where the product information is stored in MySQL. This application can query the product information in the database table based on the product name provided by the client through the MySQL interface.

Many thanks to Yang Xiangyi, a senior operations engineer at Vipshop, for helping to provide today's case.

## Case preparation

This case is also based on Ubuntu 18.04, which is also applicable to other Linux systems. The case environment I used is shown below:

- Machine configuration: 2 CPU, 8GB memory
- 预先安装 `docker`、`sysstat` 、`git`、`make` 等工具，如 `apt install docker.io sysstat make git`?

Among them, `docker` and `sysstat` have been used many times, so I won't repeat them here; `git` is used to pull the scripts needed for this case, which are stored in the Github code repository; finally, `make` is a common build tool, which is used here to run today's case.

The case consists of a total of three containers, including a MySQL database application, a product search application, and a data processing application. Among them, the product search application provides an interface in the form of HTTP:

- <ins>/</ins>: returns the Index Page;
- <ins>/db/insert/products/</ins>: inserts information about a specified number of products;
- <ins>/products/</ins>: Query information about the specified product and return the processing time.

Since there are more applications, to make it easier for you to run them, I've packaged the same into a few Docker images and pushed them to Github. This way, you only need to run a few commands and you're ready to launch.

Today's case requires two virtual machines, one of which serves as the target machine for the case study, running the Flask application, which has an IP address of 192.168.0.10; the other serves as the client, requesting the hotness of words. I drew a diagram to represent their relationship.

<center><a href="https://ibb.co/k5z306B">
    <img src="https://i.ibb.co/Xj6ypkL/28-case-diagram.png" alt="28-case-diagram" border="0">
</a></center>

Next, open two terminals, SSH into each of the two virtual machines, and install the above tools in the first virtual machine.

As before, all commands in this case are run as root by default. If you are logged in as a normal user, run the `sudo su root` command to switch to the root user.

Here, the preparation work is done. Next, let's get into the action.

## Case analysis

First, we execute the following command in the first terminal to pull the script needed for this case:

```shell
$ git clone https://github.com/feiskyer/linux-perf-examples
$ cd linux-perf-examples/mysql-slow
```

Next, execute the following command to run the target application for this time. Under normal circumstances, you should see the following output:

```shell
# 注意下面的随机字符串是容器ID，每次运行均会不同，并且你不需要关注它，因为我们只会用到名字
$ make run
docker run --name=mysql -itd -p 10000:80 -m 800m feisky/mysql:5.6
WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.
4156780da5be0b9026bcf27a3fa56abc15b8408e358fa327f472bcc5add4453f
docker run --name=dataservice -itd --privileged feisky/mysql-dataservice
f724d0816d7e47c0b2b1ff701e9a39239cb9b5ce70f597764c793b68131122bb
docker run --name=app --network=container:mysql -itd feisky/mysql-slow
81d3392ba25bb8436f6151662a13ff6182b6bc6f2a559fc2e9d873cd07224ab6
```

Then, run the docker ps command again to confirm that all three containers are in the running (Up) state:

```shell
$ docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                             NAMES
9a4e3c580963        feisky/mysql-slow          "python /app.py"         42 seconds ago      Up 36 seconds                                         app
2a47aab18082        feisky/mysql-dataservice   "python /dataservice…"   46 seconds ago      Up 41 seconds                                         dataservice
4c3ff7b24748        feisky/mysql:5.6           "docker-entrypoint.s…"   47 seconds ago      Up 46 seconds       3306/tcp, 0.0.0.0:10000->80/tcp   mysql
```

The MySQL database startup process requires some initialization work, which usually takes a few minutes. You can run the `docker logs` command to see how it starts up.

When you see the following output, MySQL has finished initializing and is ready to receive external requests:

And the product search application is listening on port 10000. You can stop the `docker logs` command by pressing Ctrl+C; then, run the following command to make sure it is also running properly. If everything is fine, you will see the output of the Index Page:

```shell
$ curl http://127.0.0.1:10000/
Index Page
```

Next, run the `make init` command to initialize the database and insert 10,000 items of information. This process is slow, for example, on my machine it took more than ten minutes. After waiting patiently for a while, you will see the following output:

```shell
$ make init
docker exec -i mysql mysql -uroot -P3306 < tables.sql
curl http://127.0.0.1:10000/db/insert/products/10000
insert 10000 lines
```

Next, let's switch to the second terminal and access the product search interface to see if we can find the desired product. Execute the following `curl` command:

```shell
$ curl http://192.168.0.10:10000/products/geektime
Got data: () in 15.364538192749023 sec
```

After a short wait, you will find that the interface returns empty data and takes more than 15 seconds to process. Such a slow response time is unbearable, so what went wrong?

Since you're using MySQL today, you'll probably guess that it's a slow query problem.

But don't worry, before the specific analysis, we put the `curl` command into a loop in order to avoid the client's request ending during the analysis. Also, to avoid overstressing the system, we set it to wait for 5 seconds after each query before starting a new request.

So, you can go ahead and execute the following command in Terminal 2:

```shell
$ while true; do curl http://192.168.0.10:10000/products/geektime; sleep 5; done
```

Next, go back to Terminal 1 and analyze the reasons for the slow response of the interface. However, after reverting to Terminal 1, you will find that the system response has also become significantly slower, and you will have to pause for a while before you can see the output when executing a random command.

This is similar to the phenomenon in the previous section, so it seems that we still need to observe the system's resource usage, such as CPU, memory, and disk I/O.

First, let's run the `top` command in Terminal 1 to analyze the system's CPU usage:

```shell
$ top
top - 12:02:15 up 6 days,  8:05,  1 user,  load average: 0.66, 0.72, 0.59
Tasks: 137 total,   1 running,  81 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.7 us,  1.3 sy,  0.0 ni, 35.9 id, 62.1 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.3 us,  0.7 sy,  0.0 ni, 84.7 id, 14.3 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  8169300 total,  7238472 free,   546132 used,   384696 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7316952 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
27458 999       20   0  833852  57968  13176 S   1.7  0.7   0:12.40 mysqld
27617 root      20   0   24348   9216   4692 S   1.0  0.1   0:04.40 python
 1549 root      20   0  236716  24568   9864 S   0.3  0.3  51:46.57 python3
22421 root      20   0       0      0      0 I   0.3  0.0   0:01.16 kworker/u
```

Looking at the output of `top`, we see that the iowait is relatively high for both CPUs, especially for CPU0, where the iowait is already over 60%. For each process, the CPU usage rate is not high, the highest is only 1.7%.

Since CPU is not a big suspect, the problem should be in I/O. Let's stop the `top` command by pressing Ctrl+C in the first terminal; then, execute the following `iostat` command to see if there are any I/O performance problems:

```shell
$ iostat -d -x 1
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
...
sda            273.00    0.00  32568.00      0.00     0.00     0.00   0.00   0.00    7.90    0.00   1.16   119.30     0.00   3.56  97.20
```

The output of iostat should be very familiar to you. Looking at this interface, we see that disk sda is reading 32 MB of data per second, while I/O usage is at 97%, close to saturation, which means that disk sda reads are indeed hitting a performance bottleneck.

So how do we know which processes are responsible for these I/O requests? We can, of course, turn to our old friend, `pidstat`. Next, in Terminal 1, press Ctrl+C to stop the `iostat` command, and then run the following `pidstat` command to observe the I/O of the processes:

```shell
# -d选项表示展示进程的I/O情况
$ pidstat -d 1
12:04:11      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
12:04:12      999     27458  32640.00      0.00      0.00       0  mysqld
12:04:12        0     27617      4.00      4.00      0.00       3  python
12:04:12        0     27864      0.00      4.00      0.00       0  systemd-journal
```

From the output of `pidstat`, we can see that the mysqld process with PID 27458 is doing a lot of reading, and the read speed is 32 MB/s, which is consistent with what we just found from `iostat`. Comparing the two results, we naturally find the source of the disk I/O bottleneck, namely the mysqld process.

But that's not all. Naturally, we have to wonder why mysqld is reading large amounts of disk data. As we mentioned earlier in the speculation, this could be a slow query problem.

However, if you think about it, most slow queries are characterized by high CPU usage (e.g., 100%), but what we see here is an I/O problem. It seems that this is not just a slow query problem, but we need to analyze the data read by MySQL.

To analyze the data read by the process, of course, we have to rely on the `strace` + `lsof` combination we used in the previous section.

Next, again in Terminal 1, execute the `strace` command and specify the mysqld process number 27458. we know that MySQL is a multi-threaded database application, so in order not to miss the data reads of these threads, you should remember to execute the `stace` command with the -f argument: `stace`:

```shell
$ strace -f -p 27458
[pid 28014] read(38, "934EiwT363aak7VtqF1mHGa4LL4Dhbks"..., 131072) = 131072
[pid 28014] read(38, "hSs7KBDepBqA6m4ce6i6iUfFTeG9Ot9z"..., 20480) = 20480
[pid 28014] read(38, "NRhRjCSsLLBjTfdqiBRLvN9K6FRfqqLm"..., 131072) = 131072
[pid 28014] read(38, "AKgsik4BilLb7y6OkwQUjjqGeCTQTaRl"..., 24576) = 24576
[pid 28014] read(38, "hFMHx7FzUSqfFI22fQxWCpSnDmRjamaW"..., 131072) = 131072
[pid 28014] read(38, "ajUzLmKqivcDJSkiw7QWf2ETLgvQIpfC"..., 20480) = 20480
```

If you look for a while, you will see that thread 28014 is reading a lot of data and the descriptor number of the file it is reading is 38. Which file does the 38 here correspond to? We can check this suspicious thread and file by executing the following `lsof` command and specifying the thread number 28014:

```shell
$ lsof -p 28014
```

Strangely enough, `lsof` does not give any output. In fact, if you look at the return value of the `lsof` command, you will see that the execution of this command failed.

We know that in SHELL, the special scalar `$? ` indicates the return value of the previous command when it exited. If you look at this special scalar, you will see that the return value is 1. But don't forget that in Linux, the return value is 0, which means the command was executed successfully. A return value of 1 obviously indicates a failure.

```shell
$ echo $?
1
```

Why did the `lsof` command fail to execute? I hope you will pause here and think about the reason first. Remember my saying, when you can't explain a phenomenon, check the documentation of the tool first.

In fact, by looking up the documentation for `lsof`, you will find that the -p parameter requires a process number, and we just passed in a thread number, so `lsof` failed. You see, any detail can be a "roadblock" to performance analysis.

Looking back, we see that mysqld's process number is 27458, and 28014 is just one of its threads. Also, if you look at the threads of the mysqld process, you will see that mysqld actually has many other threads running:

```shell
# -t表示显示线程，-a表示显示命令行参数
$ pstree -t -a -p 27458
mysqld,27458 --log_bin=on --sync_binlog=1
...
  ├─{mysqld},27922
  ├─{mysqld},27923
  └─{mysqld},28014
```

Once you have found the cause, the `lsof` problem is easy to solve. Replace the thread number with the process number and continue with the `lsof` command:

```shell
$ lsof -p 27458
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
...
mysqld  27458  999   38u   REG    8,1 512440000 2601895 /var/lib/mysql/test/products.MYD
```

This time we get the output of `lsof`. From the output, we can see that the mysqld process did open a large number of files, and based on the file descriptor (FD) number, we know that the file with descriptor 38 is a file with path <ins>/var/lib/mysql/test/products.MYD</ins>. Note here that the u after 38 means that mysqld is accessing the file as a read and write.

For those of you familiar with MySQL, you may laugh when you see this file:

- MYD file, which is the file used by the MyISAM engine to store the table data;
- the file name is the name of the data table;
- and the parent directory of this file, which is the name of the database.

In other words, this file tells us that mysqld is reading the products table in the database test.

In fact, you can run the following command to see what mysqld is storing when it manages the database test. Note, however, that since MySQL is running in a container, you need to go to the container via `docker exec` to see:

```shell
$ docker exec -it mysql ls /var/lib/mysql/test/
db.opt    products.MYD  products.MYI  products.frm
```

As you can see here, there are four files in the /var/lib/mysql/test/ directory, each of which serves the following purposes:

- MYD file to store table data;
- MYI file to store table indexes;
- frm file to store table meta-information (such as table structure);
- opt file to store database meta-information (such as character set, character checksum rules, etc.).

Of course, seeing all this, you may still have a question, that is, are these files the database files that mysqld is using or not? Is it possible that they are old data that is no longer in use? Actually, this is easy to confirm by looking up the mysqld configuration data path.

You can go ahead and execute the following command in Terminal 1:

```shell
$ docker exec -i -t mysql mysql -e 'show global variables like "%datadir%";'
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| datadir       | /var/lib/mysql/ |
+---------------+-----------------+
```

Here you can see that <ins>/var/lib/mysql/</ins> is indeed the data storage directory being used by mysqld. The database test and the data table products, which we just analyzed, are both in use.

> Note: Actually, the results of lsof can already confirm that they are all files that mysqld is accessing. The query
> datadir is just to change the way you think, to further confirm.

Now that you have identified the database and tables, the next step is to figure out what SQL is being executed in the database. Let's go ahead and run the following `docker exec` command in Terminal 1 to enter the MySQL command line interface:

```shell
$ docker exec -i -t mysql mysql
...

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

The next step you should be able to think of is to execute the `show processlist` command in the MySQL command line interface to see the currently executing SQL statements.

However, to make sure that the SQL statements are not truncated, here we can execute the `show full processlist` command. If everything works, you should see the following output:

```plaintext
mysql> show full processlist;
+----+------+-----------------+------+---------+------+--------------+-----------------------------------------------------+
| Id | User | Host            | db   | Command | Time | State        | Info                                                |
+----+------+-----------------+------+---------+------+--------------+-----------------------------------------------------+
| 27 | root | localhost       | test | Query   |    0 | init         | show full processlist                               |
| 28 | root | 127.0.0.1:42262 | test | Query   |    1 | Sending data | select * from products where productName='geektime' |
+----+------+-----------------+------+---------+------+--------------+-----------------------------------------------------+
2 rows in set (0.00 sec)
```

In this output

- db indicates the name of the database;
- Command indicates the type of SQL;
- Time indicates the execution time;
- State indicates the status;
- and Info contains the full SQL statement.

Execute the `show full processlist` command a few more times and you will see that the `select * from products where productName='geektime'` SQL statement takes longer to execute.

Recall that the product name 🔗<http://192.168.0.10:10000/products/geektime>, which we queried in Terminal 2 at the beginning of the case, has a geektime that also matches the conditions of this query statement.

We know that MySQL's slow query problem is most likely caused by not making good use of indexes, so is this query statement the case? How can we confirm that the query statement is using indexes?

Actually, MySQL's built-in `explain` command can help you solve this problem. Go ahead and run the following `explain` command in the MySQL terminal:

```shell
# 切换到test库
mysql> use test;
# 执行explain命令
mysql> explain select * from products where productName='geektime';
+----+-------------+----------+------+---------------+------+---------+------+-------+-------------+
| id | select_type | table    | type | possible_keys | key  | key_len | ref  | rows  | Extra       |
+----+-------------+----------+------+---------------+------+---------+------+-------+-------------+
|  1 | SIMPLE      | products | ALL  | NULL          | NULL | NULL    | NULL | 10000 | Using where |
+----+-------------+----------+------+---------------+------+---------+------+-------+-------------+
1 row in set (0.00 sec)
```

Observe the output this time. There are a few more important fields in this interface that you need to pay attention to, so I'll take this output as an example and explain each of them:

- select_type means the query type, and SIMPLE here means this query does not include UNION queries or subqueries;
- table means the name of the data table, here is products;
- type means the query type, ALL here means a full table query, but index queries should be index type;
- possible_ keys indicates the possible indexes, here is NULL;
- key indicates the exact indexes that will be used, here is also NULL;
- rows indicates the number of rows scanned by the query, here is 10000.

Based on this information, we can determine that this query statement does not use indexes at all, so the query will scan the whole table and the number of rows scanned is up to 10000. It's no wonder that the response is so slow.

At this point, it should be easy for you to think of an optimization method, no indexes then we will build their own, to the productName to build an index can be. However, before you add an index, you need to find out what the table structure really looks like.

Execute the following MySQL command to query the structure of the products table, and you will see that it only has an id primary key, and does not include an index for productName:

```shell
mysql> show create table products;
...
| products | CREATE TABLE `products` (
  `id` int(11) NOT NULL,
  `productCode` text NOT NULL COMMENT '产品代码',
  `productName` text NOT NULL COMMENT '产品名称',
...
  PRIMARY KEY (`id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC |
...
```

Next, we can create an index for productName by executing the following `CREATE INDEX` command:

```shell
mysql> CREATE INDEX products_index ON products (productName);
ERROR 1170 (42000): BLOB/TEXT column 'productName' used in key specification without a key length
```

However, the striking ERROR tells us that the command failed to run. According to the error message, productName is a BLOB/TEXT type and needs to be set to a length. So, to create an index, you must specify a prefix length for productName.

What is the appropriate prefix length to set? There is actually a special algorithm to determine the length of the index by calculating the selectivity of the prefix length. However, we can simplify it a bit by directly creating the index using a fixed value (e.g. 64) and executing the following command:

```shell
mysql> CREATE INDEX products_index ON products (productName(64));
Query OK, 10000 rows affected (14.45 sec)
Records: 10000  Duplicates: 0  Warnings: 0
```

Now you can see that the indexes have been built. All that can be done is done, so finally it's time to check if the performance problem has been solved.

Let's switch to Terminal 2 and see the results of the `curl` command that is still being executed:

```plaintext
Got data: ()in 15.383180141448975 sec
Got data: ()in 15.384996891021729 sec
Got data: ()in 0.0021054744720458984 sec
Got data: ()in 0.003951072692871094 sec
```

Apparently, the query time has been reduced from 15 seconds to 3 milliseconds. It seems that the lack of indexes is really the culprit of this performance problem, and solving the indexes solves the problem of slow queries.

## Case thinking

By this point, the problem of slow query of the product search application has been solved perfectly. However, there is one more point I would like to clarify about this case.

I don't know if you remember, but at the beginning of the case, we started several container applications. In addition to the MySQL and Product Search applications, there was also a DataService application. Why did this case start with a seemingly unrelated application running?

In fact, DataService is a disruptive application that severely impacts MySQL performance. Leaving aside the index optimization method described above, there is another optimization method in this case, which is to stop the DataService application.

Next, let's delete the database indexes and go back to the original state; then stop the DataService application and see how the optimization works.

First, we stop the `curl` command in Terminal 2, then go back to Terminal 1 and execute the following command to delete the indexes:

```shell
# 删除索引
$ docker exec -i -t mysql mysql

mysql> use test;
mysql> DROP INDEX products_index ON products;
```

Next, re-run the `curl` command in Terminal 2. This time, of course, you will notice that the processing time has become slower again: the

```shell
$ while true; do curl http://192.168.0.10:10000/products/geektime; sleep 5; done
Got data: ()in 16.884345054626465 sec
```

Next, go back to Terminal 1 again and execute the following command to stop the DataService application:

```shell
# 停止 DataService 应用
$ docker rm -f dataservice
```

Finally, we go back to Terminal 2 and observe the results of `curl`:

```plaintext
Got data: ()in 16.884345054626465 sec
Got data: ()in 15.238174200057983 sec
Got data: ()in 0.12604427337646484 sec
Got data: ()in 0.1101069450378418 sec
Got data: ()in 0.11235237121582031 sec
```

Sure enough, after stopping DataService, the processing time is reduced from 15 seconds to 0.1 seconds, which is not as good as the 3 milliseconds after adding indexes, but the optimization effect is still very obvious compared to 15 seconds.

So, in this case, is there any I/O bottleneck left?

Let's switch to Terminal 1 and run the following `vmstat` command (note that it is not `iostat`, for reasons to be explained later) to observe the change in I/O:

```shell
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  1      0 6809304   1368 856744    0    0 32640     0   52  478  1  0 50 49  0
 0  1      0 6776620   1368 889456    0    0 32640     0   33  490  0  0 50 49  0
 0  0      0 6747540   1368 918576    0    0 29056     0   42  568  0  0 56 44  0
 0  0      0 6747540   1368 918576    0    0     0     0   40  141  1  0 100  0  0
 0  0      0 6747160   1368 918576    0    0     0     0   40  148  0  1 99  0  0
```

You can see that the disk read (bi) and iowait (wa) are still quite large at first, but it doesn't take long for both to become 0. In other words, the I/O bottleneck disappears.

Why is this? Let's leave the reason open as a thought for today.

To go back and explain what I just did, I didn't use the `iostat` command to check the I/O situation, but rather -. In fact, compared to `iostat`, `vmstat` can provide CPU, memory and I/O usage at the same time.

During performance analysis, being able to combine multiple metrics and analyze them in the context of how the system works is often unexpectedly helpful in explaining performance phenomena.

## Summary

Today we analyzed a product search application. We first analyzed the CPU and disk usage of the system with `top` and `iostat` and found the disk I/O bottleneck.

Then, with the help of `pidstat`, we found that the bottleneck was caused by mysqld. Then, we used `strace` and `lsof` to find out which files mysqld was reading. Also, based on the names and paths of the files, we found out which databases and data tables mysqld was operating on. Combining this information, we determined that this was a slow query problem caused by not utilizing indexes.

So, we logged into the MySQL command line terminal and verified with the database analysis tool, and found that the fields accessed by the MySQL query statement were, indeed, not indexed. So, adding indexes solved the performance problem of the case.

## Thinking

Finally, to leave you with a thought, which I also mentioned in the last part of the case, the processing time of the product search application was reduced from 15 seconds to 0.1 seconds after stopping the DataService. Why is that?

I'll give a small hint. You can first check the 🔗[source](https://github.com/feiskyer/linux-perf-examples/blob/master/mysql-slow/dataservice.py) of dataservice.py and you will find that , the DataService is actually reading and writing a small file containing only the "data" string. But before it reads the file, it changes <ins>/proc/sys/vm/drop_caches</ins> to 1.

Do you remember what this action does? If not, you can use `man` to look up the documentation for the proc file system.

Feel free to discuss with me in the comments section, and feel free to share this article with your colleagues and friends. Let's rehearse together in the real world and make progress in the exchange.
