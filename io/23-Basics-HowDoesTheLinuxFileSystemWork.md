# 23 Basics: How does the Linux file system work?

With the previous CPU and memory modules, I believe that you have already mastered the CPU and memory performance analysis and optimization ideas. Starting from this section, we will move on to the next important module - file system and disk I/O performance.

Like CPU and memory, disk and file system management are the core functions of an operating system.

- The disk provides the most basic persistent storage for the system.
- The file system builds on the disk and provides a tree structure for managing files.

So, how do disks and file systems work? And what metrics are there to measure their performance?

Today, I'll take you on a first look at how the Linux file system works. How disks work, we'll learn in the next section.

## Index nodes and directory entries

A file system, itself, is a mechanism for organizing and managing files on a storage device. Different ways of organizing them result in different file systems.

The most important thing you have to remember is that in Linux everything is a file. Not only ordinary files and directories, but also block devices, sockets, pipes, etc., are managed through a unified file system.

To facilitate management, the Linux file system allocates two data structures to each file, index node and directory entry. They are mainly used to record the meta-information and directory structure of the files.

- The index node, referred to as inode, is used to record the metadata of a file, such as inode number, file size, access rights, modification date, location of data, etc. The index node corresponds to the file one-by-one, and it is stored persistently to disk, just like the file content. So remember, index nodes also take up disk space.
- A directory entry, referred to as a dentry, is used to record the name of a file, a pointer to the index node, and its association with other directory entries. Multiple associated directory entries make up the directory structure of the file system. However, unlike an index node, a directory entry is a memory data structure maintained by the kernel, so it is often also called a directory entry cache.

In other words, an index node is a unique identifier for each file, while a directory entry maintains the tree structure of the file system. The relationship between directory entries and index nodes is many-to-one, which you can simply understand to mean that a file can have multiple aliases.

For example, the aliases created for a file by hard linking will correspond to different directory entries, but these directory entries are still essentially links to the same file, so they have the same index node.

The index nodes and directory entries record the metadata of the file and the directory relationships between the files, so specifically, how is the file data actually stored? Is it just written directly to the disk?

In fact, the smallest unit of disk reading and writing is the sector, but the sector is only 512B in size, so if we read and write such a small unit every time, the efficiency must be very low. So, the file system then makes logical blocks out of consecutive sectors, and then uses the logical block as the smallest unit to manage the data each time. A common logical block size is 4KB, which means it consists of 8 contiguous sectors.

To help you understand the relationship between directory entries, index nodes, and file data, I have drawn a schematic diagram. You can cross-reference this diagram to recall what has just been taught and to tie together the knowledge and details.


<center><a href="https://ibb.co/4mP77sx">
    <img src="https://i.ibb.co/X2SbbyT/Dentry-Inode-File-Data.png" alt="Dentry-Inode-File-Data" border="0">
</a></center>

However, there are two things you need to be aware of here.

First, directory entries are themselves a memory cache, while index nodes are data stored on disk. In the previous Buffer and Cache principles, I mentioned that in order to reconcile the performance difference between slow disks and fast CPUs, the file contents are cached in a page-cache Cache.

Then, you should expect that these index nodes are naturally cached into memory as well, speeding up file access.

Second, when the disk performs file system formatting, it is divided into three storage areas, the superblock, index node area, and data block area. Among them, the

- Superblock, which stores the state of the entire file system.
- The index node area, which is used to store the index nodes.
- The data block area, on the other hand, is used to store file data.

## Virtual File System

Directory entries, index nodes, logical blocks, and superblocks make up the four basic elements of the Linux file system. However, in order to support a variety of different file systems, the Linux kernel introduces an abstraction layer, the Virtual File System (VFS), in the middle of the user process and the file system.

VFS defines a set of data structures and standard interfaces that are supported by all file systems. This way, user processes and other subsystems in the kernel only need to interact with the unified interface provided by VFS, and do not need to care about the implementation details of the various underlying file systems.

Here, I've drawn a diagram of the Linux file system architecture to help you better understand the relationship between system calls, VFS, cache, file system, and block storage.

<center><a href="https://imgbb.com/">
    <img src="https://i.ibb.co/3fGP12X/Architecture-Diagram-Of-Linux-File-System.png" alt="Architecture-Diagram-Of-Linux-File-System" border="0">
</a></center>

With this diagram, you can see that underneath VFS, Linux supports a variety of file systems such as Ext4, XFS, NFS, and so on. According to the storage location, these file systems can be divided into three categories.

- The first category is disk-based file systems, which means that data is stored directly on the computer's locally mounted disk. The common Ext4, XFS, OverlayFS, etc., are all such file systems.
- The second type is memory-based file system, which is often referred to as virtual file system. This type of file system does not require any disk allocated storage space, but will occupy memory. The /proc file system, which we often use, is actually one of the most common virtual file systems. In addition, the /sys file system also belongs to this category and exports hierarchical kernel objects to user space.
- The third category is network file systems, which are file systems used to access data from other computers, such as NFS, SMB, iSCSI, etc.

