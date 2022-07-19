**Channel**

在使用Bootstrap初始化客户端及服务端时，典型的流程如下

```
//客户端初始化
Bootstrap b = new Bootstrap();
b.group(group)
        .channel(NioSocketChannel.class)
        .option(ChannelOption.TCP_NODELAY,true)
        .handler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) throws Exception{
               
            }
        });
ChannelFuture future = b.connect("localhost",8080).sync();
```

```
//服务端初始化
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();

try{
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup,workerGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception{
                    
                }
            })
            .option(ChannelOption.SO_BACKLOG,128)
            .childOption(ChannelOption.SO_KEEPALIVE,true);

    ChannelFuture future = b.bind(port).sync();
```

过程很相似，首先以客户端为例分析Channel的初始化流程。

channel(NioServerSocketChannel.class)方法返回一个channelFactory，由工厂模式新建一个NioServerSocketChannel。

```
public B channel(Class<? extends C> channelClass) {
    return this.channelFactory((io.netty.channel.ChannelFactory)(new 				ReflectiveChannelFactory((Class)ObjectUtil.checkNotNull(channelClass, "channelClass"))));
}
```

ChannelFactory只提供了一个方法，来看ReflectiveChannelFactory中的具体实现。

```
public interface ChannelFactory<T extends Channel> {
    T newChannel();
}
```

```
public T newChannel() {
    try {
    	//调用的是Channel的无参构造
        return (Channel)this.constructor.newInstance();
    } catch (Throwable var2) {
        throw new ChannelException("Unable to create Channel from class " + this.constructor.getDeclaringClass(), var2);
    }
}
```

而在客户端初始化过程中，最常用的是NioSocketChannel,因此最终调用的是NioSocketChannel的无参构造函数。但是这里的channel()方法只是返回了一个工厂类，并没有真正的创建对象。

NioSocketChannel的创建是在Bootstrap.connect()方法中：

```
public ChannelFuture connect(String inetHost, int inetPort) {
    return this.connect(InetSocketAddress.createUnresolved(inetHost, inetPort));
}
```

继续看connect方法

```
public ChannelFuture connect(SocketAddress remoteAddress) {
	//做一些检查工作
    ObjectUtil.checkNotNull(remoteAddress, "remoteAddress");
    this.validate();
    return this.doResolveAndConnect(remoteAddress, this.config.localAddress());
}
```

继续看doResolveAndConnect方法

```
private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
    ChannelFuture regFuture = this.initAndRegister();
    final Channel channel = regFuture.channel();
    ...
    return promise;
}
```

可以看出，构建工作在initAndRegister中完成

```
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = this.channelFactory.newChannel();
        this.init(channel);
    } catch (Throwable var3) {
        ...
    }
}
```

由之前的分析可知，这里的newChannel调用NioServerSocketChannel的无参构造方法，而无参构造方法会最终调用NioSocketChannel(Channel parent, java.nio.channels.SocketChannel socket)方法

```
public NioSocketChannel() {
    this(DEFAULT_SELECTOR_PROVIDER);
}

public NioSocketChannel(SelectorProvider provider) {
	//这里会创建一个java.nio.channels.SocketChannel对象，因此NioSocketChannel对象底层是一个		//SocketChannel对象
    this(newSocket(provider));
}

public NioSocketChannel(java.nio.channels.SocketChannel socket) {
    this((Channel)null, socket);
}

public NioSocketChannel(Channel parent, java.nio.channels.SocketChannel socket) {
    super(parent, socket);
    this.config = new NioSocketChannelConfig(this, socket.socket());
}
```

父类构造函数如下

```
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    //客户端的第一次连接，感兴趣的事件自然是服务端的回应
    this.readInterestOp = readInterestOp;

    try {
    	//设置非阻塞模式
        ch.configureBlocking(false);
    } catch (IOException var7) {
        ...
    }
}
```

至此，客户端channel初始化基本完成了，服务端其实流程基本一致，只不过其底层为一个ServerSocketChannel且感兴趣的事件为accept

```
public NioServerSocketChannel(java.nio.channels.ServerSocketChannel channel) {
		//这里的16代表accept
        super((Channel)null, channel, 16);
        this.config = new NioServerSocketChannelConfig(this, this.javaChannel().socket());
    }
```

```
//SelectionKey中16代表ACCEPT事件
public static final int OP_ACCEPT = 1 << 4;
```