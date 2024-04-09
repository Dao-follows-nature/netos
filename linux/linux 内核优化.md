# linux 内核优化

Linux 内核优化可以分为以下几个类别，针对不同方面进行调整和优化

1. 内存管理优化：涉及内存分配、页面调度、缓存管理等方面的优化。这包括调整内存分配策略、页面置换算法、页表管理方式等，以提高内存使用效率和性能。
   - `/proc/sys/vm/`：包含与虚拟内存管理相关的参数，如页面大小、页面调度算法等。
   - `/proc/sys/kernel/`：包含与内核行为和内存管理相关的参数，如OOM（Out of Memory）行为、内存回收策略等。
2. 调度器优化：涉及进程调度和 CPU 资源分配的优化。调度器负责决定哪个进程在给定时间片内运行，以及如何分配 CPU 时间。调整调度策略和参数可以改善系统的响应性能和负载均衡。
   - `/proc/sys/kernel/sched/`：包含与调度器相关的参数，如调度策略、时间片长度等。
3. 文件系统优化：涉及文件系统的性能和可靠性方面的优化。包括选择合适的文件系统类型、调整文件系统参数、启用适当的文件系统特性等，以提高文件系统的读写性能和数据保护能力。
   - `/proc/sys/fs/`：包含与文件系统相关的参数，如缓存管理、文件句柄限制等。
4. 网络优化：涉及网络栈和网络设备的优化。包括调整网络协议栈参数、网络缓冲区大小、队列管理算法等，以提升网络传输性能和吞吐量。
   - `/proc/sys/net/`：包含与网络协议栈和网络设备相关的参数，如网络缓冲区大小、连接追踪设置等。
5. I/O 子系统优化：涉及磁盘 I/O、块设备和存储系统的优化。这包括调整磁盘调度算法、启用磁盘缓存、优化 RAID 设置等，以提高磁盘访问性能和数据可靠性。
   - `/proc/sys/block/`：包含与块设备和磁盘 I/O 相关的参数，如磁盘调度算法、读写缓存设置等。
6. 安全性和稳定性优化：涉及内核安全性和稳定性方面的优化。包括启用适当的安全功能、设置合理的权限和访问控制、修复漏洞和错误等，以提高系统的安全性和稳定性。
   - `/proc/sys/kernel/`：包含与内核安全性和稳定性相关的参数，如系统调用限制、地址空间随机化等。



## 内核优化参考示例

请谨慎调整这些参数，以避免超出系统资源限制或引发不稳定的行为。