These are file systems that have to be mounted to a subdirectory (called a mount point) in the VFS directory tree before the files in it can be accessed. In the first category, disk-based file systems, for example, when installing the system, a root directory (/) is mounted first, and other file systems (such as other disk partitions, /proc file systems, /sys file systems, NFS, etc.) are mounted in the root directory.

## File System I/O

After mounting a file system to a mount point, you can then access the files it manages through the mount point.VFS provides a standard set of file access interfaces. These interfaces are provided to applications as system calls.

Take the cat command, which first calls open() to open a file, then read() to read the contents of the file, and finally write() to output the contents of the file to the standard output of the console:

```shell
int open(const char *pathname, int flags, mode_t mode); 
ssize_t read(int fd, void *buf, size_t count); 
ssize_t write(int fd, const void *buf, size_t count); 
```

There are various differences in the way files are read and written, resulting in a variety of I/O classifications. The most common ones are buffered vs. non-buffered I/O, direct vs. non-direct I/O, blocking vs. non-blocking I/O, synchronous vs. asynchronous I/O, and so on. Next, we will look at these four classifications in detail.

First, file I/O can be classified into cached I/O and uncached I/O based on whether or not the standard library cache is utilized.

- Buffered I/O means that the standard library cache is used to speed up access to the file, and the file is then accessed internally by the standard library through system scheduling.
- Non-buffered I/O refers to accessing files directly through system calls, without going through the standard library cache.
 
Note that by "caching", we mean the cache implemented internally by the standard library. For example, as you may have seen, many programs do not actually output until they encounter a line break, and the content before the line break is actually temporarily cached by the standard library.

Whether buffered I/O or unbuffered I/O, they still end up going through system calls to access the file. And as we know from the previous section, the system call is followed by a page cache to reduce the I/O operations on disk.

Second, file I/O can be classified into direct I/O and non-direct I/O based on whether or not the operating system's page cache is used.

- Direct I/O means that the file is accessed directly by interacting with the file system, skipping the OS page cache.
- Non-direct I/O is the opposite, where files are read and written through the system's page cache before they are actually written to disk by the kernel or additional system calls.

To implement direct I/O, you need to specify the O_DIRECT flag in the system call. If it is not set, the default is non-direct I/O.

Note, however, that direct I/O, non-direct I/O, is still essentially interacting with the file system. If you are in a scenario such as a database, you will also see that skipping the file system to read and write to disk is what we usually call bare I/O.

Third, file I/O can be classified into blocking I/O and non-blocking I/O according to whether the application blocks its own operation:

- Blocking I/O means that after an application performs an I/O operation, if it does not get a response, it blocks the current thread and naturally cannot perform other tasks.
- Non-blocking I/O means that after an application performs an I/O operation, it does not block the current thread and can continue to perform other tasks and then get the result of the call through polling or event notification.

For example, when accessing a pipe or network socket, setting the O_NONBLOCK flag means that the access is done in a non-blocking way; if nothing is set, the default is blocking access.

Fourth, file I/O can be classified into synchronous and asynchronous I/O according to whether or not to wait for the response result:

- By synchronous I/O, it means that after the application executes I/O operation, it has to wait until the whole I/O is completed to get the I/O response.
- Asynchronous I/O means that after the application executes an I/O operation, it does not have to wait for the completion and the response after completion, but just continues to execute. When this I/O completes, the response will tell the application with an event notification.

For example, when operating on a file, if you set the O_SYNC or O_DSYNC flag, it means synchronous I/O. If you set O_DSYNC, you have to wait for the file data to be written to disk before returning, while O_SYNC, on top of O_DSYNC, requires the file metadata to be written to disk before returning.

For example, if the O_ASYNC option is set when accessing a pipe or network socket, the corresponding I/O will be asynchronous I/O. In this case, the kernel will then notify the process whether the file is available for reading or writing by SIGIO or SIGPOLL.

As you may have noticed, many of the concepts here are also often found in network programming. Non-blocking I/O, for example, is often used in conjunction with select/poll for I/O on network sockets.

You should also understand the profound meaning of "everything is a file in Linux". Whether it is normal files and block devices, network sockets and pipes, etc., they are all accessed through a unified VFS interface.

## Performance observation

After learning so much about the principles of the file system, you probably can't wait to get your hands on it and observe how it performs.

Next, open a terminal, SSH to the server, and join me in exploring how to observe the performance of the file system.

### Capacity

One of the most common problems for file systems is running out of space. Of course, you probably know this yourself, and with the `df` command, you can check the disk space usage of the file system. For example:

```shell
$ df /dev/sda1 
Filesystem     1K-blocks    Used Available Use% Mounted on 
/dev/sda1       30308240 3167020  27124836  11% /
```

