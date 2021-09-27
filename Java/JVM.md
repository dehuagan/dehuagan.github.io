# 双亲委派机制
## 为什么需要双亲委派机制
双亲委派保证类加载器，自下而上的委派，又自上而下的加载，<font size=4 color="green">保证每一个类在各个类加载器中都是同一个类</font>。

一个非常明显的目的就是保证java官方的类库<JAVA_HOME>\lib和扩展类库<JAVA_HOME>\lib\ext的加载安全性，不会被开发者覆盖。

例如类java.lang.Object，它存放在rt.jar之中，无论哪个类加载器要加载这个类，最终都是委派给启动类加载器加载，因此Object类在程序的各种类加载器环境中都是同一个类。

如果开发者自己开发开源框架，也可以自定义类加载器，利用双亲委派模型，保护自己框架需要加载的类不被应用程序覆盖。

## loadClass过程
1、先检查类是否已经被加载过

2、若没有加载则调用父加载器的loadClass()方法进行加载

3、若父加载器为空则默认使用启动类加载器作为父加载器。

4、如果父类加载失败，抛出ClassNotFoundException异常后，再调用自己的findClass()方法进行加载。

## 破坏双亲委派
### why
因为在某些情况下父类加载器需要委托子类加载器去加载class文件。受到加载范围的限制，父类加载器无法加载到需要的文件，以Driver接口为例，由于Driver接口定义在jdk当中的，而其实现由各个数据库的服务商来提供，比如mysql的就写了MySQL Connector，那么问题就来了，DriverManager（也由jdk提供）要加载各个实现了Driver接口的实现类，然后进行管理，但是DriverManager由启动类加载器加载，只能记载JAVA_HOME的lib下文件，而其实现是由服务商提供的，由系统类加载器加载，这个时候就需要启动类加载器来委托子类来加载Driver实现，从而破坏了双亲委派，这里仅仅是举了破坏双亲委派的其中一个情况。

### 破坏双亲委派的实现
我们结合Driver来看一下在spi（Service Provider Inteface）中如何实现破坏双亲委派。

先从DriverManager开始看，平时我们通过DriverManager来获取数据库的Connection：

```js
String url = "jdbc:mysql://localhost:3306/xxx";
Connection conn = java.sql.DrivaerManager.getConnection(url,"root","root");
```

在调用DriverManager的时候，会先初始化类，调用其中的静态块：
```js
static {
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}

private static void loadInitialDrivers () {
    ...
    // 加载Driver的实现类
    AccessController.doPrivileged(new PrivilegedAction<void>() {
        public void run() {
            ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
            Iterator<Driver> driversIterator = loadedDrivers.iterator();
            try{
                while(driversIterator.hasNext()){
                    driversIterator.next();
                }
            } catch(Throwable t){

            }
            return null;
        }
    });
    ...
}
```

重点来看一下ServiceLoader.load(Driver.class)：

```js
public static <S> ServiceLoader<S> load(Class<S> service) {
    //获取当前线程中的上下文类加载器
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
```

可以看到，load方法调用获取了当前线程中的上下文类加载器，那么上下文类加载器放的是什么加载器呢？

在sun.misc.Launcher中，我们找到了答案，在Launcher初始化的时候，会获取AppClassLoader，然后将其设置为上下文类加载器，而这个AppClassLoader，就是之前上文提到的系统类加载器Application ClassLoader，所以上下文类加载器默认情况下就是系统加载器。

![img](../img/loadclass.png)

