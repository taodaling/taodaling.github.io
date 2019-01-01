---
categories: tool
layout: post
---

- Table
{:toc}
# jstack

```sh
jstack [option] pid
```

`jstack`命令行工具，可以连接到特定进程或核心文件，并打印连接到JVM的所有线程的栈信息，包括Java线程和JVM内部线程，以及可选的本地栈进程（本地方法）。该工具还可以用于检测死锁。

调用`jsadebugd`可以获得远程主机上的某个进程的堆栈信息。

`-l`选项，命令`jstack`工具查找堆内被持有的同步对象，并打印`java.util.concurrent.locks`的信息。如果不指定这个选项，则只对象监视器。

线程的dump信息也可以通过编程的方式获得（`Thread.getAllStackTraces`）。

如果进程被挂起，对该进程调用`jstack`命令，将无法获得响应。你可以使用`-F`选项强制进行栈信息dump。

jstack工具也可以打印混合栈信息，即，它除了可以打印java栈外还会打印本地栈帧。所谓的本地栈帧是指与VM代码或JNI/native代码相关的C/C++帧。要打印混合栈，你需要使用`-m`选项。

# jinfo

```sh
jinfo [option] pid
```

jinfo工具用于打印某个jvm进程的java配置信息。配置信息是由Java系统变量、Java虚拟机命令行参数组成的。如果给定的进程是64字节的虚拟机，那么也许你会需要指定`-J-d64`选项，比如：

```sh
jinfo -J-d64 pid
```

如果你想查看一个运行中的JVM的classpath、系统变量、命令行参数等信息，你可以使用jinfo得到。

# jstat

```sh
jstat [generalOption | outputOption vmid [interval[s|ms] [count]]]
```

jstat工具可以展现一个被增强的JVM的性能相关数据。目标JVM利用它的虚拟机的标识符或者vmid指定。

## generalOption

一个通用的命令行选项：

- -help: 展现帮助信息
- -options: 展现统计选项列表。

## outputOptions

一个或多个输出选项，包含一个单独的statOption，加上`-t`,`-h`,`-J`选项中的任意选项。

如果你没有指定通用选项，那么你可以指定输出选项，输出选项决定了jstat的输出的格式和内容。

输出被格式化为表格，每列用空格隔开，首行包含了描述列的标题信息。

命令行选项：

- -h n:每n个样本打印一次列首（标题），n必须为正数，默认为0，表示仅打印一次。
- -t:将时间戳作为第一列进行展示，时间戳为当前时间减去JVM启动时间。
- -J javaOption:向java应用启动程序传递javaOption。

不建议写脚本来解析jstat的输出内容，因为它的格式很有可能会在未来的发行版中被改动。

## statOption

- class:类加载器的性能表现数据

- compiler:HotSpot的JIT编译器的性能表现数据
- gc:被垃圾回收的堆的性能表现数据
- gccapacity:内存分代的容量以及已用空间。
- gccause:GC的数据摘要，以及上一次垃圾回收事件的触发原因
- gcnew:新代的性能表现数据
- gcnewcapacity:新代的容量和已用空间
- gcold:永久代和年老代的性能表现数据
- gcoldcapacity:年老代的大小数据
- gcpermcapacity:永久代的大小数据
- gcutil: gc数据摘要
- printcompilation:HotSpot方法编译数据。

### class

| Column   | Description                                             |
| -------- | ------------------------------------------------------- |
| Loaded   | Number of classes loaded.                               |
| Bytes    | Number of Kbytes loaded.                                |
| Unloaded | Number of classes unloaded.                             |
| Bytes    | Number of Kbytes unloaded.                              |
| Time     | Time spent performing class load and unload operations. |

### compiler

| Column       | Description                                            |
| ------------ | ------------------------------------------------------ |
| Compiled     | Number of compilation tasks performed.                 |
| Failed       | Number of compilation tasks that failed.               |
| Invalid      | Number of compilation tasks that were invalidated.     |
| Time         | Time spent performing compilation tasks.               |
| FailedType   | Compile type of the last failed compilation.           |
| FailedMethod | Class name and method for the last failed compilation. |

