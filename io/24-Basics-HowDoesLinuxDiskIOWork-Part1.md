# 24 Basics: How does Linux disk I/O work(part 1)?

In the previous section, we learned how the Linux file system works. To recap, a file system is a mechanism for organizing and managing the files on a storage device. On top of the various file system implementations, Linux abstracts a virtual file system, VFS, which defines a set of data structures and standard interfaces that are supported by all file systems.

This way, applications only need to interact with the unified interface provided by VFS, and do not need to pay attention to the specific implementation of the file system; for specific file systems, they only need to follow the VFS standard, and can seamlessly support various applications.

Internally, VFS manages files through data structures such as directory entries, index nodes, logical blocks, and superblocks.

- Directory entries, which record the names of files and the directory relationships between files and other directory entries.
- Index nodes, which record the file's metadata.
- Logical blocks, which are the smallest read and write units consisting of contiguous disk sectors, are used to store file data.
- Superblock, which is used to record the status of the file system as a whole, such as the usage of index nodes and logical blocks.

Among them, directory entries are a memory cache; and superblocks, index nodes and logical blocks, are persistent data stored in the disk.

So, thinking further, how does the disk work? And what metrics can be used to measure its performance?

Next, I'll take a look at how Linux disk I/O works.

## Disk

Disks are devices that can be persistently stored, and depending on the storage medium, common disks can be divided into two categories: mechanical disks and solid-state disks.

The first type, mechanical disks, is also known as Hard Disk Driver (HDD), usually abbreviated as HDD. mechanical disks are mainly composed of platters and read/write heads, and data is stored in the disk's ring-like track. Before reading or writing data, you need to move the read/write head and locate the track where the data is located before you can access it.

Obviously, if the I/O requests happen to be contiguous, then there is no need for track addressing and the best performance is naturally obtained. This is actually the familiar principle of how sequential I/O works. The opposite, of course, is random I/O, which requires constantly moving the head to locate the data, so the read/write speed will be slower.

The second type, Solid State Disk (SSD), often abbreviated as SSD, consists of solid-state electronic components. Solid state disks do not require track addressing, so both sequential I/O and random I/O performance are much better than mechanical disks.

In fact, random I/O of the same disk is much slower than sequential I/O regardless of mechanical or solid-state disks, and the reason is obvious.

- For mechanical disks, as we just mentioned, random I/O is naturally slower than sequential I/O because it requires more head seeking and platter rotation.
- For solid state disks, although their random performance is much better than mechanical drives, they also have the same "erase before write" limitation. Random reads and writes can lead to a lot of garbage collection, so correspondingly, random I/O performance is still much worse than continuous I/O.
- In addition, continuous I/O can reduce the number of I/O requests by pre-reading, which is one of the reasons for its excellent performance. Many performance-optimized solutions also optimize I/O performance from this perspective.

In addition, mechanical and solid state disks each have a minimum read/write unit.

- The smallest read/write unit for mechanical disks is the sector, which is typically 512 bytes in size.
- Solid-state disks have a minimum read/write unit of pages, typically 4KB, 8KB, etc.

As I mentioned in the previous section, it would be inefficient to read and write such small units as 512 bytes at a time. So, the file system takes contiguous sectors or pages, forms them into logical blocks, and then uses the logical block as the smallest unit to manage the data. A common logical block size is 4KB, which means that 8 consecutive sectors, or a single page, can form a logical block.

In addition to storage media, another common classification is by interface, for example, hard drives can be classified as IDE (Integrated Drive Electronics), SCSI (Small Computer System Interface), SAS (Serial Attached SCSI), SATA (Serial ATA), FC (Fibre Channel) and so on.

Different interfaces are often assigned different device names. For example, IDE devices are assigned a device name with an hd prefix, SCSI and SATA devices are assigned a device name with an sd prefix. In case of multiple disks of the same type, they are numbered in alphabetical order of a, b, c, etc.

In addition to the classification of the disks themselves, when you plug them into the server, they can be divided into a variety of different architectures according to the different ways they are used.

The simplest is to use them directly as standalone disk devices. These disks are also often divided into logical partitions, each numbered as needed. For example, /dev/sda, which we have used many times before, can be divided into two partitions, /dev/sda1 and /dev/sda2.

Another common architecture is to combine multiple disks into a single logical disk to form a Redundant Array of Independent Disks (RAID), which can improve data access performance and enhance data storage reliability.

Depending on the capacity, performance and reliability requirements, RAID can generally be divided into multiple levels, such as RAID0, RAID1, RAID5, RAID10 and so on.

- RAID0 has optimal read and write performance, but does not provide data redundancy.
- Other levels of RAID, on the other hand, provide data redundancy with a degree of optimization for read and write performance.

