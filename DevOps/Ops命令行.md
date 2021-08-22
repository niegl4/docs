[TOC]



#### uptime

当前时间，系统运行时间，正在登录用户数，1、5、15分钟平均负载。

watch -d uptime

#### mpstat

每个CPU的性能指标，所有CPU的平均指标。

mpstat -P ALL 5

#### pidstat

查看进程的CPU，内存，I/O，上下文切换等性能指标。

pidstat -u 5  //-u cpu

pidstat -wt 5  //-w上下文 -t 线程

pidstat -p $PID  //-p 指定进程

pidstat -d -p $PID  //-d I/O统计数据

#### vmstat

系统的内存，I/O，中断，上下文切换，CPU。

vmstat 5

#### top

平均负载，CPU使用率，内存，进程的状态、CPU、内存。

#### perf

与ab压测配合，进行接口优化。

perf record -g -p $PID //-g 开启调用关系采样，-p指定进程

perf report

#### ab

接口压测。

ab -c 并发数 -n 总请求数 http://x.x.x.x:xx/

#### pstree

进程关系。

pstree -aps $PID  //-a 输出命令行；-p 指定进程；-s 展示父进程。

#### dstat

同时查看CPU，I/O的使用情况。

dstat 5

#### /proc

##### /proc/softirqs

watch -d cat /proc/softirqs

##### /proc/interrupts

watch -d cat /proc/interrupts

#### sar

网络收发的吞吐量（BPS，byte/s），网络收发的PPS（PPS，package/s）。

sar -n DEV 1  //-n DEV 显示网络收发的报告。

系统内存换页情况，内存使用率，缓存和缓冲区用量，swap使用情况。

sar -r -S 1  // -r 显示内存使用情况，-S 显示Swap使用情况。

#### slabtop

所有dentry和各种文件系统inode的缓存情况。

#### iostat

磁盘的使用率，IOPS，吞吐量等。

iostat -d -x 1// -d 显示i/o性能指标；-x 显示扩展统计。

#### iotop

按i/o大小对进程排序。

#### strace

分析系统调用。

strace -f -T -tt -p $PID  //-f 跟踪子进程和子线程；-T 显示系统调用的时长；-tt 显示跟踪时间；-p 指定进程。列出所有系统调用。

strace -f -T -tt -p $PID -e fdatasync  //-e 指定特定的系统调用。

#### lsof

查看进程打开的文件。

lsof -p $PID  //-p 指定进程。

#### ss

查询网络的连接信息。

ss -nlpt  // -n 显示数字地址和端口，而不是名字；-l 只显示Listen socket；-p 显示进程信息；-t 只显示TCP socket。

#### ping

测试主机连通性和延迟。

ping -c3 x.x.x.x  //-c3 发送3次ICMP包后停止。

#### nslookup

查询域名的A记录。

#### dig

查询域名的A记录。

dig -t A config-manager.svc-test.svc.cluster.local @172.30.2.204

#### tcpdump+wireshark