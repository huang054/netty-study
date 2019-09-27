







#                                   netty源码感悟

## 1.首先看个example，netty里面的demo

```java
public final class EchoServer {

    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));

    public static void main(String[] args) throws Exception {
        // Configure SSL.
        final SslContext sslCtx;
        if (SSL) {
            SelfSignedCertificate ssc = new SelfSignedCertificate();
            sslCtx = SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey()).build();
        } else {
            sslCtx = null;
        }

        // Configure the server.
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        final EchoServerHandler serverHandler = new EchoServerHandler();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(serverHandler);
                 }
             });

            // Start the server.
            ChannelFuture f = b.bind(PORT).sync();

            // Wait until the server socket is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```



我们可以看看

 EventLoopGroup workerGroup = new NioEventLoopGroup();

这行代码debug进去

```java
public NioEventLoopGroup() {
    this(0);
}
```

无参构造器，一路this下去会发现

```java
private static final int DEFAULT_EVENT_LOOP_THREADS;

static {
    DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
            "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

    if (logger.isDebugEnabled()) {
        logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
    }
}

/**
 * @see MultithreadEventExecutorGroup#MultithreadEventExecutorGroup(int, Executor, Object...)
 */
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
```

看到这里就会发现是构造线程池，而且不指定的时候是用默认的DEFAULT_EVENT_LOOP_THREADS，这个int值在静态代码块里赋值是你系统cpu的二倍

继续super和this下去就会发现

```java
private final EventExecutor[] children;
/**
 * Create a new instance.
 *
 * @param nThreads          the number of threads that will be used by this instance.
 * @param executor          the Executor to use, or {@code null} if the default should be used.
 * @param chooserFactory    the {@link EventExecutorChooserFactory} to use.
 * @param args              arguments which will passed to each {@link #newChild(Executor, Object...)} call
 */
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                        EventExecutorChooserFactory chooserFactory, Object... args) {
    if (nThreads <= 0) {
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
    }

    if (executor == null) {
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }

    children = new EventExecutor[nThreads];

    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
            // TODO: Think about if this is a good exception type
            throw new IllegalStateException("failed to create a child event loop", e);
        } finally {
            if (!success) {
                for (int j = 0; j < i; j ++) {
                    children[j].shutdownGracefully();
                }

                for (int j = 0; j < i; j ++) {
                    EventExecutor e = children[j];
                    try {
                        while (!e.isTerminated()) {
                            e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                        }
                    } catch (InterruptedException interrupted) {
                        // Let the caller handle the interruption.
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }
    }

    chooser = chooserFactory.newChooser(children);

    final FutureListener<Object> terminationListener = new FutureListener<Object>() {
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            if (terminatedChildren.incrementAndGet() == children.length) {
                terminationFuture.setSuccess(null);
            }
        }
    };

    for (EventExecutor e: children) {
        e.terminationFuture().addListener(terminationListener);
    }

    Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
    Collections.addAll(childrenSet, children);
    readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```

ecentloopgroup的初始化children = new EventExecutor[nThreads];

这里应该是对这些线程初始化  看看  children[i] = newChild(executor, args); 这个方法

```java
@Override
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
        ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
}
 NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
                 SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
                 EventLoopTaskQueueFactory queueFactory) {
        super(parent, executor, false, newTaskQueue(queueFactory), newTaskQueue(queueFactory),
                rejectedExecutionHandler);
        if (selectorProvider == null) {
            throw new NullPointerException("selectorProvider");
        }
        if (strategy == null) {
            throw new NullPointerException("selectStrategy");
        }
        provider = selectorProvider;
        final SelectorTuple selectorTuple = openSelector();
        selector = selectorTuple.selector;
        unwrappedSelector = selectorTuple.unwrappedSelector;
        selectStrategy = strategy;
    }
```

​     final SelectorTuple selectorTuple = openSelector();由于我是在window平台，最后调用nio的WindowsSelectorImpl

```java
WindowsSelectorImpl(SelectorProvider var1) throws IOException {
    super(var1);
    this.wakeupSourceFd = ((SelChImpl)this.wakeupPipe.source()).getFDVal();
    SinkChannelImpl var2 = (SinkChannelImpl)this.wakeupPipe.sink();
    var2.sc.socket().setTcpNoDelay(true);
    this.wakeupSinkFd = var2.getFDVal();
    this.pollWrapper.addWakeupSocket(this.wakeupSourceFd, 0);
}
```

然后会选择一个eventgroup   ->chooser = chooserFactory.newChooser(children);

```java
@SuppressWarnings("unchecked")
@Override
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) {
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}
```

如果是二的n次方用位运算，如果不是就是%取余数，

对性能要求极致

初始化每个nioeventloop，下面看看 b.group(bossGroup, workerGroup)方法

```java
/**
 * Set the {@link EventLoopGroup} for the parent (acceptor) and the child (client). These
 * {@link EventLoopGroup}'s are used to handle all the events and IO for {@link ServerChannel} and
 * {@link Channel}'s.
 */
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

 super.group(parentGroup); 这个方法跟踪到AbstractBootstrap

```java
    volatile EventLoopGroup group;
/**
 * The {@link EventLoopGroup} which is used to handle all the events for the to-be-created
 * {@link Channel}
 */
public B group(EventLoopGroup group) {
    if (group == null) {
        throw new NullPointerException("group");
    }
    if (this.group != null) {
        throw new IllegalStateException("group set already");
    }
    this.group = group;
    return self();
}
```

这个group就是bossgroup，而workgroup在ServerBootstrap中

下面看看server里面的.channel(NioServerSocketChannel.class) 

```java
/**
 * The {@link Class} which is used to create {@link Channel} instances from.
 * You either use this or {@link #channelFactory(io.netty.channel.ChannelFactory)} if your
 * {@link Channel} implementation has no no-args constructor.
 */
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}
 public ReflectiveChannelFactory(Class<? extends T> clazz) {
        ObjectUtil.checkNotNull(clazz, "clazz");
        try {
            this.constructor = clazz.getConstructor();
        } catch (NoSuchMethodException e) {
            throw new IllegalArgumentException("Class " + StringUtil.simpleClassName(clazz) +
                    " does not have a public non-arg constructor", e);
        }
    }
  @Deprecated
    public B channelFactory(ChannelFactory<? extends C> channelFactory) {
        if (channelFactory == null) {
            throw new NullPointerException("channelFactory");
        }
        if (this.channelFactory != null) {
            throw new IllegalStateException("channelFactory set already");
        }

        this.channelFactory = channelFactory;
        return self();
    }
```

就是通过反射进行NioServerSocketChannel的初始化并且赋值到AbstractBootstrap中，由于通过NioServerSocketChannel的无参构造器初始化

```java
public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
 private static ServerSocketChannel newSocket(SelectorProvider provider) {
        try {
            /**
             *  Use the {@link SelectorProvider} to open {@link SocketChannel} and so remove condition in
             *  {@link SelectorProvider#provider()} which is called by each ServerSocketChannel.open() otherwise.
             *
             *  See <a href="https://github.com/netty/netty/issues/2308">#2308</a>.
             */
            return provider.openServerSocketChannel();
        } catch (IOException e) {
            throw new ChannelException(
                    "Failed to open a server socket.", e);
        }
    }
//最后调用java的nio的api   openServerSocketChannel
```

```java
.option(ChannelOption.SO_BACKLOG, 100)
.handler(new LoggingHandler(LogLevel.INFO))
```

ChannelOption为实现socket协议当请求处理不过来的时候存放队列长度，而loggingHandler为日志

```java
.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline p = ch.pipeline();
        if (sslCtx != null) {
            p.addLast(sslCtx.newHandler(ch.alloc()));
        }
        //p.addLast(new LoggingHandler(LogLevel.INFO));
        p.addLast(serverHandler);
    }
});
```

这里面首先看childHandler方法

```java
    private volatile ChannelHandler childHandler;
/**
 * Set the {@link ChannelHandler} which is used to serve the request for the {@link Channel}'s.
 */
public ServerBootstrap childHandler(ChannelHandler childHandler) {
    if (childHandler == null) {
        throw new NullPointerException("childHandler");
    }
    this.childHandler = childHandler;
    return this;
}
```

也是将childHandler 赋值给ServerBootstrap中的childHandler ，然后看看initChannel

```java
@SuppressWarnings("unchecked")
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.add(ctx)) { // Guard against re-entrance.
        try {
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            // Explicitly call exceptionCaught(...) as we removed the handler before calling initChannel(...).
            // We do so to prevent multiple calls to initChannel(...).
            exceptionCaught(ctx, cause);
        } finally {
            ChannelPipeline pipeline = ctx.pipeline();
            if (pipeline.context(this) != null) {
                pipeline.remove(this);
            }
        }
        return true;
    }
    return false;
}
```

```java
private final Set<ChannelHandlerContext> initMap = Collections.newSetFromMap(
        new ConcurrentHashMap<ChannelHandlerContext, Boolean>());
```

add成功进行initChannel方法。ChannelHandlerContext就是一个pipeline里的上下文，从里面取出channel进行初始化  pipeline.context(this) 中，然后删除自己 pipeline.remove(this);只把handler业务放进pipeline

```java
@Override
public final ChannelHandlerContext context(ChannelHandler handler) {
    if (handler == null) {
        throw new NullPointerException("handler");
    }

    AbstractChannelHandlerContext ctx = head.next;
    for (;;) {

        if (ctx == null) {
            return null;
        }

        if (ctx.handler() == handler) {
            return ctx;
        }

        ctx = ctx.next;
    }
}
```

pipeline是一个链表，首先取到头节点的下一个节点，如果下个节点的handler等于要添加的 直接返回，否则加在下一个节点上

```java
if (sslCtx != null) {
    p.addLast(sslCtx.newHandler(ch.alloc()));
}
//p.addLast(new LoggingHandler(LogLevel.INFO));
p.addLast(serverHandler);
```

可以理解为业务handler依次加到pipeline这个链表上面

```java
new ChannelInitializer这个类初始化就把抽象initchannel 方法给我们自己实现
```

```java
@Sharable
public abstract class ChannelInitializer<C extends Channel> extends ChannelInboundHandlerAdapter {

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(ChannelInitializer.class);
    // We use a Set as a ChannelInitializer is usually shared between all Channels in a Bootstrap /
    // ServerBootstrap. This way we can reduce the memory usage compared to use Attributes.
    private final Set<ChannelHandlerContext> initMap = Collections.newSetFromMap(
            new ConcurrentHashMap<ChannelHandlerContext, Boolean>());