### gc

| Column | Description                               |
| ------ | ----------------------------------------- |
| S0C    | Current survivor space 0 capacity (KB).   |
| S1C    | Current survivor space 1 capacity (KB).   |
| S0U    | Survivor space 0 utilization (KB).        |
| S1U    | Survivor space 1 utilization (KB).        |
| EC     | Current eden space capacity (KB).         |
| EU     | Eden space utilization (KB).              |
| OC     | Current old space capacity (KB).          |
| OU     | Old space utilization (KB).               |
| PC     | Current permanent space capacity (KB).    |
| PU     | Permanent space utilization (KB).         |
| YGC    | Number of young generation GC Events.     |
| YGCT   | Young generation garbage collection time. |
| FGC    | Number of full GC events.                 |
| FGCT   | Full garbage collection time.             |
| GCT    | Total garbage collection time.            |

### gccapacity

| Column | Description                                 |
| ------ | ------------------------------------------- |
| NGCMN  | Minimum new generation capacity (KB).       |
| NGCMX  | Maximum new generation capacity (KB).       |
| NGC    | Current new generation capacity (KB).       |
| S0C    | Current survivor space 0 capacity (KB).     |
| S1C    | Current survivor space 1 capacity (KB).     |
| EC     | Current eden space capacity (KB).           |
| OGCMN  | Minimum old generation capacity (KB).       |
| OGCMX  | Maximum old generation capacity (KB).       |
| OGC    | Current old generation capacity (KB).       |
| OC     | Current old space capacity (KB).            |
| PGCMN  | Minimum permanent generation capacity (KB). |
| PGCMX  | Maximum Permanent generation capacity (KB). |
| PGC    | Current Permanent generation capacity (KB). |
| PC     | Current Permanent space capacity (KB).      |
| YGC    | Number of Young generation GC Events.       |
| FGC    | Number of Full GC Events.                   |

### gccause

| Column | Description                          |
| ------ | ------------------------------------ |
| LGCC   | Cause of last Garbage Collection.    |
| GCC    | Cause of current Garbage Collection. |

### gcnew

| Column | Description                               |
| ------ | ----------------------------------------- |
| S0C    | Current survivor space 0 capacity (KB).   |
| S1C    | Current survivor space 1 capacity (KB).   |
| S0U    | Survivor space 0 utilization (KB).        |
| S1U    | Survivor space 1 utilization (KB).        |
| TT     | Tenuring threshold.                       |
| MTT    | Maximum tenuring threshold.               |
| DSS    | Desired survivor size (KB).               |
| EC     | Current eden space capacity (KB).         |
| EU     | Eden space utilization (KB).              |
| YGC    | Number of young generation GC events.     |
| YGCT   | Young generation garbage collection time. |

### gcnewcapacity

| Column | Description                             |
| ------ | --------------------------------------- |
| NGCMN  | Minimum new generation capacity (KB).   |
| NGCMX  | Maximum new generation capacity (KB).   |
| NGC    | Current new generation capacity (KB).   |
| S0CMX  | Maximum survivor space 0 capacity (KB). |
| S0C    | Current survivor space 0 capacity (KB). |
| S1CMX  | Maximum survivor space 1 capacity (KB). |
| S1C    | Current survivor space 1 capacity (KB). |
| ECMX   | Maximum eden space capacity (KB).       |
| EC     | Current eden space capacity (KB).       |
| YGC    | Number of young generation GC events.   |
| FGC    | Number of Full GC Events.               |

### gcold

| Column | Description                            |
| ------ | -------------------------------------- |
| PC     | Current permanent space capacity (KB). |
| PU     | Permanent space utilization (KB).      |
| OC     | Current old space capacity (KB).       |
| OU     | old space utilization (KB).            |
| YGC    | Number of young generation GC events.  |
| FGC    | Number of full GC events.              |
| FGCT   | Full garbage collection time.          |
| GCT    | Total garbage collection time.         |

