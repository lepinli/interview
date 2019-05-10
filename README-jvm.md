# JVM 基础 #

知识点汇总github地址 [https://github.com/bage2014/interview](https://github.com/bage2014/interview)

## JVM理论基础 ##

### 内存划分 ###
JVM规范，将内存分为 程序计数器、Java栈，也叫虚拟机栈、本地方法栈、方法区、堆

- 程序计数器
程序指令保存，从当前指令到下一个指令，从程序计数器获取下一个指令的地址，直到执行所有的指令；线程私有
- Java栈(虚拟机栈)
保存方法栈帧，当调用一个方法，则新创建一个栈帧，当前的方法始终保持在栈帧的顶部；线程私有
- 本地方法栈
保存**本地方法**栈帧，当调用一个**本地方法**，则新创建一个栈帧，当前的方法始终保持在栈帧的顶部；线程私有
- 方法区
保存类信息、静态常量、常量；线程共享
- 堆
保存对象，最最主要的垃圾收集之处；线程共享

### 对象存活 ###

- 引用计数算法
对对象引用进行计数，引用则计数器 +1，引用失效 -1；计数为 0 时候认为不在引用，可以进行回收；不能处理对象循环引用问题

- 可达性分析算法
主要采取此方式，采用虚拟机栈、类引用对象、常量对象、本地方法引用对象作为根，判断对象到根是否存在引用关系，不可达则认为不在引用，可以进行回收；

### GC回收算法 ###

- 标记-清除算法
对要回收的对象，先进行标志，后进行清除，你懂得；但是，久之，会存在内存不连续，比如，一次垃圾回收，回收了，(0,1)和(0,3)和(0,5)三个位置，但是没有回收(0，2)和(0，4)，那下次的内存，就无法使用(0，1)-(0，5)的连续空间；

- 复制算法
为改进上面的内存碎片问题而产生，对内存分为两部分，交替回收其中一部分，存活的对象复制到另一部分空间，你懂得；但是，有点浪费，比如，内存分为，(0,1)-(0,3)和(0,3)-(0,5)两个部分，某次回收(0,1)-(0,3)空间，将存活对象拷贝到(0,3)-(0,5)，而后在(0,1)-(0,3)分配对象；

- 标记-整理算法
为改进上面的内存浪费问题而产生，对要回收的对象，先进行标志，后进行清除，你懂得，然后，将存活的对象，进行整理，移动到边界位置，似的剩余空间连续；比如，一次垃圾回收，回收了，(0,1)和(0,3)和(0,5)三个位置，但是没有回收(0，2)和(0，4)，然后，将(0,2)和(0,4)移动到(0,1)和(0,2)，使得(0，3)-(0，5)的空间连续；