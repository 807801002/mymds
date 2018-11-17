---
typora-root-url: ./
typora-copy-images-to: ./
---

# 第二部分　自动内存管理机制

## 第2章 Java内存区域与内存溢出异常

### 2.2 运行时数据区域



### 2.2.4 Java堆



## 第4章　虚拟机性能监控与故障处理工具

### 4.2　JDK的命令行工具

#### 4.2.4　jmap：Java内存映像工具

<center>表4-4   jmap工具主要选项</center>

| 选项           | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| -dump          | 生成Java堆转储快照。格式为-dump:[live, ]format=b, file=<filename>,其中live子参数说明是否只dump出存活的对象 |
| -finalizerinfo | 显示在F-Queue中等待Finalizer线程执行finalize方法的对象。只在Linux/Solaris平台有效 |
| -heap          | 显示Java堆详细信息，如使用哪种回收器、参数配置、分代状况等。只在Linux/Solaris平台有效 |
| -histo         | 显示堆中对象统计信息，包括类、实例对象、合计容量             |
| -permstat      | 以ClassLoader为统计口径显示永久代内存状态，只在Linux/Solaris平台下有效 |
| -F             | 当虚拟机进程对-dump选项没有响应时，可使用这个选项强制生成dump快照。只在Linux/Solaris平台下有效 |

```
using parallel threads in the new generation.  ##新生代采用的是并行线程处理方式
using thread-local object allocation.   
Concurrent Mark-Sweep GC   ##同步并行垃圾回收

Heap Configuration:  ##堆配置情况
   MinHeapFreeRatio = 40 ##最小堆使用比例
   MaxHeapFreeRatio = 70 ##最大堆可用比例
   MaxHeapSize      = 2147483648 (2048.0MB) ##最大堆空间大小
   NewSize          = 268435456 (256.0MB) ##新生代分配大小
   MaxNewSize       = 268435456 (256.0MB) ##最大可新生代分配大小
   OldSize          = 5439488 (5.1875MB) ##老生代大小
   NewRatio         = 2  ##新生代比例
   SurvivorRatio    = 8 ##新生代与suvivor的比例
   PermSize         = 134217728 (128.0MB) ##perm区大小
   MaxPermSize      = 134217728 (128.0MB) ##最大可分配perm区大小

Heap Usage: ##堆使用情况
New Generation (Eden + 1 Survivor Space):  ##新生代（伊甸区 + survior空间）
   capacity = 241631232 (230.4375MB)  ##伊甸区容量
   used     = 77776272 (74.17323303222656MB) ##已经使用大小
   free     = 163854960 (156.26426696777344MB) ##剩余容量
   32.188004570534986% used ##使用比例
Eden Space:  ##伊甸区
   capacity = 214827008 (204.875MB) ##伊甸区容量
   used     = 74442288 (70.99369812011719MB) ##伊甸区使用
   free     = 140384720 (133.8813018798828MB) ##伊甸区当前剩余容量
   34.65220164496263% used ##伊甸区使用情况
From Space: ##survior1区
   capacity = 26804224 (25.5625MB) ##survior1区容量
   used     = 3333984 (3.179534912109375MB) ##surviror1区已使用情况
   free     = 23470240 (22.382965087890625MB) ##surviror1区剩余容量
   12.43827838477995% used ##survior1区使用比例
To Space: ##survior2 区
   capacity = 26804224 (25.5625MB) ##survior2区容量
   used     = 0 (0.0MB) ##survior2区已使用情况
   free     = 26804224 (25.5625MB) ##survior2区剩余容量
   0.0% used ## survior2区使用比例
concurrent mark-sweep generation: ##老生代使用情况
   capacity = 1879048192 (1792.0MB) ##老生代容量
   used     = 30847928 (29.41887664794922MB) ##老生代已使用容量
   free     = 1848200264 (1762.5811233520508MB) ##老生代剩余容量
   1.6416783843721663% used ##老生代使用比例
Perm Generation: ##perm区使用情况
   capacity = 134217728 (128.0MB) ##perm区容量
   used     = 47303016 (45.111671447753906MB) ##perm区已使用容量
   free     = 86914712 (82.8883285522461MB) ##perm区剩余容量
   35.24349331855774% used ##perm区使用比例
```



