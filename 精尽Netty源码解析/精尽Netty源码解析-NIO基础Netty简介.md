# 精尽 Netty 源码分析 —— 调试环境搭建

## 7. 为什么使用 Netty

如下内容，引用自我的基友闪电侠的分享

> FROM [《Netty 源码分析之服务端启动全解析》](https://www.jianshu.com/p/c5068caab217)
>
> netty 底层基于 jdk 的 NIO ，我们为什么不直接基于 jdk 的 nio 或者其他 nio 框架？下面是我总结出来的原因
>
> 1. 使用 jdk 自带的 nio 需要了解太多的概念，编程复杂
> 2. netty 底层 IO 模型随意切换，而这一切只需要做微小的改动
> 3. netty 自带的拆包解包，异常检测等机制让你从nio的繁重细节中脱离出来，让你只需要关心业务逻辑
> 4. netty 解决了 jdk 的很多包括空轮训在内的 bug
> 5. netty 底层对线程，selector 做了很多细小的优化，精心设计的 reactor 线程做到非常高效的并发处理
> 6. 自带各种协议栈让你处理任何一种通用协议都几乎不用亲自动手
> 7. netty 社区活跃，遇到问题随时邮件列表或者 issue
> 8. netty 已经历各大rpc框架，消息中间件，分布式通信中间件线上的广泛验证，健壮性无比强大

# 精尽 Netty 源码分析 —— NIO 基础（一）之简介

2018-02-01

## 1. 概述

Java NIO( New IO 或者 Non Blocking IO ) ，从 Java 1.4 版本开始引入的**非阻塞** IO ，用于替换**标准**( 有些文章也称为**传统**，或者 Blocking IO 。下文统称为 BIO ) Java IO API 的 IO API 。

> 老艿艿：在一些文章中，会将 Java NIO 描述成**异步** IO ，实际是不太正确的： Java NIO 是**同步** IO ，Java AIO ( 也称为 NIO 2 )是**异步** IO。具体原因，推荐阅读文章：
>
> - [《异步和非阻塞一样吗? (内容涉及 BIO, NIO, AIO, Netty)》](https://blog.csdn.net/matthew_zhang/article/details/71328697) 。
> - [《BIO与NIO、AIO的区别(这个容易理解)》](https://blog.csdn.net/skiof007/article/details/52873421)
>
> 总结来说，在 **Unix IO 模型**的语境下：
>
> - 同步和异步的区别：数据拷贝阶段是否需要完全由操作系统处理。
> - 阻塞和非阻塞操作：是针对发起 IO 请求操作后，是否有立刻返回一个标志信息而不让请求线程等待。
>
> 因此，Java NIO 是**同步**且非阻塞的 IO 。

## 2. 核心组件

Java NIO 由如下**三个**核心组件组成：

- Channel
- Buffer
- Selector

后续的每篇文章，我们会分享对应的一个组件。

## 3. NIO 和 BIO 的对比

NIO 和 BIO 的区别主要体现在三个方面：

| NIO                  | BIO              |
| -------------------- | ---------------- |
| 基于缓冲区( Buffer ) | 基于流( Stream ) |
| **非**阻塞 IO        | 阻塞 IO          |
| 选择器( Selector )   | 无               |

- 其中，选择器( Selector )是 NIO 能实现**非**阻塞的基础。

### 3.1 基于 Buffer 与基于 Stream

BIO 是面向字节流或者字符流的，而在 NIO 中，它摒弃了传统的 IO 流，而是引入 Channel 和 Buffer 的概念：从 Channel 中读取数据到 Buffer 中，或者将数据从 Buffer 中写到 Channel 中。

① 那么什么是**基于 Stream**呢？

在一般的 Java IO 操作中，我们以**流式**的方式，**顺序**的从一个 Stream 中读取一个或者多个字节，直至读取所有字节。因为它没有缓存区，所以我们就不能随意改变读取指针的位置。

② 那么什么是**基于 Buffer** 呢？

基于 Buffer 就显得有点不同了。我们在从 Channel 中读取数据到 Buffer 中，这样 Buffer 中就有了数据后，我们就可以对这些数据进行操作了。并且不同于一般的 Java IO 操作那样是**顺序**操作，NIO 中我们可以随意的读取任意位置的数据，这样大大增加了处理过程中的灵活性。

> 老艿艿：**写入**操作，也符合上述**读取**操作的情况。

### 3.2 阻塞与非阻塞 IO

Java IO 的各种流是**阻塞**的 IO 操作。这就意味着，当一个线程执行读或写 IO 操作时，该线程会被**阻塞**，直到有一些数据被读取，或者数据完全写入。

------

Java NIO 可以让我们**非阻塞**的使用 IO 操作。例如：

- 当一个线程执行从 Channel 执行读取 IO 操作时，当此时有数据，则读取数据并返回；当此时无数据，则直接返回**而不会阻塞当前线程**。
- 当一个线程执行向 Channel 执行写入 IO 操作时，**不需要阻塞等待它完全写入**，这个线程同时可以做别的事情。

也就是说，线程可以将非阻塞 IO 的空闲时间用于在其他 Channel 上执行 IO 操作。所以，一个单独的线程，可以管理多个 Channel 的读取和写入 IO 操作。

### 3.3 Selector

Java NIO 引入 Selector ( 选择器 )的概念，它是 Java NIO 得以实现非阻塞 IO 操作的**最最最关键**。

我们可以注册**多个** Channel 到**一个** Selector 中。而 Selector 内部的机制，就可以自动的为我们不断的执行查询( select )操作，判断这些注册的 Channel 是否有**已就绪的 IO 事件( 例如可读，可写，网络连接已完成 )**。
通过这样的机制，**一个**线程通过使用**一个** Selector ，就可以非常简单且高效的来管理**多个** Channel 了。

## 4. NIO 和 AIO 的对比

考虑到 Netty 4.1.X 版本实际并未基于 Java AIO 实现，所以我们就省略掉这块内容。那么，感兴趣的同学，可以自己 Google 下 Java NIO 和 Java AIO 的对比。

具体为什么 Netty 4.1.X 版本不支持 Java AIO 的原因，可参见 [《Netty（二）：Netty 为啥去掉支持 AIO ?》](https://juejin.im/entry/5a8ed33b6fb9a0634c26801c) 文章。

也因此，Netty 4.1.X 一般情况下，使用的是**同步非阻塞的 NIO 模型**。当然，如果真的有必要，也可以使用**同步阻塞的 BIO 模型**。

## 666. 彩蛋

参考文章如下：

- [《高并发 Java（8）：NIO 和 AIO》](http://www.importnew.com/21341.html)
- [《Java NIO 的前生今世 之一 简介》](https://segmentfault.com/a/1190000006824091)
- [《Java NIO 系列教程（十二） Java NIO 与 IO》](http://ifeve.com/java-nio-vs-io/)

# 精尽 Netty 源码分析 —— NIO 基础（二）之 Channel

2018-02-05

## 1. 概述

在 Java NIO 中，基本上所有的 IO 操作都是从 Channel 开始。数据可以从 Channel 读取到 Buffer 中，也可以从 Buffer 写到 Channel 中。如下图所示：

![Buffer <=> Channel](http://www.iocoder.cn/images/Netty/2018_02_05/01.png)

## 2. NIO Channel 对比 Java Stream

NIO Channel **类似** Java Stream ，但又有几点不同：

1. 对于**同一个** Channel ，我们可以从它读取数据，也可以向它写入数据。而对于**同一个** Stream ，通畅要么只能读，要么只能写，二选一( 有些文章也描述成“单向”，也是这个意思 )。
2. Channel 可以**非阻塞**的读写 IO 操作，而 Stream 只能**阻塞**的读写 IO 操作。
3. Channel **必须配合** Buffer 使用，总是先读取到一个 Buffer 中，又或者是向一个 Buffer 写入。也就是说，我们无法绕过 Buffer ，直接向 Channel 写入数据。

## 3. Channel 的实现

Channel 在 Java 中，作为一个**接口**，`java.nio.channels.Channel` ，定义了 IO 操作的**连接与关闭**。代码如下：

```
public interface Channel extends Closeable {

    /**
     * 判断此通道是否处于打开状态。 
     */
    public boolean isOpen();

    /**
     *关闭此通道。
     */
    public void close() throws IOException;

}
```

Channel 有非常多的实现类，最为重要的**四个** Channel 实现类如下：

- SocketChannel ：一个客户端用来**发起** TCP 的 Channel 。
- ServerSocketChannel ：一个服务端用来**监听**新进来的连接的 TCP 的 Channel 。对于每一个新进来的连接，都会创建一个对应的 SocketChannel 。
- DatagramChannel ：通过 UDP 读写数据。
- FileChannel ：从文件中，读写数据。

> 老艿艿：因为 [《Java NIO 系列教程》](http://ifeve.com/java-nio-all/) 对上述的 Channel 解释的非常不错，我就直接引用啦。
>
> 我们在使用 Netty 时，主要使用 TCP 协议，所以胖友可以只看 [「3.2 SocketChannel」](http://svip.iocoder.cn/Netty/nio-2-channel/#) 和 [「3.1 ServerSocketChannel」](http://svip.iocoder.cn/Netty/nio-2-channel/#) 。

### 3.1 ServerSocketChannel

[《Java NIO系列教程（九） ServerSocketChannel》](http://ifeve.com/server-socket-channel/)

### 3.2 SocketChannel

[《Java NIO 系列教程（八） SocketChannel》](http://ifeve.com/socket-channel/)

### 3.3 DatagramChannel

[《Java NIO系列教程（十） Java NIO DatagramChannel》](http://ifeve.com/datagram-channel/)

### 3.4 FileChannel

[《Java NIO系列教程（七） FileChannel》](http://ifeve.com/file-channel/)

## 666. 彩蛋

参考文章如下：

- [《Java NIO 系列教程（二） Channel》](http://ifeve.com/channels/)

# 精尽 Netty 源码分析 —— NIO 基础（三）之 Buffer

2018-02-10

## 1. 概述

一个 Buffer ，本质上是内存中的一块，我们可以将数据写入这块内存，之后从这块内存获取数据。通过将这块内存封装成 NIO Buffer 对象，并提供了一组常用的方法，方便我们对该块内存的读写。

Buffer 在 `java.nio` 包中实现，被定义成**抽象类**，从而实现一组常用的方法。整体类图如下：

![类图](http://www.iocoder.cn/images/Netty/2018_02_10/01.png)

- 我们可以将 Buffer 理解为**一个数组的封装**，例如 IntBuffer、CharBuffer、ByteBuffer 等分别对应 `int[]`、`char[]`、`byte[]` 等。
- MappedByteBuffer 用于实现内存映射文件，不是本文关注的重点。因此，感兴趣的胖友，可以自己 Google 了解，还是蛮有趣的。

## 2. 基本属性

Buffer 中有 **4** 个非常重要的属性：`capacity`、`limit`、`position`、`mark` 。代码如下：

```
public abstract class Buffer {

    // Invariants: mark <= position <= limit <= capacity
    private int mark = -1;
    private int position = 0;
    private int limit;
    private int capacity;

    // Used only by direct buffers
    // NOTE: hoisted here for speed in JNI GetDirectBufferAddress
    long address;

    Buffer(int mark, int pos, int lim, int cap) {       // package-private
        if (cap < 0)
            throw new IllegalArgumentException("Negative capacity: " + cap);
        this.capacity = cap;
        limit(lim);
        position(pos);
        if (mark >= 0) {
            if (mark > pos)
                throw new IllegalArgumentException("mark > position: ("
                                                   + mark + " > " + pos + ")");
            this.mark = mark;
        }
    }
    
    // ... 省略具体方法的代码
}
```

- `capacity` 属性，容量，Buffer 能容纳的数据元素的**最大值**。这一容量在 Buffer 创建时被赋值，并且**永远不能被修改**。

- Buffer 分成**写模式**和**读模式**两种情况。如下图所示：


![åæ¨¡å¼ v.s. è¯"æ¨¡å¼](http://www.iocoder.cn/images/Netty/2018_02_10/02.png)


  - 从图中，我们可以看到，两种模式下，`position` 和 `limit` 属性分别代表不同的含义。下面，我们来分别看看。

- `position`属性，位置，初始值为 0 。

  - **写**模式下，每往 Buffer 中写入一个值，`position` 就自动加 1 ，代表下一次的写入位置。
  - **读**模式下，每从 Buffer 中读取一个值，`position` 就自动加 1 ，代表下一次的读取位置。( *和写模式类似* )

- `limit`属性，上限。

  - **写**模式下，代表最大能写入的数据上限位置，这个时候 `limit` 等于 `capacity` 。
  - **读**模式下，在 Buffer 完成所有数据写入后，通过调用 `#flip()` 方法，切换到**读**模式。此时，`limit` 等于 Buffer 中实际的数据大小。因为 Buffer 不一定被写满，所以不能使用 `capacity` 作为实际的数据大小。

- `mark` 属性，标记，通过`#mark()`方法，记录当前`position`；通过`reset()`方法，恢复`position`为标记。

  - **写**模式下，标记上一次写位置。
  - **读**模式下，标记上一次读位置。

- 从代码注释上，我们可以看到，四个属性总是遵循如下大小关系：

  ```
  mark <= position <= limit <= capacity
  ```

------

写到此处，忍不住吐槽了下，Buffer 的读模式和写模式，我认为是有一点“**糟糕**”。相信大多数人在理解的时候，都会开始一脸懵逼的状态。相比较来说，Netty 的 ByteBuf 就**优雅**的非常多，基本属性设计如下：

```
0 <= readerIndex <= writerIndex <= capacity
```

- 通过 `readerIndex` 和 `writerIndex` 两个属性，避免出现读模式和写模式的切换。

## 3. 创建 Buffer

① 每个 Buffer 实现类，都提供了 `#allocate(int capacity)` 静态方法，帮助我们快速**实例化**一个 Buffer 对象。以 ByteBuffer 举例子，代码如下：

```
// ByteBuffer.java
public static ByteBuffer allocate(int capacity) {
    if (capacity < 0)
        throw new IllegalArgumentException();
    return new HeapByteBuffer(capacity, capacity);
}
```

- ByteBuffer 实际是个抽象类，返回的是它的**基于堆内( Non-Direct )内存**的实现类 HeapByteBuffer 的对象。

② 每个 Buffer 实现类，都提供了 `#wrap(array)` 静态方法，帮助我们将其对应的数组**包装**成一个 Buffer 对象。还是以 ByteBuffer 举例子，代码如下：

```
// ByteBuffer.java
public static ByteBuffer wrap(byte[] array, int offset, int length){
    try {
        return new HeapByteBuffer(array, offset, length);
    } catch (IllegalArgumentException x) {
        throw new IndexOutOfBoundsException();
    }
}

public static ByteBuffer wrap(byte[] array) {
    return wrap(array, 0, array.length);
}
```

- 和 `#allocate(int capacity)` 静态方法**一样**，返回的也是 HeapByteBuffer 的对象。

③ 每个 Buffer 实现类，都提供了 `#allocateDirect(int capacity)` 静态方法，帮助我们快速**实例化**一个 Buffer 对象。以 ByteBuffer 举例子，代码如下：

```
// ByteBuffer.java
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
```

- 和 `#allocate(int capacity)` 静态方法**不一样**，返回的是它的**基于堆外( Direct )内存**的实现类 DirectByteBuffer 的对象。

### 3.1 关于 Direct Buffer 和 Non-Direct Buffer 的区别

> FROM [《Java NIO 的前生今世 之三 NIO Buffer 详解》](https://segmentfault.com/a/1190000006824155)
>
> **Direct Buffer:**
>
> - 所分配的内存不在 JVM 堆上, 不受 GC 的管理.(但是 Direct Buffer 的 Java 对象是由 GC 管理的, 因此当发生 GC, 对象被回收时, Direct Buffer 也会被释放)
> - 因为 Direct Buffer 不在 JVM 堆上分配, 因此 Direct Buffer 对应用程序的内存占用的影响就不那么明显(实际上还是占用了这么多内存, 但是 JVM 不好统计到非 JVM 管理的内存.)
> - 申请和释放 Direct Buffer 的开销比较大. 因此正确的使用 Direct Buffer 的方式是在初始化时申请一个 Buffer, 然后不断复用此 buffer, 在程序结束后才释放此 buffer.
> - 使用 Direct Buffer 时, 当进行一些底层的系统 IO 操作时, 效率会比较高, 因为此时 JVM 不需要拷贝 buffer 中的内存到中间临时缓冲区中.
>
> **Non-Direct Buffer:**
>
> - 直接在 JVM 堆上进行内存的分配, 本质上是 byte[] 数组的封装.
> - 因为 Non-Direct Buffer 在 JVM 堆中, 因此当进行操作系统底层 IO 操作中时, 会将此 buffer 的内存复制到中间临时缓冲区中. 因此 Non-Direct Buffer 的效率就较低.

笔者之前研究 JVM 内存时，也整理过一个脑图，感兴趣的胖友可以下载：[传送门](http://static.iocoder.cn/Java%E5%86%85%E5%AD%98.xmind) 。

## 4. 向 Buffer 写入数据

每个 Buffer 实现类，都提供了 `#put(...)` 方法，向 Buffer 写入数据。以 ByteBuffer 举例子，代码如下：

```
// 写入 byte
public abstract ByteBuffer put(byte b); 
public abstract ByteBuffer put(int index, byte b);
// 写入 byte 数组
public final ByteBuffer put(byte[] src) { ... }
public ByteBuffer put(byte[] src, int offset, int length) {...}
// ... 省略，还有其他 put 方法
```

对于 Buffer 来说，有一个非常重要的操作就是，我们要讲来自 Channel 的数据写入到 Buffer 中。在系统层面上，这个操作我们称为**读操作**，因为数据是从外部( 文件或者网络等 )读取到内存中。示例如下：

```
int num = channel.read(buffer);
```

- 上述方法会返回从 Channel 中写入到 Buffer 的数据大小。对应方法的代码如下：

  ```
  public interface ReadableByteChannel extends Channel {
  
      public int read(ByteBuffer dst) throws IOException;
      
  }
  ```

> 注意，通常在说 NIO 的读操作的时候，我们说的是从 Channel 中读数据到 Buffer 中，对应的是对 Buffer 的写入操作，初学者需要理清楚这个。

## 5. 从 Buffer 读取数据

每个 Buffer 实现类，都提供了 `#get(...)` 方法，从 Buffer 读取数据。以 ByteBuffer 举例子，代码如下：

```
// 读取 byte
public abstract byte get();
public abstract byte get(int index);
// 读取 byte 数组
public ByteBuffer get(byte[] dst, int offset, int length) {...}
public ByteBuffer get(byte[] dst) {...}
// ... 省略，还有其他 get 方法
```

对于 Buffer 来说，还有一个非常重要的操作就是，我们要讲来向 Channel 的写入 Buffer 中的数据。在系统层面上，这个操作我们称为**写操作**，因为数据是从内存中写入到外部( 文件或者网络等 )。示例如下：

```
int num = channel.write(buffer);
```

- 上述方法会返回向 Channel 中写入 Buffer 的数据大小。对应方法的代码如下：

  ```
  public interface WritableByteChannel extends Channel {
  
      public int write(ByteBuffer src) throws IOException;
      
  }
  ```

## 6. rewind() v.s. flip() v.s. clear()

### 6.1 flip

如果要读取 Buffer 中的数据，需要切换模式，**从写模式切换到读模式**。对应的为 `#flip()` 方法，代码如下：

```
public final Buffer flip() {
    limit = position; // 设置读取上限
    position = 0; // 重置 position
    mark = -1; // 清空 mark
    return this;
}
```

使用示例，代码如下：

```
buf.put(magic);    // Prepend header
in.read(buf);      // Read data into rest of buffer
buf.flip();        // Flip buffer
channel.write(buf);    // Write header + data to channel
```

### 6.2 rewind

`#rewind()` 方法，可以**重置** `position` 的值为 0 。因此，我们可以重新**读取和写入** Buffer 了。

大多数情况下，该方法主要针对于**读模式**，所以可以翻译为“倒带”。也就是说，和我们当年的磁带倒回去是一个意思。代码如下：

```
public final Buffer rewind() {
    position = 0; // 重置 position
    mark = -1; // 清空 mark
    return this;
}
```

- 从代码上，和 `#flip()` 相比，非常类似，除了少了第一行的 `limit = position` 的代码块。

使用示例，代码如下：

```
channel.write(buf);    // Write remaining data
buf.rewind();      // Rewind buffer
buf.get(array);    // Copy data into array
```

### 6.3 clear

`#clear()` 方法，可以“**重置**” Buffer 的数据。因此，我们可以重新**读取和写入** Buffer 了。

大多数情况下，该方法主要针对于**写模式**。代码如下：

```
public final Buffer clear() {
    position = 0; // 重置 position
    limit = capacity; // 恢复 limit 为 capacity
    mark = -1; // 清空 mark
    return this;
}
```

- 从源码上，我们可以看出，Buffer 的数据实际并未清理掉，所以使用时需要注意。
- 读模式下，尽量不要调用 `#clear()` 方法，因为 `limit` 可能会被错误的赋值为 `capacity` 。相比来说，调用 `#rewind()`更合理，如果有重读的需求。

使用示例，代码如下：

```
buf.clear();     // Prepare buffer for reading
in.read(buf);    // Read data
```

## 7. mark() 搭配 reset()

### 7.1 mark

`#mark()` 方法，保存当前的 `position` 到 `mark` 中。代码如下：

```
public final Buffer mark() {
    mark = position;
    return this;
}
```

### 7.2 reset

`#reset()` 方法，恢复当前的 `postion` 为 `mark` 。代码如下：

```
public final Buffer reset() {
    int m = mark;
    if (m < 0)
        throw new InvalidMarkException();
    position = m;
    return this;
}
```

## 8. 其它方法

Buffer 中还有其它方法，比较简单，所以胖友自己研究噢。代码如下：

```
// ========== capacity ==========
public final int capacity() {
    return capacity;
}

// ========== position ==========
public final int position() {
    return position;
}

public final Buffer position(int newPosition) {
    if ((newPosition > limit) || (newPosition < 0))
        throw new IllegalArgumentException();
    position = newPosition;
    if (mark > position) mark = -1;
    return this;
}

// ========== limit ==========
public final int limit() {
    return limit;
}
    
public final Buffer limit(int newLimit) {
    if ((newLimit > capacity) || (newLimit < 0))
        throw new IllegalArgumentException();
    limit = newLimit;
    if (position > limit) position = limit;
    if (mark > limit) mark = -1;
    return this;
}

// ========== mark ==========
final int markValue() {                             // package-private
    return mark;
}

final void discardMark() {                          // package-private
    mark = -1;
}

// ========== 数组相关 ==========
public final int remaining() {
    return limit - position;
}

public final boolean hasRemaining() {
    return position < limit;
}

public abstract boolean hasArray();

public abstract Object array();

public abstract int arrayOffset();

public abstract boolean isDirect();

// ========== 下一个读 / 写 position ==========
final int nextGetIndex() {                          // package-private
    if (position >= limit)
        throw new BufferUnderflowException();
    return position++;
}

final int nextGetIndex(int nb) {                    // package-private
    if (limit - position < nb)
        throw new BufferUnderflowException();
    int p = position;
    position += nb;
    return p;
}

final int nextPutIndex() {                          // package-private
    if (position >= limit)
        throw new BufferOverflowException();
    return position++;
}

final int nextPutIndex(int nb) {                    // package-private
    if (limit - position < nb)
        throw new BufferOverflowException();
    int p = position;
    position += nb;
    return p;
}

final int checkIndex(int i) {                       // package-private
    if ((i < 0) || (i >= limit))
        throw new IndexOutOfBoundsException();
    return i;
}

final int checkIndex(int i, int nb) {               // package-private
    if ((i < 0) || (nb > limit - i))
        throw new IndexOutOfBoundsException();
    return i;
}

// ========== 其它方法 ==========
final void truncate() {                             // package-private
    mark = -1;
    position = 0;
    limit = 0;
    capacity = 0;
}

static void checkBounds(int off, int len, int size) { // package-private
    if ((off | len | (off + len) | (size - (off + len))) < 0)
        throw new IndexOutOfBoundsException();
}
```

## 666. 彩蛋

参考文章如下：

- [《Java NIO：Buffer、Channel 和 Selector》](http://www.importnew.com/28007.html)
- [《Java NIO系列教程（三） Buffer》](http://ifeve.com/buffers/)
- [《Java NIO 的前生今世 之三 NIO Buffer 详解》](https://segmentfault.com/a/1190000006824155)
- [《深入浅出NIO之Channel、Buffer》](https://www.jianshu.com/p/052035037297)
- [《NIO学习笔记——缓冲区（Buffer）详解》](https://blog.csdn.net/fuyuwei2015/article/details/73521681)

# 精尽 Netty 源码分析 —— NIO 基础（四）之 Selector

2018-02-15

## 1. 概述

Selector ， 一般称为**选择器**。它是 Java NIO 核心组件中的一个，用于轮询一个或多个 NIO Channel 的状态是否处于可读、可写。如此，一个线程就可以管理多个 Channel ，也就说可以管理多个网络连接。也因此，Selector 也被称为**多路复用器**。

那么 Selector 是如何轮询的呢？

- 首先，需要将 Channel 注册到 Selector 中，这样 Selector 才知道哪些 Channel 是它需要管理的。
- 之后，Selector 会不断地轮询注册在其上的 Channel 。如果某个 Channel 上面发生了读或者写事件，这个 Channel 就处于就绪状态，会被 Selector 轮询出来，然后通过 SelectionKey 可以获取就绪 Channel 的集合，进行后续的 I/O 操作。

下图是一个 Selector 管理三个 Channel 的示例：

![Selector <=> Channel](http://www.iocoder.cn/images/Netty/2018_02_15/01.png)

## 2. 优缺点

① **优点**

使用一个线程**能够**处理多个 Channel 的优点是，只需要更少的线程来处理 Channel 。事实上，可以使用一个线程处理所有的 Channel 。对于操作系统来说，线程之间上下文切换的开销很大，而且每个线程都要占用系统的一些资源( 例如 CPU、内存 )。因此，使用的线程越少越好。

② **缺点**

因为在一个线程中使用了多个 Channel ，因此会造成每个 Channel 处理效率的降低。

当然，Netty 在设计实现上，通过 n 个线程处理多个 Channel ，从而很好的解决了这样的缺点。其中，n 的指的是有限的线程数，默认情况下为 CPU * 2 。

## 3. Selector 类图

Selector 在 `java.nio` 包中，被定义成**抽象类**，整体实现类图如下：

![Selector 类图](http://www.iocoder.cn/images/Netty/2018_02_15/02.png)

- Selector 的实现不是本文的重点，感兴趣的胖友可以看看占小狼的 [《深入浅出NIO之Selector实现原理》](https://www.jianshu.com/p/0d497fe5484a) 。

## 3. 创建 Selector

通过 `#open()` 方法，我们可以创建一个 Selector 对象。代码如下：

```
Selector selector = Selector.open();
```

## 4. 注册 Chanel 到 Selector 中

为了让 Selector 能够管理 Channel ，我们需要将 Channel 注册到 Selector 中。代码如下：

```
channel.configureBlocking(false); // <1>
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```

- **注意**，如果一个 Channel 要注册到 Selector 中，那么该 Channel 必须是**非阻塞**，所以 `<1>` 处的 `channel.configureBlocking(false);` 代码块。也因此，FileChannel 是不能够注册到 Channel 中的，因为它是**阻塞**的。

- 在 `#register(Selector selector, int interestSet)` 方法的**第二个参数**，表示一个“interest 集合”，意思是通过 Selector 监听 Channel 时，对**哪些**( 可以是多个 )事件感兴趣。可以监听四种不同类型的事件：

  - Connect ：连接完成事件( TCP 连接 )，仅适用于客户端，对应 `SelectionKey.OP_CONNECT` 。
  - Accept ：接受新连接事件，仅适用于服务端，对应 `SelectionKey.OP_ACCEPT` 。
  - Read ：读事件，适用于两端，对应 `SelectionKey.OP_READ` ，表示 Buffer 可读。
  - Write ：写时间，适用于两端，对应 `SelectionKey.OP_WRITE` ，表示 Buffer 可写。

  Channel 触发了一个事件，意思是该事件已经就绪：

- 一个 Client Channel Channel 成功连接到另一个服务器，称为“连接就绪”。

- 一个 Server Socket Channel 准备好接收新进入的连接，称为“接收就绪”。

- 一个有数据可读的 Channel ，可以说是“读就绪”。

- 一个等待写数据的 Channel ，可以说是“写就绪”。

------

因为 Selector 可以对 Channel 的**多个**事件感兴趣，所以在我们想要注册 Channel 的多个事件到 Selector 中时，可以使用**或运算** `|` 来组合多个事件。示例代码如下：

```
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```

------

实际使用时，我们会有**改变** Selector 对 Channel 感兴趣的事件集合，可以通过再次调用 `#register(Selector selector, int interestSet)` 方法来进行变更。示例代码如下：

```
channel.register(selector, SelectionKey.OP_READ);
channel.register(selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);
```

- 初始时，Selector 仅对 Channel 的 `SelectionKey.OP_READ` 事件感兴趣。
- 修改后，Selector 仅对 Channel 的 `SelectionKey.OP_READ` 和 `SelectionKey.OP_WRITE)` 事件**都**感兴趣。

## 5. SelectionKey 类

上一小节, 当我们调用 Channel 的 `#register(...)` 方法，向 Selector 注册一个 Channel 后，会返回一个 SelectionKey 对象。那么 SelectionKey 是什么呢？SelectionKey 在 `java.nio.channels` 包下，被定义成一个**抽象类**，表示一个 Channel 和一个 Selector 的注册关系，包含如下内容：

- interest set ：感兴趣的事件集合。
- ready set ：就绪的事件集合。
- Channel
- Selector
- attachment ：*可选的*附加对象。

### 5.1 interest set

通过调用 `#interestOps()` 方法，返回感兴趣的事件集合。示例代码如下：

```
int interestSet = selectionKey.interestOps();

// 判断对哪些事件感兴趣
boolean isInterestedInAccept  = interestSet & SelectionKey.OP_ACCEPT != 0;
boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT != 0;
boolean isInterestedInRead    = interestSet & SelectionKey.OP_READ != 0;
boolean isInterestedInWrite   = interestSet & SelectionKey.OP_WRITE != 0;
```

- 其中每个事件 Key 在 SelectionKey 中枚举，通过位( bit ) 表示。代码如下：

  ```
  //  SelectionKey.java
  
  public static final int OP_READ = 1 << 0;
  public static final int OP_WRITE = 1 << 2;
  public static final int OP_CONNECT = 1 << 3;
  public static final int OP_ACCEPT = 1 << 4;
  ```

  - 所以，在上述示例的后半段的代码，可以通过与运算 `&` 来判断是否对指定事件感兴趣。

### 5.2 ready set

通过调用 `#readyOps()` 方法，返回就绪的事件集合。示例代码如下：

```
int readySet = selectionKey.readyOps();

// 判断哪些事件已就绪
selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();
```

- 相比 interest set 来说，ready set 已经内置了判断事件的方法。代码如下：

  ```
  // SelectionKey.java
  public final boolean isReadable() {
      return (readyOps() & OP_READ) != 0;
  }
  public final boolean isWritable() {
      return (readyOps() & OP_WRITE) != 0;
  }
  public final boolean isConnectable() {
      return (readyOps() & OP_CONNECT) != 0;
  }
  public final boolean isAcceptable() {
      return (readyOps() & OP_ACCEPT) != 0;
  }
  ```

### 5.3 attachment

通过调用 `#attach(Object ob)` 方法，可以向 SelectionKey 添加附加对象；通过调用 `#attachment()` 方法，可以获得 SelectionKey 获得附加对象。示例代码如下：

```
selectionKey.attach(theObject);
Object attachedObj = selectionKey.attachment();
```

又获得在注册时，直接添加附加对象。示例代码如下：

```
SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);
```

## 6. 通过 Selector 选择 Channel

在 Selector 中，提供三种类型的选择( select )方法，返回当前有感兴趣事件准备就绪的 Channel **数量**：

```
// Selector.java

// 阻塞到至少有一个 Channel 在你注册的事件上就绪了。
public abstract int select() throws IOException;

// 在 `#select()` 方法的基础上，增加超时机制。
public abstract int select(long timeout) throws IOException;

// 和 `#select()` 方法不同，立即返回数量，而不阻塞。
public abstract int selectNow() throws IOException;
```

- 有一点**非常需要注意**：select 方法返回的 `int` 值，表示有多少 Channel 已经就绪。亦即，**自上次调用 select 方法后有多少 Channel 变成就绪状态**。如果调用 select 方法，因为有一个 Channel 变成就绪状态则返回了 1 ；若再次调用 select 方法，如果另一个 Channel 就绪了，它会再次返回1。如果对第一个就绪的 Channel 没有做任何操作，现在就有两个就绪的 Channel ，**但在每次 select 方法调用之间，只有一个 Channel 就绪了，所以才返回 1**。

## 7. 获取可操作的 Channel

一旦调用了 select 方法，并且返回值表明有一个或更多个 Channel 就绪了，然后可以通过调用Selector 的 `#selectedKeys()` 方法，访问“已选择键集( selected key set )”中的**就绪** Channel 。示例代码所示：

```
Set selectedKeys = selector.selectedKeys();
```

- 注意，当有**新增就绪**的 Channel ，需要先调用 select 方法，才会添加到“已选择键集( selected key set )”中。否则，我们直接调用 `#selectedKeys()` 方法，是无法获得它们对应的 SelectionKey 们。

## 8. 唤醒 Selector 选择

某个线程调用 `#select()` 方法后，发生阻塞了，即使没有通道已经就绪，也有办法让其从 `#select()` 方法返回。

- 只要让其它线程在第一个线程调用 `select()` 方法的那个 Selector 对象上，调用该 Selector 的 `#wakeup()` 方法，进行唤醒该 Selector 即可。
- 那么，阻塞在 `#select()`方法上的线程，会立马返回。

> Selector 的 `#select(long timeout)` 方法，若未超时的情况下，也可以满足上述方式。

注意，如果有其它线程调用了 `#wakeup()` 方法，但当前没有线程阻塞在 `#select()` 方法上，下个调用 `#select()`方法的线程会立即被唤醒。😈 有点神奇。

## 9. 关闭 Selector

当我们不再使用 Selector 时，可以调用 Selector 的 `#close()` 方法，将它进行关闭。

- Selector 相关的所有 SelectionKey 都**会失效**。
- Selector 相关的所有 Channel 并**不会关闭**。

注意，此时若有线程阻塞在 `#select()` 方法上，也会被唤醒返回。

## 10. 简单 Selector 示例

如下是一个简单的 Selector 示例，创建一个 Selector ，并将一个 Channel注册到这个 Selector上( Channel 的初始化过程略去 )，然后持续轮询这个 Selector 的四种事件( 接受，连接，读，写 )是否就绪。代码如下：

> 老艿艿：本代码取自 [《Java NIO系列教程（六） Selector》](http://ifeve.com/selectors/) 提供的示例，实际生产环境下并非这样的代码。🙂 最佳的实践，我们将在 Netty 中看到。

```
// 创建 Selector
Selector selector = Selector.open();
// 注册 Channel 到 Selector 中
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
while (true) {
      // 通过 Selector 选择 Channel 
	int readyChannels = selector.select();
	if (readyChannels == 0) {
	   continue;
	}
	// 获得可操作的 Channel
	Set selectedKeys = selector.selectedKeys();
	// 遍历 SelectionKey 数组
	Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
	while (keyIterator.hasNext()) {
		SelectionKey key = keyIterator.next();
		if (key.isAcceptable()) {
			// a connection was accepted by a ServerSocketChannel.
		} else if (key.isConnectable()) {
			// a connection was established with a remote server.
		} else if (key.isReadable()) {
			// a channel is ready for reading
		} else if (key.isWritable()) {
			// a channel is ready for writing
		}
		// 移除
		keyIterator.remove(); // <1>
	}
}
```

- 注意

  , 在每次迭代时, 我们都调用

   

  ```
  keyIterator.remove()
  ```

   

  代码块，将这个 key 从迭代器中删除。

  - 因为 `#select()` 方法仅仅是简单地将就绪的 Channel 对应的 SelectionKey 放到 selected keys 集合中。
  - 因此，如果我们从 selected keys 集合中，获取到一个 key ，但是没有将它删除，那么下一次 `#select` 时, 这个 SelectionKey 还在 selectedKeys 中.

## 666. 彩蛋

参考文章如下：

- [《Java NIO系列教程（六） Selector》](http://ifeve.com/selectors/)
- [《Java NIO之Selector（选择器）》](https://www.cnblogs.com/snailclimb/p/9086334.html)
- [《Java NIO 的前生今世 之四 NIO Selector 详解》](https://segmentfault.com/a/1190000006824196)

# 精尽 Netty 源码分析 —— NIO 基础（五）之示例

2018-02-20

## 1. 概述

在前面的四篇文章，我们已经对 NIO 的概念已经有了一定的了解。当然，胖友也可能和我一样，已经被一堆概念烦死了。

那么本文，我们撸起袖子，就是干代码，不瞎比比了。

当然，下面更多的是提供一个 NIO 示例。真正生产级的 NIO 代码，建议胖友重新写，或者直接使用 Netty 。

代码仓库在 [example/yunai/nio](https://github.com/YunaiV/netty/tree/f7016330f1483021ef1c38e0923e1c8b7cef0d10/example/src/main/java/io/netty/example/yunai/nio) 目录下。一共 3 个类：

- NioServer ：NIO 服务端。
- NioClient ：NIO 客户端。
- CodecUtil ：消息编解码工具类。

## 2. 服务端

```
  1: public class NioServer {
  2: 
  3:     private ServerSocketChannel serverSocketChannel;
  4:     private Selector selector;
  5: 
  6:     public NioServer() throws IOException {
  7:         // 打开 Server Socket Channel
  8:         serverSocketChannel = ServerSocketChannel.open();
  9:         // 配置为非阻塞
 10:         serverSocketChannel.configureBlocking(false);
 11:         // 绑定 Server port
 12:         serverSocketChannel.socket().bind(new InetSocketAddress(8080));
 13:         // 创建 Selector
 14:         selector = Selector.open();
 15:         // 注册 Server Socket Channel 到 Selector
 16:         serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
 17:         System.out.println("Server 启动完成");
 18: 
 19:         handleKeys();
 20:     }
 21: 
 22:     private void handleKeys() throws IOException {
 23:         while (true) {
 24:             // 通过 Selector 选择 Channel
 25:             int selectNums = selector.select(30 * 1000L);
 26:             if (selectNums == 0) {
 27:                 continue;
 28:             }
 29:             System.out.println("选择 Channel 数量：" + selectNums);
 30: 
 31:             // 遍历可选择的 Channel 的 SelectionKey 集合
 32:             Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
 33:             while (iterator.hasNext()) {
 34:                 SelectionKey key = iterator.next();
 35:                 iterator.remove(); // 移除下面要处理的 SelectionKey
 36:                 if (!key.isValid()) { // 忽略无效的 SelectionKey
 37:                     continue;
 38:                 }
 39: 
 40:                 handleKey(key);
 41:             }
 42:         }
 43:     }
 44: 
 45:     private void handleKey(SelectionKey key) throws IOException {
 46:         // 接受连接就绪
 47:         if (key.isAcceptable()) {
 48:             handleAcceptableKey(key);
 49:         }
 50:         // 读就绪
 51:         if (key.isReadable()) {
 52:             handleReadableKey(key);
 53:         }
 54:         // 写就绪
 55:         if (key.isWritable()) {
 56:             handleWritableKey(key);
 57:         }
 58:     }
 59: 
 60:     private void handleAcceptableKey(SelectionKey key) throws IOException {
 61:         // 接受 Client Socket Channel
 62:         SocketChannel clientSocketChannel = ((ServerSocketChannel) key.channel()).accept();
 63:         // 配置为非阻塞
 64:         clientSocketChannel.configureBlocking(false);
 65:         // log
 66:         System.out.println("接受新的 Channel");
 67:         // 注册 Client Socket Channel 到 Selector
 68:         clientSocketChannel.register(selector, SelectionKey.OP_READ, new ArrayList<String>());
 69:     }
 70: 
 71:     private void handleReadableKey(SelectionKey key) throws IOException {
 72:         // Client Socket Channel
 73:         SocketChannel clientSocketChannel = (SocketChannel) key.channel();
 74:         // 读取数据
 75:         ByteBuffer readBuffer = CodecUtil.read(clientSocketChannel);
 76:         // 处理连接已经断开的情况
 77:         if (readBuffer == null) {
 78:             System.out.println("断开 Channel");
 79:             clientSocketChannel.register(selector, 0);
 80:             return;
 81:         }
 82:         // 打印数据
 83:         if (readBuffer.position() > 0) {
 84:             String content = CodecUtil.newString(readBuffer);
 85:             System.out.println("读取数据：" + content);
 86: 
 87:             // 添加到响应队列
 88:             List<String> responseQueue = (ArrayList<String>) key.attachment();
 89:             responseQueue.add("响应：" + content);
 90:             // 注册 Client Socket Channel 到 Selector
 91:             clientSocketChannel.register(selector, SelectionKey.OP_WRITE, key.attachment());
 92:         }
 93:     }
 94: 
 95:     @SuppressWarnings("Duplicates")
 96:     private void handleWritableKey(SelectionKey key) throws ClosedChannelException {
 97:         // Client Socket Channel
 98:         SocketChannel clientSocketChannel = (SocketChannel) key.channel();
 99: 
100:         // 遍历响应队列
101:         List<String> responseQueue = (ArrayList<String>) key.attachment();
102:         for (String content : responseQueue) {
103:             // 打印数据
104:             System.out.println("写入数据：" + content);
105:             // 返回
106:             CodecUtil.write(clientSocketChannel, content);
107:         }
108:         responseQueue.clear();
109: 
110:         // 注册 Client Socket Channel 到 Selector
111:         clientSocketChannel.register(selector, SelectionKey.OP_READ, responseQueue);
112:     }
113: 
114:     public static void main(String[] args) throws IOException {
115:         NioServer server = new NioServer();
116:     }
117: 
118: }
```

整块代码我们可以分成 3 部分：

- 构造方法：初始化 NIO 服务端。
- `#handleKeys()` 方法：基于 Selector 处理 IO 操作。
- `#main(String[] args)` 方法：创建 NIO 服务端。

下面，我们逐小节来分享。

### 2.1 构造方法

> 对应【第 3 至 20 行】的代码。

- `serverSocketChannel` 属性，服务端的 ServerSocketChannel ，在【第 7 至 12 行】的代码进行初始化，重点是此处启动了服务端，并监听指定端口( 此处为 8080 )。
- `selector` 属性，选择器，在【第 14 至 16 行】的代码进行初始化，重点是此处将 `serverSocketChannel` 到 `selector`中，并对 `SelectionKey.OP_ACCEPT` 事件感兴趣。这样子，在客户端连接服务端时，我们就可以处理该 IO 事件。
- 第 19 行：调用 `#handleKeys()` 方法，基于 Selector 处理 IO 事件。

### 2.2 handleKeys

> 对应【第 22 至 43 行】的代码。

- 第 23 行：死循环。本文的示例，不考虑服务端关闭的逻辑。

- 第 24 至 29 行：调用 `Selector#select(long timeout)` 方法，每 30 秒阻塞等待有就绪的 IO 事件。此处的 30 秒为笔者随意写的，实际也可以改成其他超时时间，或者 `Selector#select()` 方法。当不存在就绪的 IO 事件，直接 `continue` ，继续下一次阻塞等待。

- 第 32 行：调用`Selector#selectedKeys()`  方法，获得有就绪的 IO 事件( 也可以称为“选择的” ) Channel 对应的 SelectionKey 集合。

  - 第 33 行 至 35 行：遍历 `iterator` ，进行逐个 SelectionKey 处理。重点注意下，处理完需要进行移除，具体原因，在 [《精尽 Netty 源码分析 —— NIO 基础（四）之 Selector》「10. 简单 Selector 示例」](http://svip.iocoder.cn/Netty/nio-4-selector/#10-%E7%AE%80%E5%8D%95-Selector-%E7%A4%BA%E4%BE%8B) 有详细解析。
  - 第 36 至 38 行：在遍历的过程中，可能该 SelectionKey 已经**失效**，直接 `continue` ，不进行处理。
  - 第 40 行：调用 `#handleKey()` 方法，逐个 SelectionKey 处理。

#### 2.2.1 handleKey

> 对应【第 45 至 58 行】的代码。

- 通过调用 SelectionKey 的 `#isAcceptable()`、`#isReadable()`、`#isWritable()` 方法，**分别**判断 Channel 是**接受连接**就绪，还是**读**就绪，或是**写**就绪，并调用相应的 `#handleXXXX(SelectionKey key)` 方法，处理对应的 IO 事件。
- 因为 SelectionKey 可以**同时**对**一个** Channel 的**多个**事件感兴趣，所以此处的代码都是 `if` 判断，而不是 `if else` 判断。😈 虽然，考虑到让示例更简单，本文的并未编写同时对一个 Channel 的多个事件感兴趣，后续我们会在 Netty 的源码解析中看到。
- `SelectionKey.OP_CONNECT` 使用在**客户端**中，所以此处不需要做相应的判断和处理。

#### 2.2.2 handleAcceptableKey

> 对应【第 60 至 69 行】的代码。

- 第 62 行：调用 `ServerSocketChannel#accept()` 方法，获得连接的客户端的 SocketChannel 。

- 第 64 行：配置客户端的 SocketChannel 为非阻塞，否则无法使用 Selector 。

- 第 66 行：打印日志，方便调试。实际场景下，使用 Logger 而不要使用 `System.out` 进行输出。

- 第 68 行：注册客户端的 SocketChannel 到`selector`中，并对`SelectionKey.OP_READ`事件感兴趣。这样子，在客户端发送消息( 数据 )到服务端时，我们就可以处理该 IO 事件。

  - 为什么不对 `SelectionKey.OP_WRITE` 事件感兴趣呢？因为这个时候，服务端一般不会主动向客户端发送消息，所以不需要对 `SelectionKey.OP_WRITE` 事件感兴趣。
  - 细心的胖友会发现，`Channel#register(Selector selector, int ops, Object attachment)` 方法的第 3 个参数，我们注册了 SelectionKey 的 `attachment` 属性为 `new ArrayList<String>()` ，这又是为什么呢？结合下面的 `#handleReadableKey(Selection key)` 方法，我们一起解析。

#### 2.2.3 handleReadableKey

> 对应【第 71 至 93 行】的代码。

- 第 73 行：调用 `SelectionKey#channel()` 方法，获得该 SelectionKey 对应的 SocketChannel ，即客户端的 SocketChannel 。

- 第 75 行：调用 `CodecUtil#read(SocketChannel channel)` 方法，读取数据。具体代码如下：

  ```
  // CodecUtil.java
  
  public static ByteBuffer read(SocketChannel channel) {
      // 注意，不考虑拆包的处理
      ByteBuffer buffer = ByteBuffer.allocate(1024);
      try {
          int count = channel.read(buffer);
          if (count == -1) {
              return null;
          }
      } catch (IOException e) {
          throw new RuntimeException(e);
      }
      return buffer;
  }
  ```

  - 考虑到示例的简单性，数据的读取，就不考虑拆包的处理。不理解的胖友，可以自己 Google 下。
  - 调用 `SocketChannel#read(ByteBuffer)` 方法，读取 Channel 的缓冲区的数据到 ByteBuffer 中。若返回的结果( `count` ) 为 -1 ，意味着客户端连接已经断开，我们直接返回 `null` 。为什么是返回 `null` 呢？下面继续见分晓。

- 第 76 至 81 行：若读取数据返回的结果为 `null` 时，意味着客户端的连接已经断开，因此取消注册 `selector` 对该客户端的 SocketChannel 的感兴趣的 IO 事件。通过调用注册方法，并且第 2 个参数 `ops` 为 0 ，可以达到取消注册的效果。😈 感兴趣的胖友，可以将这行代码进行注释，测试下效果就很容易明白了。

- 第 83 行：通过调用 `ByteBuffer#position()` 大于 0 ，来判断**实际**读取到数据。

  - 第 84 至 85 行：调用 `CodecUtil#newString(ByteBuffer)` 方法，格式化为字符串，并进行打印。代码如下：

    ```
    // CodecUtil.java
    
    public static String newString(ByteBuffer buffer) {
        buffer.flip();
        byte[] bytes = new byte[buffer.remaining()];
        System.arraycopy(buffer.array(), buffer.position(), bytes, 0, buffer.remaining());
        try {
            return new String(bytes, "UTF-8");
        } catch (UnsupportedEncodingException e) {
            throw new RuntimeException(e);
        }
    }
    ```

    - 注意，需要调用 `ByteBuffer#flip()` 方法，将 ByteBuffer 从**写**模式切换到**读**模式。

  - 第 86 行：一般在此处，我们可以进行一些业务逻辑的处理，并返回处理的相应结果。例如，我们熟悉的 Request / Response 的处理。当然，考虑到性能，我们甚至可以将逻辑的处理，丢到逻辑线程池。

    - 😈 如果不理解，木有关系，在 [《精尽 Dubbo 源码分析 —— NIO 服务器（二）之 Transport 层》「8. Dispacher」](http://svip.iocoder.cn/Dubbo/remoting-api-transport/) 中，有详细解析。
    - 🙂 考虑到示例的简洁性，所以在【第 88 至 89 行】的代码中，我们直接返回（`"响应："` + 请求内容）给客户端。

  - 第 88 行：通过调用`SelectionKey#attachment()`方法，获得我们附加在 SelectionKey 的响应队列(`responseQueue`)。可能有胖友会问啦，为什么不调用`SocketChannel#write(ByteBuf)`方法，直接写数据给客户端呢？虽然大多数情况下，SocketChannel 都是可写的，但是如果写入比较频繁，超过 SocketChannel 的缓存区大小，就会导致数据“丢失”，并未写给客户端。

    - 所以，此处笔者在示例中，处理的方式为添加响应数据到 `responseQueue` 中，并在【第 91 行】的代码中，注册客户端的 SocketChannel 到 `selector` 中，并对 `SelectionKey.OP_WRITE` 事件感兴趣。这样子，在 SocketChannel **写就绪**时，在 `#handleWritableKey(SelectionKey key)` 方法中，统一处理写数据给客户端。
    - 当然，还是因为是示例，所以这样的实现方式不是最优。在 Netty 中，具体的实现方式是，先尝试调用 `SocketChannel#write(ByteBuf)` 方法，写数据给客户端。若写入失败( 方法返回结果为 0 )时，再进行类似笔者的上述实现方式。牛逼！Netty ！
    - 如果不太理解分享的原因，可以再阅读如下两篇文章：
      - [《深夜对话：NIO 中 SelectionKey.OP_WRITE 你了解多少》](https://mp.weixin.qq.com/s/V4tEH1j64FHFmB8bReNI7g)
      - [《Java.nio 中 socketChannle.write() 返回 0 的简易解决方案》](https://blog.csdn.net/a34140974/article/details/48464845)

  - 第 91 行：有一点需要注意，`Channel#register(Selector selector, int ops, Object attachment)` 方法的第 3 个参数，需要继续传入响应队列( `responseQueue` )，因为每次注册生成**新**的 SelectionKey 。若不传入，下面的 `#handleWritableKey(SelectionKey key)` 方法，会获得不到响应队列( `responseQueue` )。

#### 2.2.4 handleWritableKey

> 对应【第 96 至 112 行】的代码。

- 第 98 行：调用 `SelectionKey#channel()` 方法，获得该 SelectionKey 对应的 SocketChannel ，即客户端的 SocketChannel 。

- 第 101 行：通过调用 `SelectionKey#attachment()` 方法，获得我们**附加**在 SelectionKey 的响应队列( `responseQueue`)。

  - 第 102 行：遍历响应队列。

  - 第 106 行：调用 `CodeUtil#write(SocketChannel, content)` 方法，写入响应数据给客户端。代码如下：

    ```
     // CodecUtil.java
     
    public static void write(SocketChannel channel, String content) {
        // 写入 Buffer
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        try {
            buffer.put(content.getBytes("UTF-8"));
        } catch (UnsupportedEncodingException e) {
            throw new RuntimeException(e);
        }
        // 写入 Channel
        buffer.flip();
        try {
            // 注意，不考虑写入超过 Channel 缓存区上限。
            channel.write(buffer);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
    ```

    - 代码比较简单，**还是要注意**，需要调用 `ByteBuffer#flip()` 方法，将 ByteBuffer 从**写**模式切换到**读**模式。

- 第 111 行：**注意**，再结束写入后，需要**重新**注册客户端的 SocketChannel 到 `selector` 中，并对 `SelectionKey.OP_READ`事件感兴趣。为什么呢？其实还是我们在上文中提到的，大多数情况下，SocketChannel **都是写就绪的**，如果不取消掉注册掉对 `SelectionKey.OP_READ` 事件感兴趣，就会导致反复触发无用的写事件处理。😈 感兴趣的胖友，可以将这行代码进行注释，测试下效果就很容易明白了。

### 2.3 main

> 对应【第 114 至 116 行】

- 比较简单，就是创建一个 NioServer 对象。

撸到此处，我们可以直接通过 `telnet 127.0.0.1 8080` 的方式，连接服务端，进行读写数据的测试。

## 3. 客户端

客户端的实现代码，绝大数和服务端相同，所以我们分析的相对会简略一些。不然，自己都嫌弃自己太啰嗦了。

```
  1: public class NioClient {
  2: 
  3:     private SocketChannel clientSocketChannel;
  4:     private Selector selector;
  5:     private final List<String> responseQueue = new ArrayList<String>();
  6: 
  7:     private CountDownLatch connected = new CountDownLatch(1);
  8: 
  9:     public NioClient() throws IOException, InterruptedException {
 10:         // 打开 Client Socket Channel
 11:         clientSocketChannel = SocketChannel.open();
 12:         // 配置为非阻塞
 13:         clientSocketChannel.configureBlocking(false);
 14:         // 创建 Selector
 15:         selector = Selector.open();
 16:         // 注册 Server Socket Channel 到 Selector
 17:         clientSocketChannel.register(selector, SelectionKey.OP_CONNECT);
 18:         // 连接服务器
 19:         clientSocketChannel.connect(new InetSocketAddress(8080));
 20: 
 21:         new Thread(new Runnable() {
 22:             @Override
 23:             public void run() {
 24:                 try {
 25:                     handleKeys();
 26:                 } catch (IOException e) {
 27:                     e.printStackTrace();
 28:                 }
 29:             }
 30:         }).start();
 31: 
 32:         if (connected.getCount() != 0) {
 33:             connected.await();
 34:         }
 35:         System.out.println("Client 启动完成");
 36:     }
 37: 
 38:     @SuppressWarnings("Duplicates")
 39:     private void handleKeys() throws IOException {
 40:         while (true) {
 41:             // 通过 Selector 选择 Channel
 42:             int selectNums = selector.select(30 * 1000L);
 43:             if (selectNums == 0) {
 44:                 continue;
 45:             }
 46: 
 47:             // 遍历可选择的 Channel 的 SelectionKey 集合
 48:             Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
 49:             while (iterator.hasNext()) {
 50:                 SelectionKey key = iterator.next();
 51:                 iterator.remove(); // 移除下面要处理的 SelectionKey
 52:                 if (!key.isValid()) { // 忽略无效的 SelectionKey
 53:                     continue;
 54:                 }
 55: 
 56:                 handleKey(key);
 57:             }
 58:         }
 59:     }
 60: 
 61:     private synchronized void handleKey(SelectionKey key) throws IOException {
 62:         // 接受连接就绪
 63:         if (key.isConnectable()) {
 64:             handleConnectableKey(key);
 65:         }
 66:         // 读就绪
 67:         if (key.isReadable()) {
 68:             handleReadableKey(key);
 69:         }
 70:         // 写就绪
 71:         if (key.isWritable()) {
 72:             handleWritableKey(key);
 73:         }
 74:     }
 75: 
 76:     private void handleConnectableKey(SelectionKey key) throws IOException {
 77:         // 完成连接
 78:         if (!clientSocketChannel.isConnectionPending()) {
 79:             return;
 80:         }
 81:         clientSocketChannel.finishConnect();
 82:         // log
 83:         System.out.println("接受新的 Channel");
 84:         // 注册 Client Socket Channel 到 Selector
 85:         clientSocketChannel.register(selector, SelectionKey.OP_READ, responseQueue);
 86:         // 标记为已连接
 87:         connected.countDown();
 88:     }
 89: 
 90:     @SuppressWarnings("Duplicates")
 91:     private void handleReadableKey(SelectionKey key) throws ClosedChannelException {
 92:         // Client Socket Channel
 93:         SocketChannel clientSocketChannel = (SocketChannel) key.channel();
 94:         // 读取数据
 95:         ByteBuffer readBuffer = CodecUtil.read(clientSocketChannel);
 96:         // 打印数据
 97:         if (readBuffer.position() > 0) { // 写入模式下，
 98:             String content = CodecUtil.newString(readBuffer);
 99:             System.out.println("读取数据：" + content);
100:         }
101:     }
102: 
103:     @SuppressWarnings("Duplicates")
104:     private void handleWritableKey(SelectionKey key) throws ClosedChannelException {
105:         // Client Socket Channel
106:         SocketChannel clientSocketChannel = (SocketChannel) key.channel();
107: 
108:         // 遍历响应队列
109:         List<String> responseQueue = (ArrayList<String>) key.attachment();
110:         for (String content : responseQueue) {
111:             // 打印数据
112:             System.out.println("写入数据：" + content);
113:             // 返回
114:             CodecUtil.write(clientSocketChannel, content);
115:         }
116:         responseQueue.clear();
117: 
118:         // 注册 Client Socket Channel 到 Selector
119:         clientSocketChannel.register(selector, SelectionKey.OP_READ, responseQueue);
120:     }
121: 
122:     public synchronized void send(String content) throws ClosedChannelException {
123:         // 添加到响应队列
124:         responseQueue.add(content);
125:         // 打印数据
126:         System.out.println("写入数据：" + content);
127:         // 注册 Client Socket Channel 到 Selector
128:         clientSocketChannel.register(selector, SelectionKey.OP_WRITE, responseQueue);
129:         selector.wakeup();
130:     }
131: 
132:     public static void main(String[] args) throws IOException, InterruptedException {
133:         NioClient client = new NioClient();
134:         for (int i = 0; i < 30; i++) {
135:             client.send("nihao: " + i);
136:             Thread.sleep(1000L);
137:         }
138:     }
139: 
140: }
```

整块代码我们可以分成 3 部分：

- 构造方法：初始化 NIO 客户端。
- `#handleKeys()` 方法：基于 Selector 处理 IO 操作。
- `#main(String[] args)` 方法：创建 NIO 客户端，并向服务器发送请求数据。

下面，我们逐小节来分享。

### 3.1 构造方法

> 对应【第 3 至 36 行】的代码。

- `clientSocketChannel` 属性，客户端的 SocketChannel ，在【第 9 至 13 行】和【第 19 行】的代码进行初始化，重点是此处连接了指定服务端。

- `selector` 属性，选择器，在【第 14 至 17 行】的代码进行初始化，重点是此处将 `clientSocketChannel` 到 `selector`中，并对 `SelectionKey.OP_CONNECT` 事件感兴趣。这样子，在客户端连接服务端**成功**时，我们就可以处理该 IO 事件。

- `responseQueue` 属性，直接声明为 NioClient 的成员变量，是为了方便 `#send(String content)` 方法的实现。

- 第 21 至 30 行：调用 `#handleKeys()` 方法，基于 Selector 处理 IO 事件。比较特殊的是，我们是启动了一个**线程**进行处理。因为在后续的 `#main()` 方法中，我们需要调用发送请求数据的方法，不能直接在**主线程**，轮询处理 IO 事件。😈 机智的胖友，可能已经发现，NioServer 严格来说，也是应该这样处理。

- 第 32 至 34 行：通过 CountDownLatch 来实现阻塞等待客户端成功连接上服务端。具体的`CountDownLatch#countDown()`方法，在`#handleConnectableKey(SelectionKey key)`方法中调用。当然，除了可以使用 CountDownLatch 来实现阻塞等待，还可以通过如下方式:

  - Object 的 wait 和 notify 的方式。
  - Lock 的 await 和 notify 的方式。
  - Queue 的阻塞等待方式。
  - 😈 开心就好，皮一下很开心。

### 3.2 handleKeys

> 对应【第 38 至 59 行】的代码。

**完全**和 NioServer 中的该方法一模一样，省略。

#### 3.2.1 handleKey

> 对应【第 61 至 74 行】的代码。

**大体**逻辑和 NioServer 中的该方法一模一样，差别将对 `SelectionKey.OP_WRITE` 事件的处理改成对 `SelectionKey.OP_CONNECT` 事件的处理。

#### 3.3.2 handleConnectableKey

> 对应【第 76 至 88 行】的代码。

- 第 77 至 81 行：判断客户端的 SocketChannel 上是否**正在进行连接**的操作，若是，则完成连接。
- 第 83 行：打印日志。
- 第 85 行：注册客户端的 SocketChannel 到 `selector` 中，并对 `SelectionKey.OP_READ` 事件感兴趣。这样子，在客户端接收到到服务端的消息( 数据 )时，我们就可以处理该 IO 事件。
- 第 87 行：调用 `CountDownLatch#countDown()` 方法，结束 NioClient 构造方法中的【第 32 至 34 行】的阻塞等待连接完成。

#### 3.3.3 handleReadableKey

> 对应【第 91 至 101 行】的代码。

**大体**逻辑和 NioServer 中的该方法一模一样，**去掉响应请求的相关逻辑**。😈 如果不去掉，就是客户端和服务端互发消息的“死循环”了。

#### 3.3.4 handleWritableKey

> 对应【第 103 至 120 行】的代码。

**完全**和 NioServer 中的该方法一模一样。

### 3.3 send

> 对应【第 122 至 130 行】的代码。

客户端发送请求消息给服务端。

- 第 124 行：添加到响应队列( `responseQueue` ) 中。

- 第 126 行：打印日志。

- 第 128 行：注册客户端的 SocketChannel 到 `selector` 中，并对 `SelectionKey.OP_WRITE` 事件感兴趣。具体的原因，和 NioServer 的 `#handleReadableKey(SelectionKey key)` 方法的【第 88 行】一样。

- 第 129 行：调用`Selector#wakeup()`方法，唤醒`#handleKeys()`方法中，`Selector#select(long timeout)`方法的阻塞等待。

  - 因为，在 `Selector#select(long timeout)` 方法的实现中，是以调用**当时**，对 SocketChannel 的感兴趣的事件 。
  - 所以，在【第 128 行】的代码中，即使修改了对 SocketChannel 的感兴趣的事件，也不会结束 `Selector#select(long timeout)` 方法的阻塞等待。因此，需要进行唤醒操作。
  - 😈 感兴趣的胖友，可以将这行代码进行注释，测试下效果就很容易明白了。

### 3.4 main

> 对应【第 132 至 137 行】的代码。

- 第 133 行：创建一个 NioClient 对象。
- 第 134 至 137 行：每秒发送一次请求。考虑到代码没有处理拆包的逻辑，所以增加了间隔 1 秒的 sleep 。

## 666. 彩蛋

呼呼，凌晨 1 点。困累，写的有点着急了。简单 Review 了一遍，如果有不正确的，烦请斧正！谢谢！

推荐阅读文章如下：

- [《【NIO系列】—— Reactor 模式》](https://mp.weixin.qq.com/s/GpeaNowZKo1plaES9oxZ7g)
- [《lanux/java-demo/nio/example》](https://github.com/lanux/java-demo/tree/5b29c4b0d0056578a6eaa847e0d1efc9e42e48a4/src/main/java/com/lanux/io/nio)

# 精尽 Netty 源码分析 —— Netty 简介（一）之项目结构

2018-03-01

## 1. 概述

本文主要分享 **Netty 的项目结构**。
希望通过本文能让胖友对 Netty 的整体项目有个简单的了解。

在拉取 Netty 项目后，我们会发现拆分了**好多** Maven 项目。是不是内心一紧，产生了恐惧感？不要方，我们就是继续怼。

![项目结构](http://www.iocoder.cn/images/Netty/2018_03_01/01.png)

## 2. 代码统计

这里先分享一个小技巧。笔者在开始源码学习时，会首先了解项目的代码量。

**第一种方式**，使用 [IDEA Statistic](https://plugins.jetbrains.com/plugin/4509-statistic) 插件，统计整体代码量。

![Statistic 统计代码量](http://www.iocoder.cn/images/Netty/2018_03_01/02.png)

我们可以粗略的看到，总的代码量在 251365 行。这其中还包括单元测试，示例等等代码。
所以，不慌。

**第二种方式**，使用 [Shell 脚本命令逐个 Maven 模块统计](http://blog.csdn.net/yhhwatl/article/details/52623879) 。

一般情况下，笔者使用 `find . -name "*.java"|xargs cat|grep -v -e ^$ -e ^\s*\/\/.*$|wc -l` 。这个命令只过滤了**部分注释**，所以相比 [IDEA Statistic](https://plugins.jetbrains.com/plugin/4509-statistic) 会**偏多**。

当然，考虑到准确性，胖友需要手动 `cd` 到每个 Maven 项目的 `src/main/java` 目录下，以达到排除单元测试的代码量。

![Shell 脚本统计代码量](http://www.iocoder.cn/images/Netty/2018_03_01/03.png)

- 😈 偷懒了下，暂时只统计**核心**模块，未统计**拓展**模块。

## 3. 架构图

在看具体每个 Netty 的 Maven 项目之前，我们还是先来看看 Netty 的整体架构图。

![架构图](http://www.iocoder.cn/images/Netty/2018_03_01/04.png)

- Core

   

  ：核心部分，是底层的网络通用抽象和部分实现。

  - Extensible Event Model ：可拓展的事件模型。Netty 是基于事件模型的网络应用框架。
  - Universal Communication API ：通用的通信 API 层。Netty 定义了一套抽象的通用通信层的 API 。
  - Zero-Copy-Capable Rich Byte Buffer ：支持零拷贝特性的 Byte Buffer 实现。

- Transport Services

   

  ：传输( 通信 )服务，具体的网络传输的定义与实现。

  - Socket & Datagram ：TCP 和 UDP 的传输实现。
  - HTTP Tunnel ：HTTP 通道的传输实现。
  - In-VM Piple ：JVM 内部的传输实现。😈 理解起来有点怪，后续看具体代码，会易懂。

- **Protocol Support** ：协议支持。Netty 对于一些通用协议的编解码实现。例如：HTTP、Redis、DNS 等等。

## 4. 项目依赖图

Netty 的 Maven 项目之间**主要依赖**如下图：

![依赖图](http://www.iocoder.cn/images/Netty/2018_03_01/07.png)

- 本图省略**非主要依赖**。例如，`handler-proxy` 对 `codec` 有依赖，但是并未画出。
- 本图省略**非主要的项目**。例如，`resolver`、`testsuite`、`example` 等等。

下面，我们来详细介绍每个项目。

## 5. common

`common` 项目，该项目是一个通用的工具类项目，几乎被所有的其它项目依赖使用，它提供了一些数据类型处理工具类，并发编程以及多线程的扩展，计数器等等通用的工具类。

![common 项目](http://www.iocoder.cn/images/Netty/2018_03_01/05.png)

## 6. buffer

> 该项目实现了 Netty 架构图中的 Zero-Copy-Capable Rich Byte Buffer 。

`buffer` 项目，该项目下是 Netty 自行实现的一个 Byte Buffer 字节缓冲区。该包的实现相对于 JDK 自带的 ByteBuffer 有很多**优点**：无论是 API 的功能，使用体验，性能都要更加优秀。它提供了**一系列( 多种 )**的抽象定义以及实现，以满足不同场景下的需要。

![buffer 项目](http://www.iocoder.cn/images/Netty/2018_03_01/06.png)

## 7. transport

> 该项是核心项目，实现了 Netty 架构图中 Transport Services、Universal Communication API 和 Extensible Event Model 等多部分内容。

`transport` 项目，该项目是网络传输通道的抽象和实现。它定义通信的统一通信 API ，统一了 JDK 的 OIO、NIO ( 不包括 AIO )等多种编程接口。

![transport 项目](http://www.iocoder.cn/images/Netty/2018_03_01/08.png)

另外，它提供了多个子项目，实现不同的传输类型。例如：`transport-native-epoll`、`transport-native-kqueue`、`transport-rxtx`、`transport-udt` 和 `transport-sctp` 等等。

## 8. codec

> 该项目实现了Netty 架构图中的 Protocol Support 。

`codec` 项目，该项目是协议编解码的抽象与**部分**实现：JSON、Google Protocol、Base64、XML 等等。

![codec 项目](http://www.iocoder.cn/images/Netty/2018_03_01/09.png)

另外，它提供了多个子项目，实现不同协议的编解码。例如：`codec-dns`、`codec-haproxy`、`codec-http`、`codec-http2`、`codec-mqtt`、`codec-redis`、`codec-memcached`、`codec-smtp`、`codec-socks`、`codec-stomp`、`codec-xml` 等等。

## 9. handler

`handler` 项目，该项目是提供**内置的**连接通道处理器( ChannelHandler )实现类。例如：SSL 处理器、日志处理器等等。

![handler 项目](http://www.iocoder.cn/images/Netty/2018_03_01/10.png)

另外，它提供了一个子项目 `handler-proxy` ，实现对 HTTP、Socks 4、Socks 5 的代理转发。

## 10. example

`example` 项目，该项目是提供各种 Netty 使用示例，良心开源项目。

![example 项目](http://www.iocoder.cn/images/Netty/2018_03_01/11.png)

## 11. 其它项目

Netty 中还有其它项目，考虑到不是本系列的重点，就暂时进行省略。

- `all` ：All In One 的 `pom` 声明。

- `bom` ：Netty Bill Of Materials 的缩写，不了解的胖友，可以看看 [《Maven 与Spring BOM( Bill Of Materials )简化 Spring 版本控制》](https://blog.csdn.net/fanxiaobin577328725/article/details/66974896) 。

- `microbench` ：微基准测试。

- `resolver` ：终端( Endpoint ) 的地址解析器。

- `resolver-dns`

- `tarball` ：All In One 打包工具。

- `testsuite` ：测试集。

  > 测试集( TestSuite ) ：测试集是把多个相关测试归入一个组的表达方式。在 Junit 中，如果我们没有明确的定义一个测试集，那么 Juint 会自动的提供一个测试集。一个测试集一般将同一个包的测试类归入一组。

- `testsuite-autobahhn`

- `testsuite-http2`

- `testsuite-osgi`

## 666. 彩蛋

本文基于杨武兵大佬的 [《Netty 源码分析系列 —— 概述》](https://my.oschina.net/ywbrj042/blog/856596) 进行修改。

# 精尽 Netty 源码分析 —— Netty 简介（二）之核心组件

2018-03-01

## 1. 概述

什么是 Netty ？

> Netty 是一款提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发高性能、高可靠性的网络服务器和客户端程序。
>
> 也就是说，Netty 是一个基于 NIO 的客户、服务器端编程框架。使用 Netty 可以确保你快速和简单地开发出一个网络应用，例如实现了某种协议的客户，服务端应用。Netty 相当简化和流线化了网络应用的编程开发过程，例如，TCP 和 UDP 的 socket 服务开发。
>
> （以上摘自百度百科）。

Netty 具有如下特性( 摘自《Netty in Action》 )

| 分类     | Netty的特性                                                  |
| -------- | ------------------------------------------------------------ |
| 设计     | 1. 统一的 API ，支持多种传输类型( 阻塞和非阻塞的 )  2. 简单而强大的线程模型  3. 真正的无连接数据报套接字( UDP )支持  4. 连接逻辑组件( ChannelHander 中顺序处理消息 )以及组件复用( 一个 ChannelHandel 可以被多个ChannelPipeLine 复用 ) |
| 易于使用 | 1. 详实的 Javadoc 和大量的示例集  2. 不需要超过 JDK 1.6+ 的依赖 |
| 性能     | 拥有比 Java 的核心 API 更高的吞吐量以及更低的延迟( 得益于池化和复用 )，更低的资源消耗以及最少的内存复制 |
| 健壮性   | 1. 不会因为慢速、快速或者超载的连接而导致 OutOfMemoryError  2. 消除在高速网络中 NIO 应用程序常见的不公平读 / 写比率 |
| 安全性   | 完整的 SSL/TLS 以及 StartTLs 支持，可用于受限环境下，如 Applet 和 OSGI |
| 社区驱动 | 发布快速而且频繁                                             |

## 2. Netty核心组件

为了后期更好地理解和进一步深入 Netty，有必要总体认识一下 Netty 所用到的核心组件以及他们在整个 Netty 架构中是如何协调工作的。

Netty 有如下几个核心组件：

- Bootstrap & ServerBootstrap
- Channel
- ChannelFuture
- EventLoop & EventLoopGroup
- ChannelHandler
- ChannelPipeline

核心组件的高层类图如下：

![高层类图](http://www.iocoder.cn/images/Netty/2018_03_05/01.png)

### 2.1 Bootstrap & ServerBootstrap

这 2 个类都继承了AbstractBootstrap，因此它们有很多相同的方法和职责。它们都是**启动器**，能够帮助 Netty 使用者更加方便地组装和配置 Netty ，也可以更方便地启动 Netty 应用程序。相比使用者自己从头去将 Netty 的各部分组装起来要方便得多，降低了使用者的学习和使用成本。它们是我们使用 Netty 的入口和最重要的 API ，可以通过它来连接到一个主机和端口上，也可以通过它来绑定到一个本地的端口上。总的来说，**它们两者之间相同之处要大于不同**。

> 老艿艿：Bootstrap & ServerBootstrap 对于 Netty ，就相当于 Spring Boot 是 Spring 的启动器。

它们和其它组件之间的关系是它们将 Netty 的其它组件进行组装和配置，所以它们会组合和直接或间接依赖其它的类。

Bootstrap 用于启动一个 Netty TCP 客户端，或者 UDP 的一端。

- 通常使用 `#connet(...)` 方法连接到远程的主机和端口，作为一个 Netty TCP 客户端。
- 也可以通过 `#bind(...)` 方法绑定本地的一个端口，作为 UDP 的一端。
- 仅仅需要使用**一个** EventLoopGroup 。

ServerBootstrap 往往是用于启动一个 Netty 服务端。

- 通常使用 `#bind(...)` 方法绑定本地的端口上，然后等待客户端的连接。
- 使用**两个** EventLoopGroup 对象( 当然这个对象可以引用同一个对象 )：第一个用于处理它本地 Socket **连接**的 IO 事件处理，而第二个责负责处理远程客户端的 IO 事件处理。

### 2.2 Channel

Channel 是 Netty 网络操作抽象类，它除了包括基本的 I/O 操作，如 bind、connect、read、write 之外，还包括了 Netty 框架相关的一些功能，如获取该 Channel 的 EventLoop 。

在传统的网络编程中，作为核心类的 Socket ，它对程序员来说并不是那么友好，直接使用其成本还是稍微高了点。而 Netty 的 Channel 则提供的一系列的 API ，它大大降低了直接与 Socket 进行操作的复杂性。而相对于原生 NIO 的 Channel，Netty 的 Channel 具有如下优势( 摘自《Netty权威指南( 第二版 )》) ：

- 在 Channel 接口层，采用 Facade 模式进行统一封装，将网络 I/O 操作、网络 I/O 相关联的其他操作封装起来，统一对外提供。
- Channel 接口的定义尽量大而全，为 SocketChannel 和 ServerSocketChannel 提供统一的视图，由不同子类实现不同的功能，公共功能在抽象父类中实现，最大程度地实现功能和接口的重用。
- 具体实现采用聚合而非包含的方式，将相关的功能类聚合在 Channel 中，由 Channel 统一负责和调度，功能实现更加灵活。

### 2.2 EventLoop && EventLoopGroup

Netty 基于**事件驱动模型**，使用不同的事件来通知我们状态的改变或者操作状态的改变。它定义了在整个连接的生命周期里当有事件发生的时候处理的核心抽象。

Channel 为Netty 网络操作抽象类，EventLoop 负责处理注册到其上的 Channel 处理 I/O 操作，两者配合参与 I/O 操作。

EventLoopGroup 是一个 EventLoop 的分组，它可以获取到一个或者多个 EventLoop 对象，因此它提供了迭代出 EventLoop 对象的方法。

下图是 Channel、EventLoop、Thread、EventLoopGroup 之间的关系( 摘自《Netty In Action》) ：

![Channel、EventLoop、Thread、EventLoopGroup](http://www.iocoder.cn/images/Netty/2018_03_05/02.png)

- 一个 EventLoopGroup 包含一个或多个 EventLoop ，即 EventLoopGroup : EventLoop = `1 : n` 。
- 一个 EventLoop 在它的生命周期内，只能与一个 Thread 绑定，即 EventLoop : Thread = `1 : 1` 。
- 所有有 EventLoop 处理的 I/O 事件都将在它**专有**的 Thread 上被处理，从而保证线程安全，即 Thread : EventLoop = `1 : 1`。
- 一个 Channel 在它的生命周期内只能注册到一个 EventLoop 上，即 Channel : EventLoop = `n : 1` 。
- 一个 EventLoop 可被分配至一个或多个 Channel ，即 EventLoop : Channel = `1 : n` 。

当一个连接到达时，Netty 就会创建一个 Channel，然后从 EventLoopGroup 中分配一个 EventLoop 来给这个 Channel 绑定上，在该 Channel 的整个生命周期中都是有这个绑定的 EventLoop 来服务的。

### 2.4 ChannelFuture

Netty 为异步非阻塞，即所有的 I/O 操作都为异步的，因此，我们不能立刻得知消息是否已经被处理了。Netty 提供了 ChannelFuture 接口，通过该接口的 `#addListener(...)` 方法，注册一个 ChannelFutureListener，当操作执行成功或者失败时，监听就会自动触发返回结果。

### 2.5 ChannelHandler

ChannelHandler ，连接通道处理器，我们使用 Netty 中**最常用**的组件。ChannelHandler 主要用来处理各种事件，这里的事件很广泛，比如可以是连接、数据接收、异常、数据转换等。

ChannelHandler 有两个核心子类 ChannelInboundHandler 和 ChannelOutboundHandler，其中 ChannelInboundHandler 用于接收、处理入站( Inbound )的数据和事件，而 ChannelOutboundHandler 则相反，用于接收、处理出站( Outbound )的数据和事件。

- ChannelInboundHandler 的实现类还包括一系列的 **Decoder** 类，对输入字节流进行解码。
- ChannelOutboundHandler 的实现类还包括一系列的 **Encoder** 类，对输入字节流进行编码。

ChannelDuplexHandler 可以**同时**用于接收、处理入站和出站的数据和时间。

ChannelHandler 还有其它的一系列的抽象实现 Adapter ，以及一些用于编解码具体协议的 ChannelHandler 实现类。

### 2.6 ChannelPipeline

ChannelPipeline 为 ChannelHandler 的**链**，提供了一个容器并定义了用于沿着链传播入站和出站事件流的 API 。一个数据或者事件可能会被多个 Handler 处理，在这个过程中，数据或者事件经流 ChannelPipeline ，由 ChannelHandler 处理。在这个处理过程中，一个 ChannelHandler 接收数据后处理完成后交给下一个 ChannelHandler，或者什么都不做直接交给下一个 ChannelHandler。

![ChannelPipeline](http://www.iocoder.cn/images/Netty/2018_03_05/03.png)

- 当一个数据流进入 ChannelPipeline 时，它会从 ChannelPipeline 头部开始，传给第一个 ChannelInboundHandler 。当第一个处理完后再传给下一个，一直传递到管道的尾部。
- 与之相对应的是，当数据被写出时，它会从管道的尾部开始，先经过管道尾部的“最后”一个ChannelOutboundHandler ，当它处理完成后会传递给前一个 ChannelOutboundHandler 。

上图更详细的，可以是如下过程：

```
*                                                 I/O Request
*                                            via {@link Channel} or
*                                        {@link ChannelHandlerContext}
*                                                      |
*  +---------------------------------------------------+---------------+
*  |                           ChannelPipeline         |               |
*  |                                                  \|/              |
*  |    +---------------------+            +-----------+----------+    |
*  |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
*  |    +----------+----------+            +-----------+----------+    |
*  |              /|\                                  |               |
*  |               |                                  \|/              |
*  |    +----------+----------+            +-----------+----------+    |
*  |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
*  |    +----------+----------+            +-----------+----------+    |
*  |              /|\                                  .               |
*  |               .                                   .               |
*  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
*  |        [ method call]                       [method call]         |
*  |               .                                   .               |
*  |               .                                  \|/              |
*  |    +----------+----------+            +-----------+----------+    |
*  |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
*  |    +----------+----------+            +-----------+----------+    |
*  |              /|\                                  |               |
*  |               |                                  \|/              |
*  |    +----------+----------+            +-----------+----------+    |
*  |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
*  |    +----------+----------+            +-----------+----------+    |
*  |              /|\                                  |               |
*  +---------------+-----------------------------------+---------------+
*                  |                                  \|/
*  +---------------+-----------------------------------+---------------+
*  |               |                                   |               |
*  |       [ Socket.read() ]                    [ Socket.write() ]     |
*  |                                                                   |
*  |  Netty Internal I/O Threads (Transport Implementation)            |
*  +-------------------------------------------------------------------+
```

------

当 ChannelHandler 被添加到 ChannelPipeline 时，它将会被分配一个 **ChannelHandlerContext** ，它代表了 ChannelHandler 和 ChannelPipeline 之间的绑定。其中 ChannelHandler 添加到 ChannelPipeline 中，通过 ChannelInitializer 来实现，过程如下：

1. 一个 ChannelInitializer 的实现对象，被设置到了 BootStrap 或 ServerBootStrap 中。
2. 当 `ChannelInitializer#initChannel()` 方法被调用时，ChannelInitializer 将在 ChannelPipeline 中创建**一组**自定义的 ChannelHandler 对象。
3. ChannelInitializer 将它自己从 ChannelPipeline 中移除。

> ChannelInitializer 是一个特殊的 ChannelInboundHandlerAdapter 抽象类。

## 666. 彩蛋

本文整理于如下两篇文章：

- 小明哥 [《【死磕 Netty 】—— Netty的核心组件》](http://cmsblogs.com/?p=2467)
- 杨武兵 [《Netty 源码分析系列 —— 概述》](https://my.oschina.net/ywbrj042/blog/856596)
- 乒乓狂魔 [《Netty 源码分析（一）概览》](https://my.oschina.net/pingpangkuangmo/blog/734051)

😈 见谅，不擅长写理论型的内容哈。

# 精尽 Netty 源码分析 —— 启动（一）之服务端

2018-04-01

## 1. 概述

对于所有 Netty 的新手玩家，我们**最先**使用的就是 Netty 的 Bootstrap 和 ServerBootstrap 组这两个“**启动器**”组件。它们在 `transport` 模块的 `bootstrap` 包下实现，如下图所示：

![`bootstrap` 包](http://www.iocoder.cn/images/Netty/2018_04_01/01.png)

在图中，我们可以看到三个以 Bootstrap 结尾的类，类图如下：

![Bootstrap 类图](http://www.iocoder.cn/images/Netty/2018_04_01/02.png)

- 为什么是这样的类关系呢？因为 ServerBootstrap 和 Bootstrap **大部分**的方法和职责都是相同的。

本文仅分享 ServerBootstrap 启动 Netty 服务端的过程。下一篇文章，我们再分享 Bootstrap 分享 Netty 客户端。

## 2. ServerBootstrap 示例

下面，我们先来看一个 ServerBootstrap 的使用示例，就是我们在 [《精尽 Netty 源码分析 —— 调试环境搭建》](http://svip.iocoder.cn/Netty/build-debugging-environment/#5-1-EchoServer) 搭建的 EchoServer 示例。代码如下：

```
 1: public final class EchoServer {
 2: 
 3:     static final boolean SSL = System.getProperty("ssl") != null;
 4:     static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));
 5: 
 6:     public static void main(String[] args) throws Exception {
 7:         // Configure SSL.
 8:         // 配置 SSL
 9:         final SslContext sslCtx;
10:         if (SSL) {
11:             SelfSignedCertificate ssc = new SelfSignedCertificate();
12:             sslCtx = SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey()).build();
13:         } else {
14:             sslCtx = null;
15:         }
16: 
17:         // Configure the server.
18:         // 创建两个 EventLoopGroup 对象
19:         EventLoopGroup bossGroup = new NioEventLoopGroup(1); // 创建 boss 线程组 用于服务端接受客户端的连接
20:         EventLoopGroup workerGroup = new NioEventLoopGroup(); // 创建 worker 线程组 用于进行 SocketChannel 的数据读写
21:         // 创建 EchoServerHandler 对象
22:         final EchoServerHandler serverHandler = new EchoServerHandler();
23:         try {
24:             // 创建 ServerBootstrap 对象
25:             ServerBootstrap b = new ServerBootstrap();
26:             b.group(bossGroup, workerGroup) // 设置使用的 EventLoopGroup
27:              .channel(NioServerSocketChannel.class) // 设置要被实例化的为 NioServerSocketChannel 类
28:              .option(ChannelOption.SO_BACKLOG, 100) // 设置 NioServerSocketChannel 的可选项
29:              .handler(new LoggingHandler(LogLevel.INFO)) // 设置 NioServerSocketChannel 的处理器
30:              .childHandler(new ChannelInitializer<SocketChannel>() {
31:                  @Override
32:                  public void initChannel(SocketChannel ch) throws Exception { // 设置连入服务端的 Client 的 SocketChannel 的处理器
33:                      ChannelPipeline p = ch.pipeline();
34:                      if (sslCtx != null) {
35:                          p.addLast(sslCtx.newHandler(ch.alloc()));
36:                      }
37:                      //p.addLast(new LoggingHandler(LogLevel.INFO));
38:                      p.addLast(serverHandler);
39:                  }
40:              });
41: 
42:             // Start the server.
43:             // 绑定端口，并同步等待成功，即启动服务端
44:             ChannelFuture f = b.bind(PORT).sync();
45: 
46:             // Wait until the server socket is closed.
47:             // 监听服务端关闭，并阻塞等待
48:             f.channel().closeFuture().sync();
49:         } finally {
50:             // Shut down all event loops to terminate all threads.
51:             // 优雅关闭两个 EventLoopGroup 对象
52:             bossGroup.shutdownGracefully();
53:             workerGroup.shutdownGracefully();
54:         }
55:     }
56: }
```

- 第 7 至 15 行：配置 SSL ，暂时可以忽略。
- 第 17 至 20 行：创建两个 EventLoopGroup 对象。
  - **boss** 线程组：用于服务端接受客户端的**连接**。
  - **worker** 线程组：用于进行客户端的 SocketChannel 的**数据读写**。
  - 关于为什么是**两个** EventLoopGroup 对象，我们在后续的文章，进行分享。
- 第 22 行：创建 [`io.netty.example.echo.EchoServerHandler`](https://github.com/YunaiV/netty/blob/f7016330f1483021ef1c38e0923e1c8b7cef0d10/example/src/main/java/io/netty/example/echo/EchoServerHandler.java) 对象。
- 第 24 行：创建 ServerBootstrap 对象，用于设置服务端的启动配置。
  - 第 26 行：调用 `#group(EventLoopGroup parentGroup, EventLoopGroup childGroup)` 方法，设置使用的 EventLoopGroup 。
  - 第 27 行：调用 `#channel(Class<? extends C> channelClass)` 方法，设置要被实例化的 Channel 为 NioServerSocketChannel 类。在下文中，我们会看到该 Channel 内嵌了 `java.nio.channels.ServerSocketChannel` 对象。是不是很熟悉 😈 ？
  - 第 28 行：调用 `#option(ChannelOption<T> option, T value)` 方法，设置 NioServerSocketChannel 的可选项。在 [`io.netty.channel.ChannelOption`](https://github.com/YunaiV/netty/blob/f7016330f1483021ef1c38e0923e1c8b7cef0d10/transport/src/main/java/io/netty/channel/ChannelOption.java) 类中，枚举了相关的可选项。
  - 第 29 行：调用 `#handler(ChannelHandler handler)` 方法，设置 NioServerSocketChannel 的处理器。在本示例中，使用了 `io.netty.handler.logging.LoggingHandler` 类，用于打印服务端的每个事件。详细解析，见后续文章。
  - 第 30 至 40 行：调用 `#childHandler(ChannelHandler handler)` 方法，设置连入服务端的 Client 的 SocketChannel 的处理器。在本实例中，使用 ChannelInitializer 来初始化连入服务端的 Client 的 SocketChannel 的处理器。
- 第 44 行：**先**调用 `#bind(int port)` 方法，绑定端口，**后**调用 `ChannelFuture#sync()` 方法，阻塞等待成功。这个过程，就是“**启动服务端**”。
- 第 48 行：**先**调用 `#closeFuture()` 方法，**监听**服务器关闭，**后**调用 `ChannelFuture#sync()` 方法，阻塞等待成功。😈 注意，此处不是关闭服务器，而是“**监听**”关闭。
- 第 49 至 54 行：执行到此处，说明服务端已经关闭，所以调用 `EventLoopGroup#shutdownGracefully()` 方法，分别关闭两个 EventLoopGroup 对象。

> 老艿艿的自我吐槽：😈 貌似又啰嗦了一把？？？

## 3. AbstractBootstrap

我们再一起来看看 AbstractBootstrap 的代码实现。因为 ServerBootstrap 和 Bootstrap 都实现这个类，所以和 ServerBootstrap 相关度高的方法，我们会放在 [「4. ServerBootstrap」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 中分享，而和 Bootstrap 相关度高的方法，我们会放在下一篇 Bootstrap 的文章分享。

> 老艿艿：下面我会开始分享一系列的 AbstractBootstrap 的配置方法，如果比较熟悉的胖友，可以直接跳到 [「3.13 bind」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 开始看。

### 3.1 构造方法

```
public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> implements Cloneable {

    /**
     * EventLoopGroup 对象
     */
    volatile EventLoopGroup group;
    /**
     * Channel 工厂，用于创建 Channel 对象。
     */
    @SuppressWarnings("deprecation")
    private volatile ChannelFactory<? extends C> channelFactory;
    /**
     * 本地地址
     */
    private volatile SocketAddress localAddress;
    /**
     * 可选项集合
     */
    private final Map<ChannelOption<?>, Object> options = new LinkedHashMap<ChannelOption<?>, Object>();
    /**
     * 属性集合
     */
    private final Map<AttributeKey<?>, Object> attrs = new LinkedHashMap<AttributeKey<?>, Object>();
    /**
     * 处理器
     */
    private volatile ChannelHandler handler;

    AbstractBootstrap() {
        // Disallow extending from a different package.
    }

    AbstractBootstrap(AbstractBootstrap<B, C> bootstrap) {
        group = bootstrap.group;
        channelFactory = bootstrap.channelFactory;
        handler = bootstrap.handler;
        localAddress = bootstrap.localAddress;
        synchronized (bootstrap.options) { // <1>
            options.putAll(bootstrap.options);
        }
        synchronized (bootstrap.attrs) { // <2>
            attrs.putAll(bootstrap.attrs);
        }
    }
    
    // ... 省略无关代码
}
```

- AbstractBootstrap 是个

  抽象类

  ，并且实现 Cloneable 接口。另外，它声明了

   

  ```
  B
  ```

   

  、

  ```
  C
  ```

   

  两个泛型：

  - `B` ：继承 AbstractBootstrap 类，用于表示**自身**的类型。
  - `C` ：继承 Channel 类，表示表示**创建**的 Channel 类型。

- 每个属性比较简单，结合下面我们要分享的每个方法，就更易懂啦。

- 在 `<1>` 和 `<2>` 两处，比较神奇的使用了 `synchronized` 修饰符。老艿艿也是疑惑了一下，但是这并难不倒我。因为传入的 `bootstrap` 参数的 `options` 和 `attrs` 属性，可能在另外的线程被修改( 例如，我们下面会看到的 `#option(hannelOption<T> option, T value)` 方法)，通过 `synchronized` 来同步，解决此问题。

### 3.2 self

`#self()` 方法，返回自己。代码如下：

```
private B self() {
    return (B) this;
}
```

- 这里就使用到了 AbstractBootstrap 声明的 `B` 泛型。

### 3.3 group

`#group(EventLoopGroup group)` 方法，设置 EventLoopGroup 到 `group` 中。代码如下：

```
public B group(EventLoopGroup group) {
    if (group == null) {
        throw new NullPointerException("group");
    }
    if (this.group != null) { // 不允许重复设置
        throw new IllegalStateException("group set already");
    }
    this.group = group;
    return self();
}
```

- 最终调用 `#self()` 方法，返回自己。实际上，AbstractBootstrap 整个方法的调用，基本都是[“**链式调用**”](https://en.wikipedia.org/wiki/Method_chaining#Java)。

### 3.4 channel

`#channel(Class<? extends C> channelClass)` 方法，设置要被**实例化**的 Channel 的类。代码如下：

```
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}
```

- 虽然传入的 `channelClass` 参数，但是会使用 `io.netty.channel.ReflectiveChannelFactory` 进行封装。

- 调用 `#channelFactory(io.netty.channel.ChannelFactory<? extends C> channelFactory)` 方法，设置 `channelFactory` 属性。代码如下：

  ```
  public B channelFactory(io.netty.channel.ChannelFactory<? extends C> channelFactory) {
      return channelFactory((ChannelFactory<C>) channelFactory);
  }
  
  @Deprecated
  public B channelFactory(io.netty.bootstrap.ChannelFactory<? extends C> channelFactory) {
      if (channelFactory == null) {
          throw new NullPointerException("channelFactory");
      }
      if (this.channelFactory != null) { // 不允许重复设置
          throw new IllegalStateException("channelFactory set already");
      }
  
      this.channelFactory = channelFactory;
      return self();
  }
  ```

  - 我们可以看到有两个 `#channelFactory(...)` 方法，并且第**二**个是 `@Deprecated` 的方法。从 ChannelFactory 使用的**包名**，我们就可以很容易的判断，最初 ChannelFactory 在 `bootstrap` 中，后**重构**到 `channel` 包中。

#### 3.4.1 ChannelFactory

`io.netty.channel.ChannelFactory` ，Channel 工厂**接口**，用于创建 Channel 对象。代码如下：

```
public interface ChannelFactory<T extends Channel> extends io.netty.bootstrap.ChannelFactory<T> {

    /**
     * Creates a new channel.
     *
     * 创建 Channel 对象
     *
     */
    @Override
    T newChannel();

}
```

- `#newChannel()` 方法，用于创建 Channel 对象。

#### 3.4.2 ReflectiveChannelFactory

`io.netty.channel.ReflectiveChannelFactory` ，实现 ChannelFactory 接口，反射调用默认构造方法，创建 Channel 对象的工厂实现类。代码如下：

```
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    /**
     * Channel 对应的类
     */
    private final Class<? extends T> clazz;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        if (clazz == null) {
            throw new NullPointerException("clazz");
        }
        this.clazz = clazz;
    }

    @Override
    public T newChannel() {
        try {
            // 反射调用默认构造方法，创建 Channel 对象
            return clazz.getConstructor().newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }

}
```

- 重点看 `clazz.getConstructor().newInstance()` 代码块。

### 3.5 localAddress

`#localAddress(...)` 方法，设置创建 Channel 的本地地址。有四个**重载**的方法，代码如下：

```
public B localAddress(SocketAddress localAddress) {
    this.localAddress = localAddress;
    return self();
}

public B localAddress(int inetPort) {
    return localAddress(new InetSocketAddress(inetPort));
}

public B localAddress(String inetHost, int inetPort) {
    return localAddress(SocketUtils.socketAddress(inetHost, inetPort));
}

public B localAddress(InetAddress inetHost, int inetPort) {
    return localAddress(new InetSocketAddress(inetHost, inetPort));
}
```

- 一般情况下，不会调用该方法进行配置，而是调用 `#bind(...)` 方法，例如 [「2. ServerBootstrap 示例」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 。

### 3.6 option

`#option(ChannelOption<T> option, T value)` 方法，设置创建 Channel 的可选项。代码如下：

```
public <T> B option(ChannelOption<T> option, T value) {
    if (option == null) {
        throw new NullPointerException("option");
    }
    if (value == null) { // 空，意味着移除
        synchronized (options) {
            options.remove(option);
        }
    } else { // 非空，进行修改
        synchronized (options) {
            options.put(option, value);
        }
    }
    return self();
}
```

### 3.7 attr

`#attr(AttributeKey<T> key, T value)` 方法，设置创建 Channel 的属性。代码如下：

```
public <T> B attr(AttributeKey<T> key, T value) {
    if (key == null) {
        throw new NullPointerException("key");
    }
    if (value == null) { // 空，意味着移除
        synchronized (attrs) {
            attrs.remove(key);
        }
    } else { // 非空，进行修改
        synchronized (attrs) {
            attrs.put(key, value);
        }
    }
    return self();
}
```

- 怎么理解 `attrs` 属性呢？我们可以理解成 `java.nio.channels.SelectionKey` 的 `attachment` 属性，并且类型为 Map 。

### 3.8 handler

`#handler(ChannelHandler handler)` 方法，设置创建 Channel 的处理器。代码如下：

```
public B handler(ChannelHandler handler) {
    if (handler == null) {
        throw new NullPointerException("handler");
    }
    this.handler = handler;
    return self();
}
```

### 3.9 validate

`#validate()` 方法，校验配置是否正确。代码如下：

```
public B validate() {
    if (group == null) {
        throw new IllegalStateException("group not set");
    }
    if (channelFactory == null) {
        throw new IllegalStateException("channel or channelFactory not set");
    }
    return self();
}
```

- 在 `#bind(...)` 方法中，绑定本地地址时，会调用该方法进行校验。

### 3.10 clone

`#clone()` **抽象**方法，克隆一个 AbstractBootstrap 对象。代码如下。

```
/**
 * Returns a deep clone of this bootstrap which has the identical configuration.  This method is useful when making
 * multiple {@link Channel}s with similar settings.  Please note that this method does not clone the
 * {@link EventLoopGroup} deeply but shallowly, making the group a shared resource.
 */
@Override
public abstract B clone();
```

- 来自实现 Cloneable 接口，在子类中实现。这是

  深

  拷贝，即创建一个新对象，但不是所有的属性是

  深

  拷贝。可参见

   

  「3.1 构造方法」

  ：

  - **浅**拷贝属性：`group`、`channelFactory`、`handler`、`localAddress` 。
  - **深**拷贝属性：`options`、`attrs` 。

### 3.11 config

`#config()` 方法，返回当前 AbstractBootstrap 的配置对象。代码如下：

```
public abstract AbstractBootstrapConfig<B, C> config();
```

#### 3.11.1 AbstractBootstrapConfig

`io.netty.bootstrap.AbstractBootstrapConfig` ，BootstrapConfig **抽象类**。代码如下：

```
public abstract class AbstractBootstrapConfig<B extends AbstractBootstrap<B, C>, C extends Channel> {

    protected final B bootstrap;

    protected AbstractBootstrapConfig(B bootstrap) {
        this.bootstrap = ObjectUtil.checkNotNull(bootstrap, "bootstrap");
    }
    
    public final SocketAddress localAddress() {
        return bootstrap.localAddress();
    }
    
    public final ChannelFactory<? extends C> channelFactory() {
        return bootstrap.channelFactory();
    }

    public final ChannelHandler handler() {
        return bootstrap.handler();
    }

    public final Map<ChannelOption<?>, Object> options() {
        return bootstrap.options();
    }

    public final Map<AttributeKey<?>, Object> attrs() {
        return bootstrap.attrs();
    }

    public final EventLoopGroup group() {
        return bootstrap.group();
    }
    
}
```

- `bootstrap` 属性，对应的启动类对象。在每个方法中，我们可以看到，都是直接调用 `boostrap` 属性对应的方法，读取对应的配置。

------

AbstractBootstrapConfig 的整体类图如下：![AbstractBootstrapConfig 类图](http://www.iocoder.cn/images/Netty/2018_04_01/03.png)

- 每个 Config 类，对应一个 Bootstrap 类。
- ServerBootstrapConfig 和 BootstrapConfig 的实现代码，和 AbstractBootstrapConfig 基本一致，所以胖友自己去查看噢。

### 3.12 setChannelOptions

`#setChannelOptions(...)` **静态**方法，设置传入的 Channel 的**多个**可选项。代码如下：

```
static void setChannelOptions(
        Channel channel, Map<ChannelOption<?>, Object> options, InternalLogger logger) {
    for (Map.Entry<ChannelOption<?>, Object> e: options.entrySet()) {
        setChannelOption(channel, e.getKey(), e.getValue(), logger);
    }
}

static void setChannelOptions(
        Channel channel, Map.Entry<ChannelOption<?>, Object>[] options, InternalLogger logger) {
    for (Map.Entry<ChannelOption<?>, Object> e: options) {
        setChannelOption(channel, e.getKey(), e.getValue(), logger);
    }
}
```

- 在两个方法的内部，**都**调用 `#setChannelOption(Channel channel, ChannelOption<?> option, Object value, InternalLogger logger)` **静态**方法，设置传入的 Channel 的**一个**可选项。代码如下：

  ```
  private static void setChannelOption(
          Channel channel, ChannelOption<?> option, Object value, InternalLogger logger) {
      try {
          if (!channel.config().setOption((ChannelOption<Object>) option, value)) {
              logger.warn("Unknown channel option '{}' for channel '{}'", option, channel);
          }
      } catch (Throwable t) {
          logger.warn("Failed to set channel option '{}' with value '{}' for channel '{}'", option, value, channel, t);
      }
  }
  ```

  - 不同于 [「3.6 option」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 方法，它是设置要创建的 Channel 的可选项。而 `#setChannelOption(...)` 方法，它是设置已经创建的 Channel 的可选项。

### 3.13 bind

> 老艿艿：`#bind(...)` 方法，也可以启动 UDP 的一端，考虑到这个系列主要分享 Netty 在 NIO 相关的源码解析，所以如下所有的分享，都不考虑 UDP 的情况。

`#bind(...)` 方法，绑定端口，启动服务端。代码如下：

```
public ChannelFuture bind() {
    // 校验服务启动需要的必要参数
    validate();
    SocketAddress localAddress = this.localAddress;
    if (localAddress == null) {
        throw new IllegalStateException("localAddress not set");
    }
    // 绑定本地地址( 包括端口 )
    return doBind(localAddress);
}

public ChannelFuture bind(int inetPort) {
    return bind(new InetSocketAddress(inetPort));
}

public ChannelFuture bind(String inetHost, int inetPort) {
    return bind(SocketUtils.socketAddress(inetHost, inetPort));
}

public ChannelFuture bind(InetAddress inetHost, int inetPort) {
    return bind(new InetSocketAddress(inetHost, inetPort));
}

public ChannelFuture bind(SocketAddress localAddress) {
    // 校验服务启动需要的必要参数
    validate();
    if (localAddress == null) {
        throw new NullPointerException("localAddress");
    }
    // 绑定本地地址( 包括端口 )
    return doBind(localAddress);
}
```

- 该方法返回的是 ChannelFuture 对象，也就是**异步**的绑定端口，启动服务端。如果需要**同步**，则需要调用 `ChannelFuture#sync()` 方法。

`#bind(...)` 方法，核心流程如下图：

![核心流程](http://www.iocoder.cn/images/Netty/2018_04_01/04.png)

- 主要有 4 个步骤，下面我们来拆解代码，看看和我们在 [《精尽 Netty 源码分析 —— NIO 基础（五）之示例》](http://svip.iocoder.cn/Netty/nio-5-demo/?self) 的 NioServer 的代码，是**怎么对应**的。

#### 3.13.1 doBind

`#doBind(final SocketAddress localAddress)` 方法，代码如下：

```
 1: private ChannelFuture doBind(final SocketAddress localAddress) {
 2:     // 初始化并注册一个 Channel 对象，因为注册是异步的过程，所以返回一个 ChannelFuture 对象。
 3:     final ChannelFuture regFuture = initAndRegister();
 4:     final Channel channel = regFuture.channel();
 5:     if (regFuture.cause() != null) { // 若发生异常，直接进行返回。
 6:         return regFuture;
 7:     }
 8: 
 9:     // 绑定 Channel 的端口，并注册 Channel 到 SelectionKey 中。
10:     if (regFuture.isDone()) { // 未
11:         // At this point we know that the registration was complete and successful.
12:         ChannelPromise promise = channel.newPromise();
13:         doBind0(regFuture, channel, localAddress, promise); // 绑定
14:         return promise;
15:     } else {
16:         // Registration future is almost always fulfilled already, but just in case it's not.
17:         final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
18:         regFuture.addListener(new ChannelFutureListener() {
19:             @Override
20:             public void operationComplete(ChannelFuture future) throws Exception {
22:                 Throwable cause = future.cause();
23:                 if (cause != null) {
24:                     // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
25:                     // IllegalStateException once we try to access the EventLoop of the Channel.
26:                     promise.setFailure(cause);
27:                 } else {
28:                     // Registration was successful, so set the correct executor to use.
29:                     // See https://github.com/netty/netty/issues/2586
30:                     promise.registered();
31: 
32:                     doBind0(regFuture, channel, localAddress, promise); // 绑定
33:                 }
34:             }
35:         });
36:         return promise;
37:     }
38: }
```

- 第 3 行：调用

   

  ```
  #initAndRegister()
  ```

   

  方法，初始化并注册一个 Channel 对象。因为注册是

  异步

  的过程，所以返回一个 ChannelFuture 对象。详细解析，见

   

  「3.14 initAndRegister」

   

  。

  - 第 5 至 7 行：若发生异常，直接进行返回。

- 第 9 至 37 行：因为注册是

  异步

  的过程，有可能已完成，有可能未完成。所以实现代码分成了【第 10 至 14 行】和【第 15 至 36 行】分别处理已完成和未完成的情况。

  - **核心**在【第 13 行】或者【第 32 行】的代码，调用 `#doBind0(final ChannelFuture regFuture, final Channel channel, final SocketAddress localAddress, final ChannelPromise promise)` 方法，绑定 Channel 的端口，并注册 Channel 到 SelectionKey 中。
  - 如果**异步**注册对应的 ChanelFuture 未完成，则调用 `ChannelFuture#addListener(ChannelFutureListener)` 方法，添加监听器，在**注册**完成后，进行回调执行 `#doBind0(...)` 方法的逻辑。详细解析，见 见 [「3.13.2 doBind0」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 。
  - 所以总结来说，**bind 的逻辑，执行在 register 的逻辑之后**。
  - TODO 1001 Promise 2. PendingRegistrationPromise

#### 3.13.2 doBind0

> 老艿艿：此小节的内容，胖友先看完 [「3.14 initAndRegister」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 的内容在回过头来看。因为 `#doBind0(...)`方法的执行，在 `#initAndRegister()` 方法之后。

`#doBind0(...)` 方法，执行 Channel 的端口绑定逻辑。代码如下：

```
 1: private static void doBind0(
 2:         final ChannelFuture regFuture, final Channel channel,
 3:         final SocketAddress localAddress, final ChannelPromise promise) {
 4: 
 5:     // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
 6:     // the pipeline in its channelRegistered() implementation.
 7:     channel.eventLoop().execute(new Runnable() {
 8:         @Override
 9:         public void run() {
11:             // 注册成功，绑定端口
12:             if (regFuture.isSuccess()) {
13:                 channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
14:             // 注册失败，回调通知 promise 异常
15:             } else {
16:                 promise.setFailure(regFuture.cause());
17:             }
18:         }
19:     });
20: }
```

- 第 7 行：调用 EventLoop 执行 Channel 的端口绑定逻辑。但是，实际上当前线程已经是 EventLoop 所在的线程了，为何还要这样操作呢？答案在【第 5 至 6 行】的英语注释。感叹句，Netty 虽然代码量非常庞大且复杂，但是英文注释真的是非常齐全，包括 Github 的 issue 对代码提交的描述，也非常健全。

- 第 14 至 17 行：注册失败，回调通知 `promise` 异常。

- 第 11 至 13 行：注册成功，调用

   

  ```
  Channel#bind(SocketAddress localAddress, ChannelPromise promise)
  ```

   

  方法，执行 Channel 的端口绑定逻辑。后续的方法栈调用如下图：



  - 还是老样子，我们先省略掉 pipeline 的内部实现代码，从 `AbstractUnsafe#bind(final SocketAddress localAddress, final ChannelPromise promise)` 方法，继续向下分享。

`AbstractUnsafe#bind(final SocketAddress localAddress, final ChannelPromise promise)` 方法，Channel 的端口绑定逻辑。代码如下：

```
 1: @Override
 2: public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
 3:     // 判断是否在 EventLoop 的线程中。
 4:     assertEventLoop();
 5: 
 6:     if (!promise.setUncancellable() || !ensureOpen(promise)) {
 7:         return;
 8:     }
 9: 
10:     // See: https://github.com/netty/netty/issues/576
11:     if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
12:         localAddress instanceof InetSocketAddress &&
13:         !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
14:         !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
15:         // Warn a user about the fact that a non-root user can't receive a
16:         // broadcast packet on *nix if the socket is bound on non-wildcard address.
17:         logger.warn(
18:                 "A non-root user can't receive a broadcast packet if the socket " +
19:                 "is not bound to a wildcard address; binding to a non-wildcard " +
20:                 "address (" + localAddress + ") anyway as requested.");
21:     }
22: 
23:     // 记录 Channel 是否激活
24:     boolean wasActive = isActive();
25: 
26:     // 绑定 Channel 的端口
27:     try {
28:         doBind(localAddress);
29:     } catch (Throwable t) {
30:         safeSetFailure(promise, t);
31:         closeIfClosed();
32:         return;
33:     }
34: 
35:     // 若 Channel 是新激活的，触发通知 Channel 已激活的事件。
36:     if (!wasActive && isActive()) {
37:         invokeLater(new Runnable() {
38:             @Override
39:             public void run() {
40:                 pipeline.fireChannelActive();
41:             }
42:         });
43:     }
44: 
45:     // 回调通知 promise 执行成功
46:     safeSetSuccess(promise);
47: }
```

- 第 4 行：调用 `#assertEventLoop()` 方法，判断是否在 EventLoop 的线程中。即该方法，只允许在 EventLoop 的线程中执行。代码如下：

  ```
  // AbstractUnsafe.java
  private void assertEventLoop() {
      assert !registered || eventLoop.inEventLoop();
  }
  ```

- 第 6 至 8 行：和 `#register0(...)` 方法的【第 5 至 8 行】的代码，是一致的。

- 第 10 至 21 行：<https://github.com/netty/netty/issues/576>

- 第 24 行：调用 `#isActive()` 方法，获得 Channel 是否激活( active )。NioServerSocketChannel 对该方法的实现代码如下：

  ```
  // NioServerSocketChannel.java
  
  @Override
  public boolean isActive() {
      return javaChannel().socket().isBound();
  }
  ```

  - NioServerSocketChannel 的 `#isActive()` 的方法实现，判断 ServerSocketChannel 是否绑定端口。此时，一般返回的是 `false` 。

- 第 28 行：调用 `#doBind(SocketAddress localAddress)` 方法，绑定 Channel 的端口。代码如下：

  ```
  // NioServerSocketChannel.java
  
  @Override
  protected void doBind(SocketAddress localAddress) throws Exception {
      if (PlatformDependent.javaVersion() >= 7) {
          javaChannel().bind(localAddress, config.getBacklog());
      } else {
          javaChannel().socket().bind(localAddress, config.getBacklog());
      }
  }
  ```

  - 【重要】到了此处，服务端的 Java 原生 NIO ServerSocketChannel 终于绑定端口。😈

- 第 36 行：再次调用 `#isActive()` 方法，获得 Channel 是否激活。此时，一般返回的是 `true` 。因此，Channel 可以认为是**新激活**的，满足【第 36 至 43 行】代码的执行条件。

  - 第 37 行：调用 `#invokeLater(Runnable task)` 方法，提交任务，让【第 40 行】的代码执行，异步化。代码如下：

    ```
    // AbstractUnsafe.java
    private void invokeLater(Runnable task) {
        try {
            // This method is used by outbound operation implementations to trigger an inbound event later.
            // They do not trigger an inbound event immediately because an outbound operation might have been
            // triggered by another inbound event handler method.  If fired immediately, the call stack
            // will look like this for example:
            //
            //   handlerA.inboundBufferUpdated() - (1) an inbound handler method closes a connection.
            //   -> handlerA.ctx.close()
            //      -> channel.unsafe.close()
            //         -> handlerA.channelInactive() - (2) another inbound handler method called while in (1) yet
            //
            // which means the execution of two inbound handler methods of the same handler overlap undesirably.
            eventLoop().execute(task);
        } catch (RejectedExecutionException e) {
            logger.warn("Can't invoke task later as EventLoop rejected it", e);
        }
    }
    ```

    - 从实现代码可以看出，是通过提交一个新的任务到 EventLoop 的线程中。
    - 英文注释虽然有丢丢长，但是胖友耐心看完。有道在手，英文不愁啊。

  - 第 40 行：调用 `DefaultChannelPipeline#fireChannelActive()` 方法，触发 Channel 激活的事件。详细解析，见 [「3.13.3 beginRead」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 。

- 第 46 行：调用 `#safeSetSuccess(ChannelPromise)` 方法，回调通知 `promise` 执行成功。此处的通知，对应回调的是我们添加到 `#bind(...)` 方法返回的 ChannelFuture 的 ChannelFutureListener 的监听器。示例代码如下：

  ```
  ChannelFuture f = b.bind(PORT).addListener(new ChannelFutureListener() { // 回调的就是我！！！
      @Override
      public void operationComplete(ChannelFuture future) throws Exception {
          System.out.println("测试下被触发");
      }
  }).sync();
  ```

#### 3.13.3 beginRead

> 老艿艿：此小节的内容，胖友先看完 [「3.14 initAndRegister」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 的内容在回过头来看。因为 `#beginRead(...)`方法的执行，在 `#doBind0(...)` 方法之后。

在 `#bind(final SocketAddress localAddress, final ChannelPromise promise)` 方法的【第 40 行】代码，调用 `Channel#bind(SocketAddress localAddress, ChannelPromise promise)` 方法，触发 Channel 激活的事件。后续的方法栈调用如下图：![触发 Channel 激活的事件](http://www.iocoder.cn/images/Netty/2018_04_01/10.png)

```
* 还是老样子，我们先省略掉 pipeline 的内部实现代码，从 `AbstractUnsafe#beginRead()` 方法，继续向下分享。
```

`AbstractUnsafe#beginRead()` 方法，开始读取操作。代码如下：

```
@Override
public final void beginRead() {
    // 判断是否在 EventLoop 的线程中。
    assertEventLoop();

    // Channel 必须激活
    if (!isActive()) {
        return;
    }

    // 执行开始读取
    try {
        doBeginRead();
    } catch (final Exception e) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireExceptionCaught(e);
            }
        });
        close(voidPromise());
    }
}
```

- 调用 `Channel#doBeginRead()` 方法，执行开始读取。对于 NioServerSocketChannel 来说，该方法实现代码如下：

  ```
  // AbstractNioMessageChannel.java
  @Override
  protected void doBeginRead() throws Exception {
      if (inputShutdown) {
          return;
      }
      super.doBeginRead();
  }
  
  // AbstractNioChannel.java
  @Override
  protected void doBeginRead() throws Exception {
      // Channel.read() or ChannelHandlerContext.read() was called
      final SelectionKey selectionKey = this.selectionKey;
      if (!selectionKey.isValid()) {
          return;
      }
  
      readPending = true;
  
      final int interestOps = selectionKey.interestOps();
      if ((interestOps & readInterestOp) == 0) {
          selectionKey.interestOps(interestOps | readInterestOp);
      }
  }
  ```

  - 【重要】在最后几行，我们可以看到，调用 `SelectionKey#interestOps(ops)` 方法，将我们创建 NioServerSocketChannel 时，设置的 `readInterestOp = SelectionKey.OP_ACCEPT` 添加为感兴趣的事件。也就说，服务端可以开始处理客户端的连接事件。

### 3.14 initAndRegister

`#initAndRegister()` 方法，初始化并注册一个 Channel 对象，并返回一个 ChannelFuture 对象。代码如下：

```
  1: final ChannelFuture initAndRegister() {
 2:     Channel channel = null;
 3:     try {
 4:         // 创建 Channel 对象
 5:         channel = channelFactory.newChannel();
 6:         // 初始化 Channel 配置
 7:         init(channel);
 8:     } catch (Throwable t) {
 9:         if (channel != null) { // 已创建 Channel 对象
10:             // channel can be null if newChannel crashed (eg SocketException("too many open files"))
11:             channel.unsafe().closeForcibly(); // 强制关闭 Channel
12:             // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
13:             return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
14:         }
15:         // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
16:         return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
17:     }
18: 
19:     // 注册 Channel 到 EventLoopGroup 中
20:     ChannelFuture regFuture = config().group().register(channel);
21:     if (regFuture.cause() != null) {
22:         if (channel.isRegistered()) {
23:             channel.close();
24:         } else {
25:             channel.unsafe().closeForcibly(); // 强制关闭 Channel
26:         }
27:     }
28: 
29:     return regFuture;
30: }
```

- 第 5 行：调用

   

  ```
  ChannelFactory#newChannel()
  ```

   

  方法，创建 Channel 对象。在本文的示例中，会使用 ReflectiveChannelFactory 创建 NioServerSocketChannel 对象。详细解析，见

   

  「3.14.1 创建 Channel」

   

  。

  - 第 16 行：返回带异常的 DefaultChannelPromise 对象。因为创建 Channel 对象失败，所以需要创建一个 FailedChannel 对象，设置到 DefaultChannelPromise 中才可以返回。

- 第 7 行：调用

   

  ```
  #init(Channel)
  ```

   

  方法，初始化 Channel 配置。详细解析，见

   

  「3.14.1 创建 Channel」

   

  。

  - 第 9 至 14 行：返回带异常的 DefaultChannelPromise 对象。因为初始化 Channel 对象失败，所以需要调用 `#closeForcibly()` 方法，强制关闭 Channel 。

- 第 20 行：首先获得 EventLoopGroup 对象，后调用 `EventLoopGroup#register(Channel)` 方法，注册 Channel 到 EventLoopGroup 中。实际在方法内部，EventLoopGroup 会分配一个 EventLoop 对象，将 Channel 注册到其上。详细解析，见 [「3.14.3 注册 Channel 到 EventLoopGroup」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 。

  - 第 22 至 23 行：若发生异常，并且 Channel 已经注册成功，则调用 `#close()` 方法，正常关闭 Channel 。

  - 第 24 至 26 行：若发生异常，并且 Channel 并未注册成功，则调用 `#closeForcibly()` 方法，强制关闭 Channel 。也就是说，和【第 9 至 14 行】一致。为什么会有正常和强制关闭 Channel 两种不同的处理呢？我们来看下 `#close()` 和 `#closeForcibly()` 方法的注释：

    ```
    // Channel.java#Unsafe
    
    /**
     * Close the {@link Channel} of the {@link ChannelPromise} and notify the {@link ChannelPromise} once the
     * operation was complete.
     */
    void close(ChannelPromise promise);
    
    /**
     * Closes the {@link Channel} immediately without firing any events.  Probably only useful
     * when registration attempt failed.
     */
    void closeForcibly();
    ```

    - 调用的前提，在于 Channel 是否注册到 EventLoopGroup 成功。😈 因为注册失败，也不好触发相关的事件。

#### 3.14.1 创建 Channel 对象

考虑到本文的内容，我们以 NioServerSocketChannel 的创建过程作为示例。流程图如下：

![创建 NioServerSocketChannel 对象](http://www.iocoder.cn/images/Netty/2018_04_01/05.png)

- 我们可以看到，整个流程涉及到 NioServerSocketChannel 的父类们。类图如下： ![Channel 类图](http://www.iocoder.cn/images/Netty/2018_04_01/06.png)

- 可能有部分胖友对 Netty Channel 的定义不是很理解，如下是官方的英文注释：

  > A nexus to a network socket or a component which is capable of I/O operations such as read, write, connect, and bind

  - 简单点来说，我们可以把 Netty Channel 和 Java 原生 Socket 对应，而 Netty NIO Channel 和 Java 原生 NIO SocketChannel 对象。

下面，我们来看看整个 NioServerSocketChannel 的创建过程的代码实现。

##### 3.14.1.1 NIOSERVERSOCKETCHANNEL

```
private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();

private final ServerSocketChannelConfig config;

public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}

public NioServerSocketChannel(SelectorProvider provider) {
    this(newSocket(provider));
}
```

- `DEFAULT_SELECTOR_PROVIDER` **静态**属性，默认的 SelectorProvider 实现类。

- `config` 属性，Channel 对应的配置对象。每种 Channel 实现类，也会对应一个 ChannelConfig 实现类。例如，NioServerSocketChannel 类，对应 ServerSocketChannelConfig 配置类。

  > ChannelConfig 的官网英文描述： A set of configuration properties of a Channel.

- 在构造方法中，调用 `#newSocket(SelectorProvider provider)` 方法，创建 NIO 的 ServerSocketChannel 对象。代码如下：

  ```
  private static ServerSocketChannel newSocket(SelectorProvider provider) {
      try {
          return provider.openServerSocketChannel();
      } catch (IOException e) {
          throw new ChannelException("Failed to open a server socket.", e);
      }
  }
  ```

  - 😈 是不是很熟悉这样的代码，效果和 `ServerSocketChannel#open()` 方法创建 ServerSocketChannel 对象是一致。

- `#NioServerSocketChannel(ServerSocketChannel channel)` 构造方法，代码如下：

  ```
  public NioServerSocketChannel(ServerSocketChannel channel) {
      super(null, channel, SelectionKey.OP_ACCEPT);
      config = new NioServerSocketChannelConfig(this, javaChannel().socket());
  }
  ```

  - 调用父 AbstractNioMessageChannel 的构造方法。详细解析，见 [「3.14.1.2 AbstractNioMessageChannel」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 。注意传入的 SelectionKey 的值为 `OP_ACCEPT` 。
  - 初始化 `config` 属性，创建 NioServerSocketChannelConfig 对象。

##### 3.14.1.2 ABSTRACTNIOMESSAGECHANNEL

```
protected AbstractNioMessageChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent, ch, readInterestOp);
}
```

- 直接调用父 AbstractNioChannel 的构造方法。详细解析，见 [「3.14.1.3 AbstractNioChannel」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 。

##### 3.14.1.3 ABSTRACTNIOCHANNEL

```
private final SelectableChannel ch;
protected final int readInterestOp;

protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    try {
        ch.configureBlocking(false);
    } catch (IOException e) {
        try {
            ch.close();
        } catch (IOException e2) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to close a partially initialized socket.", e2);
            }
        }

        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}
```

- `ch` 属性，**Netty NIO Channel 对象，持有的 Java 原生 NIO 的 Channel 对象**。

- ```
  readInterestOp
  ```

   

  属性，感兴趣的读事件的操作位值。

  - 目前笔者看了 AbstractNioMessageChannel 是 `SelectionKey.OP_ACCEPT` ， 而 AbstractNioByteChannel 是 `SelectionKey.OP_READ` 。
  - 详细的用途，我们会在 [「3.13.3 beginRead」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 看到。

- 调用父 AbstractNioChannel 的构造方法。详细解析，见 [「3.14.1.4 AbstractChannel」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 。

- 调用

   

  ```
  SelectableChannel#configureBlocking(false)
  ```

   

  方法，设置 NIO Channel 为

  非阻塞

  。😈 这块代码是不是非常熟悉哟。

  - 若发生异常，关闭 NIO Channel ，并抛出异常。

##### 3.14.1.4 ABSTRACTCHANNEL

```
/**
 * 父 Channel 对象
 */
private final Channel parent;
/**
 * Channel 编号
 */
private final ChannelId id;
/**
 * Unsafe 对象
 */
private final Unsafe unsafe;
/**
 * DefaultChannelPipeline 对象
 */
private final DefaultChannelPipeline pipeline;

protected AbstractChannel(Channel parent) {
    this.parent = parent;
    // 创建 ChannelId 对象
    id = newId();
    // 创建 Unsafe 对象
    unsafe = newUnsafe();
    // 创建 DefaultChannelPipeline 对象
    pipeline = newChannelPipeline();
}
```

- `parent` 属性，父 Channel 对象。对于 NioServerSocketChannel 的 `parent` 为空。

- `id` 属性，Channel 编号对象。在构造方法中，通过调用 `#newId()` 方法，进行创建。本文就先不分享，感兴趣的胖友自己看。

- `unsafe` 属性，Unsafe 对象。在构造方法中，通过调用 `#newUnsafe()` 方法，进行创建。本文就先不分享，感兴趣的胖友自己看。

  - 这里的 Unsafe 并不是我们常说的 Java 自带的`sun.misc.Unsafe` ，而是 `io.netty.channel.Channel#Unsafe`。

    ```
    // Channel.java#Unsafe
    /**
    * <em>Unsafe</em> operations that should <em>never</em> be called from user-code. These methods
    * are only provided to implement the actual transport, and must be invoked from an I/O thread except for the
    * following methods:
    * <ul>
    *   <li>{@link #localAddress()}</li>
    *   <li>{@link #remoteAddress()}</li>
    *   <li>{@link #closeForcibly()}</li>
    *   <li>{@link #register(EventLoop, ChannelPromise)}</li>
    *   <li>{@link #deregister(ChannelPromise)}</li>
    *   <li>{@link #voidPromise()}</li>
    * </ul>
    */
    ```

    - 这就是为什么叫 Unsafe 的原因。按照上述官网类的英文注释，Unsafe 操作不允许被用户代码使用。这些函数是真正用于数据传输操作，必须被IO线程调用。
    - 实际上，Channel 真正的具体操作，通过调用对应的 Unsafe 实现。😈 下文，我们将会看到。

  - Unsafe 不是一个具体的类，而是一个定义在 Channel 接口中的接口。不同的 Channel 类对应不同的 Unsafe 实现类。整体类图如下：



    - 对于 NioServerSocketChannel ，Unsafe 的实现类为 NioMessageUnsafe 。

- `pipeline` 属性，DefaultChannelPipeline 对象。在构造方法中，通过调用 `#newChannelPipeline()` 方法，进行创建。本文就先不分享，感兴趣的胖友自己看。

  > ChannelPipeline 的英文注释：A list of ChannelHandlers which handles or intercepts inbound events and outbound operations of a Channel 。

##### 3.14.1.5 小结

看到此处，我们来对 [「3.1.4.1 创建 Channel 对象」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 作一个小结。

对于一个 Netty NIO Channel 对象，它会包含如下几个核心组件：

- ChannelId
- Unsafe
- Pipeline
  - ChannelHandler
- ChannelConfig
- **Java 原生 NIO Channel**

如果不太理解，可以撸起袖子，多调试几次。

#### 3.14.2 初始化 Channel 配置

`#init(Channel channel)` 方法，初始化 Channel 配置。它是个**抽象**方法，由子类 ServerBootstrap 或 Bootstrap 自己实现。代码如下：

```
abstract void init(Channel channel) throws Exception;
```

- ServerBootstrap 对该方法的实现，我们在 [「4. ServerBootstrap」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 中，详细解析。

#### 3.14.3 注册 Channel 到 EventLoopGroup

`EventLoopGroup#register(Channel channel)` 方法，注册 Channel 到 EventLoopGroup 中。整体流程如下：

![register 流程](http://www.iocoder.cn/images/Netty/2018_04_01/08.png)

##### 3.14.3.1 REGISTER

EventLoopGroup 和 EventLoop 不是本文的重点，所以省略 1 + 2 + 3 部分的代码，从第 4 步的 `AbstractUnsafe#register(EventLoop eventLoop, final ChannelPromise promise)` 方法开始，代码如下：

```
 1: @Override
 2: public final void register(EventLoop eventLoop, final ChannelPromise promise) {
 3:     // 校验传入的 eventLoop 非空
 4:     if (eventLoop == null) {
 5:         throw new NullPointerException("eventLoop");
 6:     }
 7:     // 校验未注册
 8:     if (isRegistered()) {
 9:         promise.setFailure(new IllegalStateException("registered to an event loop already"));
10:         return;
11:     }
12:     // 校验 Channel 和 eventLoop 匹配
13:     if (!isCompatible(eventLoop)) {
14:         promise.setFailure(new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
15:         return;
16:     }
17: 
18:     // 设置 Channel 的 eventLoop 属性
19:     AbstractChannel.this.eventLoop = eventLoop;
20: 
21:     // 在 EventLoop 中执行注册逻辑
22:     if (eventLoop.inEventLoop()) {
23:         register0(promise);
24:     } else {
25:         try {
26:             eventLoop.execute(new Runnable() {
27:                 @Override
28:                 public void run() {
31:                     register0(promise);
32:                 }
33:             });
34:         } catch (Throwable t) {
35:             logger.warn("Force-closing a channel whose registration task was not accepted by an event loop: {}", AbstractChannel.this, t);
36:             closeForcibly();
37:             closeFuture.setClosed();
38:             safeSetFailure(promise, t);
39:         }
40:     }
41: }
```

- 第 3 至 6 行：校验传入的 `eventLoop` 参数非空。

- 第 7 至 11 行：调用 `#isRegistered()` 方法，校验未注册。代码如下：

  ```
  // AbstractChannel.java
  
  /**
   * 是否注册
   */
  private volatile boolean registered;
  
  @Override
  public boolean isRegistered() {
      return registered;
  }
  ```

- 第 12 至 16 行：校验 Channel 和 `eventLoop` 类型是否匹配，因为它们都有多种实现类型。代码如下：

  ```
  @Override
  protected boolean isCompatible(EventLoop loop) {
      return loop instanceof NioEventLoop;
  }
  ```

  - 要求 `eventLoop` 的类型为 NioEventLoop 。

- 第 19 行：【重要】设置 Channel 的 `eventLoop` 属性。

- 第 21 至 40 行：在

   

  ```
  evnetLoop
  ```

   

  中，调用

   

  ```
  #register0()
  ```

   

  方法，执行注册的逻辑。详细解析，见

   

  「3.14.3.2 register0」

   

  。

  - 第 34 至 39 行：若调用

     

    ```
    EventLoop#execute(Runnable)
    ```

     

    方法发生异常，则进行处理：

    - 第 36 行：调用 `AbstractUnsafe#closeForcibly()` 方法，强制关闭 Channel 。
    - 第 37 行：调用 `CloseFuture#setClosed()` 方法，通知 `closeFuture` 已经关闭。详细解析，见 [《精尽 Netty 源码解析 —— Channel（七）之 close 操作》](http://svip.iocoder.cn/Netty/Channel-7-close/) 。
    - 第 38 行：调用 `AbstractUnsafe#safeSetFailure(ChannelPromise promise, Throwable cause)` 方法，回调通知 `promise` 发生该异常。

##### 3.14.3.2 REGISTER0

`#register0(ChannelPromise promise)` 方法，注册逻辑。代码如下：

```
 1: private void register0(ChannelPromise promise) {
 2:     try {
 3:         // check if the channel is still open as it could be closed in the mean time when the register
 4:         // call was outside of the eventLoop
 5:         if (!promise.setUncancellable() // TODO 1001 Promise
 6:                 || !ensureOpen(promise)) { // 确保 Channel 是打开的
 7:             return;
 8:         }
 9:         // 记录是否为首次注册
10:         boolean firstRegistration = neverRegistered;
11: 
12:         // 执行注册逻辑
13:         doRegister();
14: 
15:         // 标记首次注册为 false
16:         neverRegistered = false;
17:         // 标记 Channel 为已注册
18:         registered = true;
19: 
20:         // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
21:         // user may already fire events through the pipeline in the ChannelFutureListener.
22:         pipeline.invokeHandlerAddedIfNeeded();
23: 
24:         // 回调通知 `promise` 执行成功
25:         safeSetSuccess(promise);
26: 
27:         // 触发通知已注册事件
28:         pipeline.fireChannelRegistered();
29: 
30:         // TODO 芋艿
31:         // Only fire a channelActive if the channel has never been registered. This prevents firing
32:         // multiple channel actives if the channel is deregistered and re-registered.
33:         if (isActive()) {
34:             if (firstRegistration) {
35:                 pipeline.fireChannelActive();
36:             } else if (config().isAutoRead()) {
37:                 // This channel was registered before and autoRead() is set. This means we need to begin read
38:                 // again so that we process inbound data.
39:                 //
40:                 // See https://github.com/netty/netty/issues/4805
41:                 beginRead();
42:             }
43:         }
44:     } catch (Throwable t) {
45:         // Close the channel directly to avoid FD leak.
46:         closeForcibly();
47:         closeFuture.setClosed();
48:         safeSetFailure(promise, t);
49:     }
50: }
```

- 第 5 行：// TODO 1001 Promise

- 第 6 行：调用 `#ensureOpen(ChannelPromise)` 方法，确保 Channel 是打开的。代码如下：

  ```
  // AbstractUnsafe.java
  protected final boolean ensureOpen(ChannelPromise promise) {
      if (isOpen()) {
          return true;
      }
  
      // 若未打开，回调通知 promise 异常
      safeSetFailure(promise, ENSURE_OPEN_CLOSED_CHANNEL_EXCEPTION);
      return false;
  }
  
  // AbstractNioChannel.java
  @Override
  public boolean isOpen() {
      return ch.isOpen();
  }
  ```

- 第 10 行：记录是否**首次**注册。`neverRegistered` 变量声明在 AbstractUnsafe 中，代码如下：

  ```
  /**
   * 是否重未注册过，用于标记首次注册
   *
   * true if the channel has never been registered, false otherwise
   */
  private boolean neverRegistered = true;
  ```

- 第 13 行：调用 `#doRegister()` 方法，执行注册逻辑。代码如下：

  ```
  // NioUnsafe.java
    1: @Override
    2: protected void doRegister() throws Exception {
    3:     boolean selected = false;
    4:     for (;;) {
    5:         try {
    6:             selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
    7:             return;
    8:         } catch (CancelledKeyException e) {
    9:             // TODO TODO 1003 doRegister 异常
   10:             if (!selected) {
   11:                 // Force the Selector to select now as the "canceled" SelectionKey may still be
   12:                 // cached and not removed because no Select.select(..) operation was called yet.
   13:                 eventLoop().selectNow();
   14:                 selected = true;
   15:             } else {
   16:                 // We forced a select operation on the selector before but the SelectionKey is still cached
   17:                 // for whatever reason. JDK bug ?
   18:                 throw e;
   19:             }
   20:         }
   21:     }
   22: }
  ```

  - 第 6 行：调用 `#unwrappedSelector()` 方法，返回 Java 原生 NIO Selector 对象。代码如下：

    ```
    // NioEventLoop.java
    
    private Selector unwrappedSelector;
    
    Selector unwrappedSelector() {
        return unwrappedSelector;
    }
    ```

    - 每个 NioEventLoop 对象上，都**独有**一个 Selector 对象。

  - 第 6 行：调用 `#javaChannel()` 方法，获得 Java 原生 NIO 的 Channel 对象。

  - 第 6 行：【重要】调用 `SelectableChannel#register(Selector sel, int ops, Object att)` 方法，注册 Java 原生 NIO 的 Channel 对象到 Selector 对象上。相信胖友对这块的代码是非常熟悉的，但是为什么感兴趣的事件是为 **0** 呢？正常情况下，对于服务端来说，需要注册 `SelectionKey.OP_ACCEPT` 事件呢！这样做的**目的**是( 摘自《Netty权威指南（第二版）》 )：

    > 1. 注册方式是多态的，它既可以被 NIOServerSocketChannel 用来监听客户端的连接接入，也可以注册 SocketChannel 用来监听网络读或者写操作。
    >
    > 2. 通过
    >
    >     
    >
    >    ```
    >    SelectionKey#interestOps(int ops)
    >    ```
    >
    >     
    >
    >    方法可以方便地修改监听操作位。所以，此处注册需要获取 SelectionKey 并给 AbstractNIOChannel 的成员变量
    >
    >     
    >
    >    ```
    >    selectionKey
    >    ```
    >
    >     
    >
    >    赋值。
    >
    >    - 如果不理解，没关系，在下文中，我们会看到服务端对 `SelectionKey.OP_ACCEPT` 事件的关注。😈

  - 第 8 至 20 行：TODO 1003 doRegister 异常

- 第 16 行：标记首次注册为 `false` 。

- 第 18 行：标记 Channel 为已注册。`registered` 变量声明在 AbstractChannel 中，代码如下：

  ```
  /**
   * 是否注册
   */
  private volatile boolean registered;
  ```

- 第 22 行：调用 `DefaultChannelPipeline#invokeHandlerAddedIfNeeded()` 方法，触发 ChannelInitializer 执行，进行 Handler 初始化。也就是说，我们在 [「4.init」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 写的 ServerBootstrap 对 Channel 设置的 ChannelInitializer 将被执行，进行 Channel 的 Handler 的初始化。

  - 具体的 pipeline 的内部调用过程，我们在后续文章分享。

- 第 25 行：调用 `#safeSetSuccess(ChannelPromise promise)` 方法，回调通知 `promise` 执行。在 [「3.13.1 doBind」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#)小节，我们向 `regFuture` 注册的 ChannelFutureListener ，就会被**立即回调执行**。

  > 老艿艿：胖友在看完这小节的内容，可以调回 [「3.13.2 doBind0」](http://svip.iocoder.cn/Netty/bootstrap-1-server/#) 小节的内容继续看。

- 第 28 行：调用 `DefaultChannelPipeline#invokeHandlerAddedIfNeeded()` 方法，触发通知 Channel 已注册的事件。

  - 具体的 pipeline 的内部调用过程，我们在后续文章分享。
  - 笔者目前调试下来，没有涉及服务端启动流程的逻辑代码。

- 第 33 至 43 行：TODO 芋艿

- 第 44 至 49 行：发生异常，和 `#register(EventLoop eventLoop, final ChannelPromise promise)` 方法的处理异常的代码，是一致的。

## 4. ServerBootstrap

`io.netty.bootstrap.ServerBootstrap` ，实现 AbstractBootstrap 抽象类，用于 Server 的启动器实现类。

### 4.1 构造方法

```
/**
 * 启动类配置对象
 */
private final ServerBootstrapConfig config = new ServerBootstrapConfig(this);
/**
 * 子 Channel 的可选项集合
 */
private final Map<ChannelOption<?>, Object> childOptions = new LinkedHashMap<ChannelOption<?>, Object>();
/**
 * 子 Channel 的属性集合
 */
private final Map<AttributeKey<?>, Object> childAttrs = new LinkedHashMap<AttributeKey<?>, Object>();
/**
 * 子 Channel 的 EventLoopGroup 对象
 */
private volatile EventLoopGroup childGroup;
/**
 * 子 Channel 的处理器
 */
private volatile ChannelHandler childHandler;

public ServerBootstrap() { }

private ServerBootstrap(ServerBootstrap bootstrap) {
    super(bootstrap);
    childGroup = bootstrap.childGroup;
    childHandler = bootstrap.childHandler;
    synchronized (bootstrap.childOptions) {
        childOptions.putAll(bootstrap.childOptions);
    }
    synchronized (bootstrap.childAttrs) {
        childAttrs.putAll(bootstrap.childAttrs);
    }
}
```

- `config` 属性，ServerBootstrapConfig 对象，启动类配置对象。
- 在 Server 接受**一个** Client 的连接后，会创建**一个**对应的 Channel 对象。因此，我们看到 ServerBootstrap 的 `childOptions`、`childAttrs`、`childGroup`、`childHandler` 属性，都是这种 Channel 的可选项集合、属性集合、EventLoopGroup 对象、处理器。下面，我们会看到 ServerBootstrap 针对这些配置项的设置方法。

### 4.2 group

`#group(..)` 方法，设置 EventLoopGroup 到 `group`、`childGroup` 中。代码如下：

```
@Override
public ServerBootstrap group(EventLoopGroup group) {
    return group(group, group);
}

public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    super.group(parentGroup);
    if (childGroup == null) {
        throw new NullPointerException("childGroup");
    }
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
    this.childGroup = childGroup;
    return this;
}
```

- 当只传入一个 EventLoopGroup 对象时，即调用的是 `#group(EventLoopGroup group)` 时，`group` 和 `childGroup` 使用同一个。一般情况下，我们不使用这个方法。

### 4.3 childOption

`#childOption(ChannelOption<T> option, T value)` 方法，设置子 Channel 的可选项。代码如下：

```
public <T> ServerBootstrap childOption(ChannelOption<T> childOption, T value) {
    if (childOption == null) {
        throw new NullPointerException("childOption");
    }
    if (value == null) { // 空，意味着移除
        synchronized (childOptions) {
            childOptions.remove(childOption);
        }
    } else { // 非空，进行修改
        synchronized (childOptions) {
            childOptions.put(childOption, value);
        }
    }
    return this;
}
```

### 4.4 childAttr

`#childAttr(AttributeKey<T> key, T value)` 方法，设置子 Channel 的属性。代码如下：

```
public <T> ServerBootstrap childAttr(AttributeKey<T> childKey, T value) {
    if (childKey == null) {
        throw new NullPointerException("childKey");
    }
    if (value == null) { // 空，意味着移除
        childAttrs.remove(childKey);
    } else { // 非空，进行修改
        childAttrs.put(childKey, value);
    }
    return this;
}
```

### 4.5 childHandler

`#childHandler(ChannelHandler handler)` 方法，设置子 Channel 的处理器。代码如下：

```
public ServerBootstrap childHandler(ChannelHandler childHandler) {
    if (childHandler == null) {
        throw new NullPointerException("childHandler");
    }
    this.childHandler = childHandler;
    return this;
}
```

### 4.6 validate

`#validate()` 方法，校验配置是否正确。代码如下：

```
@Override
public ServerBootstrap validate() {
    super.validate();
    if (childHandler == null) {
        throw new IllegalStateException("childHandler not set");
    }
    if (childGroup == null) {
        logger.warn("childGroup is not set. Using parentGroup instead.");
        childGroup = config.group();
    }
    return this;
}
```

### 4.7 clone

`#clone()` 方法，克隆 ServerBootstrap 对象。代码如下：

```
@Override
public ServerBootstrap clone() {
    return new ServerBootstrap(this);
}
```

- 调用参数为 `bootstrap` 为 ServerBootstrap 构造方法，克隆一个 ServerBootstrap 对象。

### 4.8 init

`#init(Channel channel)` 方法，初始化 Channel 配置。代码如下：

```
1: @Override
2: void init(Channel channel) throws Exception {
3:     // 初始化 Channel 的可选项集合
4:     final Map<ChannelOption<?>, Object> options = options0();
5:     synchronized (options) {
6:         setChannelOptions(channel, options, logger);
7:     }
8: 
9:     // 初始化 Channel 的属性集合
10:     final Map<AttributeKey<?>, Object> attrs = attrs0();
11:     synchronized (attrs) {
12:         for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
13:             @SuppressWarnings("unchecked")
14:             AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
15:             channel.attr(key).set(e.getValue());
16:         }
17:     }
18: 
19:     ChannelPipeline p = channel.pipeline();
20: 
21:     // 记录当前的属性
22:     final EventLoopGroup currentChildGroup = childGroup;
23:     final ChannelHandler currentChildHandler = childHandler;
24:     final Entry<ChannelOption<?>, Object>[] currentChildOptions;
25:     final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
26:     synchronized (childOptions) {
27:         currentChildOptions = childOptions.entrySet().toArray(newOptionArray(0));
28:     }
29:     synchronized (childAttrs) {
30:         currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
31:     }
32: 
33:     // 添加 ChannelInitializer 对象到 pipeline 中，用于后续初始化 ChannelHandler 到 pipeline 中。
34:     p.addLast(new ChannelInitializer<Channel>() {
35:         @Override
36:         public void initChannel(final Channel ch) throws Exception {
38:             final ChannelPipeline pipeline = ch.pipeline();
39: 
40:             // 添加配置的 ChannelHandler 到 pipeline 中。
41:             ChannelHandler handler = config.handler();
42:             if (handler != null) {
43:                 pipeline.addLast(handler);
44:             }
45: 
46:             // 添加 ServerBootstrapAcceptor 到 pipeline 中。
47:             // 使用 EventLoop 执行的原因，参见 https://github.com/lightningMan/netty/commit/4638df20628a8987c8709f0f8e5f3679a914ce1a
48:             ch.eventLoop().execute(new Runnable() {
49:                 @Override
50:                 public void run() {
52:                     pipeline.addLast(new ServerBootstrapAcceptor(
53:                             ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
54:                 }
55:             });
56:         }
57:     });
58: }}
```

- 第 3 至 7 行：将启动器配置的可选项集合，调用 `#setChannelOptions(channel, options, logger)` 方法，设置到 Channel 的可选项集合中。

- 第 9 至 17 行：将启动器配置的属性集合，设置到 Channel 的属性集合中。

- 第 21 至 31 行：记录启动器配置的**子 Channel**的属性，用于【第 52 至 53 行】的代码，创建 ServerBootstrapAcceptor 对象。

- 第 34 至 57 行：创建 ChannelInitializer 对象，添加到 pipeline 中，用于后续初始化 ChannelHandler 到 pipeline 中。

  - 第 40 至 44 行：添加启动器配置的 ChannelHandler 到 pipeline 中。

  - 第 46 至 55 行：创建 ServerBootstrapAcceptor 对象，添加到 pipeline 中。为什么使用 EventLoop 执行**添加的过程**？如果启动器配置的处理器，并且 ServerBootstrapAcceptor 不使用 EventLoop 添加，则会导致 ServerBootstrapAcceptor 添加到配置的处理器之前。示例代码如下：

    ```
    ServerBootstrap b = new ServerBootstrap();
    b.handler(new ChannelInitializer<Channel>() {
    
        @Override
        protected void initChannel(Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new LoggingHandler(LogLevel.INFO));
                }
            });
        }
    
    });
    ```

    - Netty 官方的提交，可见 [github commit](https://github.com/lightningMan/netty/commit/4638df20628a8987c8709f0f8e5f3679a914ce1a) 。

  - ServerBootstrapAcceptor 也是一个 ChannelHandler 实现类，用于接受客户端的连接请求。详细解析，见后续文章。

  - 该 ChannelInitializer 的初始化的执行，在 `AbstractChannel#register0(ChannelPromise promise)` 方法中触发执行。

  - 那么为什么要使用 ChannelInitializer 进行处理器的初始化呢？而不直接添加到 pipeline 中。例如修改为如下代码：

    ```
    final Channel ch = channel;
    final ChannelPipeline pipeline = ch.pipeline();
    
    // 添加配置的 ChannelHandler 到 pipeline 中。
    ChannelHandler handler = config.handler();
    if (handler != null) {
        pipeline.addLast(handler);
    }
    
    // 添加 ServerBootstrapAcceptor 到 pipeline 中。
    // 使用 EventLoop 执行的原因，参见 https://github.com/lightningMan/netty/commit/4638df20628a8987c8709f0f8e5f3679a914ce1a
    ch.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            System.out.println(Thread.currentThread() + ": ServerBootstrapAcceptor");
            pipeline.addLast(new ServerBootstrapAcceptor(
                    ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
        }
    });
    ```

    - 因为此时 Channel 并未注册到 EventLoop 中。如果调用 `EventLoop#execute(Runnable runnable)` 方法，会抛出 `Exception in thread "main" java.lang.IllegalStateException: channel not registered to an event loop` 异常。

## 666. 彩蛋

Netty 服务端启动涉及的流程非常多，所以有不理解的地方，胖友可以多多调试。在其中涉及到的 EventLoopGroup、EventLoop、Pipeline 等等组件，我们后在后续的文章，正式分享。

另外，也推荐如下和 Netty 服务端启动相关的文章，以加深理解：

- 闪电侠 [《Netty 源码分析之服务端启动全解析》](https://www.jianshu.com/p/c5068caab217)
- 小明哥 [《【死磕 Netty 】—— 服务端启动过程分析》](http://cmsblogs.com/?p=2470)
- 占小狼 [《Netty 源码分析之服务启动》](https://www.jianshu.com/p/e577803f0fb8)
- 杨武兵 [《Netty 源码分析系列 —— Bootstrap》](https://my.oschina.net/ywbrj042/blog/868798)
- 永顺 [《Netty 源码分析之 一 揭开 Bootstrap 神秘的红盖头 (服务器端)》](https://segmentfault.com/a/1190000007283053)

# 精尽 Netty 源码分析 —— 启动（二）之客户端

2018-04-05

## 1. 概述

本文，我们来分享 Bootstrap 分享 Netty 客户端。因为我们日常使用 Netty 主要使用 NIO 部分，所以本文也只分享 Netty NIO 客户端。

## 2. Bootstrap 示例

下面，我们先来看一个 ServerBootstrap 的使用示例，就是我们在 [《精尽 Netty 源码分析 —— 调试环境搭建》](http://svip.iocoder.cn/Netty/build-debugging-environment/#5-2-EchoClient) 搭建的 EchoClient 示例。代码如下：

```
public final class EchoClient {

    static final boolean SSL = System.getProperty("ssl") != null;
    static final String HOST = System.getProperty("host", "127.0.0.1");
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));
    static final int SIZE = Integer.parseInt(System.getProperty("size", "256"));

    public static void main(String[] args) throws Exception {
        // Configure SSL.git
        // 配置 SSL
        final SslContext sslCtx;
        if (SSL) {
            sslCtx = SslContextBuilder.forClient()
                .trustManager(InsecureTrustManagerFactory.INSTANCE).build();
        } else {
            sslCtx = null;
        }

        // Configure the client.
        // 创建一个 EventLoopGroup 对象
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            // 创建 Bootstrap 对象
            Bootstrap b = new Bootstrap();
            b.group(group) // 设置使用的 EventLoopGroup
             .channel(NioSocketChannel.class) // 设置要被实例化的为 NioSocketChannel 类
             .option(ChannelOption.TCP_NODELAY, true) // 设置 NioSocketChannel 的可选项
             .handler(new ChannelInitializer<SocketChannel>() { // 设置 NioSocketChannel 的处理器
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc(), HOST, PORT));
                     }
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(new EchoClientHandler());
                 }
             });

            // Start the client.
            // 连接服务器，并同步等待成功，即启动客户端
            ChannelFuture f = b.connect(HOST, PORT).sync();

            // Wait until the connection is closed.
            // 监听客户端关闭，并阻塞等待
            f.channel().closeFuture().sync();
        } finally {
            // Shut down the event loop to terminate all threads.
            // 优雅关闭一个 EventLoopGroup 对象
            group.shutdownGracefully();
        }
    }
}
```

- 示例比较简单，已经添加中文注释，胖友自己查看。

## 3. Bootstrap

`io.netty.bootstrap.Bootstrap` ，实现 AbstractBootstrap 抽象类，用于 Client 的启动器实现类。

### 3.1 构造方法

```
/**
 * 默认地址解析器对象
 */
private static final AddressResolverGroup<?> DEFAULT_RESOLVER = DefaultAddressResolverGroup.INSTANCE;

/**
 * 启动类配置对象
 */
private final BootstrapConfig config = new BootstrapConfig(this);
/**
 * 地址解析器对象
 */
@SuppressWarnings("unchecked")
private volatile AddressResolverGroup<SocketAddress> resolver = (AddressResolverGroup<SocketAddress>) DEFAULT_RESOLVER;
/**
 * 连接地址
 */
private volatile SocketAddress remoteAddress;

public Bootstrap() { }

private Bootstrap(Bootstrap bootstrap) {
    super(bootstrap);
    resolver = bootstrap.resolver;
    remoteAddress = bootstrap.remoteAddress;
}
```

- `config` 属性，BootstrapConfig 对象，启动类配置对象。
- `resolver` 属性，地址解析器对象。绝大多数情况下，使用 `DEFAULT_RESOLVER` 即可。
- `remoteAddress` 属性，连接地址。

### 3.2 resolver

`#resolver(AddressResolverGroup<?> resolver)` 方法，设置 `resolver` 属性。代码如下：

```
public Bootstrap resolver(AddressResolverGroup<?> resolver) {
    this.resolver = (AddressResolverGroup<SocketAddress>) (resolver == null ? DEFAULT_RESOLVER : resolver);
    return this;
}
```

### 3.3 remoteAddress

`#remoteAddress(...)` 方法，设置 `remoteAddress` 属性。代码如下：

```
public Bootstrap resolver(AddressResolverGroup<?> resolver) {
    this.resolver = (AddressResolverGroup<SocketAddress>) (resolver == null ? DEFAULT_RESOLVER : resolver);
    return this;
}

public Bootstrap remoteAddress(SocketAddress remoteAddress) {
    this.remoteAddress = remoteAddress;
    return this;
}

public Bootstrap remoteAddress(String inetHost, int inetPort) {
    remoteAddress = InetSocketAddress.createUnresolved(inetHost, inetPort);
    return this;
}

public Bootstrap remoteAddress(InetAddress inetHost, int inetPort) {
    remoteAddress = new InetSocketAddress(inetHost, inetPort);
    return this;
}
```

### 3.4 validate

`#validate()` 方法，校验配置是否正确。代码如下：

```
@Override
public Bootstrap validate() {
    // 父类校验
    super.validate();
    // handler 非空
    if (config.handler() == null) {
        throw new IllegalStateException("handler not set");
    }
    return this;
}
```

- 在 `#connect(...)` 方法中，连接服务端时，会调用该方法进行校验。

### 3.5 clone

`#clone(...)` 方法，克隆 Bootstrap 对象。代码如下：

```
@Override
public Bootstrap clone() {
    return new Bootstrap(this);
}

public Bootstrap clone(EventLoopGroup group) {
    Bootstrap bs = new Bootstrap(this);
    bs.group = group;
    return bs;
}
```

- 两个克隆方法，都是调用参数为 `bootstrap` 为 Bootstrap 构造方法，克隆一个 Bootstrap 对象。差别在于，下面的方法，多了对 `group` 属性的赋值。

### 3.6 connect

`#connect(...)` 方法，连接服务端，即启动客户端。代码如下：

```
public ChannelFuture connect() {
    // 校验必要参数
    validate();
    SocketAddress remoteAddress = this.remoteAddress;
    if (remoteAddress == null) {
        throw new IllegalStateException("remoteAddress not set");
    }
    // 解析远程地址，并进行连接
    return doResolveAndConnect(remoteAddress, config.localAddress());
}

public ChannelFuture connect(String inetHost, int inetPort) {
    return connect(InetSocketAddress.createUnresolved(inetHost, inetPort));
}

public ChannelFuture connect(InetAddress inetHost, int inetPort) {
    return connect(new InetSocketAddress(inetHost, inetPort));
}

public ChannelFuture connect(SocketAddress remoteAddress) {
    // 校验必要参数
    validate();
    if (remoteAddress == null) {
        throw new NullPointerException("remoteAddress");
    }
    // 解析远程地址，并进行连接
    return doResolveAndConnect(remoteAddress, config.localAddress());
}

public ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress) {
    // 校验必要参数
    validate();
    if (remoteAddress == null) {
        throw new NullPointerException("remoteAddress");
    }
    // 解析远程地址，并进行连接
    return doResolveAndConnect(remoteAddress, localAddress);
}
```

- 该方法返回的是 ChannelFuture 对象，也就是**异步**的连接服务端，启动客户端。如果需要**同步**，则需要调用 `ChannelFuture#sync()` 方法。

`#connect(...)` 方法，核心流程如下图：

`#bind(...)` 方法，核心流程如下图：

![核心流程](http://www.iocoder.cn/images/Netty/2018_04_05/01.png)

- 主要有 5 个步骤，下面我们来拆解代码，看看和我们在 [《精尽 Netty 源码分析 —— NIO 基础（五）之示例》](http://svip.iocoder.cn/Netty/nio-5-demo/?self) 的 NioClient 的代码，是**怎么对应**的。
- 相比 `#bind(...)` 方法的流程，主要是**绿色**的 2 个步骤。

#### 3.6.1 doResolveAndConnect

`#doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress)` 方法，代码如下：

```
 1: private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
 2:     // 初始化并注册一个 Channel 对象，因为注册是异步的过程，所以返回一个 ChannelFuture 对象。
 3:     final ChannelFuture regFuture = initAndRegister();
 4:     final Channel channel = regFuture.channel();
 5: 
 6:     if (regFuture.isDone()) {
 7:         // 若执行失败，直接进行返回。
 8:         if (!regFuture.isSuccess()) {
 9:             return regFuture;
10:         }
11:         // 解析远程地址，并进行连接
12:         return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
13:     } else {
14:         // Registration future is almost always fulfilled already, but just in case it's not.
15:         final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
16:         regFuture.addListener(new ChannelFutureListener() {
17: 
18:             @Override
19:             public void operationComplete(ChannelFuture future) throws Exception {
20:                 // Directly obtain the cause and do a null check so we only need one volatile read in case of a
21:                 // failure.
22:                 Throwable cause = future.cause();
23:                 if (cause != null) {
24:                     // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
25:                     // IllegalStateException once we try to access the EventLoop of the Channel.
26:                     promise.setFailure(cause);
27:                 } else {
28:                     // Registration was successful, so set the correct executor to use.
29:                     // See https://github.com/netty/netty/issues/2586
30:                     promise.registered();
31: 
32:                     // 解析远程地址，并进行连接
33:                     doResolveAndConnect0(channel, remoteAddress, localAddress, promise);
34:                 }
35:             }
36: 
37:         });
38:         return promise;
39:     }
40: }
```

- 第 3 行：调用

   

  ```
  #initAndRegister()
  ```

   

  方法，初始化并注册一个 Channel 对象。因为注册是

  异步

  的过程，所以返回一个 ChannelFuture 对象。详细解析，见

   

  「3.7 initAndRegister」

   

  。

  - 第 6 至 10 行：若执行失败，直接进行返回 `regFuture` 对象。

- 第 9 至 37 行：因为注册是

  异步

  的过程，有可能已完成，有可能未完成。所以实现代码分成了【第 12 行】和【第 13 至 37 行】分别处理已完成和未完成的情况。

  - **核心**在【第 12 行】或者【第 33 行】的代码，调用 `#doResolveAndConnect0(final Channel channel, SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise)` 方法，解析远程地址，并进行连接。
  - 如果**异步**注册对应的 ChanelFuture 未完成，则调用 `ChannelFuture#addListener(ChannelFutureListener)` 方法，添加监听器，在**注册**完成后，进行回调执行 `#doResolveAndConnect0(...)` 方法的逻辑。详细解析，见 [「3.6.2 doResolveAndConnect0」](http://svip.iocoder.cn/Netty/bootstrap-2-client/#) 。
  - 所以总结来说，**resolve 和 connect 的逻辑，执行在 register 的逻辑之后**。

#### 3.6.2 doResolveAndConnect0

> 老艿艿：此小节的内容，胖友先看完 [「3.7 initAndRegister」](http://svip.iocoder.cn/Netty/bootstrap-2-client/#) 的内容在回过头来看。因为 `#doResolveAndConnect0(...)` 方法的执行，在 `#initAndRegister()` 方法之后。

`#doResolveAndConnect0(...)` 方法，解析远程地址，并进行连接。代码如下：

```
 1: private ChannelFuture doResolveAndConnect0(final Channel channel, SocketAddress remoteAddress,
 2:                                            final SocketAddress localAddress, final ChannelPromise promise) {
 3:     try {
 4:         final EventLoop eventLoop = channel.eventLoop();
 5:         final AddressResolver<SocketAddress> resolver = this.resolver.getResolver(eventLoop);
 6: 
 7:         if (!resolver.isSupported(remoteAddress) || resolver.isResolved(remoteAddress)) {
 8:             // Resolver has no idea about what to do with the specified remote address or it's resolved already.
 9:             doConnect(remoteAddress, localAddress, promise);
10:             return promise;
11:         }
12: 
13:         // 解析远程地址
14:         final Future<SocketAddress> resolveFuture = resolver.resolve(remoteAddress);
15: 
16:         if (resolveFuture.isDone()) {
17:             // 解析远程地址失败，关闭 Channel ，并回调通知 promise 异常
18:             final Throwable resolveFailureCause = resolveFuture.cause();
19:             if (resolveFailureCause != null) {
20:                 // Failed to resolve immediately
21:                 channel.close();
22:                 promise.setFailure(resolveFailureCause);
23:             } else {
24:                 // Succeeded to resolve immediately; cached? (or did a blocking lookup)
25:                 // 连接远程地址
26:                 doConnect(resolveFuture.getNow(), localAddress, promise);
27:             }
28:             return promise;
29:         }
30: 
31:         // Wait until the name resolution is finished.
32:         resolveFuture.addListener(new FutureListener<SocketAddress>() {
33:             @Override
34:             public void operationComplete(Future<SocketAddress> future) throws Exception {
35:                 // 解析远程地址失败，关闭 Channel ，并回调通知 promise 异常
36:                 if (future.cause() != null) {
37:                     channel.close();
38:                     promise.setFailure(future.cause());
39:                 // 解析远程地址成功，连接远程地址
40:                 } else {
41:                     doConnect(future.getNow(), localAddress, promise);
42:                 }
43:             }
44:         });
45:     } catch (Throwable cause) {
46:         // 发生异常，并回调通知 promise 异常
47:         promise.tryFailure(cause);
48:     }
49:     return promise;
50: }
```

- 第 3 至 14 行：使用

   

  ```
  resolver
  ```

   

  解析远程地址。因为解析是

  异步

  的过程，所以返回一个 Future 对象。

  - 详细的解析远程地址的代码，考虑到暂时不是本文的重点，所以暂时省略。😈 老艿艿猜测胖友应该也暂时不感兴趣，哈哈哈。

- 第 16 至 44 行：因为注册是

  异步

  的过程，有可能已完成，有可能未完成。所以实现代码分成了【第 16 至 29 行】和【第 31 至 44 行】分别处理已完成和未完成的情况。

  - **核心**在【第 26 行】或者【第 41 行】的代码，调用 `#doConnect(...)` 方法，连接远程地址。
  - 如果**异步**解析对应的 Future 未完成，则调用 `Future#addListener(FutureListener)` 方法，添加监听器，在**解析**完成后，进行回调执行 `#doConnect(...)` 方法的逻辑。详细解析，见 见 [「3.13.3 doConnect」](http://svip.iocoder.cn/Netty/bootstrap-2-client/#) 。
  - 所以总结来说，**connect 的逻辑，执行在 resolve 的逻辑之后**。
  - 老艿艿目前使用 [「2. Bootstrap 示例」](http://svip.iocoder.cn/Netty/bootstrap-2-client/#) 测试下来，符合【第 16 至 30 行】的条件，即无需走**异步**的流程。

#### 3.6.3 doConnect

`#doConnect(...)` 方法，执行 Channel 连接远程地址的逻辑。代码如下：

```
 1: private static void doConnect(final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise connectPromise) {
 2: 
 3:     // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
 4:     // the pipeline in its channelRegistered() implementation.
 5:     final Channel channel = connectPromise.channel();
 6:     channel.eventLoop().execute(new Runnable() {
 7: 
 8:         @Override
 9:         public void run() {
10:             if (localAddress == null) {
11:                 channel.connect(remoteAddress, connectPromise);
12:             } else {
13:                 channel.connect(remoteAddress, localAddress, connectPromise);
14:             }
15:             connectPromise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
16:         }
17: 
18:     });
19: }
```

- 第 6 行：调用 EventLoop 执行 Channel 连接远程地址的逻辑。但是，实际上当前线程已经是 EventLoop 所在的线程了，为何还要这样操作呢？答案在【第 3 至 4 行】的英语注释。感叹句，Netty 虽然代码量非常庞大且复杂，但是英文注释真的是非常齐全，包括 Github 的 issue 对代码提交的描述，也非常健全。

- 第 10 至 14 行：调用

   

  ```
  Channel#connect(...)
  ```

   

  方法，执行 Channel 连接远程地址的逻辑。后续的方法栈调用如下图：



  - 还是老样子，我们先省略掉 pipeline 的内部实现代码，从 `AbstractNioUnsafe#connect(final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise)` 方法，继续向下分享。

`AbstractNioUnsafe#connect(final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise)` 方法，执行 Channel 连接远程地址的逻辑。代码如下：

```
 1: @Override
 2: public final void connect(final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
 3:     if (!promise.setUncancellable() || !ensureOpen(promise)) {
 4:         return;
 5:     }
 6: 
 7:     try {
 8:         // 目前有正在连接远程地址的 ChannelPromise ，则直接抛出异常，禁止同时发起多个连接。
 9:         if (connectPromise != null) {
10:             // Already a connect in process.
11:             throw new ConnectionPendingException();
12:         }
13: 
14:         // 记录 Channel 是否激活
15:         boolean wasActive = isActive();
16: 
17:         // 执行连接远程地址
18:         if (doConnect(remoteAddress, localAddress)) {
19:             fulfillConnectPromise(promise, wasActive);
20:         } else {
21:             // 记录 connectPromise
22:             connectPromise = promise;
23:             // 记录 requestedRemoteAddress
24:             requestedRemoteAddress = remoteAddress;
25: 
26:             // 使用 EventLoop 发起定时任务，监听连接远程地址超时。若连接超时，则回调通知 connectPromise 超时异常。
27:             // Schedule connect timeout.
28:             int connectTimeoutMillis = config().getConnectTimeoutMillis(); // 默认 30 * 1000 毫秒
29:             if (connectTimeoutMillis > 0) {
30:                 connectTimeoutFuture = eventLoop().schedule(new Runnable() {
31:                     @Override
32:                     public void run() {
33:                         ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
34:                         ConnectTimeoutException cause = new ConnectTimeoutException("connection timed out: " + remoteAddress);
35:                         if (connectPromise != null && connectPromise.tryFailure(cause)) {
36:                             close(voidPromise());
37:                         }
38:                     }
39:                 }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
40:             }
41: 
42:             // 添加监听器，监听连接远程地址取消。
43:             promise.addListener(new ChannelFutureListener() {
44:                 @Override
45:                 public void operationComplete(ChannelFuture future) throws Exception {
46:                     if (future.isCancelled()) {
47:                         // 取消定时任务
48:                         if (connectTimeoutFuture != null) {
49:                             connectTimeoutFuture.cancel(false);
50:                         }
51:                         // 置空 connectPromise
52:                         connectPromise = null;
53:                         close(voidPromise());
54:                     }
55:                 }
56:             });
57:         }
58:     } catch (Throwable t) {
59:         // 回调通知 promise 发生异常
60:         promise.tryFailure(annotateConnectException(t, remoteAddress));
61:         closeIfClosed();
62:     }
63: }
```

- 第 8 至 12 行：目前有正在连接远程地址的 ChannelPromise ，则直接抛出异常，禁止同时发起多个连接。`connectPromise` 变量，定义在 AbstractNioChannel 类中，代码如下：

  ```
  /**
   * 目前正在连接远程地址的 ChannelPromise 对象。
   *
   * The future of the current connection attempt.  If not null, subsequent
   * connection attempts will fail.
   */
  private ChannelPromise connectPromise;
  ```

- 第 15 行：调用 `#isActive()` 方法，获得 Channel 是否激活。NioSocketChannel 对该方法的实现代码如下：

  ```
  @Override
  public boolean isActive() {
      SocketChannel ch = javaChannel();
      return ch.isOpen() && ch.isConnected();
  }
  ```

  - 判断 SocketChannel 是否处于打开，并且连接的状态。此时，一般返回的是 `false` 。

- 第 18 行：调用 `#doConnect(SocketAddress remoteAddress, SocketAddress localAddress)` 方法，执行连接远程地址。代码如下：

  ```
  // NioSocketChannel.java
    1: @Override
    2: protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
    3:     // 绑定本地地址
    4:     if (localAddress != null) {
    5:         doBind0(localAddress);
    6:     }
    7: 
    8:     boolean success = false; // 执行是否成功
    9:     try {
   10:         // 连接远程地址
   11:         boolean connected = SocketUtils.connect(javaChannel(), remoteAddress);
   12:         // 若未连接完成，则关注连接( OP_CONNECT )事件。
   13:         if (!connected) {
   14:             selectionKey().interestOps(SelectionKey.OP_CONNECT);
   15:         }
   16:         // 标记执行是否成功
   17:         success = true;
   18:         // 返回是否连接完成
   19:         return connected;
   20:     } finally {
   21:         // 执行失败，则关闭 Channel
   22:         if (!success) {
   23:             doClose();
   24:         }
   25:     }
   26: }
  ```

  - 第 3 至 6 行：若 `localAddress` 非空，则调用 `#doBind0(SocketAddress)` 方法，绑定本地地址。一般情况下，NIO Client 是不需要绑定本地地址的。默认情况下，系统会随机分配一个可用的本地地址，进行绑定。

  - 第 11 行：调用 `SocketUtils#connect(SocketChannel socketChannel, SocketAddress remoteAddress)` 方法，Java 原生 NIO SocketChannel 连接 远程地址，并返回是否连接完成( 成功 )。代码如下：

    ```
    public static boolean connect(final SocketChannel socketChannel, final SocketAddress remoteAddress) throws IOException {
        try {
            return AccessController.doPrivileged(new PrivilegedExceptionAction<Boolean>() {
                @Override
                public Boolean run() throws IOException {
                    return socketChannel.connect(remoteAddress);
                }
            });
        } catch (PrivilegedActionException e) {
            throw (IOException) e.getCause();
        }
    }
    ```

    - 可能有胖友有和我一样的疑问，为什么将 connect 操作包在 AccessController 中呢？我们来看下 SocketUtils 类的注释：

      ```
      /**
       * Provides socket operations with privileges enabled. This is necessary for applications that use the
       * {@link SecurityManager} to restrict {@link SocketPermission} to their application. By asserting that these
       * operations are privileged, the operations can proceed even if some code in the calling chain lacks the appropriate
       * {@link SocketPermission}.
       */
      ```

      - 一般情况下，我们用不到，所以也可以暂时不用理解。
      - 感兴趣的胖友，可以 Google “AccessController” 关键字，或者阅读 [《Java 安全模型介绍》](https://www.ibm.com/developerworks/cn/java/j-lo-javasecurity/index.html) 。

  - 【重要】第 12 至 15 行：若连接未完成( `connected == false` )时，我们可以看到，调用 `SelectionKey#interestOps(ops)` 方法，添加连接事件( `SelectionKey.OP_CONNECT` )为感兴趣的事件。也就说，也就是说，当连接远程地址成功时，Channel 对应的 Selector 将会轮询到该事件，可以进一步处理。

  - 第 20 至 25 行：若执行失败( `success == false` )时，调用 `#doClose()` 方法，关闭 Channel 。

- 第 18 至 19 行：笔者测试下来，`#doConnect(SocketChannel socketChannel, SocketAddress remoteAddress)` 方法的结果为 `false` ，所以不会执行【第 19 行】代码的 `#fulfillConnectPromise(ChannelPromise promise, boolean wasActive)` 方法，而是执行【第 20 至 57 行】的代码逻辑。

- 第 22 行：记录 `connectPromise` 。

- 第 24 行：记录 `requestedRemoteAddress` 。`requestedRemoteAddress` 变量，在 AbstractNioChannel 类中定义，代码如下：

  ```
  /**
   * 正在连接的远程地址
   */
  private SocketAddress requestedRemoteAddress;
  ```

- 第 26 至 40 行：调用 `EventLoop#schedule(Runnable command, long delay, TimeUnit unit)` 方法，发起定时任务 `connectTimeoutFuture` ，监听连接远程地址**是否超时**。若连接超时，则回调通知 `connectPromise` 超时异常。`connectPromise` 变量，在 AbstractNioChannel 类中定义，代码如下：

  ```
  /**
   * 连接超时监听 ScheduledFuture 对象。
   */
  private ScheduledFuture<?> connectTimeoutFuture;
  ```

- 第 42 至 57 行：调用 `ChannelPromise#addListener(ChannelFutureListener)` 方法，添加监听器，监听连接远程地址**是否取消**。若取消，则取消 `connectTimeoutFuture` 任务，并置空 `connectPromise` 。这样，客户端 Channel 可以发起下一次连接。

#### 3.6.4 finishConnect

看到此处，可能胖友会有疑问，客户端的连接在哪里完成呢？答案在 `AbstractNioUnsafe#finishConnect()` 方法中。而该方法通过 Selector 轮询到 `SelectionKey.OP_CONNECT` 事件时，进行触发。调用栈如下图：![finishConnect 调用栈](http://www.iocoder.cn/images/Netty/2018_04_05/03.png)

```
* 哈哈哈，还是老样子，我们先省略掉 EventLoop 的内部实现代码，从 `AbstractNioUnsafe#finishConnect()` 方法，继续向下分享。
```

`AbstractNioUnsafe#finishConnect()` 方法，完成客户端的连接。代码如下：

```
 1: @Override
 2: public final void finishConnect() {
 3:     // Note this method is invoked by the event loop only if the connection attempt was
 4:     // neither cancelled nor timed out.
 5:     // 判断是否在 EventLoop 的线程中。
 6:     assert eventLoop().inEventLoop();
 7: 
 8:     try {
 9:         // 获得 Channel 是否激活
10:         boolean wasActive = isActive();
11:         // 执行完成连接
12:         doFinishConnect();
13:         // 通知 connectPromise 连接完成
14:         fulfillConnectPromise(connectPromise, wasActive);
15:     } catch (Throwable t) {
16:         // 通知 connectPromise 连接异常
17:         fulfillConnectPromise(connectPromise, annotateConnectException(t, requestedRemoteAddress));
18:     } finally {
19:         // 取消 connectTimeoutFuture 任务
20:         // Check for null as the connectTimeoutFuture is only created if a connectTimeoutMillis > 0 is used
21:         // See https://github.com/netty/netty/issues/1770
22:         if (connectTimeoutFuture != null) {
23:             connectTimeoutFuture.cancel(false);
24:         }
25:         // 置空 connectPromise
26:         connectPromise = null;
27:     }
28: }
```

- 第 6 行：判断是否在 EventLoop 的线程中。
- 第 10 行：调用 `#isActive()` 方法，获得 Channel 是否激活。笔者调试时，此时返回 `false` ，因为连接还没完成。
- 第 12 行：调用 `#doFinishConnect()` 方法，执行完成连接的逻辑。详细解析，见 [「3.6.4.1 doFinishConnect」](http://svip.iocoder.cn/Netty/bootstrap-2-client/#) 。
- 第 14 行：执行完成连接**成功**，调用 `#fulfillConnectPromise(ChannelPromise promise, boolean wasActive)` 方法，通知 `connectPromise` 连接完成。详细解析，见 [「3.6.4.2 fulfillConnectPromise 成功」](http://svip.iocoder.cn/Netty/bootstrap-2-client/#) 。
- 第 15 至 17 行：执行完成连接**异常**，调用 `#fulfillConnectPromise(ChannelPromise promise, Throwable cause)`方法，通知 `connectPromise` 连接异常。详细解析，见 [「3.6.4.3 fulfillConnectPromise 异常」](http://svip.iocoder.cn/Netty/bootstrap-2-client/#) 。
- 第 18 至 27 行：执行完成连接**结束**，取消 `connectTimeoutFuture` 任务，并置空 `connectPromise` 。

##### 3.6.4.1 DOFINISHCONNECT

`NioSocketChannel#doFinishConnect()` 方法，执行完成连接的逻辑。代码如下：

```
@Override
protected void doFinishConnect() throws Exception {
    if (!javaChannel().finishConnect()) {
        throw new Error();
    }
}
```

- 【重要】是不是非常熟悉的，调用 `SocketChannel#finishConnect()` 方法，完成连接。😈 美滋滋。

##### 3.6.4.2 FULFILLCONNECTPROMISE 成功

`AbstractNioUnsafe#fulfillConnectPromise(ChannelPromise promise, Throwable cause)` 方法，通知 `connectPromise` 连接完成。代码如下：

```
 1: private void fulfillConnectPromise(ChannelPromise promise, boolean wasActive) {
 2:     if (promise == null) {
 3:         // Closed via cancellation and the promise has been notified already.
 4:         return;
 5:     }
 6: 
 7:     // 获得 Channel 是否激活
 8:     // Get the state as trySuccess() may trigger an ChannelFutureListener that will close the Channel.
 9:     // We still need to ensure we call fireChannelActive() in this case.
10:     boolean active = isActive();
11: 
12:     // 回调通知 promise 执行成功
13:     // trySuccess() will return false if a user cancelled the connection attempt.
14:     boolean promiseSet = promise.trySuccess();
15: 
16:     // 若 Channel 是新激活的，触发通知 Channel 已激活的事件。
17:     // Regardless if the connection attempt was cancelled, channelActive() event should be triggered,
18:     // because what happened is what happened.
19:     if (!wasActive && active) {
20:         pipeline().fireChannelActive();
21:     }
22: 
23:     // If a user cancelled the connection attempt, close the channel, which is followed by channelInactive().
24:     // TODO 芋艿
25:     if (!promiseSet) {
26:         close(voidPromise());
27:     }
28: }
```

- 第 10 行：调用 `#isActive()` 方法，获得 Channel 是否激活。笔者调试时，此时返回 `true` ，因为连接已经完成。

- 第 14 行：回调通知 `promise` 执行成功。此处的通知，对应回调的是我们添加到 `#connect(...)` 方法返回的 ChannelFuture 的 ChannelFutureListener 的监听器。示例代码如下：

  ```
  ChannelFuture f = b.connect(HOST, PORT).addListener(new ChannelFutureListener() { // 回调的就是我！！！
      @Override
      public void operationComplete(ChannelFuture future) throws Exception {
          System.out.println("连接完成");
      }
  }).sync();
  ```

- 第 19 行：因为

   

  ```
  wasActive == false
  ```

   

  并且

   

  ```
  active == true
  ```

   

  ，因此，Channel 可以认为是

  新激活

  的，满足【第 20 行】代码的执行条件。

  - 第 40 行：调用 `DefaultChannelPipeline#fireChannelActive()` 方法，触发 Channel 激活的事件。【重要】后续的流程，和 NioServerSocketChannel 一样，也就说，会调用到 `AbstractUnsafe#beginRead()` 方法。这意味着什么呢？将我们创建 NioSocketChannel 时，设置的 `readInterestOp = SelectionKey.OP_READ` 添加为感兴趣的事件。也就说，客户端可以读取服务端发送来的数据。
  - 关于 `AbstractUnsafe#beginRead()` 方法的解析，见 [《精尽 Netty 源码分析 —— 启动（一）之服务端》的 「3.13.3 beginRead」](http://svip.iocoder.cn/Netty/bootstrap-1-server/?self) 部分。

- 第 23 至 27 行：TODO 芋艿 1004 fulfillConnectPromise promiseSet

##### 3.6.4.3 FULFILLCONNECTPROMISE 异常

`#fulfillConnectPromise(ChannelPromise promise, Throwable cause)` 方法，通知 `connectPromise` 连接异常。代码如下：

```
private void fulfillConnectPromise(ChannelPromise promise, Throwable cause) {
    if (promise == null) {
        // Closed via cancellation and the promise has been notified already.
        return;
    }

    // 回调通知 promise 发生异常
    // Use tryFailure() instead of setFailure() to avoid the race against cancel().
    promise.tryFailure(cause);
    // 关闭
    closeIfClosed();
}
```

- 比较简单，已经添加中文注释，胖友自己查看。

### 3.7 initAndRegister

Bootstrap 继承 AbstractBootstrap 抽象类，所以 `#initAndRegister()` 方法的流程上是一致的。所以和 ServerBootstrap 的差别在于：

1. 创建的 Channel 对象不同。
2. 初始化 Channel 配置的代码实现不同。

#### 3.7.1 创建 Channel 对象

考虑到本文的内容，我们以 NioSocketChannel 的创建过程作为示例。创建 NioSocketChannel 对象的流程，和 NioServerSocketChannel 基本是一致的，所以流程图我们就不提供了，直接开始撸源码。

##### 3.7.1.1 NIOSOCKETCHANNEL

```
private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();

private final SocketChannelConfig config;

public NioSocketChannel() {
    this(DEFAULT_SELECTOR_PROVIDER);
}

public NioSocketChannel(SelectorProvider provider) {
    this(newSocket(provider));
}
```

- `DEFAULT_SELECTOR_PROVIDER` **静态**属性，默认的 SelectorProvider 实现类。

- `config` 属性，Channel 对应的配置对象。每种 Channel 实现类，也会对应一个 ChannelConfig 实现类。例如，NioSocketChannel 类，对应 SocketChannelConfig 配置类。

- 在构造方法中，调用 `#newSocket(SelectorProvider provider)` 方法，创建 NIO 的 ServerSocketChannel 对象。代码如下：

  ```
  private static SocketChannel newSocket(SelectorProvider provider) {
      try {
          /**
           *  Use the {@link SelectorProvider} to open {@link SocketChannel} and so remove condition in
           *  {@link SelectorProvider#provider()} which is called by each SocketChannel.open() otherwise.
           *
           *  See <a href="https://github.com/netty/netty/issues/2308">#2308</a>.
           */
          return provider.openSocketChannel();
      } catch (IOException e) {
          throw new ChannelException("Failed to open a socket.", e);
      }
  }
  ```

  - 😈 是不是很熟悉这样的代码，效果和 `SocketChannel#open()` 方法创建 SocketChannel 对象是一致。

- `#NioSocketChannel(SocketChannel channel)` 构造方法，代码如下：

  ```
  public NioSocketChannel(SocketChannel socket) {
      this(null, socket);
  }
  
  public NioSocketChannel(Channel parent, SocketChannel socket) {
      super(parent, socket);
      config = new NioSocketChannelConfig(this, socket.socket());
  }
  ```

  - 调用父 AbstractNioByteChannel 的构造方法。详细解析，见 [「3.7.1.2 AbstractNioByteChannel」](http://svip.iocoder.cn/Netty/bootstrap-2-client/#) 。
  - 初始化 `config` 属性，创建 NioSocketChannelConfig 对象。

##### 3.7.1.2 ABSTRACTNIOBYTECHANNEL

```
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
    super(parent, ch, SelectionKey.OP_READ);
}
```

- 调用父 AbstractNioChannel 的构造方法。后续的构造方法，和 NioServerSocketChannel 是一致的。
  - 注意传入的 SelectionKey 的值为 `OP_READ` 。

#### 3.7.2 初始化 Channel 配置

`#init(Channel channel)` 方法，初始化 Channel 配置。代码如下：

```
@Override
void init(Channel channel) throws Exception {
    ChannelPipeline p = channel.pipeline();

    // 添加处理器到 pipeline 中
    p.addLast(config.handler());

    // 初始化 Channel 的可选项集合
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        setChannelOptions(channel, options, logger);
    }

    // 初始化 Channel 的属性集合
    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            channel.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
        }
    }
}
```

- 比较简单，已经添加中文注释，胖友自己查看。

## 666. 彩蛋

撸完 Netty 服务端启动之后，再撸 Netty 客户端启动之后，出奇的顺手。美滋滋。

另外，也推荐如下和 Netty 客户端启动相关的文章，以加深理解：

- 杨武兵 [《Netty 源码分析系列 —— Bootstrap》](https://my.oschina.net/ywbrj042/blog/868798)
- 永顺 [《Netty 源码分析之 一 揭开 Bootstrap 神秘的红盖头 (客户端)》](https://segmentfault.com/a/1190000007282789)