    /**
     * This method will be called once the {@link Channel} was registered. After the method returns this instance
     * will be removed from the {@link ChannelPipeline} of the {@link Channel}.
     *
     * @param ch            the {@link Channel} which was registered.
     * @throws Exception    is thrown if an error occurs. In that case it will be handled by
     *                      {@link #exceptionCaught(ChannelHandlerContext, Throwable)} which will by default close
     *                      the {@link Channel}.
     */
    protected abstract void initChannel(C ch) throws Exception;

    @Override
    @SuppressWarnings("unchecked")
    public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        // Normally this method will never be called as handlerAdded(...) should call initChannel(...) and remove
        // the handler.
        if (initChannel(ctx)) {
            // we called initChannel(...) so we need to call now pipeline.fireChannelRegistered() to ensure we not
            // miss an event.
            ctx.pipeline().fireChannelRegistered();

            // We are done with init the Channel, removing all the state for the Channel now.
            removeState(ctx);
        } else {
            // Called initChannel(...) before which is the expected behavior, so just forward the event.
            ctx.fireChannelRegistered();
        }
    }
```

上面都是一些初始化工作，下面就是netty服务启动的核心

## netty的启动

```java
// Start the server.
ChannelFuture f = b.bind(PORT).sync();
```

bind方法调用这个方法

```java
/**
 * Create a new {@link Channel} and bind it.
 */
public ChannelFuture bind(SocketAddress localAddress) {
    validate();
    if (localAddress == null) {
        throw new NullPointerException("localAddress");
    }
    return doBind(localAddress);
}
```

其中validate()是验证AbstractBootstrap中EventLoopGroup和ChannelFactory是否为null。

关键方法是调用了doBind方法

```java
  private ChannelFuture doBind(final SocketAddress localAddress) {
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();

                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }

```

我们下面来看看initAndRegister方法

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            // channel can be null if newChannel crashed (eg SocketException("too many open files"))
            channel.unsafe().closeForcibly();
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }

    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }

    // If we are here and the promise is not failed, it's one of the following cases:
    // 1) If we attempted registration from the event loop, the registration has been completed at this point.
    //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
    // 2) If we attempted registration from the other thread, the registration request has been successfully
    //    added to the event loop's task queue for later execution.
    //    i.e. It's safe to attempt bind() or connect() now:
    //         because bind() or connect() will be executed *after* the scheduled registration task is executed
    //         because register(), bind(), and connect() are all bound to the same thread.

    return regFuture;
}
```

  channel = channelFactory.newChannel();通过反射实例化一个channel

然后init方法

```java
@Override
void init(Channel channel) {
    setChannelOptions(channel, options0().entrySet().toArray(newOptionArray(0)), logger);
    setAttributes(channel, attrs0().entrySet().toArray(newAttrArray(0)));

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions =
            childOptions.entrySet().toArray(newOptionArray(0));
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

```java
ChannelPipeline p = channel.pipeline();
```

从channel里面获取到pipeline

```java
if (handler != null) {
    pipeline.addLast(handler);
}
```

handler不为null时候添加到pipeline这个链表的最后

```java
ch.eventLoop().execute(new Runnable() {
    @Override
    public void run() {
        pipeline.addLast(new ServerBootstrapAcceptor(
                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
    }
});
```

最后从channel的eventloop中执行线程任务用ServerBootstrapAcceptor对通道，workergroup 这些进行包装，下面我们看看addlast方法

```java
private void addLast0(AbstractChannelHandlerContext newCtx) {
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
}
```

就是个链表通过prev 和next的指向添加元素，下面接着initAndRegister逻辑往下走

```java
ChannelFuture regFuture = config().group().register(channel);
```

config返回bootstrap 。group从bootstrap 中获取到EventLoopGroup，然后看看register方法

```java
@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
 @Override
    public ChannelFuture register(Channel channel) {
        return next().register(channel);
    }
 @Override
    public EventLoop next() {
        return (EventLoop) super.next();
    }
```

通过一个Promise把channel注册到EventLoop，eventloop从EventLoopGroup里按照算法取一条

和上面的算法基本一致

```java
@Override
public EventExecutor next() {
    return chooser.next();
}
```

```java
  @Override
    public ChannelFuture register(Channel channel) {
        return register(new DefaultChannelPromise(channel, this));
    }
@Override
public ChannelFuture register(ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
   @Override
        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            if (eventLoop == null) {
                throw new NullPointerException("eventLoop");
            }
            if (isRegistered()) {
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
                return;
            }
            if (!isCompatible(eventLoop)) {
                promise.setFailure(
                        new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                return;
            }

            AbstractChannel.this.eventLoop = eventLoop;

            if (eventLoop.inEventLoop()) {
                register0(promise);
            } else {
                try {
                    eventLoop.execute(new Runnable() {
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    closeFuture.setClosed();
                    safeSetFailure(promise, t);
                }
            }
        }
 @Override
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
        addTask(task);
        if (!inEventLoop) {
            startThread();
            if (isShutdown()) {
                boolean reject = false;
                try {
                    if (removeTask(task)) {
                        reject = true;
                    }
                } catch (UnsupportedOperationException e) {
                    // The task queue does not support removal so the best thing we can do is to just move on and
                    // hope we will be able to pick-up the task before its completely terminated.
                    // In worst case we will log on termination.
                }
                if (reject) {
                    reject();
                }
            }
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
```

doBind0最后在触发channelRegistered（）之前调用此方法。 给用户处理程序一个设置的机会

```java
@Override
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }

    boolean inEventLoop = inEventLoop();
    addTask(task);
    if (!inEventLoop) {
        startThread();
        if (isShutdown()) {
            boolean reject = false;
            try {
                if (removeTask(task)) {
                    reject = true;
                }
            } catch (UnsupportedOperationException e) {
                // The task queue does not support removal so the best thing we can do is to just move on and
                // hope we will be able to pick-up the task before its completely terminated.
                // In worst case we will log on termination.
            }
            if (reject) {
                reject();
            }
        }
    }

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}
 private void startThread() {
        if (state == ST_NOT_STARTED) {
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                boolean success = false;
                try {
                    doStartThread();
                    success = true;
                } finally {
                    if (!success) {
                        STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                    }
                }
            }
        }
    }
 private void doStartThread() {
        assert thread == null;
        executor.execute(new Runnable() {
            @Override
            public void run() {
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }

                boolean success = false;
                updateLastExecutionTime();
                try {
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                    for (;;) {
                        int oldState = state;
                        if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                                SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                            break;
                        }
                    }

                    // Check if confirmShutdown() was called at the end of the loop.
                    if (success && gracefulShutdownStartTime == 0) {
                        if (logger.isErrorEnabled()) {
                            logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                    SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
                                    "be called before run() implementation terminates.");
                        }
                    }

                    try {
                        // Run all remaining tasks and shutdown hooks.
                        for (;;) {
                            if (confirmShutdown()) {
                                break;
                            }
                        }
                    } finally {
                        try {
                            cleanup();
                        } finally {
                            // Lets remove all FastThreadLocals for the Thread as we are about to terminate and notify
                            // the future. The user may block on the future and once it unblocks the JVM may terminate
                            // and start unloading classes.
                            // See https://github.com/netty/netty/issues/6596.
                            FastThreadLocal.removeAll();

                            STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                            threadLock.countDown();
                            if (logger.isWarnEnabled() && !taskQueue.isEmpty()) {
                                logger.warn("An event executor terminated with " +
                                        "non-empty task queue (" + taskQueue.size() + ')');
                            }
                            terminationFuture.setSuccess(null);
                        }
                    }
                }
            }
        });
    }
```

注意SingleThreadEventExecutor.this.run();这个并不是调用的thread的run方法，而是调用自己的run方法

```java
@Override
protected void run() {
    for (;;) {
        try {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;

                case SelectStrategy.BUSY_WAIT:
                    // fall-through to SELECT since the busy-wait is not supported with NIO

                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false));

                    // 'wakenUp.compareAndSet(false, true)' is always evaluated
                    // before calling 'selector.wakeup()' to reduce the wake-up
                    // overhead. (Selector.wakeup() is an expensive operation.)
                    //
                    // However, there is a race condition in this approach.
                    // The race condition is triggered when 'wakenUp' is set to
                    // true too early.
                    //
                    // 'wakenUp' is set to true too early if:
                    // 1) Selector is waken up between 'wakenUp.set(false)' and
                    //    'selector.select(...)'. (BAD)
                    // 2) Selector is waken up between 'selector.select(...)' and
                    //    'if (wakenUp.get()) { ... }'. (OK)
                    //
                    // In the first case, 'wakenUp' is set to true and the
                    // following 'selector.select(...)' will wake up immediately.
                    // Until 'wakenUp' is set to false again in the next round,
                    // 'wakenUp.compareAndSet(false, true)' will fail, and therefore
                    // any attempt to wake up the Selector will fail, too, causing
                    // the following 'selector.select(...)' call to block
                    // unnecessarily.
                    //
                    // To fix this problem, we wake up the selector again if wakenUp
                    // is true immediately after selector.select(...).
                    // It is inefficient in that it wakes up the selector for both
                    // the first case (BAD - wake-up required) and the second case
                    // (OK - no wake-up required).

                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                    // fall through
                default:
                }
            } catch (IOException e) {
                // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                // the selector and retry. https://github.com/netty/netty/issues/8566
                rebuildSelector0();
                handleLoopException(e);
                continue;
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    runAllTasks();
                }
            } else {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // Always handle shutdown even if the loop processing threw an exception.
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
 private void processSelectedKeys() {
        if (selectedKeys != null) {
            processSelectedKeysOptimized();
        } else {
            processSelectedKeysPlain(selector.selectedKeys());
        }
    }
```

随便进入一个看看，这个if else判断只不过是分条件执行不同逻辑

```java
private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];
        // null out entry in the array to allow to have it GC'ed once the Channel close
        // See https://github.com/netty/netty/issues/2363
        selectedKeys.keys[i] = null;

        final Object a = k.attachment();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (needsToSelectAgain) {
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.reset(i + 1);

            selectAgain();
            i = -1;
        }
    }
}

