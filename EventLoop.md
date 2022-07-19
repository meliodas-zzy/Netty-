**NioEventLoop结构图**

<img src="C:\Users\zhanyu\AppData\Roaming\Typora\typora-user-images\image-20220719211732137.png" alt="image-20220719211732137" style="zoom:80%;" />

NioEventLoop的继承结构还是挺复杂的。。

**NioEventLoopGroup结构图**

<img src="C:\Users\zhanyu\AppData\Roaming\Typora\typora-user-images\image-20220719211849105.png" alt="image-20220719211849105" style="zoom:80%;" />

可以看到NioEventLoop和NioEventLoopGroup都实现了Executor接口，因此二者都是线程池。

**NioEventLoopGroup源码**

首先从NioEventLoopGroup的构造函数开始

```
public NioEventLoopGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory, SelectorProvider selectorProvider, SelectStrategyFactory selectStrategyFactory, RejectedExecutionHandler rejectedExecutionHandler, EventLoopTaskQueueFactory taskQueueFactory, EventLoopTaskQueueFactory tailTaskQueueFactory) 
{
    super(nThreads, executor, chooserFactory, new Object[]{selectorProvider, selectStrategyFactory, rejectedExecutionHandler, taskQueueFactory, tailTaskQueueFactory});
}
```

这个是NioEventLoopGroup构造函数中参数最全的一个。这些参数含义如下

* nThreads：线程池中线程的数量，也即NioEventLoop的数量，NioEventLoopGroup中包含一个children数组，用于存放NioEventLoop
* executor: 这个线程池是给NioEventLoop使用的
* chooserFactory：选择策略，负责选择一个线程来执行任务
* selectorProvider: 用来实例化selector
* rejectedExecutionHandler: 拒绝策略

调用NioEventLoopGroup的父类构造函数时有一些重要的步骤，这里列出来一部分。

* 若之前没有指定nThreads值，则会默认设置为DEFAULT_EVENT_LOOP_THREADS，这个数值是CPU核心数 * 2。

```
super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, chooserFactory, args);
```

* 初始化一个childred数组，他的作用是用来存放NioEventLoop

```
this.children = new EventExecutor[nThreads];
```

```
for(int i = 0; i < nThreads; ++i) {
    ...
    this.children[i] = this.newChild((Executor)executor, args);
    ...
}
```

* 初始化executor

```
if (executor == null) {
    executor = new ThreadPerTaskExecutor(this.newDefaultThreadFactory());
}
```

* 初始化选择策略

```
this.chooser = chooserFactory.newChooser(this.children);
```

选择策略主要有两种，根据线程的数量是否为2 ^ n，分为两种

```
public EventExecutorChooserFactory.EventExecutorChooser newChooser(EventExecutor[] executors) {
    return (EventExecutorChooserFactory.EventExecutorChooser)(isPowerOfTwo(executors.length) ? new PowerOfTwoEventExecutorChooser(executors) : new GenericEventExecutorChooser(executors));
}
```

线程数量为2 ^ n时，借助公式 num & 2 ^ n - 1 = num % 2 ^ n

```
public EventExecutor next() {
            return this.executors[this.idx.getAndIncrement() & this.executors.length - 1];
        }
```

否则就取余

```
public EventExecutor next() {
    return this.executors[(int)Math.abs(this.idx.getAndIncrement() % (long)this.executors.length)];
}
```

前面有说道NioEventLoopGroup中的children数组存放的是NioEventLoop,所以二者的关联就要从这里开始分析，在初始化childred数组时，调用了newChild方法，NioEventLoopGroup重写了这个方法，用来创建NioEventLoop

