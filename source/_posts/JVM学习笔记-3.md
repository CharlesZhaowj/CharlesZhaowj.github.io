---
title: JVM学习笔记_3
date: 2019-07-03 22:07:50
tags:
    - Java
    - JVM
    - 技术
---

# JVM学习笔记3_虚拟机性能监控与故障处理工具

## 参考资料
周志明《深入理解Java虚拟机》
<!--more-->

## 系统问题的定位
1. 知识、经验是关键基础
2. 数据（运行日志、异常堆栈、GC日志、线程快照/threaddump/javacore文件、堆转储快照/heapdump/hprof文件、etc）
3. 工具不是万能的，还是要看自己怎么用

## JDK命令行工具
1. Sun送给Java开发的
2. 是`jdk/lib/tool.jar`类库的一层包装
3. 不同的JDK和OS支持的功能可能差异较大

### Sun JDK 监控和故障处理工具一览
名称|全名|主要作用
--|--|--
jps|JVM Process Status Tool|显示指定系统内所有的HotSpot虚拟机进程
jstat|JVM statistics Monitoring Tool|收集HotSpot虚拟机各方面的运行数据
jinfo|Configuration Info for Java|显示虚拟机配置信息
jmap|Memory Map for Java|生成虚拟机的内存转储快照（heapdump文件）
jhat|JVM heap dump browser|分析heapdump文件，建立一个http/html服务器，用户可以在浏览器上查看分析结果
jstack|stack trace for java，显示虚拟机的线程快照

### jps_虚拟机进程状况工具
很多JDK小工具都参考了UNIX命令的命名方式

1. jps能查到虚拟机进程、显示虚拟机主类名称
2. LVMID(local virtual machine Identifier)对应本地操作系统的PID
3. 格式`jps [options] [hostid]`，其中`hostid`是RMI注册表种注册的主机名（用RMI协议查询开启了RMI服务的远程虚拟机进程状态）

#### jps工具主要选项
选项|作用
--|--
-q|只输出LVMID，省略主类名称
-m|输出虚拟机启动时传给主类的参数
-l|输出主类全名，如果进程执行的是Jar包，输出Jar路径
-v|输出虚拟机进程启动时的JVM参数

### jstat_虚拟机统计信息监视工具
1. 显示本地或远程（还是RMI）虚拟机进程中的类装载、内存、辣鸡收集、JIT编译等运行数据
2. 不通过GUI图形界面
3. 格式`jstat [option vmid [interval[s|ms] [count]]]`(PS: 远程的虚拟机进程VMID格式应为：`[protocol:][//]lvmid[@hostname[:port]/servername`)
4. 参数`interval`以及`count`代表查询间隔和次数，如果省略就说明只查一次，举例：`jstat -gc 2764 250 20`表示250毫秒一次查询进程2764的gc状况，共250次

#### jstat工具主要选项
选项|作用
--|--
-class|类装载、卸载数量、总空间以及类装载耗时
-gc|监视Java堆状况、包括Eden区，两个survivor区、老年代、永久代等的容量、已用空间、GC耗时等信息合计
-gccapacity|基本与-gc相同，主要关注堆中各个区域使用到的最大最小空间
-gcutil|监视内容与-gc基本相同，主要关注已使用空间占总空间的百分比
-gccause|与-gcutil功能一样，会额外输出导致上次gc的原因
-gcnew|监视新生代GC状况
-gcnewcapacity|gcnew和gccapacity的结合
-gcold|监视老年代GC状况
-gcoldcapacity|gcold+gccapacity
-gcpermcapacity|输出永久代使用到的最大、最小空间
-compiler|输出JIT编译器编译过的方法、耗时等信息
-printcompilation|输出已经被JIT编译的方法

### jinfo: Java信息配置工具
1. 实时查看和调整虚拟机各项参数
2. 格式`jinfo [option] pid`
3. 比如查询`CMSInitiatingOccupancyFraction`参数值:`jinfo -flag CMSInitiatingOccupancyFraction 1233`

### jmap: Java内存映像工具
1. 生成堆转储快照（称作 headdump 或者 dump）
2. 除了获取dump文件，还可以查询finalize执行队列、Java堆和永久代的详细信息（例如空间使用率、选择的GC方案等）
3. 命令格式： `jmap [ option ] pid`

#### jmap工具主要选项
选项|作用
--|--
-dump|生成Java堆转储快照，格式为`-dump:[live, ]format=b, file=<filename>`,其中live子参数说明是否只dump出存活的对象
-finalizerinfo|显示在F-Queue中等待Finalizer线程执行finalize方法的对象，只在Linux/Solaris中生效
-heap|显示Java堆详细信息，如GC策略、参数配置、分代状况等，只在Linux/Solaris中生效
-histo|显示堆中对象统计信息，包括类、实例数量、合计容量等
-permstat|以ClassLoader为统计口径显示永久代内存状态。只在Linux/Solaris平台下有效
-F|当虚拟机进程对-dump选项没有响应时，使用该选项可以强制生成快照

### jhat: 虚拟机堆转储快照分析工具
1. 内置了微型HTTP/HTML服务器
2. 平时一般不怎么会用的，因为分析功能比较简陋
3. 所以一般会用`Visual VM`、`IBM HeapAnalyzer`、`Eclipse Memoray Analyzer`等工具
4. 所以就不再详述le

### jstack: Java堆栈分析工具
1. 生成虚拟机当前时刻的线程快照（一般称为 threaddump 或者 javacore 文件）
2. 线程快照--当前虚拟机内每一条线程正在执行的方法堆栈的集合。生成线程快照的作用是定位线程出现长时间停顿的原因
3. 命令格式`jstack [option] vmid`

#### jstack工具主要选项
选项|作用
--|--
-F|当正常输出的请求不被响应时，强制输出线程堆栈
-l|除堆栈外、显示关于锁的附加信息
-m|如果调用到本地方法的话，可以显示C/C++