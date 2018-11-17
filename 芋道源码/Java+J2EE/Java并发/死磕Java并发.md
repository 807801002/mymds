# 深入分析 volatile 的实现原理

## 剖析volatile原理

> `volatile`可以保证线程可见性且提供了一定的有序性，但是无法保证原子性。在JVM底层，`volatile`采用**内存屏障**来实现的

上面那段话有两层语义：

* 保证可见性，不保证原子性
* 禁止指令重排序

# Java 内存模型之happens-before

在[《【死磕 Java 并发】—– 深入分析 volatile 的实现原理》](http://www.iocoder.cn/JUC/sike/volatile)中，提到过由于存在线程本地内存和主内存的原因，再加上重排序，会导致多线程环境下存在可见性的问题。那么我们正确使用同步、锁的情况下，线程A修改了变量`a`，何时对线程B可见？

> 在JMM中，如果一个操作的执行结果需要对另一个操作可见，那么这两个操作之间必须存在happens-before关系。

happens-before原则非常重要，它是判断数据是否存在竞争、线程是否安全的主要依据，依靠这个原则，我们解决在并发环境下两个操作之间是否存在冲突的所有问题。

## 2.规则



# J.U.C 之 AQS：AQS 简介

Java的内置锁一直是备受争议的，在JDK1.6之前`synchronized`这个重量级锁其性能都是较为低下，虽然在1.6后，进行大量的优化策略（[《【死磕 Java 并发】—– 深入分析 synchronized 的实现原理》](http://www.iocoder.cn/JUC/sike/synchronized)），但是与Lock相比，`synchronized`还是存在缺陷的：虽然`synchronized`提供了便捷性的隐式获取锁释放锁机制（基于JMM机制），但是它却缺少获取锁与释放锁的可操作性，可中断、超时获取锁，且它为独占式在高并发场景下性能大打折扣。

## 1.简介

AQS， AbstractQueuedSynchronizer，即队列同步器，它是构建锁或者其他同步组件的基础框架（如ReentrantLock、ReentrantReadWriteLock、Semaphore等）

## 2.优势

在基于AQS构建的同步器中，只能在一个时刻发生阻塞，从而降低上下文切换的开销，提高了吞吐量。同时在设计AQS时充分考虑了可伸缩性，因此，J.U.C中，所有基于AQS构建的同步器均可以获得这个优势

## 3.同步状态

AQS的主要使用方式是继承，子类通过继承同步器，并实现它的抽象方法来管理同步状态。AQS使用一个`int`类型的成员变量`state`来表示同步状态

* 当`state > 0 `时，表示已经获取了锁
* 当`state = 0`时，表示释放了锁。

它提供了三个方法，来对同步状态`state`进行操作，并且AQS可以确保对`state`的操作是安全的

* `#getState()`
* `#setState(int newState)`
* `#compareAndSetState(int expect, int update)`

## 4.同步队列

AQS通过内置的FIFO同步队列来完成资源获取线程的排队工作：

* 如果当前线程获取同步状态失败(锁)时，AQS则会将当前线程以及等待状态等信息构造成一个节点(Node)并将其加入同步队列，同时会阻塞当前线程
* 当同步状态释放时，则会把节点中的线程唤醒，使其再次尝试获取同步状态

## 5.主要内置方法

- `#getState()`
- `#setState(int newState)`
- `#compareAndSetState(int expect, int update)`
- 【可重写】`#tryAcquire(int arg)`
- 【可重写】`#tryAcquire(int arg)`
- 【可重写】`#tryAcquire(int arg)`
- 【可重写】`#tryAcquire(int arg)`
- 【可重写】`#tryAcquire(int arg)`
- `#acquire(int arg)`
- `#acquireInterruptibly(int arg)`
- `#tryAcquireNanos(int arg, long nanos)`
- `#acquireShared(int arg)`
- `#acquireSharedInterruptibly(int arg)`
- `#tryAcquireSharedNanos(int arg, long nanosTimeout)`
- `#release(int arg)`
- `#releaseShared(int arg)`

从上面的方法看下来，基本上可以分成3类

* **独占式**获取与释放同步状态
* **共享式**获取与释放同步状态
* 查询**同步队列**中的等待线程情况

# J.U.C 之 AQS：CLH 同步队列

AQS内部维护者一个FIFO队列，该队列就是CLH同步队列

## 1.简介

CLH同步队列是一个FIFO双向队列，AQS依赖它来完成同步状态的管理

* 当前线程如果获取同步状态失败时，AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程
* 当同步状态释放时，会把首节点唤醒（公平锁），使其再次尝试获取同步状态

# J.U.C 之并发工具类：CountDownLatch

Java四大并发工具，CountDownLatch与CyclicBarrier有点类似

# J.U.C 之并发工具类：Semaphore

## 1.简介

信号量Semaphore是控制访问多个共享资源的计数器，和CountDownLatch一样，其本质上是一个“**共享锁**”

#  J.U.C 之并发工具类：Exchanger

Exchanger是最简答的也是最复杂的，简单在于API非常简单，就一个人构造方法和两个exchange()方法，最复杂在于它的实现是最复杂的

在API是这么介绍的：可以再对元素进行配对和交换的线程同步点。每个线程将条目上的某个方法呈现个exchange方法，与伙伴线程进行匹配，并且在返回时接收其伙伴的对象。Exchanger可被视为SynchronousQueue的双向形式。Exchanger可能在应用程序（比如遗传算法和管道设计）中很有用

Exchanger，它允许在并发任务之间交换数据。具体说，Exchanger类允许在两个线程之间定义同步点。当两个线程都达到同步点时，它们交换数据结构，因此第一个现成的数据结构进入到第二个线程中，第二个线程的数据结构进入到第一个线程中。



# 总结

Java并发包括：Synchronized、volatile、jmm、AQS、CLH、Fork/Join框架、两种锁、四大并发工具、三大并发容器、七大阻塞队列、两种线程池、十三个原子操作类