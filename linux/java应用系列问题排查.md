# java应用系列问题排查

查看  Java 堆详细信息

```shell
jmap -heap {PID} #java 8
```

```shell
jhsdb jmap --heap --pid {PID} # java 11
```

![image-20240410114531645](E:\笔记\linux\images\image-20240410114531645.png)

**Heap Usage**：

- `regions`：堆内存被划分为的区域总数。
- `capacity`：堆内存的总容量。
- `used`：当前已使用的堆内存量。
- `free`：当前堆内存的空闲量。
- `used%`：当前已使用堆内存的百分比。



查看线程

{PID} 为java 得进程iD号，最后获得的进程id 是16进制的

```shell
ps -mp {PID} -o THREAD,tid | awk 'NR!=1 && NR!=2 { printf "%s %d\n",$2,$8 }' | sort -rn | head -5
```

抓取线程日志信息

```shell
jstack -l {pid}  > test.log 
```

查看垃圾回收

```shell
jstat -gcutil {PID} 2000 10
```

## 关于调优

> 注意一下参数请你自己根据自己的情况自己更改参数

### 传统启动

```shell
java -jar -Xms16G -Xmx16G -Xmn4G -XX:MetaspaceSize=512m  -XX:MaxMetaspaceSize=2048m  -XX:SurvivorRatio=10 -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=70 -XX:+UseCMSInitiatingOccupancyOnly -Djava.awt.headless=true -XX:+DisableExplicitGC
```

参数详解：

`-Xms`: 初始堆内存大小，堆内存设置为物理内存的3/4左右

`-Xmx`: 最大堆内存大小，堆内存设置为物理内存的3/4左右

`-Xmn`: 年轻代的初始和最大堆内存大小。如果不设置 `-Xmn`，JVM 会根据整个堆的大小和其他相关参数自动计算年轻代的大小。经验法则是将年轻代的大小设置为整个堆大小的 1/3 到 1/4

`-XX:MetaspaceSize`: 设置元空间的初始大小

`-XX:MaxMetaspaceSize`: 设置元空间的最大大小

`-XX:SurvivorRatio=10`: 定义了年轻代中的Eden区与一个Survivor区的大小比例。`SurvivorRatio=10`意味着Eden区的大小是Survivor区大小的10倍。这个比例会影响垃圾回收时对象晋升到老年代之前在年轻代中的生存空间。调整这个比例可以帮助优化垃圾回收性能，减少内存浪费，并可能减少老年代的垃圾回收频率。

`-XX:+UseConcMarkSweepGC`:启用了并发标记清除（CMS）垃圾收集器。CMS是一种以减少应用程序暂停时间为目标的垃圾收集器，它通过并发地标记和清除垃圾来减少GC停顿时间。CMS通常用于需要快速响应和减少停顿时间的应用程序。

`-XX:CMSInitiatingOccupancyFraction=70`: 设置了CMS垃圾收集器开始执行老年代回收的阈值。当老年代的使用率达到70%时，CMS将开始执行垃圾回收。

`-XX:+UseCMSInitiatingOccupancyOnly`:JVM将仅使用`CMSInitiatingOccupancyFraction`参数设置的阈值来触发CMS垃圾回收，而不会考虑其他可能的GC触发条件。这有助于简化GC触发机制，使得GC行为更加可预测。

`-Djava.awt.headless=true`:设置此属性可以避免JVM加载与GUI相关的类和资源，从而减少内存占用和启动时间。

`-XX:+DisableExplicitGC`:启用此参数后，JVM将忽略应用程序代码中显式的`System.gc()`调用。这可以防止应用程序代码中的GC请求影响JVM的垃圾回收策略和性能。

### 容器启动

```dockerfile
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:InitialRAMPercentage=80.0 -XX:MinRAMPercentage=80.0 -XX:MaxRAMPercentage=80.0 -XX:-UseAdaptiveSizePolicy"
ENTRYPOINT ["java","-jar $JAVA_OPTS","/test.jar"]
```

参数详解：

`-XX:+UseContainerSupport`:在容器化环境中，通常需要限制应用程序使用的资源量，以避免对容器管理系统或其他容器造成影响。启用`UseContainerSupport`参数后，JVM会更好地与容器的资源限制配合工作。

`-XX:InitialRAMPercentage=80.0`: JVM启动时的初始堆内存占用百分比。`80.0`表示JVM将根据系统可用内存的80%来设置初始堆大小。这对于动态分配内存的场景非常有用，尤其是在具有大量可用内存的系统上，可以避免JVM一开始就占用过多内存。

`-XX:MinRAMPercentage=80.0`:JVM堆内存的最小百分比。设置为`80.0`意味着JVM将确保至少使用系统可用内存的80%作为堆内存。

`-XX:MaxRAMPercentage=80.0`:JVM堆内存的最大百分比,JVM将确保最大使用系统可用内存的80%作为堆内存，防止JVM在内存资源有限的环境中过度消耗内存，从而影响系统的稳定性。

`-XX:-UseAdaptiveSizePolicy`：禁用 JVM 自适应大小策略的参数，JVM的堆内存分配将不会根据运行时的性能数据进行动态调整。



开启gc日志

```
-XX:+UseGCLogFileRotation
-XX:NumberofGcLogFiles=3
-XX:GCLogFilesize=28M
-Xloggc:/opt/logs/gc/project_name-gc-%t.log
-XX+PrintGCDetails
-XX+PrintGCDatestamps
-XX+PrintGCcause
-XX:+HeapDumpOnOutOfMemoryError
```

`-XX:+UseGCLogFileRotation`：启用GC日志文件的自动轮换功能。这意味着当GC日志文件达到一定大小时，会自动创建一个新的日志文件，而不是覆盖旧的日志文件。

`-XX:NumberOfGCLogFiles=3`：设置GC日志文件的最大数量，`3` 表示JVM将保留最近的3个GC日志文件。当达到这个数量时，最旧的日志文件将被删除以为新文件腾出空间。

`-XX:GCLogFilesize=28M`: 设置单个GC日志文件的最大大小,`28M` 表示每个GC日志文件的大小上限为28兆字节（MB）。当文件达到这个大小时，将触发日志文件轮换。

`-Xloggc:/opt/logs/gc/project_name-gc-%t.log`：指定GC日志文件的存储路径和文件名文件名为 project_name-gc-<timestamp>.log，其中 <timestamp> 是文件创建时的时间戳。

`-XX+PrintGCDetails`:启用GC日志的详细输出。启用后，每次GC事件发生时，JVM都会在日志中记录详细信息，包括GC的类型（Minor GC/Full GC）、GC前后的堆内存使用情况、GC持续时间等。

`-XX:+PrintGCDateStamps`: 在GC日志中添加日期和时间戳。这有助于分析日志时确定GC事件的确切时间，从而更好地理解GC行为和性能问题。

`-XX:+PrintGCcause`: 在GC日志中记录导致GC事件发生的原因。

`-XX:+HeapDumpOnOutOfMemoryError`:在发生内存溢出（OutOfMemoryError）时自动生成堆内存转储（Heap Dump）文件的功能。堆内存转储文件包含了JVM堆的快照，可以用于后续的内存泄漏分析和问题诊断。
