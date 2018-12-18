---
title: 记一次从hs_err_log中的codecache学习
---
因为学生时代的我，只能用阿里云或者腾讯云的学生服务器，服务器内存只有1G，所以经常就是部署了一两个项目之后，服务器内存达到70%,然后有时候项目访问量大点，就会导致内存溢出，服务器死掉。

### 分析
每次服务器死掉之后，都会生成hs_err_pidxxx文件,默认情况下，这个文件会产生在工作目录下。但是可以在Java启动参数通过下面的设置来改变这个文件的位置和命名规则。

```
java -XX:ErrorFile=/var/log/java/java_error_%p.log
```

将这个错误文件放在/var/log/java下，并且以java_error_pid.log的形式出现。简单分析后，发现里面有个codecache的概念。

```
#
# There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (mmap) failed to map 65536 bytes for committing reserved memory.
# Possible reasons:
#   The system is out of physical RAM or swap space
#   In 32 bit mode, the process size limit was hit
# Possible solutions:
#   Reduce memory load on the system
#   Increase physical memory or swap space
#   Check if swap backing store is full
#   Use 64 bit Java on a 64 bit OS
#   Decrease Java heap size (-Xmx/-Xms)
#   Decrease number of Java threads
#   Decrease Java thread stack sizes (-Xss)
#   Set larger code cache with -XX:ReservedCodeCacheSize=
# This output file may be truncated or incomplete.
#
#  Out of Memory Error (os_linux.cpp:2627), pid=27760, tid=139853318088448
#
# JRE version: Java(TM) SE Runtime Environment (8.0_91-b14) (build 1.8.0_91-b14)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (25.91-b14 mixed mode linux-amd64 compressed oops)
# Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
#
```
上面的简单翻译是：