```

这里就是个死循环一直监听有没新的链接事件

最后调用channel的unsafe方法来注册eventloop和promise，promise就是继承furture用来查看register完



```java

```

如果ineventloop表示有绑定，直接next.invokeChannelReadComplete()

如果没有就  从child挑选个从新执行，这个会执行nioeventloop中的run方法 executor.execute(tasks.invokeChannelReadCompleteTask);

到底是成功失败取消等等状态的，就是一种保证

如果我们在这里，但承诺没有失败，则是以下情况之一：
          如果我们尝试从事件循环中注册，则注册已完成。
         即立即尝试bind（）或connect（）是安全的，因为该通道已被注册。
          如果我们尝试从另一个线程进行注册，则注册请求已成功
         添加到事件循环的任务队列中，以供以后执行。
         即立即尝试bind（）或connect（）是安全的：
         因为bind（）或connect（）将在计划的注册任务执行后*执行
         因为register（），bind（）和connect（）都绑定到同一线程。

bind后面的sync

```java
@Override
public Promise<V> await() throws InterruptedException {
    if (isDone()) {
        return this;
    }

    if (Thread.interrupted()) {
        throw new InterruptedException(toString());
    }

    checkDeadLock();

    synchronized (this) {
        while (!isDone()) {
            incWaiters();
            try {
                wait();
            } finally {
                decWaiters();
            }
        }
    }
    return this;
}
```

就是调用thread的wait一直阻塞着

```java
} finally {
    // Shut down all event loops to terminate all threads.
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```

利用java的finally 最后程序终止的时候关闭所有事件循环以终止所有线程。

当接受到一个新链接的时候就会进入到上面死循环中的processSelectedKey方法

```java
  private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        if (!k.isValid()) {
            final EventLoop eventLoop;
            try {
                eventLoop = ch.eventLoop();
            } catch (Throwable ignored) {
                // If the channel implementation throws an exception because there is no event loop, we ignore this
                // because we are only trying to determine if ch is registered to this event loop and thus has authority
                // to close ch.
                return;
            }
            // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
            // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
            // still healthy and should not be closed.
            // See https://github.com/netty/netty/issues/5125
            if (eventLoop != this || eventLoop == null) {
                return;
            }
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
            return;
        }

        try {
            int readyOps = k.readyOps();
            // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
            // the NIO JDK channel implementation may throw a NotYetConnectedException.
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
                // See https://github.com/netty/netty/issues/924
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);

                unsafe.finishConnect();
            }

            // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
                ch.unsafe().forceFlush();
            }

            // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
            // to a spin loop
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }
```

然后进入到     unsafe.read();

```java
 private final class NioMessageUnsafe extends AbstractNioUnsafe {

        private final List<Object> readBuf = new ArrayList<Object>();

        @Override
        public void read() {
            assert eventLoop().inEventLoop();
            final ChannelConfig config = config();
            final ChannelPipeline pipeline = pipeline();
            final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
            allocHandle.reset(config);

            boolean closed = false;
            Throwable exception = null;
            try {
                try {
                    do {
                        int localRead = doReadMessages(readBuf);
                        if (localRead == 0) {
                            break;
                        }
                        if (localRead < 0) {
                            closed = true;
                            break;
                        }

                        allocHandle.incMessagesRead(localRead);
                    } while (allocHandle.continueReading());
                } catch (Throwable t) {
                    exception = t;
                }

                int size = readBuf.size();
                for (int i = 0; i < size; i ++) {
                    readPending = false;
                    pipeline.fireChannelRead(readBuf.get(i));
                }
                readBuf.clear();
                allocHandle.readComplete();
                pipeline.fireChannelReadComplete();

                if (exception != null) {
                    closed = closeOnReadError(exception);

                    pipeline.fireExceptionCaught(exception);
                }

                if (closed) {
                    inputShutdown = true;
                    if (isOpen()) {
                        close(voidPromise());
                    }
                }
            } finally {
                // Check if there is a readPending which was not processed yet.
                // This could be for two reasons:
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
                //
                // See https://github.com/netty/netty/issues/2254
                if (!readPending && !config.isAutoRead()) {
                    removeReadOp();
                }
            }
        }
    }