```shell
net.ipv4.ip_forward = 1 #是否开启数据包转发
net.ipv4.tcp_keepalive_time = 600	 # 表示 TCP 连接的空闲时间阈值，默认值为 7200 秒，当一个 TCP 连接处于空闲状态（没有数据传输）并超过这个阈值时，系统会开始发送保活探测数据包
net.ipv4.tcp_keepalive_probes = 10   # 默认值为 9，表示在认定连接失效之前发送的保活探测数据包的数量。当一个 TCP 连接处于空闲状态并超过空闲时间阈值时，系统会开始发送保活探测数据包
net.ipv4.tcp_keepalive_intvl = 30    # 默认值为 75 秒，表示连续发送保活探测数据包之间的时间间隔。
net.ipv4.tcp_timestamps = 0			 # 禁用 TCP 时间戳选项，从而减少数据包的大小和处理开销。默认值为 1，表示启用 TCP 时间戳选项。
net.ipv4.tcp_orphan_retries = 3		 # 表示孤立连接的重试次数，孤立连接是指服务器端已经关闭了连接，但客户端仍然保持连接打开状态，当服务器端检测到孤立连接时，会发送 FIN 包来终止连接。该参数指定了服务器在终止孤立连接之前的重试次数。
net.ipv4.tcp_syncookies = 1          # 启用SYN cookies，减少半开连接数量
net.ipv4.tcp_tw_reuse = 1            # 启用TIME-WAIT重用
net.ipv4.tcp_fin_timeout = 1		 #默认值为 60 秒，指定了 TCP 连接处于 FIN-WAIT-2 状态的超时时间
net.ipv4.tcp_max_syn_backlog = 16384 #指定了 SYN 队列的最大长度，用于处理传入连接请求
net.ipv4.tcp_max_tw_buckets = 131072 #表示系统同时保持TIME_WAIT的最大数量，如果超过这个数字，TIME_WAIT将立刻被清除并打印警告信息。
net.ipv4.tcp_rmem = 4096	87380	6291456 #TCP 套接字接收缓冲区的最小值、默认值和最大值（以字节为单位）。
net.ipv4.tcp_wmem = 4096    20480   4194304 #TCP 套接字发送缓冲区的最小值、默认值和最大值（以字节为单位）。
net.ipv4.tcp_mem = 4096 8192 16777216  #TCP 套接字共享的内存池的最小值、默认值和最大值（以页为单位）。
net.nf_conntrack_max = 25000000
net.netfilter.nf_conntrack_max = 25000000 #指定了连接跟踪表的最大条目数。连接跟踪表用于存储当前活动的网络连接信息。net.nf_conntrack_max 是旧的参数名，net.netfilter.nf_conntrack_max 是新的参数名。两者具有相同的功能。较大的连接跟踪表大小可以容纳更多的活动连接，适用于高负载的网络环境。
net.netfilter.nf_conntrack_tcp_timeout_established=180 #定了 TCP 连接在状态为 "ESTABLISHED"（已建立连接）时的超时时间（以秒为单位）。内核会根据超时时间来判断连接是否超时并将其从连接跟踪表中删除。
net.netfilter.nf_conntrack_tcp_timeout_time_wait=120 #指定了 TCP 连接在状态为 "TIME_WAIT"（等待关闭）时的超时时间（以秒为单位）。内核会根据超时时间来判断连接是否超时并将其从连接跟踪表中删除。
net.netfilter.nf_conntrack_tcp_timeout_close_wait=60 # TCP 连接在状态为 "CLOSE_WAIT"（等待关闭确认）时的超时时间（以秒为单位）。内核会根据超时时间来判断连接是否超时并将其从连接跟踪表中删除。
net.netfilter.nf_conntrack_tcp_timeout_fin_wait=12 #TCP 连接在状态为 "FIN_WAIT"（等待远端关闭）时的超时时间（以秒为单位）。内核会根据超时时间来判断连接是否超时并将其从连接跟踪表中删除。
net.core.somaxconn =  326780		# 增加监听队列长度 系统中每个监听套接字（如 TCP 或 UDP）的最大挂起连接队列长度
vm.swappiness = 0					 # 关闭swap分区，降低系统对swap的使用，定义了系统对swap的使用倾向
kernel.pid_max=4194303				 # 系统可以分配的最大进程标识符（PID）数量
fs.inotify.max_user_watches=1048576  #Inotify 是 Linux 内核提供的一种机制，用于监视文件系统中的文件和目录的变化。通过增加 max_user_watches 参数的值，可以增加每个用户可以同时监视的文件和目录的数量。
fs.inotify.max_user_instances=8192 # 可以增加每个用户可以同时创建的 Inotify 实例的数量。
fs.inotify.max_queued_events=16384 # 可以增加每个 Inotify 实例可以排队的事件的数量。
kernel.sysrq = 1						 # 启用SysRq可以在系统出现问题时提供一种安全的操作方式。
kernel.softlockup_all_cpu_backtrace = 1 #启用或禁用内核在检测到软锁定（soft lockup）时记录所有 CPU 的回溯信息,软锁定指的是内核中某个 CPU 被长时间占用而无法响应其他任务的情况。当该参数设置为 1 时，内核会记录所有 CPU 的回溯信息，以便诊断软锁定问题
kernel.softlockup_panic=1 #当该参数设置为 1 时，内核会在检测到软锁定时触发系统崩溃，以确保系统可以尽快恢复正常运行。触发系统崩溃可以帮助诊断软锁定的原因，并采取相应的修复措施。

```



## 日志报错 fork：Cannot allocate memory （-bash: fork: 无法分配内存）处理 

系统内部的总进程数超过了系统默认得进程数

```shell
echo "kernel.pid_max=1000000 " >> /etc/sysctl.conf;sysctl -p
```



## kernel: TCP: time wait bucket table overflow 错误处理

处于 time wait 状态的 TCP 套接字，超过 net.ipv4.tcp_max_tw_buckets 限制。

```shell
vim /etc/sysctl.conf
net.ipv4.tcp_max_tw_buckets= 16000 #将该值增大
```



## request_sock_tcp possible syn flooding on port错误处理 和WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128. 错误处理

增加 TCP 连接队列的最大长度（tcp_max_syn_backlog）：

- `net.ipv4.tcp_max_syn_backlog`
  这个参数定义了 TCP 连接队列的最大长度。在高并发情况下，可以适当增大这个值，以防止连接队列溢出。

