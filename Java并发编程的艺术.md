---
typora-root-url: ./
typora-copy-images-to: ./
---

# 1 并发编程的挑战

## 1.1 上下文切换

### 1.1.1 多线程一定快吗

### 1.1.2 测试上下文切换次数和时长

### 1.1.3 如何减少上下文切换

减少上下文切换的方法有无锁并发编程、CAS算法、使用最少线程和使用协程。

* 无锁并发编程  多线程竞争锁时，会引起上下文切换，所以多线程处理数据时，可以用一些办法来避免使用锁，如将数据的ID按照Hash算法取模分段，不同的线程处理不同段的数据。
* CAS算法  Java的Atomic包使用CAS算法来更新数据，而不需要加锁。
* 使用最少线程  避免创建不需要的线程，比如人物很少，但是创建了很多线程来处理，这样会造成大量线程都处于等待状态。
* 协程  在单线程里实现多任务的调度，并在单线程里维持多个任务之间的切换。

### 1.1.4 减少上下文切换实战

## 1.2 死锁

避免死锁的几种常见方法

* 避免一个线程同时获取多个锁
* 避免一个线程在锁内同时占用多个资源，尽量保证每个锁只占用一个资源
* 尝试使用定时锁，使用lock.tryLock(timeout)来替代使用内部锁机制
* 对于数据库锁，加锁和解锁必须在一个数据库连接里，否则会出现解锁失败的情况

# 2 Java并发机制的底层实现原理

## 2.1 volatile的应用

<font color="blue">1.volatile的定义与实现原理</font>

Java语言规范第3版中对volatile的定义如下： Java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致地更新，线程应该确保通过排他锁单独获得这个变量。Java语言提供了volatile，在某些情况下比排他锁更加方便。如果一个字段被声明了volatile，Java线程内存模型确保所有线程看到这个变量的值是一致的。

<center>CPU术语定义</center>

| 术语       | 英文单词               | 术语描述                                                     |
| ---------- | ---------------------- | ------------------------------------------------------------ |
| 内存屏障   | memory barries         | 是一组处理器指令，用于实现堆内存操作的顺序限制               |
| 缓冲行     | cache line             | 缓存中可以分配的最小存储单位。处理器填写缓存线时会加载整个缓存线，需要使用多个主内存读周期 |
| 原子操作   | atomic operation       | 不可中断的一个或一系列操作                                   |
| 缓存行填充 | cache file fill        | 档处理器识别到从内存中读取操作数是可缓存的，处理器读取整个缓存行到适当的缓存（L1,L2,L3的或所有） |
| 缓存命中   | cache hit              | 如果进行高速缓存行填充操作的内存位置仍然是下次处理器访问的地址时，处理器从缓存中读取操作数，而不是从内存读取 |
| 写命中     | write hit              | 当处理器将操作数写回到一个内存缓存区域时，它首先会检查这个缓存的内存地址是否在缓存行中，如果存在一个有效的缓存行，则处理器将这个操作数写回到缓存，而不是写回到内存，这个操作被称为写命中 |
| 写缺失     | write misses the cache | 一个有效的缓存行被写入到不存在的内存区域                     |



## 2.2 synchronized的实现原理与应用

Java中的每一个对象都可以作为锁。具体表现为以下3种形式

- 对于普通方法，锁是当前实例对象
- 对于静态同步方法，锁是当前类的Class对象
- 对于同步方法块，锁是Synchronized括号里配置的对象

JVM基于进入和退出Monitor对象来实现方法同步和代码同步，但两者的实现细节不一样。代码块同步是使用monitorenter和monitorexit指令实现的，而方法同步是使用另外一种方式实现的，细节在JVM规范里并没有详细说明。但是，方法的同步同样可以使用这两个指令来实现。

monitorenter指令是在编译后插入到代码块的开始位置，而monitorexit是插入到方法结束处和异常处，JVM要保证每个monitorenter必须有对应的monitorexit与之配对。任何对象都有一个monitor与之关联，当且一个monitor被持有后，它将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor的所有权，即尝试获得对象的锁。