```

这里就是jdk的nio方法调用 读字节流等等

给新来的channel绑定到一个workeventloop上，并且调用execute方法

```java
  @Override
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
        addTask(task);
        if (!inEventLoop) {
            startThread();
            if (isShutdown()) {
                boolean reject = false;
                try {
                    if (removeTask(task)) {
                        reject = true;
                    }
                } catch (UnsupportedOperationException e) {
                    // The task queue does not support removal so the best thing we can do is to just move on and
                    // hope we will be able to pick-up the task before its completely terminated.
                    // In worst case we will log on termination.
                }
                if (reject) {
                    reject();
                }
            }
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
```

到这里又回到了要不要启动一个新线程判断，和bind流程差不多只不过bind是bosseventloop  ，而真正处理数据的读写是workeventloop  

## pipeline

NioServerSocketChannel初始化时候会调用

```java
public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
  public NioServerSocketChannel(ServerSocketChannel channel) {
        super(null, channel, SelectionKey.OP_ACCEPT);
        config = new NioServerSocketChannelConfig(this, javaChannel().socket());
    }
```

这个有参构造器会调用父类的有参构造器，super到AbstractChannel

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
```

```java
TailContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, TAIL_NAME, TailContext.class);
    setAddComplete();
}
```

这里会初始化pipeline，下面我们来看看newChannelPipeline()的逻辑

```java
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
```

首先检查channel是否为null

实例化SucceededChannelFuture，把chaneel传进去对类的变量初始化

这里VoidChannelPromise 就是一个channel的promise保证通过future的回调保证没异常，最后把future

=null让gc回收

```java
tail = new TailContext(this);
head = new HeadContext(this);

head.next = tail;
tail.prev = head;
```

这里就是pipeline链表的初始化，头节点和尾节点的初始化，并且head。next为tail，tail的上个节点为head

```java
  b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 100)
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline p = ch.pipeline();
                            if (sslCtx != null) {
                                p.addLast(sslCtx.newHandler(ch.alloc()));
                            }
                            //p.addLast(new LoggingHandler(LogLevel.INFO));
                            p.addLast(workerGroup,serverHandler);
                        }
                    });
```

我们在server端添加逻辑就是p.addLast这个方法   我们来看看addLast是做了什么

```java
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
            addLast(executor, null, h);
        }

        return this;
    }
  @Override
    public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {
            checkMultiplicity(handler);

            newCtx = newContext(group, filterName(name, handler), handler);

            addLast0(newCtx);

            // If the registered is false it means that the channel was not registered on an eventLoop yet.
            // In this case we add the context to the pipeline and add a task that will call
            // ChannelHandler.handlerAdded(...) once the channel is registered.
            if (!registered) {
                newCtx.setAddPending();
                callHandlerCallbackLater(newCtx, true);
                return this;
            }

            EventExecutor executor = newCtx.executor();
            if (!executor.inEventLoop()) {
                callHandlerAddedInEventLoop(newCtx, executor);
                return this;
            }
        }
        callHandlerAdded0(newCtx);
        return this;
    }
  private void addLast0(AbstractChannelHandlerContext newCtx) {
        AbstractChannelHandlerContext prev = tail.prev;
        newCtx.prev = prev;
        newCtx.next = tail;
        prev.next = newCtx;
        tail.prev = newCtx;
    }

```

可以看到 addLast(EventExecutorGroup executor, ChannelHandler... handlers)方法我们可以指定EventExecutorGroup ，一般这里我们一般没有指定



```java
 newCtx = newContext(group, filterName(name, handler), handler);
addLast0(newCtx);
```

首先我们得创建一个节点，然后通过addLast0方法加在最后，很简单得通过链表操作把新节点插入尾节点前一个位置，这里简单地用`synchronized`方法是为了防止多线程并发操作pipeline底层的双向链表 

下面看看上面得 checkMultiplicity(handler);

```java
private static void checkMultiplicity(ChannelHandler handler) {
    if (handler instanceof ChannelHandlerAdapter) {
        ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
        if (!h.isSharable() && h.added) {
            throw new ChannelPipelineException(
                    h.getClass().getName() +
                    " is not a @Sharable handler, so can't be added or removed multiple times.");
        }
        h.added = true;
    }
}
```

可以看到handler里面有个added是否添加过，还有isSharable是共享得，有没有一种springioc得感觉，added

表示ioc容器有没这个bean ，shareable表示他的作用域，

```java
public boolean isSharable() {
    /**
     * Cache the result of {@link Sharable} annotation detection to workaround a condition. We use a
     * {@link ThreadLocal} and {@link WeakHashMap} to eliminate the volatile write/reads. Using different
     * {@link WeakHashMap} instances per {@link Thread} is good enough for us and the number of
     * {@link Thread}s are quite limited anyway.
     *
     * See <a href="https://github.com/netty/netty/issues/2289">#2289</a>.
     */
    Class<?> clazz = getClass();
    Map<Class<?>, Boolean> cache = InternalThreadLocalMap.get().handlerSharableCache();
    Boolean sharable = cache.get(clazz);
    if (sharable == null) {
        sharable = clazz.isAnnotationPresent(Sharable.class);
        cache.put(clazz, sharable);
    }
    return sharable;
}
```
原来这个方法是通过反射获取他有没@Sharable注解，来判断是否共享

### 创建节点

​    我们看看   newCtx = newContext(group, filterName(name, handler), handler);这个创建新节点

```java
private String filterName(String name, ChannelHandler handler) {
    if (name == null) {
        return generateName(handler);
    }
    checkDuplicateName(name);
    return name;
}
```

这里就是name为空时候，自动生成一个name，并且还要检车这个name有没重复

```java
private String generateName(ChannelHandler handler) {
    Map<Class<?>, String> cache = nameCaches.get();
    Class<?> handlerType = handler.getClass();
    String name = cache.get(handlerType);
    if (name == null) {
        name = generateName0(handlerType);
        cache.put(handlerType, name);
    }

    // It's not very likely for a user to put more than one handler of the same type, but make sure to avoid
    // any name conflicts.  Note that we don't cache the names generated here.
    if (context0(name) != null) {
        String baseName = name.substring(0, name.length() - 1); // Strip the trailing '0'.
        for (int i = 1;; i ++) {
            String newName = baseName + i;
            if (context0(newName) == null) {
                name = newName;
                break;
            }
        }
    }
    return name;
}
```

首先从threadlocal里get 一个map，key为handler得class，value为name，判断name为空就generateName0(handlerType);并且放回threadlocal

```java
private static String generateName0(Class<?> handlerType) {
    return StringUtil.simpleClassName(handlerType) + "#0";
}
```

就是通过handler得classname加上#0，生成name

```java
private AbstractChannelHandlerContext context0(String name) {
    AbstractChannelHandlerContext context = head.next;
    while (context != tail) {
        if (context.name().equals(name)) {
            return context;
        }
        context = context.next;
    }
    return null;
}
```

通过链表得遍历判断有没重名得name，有就返回重名得节点，没有返回null

```java
String baseName = name.substring(0, name.length() - 1); // Strip the trailing '0'.
for (int i = 1;; i ++) {
    String newName = baseName + i;
    if (context0(newName) == null) {
        name = newName;
        break;
    }
}
```

有重名就把最后得0改成其他数字，死循环判断，下面就是newCtx = newContext(group, filterName(name, handler), handler);逻辑，由于我们没有指定EventLoopGroup，所以这里是null

```java
final class DefaultChannelHandlerContext extends AbstractChannelHandlerContext {

    private final ChannelHandler handler;

    DefaultChannelHandlerContext(
            DefaultChannelPipeline pipeline, EventExecutor executor, String name, ChannelHandler handler) {
        super(pipeline, executor, name, handler.getClass());
        this.handler = handler;
    }

    @Override
    public ChannelHandler handler() {
        return handler;
    }
}
```

初始化DefaultChannelHandlerContext，handler传递过来，下面看看父类做了什么,mask0这个用更细维度的控制代替了以前的out和in的控制

```java
AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor,
                              String name, Class<? extends ChannelHandler> handlerClass) {
    this.name = ObjectUtil.checkNotNull(name, "name");
    this.pipeline = pipeline;
    this.executor = executor;
    this.executionMask = mask(handlerClass);
    // Its ordered if its driven by the EventLoop or the given Executor is an instanceof OrderedEventExecutor.
    ordered = executor == null || executor instanceof OrderedEventExecutor;
}
 static int mask(Class<? extends ChannelHandler> clazz) {
        // Try to obtain the mask from the cache first. If this fails calculate it and put it in the cache for fast
        // lookup in the future.
        Map<Class<? extends ChannelHandler>, Integer> cache = MASKS.get();
        Integer mask = cache.get(clazz);
        if (mask == null) {
            mask = mask0(clazz);
            cache.put(clazz, mask);
        }
        return mask;
    }
    /**
     * Calculate the {@code executionMask}.
     */
    private static int mask0(Class<? extends ChannelHandler> handlerType) {
        int mask = MASK_EXCEPTION_CAUGHT;
        try {
            if (ChannelInboundHandler.class.isAssignableFrom(handlerType)) {
                mask |= MASK_ALL_INBOUND;

                if (isSkippable(handlerType, "channelRegistered", ChannelHandlerContext.class)) {
                    mask &= ~MASK_CHANNEL_REGISTERED;
                }
                if (isSkippable(handlerType, "channelUnregistered", ChannelHandlerContext.class)) {
                    mask &= ~MASK_CHANNEL_UNREGISTERED;
                }
                if (isSkippable(handlerType, "channelActive", ChannelHandlerContext.class)) {
                    mask &= ~MASK_CHANNEL_ACTIVE;
                }
                if (isSkippable(handlerType, "channelInactive", ChannelHandlerContext.class)) {
                    mask &= ~MASK_CHANNEL_INACTIVE;
                }
                if (isSkippable(handlerType, "channelRead", ChannelHandlerContext.class, Object.class)) {
                    mask &= ~MASK_CHANNEL_READ;
                }
                if (isSkippable(handlerType, "channelReadComplete", ChannelHandlerContext.class)) {
                    mask &= ~MASK_CHANNEL_READ_COMPLETE;
                }
                if (isSkippable(handlerType, "channelWritabilityChanged", ChannelHandlerContext.class)) {
                    mask &= ~MASK_CHANNEL_WRITABILITY_CHANGED;
                }
                if (isSkippable(handlerType, "userEventTriggered", ChannelHandlerContext.class, Object.class)) {
                    mask &= ~MASK_USER_EVENT_TRIGGERED;
                }
            }

            if (ChannelOutboundHandler.class.isAssignableFrom(handlerType)) {
                mask |= MASK_ALL_OUTBOUND;

                if (isSkippable(handlerType, "bind", ChannelHandlerContext.class,
                        SocketAddress.class, ChannelPromise.class)) {
                    mask &= ~MASK_BIND;
                }
                if (isSkippable(handlerType, "connect", ChannelHandlerContext.class, SocketAddress.class,
                        SocketAddress.class, ChannelPromise.class)) {
                    mask &= ~MASK_CONNECT;
                }
                if (isSkippable(handlerType, "disconnect", ChannelHandlerContext.class, ChannelPromise.class)) {
                    mask &= ~MASK_DISCONNECT;
                }
                if (isSkippable(handlerType, "close", ChannelHandlerContext.class, ChannelPromise.class)) {
                    mask &= ~MASK_CLOSE;
                }
                if (isSkippable(handlerType, "deregister", ChannelHandlerContext.class, ChannelPromise.class)) {
                    mask &= ~MASK_DEREGISTER;
                }
                if (isSkippable(handlerType, "read", ChannelHandlerContext.class)) {
                    mask &= ~MASK_READ;
                }
                if (isSkippable(handlerType, "write", ChannelHandlerContext.class,
                        Object.class, ChannelPromise.class)) {
                    mask &= ~MASK_WRITE;
                }
                if (isSkippable(handlerType, "flush", ChannelHandlerContext.class)) {
                    mask &= ~MASK_FLUSH;
                }
            }

            if (isSkippable(handlerType, "exceptionCaught", ChannelHandlerContext.class, Throwable.class)) {
                mask &= ~MASK_EXCEPTION_CAUGHT;
            }
        } catch (Exception e) {
            // Should never reach here.
            PlatformDependent.throwException(e);
        }

        return mask;
    }
```

mask(handlerClass)首先从threadlocal里获取掩码，如果为null生成放回

ordered = executor == null || executor instanceof OrderedEventExecutor;由于我们传得null，所以这里为true，当然我们也可以传OrderedEventExecutor得子类

我们继续走addLast逻辑

```java
if (!registered) {
    newCtx.setAddPending();
    callHandlerCallbackLater(newCtx, true);
    return this;
}
```

如果为false没注册，在这种情况下，我们将上下文添加到eventLoop中，并添加一个将调用的任务，

然后执行callHandlerCallbackLater(newCtx, true)相当于回调得逻辑

```java
private void callHandlerCallbackLater(AbstractChannelHandlerContext ctx, boolean added) {
    assert !registered;

    PendingHandlerCallback task = added ? new PendingHandlerAddedTask(ctx) : new PendingHandlerRemovedTask(ctx);
    PendingHandlerCallback pending = pendingHandlerCallbackHead;
    if (pending == null) {
        pendingHandlerCallbackHead = task;
    } else {
        // Find the tail of the linked-list.
        while (pending.next != null) {
            pending = pending.next;
        }
        pending.next = task;
    }
}
```

这里就是new 一个add得task，added为true就是add得task，added为false就是remove得task，下面得逻辑就是降新的task添加到尾节点

如果注册了就会走下面逻辑

```java
  EventExecutor executor = newCtx.executor();
    if (!executor.inEventLoop()) {
        callHandlerAddedInEventLoop(newCtx, executor);
        return this;
    }
}
callHandlerAdded0(newCtx);
```

从pipeline得channel获取到EventExecutor，如果在EventLoop种则通过线程执行callHandlerAdded0，如果不在

则直接执行callHandlerAdded0

同理删除节点也可以参照增加节点只不过是链表得操作，删除回调

```java
private void callHandlerRemoved0(final AbstractChannelHandlerContext ctx) {
    // Notify the complete removal.
    try {
        ctx.callHandlerRemoved();
    } catch (Throwable t) {
        fireExceptionCaught(new ChannelPipelineException(
                ctx.handler().getClass().getName() + ".handlerRemoved() has thrown an exception.", t));
    }
}
```

这样也就进入到了我们得业务类里面

### 初识Unsafe

pipeline 中得io读写都是通过Unsafe来完成，下面我们看看Unsafe这个类

```java
interface Unsafe {

    /**
     * Return the assigned {@link RecvByteBufAllocator.Handle} which will be used to allocate {@link ByteBuf}'s when
     * receiving data.
     */
    RecvByteBufAllocator.Handle recvBufAllocHandle();

    /**
     * Return the {@link SocketAddress} to which is bound local or
     * {@code null} if none.
     */
    SocketAddress localAddress();

    /**
     * Return the {@link SocketAddress} to which is bound remote or
     * {@code null} if none is bound yet.
     */
    SocketAddress remoteAddress();

    /**
     * Register the {@link Channel} of the {@link ChannelPromise} and notify
     * the {@link ChannelFuture} once the registration was complete.
     */
    void register(EventLoop eventLoop, ChannelPromise promise);

    /**
     * Bind the {@link SocketAddress} to the {@link Channel} of the {@link ChannelPromise} and notify
     * it once its done.
     */
    void bind(SocketAddress localAddress, ChannelPromise promise);

    /**
     * Connect the {@link Channel} of the given {@link ChannelFuture} with the given remote {@link SocketAddress}.
     * If a specific local {@link SocketAddress} should be used it need to be given as argument. Otherwise just
     * pass {@code null} to it.
     *
     * The {@link ChannelPromise} will get notified once the connect operation was complete.
     */
    void connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);

    /**
     * Disconnect the {@link Channel} of the {@link ChannelFuture} and notify the {@link ChannelPromise} once the
     * operation was complete.
     */
    void disconnect(ChannelPromise promise);

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

    /**
     * Deregister the {@link Channel} of the {@link ChannelPromise} from {@link EventLoop} and notify the
     * {@link ChannelPromise} once the operation was complete.
     */
    void deregister(ChannelPromise promise);

    /**
     * Schedules a read operation that fills the inbound buffer of the first {@link ChannelInboundHandler} in the
     * {@link ChannelPipeline}.  If there's already a pending read operation, this method does nothing.
     */
    void beginRead();

    /**
     * Schedules a write operation.
     */
    void write(Object msg, ChannelPromise promise);

    /**
     * Flush out all write operations scheduled via {@link #write(Object, ChannelPromise)}.
     */
    void flush();

    /**
     * Return a special ChannelPromise which can be reused and passed to the operations in {@link Unsafe}.
     * It will never be notified of a success or error and so is only a placeholder for operations
     * that take a {@link ChannelPromise} as argument but for which you not want to get notified.
     */
    ChannelPromise voidPromise();

    /**
     * Returns the {@link ChannelOutboundBuffer} of the {@link Channel} where the pending write requests are stored.
     */
    ChannelOutboundBuffer outboundBuffer();
}
```

按功能可以分为分配内存，Socket四元组信息，注册事件循环，绑定网卡端口，Socket的连接和关闭，Socket的读写，看的出来，这些操作都是和jdk底层相关 

```java
public interface NioUnsafe extends Unsafe {
    /**
     * Return underlying {@link SelectableChannel}
     */
    SelectableChannel ch();

    /**
     * Finish connect
     */
    void finishConnect();

    /**
     * Read from underlying {@link SelectableChannel}
     */
    void read();

    void forceFlush();
}
```

nio得也增加了几个方法，在网上找了个类继承图

![img](https://github.com/huang054/netty-study/blob/master/nio.jpg) 

可以看到主要分成byte和message，关于链接字节得读取在byte里面，链接得操作在message里面

，基本都是调用jdk得nio操作

```java
  @Override
    public final void read() {
        final ChannelConfig config = config();
        if (shouldBreakReadReady(config)) {
            clearReadPending();
            return;
        }
        final ChannelPipeline pipeline = pipeline();
        final ByteBufAllocator allocator = config.getAllocator();
        final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
        allocHandle.reset(config);

        ByteBuf byteBuf = null;
        boolean close = false;
        try {
            do {
                byteBuf = allocHandle.allocate(allocator);
                allocHandle.lastBytesRead(doReadBytes(byteBuf));
                if (allocHandle.lastBytesRead() <= 0) {
                    // nothing was read. release the buffer.
                    byteBuf.release();
                    byteBuf = null;
                    close = allocHandle.lastBytesRead() < 0;
                    if (close) {
                        // There is nothing left to read as we received an EOF.
                        readPending = false;
                    }
                    break;
                }

                allocHandle.incMessagesRead(1);
                readPending = false;
                pipeline.fireChannelRead(byteBuf);
                byteBuf = null;
            } while (allocHandle.continueReading());

            allocHandle.readComplete();
            pipeline.fireChannelReadComplete();

            if (close) {
                closeOnRead(pipeline);
            }
        } catch (Throwable t) {
            handleReadException(pipeline, byteBuf, t, close, allocHandle);
        } finally {
            // Check if there is a readPending which was not processed yet.
            // This could be for two reasons:
            // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
            // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
            //
            // See https://github.com/netty/netty/issues/2254
            if (!readPending && !config.isAutoRead()) {
                removeReadOp();
            }
        }
    }
}
```

看看它的read方法，找到与pipeline相关得pipeline.fireChannelRead(byteBuf);

```java
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
    final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRead(m);
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRead(m);
            }
        });
    }
}
 private void invokeChannelRead(Object msg) {
        if (invokeHandler()) {
            try {
                ((ChannelInboundHandler) handler()).channelRead(this, msg);
            } catch (Throwable t) {
                notifyHandlerException(t);
            }
        } else {
            fireChannelRead(msg);
        }
    }
```

最后及时channelRead执行我们得业务方法

## NioEventLoop 

首先我们来看看eventloop中的run方法，上面我们知道启动的时候由线程执行了eventloop的run方法

```java
@Override
protected void run() {
    for (;;) {
        try {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;

                case SelectStrategy.BUSY_WAIT:
                    // fall-through to SELECT since the busy-wait is not supported with NIO

                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false));

                    // 'wakenUp.compareAndSet(false, true)' is always evaluated
                    // before calling 'selector.wakeup()' to reduce the wake-up
                    // overhead. (Selector.wakeup() is an expensive operation.)
                    //
                    // However, there is a race condition in this approach.
                    // The race condition is triggered when 'wakenUp' is set to
                    // true too early.
                    //
                    // 'wakenUp' is set to true too early if:
                    // 1) Selector is waken up between 'wakenUp.set(false)' and
                    //    'selector.select(...)'. (BAD)
                    // 2) Selector is waken up between 'selector.select(...)' and
                    //    'if (wakenUp.get()) { ... }'. (OK)
                    //
                    // In the first case, 'wakenUp' is set to true and the
                    // following 'selector.select(...)' will wake up immediately.
                    // Until 'wakenUp' is set to false again in the next round,
                    // 'wakenUp.compareAndSet(false, true)' will fail, and therefore
                    // any attempt to wake up the Selector will fail, too, causing
                    // the following 'selector.select(...)' call to block
                    // unnecessarily.
                    //
                    // To fix this problem, we wake up the selector again if wakenUp
                    // is true immediately after selector.select(...).
                    // It is inefficient in that it wakes up the selector for both
                    // the first case (BAD - wake-up required) and the second case
                    // (OK - no wake-up required).

                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                    // fall through
                default:
                }
            } catch (IOException e) {
                // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                // the selector and retry. https://github.com/netty/netty/issues/8566
                rebuildSelector0();
                handleLoopException(e);
                continue;
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    runAllTasks();
                }
            } else {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // Always handle shutdown even if the loop processing threw an exception.
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```

1.首先轮询注册到eventloop线程对用的selector上的所有的channel的IO事件 

 if (wakenUp.get()) {
                        selector.wakeup();
                    }

2.处理产生网络IO事件的channel 

 processSelectedKeys();

3.处理任务队列 

  runAllTasks();

我们看看select操作select(wakenUp.getAndSet(false));

```java
private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
        int selectCnt = 0;
        long currentTimeNanos = System.nanoTime();
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

        long normalizedDeadlineNanos = selectDeadLineNanos - initialNanoTime();
        if (nextWakeupTime != normalizedDeadlineNanos) {
            nextWakeupTime = normalizedDeadlineNanos;
        }

        for (;;) {
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
            if (timeoutMillis <= 0) {
                if (selectCnt == 0) {
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }

            // If a task was submitted when wakenUp value was true, the task didn't get a chance to call
            // Selector#wakeup. So we need to check task queue again before executing select operation.
            // If we don't, the task might be pended until select operation was timed out.
            // It might be pended until idle timeout if IdleStateHandler existed in pipeline.
            if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                selector.selectNow();
                selectCnt = 1;
                break;
            }

            int selectedKeys = selector.select(timeoutMillis);
            selectCnt ++;

            if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                // - Selected something,
                // - waken up by user, or
                // - the task queue has a pending task.
                // - a scheduled task is ready for processing
                break;
            }
            if (Thread.interrupted()) {
                // Thread was interrupted so reset selected keys and break so we not run into a busy loop.
                // As this is most likely a bug in the handler of the user or it's client library we will
                // also log it.
                //
                // See https://github.com/netty/netty/issues/2426
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely because " +
                            "Thread.currentThread().interrupt() was called. Use " +
                            "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                }
                selectCnt = 1;
                break;
            }

            long time = System.nanoTime();
            if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                // timeoutMillis elapsed without anything selected.
                selectCnt = 1;
            } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                    selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                // The code exists in an extra method to ensure the method is not too big to inline as this
                // branch is not very likely to get hit very frequently.
                selector = selectRebuildSelector(selectCnt);
                selectCnt = 1;
                break;
            }

            currentTimeNanos = time;
        }

        if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
            if (logger.isDebugEnabled()) {
                logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                        selectCnt - 1, selector);
            }
        }
    } catch (CancelledKeyException e) {
        if (logger.isDebugEnabled()) {
            logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                    selector, e);
        }
        // Harmless exception - log anyway
    }
}
```

我们可以看到，NioEventLoop中reactor线程的select操作也是一个for循环，在for循环第一步中，如果发现当前的定时任务队列中有任务的截止事件快到了(<=0.5ms)，就跳出循环。此外，跳出之前如果发现目前为止还没有进行过select操作（`if (selectCnt == 0)`），那么就调用一次`selectNow()`，该方法会立即返回，不会阻塞

这里说明一点，netty里面定时任务队列是按照延迟时间从小到大进行排序， `delayNanos(currentTimeNanos)`方法即取出第一个定时任务的延迟时间

2.轮询过程中发现有任务加入，中断本次轮询 

```java
for (;;) {
    long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
    if (timeoutMillis <= 0) {
        if (selectCnt == 0) {
            selector.selectNow();
            selectCnt = 1;
        }
        break;
    }

    // If a task was submitted when wakenUp value was true, the task didn't get a chance to call
    // Selector#wakeup. So we need to check task queue again before executing select operation.
    // If we don't, the task might be pended until select operation was timed out.
    // It might be pended until idle timeout if IdleStateHandler existed in pipeline.
    if (hasTasks() && wakenUp.compareAndSet(false, true)) {
        selector.selectNow();
        selectCnt = 1;
        break;
    }
```

3.阻塞式select操作 

```java
int selectedKeys = selector.select(timeoutMillis);
selectCnt ++;

if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
    // - Selected something,
    // - waken up by user, or
    // - the task queue has a pending task.
    // - a scheduled task is ready for processing
    break;
}
```

- 轮询到IO事件 （`selectedKeys != 0`）

- oldWakenUp 参数为true

- 任务队列里面有任务（`hasTasks`）

- 第一个定时任务即将要被执行 （`hasScheduledTasks（）`）

- 用户主动唤醒（`wakenUp.get()`）

  解决nio的空轮询bug

  ```java
   long time = System.nanoTime();
      if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
          // timeoutMillis elapsed without anything selected.
          selectCnt = 1;
      } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
              selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
          // The code exists in an extra method to ensure the method is not too big to inline as this
          // branch is not very likely to get hit very frequently.
          selector = selectRebuildSelector(selectCnt);
          selectCnt = 1;
          break;
      }
  
      currentTimeNanos = time;
  }
  ```

​       当空轮询的次数超过一个阀值的时候，默认是512，就开始重建selector 

​     下面我们回到run方法中的processSelectedKeys();

​         

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```

一种是处理优化过的selectedKeys，一种是正常的处理 SelectedSelectionKeySet中用数组代替了jdk的set，算法复杂度降低

```java
private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];
        // null out entry in the array to allow to have it GC'ed once the Channel close
        // See https://github.com/netty/netty/issues/2363
        selectedKeys.keys[i] = null;

        final Object a = k.attachment();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (needsToSelectAgain) {
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.reset(i + 1);

            selectAgain();
            i = -1;
        }
    }
}
```

对于boss NioEventLoop来说，轮询到的是基本上就是连接事件，后续的事情就通过他的pipeline将连接扔给一个worker NioEventLoop处理 对于worker NioEventLoop来说，轮询到的基本上都是io读写事件，后续的事情就是通过他的pipeline将读取到的字节流传递给每个channelHandler来处理 

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            // If the channel implementation throws an exception because there is no event loop, we ignore this
            // because we are only trying to determine if ch is registered to this event loop and thus has authority
            // to close ch.
            return;
        }
        // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
        // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
        // still healthy and should not be closed.
        // See https://github.com/netty/netty/issues/5125
        if (eventLoop != this || eventLoop == null) {
            return;
        }
        // close the channel if the key is not valid anymore
        unsafe.close(unsafe.voidPromise());
        return;
    }

    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }

        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```