### gcoldcapacity

| Column | Description                           |
| ------ | ------------------------------------- |
| OGCMN  | Minimum old generation capacity (KB). |
| OGCMX  | Maximum old generation capacity (KB). |
| OGC    | Current old generation capacity (KB). |
| OC     | Current old space capacity (KB).      |
| YGC    | Number of young generation GC events. |
| FGC    | Number of full GC events.             |
| FGCT   | Full garbage collection time.         |
| GCT    | Total garbage collection time.        |

### gcpermcapacity

| Column | Description                                 |
| ------ | ------------------------------------------- |
| PGCMN  | Minimum permanent generation capacity (KB). |
| PGCMX  | Maximum permanent generation capacity (KB). |
| PGC    | Current permanent generation capacity (KB). |
| PC     | Current permanent space capacity (KB).      |
| YGC    | Number of young generation GC events.       |
| FGC    | Number of full GC events.                   |
| FGCT   | Full garbage collection time.               |
| GCT    | Total garbage collection time.              |

### gcutil

| Column | Description                                                  |
| ------ | ------------------------------------------------------------ |
| S0     | Survivor space 0 utilization as a percentage of the space's current capacity. |
| S1     | Survivor space 1 utilization as a percentage of the space's current capacity. |
| E      | Eden space utilization as a percentage of the space's current capacity. |
| O      | Old space utilization as a percentage of the space's current capacity. |
| P      | Permanent space utilization as a percentage of the space's current capacity. |
| YGC    | Number of young generation GC events.                        |
| YGCT   | Young generation garbage collection time.                    |
| FGC    | Number of full GC events.                                    |
| FGCT   | Full garbage collection time.                                |
| GCT    | Total garbage collection time.                               |

### printcompilation

| Column   | Description                                                  |
| -------- | ------------------------------------------------------------ |
| Compiled | Number of compilation tasks performed by the most recently compiled method. |
| Size     | Number of bytes of bytecode of the most recently compiled method. |
| Type     | Compilation type of the most recently compiled method.       |
| Method   | Class name and method name identifying the most recently compiled method. Class name uses "/" instead of "." as namespace separator. Method name is the method within the given class. The format for these two fields is consistent with the HotSpot - **XX:+PrintComplation** option. |

# jmap

```sh
jmap [option] pid
```

`jmap`工具用于显示一个给定进程的共享对象内存映射后堆内存细节。如果给定进程为64字节虚拟机，你可能需要指定`-J-d64`选项。

选项：

- <无选项>:jmap仅显示共享对象映射。对于虚拟机加载的所有共享对象，开始地址、映射大小、共享对象的全文件路径都会被显示。
- -dump:[live,]format=b,file=\<filename\>: 按照hprof的二进制格式将java堆dump到指定文件中。如果指定了live，将仅dump存活对象。之后你可以通过jhat（Java Heap Analysis Tool）来阅读生成文件。
- -finalizerinfo: 显示等待析构的对象的信息。
- -heap: 显示堆摘要，使用的GC算法，堆配置以及分代信息。
- -histo[:live]: 显示描述堆的直方图。对于每个类，实例的数目，占用的内存的字节数，以及全限定类名都会被显示。虚拟机内部类名带有'*'前缀。如果指定了live，那么仅存活对象被计数。
- -permstat:显示永久代中类加载器的数据。对于每个类加载器，它的名字，存活状态，地址，父加载器，以及它所加载过的类的数目和大小都会被显示。此外，常量池中的字符串的数目和大小也会被显示。
- -F: 强制，如果pid进程没有响应，需要带上这个参数（使用这个参数就不能带上live参数）。
- -h: 显示帮助信息。
- -help: 显示帮助信息
- -J\<flag\>: 传递\<flag\>到JVM。