### 2.2.1 Java对象头

synchronized用的锁是存在Java对象头里的。如果对象是数组类型，则虚拟机用3个字宽（Word）存储对象头，如果对象是非数组类型，则用2字节宽存储对象头。在32位虚拟机中，1字宽等于4字节，即32bit

<center>表2-2 Java对象头长度</center>

| 长度     | 内容                   | 说明                             |
| -------- | ---------------------- | -------------------------------- |
| 32/64bit | Mark Word              | 存储对象的hashCode或锁信息       |
| 32/64bit | Class Metadata Address | 存储对象类型数据指针             |
| 32/32bit | Array length           | 数组的长度（如果当前对象是数组） |

Java对象头里的Mark Word里默认存储对象是HashCode、分代年龄和锁标记位。32位JVM的Mark Word的默认存储结构如表2-3所示。

<center>表2-3 Java对象头的存储结构</center>

| 锁状态   | 25bit          | 4bit         | 1bit是否偏向锁 | 2bit锁标志位 |
| -------- | -------------- | ------------ | -------------- | ------------ |
| 无锁状态 | 对象的HashCode | 对象分代年龄 | 0              | 01           |

在运行期间，Mark Word里存储的数据会随着锁标记位的变化而变化。Mark Word可能变化为存储以下4种数据，如表2-4所示

<center>表2-4 Mark Word的状态变化</center>

![1541064612633](/1541064612633.png)

在64位虚拟机下，Mark Word是64bit大小的，其存储结构如2-5所示

<center>表2-5  Mark Word的存储结构</center>

![1541064636709](/1541064636709.png)

### 2.2.2 锁的升级与对比

## 2.3 原子操作的实现原理

原子操作意为“不可被中断的一个或一系列操作”。

<font color="blue">1.术语定义</font>

<center>CPU术语定义</center>

| 术语名称     | 英文                   | 解释                                                         |
| ------------ | ---------------------- | ------------------------------------------------------------ |
| 缓存行       | Cache line             | 缓存的最小操作单位                                           |
| 比较并交换   | Compare and Swap       | CAS操作需要输入两个数值，一个旧值（期望操作前的值）和一个新值，在操作期间先比较旧值有没有发生变化，如果没有发生变化，才交换成新值，发生变化则不交换 |
| CPU流水线    | CPU pipeline           | CPU流水线的工作方式就像工业生产上的配置流水线，在CPU中由5~6个不同功能的电路单元组成一条指令处理流水线，然后将一条X86指令分成5~6步后再由这些电路单元分别执行，这样就能实现在一个CPU时钟周期完成一条指令，因此提高运算速度 |
| 内存顺序冲突 | Memory order violation | 内存顺序冲突一般是由假共享引起的，假共享是指多个CPU同时修改同一个缓存行的不同部分而引起其中一个CPU的操作无效，当出现这个内存顺序冲突时，CPU必须清空流水线 |

<font color="blue">2.处理器如何实现原子操作</font>

32位IA-32处理器使用基于缓存加锁或总线加锁的方式来实现多处理器之间的原子性。首先处理器会自动保证基本的内存操作的原子性。处理器保证从系统内存中读取或写入一个字节是原子的，意思是当一个处理器读取一个字节时，其他处理器不能访问这个字节的内存地址。Pentium 6和最新的处理器能自动保证单处理器对同一个缓存行里进行16/32/64位的操作是原子的，但是复杂的内存操作处理器是不能自动保证其原子性的，比如跨总线宽度、跨多个缓存航和跨页表的访问。但是，处理器提供总线锁定和缓存锁定两个两个机制来保证复杂内存操作的原子性。

<font color="blue">3.Java如何实现原子操作</font>

在Java中可以通过锁和循环CAS的方式来实现原子操作

(1)使用循环CAS实现原子操作

JVM中的CAS操作利用了处理器提供的CMPXCHG指令实现的。自旋CAS实现的基本思路就是循环进行CAS操作直到成功为止。