还有个else分支，我们也来看一下 else部分的代码如下 

```java
private static void processSelectedKey(SelectionKey k, NioTask<SelectableChannel> task) {
    int state = 0;
    try {
        task.channelReady(k.channel(), k);
        state = 1;
    } catch (Exception e) {
        k.cancel();
        invokeChannelUnregistered(task, k, e);
        state = 2;
    } finally {
        switch (state) {
        case 0:
            k.cancel();
            invokeChannelUnregistered(task, k, null);
            break;
        case 1:
            if (!k.isValid()) { // Cancelled by channelReady()
                invokeChannelUnregistered(task, k, null);
            }
            break;
        }
    }
}
```

回到上面的方法

```java
if (needsToSelectAgain) {
    // null out entries in the array to allow to have it GC'ed once the Channel close
    // See https://github.com/netty/netty/issues/2363
    selectedKeys.reset(i + 1);

    selectAgain();
    i = -1;
}
```

判断是否需要重新注册，NioEventLoop.java 的cancel把needsToSelectAgain设置true

```java
void cancel(SelectionKey key) {
    key.cancel();
    cancelledKeys ++;
    if (cancelledKeys >= CLEANUP_INTERVAL) {
        cancelledKeys = 0;
        needsToSelectAgain = true;
    }
}
```

