## The difference between iowait% in `top` and %wait in `pidstat`

Some students will compare **%wait** in `pidstat` with **iowait%** (abbreviated **wa**) in `top`, but this is meaningless because they are two completely unrelated metrics.

- In `pidstat`, %wait indicates the percentage of time the process waits for CPU.
- In `top`, iowait% indicates the percentage of CPU time spent waiting for I/O.

For example, if there are a large number of threads in the system, with more threads than CPU cores, you should observe a significant increase in the time these threads spend waiting for the CPU.
On the other hand, iowait% reflects how long the process waits for an IO response.