从Java1.5开始，JDK并发包里提供了一些类来支持原子操作，如AtomicBoolean、AtomicInteger、AtomicLong。

(2) CAS实现原子操作的三大问题

1) ABA问题。CAS需要在操作值的时候，检查值有没有发生变化，如果没有发生变化则更新，一旦一个值原来是A，变成了B，又变成A，那么使用CASA进行检查时会发现它的值没有发生变化，但实际变化了.ABA问题的解决思路就是使用版本号。

2)循环时间长开销大

3)只能保证一个共享变量的原子操作

# 3 Java内存模型

## 3.1 Java内存模型的基础

### 3.1.3 从源代码到指令序列的重排序

在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排序。重排序分为3种类型

1）**编译器优化**的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序

2）**指令级并行**的重排序。现代处理器采用了指令级并行技术（Instruction-Level Parallelism, ILP）来讲多条指令进行重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。

3）**内存系统**的重排序。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行

从Java源码到最终实际执行的指令序列，会分别经历下面的3中重排序

源代码->1:编译器优化重排序->2:指令级并行重排序->3:内存系统重排序->最终执行的指令序列

上述1属于编译器重排序，2和3属于处理器重排序。这些重排序可能会导致多线程程序出现内存可见性问题。对于编译器，JMM的编译器重新排序规则会禁止特定类型的编译器重排序（不是所有的编译器重排序都要禁止）。对于处理器重排序，JMM的处理器冲排序规则会要求Java编译器在生成指令序列时，插入特定类型的内存屏障（Memory Barries，Intel称之为Menory Fence）指令，通过内存屏障指令来禁止特定类型的处理器重排序。

JMM属于语言级的内存模型，它确保在不同的编译器和不同的处理器平台之上，通过禁止特定类型的编译器重排序和处理器重排序，为程序员提供一致的内存可见性保证。

### 3.1.5 happens-before简介

从JDK5开始，Java使用新的JSR-133内存模型。JSR-133使用hapens-before的概念来阐述操作之间的内存可见性。在JMM中，如果一个操作执行结果需要对另一个操作可见，那么这两个操作之间必须要存在happens-before关系。这里提到的两个操作既可以是在一个线程之内，也可以是在不同线程之间。

与程序员密切相关的hapopens-before规则如下

* 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作
* 监视器锁原则：对一个锁的解锁，happens-before于随后对这个锁的加锁
* volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读
* 传递性：如果A happens-before B,且B happens-before C，那么A happens-before C

<font color="blue">注意</font> 两个操作之间具有happens-before关系，并不意味着前一个操作必须在后一个操作之前执行！happens-before仅仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前（the first is visible to and ordered before the second）。

<center>图3-5 happens-before与JMM的关系</center>

![1541678721163](/1541678721163.png)

## 3.2 重排序

### 3.2.1 数据依赖性

如果两个操作访问同一个变量，且这两个操作中有一个为写操作，此时这两个操作之间就存在数据依赖性。

<center>表3-4 数据依赖类型表</center>

| 名称   | 代码示例 | 说明                         |
| ------ | -------- | ---------------------------- |
| 写后读 | a=1;b=a; | 写一个变量之后，再读这个位置 |
| 写后写 | a=1;a=2; | 写一个变量之后，再写这个变量 |
| 读后写 | a=b;b=1; | 读一个变量之后，再写这个变量 |

### 3.2.2 as-if-serial语义

as-if-serial语义的意思是：不管怎么重排序（编译器和处理器为了提高并行度），（单线程）程序的执行结果不能被改变。编译器、runtime和处理器都必须遵守as-if-serial语义

## 3.3 顺序一致性

顺序一致性内存模型是一个理论参考模型，在设计的时候，处理器的内存模型和编程语言的内存模型都会以顺序一致性内存模型作为参照

### 3.3.1 数据竞争与顺序一致性

## 3.6 final域的内存语义

与锁和volatile相比，对final域的读和写更像是普通变量的访问。

### 3.6.1 final域的重排序规则

对于final域，编译器和处理器要遵守两个重排序规则

1) 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

2）初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。