#### 4.2.6　jstack：Java堆栈跟踪工具

<center>表4-5 jstack工具主要选项</center>

| 选项 | 作用                                         |
| ---- | -------------------------------------------- |
| -F   | 当正常输出的请求不被响应时，强制输出线程堆栈 |
| -l   | 除堆栈外，显示关于锁的附加信息               |
| -m   | 如果调用本地方法的话，可以显示C/C++堆栈      |

# 第7章 虚拟机类加载机制

## 7.1 概述

虚拟机把描述类数据的Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是虚拟机的类加载机制

## 7.4 类加载器

### 7.4.2 双亲委派模型

从Java虚拟机的角度讲，只存在两种不同的类加载器：一种是启动类加载器（Bootstrap ClassLoader），这个类加载器使用C++语言实现，是虚拟机自身的一部分；另一种就是所有其他的类加载器，这些类加载器都由Java语言实现，独立于虚拟机外部，并且都继承自抽象类java.lang.ClassLoader.

从Java开发人员的角度看，类加载器还可以划分得更细致一些，绝大部分Java程序员都会使用到以下3种系统提供的类加载器

启动类加载器（Bootstrap ClassLoader）：前面已经介绍过，这个类加载器负责将存放在<JAVA_HOME>\lib目录中的，或者被-Xbootclasspath参数所指定的路径中的，并且是虚拟机识别的（仅按照文件名识别，如rt.jar，名字不符合的类库即使放在lib目录中页不会被加载）类库加载到虚拟机内存中。启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求为派给引导类加载器，那直接使用null替代即可，如代码清单7-9所示为java.lang.ClassLoader.getClassLoader()方法的代码片段

<center>代码清单7-9 ClassLoader.getClassLoader()方法的代码片段</center>

```java
/**
Returns the class loader for the class.Some implementations may use null to represent the bootstrap class loader.This method will return null in such implementations if this class was loaded by the bootstrap class loader.
*/
public ClassLoader getClassLoader() {
	ClassLoader cl=getClassLoader0();
	if(cl == null)
		return null;
	SecurityManager sm = System.getSecurityManager();
	if（sm！=null) {
		ClassLoader ccl = ClassLoader.getCallerClassLoader();
		if（ccl != null && ccl != cl && !cl.isAncestor(ccl)) {
			sm.checkPermission（SecurityConstants.GET_CLASSLOADER_PERMISSION）;
		}
	}
	return cl;
}
```

扩展类加载器（Extension ClassLoader）：这个加载器由sun.misc.Launcher$ExtClassLoader实现，它负责加载<JAVA_HOME>\lib\ext目录中的，或者被java.ext.dirs系统变量所指定的路径中的所有类库，开发者和可以直接使用扩展类加载器。

应用程序加载器（Application ClassLoader）：这个类加载器由sun.misc.Launcher$App-ClassLoader()实现。由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，所以一般也称它为系统类加载器。它负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的加载器，一般情况下这个就是程序中默认的加载器。

我们的应用程序都是由着3种类加载器互相配合进行加载的，如果有必要，还可以加入自己定义的类加载器。这些类加载器之间的关系如图7-2所示

<center>图7-2 类加载器双亲委派模型</center>

![1541403659828](/1541403659828.png)

图7-2中展示的类加载器之间的这种层次关系，称为类加载器的双亲委派模型（Parents Delegation Model）。双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应当有自己的父类加载器。这里类加载器之间的父子关系一般不会以继承（Inheritance）的关系来实现，而是都使用组合（Composition）关系来复用父加载器的代码。

类加载器的双亲委派模型在JDK1.2期间被引入并广泛应用于之后的几乎所有的Java程序中，但它并不是一个强制性的约束模型，而是Java设计者推荐给开发者的一种类加载器的实现方式。

