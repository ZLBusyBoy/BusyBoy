# CPU过高的分析与解决方案
## 问题描述
&nbsp;&nbsp;&nbsp;&nbsp;<font  size =3>服务器是8核32G的，也就是说同时可用的共有8个CPU，一个CPU可以使用高达100%，8个CPU的话可以高达800%。前两天发现了一个CPU过高的问题，平时项目运行CPU也就是在10%，但是前两天发布之后突然发现CPU一直在200%左右打转，一直稳高不降。下面的例子只是参考（当时的情况没有截图o(╯□╰)o）。执行**`top`**命令查看占用CPU高的进程。</font>

```
top - 13:30:14 up 611 days,  2:06,  1 user,  load average: 6.77, 6.53, 6.34
Tasks: 137 total,   1 running, 136 sleeping,   0 stopped,   0 zombie
%Cpu(s): 74.8 us,  2.8 sy,  0.0 ni, 21.6 id,  0.0 wa,  0.0 hi,  0.8 si,  0.0 st
KiB Mem:  32783044 total, 32165440 used,   617604 free,   117916 buffers
KiB Swap:        0 total,        0 used,        0 free.  4187724 cached Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                           
 6795 root      20   0 27.299g 0.021t   7160 S 625.9 70.1  22775:31 java                                                                                              
23631 root      20   0 1459736 1.258g    608 S   1.3  4.0   1535:06 redis-server                                                                                      
 8005 mysql     20   0 3417268 1.200g      0 S   0.0  3.8   1336:47 mysqld                                                                                            
    1 root      20   0   94680  56000   1312 S   0.0  0.2  15:10.94 systemd                                                                                           
32087 root      20   0  171148  10364   4192 S   1.0  0.0 134:42.99 AliYunDun                                                                                         
 4816 root      20   0   36792   7992   7708 S   0.0  0.0   9:03.10 systemd-journal                                                                                   
 9160 polkitd   20   0  524708   6388    836 S   0.0  0.0   3:33.12 polkitd                                                                                           
  678 root      20   0  725108   6036   5204 S   0.0  0.0   7:49.42 rsyslogd                                                                                          
32525 root      20   0  141228   5372   4108 S   0.0  0.0   0:00.16 sshd                                                                                              
32527 root      20   0  116544   3392   1780 S   0.0  0.0   0:00.04 bash                                                                                              
18526 root      20   0  116648   1684      4 S   0.0  0.0   0:00.28 bash                                                                                              

```

## 解决方案
&nbsp;&nbsp;&nbsp;&nbsp;<font  size =3></font>
&nbsp;&nbsp;&nbsp;&nbsp;<font  size =3>由上图我们可以看到**`Java`**进程占用的CPU高达600%，但具体是因为那些任务造成的在这是无法直接看出来的。</font>

- &nbsp;&nbsp;&nbsp;&nbsp;<font  size =3 color = '3A5FCD'>一、打印线程堆栈信息 ---- jstack命令</font>

&nbsp;&nbsp;&nbsp;&nbsp;<font  size =3>jstack命令可用于导出Java应用程序的线程堆栈信息，可以使用重定向将输出保存到文件中。</font>

```
jstack 6795 > jstack6795
```
```
"Low Memory Detector" daemon prio=10 tid=0x081465f8 nid=0x7 runnable [0x00000000..0x00000000]  
        "CompilerThread0" daemon prio=10 tid=0x08143c58 nid=0x6 waiting on condition [0x00000000..0xfb5fd798]  
        "Signal Dispatcher" daemon prio=10 tid=0x08142f08 nid=0x5 waiting on condition [0x00000000..0x00000000]  
        "Finalizer" daemon prio=10 tid=0x08137ca0 nid=0x4 in Object.wait() [0xfbeed000..0xfbeeddb8]  
  
        at java.lang.Object.wait(Native Method)  
  
        - waiting on <0xef600848> (a java.lang.ref.ReferenceQueue$Lock)  
  
        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:116)  
  
        - locked <0xef600848> (a java.lang.ref.ReferenceQueue$Lock)  
  
        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:132)  
  
        at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:159)  
  
        "Reference Handler" daemon prio=10 tid=0x081370f0 nid=0x3 in Object.wait() [0xfbf4a000..0xfbf4aa38]  
  
        at java.lang.Object.wait(Native Method)  
  
        - waiting on <0xef600758> (a java.lang.ref.Reference$Lock)  
  
        at java.lang.Object.wait(Object.java:474)  
  
        at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:116)  
  
        - locked <0xef600758> (a java.lang.ref.Reference$Lock)  
  
        "VM Thread" prio=10 tid=0x08134878 nid=0x2 runnable  
  
        "VM Periodic Task Thread" prio=10 tid=0x08147768 nid=0x8 waiting on condition 
```