每隔256个channel从selector上移除的时候，就标记 needsToSelectAgain 为true 

保证现存的SelectionKey及时有效 

## netty中的task

场景



```java
@Sharable
public class EchoServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ctx.channel().eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                //...
            }
        });
        ctx.channel().eventLoop().schedule(new Runnable() {
            @Override
            public void run() {

            }
        }, 60, TimeUnit.SECONDS);
        ctx.write(msg);
    }
}
```

第一个是直接起一个任务，第二个是个定时任务

netty对这是怎么处理的了

第一个点进去执行

```java
   @Override
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
        addTask(task);
        if (!inEventLoop) {
            startThread();
            if (isShutdown()) {
                boolean reject = false;
                try {
                    if (removeTask(task)) {
                        reject = true;
                    }
                } catch (UnsupportedOperationException e) {
                    // The task queue does not support removal so the best thing we can do is to just move on and
                    // hope we will be able to pick-up the task before its completely terminated.
                    // In worst case we will log on termination.
                }
                if (reject) {
                    reject();
                }
            }
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
```

就是把调用 addTask(task);把任务add到queue中

```java
 final boolean offerTask(Runnable task) {
        if (isShutdown()) {
            reject();
        }
        return taskQueue.offer(task);
    }
public abstract class SingleThreadEventExecutor extends AbstractScheduledEventExecutor implements OrderedEventExecutor {

    static final int DEFAULT_MAX_PENDING_EXECUTOR_TASKS = Math.max(16,
            SystemPropertyUtil.getInt("io.netty.eventexecutor.maxPendingTasks", Integer.MAX_VALUE));

    private static final InternalLogger logger =
            InternalLoggerFactory.getInstance(SingleThreadEventExecutor.class);

    private static final int ST_NOT_STARTED = 1;
    private static final int ST_STARTED = 2;
    private static final int ST_SHUTTING_DOWN = 3;
    private static final int ST_SHUTDOWN = 4;
    private static final int ST_TERMINATED = 5;

    private static final Runnable WAKEUP_TASK = new Runnable() {
        @Override
        public void run() {
            // Do nothing.
        }
    };
    private static final Runnable NOOP_TASK = new Runnable() {
        @Override
        public void run() {
            // Do nothing.
        }
    };

    private static final AtomicIntegerFieldUpdater<SingleThreadEventExecutor> STATE_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(SingleThreadEventExecutor.class, "state");
    private static final AtomicReferenceFieldUpdater<SingleThreadEventExecutor, ThreadProperties> PROPERTIES_UPDATER =
            AtomicReferenceFieldUpdater.newUpdater(
                    SingleThreadEventExecutor.class, ThreadProperties.class, "threadProperties");

    private final Queue<Runnable> taskQueue;
```

### 定时调度任务

```java
 @Override
    public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
        ObjectUtil.checkNotNull(command, "command");
        ObjectUtil.checkNotNull(unit, "unit");
        if (delay < 0) {
            delay = 0;
        }
        validateScheduled0(delay, unit);

        return schedule(new ScheduledFutureTask<Void>(
                this, command, null, ScheduledFutureTask.deadlineNanos(unit.toNanos(delay))));
    }
 private <V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
        if (inEventLoop()) {
            scheduledTaskQueue().add(task);
        } else {
            executeScheduledRunnable(new Runnable() {
                @Override
                public void run() {
                    scheduledTaskQueue().add(task);
                }
            }, true, task.deadlineNanos());
        }

        return task;
    }
```

可以看到这里如果在eventloop，就直接加到一个队列scheduledTaskQueue



```java
 PriorityQueue<ScheduledFutureTask<?>> scheduledTaskQueue() {
        if (scheduledTaskQueue == null) {
            scheduledTaskQueue = new DefaultPriorityQueue<ScheduledFutureTask<?>>(
                    SCHEDULED_FUTURE_TASK_COMPARATOR,
                    // Use same initial capacity as java.util.PriorityQueue
                    11);
        }
        return scheduledTaskQueue;
    }
```

这个队列就是一个优先级队列，例如一个订单的超时时间失效也可以这么设计

当然如果不在eventloop中怎么办

```java
 executeScheduledRunnable(new Runnable() {
                @Override
                public void run() {
                    scheduledTaskQueue().add(task);
                }
            }, true, task.deadlineNanos());
```

我就把他包装成一个任务放进scheduledTaskQueue中，最后还是eventloop来执行

```java
final class ScheduledFutureTask<V> extends PromiseTask<V> implements ScheduledFuture<V>, PriorityQueueNode {
}
```

和jdk中优先级队列玩法差不多，下面就是他的run方法

```java
    @Override
    public void run() {
        assert executor().inEventLoop();
        try {
            if (periodNanos == 0) {
                if (setUncancellableInternal()) {
                    V result = task.call();
                    setSuccessInternal(result);
                }
            } else {
                // check if is done as it may was cancelled
                if (!isCancelled()) {
                    task.call();
                    if (!executor().isShutdown()) {
                        if (periodNanos > 0) {
                            deadlineNanos += periodNanos;
                        } else {
                            deadlineNanos = nanoTime() - periodNanos;
                        }
                        if (!isCancelled()) {
                            // scheduledTaskQueue can never be null as we lazy init it before submit the task!
                            Queue<ScheduledFutureTask<?>> scheduledTaskQueue =
                                    ((AbstractScheduledEventExecutor) executor()).scheduledTaskQueue;
                            assert scheduledTaskQueue != null;
                            scheduledTaskQueue.add(this);
                        }
                    }
                }
            }
        } catch (Throwable cause) {
            setFailureInternal(cause);
        }
    }
```

次任务执行完毕之后，间隔多长时间之后再次执行，截止时间为当前时间加上间隔时间

```java
  /**
     * Poll all tasks from the task queue and run them via {@link Runnable#run()} method.  This method stops running
     * the tasks in the task queue and returns if it ran longer than {@code timeoutNanos}.
     */
    protected boolean runAllTasks(long timeoutNanos) {
        fetchFromScheduledTaskQueue();
        Runnable task = pollTask();
        if (task == null) {
            afterRunningAllTasks();
            return false;
        }

        final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
        long runTasks = 0;
        long lastExecutionTime;
        for (;;) {
            safeExecute(task);

            runTasks ++;

            // Check timeout every 64 tasks because nanoTime() is relatively expensive.
            // XXX: Hard-coded value - will make it configurable if it is really a problem.
            if ((runTasks & 0x3F) == 0) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                if (lastExecutionTime >= deadline) {
                    break;
                }
            }

            task = pollTask();
            if (task == null) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                break;
            }
        }

        afterRunningAllTasks();
        this.lastExecutionTime = lastExecutionTime;
        return true;
    }
```