双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载器请求最终都应该传送到顶层的启动类加载器中，只有当福类加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。

使用双亲委派模型来组织类加载器之间的关系，有一个显著的好处就是Java类随着它的类加载器一期具备了一种带有优先级的层次关系，例如类java.lang.Object，它存放在rt.jar之中，无论哪个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。相反，如果没有使用双亲委派模型，由各个类加载器自行去加在的话，如果用户自己编写了一个成为java.lang.Object类，Java类型体系中最基础的行为也就无法保证，应用程序也将变得一片混乱。尝试去编写一个与rt.jar类库中已有类重名的Java类，将会发现可以正常编译，但永远无法被加载运行。

### 7.4.3 破坏双亲委派模型

上文提到通过双亲委派模型并不是一个强制性的约束模型，而是Java设计者推荐给开发者的类加载器实现方式。

双亲委派模型的第一次“被破坏”发生在双亲委派模型出现之前——JDK1.2发布之前。由于双亲委派模型在JDK1.2之后才被引入，而类加载器和抽象类java.lang.classloader则在JDK1.0时代就已经存在，面对已经存在的用户自定义类加载器的实现代码，Java设计者引入双亲委派模型时不得不做出一些妥协。为了向前兼容，JDK1.2之后的java.lang.ClassLoader添加了一个新的protected方法findClass()，在此之前，用户去继承了java.lang.ClassLoader的唯一目的就是为了重写loadClass()fangfa ,因为虚拟机在进行类加载的时候会调用加载器的私有方法loadClassInternal()，而这个方法的唯一逻辑就是去调用自己的loadClass()。



OSGi环境下，类加载器不再是双亲委派模型中的树状结构，而是进一步发展为更加复杂的网状结构，当收到类加载器请求时，OSGi将按照下面的顺序进行类搜索

1)将以Java.*开头的类委派给父类加载器加载

2)否则，将委派列表名单内的类委派给父加载器加载

3)否则，将import列表中的类委派给Export这个类的Bundle的类加载器加载

4)否则，查找当前Bundle的ClassPath，使用自己的类加载器加载

5)否则，查找类是否在自己的Fragment Bundle中，如果在，则委派给Fragment Bundle的类加载器加载

6)否则，查找Dynamic Import列表的Bundle，委派给对应的Bundle的类加载器加载

7)否则，查找失败

# 第五部分 高效并发

## 第12章 Java内存模型与线程

下面是Java内存模型下一些“天然的”happens-before，这些happens-before无需任何同步器协助就已经存在，可以在编码中直接使用。如果两个操作之间的关系不在此列，并且无法从下列规则推导出来的话，它们就没有顺序性保障，虚拟机可以对它们任意地进行重排序

* 程序次序规则（Program Order Rule）在一个线程内，按照程序代码顺序，书写在前面的操作现行发生于书写在后面的操作。准确地说，应该是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结构
* 管程锁定规则（Monitor Lock Rule）一个unlock操作先行发生于后面同一个锁的lock操作。这里必须强调的是同一个锁，而“后面”是指时间上的先后顺序
* volatile变量规则（Volatile Variable Rule）对一个volatile变量的写操作先行发生于后面对这个变量的读操作，这里的“后面”同样是指时间上的先后顺序
* 线程启动规则（Thread Start Rule）Thread对象的start()方法先行发生于此线程的每一个动作
* 线程终止规则（Thread Termination Rule）线程中的所有操作斗先行于发生于对此线程的终止检测，我们通过Thread.join()方法结束、Thread.isAlive()的返回值等手段检测到线程已经终止执行
* 线程中断规则（Thread Interruption Rule）对线程interrupt()方法的调用先行与发生被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupted()方法检测到是否有中断发生
* 对象终结规则（Finalizer Rule）一个对象的初始化完成（构造函数执行结束）先行与发生于它的finalize()方法的开始
* 传递性（Transitivity）如果A先行发生于操作B，操作B先行发生于操作C，那就可以得出操作A先行发生于操作C的结论