&nbsp;&nbsp;&nbsp;&nbsp;<font  size =3>在打印的堆栈日志文件中，**`tid`**和**`nid`**的含义：</font>

```
nid : 对应的Linux操作系统下的tid线程号，也就是前面转化的16进制数字 
tid: 这个应该是jvm的jmm内存规范中的唯一地址定位
```

&nbsp;&nbsp;&nbsp;&nbsp;<font  size =3>在CPU过高的情况下，查找响应的线程，一般定位都是用**`nid`**来定位的。而如果发生死锁之类的问题，一般用**`tid`**来定位。</font>

- &nbsp;&nbsp;&nbsp;&nbsp;<font  size =3 color = '3A5FCD'>定位CPU高的线程打印其nid</font>

&nbsp;&nbsp;&nbsp;&nbsp;<font  size =3>查看线程下具体进程信息的命令如下：</font>

```
top -H -p 6735 
```

```
top - 14:20:09 up 611 days,  2:56,  1 user,  load average: 13.19, 7.76, 7.82
Threads: 6991 total,  17 running, 6974 sleeping,   0 stopped,   0 zombie
%Cpu(s): 90.4 us,  2.1 sy,  0.0 ni,  7.0 id,  0.0 wa,  0.0 hi,  0.4 si,  0.0 st
KiB Mem:  32783044 total, 32505008 used,   278036 free,   120304 buffers
KiB Swap:        0 total,        0 used,        0 free.  4497428 cached Mem

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                                            
 6800 root      20   0 27.299g 0.021t   7172 S 54.7 70.1 187:55.61 java                                                                                               
 6803 root      20   0 27.299g 0.021t   7172 S 54.4 70.1 187:52.59 java                                                                                               
 6798 root      20   0 27.299g 0.021t   7172 S 53.7 70.1 187:55.08 java                                                                                               
 6801 root      20   0 27.299g 0.021t   7172 S 53.7 70.1 187:55.25 java                                                                                               
 6797 root      20   0 27.299g 0.021t   7172 S 53.1 70.1 187:52.78 java                                                                                               
 6804 root      20   0 27.299g 0.021t   7172 S 53.1 70.1 187:55.76 java                                                                                               
 6802 root      20   0 27.299g 0.021t   7172 S 52.1 70.1 187:54.79 java                                                                                               
 6799 root      20   0 27.299g 0.021t   7172 S 51.8 70.1 187:53.36 java                                                                                               
 6807 root      20   0 27.299g 0.021t   7172 S 13.6 70.1  48:58.60 java                                                                                               
11014 root      20   0 27.299g 0.021t   7172 R  8.4 70.1   8:00.32 java                                                                                               
10642 root      20   0 27.299g 0.021t   7172 R  6.5 70.1   6:32.06 java                                                                                               
 6808 root      20   0 27.299g 0.021t   7172 S  6.1 70.1 159:08.40 java                                                                                               
11315 root      20   0 27.299g 0.021t   7172 S  3.9 70.1   5:54.10 java                                                                                               
12545 root      20   0 27.299g 0.021t   7172 S  3.9 70.1   6:55.48 java                                                                                               
23353 root      20   0 27.299g 0.021t   7172 S  3.9 70.1   2:20.55 java                                                                                               
24868 root      20   0 27.299g 0.021t   7172 S  3.9 70.1   2:12.46 java                                                                                               
 9146 root      20   0 27.299g 0.021t   7172 S  3.6 70.1   7:42.72 java   
```

&nbsp;&nbsp;&nbsp;&nbsp;<font  size =3>由此可以看出占用CPU较高的线程，但是这些还不高，无法直接定位到具体的类。`nid`是16进制的，所以我们要获取线程的16进制ID：</font>

```
printf "%x\n" 6800 
```

```
输出结果:45cd
```

&nbsp;&nbsp;&nbsp;&nbsp;<font  size =3>然后根据输出结果到jstack打印的堆栈日志中查定位：</font>

```
"catalina-exec-5692" daemon prio=10 tid=0x00007f3b05013800 nid=0x45cd waiting on condition [0x00007f3ae08e3000]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000006a7800598> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:226)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2082)
        at java.util.concurrent.LinkedBlockingQueue.poll(LinkedBlockingQueue.java:467)
        at org.apache.tomcat.util.threads.TaskQueue.poll(TaskQueue.java:86)
        at org.apache.tomcat.util.threads.TaskQueue.poll(TaskQueue.java:32)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
        at java.lang.Thread.run(Thread.java:745)
```

&nbsp;&nbsp;&nbsp;&nbsp;<font  size =3>剩下的就是去代码中查找具体的问题了。</font>
&nbsp;&nbsp;&nbsp;&nbsp;<font  size =3></font>
&nbsp;&nbsp;&nbsp;&nbsp;<font  size =3></font>


















































































































































































































































































































































































































































































































































































































