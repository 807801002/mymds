# 精尽 Netty 源码解析 —— ChannelPipeline（一）之初始化

2018-06-01

## 1. 概述

在 [《精尽 Netty 源码分析 —— Netty 简介（二）之核心组件》](http://svip.iocoder.cn/Netty/intro-2/?self) 中，对 EventLoopGroup 和 EventLoop 做了定义，我们再来回顾下：

> ChannelPipeline 为 ChannelHandler 的**链**，提供了一个容器并定义了用于沿着链传播入站和出站事件流的 API 。一个数据或者事件可能会被多个 Handler 处理，在这个过程中，数据或者事件经流 ChannelPipeline ，由 ChannelHandler 处理。在这个处理过程中，一个 ChannelHandler 接收数据后处理完成后交给下一个 ChannelHandler，或者什么都不做直接交给下一个 ChannelHandler。

因为 ChannelPipeline 涉及的代码量较大，所以笔者会分成好几篇文章分别分享。而本文，我们来分享 ChannelPipeline 的**初始化**。也因此，本文更多是体现 ChannelPipeline 的**整体性**，所以不会过多介绍每个类的具体的**每个方法**的实现。

## 2. ChannelPipeline

`io.netty.channel.ChannelPipeline` ，继承 ChannelInboundInvoker、ChannelOutboundInvoker、Iterable 接口，Channel Pipeline 接口。代码如下：

```
public interface ChannelPipeline
        extends ChannelInboundInvoker, ChannelOutboundInvoker, Iterable<Entry<String, ChannelHandler>> {

    // ========== 添加 ChannelHandler 相关 ==========
    ChannelPipeline addFirst(String name, ChannelHandler handler);
    ChannelPipeline addFirst(EventExecutorGroup group, String name, ChannelHandler handler);
    ChannelPipeline addLast(String name, ChannelHandler handler);
    ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler);
    ChannelPipeline addBefore(String baseName, String name, ChannelHandler handler);
    ChannelPipeline addBefore(EventExecutorGroup group, String baseName, String name, ChannelHandler handler);
    ChannelPipeline addAfter(String baseName, String name, ChannelHandler handler);
    ChannelPipeline addAfter(EventExecutorGroup group, String baseName, String name, ChannelHandler handler);
    ChannelPipeline addFirst(ChannelHandler... handlers);
    ChannelPipeline addFirst(EventExecutorGroup group, ChannelHandler... handlers);
    ChannelPipeline addLast(ChannelHandler... handlers);
    ChannelPipeline addLast(EventExecutorGroup group, ChannelHandler... handlers);

    // ========== 移除 ChannelHandler 相关 ==========
    ChannelPipeline remove(ChannelHandler handler);
    ChannelHandler remove(String name);
    <T extends ChannelHandler> T remove(Class<T> handlerType);
    ChannelHandler removeFirst();
    ChannelHandler removeLast();
    
    // ========== 替换 ChannelHandler 相关 ==========
    ChannelPipeline replace(ChannelHandler oldHandler, String newName, ChannelHandler newHandler);
    ChannelHandler replace(String oldName, String newName, ChannelHandler newHandler);
    <T extends ChannelHandler> T replace(Class<T> oldHandlerType, String newName, ChannelHandler newHandler);

    // ========== 查询 ChannelHandler 相关 ==========
    ChannelHandler first();
    ChannelHandlerContext firstContext();
    ChannelHandler last();
    ChannelHandlerContext lastContext();
    ChannelHandler get(String name);
    <T extends ChannelHandler> T get(Class<T> handlerType);
    ChannelHandlerContext context(ChannelHandler handler);
    ChannelHandlerContext context(String name);
    ChannelHandlerContext context(Class<? extends ChannelHandler> handlerType);
    List<String> names();

    // ========== Channel 相关 ==========
    Channel channel();

    // ========== ChannelInboundInvoker 相关 ==========    
    @Override
    ChannelPipeline fireChannelRegistered();
    @Override
    ChannelPipeline fireChannelUnregistered();
    @Override
    ChannelPipeline fireChannelActive();
    @Override
    ChannelPipeline fireChannelInactive();
    @Override
    ChannelPipeline fireExceptionCaught(Throwable cause);
    @Override
    ChannelPipeline fireUserEventTriggered(Object event);
    @Override
    ChannelPipeline fireChannelRead(Object msg);
    @Override
    ChannelPipeline fireChannelReadComplete();
    @Override
    ChannelPipeline fireChannelWritabilityChanged();

    // ========== ChannelOutboundInvoker 相关 ==========    
    @Override
    ChannelPipeline flush();
    
}
```

虽然接口的方法比较多，笔者做了归类如下：

- ChannelHandler 的增删改查的相关方法。
- Channel 的相关方法，目前只有一个。
- 继承自 ChannelInboundInvoker 的相关方法。
- 继承自 ChannelOutboundInvoker 的相关方法。

有可能会疑惑为什么继承 Iterable 接口？因为 ChannelPipeline 是 ChannelHandler 的**链**。

ChannelPipeline 的类图如下：

![ChannelPipeline 类图](http://www.iocoder.cn/images/Netty/2018_06_01/01.png)

### 2.1 ChannelInboundInvoker

`io.netty.channel.ChannelInboundInvoker` ，Channel Inbound Invoker( 调用者 ) 接口。代码如下：

```
ChannelPipeline fireChannelRegistered();
ChannelPipeline fireChannelUnregistered();
ChannelPipeline fireChannelActive();
ChannelPipeline fireChannelInactive();
ChannelPipeline fireExceptionCaught(Throwable cause);
ChannelPipeline fireUserEventTriggered(Object event);
ChannelPipeline fireChannelRead(Object msg);
ChannelPipeline fireChannelReadComplete();
ChannelPipeline fireChannelWritabilityChanged();
```

- 通知 Channel 事件的接口方法。

### 2.2 ChannelOutboundInvoker

`io.netty.channel.ChannelOutboundInvoker` ，Channel Outbound Invoker( 调用者 ) 接口。代码如下：

```
// ========== Channel 操作相关 ==========    
ChannelFuture bind(SocketAddress localAddress);
ChannelFuture connect(SocketAddress remoteAddress);
ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress);
ChannelFuture disconnect();
ChannelFuture close();
ChannelFuture deregister();
ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise);
ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise);
ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);
ChannelFuture disconnect(ChannelPromise promise);
ChannelFuture close(ChannelPromise promise);
ChannelFuture deregister(ChannelPromise promise);
ChannelOutboundInvoker read();
ChannelFuture write(Object msg);
ChannelFuture write(Object msg, ChannelPromise promise);
ChannelOutboundInvoker flush();
ChannelFuture writeAndFlush(Object msg, ChannelPromise promise);
ChannelFuture writeAndFlush(Object msg);

// ========== Promise 相关 ==========    
ChannelPromise newPromise();
ChannelProgressivePromise newProgressivePromise();
ChannelFuture newSucceededFuture();
ChannelFuture newFailedFuture(Throwable cause);
ChannelPromise voidPromise();
```

- 发起 Channel 操作的接口方法。
- 创建 Promise 对象的接口方法。

### 2.3 Outbound v.s Inbound 事件

在 [《Netty 源码分析之 二 贯穿Netty 的大动脉 ── ChannelPipeline (二)》](https://segmentfault.com/a/1190000007309311) 中，笔者看到一个比较不错的总结：

> 老艿艿：因为要加一些注释，所以暂时不使用引用。

**对于 Outbound 事件**：

- Outbound 事件是【请求】事件(由 Connect 发起一个请求, 并最终由 Unsafe 处理这个请求)

- Outbound 事件的发起者是 Channel

- Outbound 事件的处理者是 Unsafe

- Outbound 事件在 Pipeline 中的传输方向是 `tail` -> `head`

  > 旁白：Outbound 翻译为“出站”，所以从 `tail`( 尾 )到 `head`( 头 )也合理。
  >
  > 至于什么是 `head` 和 `tail` ，等看了具体的 ChannelPipeline 实现类 DefaultChannelPipeline 再说。

- 在 ChannelHandler 中处理事件时, 如果这个 Handler 不是最后一个 Handler, 则需要调用 `ctx.xxx` (例如 `ctx.connect` ) 将此事件继续传播下去. 如果不这样做, 那么此事件的传播会提前终止.

- Outbound 事件流: `Context.OUT_EVT` -> `Connect.findContextOutbound` -> `nextContext.invokeOUT_EVT` -> `nextHandler.OUT_EVT` -> `nextContext.OUT_EVT`

**对于 Inbound 事件**：

- Inbound 事件是【通知】事件, 当某件事情已经就绪后, 通知上层.

- Inbound 事件发起者是 Unsafe

- Inbound 事件的处理者是 TailContext, 如果用户没有实现自定义的处理方法, 那么Inbound 事件默认的处理者是 TailContext, 并且其处理方法是空实现.

- Inbound 事件在 Pipeline 中传输方向是 `head`( 头 ) -> `tail`( 尾 )

  > 旁白：Inbound 翻译为“入站”，所以从 `head`( 头 )到 `tail`( 尾 )也合理。

- 在 ChannelHandler 中处理事件时, 如果这个 Handler 不是最后一个 Handler, 则需要调用 `ctx.fireIN_EVT` (例如 `ctx.fireChannelActive` ) 将此事件继续传播下去. 如果不这样做, 那么此事件的传播会提前终止.

- Inbound 事件流: `Context.fireIN_EVT` -> `Connect.findContextInbound` -> `nextContext.invokeIN_EVT` -> `nextHandler.IN_EVT` -> `nextContext.fireIN_EVT`

Outbound 和 Inbound 事件十分的镜像, 并且 Context 与 Handler 直接的调用关系是否容易混淆, 因此读者在阅读这里的源码时, 需要特别的注意。

## 3. DefaultChannelPipeline

`io.netty.channel.DefaultChannelPipeline` ，实现 ChannelPipeline 接口，默认 ChannelPipeline 实现类。😈 实际上，也只有这个实现类。

### 3.1 静态属性

```
/**
 * {@link #head} 的名字
 */
private static final String HEAD_NAME = generateName0(HeadContext.class);
/**
 * {@link #tail} 的名字
 */
private static final String TAIL_NAME = generateName0(TailContext.class);

/**
 * 名字({@link AbstractChannelHandlerContext#name})缓存 ，基于 ThreadLocal ，用于生成在线程中唯一的名字。
 */
private static final FastThreadLocal<Map<Class<?>, String>> nameCaches = new FastThreadLocal<Map<Class<?>, String>>() {

    @Override
    protected Map<Class<?>, String> initialValue() throws Exception {
        return new WeakHashMap<Class<?>, String>();
    }

};

/**
 * {@link #estimatorHandle} 的原子更新器
 */
private static final AtomicReferenceFieldUpdater<DefaultChannelPipeline, MessageSizeEstimator.Handle> ESTIMATOR =
        AtomicReferenceFieldUpdater.newUpdater(
                DefaultChannelPipeline.class, MessageSizeEstimator.Handle.class, "estimatorHandle");
```

- `HEAD_NAME` 和 `TAIL_NAME` 静态属性，通过调用 `#generateName0(Class<?> handlerType)` 方法，生成对应的名字。代码如下：

  ```
  private static String generateName0(Class<?> handlerType) {
      return StringUtil.simpleClassName(handlerType) + "#0";
  }
  ```

  - 即 `HEAD_NAME = "HeadContext#0"`，`TAIL_NAME= "TailContext#0"` 。

- `nameCaches` 静态属性，名字( `AbstractChannelHandlerContext.name` )缓存 ，基于 ThreadLocal ，用于生成**在线程中唯一的名字**。详细解析，见 [《精尽 Netty 源码解析 —— Pipeline（二）之添加 ChannelHandler》](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler) 。

- `ESTIMATOR` 静态属性，`estimatorHandle` 属性的**原子**更新器。

### 3.2 构造方法

```
/**
 * Head 节点
 */
final AbstractChannelHandlerContext head;
/**
 * Tail 节点
 */
final AbstractChannelHandlerContext tail;

/**
 * 所属 Channel 对象
 */
private final Channel channel;
/**
 * 成功的 Promise 对象
 */
private final ChannelFuture succeededFuture;
/**
 * 不进行通知的 Promise 对象
 *
 * 用于一些方法执行，需要传入 Promise 类型的方法参数，但是不需要进行通知，就传入该值
 *
 * @see io.netty.channel.AbstractChannel.AbstractUnsafe#safeSetSuccess(ChannelPromise) 
 */
private final VoidChannelPromise voidPromise;
/**
 * TODO 1008 DefaultChannelPipeline 字段用途
 */
private final boolean touch = ResourceLeakDetector.isEnabled();

/**
 * 子执行器集合。
 *
 * 默认情况下，ChannelHandler 使用 Channel 所在的 EventLoop 作为执行器。
 * 但是如果有需要，也可以自定义执行器。详细解析，见 {@link #childExecutor(EventExecutorGroup)} 。
 * 实际情况下，基本不会用到。和基友【闪电侠】沟通过。
 */
private Map<EventExecutorGroup, EventExecutor> childExecutors;
/**
 * TODO 1008 DefaultChannelPipeline 字段用途
 */
private volatile MessageSizeEstimator.Handle estimatorHandle;
/**
 * 是否首次注册
 */
private boolean firstRegistration = true;

/**
 * This is the head of a linked list that is processed by {@link #callHandlerAddedForAllHandlers()} and so process
 * all the pending {@link #callHandlerAdded0(AbstractChannelHandlerContext)}.
 *
 * We only keep the head because it is expected that the list is used infrequently and its size is small.
 * Thus full iterations to do insertions is assumed to be a good compromised to saving memory and tail management
 * complexity.
 * 
 * 准备添加 ChannelHandler 的回调
 */
private PendingHandlerCallback pendingHandlerCallbackHead;

/**
 * Set to {@code true} once the {@link AbstractChannel} is registered.Once set to {@code true} the value will never
 * change.
 * Channel 是否已注册
 */
private boolean registered;

protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    // succeededFuture 的创建
    succeededFuture = new SucceededChannelFuture(channel, null);
    // voidPromise 的创建
    voidPromise =  new VoidChannelPromise(channel, true);

    // 创建 Tail 及诶点
    tail = new TailContext(this); // <1>
    // 创建 Head 节点
    head = new HeadContext(this); // <2>

    // 相互指向 <3>
    head.next = tail;
    tail.prev = head;
}
```

- `head` 属性，Head 节点，在构造方法的 `<1>` 处初始化。详细解析，见 [「4.2 HeadContext」](http://svip.iocoder.cn/Netty/ChannelPipeline-1-init/#) 。

- `tail` 节点，Tail 节点，在构造方法的 `<2>` 处初始化。详细解析，见 [「4.3 TailContext」](http://svip.iocoder.cn/Netty/ChannelPipeline-1-init/#) 。

- 在构造方法的 `<3>` 处，`head` 节点向**下**指向 `tail` 节点，`tail` 节点向**上**指向 `head` 节点，从而形成**相互**的指向。即如下图所示：

  > FROM [《netty 源码分析之 pipeline(一)》](https://www.jianshu.com/p/6efa9c5fa702)
  >
  > ![pipeline 节点链(默认)](http://www.iocoder.cn/images/Netty/2018_06_01/02.png)

  - pipeline 中的节点的数据结构是 ChannelHandlerContext 类。每个 ChannelHandlerContext 包含**一个** ChannelHandler、它的**上下**节点( **从而形成 ChannelHandler 链** )、以及其他上下文。详细解析，见 [「4. ChannelHandlerContext」](http://svip.iocoder.cn/Netty/ChannelPipeline-1-init/#) 。

  - 默认情况下，pipeline 有 `head` 和 `tail` 节点，形成默认的 ChannelHandler 链。而我们可以在它们之间，加入自定义的 ChannelHandler 节点。如下图所示：

    > FROM [《netty 源码分析之 pipeline(一)》](https://www.jianshu.com/p/6efa9c5fa702)
    >
    > ![pipeline 节点链(自定义)](http://www.iocoder.cn/images/Netty/2018_06_01/03.png)

- `childExecutors` 属性，子执行器集合。默认情况下，ChannelHandler 使用 Channel 所在的 EventLoop 作为执行器。

  - 但是如果有需要，也可以自定义执行器。详细解析，见 [《精尽 Netty 源码解析 —— Pipeline（二）之添加 ChannelHandler》](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler)。
  - 实际情况下，基本不会用到。和基友【闪电侠】沟通过。

- `pendingHandlerCallbackHead` 属性，准备添加 ChannelHandler 的回调。详细解析，见 [《精尽 Netty 源码解析 —— Pipeline（二）之添加 ChannelHandler》](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler) 。

- `registered` 属性，Channel 是否已注册。详细解析，见 [《精尽 Netty 源码解析 —— Pipeline（二）之添加 ChannelHandler》](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler)。

- `firstRegistration` 属性，是否首次注册。详细解析，见 [《精尽 Netty 源码解析 —— Pipeline（二）之添加 ChannelHandler》](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler) 。

### 3.3 其他方法

DefaultChannelPipeline 中的其他方法，详细解析，见后续的文章。

## 4. ChannelHandlerContext

`io.netty.channel.ChannelHandlerContext` ，继承 ChannelInboundInvoker、ChannelOutboundInvoker、AttributeMap 接口，ChannelHandler Context( 上下文 )接口，作为 ChannelPipeline 中的**节点**。代码如下：

```
// ========== Context 相关 ==========
String name();
Channel channel();
EventExecutor executor();
ChannelHandler handler();
ChannelPipeline pipeline();
boolean isRemoved(); // 是否已经移除

// ========== ByteBuf 相关 ==========    
ByteBufAllocator alloc();

// ========== ChannelInboundInvoker 相关 ==========    
@Override
ChannelHandlerContext fireChannelRegistered();
@Override
ChannelHandlerContext fireChannelUnregistered();
@Override
ChannelHandlerContext fireChannelActive();
@Override
ChannelHandlerContext fireChannelInactive();
@Override
ChannelHandlerContext fireExceptionCaught(Throwable cause);
@Override
ChannelHandlerContext fireUserEventTriggered(Object evt);
@Override
ChannelHandlerContext fireChannelRead(Object msg);
@Override
ChannelHandlerContext fireChannelReadComplete();
@Override
ChannelHandlerContext fireChannelWritabilityChanged();

// ========== ChannelOutboundInvoker 相关 ==========
@Override
ChannelHandlerContext read();
@Override
ChannelHandlerContext flush();

// ========== AttributeMap 相关 ==========
@Deprecated
@Override
<T> Attribute<T> attr(AttributeKey<T> key);
@Deprecated
@Override
<T> boolean hasAttr(AttributeKey<T> key);
```

虽然接口的方法比较多，笔者做了归类如下：

- Context 相关的接口方法。
- 继承自 ChannelInboundInvoker 的相关方法，*和 ChannelPipeline 一样*。
- 继承自 ChannelOutboundInvoker 的相关方法，*和 ChannelPipeline 一样*。
- 继承自 AttributeMap 的相关方法，实际上已经废弃( `@Deprecated` )了，不再从 ChannelHandlerContext 中获取，而是从 Channel 中获取。

ChannelHandlerContext 的类图如下：

![ChannelHandlerContext 类图](http://www.iocoder.cn/images/Netty/2018_06_01/04.png)

- 😈 类图中的 AttributeMap 和 DefaultAttributeMap 可以无视。

### 4.1 AbstractChannelHandlerContext

`io.netty.channel.AbstractChannelHandlerContext` ，实现 ChannelHandlerContext、ResourceLeakHint 接口，继承 DefaultAttributeMap 类，ChannelHandlerContext 抽象基类。

#### 4.1.1 静态属性

```
/**
 * Neither {@link ChannelHandler#handlerAdded(ChannelHandlerContext)}
 * nor {@link ChannelHandler#handlerRemoved(ChannelHandlerContext)} was called.
 */
private static final int INIT = 0; // 初始化
/**
 * {@link ChannelHandler#handlerAdded(ChannelHandlerContext)} is about to be called.
 */
private static final int ADD_PENDING = 1; // 添加准备中
/**
 * {@link ChannelHandler#handlerAdded(ChannelHandlerContext)} was called.
 */
private static final int ADD_COMPLETE = 2; // 已添加
/**
 * {@link ChannelHandler#handlerRemoved(ChannelHandlerContext)} was called.
 */
private static final int REMOVE_COMPLETE = 3; // 已移除

/**
 * {@link #handlerState} 的原子更新器
 */
private static final AtomicIntegerFieldUpdater<AbstractChannelHandlerContext> HANDLER_STATE_UPDATER = AtomicIntegerFieldUpdater.newUpdater(AbstractChannelHandlerContext.class, "handlerState");

// ========== 非静态属性 ==========

/**
 * 处理器状态
 */
private volatile int handlerState = INIT;
```

- ```
  handlerState
  ```

   

  属性(

   

  非静态

  属性，放这里主要是为了统一讲 )，处理器状态。共有

   

  4

   

  种状态。状态变迁如下图：



  - 详细解析，见 [「4.1.3 setAddComplete」](http://svip.iocoder.cn/Netty/ChannelPipeline-1-init/#)、[「4.1.4 setRemoved」](http://svip.iocoder.cn/Netty/ChannelPipeline-1-init/#)、[「4.1.5 setAddPending」](http://svip.iocoder.cn/Netty/ChannelPipeline-1-init/#) 中。

- `HANDLER_STATE_UPDATER` **静态**属性，`handlerState` 的原子更新器。

#### 4.1.2 构造方法

```
/**
 * 上一个节点
 */
volatile AbstractChannelHandlerContext next;
/**
 * 下一个节点
 */
volatile AbstractChannelHandlerContext prev;
/**
 * 是否为 inbound
 */
private final boolean inbound;
/**
 * 是否为 outbound
 */
private final boolean outbound;
/**
 * 所属 pipeline
 */
private final DefaultChannelPipeline pipeline;
/**
 * 名字
 */
private final String name;
/**
 * 是否使用有序的 EventExecutor ( {@link #executor} )，即 OrderedEventExecutor
 */
private final boolean ordered;

// Will be set to null if no child executor should be used, otherwise it will be set to the
// child executor.
/**
 * EventExecutor 对象
 */
final EventExecutor executor;
/**
 * 成功的 Promise 对象
 */
private ChannelFuture succeededFuture;

// Lazily instantiated tasks used to trigger events to a handler with different executor. 懒加载
// There is no need to make this volatile as at worse it will just create a few more instances then needed.
/**
 * 执行 Channel ReadComplete 事件的任务
 */
private Runnable invokeChannelReadCompleteTask;
/**
 * 执行 Channel Read 事件的任务
 */
private Runnable invokeReadTask;
/**
 * 执行 Channel WritableStateChanged 事件的任务
 */
private Runnable invokeChannelWritableStateChangedTask;
/**
 * 执行 flush 事件的任务
 */
private Runnable invokeFlushTask;
/**
 * 处理器状态
 */
private volatile int handlerState = INIT;

AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name,
                              boolean inbound, boolean outbound) {
    this.name = ObjectUtil.checkNotNull(name, "name");
    this.pipeline = pipeline;
    this.executor = executor;
    this.inbound = inbound;
    this.outbound = outbound;
    // Its ordered if its driven by the EventLoop or the given Executor is an instanceof OrderedEventExecutor.
    ordered = executor == null || executor instanceof OrderedEventExecutor; // <1>
}
```

- `next`、`prev` 属性，分别记录上、下一个节点。

- Handler 相关属性：

  - 在 AbstractChannelHandlerContext 抽象类中，按照我们上文的分享，应该会看到一个类型为 ChannelHandler 的处理器，但是**实际并不是这样**。而是，😈 我们下文 DefaultChannelHandlerContext、TailContext、HeadContext 见。
  - `inbound`、`outbound` 属性，分别是否为 Inbound、Outbound 处理器。
  - `name` 属性，处理器名字。
  - `handlerState` 属性，处理器状态，初始为 `INIT` 。

- ```
  executor
  ```

   

  属性，EventExecutor 对象

  - `ordered` 属性，是否使用有序的 `executor`，即 OrderedEventExecutor ，在构造方法的 `<1>` 处理的初始化。

- `pipeline` 属性，所属 DefaultChannelPipeline 对象。

#### 4.1.3 setAddComplete

`#setAddComplete()` 方法，设置 ChannelHandler 添加完成。完成后，状态有两种结果：

1. `REMOVE_COMPLETE`
2. `ADD_COMPLETE`

代码如下：

```
final void setAddComplete() {
    for (;;) {
        int oldState = handlerState;
        // Ensure we never update when the handlerState is REMOVE_COMPLETE already.
        // oldState is usually ADD_PENDING but can also be REMOVE_COMPLETE when an EventExecutor is used that is not
        // exposing ordering guarantees.
        if (oldState == REMOVE_COMPLETE || HANDLER_STATE_UPDATER.compareAndSet(this, oldState, ADD_COMPLETE)) {
            return;
        }
    }
}
```

- 循环 + CAS 保证多线程下的安全变更 `handlerState` 属性。

#### 4.1.4 setRemoved

`#setRemoved()` 方法，设置 ChannelHandler 已移除。代码如下：

```
final void setRemoved() {
    handlerState = REMOVE_COMPLETE;
}
```

#### 4.1.5 setAddPending

`#setAddPending()` 方法，设置 ChannelHandler 准备添加中。代码如下：

```
final void setAddPending() {
    boolean updated = HANDLER_STATE_UPDATER.compareAndSet(this, INIT, ADD_PENDING);
    assert updated; // This should always be true as it MUST be called before setAddComplete() or setRemoved().
}
```

- 当且仅当 `INIT` 可修改为 `ADD_PENDING` 。理论来说，这是一个绝对会成功的操作，原因见英文注释。

#### 4.1.6 其他方法

AbstractChannelHandlerContext 中的其他方法，详细解析，见后续的文章。

### 4.2 HeadContext

HeadContext ，实现 ChannelOutboundHandler、ChannelInboundHandler 接口，继承 AbstractChannelHandlerContext 抽象类，**pipe 头节点** Context 实现类。

> HeadContext 是 DefaultChannelPipeline 的内部类。

#### 4.2.1 构造方法

```
private final Unsafe unsafe;

HeadContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, HEAD_NAME, false, true); // <1>
    unsafe = pipeline.channel().unsafe(); // <2>
    setAddComplete(); // <3>
}
```

- `<1>` 处，调用父 AbstractChannelHandlerContext 的构造方法，设置 `inbound = false`、`outbound = true` 。

- `<2>` 处，使用 Channel 的 Unsafe 作为 `unsafe` 属性。HeadContext 实现 ChannelOutboundHandler 接口的方法，都会调用 Unsafe 对应的方法。代码如下：

  ```
  @Override
  public void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception {
      unsafe.bind(localAddress, promise);
  }
  
  @Override
  public void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) throws Exception {
      unsafe.connect(remoteAddress, localAddress, promise);
  }
  
  @Override
  public void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
      unsafe.disconnect(promise);
  }
  
  @Override
  public void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
      unsafe.close(promise);
  }
  
  @Override
  public void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
      unsafe.deregister(promise);
  }
  
  @Override
  public void read(ChannelHandlerContext ctx) {
      unsafe.beginRead();
  }
  
  @Override
  public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
      unsafe.write(msg, promise);
  }
  
  @Override
  public void flush(ChannelHandlerContext ctx) throws Exception {
      unsafe.flush();
  }
  ```

  - 这也就是为什么设置 `outbound = true` 的原因。

- `<3>` 处，调用 `#setAddComplete()` 方法，设置 ChannelHandler 添加完成。此时，`handlerStatus` 会变成 `ADD_COMPLETE` 状态。

#### 4.2.2 handler

`#handler()` 方法，返回自己作为 Context 的 **ChannelHandler** 。代码如下：

```
@Override
public ChannelHandler handler() {
    return this;
}
```

- 因为 HeadContext ，实现 ChannelOutboundHandler、ChannelInboundHandler 接口，而它们本身就是 ChannelHandler 。

#### 4.2.3 其他方法

HeadContext 中的其他方法，详细解析，见后续的文章。

### 4.3 TailContext

TailContext ，实现 ChannelInboundHandler 接口，继承 AbstractChannelHandlerContext 抽象类，**pipe 尾节点**Context 实现类。

> TailContext 是 DefaultChannelPipeline 的内部类。

#### 4.3.1 构造方法

```
TailContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, TAIL_NAME, true, false); // <1>
    setAddComplete(); // <2>
}
```

- `<1>` 处，调用父 AbstractChannelHandlerContext 的构造方法，设置 `inbound = true`、`outbound = false` ，和 HeadContext **相反**。
- `<2>` 处，调用 `#setAddComplete()` 方法，设置 ChannelHandler 添加完成。此时，`handlerStatus` 会变成 `ADD_COMPLETE` 状态。

#### 4.3.2 handler

`#handler()` 方法，返回自己作为 Context 的 **ChannelHandler** 。代码如下：

```
@Override
public ChannelHandler handler() {
    return this;
}
```

- 因为 HeadContext ，实现 ChannelInboundHandler 接口，而它们本身就是 ChannelHandler 。

#### 4.3.3 其他方法

TailContext 中的其他方法，详细解析，见后续的文章。

### 4.4 DefaultChannelHandlerContext

`io.netty.channel.DefaultChannelHandlerContext` ，实现 AbstractChannelHandlerContext 抽象类。代码如下：

```
final class DefaultChannelHandlerContext extends AbstractChannelHandlerContext {

    private final ChannelHandler handler;

    DefaultChannelHandlerContext(
            DefaultChannelPipeline pipeline, EventExecutor executor, String name, ChannelHandler handler) {
        super(pipeline, executor, name, isInbound(handler), isOutbound(handler)); // <1>
        if (handler == null) {
            throw new NullPointerException("handler");
        }
        this.handler = handler; // <2>
    }

    @Override
    public ChannelHandler handler() {
        return handler;
    }

    private static boolean isInbound(ChannelHandler handler) {
        return handler instanceof ChannelInboundHandler;
    }

    private static boolean isOutbound(ChannelHandler handler) {
        return handler instanceof ChannelOutboundHandler;
    }

}
```

- 不同于 HeadContext、TailContext，它们自身就是一个 Context 的同时，也是一个 ChannelHandler 。而 DefaultChannelHandlerContext 是**内嵌** 一个 ChannelHandler 对象，即 `handler` 。这个属性通过构造方法传入，在 `<2>` 处进行赋值。
- `<1>` 处，调用父 AbstractChannelHandlerContext 的构造方法，通过判断传入的 `handler` 是否为 ChannelInboundHandler 和 ChannelOutboundHandler 来分别判断是否为 `inbound` 和 `outbound` 。

## 666. 彩蛋

推荐阅读如下文章：

- 闪电侠 [《netty 源码分析之 pipeline(一)》](https://www.jianshu.com/p/6efa9c5fa702)
- 永顺 [《Netty 源码分析之 二 贯穿Netty 的大动脉 ── ChannelPipeline (一)》](https://segmentfault.com/a/1190000007308934)
- 占小狼 [《Netty 源码分析之 ChannelPipeline》](https://www.jianshu.com/p/3876874306d5)

# 精尽 Netty 源码解析 —— ChannelPipeline（二）之添加 ChannelHandler

2018-06-04

## 1. 概述

本文我们来分享，**添加** ChannelHandler 到 pipeline 中的代码具体实现。

在 [《精尽 Netty 源码解析 —— ChannelPipeline（一）之初始化》](http://svip.iocoder.cn/Netty/ChannelPipeline-1-init) 中，我们看到 ChannelPipeline 定义了一大堆**添加** ChannelHandler 的接口方法：

```
ChannelPipeline addFirst(String name, ChannelHandler handler);
ChannelPipeline addFirst(EventExecutorGroup group, String name, ChannelHandler handler);
ChannelPipeline addLast(String name, ChannelHandler handler);
ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler);
ChannelPipeline addBefore(String baseName, String name, ChannelHandler handler);
ChannelPipeline addBefore(EventExecutorGroup group, String baseName, String name, ChannelHandler handler);
ChannelPipeline addAfter(String baseName, String name, ChannelHandler handler);
ChannelPipeline addAfter(EventExecutorGroup group, String baseName, String name, ChannelHandler handler);
ChannelPipeline addFirst(ChannelHandler... handlers);
ChannelPipeline addFirst(EventExecutorGroup group, ChannelHandler... handlers);
ChannelPipeline addLast(ChannelHandler... handlers);
ChannelPipeline addLast(EventExecutorGroup group, ChannelHandler... handlers);
```

- 考虑到实际当中，我们使用 `#addLast(ChannelHandler... handlers)` 方法较多，所以本文只分享这个方法的具体实现。

## 2. addLast

`#addLast(ChannelHandler... handlers)` 方法，添加任意数量的 ChannelHandler 对象。代码如下：

```
@Override
public final ChannelPipeline addLast(ChannelHandler... handlers) {
    return addLast(null, handlers);
}

@Override
public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
    if (handlers == null) {
        throw new NullPointerException("handlers");
    }

    for (ChannelHandler h: handlers) {
        if (h == null) {
            break;
        }
        addLast(executor, null, h); // <1>
    }

    return this;
}
```

- `<1>` 处，调用 `#addLast(EventExecutorGroup group, String name, ChannelHandler handler)` 方法，添加一个 ChannelHandler 对象到 pipeline 中。

`#addLast(EventExecutorGroup group, String name, ChannelHandler handler)` 方法，代码如下：

```
 1: @Override
 2: @SuppressWarnings("Duplicates")
 3: public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
 4:     final AbstractChannelHandlerContext newCtx;
 5:     synchronized (this) { // 同步，为了防止多线程并发操作 pipeline 底层的双向链表
 6:         // 检查是否有重复 handler
 7:         checkMultiplicity(handler);
 8: 
 9:         // 创建节点名
10:         // 创建节点
11:         newCtx = newContext(group, filterName(name, handler), handler);
12: 
13:         // 添加节点
14:         addLast0(newCtx);
15: 
16:         // <1> pipeline 暂未注册，添加回调。再注册完成后，执行回调。详细解析，见 {@link #invokeHandlerAddedIfNeeded} 方法。
17:         // If the registered is false it means that the channel was not registered on an eventloop yet.
18:         // In this case we add the context to the pipeline and add a task that will call
19:         // ChannelHandler.handlerAdded(...) once the channel is registered.
20:         if (!registered) {
21:             // 设置 AbstractChannelHandlerContext 准备添加中
22:             newCtx.setAddPending();
23:             // 添加 PendingHandlerCallback 回调
24:             callHandlerCallbackLater(newCtx, true);
25:             return this;
26:         }
27: 
28:         // <2> 不在 EventLoop 的线程中，提交 EventLoop 中，执行回调用户方法
29:         EventExecutor executor = newCtx.executor();
30:         if (!executor.inEventLoop()) {
31:             // 设置 AbstractChannelHandlerContext 准备添加中
32:             newCtx.setAddPending();
33:             // 提交 EventLoop 中，执行回调 ChannelHandler added 事件
34:             executor.execute(new Runnable() {
35:                 @Override
36:                 public void run() {
37:                     callHandlerAdded0(newCtx);
38:                 }
39:             });
40:             return this;
41:         }
42:     }
43: 
44:     // <3> 回调 ChannelHandler added 事件
45:     callHandlerAdded0(newCtx);
46:     return this;
47: }
```

- 第 5 行：`synchronized` 同步，为了防止多线程并发操作 pipeline 底层的双向链表。
- 第 7 行：调用 `#checkMultiplicity(ChannelHandler)` 方法，校验是否重复的 ChannelHandler 。详细解析，见 [「3. checkMultiplicity」](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler/#) 。
- 第 11 行：调用 `#filterName(String name, ChannelHandler handler)` 方法，获得 ChannelHandler 的名字。详细解析，见 [「4. filterName」](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler/#) 。
- 第 11 行：调用 `#newContext(EventExecutorGroup group, String name, ChannelHandler handler)` 方法，创建 **DefaultChannelHandlerContext** 节点。详细解析，见 [「5. newContext」](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler/#) 。
- 第 14 行：`#addLast0(AbstractChannelHandlerContext newCtx)` 方法，添加到最后一个节点。详细解析，见 [「6. addLast0」](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler/#) 。
- ========== 后续分成 3 种情况 ==========
- `<1>`
- 第 20 行：Channel 并未注册。这种情况，发生于 ServerBootstrap 启动的过程中。在 `ServerBootstrap#init(Channel channel)` 方法中，会添加 ChannelInitializer 对象到 pipeline 中，恰好此时 Channel 并未注册。
- 第 22 行：调用 `AbstractChannelHandlerContext#setAddPending()` 方法，设置 AbstractChannelHandlerContext **准备添加中**。
- 第 24 行：调用 `#callHandlerCallbackLater(AbstractChannelHandlerContext, added)` 方法，添加 PendingHandlerAddedTask 回调。在 Channel 注册完成后，执行该回调。详细解析，见 [「8. PendingHandlerCallback」](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler/#) 。
- `<2>`
- 第 30 行：不在 EventLoop 的线程中。
- 第 32 行：调用 `AbstractChannelHandlerContext#setAddPending()` 方法，设置 AbstractChannelHandlerContext **准备添加中**。
- 第 34 至 39 行：提交 EventLoop 中，调用 `#callHandlerAdded0(AbstractChannelHandlerContext)` 方法，执行回调 ChannelHandler 添加完成( added )事件。详细解析，见 [「7. callHandlerAdded0」](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler/#) 。
- `<3>`
- 这种情况，是 `<2>` 在 EventLoop 的线程中的版本。也因为此，已经确认在 EventLoop 的线程中，所以不需要在 `synchronized` 中。
- 第 45 行：和【第 37 行】的代码一样，调用 `#callHandlerAdded0(AbstractChannelHandlerContext)` 方法，执行回调 ChannelHandler 添加完成( added )事件。

## 3. checkMultiplicity

`#checkMultiplicity(ChannelHandler handler)` 方法，校验是否重复的 ChannelHandler 。代码如下：

```
private static void checkMultiplicity(ChannelHandler handler) {
    if (handler instanceof ChannelHandlerAdapter) {
        ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
        // 若已经添加，并且未使用 @Sharable 注解，则抛出异常
        if (!h.isSharable() && h.added) {
            throw new ChannelPipelineException(
                    h.getClass().getName() +
                    " is not a @Sharable handler, so can't be added or removed multiple times.");
        }
        // 标记已经添加
        h.added = true;
    }
}
```

- 在 pipeline 中，一个创建的 ChannelHandler 对象，如果不使用 Netty `@Sharable` 注解，则只能添加到一个 Channel 的 pipeline 中。所以，如果我们想要重用一个 ChannelHandler 对象( 例如在 Spring 环境中 )，则必须给这个 ChannelHandler 添加 `@Sharable` 注解。

例如，在 Dubbo 的 `com.alibaba.dubbo.remoting.transport.netty.NettyHandler` 处理器，它就使用了 `@Sharable` 注解。

## 4. filterName

`#filterName(String name, ChannelHandler handler)` 方法，获得 ChannelHandler 的名字。代码如下：

```
private String filterName(String name, ChannelHandler handler) {
    if (name == null) { // <1>
        return generateName(handler);
    }
    checkDuplicateName(name); // <2>
    return name;
}
```

- `<1>` 处，若**未**传入默认的名字 `name` ，则调用 `#generateName(ChannelHandler)` 方法，根据 ChannelHandler 生成一个**唯一**的名字。详细解析，见 [「4.1 generateName」](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler/#) 。
- `<2>` 处，若**已**传入默认的名字 `name` ，则调用 `#checkDuplicateName(String name)` 方法，校验名字唯一。详细解析，见 [「4.2 checkDuplicateName」](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler/#) 。

### 4.1 generateName

`#generateName(ChannelHandler)` 方法，根据 ChannelHandler 生成一个**唯一**名字。代码如下：

```
 1: private String generateName(ChannelHandler handler) {
 2:     // 从缓存中查询，是否已经生成默认名字
 3:     Map<Class<?>, String> cache = nameCaches.get();
 4:     Class<?> handlerType = handler.getClass();
 5:     String name = cache.get(handlerType);
 6:     // 若未生成过，进行生成
 7:     if (name == null) {
 8:         name = generateName0(handlerType);
 9:         cache.put(handlerType, name);
10:     }
11: 
12:     // 判断是否存在相同名字的节点
13:     // It's not very likely for a user to put more than one handler of the same type, but make sure to avoid
14:     // any name conflicts.  Note that we don't cache the names generated here.
15:     if (context0(name) != null) {
16:         // 若存在，则使用基础名字 + 编号，循环生成，直到一个是唯一的
17:         String baseName = name.substring(0, name.length() - 1); // Strip the trailing '0'.
18:         for (int i = 1;; i ++) {
19:             String newName = baseName + i;
20:             if (context0(newName) == null) { // // 判断是否存在相同名字的节点
21:                 name = newName;
22:                 break;
23:             }
24:         }
25:     }
26:     return name;
27: }
```

- 第 2 至 5 行：从缓存

   

  ```
  nameCaches
  ```

   

  中，查询是否已经生成

  默认

  名字。

  - 若未生成过，调用 `#generateName0(ChannelHandler)` 方法，进行生成。而后，添加到缓存 `nameCaches` 中。

- 第 15 行：调用 `#context0(String name)` 方法，判断是否存在相同名字的节点。代码如下：

  ```
  private AbstractChannelHandlerContext context0(String name) {
      AbstractChannelHandlerContext context = head.next;
      // 顺序向下遍历节点，判断是否有指定名字的节点。如果有，则返回该节点。
      while (context != tail) {
          if (context.name().equals(name)) {
              return context;
          }
          context = context.next;
      }
      return null;
  }
  ```

  - 顺序向下遍历节点，判断是否有指定名字的节点。如果有，则返回该节点。

- 第 15 至 25 行：若存在相同名字的节点，则使用**基础**名字 + 编号，循环生成，直到一个名字是**唯一**的，然后结束循环。

### 4.2 checkDuplicateName

`#checkDuplicateName(String name)` 方法，校验名字唯一。代码如下：

```
private void checkDuplicateName(String name) {
    if (context0(name) != null) {
        throw new IllegalArgumentException("Duplicate handler name: " + name);
    }
}
```

- 通过调用 `#context0(String name)` 方法，获得指定名字的节点。若存在节点，意味着**不唯一**，抛出 IllegalArgumentException 异常。

## 5. newContext

`#newContext(EventExecutorGroup group, String name, ChannelHandler handler)` 方法，创建 **DefaultChannelHandlerContext** 节点。而这个节点，**内嵌**传入的 ChannelHandler 参数。代码如下：

```
private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
    return new DefaultChannelHandlerContext(this, childExecutor(group) /** <1> **/, name, handler);
}
```

- `<1>` 处，调用 `#childExecutor(EventExecutorGroup group)` 方法，创建**子**执行器。代码如下：

  ```
  private EventExecutor childExecutor(EventExecutorGroup group) {
      // <1> 不创建子执行器
      if (group == null) {
          return null;
      }
      // <2> 根据配置项 SINGLE_EVENTEXECUTOR_PER_GROUP ，每个 Channel 从 EventExecutorGroup 获得不同 EventExecutor 执行器
      Boolean pinEventExecutor = channel.config().getOption(ChannelOption.SINGLE_EVENTEXECUTOR_PER_GROUP);
      if (pinEventExecutor != null && !pinEventExecutor) {
          return group.next();
      }
      // <3> 通过 childExecutors 缓存实现，一个 Channel 从 EventExecutorGroup 获得相同 EventExecutor 执行器
      Map<EventExecutorGroup, EventExecutor> childExecutors = this.childExecutors;
      if (childExecutors == null) {
          // Use size of 4 as most people only use one extra EventExecutor.
          childExecutors = this.childExecutors = new IdentityHashMap<EventExecutorGroup, EventExecutor>(4);
      }
      // Pin one of the child executors once and remember it so that the same child executor
      // is used to fire events for the same channel.
      EventExecutor childExecutor = childExecutors.get(group);
      // 缓存不存在，进行 从 EventExecutorGroup 获得 EventExecutor 执行器
      if (childExecutor == null) {
          childExecutor = group.next();
          childExecutors.put(group, childExecutor); // 进行缓存
      }
      return childExecutor;
  }
  ```

  - 一共有三种情况：
    - `<1>` ，当**不传入** EventExecutorGroup 时，不创建**子**执行器。即，使用 Channel 所注册的 EventLoop 作为执行器。**对于我们日常使用，基本完全都是这种情况**。所以，下面两种情况，胖友不理解也是没关系的。
    - `<2>` ，根据配置项 `ChannelOption.SINGLE_EVENTEXECUTOR_PER_GROUP` ，每个 Channel 从 EventExecutorGroup 获得**不同** EventExecutor 执行器。
    - `<3>` ，通过 `childExecutors` 缓存实现，每个 Channel 从 EventExecutorGroup 获得**相同** EventExecutor 执行器。是否获得相同的 EventExecutor 执行器，这就是 `<2>`、`<3>` 的不同。

- **注意**，创建的是 DefaultChannelHandlerContext 对象。

## 6. addLast0

`#addLast0(AbstractChannelHandlerContext newCtx)` 方法，添加到最后一个节点。**注意**，实际上，是添加到 `tail` 节点**之前**。代码如下：

```
private void addLast0(AbstractChannelHandlerContext newCtx) {
    // 获得 tail 节点的前一个节点
    AbstractChannelHandlerContext prev = tail.prev;
    // 新节点，指向 prev 和 tail 节点
    newCtx.prev = prev; // <1>
    newCtx.next = tail; // <2>
    // 在 prev 和 tail ，指向新节点
    prev.next = newCtx; // <3>
    tail.prev = newCtx; // <4>
}
```

> FROM 闪电侠 [《Netty 源码分析之pipeline(一)》](https://www.jianshu.com/p/6efa9c5fa702)
>
> - 用下面这幅图可见简单的表示这段过程，说白了，其实就是一个双向链表的插入操作：
>
>   ![添加节点过程](http://www.iocoder.cn/images/Netty/2018_06_04/01.png)
>
> - 操作完毕，该节点就加入到 pipeline 中：
>
>   ![添加节点之后](http://www.iocoder.cn/images/Netty/2018_06_04/02.png)

## 7. callHandlerAdded0

`#callHandlerAdded0(AbstractChannelHandlerContext)` 方法，执行回调 ChannelHandler 添加完成( added )事件。代码如下：

```
 1: private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
 2:     try {
 3:         // We must call setAddComplete before calling handlerAdded. Otherwise if the handlerAdded method generates
 4:         // any pipeline events ctx.handler() will miss them because the state will not allow it.
 5:         // 设置 AbstractChannelHandlerContext 已添加
 6:         ctx.setAddComplete();
 7:         // 回调 ChannelHandler 添加完成( added )事件
 8:         ctx.handler().handlerAdded(ctx);
 9:     } catch (Throwable t) {
10:         // 发生异常，移除该节点
11:         boolean removed = false;
12:         try {
13:             remove0(ctx); // 移除
14:             try {
15:                 ctx.handler().handlerRemoved(ctx); // 回调 ChannelHandler 移除完成( removed )事件
16:             } finally {
17:                 ctx.setRemoved(); // 标记节点已移除
18:             }
19:             removed = true; // 标记移除成功
20:         } catch (Throwable t2) {
21:             if (logger.isWarnEnabled()) {
22:                 logger.warn("Failed to remove a handler: " + ctx.name(), t2);
23:             }
24:         }
25: 
26:         // 触发异常的传播
27:         if (removed) {
28:             fireExceptionCaught(new ChannelPipelineException(
29:                     ctx.handler().getClass().getName() +
30:                     ".handlerAdded() has thrown an exception; removed.", t));
31:         } else {
32:             fireExceptionCaught(new ChannelPipelineException(
33:                     ctx.handler().getClass().getName() +
34:                     ".handlerAdded() has thrown an exception; also failed to remove.", t));
35:         }
36:     }
37: }
```

- 第 6 行：调用 `AbstractChannelHandlerContext#setAddComplete()` 方法，设置 AbstractChannelHandlerContext 已添加。

- 第 8 行：调用 `ChannelHandler#handlerAdded(AbstractChannelHandlerContext)` 方法，回调 ChannelHandler 添加完成( added )事件。一般来说，通过这个方法，来初始化 ChannelHandler 。**注意**，因为这个方法的执行在 EventLoop 的线程中，所以要尽量避免执行时间过长。

- 第 9 行：发生异常。

  - 第 10 至 24 行：移除该节点( ChannelHandler )。详细解析，见

     

    《精尽 Netty 源码解析 —— ChannelPipeline（三）之移除 ChannelHandler》

     

    。

    - 😈 所以，`ChannelHandler#handlerAdded(AbstractChannelHandlerContext)` 方法的执行**异常**时，将被移除。

  - 第 26 至 35 行：触发异常的传播。详细解析，见 [《精尽 Netty 源码解析 —— ChannelPipeline（六）之异常事件的传播》](http://svip.iocoder.cn/Netty/ChannelPipeline-6-exception) 。

## 8. PendingHandlerCallback

PendingHandlerCallback ，实现 Runnable 接口，等待添加 ChannelHandler 回调抽象类。代码如下：

> PendingHandlerCallback 是 DefaultChannelPipeline 的内部静态类。

```
private abstract static class PendingHandlerCallback implements Runnable {

    /**
     * AbstractChannelHandlerContext 节点
     */
    final AbstractChannelHandlerContext ctx;
    /**
     * 下一个回调 PendingHandlerCallback 对象
     */
    PendingHandlerCallback next;

    PendingHandlerCallback(AbstractChannelHandlerContext ctx) {
        this.ctx = ctx;
    }

    /**
     * 执行方法
     */
    abstract void execute();

}
```

- 通过 `ctx` 和 `next` 字段，形成**回调链**。
- `#execute()` 抽象方法，通过实现它，执行回调逻辑。

------

**为什么会有 PendingHandlerCallback 呢**？

因为 ChannelHandler 添加到 pipeline 中，会触发 ChannelHandler 的添加完成( added )事件，并且该事件需要在 Channel 所属的 EventLoop 中执行。

但是 Channel 并未注册在 EventLoop 上时，需要暂时将“触发 ChannelHandler 的添加完成( added )事件”的逻辑，作为一个 PendingHandlerCallback 进行“缓存”。在 Channel 注册到 EventLoop 上时，进行回调执行。

------

PendingHandlerCallback 有两个实现类：

- PendingHandlerAddedTask
- PendingHandlerRemovedTask

本文只分享 PendingHandlerAddedTask 的代码实现。

### 8.1 PendingHandlerAddedTask

PendingHandlerAddedTask 实现 PendingHandlerCallback 抽象类，用于回调添加 ChannelHandler 节点。代码如下：

```
private final class PendingHandlerAddedTask extends PendingHandlerCallback {

    PendingHandlerAddedTask(AbstractChannelHandlerContext ctx) {
        super(ctx);
    }

    @Override
    public void run() {
        callHandlerAdded0(ctx);
    }

    @Override
    void execute() {
        EventExecutor executor = ctx.executor();
        // 在 EventLoop 的线程中，回调 ChannelHandler added 事件
        if (executor.inEventLoop()) {
            callHandlerAdded0(ctx);
        } else {
            // 提交 EventLoop 中，执行回调 ChannelHandler added 事件
            try {
                executor.execute(this); // <1>
            } catch (RejectedExecutionException e) {
                if (logger.isWarnEnabled()) {
                    logger.warn(
                            "Can't invoke handlerAdded() as the EventExecutor {} rejected it, removing handler {}.",
                            executor, ctx.name(), e);
                }
                // 发生异常，进行移除
                remove0(ctx);
                // 标记 AbstractChannelHandlerContext 为已移除
                ctx.setRemoved();
            }
        }
    }
    
}
```

- 在 `#execute()` 实现方法中，我们可以看到，和 `#addLast(EventExecutorGroup group, String name, ChannelHandler handler)` 方法的【第 28 至 45 行】的代码比较类似，目的是，在 EventLoop 的线程中，执行 `#callHandlerAdded0(AbstractChannelHandlerContext)` 方法，回调 ChannelHandler 添加完成( added )事件。
- `<1>` 处，为什么 PendingHandlerAddedTask 可以直接提交到 EventLoop 中呢？因为 PendingHandlerAddedTask 是个 Runnable ，这也就是为什么 PendingHandlerCallback 实现 Runnable 接口的原因。

> 老艿艿：下面开始分享的方法，属于 DefaultChannelPipeline 类。

### 8.2 callHandlerCallbackLater

`#callHandlerCallbackLater(AbstractChannelHandlerContext ctx, boolean added)` 方法，添加 PendingHandlerCallback 回调。代码如下：

```
/**
 * This is the head of a linked list that is processed by {@link #callHandlerAddedForAllHandlers()} and so process
 * all the pending {@link #callHandlerAdded0(AbstractChannelHandlerContext)}.
 *
 * We only keep the head because it is expected that the list is used infrequently and its size is small.
 * Thus full iterations to do insertions is assumed to be a good compromised to saving memory and tail management
 * complexity.
 *
 * 准备添加 ChannelHandler 的回调
 *
 * @see #callHandlerCallbackLater(AbstractChannelHandlerContext, boolean)
 */
private PendingHandlerCallback pendingHandlerCallbackHead;
    
  1: private void callHandlerCallbackLater(AbstractChannelHandlerContext ctx, boolean added) {
  2:     assert !registered;
  3: 
  4:     // 创建 PendingHandlerCallback 对象
  5:     PendingHandlerCallback task = added ? new PendingHandlerAddedTask(ctx) : new PendingHandlerRemovedTask(ctx);
  6:     PendingHandlerCallback pending = pendingHandlerCallbackHead;
  7:     // 若原 pendingHandlerCallbackHead 不存在，则赋值给它
  8:     if (pending == null) {
  9:         pendingHandlerCallbackHead = task;
 10:     // 若原 pendingHandlerCallbackHead 已存在，则最后一个回调指向新创建的回调
 11:     } else {
 12:         // Find the tail of the linked-list.
 13:         while (pending.next != null) {
 14:             pending = pending.next;
 15:         }
 16:         pending.next = task;
 17:     }
 18: }
```

- `added` 方法参数，表示是否是添加 ChannelHandler 的回调。所以在【第 5 行】的代码，根据 `added` 是否为 `true` ，创建 PendingHandlerAddedTask 或 PendingHandlerRemovedTask 对象。在本文中，当然创建的是 PendingHandlerAddedTask 。
- 第 7 至 17 行：将创建的 PendingHandlerCallback 对象，“添加”到 `pendingHandlerCallbackHead` 中。

### 8.3 invokeHandlerAddedIfNeeded

`#invokeHandlerAddedIfNeeded()` 方法，执行**在 PendingHandlerCallback 中**的 ChannelHandler 添加完成( added )事件。它被两个方法所调用：

- `AbstractUnsafe#register0(ChannelPromise promise)` 方法

  - 原因是：

    ```
    // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
    // user may already fire events through the pipeline in the ChannelFutureListener.
    ```

    - 例如 ServerBootstrap 通过 ChannelInitializer 注册自定义的 ChannelHandler 到 pipeline 上的情况。

  - 调用栈如下图：![register0](http://www.iocoder.cn/images/Netty/2018_06_04/03.png)

- ```
  HeadContext#channelRegistered(ChannelHandlerContext ctx)
  ```

   

  方法。

  - 笔者调试下来，对于 Netty NIO Server 和 NIO Client 貌似没啥作用，因为已经在 `AbstractUnsafe#register0(ChannelPromise promise)` 中触发。胖友也可以自己调试下。
  - 调用栈如下图：![channelRegistered](http://www.iocoder.cn/images/Netty/2018_06_04/04.png)

------

`#invokeHandlerAddedIfNeeded()` 方法，代码如下：

```
/**
 * 是否首次注册
 *
 * {@link #invokeHandlerAddedIfNeeded()}
 */
private boolean firstRegistration = true;

final void invokeHandlerAddedIfNeeded() {
    assert channel.eventLoop().inEventLoop(); // 必须在 EventLoop 的线程中
    // 仅有首次注册有效 <1>
    if (firstRegistration) {
        // 标记非首次注册
        firstRegistration = false;
        
        // 执行在 PendingHandlerCallback 中的 ChannelHandler 添加完成( added )事件 // <2>
        // We are now registered to the EventLoop. It's time to call the callbacks for the ChannelHandlers,
        // that were added before the registration was done.
        callHandlerAddedForAllHandlers();
    }
}
```

- ```
  <1>
  ```

   

  处，仅有首次注册有效(

   

  ```
  firstRegistration = true
  ```

   

  ) 时。而后，标记

   

  ```
  firstRegistration = false
  ```

   

  。

  - 这也就是笔者为什么说，`HeadContext#channelRegistered(ChannelHandlerContext ctx)` 方法对这个方法的调用，是没有效果的。

- `<2>` 处，调用 `#callHandlerAddedForAllHandlers()` 方法，执行**在 PendingHandlerCallback 中**的 ChannelHandler 添加完成( added )事件。代码如下：

  ```
   1: private void callHandlerAddedForAllHandlers() {
   2:     final PendingHandlerCallback pendingHandlerCallbackHead;
   3:     // 获得 pendingHandlerCallbackHead
   4:     synchronized (this) {
   5:         assert !registered;
   6: 
   7:         // This Channel itself was registered.
   8:         registered = true; // 标记已注册
   9: 
  10:         pendingHandlerCallbackHead = this.pendingHandlerCallbackHead;
  11:         // Null out so it can be GC'ed.
  12:         this.pendingHandlerCallbackHead = null; // 置空，help gc
  13:     }
  14: 
  15:     // 顺序向下，执行 PendingHandlerCallback 的回调
  16:     // This must happen outside of the synchronized(...) block as otherwise handlerAdded(...) may be called while
  17:     // holding the lock and so produce a deadlock if handlerAdded(...) will try to add another handler from outside
  18:     // the EventLoop.
  19:     PendingHandlerCallback task = pendingHandlerCallbackHead;
  20:     while (task != null) {
  21:         task.execute();
  22:         task = task.next;
  23:     }
  24: }
  ```

  - 第 3 至 13 行：获得

     

    ```
    pendingHandlerCallbackHead
    ```

     

    变量。

    - 第 8 行：标记 `registered = true` ，表示已注册。
    - 第 10 至 12 行：置空对象的 `pendingHandlerCallbackHead` 属性，help GC 。
    - 使用 `synchronized` 的原因，和 `#addLast(EventExecutorGroup group, String name, ChannelHandler handler)` 的【第 16 至 26 行】的代码需要对 `pendingHandlerCallbackHead` 互斥，避免并发修改的问题。

  - 第 15 至 23 行：顺序循环向下，调用

     

    ```
    PendingHandlerCallback#execute()
    ```

     

    方法，执行 PendingHandlerCallback 的回调，从而将 ChannelHandler 添加到 pipeline 中。

    - 这里不适用 `synchronized` 的原因，看英文注释哈。

## 666. 彩蛋

**添加** ChannelHandler 到 pipeline 中的代码，大部分的比较简单。比较复杂的可能是，[「8. PendingHandlerCallback」](http://svip.iocoder.cn/Netty/Pipeline-2-add-channel-handler/#) 中，调用的过程涉及**回调**，所以理解上稍微可能困难。胖友可以多多调试进行解决噢。

推荐阅读文章：

- 闪电侠 [《Netty 源码分析之 pipeline(一)》](https://www.jianshu.com/p/6efa9c5fa702)
- Hypercube [《自顶向下深入分析 Netty（七）–ChannelPipeline 源码实现》](https://www.jianshu.com/p/0e15165714fc)

# 精尽 Netty 源码解析 —— ChannelPipeline（三）之移除 ChannelHandler

2018-06-07

## 1. 概述

本文我们来分享，从 pipeline 中**移除** ChannelHandler 的代码具体实现。

在 [《精尽 Netty 源码解析 —— ChannelPipeline（一）之初始化》](http://svip.iocoder.cn/Netty/ChannelPipeline-1-init) 中，我们看到 ChannelPipeline 定义了一大堆**移除** ChannelHandler 的接口方法：

```
ChannelPipeline remove(ChannelHandler handler);
ChannelHandler remove(String name);
<T extends ChannelHandler> T remove(Class<T> handlerType);
ChannelHandler removeFirst();
ChannelHandler removeLast();
```

- 本文仅分享 `#remove(ChannelHandler handler)` 方法，从 pipeline **移除**指定的 ChannelHandler 对象。

## 2. remove

`#remove(ChannelHandler handler)` 方法，从 pipeline **移除**指定的 ChannelHandler 对象。代码如下：

```
@Override
public final ChannelPipeline remove(ChannelHandler handler) {
    remove(getContextOrDie(handler));
    return this;
}
```

- 调用 `#getContextOrDie(ChannelHandler handler)` 方法，获得对应的 AbstractChannelHandlerContext 节点。代码如下：

  ```
  private AbstractChannelHandlerContext getContextOrDie(ChannelHandler handler) {
      AbstractChannelHandlerContext ctx = (AbstractChannelHandlerContext) context(handler);
      if (ctx == null) { // die
          throw new NoSuchElementException(handler.getClass().getName());
      } else {
          return ctx;
      }
  }
  
  @Override
  public final ChannelHandlerContext context(ChannelHandler handler) {
      if (handler == null) {
          throw new NullPointerException("handler");
      }
      AbstractChannelHandlerContext ctx = head.next;
      // 循环，获得指定 ChannelHandler 对象的节点
      for (;;) {
          if (ctx == null) {
              return null;
          }
          if (ctx.handler() == handler) { // ChannelHandler 相等
              return ctx;
          }
          ctx = ctx.next;
      }
  }
  ```

  - 方法使用 Die 的原因是，获得不到节点的情况下，抛出 NoSuchElementException 异常。

- 调用 `#remove(AbstractChannelHandlerContext ctx)` 方法，移除指定 AbstractChannelHandlerContext 节点。

------

`#remove(AbstractChannelHandlerContext ctx)` 方法，移除指定 AbstractChannelHandlerContext 节点。代码如下：

> 代码的整体结构，和 `#addLast(EventExecutorGroup group, String name, ChannelHandler handler)` 方法是**一致**的。

```
 1: private AbstractChannelHandlerContext remove(final AbstractChannelHandlerContext ctx) {
 2:     assert ctx != head && ctx != tail;
 3: 
 4:     synchronized (this) { // 同步，为了防止多线程并发操作 pipeline 底层的双向链表
 5:         // 移除节点
 6:         remove0(ctx);
 7: 
 8:         // pipeline 暂未注册，添加回调。再注册完成后，执行回调。详细解析，见 {@link #callHandlerCallbackLater} 方法。
 9:         // If the registered is false it means that the channel was not registered on an eventloop yet.
10:         // In this case we remove the context from the pipeline and add a task that will call
11:         // ChannelHandler.handlerRemoved(...) once the channel is registered.
12:         if (!registered) {
13:             callHandlerCallbackLater(ctx, false);
14:             return ctx;
15:         }
16: 
17:         // 不在 EventLoop 的线程中，提交 EventLoop 中，执行回调用户方法
18:         EventExecutor executor = ctx.executor();
19:         if (!executor.inEventLoop()) {
20:             // 提交 EventLoop 中，执行回调 ChannelHandler removed 事件
21:             executor.execute(new Runnable() {
22:                 @Override
23:                 public void run() {
24:                     callHandlerRemoved0(ctx);
25:                 }
26:             });
27:             return ctx;
28:         }
29:     }
30: 
31:     // 回调 ChannelHandler removed 事件
32:     callHandlerRemoved0(ctx);
33:     return ctx;
34: }
```

- 第 4 行：`synchronized` 同步，为了防止多线程并发操作 pipeline 底层的双向链表。
- 第 6 行：调用 `#remove0(AbstractChannelHandlerContext ctx)` 方法，从 pipeline **移除**指定的 AbstractChannelHandlerContext 节点。详细解析，见 [「3. remove0」](http://svip.iocoder.cn/Netty/ChannelPipeline-3-remove-channel-handler/#) 。
- ========== 后续分成 3 种情况 ==========
- `<1>`
- 第 12 行：Channel 并未注册。
- 第 13 行：调用 `#callHandlerCallbackLater(AbstractChannelHandlerContext, added)` 方法，添加 PendingHandlerRemovedTask 回调。在 Channel 注册完成后，执行该回调。详细解析，见 [「8. PendingHandlerCallback」](http://svip.iocoder.cn/Netty/ChannelPipeline-3-remove-channel-handler/#) 。
- `<2>`
- 第 19 行：不在 EventLoop 的线程中。
- 第 20 至 26 行：提交 EventLoop 中，调用 `#callHandlerRemoved0(AbstractChannelHandlerContext)` 方法，执行回调 ChannelHandler 移除完成( removed )事件。详细解析，见 [「4. callHandlerRemoved0」](http://svip.iocoder.cn/Netty/ChannelPipeline-3-remove-channel-handler/#) 。
- `<3>`
- 这种情况，是 `<2>` 在 EventLoop 的线程中的版本。也因为此，已经确认在 EventLoop 的线程中，所以不需要在 `synchronized` 中。
- 第 32 行：和【第 24 行】的代码一样，调用 `#callHandlerRemoved0(AbstractChannelHandlerContext)` 方法，执行回调 ChannelHandler 移除完成( removed )事件。

## 3. remove0

`#remove0(AbstractChannelHandlerContext ctx)` 方法，从 pipeline **移除**指定的 AbstractChannelHandlerContext 节点。代码如下：

```
private static void remove0(AbstractChannelHandlerContext ctx) {
    // 获得移除节点的前后节点
    AbstractChannelHandlerContext prev = ctx.prev;
    AbstractChannelHandlerContext next = ctx.next;
    // 前后节点互相指向
    prev.next = next;
    next.prev = prev;
}
```

> FROM 闪电侠 [《netty 源码分析之 pipeline(一)》](https://www.jianshu.com/p/6efa9c5fa702)
>
> - 经历的过程要比添加节点要简单，可以用下面一幅图来表示
>
>   ![删除节点过程](http://www.iocoder.cn/images/Netty/2018_06_07/01.png)
>
> - 最后的结果为
>
>   ![删除节点之后](http://www.iocoder.cn/images/Netty/2018_06_07/02.png)
>
> 结合这两幅图，可以很清晰地了解移除 Handler 的过程，另外，被删除的节点因为没有对象引用到，果过段时间就会被 gc 自动回收。

## 4. callHandlerRemoved0

`#callHandlerRemoved0(AbstractChannelHandlerContext)` 方法，执行回调 ChannelHandler 移除完成( removed )事件。代码如下：

```
 1: private void callHandlerRemoved0(final AbstractChannelHandlerContext ctx) {
 2:     // Notify the complete removal.
 3:     try {
 4:         try {
 5:             // 回调 ChannelHandler 移除完成( removed )事件
 6:             ctx.handler().handlerRemoved(ctx);
 7:         } finally {
 8:             // 设置 AbstractChannelHandlerContext 已移除
 9:             ctx.setRemoved();
10:         }
11:     } catch (Throwable t) {
12:         // 触发异常的传播
13:         fireExceptionCaught(new ChannelPipelineException(
14:                 ctx.handler().getClass().getName() + ".handlerRemoved() has thrown an exception.", t));
15:     }
16: }
```

- 第 6 行：调用 `ChannelHandler#handlerRemoved(AbstractChannelHandlerContext)` 方法，回调 ChannelHandler 移除完成( removed )事件。一般来说，通过这个方法，来释放 ChannelHandler 占用的资源。**注意**，因为这个方法的执行在 EventLoop 的线程中，所以要尽量避免执行时间过长。
- 第 9 行：调用 `AbstractChannelHandlerContext#setRemoved()` 方法，设置 AbstractChannelHandlerContext 已移除。
- 第 11 至 15 行：发生异常，触发异常的传播。详细解析，见 [《精尽 Netty 源码解析 —— ChannelPipeline（六）之异常事件的传播》](http://svip.iocoder.cn/Netty/ChannelPipeline-6-exception) 。

## 5. PendingHandlerRemovedTask

PendingHandlerRemovedTask 实现 PendingHandlerCallback 抽象类，用于回调移除 ChannelHandler 节点。代码如下：

```
private final class PendingHandlerRemovedTask extends PendingHandlerCallback {

    PendingHandlerRemovedTask(AbstractChannelHandlerContext ctx) {
        super(ctx);
    }

    @Override
    public void run() {
        callHandlerRemoved0(ctx);
    }

    @Override
    void execute() {
        EventExecutor executor = ctx.executor();
        // 在 EventLoop 的线程中，回调 ChannelHandler removed 事件
        if (executor.inEventLoop()) {
            callHandlerRemoved0(ctx);
        } else {
            // 提交 EventLoop 中，执行回调 ChannelHandler removed 事件
            try {
                executor.execute(this); // <1>
            } catch (RejectedExecutionException e) {
                if (logger.isWarnEnabled()) {
                    logger.warn(
                            "Can't invoke handlerRemoved() as the EventExecutor {} rejected it," +
                                    " removing handler {}.", executor, ctx.name(), e);
                }
                // 标记 AbstractChannelHandlerContext 为已移除
                // remove0(...) was call before so just call AbstractChannelHandlerContext.setRemoved().
                ctx.setRemoved();
            }
        }
    }
    
}
```

- 在 `#execute()` 实现方法中，我们可以看到，和 `#remove((AbstractChannelHandlerContext ctx)` 方法的【第 17 至 32 行】的代码比较类似，目的是，在 EventLoop 的线程中，执行 `#callHandlerRemoved0(AbstractChannelHandlerContext)` 方法，回调 ChannelHandler 移除完成( removed )事件。
- `<1>` 处，为什么 PendingHandlerRemovedTask 可以直接提交到 EventLoop 中呢？因为 PendingHandlerRemovedTask 是个 Runnable ，这也就是为什么 PendingHandlerCallback 实现 Runnable 接口的原因。

## 666. 彩蛋

水文一小篇。推荐阅读文章：

- 闪电侠 [《Netty 源码分析之 pipeline(一)》](https://www.jianshu.com/p/6efa9c5fa702)

# 精尽 Netty 源码解析 —— ChannelPipeline（四）之 Outbound 事件的传播

2018-06-10

## 1. 概述

本文我们来分享，在 pipeline 中的 **Outbound 事件的传播**。我们先来回顾下 Outbound 事件的定义：

> 老艿艿：A01、A02 等等，是我们每条定义的编号。

- [x] A01：Outbound 事件是【请求】事件(由 Connect 发起一个请求, 并最终由 Unsafe 处理这个请求)

  > 老艿艿：A01 = A02 + A03

- [x] A02：Outbound 事件的发起者是 Channel

- [x] A03：Outbound 事件的处理者是 Unsafe

- [x] A04：Outbound 事件在 Pipeline 中的传输方向是 `tail` -> `head`

- [x] A05：在 ChannelHandler 中处理事件时, 如果这个 Handler 不是最后一个 Handler ，则需要调用 `ctx.xxx` (例如 `ctx.connect` ) 将此事件继续传播下去. 如果不这样做, 那么此事件的传播会提前终止.

- [x] A06：Outbound 事件流: `Context.OUT_EVT` -> `Connect.findContextOutbound` -> `nextContext.invokeOUT_EVT` -> `nextHandler.OUT_EVT` -> `nextContext.OUT_EVT`

下面，我们来跟着代码，理解每条定义。

## 2. ChannelOutboundInvoker

在 `io.netty.channel.ChannelOutboundInvoker` 接口中，定义了所有 Outbound 事件对应的方法：

```
ChannelFuture bind(SocketAddress localAddress);
ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise);

ChannelFuture connect(SocketAddress remoteAddress);
ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress);
ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise);
ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);

ChannelFuture disconnect();
ChannelFuture disconnect(ChannelPromise promise);

ChannelFuture close();
ChannelFuture close(ChannelPromise promise);

ChannelFuture deregister();
ChannelFuture deregister(ChannelPromise promise);

ChannelOutboundInvoker read();

ChannelFuture write(Object msg);
ChannelFuture write(Object msg, ChannelPromise promise);

ChannelOutboundInvoker flush();

ChannelFuture writeAndFlush(Object msg, ChannelPromise promise);
ChannelFuture writeAndFlush(Object msg);
```

而 ChannelOutboundInvoker 的**部分**子类/接口如下图：

![类图](http://www.iocoder.cn/images/Netty/2018_06_10/01.png)

- 我们可以看到类图，有 Channel、ChannelPipeline、AbstractChannelHandlerContext 都继承/实现了该接口。那这意味着什么呢？我们继续往下看。

在 [《精尽 Netty 源码解析 —— 启动（一）之服务端》](http://svip.iocoder.cn/Netty/bootstrap-1-server/) 中，我们可以看到 Outbound 事件的其中之一 **bind** ，本文就以 **bind** 的过程，作为示例。调用栈如下：

![调用栈](http://www.iocoder.cn/images/Netty/2018_06_10/02.png)

- `AbstractChannel#bind(SocketAddress localAddress, ChannelPromise promise)` 方法，代码如下：

  ```
  @Override
  public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
      return pipeline.bind(localAddress, promise);
  }
  ```

  - ```
    AbstractChannel#bind(SocketAddress localAddress, ChannelPromise promise)
    ```

     

    方法，实现的自 ChannelOutboundInvoker 接口。

    - Channel 是 **bind** 的发起者，**这符合 Outbound 事件的定义 A02** 。

  - 在方法内部，会调用

     

    ```
    ChannelPipeline#bind(SocketAddress localAddress, ChannelPromise promise)
    ```

     

    方法，而这个方法，也是实现的自 ChannelOutboundInvoker 接口。

    从这里可以看出，对于 ChannelOutboundInvoker 接口方法的实现，Channel 对它的实现，会调用 ChannelPipeline 的对应方法

    ( ( 有一点绕，胖友理解下 ) )。

    - 那么接口下，让我们看看 `ChannelPipeline#bind(SocketAddress localAddress, ChannelPromise promise)` 方法的具体实现。

## 3. DefaultChannelPipeline

`DefaultChannelPipeline#bind(SocketAddress localAddress, ChannelPromise promise)` 方法的实现，代码如下：

```
@Override
public final ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return tail.bind(localAddress, promise);
}
```

- 在方法内部，会调用

   

  ```
  TailContext#bind(SocketAddress localAddress, ChannelPromise promise)
  ```

   

  方法。

  这符合 Outbound 事件的定义 A04

   

  。

  - 实际上，TailContext 的该方法，继承自 AbstractChannelHandlerContext 抽象类，而 AbstractChannelHandlerContext 实现了 ChannelOutboundInvoker 接口。*从这里可以看出，对于 ChannelOutboundInvoker 接口方法的实现，ChannelPipeline 对它的实现，会调用 AbstractChannelHandlerContext 的对应方法*( 有一点绕，胖友理解下 )。

## 4. AbstractChannelHandlerContext

`AbstractChannelHandlerContext#bind(SocketAddress localAddress, ChannelPromise promise)` 方法的实现，代码如下：

```
 1: @Override
 2: public ChannelFuture bind(final SocketAddress localAddress, final ChannelPromise promise) {
 3:     if (localAddress == null) {
 4:         throw new NullPointerException("localAddress");
 5:     }
 6:     // 判断是否为合法的 Promise 对象
 7:     if (isNotValidPromise(promise, false)) {
 8:         // cancelled
 9:         return promise;
10:     }
11: 
12:     // 获得下一个 Outbound 节点
13:     final AbstractChannelHandlerContext next = findContextOutbound();
14:     // 获得下一个 Outbound 节点的执行器
15:     EventExecutor executor = next.executor();
16:     // 调用下一个 Outbound 节点的 bind 方法
17:     if (executor.inEventLoop()) {
18:         next.invokeBind(localAddress, promise);
19:     } else {
20:         safeExecute(executor, new Runnable() {
21:             @Override
22:             public void run() {
23:                 next.invokeBind(localAddress, promise);
24:             }
25:         }, promise, null);
26:     }
27:     return promise;
28: }
```

- 第 6 至 10 行：判断 `promise` 是否为合法的 Promise 对象。代码如下：

  ```
  private boolean isNotValidPromise(ChannelPromise promise, boolean allowVoidPromise) {
      if (promise == null) {
          throw new NullPointerException("promise");
      }
  
      // Promise 已经完成
      if (promise.isDone()) {
          // Check if the promise was cancelled and if so signal that the processing of the operation
          // should not be performed.
          //
          // See https://github.com/netty/netty/issues/2349
          if (promise.isCancelled()) {
              return true;
          }
          throw new IllegalArgumentException("promise already done: " + promise);
      }
  
      // Channel 不符合
      if (promise.channel() != channel()) {
          throw new IllegalArgumentException(String.format(
                  "promise.channel does not match: %s (expected: %s)", promise.channel(), channel()));
      }
  
      // DefaultChannelPromise 合法 // <1>
      if (promise.getClass() == DefaultChannelPromise.class) {
          return false;
      }
      // 禁止 VoidChannelPromise 
      if (!allowVoidPromise && promise instanceof VoidChannelPromise) {
          throw new IllegalArgumentException(
                  StringUtil.simpleClassName(VoidChannelPromise.class) + " not allowed for this operation");
      }
      // 禁止 CloseFuture
      if (promise instanceof AbstractChannel.CloseFuture) {
          throw new IllegalArgumentException(
                  StringUtil.simpleClassName(AbstractChannel.CloseFuture.class) + " not allowed in a pipeline");
      }
      return false;
  }
  ```

  - 虽然方法很长，重点是 `<1>` 处，`promise` 的类型为 DefaultChannelPromise 。

- 第 13 行：【重要】调用 `#findContextOutbound()` 方法，获得下一个 Outbound 节点。代码如下：

  ```
  private AbstractChannelHandlerContext findContextOutbound() {
      // 循环，向前获得一个 Outbound 节点
      AbstractChannelHandlerContext ctx = this;
      do {
          ctx = ctx.prev;
      } while (!ctx.outbound);
      return ctx;
  }
  ```

  - 循环，**向前**获得一个 Outbound 节点。
  - 循环，**向前**获得一个 Outbound 节点。
  - 循环，**向前**获得一个 Outbound 节点。
  - 😈 重要的事情说三遍，对于 Outbound 事件的传播，是从 pipeline 的尾巴到头部，**这符合 Outbound 事件的定义 A04** 。

- 第 15 行：调用 `AbstractChannelHandlerContext#executor()` 方法，获得下一个 Outbound 节点的执行器。代码如下：

  ```
  // Will be set to null if no child executor should be used, otherwise it will be set to the
  // child executor.
  /**
   * EventExecutor 对象
   */
  final EventExecutor executor;
  
  @Override
  public EventExecutor executor() {
      if (executor == null) {
          return channel().eventLoop();
      } else {
          return executor;
      }
  }
  ```

  - 如果未设置**子执行器**，则使用 Channel 的 EventLoop 作为执行器。😈 一般情况下，我们可以忽略**子执行器**的逻辑，也就是说，可以直接认为是使用 **Channel 的 EventLoop 作为执行器**。

- 第 16 至 26 行：**在 EventLoop 的线程中**，调用**下一个节点**的 `AbstractChannelHandlerContext#invokeBind(SocketAddress localAddress, ChannelPromise promise)` 方法，传播 **bind** 事件给**下一个节点**。

  - 第 20 至 25 行：如果不在 EventLoop 的线程中，会调用 `#safeExecute(EventExecutor executor, Runnable runnable, ChannelPromise promise, Object msg)` 方法，提交到 EventLoop 的线程中执行。代码如下：

    ```
    private static void safeExecute(EventExecutor executor, Runnable runnable, ChannelPromise promise, Object msg) {
        try {
            // 提交 EventLoop 的线程中，进行执行任务
            executor.execute(runnable);
        } catch (Throwable cause) {
            try {
                // 发生异常，回调通知 promise 相关的异常
                promise.setFailure(cause);
            } finally {
                // 释放 msg 相关的资源
                if (msg != null) {
                    ReferenceCountUtil.release(msg);
                }
            }
        }
    }
    ```

    - x

------

`AbstractChannelHandlerContext#invokeBind(SocketAddress localAddress, ChannelPromise promise)` 方法，代码如下：

```
 1: private void invokeBind(SocketAddress localAddress, ChannelPromise promise) {
 2:     if (invokeHandler()) { // 判断是否符合的 ChannelHandler
 3:         try {
 4:             // 调用该 ChannelHandler 的 bind 方法
 5:             ((ChannelOutboundHandler) handler()).bind(this, localAddress, promise);
 6:         } catch (Throwable t) {
 7:             notifyOutboundHandlerException(t, promise); // 通知 Outbound 事件的传播，发生异常
 8:         }
 9:     } else {
10:         // 跳过，传播 Outbound 事件给下一个节点
11:         bind(localAddress, promise);
12:     }
13: }
```

- 第 2 行：调用 `#invokeHandler()` 方法，判断是否符合的 ChannelHandler 。代码如下：

  ```
  /**
   * Makes best possible effort to detect if {@link ChannelHandler#handlerAdded(ChannelHandlerContext)} was called
   * yet. If not return {@code false} and if called or could not detect return {@code true}.
   *
   * If this method returns {@code false} we will not invoke the {@link ChannelHandler} but just forward the event.
   * This is needed as {@link DefaultChannelPipeline} may already put the {@link ChannelHandler} in the linked-list
   * but not called {@link ChannelHandler#handlerAdded(ChannelHandlerContext)}.
   */
  private boolean invokeHandler() {
      // Store in local variable to reduce volatile reads.
      int handlerState = this.handlerState;
      return handlerState == ADD_COMPLETE || (!ordered && handlerState == ADD_PENDING);
  }
  ```

  - 对于 `ordered = true` 的节点，必须 ChannelHandler 已经添加完成。
  - 对于 `ordered = false` 的节点，没有 ChannelHandler 的要求。

- 第 9 至 12 行：若是**不符合**的 ChannelHandler ，则**跳过**该节点，调用 `AbstractChannelHandlerContext#bind(SocketAddress localAddress, ChannelPromise promise)` 方法，传播 Outbound 事件给下一个节点。即，又回到 [「4. AbstractChannelHandlerContext」](http://svip.iocoder.cn/Netty/Pipeline-4-outbound/#) 的开头。

- 第 2 至 8 行：若是**符合**的 ChannelHandler ：

  - 第 5 行：调用 ChannelHandler 的 `#bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)` 方法，处理 bind 事件。

    - 😈 实际上，此时节点的数据类型为 DefaultChannelHandlerContext 类。若它被认为是 Outbound 节点，那么他的处理器的类型会是 **ChannelOutboundHandler** 。而 `io.netty.channel.ChannelOutboundHandler` 类似 ChannelOutboundInvoker ，定义了对每个 Outbound 事件的处理。代码如下：

      ```
      void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception;
      
      void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) throws Exception;
      
      void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
      
      void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
      
      void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
      
      void read(ChannelHandlerContext ctx) throws Exception;
      
      void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception;
      
      void flush(ChannelHandlerContext ctx) throws Exception;
      ```

      - 胖友自己对比下噢。

    - 如果节点的 `ChannelOutboundHandler#bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)` 方法的实现，不调用 `AbstractChannelHandlerContext#bind(SocketAddress localAddress, ChannelPromise promise)` 方法，就不会传播 Outbound 事件给下一个节点。**这就是 Outbound 事件的定义 A05** 。可能有点绕，我们来看下 Netty LoggingHandler 对该方法的实现代码：

      ```
      final class LoggingHandler implements ChannelInboundHandler, ChannelOutboundHandler {
      
          // ... 省略无关方法
          
          @Override
          public void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception {
              // 打印日志
              log(Event.BIND, "localAddress=" + localAddress);
              // 传递 bind 事件，给下一个节点
              ctx.bind(localAddress, promise); // <1>
          }
      }
      ```

      - 如果把 `<1>` 处的代码去掉，bind 事件将不会传播给下一个节点！！！**一定要注意**。

    - 这块的逻辑非常重要，如果胖友觉得很绕，一定要自己多调试 + 调试 + 调试。

  - 第 7 行：如果发生异常，调用 `#notifyOutboundHandlerException(Throwable, Promise)` 方法，通知 Outbound 事件的传播，发生异常。详细解析，见 [《精尽 Netty 源码解析 —— ChannelPipeline（六）之异常事件的传播》](http://svip.iocoder.cn/Netty/ChannelPipeline-6-exception) 。

------

本小节的整个代码实现，**就是 Outbound 事件的定义 A06**的体现。而随着 Outbound 事件在节点不断从 pipeline 的尾部到头部的传播，最终会到达 HeadContext 节点。

## 5. HeadContext

`HeadContext#bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)` 方法，代码如下：

```
@Override
public void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception {
    unsafe.bind(localAddress, promise);
}
```

- 调用 `Unsafe#bind(SocketAddress localAddress, ChannelPromise promise)` 方法，进行 bind 事件的处理。也就是说 Unsafe 是 **bind** 的处理着，**这符合 Outbound 事件的定义 A03** 。
- 而后续的逻辑，就是 [《精尽 Netty 源码分析 —— 启动（一）之服务端》](http://svip.iocoder.cn/Netty/bootstrap-1-server/) 的 [「3.13.2 doBind0」](http://svip.iocoder.cn/Netty/Pipeline-4-outbound/#) 小节，从 `Unsafe#bind(SocketAddress localAddress, ChannelPromise promise)` 方法，开始。
- 至此，整个 pipeline 的 Outbound 事件的传播结束。

## 6. 关于其他 Outbound 事件

本文暂时只分享了 **bind** 这个 Outbound 事件。剩余的其他事件，胖友可以自己进行调试和理解。例如：**connect** 事件，并且结合 [《精尽 Netty 源码分析 —— 启动（二）之客户端》](http://svip.iocoder.cn/Netty/bootstrap-2-client/) 一文。

## 666. 彩蛋

*推荐阅读文章：

- 闪电侠 [《netty 源码分析之 pipeline(二)》](https://www.jianshu.com/p/087b7e9a27a2)*

# 精尽 Netty 源码解析 —— ChannelPipeline（五）之 Inbound 事件的传播

2018-06-16

## 1. 概述

本文我们来分享，在 pipeline 中的 **Inbound 事件的传播**。我们先来回顾下 Inbound 事件的定义：

> 老艿艿：B01、B02 等等，是我们每条定义的编号。

- [x] B01：Inbound 事件是【通知】事件, 当某件事情已经就绪后, 通知上层.

  > 老艿艿：B01 = B02 + B03

- [x] B02：Inbound 事件发起者是 Unsafe

- [x] B03：Inbound 事件的处理者是 TailContext, 如果用户没有实现自定义的处理方法, 那么Inbound 事件默认的处理者是 TailContext, 并且其处理方法是空实现.

- [x] B04：Inbound 事件在 Pipeline 中传输方向是 `head`( 头 ) -> `tail`( 尾 )

- [x] B05：在 ChannelHandler 中处理事件时, 如果这个 Handler 不是最后一个 Handler, 则需要调用 `ctx.fireIN_EVT` (例如 `ctx.fireChannelActive` ) 将此事件继续传播下去. 如果不这样做, 那么此事件的传播会提前终止.

- [x] B06：Inbound 事件流: `Context.fireIN_EVT` -> `Connect.findContextInbound` -> `nextContext.invokeIN_EVT` -> `nextHandler.IN_EVT` -> `nextContext.fireIN_EVT`

Outbound 和 Inbound 事件十分的镜像，所以，接下来我们来跟着的代码，和 [《精尽 Netty 源码解析 —— ChannelPipeline（四）之 Outbound 事件的传播》](http://svip.iocoder.cn/Netty/Pipeline-4-inbound) 会非常相似。

## 2. ChannelInboundInvoker

在 `io.netty.channel.ChannelInboundInvoker` 接口中，定义了所有 Inbound 事件对应的方法：

```
ChannelInboundInvoker fireChannelRegistered();
ChannelInboundInvoker fireChannelUnregistered();

ChannelInboundInvoker fireChannelActive();
ChannelInboundInvoker fireChannelInactive();

ChannelInboundInvoker fireExceptionCaught(Throwable cause);

ChannelInboundInvoker fireUserEventTriggered(Object event);

ChannelInboundInvoker fireChannelRead(Object msg);
ChannelInboundInvoker fireChannelReadComplete();

ChannelInboundInvoker fireChannelWritabilityChanged();
```

而 ChannelInboundInvoker 的**部分**子类/接口如下图：

![类图](http://www.iocoder.cn/images/Netty/2018_06_13/01.png)

- 我们可以看到类图，有 ChannelPipeline、AbstractChannelHandlerContext 都继承/实现了该接口。那这意味着什么呢？我们继续往下看。
- 相比来说，Channel 实现了 ChannelOutboundInvoker 接口，但是**不实现** ChannelInboundInvoker 接口。

在 [《精尽 Netty 源码解析 —— 启动（一）之服务端》](http://svip.iocoder.cn/Netty/bootstrap-1-server/) 中，我们可以看到 Inbound 事件的其中之一 **fireChannelActive** ，本文就以 **fireChannelActive** 的过程，作为示例。调用栈如下：

![调用栈](http://www.iocoder.cn/images/Netty/2018_06_13/02.png)

- `AbstractUnsafe#bind(final SocketAddress localAddress, final ChannelPromise promise)` 方法，代码如下：

  ```
  @Override
  public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
      // 判断是否在 EventLoop 的线程中。
      assertEventLoop();
  
      // ... 省略部分代码
     
      // 记录 Channel 是否激活
      boolean wasActive = isActive();
  
      // 绑定 Channel 的端口
      doBind(localAddress);
  
      // 若 Channel 是新激活的，触发通知 Channel 已激活的事件。 
      if (!wasActive && isActive()) {
          invokeLater(new Runnable() {
              @Override
              public void run() {
                  pipeline.fireChannelActive(); // <1>
              }
          });
      }
  
      // 回调通知 promise 执行成功
      safeSetSuccess(promise);
  }
  ```

  - 在

     

    ```
    <1>
    ```

     

    处，调用

     

    ```
    ChannelPipeline#fireChannelActive()
    ```

     

    方法。

    - Unsafe 是 **fireChannelActive** 的发起者，**这符合 Inbound 事件的定义 B02** 。
    - 那么接口下，让我们看看 `ChannelPipeline#fireChannelActive()` 方法的具体实现。

## 3. DefaultChannelPipeline

`DefaultChannelPipeline#fireChannelActive()` 方法的实现，代码如下：

```
@Override
public final ChannelPipeline fireChannelActive() {
    AbstractChannelHandlerContext.invokeChannelActive(head);
    return this;
}
```

- 在方法内部，会调用

   

  ```
  AbstractChannelHandlerContext#invokeChannelActive(final AbstractChannelHandlerContext next)
  ```

   

  方法，而方法参数是

   

  ```
  head
  ```

   

  ，

  这符合 Inbound 事件的定义 B04

   

  。

  - 实际上，HeadContext 的该方法，继承自 AbstractChannelHandlerContext 抽象类，而 AbstractChannelHandlerContext 实现了 ChannelInboundInvoker 接口。*从这里可以看出，对于 ChannelInboundInvoker 接口方法的实现，ChannelPipeline 对它的实现，会调用 AbstractChannelHandlerContext 的对应方法*( 有一点绕，胖友理解下 )。

## 4. AbstractChannelHandlerContext#invokeChannelActive

`AbstractChannelHandlerContext#invokeChannelActive(final AbstractChannelHandlerContext next)` **静态**方法，代码如下：

```
 1: static void invokeChannelActive(final AbstractChannelHandlerContext next) {
 2:     // 获得下一个 Inbound 节点的执行器
 3:     EventExecutor executor = next.executor();
 4:     // 调用下一个 Inbound 节点的 Channel active 方法
 5:     if (executor.inEventLoop()) {
 6:         next.invokeChannelActive();
 7:     } else {
 8:         executor.execute(new Runnable() {
 9:             @Override
10:             public void run() {
11:                 next.invokeChannelActive();
12:             }
13:         });
14:     }
15: }
```

- 方法参数 `next` ，下一个 Inbound 节点。

- 第 3 行：调用 `AbstractChannelHandlerContext#executor()` 方法，获得下一个 Inbound 节点的执行器。

- 第 4 至 14 行：调用 `AbstractChannelHandlerContext#invokeChannelActive()` 方法，调用下一个 Inbound 节点的 Channel active 方法。

  - 在 [「3. DefaultChannelPipeline」](http://svip.iocoder.cn/Netty/Pipeline-5-inbound/#) 中，我们可以看到传递的**第一个** `next` 方法参数为 `head`( HeadContext ) 节点。

  - 代码如下：

    ```
     1: private void invokeChannelActive() {
     2:     if (invokeHandler()) { // 判断是否符合的 ChannelHandler
     3:         try {
     4:             // 调用该 ChannelHandler 的 Channel active 方法
     5:             ((ChannelInboundHandler) handler()).channelActive(this);
     6:         } catch (Throwable t) {
     7:             notifyHandlerException(t);  // 通知 Inbound 事件的传播，发生异常
     8:         }
     9:     } else {
    10:         // 跳过，传播 Inbound 事件给下一个节点
    11:         fireChannelActive();
    12:     }
    13: }
    ```

    - 第 2 行：调用 `#invokeHandler()` 方法，判断是否符合的 ChannelHandler 。

    - 第 9 至 12 行：若是**不符合**的 ChannelHandler ，则**跳过**该节点，调用 `AbstractChannelHandlerContext#fireChannelActive(` 方法，传播 Inbound 事件给下一个节点。详细解析，见 [「6. AbstractChannelHandlerContext#fireChannelActive」](http://svip.iocoder.cn/Netty/Pipeline-5-inbound/#) 。

    - 第 2 至 8 行：若是**符合**的 ChannelHandler ：

      - 第 5 行：调用 ChannelHandler 的 `#channelActive(ChannelHandlerContext ctx)` 方法，处理 Channel active 事件。

        - 😈 实际上，此时节点的数据类型为 DefaultChannelHandlerContext 类。若它被认为是 Inbound 节点，那么他的处理器的类型会是 **ChannelInboundHandler** 。而 `io.netty.channel.ChannelInboundHandler` 类似 ChannelInboundInvoker ，定义了对每个 Inbound 事件的处理。代码如下：

          ```
          void channelRegistered(ChannelHandlerContext ctx) throws Exception;
          void channelUnregistered(ChannelHandlerContext ctx) throws Exception;
          
          void channelActive(ChannelHandlerContext ctx) throws Exception;
          void channelInactive(ChannelHandlerContext ctx) throws Exception;
          
          void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception;
          
          void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;
          void channelReadComplete(ChannelHandlerContext ctx) throws Exception;
          
          void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception;
           
          @Override
          @SuppressWarnings("deprecation")
          void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception; // 不属于 Inbound 事件
          ```

          - 胖友自己对比下噢。

        - 如果节点的 `ChannelInboundHandler#channelActive(ChannelHandlerContext ctx` 方法的实现，不调用 `AbstractChannelHandlerContext#fireChannelActive()` 方法，就不会传播 Inbound 事件给下一个节点。**这就是 Inbound 事件的定义 B05** 。可能有点绕，我们来看下 Netty LoggingHandler 对该方法的实现代码：

          ```
          final class LoggingHandler implements ChannelInboundHandler, ChannelOutboundHandler {
          
              // ... 省略无关方法
              
              @Override
              public void channelActive(ChannelHandlerContext ctx) throws Exception {
                  // 打印日志
                  if (logger.isEnabled(internalLevel)) {
                      logger.log(internalLevel, format(ctx, "ACTIVE"));
                  }
                  // 传递 Channel active 事件，给下一个节点
                  ctx.fireChannelActive(); // <1>
              }
          }
          ```

          - 如果把 `<1>` 处的代码去掉，Channel active 事件 事件将不会传播给下一个节点！！！**一定要注意**。

        - 这块的逻辑非常重要，如果胖友觉得很绕，一定要自己多调试 + 调试 + 调试。

      - 第 7 行：如果发生异常，调用 `#notifyHandlerException(Throwable)` 方法，通知 Inbound 事件的传播，发生异常。详细解析，见 [《精尽 Netty 源码解析 —— ChannelPipeline（六）之异常事件的传播》](http://svip.iocoder.cn/Netty/ChannelPipeline-6-exception) 。

## 5. HeadContext

`HeadContext#invokeChannelActive()` 方法，代码如下：

```
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    // 传播 Channel active 事件给下一个 Inbound 节点 <1>
    ctx.fireChannelActive();

    // 执行 read 逻辑 <2>
    readIfIsAutoRead();
}
```

- `<1>` 处，调用 `AbstractChannelHandlerContext#fireChannelActive()` 方法，传播 Channel active 事件给下一个 Inbound 节点。详细解析，见 [「6. AbstractChannelHandlerContext」](http://svip.iocoder.cn/Netty/Pipeline-5-inbound/#) 中。

- `<2>` 处，调用 `HeadContext#readIfIsAutoRead()` 方法，执行 read 逻辑。代码如下：

  ```
  // HeadContext.java
  private void readIfIsAutoRead() {
      if (channel.config().isAutoRead()) { 
          channel.read();
      }
  }
  
  // AbstractChannel.java
  @Override
  public Channel read() {
      pipeline.read();
      return this;
  }
  ```

  - 该方法内部，会调用 `Channel#read()` 方法，而后通过 pipeline 传递该 **read** OutBound 事件，最终调用 `HeadContext#read()` 方法，代码如下：

    ```
    @Override
    public void read(ChannelHandlerContext ctx) {
        unsafe.beginRead();
    }
    ```

    - 后续的逻辑，便是 [《精尽 Netty 源码分析 —— 启动（一）之服务端》](http://svip.iocoder.cn/Netty/bootstrap-1-server/) 的 [3.13.3 beginRead](http://svip.iocoder.cn/Netty/Pipeline-5-inbound/#) 小节，胖友可以自己再去回顾下。

  - 这里说的是 **OutBound** 事件，不是本文的 InBound 事件。所以，胖友不要搞混哈。只能说是对 [《精尽 Netty 源码分析 —— 启动（一）之服务端》](http://svip.iocoder.cn/Netty/bootstrap-1-server/) 的 [3.13.3 beginRead](http://svip.iocoder.cn/Netty/Pipeline-5-inbound/#) 小节的补充。

## 6. AbstractChannelHandlerContext#fireChannelActive

`AbstractChannelHandlerContext#fireChannelActive()` 方法，代码如下：

```
@Override
public ChannelHandlerContext fireChannelActive() {
    // 获得下一个 Inbound 节点的执行器
    // 调用下一个 Inbound 节点的 Channel active 方法
    invokeChannelActive(findContextInbound());
    return this;
}
```

- 【重要】调用 `AbstractChannelHandlerContext#findContextInbound()` 方法，获得下一个 Inbound 节点的执行器。代码如下：

  ```
  private AbstractChannelHandlerContext findContextInbound() {
      // 循环，向后获得一个 Inbound 节点
      AbstractChannelHandlerContext ctx = this;
      do {
          ctx = ctx.next;
      } while (!ctx.inbound);
      return ctx;
  }
  ```

  - 循环，**向后**获得一个 Inbound 节点。
  - 循环，**向后**获得一个 Inbound 节点。
  - 循环，**向后**获得一个 Inbound 节点。
  - 😈 重要的事情说三遍，对于 Inbound 事件的传播，是从 pipeline 的头部到尾部，**这符合 Inbound 事件的定义 B04** 。

- 调用 `AbstractChannelHandlerContext#invokeChannelActive(AbstractChannelHandlerContext)` **静态**方法，调用下一个 Inbound 节点的 Channel active 方法。即，又回到 [「4. AbstractChannelHandlerContext#invokeChannelActive」](http://svip.iocoder.cn/Netty/Pipeline-5-inbound/#) 的开头。

------

本小节的整个代码实现，**就是 Inbound 事件的定义 B06** 的体现。而随着 Inbound 事件在节点不断从 pipeline 的头部到尾部的传播，最终会到达 TailContext 节点。

## 7. TailContext

`TailContext#channelActive(ChannelHandlerContext ctx)` 方法，代码如下：

```
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    onUnhandledInboundChannelActive();
}
```

- 在方法内部，会调用 `DefaultChannelPipeline#onUnhandledInboundChannelActive()` 方法，代码如下：

  ```
  /**
   * Called once the {@link ChannelInboundHandler#channelActive(ChannelHandlerContext)}event hit
   * the end of the {@link ChannelPipeline}.
   */
  protected void onUnhandledInboundChannelActive() {
  }
  ```

  - 该方法是个**空**方法，**这符合 Inbound 事件的定义 B03** 。
  - 至此，整个 pipeline 的 Inbound 事件的传播结束。

# 8. 关于其他 Inbound 事件

本文暂时只分享了 **firecChannelActive** 这个 Inbound 事件。剩余的其他事件，胖友可以自己进行调试和理解。例如：**fireChannelRegistered** 事件，并且结合 [《精尽 Netty 源码分析 —— 启动（一）之服务端》](http://svip.iocoder.cn/Netty/bootstrap-1-server/) 一文。

## 666. 彩蛋

推荐阅读文章：

- 闪电侠 [《netty 源码分析之 pipeline(二)》](https://www.jianshu.com/p/087b7e9a27a2)

感觉上来说，Inbound 事件的传播，比起 Outbound 事件的传播，会相对“绕”一点点。简化来说，实际大概是如下：

```
Unsafe 开始 => DefaultChannelPipeline#fireChannelActive

=> HeadContext#invokeChannelActive => DefaultChannelHandlerContext01#fireChannelActive

=> DefaultChannelHandlerContext01#invokeChannelActive => DefaultChannelHandlerContext02#fireChannelActive
...
=> DefaultChannelHandlerContext99#fireChannelActive => TailContext#fireChannelActive

=> TailContext#invokeChannelActive => 结束
```

笔者觉得可能解释的也有点“绕”，如果不理解或者有地方写的有误解，欢迎来叨叨，以便我们能一起优化这篇文章。

# 精尽 Netty 源码解析 —— ChannelPipeline（六）之异常事件的传播

2018-06-19

## 1. 概述

在 [《精尽 Netty 源码解析 —— ChannelPipeline（四）之 Outbound 事件的传播》](http://svip.iocoder.cn/Netty/Pipeline-4-outbound/) 和 [《精尽 Netty 源码解析 —— ChannelPipeline（五）之 Inbound 事件的传播》](http://svip.iocoder.cn/Netty/Pipeline-5-inbound/) 中，我们看到 Outbound 和 Inbound 事件在 pipeline 中的传播逻辑。但是，无可避免，传播的过程中，可能会发生异常，那是怎么处理的呢？

本文，我们就来分享分享这块。

## 2. notifyOutboundHandlerException

我们以 Outbound 事件中的 **bind** 举例子，代码如下：

```
// AbstractChannelHandlerContext.java

private void invokeBind(SocketAddress localAddress, ChannelPromise promise) {
    if (invokeHandler()) { // 判断是否符合的 ChannelHandler
        try {
            // 调用该 ChannelHandler 的 bind 方法 <1>
            ((ChannelOutboundHandler) handler()).bind(this, localAddress, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise); // 通知 Outbound 事件的传播，发生异常 <2>
        }
    } else {
        // 跳过，传播 Outbound 事件给下一个节点
        bind(localAddress, promise);
    }
}
```

- 在 `<1>` 处，调用 `ChannelOutboundHandler#bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)` 方法**发生异常**时，会在 `<2>` 处调用 `AbstractChannelHandlerContext#notifyOutboundHandlerException(Throwable cause, ChannelPromise promise)` 方法，通知 Outbound 事件的传播，发生异常。
- 其他 Outbound 事件，大体的代码也是和 `#invokeBind(SocketAddress localAddress, ChannelPromise promise)`是一致的。如下图所示：![类图](http://www.iocoder.cn/images/Netty/2018_06_16/01.png)

------

`AbstractChannelHandlerContext#notifyOutboundHandlerException(Throwable cause, ChannelPromise promise)` 方法，通知 Outbound 事件的传播，发生异常。代码如下：

```
private static void notifyOutboundHandlerException(Throwable cause, ChannelPromise promise) {
    // Only log if the given promise is not of type VoidChannelPromise as tryFailure(...) is expected to return
    // false.
    PromiseNotificationUtil.tryFailure(promise, cause, promise instanceof VoidChannelPromise ? null : logger);
}
```

- 在方法内部，会调用 `PromiseNotificationUtil#tryFailure(Promise<?> p, Throwable cause, InternalLogger logger)` 方法，通知 bind 事件对应的 Promise 对应的监听者们。代码如下：

  ```
  public static void tryFailure(Promise<?> p, Throwable cause, InternalLogger logger) {
      if (!p.tryFailure(cause) && logger != null) {
          Throwable err = p.cause();
          if (err == null) {
              logger.warn("Failed to mark a promise as failure because it has succeeded already: {}", p, cause);
          } else {
              logger.warn(
                      "Failed to mark a promise as failure because it has failed already: {}, unnotified cause: {}",
                      p, ThrowableUtil.stackTraceToString(err), cause);
          }
      }
  }
  ```

  - 以 bind 事件来举一个监听器的例子。代码如下：

    ```
    ChannelFuture f = b.bind(PORT).addListener(new ChannelFutureListener() { // <1> 监听器就是我！
        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
            System.out.println("异常：" + future.casue());
        }
    }).sync();
    ```

    - `<1>` 处的监听器，就是示例。当发生异常时，就会通知该监听器，对该异常做进一步**自定义**的处理。**也就是说，该异常不会在 pipeline 中传播**。

  - 我们再来看看怎么通知监听器的源码实现。调用 `DefaultPromise#tryFailure(Throwable cause)` 方法，通知 Promise 的监听器们，发生了异常。代码如下：

    ```
    @Override
    public boolean tryFailure(Throwable cause) {
        if (setFailure0(cause)) { // 设置 Promise 的结果
            // 通知监听器
            notifyListeners();
            // 返回成功
            return true;
        }
        // 返回失败
        return false;
    }
    ```

    - 若 `DefaultPromise#setFailure0(Throwable cause)` 方法，设置 Promise 的结果为方法传入的异常。但是有可能会传递失败，例如说，Promise 已经被设置了结果。
    - 如果该方法返回 `false` 通知 Promise 失败，那么 `PromiseNotificationUtil#tryFailure(Promise<?> p, Throwable cause, InternalLogger logger)` 方法的后续，就会使用 `logger` 打印错误日志。

## 3. notifyHandlerException

我们以 Inbound 事件中的 **fireChannelActive** 举例子，代码如下：

```
private void invokeChannelActive() {
    if (invokeHandler()) { // 判断是否符合的 ChannelHandler
        try {
            // 调用该 ChannelHandler 的 Channel active 方法 <1>
            ((ChannelInboundHandler) handler()).channelActive(this);
        } catch (Throwable t) {
            notifyHandlerException(t);  // 通知 Inbound 事件的传播，发生异常 <2>
        }
    } else {
        // 跳过，传播 Inbound 事件给下一个节点
        fireChannelActive();
    }
}
```

- 在 `<1>` 处，调用 `ChannelInboundHandler#channelActive(ChannelHandlerContext ctx)` 方法**发生异常**时，会在 `<2>` 处调用 `AbstractChannelHandlerContext#notifyHandlerException(Throwable cause)` 方法，通知 Inbound 事件的传播，发生异常。

- 其他 Inbound 事件，大体的代码也是和

   

  ```
  #invokeChannelActive()
  ```

   

  是一致的。如下图所示：



  - 😈 **注意，笔者在写的时候，突然发现 Outbound 事件中的 read 和 flush 的异常处理方式和 Inbound 事件是一样的**。
  - 😈 **注意，笔者在写的时候，突然发现 Outbound 事件中的 read 和 flush 的异常处理方式和 Inbound 事件是一样的**。
  - 😈 **注意，笔者在写的时候，突然发现 Outbound 事件中的 read 和 flush 的异常处理方式和 Inbound 事件是一样的**。

------

`AbstractChannelHandlerContext#notifyHandlerException(Throwable cause)` 方法，通知 Inbound 事件的传播，发生异常。代码如下：

```
private void notifyHandlerException(Throwable cause) {
    // <1> 如果是在 `ChannelHandler#exceptionCaught(ChannelHandlerContext ctx, Throwable cause)` 方法中，仅打印错误日志。否则会形成死循环。
    if (inExceptionCaught(cause)) {
        if (logger.isWarnEnabled()) {
            logger.warn(
                    "An exception was thrown by a user handler " +
                            "while handling an exceptionCaught event", cause);
        }
        return;
    }

    // <2> 在 pipeline 中，传播 Exception Caught 事件
    invokeExceptionCaught(cause);
}
```

- `<1>` 处，调用 `AbstractChannelHandlerContext#inExceptionCaught(Throwable cause)` 方法，如果是在 `ChannelHandler#exceptionCaught(ChannelHandlerContext ctx, Throwable cause)` 方法中，**发生异常**，仅打印错误日志，**并 return 返回** 。否则会形成死循环。代码如下：

  ```
  private static boolean inExceptionCaught(Throwable cause) {
      do {
          StackTraceElement[] trace = cause.getStackTrace();
          if (trace != null) {
              for (StackTraceElement t : trace) { // 循环 StackTraceElement
                  if (t == null) {
                      break;
                  }
                  if ("exceptionCaught".equals(t.getMethodName())) { // 通过方法名判断
                      return true;
                  }
              }
          }
          cause = cause.getCause();
      } while (cause != null); // 循环异常的 cause() ，直到到没有
      
      return false;
  }
  ```

  - 通过 StackTraceElement 的方法名来判断，是不是 `ChannelHandler#exceptionCaught(ChannelHandlerContext ctx, Throwable cause)` 方法。

- `<2>` 处，调用 `AbstractChannelHandlerContext#invokeExceptionCaught(Throwable cause)` 方法，在 pipeline 中，传递 Exception Caught 事件。在下文中，我们会看到，和 [《精尽 Netty 源码解析 —— ChannelPipeline（五）之 Inbound 事件的传播》](http://svip.iocoder.cn/Netty/Pipeline-5-inbound/) 的逻辑( `AbstractChannelHandlerContext#invokeChannelActive()` )是**一致**的。

  - 比较特殊的是，Exception Caught 事件在 pipeline 的起始节点，不是 `head` 头节点，而是**发生异常的当前节点开始**。怎么理解好呢？对于在 pipeline 上传播的 Inbound **xxx** 事件，在发生异常后，转化成 **Exception Caught** 事件，继续从当前节点，继续向下传播。

  - 如果 **Exception Caught** 事件在 pipeline 中的传播过程中，一直没有处理掉该异常的节点，最终会到达尾节点 `tail` ，它对 `#exceptionCaught(ChannelHandlerContext ctx, Throwable cause)` 方法的实现，代码如下：

    ```
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        onUnhandledInboundException(cause);
    }
    ```

    - 在方法内部，会调用 `DefaultChannelPipeline#onUnhandledInboundException(Throwable cause)` 方法，代码如下：

      ```
      /**
       * Called once a {@link Throwable} hit the end of the {@link ChannelPipeline} without been handled by the user
       * in {@link ChannelHandler#exceptionCaught(ChannelHandlerContext, Throwable)}.
       */
      protected void onUnhandledInboundException(Throwable cause) {
          try {
              logger.warn(
                      "An exceptionCaught() event was fired, and it reached at the tail of the pipeline. " +
                              "It usually means the last handler in the pipeline did not handle the exception.",
                      cause);
          } finally {
              ReferenceCountUtil.release(cause);
          }
      }
      ```

      - 打印**告警**日志，并调用 `ReferenceCountUtil#release(Throwable)` 方法，释放需要释放的资源。

      - 从英文注释中，我们也可以看到，这种情况出现在**使用者**未定义合适的 ChannelHandler 处理这种异常，所以对于这种情况下，`tail` 节点只好打印**告警**日志。

      - 实际使用时，笔者建议胖友一定要定义 ExceptionHandler ，能够处理掉所有的异常，而不要使用到 `tail` 节点的异常处理。😈

      - 好基友【闪电侠】对尾节点 `tail` 做了很赞的总结

        > 总结一下，tail 节点的作用就是结束事件传播，并且对一些重要的事件做一些善意提醒

## 666. 彩蛋

推荐阅读文章：

- 闪电侠 [《netty 源码分析之 pipeline(二)》](https://www.jianshu.com/p/087b7e9a27a2)

# 精尽 Netty 源码解析 —— Channel（一）之简介

2018-07-01

## 1. 概述

在前面的文章中，我们已经不断看到 Netty Channel 的身影，例如：

- 在 [《精尽 Netty 源码分析 —— 启动（一）之服务端》](http://svip.iocoder.cn/Netty/bootstrap-1-server/) 中，我们看了服务端 NioServerSocketChannel **对象创建**的过程。
- 在 [《精尽 Netty 源码分析 —— 启动（二）之客户端》](http://svip.iocoder.cn/Netty/bootstrap-2-client/) 中，我们看了客户端 NioSocketChannel **对象创建**的过程。

但是，考虑到本小节的后续文章，我们还是需要这样一篇文章，整体性的再看一次 Channel 的面貌。

## 2. Channel

`io.netty.channel.Channel` ，实现 AttributeMap、ChannelOutboundInvoker、Comparable 接口，Netty Channel 接口。

在 [《精尽 Netty 源码分析 —— Netty 简介（一）之项目结构》](http://svip.iocoder.cn/Netty/intro-1/) 中，我们对 Channel 的组件定义如下：

> Channel 是 Netty 网络操作抽象类，它除了包括基本的 I/O 操作，如 bind、connect、read、write 之外，还包括了 Netty 框架相关的一些功能，如获取该 Channel 的 EventLoop 。
>
> 在传统的网络编程中，作为核心类的 Socket ，它对程序员来说并不是那么友好，直接使用其成本还是稍微高了点。而 Netty 的 Channel 则提供的一系列的 API ，它大大降低了直接与 Socket 进行操作的复杂性。而相对于原生 NIO 的 Channel，Netty 的 Channel 具有如下优势( 摘自《Netty权威指南( 第二版 )》) ：
>
> - 在 Channel 接口层，采用 Facade 模式进行统一封装，将网络 I/O 操作、网络 I/O 相关联的其他操作封装起来，统一对外提供。
> - Channel 接口的定义尽量大而全，为 SocketChannel 和 ServerSocketChannel 提供统一的视图，由不同子类实现不同的功能，公共功能在抽象父类中实现，最大程度地实现功能和接口的重用。
> - 具体实现采用聚合而非包含的方式，将相关的功能类聚合在 Channel 中，由 Channel 统一负责和调度，功能实现更加灵活。

### 2.1 基础查询

```
/**
 * Returns the globally unique identifier of this {@link Channel}.
 *
 * Channel 的编号
 */
ChannelId id();

/**
 * Return the {@link EventLoop} this {@link Channel} was registered to.
 *
 * Channel 注册到的 EventLoop
 */
EventLoop eventLoop();

/**
 * Returns the parent of this channel.
 *
 * 父 Channel 对象
 *
 * @return the parent channel.
 *         {@code null} if this channel does not have a parent channel.
 */
Channel parent();

/**
 * Returns the configuration of this channel.
 *
 * Channel 配置参数
 */
ChannelConfig config();

/**
 * Returns an <em>internal-use-only</em> object that provides unsafe operations.
 *
 * Unsafe 对象
 */
Unsafe unsafe();

/**
 * Return the assigned {@link ChannelPipeline}.
 *
 * ChannelPipeline 对象，用于处理 Inbound 和 Outbound 事件的处理
 */
ChannelPipeline pipeline();

/**
 * Return the assigned {@link ByteBufAllocator} which will be used to allocate {@link ByteBuf}s.
 *
 * ByteBuf 分配器
 */
ByteBufAllocator alloc();

/**
 * Returns the local address where this channel is bound to.  The returned
 * {@link SocketAddress} is supposed to be down-cast into more concrete
 * type such as {@link InetSocketAddress} to retrieve the detailed
 * information.
 *
 * 本地地址
 *
 * @return the local address of this channel.
 *         {@code null} if this channel is not bound.
 */
SocketAddress localAddress();
/**
 * Returns the remote address where this channel is connected to.  The
 * returned {@link SocketAddress} is supposed to be down-cast into more
 * concrete type such as {@link InetSocketAddress} to retrieve the detailed
 * information.
 *
 * 远端地址
 *
 * @return the remote address of this channel.
 *         {@code null} if this channel is not connected.
 *         If this channel is not connected but it can receive messages
 *         from arbitrary remote addresses (e.g. {@link DatagramChannel},
 *         use {@link DatagramPacket#recipient()} to determine
 *         the origination of the received message as this method will
 *         return {@code null}.
 */
SocketAddress remoteAddress();
```

- 自身基本信息有 `#id()`、`#parent()`、`#config()`、`#localAddress()`、`#remoteAddress()` 方法。
- 每个 Channel 都有的核心组件有 `#eventLoop()`、`#unsafe()`、`#pipeline()`、`#alloc()` 方法。

### 2.2 状态查询

```
/**
 * Returns {@code true} if the {@link Channel} is open and may get active later
 *
 * Channel 是否打开。
 *
 * true 表示 Channel 可用
 * false 表示 Channel 已关闭，不可用
 */
boolean isOpen();

/**
 * Returns {@code true} if the {@link Channel} is registered with an {@link EventLoop}.
 *
 * Channel 是否注册
 *
 * true 表示 Channel 已注册到 EventLoop 上
 * false 表示 Channel 未注册到 EventLoop 上
 */
boolean isRegistered();

/**
 * Return {@code true} if the {@link Channel} is active and so connected.
 *
 * Channel 是否激活
 *
 * 对于服务端 ServerSocketChannel ，true 表示 Channel 已经绑定到端口上，可提供服务
 * 对于客户端 SocketChannel ，true 表示 Channel 连接到远程服务器
 */
boolean isActive();

/**
 * Returns {@code true} if and only if the I/O thread will perform the
 * requested write operation immediately.  Any write requests made when
 * this method returns {@code false} are queued until the I/O thread is
 * ready to process the queued write requests.
 *
 * Channel 是否可写
 *
 * 当 Channel 的写缓存区 outbound 非 null 且可写时，返回 true
 */
boolean isWritable();
/**
 * 获得距离不可写还有多少字节数
 * 
 * Get how many bytes can be written until {@link #isWritable()} returns {@code false}.
 * This quantity will always be non-negative. If {@link #isWritable()} is {@code false} then 0.
 */
long bytesBeforeUnwritable();
/**
 * 获得距离可写还要多少字节数
 * 
 * Get how many bytes must be drained from underlying buffers until {@link #isWritable()} returns {@code true}.
 * This quantity will always be non-negative. If {@link #isWritable()} is {@code true} then 0.
 */
long bytesBeforeWritable();
```

一个**正常结束**的 Channel 状态转移有**两**种情况：

- 服务端用于绑定( bind )的 Channel 、或者客户端发起连接( connect )的 Channel 。

  ```
  REGISTERED -> CONNECT/BIND -> ACTIVE -> CLOSE -> INACTIVE -> UNREGISTERED
  ```

- 服务端接受( accept )客户端的 Channel 。

  ```
  REGISTERED -> ACTIVE -> CLOSE -> INACTIVE -> UNREGISTERED
  ```

一个**异常关闭**的 Channel 状态转移不符合上面的。

### 2.3 IO 操作

```
@Override
Channel read();

@Override
Channel flush();
```

- 这两个方法，继承自 ChannelOutboundInvoker 接口。实际还有如下几个：

  ```
  ChannelFuture bind(SocketAddress localAddress);
  ChannelFuture connect(SocketAddress remoteAddress);
  ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress);
  ChannelFuture disconnect();
  ChannelFuture close();
  ChannelFuture deregister();
  ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise);
  ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise);
  ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);
  ChannelFuture disconnect(ChannelPromise promise);
  ChannelFuture close(ChannelPromise promise);
  ChannelFuture deregister(ChannelPromise promise);
  ChannelOutboundInvoker read();
  ChannelFuture write(Object msg);
  ChannelFuture write(Object msg, ChannelPromise promise);
  ChannelOutboundInvoker flush();
  ChannelFuture writeAndFlush(Object msg, ChannelPromise promise);
  ChannelFuture writeAndFlush(Object msg);
  ```

- 对比下来，我们会发现 Channel 重写 ChannelOutboundInvoker 这两个接口的原因是：将返回值从 ChannelOutboundInvoker 修改成 Channel 。

- 我们看到除了 `#read()` 和 `#flush()` 方法，其它方法的返回值的类型都是 ChannelFuture ，这表明这些操作是**异步** IO 的过程。

### 2.4 异步结果 Future

```
/**
 * Returns the {@link ChannelFuture} which will be notified when this
 * channel is closed.  This method always returns the same future instance.
 *
 * Channel 关闭的 Future 对象
 */
ChannelFuture closeFuture();
```

- 除了自定义的 `#closeFuture()` 方法，也从 ChannelOutboundInvoker 接口继承了几个接口方法：

  ```
  ChannelPromise newPromise();
  ChannelProgressivePromise newProgressivePromise();
  
  ChannelFuture newSucceededFuture();
  ChannelFuture newFailedFuture(Throwable cause);
  
  ChannelPromise voidPromise();
  ```

  - 通过这些接口方法，可创建或获得和该 Channel 相关的 Future / Promise 对象。

### 2.5 类图

Channel 的子接口和实现类如下图：

![Channel 的子接口和实现类](http://www.iocoder.cn/images/Netty/2018_07_01/01.png)

- 本图包含了 NIO、OIO、Local、Embedded 四种 Channel 实现类。说明如下：![Channel 四种 Channel 实现类的说明](http://www.iocoder.cn/images/Netty/2018_07_01/02.png)
- 本系列仅分享 NIO 部分，所以裁剪类图如下：![NIO Channel 类图](http://www.iocoder.cn/images/Netty/2018_07_01/03.png)

## 3. Unsafe

Unsafe **接口**，定义在在 `io.netty.channel.Channel` 内部，和 Channel 的操作**紧密结合**，下文我们将看到。

Unsafe 直译中文为“不安全”，就是告诉我们，**无需**且**不必要**在我们使用 Netty 的代码中，**不能直接**调用 Unsafe 相关的方法。Netty 注释说明如下：

```
/**
 * <em>Unsafe</em> operations that should <em>never</em> be called from user-code. 
 * 
 * These methods are only provided to implement the actual transport, and must be invoked from an I/O thread except for the
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

😈 当然，对于我们想要了解 Netty 内部实现的胖友，那必须开扒它的代码实现落。因为它和 Channel 密切相关，所以我们也对它的接口做下分类。

### 3.1 基础查询

```
/**
 * Return the assigned {@link RecvByteBufAllocator.Handle} which will be used to allocate {@link ByteBuf}'s when
 * receiving data.
 *
 * ByteBuf 分配器的处理器
 */
RecvByteBufAllocator.Handle recvBufAllocHandle();

/**
 * Return the {@link SocketAddress} to which is bound local or
 * {@code null} if none.
 *
 * 本地地址
 */
SocketAddress localAddress();

/**
 * Return the {@link SocketAddress} to which is bound remote or
 * {@code null} if none is bound yet.
 *
 * 远端地址
 */
SocketAddress remoteAddress();
```

### 3.2 状态查询

无 😈

### 3.3 IO 操作

```
void register(EventLoop eventLoop, ChannelPromise promise);
void bind(SocketAddress localAddress, ChannelPromise promise);
void connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);
void disconnect(ChannelPromise promise);
void close(ChannelPromise promise);
void closeForcibly();
void deregister(ChannelPromise promise);
void beginRead();
void write(Object msg, ChannelPromise promise);
void flush();

/**
 * Returns the {@link ChannelOutboundBuffer} of the {@link Channel} where the pending write requests are stored.
 */
ChannelOutboundBuffer outboundBuffer();
```

### 3.4 异步结果 Future

```
/**
 * Return a special ChannelPromise which can be reused and passed to the operations in {@link Unsafe}.
 * It will never be notified of a success or error and so is only a placeholder for operations
 * that take a {@link ChannelPromise} as argument but for which you not want to get notified.
 */
ChannelPromise voidPromise();
```

### 3.5 类图

Unsafe 的子接口和实现类如下图：

![Unsafe 的子接口和实现类](http://www.iocoder.cn/images/Netty/2018_07_01/04.png)

- 已经经过裁剪，仅保留 NIO Channel 相关的 Unsafe 的子接口和实现类部分。
- 我们会发现，对于 Channel 和 Unsafe 来说，类名中包含 Byte 是属于客户端的，Message 是属于服务端的。

## 4. ChanelId

`io.netty.channel.ChannelId` 实现 Serializable、Comparable 接口，Channel 编号接口。代码如下：

```
public interface ChannelId extends Serializable, Comparable<ChannelId> {

    /**
     * Returns the short but globally non-unique string representation of the {@link ChannelId}.
     *
     * 全局非唯一
     */
    String asShortText();

    /**
     * Returns the long yet globally unique string representation of the {@link ChannelId}.
     *
     * 全局唯一
     */
    String asLongText();

}
```

- `#asShortText()` 方法，返回的编号，短，但是全局非唯一。
- `#asLongText()` 方法，返回的编号，长，但是全局唯一。

ChanelId 的**默认**实现类为 `io.netty.channel.DefaultChannelId` ，我们主要看看它是如何生成 Channel 的**两种**编号的。代码如下：

```
@Override
public String asShortText() {
    String shortValue = this.shortValue;
    if (shortValue == null) {
        this.shortValue = shortValue = ByteBufUtil.hexDump(data, data.length - RANDOM_LEN, RANDOM_LEN);
    }
    return shortValue;
}

@Override
public String asLongText() {
    String longValue = this.longValue;
    if (longValue == null) {
        this.longValue = longValue = newLongValue();
    }
    return longValue;
}
```

- 对于 `#asShortText()` 方法，仅使用最后 4 字节的随机数字，并转换成 16 进制的数字字符串。也因此，短，但是全局非唯一。

- 对于 `#asLongText()` 方法，通过调用 `#newLongValue()` 方法生成。代码如下：

  ```
  private String newLongValue() {
      StringBuilder buf = new StringBuilder(2 * data.length + 5); // + 5 的原因是有 5 个 '-'
      int i = 0;
      i = appendHexDumpField(buf, i, MACHINE_ID.length); // MAC 地址。
      i = appendHexDumpField(buf, i, PROCESS_ID_LEN); // 进程 ID 。4 字节。
      i = appendHexDumpField(buf, i, SEQUENCE_LEN); // 32 位数字，顺序增长。4 字节。
      i = appendHexDumpField(buf, i, TIMESTAMP_LEN); // 时间戳。8 字节。
      i = appendHexDumpField(buf, i, RANDOM_LEN); // 32 位数字，随机。4 字节。
      assert i == data.length;
      return buf.substring(0, buf.length() - 1);
  }
  
  private int appendHexDumpField(StringBuilder buf, int i, int length) {
      buf.append(ByteBufUtil.hexDump(data, i, length));
      buf.append('-');
      i += length;
      return i;
  }
  ```

  - 具体的生成规则，见代码。最终也是 16 进制的数字。也因此，长，但是全局唯一。

## 5. ChannelConfig

`io.netty.channel.ChannelConfig` ，Channel 配置接口。代码如下：

```
Map<ChannelOption<?>, Object> getOptions();
<T> T getOption(ChannelOption<T> option);
boolean setOptions(Map<ChannelOption<?>, ?> options);
<T> boolean setOption(ChannelOption<T> option, T value);

int getConnectTimeoutMillis();
ChannelConfig setConnectTimeoutMillis(int connectTimeoutMillis);

@Deprecated
int getMaxMessagesPerRead();
@Deprecated
ChannelConfig setMaxMessagesPerRead(int maxMessagesPerRead);

int getWriteSpinCount();
ChannelConfig setWriteSpinCount(int writeSpinCount);

ByteBufAllocator getAllocator();
ChannelConfig setAllocator(ByteBufAllocator allocator);

<T extends RecvByteBufAllocator> T getRecvByteBufAllocator();
ChannelConfig setRecvByteBufAllocator(RecvByteBufAllocator allocator);

boolean isAutoRead();
ChannelConfig setAutoRead(boolean autoRead);

boolean isAutoClose();
ChannelConfig setAutoClose(boolean autoClose);

int getWriteBufferHighWaterMark();
ChannelConfig setWriteBufferHighWaterMark(int writeBufferHighWaterMark);

int getWriteBufferLowWaterMark();
ChannelConfig setWriteBufferLowWaterMark(int writeBufferLowWaterMark);

MessageSizeEstimator getMessageSizeEstimator();
ChannelConfig setMessageSizeEstimator(MessageSizeEstimator estimator);

WriteBufferWaterMark getWriteBufferWaterMark();
ChannelConfig setWriteBufferWaterMark(WriteBufferWaterMark writeBufferWaterMark);
```

- 调用 `#setOption(ChannelOption<T> option, T value)` 方法时，会调用相应的 `#setXXX(...)` 方法。代码如下：

  ```
  // DefaultChannelConfig.java
  
  @Override
  @SuppressWarnings("deprecation")
  public <T> boolean setOption(ChannelOption<T> option, T value) {
      validate(option, value);
  
      if (option == CONNECT_TIMEOUT_MILLIS) {
          setConnectTimeoutMillis((Integer) value);
      } else if (option == MAX_MESSAGES_PER_READ) {
          setMaxMessagesPerRead((Integer) value);
      } else if (option == WRITE_SPIN_COUNT) {
          setWriteSpinCount((Integer) value);
      } else if (option == ALLOCATOR) {
          setAllocator((ByteBufAllocator) value);
      } else if (option == RCVBUF_ALLOCATOR) {
          setRecvByteBufAllocator((RecvByteBufAllocator) value);
      } else if (option == AUTO_READ) {
          setAutoRead((Boolean) value);
      } else if (option == AUTO_CLOSE) {
          setAutoClose((Boolean) value);
      } else if (option == WRITE_BUFFER_HIGH_WATER_MARK) {
          setWriteBufferHighWaterMark((Integer) value);
      } else if (option == WRITE_BUFFER_LOW_WATER_MARK) {
          setWriteBufferLowWaterMark((Integer) value);
      } else if (option == WRITE_BUFFER_WATER_MARK) {
          setWriteBufferWaterMark((WriteBufferWaterMark) value);
      } else if (option == MESSAGE_SIZE_ESTIMATOR) {
          setMessageSizeEstimator((MessageSizeEstimator) value);
      } else if (option == SINGLE_EVENTEXECUTOR_PER_GROUP) {
          setPinEventExecutorPerGroup((Boolean) value);
      } else {
          return false;
      }
  }
  ```

- ChannelConfig 的配置项 `io.netty.channel.ChannelOption` 很多，胖友可以看下 [《Netty：option 和 childOption 参数设置说明》](https://www.jianshu.com/p/0bff7c020af2) ，了解感兴趣的配置项。

### 5.1 类图

ChannelConfig 的子接口和实现类如下图：

![ChannelConfig 的子接口和实现类](http://www.iocoder.cn/images/Netty/2018_07_01/05.png)

- 已经经过裁剪，仅保留 NIO Channel 相关的 ChannelConfig 的子接口和实现类部分。

## 666. 彩蛋

正如文头所说，在前面的文章中，我们已经不断看到 Netty Channel 的身影，例如：

- 在 [《精尽 Netty 源码分析 —— 启动（一）之服务端》](http://svip.iocoder.cn/Netty/bootstrap-1-server/) 中，我们看了服务端 NioServerSocketChannel **bind** 的过程。
- 在 [《精尽 Netty 源码分析 —— 启动（二）之客户端》](http://svip.iocoder.cn/Netty/bootstrap-2-client/) 中，我们看了客户端 NioSocketChannel **connect** 的过程。

在后续的文章中，我们会分享 Netty NIO Channel 的其他操作，😈 一篇一个操作。

------

推荐阅读文章：

- Hypercube [《自顶向下深入分析 Netty（六）–Channel总述》](https://www.jianshu.com/p/fffc18d33159)

# 精尽 Netty 源码解析 —— Channel（二）之 accept 操作

2018-07-04

## 1. 概述

本文分享 Netty NIO 服务端 NioServerSocketChannel 接受( **accept** )客户端连接的过程。简单来说：

1. 服务端 NioServerSocketChannel 的 boss EventLoop 线程轮询是否有新的客户端连接接入。
2. 当轮询到有新的连接接入，封装连入的客户端的 SocketChannel 为 Netty NioSocketChannel 对象。
3. 选择一个服务端 NioServerSocketChannel 的 worker EventLoop ，将客户端的 NioSocketChannel 注册到其上。并且，注册客户端的 NioSocketChannel 的读事件，开始轮询该客户端是否有数据写入。

下面，让我们来看看具体的代码实现。

## 2. NioMessageUnsafe#read

> 老艿艿：有点不知道怎么取标题好，直接用方法名吧。

在 NioEventLoop 的 `#processSelectedKey(SelectionKey k, AbstractNioChannel ch)` 方法中，我们会看到这样一段代码：

```
// SelectionKey.OP_READ 或 SelectionKey.OP_ACCEPT 就绪
// readyOps == 0 是对 JDK Bug 的处理，防止空的死循环
// Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
// to a spin loop
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    unsafe.read();
}
```

- 当 `(readyOps & SelectionKey.OP_ACCEPT) != 0` 时，这就是服务端 NioServerSocketChannel 的 boss EventLoop 线程**轮询到**有新的客户端连接接入。
- 然后，调用 `NioMessageUnsafe#read()` 方法，“读取”( 😈 这个抽象很灵性 )新的客户端连接连入。

------

`NioMessageUnsafe#read()` 方法，代码如下：

```
 1: private final class NioMessageUnsafe extends AbstractNioUnsafe {
 2: 
 3:     /**
 4:      * 新读取的客户端连接数组
 5:      */
 6:     private final List<Object> readBuf = new ArrayList<Object>();
 7: 
 8:     @SuppressWarnings("Duplicates")
 9:     @Override
10:     public void read() {
11:         assert eventLoop().inEventLoop();
12:         final ChannelConfig config = config();
13:         final ChannelPipeline pipeline = pipeline();
14:         // 获得 RecvByteBufAllocator.Handle 对象
15:         final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
16:         // 重置 RecvByteBufAllocator.Handle 对象
17:         allocHandle.reset(config);
18: 
19:         boolean closed = false;
20:         Throwable exception = null;
21:         try {
22:             try {
23:                 do {
24:                     // 读取客户端的连接到 readBuf 中
25:                     int localRead = doReadMessages(readBuf);
26:                     // 无可读取的客户端的连接，结束
27:                     if (localRead == 0) {
28:                         break;
29:                     }
30:                     // 读取出错
31:                     if (localRead < 0) {
32:                         closed = true; // 标记关闭
33:                         break;
34:                     }
35: 
36:                     // 读取消息数量 + localRead
37:                     allocHandle.incMessagesRead(localRead);
38:                 } while (allocHandle.continueReading()); // 循环判断是否继续读取
39:             } catch (Throwable t) {
40:                 // 记录异常
41:                 exception = t;
42:             }
43: 
44:             // 循环 readBuf 数组，触发 Channel read 事件到 pipeline 中。
45:             int size = readBuf.size();
46:             for (int i = 0; i < size; i ++) {
47:                 // TODO 芋艿
48:                 readPending = false;
49:                 // 在内部，会通过 ServerBootstrapAcceptor ，将客户端的 Netty NioSocketChannel 注册到 EventLoop 上
50:                 pipeline.fireChannelRead(readBuf.get(i));
51:             }
52:             // 清空 readBuf 数组
53:             readBuf.clear();
54:             // 读取完成
55:             allocHandle.readComplete();
56:             // 触发 Channel readComplete 事件到 pipeline 中。
57:             pipeline.fireChannelReadComplete();
58: 
59:             // 发生异常
60:             if (exception != null) {
61:                 // 判断是否要关闭 TODO 芋艿
62:                 closed = closeOnReadError(exception);
63: 
64:                 // 触发 exceptionCaught 事件到 pipeline 中。
65:                 pipeline.fireExceptionCaught(exception);
66:             }
67: 
68:             if (closed) {
69:                 // TODO 芋艿
70:                 inputShutdown = true;
71:                 // TODO 芋艿
72:                 if (isOpen()) {
73:                     close(voidPromise());
74:                 }
75:             }
76:         } finally {
77:             // Check if there is a readPending which was not processed yet.
78:             // This could be for two reasons:
79:             // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
80:             // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
81:             //
82:             // See https://github.com/netty/netty/issues/2254
83:             // TODO 芋艿
84:             if (!readPending && !config.isAutoRead()) {
85:                 removeReadOp();
86:             }
87:         }
88:     }
89: }
```

- 😈 NioMessageUnsafe 只有一个 `#read()` 方法，而该方法，“读取”新的客户端连接连入。

- 第 15 行：调用 `Unsafe#recvBufAllocHandle()` 方法，获得 获得 RecvByteBufAllocator.Handle 对象。默认情况下，返回的是 AdaptiveRecvByteBufAllocator.HandleImpl 对象。关于它的内容，我们放在 ByteBuf 相关的文章，详细解析。

  - 第 17 行：调用 `DefaultMaxMessagesRecvByteBufAllocator.MaxMessageHandle#reset(ChannelConfig)` 方法，重置 RecvByteBufAllocator.Handle 对象。代码如下：

    ```
    @Override
    public void reset(ChannelConfig config) {
        this.config = config; // 重置 ChannelConfig 对象
        maxMessagePerRead = maxMessagesPerRead(); // 重置 maxMessagePerRead 属性
        totalMessages = totalBytesRead = 0; // 重置 totalMessages 和 totalBytesRead 属性
    }
    ```

    - 注意，AdaptiveRecvByteBufAllocator.HandleImpl 继承 DefaultMaxMessagesRecvByteBufAllocator.MaxMessageHandle 抽象类。

- 第 22 至 42 行：**while 循环** “读取”新的客户端连接连入。

  - 第 25 行： 调用 `NioServerSocketChannel#doReadMessages(List<Object> buf)` 方法，读取客户端的连接到 `readBuf` 中。详细解析，胖友先跳到 [「3. AbstractNioMessageChannel#doReadMessages」](http://svip.iocoder.cn/Netty/Channel-2-accept/#) 中，看完记得回到此处。

  - 第 25 至 29 行：无可读取的客户端的连接，结束循环。

  - 第 30 至 34 行：读取出错，**标记关闭服务端**，并结束循环。目前我们看到 `NioServerSocketChannel#doReadMessages(List<Object> buf)` 方法的实现，返回的结果只会存在 0 和 1 ，也就是说不会出现这种情况。笔者又去翻了别的实现类，例如 `NioDatagramChannel#doReadMessages(List<Object> buf)` 方法，在发生异常时，会返回 -1 。

  - 第 37 行：调用 `AdaptiveRecvByteBufAllocator.HandleImpl#incMessagesRead(int amt)` 方法，读取消息( 客户端 )数量 + `localRead` 。代码如下：

    ```
    @Override
    public final void incMessagesRead(int amt) {
        totalMessages += amt;
    }
    ```

    - 对于 AdaptiveRecvByteBufAllocator.HandleImpl 来说，考虑到**抽象**的需要，所以统一使用“消息”的说法。

  - 第 38 行：调用 `AdaptiveRecvByteBufAllocator.HandleImpl#incMessagesRead(int amt)#continueReading()` 方法，判断是否循环是否继续，读取( 接受 )新的客户端连接。代码如下：

    ```
    // AdaptiveRecvByteBufAllocator.HandleImpl.java
    @Override
    public boolean continueReading() {
        return continueReading(defaultMaybeMoreSupplier);
    }
    
    // DefaultMaxMessagesRecvByteBufAllocator.MaxMessageHandle.java
    @Override
    public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
        return config.isAutoRead() &&
               (!respectMaybeMoreData || maybeMoreDataSupplier.get()) &&
               totalMessages < maxMessagePerRead &&
               totalBytesRead > 0; // <1>
    }
    ```

    - 因为 `<1>` 处，此时 `totalBytesRead` 等于 0 ，所以会返回 **false** 。因此，循环会结束。也因此，对于 NioServerSocketChannel 来说，**每次只接受一个新的客户端连接**。😈 当然，因为服务端 NioServerSocketChannel 对 `Selectionkey.OP_ACCEPT` 事件感兴趣，所以**后续的新的客户端连接还是会被接受的**。

  - 第 39 至 42 行：读取过程中发生异常，记录该异常到 `exception` 中，同时结束循环。

- 第 44 至 51 行：循环

   

  ```
  readBuf
  ```

   

  数组，触发 Channel read 事件到 pipeline 中。

  - 第 48 行：TODO 芋艿 细节

  - 第 50 行：调用

     

    ```
    ChannelPipeline#fireChannelRead(Object msg)
    ```

     

    方法，触发 Channel read 事件到 pipeline 中。

    - **注意**，传入的方法参数是新接受的客户端 NioSocketChannel 连接。
    - 在内部，会通过 ServerBootstrapAcceptor ，将客户端的 Netty NioSocketChannel 注册到 EventLoop 上。详细解析，胖友先跳到 [「4. ServerBootstrapAcceptor」](http://svip.iocoder.cn/Netty/Channel-2-accept/#) 中，看完记得回到此处。

- 第 53 行：清空 `readBuf` 数组。

- 第 55 行：调用 `RecvByteBufAllocator.Handle#readComplete()` 方法，读取完成。暂无重要的逻辑，不详细解析。

- 第 57 行：调用 `ChannelPipeline#fireChannelReadComplete()` 方法，触发 Channel readComplete 事件到 pipeline 中。

  - *如果有需要，胖友可以自定义处理器，处理该事件。一般情况下，不需要*。

  - 如果没有自定义 ChannelHandler 进行处理，最终会被 pipeline 中的尾节点 TailContext 所处理。代码如下：

    ```
    // TailContext.java
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        onUnhandledInboundChannelReadComplete();
    }
    
    // DefaultChannelPipeline.java
    protected void onUnhandledInboundChannelReadComplete() {
    }
    ```

    - 具体的调用是**空方法**。

- 第 60 至 66 行：

  ```
  exception
  ```

   

  非空，说明在接受连接过程中发生异常。

  - 第 62 行：TODO 芋艿 细节

  - 第 65 行： 调用

     

    ```
    ChannelPipeline#fireExceptionCaught(Throwable)
    ```

     

    方法，触发 exceptionCaught 事件到 pipeline 中。

    - 默认情况下，会使用 ServerBootstrapAcceptor 处理该事件。详细解析，见 [「4.3 exceptionCaught」](http://svip.iocoder.cn/Netty/Channel-2-accept/#) 。
    - *如果有需要，胖友可以自定义处理器，处理该事件。一般情况下，不需要*。

- 第 68 至 75 行：TODO 芋艿 细节

- 第 76 至 87 行：TODO 芋艿 细节

## 3. AbstractNioMessageChannel#doReadMessages

`doReadMessages(List<Object> buf)` **抽象**方法，读取客户端的连接到方法参数 `buf` 中。它是一个**抽象**方法，定义在 AbstractNioMessageChannel 抽象类中。代码如下：

```
/**
 * Read messages into the given array and return the amount which was read.
 */
protected abstract int doReadMessages(List<Object> buf) throws Exception;
```

- 返回值为读取到的数量。

NioServerSocketChannel 对该方法的实现代码如下：

```
  1: @Override
  2: protected int doReadMessages(List<Object> buf) throws Exception {
  3:     // 接受客户端连接
  4:     SocketChannel ch = SocketUtils.accept(javaChannel());
  5: 
  6:     try {
  7:         // 创建 Netty NioSocketChannel 对象
  8:         if (ch != null) {
  9:             buf.add(new NioSocketChannel(this, ch));
 10:             return 1;
 11:         }
 12:     } catch (Throwable t) {
 13:         logger.warn("Failed to create a new channel from an accepted socket.", t);
 14:         // 发生异常，关闭客户端的 SocketChannel 连接
 15:         try {
 16:             ch.close();
 17:         } catch (Throwable t2) {
 18:             logger.warn("Failed to close a socket.", t2);
 19:         }
 20:     }
 21: 
 22:     return 0;
 23: }
 
 @Override
protected ServerSocketChannel javaChannel() {
    return (ServerSocketChannel) super.javaChannel();
}
```

- 第 4 行：调用 `SocketUtils#accept(ServerSocketChannel serverSocketChannel)` 方法，接受客户端连接。代码如下：

  ```
  public static SocketChannel accept(final ServerSocketChannel serverSocketChannel) throws IOException {
      try {
          return AccessController.doPrivileged(new PrivilegedExceptionAction<SocketChannel>() {
              @Override
              public SocketChannel run() throws IOException {
                  return serverSocketChannel.accept(); // <1>
              }
          });
      } catch (PrivilegedActionException e) {
          throw (IOException) e.getCause();
      }
  }
  ```

  - 重点是看 `<1>` 处，调用 `ServerSocketChannel#accept()` 方法，接受客户端连接。

- 第 9 行：基于客户端的 NIO ServerSocket ，创建 Netty NioSocketChannel 对象。整个过程，就是

   

  《精尽 Netty 源码分析 —— 启动（二）之客户端》

   

  的

   

  「3.7.1 创建 Channel 对象」

   

  小节。

  - 第 10 行：返回 1 ，表示成功接受了 1 个新的客户端连接。

- 第 12 至 20 行：发生异常，关闭客户端的 SocketChannel 连接，并打印

  告警

  日志。

  - 第 22 行：返回 0 ，表示成功接受 0 个新的客户端连接。

## 4. ServerBootstrapAcceptor

ServerBootstrapAcceptor ，继承 ChannelInboundHandlerAdapter 类，服务器接收器( acceptor )，负责将接受的客户端的 NioSocketChannel 注册到 EventLoop 中。

另外，从继承的是 ChannelInboundHandlerAdapter 类，可以看出它是 Inbound 事件处理器。

### 4.1 构造方法

在服务端的启动过程中，我们看到 ServerBootstrapAcceptor 注册到服务端的 NioServerSocketChannel 的 pipeline 的尾部，代码如下：

```
// 记录当前的属性
final EventLoopGroup currentChildGroup = childGroup;
final ChannelHandler currentChildHandler = childHandler;
final Entry<ChannelOption<?>, Object>[] currentChildOptions;
final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
synchronized (childOptions) {
    currentChildOptions = childOptions.entrySet().toArray(newOptionArray(0));
}
synchronized (childAttrs) {
    currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
}

// 添加 ChannelInitializer 对象到 pipeline 中，用于后续初始化 ChannelHandler 到 pipeline 中。
p.addLast(new ChannelInitializer<Channel>() {

    @Override
    public void initChannel(final Channel ch) throws Exception {
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
                pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs)); // <1>
            }
        });
    }

});
```

- 即 `<1>` 处。也是在此处，创建了 ServerBootstrapAcceptor 对象。代码如下：

  ```
  private final EventLoopGroup childGroup;
  private final ChannelHandler childHandler;
  private final Entry<ChannelOption<?>, Object>[] childOptions;
  private final Entry<AttributeKey<?>, Object>[] childAttrs;
  /**
   * 自动恢复接受客户端连接的任务
   */
  private final Runnable enableAutoReadTask;
  
  ServerBootstrapAcceptor(
          final Channel channel, EventLoopGroup childGroup, ChannelHandler childHandler,
          Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
      this.childGroup = childGroup;
      this.childHandler = childHandler;
      this.childOptions = childOptions;
      this.childAttrs = childAttrs;
  
      // Task which is scheduled to re-enable auto-read.
      // It's important to create this Runnable before we try to submit it as otherwise the URLClassLoader may
      // not be able to load the class because of the file limit it already reached.
      //
      // See https://github.com/netty/netty/issues/1328
      enableAutoReadTask = new Runnable() { // <2>
          @Override
          public void run() {
              channel.config().setAutoRead(true);
          }
      };
  }
  ```

  - `enableAutoReadTask` 属性，自动恢复接受客户端连接的任务，在 `<2>` 处初始化。具体的使用，我们在 [「4.3 exceptionCaught」](http://svip.iocoder.cn/Netty/Channel-2-accept/#) 中，详细解析。

### 4.2 channelRead

`#channelRead(ChannelHandlerContext ctx, Object msg)` 方法，将接受的客户端的 NioSocketChannel 注册到 EventLoop 中。代码如下：

```
 1: @Override
 2: public void channelRead(ChannelHandlerContext ctx, Object msg) {
 3:     // 老艿艿：如下的注释，先暂时认为是接受的客户端的 NioSocketChannel
 4: 
 5:     // 接受的客户端的 NioSocketChannel 对象
 6:     final Channel child = (Channel) msg;
 7:     // 添加 NioSocketChannel 的处理器
 8:     child.pipeline().addLast(childHandler);
 9:     // 设置 NioSocketChannel 的配置项
10:     setChannelOptions(child, childOptions, logger);
11:     // 设置 NioSocketChannel 的属性
12:     for (Entry<AttributeKey<?>, Object> e: childAttrs) {
13:         child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
14:     }
15: 
16:     try {
17:         // 注册客户端的 NioSocketChannel 到 work EventLoop 中。
18:         childGroup.register(child).addListener(new ChannelFutureListener() {
19: 
20:             @Override
21:             public void operationComplete(ChannelFuture future) throws Exception {
22:                 // 注册失败，关闭客户端的 NioSocketChannel
23:                 if (!future.isSuccess()) {
24:                     forceClose(child, future.cause());
25:                 }
26:             }
27: 
28:         });
29:     } catch (Throwable t) {
30:         // 发生异常，强制关闭客户端的 NioSocketChannel
31:         forceClose(child, t);
32:     }
33: }
```

- 为了方便描述，我们统一认为接受的客户端连接为 NioSocketChannel 对象。

- 第 6 行：接受的客户端的 NioSocketChannel 对象。

  - 第 8 行：调用 `ChannelPipeline#addLast(childHandler)` 方法，将配置的子 Channel 的处理器，添加到 NioSocketChannel 中。
  - 第 10 至 14 行：设置 NioSocketChannel 的配置项、属性。

- 第 17 至 28 行：调用 `EventLoopGroup#register(Channel channel)` 方法，将客户端的 NioSocketChannel 对象，从 worker EventLoopGroup 中选择一个 EventLoop ，注册到其上。

  - 后续的逻辑，就和 [《精尽 Netty 源码分析 —— 启动（一）之服务端》](http://svip.iocoder.cn/Netty/bootstrap-1-server/) 的注册逻辑**基本一致**( 虽然说，文章写的是 NioServerSocketChannel 的注册逻辑 )。

  - 在注册完成之后，该 worker EventLoop 就会开始轮询该客户端是否有数据写入。

  - 第 18 至 28 行：添加监听器，如果注册失败，则调用 `#forceClose(Channel child, Throwable t)` 方法，强制关闭客户端的 NioSocketChannel 连接。代码如下：

    ```
    private static void forceClose(Channel child, Throwable t) {
        child.unsafe().closeForcibly();
        logger.warn("Failed to register an accepted channel: {}", child, t);
    }
    ```

    - 在该方法内部，会调用 `Unsafe#closeForcibly()` 方法，强制关闭客户端的 NioSocketChannel 。

  - 第 29 至 32 行：发生异常，则调用 `#forceClose(Channel child, Throwable t)` 方法，强制关闭客户端的 NioSocketChannel 连接。

### 4.3 exceptionCaught

`#exceptionCaught(ChannelHandlerContext ctx, Throwable cause)` 方法，当捕获到异常时，**暂停 1 秒**，不再接受新的客户端连接；而后，再恢复接受新的客户端连接。代码如下：

```
 1: @Override
 2: public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
 3:     final ChannelConfig config = ctx.channel().config();
 4:     if (config.isAutoRead()) {
 5:         // 关闭接受新的客户端连接
 6:         // stop accept new connections for 1 second to allow the channel to recover
 7:         // See https://github.com/netty/netty/issues/1328
 8:         config.setAutoRead(false);
 9:         // 发起 1 秒的延迟任务，恢复重启开启接受新的客户端连接
10:         ctx.channel().eventLoop().schedule(enableAutoReadTask, 1, TimeUnit.SECONDS);
11:     }
12: 
13:     // 继续传播 exceptionCaught 给下一个节点
14:     // still let the exceptionCaught event flow through the pipeline to give the user
15:     // a chance to do something with it
16:     ctx.fireExceptionCaught(cause);
17: }
```

- 第 8 行：调用 `ChannelConfig#setAutoRead(false)` 方法，关闭接受新的客户端连接。代码如下：

  ```
  // DefaultChannelConfig.java
  /**
   * {@link #autoRead} 的原子更新器
   */
  private static final AtomicIntegerFieldUpdater<DefaultChannelConfig> AUTOREAD_UPDATER = AtomicIntegerFieldUpdater.newUpdater(DefaultChannelConfig.class, "autoRead");
  /**
   * 是否开启自动读取的开关
   *
   * 1 - 开启
   * 0 - 关闭
   */
  @SuppressWarnings("FieldMayBeFinal")
  private volatile int autoRead = 1;
  
  @Override
  public ChannelConfig setAutoRead(boolean autoRead) {
      // 原子更新，并且获得更新前的值 <1>
      boolean oldAutoRead = AUTOREAD_UPDATER.getAndSet(this, autoRead ? 1 : 0) == 1;
      // 发起读取 <2.1>
      if (autoRead && !oldAutoRead) {
          channel.read();
      // 关闭读取 <2.2>
      } else if (!autoRead && oldAutoRead) {
          autoReadCleared();
      }
      return this;
  }
  ```

  - ```
    autoRead
    ```

     

    字段，是否开启自动读取的开关。😈 笔者原本以为是个

     

    ```
    boolean
    ```

     

    类型，是不是胖友也是。其中，1 表示开启，0 表示关闭。

    - `AUTOREAD_UPDATER` 静态变量，对 `autoRead` 字段的原子更新器。

  - `<1>` 处，使用 `AUTOREAD_UPDATER` 更新 `autoRead` 字段，并获得更新前的值。为什么需要获取更新前的值呢？在后续的 `<2.1>` 和 `<2.2>` 中，当 `autoRead` 有变化时候，才进行后续的逻辑。

  - 😈 下面的逻辑，我们按照 `channel` 的类型为 NioServerSocketChannel 来分享。

  - `<2.1>` 处，`autoRead && !oldAutoRead` 返回 `true` ，意味着恢复重启开启接受新的客户端连接。所以调用 `NioServerSocketChannel#read()` 方法，后续的逻辑，就是 [《精尽 Netty 源码分析 —— 启动（一）之服务端》](http://svip.iocoder.cn/Netty/bootstrap-1-server/) 的 [「3.13.3 beginRead」](http://svip.iocoder.cn/Netty/Channel-2-accept/#) 的逻辑。

  - `<2.2>` 处，`!autoRead && oldAutoRead` 返回 `false` ，意味着关闭接受新的客户端连接。所以调用 `#autoReadCleared()` 方法，移除对 `SelectionKey.OP_ACCEPT` 事件的感兴趣。

    ```
    // NioServerSocketChannel.java
    
    @Override
    protected void autoReadCleared() {
        clearReadPending();
    }
    ```

    - 在方法内部，会调用 `#clearReadPending()` 方法，代码如下：

      ```
      protected final void clearReadPending() {
          if (isRegistered()) {
              EventLoop eventLoop = eventLoop();
              if (eventLoop.inEventLoop()) {
                  clearReadPending0();
              } else {
                  eventLoop.execute(clearReadPendingRunnable);
              }
          } else {
              // Best effort if we are not registered yet clear readPending. This happens during channel initialization.
              // NB: We only set the boolean field instead of calling clearReadPending0(), because the SelectionKey is
              // not set yet so it would produce an assertion failure.
              readPending = false;
          }
      }
      
      private final Runnable clearReadPendingRunnable = new Runnable() {
          @Override
          public void run() {
              clearReadPending0();
          }
      };
      
      private void clearReadPending0() {
          // TODO 芋艿
          readPending = false;
          // 移除对“读”事件的感兴趣。
          ((AbstractNioUnsafe) unsafe()).removeReadOp();
      }
      ```

      - 最终的结果，是在 EventLoop 的线程中，调用 `AbstractNioUnsafe#clearReadPending0()` 方法，移除对“**读**”事件的感兴趣( 对于 NioServerSocketChannel 的 “**读**“事件就是 `SelectionKey.OP_ACCEPT` )。代码如下：

        ```
        // AbstractNioUnsafe.java
        
        protected final void removeReadOp() {
            SelectionKey key = selectionKey();
            // 忽略，如果 SelectionKey 不合法，例如已经取消
            // Check first if the key is still valid as it may be canceled as part of the deregistration
            // from the EventLoop
            // See https://github.com/netty/netty/issues/2104
            if (!key.isValid()) {
                return;
            }
            // 移除对“读”事件的感兴趣。
            int interestOps = key.interestOps();
            if ((interestOps & readInterestOp) != 0) {
                // only remove readInterestOp if needed
                key.interestOps(interestOps & ~readInterestOp);
            }
        }
        ```

        - 通过取反求并，后调用 `SelectionKey#interestOps(interestOps)` 方法，**仅**移除对“读”事件的感兴趣。
        - 😈 整个过程的调用链，有丢丢长，胖友可以回看，或者多多调试。

- 第 10 行：调用 `EventLoop#schedule(Runnable command, long delay, TimeUnit unit)` 方法，发起 1 秒的延迟任务，恢复重启开启接受新的客户端连接。该定时任务会调用 `ChannelConfig#setAutoRead(true)` 方法，即对应 `<2.1>` 情况。

- 第 16 行：调用 `ChannelHandlerContext#fireExceptionCaught(cause)` 方法，继续传播 exceptionCaught 给下一个节点。具体的原因，可看英文注释。

## 666. 彩蛋

推荐阅读文章：

- 闪电侠 [《netty 源码分析之新连接接入全解析》](https://www.jianshu.com/p/0242b1d4dd21)
- 占小狼 [《Netty 源码分析之 accept 过程》](https://www.jianshu.com/p/ffc6fd82e32b)

# 精尽 Netty 源码解析 —— Channel（三）之 read 操作

2018-07-04

## 1. 概述

本文分享 Netty NIO 服务端读取( **read** )来自客户端数据的过程、和 Netty NIO 客户端接收( **read** )来自服务端数据的结果。实际上，这两者的实现逻辑是一致的：

- 客户端就不用说了，自身就使用了 Netty NioSocketChannel 。
- 服务端在接受客户端连接请求后，会创建客户端对应的 Netty NioSocketChannel 。

因此，我们统一叫做 NioSocketChannel 读取( **read** )对端的数据的过程。

------

NioSocketChannel 读取( **read** )对端的数据的过程，简单来说：

1. NioSocketChannel 所在的 EventLoop 线程轮询是否有新的数据写入。
2. 当轮询到有新的数据写入，NioSocketChannel 读取数据，并提交到 pipeline 中进行处理。

比较简单，和 [《精尽 Netty 源码解析 —— Channel（二）之 accept 操作》](http://svip.iocoder.cn/Netty/Channel-2-accept) 有几分相似。或者我们可以说：

- NioServerSocketChannel 读取新的连接。
- NioSocketChannel 读取新的数据。

## 2. NioByteUnsafe#read

NioByteUnsafe ，实现 AbstractNioUnsafe 抽象类，AbstractNioByteChannel 的 Unsafe 实现类。代码如下：

```
protected class NioByteUnsafe extends AbstractNioUnsafe {

    public final void read() { /** 省略内部实现 **/ }

    private void handleReadException(ChannelPipeline pipeline, ByteBuf byteBuf, Throwable cause, boolean close, RecvByteBufAllocator.Handle allocHandle) { /** 省略内部实现 **/ }

    private void closeOnRead(ChannelPipeline pipeline) { /** 省略内部实现 **/ }

}
```

- 一共有 3 个方法。但是实现上，入口为 `#read()` 方法，而另外 2 个方法被它所调用。所以，我们赶紧开始 `#read()` 方法的理解吧。

### 2.1 read

在 NioEventLoop 的 `#processSelectedKey(SelectionKey k, AbstractNioChannel ch)` 方法中，我们会看到这样一段代码：

```
// SelectionKey.OP_READ 或 SelectionKey.OP_ACCEPT 就绪
// readyOps == 0 是对 JDK Bug 的处理，防止空的死循环
// Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
// to a spin loop
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    unsafe.read();
}
```

- 当 `(readyOps & SelectionKey.OP_READ) != 0` 时，这就是 NioSocketChannel 所在的 EventLoop 的线程**轮询到**有新的数据写入。
- 然后，调用 `NioByteUnsafe#read()` 方法，读取新的写入数据。

------

`NioByteUnsafe#read()` 方法，读取新的写入数据。代码如下：

```
 1: @Override
 2: @SuppressWarnings("Duplicates")
 3: public final void read() {
 4:     final ChannelConfig config = config();
 5:     // 若 inputClosedSeenErrorOnRead = true ，移除对 SelectionKey.OP_READ 事件的感兴趣。
 6:     if (shouldBreakReadReady(config)) {
 7:         clearReadPending();
 8:         return;
 9:     }
10:     final ChannelPipeline pipeline = pipeline();
11:     final ByteBufAllocator allocator = config.getAllocator();
12:     // 获得 RecvByteBufAllocator.Handle 对象
13:     final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
14:     // 重置 RecvByteBufAllocator.Handle 对象
15:     allocHandle.reset(config);
16: 
17:     ByteBuf byteBuf = null;
18:     boolean close = false; // 是否关闭连接
19:     try {
20:         do {
21:             // 申请 ByteBuf 对象
22:             byteBuf = allocHandle.allocate(allocator);
23:             // 读取数据
24:             // 设置最后读取字节数
25:             allocHandle.lastBytesRead(doReadBytes(byteBuf));
26:             // <1> 未读取到数据
27:             if (allocHandle.lastBytesRead() <= 0) {
28:                 // 释放 ByteBuf 对象
29:                 // nothing was read. release the buffer.
30:                 byteBuf.release();
31:                 // 置空 ByteBuf 对象
32:                 byteBuf = null;
33:                 // 如果最后读取的字节为小于 0 ，说明对端已经关闭
34:                 close = allocHandle.lastBytesRead() < 0;
35:                 // TODO
36:                 if (close) {
37:                     // There is nothing left to read as we received an EOF.
38:                     readPending = false;
39:                 }
40:                 // 结束循环
41:                 break;
42:             }
43: 
44:             // <2> 读取到数据
45: 
46:             // 读取消息数量 + localRead
47:             allocHandle.incMessagesRead(1);
48:             // TODO 芋艿 readPending
49:             readPending = false;
50:             // 触发 Channel read 事件到 pipeline 中。 TODO
51:             pipeline.fireChannelRead(byteBuf);
52:             // 置空 ByteBuf 对象
53:             byteBuf = null;
54:         } while (allocHandle.continueReading()); // 循环判断是否继续读取
55: 
56:         // 读取完成
57:         allocHandle.readComplete();
58:         // 触发 Channel readComplete 事件到 pipeline 中。
59:         pipeline.fireChannelReadComplete();
60: 
61:         // 关闭客户端的连接
62:         if (close) {
63:             closeOnRead(pipeline);
64:         }
65:     } catch (Throwable t) {
66:         handleReadException(pipeline, byteBuf, t, close, allocHandle);
67:     } finally {
68:         // TODO 芋艿 readPending
69:         // Check if there is a readPending which was not processed yet.
70:         // This could be for two reasons:
71:         // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
72:         // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
73:         //
74:         // See https://github.com/netty/netty/issues/2254
75:         if (!readPending && !config.isAutoRead()) {
76:             removeReadOp();
77:         }
78:     }
79: }
```

- 第 5 至 9 行：若 inputClosedSeenErrorOnRead = true ，移除对 SelectionKey.OP_READ 事件的感兴趣。详细解析，见 [《精尽 Netty 源码解析 —— Channel（七）之 close 操作》](http://svip.iocoder.cn/Netty/Channel-7-close/) 的 [「5. 服务端处理客户端主动关闭连接」](http://svip.iocoder.cn/Netty/Channel-3-read/#) 小节。

- 第 12 至 15 行：获得 RecvByteBufAllocator.Handle 对象，并重置它。这里的逻辑，和 `NioMessageUnsafe#read()` 方法的【第 14 至 17 行】的代码是一致的。相关的解析，见 [《精尽 Netty 源码解析 —— Channel（二）之 accept 操作》](http://svip.iocoder.cn/Netty/Channel-2-accept) 。

- 第 20 至 64 行：**while 循环** 读取新的写入数据。

  - 第 22 行：调用 `RecvByteBufAllocator.Handle#allocate(ByteBufAllocator allocator)` 方法，申请 ByteBuf 对象。关于它的内容，我们放在 ByteBuf 相关的文章，详细解析。

  - 第 25 行：调用 `AbstractNioByteChannel#doReadBytes(ByteBuf buf)` 方法，读取数据。详细解析，胖友先跳到 [「3. AbstractNioMessageChannel#doReadMessages」](http://svip.iocoder.cn/Netty/Channel-3-read/#) 中，看完记得回到此处。

  - 第 25 行：调用 `RecvByteBufAllocator.Handle#lastBytesRead(int bytes)` 方法，设置**最后**读取字节数。代码如下：

    ```
    // AdaptiveRecvByteBufAllocator.HandleImpl.java
    @Override
    public void lastBytesRead(int bytes) {
        // If we read as much as we asked for we should check if we need to ramp up the size of our next guess.
        // This helps adjust more quickly when large amounts of data is pending and can avoid going back to
        // the selector to check for more data. Going back to the selector can add significant latency for large
        // data transfers.
        if (bytes == attemptedBytesRead()) {
            record(bytes);
        }
        super.lastBytesRead(bytes);
    }
    
    // DefaultMaxMessagesRecvByteBufAllocator.MaxMessageHandle.java
    @Override
    public void lastBytesRead(int bytes) {
        lastBytesRead = bytes; // 设置最后一次读取字节数 <1>
        if (bytes > 0) {
            totalBytesRead += bytes; // 总共读取字节数
        }
    }
    ```

    - 代码比较多，我们只看重点，当然也不细讲。
    - 在 `<1>` 处，设置最后一次读取字节数。

  - 读取有，有两种结果，**是**/**否**读取到数据。

  - `<1>` **未**读取到数据，即 `allocHandle.lastBytesRead() <= 0` 。

  - 第 30 行：调用

     

    ```
    ByteBuf#release()
    ```

     

    方法，释放 ByteBuf 对象。

    - 第 32 行：置空 ByteBuf 对象。

  - 第 34 行：如果最后读取的字节为小于 0 ，说明对端已经关闭。

  - 第 35 至 39 行：TODO 芋艿 细节

  - 第 41 行：`break` 结束循环。

  - `<2>` **有**读取到数据，即 `allocHandle.lastBytesRead() > 0` 。

  - 第 47 行：调用 `AdaptiveRecvByteBufAllocator.HandleImpl#incMessagesRead(int amt)` 方法，读取消息( 客户端 )数量 + `localRead = 1` 。

  - 第 49 行：TODO 芋艿 readPending

  - 第 51 行：调用 `ChannelPipeline#fireChannelRead(Object msg)` 方法，触发 Channel read 事件到 pipeline 中。

    - **注意**，一般情况下，我们会在自己的 Netty 应用程序中，自定义 ChannelHandler 处理读取到的数据。😈 当然，此时读取的数据，大多数情况下是需要在解码( Decode )。关于这一块，在后续关于 Codec ( 编解码 )的文章中，详细解析。

    - 如果没有自定义 ChannelHandler 进行处理，最终会被 pipeline 中的尾节点 TailContext 所处理。代码如下：

      ```
      // TailContext.java
      @Override
      public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
          onUnhandledInboundMessage(msg);
      }
      
      // DefaultChannelPipeline.java
      protected void onUnhandledInboundMessage(Object msg) {
          try {
              logger.debug("Discarded inbound message {} that reached at the tail of the pipeline. " + "Please check your pipeline configuration.", msg);
          } finally {
              ReferenceCountUtil.release(msg);
          }
      }
      ```

      - 最终也会**释放** ByteBuf 对象。这就是为什么【第 53 行】的代码，只去置空 ByteBuf 对象，而不用再去释放的原因。

  - 第 53 行：置空 ByteBuf 对象。

  - 第 54 行：调用 `AdaptiveRecvByteBufAllocator.HandleImpl#incMessagesRead(int amt)#continueReading()` 方法，判断是否循环是否继续，读取新的数据。代码如下：

    ```
    // DefaultMaxMessagesRecvByteBufAllocator.MaxMessageHandle.java
    private final UncheckedBooleanSupplier defaultMaybeMoreSupplier = new UncheckedBooleanSupplier() {
        @Override
        public boolean get() {
            return attemptedBytesRead == lastBytesRead; // 最后读取的字节数，是否等于，最大可写入的字节数
        }
    };
    
    @Override
    public boolean continueReading() {
        return continueReading(defaultMaybeMoreSupplier);
    }
    
    @Override
    public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
        return config.isAutoRead() &&
               (!respectMaybeMoreData || maybeMoreDataSupplier.get()) && // <1>
               totalMessages < maxMessagePerRead &&
               totalBytesRead > 0;
    }
    ```

    - 一般情况下，最后读取的字节数，**不等于**最大可写入的字节数，即 `<1>` 处的代码 `UncheckedBooleanSupplier#get()` 返回 `false` ，则不再进行数据读取。因为 😈 也没有数据可以读取啦。

- 第 57 行：调用 `RecvByteBufAllocator.Handle#readComplete()` 方法，读取完成。暂无重要的逻辑，不详细解析。

- 第 59 行：调用 `ChannelPipeline#fireChannelReadComplete()` 方法，触发 Channel readComplete 事件到 pipeline 中。

  - *如果有需要，胖友可以自定义处理器，处理该事件。一般情况下，不需要*。

  - 如果没有自定义 ChannelHandler 进行处理，最终会被 pipeline 中的尾节点 TailContext 所处理。代码如下：

    ```
    // TailContext.java
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        onUnhandledInboundChannelReadComplete();
    }
    
    // DefaultChannelPipeline.java
    protected void onUnhandledInboundChannelReadComplete() {
    }
    ```

    - 具体的调用是**空方法**。

- 第 61 至 64 行：关闭客户端的连接。详细解析，见 [《精尽 Netty 源码解析 —— Channel（七）之 close 操作》](http://svip.iocoder.cn/Netty/Channel-7-close/) 的 [「5. 服务端处理客户端主动关闭连接」](http://svip.iocoder.cn/Netty/Channel-3-read/#) 小节。

- 第 65 至 66 行：当发生异常时，调用 `#handleReadException(hannelPipeline pipeline, ByteBuf byteBuf, Throwable cause, boolean close, RecvByteBufAllocator.Handle allocHandle)` 方法，处理异常。详细解析，见 [「2.2 handleReadException」](http://svip.iocoder.cn/Netty/Channel-3-read/#) 中。

- 第 67 至 78 行：TODO 芋艿 细节

### 2.2 handleReadException

`#handleReadException(hannelPipeline pipeline, ByteBuf byteBuf, Throwable cause, boolean close, RecvByteBufAllocator.Handle allocHandle)` 方法，处理异常。代码如下：

```
 1: private void handleReadException(ChannelPipeline pipeline, ByteBuf byteBuf, Throwable cause, boolean close, RecvByteBufAllocator.Handle allocHandle) {
 2:     if (byteBuf != null) {
 3:         if (byteBuf.isReadable()) {
 4:             // TODO 芋艿 细节
 5:             readPending = false;
 6:             // 触发 Channel read 事件到 pipeline 中。
 7:             pipeline.fireChannelRead(byteBuf);
 8:         } else {
 9:             // 释放 ByteBuf 对象
10:             byteBuf.release();
11:         }
12:     }
13:     // 读取完成
14:     allocHandle.readComplete();
15:     // 触发 Channel readComplete 事件到 pipeline 中。
16:     pipeline.fireChannelReadComplete();
17:     // 触发 exceptionCaught 事件到 pipeline 中。
18:     pipeline.fireExceptionCaught(cause);
19:     // // TODO 芋艿 细节
20:     if (close || cause instanceof IOException) {
21:         closeOnRead(pipeline);
22:     }
23: }
```

- 第 2 行：`byteBuf` 非空，说明在发生异常之前，至少申请 ByteBuf 对象是**成功**的。

  - 第 3 行：调用 `ByteBuf#isReadable()` 方法，判断 ByteBuf 对象是否可读，即剩余可读的字节数据。

    - 该方法的英文注释如下：

      ```
      /**
       * Returns {@code true}
       * if and only if {@code (this.writerIndex - this.readerIndex)} is greater
       * than {@code 0}.
       */
      public abstract boolean isReadable();
      ```

      - 即 `this.writerIndex - this.readerIndex > 0` 。

    - 第 5 行：TODO 芋艿 细节

    - 第 7 行：调用 `ChannelPipeline#fireChannelRead(Object msg)` 方法，触发 Channel read 事件到 pipeline 中。

  - 第 8 至 11 行：ByteBuf 对象不可读，所以调用 `ByteBuf#release()` 方法，释放 ByteBuf 对象。

- 第 14 行：调用 `RecvByteBufAllocator.Handle#readComplete()` 方法，读取完成。暂无重要的逻辑，不详细解析。

- 第 16 行：调用 `ChannelPipeline#fireChannelReadComplete()` 方法，触发 Channel readComplete 事件到 pipeline 中。

- 第 18 行：调用 `ChannelPipeline#fireExceptionCaught(Throwable)` 方法，触发 exceptionCaught 事件到 pipeline 中。

  - **注意**，一般情况下，我们会在自己的 Netty 应用程序中，自定义 ChannelHandler 处理异常。

  - 如果没有自定义 ChannelHandler 进行处理，最终会被 pipeline 中的尾节点 TailContext 所处理。代码如下：

    ```
    // TailContext.java
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        onUnhandledInboundException(cause);
    }
    
    // DefaultChannelPipeline.java
    protected void onUnhandledInboundException(Throwable cause) {
        try {
            logger.warn("An exceptionCaught() event was fired, and it reached at the tail of the pipeline. " +
                            "It usually means the last handler in the pipeline did not handle the exception.",
                    cause);
        } finally {
            ReferenceCountUtil.release(cause);
        }
    }
    ```

    - 打印**告警**日志。
    - 调用 `ReferenceCountUtil#release(Object msg)` 方法，释放和异常相关的资源。

- 第 19 至 22 行：TODO 芋艿，细节

### 2.3 closeOnRead

TODO 芋艿，细节

## 3. AbstractNioByteChannel#doReadBytes

`doReadBytes(ByteBuf buf)` **抽象**方法，读取写入的数据到方法参数 `buf` 中。它是一个**抽象**方法，定义在 AbstractNioByteChannel 抽象类中。代码如下：

```
/**
 * Read bytes into the given {@link ByteBuf} and return the amount.
 */
protected abstract int doReadBytes(ByteBuf buf) throws Exception;
```

- 返回值为读取到的字节数。
- **当返回值小于 0 时，表示对端已经关闭**。

NioSocketChannel 对该方法的实现代码如下：

```
1: @Override
2: protected int doReadBytes(ByteBuf byteBuf) throws Exception {
3:     // 获得 RecvByteBufAllocator.Handle 对象
4:     final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
5:     // 设置最大可读取字节数量。因为 ByteBuf 目前最大写入的大小为 byteBuf.writableBytes()
6:     allocHandle.attemptedBytesRead(byteBuf.writableBytes());
7:     // 读取数据到 ByteBuf 中
8:     return byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());
9: }
```

- 第 4 行：获得 RecvByteBufAllocator.Handle 对象。

  - 第 6 行：设置最大可读取字节数量。因为 ByteBuf 对象**目前**最大可写入的大小为 `ByteBuf#writableBytes()` 的长度。

- 第 8 行：调用 `ByteBuf#writeBytes(ScatteringByteChannel in, int length)` 方法，读取数据到 ByteBuf 对象中。因为 ByteBuf 有多种实现，我们以默认的 PooledUnsafeDirectByteBuf 举例子。代码如下：

  ```
  // AbstractByteBuf.java
  @Override
  public int writeBytes(ScatteringByteChannel in, int length) throws IOException {
      ensureWritable(length);
      int writtenBytes = setBytes(writerIndex, in, length); // <1>
      if (writtenBytes > 0) { // <3>
          writerIndex += writtenBytes;
      }
      return writtenBytes;
  }
  
  // PooledUnsafeDirectByteBuf.java
  @Override
  public int setBytes(int index, ScatteringByteChannel in, int length) throws IOException {
      checkIndex(index, length);
      ByteBuffer tmpBuf = internalNioBuffer();
      index = idx(index);
      tmpBuf.clear().position(index).limit(index + length);
      try {
          return in.read(tmpBuf); // <2>
      } catch (ClosedChannelException ignored) {
          return -1;
      }
  }
  ```

  - 代码比较多，我们只看重点，当然也不细讲。还是那句话，关于 ByteBuf 的内容，我们在 ByteBuf 相关的文章详细解析。

  - 在 `<1>` 处，会调用 `#setBytes(int index, ScatteringByteChannel in, int length)` 方法。

  - 在

     

    ```
    <2>
    ```

     

    处，会调用 Java NIO 的

     

    ```
    ScatteringByteChannel#read(ByteBuffer)
    ```

     

    方法，读取

    数据

    到临时的 Java NIO ByteBuffer 中。

    - 在对端未断开时，返回的是读取数据的**字节数**。
    - 在对端已断开时，返回 `-1` ，表示断开。这也是为什么 `<3>` 处做了 `writtenBytes > 0` 的判断的原因。

## 666. 彩蛋

推荐阅读文章：

- 闪电侠 [《深入浅出 Netty read》](https://www.jianshu.com/p/6b48196b5043)
- Hypercube [《自顶向下深入分析Netty（六）– Channel源码实现》](https://www.jianshu.com/p/9258af254e1d)



# 精尽 Netty 源码解析 —— Channel（四）之 write 操作

2018-07-10

## 1. 概述

本文分享 Netty NioSocketChannel **写入**对端数据的过程。和**写入**相关的，在 Netty Channel 有三种 API 方法：

```
ChannelFuture write(Object msg)
ChannelFuture write(Object msg, ChannelPromise promise);

ChannelOutboundInvoker flush();

ChannelFuture writeAndFlush(Object msg);
ChannelFuture writeAndFlush(Object msg, ChannelPromise promise);
```

原生的 Java NIO SocketChannel 只有一种 write 方法，将数据写到对端。而 Netty Channel 竟然有三种方法，我们来一个个看看：

- write 方法：将数据写到

  内存队列

  中。

  - 也就是说，此时数据**并没有**写入到对端。

- flush 方法：刷新

  内存队列

  ，将其中的数据写入到对端。

  - 也就是说，此时数据才**真正**写到对端。

- writeAndFlush 方法：write + flush 的组合，将数据写到内存队列后，立即刷新

  内存队列

  ，又将其中的数据写入到对端。

  - 也就是说，此时数据**已经**写到对端。

严格来说，上述的描述不是完全准确。因为 Netty Channel 的 `#write(Object msg, ...)` 和 `#writeAndFlush(Object msg, ...)` 方法，是**异步写入**的过程，需要通过监听返回的 ChannelFuture 来确实是真正写入。例如：

```
// 方式一：异步监听
channel.write(msg).addListener(new ChannelFutureListener() {
                
    @Override
    public void operationComplete(ChannelFuture future) throws Exception {
        // ... 相关逻辑，例如是否成功？    
    }
    
});

// 方式二：同步异步写入结果
channel.write(msg).sync();
```

- 所以，胖友实际在使用时，一定要注意。😈 如果感兴趣，可以看看 Dubbo 和 Motan 等等 RPC 框架是怎么使用这个 API 方法的。
- 😈 **有一点一定非常肯定要注意**，`#write(Object msg, ...)` 方法返回的 Promise 对象，只有在数据真正被 `#flush()` 方法调用执行完成后，才会被回调通知。如果胖友不理解，请自己测试一下。

------

考虑到 Netty NioSocketChannel **写入**对端数据的代码太多，所以笔者拆成 write 和 flush 相关的两篇文章。所以，本文当然是 write 相关的文章。当然，这两个操作相关性很高，所以本文也会包括 flush 部分的内容。

## 2. AbstractChannel

AbstractChannel 对 `#write(Object msg, ...)` 方法的实现，代码如下：

```
@Override
public ChannelFuture write(Object msg) {
    return pipeline.write(msg);
}

@Override
public ChannelFuture write(Object msg, ChannelPromise promise) {
    return pipeline.write(msg, promise);
}
```

- 在方法内部，会调用对应的

   

  ```
  ChannelPipeline#write(Object msg, ...)
  ```

   

  方法，将 write 事件在 pipeline 上传播。详细解析，见

   

  「3. DefaultChannelPipeline」

   

  。

  - 最终会传播 write 事件到 `head` 节点，将数据写入到内存队列中。详细解析，见 [「5. HeadContext」](http://svip.iocoder.cn/Netty/Channel-4-write/#) 。

## 3. DefaultChannelPipeline

`DefaultChannelPipeline#write(Object msg, ...)` 方法，代码如下：

```
@Override
public final ChannelFuture write(Object msg) {
    return tail.write(msg);
}

@Override
public final ChannelFuture write(Object msg, ChannelPromise promise) {
    return tail.write(msg, promise);
}
```

- 在方法内部，会调用 `TailContext#write(Object msg, ...)` 方法，将 write 事件在 pipeline 中，从尾节点向头节点传播。详细解析，见 [「4. TailContext」](http://svip.iocoder.cn/Netty/Channel-4-write/#) 。

## 4. TailContext

TailContext 对 `TailContext#write(Object msg, ...)` 方法的实现，是从 AbstractChannelHandlerContext 抽象类继承，代码如下：

```
 1: @Override
 2: public ChannelFuture write(Object msg) {
 3:     return write(msg, newPromise());
 4: }
 5: 
 6: @Override
 7: public ChannelFuture write(final Object msg, final ChannelPromise promise) {
 8:     // 消息( 数据 )为空，抛出异常
 9:     if (msg == null) {
10:         throw new NullPointerException("msg");
11:     }
12: 
13:     try {
14:         // 判断是否为合法的 Promise 对象
15:         if (isNotValidPromise(promise, true)) {
16:             // 释放消息( 数据 )相关的资源
17:             ReferenceCountUtil.release(msg);
18:             // cancelled
19:             return promise;
20:         }
21:     } catch (RuntimeException e) {
22:         // 发生异常，释放消息( 数据 )相关的资源
23:         ReferenceCountUtil.release(msg);
24:         throw e;
25:     }
26: 
27:     // 写入消息( 数据 )到内存队列
28:     write(msg, false, promise);
29:     return promise;
30: }
```

- 在【第 2 行】的代码，我们可以看到，`#write(Object msg)` 方法，会调用 `#write(Object msg, ChannelPromise promise)` 方法。

  - 缺少的 `promise` 方法参数，通过调用 `#newPromise()` 方法，进行创建 Promise 对象，代码如下：

    ```
    @Override
    public ChannelPromise newPromise() {
        return new DefaultChannelPromise(channel(), executor());
    }
    ```

    - 返回 DefaultChannelPromise 对象。

  - 在【第 29 行】的代码，返回的结果就是传入的 `promise` 对象。

- 第 8 至 11 行：若消息( 消息 )为空，抛出异常。

- 第 15 行：调用

   

  ```
  #isNotValidPromise(promise, true)
  ```

   

  方法，判断是否为

  不合法

  的 Promise 对象。该方法，在

   

  《精尽 Netty 源码解析 —— ChannelPipeline（四）之 Outbound 事件的传播》

   

  中已经详细解析。

  - 第 17 行：调用 `ReferenceCountUtil#release(msg)` 方法，释放释放消息( 数据 )相关的资源。
  - 第 19 行：返回 `promise` 对象。一般情况下，出现这种情况是 `promise` 已经被取消，所以不再有必要写入数据。或者说，**写入数据的操作被取消**。
  - 第 21 至 25 行：若发生异常， 调用 `ReferenceCountUtil#release(msg)` 方法，释放释放消息( 数据 )相关的资源。最终，会抛出该异常。

- 第 28 行：调用 `#write(Object msg, boolean flush, ChannelPromise promise)` 方法，写入消息( 数据 )到内存队列。代码如下：

  ```
   1: private void write(Object msg, boolean flush, ChannelPromise promise) {
   2:     // 获得下一个 Outbound 节点
   3:     AbstractChannelHandlerContext next = findContextOutbound();
   4:     // 记录 Record 记录
   5:     final Object m = pipeline.touch(msg, next);
   6:     EventExecutor executor = next.executor();
   7:     // 在 EventLoop 的线程中
   8:     if (executor.inEventLoop()) {
   9:         // 执行 writeAndFlush 事件到下一个节点
  10:         if (flush) {
  11:             next.invokeWriteAndFlush(m, promise);
  12:         // 执行 write 事件到下一个节点
  13:         } else {
  14:             next.invokeWrite(m, promise);
  15:         }
  16:     } else {
  17:         AbstractWriteTask task;
  18:         // 创建 writeAndFlush 任务
  19:         if (flush) {
  20:             task = WriteAndFlushTask.newInstance(next, m, promise);
  21:         // 创建 write 任务
  22:         }  else {
  23:             task = WriteTask.newInstance(next, m, promise);
  24:         }
  25:         // 提交到 EventLoop 的线程中，执行该任务
  26:         safeExecute(executor, task, promise, m);
  27:     }
  28: }
  ```

  - 方法参数 `flush` 为 `true` 时，该方法执行的是 write + flush 的组合操作，即将数据写到内存队列后，立即刷新**内存队列**，又将其中的数据写入到对端。

  - 第 3 行：调用 `#findContextOutbound()` 方法，获得**下一个** Outbound 节点。

  - 第 5 行：调用 `DefaultChannelPipeline#touch(Object msg, AbstractChannelHandlerContext next)` 方法，记录 Record 记录。代码如下：

    ```
    // DefaultChannelPipeline.java
    final Object touch(Object msg, AbstractChannelHandlerContext next) {
        return touch ? ReferenceCountUtil.touch(msg, next) : msg;
    }
    
    // ReferenceCountUtil.java
    /**
     * Tries to call {@link ReferenceCounted#touch(Object)} if the specified message implements
     * {@link ReferenceCounted}.  If the specified message doesn't implement {@link ReferenceCounted},
     * this method does nothing.
     */
    @SuppressWarnings("unchecked")
    public static <T> T touch(T msg, Object hint) {
        if (msg instanceof ReferenceCounted) {
            return (T) ((ReferenceCounted) msg).touch(hint);
        }
        return msg;
    }
    ```

    - 详细解析，见 [《精尽 Netty 源码解析 —— Buffer 之 ByteBuf（三）内存泄露检测》](http://svip.iocoder.cn/Netty/ByteBuf-1-3-ByteBuf-resource-leak-detector/) 。

  - 第 7 行：**在** EventLoop 的线程中。

    - 第 10 至 11 行：如果 `flush = true` 时，调用 `AbstractChannelHandlerContext#invokeWriteAndFlush()` 方法，执行 writeAndFlush 事件到下一个节点。
    - 第 12 至 15 行：如果 `flush = false` 时，调用 `AbstractChannelHandlerContext#invokeWrite()` 方法，执行 write 事件到下一个节点。
    - 后续的逻辑，和 [《精尽 Netty 源码解析 —— ChannelPipeline（四）之 Outbound 事件的传播》](http://svip.iocoder.cn/Netty/Pipeline-4-outbound/) 分享的 **bind** 事件在 pipeline 中的传播是**基本一致**的。
    - 随着 write 或 writeAndFlush **事件**不断的向下一个节点传播，最终会到达 HeadContext 节点。详细解析，见 [「5. HeadContext」](http://svip.iocoder.cn/Netty/Channel-4-write/#) 。

  - 第 16 行：

    不在

     

    EventLoop 的线程中。

    - 第 19 至 20 行：如果 `flush = true` 时，创建 WriteAndFlushTask 任务。
    - 第 21 至 24 行：如果 `flush = false` 时，创建 WriteTask 任务。
    - 第 26 行：调用 `#safeExecute(executor, task, promise, m)` 方法，提交到 EventLoop 的线程中，执行该任务。从而实现，**在** EventLoop 的线程中，执行 writeAndFlush 或 write 事件到下一个节点。详细解析，见 [「7. AbstractWriteTask」](http://svip.iocoder.cn/Netty/Channel-4-write/#) 中。

- 第 29 行：返回 `promise` 对象。

## 5. HeadContext

在 pipeline 中，write 事件最终会到达 HeadContext 节点。而 HeadContext 的 `#write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise)` 方法，会处理该事件，代码如下：

```
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}
```

- 在方法内部，会调用 `AbstractUnsafe#write(Object msg, ChannelPromise promise)` 方法，将数据写到**内存队列**中。详细解析，见 [「6. AbstractUnsafe」](http://svip.iocoder.cn/Netty/Channel-4-write/#) 。

## 6. AbstractUnsafe

`AbstractUnsafe#write(Object msg, ChannelPromise promise)` 方法，将数据写到**内存队列**中。代码如下：

```
/**
 * 内存队列
 */
private volatile ChannelOutboundBuffer outboundBuffer = new ChannelOutboundBuffer(AbstractChannel.this);

  1: @Override
  2: public final void write(Object msg, ChannelPromise promise) {
  3:     assertEventLoop();
  4: 
  5:     ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
  6:     // 内存队列为空
  7:     if (outboundBuffer == null) {
  8:         // 内存队列为空，一般是 Channel 已经关闭，所以通知 Promise 异常结果
  9:         // If the outboundBuffer is null we know the channel was closed and so
 10:         // need to fail the future right away. If it is not null the handling of the rest
 11:         // will be done in flush0()
 12:         // See https://github.com/netty/netty/issues/2362
 13:         safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
 14:         // 释放消息( 对象 )相关的资源
 15:         // release message now to prevent resource-leak
 16:         ReferenceCountUtil.release(msg);
 17:         return;
 18:     }
 19: 
 20:     int size;
 21:     try {
 22:         // 过滤写入的消息( 数据 )
 23:         msg = filterOutboundMessage(msg);
 24:         // 计算消息的长度
 25:         size = pipeline.estimatorHandle().size(msg);
 26:         if (size < 0) {
 27:             size = 0;
 28:         }
 29:     } catch (Throwable t) {
 30:         // 通知 Promise 异常结果
 31:         safeSetFailure(promise, t);
 32:         // 释放消息( 对象 )相关的资源
 33:         ReferenceCountUtil.release(msg);
 34:         return;
 35:     }
 36: 
 37:     // 写入消息( 数据 )到内存队列
 38:     outboundBuffer.addMessage(msg, size, promise);
 39: }
```

- `outboundBuffer` 属性，内存队列，用于缓存写入的数据( 消息 )。

- 第 7 行：内存队列为空，一般是 Channel

   

  已经关闭

  。

  - 调用 `#safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION)` 方法，通知 Promise 异常结果。
  - 第 16 行：调用 `ReferenceCountUtil#release(msg)` 方法，释放释放消息( 数据 )相关的资源。
  - 第 17 行：`return` ，结束执行。

- 第 23 行：调用 `AbstractNioByteChannel#filterOutboundMessage(msg)` 方法，过滤写入的消息( 数据 )。代码如下：

  ```
  // AbstractNioByteChannel.java
  
  @Override
  protected final Object filterOutboundMessage(Object msg) {
      // <1> ByteBuf 的情况
      if (msg instanceof ByteBuf) {
          ByteBuf buf = (ByteBuf) msg;
          // 已经是内存 ByteBuf
          if (buf.isDirect()) {
              return msg;
          }
  
          // 非内存 ByteBuf ，需要进行创建封装
          return newDirectBuffer(buf);
      }
  
      // <2> FileRegion 的情况
      if (msg instanceof FileRegion) {
          return msg;
      }
  
      // <3> 不支持其他类型
      throw new UnsupportedOperationException("unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
  }
  ```

  - `<1>` 处，消息( 数据 )是 ByteBuf 类型，如果是非 Direct ByteBuf 对象，需要调用 `#newDirectBuffer(ByteBuf)` 方法，复制封装成 Direct ByteBuf 对象。原因是：在使用 Socket 传递数据时性能很好，由于数据直接在内存中，不存在从 JVM 拷贝数据到直接缓冲区的过程，性能好。( 来自 [《[netty核心类\]–缓冲区ByteBuf》](https://blog.csdn.net/u010853261/article/details/53690780) )
  - `<2>` 处，消息( 数据 )是 FileRegion 类型，直接返回。
  - `<3>` 处，不支持其他数据类型。

- 第 24 至 28 行：计算消息的长度。

- 第 29 行：若发生异常时：

  - 第 31 行：调用 `#safeSetFailure(promise, Throwable t)` 方法，通知 Promise 异常结果。
  - 第 33 行：调用 `ReferenceCountUtil#release(msg)` 方法，释放释放消息( 数据 )相关的资源。
  - 第 34 行：`return` ，结束执行。

- 第 38 行：调用 `ChannelOutboundBuffer#addMessage(msg, size, promise)` 方法，写入消息( 数据 )到内存队列。关于 ChannelOutboundBuffer ，我们在 [《精尽 Netty 源码解析 —— Channel（五）之 flush 操作》](http://svip.iocoder.cn/Netty/Channel-5-flush) 中，详细解析。

😈 至此，write 操作，将数据写到**内存队列**中的过程。
🙂 当然，想要写入数据到对端的过程，还是需要看完 [《精尽 Netty 源码解析 —— Channel（五）之 flush 操作》](http://svip.iocoder.cn/Netty/Channel-5-flush) 一文。

## 7. AbstractWriteTask

AbstractWriteTask ，实现 Runnable 接口，写入任务**抽象类**。它有两个子类实现：

- WriteTask ，write 任务实现类。
- WriteAndFlushTask ，write + flush 任务实现类。

它们都是 AbstractChannelHandlerContext 的内部静态类。那么让我们先来 AbstractWriteTask 的代码。

### 7.1 构造方法

```
/**
 * 提交任务时，是否计算 AbstractWriteTask 对象的自身占用内存大小
 */
private static final boolean ESTIMATE_TASK_SIZE_ON_SUBMIT = SystemPropertyUtil.getBoolean("io.netty.transport.estimateSizeOnSubmit", true);

/**
 * 每个 AbstractWriteTask 对象自身占用内存的大小。
 */
// Assuming a 64-bit JVM, 16 bytes object header, 3 reference fields and one int field, plus alignment
private static final int WRITE_TASK_OVERHEAD = SystemPropertyUtil.getInt("io.netty.transport.writeTaskSizeOverhead", 48);

private final Recycler.Handle<AbstractWriteTask> handle;
/**
 * pipeline 中的节点
 */
private AbstractChannelHandlerContext ctx;
/**
 * 消息( 数据 )
 */
private Object msg;
/**
 * Promise 对象
 */
private ChannelPromise promise;
/**
 * 对象大小
 */
private int size;

@SuppressWarnings("unchecked")
private AbstractWriteTask(Recycler.Handle<? extends AbstractWriteTask> handle) {
    this.handle = (Recycler.Handle<AbstractWriteTask>) handle;
}
```

- 每个字段，看代码注释。

- `ESTIMATE_TASK_SIZE_ON_SUBMIT` **静态**字段，提交任务时，是否计算 AbstractWriteTask 对象的自身占用内存大小。

- ```
  WRITE_TASK_OVERHEAD
  ```

   

  静态

  字段，每个 AbstractWriteTask 对象自身占用内存的大小。为什么占用的 48 字节呢？

  - `- 16 bytes object header` ，对象头，16 字节。
  - `- 3 reference fields` ，3 个**对象引用**字段，3 * 8 = 24 字节。
  - `- 1 int fields` ，1 个 `int` 字段，4 字节。
  - `padding` ，补齐 8 字节的整数倍，因此 4 字节。
  - 因此，合计 48 字节( 64 位的 JVM 虚拟机，并且不考虑压缩 )。
  - 如果不理解的胖友，可以看看 [《JVM中 对象的内存布局 以及 实例分析》](https://www.jianshu.com/p/12a3c97dc2b7) 。

- `handle` 字段，Recycler 处理器。而 Recycler 是 Netty 用来实现对象池的工具类。在网络通信中，写入是非常频繁的操作，因此通过 Recycler 重用 AbstractWriteTask 对象，减少对象的频繁创建，降低 GC 压力，提升性能。

### 7.2 init

`#init(AbstractWriteTask task, AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise)` 方法，初始化 AbstractWriteTask 对象。代码如下：

```
protected static void init(AbstractWriteTask task, AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    task.ctx = ctx;
    task.msg = msg;
    task.promise = promise;
    // 计算 AbstractWriteTask 对象大小 <1>
    if (ESTIMATE_TASK_SIZE_ON_SUBMIT) {
        task.size = ctx.pipeline.estimatorHandle().size(msg) + WRITE_TASK_OVERHEAD;
        // 增加 ChannelOutboundBuffer 的 totalPendingSize 属性  <2>
        ctx.pipeline.incrementPendingOutboundBytes(task.size);
    } else {
        task.size = 0;
    }
}
```

- 在下文中，我们会看到 AbstractWriteTask 对象是从 Recycler 中获取，所以获取完成后，需要通过该方法，初始化该对象的属性。

- `<1>` 处，计算 AbstractWriteTask 对象大小。并且在 `<2>` 处，调用 `ChannelPipeline#incrementPendingOutboundBytes(long size)` 方法，增加 ChannelOutboundBuffer 的 `totalPendingSize` 属性。代码如下：

  ```
  // DefaultChannelPipeline.java
  @UnstableApi
  protected void incrementPendingOutboundBytes(long size) {
      ChannelOutboundBuffer buffer = channel.unsafe().outboundBuffer();
      if (buffer != null) {
          buffer.incrementPendingOutboundBytes(size);
      }
  }
  ```

  - 内部，会调用 `ChannelOutboundBuffer#incrementPendingOutboundBytes(long size)` 方法，增加 ChannelOutboundBuffer 的 `totalPendingSize` 属性。详细解析，见 [《精尽 Netty 源码解析 —— Channel（五）之 flush 操作》](http://svip.iocoder.cn/Netty/Channel-5-flush) 的 [「10.1 incrementPendingOutboundBytes」](http://svip.iocoder.cn/Netty/Channel-4-write/#) 小节。

### 7.3 run

`#run()` **实现**方法，

```
 1: @Override
 2: public final void run() {
 3:     try {
 4:         // 减少 ChannelOutboundBuffer 的 totalPendingSize 属性 <1>
 5:         // Check for null as it may be set to null if the channel is closed already
 6:         if (ESTIMATE_TASK_SIZE_ON_SUBMIT) {
 7:             ctx.pipeline.decrementPendingOutboundBytes(size);
 8:         }
 9:         // 执行 write 事件到下一个节点
10:         write(ctx, msg, promise);
11:     } finally {
12:         // 置空，help gc
13:         // Set to null so the GC can collect them directly
14:         ctx = null;
15:         msg = null;
16:         promise = null;
17:         // 回收对象
18:         handle.recycle(this);
19:     }
20: }
```

- 在 `<1>` 处， 调用 `ChannelPipeline#decrementPendingOutboundBytes(long size)` 方法，减少 ChannelOutboundBuffer 的 `totalPendingSize` 属性。代码如下：

  ```
  @UnstableApi
  protected void decrementPendingOutboundBytes(long size) {
      ChannelOutboundBuffer buffer = channel.unsafe().outboundBuffer();
      if (buffer != null) {
          buffer.decrementPendingOutboundBytes(size);
      }
  }
  ```

  - 内部，会调用 `ChannelOutboundBuffer#decrementPendingOutboundBytes(long size)` 方法，减少 ChannelOutboundBuffer 的 `totalPendingSize` 属性。详细解析，见 [《精尽 Netty 源码解析 —— Channel（五）之 flush 操作》](http://svip.iocoder.cn/Netty/Channel-5-flush) 的 [「10.2 decrementPendingOutboundBytes」](http://svip.iocoder.cn/Netty/Channel-4-write/#) 小节。

- 第 10 行：调用 `#write(ctx, msg, promise)` 方法，执行 write 事件到下一个节点。代码如下：

  ```
  protected void write(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
      ctx.invokeWrite(msg, promise);
  }
  ```

- 第 11 至 19 行：置空 AbstractWriteTask 对象，并调用 `Recycler.Handle#recycle(this)` 方法，回收该对象。

### 7.4 WriteTask

WriteTask ，实现 SingleThreadEventLoop.NonWakeupRunnable 接口，继承 AbstractWriteTask 抽象类，write 任务实现类。

**为什么会实现 SingleThreadEventLoop.NonWakeupRunnable 接口呢**？write 操作，仅仅将数据写到**内存队列**中，无需唤醒 EventLoop ，从而提升性能。关于 SingleThreadEventLoop.NonWakeupRunnable 接口，在 [《精尽 Netty 源码解析 —— EventLoop（三）之 EventLoop 初始化》](http://svip.iocoder.cn/Netty/EventLoop-3-EventLoop-init) 有详细解析。

#### 7.4.1 newInstance

`#newInstance(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise)` 方法，创建 WriteTask 对象。代码如下：

```
private static final Recycler<WriteTask> RECYCLER = new Recycler<WriteTask>() {

    @Override
    protected WriteTask newObject(Handle<WriteTask> handle) {
        return new WriteTask(handle); // 创建 WriteTask 对象
    }

};

private static WriteTask newInstance(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    // 从 Recycler 的对象池中获得 WriteTask 对象
    WriteTask task = RECYCLER.get();
    // 初始化 WriteTask 对象的属性
    init(task, ctx, msg, promise);
    return task;
}
```

#### 7.4.2 构造方法

```
private WriteTask(Recycler.Handle<WriteTask> handle) {
    super(handle);
}
```

#### 7.4.3 write

WriteTask 无需实现 `#write(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise)` 方法，直接**重用**父类该方法即可。

### 7.5 WriteAndFlushTask

WriteAndFlushTask ，继承 WriteAndFlushTask 抽象类，write + flush 任务实现类。

#### 7.5.1 newInstance

`#newInstance(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise)` 方法，创建 WriteAndFlushTask 对象。代码如下：

```
private static final Recycler<WriteAndFlushTask> RECYCLER = new Recycler<WriteAndFlushTask>() {

    @Override
    protected WriteAndFlushTask newObject(Handle<WriteAndFlushTask> handle) {
        return new WriteAndFlushTask(handle); // 创建 WriteAndFlushTask 对象
    }

};

private static WriteAndFlushTask newInstance(AbstractChannelHandlerContext ctx, Object msg,  ChannelPromise promise) {
    // 从 Recycler 的对象池中获得 WriteTask 对象
    WriteAndFlushTask task = RECYCLER.get();
    // 初始化 WriteTask 对象的属性
    init(task, ctx, msg, promise);
    return task;
}
```

#### 7.5.2 构造方法

```
private WriteAndFlushTask(Recycler.Handle<WriteAndFlushTask> handle) {
    super(handle);
}
```

#### 7.5.3 write

`#write(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise)` 方法，在父类的该方法的基础上，增加执行 **flush** 事件到下一个节点。代码如下：

```
@Override
public void write(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    // 执行 write 事件到下一个节点
    super.write(ctx, msg, promise);
    // 执行 flush 事件到下一个节点
    ctx.invokeFlush();
}
```

## 666. 彩蛋

最后，我们来看一个真的彩蛋，嘿嘿嘿。

在一些 ChannelHandler 里，我们想要写入数据到对端，可以有两种写法，代码如下：

```
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ctx.write(msg); // <1>
    ctx.channel().write(msg); // <2>
}
```

这两者有什么异同呢？

- `<2>` 种，实际就是本文所描述的，将 write 事件，从 pipeline 的 `tail` 节点到 `head` 节点的过程。
- `<1>` 种，和 `<2>` 种**不同**，将 write 事件，从当前的 `ctx` 节点的**下一个**节点到 `head` 节点的过程。
- 为什么呢？胖友自己调试理解下。😁😁😁

------

推荐阅读文章：

- 占小狼 [《深入浅出Netty write》](https://www.jianshu.com/p/1ad424c53e80)
- 闪电侠 [《netty 源码分析之 writeAndFlush 全解析》](https://www.jianshu.com/p/feaeaab2ce56)

# 精尽 Netty 源码解析 —— Channel（五）之 flush 操作

2018-07-13

## 1. 概述

本文接 [《精尽 Netty 源码解析 —— Channel（四）之 write 操作》](http://svip.iocoder.cn/Netty/Channel-4-write/) ，分享 Netty Channel 的 `#flush()` 方法，刷新**内存队列**，将其中的数据写入到对端。

在本文中，我们会发现，`#flush()` 方法和 `#write(Object msg, ...)` **正常**情况下，经历的流程是**差不多**的，例如在 pipeline 中对事件的传播，从 `tail` 节点传播到 `head` 节点，最终交由 Unsafe 处理，而差异点就是 Unsafe 的处理方式**不同**：

- write 方法：将数据写到**内存队列**中。
- flush 方法：刷新**内存队列**，将其中的数据写入到对端。

当然，上述描述仅仅指的是**正常**情况下，在**异常**情况下会有所不同。我们知道，Channel 大多数情况下是**可写**的，所以不需要专门去注册 `SelectionKey.OP_WRITE` 事件。所以在 Netty 的实现中，默认 Channel 是**可写**的，当写入失败的时候，再去注册 `SelectionKey.OP_WRITE` 事件。这意味着什么呢？在 `#flush()` 方法中，如果写入数据到 Channel 失败，会通过注册 `SelectionKey.OP_WRITE` 事件，然后在轮询到 Channel **可写** 时，再“回调” `#forceFlush()` 方法。

是不是非常巧妙？！让我直奔代码，大口吃肉，潇洒撸码。

> 下文的 [「2.」](http://svip.iocoder.cn/Netty/Channel-5-flush/#)、[「3.」](http://svip.iocoder.cn/Netty/Channel-5-flush/#)、[「4.」](http://svip.iocoder.cn/Netty/Channel-5-flush/#)、[「5.」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 和 [《精尽 Netty 源码解析 —— Channel（四）之 write 操作》](http://svip.iocoder.cn/Netty/Channel-4-write) 非常**类似**，所以胖友可以快速浏览。真正的**差异**，从 [「6.」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 开始。

## 2. AbstractChannel

AbstractChannel 对 `#flush()` 方法的实现，代码如下：

```
@Override
public Channel flush() {
    pipeline.flush();
    return this;
}
```

- 在方法内部，会调用对应的

   

  ```
  ChannelPipeline#flush()
  ```

   

  方法，将 flush 事件在 pipeline 上传播。详细解析，见

   

  「3. DefaultChannelPipeline」

   

  。

  - 最终会传播 flush 事件到 `head` 节点，刷新**内存队列**，将其中的数据写入到对端。详细解析，见 [「5. HeadContext」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

## 3. DefaultChannelPipeline

`DefaultChannelPipeline#flush()` 方法，代码如下：

```
@Override
public final ChannelPipeline flush() {
    tail.flush();
    return this;
}
```

- 在方法内部，会调用 `TailContext#flush()` 方法，将 flush 事件在 pipeline 中，从尾节点向头节点传播。详细解析，见 [「4. TailContext」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

## 4. TailContext

TailContext 对 `TailContext#flush()` 方法的实现，是从 AbstractChannelHandlerContext 抽象类继承，代码如下：

```
 1: @Override
 2: public ChannelHandlerContext flush() {
 3:     // 获得下一个 Outbound 节点
 4:     final AbstractChannelHandlerContext next = findContextOutbound();
 5:     EventExecutor executor = next.executor();
 6:     // 在 EventLoop 的线程中
 7:     if (executor.inEventLoop()) {
 8:         // 执行 flush 事件到下一个节点
 9:         next.invokeFlush();
10:     // 不在 EventLoop 的线程中
11:     } else {
12:         // 创建 flush 任务
13:         Runnable task = next.invokeFlushTask;
14:         if (task == null) {
15:             next.invokeFlushTask = task = new Runnable() {
16:                 @Override
17:                 public void run() {
18:                     next.invokeFlush();
19:                 }
20:             };
21:         }
22:         // 提交到 EventLoop 的线程中，执行该任务
23:         safeExecute(executor, task, channel().voidPromise(), null);
24:     }
25: 
26:     return this;
27: }
```

- 第 4 行：调用 `#findContextOutbound()` 方法，获得**下一个** Outbound 节点。

- 第 7 行：

  在

   

  EventLoop 的线程中。

  - 第 12 至 15 行：调用 `AbstractChannelHandlerContext#invokeFlush()()` 方法，执行 flush 事件到下一个节点。
  - 后续的逻辑，和 [《精尽 Netty 源码解析 —— ChannelPipeline（四）之 Outbound 事件的传播》](http://svip.iocoder.cn/Netty/Pipeline-4-outbound/) 分享的 **bind** 事件在 pipeline 中的传播是**基本一致**的。
  - 随着 flush **事件**不断的向下一个节点传播，最终会到达 HeadContext 节点。详细解析，见 [「5. HeadContext」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

- 第 16 行：

  不在

   

  EventLoop 的线程中。

  - 第 12 至 21 行：创建 flush 任务。该任务的内部的调用【第 18 行】的代码，和【第 9 行】的代码是**一致**的。
  - 第 23 行：调用 `#safeExecute(executor, task, promise, m)` 方法，提交到 EventLoop 的线程中，执行该任务。从而实现，**在** EventLoop 的线程中，执行 flush 事件到下一个节点。

## 5. HeadContext

在 pipeline 中，flush 事件最终会到达 HeadContext 节点。而 HeadContext 的 `#flush()` 方法，会处理该事件，代码如下：

```
@Override
public void flush(ChannelHandlerContext ctx) throws Exception {
    unsafe.flush();
}
```

- 在方法内部，会调用 `AbstractUnsafe#flush()` 方法，刷新**内存队列**，将其中的数据写入到对端。详细解析，见 [「6. AbstractUnsafe」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

## 6. AbstractUnsafe

`AbstractUnsafe#flush()` 方法，刷新**内存队列**，将其中的数据写入到对端。代码如下：

```
 1: @Override
 2: public final void flush() {
 3:     assertEventLoop();
 4: 
 5:     // 内存队列为 null ，一般是 Channel 已经关闭，所以直接返回。
 6:     ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
 7:     if (outboundBuffer == null) {
 8:         return;
 9:     }
10: 
11:     // 标记内存队列开始 flush
12:     outboundBuffer.addFlush();
13:     // 执行 flush
14:     flush0();
15: }
```

- 第 5 至 9 行：内存队列为 `null` ，一般是 Channel **已经关闭**，所以直接返回。

- 第 12 行：调用 `ChannelOutboundBuffer#addFlush()` 方法，标记内存队列开始 **flush** 。详细解析，见 [「8.4 addFlush」](http://svip.iocoder.cn/Netty/Channel-5-flush/#)。

- 第 14 行：调用 `#flush0()` 方法，执行 flush 操作。代码如下：

  ```
  /**
   * 是否正在 flush 中，即正在调用 {@link #flush0()} 中
   */
  private boolean inFlush0;
  
    1: @SuppressWarnings("deprecation")
    2: protected void flush0() {
    3:     // 正在 flush 中，所以直接返回。
    4:     if (inFlush0) {
    5:         // Avoid re-entrance
    6:         return;
    7:     }
    8: 
    9:     // 内存队列为 null ，一般是 Channel 已经关闭，所以直接返回。
   10:     // 内存队列为空，无需 flush ，所以直接返回
   11:     final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
   12:     if (outboundBuffer == null || outboundBuffer.isEmpty()) {
   13:         return;
   14:     }
   15: 
   16:     // 标记正在 flush 中。
   17:     inFlush0 = true;
   18: 
   19:     // 若未激活，通知 flush 失败
   20:     // Mark all pending write requests as failure if the channel is inactive.
   21:     if (!isActive()) {
   22:         try {
   23:             if (isOpen()) {
   24:                 outboundBuffer.failFlushed(FLUSH0_NOT_YET_CONNECTED_EXCEPTION, true);
   25:             } else {
   26:                 // Do not trigger channelWritabilityChanged because the channel is closed already.
   27:                 outboundBuffer.failFlushed(FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
   28:             }
   29:         } finally {
   30:             // 标记不在 flush 中。
   31:             inFlush0 = false;
   32:         }
   33:         return;
   34:     }
   35: 
   36:     // 执行真正的写入到对端
   37:     try {
   38:         doWrite(outboundBuffer);
   39:     } catch (Throwable t) {
   40:         // TODO 芋艿 细节
   41:         if (t instanceof IOException && config().isAutoClose()) {
   42:             /**
   43:              * Just call {@link #close(ChannelPromise, Throwable, boolean)} here which will take care of
   44:              * failing all flushed messages and also ensure the actual close of the underlying transport
   45:              * will happen before the promises are notified.
   46:              *
   47:              * This is needed as otherwise {@link #isActive()} , {@link #isOpen()} and {@link #isWritable()}
   48:              * may still return {@code true} even if the channel should be closed as result of the exception.
   49:              */
   50:             close(voidPromise(), t, FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
   51:         } else {
   52:             try {
   53:                 shutdownOutput(voidPromise(), t);
   54:             } catch (Throwable t2) {
   55:                 close(voidPromise(), t2, FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
   56:             }
   57:         }
   58:     } finally {
   59:         // 标记不在 flush 中。
   60:         inFlush0 = false;
   61:     }
   62: }
  ```

  - `inFlush0` 字段，是否正在 flush 中，即正在调用 `#flush0()` 中。

  - 第 3 至 7 行：正在 flush 中，所以直接返回。

  - 第 9 至 14 行：

    - `outboundBuffer == null` ，内存队列为 `null` ，一般是 Channel 已经**关闭**，所以直接返回。
    - `outboundBuffer.isEmpty()` ，内存队列为空，无需 flush ，所以直接返回。

  - 第 17 行：设置 `inFlush0` 为 `true` ，表示正在 flush 中。

  - 第 19 至 34 行：调用

     

    ```
    #isActive()
    ```

     

    方法，发现 Channel

     

    未激活

    ，在根据 Channel

     

    是否打开

    ，调用

     

    ```
    ChannelOutboundBuffer#failFlushed(Throwable cause, boolean notify)
    ```

     

    方法，通知 flush 失败

    异常

    。详细解析，见

     

    「8.6 failFlushed」

     

    。

    - 第 29 至 33 行：最终，设置 `inFlush0` 为 `false` ，表示结束 flush 操作，最后 `return` 返回。

  - 第 38 行：调用

     

    ```
    AbstractChannel#doWrite(outboundBuffer)
    ```

     

    方法，

    执行真正的写入到对端

    。详细解析，见

     

    「7. NioSocketChannel」

     

    。

    - 第 39 至 57 行：TODO 芋艿 细节
    - 第 58 至 61 行：同【第 29 至 33】的代码和目的。

- 实际上，AbstractNioUnsafe **重写**了 `#flush0()` 方法，代码如下：

  ```
  @Override
  protected final void flush0() {
      // Flush immediately only when there's no pending flush.
      // If there's a pending flush operation, event loop will call forceFlush() later,
      // and thus there's no need to call it now.
      if (!isFlushPending()) {
          super.flush0();
      }
  }
  ```

  - 在执行父类 AbstractUnsafe 的 `#flush0()` 方法时，先调用 `AbstractNioUnsafe#isFlushPending()` 判断，是否已经处于 flush **准备**中。代码如下：

    ```
    private boolean isFlushPending() {
        SelectionKey selectionKey = selectionKey();
        return selectionKey.isValid() // 合法
                && (selectionKey.interestOps() & SelectionKey.OP_WRITE) != 0; // 对 SelectionKey.OP_WRITE 事件不感兴趣。
    }
    ```

    - 是不是有点懵 x ？在文初，我们提到：“所以在 Netty 的实现中，默认 Channel 是**可写**的，当写入失败的时候，再去注册 `SelectionKey.OP_WRITE` 事件。这意味着什么呢？在 `#flush()` 方法中，如果写入数据到 Channel 失败，会通过注册 `SelectionKey.OP_WRITE` 事件，然后在轮询到 Channel **可写** 时，再“回调” `#forceFlush()` 方法”。
    - 这就是这段代码的目的，如果处于对 `SelectionKey.OP_WRITE` 事件感兴趣，说明 Channel 此时是**不可写**的，那么调用父类 AbstractUnsafe 的 `#flush0()` 方法，**也没有意义**，所以就不调用。
    - 😈 逻辑上，略微有点复杂，胖友好好理解下。

## 7. NioSocketChannel

`AbstractChannel#doWrite(ChannelOutboundBuffer in)` **抽象**方法，**执行真正的写入到对端**。定义在 AbstractChannel **抽象**类中，代码如下：

```
/**
 * Flush the content of the given buffer to the remote peer.
 */
protected abstract void doWrite(ChannelOutboundBuffer in) throws Exception;
```

------

NioSocketChannel 对该**抽象**方法，实现代码如下：

```
 1: @Override
 2: protected void doWrite(ChannelOutboundBuffer in) throws Exception {
 3:     SocketChannel ch = javaChannel();
 4:     // 获得自旋写入次数
 5:     int writeSpinCount = config().getWriteSpinCount();
 6:     do {
 7:         // 内存队列为空，结束循环，直接返回
 8:         if (in.isEmpty()) {
 9:             // 取消对 SelectionKey.OP_WRITE 的感兴趣
10:             // All written so clear OP_WRITE
11:             clearOpWrite();
12:             // Directly return here so incompleteWrite(...) is not called.
13:             return;
14:         }
15: 
16:         // 获得每次写入的最大字节数
17:         // Ensure the pending writes are made of ByteBufs only.
18:         int maxBytesPerGatheringWrite = ((NioSocketChannelConfig) config).getMaxBytesPerGatheringWrite();
19:         // 从内存队列中，获得要写入的 ByteBuffer 数组
20:         ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
21:         // 写入的 ByteBuffer 数组的个数
22:         int nioBufferCnt = in.nioBufferCount();
23: 
24:         // 写入 ByteBuffer 数组，到对端
25:         // Always us nioBuffers() to workaround data-corruption.
26:         // See https://github.com/netty/netty/issues/2761
27:         switch (nioBufferCnt) {
28:             case 0:
29:                 // 芋艿 TODO 1014 扣 doWrite0 的细节
30:                 // We have something else beside ByteBuffers to write so fallback to normal writes.
31:                 writeSpinCount -= doWrite0(in);
32:                 break;
33:             case 1: {
34:                 // Only one ByteBuf so use non-gathering write
35:                 // Zero length buffers are not added to nioBuffers by ChannelOutboundBuffer, so there is no need
36:                 // to check if the total size of all the buffers is non-zero.
37:                 ByteBuffer buffer = nioBuffers[0];
38:                 int attemptedBytes = buffer.remaining();
39:                 // 执行 NIO write 调用，写入单个 ByteBuffer 对象到对端
40:                 final int localWrittenBytes = ch.write(buffer);
41:                 // 写入字节小于等于 0 ，说明 NIO Channel 不可写，所以注册 SelectionKey.OP_WRITE ，等待 NIO Channel 可写，并返回以结束循环
42:                 if (localWrittenBytes <= 0) {
43:                     incompleteWrite(true);
44:                     return;
45:                 }
46:                 // TODO 芋艿 调整每次写入的最大字节数
47:                 adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
48:                 // 从内存队列中，移除已经写入的数据( 消息 )
49:                 in.removeBytes(localWrittenBytes);
50:                 // 写入次数减一
51:                 --writeSpinCount;
52:                 break;
53:             }
54:             default: {
55:                 // Zero length buffers are not added to nioBuffers by ChannelOutboundBuffer, so there is no need
56:                 // to check if the total size of all the buffers is non-zero.
57:                 // We limit the max amount to int above so cast is safe
58:                 long attemptedBytes = in.nioBufferSize();
59:                 // 执行 NIO write 调用，写入多个 ByteBuffer 到对端
60:                 final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
61:                 // 写入字节小于等于 0 ，说明 NIO Channel 不可写，所以注册 SelectionKey.OP_WRITE ，等待 NIO Channel 可写，并返回以结束循环
62:                 if (localWrittenBytes <= 0) {
63:                     incompleteWrite(true);
64:                     return;
65:                 }
66:                 // TODO 芋艿 调整每次写入的最大字节数
67:                 // Casting to int is safe because we limit the total amount of data in the nioBuffers to int above.
68:                 adjustMaxBytesPerGatheringWrite((int) attemptedBytes, (int) localWrittenBytes, maxBytesPerGatheringWrite);
69:                 // 从内存队列中，移除已经写入的数据( 消息 )
70:                 in.removeBytes(localWrittenBytes);
71:                 // 写入次数减一
72:                 --writeSpinCount;
73:                 break;
74:             }
75:         }
76:     } while (writeSpinCount > 0); // 循环自旋写入
77: 
78:     // 内存队列中的数据未完全写入，说明 NIO Channel 不可写，所以注册 SelectionKey.OP_WRITE ，等待 NIO Channel 可写
79:     incompleteWrite(writeSpinCount < 0);
80: }
```

- 第 3 行：调用 `#javaChannel()` 方法，获得 Java NIO **原生** SocketChannel 。

- 第 5 行：调用 `ChannelConfig#getWriteSpinCount()` 方法，获得**自旋**写入次数 N 。在【第 6 至 76 行】的代码，我们可以看到，不断**自旋**写入 N 次，直到完成写入结束。关于该配置项，官方注释如下：

  ```
  /**
   * Returns the maximum loop count for a write operation until {@link WritableByteChannel#write(ByteBuffer)} returns a non-zero value.
   * It is similar to what a spin lock is used for in concurrency programming.
   * It improves memory utilization and write throughput depending on the platform that JVM runs on.  The default value is {@code 16}.
   */
  int getWriteSpinCount();
  ```

  - 默认值为 `DefaultChannelConfig.writeSpinCount = 16` ，可配置修改，一般不需要。

- 第 6 至 76 行：不断**自旋**写入 N 次，直到完成写入结束。

- 第 8 行：调用 `ChannelOutboundBuffer#isEmpty()` 方法，内存队列为空，结束循环，直接返回。

  - 第 10 行：因为在 Channel **不可写**的时候，会注册 `SelectionKey.OP_WRITE` ，等待 NIO Channel 可写。而后会”回调” `#forceFlush()` 方法，该方法内部也会调用 `#doWrite(ChannelOutboundBuffer in)` 方法。所以在完成内部队列的数据向对端写入时候，需要调用 `#clearOpWrite()` 方法，代码如下：

    ```
    protected final void clearOpWrite() {
        final SelectionKey key = selectionKey();
        // Check first if the key is still valid as it may be canceled as part of the deregistration
        // from the EventLoop
        // See https://github.com/netty/netty/issues/2104
        if (!key.isValid()) { // 合法
            return;
        }
        final int interestOps = key.interestOps();
        // 若注册了 SelectionKey.OP_WRITE ，则进行取消
        if ((interestOps & SelectionKey.OP_WRITE) != 0) {
            key.interestOps(interestOps & ~SelectionKey.OP_WRITE);
        }
    }
    ```

    - 😈 胖友看下代码注释。

- 第 18 行：调用 `NioSocketChannelConfig#getMaxBytesPerGatheringWrite()` 方法，获得每次写入的最大字节数。// TODO 芋艿 调整每次写入的最大字节数

- 第 20 行：调用

   

  ```
  ChannelOutboundBuffer#nioBuffers(int maxCount, long maxBytes)
  ```

   

  方法，从内存队列中，获得要写入的 ByteBuffer 数组。

  注意

  ，如果内存队列中数据量很大，可能获得的仅仅是一部分数据。详细解析，见

   

  「8.5 nioBuffers」

   

  。

  - 第 22 行：获得写入的 ByteBuffer 数组的个数。为什么不直接调用数组的 `#length()` 方法呢？因为返回的 ByteBuffer 数组是**预先生成的数组缓存**，存在不断重用的情况，所以不能直接使用 `#length()` 方法，而是要调用 `ChannelOutboundBuffer#nioBufferCount()` 方法，获得写入的 ByteBuffer 数组的个数。详细解析，见 [「8.5 nioBuffers」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。
  - 后续根据 `nioBufferCnt` 的数值，分成**三种**情况。

- **(づ￣3￣)づ╭❤～ 第一种**，`nioBufferCnt = 0` 。

- 芋艿 TODO 1014 扣 doWrite0 的细节，应该是内部的数据为 FileRegion ，可以暂时无视，不影响本文理解。

- **(づ￣3￣)づ╭❤～ 第二种**，`nioBufferCnt = 1` 。

- 第 40 行：调用 Java **原生** `SocketChannel#write(ByteBuffer buffer)` 方法，执行 NIO write 调用，写入**单个** ByteBuffer 对象到对端。

- 第 42 行：写入字节小于等于 0 ，说明 NIO Channel **不可写**，所以注册 `SelectionKey.OP_WRITE` ，等待 NIO Channel **可写**，并返回以结束循环。

  - 第 43 行：调用 `AbstractNioByteChannel#incompleteWrite(true)` 方法，代码如下：

    ```
    protected final void incompleteWrite(boolean setOpWrite) {
        // Did not write completely.
        // true ，注册对 SelectionKey.OP_WRITE 事件感兴趣
        if (setOpWrite) {
            setOpWrite();
        // false ，取消对 SelectionKey.OP_WRITE 事件感兴趣
        } else {
            // It is possible that we have set the write OP, woken up by NIO because the socket is writable, and then
            // use our write quantum. In this case we no longer want to set the write OP because the socket is still
            // writable (as far as we know). We will find out next time we attempt to write if the socket is writable
            // and set the write OP if necessary.
            clearOpWrite();
    
            // Schedule flush again later so other tasks can be picked up in the meantime
            // 立即发起下一次 flush 任务
            eventLoop().execute(flushTask); // <1>
        }
    }
    ```

    - `setOpWrite` 为 `true` ，调用 `#setOpWrite()` 方法，注册对 `SelectionKey.OP_WRITE` 事件感兴趣。代码如下：

      ```
      protected final void setOpWrite() {
          final SelectionKey key = selectionKey();
          // Check first if the key is still valid as it may be canceled as part of the deregistration
          // from the EventLoop
          // See https://github.com/netty/netty/issues/2104
          if (!key.isValid()) { // 合法
              return;
          }
          final int interestOps = key.interestOps();
          // 注册 SelectionKey.OP_WRITE 事件的感兴趣
          if ((interestOps & SelectionKey.OP_WRITE) == 0) {
              key.interestOps(interestOps | SelectionKey.OP_WRITE);
          }
      }
      ```

      - 【第 43 行】的代码，就是这种情况。

    - `setOpWrite` 为 `false` ，调用 `#clearOpWrite()` 方法，取消对 SelectionKey.OP_WRITE 事件感兴趣。而后，在 `<1>` 处，立即发起下一次 flush 任务。

- 第 47 行：TODO 芋艿 调整每次写入的最大字节数

- 第 49 行：调用 `ChannelOutboundBuffer#removeBytes(long writtenBytes)` 方法啊，从内存队列中，移除已经写入的数据( 消息 )。详细解析，见 [「8.7 removeBytes」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

- 第 51 行：写入次数减一。

- **(づ￣3￣)づ╭❤～ 第三种**，`nioBufferCnt > 1` 。和【第二种】基本相同，差别是在于【第 60 行】的代码，调用 Java **原生**`SocketChannel#write(ByteBuffer[] srcs, int offset, int length)` 方法，执行 NIO write 调用，写入**多个**ByteBuffer 对象到对端。😈 批量一次性写入，提升性能。

- =========== 结束 ===========

- 第 79 行：通过

   

  ```
  writeSpinCount < 0
  ```

   

  来判断，内存队列中的数据

  是否

  未完全写入。从目前逻辑看下来，笔者认为只会返回

   

  ```
  true
  ```

   

  ，即内存队列中的数据未完全写入，说明 NIO Channel 不可写，所以注册

   

  ```
  SelectionKey.OP_WRITE
  ```

   

  ，等待 NIO Channel 可写。因此，调用

   

  ```
  #incompleteWrite(true)
  ```

   

  方法。

  - 举个例子，最后一次写入，Channel 的缓冲区还剩下 10 字节可写，内存队列中剩余 90 字节，那么可以成功写入 10 字节，剩余 80 字节。😈 也就说，此时 Channel 不可写落。

### 7.1 乱入

> 老艿艿：临时插入下 AbstractNioByteChannel 和 AbstractNioMessageChannel 实现类对 `#doWrite(ChannelOutboundBuffer in)` 方法的实现。不感兴趣的胖友，可以直接跳过。

**AbstractNioByteChannel**

虽然，AbstractNioByteChannel 实现了 `#doWrite(ChannelOutboundBuffer in)` 方法，但是子类 NioSocketChannel 又覆盖实现了该方法，所以可以忽略 AbstractNioByteChannel 的实现方法了。

那么为什么 AbstractNioByteChannel 会实现了 `#doWrite(ChannelOutboundBuffer in)` 方法呢？因为 NioUdtByteConnectorChannel 和 NioUdtByteRendezvousChannel 会使用到该方法。但是，这两个类已经被**标记废弃**，因为：

```
transport udt is deprecated and so the user knows it will be removed in the future.
```

- 来自 Netty 官方提交的注释说明。

**AbstractNioMessageChannel**

虽然，AbstractNioMessageChannel 实现了 `#doWrite(ChannelOutboundBuffer in)` 方法，但是对于 NioServerSocketChannel 来说，暂时没有意义，因为：

```
@Override
protected boolean doWriteMessage(Object msg, ChannelOutboundBuffer in) throws Exception {
    throw new UnsupportedOperationException();
}

@Override
protected final Object filterOutboundMessage(Object msg) throws Exception {
    throw new UnsupportedOperationException();
}
```

- 两个方法，都是直接抛出 UnsupportedOperationException 异常。

那么为什么 AbstractNioMessageChannel 会实现了 `#doWrite(ChannelOutboundBuffer in)` 方法呢？因为 NioDatagramChannel 和 NioSctpChannel **等等**会使用到该方法。感兴趣的胖友，可以自己研究下。

## 8. ChannelOutboundBuffer

`io.netty.channel.ChannelOutboundBuffer` ，**内存队列**。

- 在 write 操作时，将数据写到 ChannelOutboundBuffer 中。
- 在 flush 操作时，将 ChannelOutboundBuffer 的数据写入到对端。

### 8.1 Entry

在 write 操作时，将数据写到 ChannelOutboundBuffer 中，都会产生一个 Entry 对象。代码如下：

```
/**
 * Recycler 对象，用于重用 Entry 对象
 */
private static final Recycler<Entry> RECYCLER = new Recycler<Entry>() {
    @Override
    protected Entry newObject(Handle<Entry> handle) {
        return new Entry(handle);
    }
};

/**
 * Recycler 处理器
 */
private final Handle<Entry> handle;
/**
 * 下一条 Entry
 */
Entry next;
/**
 * 消息（数据）
 */
Object msg;
/**
 * {@link #msg} 转化的 NIO ByteBuffer 数组
 */
ByteBuffer[] bufs;
/**
 * {@link #msg} 转化的 NIO ByteBuffer 对象
 */
ByteBuffer buf;
/**
 * Promise 对象
 */
ChannelPromise promise;
/**
 * 已写入的字节数
 */
long progress;
/**
 * 长度，可读字节数数。
 */
long total;
/**
 * 每个 Entry 预计占用的内存大小，计算方式为消息( {@link #msg} )的字节数 + Entry 对象自身占用内存的大小。
 */
int pendingSize;
/**
 * {@link #msg} 转化的 NIO ByteBuffer 的数量。
 *
 * 当 = 1 时，使用 {@link #buf}
 * 当 > 1 时，使用 {@link #bufs}
 */
int count = -1;
/**
 * 是否取消写入对端
 */
boolean cancelled;

private Entry(Handle<Entry> handle) {
    this.handle = handle;
}
```

- ```
  RECYCLER
  ```

   

  静态

  属性，用于

  重用

   

  Entry 对象。

  - `handle` 属性，Recycler 处理器，用于**回收** Entry 对象。

- `next` 属性，指向**下一条** Entry 。通过它，形成 ChannelOutboundBuffer 内部的链式存储**每条写入数据**的数据结构。

- `msg` 属性，写入的消息( 数据 )。

  - `promise` 属性，Promise 对象。当数据写入成功后，可以通过它回调通知结果。

  - `total` 属性，长度，可读字节数。通过 `#total(Object msg)` 方法来计算。代码如下：

    ```
    private static long total(Object msg) {
        if (msg instanceof ByteBuf) {
            return ((ByteBuf) msg).readableBytes();
        }
        if (msg instanceof FileRegion) {
            return ((FileRegion) msg).count();
        }
        if (msg instanceof ByteBufHolder) {
            return ((ByteBufHolder) msg).content().readableBytes();
        }
        return -1;
    }
    ```

    - 从这个方法，我们看到，`msg` 的类型，有 ByteBuf、FileRegion、ByteBufHolder 。

  - `process` 属性，已写入的字节数。详细解析，见 [「8.7.1 process」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

- ```
  count
  ```

   

  属性，

  ```
  msg
  ```

   

  属性转化的 NIO ByteBuffer 的数量。

  - `bufs` 属性，当 `count > 0` 时使用，表示 `msg` 属性转化的 NIO ByteBuffer 数组。
  - `buf` 属性，当 `count = 0` 时使用，表示 `msg` 属性转化的 NIO ByteBuffer 对象。

- `cancelled` 属性，是否取消写入对端。

- `pendingSize` 属性，每个 Entry 预计占用的内存大小，计算方式为消息( `msg` )的字节数 + Entry 对象自身占用内存的大小。

#### 8.1.1 newInstance

`#newInstance(Object msg, int size, long total, ChannelPromise promise)` **静态**方法，创建 Entry 对象。代码如下：

```
static Entry newInstance(Object msg, int size, long total, ChannelPromise promise) {
    // 通过 Recycler 重用对象
    Entry entry = RECYCLER.get();
    // 初始化属性
    entry.msg = msg;
    entry.pendingSize = size + CHANNEL_OUTBOUND_BUFFER_ENTRY_OVERHEAD;
    entry.total = total;
    entry.promise = promise;
    return entry;
}
```

- 通过 Recycler 来**重用** Entry 对象。

#### 8.1.2 recycle

`#recycle()` 方法，**回收** Entry 对象，以为下次**重用**该对象。代码如下：

```
void recycle() {
    // 重置属性
    next = null;
    bufs = null;
    buf = null;
    msg = null;
    promise = null;
    progress = 0;
    total = 0;
    pendingSize = 0;
    count = -1;
    cancelled = false;
    // 回收 Entry 对象
    handle.recycle(this);
}
```

#### 8.1.3 recycleAndGetNext

`#recycleAndGetNext()` 方法，获得下一个 Entry 对象，并**回收**当前 Entry 对象。代码如下：

```
Entry recycleAndGetNext() {
    // 获得下一个 Entry 对象
    Entry next = this.next;
    // 回收当前 Entry 对象
    recycle();
    // 返回下一个 Entry 对象
    return next;
}
```

#### 8.1.4 cancel

`#cancel()` 方法，标记 Entry 对象，取消写入到对端。在 ChannelOutboundBuffer 里，Entry 数组是通过**链式**的方式进行组织，而当某个 Entry 对象( **节点** )如果需要取消写入到对端，是通过设置 `canceled = true` 来**标记删除**。代码如下：

```
int cancel() {
    if (!cancelled) {
        // 标记取消
        cancelled = true;
        int pSize = pendingSize;

        // 释放消息( 数据 )相关的资源
        // release message and replace with an empty buffer
        ReferenceCountUtil.safeRelease(msg);
        // 设置为空 ByteBuf
        msg = Unpooled.EMPTY_BUFFER;

        // 置空属性
        pendingSize = 0;
        total = 0;
        progress = 0;
        bufs = null;
        buf = null;

        // 返回 pSize
        return pSize;
    }
    return 0;
}
```

### 8.2 构造方法

```
/**
 * Entry 对象自身占用内存的大小
 */
// Assuming a 64-bit JVM:
//  - 16 bytes object header
//  - 8 reference fields
//  - 2 long fields
//  - 2 int fields
//  - 1 boolean field
//  - padding
static final int CHANNEL_OUTBOUND_BUFFER_ENTRY_OVERHEAD = SystemPropertyUtil.getInt("io.netty.transport.outboundBufferEntrySizeOverhead", 96);

private static final InternalLogger logger = InternalLoggerFactory.getInstance(ChannelOutboundBuffer.class);

/**
 * 线程对应的 ByteBuffer 数组缓存
 * 
 * 每次调用 {@link #nioBuffers(int, long)} 会重新生成
 */
private static final FastThreadLocal<ByteBuffer[]> NIO_BUFFERS = new FastThreadLocal<ByteBuffer[]>() {

    @Override
    protected ByteBuffer[] initialValue() throws Exception {
        return new ByteBuffer[1024];
    }

};

/**
 * Channel 对象
 */
private final Channel channel;

// Entry(flushedEntry) --> ... Entry(unflushedEntry) --> ... Entry(tailEntry)
//
/**
 * 第一个( 开始 ) flush Entry
 */
// The Entry that is the first in the linked-list structure that was flushed
private Entry flushedEntry;
/**
 * 第一个未 flush Entry
 */
// The Entry which is the first unflushed in the linked-list structure
private Entry unflushedEntry;
/**
 * 尾 Entry
 */
// The Entry which represents the tail of the buffer
private Entry tailEntry;
/**
 * 已 flush 但未写入对端的 Entry 数量
 *
 * {@link #addFlush()}
 *
 * The number of flushed entries that are not written yet
 */
private int flushed;

/**
 * {@link #NIO_BUFFERS} 数组大小
 */
private int nioBufferCount;
/**
 * {@link #NIO_BUFFERS} 字节数
 */
private long nioBufferSize;
/**
 * 正在通知 flush 失败中
 */
private boolean inFail;

/**
 * {@link #totalPendingSize} 的原子更新器
 */
private static final AtomicLongFieldUpdater<ChannelOutboundBuffer> TOTAL_PENDING_SIZE_UPDATER = AtomicLongFieldUpdater.newUpdater(ChannelOutboundBuffer.class, "totalPendingSize");
/**
 * 总共等待 flush 到对端的内存大小，通过 {@link Entry#pendingSize} 来合计。
 */
@SuppressWarnings("UnusedDeclaration")
private volatile long totalPendingSize;

/**
 * {@link #unwritable} 的原子更新器
 */
private static final AtomicIntegerFieldUpdater<ChannelOutboundBuffer> UNWRITABLE_UPDATER = AtomicIntegerFieldUpdater.newUpdater(ChannelOutboundBuffer.class, "unwritable");
/**
 * 是否不可写
 */
@SuppressWarnings("UnusedDeclaration")
private volatile int unwritable;

/**
 * 触发 Channel 可写的改变的任务
 */
private volatile Runnable fireChannelWritabilityChangedTask;

ChannelOutboundBuffer(AbstractChannel channel) {
    this.channel = channel;
}
```

- `channel` 属性，所属的 Channel 对象。

- 链式结构

  - `flushedEntry` 属性，第一个( 开始 ) flush Entry 。
  - `unflushedEntry` 属性，第一个**未** flush Entry 。
  - `tailEntry` 属性，尾 Entry 。
  - `flushed` 属性， 已 flush 但未写入对端的 Entry 数量。
  - 指向关系是 `Entry(flushedEntry) --> ... Entry(unflushedEntry) --> ... Entry(tailEntry)` 。这样看，可能有点抽象，下文源码解析详细理解。

- ```
  NIO_BUFFERS
  ```

   

  静态

  属性，线程对应的 NIO ByteBuffer 数组缓存。在

   

  ```
  AbstractChannel#doWrite(ChannelOutboundBuffer)
  ```

   

  方法中，会调用

   

  ```
  ChannelOutbound#nioBuffers(int maxCount, long maxBytes)
  ```

   

  方法，初始化数组缓存。 详细解析，见

   

  「8.6 nioBuffers」

   

  中。

  - `nioBufferCount` 属性：NIO ByteBuffer 数组的**数组**大小。
  - `nioBufferSize` 属性：NIO ByteBuffer 数组的字**节**大小。

- `inFail` 属性，正在通知 flush 失败中。详细解析，见 [「8.8 failFlushed」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 中。

- ChannelOutboundBuffer 写入控制相关。😈 详细解析，见

   

  「10. ChannelOutboundBuffer」

   

  。

  - ```
    unwritable
    ```

     

    属性，是否不可写。

    - `UNWRITABLE_UPDATER` 静态属性，`unwritable` 属性的原子更新器。

  - ```
    totalPendingSize
    ```

     

    属性，所有 Entry 预计占用的内存大小，通过

     

    ```
    Entry.pendingSize
    ```

     

    来合计。

    - `TOTAL_PENDING_SIZE_UPDATER` 静态属性，`totalPendingSize` 属性的原子更新器。

  - `fireChannelWritabilityChangedTask` 属性，触发 Channel 可写的改变的**任务**。

  - ```
    CHANNEL_OUTBOUND_BUFFER_ENTRY_OVERHEAD
    ```

     

    静态

    属性，每个 Entry 对象自身占用内存的大小。为什么占用的 96 字节呢？

    - `- 16 bytes object header` ，对象头，16 字节。
    - `- 8 reference fields` ，实际是 6 个**对象引用**字段，6 * 8 = 48 字节。
    - `- 2 long fields` ，2 个 `long` 字段，2 * 8 = 16 字节。
    - `- 2 int fields` ，1 个 `int` 字段，2 * 4 = 8 字节。
    - `- 1 boolean field` ，1 个 `boolean` 字段，1 字节。
    - `padding` ，补齐 8 字节的整数倍，因此 7 字节。
    - 因此，合计 96 字节( 64 位的 JVM 虚拟机，并且不考虑压缩 )。
    - 如果不理解的胖友，可以看看 [《JVM中 对象的内存布局 以及 实例分析》](https://www.jianshu.com/p/12a3c97dc2b7) 。

### 8.3 addMessage

`#addMessage(Object msg, int size, ChannelPromise promise)` 方法，写入消息( 数据 )到内存队列。**注意**，`promise` 只有在真正完成写入到对端操作，才会进行通知。代码如下：

```
 1: /**
 2:  * Add given message to this {@link ChannelOutboundBuffer}. The given {@link ChannelPromise} will be notified once
 3:  * the message was written.
 4:  */
 5: public void addMessage(Object msg, int size, ChannelPromise promise) {
 6:     // 创建新 Entry 对象
 7:     Entry entry = Entry.newInstance(msg, size, total(msg), promise);
 8:     // 若 tailEntry 为空，将 flushedEntry 也设置为空。防御型编程，实际不会出现
 9:     if (tailEntry == null) {
10:         flushedEntry = null;
11:     // 若 tailEntry 非空，将原 tailEntry 指向新 Entry
12:     } else {
13:         Entry tail = tailEntry;
14:         tail.next = entry;
15:     }
16:     // 更新 tailEntry 为新 Entry
17:     tailEntry = entry;
18:     // 若 unflushedEntry 为空，更新为新 Entry
19:     if (unflushedEntry == null) {
20:         unflushedEntry = entry;
21:     }
22: 
23:     // 增加 totalPendingSize 计数
24:     // increment pending bytes after adding message to the unflushed arrays.
25:     // See https://github.com/netty/netty/issues/1619
26:     incrementPendingOutboundBytes(entry.pendingSize, false);
27: }
```

- 第 7 行：调用 `#newInstance(Object msg, int size, long total, ChannelPromise promise)` **静态**方法，创建 Entry 对象。

- 第 11 至 17 行：修改

  尾

  节点

   

  ```
  tailEntry
  ```

   

  为新的 Entry 节点。

  - 第 8 至 10 行：若 `tailEntry` 为空，将 `flushedEntry` 也设置为空。防御型编程，实际不会出现，胖友可以忽略。😈 当然，原因在 `#removeEntry(Entry e)` 方法。
  - 第 11 至 15 行：若 `tailEntry` 非空，将原 `tailEntry.next` 指向**新** Entry 。
  - 第 17 行：更新原 `tailEntry` 为新 Entry 。

- 第 18 至 21 行：若 `unflushedEntry` 为空，则更新为新 Entry ，此时相当于**首**节点。

- 第 23 至 26 行：`#incrementPendingOutboundBytes(long size, ...)` 方法，增加 `totalPendingSize` 计数。详细解析，见 [「10.1 incrementPendingOutboundBytes」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

------

可能有点抽象，我们来看看基友【闪电侠】对这块的解析：

> FROM 闪电侠 [《netty 源码分析之 writeAndFlush 全解析》](https://www.jianshu.com/p/feaeaab2ce56)
>
> 初次调用 `addMessage` 之后，各个指针的情况为
>
> ![img](http://static.iocoder.cn/1ff7a5d2b08b9e6160dd92e74e68145f)
>
> `fushedEntry`指向空，`unFushedEntry`和 `tailEntry` 都指向新加入的节点
>
> 第二次调用 `addMessage` 之后，各个指针的情况为
>
> ![img](http://static.iocoder.cn/1f939423f079ff491b90c8300e7ef3ea)
>
> 第 n 次调用 `addMessage`之后，各个指针的情况为
>
> ![img](http://static.iocoder.cn/c0077b0dc86ecf1b791a99eeb9664fc3)
>
> 可以看到，调用 n 次 `addMessage` ，`flushedEntry` 指针一直指向 NULL ，表示现在还未有节点需要写出到 Socket 缓冲区，而`unFushedEntry` 之后有 n 个节点，表示当前还有n个节点尚未写出到 Socket 缓冲区中去

### 8.4 addFlush

`#addFlush()` 方法，标记内存队列每个 Entry 对象，开始 **flush** 。代码如下：

> 老艿艿：总觉得这个方法名取的有点奇怪，胖友可以直接看英文注释。😈 我“翻译”不好，哈哈哈。

```
 1: public void addFlush() {
 2:     // There is no need to process all entries if there was already a flush before and no new messages
 3:     // where added in the meantime.
 4:     //
 5:     // See https://github.com/netty/netty/issues/2577
 6:     Entry entry = unflushedEntry;
 7:     if (entry != null) {
 8:         // 若 flushedEntry 为空，赋值为 unflushedEntry ，用于记录第一个( 开始 ) flush 的 Entry 。
 9:         if (flushedEntry == null) {
10:             // there is no flushedEntry yet, so start with the entry
11:             flushedEntry = entry;
12:         }
13:         // 计算 flush 的数量，并设置每个 Entry 对应的 Promise 不可取消
14:         do {
15:             // 增加 flushed
16:             flushed ++;
17:             // 设置 Promise 不可取消
18:             if (!entry.promise.setUncancellable()) { // 设置失败
19:                 // 减少 totalPending 计数
20:                 // Was cancelled so make sure we free up memory and notify about the freed bytes
21:                 int pending = entry.cancel();
22:                 decrementPendingOutboundBytes(pending, false, true);
23:             }
24:             // 获得下一个 Entry
25:             entry = entry.next;
26:         } while (entry != null);
27: 
28:         // 设置 unflushedEntry 为空，表示所有都 flush
29:         // All flushed so reset unflushedEntry
30:         unflushedEntry = null;
31:     }
32: }
```

- 第 6 至 7 行：若 `unflushedEntry` 为空，说明每个 Entry 对象已经“标记” flush 。**注意**，“标记”的方式，不是通过 Entry 对象有一个 `flushed` 字段，而是 `flushedEntry` 属性，指向第一个( 开始 ) flush 的 Entry ，而 `unflushedEntry` 置空。

- 第 8 至 12 行：若 `flushedEntry` 为空，赋值为 `unflushedEntry` ，用于记录第一个( 开始 ) flush 的 Entry 。

- 第 13 至 26 行：计算需要 flush 的 Entry 数量，并设置每个 Entry 对应的 Promise

   

  不可取消

  。

  - 第 18 至 23 行：`#decrementPendingOutboundBytes(long size, ...)` 方法，减少 `totalPendingSize` 计数。

- 第 30 行：设置 `unflushedEntry` 为空。

------

可能有点抽象，我们来看看基友【闪电侠】对这块的解析：

> FROM 闪电侠 [《netty 源码分析之 writeAndFlush 全解析》](https://www.jianshu.com/p/feaeaab2ce56)
>
> 可以结合前面的图来看，首先拿到 `unflushedEntry` 指针，然后将 `flushedEntry` 指向 `unflushedEntry` 所指向的节点，调用完毕之后，三个指针的情况如下所示
>
> ![img](http://static.iocoder.cn/ecb3df153a3df70464b524838b559232)

------

> 老艿艿：再次切回到老艿艿的频道，呼呼。

当一次需要从内存队列写到对端的数据量非常大，那么可能写着写着 Channel 的缓存区不够，导致 Channel 此时不可写。但是，这一轮 `#addFlush(...)` 标记的 Entry 对象并没有都写到对端。例如，准备写到对端的 Entry 的数量是 `flush = 10` 个，结果只写了 6 个，那么就剩下 `flush = 4` 。

但是的但是，`#addMessage(...)` 可能又不断写入新的消息( 数据 )到 ChannelOutboundBuffer 中。那会出现什么情况呢？会“分”成两段：

- `<1>` 段：自节点 `flushedEntry` 开始的 `flush` 个 Entry 节点，需要写入到对端。
- `<2>` 段：自节点 `unFlushedEntry` 开始的 Entry 节点，需要调用 `#addFlush()` 方法，添加到 `<1>` 段中。

这就很好的解释两个事情：

1. 为什么 `#addFlush()` 方法，命名是以 `"add"` 开头。
2. ChannelOutboundBuffer 的链式结构，为什么不是 `head` 和 `tail` **两个**节点，而是 `flushedEntry`、`unFlushedEntry`、`flushedEntry` **三个**节点。在此处，请允许老艿艿爆个粗口：真他 x 的巧妙啊。

#### 8.4.1 size

`#size()` 方法，获得 `flushed` 属性。代码如下：

```
/**
 * Returns the number of flushed messages in this {@link ChannelOutboundBuffer}.
 */
public int size() {
    return flushed;
}
```

#### 8.4.2 isEmpty

`#isEmpty()` 方法，是否为空。代码如下：

```
/**
 * Returns {@code true} if there are flushed messages in this {@link ChannelOutboundBuffer} or {@code false}
 * otherwise.
 */
public boolean isEmpty() {
    return flushed == 0;
}
```

### 8.5 current

`#current()` 方法，获得**当前**要写入对端的消息( 数据 )。代码如下：

```
/**
 * Return the current message to write or {@code null} if nothing was flushed before and so is ready to be written.
 */
public Object current() {
    Entry entry = flushedEntry;
    if (entry == null) {
        return null;
    }

    return entry.msg;
}
```

- 即，返回的是 `flushedEntry` 的消息( 数据 )。

### 8.6 nioBuffers

`#nioBuffers(int maxCount, long maxBytes)` 方法，获得当前要写入到对端的 NIO ByteBuffer 数组，并且获得的数组大小不得超过 `maxCount` ，字节数不得超过 `maxBytes` 。我们知道，在写入数据到 ChannelOutboundBuffer 时，一般使用的是 Netty ByteBuf 对象，但是写到 NIO SocketChannel 时，则必须使用 NIO ByteBuffer 对象，因此才有了这个方法。考虑到性能，这个方法里会使用到“**缓存**”，所以看起来会比较绕一丢丢。OK，开始看代码落：

```
/**
 * Returns an array of direct NIO buffers if the currently pending messages are made of {@link ByteBuf} only.
 * {@link #nioBufferCount()} and {@link #nioBufferSize()} will return the number of NIO buffers in the returned
 * array and the total number of readable bytes of the NIO buffers respectively.
 * <p>
 * Note that the returned array is reused and thus should not escape
 * {@link AbstractChannel#doWrite(ChannelOutboundBuffer)}.
 * Refer to {@link NioSocketChannel#doWrite(ChannelOutboundBuffer)} for an example.
 * </p>
 * @param maxCount The maximum amount of buffers that will be added to the return value.
 * @param maxBytes A hint toward the maximum number of bytes to include as part of the return value. Note that this
 *                 value maybe exceeded because we make a best effort to include at least 1 {@link ByteBuffer}
 *                 in the return value to ensure write progress is made.
 */
     
  1: public ByteBuffer[] nioBuffers(int maxCount, long maxBytes) {
  2:     assert maxCount > 0;
  3:     assert maxBytes > 0;
  4:     long nioBufferSize = 0;
  5:     int nioBufferCount = 0;
  6:     // 获得当前线程的 NIO ByteBuffer 数组缓存。
  7:     final InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
  8:     ByteBuffer[] nioBuffers = NIO_BUFFERS.get(threadLocalMap);
  9:     // 从 flushedEntry 节点，开始向下遍历
 10:     Entry entry = flushedEntry;
 11:     while (isFlushedEntry(entry) && entry.msg instanceof ByteBuf) {
 12:         // 若 Entry 节点已经取消，忽略。
 13:         if (!entry.cancelled) {
 14:             ByteBuf buf = (ByteBuf) entry.msg;
 15:             // 获得消息( 数据 )开始读取位置
 16:             final int readerIndex = buf.readerIndex();
 17:             // 获得消息( 数据 )可读取的字节数
 18:             final int readableBytes = buf.writerIndex() - readerIndex;
 19: 
 20:             // 若无可读取的数据，忽略。
 21:             if (readableBytes > 0) {
 22:                 // 前半段，可读取的字节数，不能超过 maxBytes
 23:                 // 后半段，如果第一条数据，就已经超过 maxBytes ，那么只能“强行”读取，否则会出现一直无法读取的情况。
 24:                 if (maxBytes - readableBytes < nioBufferSize && nioBufferCount != 0) {
 25:                     // If the nioBufferSize + readableBytes will overflow maxBytes, and there is at least one entry
 26:                     // we stop populate the ByteBuffer array. This is done for 2 reasons:
 27:                     // 1. bsd/osx don't allow to write more bytes then Integer.MAX_VALUE with one writev(...) call
 28:                     // and so will return 'EINVAL', which will raise an IOException. On Linux it may work depending
 29:                     // on the architecture and kernel but to be safe we also enforce the limit here.
 30:                     // 2. There is no sense in putting more data in the array than is likely to be accepted by the
 31:                     // OS.
 32:                     //
 33:                     // See also:
 34:                     // - https://www.freebsd.org/cgi/man.cgi?query=write&sektion=2
 35:                     // - http://linux.die.net/man/2/writev
 36:                     break;
 37:                 }
 38:                 // 增加 nioBufferSize
 39:                 nioBufferSize += readableBytes;
 40:                 // 初始 Entry 节点的 NIO ByteBuffer 数量
 41:                 int count = entry.count;
 42:                 if (count == -1) {
 43:                     //noinspection ConstantValueVariableUse
 44:                     entry.count = count = buf.nioBufferCount();
 45:                 }
 46:                 // 如果超过 NIO ByteBuffer 数组的大小，进行扩容。
 47:                 int neededSpace = min(maxCount, nioBufferCount + count);
 48:                 if (neededSpace > nioBuffers.length) {
 49:                     nioBuffers = expandNioBufferArray(nioBuffers, neededSpace, nioBufferCount);
 50:                     NIO_BUFFERS.set(threadLocalMap, nioBuffers);
 51:                 }
 52:                 // 初始化 Entry 节点的 buf / bufs 属性
 53:                 if (count == 1) {
 54:                     ByteBuffer nioBuf = entry.buf;
 55:                     if (nioBuf == null) {
 56:                         // cache ByteBuffer as it may need to create a new ByteBuffer instance if its a
 57:                         // derived buffer
 58:                         entry.buf = nioBuf = buf.internalNioBuffer(readerIndex, readableBytes);
 59:                     }
 60:                     nioBuffers[nioBufferCount++] = nioBuf;
 61:                 } else {
 62:                     ByteBuffer[] nioBufs = entry.bufs;
 63:                     if (nioBufs == null) {
 64:                         // cached ByteBuffers as they may be expensive to create in terms
 65:                         // of Object allocation
 66:                         entry.bufs = nioBufs = buf.nioBuffers();
 67:                     }
 68:                     for (int i = 0; i < nioBufs.length && nioBufferCount < maxCount; ++i) {
 69:                         ByteBuffer nioBuf = nioBufs[i];
 70:                         if (nioBuf == null) {
 71:                             break;
 72:                         } else if (!nioBuf.hasRemaining()) {
 73:                             continue;
 74:                         }
 75:                         nioBuffers[nioBufferCount++] = nioBuf;
 76:                     }
 77:                 }
 78: 
 79:                 // 到达 maxCount 上限，结束循环。老艿艿的想法，这里最好改成 nioBufferCount >= maxCount ，是有可能会超过的
 80:                 if (nioBufferCount == maxCount) {
 81:                     break;
 82:                 }
 83:             }
 84:         }
 85: 
 86:         // 下一个 Entry节点
 87:         entry = entry.next;
 88:     }
 89: 
 90:     // 设置 nioBufferCount 和 nioBufferSize 属性
 91:     this.nioBufferCount = nioBufferCount;
 92:     this.nioBufferSize = nioBufferSize;
 93: 
 94:     return nioBuffers;
 95: }
```

- 第 4 至 5 行：初始 `nioBufferSize`、`nioBufferCount` 计数。

- 第 6 至 8 行：获得当前线程的 NIO ByteBuffer 数组缓存。

  - 关于 InternalThreadLocalMap 和 FastThreadLocal ，胖友可以暂时忽略，后续的文章，详细解析。

- 第 10 至 11 行：从 `flushedEntry` 节点，开始向下遍历。

  - 调用 `#isFlushedEntry(Entry entry)` 方法，判断是否为已经“标记”为 flush 的 Entry 节点。代码如下：

    ```
    private boolean isFlushedEntry(Entry e) {
        return e != null && e != unflushedEntry;
    }
    ```

    - `e != unflushedEntry` ，就是我们在 [「8.4 addFlush」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 最后部分讲的，思考下。

  - `entry.msg instanceof ByteBuf` ，消息( 数据 )类型为 ByteBuf 。实际上，`msg` 的类型也可能是 FileRegion 。如果 ChannelOutboundBuffer 里的消息都是 FileRegion 类型，那就会导致这个方法返回为**空** NIO ByteBuffer 数组。

- 第 13 行：若 Entry 节点已经取消，忽略。

- 第 14 至 18 行：获得消息( 数据 )开始读取位置和可读取的字节数。

  - 第 21 行：若无可读取的数据，忽略。

- 第 22 至 37 行：

  - 前半段 `maxBytes - readableBytes < nioBufferSize` ，当前 ByteBuf 可读取的字节数，不能超过 `maxBytes` 。这个比较好理解。
  - 后半段 `nioBufferCount != 0` ，如果**第一条**数据，就已经超过 `maxBytes` ，那么只能“强行”读取，否则会出现一直无法读取的情况( 因为不能跳过这条 😈 )。

- 第 39 行：增加 `nioBufferSize` 。

- 第 40 至 45 行：调用

   

  ```
  ByteBuf#nioBufferCount()
  ```

   

  方法，初始 Entry 节点的

   

  ```
  count
  ```

   

  属性( NIO ByteBuffer 数量)。

  - 使用 `count == -1` 的原因是，`Entry.count` 未初始化时，为 `-1` 。

- 第 47 至 51 行：如果超过 NIO ByteBuffer 数组的大小，调用 `#expandNioBufferArray(ByteBuffer[] array, int neededSpace, int size)` 方法，进行扩容。详细解析，见 [「8.6.1 expandNioBufferArray」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

- 第 52 至 77 行：初始 Entry 节点的

   

  ```
  buf
  ```

   

  或

   

  ```
  bufs
  ```

   

  属性。

  - 当 `count = 1` 时，调用 `ByteBuf#internalNioBuffer(readerIndex, readableBytes)` 方法，获得 NIO ByteBuffer 对象。
  - 当 `count > 1` 时，调用 `ByteBuf#nioBuffers()` 方法，获得 NIO ByteBuffer 数组。
  - 通过 `nioBuffers[nioBufferCount++] = nioBuf` ，将 NIO ByteBuffer 赋值到结果数组 `nioBuffers` 中，并增加 `nioBufferCount` 。

- 第 79 至 82 行：到达 `maxCount` 上限，结束循环。老艿艿的想法，这里最好改成 `nioBufferCount >= maxCount` ，是有可能会超过的。

- 第 87 行：**下一个 Entry 节点**。

- 第 90 至 92 行：设置 ChannelOutboundBuffer 的 `nioBufferCount` 和 `nioBufferSize` 属性。

#### 8.6.1 expandNioBufferArray

`#expandNioBufferArray(ByteBuffer[] array, int neededSpace, int size)` 方法，进行 NIO ByteBuff 数组的**扩容**。代码如下：

```
private static ByteBuffer[] expandNioBufferArray(ByteBuffer[] array, int neededSpace, int size) {
    // 计算扩容后的数组的大小，按照 2 倍计算
    int newCapacity = array.length;
    do {
        // double capacity until it is big enough
        // See https://github.com/netty/netty/issues/1890
        newCapacity <<= 1;

        if (newCapacity < 0) {
            throw new IllegalStateException();
        }

    } while (neededSpace > newCapacity);

    // 创建新的 ByteBuffer 数组
    ByteBuffer[] newArray = new ByteBuffer[newCapacity];

    // 复制老的 ByteBuffer 数组到新的 ByteBuffer 数组中
    System.arraycopy(array, 0, newArray, 0, size);

    return newArray;
}
```

- 代码比较简单，胖友自己看下注释。

#### 8.6.2 nioBufferCount

`#nioBufferCount()` 方法，返回 `nioBufferCount` 属性。代码如下：

```
/**
 * Returns the number of {@link ByteBuffer} that can be written out of the {@link ByteBuffer} array that was
 * obtained via {@link #nioBuffers()}. This method <strong>MUST</strong> be called after {@link #nioBuffers()}
 * was called.
 */
public int nioBufferCount() {
    return nioBufferCount;
}
```

#### 8.6.3 nioBufferSize

`#nioBufferSize()` 方法，返回 `nioBufferSize` 属性。代码如下：

```
/**
 * Returns the number of bytes that can be written out of the {@link ByteBuffer} array that was
 * obtained via {@link #nioBuffers()}. This method <strong>MUST</strong> be called after {@link #nioBuffers()}
 * was called.
 */
public long nioBufferSize() {
    return nioBufferSize;
}
```

### 8.7 removeBytes

`#removeBytes(long writtenBytes)` 方法，移除已经写入 `writtenBytes` 字节对应的 Entry 对象 / 对象们。代码如下：

```
 1: public void removeBytes(long writtenBytes) {
 2:     // 循环移除
 3:     for (;;) {
 4:         // 获得当前消息( 数据 )
 5:         Object msg = current();
 6:         if (!(msg instanceof ByteBuf)) {
 7:             assert writtenBytes == 0;
 8:             break;
 9:         }
10: 
11:         final ByteBuf buf = (ByteBuf) msg;
12:         // 获得消息( 数据 )开始读取位置
13:         final int readerIndex = buf.readerIndex();
14:         // 获得消息( 数据 )可读取的字节数
15:         final int readableBytes = buf.writerIndex() - readerIndex;
16: 
17:         // 当前消息( 数据 )已被写完到对端
18:         if (readableBytes <= writtenBytes) {
19:             if (writtenBytes != 0) {
20:                 // 处理当前消息的 Entry 的写入进度
21:                 progress(readableBytes);
22:                 // 减小 writtenBytes
23:                 writtenBytes -= readableBytes;
24:             }
25:             // 移除当前消息对应的 Entry
26:             remove();
27:         // 当前消息( 数据 )未被写完到对端
28:         } else { // readableBytes > writtenBytes
29:             if (writtenBytes != 0) {
30:                 // 标记当前消息的 ByteBuf 的读取位置
31:                 buf.readerIndex(readerIndex + (int) writtenBytes);
32:                 // 处理当前消息的 Entry 的写入进度
33:                 progress(writtenBytes);
34:             }
35:             break;
36:         }
37:     }
38: 
39:     // 清除 NIO ByteBuff 数组的缓存
40:     clearNioBuffers();
41: }
```

- 第 3 行：

  循环

  ，移除已经写入

   

  ```
  writtenBytes
  ```

   

  字节对应的 Entry 对象。

  - 第 5 行：调用 `#current()` 方法，获得当前消息( 数据 )。
  - 第 12 至 15 行：获得消息( 数据 )开始读取位置和可读取的字节数。
  - `<1>` 当前消息( 数据 )**已**被写完到对端。
  - 第 21 行：调用 `#progress(long amount)` 方法，处理当前消息的 Entry 的写入进度。详细解析，见 [「8.7.1 progress」](http://svip.iocoder.cn/Netty/Channel-5-flush/#)。
  - 第 23 行：减小 `writtenBytes` 。
  - 第 26 行：调用 `#remove()` 方法，移除当前消息对应的 Entry 对象。详细解析，见 [「8.7.2 remove」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。
  - `<2》` 当前消息( 数据 )**未**被写完到对端。
  - 第 31 行：调用 `ByteBuf#readerIndex(readerIndex)` 方法，标记当前消息的 ByteBuf 的**读取位置**。
  - 第 33 行：调用 `#progress(long amount)` 方法，处理当前消息的 Entry 的写入进度。
  - 第 35 行：`break` ，结束循环。

- 第 40 行：调用 `#clearNioBuffers()` 方法，**清除** NIO ByteBuff 数组的缓存。详细解析，见 [「8.7.4 clearNioBuffers」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

#### 8.7.1 progress

`#progress(long amount)` 方法，处理当前消息的 Entry 的写入进度，主要是**通知** Promise 消息写入的进度。代码如下：

```
/**
 * Notify the {@link ChannelPromise} of the current message about writing progress.
 */
  1: public void progress(long amount) {
  2:     Entry e = flushedEntry;
  3:     assert e != null;
  4:     ChannelPromise p = e.promise;
  5:     if (p instanceof ChannelProgressivePromise) {
  6:         // 设置 Entry 对象的 progress 属性
  7:         long progress = e.progress + amount;
  8:         e.progress = progress;
  9:         // 通知 ChannelProgressivePromise 进度
 10:         ((ChannelProgressivePromise) p).tryProgress(progress, e.total);
 11:     }
 12: }
```

- 第 5 行：若 `promise` 的类型是 ChannelProgressivePromise 类型。
- 第 6 至 8 行：设置 Entry 对象的 `progress` 属性。
- 第 10 行：调用 `ChannelProgressivePromise#tryProgress(progress, total)` 方法，通知 ChannelProgressivePromise 进度。

#### 8.7.2 remove

`#remove()` 方法，移除当前消息对应的 Entry 对象，并 Promise 通知成功。代码如下：

```
 1: public boolean remove() {
 2:     Entry e = flushedEntry;
 3:     if (e == null) {
 4:         // 清除 NIO ByteBuff 数组的缓存
 5:         clearNioBuffers();
 6:         return false;
 7:     }
 8:     Object msg = e.msg;
 9: 
10:     ChannelPromise promise = e.promise;
11:     int size = e.pendingSize;
12: 
13:     // 移除指定 Entry 对象
14:     removeEntry(e);
15: 
16:     if (!e.cancelled) {
17:         // 释放消息( 数据 )相关的资源
18:         // only release message, notify and decrement if it was not canceled before.
19:         ReferenceCountUtil.safeRelease(msg);
20:         // 通知 Promise 执行成功
21:         safeSuccess(promise);
22:         // 减少 totalPending 计数
23:         decrementPendingOutboundBytes(size, false, true);
24:     }
25: 
26:     // 回收 Entry 对象
27:     // recycle the entry
28:     e.recycle();
29: 
30:     return true;
31: }
```

- 第 14 行：调用 `#removeEntry(Entry e)` 方法，移除**指定** Entry 对象。详细解析，见 [「8.7.3 removeEntry」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

- 第 16 行：若 Entry 已取消，则忽略。

- 第 19 行：`ReferenceCountUtil#safeRelease(msg)` 方法，释放消息( 数据 )相关的资源。

- 第 21 行：【**重要**】调用 `#safeSuccess(promise)` 方法，通知 Promise 执行成功。此处才是，真正触发 `Channel#write(...)` 或 `Channel#writeAndFlush(...)` 方法，返回的 Promise 的通知。`#safeSuccess(promise)` 方法的代码如下：

  ```
  private static void safeSuccess(ChannelPromise promise) {
      // Only log if the given promise is not of type VoidChannelPromise as trySuccess(...) is expected to return
      // false.
      PromiseNotificationUtil.trySuccess(promise, null, promise instanceof VoidChannelPromise ? null : logger);
  }
  ```

- 第 23 行：`#decrementPendingOutboundBytes(long size, ...)` 方法，减少 `totalPendingSize` 计数。

- 第 28 行：调用 `Entry#recycle()` 方法，**回收** Entry 对象。

#### 8.7.3 removeEntry

`#removeEntry(Entry e)` 方法，移除**指定** Entry 对象。代码如下：

```
 1: private void removeEntry(Entry e) {
 2:     // 已移除完已 flush 的 Entry 节点，置空 flushedEntry、tailEntry、unflushedEntry 。
 3:     if (-- flushed == 0) {
 4:         // processed everything
 5:         flushedEntry = null;
 6:         if (e == tailEntry) {
 7:             tailEntry = null;
 8:             unflushedEntry = null;
 9:         }
10:     // 未移除完已 flush 的 Entry 节点，flushedEntry 指向下一个 Entry 对象
11:     } else {
12:         flushedEntry = e.next;
13:     }
14: }
```

- 第 3 至 9 行：**已**移除完已 flush 的**所有** Entry 节点，置空 `flushedEntry`、`tailEntry`、`unflushedEntry` 。
- 第 10 至 13 行：**未**移除完已 flush 的**所有** Entry 节点，`flushedEntry` 指向**下一个** Entry 对象。

### 8.7.4 clearNioBuffers

`#clearNioBuffers()` 方法，**清除** NIO ByteBuff 数组的缓存。代码如下：

```
// Clear all ByteBuffer from the array so these can be GC'ed.
// See https://github.com/netty/netty/issues/3837
private void clearNioBuffers() {
    int count = nioBufferCount;
    if (count > 0) {
        // 归零 nioBufferCount 。老艿艿觉得，应该把 nioBufferSize 也归零
        nioBufferCount = 0;
        // 置空 NIO ByteBuf 数组
        Arrays.fill(NIO_BUFFERS.get(), 0, count, null);
    }
}
```

- 代码比较简单，胖友自己看注释。主要目的是 help gc 。

### 8.8 failFlushed

`#failFlushed(Throwable cause, boolean notify)` 方法，写入数据到对端**失败**，进行后续的处理，详细看代码。代码如下：

```
 1: void failFlushed(Throwable cause, boolean notify) {
 2:     // 正在通知 flush 失败中，直接返回
 3:     // Make sure that this method does not reenter.  A listener added to the current promise can be notified by the
 4:     // current thread in the tryFailure() call of the loop below, and the listener can trigger another fail() call
 5:     // indirectly (usually by closing the channel.)
 6:     //
 7:     // See https://github.com/netty/netty/issues/1501
 8:     if (inFail) {
 9:         return;
10:     }
11: 
12:     try {
13:         // 标记正在通知 flush 失败中
14:         inFail = true;
15:         // 循环，移除所有已 flush 的 Entry 节点们
16:         for (;;) {
17:             if (!remove0(cause, notify)) {
18:                 break;
19:             }
20:         }
21:     } finally {
22:         // 标记不在通知 flush 失败中
23:         inFail = false;
24:     }
25: }
```

- 第 2 至 10 行：正在通知 flush 失败中，直接返回。
- 第 14 行：标记正在通知 flush 失败中，即 `inFail = true` 。
- 第 15 至 20 行：循环，调用 `#remove0(Throwable cause, boolean notifyWritability)` 方法，移除**所有**已 flush 的 Entry 节点们。详细解析，见 [「8. remove0」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 中。
- 第 21 至 24 行：标记不在通知 flush 失败中，即 `inFail = false` 。

#### 8.8.1 remove0

`#remove0(Throwable cause, boolean notifyWritability)` 方法，移除当前消息对应的 Entry 对象，并 Promise 通知异常。代码如下：

```
 1: private boolean remove0(Throwable cause, boolean notifyWritability) {
 2:     Entry e = flushedEntry;
 3:     // 所有 flush 的 Entry 节点，都已经写到对端
 4:     if (e == null) {
 5:         // // 清除 NIO ByteBuff 数组的缓存
 6:         clearNioBuffers();
 7:         return false; // 没有后续的 flush 的 Entry 节点
 8:     }
 9:     Object msg = e.msg;
10: 
11:     ChannelPromise promise = e.promise;
12:     int size = e.pendingSize;
13: 
14:     removeEntry(e);
15: 
16:     if (!e.cancelled) {
17:         // 释放消息( 数据 )相关的资源
18:         // only release message, fail and decrement if it was not canceled before.
19:         ReferenceCountUtil.safeRelease(msg);
20:         // 通知 Promise 执行失败
21:         safeFail(promise, cause);
22:         // 减少 totalPendingSize 计数
23:         decrementPendingOutboundBytes(size, false, notifyWritability);
24:     }
25: 
26:     // 回收 Entry 对象
27:     // recycle the entry
28:     e.recycle();
29: 
30:     return true; // 还有后续的 flush 的 Entry 节点
31: }
```

- 第 3 至 8 行：若**所有** flush 的 Entry 节点，都已经写到对端，则调用 `#clearNioBuffers()` 方法，清除 NIO ByteBuff 数组的缓存。

- 第 14 行：调用 `#removeEntry(Entry e)` 方法，移除**指定** Entry 对象。详细解析，见 [「8.7.3 removeEntry」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

- 第 16 行：若 Entry 已取消，则忽略。

- 第 19 行：`ReferenceCountUtil#safeRelease(msg)` 方法，释放消息( 数据 )相关的资源。

- 第 21 行：【**重要**】调用 `#safeFail(promise)` 方法，通知 Promise 执行失败。此处才是，真正触发 `Channel#write(...)` 或 `Channel#writeAndFlush(...)` 方法，返回的 Promise 的通知。`#safeFail(promise)` 方法的代码如下：

  ```
  private static void safeFail(ChannelPromise promise, Throwable cause) {
      // Only log if the given promise is not of type VoidChannelPromise as tryFailure(...) is expected to return
      // false.
      PromiseNotificationUtil.tryFailure(promise, cause, promise instanceof VoidChannelPromise ? null : logger);
  }
  ```

- 第 23 行：调用 `#decrementPendingOutboundBytes(long size, ...)` 方法，减少 `totalPendingSize` 计数。

- 第 28 行：调用 `Entry#recycle()` 方法，**回收** Entry 对象。

### 8.9 forEachFlushedMessage

TODO 1015 forEachFlushedMessage 在 `netty-transport-native-poll` 和 `netty-transport-native-kqueue`中使用，在后续的文章解析。

### 8.10 close

`#close(...)` 方法，关闭 ChannelOutboundBuffer ，进行后续的处理，详细看代码。代码如下：

```
void close(ClosedChannelException cause) {
    close(cause, false);
}

  1: void close(final Throwable cause, final boolean allowChannelOpen) {
  2:     // 正在通知 flush 失败中
  3:     if (inFail) {
  4:         // 提交 EventLoop 的线程中，执行关闭
  5:         channel.eventLoop().execute(new Runnable() {
  6:             @Override
  7:             public void run() {
  8:                 close(cause, allowChannelOpen);
  9:             }
 10:         });
 11:         // 返回
 12:         return;
 13:     }
 14: 
 15:     // 标记正在通知 flush 失败中
 16:     inFail = true;
 17: 
 18:     if (!allowChannelOpen && channel.isOpen()) {
 19:         throw new IllegalStateException("close() must be invoked after the channel is closed.");
 20:     }
 21: 
 22:     if (!isEmpty()) {
 23:         throw new IllegalStateException("close() must be invoked after all flushed writes are handled.");
 24:     }
 25: 
 26:     // Release all unflushed messages.
 27:     try {
 28:         // 从 unflushedEntry 节点，开始向下遍历
 29:         Entry e = unflushedEntry;
 30:         while (e != null) {
 31:             // 减少 totalPendingSize
 32:             // Just decrease; do not trigger any events via decrementPendingOutboundBytes()
 33:             int size = e.pendingSize;
 34:             TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, -size);
 35: 
 36:             if (!e.cancelled) {
 37:                 // 释放消息( 数据 )相关的资源
 38:                 ReferenceCountUtil.safeRelease(e.msg);
 39:                 // 通知 Promise 执行失败
 40:                 safeFail(e.promise, cause);
 41:             }
 42:             // 回收当前节点，并获得下一个 Entry 节点
 43:             e = e.recycleAndGetNext();
 44:         }
 45:     } finally {
 46:         // 标记在在通知 flush 失败中
 47:         inFail = false;
 48:     }
 49: 
 50:     // 清除 NIO ByteBuff 数组的缓存。
 51:     clearNioBuffers();
 52: }
```

- 第 3 行：正在通知 flush 失败中：

  - 第 5 至 10 行: 提交 EventLoop 的线程中，执行关闭。
  - 第 12 行：`return` 返回。

- 第 16 行：标记正在通知 flush 失败中，即 `inFail = true` 。

- 第 28 至 30 行：从

   

  ```
  unflushedEntry
  ```

   

  节点，开始向下遍历。

  - 第 31 至 34 行：减少 `totalPendingSize` 计数。
  - 第 36 行：若 Entry 已取消，则忽略。
  - 第 38 行：调用 `ReferenceCountUtil#safeRelease(msg)` 方法，释放消息( 数据 )相关的资源。
  - 第 40 行：【**重要**】调用 `#safeFail(promise)` 方法，通知 Promise 执行失败。此处才是，真正触发 `Channel#write(...)` 或 `Channel#writeAndFlush(...)` 方法，返回的 Promise 的通知。
  - 第 43 行：调用 `Entry#recycleAndGetNext()` 方法，回收当前节点，并获得下一个 Entry 节点。

- 第 45 至 48 行：标记不在通知 flush 失败中，即 `inFail = false` 。

- 第 51 行：调用 `#clearNioBuffers()` 方法，**清除** NIO ByteBuff 数组的缓存。

## 9. NioEventLoop

在上文 [「7. NioSocketChannel」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 中，在写入到 Channel 到对端，若 TCP 数据发送缓冲区**已满**，这将导致 Channel **不写可**，此时会注册对该 Channel 的 `SelectionKey.OP_WRITE` 事件感兴趣。从而实现，再在 Channel 可写后，进行**强制** flush 。这块的逻辑，在 `NioEventLoop#processSelectedKey(SelectionKey k, AbstractNioChannel ch)`中实现，代码如下：

```
// OP_WRITE 事件就绪
// Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
if ((readyOps & SelectionKey.OP_WRITE) != 0) {
    // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
    // 向 Channel 写入数据
    ch.unsafe().forceFlush();
}
```

- 通过 Selector 轮询到 Channel 的 `OP_WRITE` 就绪时，调用 `AbstractNioUnsafe#forceFlush()` 方法，强制 flush 。代码如下：

  ```
  // AbstractNioUnsafe.java
  @Override
  public final void forceFlush() {
      // directly call super.flush0() to force a flush now
      super.flush0();
  }
  ```

  - 后续的逻辑，又回到 [「6. AbstractUnsafe」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 小节的 `#flush0()` 流程。
  - 在完成强制 flush 之后，会取消对 `SelectionKey.OP_WRITE` 事件的感兴趣。

### 9.1 如何模拟

1. 配置服务端 ServerBootstrap 的启动参数如下：

   ```
   .childOption(ChannelOption.SO_SNDBUF, 5) // Socket 参数，TCP 数据发送缓冲区大小。
   ```

2. `telnet` 到启动的服务端，发送相对长的命令，例如 `"abcdefghijklmnopqrstuvw11321321321nhdkslk"` 。

## 10. ChannelOutboundBuffer 写入控制

当我们不断调用 `#addMessage(Object msg, int size, ChannelPromise promise)` 方法，添加消息到 ChannelOutboundBuffer 内存队列中，如果**不及时** flush 写到对端( 例如程序一直未调用 `Channel#flush()` 方法，或者对端接收数据比较慢导致 Channel 不可写 )，可能会导致 **OOM 内存溢出**。所以，在 ChannelOutboundBuffer 使用 `totalPendingSize` 属性，存储所有 Entry 预计占用的内存大小( `pendingSize` )。

- 在 `totalPendingSize` 大于高水位阀值时( `ChannelConfig.writeBufferHighWaterMark` ，默认值为 64 KB )，**关闭**写开关( `unwritable` )。详细解析，见 [「10.1 incrementPendingOutboundBytes」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。
- 在 `totalPendingSize` 小于低水位阀值时( `ChannelConfig.writeBufferLowWaterMark` ，默认值为 32 KB )，**打开**写开关( `unwritable` )。详细解析，见 [「10.2 decrementPendingOutboundBytes」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

该功能，对应 Github 提交为 [《Take memory overhead of ChannelOutboundBuffer / PendingWriteQueue into account》](https://github.com/netty/netty/commit/e3cb9935c0b63357e3d51867cffe624129e7e1dd) 。

### 10.1 incrementPendingOutboundBytes

`#incrementPendingOutboundBytes(long size, ...)` 方法，增加 `totalPendingSize` 计数。代码如下：

```
 1: /**
 2:  * Increment the pending bytes which will be written at some point.
 3:  * This method is thread-safe!
 4:  */
 5: void incrementPendingOutboundBytes(long size) {
 6:     incrementPendingOutboundBytes(size, true);
 7: }
 8: 
 9: private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
10:     if (size == 0) {
11:         return;
12:     }
13: 
14:     // 增加 totalPendingSize 计数
15:     long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);
16:     // totalPendingSize 大于高水位阀值时，设置为不可写
17:     if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {
18:         setUnwritable(invokeLater);
19:     }
20: }
```

- 第 15 行：增加 `totalPendingSize` 计数。

- 第 16 至 19 行：`totalPendingSize` 大于高水位阀值时，调用 `#setUnwritable(boolean invokeLater)` 方法，设置为不可写。代码如下：

  ```
   1: private void setUnwritable(boolean invokeLater) {
   2:     for (;;) {
   3:         final int oldValue = unwritable;
   4:         // 或位操作，修改第 0 位 bits 为 1
   5:         final int newValue = oldValue | 1;
   6:         // CAS 设置 unwritable 为新值
   7:         if (UNWRITABLE_UPDATER.compareAndSet(this, oldValue, newValue)) {
   8:             // 若之前可写，现在不可写，触发 Channel WritabilityChanged 事件到 pipeline 中。
   9:             if (oldValue == 0 && newValue != 0) {
  10:                 fireChannelWritabilityChanged(invokeLater);
  11:             }
  12:             break;
  13:         }
  14:     }
  15: }
  ```

  - 第 2 行：`for` 循环，直到 CAS 修改成功
  - 第 5 行：或位操作，修改第 0 位 bits 为 1 。😈 比较神奇的是，`unwritable` 的类型不是 `boolean` ，而是 `int` 类型。通过每个 bits ，来表示**哪种**类型不可写。感兴趣的胖友，可以看看 `io.netty.handler.traffic.AbstractTrafficShapingHandler` ，使用了第 1、2、3 bits 。
  - 第 7 行：CAS 设置 `unwritable` 为新值。
  - 第 8 至 11 行：若之前可写，现在不可写，调用 `#fireChannelWritabilityChanged(boolean invokeLater)` 方法，触发 Channel WritabilityChanged 事件到 pipeline 中。详细解析，见 [「10.3 fireChannelWritabilityChanged」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

#### 10.1.1 bytesBeforeUnwritable

`#bytesBeforeUnwritable()` 方法，获得距离**不可写**还有多少字节数。代码如下：

```
public long bytesBeforeUnwritable() {
    long bytes = channel.config().getWriteBufferHighWaterMark() - totalPendingSize;
    // If bytes is negative we know we are not writable, but if bytes is non-negative we have to check writability.
    // Note that totalPendingSize and isWritable() use different volatile variables that are not synchronized
    // together. totalPendingSize will be updated before isWritable().
    if (bytes > 0) {
        return isWritable() ? bytes : 0; // 判断 #isWritable() 的原因是，可能已经被设置不可写
    }
    return 0;
}
```

- 基于**高水位**阀值来判断。

### 10.2 decrementPendingOutboundBytes

`#decrementPendingOutboundBytes(long size, ...)` 方法，减少 `totalPendingSize` 计数。代码如下：

```
 1: /**
 2:  * Decrement the pending bytes which will be written at some point.
 3:  * This method is thread-safe!
 4:  */
 5: void decrementPendingOutboundBytes(long size) {
 6:     decrementPendingOutboundBytes(size, true, true);
 7: }
 8: 
 9: private void decrementPendingOutboundBytes(long size, boolean invokeLater, boolean notifyWritability) {
10:     if (size == 0) {
11:         return;
12:     }
13: 
14:     // 减少 totalPendingSize 计数
15:     long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, -size);
16:     // totalPendingSize 小于低水位阀值时，设置为可写
17:     if (notifyWritability && newWriteBufferSize < channel.config().getWriteBufferLowWaterMark()) {
18:         setWritable(invokeLater);
19:     }
20: }
```

- 第 15 行：减少 `totalPendingSize` 计数。

- 第 16 至 19 行：`totalPendingSize` 小于低水位阀值时，调用 `#setWritable(boolean invokeLater)` 方法，设置为可写。代码如下：

  ```
   1: private void setWritable(boolean invokeLater) {
   2:     for (;;) {
   3:         final int oldValue = unwritable;
   4:         // 并位操作，修改第 0 位 bits 为 0
   5:         final int newValue = oldValue & ~1;
   6:         // CAS 设置 unwritable 为新值
   7:         if (UNWRITABLE_UPDATER.compareAndSet(this, oldValue, newValue)) {
   8:             // 若之前不可写，现在可写，触发 Channel WritabilityChanged 事件到 pipeline 中。
   9:             if (oldValue != 0 && newValue == 0) {
  10:                 fireChannelWritabilityChanged(invokeLater);
  11:             }
  12:             break;
  13:         }
  14:     }
  15: }
  ```

  - 第 2 行：`for` 循环，直到 CAS 修改成功
  - 第 5 行：并位操作，修改第 0 位 bits 为 0 。
  - 第 7 行：CAS 设置 `unwritable` 为新值。
  - 第 8 至 11 行：若之前可写，现在不可写，调用 `#fireChannelWritabilityChanged(boolean invokeLater)` 方法，触发 Channel WritabilityChanged 事件到 pipeline 中。详细解析，见 [「10.3 fireChannelWritabilityChanged」](http://svip.iocoder.cn/Netty/Channel-5-flush/#) 。

#### 10.2.1 bytesBeforeWritable

`#bytesBeforeWritable()` 方法，获得距离**可写**还要多少字节数。代码如下：

```
/**
 * Get how many bytes must be drained from the underlying buffer until {@link #isWritable()} returns {@code true}.
 * This quantity will always be non-negative. If {@link #isWritable()} is {@code true} then 0.
 */
public long bytesBeforeWritable() {
    long bytes = totalPendingSize - channel.config().getWriteBufferLowWaterMark();
    // If bytes is negative we know we are writable, but if bytes is non-negative we have to check writability.
    // Note that totalPendingSize and isWritable() use different volatile variables that are not synchronized
    // together. totalPendingSize will be updated before isWritable().
    if (bytes > 0) {
        return isWritable() ? 0 : bytes; // 判断 #isWritable() 的原因是，可能已经被设置不可写
    }
    return 0;
}
```

- 基于**低水位**阀值来判断。

### 10.3 fireChannelWritabilityChanged

`#fireChannelWritabilityChanged(boolean invokeLater)` 方法，触发 Channel WritabilityChanged 事件到 pipeline 中。代码如下：

```
private void fireChannelWritabilityChanged(boolean invokeLater) {
    final ChannelPipeline pipeline = channel.pipeline();
    // 延迟执行，即提交 EventLoop 中触发 Channel WritabilityChanged 事件到 pipeline 中
    if (invokeLater) {
        Runnable task = fireChannelWritabilityChangedTask;
        if (task == null) {
            fireChannelWritabilityChangedTask = task = new Runnable() {
                @Override
                public void run() {
                    pipeline.fireChannelWritabilityChanged();
                }
            };
        }
        channel.eventLoop().execute(task);
    // 直接触发 Channel WritabilityChanged 事件到 pipeline 中
    } else {
        pipeline.fireChannelWritabilityChanged();
    }
}
```

- 根据 `invokeLater` 的值，分成两种方式，调用 `ChannelPipeline#fireChannelWritabilityChanged()` 方法，触发 Channel WritabilityChanged 事件到 pipeline 中。具体，胖友看下代码注释。

- 后续的流程，就是 [《精尽 Netty 源码解析 —— ChannelPipeline（五）之 Inbound 事件的传播》](http://svip.iocoder.cn/Netty/Pipeline-5-inbound/) 。

- 通过 Channel WritabilityChanged 事件，配合

   

  ```
  io.netty.handler.stream.ChunkedWriteHandler
  ```

   

  处理器，实现 ChannelOutboundBuffer 写入的控制，避免 OOM 。ChunkedWriteHandler 的具体代码实现，我们在后续的文章，详细解析。

  - 所以，有一点要注意，ChannelOutboundBuffer 的 `unwritable` 属性，仅仅作为一个是否不可写的**开关**，具体需要配合响应的 ChannelHandler 处理器，才能实现“不可写”的功能。

### 10.4 isWritable

`#isWritable()` 方法，是否可写。代码如下：

```
/**
 * Returns {@code true} if and only if {@linkplain #totalPendingWriteBytes() the total number of pending bytes} did
 * not exceed the write watermark of the {@link Channel} and
 * no {@linkplain #setUserDefinedWritability(int, boolean) user-defined writability flag} has been set to
 * {@code false}.
 */
public boolean isWritable() {
    return unwritable == 0;
}
```

- 如果 `unwritable` 大于 0 ，则表示不可写。😈 一定要注意！！！

#### 10.4.1 getUserDefinedWritability

`#getUserDefinedWritability(int index)` 方法，获得指定 bits 是否可写。代码如下：

```
/**
 * Returns {@code true} if and only if the user-defined writability flag at the specified index is set to
 * {@code true}.
 */
public boolean getUserDefinedWritability(int index) {
    return (unwritable & writabilityMask(index)) == 0;
}

private static int writabilityMask(int index) {
    // 不能 < 1 ，因为第 0 bits 为 ChannelOutboundBuffer 自己使用
    // 不能 > 31 ，因为超过 int 的 bits 范围
    if (index < 1 || index > 31) {
        throw new IllegalArgumentException("index: " + index + " (expected: 1~31)");
    }
    return 1 << index;
}
```

- 为什么方法名字上会带有 `"UserDefined"` 呢？因为 `index` 不能使用 0 ，表示只允许使用用户定义( `"UserDefined"` ) bits 位，即 `[1, 31]` 。

#### 10.4.2 setUserDefinedWritability

`#setUserDefinedWritability(int index, boolean writable)` 方法，设置指定 bits 是否可写。代码如下：

```
/**
 * Sets a user-defined writability flag at the specified index.
 */
public void setUserDefinedWritability(int index, boolean writable) {
    // 设置可写
    if (writable) {
        setUserDefinedWritability(index);
    // 设置不可写
    } else {
        clearUserDefinedWritability(index);
    }
}

private void setUserDefinedWritability(int index) {
    final int mask = ~writabilityMask(index);
    for (;;) {
        final int oldValue = unwritable;
        final int newValue = oldValue & mask;
        // CAS 设置 unwritable 为新值
        if (UNWRITABLE_UPDATER.compareAndSet(this, oldValue, newValue)) {
            // 若之前不可写，现在可写，触发 Channel WritabilityChanged 事件到 pipeline 中。
            if (oldValue != 0 && newValue == 0) {
                fireChannelWritabilityChanged(true);
            }
            break;
        }
    }
}

private void clearUserDefinedWritability(int index) {
    final int mask = writabilityMask(index);
    for (;;) {
        final int oldValue = unwritable;
        final int newValue = oldValue | mask;
        if (UNWRITABLE_UPDATER.compareAndSet(this, oldValue, newValue)) {
            // 若之前可写，现在不可写，触发 Channel WritabilityChanged 事件到 pipeline 中。
            if (oldValue == 0 && newValue != 0) {
                fireChannelWritabilityChanged(true);
            }
            break;
        }
    }
}
```

- 代码比较简单，胖友自己看噢。

## 666. 彩蛋

比想象中，长的多的多的一篇文章。总的来说，绝大部分细节，都已经扣到，美滋滋。如果有解释不够清晰或错误的细节，一起多多沟通呀。

写完这篇，我简直疯了。。。。

推荐阅读文章：

- 莫那一鲁道 [《Netty 出站缓冲区 ChannelOutboundBuffer 源码解析（isWritable 属性的重要性）》](https://www.jianshu.com/p/311425d1c72f)
- tomas家的小拨浪鼓 [《Netty 源码解析 ——— writeAndFlush流程分析》](https://www.jianshu.com/p/a3443cacd081)
- 闪电侠 [《netty 源码分析之 writeAndFlush 全解析》](https://www.jianshu.com/p/feaeaab2ce56)
- 占小狼 [《深入浅出 Netty write》](https://www.jianshu.com/p/1ad424c53e80)
- Hypercube [《自顶向下深入分析Netty（六）–Channel源码实现》](https://www.jianshu.com/p/9258af254e1d)

# 精尽 Netty 源码解析 —— Channel（六）之 writeAndFlush 操作

2018-07-16

## 1. 概述

本文接 [《精尽 Netty 源码解析 —— Channel（五）之 flush 操作》](http://svip.iocoder.cn/Netty/Channel-5-flush/) ，分享 Netty Channel 的 `#writeAndFlush(Object msg, ...)` 方法，write + flush 的组合，将数据写到内存队列后，立即刷新**内存队列**，又将其中的数据写入到对端。

😈 本来是不准备写这篇的，因为内容主要是 [《精尽 Netty 源码解析 —— Channel（四）之 write 操作》](http://svip.iocoder.cn/Netty/Channel-4-write/) 和 [《精尽 Netty 源码解析 —— Channel（五）之 flush 操作》](http://svip.iocoder.cn/Netty/Channel-5-flush/) 的组合。但是，考虑到内容的完整性，于是乎就稍微水更下下。

### 2. AbstractChannel

AbstractChannel 对 `#writeAndFlush(Object msg, ...)` 方法的实现，代码如下：

```
@Override
public ChannelFuture writeAndFlush(Object msg) {
    return pipeline.writeAndFlush(msg);
}

@Override
public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
    return pipeline.writeAndFlush(msg, promise);
}
```

- 在方法内部，会调用对应的

   

  ```
  ChannelPipeline#write(Object msg, ...)
  ```

   

  方法，将 write 和 flush

   

  两个

  事件在 pipeline 上传播。详细解析，见

   

  「3. DefaultChannelPipeline」

   

  。

  - 最终会传播 write 事件到 `head` 节点，将数据写入到内存队列中。详细解析，见 [「5. HeadContext」](http://svip.iocoder.cn/Netty/Channel-6-writeAndFlush/#) 。
  - 最终会传播 flush 事件到 `head` 节点，刷新**内存队列**，将其中的数据写入到对端。详细解析，见 [「5. HeadContext」](http://svip.iocoder.cn/Netty/Channel-6-writeAndFlush/#) 。

## 3. DefaultChannelPipeline

`DefaultChannelPipeline#writeAndFlush(Object msg, ...)` 方法，代码如下：

```
@Override
public final ChannelFuture write(Object msg) {
    return tail.writeAndFlush(msg);
}

@Override
public final ChannelFuture write(Object msg, ChannelPromise promise) {
    return tail.writeAndFlush(msg, promise);
}
```

- 在方法内部，会调用 `TailContext#writeAndFlush(Object msg, ...)` 方法，将 write 和 flush **两个**事件在 pipeline 中，从尾节点向头节点传播。详细解析，见 [「4. TailContext」](http://svip.iocoder.cn/Netty/Channel-6-writeAndFlush/#) 。

## 4. TailContext

TailContext 对 `TailContext#writeAndFlush(Object msg, ...)` 方法的实现，是从 AbstractChannelHandlerContext 抽象类继承，代码如下：

```
@Override
public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
    if (msg == null) {
        throw new NullPointerException("msg");
    }

    // 判断是否为合法的 Promise 对象
    if (isNotValidPromise(promise, true)) {
        // 释放消息( 数据 )相关的资源
        ReferenceCountUtil.release(msg);
        // cancelled
        return promise;
    }

    // 写入消息( 数据 )到内存队列
    write(msg, true, promise); // <1>

    return promise;
}
```

- 这个方法，和我们在 [《精尽 Netty 源码解析 —— Channel（四）之 write 操作》](http://svip.iocoder.cn/Netty/Channel-4-write/) 的 [「4. TailContext」](http://svip.iocoder.cn/Netty/Channel-6-writeAndFlush/#) 的小节，`TailContext#write(Object msg, ...)` 方法，基本类似，差异在于 `<1>` 处，调用 `#write(Object msg, boolean flush, ChannelPromise promise)` 方法，传入的 `flush = true` 方法参数，表示 write 操作的同时，**后续**需要执行 flush 操作。代码如下：

  ```
  private void write(Object msg, boolean flush, ChannelPromise promise) {
      // 获得下一个 Outbound 节点
      AbstractChannelHandlerContext next = findContextOutbound();
      // 简化代码 😈
      // 执行 write + flush 操作
      next.invokeWriteAndFlush(m, promise);
  }
  
  private void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
      if (invokeHandler()) {
          // 执行 write 事件到下一个节点
          invokeWrite0(msg, promise);
          // 执行 flush 事件到下一个节点
          invokeFlush0();
      } else {
          writeAndFlush(msg, promise);
      }
  }
  ```

  - 在后面，就是 [《精尽 Netty 源码解析 —— Channel（四）之 write 操作》](http://svip.iocoder.cn/Netty/Channel-4-write/) 的 [「5. HeadContext」](http://svip.iocoder.cn/Netty/Channel-6-writeAndFlush/#) 的小节及其后续的小节。
  - 再在后面，就是 [《精尽 Netty 源码解析 —— Channel（五）之 flush 操作》](http://svip.iocoder.cn/Netty/Channel-5-flush/) 。

## 666. 彩蛋

😈 真的是水更，哈哈哈哈。

推荐阅读文章：

- 闪电侠 [《netty 源码分析之 writeAndFlush 全解析》](https://www.jianshu.com/p/feaeaab2ce56) 的 [「writeAndFlush: 写队列并刷新」](http://svip.iocoder.cn/Netty/Channel-6-writeAndFlush/#) 小节。

# 精尽 Netty 源码解析 —— Channel（七）之 close 操作

2018-07-19

## 1. 概述

本文分享 Netty NIO Channel 关闭( **close** )操作的过程，分成客户端和服务端 Channel **两种**关闭：

- 客户端 NioSocketChannel
  - 客户端关闭 NioSocketChannel ，断开和服务端的连接。
  - 服务端关闭 NioSocketChannel ，断开和客户端的连接。
- 服务端 NioServerSocketChannel
  - 服务端关闭 NioServerSocketChannel ，取消端口绑定，关闭服务。

上面的关闭，可能是客户端/服务端主动关闭，也可能是异常关闭。

- 关于 NioSocketChannel 的关闭，在 [「2. NioSocketChannel」](http://svip.iocoder.cn/Netty/Channel-7-close/#) 详细解析。
- 关于 NioServerSocketChannel 的关闭，在 [「3. NioSocketChannel」](http://svip.iocoder.cn/Netty/Channel-7-close/#) 详细解析。

## 2. NioSocketChannel

通过 `NioSocketChannel#close()` 方法，应用程序里可以主动关闭 NioSocketChannel 通道。代码如下：

```
// AbstractChannel.java
@Override
public ChannelFuture close() {
    return pipeline.close();
}
```

- NioSocketChannel 继承 AbstractChannel 抽象类，所以 `#close()` 方法实际是 AbstractChannel 实现的。

- 在方法内部，会调用对应的 `ChannelPipeline#close()` 方法，将 close 事件在 pipeline 上传播。而 close 事件属于 Outbound 事件，所以会从 `tail` 节点开始，最终传播到 `head` 节点，使用 Unsafe 进行关闭。代码如下：

  ```
  // DefaultChannelPipeline.java
  @Override
  public final ChannelFuture close() {
      return tail.close();
  }
  
  // TailContext.java
  @Override // FROM AbstractChannelHandlerContext.java 。因为 TailContext 继承 AbstractChannelHandlerContext 抽象类，该方法是它实现的。
  public ChannelFuture close() {
      return close(newPromise());
  }
  
  // HeadContext.java
  @Override
  public void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
      unsafe.close(promise);
  }
  ```

### 2.1 AbstractUnsafe#close

`AbstractUnsafe#close()` 方法，关闭 Channel 。代码如下：

```
@Override
public final void close(final ChannelPromise promise) {
    assertEventLoop();

    // 关闭
    close(promise, CLOSE_CLOSED_CHANNEL_EXCEPTION, CLOSE_CLOSED_CHANNEL_EXCEPTION, false);
}

  1: private void close(final ChannelPromise promise, final Throwable cause, final ClosedChannelException closeCause, final boolean notify) {
  2:     // 设置 Promise 不可取消
  3:     if (!promise.setUncancellable()) {
  4:         return;
  5:     }
  6: 
  7:     // 若关闭已经标记初始化
  8:     if (closeInitiated) {
  9:         // 关闭已经完成，直接通知 Promise 对象
 10:         if (closeFuture.isDone()) {
 11:             // Closed already.
 12:             safeSetSuccess(promise);
 13:         // 关闭未完成，通过监听器通知 Promise 对象
 14:         } else if (!(promise instanceof VoidChannelPromise)) { // Only needed if no VoidChannelPromise.
 15:             // This means close() was called before so we just register a listener and return
 16:             closeFuture.addListener(new ChannelFutureListener() {
 17:                 @Override
 18:                 public void operationComplete(ChannelFuture future) throws Exception {
 19:                     promise.setSuccess();
 20:                 }
 21:             });
 22:         }
 23:         return;
 24:     }
 25: 
 26:     // 标记关闭已经初始化
 27:     closeInitiated = true;
 28: 
 29:     // 获得 Channel 是否激活
 30:     final boolean wasActive = isActive();
 31:     // 标记 outboundBuffer 为空
 32:     final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
 33:     this.outboundBuffer = null; // Disallow adding any messages and flushes to outboundBuffer.
 34:     // 执行准备关闭
 35:     Executor closeExecutor = prepareToClose();
 36:     // 若 closeExecutor 非空
 37:     if (closeExecutor != null) {
 38:         closeExecutor.execute(new Runnable() {
 39:             @Override
 40:             public void run() {
 41:                 try {
 42:                     // 在 closeExecutor 中，执行关闭
 43:                     // Execute the close.
 44:                     doClose0(promise);
 45:                 } finally {
 46:                     // 在 EventLoop 中，执行
 47:                     // Call invokeLater so closeAndDeregister is executed in the EventLoop again!
 48:                     invokeLater(new Runnable() {
 49:                         @Override
 50:                         public void run() {
 51:                             if (outboundBuffer != null) {
 52:                                 // 写入数据( 消息 )到对端失败，通知相应数据对应的 Promise 失败。
 53:                                 // Fail all the queued messages
 54:                                 outboundBuffer.failFlushed(cause, notify);
 55:                                 // 关闭内存队列
 56:                                 outboundBuffer.close(closeCause);
 57:                             }
 58:                             // 执行取消注册，并触发 Channel Inactive 事件到 pipeline 中
 59:                             fireChannelInactiveAndDeregister(wasActive);
 60:                         }
 61:                     });
 62:                 }
 63:             }
 64:         });
 65:     // 若 closeExecutor 为空
 66:     } else {
 67:         try {
 68:             // 执行关闭
 69:             // Close the channel and fail the queued messages in all cases.
 70:             doClose0(promise);
 71:         } finally {
 72:             if (outboundBuffer != null) {
 73:                 // 写入数据( 消息 )到对端失败，通知相应数据对应的 Promise 失败。
 74:                 // Fail all the queued messages.
 75:                 outboundBuffer.failFlushed(cause, notify);
 76:                 // 关闭内存队列
 77:                 outboundBuffer.close(closeCause);
 78:             }
 79:         }
 80:         // 正在 flush 中，在 EventLoop 中执行执行取消注册，并触发 Channel Inactive 事件到 pipeline 中
 81:         if (inFlush0) {
 82:             invokeLater(new Runnable() {
 83:                 @Override
 84:                 public void run() {
 85:                     fireChannelInactiveAndDeregister(wasActive);
 86:                 }
 87:             });
 88:         // 不在 flush 中，直接执行执行取消注册，并触发 Channel Inactive 事件到 pipeline 中
 89:         } else {
 90:             fireChannelInactiveAndDeregister(wasActive);
 91:         }
 92:     }
 93: }
```

- 方法参数 `cause`、`closeCause` ，关闭的“原因”。对于 **close** 操作来说，无论是正常关闭，还是异常关闭，通过使用 **Exception** 来表示**来源**。在 AbstractChannel 类中，枚举了所有来源：

  ```
  // AbstractChannel.java
  private static final ClosedChannelException FLUSH0_CLOSED_CHANNEL_EXCEPTION = ThrowableUtil.unknownStackTrace(
          new ClosedChannelException(), AbstractUnsafe.class, "flush0()");
  private static final ClosedChannelException ENSURE_OPEN_CLOSED_CHANNEL_EXCEPTION = ThrowableUtil.unknownStackTrace(
          new ClosedChannelException(), AbstractUnsafe.class, "ensureOpen(...)");
  private static final ClosedChannelException CLOSE_CLOSED_CHANNEL_EXCEPTION = ThrowableUtil.unknownStackTrace(
          new ClosedChannelException(), AbstractUnsafe.class, "close(...)");
  private static final ClosedChannelException WRITE_CLOSED_CHANNEL_EXCEPTION = ThrowableUtil.unknownStackTrace(
          new ClosedChannelException(), AbstractUnsafe.class, "write(...)");
  private static final NotYetConnectedException FLUSH0_NOT_YET_CONNECTED_EXCEPTION = ThrowableUtil.unknownStackTrace(
          new NotYetConnectedException(), AbstractUnsafe.class, "flush0()");
  ```

- 第 2 至 5 行：调用 `ChannelPromise#setUncancellable()` 方法，设置 Promise 不可取消。

- 第 8 行：若

   

  ```
  AbstractChannel.closeInitiated
  ```

   

  为

   

  ```
  true
  ```

   

  时，表示关闭已经标记初始化，此时

  可能

  已经关闭完成。

  - 第 10 至 12 行：关闭**已经**完成，直接通知 Promise 对象。
  - 第 13 至 22 行：关闭**并未**完成，通过监听器回调通知 Promise 对象。
  - 第 23 行：`return` 结束。
  - 第 27 行：标记关闭已经初始化。

- 第 30 行：调用 `#isActive()` 方法， 获得 Channel 是否激活。

- 第 31 至 33 行：标记内存队列 `outboundBuffer` 为空。

- 第 35 行：调用 `#prepareToClose()` 方法，执行准备关闭。详细解析，胖友先跳到 [「2.2 NioSocketChannelUnsafe#prepareToClose」](http://svip.iocoder.cn/Netty/Channel-7-close/#) 中。

- 第 37 行：若

   

  ```
  closeExecutor
  ```

   

  非空，在

   

  「2.2 NioSocketChannelUnsafe#prepareToClose」

   

  中，我们已经看到如果开启

   

  ```
  SO_LINGER
  ```

   

  功能，会返回

   

  ```
  GlobalEventExecutor.INSTANCE
  ```

   

  对象。

  - 第 38 至 44 行：提交任务到 `closeExecutor` 中，**在它的线程中**，执行 `#doClose0(promise)` 方法，执行关闭。为什么要在“在它的线程中”中？回答不出来的胖友，再好好重新看下 [「2.2 NioSocketChannelUnsafe#prepareToClose」](http://svip.iocoder.cn/Netty/Channel-7-close/#) 小节。
  - 第 46 至 61 行：提交任务到 Channel 所在的 EventLoop 中，执行后续的任务。
  - 整体的逻辑和代码，和【第 66 至 91 行】的代码是**基本**一致。

- 第 66 行：若

   

  ```
  closeExecutor
  ```

   

  为空。

  - 第 70 行：调用 `#doClose0(promise)` 方法，执行**真正的**关闭。详细解析，胖友先跳到 [「2.4 doClose0」](http://svip.iocoder.cn/Netty/Channel-7-close/#) 中。

  - 第 75 行：调用 `ChannelOutboundBuffer#failFlushed(Throwable cause, boolean notify)` 方法，写入数据( 消息 )到对端失败，通知相应数据对应的 Promise 失败。详细解析，见 [《精尽 Netty 源码解析 —— Channel（五）之 flush 操作》](http://svip.iocoder.cn/Netty/Channel-5-flush/) 。

  - 第 77 行：调用 `ChannelOutboundBuffer#close(Throwable cause)` 方法，关闭内存队列。详细解析，见 [《精尽 Netty 源码解析 —— Channel（五）之 flush 操作》](http://svip.iocoder.cn/Netty/Channel-5-flush/) 。

  - 第 81 行：若

     

    ```
    inFlush0
    ```

     

    为

     

    ```
    true
    ```

     

    ，

    正在

     

    flush 中，

    在 EventLoop 中的线程中

    ，调用

     

    ```
    #fireChannelInactiveAndDeregister(boolean wasActive)
    ```

     

    方法，执行取消注册，并触发 Channel Inactive 事件到 pipeline 中。详细解析，见

     

    「2.5 AbstractUnsafe#fireChannelInactiveAndDeregister」

     

    中。

    - 第 90 行：若 `inFlush0` 为 `false` ，**不在** flush 中，**直接**调用 `#fireChannelInactiveAndDeregister(boolean wasActive)` 方法，执行取消注册，并触发 Channel Inactive 事件到 pipeline 中。

### 2.2 NioSocketChannelUnsafe#prepareToClose

`NioSocketChannelUnsafe#prepareToClose()` 方法，执行准备关闭。代码如下：

```
 1: @Override
 2: protected Executor prepareToClose() {
 3:     try {
 4:         if (javaChannel().isOpen() && config().getSoLinger() > 0) {
 5:             // We need to cancel this key of the channel so we may not end up in a eventloop spin
 6:             // because we try to read or write until the actual close happens which may be later due
 7:             // SO_LINGER handling.
 8:             // See https://github.com/netty/netty/issues/4449
 9:             doDeregister();
10:             // 返回 GlobalEventExecutor 对象
11:             return GlobalEventExecutor.INSTANCE;
12:         }
13:     } catch (Throwable ignore) {
14:         // Ignore the error as the underlying channel may be closed in the meantime and so
15:         // getSoLinger() may produce an exception. In this case we just return null.
16:         // See https://github.com/netty/netty/issues/4449
17:     }
18:     return null;
19: }
```

- 第 4 行：如果配置 `StandardSocketOptions.SO_LINGER` 大于 0 。让我们先来看下它的定义：

  ```
  Socket 参数，关闭 Socket 的延迟时间，Netty 默认值为 -1 ，表示禁用该功能。
  
  * -1 表示 socket.close() 方法立即返回，但 OS 底层会将发送缓冲区全部发送到对端。
  * 0 表示 socket.close() 方法立即返回，OS 放弃发送缓冲区的数据直接向对端发送RST包，对端收到复位错误。
  * 非 0 整数值表示调用 socket.close() 方法的线程被阻塞直到延迟时间到或发送缓冲区中的数据发送完毕，若超时，则对端会收到复位错误。
  ```

  - 按照这个定义，如果**大于 0**，如果在**真正关闭** Channel ，需要**阻塞**直到延迟时间到或发送缓冲区中的数据发送完毕。
  - 如果在 EventLoop 中执行**真正关闭** Channel 的操作，那么势必会阻塞 EventLoop 的线程。所以，在【第 11 行】的代码，返回 `GlobalEventExecutor.INSTANCE` 对象，作为执行**真正关闭** Channel 的操作的**执行器**( 它也有一个自己的线程哟 )。

- 第 9 行：调用 `#doDeregister()` 方法，执行取消注册。详细解析，胖友先跳到 [「2.2 AbstractUnsafe#doDeregister」](http://svip.iocoder.cn/Netty/Channel-7-close/#) 中。

- 【来自我表弟普架的牛逼解答，我表示点赞支持】第 9 行的：为什么要调用

   

  ```
  #doDeregister()
  ```

   

  方法呢？因为

   

  ```
  SO_LINGER
  ```

   

  大于 0 时，

  真正关闭

   

  Channel ，需要

  阻塞

  直到延迟时间到或发送缓冲区中的数据发送完毕。如果不取消该 Channel 的

   

  ```
  SelectionKey.OP_READ
  ```

   

  事件的感兴趣，就会不断触发读事件，导致 CPU 空轮询。为什么呢?在 Channel 关闭时，会

  自动

  触发

   

  ```
  SelectionKey.OP_READ
  ```

   

  事件。而且，会不断不断不断的触发，如果不进行取消

   

  ```
  SelectionKey.OP_READ
  ```

   

  事件的感兴趣。

  - 😈 感叹一句，细思极恐啊，厉害了，Netty 。

- 第 11 行：如果开启 `SO_LINGER` 功能，返回 `GlobalEventExecutor.INSTANCE` 对象。

- 第 18 行：若果关闭 `SO_LINGER` 功能，返回 `null` 对象。

- 😈 胖友，调回 [「2.1 AbstractUnsafe#close」](http://svip.iocoder.cn/Netty/Channel-7-close/#) 继续把。

### 2.3 AbstractUnsafe#doDeregister

`AbstractUnsafe#doDeregister()` 方法，执行取消注册。代码如下：

```
@Override
protected void doDeregister() throws Exception {
    eventLoop().cancel(selectionKey());
}
```

- 调用 `EventLoop#cancel(SelectionKey key)` 方法，取消 SelectionKey ，即相当于调用 `SelectionKey#cancel()` 方法。如此，对通道的读写等等 IO 就绪事件不再感兴趣，也不会做出相应的处理。

### 2.4 AbstractUnsafe#doClose0

`AbstractUnsafe#doClose0(ChannelPromise promise)` 方法，执行**真正的**关闭。代码如下：

```
 1: private void doClose0(ChannelPromise promise) {
 2:     try {
 3:         // 执行关闭
 4:         doClose();
 5:         // 通知 closeFuture 关闭完成
 6:         closeFuture.setClosed();
 7:         // 通知 Promise 关闭成功
 8:         safeSetSuccess(promise);
 9:     } catch (Throwable t) {
10:         // 通知 closeFuture 关闭完成
11:         closeFuture.setClosed();
12:         // 通知 Promise 关闭异常
13:         safeSetFailure(promise, t);
14:     }
15: }
```

- 第 4 行：调用 `#doClose()` 方法，执行关闭。这是一个**抽象**方法，NioSocketChannel 对它的实现，胖友先跳到 [「2.4.1 NioSocketChannel#doClose」 ](http://svip.iocoder.cn/Netty/Channel-7-close/#)中。

- 第 6 行：调用 `CloseFuture#setClosed()` 方法，通知 `closeFuture` 关闭完成。此处就会结束我们在 EchoClient 的阻塞监听客户端关闭。例如：

  ```
  // Wait until the connection is closed.
  // 监听客户端关闭，并阻塞等待
  f.channel().closeFuture().sync();
  ```

  - 哟哟哟，就要结束阻塞等待了。

- 第 8 行：调用 `#safeSetSuccess(promise)` 方法，通知 通知 Promise 关闭**成功**。此处就会回调我们对 `Channel#close()`方法的返回的 ChannelFuture 的监听。示例如下：

  ```
  ctx.channel().close().addListener(new ChannelFutureListener() { // 我是一个萌萌哒监听器
      @Override
      public void operationComplete(ChannelFuture future) throws Exception {
          System.out.println(Thread.currentThread() + "我会被唤醒");
      }
  });
  ```

  - 哟哟哟，要被回调了。

- 若发生异常：

  - 第 11 行：调用 `CloseFuture#setClosed()` 方法，通知 `closeFuture` 关闭完成。
  - 第 13 行: 调用 `#safeSetFailure(promise, t)` 方法，通知 通知 Promise 关闭**异常**。

#### 2.4.1 NioSocketChannel#doClose

`NioSocketChannel#doClose()` 方法，执行 Java 原生 NIO SocketChannel 关闭。代码如下：

```
1: @Override
2: protected void doClose() throws Exception {
3:     // 执行父类关闭方法
4:     super.doClose();
5:     // 执行 Java 原生 NIO SocketChannel 关闭
6:     javaChannel().close();
7: }
```

- 第 4 行：调用 `AbstractNioChannel#doClose()` 方法，执行**父类**关闭方法。代码如下：

  ```
  @Override
  protected void doClose() throws Exception {
      // 通知 connectPromise 异常失败
      ChannelPromise promise = connectPromise;
      if (promise != null) {
          // Use tryFailure() instead of setFailure() to avoid the race against cancel().
          promise.tryFailure(DO_CLOSE_CLOSED_CHANNEL_EXCEPTION);
          connectPromise = null;
      }
  
      // 取消 connectTimeoutFuture 等待
      ScheduledFuture<?> future = connectTimeoutFuture;
      if (future != null) {
          future.cancel(false);
          connectTimeoutFuture = null;
      }
  }
  ```

  - 适用于客户端**正在**发起对服务端的连接的阶段。

- 【重要】第 6 行：调用 `SocketChannel#close()` 方法，执行 Java 原生 NIO SocketChannel 关闭。

### 2.5 AbstractUnsafe#fireChannelInactiveAndDeregister

`AbstractUnsafe#fireChannelInactiveAndDeregister(boolean wasActive)` 方法，执行取消注册，并触发 Channel Inactive 事件到 pipeline 中。代码如下：

```
private void fireChannelInactiveAndDeregister(final boolean wasActive) {
    deregister(voidPromise() /** <1> **/, wasActive && !isActive() /** <2> **/); 
}

  1: private void deregister(final ChannelPromise promise, final boolean fireChannelInactive) {
  2:     // 设置 Promise 不可取消
  3:     if (!promise.setUncancellable()) {
  4:         return;
  5:     }
  6: 
  7:     // 不处于已经注册状态，直接通知 Promise 取消注册成功。
  8:     if (!registered) {
  9:         safeSetSuccess(promise);
 10:         return;
 11:     }
 12: 
 13:     // As a user may call deregister() from within any method while doing processing in the ChannelPipeline,
 14:     // we need to ensure we do the actual deregister operation later. This is needed as for example,
 15:     // we may be in the ByteToMessageDecoder.callDecode(...) method and so still try to do processing in
 16:     // the old EventLoop while the user already registered the Channel to a new EventLoop. Without delay,
 17:     // the deregister operation this could lead to have a handler invoked by different EventLoop and so
 18:     // threads.
 19:     //
 20:     // See:
 21:     // https://github.com/netty/netty/issues/4435
 22:     invokeLater(new Runnable() {
 23:         @Override
 24:         public void run() {
 25:             try {
 26:                 // 执行取消注册
 27:                 doDeregister();
 28:             } catch (Throwable t) {
 29:                 logger.warn("Unexpected exception occurred while deregistering a channel.", t);
 30:             } finally {
 31:                 // 触发 Channel Inactive 事件到 pipeline 中
 32:                 if (fireChannelInactive) {
 33:                     pipeline.fireChannelInactive();
 34:                 }
 35: 
 36:                 // Some transports like local and AIO does not allow the deregistration of
 37:                 // an open channel.  Their doDeregister() calls close(). Consequently,
 38:                 // close() calls deregister() again - no need to fire channelUnregistered, so check
 39:                 // if it was registered.
 40:                 if (registered) {
 41:                     // 标记为未注册
 42:                     registered = false;
 43:                     // 触发 Channel Unregistered 事件到 pipeline 中
 44:                     pipeline.fireChannelUnregistered();
 45:                 }
 46: 
 47:                 // 通知 Promise 取消注册成功。
 48:                 safeSetSuccess(promise);
 49:             }
 50:         }
 51:     });
 52: }
```

- `<1>` 处，传入 `#deregister(...)` 方法的第一个参数为 `unsafeVoidPromise` ，类型为 VoidChannelPromise **类**，表示需要通知 Promise 。为什么这么说呢？在 `#safeSetSuccess(promise)` 方法中，可以看到：

  ```
  protected final void safeSetSuccess(ChannelPromise promise) {
      if (!(promise instanceof VoidChannelPromise) && !promise.trySuccess()) {
          logger.warn("Failed to mark a promise as success because it is done already: {}", promise);
      }
  }
  ```

  - `!(promise instanceof VoidChannelPromise)` 代码块，表示排除 VoidChannelPromise 类型的 `promise` 。

- `<2>` 处，通过对比新老的 `active` 的值，判断是否 Channel 的状态是否从 Active 变成 Inactive 。

- 第 2 至 5 行：调用 `ChannelPromise#setUncancellable()` 方法，设置 Promise 不可取消。

- 第 7 至 11 行：不处于已经注册状态，直接通知 Promise 取消注册成功，并

   

  ```
  return
  ```

   

  返回。

  - 😈 在当前情况下，`registered = true` ，所以不符合条件。

- 第 22 行：调用

   

  ```
  #invokeLater(Runnable)
  ```

   

  方法，提交任务到 EventLoop 的线程中执行，以避免

  一个

   

  Channel 的 ChannelHandler 在

  不同

  的 EventLoop 或者线程中执行。详细的说明，可以看下【第 13 至 21 行】的英文说明。

  - 😈 实际从目前该方法的调用看下来，有可能不是从 EventLoop 的线程中调用。

- 第 27 行：调用 `AbstractUnsafe#doDeregister()` 方法，执行取消注册。在 [「2.3 AbstractUnsafe#doDeregister」](http://svip.iocoder.cn/Netty/Channel-7-close/#) 中，已经详细解析。

- 第 31 至 34 行：如果 `fireChannelInactive = true` ，调用 `ChannelPipeline#fireChannelInactive()` 方法，触发 Channel Inactive 事件到 pipeline 中。而 Channel Inactive 事件属于 Inbound 事件，所以会从 `head` 节点开始，最终传播到 `tail` 节点，目前并未执行什么逻辑，感兴趣的胖友，可以自己去看看。如果胖友业务上有需要，可以自己添加 ChannelHandler 进行处理。

- 第 40 至 42 行：标记为未注册。

- 第 44 行：调用

   

  ```
  ChannelPipeline#fireChannelUnregistered()
  ```

   

  方法，触发 Channel Unregistered 事件到 pipeline 中。而 Channel Unregistered 事件属于 Inbound 事件，所以会从

   

  ```
  head
  ```

   

  节点开始，最终传播到

   

  ```
  tail
  ```

   

  节点，目前并未执行什么逻辑，感兴趣的胖友，可以自己去看看。如果胖友业务上有需要，可以自己添加 ChannelHandler 进行处理。

  - 😈 又啰嗦了一遍，【第 31 至 34 行】的代码的逻辑。

- 第 48 行：调用 `#safeSetSuccess(promise)` 方法，通知 Promise 取消注册成功。

## 3. NioServerSocketChannel

通过 `NioServerSocketChannel#close()` 方法，应用程序里可以主动关闭 NioServerSocketChannel 通道。在具体的代码实现上，唯一的差别就是对 `AbstractNioChannel#doClose()` 方法的实现不同( 对应 [「2.4.1 NioSocketChannel#doClose」](http://svip.iocoder.cn/Netty/Channel-7-close/#) )。代码如下：

`NioSocketChannel#doClose()` 方法，执行 Java 原生 NIO SocketServerChannel 关闭。代码如下：

```
@Override
protected void doClose() throws Exception {
    javaChannel().close();
}
```

- 调用 `SocketServerChannel#close()` 方法，执行 Java 原生 NIO SocketServerChannel 关闭。

------

那么可能会有胖友有疑惑了，`#close()` 方法的实现，99.99% 都相似，那么 NioSocketChannel 和 NioServerSocketChannel 差异的关闭逻辑怎么实现呢？答案其实很简单，通过给它们配置不同的 ChannelHandler 实现类即可。

## 4. Unsafe#closeForcibly

实际上，在 Unsafe 接口上定义了 `#closeForcibly()` 方法，英文注释如下：

```
/**
 * Closes the {@link Channel} immediately without firing any events.  Probably only useful
 * when registration attempt failed.
 */
void closeForcibly();
```

- 立即关闭 Channel ，并且不触发 pipeline 上的任何事件。
- 仅仅用于 Channel 注册到 EventLoop 上失败的情况下。😈 这也就是为什么 `without firing any events` 的原因啦。

AbstractUnsafe 对该接口方法，实现代码如下：

```
@Override
public final void closeForcibly() {
    assertEventLoop();

    try {
        doClose();
    } catch (Exception e) {
        logger.warn("Failed to close a channel.", e);
    }
}
```

- 在方法内部，调用 `AbstractNioChannel#doClose()` 方法，执行 Java 原生 NIO SocketServerChannel 或 SocketChannel 关闭。
- 并且，从代码实现上，我们可以看到，确实并未触发任何 pipeline 上的事件。

## 5. 服务端处理客户端主动关闭连接

在客户端主动关闭时，服务端会收到一个 `SelectionKey.OP_READ` 事件的就绪，在调用客户端对应在服务端的 SocketChannel 的 `#read()` 方法会返回 **-1** ，从而实现在服务端关闭客户端的逻辑。在 Netty 的实现，在 `NioByteUnsafe#read()` 方法中，简化代码如下：

```
// <1>
// 读取数据
// 设置最后读取字节数
allocHandle.lastBytesRead(doReadBytes(byteBuf));
// 如果最后读取的字节为小于 0 ，说明对端已经关闭
close = allocHandle.lastBytesRead() < 0;

// 关闭客户端的连接
if (close) {
    closeOnRead(pipeline);
}
```

- `<1>` 处，读取客户端的 SocketChannel 返回 **-1** ，说明客户端已经关闭。

- `<2>` 处，调用 `#closeOnRead(ChannelPipeline pipeline)` 方法，关闭客户端的连接。代码如下：

  ```
   1: private void closeOnRead(ChannelPipeline pipeline) {
   2:     if (!isInputShutdown0()) {
   3:         // 开启连接半关闭
   4:         if (isAllowHalfClosure(config())) {
   5:             // 关闭 Channel 数据的读取
   6:             shutdownInput();
   7:             // 触发 ChannelInputShutdownEvent.INSTANCE 事件到 pipeline 中
   8:             pipeline.fireUserEventTriggered(ChannelInputShutdownEvent.INSTANCE);
   9:         } else {
  10:             close(voidPromise());
  11:         }
  12:     } else {
  13:         // 标记 inputClosedSeenErrorOnRead 为 true
  14:         inputClosedSeenErrorOnRead = true;
  15:         // 触发 ChannelInputShutdownEvent.INSTANCE 事件到 pipeline 中
  16:         pipeline.fireUserEventTriggered(ChannelInputShutdownReadComplete.INSTANCE);
  17:     }
  18: }
  ```

  - 第 2 行：调用 `NioSocketChannel#isInputShutdown0()` 方法，判断是否关闭 Channel 数据的读取。代码如下：

    ```
    // NioSocketChannel.java
    @Override
    protected boolean isInputShutdown0() {
        return isInputShutdown();
    }
    
    @Override
    public boolean isInputShutdown() {
        return javaChannel().socket().isInputShutdown() || !isActive();
    }
    
    // java.net.Socket.java
    private boolean shutIn = false;
    /**
     * Returns whether the read-half of the socket connection is closed.
     *
     * @return true if the input of the socket has been shutdown
     * @since 1.4
     * @see #shutdownInput
     */
    public boolean isInputShutdown() {
        return shutIn;
    }
    ```

    - 😈 注意看下英文注释。

  - `<1>` 第 4 行：调用 `AbstractNioByteChannel#isAllowHalfClosure()` 方法，判断是否开启连接**半关闭**的功能。代码如下：

    ```
    // AbstractNioByteChannel.java
    private static boolean isAllowHalfClosure(ChannelConfig config) {
        return config instanceof SocketChannelConfig &&
                ((SocketChannelConfig) config).isAllowHalfClosure();
    }
    ```

    - 可通过

       

      ```
      ALLOW_HALF_CLOSURE
      ```

       

      配置项开启。

      - Netty 参数，一个连接的远端关闭时本地端是否关闭，默认值为 `false` 。
      - 值为 `false`时，连接自动关闭。
      - 值为 `true` 时，触发 ChannelInboundHandler 的`#userEventTriggered()` 方法，事件 ChannelInputShutdownEvent 。

    - `<1.1>` 第 6 行：调用 `NioSocketChannel#shutdownInput()` 方法，关闭 Channel 数据的读取。代码如下：

      ```
      @Override
      public ChannelFuture shutdownInput() {
          return shutdownInput(newPromise());
      }
      
      @Override
      public ChannelFuture shutdownInput(final ChannelPromise promise) {
          EventLoop loop = eventLoop();
          if (loop.inEventLoop()) {
              shutdownInput0(promise);
          } else {
              loop.execute(new Runnable() {
                  @Override
                  public void run() {
                      shutdownInput0(promise);
                  }
              });
          }
          return promise;
      }
      
      private void shutdownInput0(final ChannelPromise promise) {
          try {
              // 关闭 Channel 数据的读取
              shutdownInput0();
              // 通知 Promise 成功
              promise.setSuccess();
          } catch (Throwable t) {
              // 通知 Promise 失败
              promise.setFailure(t);
          }
      }
      
      private void shutdownInput0() throws Exception {
          // 调用 Java NIO Channel 的 shutdownInput 方法
          if (PlatformDependent.javaVersion() >= 7) {
              javaChannel().shutdownInput();
          } else {
              javaChannel().socket().shutdownInput();
          }
      }
      ```

      - 核心是，调用 Java NIO Channel 的 shutdownInput 方法。

    - `<1.1>` 第 8 行：调用 `ChannelPipeline#fireUserEventTriggered(Object event)` 方法，触发 `ChannelInputShutdownEvent.INSTANCE` 事件到 pipeline 中。关于这个事件，胖友可以看看 [《netty 处理远程主机强制关闭一个连接》](https://my.oschina.net/chenleijava/blog/484667) 。

    - `<1.2>` 第 9 至 11 行：调用 `#close(Promise)` 方法，关闭客户端的 Channel 。后续的，就是 [「2. NioSocketChannel」](http://svip.iocoder.cn/Netty/Channel-7-close/#) 中。

- 第 12 至 17 行：

  - 第 14 行：标记 `inputClosedSeenErrorOnRead` 为 `true` 。原因如下：

    ```
    /**
     * 通道关闭读取，又错误读取的错误的标识
     *
     * 详细见 https://github.com/netty/netty/commit/ed0668384b393c3502c2136e3cc412a5c8c9056e 提交
     */
    private boolean inputClosedSeenErrorOnRead;
    ```

    - 如下是提交的说明：

      ```
      AbstractNioByteChannel will detect that the remote end of the socket has
      been closed and propagate a user event through the pipeline. However if
      the user has auto read on, or calls read again, we may propagate the
      same user events again. If the underlying transport continuously
      notifies us that there is read activity this will happen in a spin loop
      which consumes unnecessary CPU.
      ```

      - 胖友认真看下英文注释。结合 [《NIO read spin event loop spin when half closed #7801》](https://github.com/netty/netty/pull/7801) 提供的示例。

      - 在标记 `inputClosedSeenErrorOnRead = true` 后，在 `NioByteUnsafe#read()` 方法中，会主动对 `SelectionKey.OP_READ` 的感兴趣，避免空轮询。代码如下：

        ```
        // AbstractNioByteUnsafe.java
        public final void read() {
            final ChannelConfig config = config();
            // 若 inputClosedSeenErrorOnRead = true ，移除对 SelectionKey.OP_READ 事件的感兴趣。
            if (shouldBreakReadReady(config)) {
                clearReadPending(); // 移除对 SelectionKey.OP_READ 事件的感兴趣
                return;
            }
            
            // ... 省略其他代码。
        }
        
        // AbstractNioByteChannel.java
        final boolean shouldBreakReadReady(ChannelConfig config) {
            return isInputShutdown0() && (inputClosedSeenErrorOnRead || !isAllowHalfClosure(config));
        }
        ```

        - x

  - 第 16 行：调用 `ChannelPipeline#fireUserEventTriggered(Object event)` 方法，触发 `ChannelInputShutdownEvent.INSTANCE` 事件到 pipeline 中。

## 666. 彩蛋

比想象中简单的文章。但是，卡了比较久的时间。主要是针对 [《High CPU usage with SO_LINGER and sudden connection close (4.0.26.Final+) #4449》](https://github.com/netty/netty/issues/4449) 的讨论，中间请教了基友闪电侠和表弟普架。

痛并快乐的过程。如果英文好一点，相信解决的过程，可能更加愉快一些把。

# 精尽 Netty 源码解析 —— Channel（八）之 disconnect 操作

2018-07-22

## 1. 概述

本文分享 Netty NIO Channel **客户端**断开连接( **disconnect** )操作的过程。

在看 Netty NIO Channel 对 `#disconnect(ChannelPromise promise)` 方法的实现代码之前，我们先来看看 Java **原生** NIO SocketChannel 的 `#disconnect()` 方法。

- 结果，结果，结果，翻了半天，只看到 NIO SocketChannel 的父类 AbstractInterruptibleChannel 中，有 `#close()` 方法，而找不到 `#disconnect()` 方法。这个是啥情况？
- 我们又去翻了 Java **原生** UDP DatagramSocket 类，结果找到了 `#connect()` 方法。这个又是啥情况？

不卖关子了，直接说结论啦：

- Java **原生** NIO SocketChannel **不存在**，当调用 Netty `NioSocketChannel#disconnect(ChannelPromise promise)` 时，会自动转换成 **close** 操作，即 [《精尽 Netty 源码解析 —— Channel（七）之 close 操作》](http://svip.iocoder.cn/Netty/Channel-7-close/) 。
- 实际上， `Channel#disconnect(ChannelPromise promise)` 方法，是 Netty 为 UDP 设计的。

## 2. NioSocketChannel

通过 `NioSocketChannel#disconnect()` 方法，应用程序里可以主动关闭 NioSocketChannel 通道。代码如下：

```
@Override
public ChannelFuture disconnect() {
    return pipeline.disconnect();
}
```

- NioSocketChannel 继承 AbstractChannel 抽象类，所以 `#disconnect()` 方法实际是 AbstractChannel 实现的。
- 在方法内部，会调用对应的 `ChannelPipeline#disconnect()` 方法，将 disconnect 事件在 pipeline 上传播。

## 3. DefaultChannelPipeline

`DefaultChannelPipeline#disconnect()` 方法，代码如下：

```
@Override
public final ChannelPipeline disconnect() {
    tail.disconnect();
    return this;
}
```

- 在方法内部，会调用 `TailContext#disconnect()` 方法，将 flush 事件在 pipeline 中，从尾节点向头节点传播。详细解析，见 [「4. TailContext」](http://svip.iocoder.cn/Netty/Channel-8-disconnect/#) 。

## 4. TailContext

TailContext 对 `#flush()` 方法的实现，是从 AbstractChannelHandlerContext 抽象类继承，代码如下：

```
@Override
public ChannelFuture disconnect() {
    return disconnect(newPromise());
}

@Override
public ChannelFuture disconnect(final ChannelPromise promise) {
    // 判断是否为合法的 Promise 对象
    if (isNotValidPromise(promise, false)) {
        // cancelled
        return promise;
    }

    final AbstractChannelHandlerContext next = findContextOutbound();
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        // <1> 如果没有 disconnect 操作，则执行 close 事件在 pipeline 上
        // Translate disconnect to close if the channel has no notion of disconnect-reconnect.
        // So far, UDP/IP is the only transport that has such behavior.
        if (!channel().metadata().hasDisconnect()) {
            next.invokeClose(promise);
        // 如果有 disconnect 操作，则执行 disconnect 事件在 pipeline 上
        } else {
            next.invokeDisconnect(promise);
        }
    } else {
        safeExecute(executor, new Runnable() {
            @Override
            public void run() {
                // <1> 如果没有 disconnect 操作，则执行 close 事件在 pipeline 上
                if (!channel().metadata().hasDisconnect()) {
                    next.invokeClose(promise);
                    // 如果有 disconnect 操作，则执行 disconnect 事件在 pipeline 上
                } else {
                    next.invokeDisconnect(promise);
                }
            }
        }, promise, null);
    }
    return promise;
}
```

- 在

   

  ```
  <1>
  ```

   

  处，调用

   

  ```
  ChannelMetadata#hasDisconnect()
  ```

   

  方法，判断 Channel

   

  是否支持

   

  disconnect 操作。

  - 如果支持，则**转换**执行 close 事件在 pipeline 上。后续的逻辑，就是 [《精尽 Netty 源码解析 —— Channel（七）之 close 操作》](http://svip.iocoder.cn/Netty/Channel-7-close/) 。
  - 如果不支持，则**保持**执行 disconnect 事件在 pipeline 上。

- 支持 disconnect 操作的 Netty Channel 实现类有：



  - 和文头，我们提到的，只有 Java **原生** UDP DatagramSocket 支持是一致的。从 `So far, UDP/IP is the only transport that has such behavior.` 的英文注释，也能证实这一点。

- 不支持 disconnect 操作的 Netty Channel 实现类有：



  - 和文头，我们提到的，只有 Java **原生** NIO SocketChannel 不支持是一致的。

因为本系列，暂时不分享 UDP 相关的内容，所以对“执行 disconnect 事件在 pipeline 上”就不解析了。

## 666. 彩蛋

水更一篇，本来以为 Netty NIO Channel 的 disconnect 操作是个**骚**操作。