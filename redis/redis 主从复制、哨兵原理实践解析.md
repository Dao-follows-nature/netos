# redis 主从复制、哨兵原理实践解析

## redis单节点架构容易造成：

- reids发生宕机，导致单点故障
- 磁盘故障，造成数据丢失

## 主从复制的特点：

- 主节点可读可写，从节点只读不允许写

- 主节点可以有多个从节点，从节点只能有一个主节点

- 数据流是从主节点到从节点单向的

## 主从部署

|      IP       |                 操作系统                 |    服务器配置     |
| :-----------: | :--------------------------------------: | :---------------: |
| 192.168.0.110 | ubuntu-20.04.5-LTS（redis-6.12.4）master | 内存：4G CPU：1核 |
| 192.168.0.107 |      rocky9.2（reids-6.12.4）slave       | 内存：4G CPU：1核 |

**主从复制，从 5.0.0 版本开始，Redis 正式将 SLAVEOF 命令改名成了 REPLICAOF 命令并逐渐废弃原来的 SLAVEOF 命令**

- 确保保持版本一致

- 确保保持内存一致，（如果无法保证，从节点可以大于主节点，但是主节点不能大于从节点）

master

```
#redis.conf

requirepass 123456  #配置密码  
repl-backlog-size 1mb  #设置复制缓冲区大小。backlog是一个缓冲区，这是一个环形复制缓冲区，用来保存最新复制的命令。当副本断开连接一段时间时，它会累积副本数据，因此当副本想要再次连接时，通常不需要完全重新同步，但部分重新同步就足够了，只需传递副本在断开连接时丢失的部分数据。复制缓冲区越大，复制副本能够承受断开连接的时间就越长，以后能够执行部分重新同步。只有在至少连接了一个复制副本的情况下，才会分配缓冲区，没有复制副本的一段时间，内存会被释放出来，默认1mb 
repl-backlog-ttl 60 #环形缓冲复制队列存活时长

repl-diskless-sync no #是否使用无盘同步，默认为no，no为不使用无盘，需要将RDB文件保存到磁盘后再发送给slave，yes 为支持无盘，RDB 不保存至本地磁盘，而且直接通过socket文件发送给slave
repl-diskless-sync-delay 5 #diskless时复制的服务器等待的延迟时间

repl-ping-replica-period 10 #slave端向master端发送ping的时间间隔
repl-timeout 60 #主从ping连接超时时间 

min-replicas-to-write 3 #设置一个master可用的slave不能少于多少个，否则master无法执行
min-replicas-max-lag 10 #设置至少有上面数量的slave延迟时间大于多少时，master不接受写操作（拒接写入）

repl-disable-tcp-nodelay no #TCP 的 TCP_NODELAY 属性，决定数据的发送时机。配置关闭：主节点产生的数据无论大小都会及时的发送给从节点。redis默认关闭此配置，以保障较小的主从延迟。当然，这需要主从间保持较好的网络状况。
配置打开：主节点会合并较小的TCP数据包以节省宽带，默认发送时间间隔由linux内核设置决定，默认一般40ms。虽然这样大大增加了主从之间的延迟，但是对于网络状况达不到条件或者对主从延迟不敏感的情况比较适用。

```

savle

```
#redis.conf
replicaof <masterip> <masterport> #连接那个主库 和主库端口
masterauth <masterpassword>  # 主库密码
```



## 全量同步（第一次同步）

![1705334487469](./images/1705334487469.png)

1、从库向主库发送psync命令，表示要进行数据同步，主库根据这个命令的参数 来启动复制。psync 命令包含了主库的 runID 和复制进度 offset 两个参数。 

- runID，是每个 Redis 实例启动时都会自动生成的一个随机 ID，用来唯一标记这个实例。当从库和主库第一次复制时，因为不知道主库的 runID，所以将 runID 设 为“？”。
- offset，此时设为 -1，表示第一次复制。

2、主库收到 psync 命令后，会用 FULLRESYNC 响应命令带上两个参数：主库 runID 和主库 目前的复制进度 offset，返回给从库。

3、从库收到响应后，会记录下这两个参数。

**FULLRESYNC 响应表示第一次复制采用的全量复制，也就是说， 主库会把当前所有的数据都复制给从库。**

4、主库执行 bgsave 命令，生成 RDB 文件

5、将RDB快照发给从库

