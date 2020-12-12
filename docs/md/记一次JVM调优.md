------



# 记一次JVM调优

## 1 为什么需要调优

Java Web项目近一段时间频繁出现卡顿现象，每次卡顿时长也不固定，卡顿时接口无法访问，用户体验较为不好，因此需要查找出现此现象的原因。在查找原因的过程中想到JVM在GC时会出现**STW（Stop The World）**现象，和目前出现的问题较为相符，因此向JVM的GC方向查找。

通过JDK自带工具**jstat**查看GC情况，命令如下：

    jstat -gcutil pid
    
通过查看GC详情发现**Full GC**次数特别频繁，几乎几秒钟一次，我和我的小伙伴都惊呆了:scream:。在之前学习JVM调优时有注意到其他公司的Full GC有几个小时一次到两天一次的，一般一个小时一次都不能接受，我们出现的情况简直不敢相信:joy:。

查看Tomcat中的JVM配置，发现是走的默认配置。**堆**大小默认为**1G**，**新生代**与**老年代**比例为**1：2**，新生代中**S0**、**S1**、**Eden**区比例为**1：1：8**，**堆栈**大小为**1G**。默认垃圾收集器为**Parallel Scavenge（新生代、多线程、复制算法）**和**Parallel Old（老年代、多线程、标记整理）**。

再使用**jmap**查看JVM内存分配情况，命令如下：

    jmap -heap pid
    
查看发现老年代分配了六百多兆内存，Eden区分配两百多兆，S0、S1分别分配五六十兆内存。

我们在一个Tomcat中挂载了十几个项目，同时运行时会频繁创建对象，使得**堆空间**迅速被占满并进行**Minor GC**和**Full GC**，另外运行时常量池也会迅速增长，使得**MetaSpace（元空间）**频繁扩容，每次扩容都会进行**Full GC**，这两种情形都会使Full GC较为频繁。

大概分析了一下后尝试设置JVM参数。

## 2 第一次尝试

询问了一下运维人员，因为Tomcat挂载项目较多，且物理机内存较大，因此当时给Tomcat分配了30G的空间，但是因为JVM走的默认配置，堆、堆栈、元空间可能只用了几G的空间，还有很大的调整余地。

因此给JVM堆空间设置21G内存，其中新生代7G，老年代14G，堆栈1G，进入老年代最大年龄为15次，元空间最大为2G，垃圾收集器分别为Parallel Scavenge和Parallel Old。设置如下：

    -Xms21G -Xmx21G -Xmn7G -Xss1024M -XX:MaxTenuringThreshold=15 -XX:MaxMetaspceSize=2G
    
重新启动Tomcat后使用JDK工具**VisualVM**查看堆和元空间变化情况，堆空间可以看到正常起伏，元空间则持续增长（初始阶段大约为21M）、扩容，每次扩容则带来一次Full GC。然后使用jstat命令查看GC情况，起始阶段，堆变化较为正常，新生代Minor GC接近一秒一次，而Full GC已进行了几次，这是因为元空间扩容，但是后面老年代增长缓慢，所以暂时没发现异常。

持续观察一段时间后，元空间迅速增长到2G，因为**\-XX:MaxMetaspceSize**参数的最大限制为2G，因此程序挂了。

## 3 第二次尝试

因为第一次尝试是因为限制了元空间最大值，导致元空间不够，所以程序挂了。因此决定调整JVM参数，不给元空间设置最大值，而设置**\-XX:MetaspaceSize**（元空间扩容时触发Full GC的初始阈值，元空间初始默认大小仍旧为约21M，且Full GC阈值不会固定为2G，而是会根据GC情况增大或者减小）参数为2G，这样会减小项目启动时元空间扩容导致的Full GC。

另外为了更好的查看GC情形，设置参数**\-XX:+PrintGCTimeStamps**和**\-XX:+PrintGCDetails**打印GC日志，可以看到Minor GC和Full GC时释放空间及原因。

最后，希望查看是否因为项目中存在异常对象导致堆空间迅速被占满而进行Full GC，因此设置参数**\-XX:+HeapDumpBeforeFullGC**（Full GC前Dump）、**\-XX:+HeapDumpAfterFullGC**（Full GC后Dump）、**\-XX:HeapDumpPath**（文件下载路径）。下载Dump文件分析工具**Mat**，对比Full GC前后Dump文件中对象变化情况。

JVM参数修改如下：

    -Xms21G -Xmx21G -Xmn7G -Xss1024M -XX:MaxTenuringThreshold=15 -XX:MetaspaceSize=2G -XX:+PrintGCTimeStamps -XX:+PrintGCDetails -XX:+HeapDumpBeforeFullGC -XX:+HeapDumpAfterFullGC -XX:HeapDumpPath=path
    
启动后使用**VisualVM**持续观察堆和元空间，新生代GC正常，老年代增长缓慢还未出现GC，元空间持续增长还未稳定。持续观察一段时间，元空间稳定在2.5G左右。在第一次Full GC后，观察Log文件中的数据，Minor GC数据正常，一次Full GC持续3-4秒。并且在Full GC前后生成Dump文件，使用**Mat**工具对比Dump文件，观察对象变化后，没有查看到可以对象。

## 4 其他尝试

其实在两次尝试中间还修改过垃圾收集器，将默认垃圾收集器Parallel Scavenge和Parallel Old改为ParNew（复制算法、多线程）和CMS（标记清除、并发），修改后发现Full GC反而更加频繁。查看对比Parallel和CMS垃圾收集器的区别，发现Parallel垃圾收集器更注重CPU吞吐量（用户线程执行时间/（用户线程执行时间+GC时间）），CMS垃圾收集器更注重GC停顿时长。

## 5 后续计划

因为调大了堆和元空间的大小，Full GC时所需要的时间相对更长，则STW时间也更长，用户体验可能不好，如果出现此种情形，可以考虑将垃圾收集器改为G1垃圾收集器。G1是一款面向服务器的垃圾收集器，主要针对配备多颗处理器及大容量内存的机器，以极高概率满足GC停顿时间要求的同时，还具备高吞吐量性能特征。G1相对于CMS的另一个大优势，降低停顿时间是G1和CMS共同的关注点，但G1除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为M毫秒的时间片段内。