- 没有足够的内存使Java运行环境继续运行。
- 本机内存分配(mmap)未能提交保留内存映射65536字节
- 可能原因：
- 系统没有足够的物理内存或者交换空间。
- 在32为模式下，进程大小达到最大限制(https://segmentfault.com/q/1010000000777695/a-1020000000778158)

解决方案：

- 减少系统加载进内存的数据
- 增加物理内存或者交换空间(http://www.cnblogs.com/kerrycode/p/5246383.html)
- ps:这里的交换空间是类似window中的虚拟内存的概念。
- 检查交换支持是否已满
- 在64为操作系统上使用64位的java
- 减少java 堆栈大小(-Xmx/-Xms),指定最大堆栈，最小堆栈
- 减少java线程数量
- 减少java线程栈大小(-Xss)
- 设置最大代码缓存大小 (-XX:ReservedCodeCacheSize=)

其中最后一点指出，可以通过设置CodeCache 来解决内存不足问题。但是这里的CodeCache是指什么呢？之前的理解是，java是一门编译行语言，就是说所有代码是编译完之后直接执行的，但是后来才知道，就算代码直接编译成class,也是可以被解析执行的。

codecache与Java中的jit(just in time compilation)有关
(https://www.zhihu.com/question/26913901/answer/35303563)

Java执行过程中，一般是有这么几个步骤

1.将class读入内存  
2.解析class文件，进行解析执行  
3.JIT环境变量收集基本单元执行信息，比如说执行次说之类的，这里的基本单元一般精确到方法级别。  
4.判断方法是否是热点方法，是的话，通过jit编译器，将热点方法进行jit编译(重型编译)这时候热点方法会被侧重优化，所有下次再执行热点方法时候，会直接执行经过优化后的代码，这时候就能提高系统执行效率。  
5.不是热点方法的话，就继续解析执行，继续收集运行单元信息。
### Code Cache 优化
(http://blog.csdn.net/qq_28674045/article/details/51896129)
JIT在编译代码的时候，会在Code Cache中保存一些汇编指令。由于Code Cache的大小是固定的，一旦它被填满了，JVM就无法编译其他热点代码。如果Code Cache很小，就会导致部分热点代码没有被编译，所以部分本该识别为热点方法的代码，只能够被解释执行。  

各个版本的JVM对应的Code Cache 的默认大小，如下图所示：


其中，Code Cache 的最大大小，可以通过 -XX:ReservedCodeCacheSize=N来指定，初始大小使用-XX:InitialCodeCacheSize=N来指定。由于初始大小是会自动增加的，所以一般不需要设置-XX:InitialCodeCacheSize参数。
Code Cache指定多大值比较合适呢？
这个主要看系统资源是否足够。对于32bit的JVM，由于虚拟内存总大小为4GB,所以这个值不能设置的太大;对于64bit的JVM,这个值的大小设置一般不受限制的。

### 编译阀值

上面提到，编译器会自动识别到热点方法。默认配置下，一开始所有Java方法都由解释器执行。解释器记录着每个方法的调用次数和循环次数，并以这两个数值为指标去判断一个方法的"热度"。  

可以通过配置-XX:CompileThreshold=N来确定计数器的阀值，从而控制编译条件。但是如果降低了这个值，会导致JVM没有收集到足够证据，就对方法进行编译，导致编译的代码优化不够。  
如果-XX:CompileThreshold配置的两个值(一个大，一个小)在性能测试方面表现差不多，那么会推荐使用小一点的配置，主要有两个原因：  
- 可以减少应用的预热阶段warm-up period。  
- 可以防止某个方法永远得不到编译。这是应为JVM会周期的对计数进行递减。如果阀值比较大，并且方法周期性调用的时间较长，导致计数永远达不到这个阀值，从而不会进行编译。

### 探寻编译过程
通过参数 -XX:+PrintCompilation可以打开编译日志，当JIT对方法进行编译的时候，都会打印一行信息，什么方法被编译了。具体格式如下:
```
timestamp compilation_idattributes (tiered_level) method_name size deopt
```
> timestamp 是编译时相对于JVM启动的时间戳
compilation_id是内部编译任务的ID,一般都是递增的
attributes有5个字母组成，用来显示代码被编译的状态：
    %:编译时OSR
    s:方法是synchronized的
    !:方法有异常处理
    n:表示JVM产生了一些辅助代码，以便调用native方法  
    tiered_level：采用tiered compilation编译器时才会打印。
    method_name:被编译的方法名称,格式是:classname::method
    size:被编译的代码大小(单位:字节),这里的代码是Java字节码。
    deopt:如果发生了去优化，这里说明去优化的类型。
### Tiered Complication 级别
> 如果JVM采用了Tiered Compilation编译器，那么在编译日志汇总会打印方法编译的层级。总共有五个层级，分别如下:
    0:解释性代码(Interpreted Code)
    1:简单的C1编译代码(Simple C1 Compiled Code)
    2:受限的C1编译代码(Limited C1 Compiled Code)
    3:完整的C1编译代码(Full C1 Compiled Code)
    4:C2编译代码(C2 Compiled Code)  
    在一般情况下，大部分方法首先在层级3进行编译，随着JVM收集的信息不断增多， 最终代码会在层级4进行编译。原来层级3编译的代码会进入到not entrant 状态。  
    如果server队列满了，方法从server队列取出后，使用层级2进行编译(使用C1编译器,但是不会手机更多的信息以便做进一步的优化)，使得编译更快，减轻server队列的压力。随着时间的推移，JVM收集到了更多的信息，代码最终会在层级3和层级4进行编译。

    如果client队列满了，一些本来在等待使用层级3进行编译的方法，也许可以使用层级4进行编译(加快速度)，随后，直接使用层级4进行编译(不经过层级3)。  
对于非常小的方法，可能先采用层级2或者3进行编译，最终会采用层级1进行编译。  
如果由于某种原因，server编译器无法对代码进行编译，它会采用层级1进行编译。  
在反优化的过程中，代码会进入到层级0(解释执行)。  
如果编译日志中，经常采用层级2进行方法编译，此时需要考虑增加编译线程的数目。


下面附上参考的文章
JIT与JVM的三种执行模式：解释模式、编译模式、混合模式(http://www.cnblogs.com/lyhero11/p/5080306.html)

为什么 JVM 不用 JIT 全程编译？(https://www.zhihu.com/question/37389356)

HotSpot是较新的Java虚拟机技术，用来代替JIT技术,那么HotSpot和JIT是共存的吗？(https://www.zhihu.com/question/26913901/answer/35303563)

Java性能优化指南系列(三）：理解JIT编译器(http://blog.csdn.net/qq_28674045/article/details/51896129)