下面通过一些示例性代码分别说明这两个规则

```java
public class FinalExample {
	int i;                                // 普通变量
    final int j;                          // final变量
    static FinalExample obj;              
    public FinalExample() {               // 构造函数
    	i = 1;                            // 写普通域
        j = 2;                            // 写final域
    }
    public static void writer() {         // 写线程A执行
    	obj = new FinalExample();
    }
    public static void reader() {         // 读线程B执行
    	FinalExample object = obj;        // 读对象引用
        int a = object.i;                 // 读普通域
        int b = object.j;                 // 读final域
    }
}
```



### 3.6.2 写final域的重排序规则

写final域的重排序规则禁止把final域的写重排序到构造函数之外。这个规则的实现包含下面2个方面

1）JMM禁止编译器把final域的写重排序到构造函数之外。

2）编译器会在final域的写之后，构造函数return之前，插入一个StoreStore屏障。这个屏障禁止处理器吧final域的写重排序到构造函数之外。

### 3.6.3 读final域的重排序规则

读final域的重排序规则是，在一个线程中，初次读对象引用域初次读该对象包含的final域，JMM禁止处理器重排序这两个操作（注意，这个规则仅仅针对处理器）。编译器会在读final域操作的前面插入LoadLoad屏障。

# 4 Java并发编程基础

## 4.1 线程简介

### 4.1.4 线程的状态

Java线程在运行的生命周期中可能处于表4-1所示的6种不同的状态，在给定的一个时刻，线程只能处于其中的一个状态。

<center>表4-1 Java线程的状态</center>

| 状态名称     | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| NEW          | 初始状态，线程被构建，但还没有调用start()方法                |
| RUNNABLE     | 运行状态，Java线程将操作系统中的就绪和运行两种状态笼统地称作“运行中” |
| BLOCKED      | 阻塞状态，表示线程阻塞于锁                                   |
| WAITING      | 等待状态，表示线程进入等待状态，进入该状态表示当前线程需要等待其他线程做出一些特定动作（通知或中断） |
| TIME_WAITING | 超时等待状态，该状态不同于WAITING，它是可以在指定的时间自行返回的 |
| TERMINATED   | 终止状态，表示当前线程已经执行完毕                           |

<center>图4-1 Java线程状态变迁</center>

![1541064662964](/1541064662964.png)

<font color="red">注意</font>   Java将操作系统中的运行和就绪两个状态合并称为运行状态。阻塞状态是线程阻塞在进入synchronized关键字修饰的方法或代码块（获取锁）时的状态，但是阻塞在java.concurrent包中的Lock接口的线程状态却是等待状态，因为java.concurrent包中的Lock接口对于阻塞的实现均使用了LockSupport类中的相关方法。

### 4.1.4 Daemon线程

Daemon线程是一种支持型线程，因为它主要用作程序中后台调度以及支持性工作。这意味着，当一个Java虚拟机中不存在非Daemon线程的时候，Java虚拟机将退出。可以通过调用Thread.setDaemon(true)将线程设置为Daemon线程。

<font color="blue">注意</font> Daemon属性需要在启动线程之前设置，不能在启动线程之后设置

# 5 Java中的锁

## 5.1 Lock接口

<center>表5-1 Lock接口提供的synchronized关键字不具备的主要特性</center>

| 特性               | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| 尝试非阻塞地获取锁 | 当线程尝试获取锁，如果这一时刻锁没有被其他线程获取到，则成功获取并持有锁 |
| 能中断地获取锁     | 与synchronized不同，获取到锁的县城能够响应中断，当获取到锁的线程被中断时，中断异常将被抛出，同时锁会被释放 |
| 超时获取锁         | 在指定的截止时间之前获取锁，如果截止时间到了仍旧无法获取锁，则返回 |



## 5.2 队列同步器

队列同步器AbstractQueuedSynchronizer（简称同步器），是用来构建锁或其他同步组件的基础框架，它使用一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程和排队工作。

