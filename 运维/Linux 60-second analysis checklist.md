![image-20210927090220534](C:/Users/RMD-JX/Documents/%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86/%E8%BF%90%E7%BB%B4/Untitled.assets/image-20210927090220534.png)

uptime   

dmesg -T  | tail                                     Kernel errors including OOM events. 7.5.1

vmstat -SM 1                                         System-wide statistics: run queue length, swapping,  overall CPU usage

mpstat -P ALL 1                                    Per-CPU balance: a single busy CPU can indicate poor  thread scaling

pidstat 1                                                Per-process CPU usage: identify unexpected CPU  consumers, and user/system CPU time for each process.

iostat -sxz 1                                           Disk I/O statistics: IOPS and throughput, average wait  time, percent busy.

free -m                                                   Memory usage including the file system cache.

sar -n DEV 1                                           Network device I/O: packets and throughput

sar -n                                                      TCP,ETCP 1 TCP statistics: connection rates, retransmits

top                                                          Check overview