```shell
vim /etc/sysctl.conf
net.core.somaxconn = 2048
net.ipv4.tcp_max_syn_backlog = 2048
```

## TCP: out of memory -- consider tuning tcp_mem 错误处理

调整 TCP 接收缓冲区大小（tcp_rmem）和发送缓冲区大小（tcp_wmem）：

- `net.ipv4.tcp_rmem = min default max`
- `net.ipv4.tcp_wmem = min default max`
  这些参数定义了 TCP 接收和发送缓冲区的最小、默认和最大大小。您可以适当增大这些值，以满足高并发场景的需求。

```shell
vim /etc/sysctl.conf
net.ipv4.tcp_rmem = 4096	87380	6291456
net.ipv4.tcp_wmem = 4096    20480   4194304
net.ipv4.tcp_mem=8388608 8388608 8388608
```

## inotify watch 耗尽

每个 linux 进程可以持有多个 fd，每个 inotify 类型的 fd 可以 watch 多个目录，每个用户下所有进程 inotify 类型的 fd 可以 watch 的总目录数有个最大限制，这个限制可以通过内核参数配置: `fs.inotify.max_user_watches`

查看最大 inotify watch 数:

```
$ cat /proc/sys/fs/inotify/max_user_watches
```

```shell
vim /etc/sysctl.conf
fs.file-max=1100000
```

## Too many open files 错误

三个参数：fs.nr_open、nofile（分为soft nofile  和 hard nofile ） 和 fs.file-max。

其实在 Linux 上能打开多少个文件，限制有两种：

- 第一种，进程级别的，限制的是单个进程上可打开的文件数。具体参数是 soft nofile 和 fs.nr_open。它们两个的区别是 soft nofile 可以不同用户配置不同的值。而 fs.nr_open 在一台 Linux 上只能配一次。
- 第二种，系统级别的，整个系统上可打开的最大文件数，具体参数是fs.file-max。但是这个参数不限制 root 用户。

另外这几个参数之间还有耦合关系，因此还要注意以下三点：

- 1、如果你想加大 soft nofile,  那么 hard nofile 也需要一起调整。因为如果 hard nofile 设置的低， 你的 soft nofile 设置的再高都没用，实际生效的值会按二者里最低的来。
- 2、如果你加大了 hard nofile，那么 fs.nr_open 也都需要跟着一起调整。**如果不小心把 hard nofile 设置的比 fs.nr_open 大了，后果比较严重。会导致该用户无法登陆。如果设置的是 * 的话，那么所有的用户都无法登陆。**
- 3、还要注意如果你加大了 fs.nr_open，但是用的是 echo "xx" >  /proc/sys/fs/nr_open 的方式，刚改完你可能觉得没问题。只要机器一重启你的 fs.nr_open 设置就会失效，还是会无法登陆。

```shell
# vi /etc/sysctl.conf
fs.nr_open=1100000  #要比 hard nofile 大一点
fs.file-max=1100000 #多留点buffer
# sysctl -p
# vi /etc/security/limits.conf
*  soft  nofile  1000000
*  hard  nofile  1000000
```

将文件限制设置为无限制不是好主意

## /etc/security/limits.conf 文件解析

`/etc/security/limits.conf` 文件用于设置Linux系统中用户或进程的资源限制。它定义了用户或用户组可以使用的系统资源的上限，包括进程数量、文件打开数、内存限制等。通过修改该文件，可以对系统用户和进程进行限制和优化

常见的值及其说明：

- `soft`：软限制，表示当前生效的资源限制值。软限制是可修改的，但不能超过硬限制。
- `hard`：硬限制，表示软限制的上限。只有超级用户（root）可以增加硬限制。硬限制是系统的实际限制。
- `unlimited`：表示无限制。将资源限制设置为无限制可以允许用户或进程使用系统的最大资源量。
- 数值：具体的资源限制值。例如，`65535`表示设置资源限制为 65535。

具体对用户和进程的限制项：#常用值并不是全部

- **nofile（文件打开数限制）**：这个参数用于限制用户或进程可以打开的文件描述符数量。
- **nproc（进程数量限制）**：这个参数用于限制用户或进程可以创建的最大进程数量。
- **stack（线程栈大小限制）**：这个参数用于限制用户或进程的线程栈大小。
- **memlock（锁定内存限制）**：这个参数用于限制用户或进程可以锁定的内存大小。

```
#示例
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
* soft stack   unlimited
* hard stack   unlimited
```