AQS的主要使用方式是继承，通过AQS提供的3个方法getState()、setState(int newState)和compareAndSetState(int expect, int update)进行操作同步状态更改；它们能够保证状态的改变是安全的。子类推荐被定义为自定义同步组件的静态内部类，同步器自身没有实现任何同步接口，它仅仅定义了若干同步状态获取和释放的方法来供自定义同步组件使用，AQS支持独占式获取同步状态，也支持共享式获取同步状态，这样就可以实现不同类型的同步组件（ReentrantLock、ReentrantReadWriteLock和CountDownLatch等）

AQS是实现锁（也可以是任意同步组件）的关键，在锁的实现中聚合同步器，利用同步器实现锁的语义。可以这样理解二者之间的关系：锁是面向使用者的，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；AQS面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。锁和同步器很好地隔离了使用者与实现者所需关注的领域

### 5.2.2 队列同步器的实现分析

从实现角度分析同步器时如何完成线程同步的，主要包括：同步队列、独占式同步状态取与释放、共享式同步状态取与释放以及超时获取同步状态等同步器的核心数据结构与模板方法

<font color="blue">1.同步队列</font>

同步器以来内部的同步队列（一个FIFO双向队列）来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态信息构造成为一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，将其再次尝试获取同步状态。

同步队列中的节点（Node）用来保存获取同步状态失败的线程引用、等待状态以及前驱和后继节点，节点的属性类型与名称以及描述如表5-5所示

<center>表 5-5 节点的属性类型与名称以及描述</center>

![1541419200913](/1541419200913.png)

节点是构成同步队列（等待队列，在5.6节中将会介绍）的基础，同步器拥有首节点（head）和尾节点（tail），没有成功获取同步状态的线程将会成为节点加入该队列的尾部，同步队列的基本结构如图5-1所示。

<center>图5-1 同步队列的基本结果</center>

![1541512637485](/1541512637485.png)

同步器包含了两个节点类型的引用，一个指向头结点，而另一个指向尾节点。当一个线程成功获取了同步状态（或者锁），其他线程将无法取到同步状态，转而被构造成节点并加入到同步队列中，而这个加入队列的过程必须要保证线程安全，因此同步器提供了一套基于CAS的设置为节点的方法：compareAndSetTail(Node expect, Node update)，它需要传递当前线程“认为”的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联。

<center>图5-2 节点加入到同步队列</center>

![1541513872022](/1541513872022.png)

同步队列遵循FIFO，首节点是获取同步状态成功的节点，首节点的线程在释放同步状态时，将会唤醒后继节点，而后继节点将会在获取同步状态成功时将自己设置为首节点，该过程如图5-3所示

<center>图5-3 首节点的设置</center>

![1541514059784](/1541514059784.png)

设置首节点是通过获取同步状态成功的线程来完成的，由于只有一个线程能够成功获取到同步状态，因此设置头结点的方法并不需要使用CAS来保证，它只需要将首节点设置成为原首节点的后继节点并断开原首节点的next引用即可。

<font color="blue">2.独占式同步状态获取与释放</font>

通过调用AQS的acquire(int arg)方法可以获取同步状态，该方法对中断不敏感，也就是线程同步状态失败后进入同步队列中，后续对线程进行中断操作时，线程不会从同步队列中移出，该方法的代码如下

<center>代码清单5-3 同步器acquire方法</center>

```
public final void acquire(int arg) {
    if(!tryAcqire(arg) && acqireQueued(addWaiter(Node.EXCLUSIVE), arg))
    	selfInterrupt();
}
```



<font color="blue">3.共享式同步状态获取与释放</font>

<font color="blue">4.独占式超时获取同步状态</font>

<font color="blue">5.自定义同步组件——TwinsLock</font>

## 5.3 重入锁

表示该锁能够支持一个线程对资源重复加锁。除此之外，该锁还支持取锁时的公平性和非公平性选择

#### 1.实现重进入

1）线程再次获取锁。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再次成功获取

2）锁的最终释放。重复n此获取了锁，随后在第n次释放锁后，其他线程能够获取到该锁。锁的最终释放要求锁对于获取进行计数自增，锁释放时计数自减，当计数为0时表示锁已经成功释放

