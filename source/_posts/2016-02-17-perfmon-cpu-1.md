---
layout: post
title: "性能调优 - Part 1( CPU监控 )"
published: true
tags:
	- JVM
categories:
	- Java
---

在实际的线上环境中，经常需要针对线上问题找出瓶颈，这里需要用到一些Linux用于性能监控和分析的命令，并需要一些指导性的原则结合这些命令的输出来得出正确的结论。这篇文章主要针对CPU资源消耗的分析。

在Linux中, CPU主要用于中断，内核，以及用户进程的任务处理。优先级为**中断>内核>用户进程**。关于CPU资源有以下三个重要的概念：
<!-- more -->
- 上下文切换 Context Switching

	Linux是采取的抢占式调度执行任务，每个CPU统一时刻只能执行一个线程（有些CPU可以通过超线程技术同时执行多个线程）。当到达执行是相见，线程中有IO阻塞或者高优先级的线程要执行时，Linux就会切换执行的线程，这个过程需要保存当前线程的执行状态，并恢复要执行的线程的状态，这个过程叫做上下文切换。
    
    在Java应用中，典型的如进行IO操作（网络或者文件），锁等待，或者线程sleep时，当前线程会进入阻塞或者休眠状态，触发上下文切换。频繁的上下文切换会造成内核占据较多的CPU使用，使得应用的响应速度下降。
    
    CS数目可以通过`vmstat` 和 `pidstat` 命令查看。
    
- 运行队列 System Load
	
    每个CPU都维护了一个可运行的线程队列，例如一个某台机器共有四个单核CPU，Java应用共启动了8个线程，且都处于可运行状态，则平均下来每个CPU的的运行队列就有两个线程。 运行队列值越大， 意味着线程要消耗更长的时间才能执行完， 建议每个CPU核心上的运行队列的值在1~3个。
    
    运行队列可以通过`top`命令查看。
 
- 利用率
 
 	CPU利用率为CPU在用户进程，内核、中断处理、IO等待以及空闲五个部分的使用百分比，这是五个分析CPU消耗情况的关键指标。建议用户进程的CPU消耗为65%~70%, 内核态为30%~35%左右。
    
    CPU利用率可以通过`top` `vmstat` 和 `pidstat`查看。
	

### 1. top ###

top命令类似于windows的资源监控器，用于监控整个系统的CPU资源的使用情况，并可以排序动态刷新状态，提供交互式的命令。

![top命令](/images/top.png)

1. _load average_

	表示1min, 5min, 15min到现在的平均值，注意这里对于多核CPU的情况，需要除以核心数目才能得到每个CPU核心上的运行队列的大小。

```shell
[work@44224012 ~]$ nproc
24           
```

2. _us, sy, ni, id, wa ,hi, si_

当CPU消耗严重时，主要体现在us，sy, wa 或者hi的值变高，wa的值主要是CPU时间片内的IO等待造成的，hi的值主要是硬件中断造成的，如频繁的网卡接收数据。 


```
us	用户进程处理所占的百分比
sy	内核态下操作系统线程处理所占的百分比
ni	被nice命令改变优先级的任务所占的百分比
id  CPU的空闲时间所占的百分比
wa  CPU的执行过程中等待IO所占的百分比， 这个指标用来总和判断IO消耗情况。
hi	硬件中断所占的百分比，一般由文件或者网络IO引起。
si	软件中断所占的百分比
```


us高(大于建议的70%)，表示应用消耗了大部分的CPU时间， 主要是线程一直处于可运行状态，通常是在执行无阻塞、循环、正则或者纯粹的计算任务造成的。另一个造成us高的原因是频繁的GC。这里需要结合top的输出找到最耗CPU的线程，再结合jstack的输找到对应的线程。

sy高(大于建议的35%),可能是Linux花费了更多的时间来进行线程切换，这些线程多数处于不断的阻塞（锁等待，IO等待）和执行状态的变化过程中，对于Java应用重要的是找出这种状态的线程。如果很多的线程处于TIMED_WAITING（on object monitor）状态和Runnable状态的切换中，通过on obejct monitor的对应的堆栈信息可以找到系统中锁竞争激烈的代码，这造成了系统频繁的上下文切换。

```bash
#jstack -l <pid> : print additonal information about locks
```
    
3. 交互命令

_查看每个线程的CPU使用情况_ : shift+h, 此时pid显示的是十进制的线程tid

_查看每个核上的cpu消耗情况_ ： 进入top后按1


### 2. pidstat ###

pidstat最大的好处是可以具体到每个线程上的资源信息，包括CPU信息， IO信息。

```shell
# 显示pid为463的进程上所有线程(-t参数)的CPU使用情况，采样间隔为1s，采样次数为5
[work@44224012 ~]$ pidstat -p 463 -t  1 5                                                                                                                 
Linux 2.6.32_1-15-0-0 (44224012.aaf236c9-f59c-4fee-8d40-80abc7cdde60.jpaas.guid)        05/02/2016      _x86_64_        (24 CPU)                                                                                                                                                                                    
01:50:50 PM      TGID       TID    %usr %system  %guest    %CPU   CPU  Command                                                                            
01:50:51 PM       463         -    0.99    0.00    0.00    0.99    18  java                                                                               
01:50:51 PM         -       463    0.00    0.00    0.00    0.00    18  |__java                                                                            
01:50:51 PM         -       470    0.00    0.00    0.00    0.00     9  |__java                                                                            
01:50:51 PM         -       471    0.00    0.00    0.00    0.00    13  |__java                                                                            
01:50:51 PM         -       472    0.00    0.00    0.00    0.00     1  |__java                                                                            
01:50:51 PM         -       473    0.00    0.00    0.00    0.00     0  |__java
...

```

`pidstat` 可以打印出进程的context switch的信息，对于判断sy使用率过高的情况很有帮助。

```bash
pidstat -w -p <pid>
```
1. **cswch/s**

	Total number of voluntary context switches the task made per second.  A voluntary context switch occurs when a task blocks because it requires a resource that is unavailable.
    
2. **nvcswch/s**

	Total  number  of  non voluntary context switches the task made per second.  A involuntary context switch takes place when a task executes for the duration of its time slice and then is forced to relinquish the processor.



### 3. vmstat ###

`vmstat` 可以用来查看系统层级context switch的信息， 以固定时间间隔对系统采样，打印如下信息。

```bash
[work@44224012 ~]$ vmstat 1 5                                                                                                                             
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu-----                                                                          
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st                                                                          
 2  0      0 40835040 667160 32114372    0    0     6    70    0    0  4  6 89  0  0                                                                      
 3  0      0 40827760 667160 32113680    0    0     0    44 127855 253237  6  8 86  0  0                                                                  
11  0      0 40826772 667160 32114644    0    0     0   132 135581 260254  7 11 82  0  0                                                                  
 7  0      0 40828832 667160 32115720    0    0     0   160 132777 257887  8 11 82  0  0                                                                  
 4  1      0 40826496 667160 32116412    0    0     0  2132 132769 257816  9 12 79  1  0    
```
