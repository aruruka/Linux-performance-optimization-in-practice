## Customers complained that access to the game server has become slower

Due to the complexity of the client's network environment, there are often some users feedback lagging situation. What I can do on my side is also limited. Only in the first network entrance and exit, record the content of each message sent and received and the specific time stamp (accurate to ms). When encountering player feedback, and then according to the player's unique number, and the approximate time of occurrence, in the logs to find the player's response time, to speculate whether the server response is slow, or the client to the server in the middle of the route is slow. Sometimes even let the client take the initiative to report the time interval between the last request sent and received, to verify whether the client's own network environment caused. In fact, the server side have a record of the processing time of the message, usually almost nothing too time-consuming situation. When encountering delays caused by the client's network, we can only say that the new port is added, so that the client can choose an optimal entrance.

This situation is known as "network jitter". Network jitter is a very common phenomenon. You can consider more access points, private lines, CDN, etc. can be optimized for public network link latency problems.

## Will the socket that is created by the system call, bypassing VFS, also always generate a file descriptor?

<center><a href="https://ibb.co/FnJv6xx">
    <img src="https://i.ibb.co/JC2LnBB/Schematic-Of-The-Linux-Generic-Ip-Networking-Stack.png" alt="Schematic-Of-The-Linux-Generic-Ip-Networking-Stack" border="0">
</a></center>

The above is the Linux network stack diagram.
It can be seen that an application can create a socket through the VFS interface, or bypass VFS to create a socket, although the application always needs to create a socket through a system call.

The socket created by VFS will correspond to a file descriptor, which is easy to understand because VFS is a file system interface. For example, what is behind the VFS can be an actual file system, like the XFS or EXT4.

But will the socket that is created by the system call, bypassing VFS, also always generate a file descriptor?