从scheduledTaskQueue转移定时任务到taskQueue(mpsc queue)

计算本次任务循环的截止时间

执行任务

收尾

```java
  private boolean fetchFromScheduledTaskQueue() {
        if (scheduledTaskQueue == null || scheduledTaskQueue.isEmpty()) {
            return true;
        }
        long nanoTime = AbstractScheduledEventExecutor.nanoTime();
        for (;;) {
            Runnable scheduledTask = pollScheduledTask(nanoTime);
            if (scheduledTask == null) {
                return true;
            }
            if (!taskQueue.offer(scheduledTask)) {
                // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.
                scheduledTaskQueue.add((ScheduledFutureTask<?>) scheduledTask);
                return false;
            }
        }
    }
```

把到期的定时任务转到queue中

```java
 for (;;) {
            safeExecute(task);

            runTasks ++;

            // Check timeout every 64 tasks because nanoTime() is relatively expensive.
            // XXX: Hard-coded value - will make it configurable if it is really a problem.
            if ((runTasks & 0x3F) == 0) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                if (lastExecutionTime >= deadline) {
                    break;
                }
            }

            task = pollTask();
            if (task == null) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                break;
            }
        }
```

死循环执行

netty每次执行任务时候，会把快过期定时任务加到队列里面

每隔64个任务检查一下是否要退出，用时间计算，防止长时间阻塞

## writeAndFlush

### write

通过ctx.writeAndFlush(msg);是把消息写出，我们来跟踪下源码

```java
 @Override
    public ChannelFuture writeAndFlush(Object msg) {
        return pipeline.writeAndFlush(msg);
    }
```

调用了pipeline，前面我们知道pipeline其实就是个链表，上面每个节点都是handler，其中头节点是in和out两种属性，tail是in属性，in表示读，out表示写

```java
 @Override
    public final ChannelFuture writeAndFlush(Object msg) {
        return tail.writeAndFlush(msg);
    }
```

从tail节点开始调用，向head开始

```java
 @Override
    public ChannelFuture writeAndFlush(Object msg) {
        return writeAndFlush(msg, newPromise());
    }
 @Override
    public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
        write(msg, true, promise);
        return promise;
    }
private void write(Object msg, boolean flush, ChannelPromise promise) {
        ObjectUtil.checkNotNull(msg, "msg");
        try {
            if (isNotValidPromise(promise, true)) {
                ReferenceCountUtil.release(msg);
                // cancelled
                return;
            }
        } catch (RuntimeException e) {
            ReferenceCountUtil.release(msg);
            throw e;
        }

        final AbstractChannelHandlerContext next = findContextOutbound(flush ?
                (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);
        final Object m = pipeline.touch(msg, next);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            if (flush) {
                next.invokeWriteAndFlush(m, promise);
            } else {
                next.invokeWrite(m, promise);
            }
        } else {
            final AbstractWriteTask task;
            if (flush) {
                task = WriteAndFlushTask.newInstance(next, m, promise);
            }  else {
                task = WriteTask.newInstance(next, m, promise);
            }
            if (!safeExecute(executor, task, promise, m)) {
                // We failed to submit the AbstractWriteTask. We need to cancel it so we decrement the pending bytes
                // and put it back in the Recycler for re-use later.
                //
                // See https://github.com/netty/netty/issues/8343.
                task.cancel();
            }
        }
    }
```

下面看看 final AbstractChannelHandlerContext next = findContextOutbound(flush ?
                (MASK_WRITE | MASK_FLUSH) : MASK_WRITE);，字面上的意思找到out属性的handler

```java
 private AbstractChannelHandlerContext findContextOutbound(int mask) {
        AbstractChannelHandlerContext ctx = this;
        do {
            ctx = ctx.prev;
        } while ((ctx.executionMask & mask) == 0);
        return ctx;
    }
```

记得前面说过netty 4  控制更加精细了关于handler



```java
final class ChannelHandlerMask {
    private static final InternalLogger logger = InternalLoggerFactory.getInstance(ChannelHandlerMask.class);

    // Using to mask which methods must be called for a ChannelHandler.
    static final int MASK_EXCEPTION_CAUGHT = 1;
    static final int MASK_CHANNEL_REGISTERED = 1 << 1;
    static final int MASK_CHANNEL_UNREGISTERED = 1 << 2;
    static final int MASK_CHANNEL_ACTIVE = 1 << 3;
    static final int MASK_CHANNEL_INACTIVE = 1 << 4;
    static final int MASK_CHANNEL_READ = 1 << 5;
    static final int MASK_CHANNEL_READ_COMPLETE = 1 << 6;
    static final int MASK_USER_EVENT_TRIGGERED = 1 << 7;
    static final int MASK_CHANNEL_WRITABILITY_CHANGED = 1 << 8;
    static final int MASK_BIND = 1 << 9;
    static final int MASK_CONNECT = 1 << 10;
    static final int MASK_DISCONNECT = 1 << 11;
    static final int MASK_CLOSE = 1 << 12;
    static final int MASK_DEREGISTER = 1 << 13;
    static final int MASK_READ = 1 << 14;
    static final int MASK_WRITE = 1 << 15;
    static final int MASK_FLUSH = 1 << 16;
```

 各种位运算做权限控制。flush为true时候传入98304，false为32768，别问我怎么知道的，万能的sysout

这个executionMake属性是在生成handler名字的时候出现过，前面pipeline有

wirte应该是static final int MASK_CHANNEL_WRITABILITY_CHANGED = 1 << 8;   256

```java

```



就是通过位运算找到out的handler

```java
 EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            if (flush) {
                next.invokeWriteAndFlush(m, promise);
            } else {
                next.invokeWrite(m, promise);
            }
        } else {
            final AbstractWriteTask task;
            if (flush) {
                task = WriteAndFlushTask.newInstance(next, m, promise);
            }  else {
                task = WriteTask.newInstance(next, m, promise);
            }
            if (!safeExecute(executor, task, promise, m)) {
                // We failed to submit the AbstractWriteTask. We need to cancel it so we decrement the pending bytes
                // and put it back in the Recycler for re-use later.
                //
                // See https://github.com/netty/netty/issues/8343.
                task.cancel();
            }
```

拿到EventExecutor，如果在inEventLoop中就直接调用  next.invokeWrite(m, promise);写入缓冲

如果是发送就直接调用 next.invokeWriteAndFlush(m, promise);

如果不是在inEventLoop，  task = WriteAndFlushTask.newInstance(next, m, promise);写入eventloop的task

```java
  private void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
        if (invokeHandler()) {
            invokeWrite0(msg, promise);
            invokeFlush0();
        } else {
            writeAndFlush(msg, promise);
        }
    }
 private void invokeWrite0(Object msg, ChannelPromise promise) {
        try {
            ((ChannelOutboundHandler) handler()).write(this, msg, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    }
  private void invokeFlush0() {
        try {
            ((ChannelOutboundHandler) handler()).flush(this);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    }
```

看看wirte方法进入到

```java
 @Override
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
            unsafe.write(msg, promise);
        }
   @Override
        public final void write(Object msg, ChannelPromise promise) {
            assertEventLoop();

            ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null) {
                // If the outboundBuffer is null we know the channel was closed and so
                // need to fail the future right away. If it is not null the handling of the rest
                // will be done in flush0()
                // See https://github.com/netty/netty/issues/2362
                safeSetFailure(promise, newClosedChannelException(initialCloseCause));
                // release message now to prevent resource-leak
                ReferenceCountUtil.release(msg);
                return;
            }

            int size;
            try {
                msg = filterOutboundMessage(msg);
                size = pipeline.estimatorHandle().size(msg);
                if (size < 0) {
                    size = 0;
                }
            } catch (Throwable t) {
                safeSetFailure(promise, t);
                ReferenceCountUtil.release(msg);
                return;
            }

            outboundBuffer.addMessage(msg, size, promise);
        }
 public void addMessage(Object msg, int size, ChannelPromise promise) {
        Entry entry = Entry.newInstance(msg, size, total(msg), promise);
        if (tailEntry == null) {
            flushedEntry = null;
        } else {
            Entry tail = tailEntry;
            tail.next = entry;
        }
        tailEntry = entry;
        if (unflushedEntry == null) {
            unflushedEntry = entry;
        }

        // increment pending bytes after adding message to the unflushed arrays.
        // See https://github.com/netty/netty/issues/1619
        incrementPendingOutboundBytes(entry.pendingSize, false);
    }
   private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
        if (size == 0) {
            return;
        }

        long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);
        if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {
            setUnwritable(invokeLater);
        }
    }

```

 msg = filterOutboundMessage(msg);把所有的非直接内存转换成直接内存`DirectBuffer`



```java
  @Override
    protected final Object filterOutboundMessage(Object msg) {
        if (msg instanceof ByteBuf) {
            ByteBuf buf = (ByteBuf) msg;
            if (buf.isDirect()) {
                return msg;
            }

            return newDirectBuffer(buf);
        }

        if (msg instanceof FileRegion) {
            return msg;
        }

        throw new UnsupportedOperationException(
                "unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
    }
  /**
     * Returns an off-heap copy of the specified {@link ByteBuf}, and releases the original one.
     * Note that this method does not create an off-heap copy if the allocation / deallocation cost is too high,
     * but just returns the original {@link ByteBuf}..
     */
    protected final ByteBuf newDirectBuffer(ByteBuf buf) {
        final int readableBytes = buf.readableBytes();
        if (readableBytes == 0) {
            ReferenceCountUtil.safeRelease(buf);
            return Unpooled.EMPTY_BUFFER;
        }

        final ByteBufAllocator alloc = alloc();
        if (alloc.isDirectBufferPooled()) {
            ByteBuf directBuf = alloc.directBuffer(readableBytes);
            directBuf.writeBytes(buf, buf.readerIndex(), readableBytes);
            ReferenceCountUtil.safeRelease(buf);
            return directBuf;
        }

        final ByteBuf directBuf = ByteBufUtil.threadLocalDirectBuffer();
        if (directBuf != null) {
            directBuf.writeBytes(buf, buf.readerIndex(), readableBytes);
            ReferenceCountUtil.safeRelease(buf);
            return directBuf;
        }

        // Allocating and deallocating an unpooled direct buffer is very expensive; give up.
        return buf;
    }
```