6、从库清空当前数据库数据，在主库将数据同步给从库的过程中，主库不会被阻塞，仍然可以正常接收请求。但是，这些请求中的写操作并没有记录到刚刚生成的 RDB 文件 中。为了保证主从库的数据一致性，主库会在内存中用专门的 replication buffer，记录 RDB 文件生成后收到的所有写操作。



```
782:M 16 Jan 2024 22:09:33.794 * Replica 192.168.0.107:6379 asks for synchronization
782:M 16 Jan 2024 22:09:33.794 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for 'c793a2942cd3c00f55234d78c787f984960cecf2', my replication IDs are 'f0998e7efeaaaa3c4d9103099ce2ef89ea9b3d9e' and '0000000000000000000000000000000000000000')
782:M 16 Jan 2024 22:09:33.795 * Replication backlog created, my new replication IDs are 'd5f154f3c3da1e57e1f5001baf08177f5471a8d2' and '0000000000000000000000000000000000000000'
782:M 16 Jan 2024 22:09:33.795 * Starting BGSAVE for SYNC with target: disk
782:M 16 Jan 2024 22:09:33.795 * Background saving started by pid 2444
2444:C 16 Jan 2024 22:09:33.797 * DB saved on disk
2444:C 16 Jan 2024 22:09:33.797 * RDB: 0 MB of memory used by copy-on-write
782:M 16 Jan 2024 22:09:33.884 * Background saving terminated with success
782:M 16 Jan 2024 22:09:33.884 * Synchronization with replica 192.168.0.107:6379 succeeded

```



## 增量同步

从 Redis 2.8 开始，网络断了之后，主从库会采用增量复制的方式继续同步。全量复制是同步所有数据，而增量复制只会把主从库网 络断连期间主库收到的命令，同步给从库。

![1705411833318](./images/1705411833318.png)

1、从节点连接关闭后，主节点会把写操作记录在收到的写操作命令，写入 replication buffer，同时也 会把这些操作命令也写入 repl_backlog_buffer 这个缓冲区。

2、连接恢复后从库首先会给主库发送 psync 命令，并把自己当前的 slave_repl_offset 发给主库，主库会判断自己的 master_repl_offset 和 slave_repl_offset 之间的差距。然后把 master_repl_offset 和 slave_repl_offset 之间的命令操作同步给从库就行。



repl_backlog_buffer 是一个环形缓冲区，所以在 缓冲区写满后，主库会继续写入，此时，就会覆盖掉之前写入的操作。如果从库的读取速 度比较慢，就有可能导致从库还未读取的操作被主库新写的操作覆盖了，这会导致主从库 间的数据不一致。



repl-backlog-size：允许从节点最大失联多长时间*主节点offset每秒写入量



```
782:M 16 Jan 2024 22:10:47.641 # Connection with replica 192.168.0.107:6379 lost.
782:M 16 Jan 2024 22:11:01.040 * Replica 192.168.0.107:6379 asks for synchronization
782:M 16 Jan 2024 22:11:01.040 * Partial resynchronization request from 192.168.0.107:6379 accepted. Sending 165 bytes of backlog starting from offset 1.
```



查看主从复制

```
INFO replication
```

```
role:master  #指示 Redis 服务器当前的角色（role）master"（主节点）或 "slave"（从节点）。
connected_slaves:1 #表示当前连接的从节点数量
slave0:ip=192.168.0.107,port=6379,state=online,offset=823,lag=1 
master_failover_state:no-failover 
master_replid:d5f154f3c3da1e57e1f5001baf08177f5471a8d2 #主节点的复制 ID（replication ID）。
master_replid2:0000000000000000000000000000000000000000 #主节点的复制 ID（replication ID）的辅助 ID。
master_repl_offset:823 #主节点的复制偏移量（replication offset）。
second_repl_offset:-1 #从节点的复制偏移量（replication offset）。
repl_backlog_active:1  #表示复制 backlog 是否处于活跃状态。如果值为 "0"，表示复制 backlog 未激活；如果值为 "1"，表示复制 backlog 已激活。
repl_backlog_size:1048576 #复制 backlog 的大小（以字节为单位）。
repl_backlog_first_byte_offset:1 #复制 backlog 的第一个字节的偏移量。
repl_backlog_histlen:823 #复制 backlog 当前的历史长度（以字节为单位）

```

为什么使用RDB 来进行主从同步，而不是AOF日志

- 因为AOF文件比RDB文件大，网络传输比较耗时
- 从库在初始化数据时，RDB文件比AOF文件执行更快

# redis  哨兵（Sentinel）