## 5.4 读写锁

## 5.5 LockSupport工具

当需要阻塞或唤醒一个线程的时候，都会使用LockSupport工具来完成相应的工作。LockSupport定义了一组公共静态方法，这些方法提供了最基本的线程阻塞和唤醒功能，而LockSupport也称为构建同步组件的基础工具。

LockSupport定义了一组以park开头的方法来阻塞当前线程，以及unpack(Thread thread)方法来唤醒一个被阻塞的线程。Park有停车的意思，unpack则是指车辆启动离开

<center>表5-10 LockSupport提供的阻塞和唤醒方法</center>

| 方法名称                      | 描述                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| void park()                   | 阻塞当前线程，如果调用unpark(Thread thread)方法或者当前线程被中断，才能从park()方法返回 |
| void parkName(long nanos)     | 阻塞当前线程，最长不超过nanos纳秒，返回条件在park()的基础上增加了超时返回 |
| void parkUntil(long deadline) | 阻塞当前线程，知道deadline时间（从1970年开始到deadline时间的毫秒数） |
| void unpark(Thread thread)    | 唤醒处于阻塞状态的线程thread                                 |

Java 6中，LockSupport增加了park(Object blocker)、parkNanos(Object blocker, long nanos)和parkUtil(Object blocker, long deadline)3个方法，用于实现阻塞当前线程的功能，其中参数blocker是用来标识当前线程在等待的对象（以下称为阻塞对象），该对象主要用于问题排查和系统监控。



## 5.6 Condition接口

任意一个Java对象，都拥有一组监视器方法（定义在java.lang.Object上），主要包括wait()、wait(long timeout)、notify()以及notifyAll()方法，这些方法与synchronized同步关键字配合，可以实现等待/通知模式。Condition接口也提供了类似Object的监视方法，与Lock配合可以实现等待/通知模式，但是这两者在使用方式以及功能特性上还是有差别的。



# 6 Java并发容器和框架

<font color="orange">三大并发容器:ConcurrentHashMap、ConcurrentSkipListMap、ConcurrentLinkedQueue</font>

## 6.1 ConcurrentHashMap的实现原理与使用

### 6.1.1 为什么要使用ConcurrentHashMap

(1)线程不安全的HashMap

在多线程环境下，使用HashMap进行put操作会引起死循环，导致CPU利用率接近100%，所以并发情况下不能使用HashMap.例如，执行下面代码会引起死循环

```java
final HashMap<String, String> map = new HashMap<String, String>(2);
Thread t = new Thread(new Runnable() {
    @Override
    public void run() {
        for(int i = 0; i < 10000; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    map.put(UUID.randomUUID().toString(), "");
                }
            }, "ftf" + i).start();
        }
    }
}, "ftf");
t.start();
t.join();
```

HashMap在并发执行put操作时会引起死循环，是因为多线程会导致HashMap的Entry链表形成环形数据结构，一旦形成环形数据结构，Entry的next节点永远不为空，就会产生死循环获取Entry。

(2)效率低下的HashTable

HashTable容器使用synchronized来保证线程安全，但在线程竞争激烈的情况下，HashTable效率非常低下。因为当一个线程访问HashTable的同步方法，其他线程也访问HashTable的同步方法时，会进入阻塞或轮询状态。如果线程1使用put进行元素添加，线程2不但不能使用put方法添加元素，也不能使用get方法来获取元素，所以竞争越激烈效率越低。

### 6.1.2 ConcurrentHashMap的结构

## 6.2 ConcurrentLinkedQueue

### 6.2.1 ConcurrentLinkedQueue的结构

## 6.3 Java中的阻塞队列

<font color="orange">六大阻塞队列：ArrayBlockingQueue、PriorityBlockingQueue、DelayQueue、SynchronousQueue、LinkedTransferQueue、LinkedBlockingQueue</font>

### 6.3.1 什么是阻塞队列

阻塞队列（BlockingQueue)是一个支持两个附加操作的队列。这两个附加操作支持阻塞的插入和移除方法。