# GC
[详细](https://mp.weixin.qq.com/s/_AKQs-xXDHlk84HbwKUzOw)


# jvm调优命令

[详细](https://www.cnblogs.com/ityouknow/p/5714703.html)

## jps
JVM Process Status Tool,显示指定系统内所有的HotSpot虚拟机进程。

### 命令格式
jps [options] [hostid]

### option参数
- l : 输出主类全名或jar路径
- q : 只输出LVMID
- m : 输出JVM启动时传递给main()的参数
- v : 输出JVM启动时显示指定的JVM参数

其中[option]、[hostid]参数也可以不写。

### 示例
```js
$ jps -l -m
  28920 org.apache.catalina.startup.Bootstrap start
  11589 org.apache.catalina.startup.Bootstrap start
  25816 sun.tools.jps.Jps -l -m
```

## jstat
jstat(JVM statistics Monitoring)是用于监视虚拟机运行时状态信息的命令，它可以显示出虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。

### 命令格式
jstat [option] LVMID [interval] [count]

### 参数
- [option] : 操作参数
- LVMID : 本地虚拟机进程ID
- [interval] : 连续输出的时间间隔
- [count] : 连续输出的次数


### option 参数总览
| Option |	Displays… |
| ---- | ---- |
|class	|class loader的行为统计。Statistics on the behavior of the class loader.|
|compiler	| HotSpt JIT编译器行为统计。Statistics of the behavior of the HotSpot Just-in-Time compiler.|
|gc	|垃圾回收堆的行为统计。Statistics of the behavior of the garbage collected heap.|
|gccapacity	|各个垃圾回收代容量(young,old,perm)和他们相应的空间统计。Statistics of the capacities of the generations and their corresponding spaces.|
|gcutil	|垃圾回收统计概述。Summary of garbage collection statistics.|
|gccause|	垃圾收集统计概述（同-gcutil），附加最近两次垃圾回收事件的原因。Summary of garbage collection statistics (same as -gcutil), with the cause of the last and|
|gcnew|	新生代行为统计。Statistics of the behavior of the new generation.|
|gcnewcapacity|	新生代与其相应的内存空间的统计。Statistics of the sizes of the new generations and its corresponding spaces.|
|gcold	|年老代和永生代行为统计。Statistics of the behavior of the old and permanent generations.|
|gcoldcapacity	|年老代行为统计。Statistics of the sizes of the old generation.|
|gcpermcapacity	|永生代行为统计。Statistics of the sizes of the permanent generation.|
|printcompilation	|HotSpot编译方法统计。HotSpot compilation method statistics.|

### option 参数详解
-class

监视类装载、卸载数量、总空间以及耗费的时间
```js
$ jstat -class 11589
 Loaded  Bytes  Unloaded  Bytes     Time   
  7035  14506.3     0     0.0       3.67
```
- Loaded : 加载class的数量
- Bytes : class字节大小
- Unloaded : 未加载class的数量
- Bytes : 未加载class的字节大小
- Time : 加载时间


-gc

垃圾回收堆的行为统计，常用命令
```js
$ jstat -gc 1262
 S0C    S1C     S0U     S1U   EC       EU        OC         OU        PC       PU         YGC    YGCT    FGC    FGCT     GCT   
26112.0 24064.0 6562.5  0.0   564224.0 76274.5   434176.0   388518.3  524288.0 42724.7    320    6.417   1      0.398    6.815
```
C即Capacity 总容量，U即Used 已使用的容量

- S0C : survivor0区的总容量
- S1C : survivor1区的总容量
- S0U : survivor0区已使用的容量
- S1C : survivor1区已使用的容量
- EC : Eden区的总容量
- EU : Eden区已使用的容量
- OC : Old区的总容量
- OU : Old区已使用的容量
- PC 当前perm的容量 (KB)
- PU perm的使用 (KB)
- YGC : 新生代垃圾回收次数
- YGCT : 新生代垃圾回收时间
- FGC : 老年代垃圾回收次数
- FGCT : 老年代垃圾回收时间
- GCT : 垃圾回收总消耗时间
  
```js
$ jstat -gc 1262 2000 20
```
这个命令意思就是每隔2000ms输出1262的gc情况，一共输出20次


## jmap
jmap(JVM Memory Map)命令用于生成heap dump文件<font size=4 color = "green">（在故障定位(尤其是OOM)和性能分析的时候，经常会用到一些文件辅助我们排除代码问题。 这些文件记录了JVM运行期间的内存占用、线程执行等情况，这就是我们常说的dump文件）</font>，如果不使用这个命令，还阔以使用-XX:+HeapDumpOnOutOfMemoryError参数来让虚拟机出现OOM的时候·自动生成dump文件。
jmap不仅能生成dump文件，还阔以查询finalize执行队列、Java堆和永久代的详细信息，如当前使用率、当前使用的是哪种收集器等。

### 命令格式
jmap [option] LVMID

### option参数

- dump : 生成堆转储快照
- finalizerinfo : 显示在F-Queue队列等待Finalizer线程执行finalizer方法的对象
- heap : 显示Java堆详细信息
- histo : 显示堆中对象的统计信息
- permstat : to print permanent generation statistics
- F : 当-dump没有响应时，强制生成dump快照

### 示例
-dump

常用格式
```
-dump::live,format=b,file=<filename> pid 
```

dump堆到文件,format指定输出格式，live指明是活着的对象,file指定文件名

```
$ jmap -dump:live,format=b,file=dump.hprof 28920
  Dumping heap to /home/xxx/dump.hprof ...
  Heap dump file created
```
dump.hprof这个后缀是为了后续可以直接用MAT(Memory Anlysis Tool)打开。


-heap

打印heap的概要信息，GC使用的算法，heap的配置及wise heap的使用情况,可以用此来判断内存目前的使用情况以及垃圾回收情况

```js
$ jmap -heap 28920
  Attaching to process ID 28920, please wait...
  Debugger attached successfully.
  Server compiler detected.
  JVM version is 24.71-b01  

  using thread-local object allocation.
  Parallel GC with 4 thread(s)//GC 方式  

  Heap Configuration: //堆内存初始化配置
     MinHeapFreeRatio = 0 //对应jvm启动参数-XX:MinHeapFreeRatio设置JVM堆最小空闲比率(default 40)
     MaxHeapFreeRatio = 100 //对应jvm启动参数 -XX:MaxHeapFreeRatio设置JVM堆最大空闲比率(default 70)
     MaxHeapSize      = 2082471936 (1986.0MB) //对应jvm启动参数-XX:MaxHeapSize=设置JVM堆的最大大小
     NewSize          = 1310720 (1.25MB)//对应jvm启动参数-XX:NewSize=设置JVM堆的‘新生代’的默认大小
     MaxNewSize       = 17592186044415 MB//对应jvm启动参数-XX:MaxNewSize=设置JVM堆的‘新生代’的最大大小
     OldSize          = 5439488 (5.1875MB)//对应jvm启动参数-XX:OldSize=<value>:设置JVM堆的‘老生代’的大小
     NewRatio         = 2 //对应jvm启动参数-XX:NewRatio=:‘新生代’和‘老生代’的大小比率
     SurvivorRatio    = 8 //对应jvm启动参数-XX:SurvivorRatio=设置年轻代中Eden区与Survivor区的大小比值 
     PermSize         = 21757952 (20.75MB)  //对应jvm启动参数-XX:PermSize=<value>:设置JVM堆的‘永生代’的初始大小
     MaxPermSize      = 85983232 (82.0MB)//对应jvm启动参数-XX:MaxPermSize=<value>:设置JVM堆的‘永生代’的最大大小
     G1HeapRegionSize = 0 (0.0MB)  

  Heap Usage://堆内存使用情况
  PS Young Generation
  Eden Space://Eden区内存分布
     capacity = 33030144 (31.5MB)//Eden区总容量
     used     = 1524040 (1.4534378051757812MB)  //Eden区已使用
     free     = 31506104 (30.04656219482422MB)  //Eden区剩余容量
     4.614088270399305% used //Eden区使用比率
  From Space:  //其中一个Survivor区的内存分布
     capacity = 5242880 (5.0MB)
     used     = 0 (0.0MB)
     free     = 5242880 (5.0MB)
     0.0% used
  To Space:  //另一个Survivor区的内存分布
     capacity = 5242880 (5.0MB)
     used     = 0 (0.0MB)
     free     = 5242880 (5.0MB)
     0.0% used
  PS Old Generation //当前的Old区内存分布
     capacity = 86507520 (82.5MB)
     used     = 0 (0.0MB)
     free     = 86507520 (82.5MB)
     0.0% used
  PS Perm Generation//当前的 “永生代” 内存分布
     capacity = 22020096 (21.0MB)
     used     = 2496528 (2.3808746337890625MB)
     free     = 19523568 (18.619125366210938MB)
     11.337498256138392% used  

  670 interned Strings occupying 43720 bytes.
```



## jstack
jstack用于生成java虚拟机当前时刻的线程快照。线程快照是当前java虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等。 线程出现停顿的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做什么事情，或者等待什么资源。 如果java程序崩溃生成core文件，jstack工具可以用来获得core文件的java stack和native stack的信息，从而可以轻松地知道java程序是如何崩溃和在程序何处发生问题。另外，jstack工具还可以附属到正在运行的java程序中，看到当时运行的java程序的java stack和native stack的信息, 如果现在运行的java程序呈现hung的状态，jstack是非常有用的。

### 命令格式
jstack [option] LVMID

### option参数

- F : 当正常输出请求不被响应时，强制输出线程堆栈
- l : 除堆栈外，显示关于锁的附加信息
- m : 如果调用到本地方法的话，可以显示C/C++的堆栈

### 示例

```js
$ jstack -l 11494|more
2016-07-28 13:40:04
Full thread dump Java HotSpot(TM) 64-Bit Server VM (24.71-b01 mixed mode):

"Attach Listener" daemon prio=10 tid=0x00007febb0002000 nid=0x6b6f waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
        - None

"http-bio-8005-exec-2" daemon prio=10 tid=0x00007feb94028000 nid=0x7b8c waiting on condition [0x00007fea8f56e000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000cae09b80> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2043)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:104)
        at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:32)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
        at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
        - None
      .....
```

# Java内存区域
- 方法区（方法区是一个 JVM 规范，永久代与元空间都是其一种实现方式。在 JDK 1.8 之后，原来永久代的数据被分到了堆和元空间中。元空间存储类的元信息，静态变量和常量池等放入堆中。）
- 堆
- 虚拟机栈
- 本地方法栈
- 程序计数器

程序计数器、虚拟机栈、本地方法栈这3个区域是线程私有的，会随线程消亡而自动回收，所以不需要管理。

因此<font size=4 color="green">垃圾收集只需要关注堆和方法区</font>。

## 常量池，字符串常量池，运行时常量池区别

### 字符串常量池
全局字符串池里的内容是在类加载完成，经过验证，准备阶段之后在堆中生成字符串对象实例，然后将该字符串对象实例的引用值存到string pool中（记住：string pool中存的是引用值而不是具体的实例对象，具体的实例对象是在堆中开辟的一块空间存放的。）

### 常量池

class文件中除了包含类的版本、字段、方法、接口等描述信息外，还有一项信息就是常量池(constant pool table)，用于存放编译器生成的各种字面量(Literal)和符号引用(Symbolic References)。 字面量就是我们所说的常量概念，如文本字符串、被声明为final的常量值等。 符号引用是一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可（它与直接引用区分一下，直接引用一般是指向方法区的本地指针，相对偏移量或是一个能间接定位到目标的句柄）。


### 运行时常量池
jvm在执行某个类的时候，必须经过加载、连接、初始化，而连接又包括验证、准备、解析三个阶段。而当类加载到内存中后，jvm就会将class常量池中的内容存放到运行时常量池中，由此可知，运行时常量池也是每个类都有一个。类在解析之后，将符号引用替换成直接引用，与全局常量池中的引用值保持一致。

### 变化
![img](../img/stringpool1.png)

![img](../img/stringpool2.png)


# GC触发条件

## YGC
大多数情况下，对象直接在年轻代中的Eden区进行分配，如果Eden区域没有足够的空间，那么就会触发YGC（Minor GC），YGC处理的区域只有新生代。因为大部分对象在短时间内都是可收回掉的，因此YGC后只有极少数的对象能存活下来，而被移动到S0区（采用的是复制算法）。

当触发下一次YGC时，会将Eden区和S0区的存活对象移动到S1区，同时清空Eden区和S0区。当再次触发YGC时，这时候处理的区域就变成了Eden区和S1区（即S0和S1进行角色交换）。每经过一次YGC，存活对象的年龄就会加1。


## FGC
下面4种情况，对象会进入到老年代中：

- YGC时，To Survivor区不足以存放存活的对象，对象会直接进入到老年代。
- 经过多次YGC后，如果存活对象的年龄达到了设定阈值，则会晋升到老年代中。
- 动态年龄判定规则，To Survivor区中相同年龄的对象，如果其大小之和占到了 To Survivor区一半以上的空间，那么大于此年龄的对象会直接进入老年代，而不需要达到默认的分代年龄。
- 大对象：由-XX:PretenureSizeThreshold启动参数控制，若对象大小大于此值，就会绕过新生代, 直接在老年代中分配。

当晋升到老年代的对象大于了老年代的剩余空间时，就会触发FGC（Major GC），FGC处理的区域同时包括新生代和老年代。除此之外，还有以下4种情况也会触发FGC：
- 老年代的内存使用率达到了一定阈值（可通过参数调整），直接触发FGC。
- 空间分配担保：在YGC之前，会先检查老年代最大可用的连续空间是否大于新生代所有对象的总空间。如果小于，说明YGC是不安全的，则会查看参数 HandlePromotionFailure 是否被设置成了允许担保失败，如果不允许则直接触发Full GC；如果允许，那么会进一步检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果小于也会触发 Full GC。
- Metaspace（元空间）在空间不足时会进行扩容，当扩容到了-XX:MetaspaceSize 参数的指定值时，也会触发FGC。
- System.gc() 或者Runtime.gc() 被显式调用时，触发FGC。



# 如何判断对象已成垃圾
## 引用计数
引用计数其实就是为每一个内存单元设置一个计数器，当被引用的时候计数器加一，当计数器减少为 0 的时候就意味着这个单元再也无法被引用了，所以可以立即释放内存。

缺陷：循环引用

## 可达性分析
可达性分析其实就是利用标记-清除(mark-sweep)，就是标记可达对象，清除不可达对象。至于用什么方式清，清了之后要不要整理这都是后话。

标记-清除具体的做法是定期或者内存不足时进行垃圾回收，从根引用(GC Roots)开始遍历扫描，将所有扫描到的对象标记为可达，然后将所有不可达的对象回收了。

所谓的根引用包括全局变量、栈上引用、寄存器上的等

<font size=5 color = "green">而不论标记-清楚还是引用计数，其实都只关心引用类型，像一些整型啥的就不需要管。

所以 JVM 还需要判断栈上的数据是什么类型，这里又可以分为保守式 GC、半保守式 GC、和准确式 GC。</font>

## 保守式GC
保守式 GC 指的是 JVM 不会记录数据的类型，也就是无法区分内存上的某个位置的数据到底是引用类型还是非引用类型。

因此只能靠一些条件来猜测是否有指针指向。比如在栈上扫描的时候根据所在地址是否在 GC 堆的上下界之内，是否字节对齐等手段来判断这个是不是指向 GC 堆中的指针。

之所以称之为保守式 GC 是因为不符合猜测条件的肯定不是指向 GC 堆中的指针，因此那块内存没有被引用，而符合的却不一定是指针，所以是保守的猜测。

但是就有可能出现恰好有数值的值就是地址的值。

这就混乱了，所以就不能确定这是指针，只能保守认为就是指针。

因此肯定不会有误杀对象的情况。只会有对象已经死了，但是有疑似指针的存在指向它，误以为它还活着而放过了它的情况发生。


## 半保守式GC
半保守式GC，在对象上会记录类型信息而其他地方还是没有记录，因此从根扫描的话还是一样，得靠猜测。

但是得到堆内对象了之后，就能准确知晓对象所包含的信息了，因此之后 tracing 都是准确的，所以称为半保守式 GC。

现在可以得知半保守式 GC 只有根直接扫描的对象无法移动，从直接对象再追溯出去的对象可以移动，所以半保守式 GC 可以使用移动部分对象的算法，也可以使用标记-清除这种不移动对象的算法。

而保守式 GC 只能使用标记-清除算法。

## 准确式 GC
相信大家看下来已经知道准确意味 JVM 需要清晰的知晓对象的类型，包括在栈上的引用也能得知类型等。

能想到的可以在指针上打标记，来表明类型，或者在外部记录类型信息形成一张映射表。

HotSpot 用的就是映射表，这个表叫 OopMap。

在 HotSpot 中，对象的类型信息里会记录自己的 OopMap，记录了在该类型的对象内什么偏移量上是什么类型的数据，而在解释器中执行的方法可以通过解释器里的功能自动生成出 OopMap 出来给 GC 用。

被 JIT 编译过的方法，也会在特定的位置生成 OopMap，记录了执行到该方法的某条指令时栈上和寄存器里哪些位置是引用。

这些特定的位置主要在：

- 循环的末尾（非 counted 循环）
- 方法临返回前 / 调用方法的call指令后
- 可能抛异常的位置


这些位置就叫作安全点(safepoint)。

那为什么要选择这些位置插入呢？因为如果对每条指令都记录一个 OopMap 的话空间开销就过大了，因此就选择这些个关键位置来记录即可。

<font size=4 color="blue">所以在 HotSpot 中 GC 不是在任何位置都能进入的，只能在安全点进入。</font>

至此我们知晓了可以在类加载时计算得到对象类型中的 OopMap，解释器生成的 OopMap 和 JIT 生成的 OopMap ，所以 GC 的时候已经有充足的条件来准确判断对象类型。

因此称为准确式 GC。

其实还有个 JNI 调用，它们既不在解释器执行，也不会经过 JIT 编译生成，所以会缺少 OopMap。

在 HotSpot 是通过句柄包装来解决准确性问题的，像 JNI 的入参和返回值引用都通过句柄包装起来，也就是通过句柄再访问真正的对象。

这样在 GC 的时候就不用扫描 JNI 的栈帧，直接扫描句柄表就知道 JNI 引用了 GC 堆中哪些对象了。


# 在什么情况下，GC会对程序产生影响
不管YGC还是FGC，都会造成一定程度的程序卡顿（即Stop The World问题：GC线程开始工作，其他工作线程被挂起），即使采用ParNew、CMS或者G1这些更先进的垃圾回收算法，也只是在减少卡顿时间，而并不能完全消除卡顿。

那到底什么情况下，GC会对程序产生影响呢

- FGC过于频繁：FGC通常是比较慢的，少则几百毫秒，多则几秒，正常情况FGC每隔几个小时甚至几天才执行一次，对系统的影响还能接受。但是，一旦出现FGC频繁（比如几十分钟就会执行一次），这种肯定是存在问题的，它会导致工作线程频繁被停止，让系统看起来一直有卡顿现象，也会使得程序的整体性能变差。

- YGC耗时过长：一般来说，YGC的总耗时在几十或者上百毫秒是比较正常的，虽然会引起系统卡顿几毫秒或者几十毫秒，这种情况几乎对用户无感知，对程序的影响可以忽略不计。但是如果YGC耗时达到了1秒甚至几秒（都快赶上FGC的耗时了），那卡顿时间就会增大，加上YGC本身比较频繁，就会导致比较多的服务超时问题。
- FGC耗时过长：FGC耗时增加，卡顿时间也会随之增加，尤其对于高并发服务，可能导致FGC期间比较多的超时问题，可用性降低，这种也需要关注。
- YGC过于频繁：即使YGC不会引起服务超时，但是YGC过于频繁也会降低服务的整体性能，对于高并发服务也是需要关注的。

# 排查FGC问题的实践指南
## 1. 清楚从程序角度，有哪些原因导致FGC？ 
- 大对象：系统一次性加载了过多数据到内存中（比如SQL查询未做分页），导致大对象进入了老年代。
- 内存泄漏：频繁创建了大量对象，但是无法被回收（比如IO对象使用完后未调用close方法释放资源），先引发FGC，最后导致OOM.
- 程序频繁生成一些长生命周期的对象，当这些对象的存活年龄超过分代年龄时便会进入老年代，最后引发FGC. （即本文中的案例）
- 程序BUG导致动态生成了很多新类，使得 Metaspace 不断被占用，先引发FGC，最后导致OOM.
- 代码中显式调用了gc方法，包括自己的代码甚至框架中的代码。
- JVM参数设置问题：包括总内存大小、新生代和老年代的大小、Eden区和S区的大小、元空间大小、垃圾回收算法等等。

## 2. 清楚排查问题时能使用哪些工具
公司的监控系统：大部分公司都会有，可全方位监控JVM的各项指标。

JDK的自带工具，包括jmap、jstat等常用命令：

查看堆内存各区域的使用率以及GC情况
> jstat -gcutil -h20 pid 1000


查看堆内存中的存活对象，并按空间排序
> jmap -histo pid | head -n20


dump堆内存文件
> jmap -dump:format=b,file=heap pid


可视化的堆内存分析工具：JVisualVM、MAT等


## 3. 排查指南

- 查看监控，以了解出现问题的时间点以及当前FGC的频率（可对比正常情况看频率是否正常）
- 了解该时间点之前有没有程序上线、基础组件升级等情况。
- 了解JVM的参数设置，包括：堆空间各个区域的大小设置，新生代和老年代分别采用了哪些垃圾收集器，然后分析JVM参数设置是否合理。
- 再对步骤1中列出的可能原因做排除法，其中元空间被打满、内存泄漏、代码显式调用gc方法比较容易排查。
- 针对大对象或者长生命周期对象导致的FGC，可通过 jmap -histo 命令并结合dump堆内存文件作进一步分析，需要先定位到可疑对象。
- 通过可疑对象定位到具体代码再次分析，这时候要结合GC原理和JVM参数设置，弄清楚可疑对象是否满足了进入到老年代的条件才能下结论。
