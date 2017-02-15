---
published: true
title: "JVM实践 - 堆调优"
date: 2016-03-05
layout: post
---


## 打印GC日志 ##
使用下面三个参数可以打印Oracle Hotspot JVM每次的GC日志。

```
-verbose:gc : 打印GC日志
-XX:+PrintGCDateStamps ：添加时间戳
-XX:+PrintGCDetails ：打印GC详细
```

同时将GC日志重定向到单独的log文件

```
-Xloggc:<file-path> ： 保存GC日志到单独文件，方便分析
```

GC的日志可以使用以下工具分析：

1. IBM GCMV (Garbage Collection and Memory Visualizer)
这款工具可以作为eclipse插件安装并切换到GCMV视图。GCMV同时支持Oracle Hotspot JVM 和 IBM， HP等JVM实现。

## Heap Dump文件 ##

### 生成hprof文件 ###

Heap dump文件在分析JVM性能，优化应用程序，检查内存泄漏时十分重要，尤其是在发生内存溢出（OutOfMemoryError）时更加必要。通常可以设置JVM参数在OOM时候触发生成Heap Dump文件。

```
-XX:+HeapDumpOnOutofMemoryError
```

或者添加触发命令：

```
-XX:OnOutOfMemoryError="jmap -F -dump:format=b,file=java_pid%p.hprof %p; chmod +r java_pid%p.hprof; kill -9 %p"
```

对于本地的JVM进程，可以使用JDk自带的GUI Debug工具如JVisualVM来生成.hprof文件。

有一类比较复杂的情况是，在生产环境的某台机器上出现了JVM异常，但是使用jmap命令十分耗时，这会影响应用程序的执行，造成很大的暂停，甚至jmap执行可能失败。这时候可以借助Linux/Unix的gdb工具生成core文件，再使用jmap从core文件中生成hpprof文件。生成core文件非常快，通常只需要几秒钟。以下是详细步骤：

```shell
# 1. install gdb
$sudo apt-get install gdb – for Debian/Ubuntu systems.
$sudo yum install gdb – for RedHat/CentOS systems.
# 2. get the Process ID of JVM
$ps -ef|egrep "PID|[keyword]"
# 3. get the core files
$gdb --pid=[pid]
...bunch of info...
$(gdb) gcore /tmp/jvm.core
Saved corefile /tmp/jvm.core
$(gdb) detach
$(gdb) quit
# 4. get the hprof binary dump for core file using jmap
$jmap -dump:format=b,file=jvm.hprof /usr/bin/java /tmp/jvm.core
```

[[SEE ALSO]use core file to generate heap dump](http://blogs.atlassian.com/2013/03/so-you-want-your-jvms-heap/ "use core file to generate heap dump")


### 分析heap dump

hprof文件可以使用JVisualVM或者MAT(Eclipse Memory Analyzer)来装载并分析。MAT工具的使用介绍。MAT会首先解析hprof文件并生成一系列的index文件，这个过程比较耗时，但是下一次加载时会载入上一次分析好的index文件。

[[SEE ALSO]MAT工具的使用介绍](http://eclipsesource.com/blogs/2013/01/21/10-tips-for-using-the-eclipse-memory-analyzer/)

因为Heap Dump是当前内存的一份快照，因此在线上机器生成的heap dump会很大，无法用MAT GUI工具装载。可以在线下某台拥有和线上同等配置的机器上使用MAT自带的ParseHeapDump.sh文件来生成索引文件，并生成一系列的report。这样就可以方便的在开发机器上用MAT GUI加载这些生成好的report文件了。

```
./ParseHeapDump.sh ../today_heap_dump/jvm.hprof org.eclipse.mat.api:suspects
```

其他可用的report参数有：

```
org.eclipse.mat.api:suspects 
org.eclipse.mat.api:overview
org.eclipse.mat.api:top_components 
```

另外一个工具为bheapsampler, 这个工具对heap dump文件进行抽象，生成 一份更简洁的heap dump文件，虽然有点不太准确但是在MAT失败的情况下总比没有好。

[[SEE ALSO]Analyzing Large Java Heap when MAT UI fails](http://javaforu.blogspot.com/2013/11/analyzing-large-java-heap-dumps-when.html)

## JVM 参数 ##

### -XX:-UseGCOverheadLimit ###

JVM的Parallel GC如果占用应用程序大部分的执行时间。将会抛出OutOfMemoryError：如果98%的时间都浪费在垃圾回收上，而且只有不到2%的堆内存被回收的话，JVM将会抛出OutOfMemoryError。这个设计的作用是防止应用程序因为堆大小不足导致浪费时间在GC上并且毫无进展。但是可以使用参数
_-XX:-UseGCOverheadLimit_来禁用该功能。
JVM的CMS GC也有相同的功能，不过只有造成应用程序暂停的GC时间才会被计算到98%这个指标中，这一类GC通常是因为在CMS GC发生Concurrent Mode Failure或者显式的GC调用时发生（System.gc()）。

[[SEE ALSO]Java SE 6 HotSpot[tm] Virtual Machine Garbage Collection Tuning](http://www.oracle.com/technetwork/java/javase/gc-tuning-6-140523.html "Java SE 6 HotSpot[tm] Virtual Machine Garbage Collection Tuning")
