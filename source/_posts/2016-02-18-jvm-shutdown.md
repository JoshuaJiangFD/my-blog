---
layout: post
title: "JVM 实践 - 优雅停机 "
published: true
---



**引言**

在线上环境中，遇到一个定时任务无法退出的问题。观察日志发现Spring的Bean已经完成销毁，但是通过`ps aux |grep <keyword>` 观察发现进程仍然执行。 通过`jstack <pid>`工具查看所有的线程栈，线程堆栈输出如下：
<!-- more -->
```
2016-02-16 15:09:58
Full thread dump Java HotSpot(TM) 64-Bit Server VM (17.0-b17 mixed mode):

"Attach Listener" daemon prio=10 tid=0x000000004011c800 nid=0x1c73 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"DestroyJavaVM" prio=10 tid=0x00007f3548c61000 nid=0x7dfd waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"pa-sql-cache-0" prio=10 tid=0x00007f3547d71800 nid=0x7e2c waiting on condition [0x00000000418e0000]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00007f35ad0eb440> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:198)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2025)
        at java.util.concurrent.DelayQueue.take(DelayQueue.java:164)
        at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:583)
        at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:576)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:947)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:907)
        at java.lang.Thread.run(Thread.java:619)

"Low Memory Detector" daemon prio=10 tid=0x00007f354cb76000 nid=0x7e0e runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"CompilerThread1" daemon prio=10 tid=0x00007f354cb73800 nid=0x7e0d waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"CompilerThread0" daemon prio=10 tid=0x00007f354cb70800 nid=0x7e0c waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Signal Dispatcher" daemon prio=10 tid=0x00007f354cb6e800 nid=0x7e0b runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Finalizer" daemon prio=10 tid=0x00007f354cb50000 nid=0x7e0a in Object.wait() [0x00000000417df000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00007f35b7838178> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:118)
        - locked <0x00007f35b7838178> (a java.lang.ref.ReferenceQueue$Lock)
        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:134)
        at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:159)

"Reference Handler" daemon prio=10 tid=0x00007f354cb4e000 nid=0x7e09 in Object.wait() [0x0000000040c84000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x00007f35b7850150> (a java.lang.ref.Reference$Lock)
        at java.lang.Object.wait(Object.java:485)
        at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:116)
        - locked <0x00007f35b7850150> (a java.lang.ref.Reference$Lock)

"VM Thread" prio=10 tid=0x00007f354cb4a000 nid=0x7e08 runnable 

"GC task thread#0 (ParallelGC)" prio=10 tid=0x000000004012f800 nid=0x7dfe runnable 

"GC task thread#1 (ParallelGC)" prio=10 tid=0x0000000040131000 nid=0x7dff runnable 

....<skipped> 

"GC task thread#9 (ParallelGC)" prio=10 tid=0x00007f354cb04800 nid=0x7e07 runnable 

"VM Periodic Task Thread" prio=10 tid=0x00007f354cb81000 nid=0x7e0f waiting on condition 

JNI global references: 1761
```
输出发现有一个non-daemon线程 *pa-sql-cache-0* 仍然在TIMED_WAITING状态，除此之外都是daemon线程和JVM自身的GC线程等。通过代码分析，发现是一个在一个Spring Bean对象内部初始化的线程池，但是Spring在销毁Bean的时候，没有在`@PreDestory`钩子方法中关闭掉这个线程池。
这篇博客是对相关知识点的总结。

**JVM优雅关机**


**Reference**

[How to gracefully handle the SIGKILL signal in Java-stackoverflow](http://stackoverflow.com/questions/2541597/how-to-gracefully-handle-the-sigkill-signal-in-java)

[Troubleshooting Guide -Hotspot JVM](http://www.oracle.com/technetwork/java/javase/index-137495.html)

[Linux Process Singals](http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_12_01.html)
