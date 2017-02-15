---
published: true
title: "JVM实践 - JDK调试工具"
layout: post
tags:
	- JVM
categories:
	- Java
---

JDK自带的一些诊断工具可以方便的在线上机器快速获得一些信息，对定位线上内存和CPU异常问题有很大的帮助。这些工具包括命令行工具和用来调试本地JVM的图形界面工具。命令行工具在Linux机器上使用比较方便。
<!-- more -->
### jinfo

jinfo可以用来获取JVM的启动参数以及System Properties, 对于某些特定参数可以使用jinfo修改值。一个小建议，对于不记得的参数可以先使用` java -XX:+PrintFlagsFinal`获得JVM所有的参数名，然后找到你想要的参数,使用`jinfo -flag`打印出该参数的值。

```
Usage:                                                                                           
    jinfo [option] <pid>                                                                             
        (to connect to running process)                                                                   
    jinfo [option] <executable <core>                                                                     
        (to connect to a core file)                                                                     
    jinfo [option] [server_id@]<remote server IP or hostname>                                                      
        (to connect to remote debug server)                                                               
where <option> is one of:                                                                             
    -flag <name>         to print the value of the named VM flag                                                 
    -flag [+|-]<name>    to enable or disable the named VM flag                                                   
    -flag <name>=<value> to set the named VM flag to the given value                                                 
    -flags               to print VM flags                                                              
    -sysprops            to print Java system properties                                                     
    <no option>          to print both of the above                                                         
    -h | -help           to print this help message  
```

### jstat

jstat主要利用JVM内建的指令对Java应用程序的资源和性能进行实时的命令行的监控，包括了对Heap size和垃圾回收状况的监控。主要通过采样的方式以一定时间间隔打印JVM的信息，对于在线分析JVM的性能十分有帮助。使用jstat还可以直接修改运行中的JVM的参数，主要是一些打印GC日志之类的参数。

[jstat命令详解](http://blog.csdn.net/zhaozheng7758/article/details/8623549)

```
Usage: jstat -help|-options
       jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]] 
Definitions:                                                                                                                                              
  <option>      An option reported by the -options option                                                                                                 
  <vmid>        Virtual Machine Identifier. A vmid takes the following form:                                                                              
                     <lvmid>[@<hostname>[:<port>]]                                                                                                        
                Where <lvmid> is the local vm identifier for the target                                                                                   
                Java virtual machine, typically a process id; <hostname> is                                                                               
                the name of the host running the target Java virtual machine;                                                                             
                and <port> is the port number for the rmiregistry on the                                                                                  
                target host. See the jvmstat documentation for a more complete                                                                            
                description of the Virtual Machine Identifier.                                                                                            
  <lines>       Number of samples between header lines.                                                                                                   
  <interval>    Sampling interval. The following forms are allowed:                                                                                       
                    <n>["ms"|"s"]                                                                                                                         
                Where <n> is an integer and the suffix specifies the units as                                                                             
                milliseconds("ms") or seconds("s"). The default units are "ms".                                                                           
  <count>       Number of samples to take before terminating.                                                                                             
  -J<flag>      Pass <flag> directly to the runtime system. 
  
```

jstat使用一组options控制输出的内容，如下所示:

```
[work@44224014 ~]$ jstat -options                                            
-class                 #类加载器                              
-compiler               #JIT Compiler                                
-gc                   #GC堆状态                                 
-gccapacity              #各区大小                                     
-gccause                #最后一次GC统计和原因                                  
-gcnew                 #新区统计                                 
-gcnewcapacity            #新区大小                                      
-gcold                 #旧区统计                                  
-gcoldcapacity            #旧区大小                                      
-gcpermcapacity           #永久去大小                                   
-gcutil                 #GC统计汇总                                   
-printcompilation          #Hotspot编译统计
```

1. `stat –class<pid>` : 显示加载class的数量，及所占空间等信息

| 显示列名| 具体描述| 
| ------- |:-------:|
| Loaded | 装载的类的数量 |
| Bytes| 装载类所占用的字节数|
| Unloaded | 卸载类的数量| 
| Bytes | 卸载类的字节数| 
| Time | 装载和卸载类所花费的时间| 

2.  `jstat -gcutil <pid>` : 统计gc信息，以百分比方式

| 显示列名| 具体描述| 
| ------- |:-------:|
| S0  | 年轻代中第一个survivor（幸存区）已使用的占当前容量百分比 |
| S1| 年轻代中第二个survivor（幸存区）已使用的占当前容量百分比, 用于拷贝存活对象|
| E | 年轻代中Eden Space已使用的占当前容量百分比| 
| O | Old代已使用的占当前容量百分比| 
| P | Perm代已使用的占当前容量百分比| 
| YGC|从程序启动到采样时年轻代gc次数| 
| YGCT| 从应用程序启动到采样时年轻代中gc所用时间(s)| 
| FGC   |从程序启动到采样时Full GC gc次数| 
| FGCT    | 从应用程序启动到采样时Full GC所用时间(s)| 
| GCT    | 从应用程序启动到采样时GC所用时间(s)| 

3. `jstat -gc <pid>` : 统计GC信息， 以具体大小方式

```
-gc Option
Garbage-collected heap statistics 
Column  Description
S0C     Current survivor space 0 capacity (KB).
S1C     Current survivor space 1 capacity (KB).
S0U     Survivor space 0 utilization (KB).
S1U     Survivor space 1 utilization (KB).
EC      Current eden space capacity (KB).
EU      Eden space utilization (KB).
OC      Current old space capacity (KB).
OU      Old space utilization (KB).
PC      Current permanent space capacity (KB).
PU      Permanent space utilization (KB).
YGC     Number of young generation GC Events.
YGCT    Young generation garbage collection time.
FGC     Number of full GC events.
FGCT    Full garbage collection time.
GCT     Total garbage collection time.
```