The last architecture, which combines these disks into a network storage cluster, is then exposed for use by servers through network storage protocols such as NFS, SMB, iSCSI, etc.

In fact, in Linux, **disks are actually managed as a block device**, that is, they read and write data in blocks and support random reads and writes. Each block device is given two device numbers, the primary and secondary device numbers. The primary device number is used in the driver to distinguish between device types, while the secondary device number is used to number multiple devices of the same type.

### Common block layer

Similar to the Virtual File System VFS we talked about in the previous section, Linux manages the various block devices through a unified common block layer in order to reduce the impact of the differences between the different block devices.

The generic block layer is actually a block device abstraction layer in the middle of the file system and the disk driver. It has two main functions.

- The first function is similar to that of a virtual file system. Upward, it provides a standard interface for file systems and applications to access block devices; downward, it abstracts various heterogeneous disk devices into a unified block device and provides a unified framework to manage the drivers of these devices.
- The second function, the common block layer, also queues I/O requests from file systems and applications and improves the efficiency of disk read and write by reordering and merging requests.

The process of sequencing I/O requests is also known as I/O scheduling. In fact, the Linux kernel supports four I/O scheduling algorithms, namely NONE, NOOP, CFQ and DeadLine, which I will introduce here.

The first one, NONE, can't be considered an I/O scheduling algorithm, to be more precise. Because it does not use any I/O scheduler at all, it does not actually do any processing of file system and application I/O, and is commonly used in virtual machines (where disk I/O scheduling is entirely the responsibility of the physical machine).

The second type, NOOP, is the simplest type of I/O scheduling algorithm. It is actually a first-in-first-out queue that does only the most basic request merging and is commonly used for SSD disks.

The third CFQ (Completely Fair Scheduler), also known as a completely fair scheduler, is the default I/O scheduler for many distributions today. It maintains an I/O scheduling queue for each process and distributes I/O requests for each process evenly by time slice.

Similar to process CPU scheduling, CFQ also supports prioritized scheduling of process I/O, making it suitable for systems running a large number of processes, such as desktop environments, multimedia applications, etc.

Lastly, DeadLine scheduling algorithm creates separate I/O queues for read and write requests to increase the throughput of mechanical disks and ensure that requests that reach the deadline are prioritized.

### I/O stack

Once we have a clear understanding of how the disk and common block layers work, combined with the file system principles we talked about in the last issue, we can look at the I/O principles of the Linux storage system as a whole.

We can divide the I/O stack of the Linux storage system into three layers from top to bottom: the file system layer, the general-purpose block layer, and the device layer. The relationship between these three I/O layers is shown in the following diagram, which is actually a panoramic view of the I/O stack of the Linux storage system.

<center><a href="https://ibb.co/9Y7KqKn">
    <img src="https://i.ibb.co/MNmwRwn/The-Linux-Storage-Stack-Diagram.png" alt="The-Linux-Storage-Stack-Diagram" border="0">
</a></center>

（图片来自 🔗[Linux Storage Stack Diagram](https://www.thomas-krenn.com/en/wiki/Linux_Storage_Stack_Diagram) ）

Based on this panoramic view of the I/O stack, we can understand more clearly how storage system I/O works.

- The file system layer, which includes the virtual file system and various other file system specific implementations. It provides a standard file access interface for upper-level applications, and stores and manages disk data for lower-level applications through the general-purpose block layer.
- The generic block layer, including the block device I/O queue and I/O scheduler. It will queue I/O requests from the file system and then reorder and merge the requests before they are sent to the next level of the device layer.
- The device layer, which includes the storage devices and the corresponding drivers, is responsible for the I/O operations of the final physical device.

I/O of the storage system is usually the slowest part of the system. Therefore, Linux uses various caching mechanisms to optimize I/O efficiency.

Let's say, to optimize the performance of file access, various caching mechanisms such as page caching, index node caching, and directory item caching are used to reduce the direct calls to the lower-level block devices.

Similarly, in order to optimize the access efficiency of block devices, buffers are used to cache the data of block devices.

However, after talking about the abstract principles, how should we measure the I/O performance of the disk in concrete operation? I'm going to give you a sneak preview, and in the next lesson we'll look at the most commonly used disk I/O performance metrics and I/O performance tools.

## Summary

In today's article, we went through how Linux disk I/O works and learned about the Linux storage system I/O stack, which consists of the file system layer, the common block layer, and the device layer.

The common block layer is the core of Linux disk I/O. Upward, it provides a standard interface for file systems and applications to access block devices; downward, it abstracts various heterogeneous disk devices into a unified block device and will reorder and merge I/O requests from file systems and applications to improve the efficiency of disk access.