As you can see, my root filesystem only uses 11% of the space. Note also here that the total space is expressed in 1K-blocks, and you can add the -h option to `df` for better readability:

```shell
$ df -h /dev/sda1 
Filesystem      Size  Used Avail Use% Mounted on 
/dev/sda1        29G  3.1G   26G  11% /
```

But sometimes, when you are clearly running out of space, but you check the disk space with `df`, you find that there is still plenty of space left. What's going on?

I don't know if you remember a detail that I highlighted earlier. In addition to file data, index nodes also take up disk space. You can add the -i argument to the `df` command to see how index nodes are being used, as follows:

```shell
$ df -i /dev/sda1 
Filesystem      Inodes  IUsed   IFree IUse% Mounted on 
/dev/sda1      3870720 157460 3713260    5% /
```

The capacity of the index node, (that is, the number of Inodes) is set when you format the disk and is usually generated automatically by the formatting tool. When you find that there is not enough space in the index node, but plenty of space on the disk, it is most likely caused by too many small files.

So, in general, deleting these small files, or moving them to other disks where the index node is sufficient, will solve the problem.

## Cache

In the previous Cache case, I have described that you can use `free` or `vmstat` to see the size of the page cache. As a refresher, the Cache output from free is the sum of the page cache and the recyclable Slab cache, and you can get their sizes directly from <ins>/proc/meminfo</ins>:

```shell
$ cat /proc/meminfo | grep -E "SReclaimable|Cached" 
Cached:           748316 kB 
SwapCached:            0 kB 
SReclaimable:     179508 kB
```

That said, what about the cache of directory entries and index nodes in the file system, and what to look for?

In fact, the kernel uses the Slab mechanism to manage the cache of directory entries and index nodes. <ins>/proc/meminfo</ins> only gives the overall size of the Slab, for each type of Slab cache you have to check the file <ins>/proc/slabinfo</ins>.

For example, by running the following command, you can get, for example, a cache of all directory entries and various file system index nodes:

```shell
$ cat /proc/slabinfo | grep -E '^#|dentry|inode' 
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail> 
xfs_inode              0      0    960   17    4 : tunables    0    0    0 : slabdata      0      0      0 
... 
ext4_inode_cache   32104  34590   1088   15    4 : tunables    0    0    0 : slabdata   2306   2306      0hugetlbfs_inode_cache     13     13    624   13    2 : tunables    0    0    0 : slabdata      1      1      0 
sock_inode_cache    1190   1242    704   23    4 : tunables    0    0    0 : slabdata     54     54      0 
shmem_inode_cache   1622   2139    712   23    4 : tunables    0    0    0 : slabdata     93     93      0 
proc_inode_cache    3560   4080    680   12    2 : tunables    0    0    0 : slabdata    340    340      0 
inode_cache        25172  25818    608   13    2 : tunables    0    0    0 : slabdata   1986   1986      0 
dentry             76050 121296    192   21    1 : tunables    0    0    0 : slabdata   5776   5776      0
```

In this interface, the dentry line indicates the directory entry cache, the inode_cache line, the VFS index node cache, and the rest are the index node caches of various file systems.

There are more columns in <ins>/proc/slabinfo</ins>, so you can check man slabinfo for the exact meaning. `slabtop` is more commonly used in real-world performance analysis to find the type of cache that takes up the most memory.

For example, here is what I get when I run `slabtop`:

```shell
# 按下c按照缓存大小排序，按下a按照活跃对象数排序 
$ slabtop 
Active / Total Objects (% used)    : 277970 / 358914 (77.4%) 
Active / Total Slabs (% used)      : 12414 / 12414 (100.0%) 
Active / Total Caches (% used)     : 83 / 135 (61.5%) 
Active / Total Size (% used)       : 57816.88K / 73307.70K (78.9%) 
Minimum / Average / Maximum Object : 0.01K / 0.20K / 22.88K 

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME 
69804  23094   0%    0.19K   3324       21     13296K dentry 
16380  15854   0%    0.59K   1260       13     10080K inode_cache 
58260  55397   0%    0.13K   1942       30      7768K kernfs_node_cache 
   485    413   0%    5.69K     97        5      3104K task_struct 
  1472   1397   0%    2.00K     92       16      2944K kmalloc-2048
```

## Summary

Today, I take you through the workings of the Linux file system.

A file system, is a mechanism for organizing and managing the files on a storage device. In order to support all kinds of different file systems, Linux abstracts a layer of Virtual File System (VFS) over various file system implementations.

The VFS defines a set of data structures and standard interfaces that are supported by all file systems. This way, user processes and other subsystems in the kernel only need to interact with the uniform interface provided by VFS.

To reduce the impact of slow disks on performance, the file system mitigates the impact of disk latency on applications through page caching, directory item caching, and index node caching.

In terms of performance observations, today we will focus on capacity and cache metrics. In the next section, we will learn how Linux disk I/O works and get to grips with disk I/O performance observations.