## 6.4 Fork/Join框架

### 6.4.1 什么是Fork/Join框架

Fork/Join框架是Java 7提供的一个用于并行执行任务的框架，是一个把大任务分割成若干小任务，最终汇总每个小任务结果后得到大任务结果的框架。

Fork就是把一个大任务切分为若干子任务并行的执行，Join就是合并这些子任务的执行结果，最后得到这个大人物的结果。

### 6.4.4 使用Fork/Join框架

# 7 Java中的13个原子操作类

# 8 Java中的并发工具类

<font color="orange">四大并发工具:CountDownLatch、CyclicBarrier、Semaphore、Exchanger</font>

## 8.1等待多线程完成的CountDownLatch

CountDownLatch允许一个或多个线程当代其他线程完成操作

当我们调用CountDownLatch的countDown方法时，N就会减1，CountDownLatch的await方法会阻塞当前线程，直到N变成零。由于countDown方法可以用在任何地方，所以这里说的N个点，可以是N个线程，也可以是1个线程里的N个执行步骤。用在多线程时，只需要把这个CountDownLatch的引用传递到线程里即可

## 8.2 同步屏障 CyclicBarrier

CyclicBarrier的字面意思是可循环使用(Cyclic)的屏障（Barrier）。它要做的事情是，让一组线程到达屏障时（也可以叫做同步点）被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有屏障拦截的线程才会继续运行。

# 10 Executor框架

Java线程的创建与销毁需要一定的开销，如果为每一个任务创建一个新线程来执行，这些线程的创建与销毁将消耗大量的计算资源。

JDK5开始，把工作单元与执行机制分离开来。工作单元包括Runnable和Callable，而执行机制由Executor框架提供。

## 10.1 Executor框架简介

### 10.1.1 Executor框架的两级调度模型

HotSpot VM中，Java线程被一对一映射为本地操作系统线程。

在上层，Java多线程程序通常把应用分解为若干个任务，然后使用用户级的调度器（Executor框架）将这些任务映射为固定数量的线程；在底层，操作系统将这些线程映射到硬件处理器上。这种两级调度模型的示意图如图10-1所示。

应用程序通过Execurator框架控制上层的调度；而下层的调度由操作系统内核控制，下层的调度不受应用程序控制

### 10.1.2 Executor框架的结构成员

<center>图10-1 任务的两级调度模型</center>

![1541065550583](/1541065550583.png)

<font color="blue">1.Execurator框架的结构</font>

Execurator框架主要由3大部分组成如下

* 任务。包括被执行人无需要实现的接口：Runnable接口或Callable接口

* 任务的执行。包括任务执行机制的核心接口Executor，以及继承自Executor的ExecutorService接口。Executor框架有两个关键类实现了ExecutorService接口（ThreadPoolExecutor和ScheduledThreadPoolExecutor）

* 异步计算的结果。包括Future和实现Future接口的FutureTask类

Execurator框架包含的类和接口如下

* Execurator是一个接口，它是Execurator框架的基础，它将任务的提交与任务的执行分离开来
* ThreadPoolExecurator是线程池的核心实现类，用来执行被提交的任务
* ScheduledThreadPoolExecurator是一个实现类，可以在给定的延迟后运行命令，或者定期执行命令。ScheduledThreadPoolExecurator比Timer更灵活，功能更强大
* Future接口和实现Future接口的FutureTask类，代表异步计算的结果
* Runnable接口和Callable接口的实现类，都可以被ThreadPoolExecurator或Scheduled-ThreadPoolExecurator执行。

主线程首先要创建Runnable或者Callable接口的任务对象。工具类Execurators可以把一个Runnable对象封装为一个Callable对象（Execurators.callable(Runnable task)或Execurators.submit(Callable<T> task)）.

然后可以把Runnable对象直接交给ExecuratorService执行（ExecuratorService.execute(Runnable command)）；或者也可以把Runnable对象或Callable对象提交给ExecutorService执行(ExecuratorService.submit(Runnable task)或ExecuratorService.submit(Callable<T> task))