这里就是利用nio的api写入直接内存

下面看看上面的 public void addMessage(Object msg, int size, ChannelPromise promise) 方法

```java
创建一个待写出的消息节点
```

ChannelOutboundBuffer 里面的数据结构是一个单链表结构，每个节点是一个 `Entry`，`Entry` 里面包含了待写出`ByteBuf` 以及消息回调 `promise`，下面分别是三个指针的作用

1.flushedEntry 指针表示第一个被写到操作系统Socket缓冲区中的节点
2.unFlushedEntry 指针表示第一个未被写入到操作系统Socket缓冲区中的节点
3.tailEntry指针表示ChannelOutboundBuffer缓冲区的最后一个节点

调用n次`addMessage`，flushedEntry指针一直指向NULL，表示现在还未有节点需要写出到Socket缓冲区，而`unFushedEntry`之后有n个节点，表示当前还有n个节点尚未写出到Socket缓冲区中去

### flush

下面是flush，因为flush会到head中写出

```java
 private void invokeFlush0() {
        try {
            ((ChannelOutboundHandler) handler()).flush(this);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    }
   @Override
        public void flush(ChannelHandlerContext ctx) {
            unsafe.flush();
        }
@Override
        public final void flush() {
            assertEventLoop();

            ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null) {
                return;
            }

            outboundBuffer.addFlush();
            flush0();
        }


```



下面我们先看看addflush

```java
  /**
     * Add a flush to this {@link ChannelOutboundBuffer}. This means all previous added messages are marked as flushed
     * and so you will be able to handle them.
     */
    public void addFlush() {
        // There is no need to process all entries if there was already a flush before and no new messages
        // where added in the meantime.
        //
        // See https://github.com/netty/netty/issues/2577
        Entry entry = unflushedEntry;
        if (entry != null) {
            if (flushedEntry == null) {
                // there is no flushedEntry yet, so start with the entry
                flushedEntry = entry;
            }
            do {
                flushed ++;
                if (!entry.promise.setUncancellable()) {
                    // Was cancelled so make sure we free up memory and notify about the freed bytes
                    int pending = entry.cancel();
                    decrementPendingOutboundBytes(pending, false, true);
                }
                entry = entry.next;
            } while (entry != null);

            // All flushed so reset unflushedEntry
            unflushedEntry = null;
        }
    }
```

将 `flushedEntry` 指向`unflushedEntry`所指向的节点

然后看看flush0

```java
@SuppressWarnings("deprecation")
        protected void flush0() {
            if (inFlush0) {
                // Avoid re-entrance
                return;
            }

            final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null || outboundBuffer.isEmpty()) {
                return;
            }

            inFlush0 = true;

            // Mark all pending write requests as failure if the channel is inactive.
            if (!isActive()) {
                try {
                    if (isOpen()) {
                        outboundBuffer.failFlushed(new NotYetConnectedException(), true);
                    } else {
                        // Do not trigger channelWritabilityChanged because the channel is closed already.
                        outboundBuffer.failFlushed(newClosedChannelException(initialCloseCause), false);
                    }
                } finally {
                    inFlush0 = false;
                }
                return;
            }

            try {
                doWrite(outboundBuffer);
            } catch (Throwable t) {
                if (t instanceof IOException && config().isAutoClose()) {
                    /**
                     * Just call {@link #close(ChannelPromise, Throwable, boolean)} here which will take care of
                     * failing all flushed messages and also ensure the actual close of the underlying transport
                     * will happen before the promises are notified.
                     *
                     * This is needed as otherwise {@link #isActive()} , {@link #isOpen()} and {@link #isWritable()}
                     * may still return {@code true} even if the channel should be closed as result of the exception.
                     */
                    initialCloseCause = t;
                    close(voidPromise(), t, newClosedChannelException(t), false);
                } else {
                    try {
                        shutdownOutput(voidPromise(), t);
                    } catch (Throwable t2) {
                        initialCloseCause = t;
                        close(voidPromise(), t2, newClosedChannelException(t), false);
                    }
                }
            } finally {
                inFlush0 = false;
            }
        }
  @Override
    protected void doWrite(ChannelOutboundBuffer in) throws Exception {
        final SelectionKey key = selectionKey();
        final int interestOps = key.interestOps();

        for (;;) {
            Object msg = in.current();
            if (msg == null) {
                // Wrote all messages.
                if ((interestOps & SelectionKey.OP_WRITE) != 0) {
                    key.interestOps(interestOps & ~SelectionKey.OP_WRITE);
                }
                break;
            }
            try {
                boolean done = false;
                for (int i = config().getWriteSpinCount() - 1; i >= 0; i--) {
                    if (doWriteMessage(msg, in)) {
                        done = true;
                        break;
                    }
                }

                if (done) {
                    in.remove();
                } else {
                    // Did not write all messages.
                    if ((interestOps & SelectionKey.OP_WRITE) == 0) {
                        key.interestOps(interestOps | SelectionKey.OP_WRITE);
                    }
                    break;
                }
            } catch (Exception e) {
                if (continueOnWriteError()) {
                    in.remove(e);
                } else {
                    throw e;
                }
            }
        }
    }

```

调用`current()`先拿到第一个需要flush的节点的数据

```java
 public Object current() {
        Entry entry = flushedEntry;
        if (entry == null) {
            return null;
        }

        return entry.msg;
    }
```

拿到自旋锁的迭代次数

*返回写操作的最大循环数，直到
  \* {@link WritableByteChannel＃write（ByteBuffer）}返回一个非零值。
  *与并发编程中使用的自旋锁相似。
  *可以提高内存利用率和写入吞吐量，具体取决于
  \* JVM运行的平台。 默认值为{@code 16}。

自旋的方式将ByteBuf写出到jdk nio的Channel

```java
config().getWriteSpinCount()
```

自旋的方式将ByteBuf写出到jdk nio的Channel

```java
remoteAddressfor (int i = config().getWriteSpinCount() - 1; i >= 0; i--) {
                    if (doWriteMessage(msg, in)) {
                        done = true;
                        break;
                    }
                }
 @Override
    protected boolean doWriteMessage(Object msg, ChannelOutboundBuffer in) throws Exception {
        final SocketAddress remoteAddress;
        final ByteBuf data;
        if (msg instanceof AddressedEnvelope) {
            @SuppressWarnings("unchecked")
            AddressedEnvelope<ByteBuf, SocketAddress> envelope = (AddressedEnvelope<ByteBuf, SocketAddress>) msg;
            remoteAddress = envelope.recipient();
            data = envelope.content();
        } else {
            data = (ByteBuf) msg;
            remoteAddress = null;
        }

        final int dataLen = data.readableBytes();
        if (dataLen == 0) {
            return true;
        }

        final ByteBuffer nioData = data.nioBufferCount() == 1 ? data.internalNioBuffer(data.readerIndex(), dataLen)
                                                              : data.nioBuffer(data.readerIndex(), dataLen);
        final int writtenBytes;
        if (remoteAddress != null) {
            writtenBytes = javaChannel().send(nioData, remoteAddress);
        } else {
            writtenBytes = javaChannel().write(nioData);
        }
        return writtenBytes > 0;
    }
```

remoteAddress不为null时候

​         writtenBytes = javaChannel().send(nioData, remoteAddress);

jdk中api调用

最后删除节点

```java
 if (done) {
                    in.remove();
                } else {
                    // Did not write all messages.
                    if ((interestOps & SelectionKey.OP_WRITE) == 0) {
                        key.interestOps(interestOps | SelectionKey.OP_WRITE);
                    }
                    break;
                }
```

```java
 /**
     * Will remove the current message, mark its {@link ChannelPromise} as success and return {@code true}. If no
     * flushed message exists at the time this method is called it will return {@code false} to signal that no more
     * messages are ready to be handled.
     */
    public boolean remove() {
        Entry e = flushedEntry;
        if (e == null) {
            clearNioBuffers();
            return false;
        }
        Object msg = e.msg;

        ChannelPromise promise = e.promise;
        int size = e.pendingSize;

        removeEntry(e);

        if (!e.cancelled) {
            // only release message, notify and decrement if it was not canceled before.
            ReferenceCountUtil.safeRelease(msg);
            safeSuccess(promise);
            decrementPendingOutboundBytes(size, false, true);
        }

        // recycle the entry
        e.recycle();

        return true;
    }
```

removeEntry(e);

```java
  private void removeEntry(Entry e) {
        if (-- flushed == 0) {
            // processed everything
            flushedEntry = null;
            if (e == tailEntry) {
                tailEntry = null;
                unflushedEntry = null;
            }
        } else {
            flushedEntry = e.next;
        }
    }

```

这里的remove是逻辑移除，只是将flushedEntry指针移到下个节点，调用完毕之后，节点图示如下

，释放该节点数据的内存，调用 `safeSuccess` 进行回调

```java
 private static void safeSuccess(ChannelPromise promise) {
        // Only log if the given promise is not of type VoidChannelPromise as trySuccess(...) is expected to return
        // false.
        PromiseNotificationUtil.trySuccess(promise, null, promise instanceof VoidChannelPromise ? null : logger);
    }

```

仅在给定的承诺不是VoidChannelPromise类型的情况下才记录日志，因为trySuccess（...）预期会返回

所以我们可以这样写

```java
 ctx.writeAndFlush(msg).addListener(new GenericFutureListener<Future<? super Void>>() {
            @Override
            public void operationComplete(Future<? super Void> future) throws Exception {
                //TODO
            }
        });
```

writeAndFlush完结