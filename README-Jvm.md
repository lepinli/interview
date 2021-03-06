# JVM 基础 #

知识点汇总github地址 [https://github.com/bage2014/interview](https://github.com/bage2014/interview)

## JVM理论基础 ##

### 内存划分 ###

JVM规范，将内存分为 程序计数器、Java栈，也叫虚拟机栈、本地方法栈、方法区、堆

|      |      |      |      |
| ---- | ---- | ---- | ---- |
| 线程私有 | 程序计数器 | Java虚拟机栈 | 本地方法区 |
| 线程共享 | 方法区 | 堆 |  |

- 程序计数器
程序指令计数，从当前指令到下一个指令，从程序计数器获取下一个指令的地址，直到执行所有的指令；分支、循环、跳转都依赖于这个计数器实现；
- Java虚拟机栈
用于保存方法栈帧，每调用一个方法，会新创建一个栈帧，当前的方法始终保持在栈帧的顶部；存储局部变量表、动态链接、方法出口等；
- 本地方法栈
保存**本地方法**栈帧；与Java虚拟机栈类似；
- 堆
保存对象，几乎所有的对象都在堆中分配；最主要的垃圾收集之处；可以细分为新生代、老年代；更可以细分为Eden空间、From Survivor空间和To Survivor空间；可以通过  -Xms控制最小堆空间，-Xmx控制最大堆空间
- 方法区
保存类信息、常量、静态常量；运行时常量池，用于存放编译器生成的字面常量和符号引用

### 对象创建 ###

以hotspot 对象创建为例

1. 常量池中检查是否存在该类的符号引用？不存在，先执行类加载过程
2. 根据不同垃圾收集器的压缩整理功能，采用 “指针碰撞” 或 “空闲列表” 分配方式为新生对象进行内存分配
3. 内存分配过程中，要保证并发安全，采用CAS给内存分配过程加锁或将内存分配过程划分到每个线程的本地线程分配缓冲中进行；可以通过 -XX:+UseTLAB 参数控制；
4. 内存分配后，将分配到的内存空间初始化零值
5. 接下来，对对象的必要信息进行设置，包括对象属于哪个类的实例、对象哈希码、GC分代年龄等
6. 执行<init> 方法，进行程序初始化的逻辑

### 对象内存结构 ###

以hotspot 对象创建为例

| Java对象结构 | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| 对象头       | Mark Word：对象自身运行时数据；比如哈希码、GC年龄、锁标志位等 |
|              | 类型指针：非必须；指向对象所属的类；对象的元数据信息；数组长度记录 |
| 实例数据     | 对象的真正有效信息；父类子类信息都需要进行存储；             |
| 对齐填充     | 因对象大小必须是8的倍数，作对齐填充使用；非必须；            |

### 对象访问方式 ###

#### 句柄

​		 Java 栈本地变量表

​						||

​					 句柄池

​				         || 

对象实例数据   +   对象类型数据



#### 直接指针

​		 Java 栈本地变量表

​						||

对象实例数据   +   对象类型指针

​				         || 

​				对象类型数据

#### 对比

- 句柄；稳定，对象移动时，只会改变句柄中的实例数据指针，引用本身不修改
- 直接指针；快！比如hotspot 就是使用直接指针方式

### 对象存活 ###

如何确认一个对象是否可以回收？

#### 引用计数算法 ####

- 基本思想

给对象设置一个引用计数器；每一个引用，计数器加一，失效时候，计数器减一；计数器等于零时候，表示可以回收

- 优点

实现简单；效率高

- 缺点

不能处理对象循环引用问题

#### 可达性分析算法 ####

- 基本思想

采用虚拟机栈、类引用对象、常量对象、本地方法引用对象作为根，从根节点向下搜索，判断跟节点到当前对象节点是否存在引用关系，不可达则认为不在引用，可以进行回收

- 优点

可以解决对象循环引用问题

- 缺点

实现难度较大些，效率低些

#### 对象引用方式 ####

引用强度依次为：强引用 > 软引用 > 弱引用 > 虚引用

jdk1.2之后，才开始出现了 软、弱、虚引用；

- 强引用

普遍存在，最常见的引用；只要引用还在，jvm就不会回收；比如 Person p = new Person();

- 软引用

描述非必须的对象引用关系，在内存溢出之前被回收；比如 SoftReference<Object> obj = new SoftReference<>(someObj);

- 弱引用

描述非必须的对象引用关系，在下一次垃圾回收时被回收；比如 WeakReference<Object> obj = new WeakReference<>(someObj);

- 虚引用

最弱的对象引用关系，在对象被垃圾回收时时收到一个通知；虚引用必须和引用队列 （ReferenceQueue）联合使用，当对象被垃圾回收器准备回收时，则会把这个虚引用加入到与之关联的引用队列中；比如 PhantomReference<Object> obj3 = new PhantomReference<>(someObj,someQueue)；



### 垃圾回收算法 ###

- 标记清除算法

对要回收的对象，先进行标志，后进行清除；久之，会存在内存不连续；比如，一次垃圾回收，回收了，(0,1)和(0,3)和(0,5)三个位置，但是没有回收(0，2)和(0，4)，那下次的内存，就无法使用(0，1)-(0，5)的连续空间；标记、清除的效率都不高；

- 复制算法

为改进标记清除算法产生的内存碎片问题，对内存分为等大小两部分，交替回收其中一部分，存活的对象复制到另一部分空间；每次使用只能使用其中一半的内存，有点浪费，比如，内存分为，(0,1)-(0,3)和(0,3)-(0,5)两个部分，某次回收(0,1)-(0,3)空间，将存活对象拷贝到(0,3)-(0,5)，而后在(0,1)-(0,3)分配对象；但是高效简单；适用于新生代，新生代属于朝生夕死，可以按照特定比例进行回收，比如 8 : 1 : 1 ；这样每次只浪费 10 % 的内存空间；同时，当真的出现了超过 10% 的对象存活，则使用老年代进行担保；

- 标记整理算法

复制算法对于对象存活率较高的老年代，需要进行很多的复制，效率会降低；同时，存活对象也可能大于 50%，又没有其他的空间可以进行担保 ；老年代一般选取的是标记整理算法；对要回收的对象，先进行标志，后进行清除，然后，将存活的对象，进行整理，移动到边界位置，似的剩余空间连续；比如，一次垃圾回收，回收了，(0,1)和(0,3)和(0,5)三个位置，但是没有回收(0，2)和(0，4)，然后，将(0,2)和(0,4)移动到(0,1)和(0,2)，使得(0，3)-(0，5)的空间连续；

- 分代整理算法

不算一种新的思想算法，仅仅是根据不同的场景，进行了分代收集，采取不同的手机算法进行组装，进而选择合适的回收算法；一般来说；年轻代(新生代)采用的是复制算法，老年代采用标记清除或标记整理算法

#### 垃圾收集器 ####

- Serial & Serial Old 

单线程收集器；

对于单CPU来说，没有多线程交互开销，简单高效；

JDK1.3之前的唯一垃圾收集器，历史最悠久；

在垃圾收集回收过程中，会暂停其他用户所有的线程工作；

Serial 作用于新生代，采用复制算法；

Serial Old 作用于老年代，采用标记整理算法；

- ParNew

Serial 的多线程版本；

仅仅适用于新生代，采用复制算法；

对于单CPU来说，使用无意义，不如直接使用Serial收集器；

在垃圾收集回收过程中，同样会暂停其他用户所有的线程工作；

- Parallel Scavenge & Parallel Old

多线程收集；

Parallel Scavenge 始于JDK1.4，作用于新生代，采用复制算法；

Parallel Old 始于JDK1.6，作用于老年代，采用标记整理算法；

在垃圾收集回收过程中，同样会暂停其他用户所有的线程工作；

以吞吐量为设计关注点；

-XX:MaxGCPauseMilis 控制最大垃圾停顿时间；

-XX:GCTimeRatio 吞吐量大小设置；

-XX:UseAdaptiveSizePolicy 虚拟机自适应策略开关；

- CMS

以回收停顿时间为设计关注点；

仅仅适用于新生代，采用标记清除算法；

整体上说，因为耗时最长的并发标记和并发清除过程可以与用户线程并发执行，可以认为来回收过程可以于用户线程并发执行；

会存在一些不足，比如内存碎片、浮动垃圾、CPU敏感等

- G1



### 类加载过程 ###
jvm中class类的加载过程，大致分为这几个步骤

- 加载（load）
 - 根据全类名，加载类的二进制字节流
 - 将字节流转存方法区
 - 生成Class对象作为访问入口

- 验证（verify）
 - class文件的格式验证，验证是否符合JVM规范
 - class中的元数据验证，验证是否符合Java规范
 - class的字节码验证，验证数据流控制流不会危害JVM环境

- 准备（prepare）
 - 给变量分配内存
 - 初始化零值（比如int默认为0，boolean默认为false）
 - final变量直接赋值

- 解析
 - 符号引用变为直接引用
 - 类、字段、方法、接口方法解析

- 初始化
 - 初始化变量
 - 构造函数
 - static块


### 双亲委派机制 ###
Java中，大概有三种类型加载器，启动类加载器（Bootstrap）<- 标准扩展类加载器（Extension）<- 应用程序类加载器（Application ）<- 上下文类加载器(Custom)，从右到左，尽量父类进行加载，当父类无法进行加载时候，才会使用子类进行加载

- 意义
 - 防止同一个JVM，内存中出现两份class二进制字节码

- 加载过程
 - 从已加载的类查找是否已经存在，存在不需要再次加载
 - 若不存在，则去parent中查找，存在不需要再次加载
 - 若不存在，递归在parent中查找，直到找到为止
 - 若找遍所有parent均不存在，且当前加载器已经没有parent加载器，则调用当前类加载器的findClass方法，如果能加载，结束
 - 如果不能，则递归返回child类加载器，继续调用findClass方法，如果能加载，结束
 - 如果找遍所有child的findClass方法，还是不能加载，则抛出异常

- 破坏双亲委派机制
 - 将parent设为null
 - 重写load(String,boolean)方法，改变类的查找机制。

## JVM参数 ##

- -Xms

堆初始值 50M，此值可以设置与-Xmx相同，以避免每次垃圾回收完成后JVM重新分配内存

```
-Xms50m
```



- -Xmx

堆最大可用值 50M

```
-Xmx50m
```

xms 和 xmx 为什么要设置成一样？

设置-Xms、-Xmx 相等以避免在每次GC 后调整堆的大小。

这两个值一般怎么赋值？多大合适？



- -Xmn

新生代最大可用值1M

```
-Xmn1m
```

整个堆大小 = 年轻代大小 + 年老代大小 + 持久代大小



- -Xss

线程的私有栈大小1M

```
-Xss1m
```



- -XX:PrintGC

出发GC时，打印日志

```
-XX:+PrintGC
```



- -XX:PrintGCDetails

出发GC时，打印详细日志

```
-XX:+PrintGCDetails
```