```
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    SelectorProvider selectorProvider = (SelectorProvider)args[0];
    SelectStrategyFactory selectStrategyFactory = (SelectStrategyFactory)args[1];
    RejectedExecutionHandler rejectedExecutionHandler = (RejectedExecutionHandler)args[2];
    EventLoopTaskQueueFactory taskQueueFactory = null;
    EventLoopTaskQueueFactory tailTaskQueueFactory = null;
    int argsLength = args.length;
    if (argsLength > 3) {
        taskQueueFactory = (EventLoopTaskQueueFactory)args[3];
    }

    if (argsLength > 4) {
        tailTaskQueueFactory = (EventLoopTaskQueueFactory)args[4];
    }

    return new NioEventLoop(this, executor, selectorProvider, selectStrategyFactory.newSelectStrategy(), rejectedExecutionHandler, taskQueueFactory, tailTaskQueueFactory);
}
```

其中调用了NioEventLoop的构造方法

```
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider, SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler, EventLoopTaskQueueFactory taskQueueFactory, EventLoopTaskQueueFactory tailTaskQueueFactory) {
    super(parent, executor, false, newTaskQueue(taskQueueFactory), newTaskQueue(tailTaskQueueFactory), rejectedExecutionHandler);
    this.provider = (SelectorProvider)ObjectUtil.checkNotNull(selectorProvider, "selectorProvider");
    this.selectStrategy = (SelectStrategy)ObjectUtil.checkNotNull(strategy, "selectStrategy");
    SelectorTuple selectorTuple = this.openSelector();
    this.selector = selectorTuple.selector;
    this.unwrappedSelector = selectorTuple.unwrappedSelector;
}
```

构造方法中含盖了NioEventLoop主要的成员，不再详细分析了，直接写出一些结论：

* NioEventLoop内部有一个线程池，但只有一个线程（因为他是SingleThreadEventExecutor的子类）。

* NioEventLoop会有属于自己的一个Selector
* NioEventLoop中有一个taskQueue用来存放提交过来的任务，当任务满时，会执行线程池的拒绝策略



NioEventLoop是和Channel绑定的，且可以绑定多个Channel，但一个Channel只能绑定一个NioEventLoop。执行Channel传来的任务的肯定是一个线程，而NioEventLoop正好是一个线程池，但是之前的源码中并没有进行绑定工作，也没有真正的创建线程，那么这些工作什么时候进行呢？实际上是还是要在初始化Channel的initRegister方法中。

**Register流程**

```
final ChannelFuture initAndRegister() {
    Channel channel = null;

    try {
        channel = this.channelFactory.newChannel();
        this.init(channel);
    } catch (Throwable var3) {
        if (channel != null) {
            channel.unsafe().closeForcibly();
            return (new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE)).setFailure(var3);
        }

        return (new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE)).setFailure(var3);
    }

    ChannelFuture regFuture = this.config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }

    return regFuture;
}
```

在ChannelFuture regFuture = this.config().group().register(channel);中，会完成绑定及创建线程工作。

```
public ChannelFuture register(Channel channel) {
	//next()方法会根据之前说的选择策略来选择一个NioEventLoop然后执行register
    return this.next().register(channel);
}
```

继续向下 看

```
public ChannelFuture register(ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    //这里就完成了Channel与NioEventLoop的绑定
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```



```
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    ...
    //绑定
    AbstractChannel.this.eventLoop = eventLoop;
    if (eventLoop.inEventLoop()) {
        this.register0(promise);
    } else {
        try {
        	//要执行任务了，那肯定要启动线程
            eventLoop.execute(new Runnable() {
                public void run() {
                    AbstractUnsafe.this.register0(promise);
                }
            });
        } 
       ...
    }
}
```



```
private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = this.inEventLoop();
    this.addTask(task);
    if (!inEventLoop) {
        this.startThread();
        if (this.isShutdown()) {
            boolean reject = false;

            try {
                if (this.removeTask(task)) {
                    reject = true;
                }
            } catch (UnsupportedOperationException var6) {
            }

            if (reject) {
                reject();
            }
        }
    }

    if (!this.addTaskWakesUp && immediate) {
        this.wakeup(inEventLoop);
    }

}
```

至此线程已经启动了，之后NioEventLoop会执行Select操作，等待任务到来然后处理。这也是NioEventLoop工作的核心



