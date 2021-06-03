笔者最近在看Netty相关的东西，想把过程中所学到的和感悟记录下来，于是决定单独开一个专栏，专门记录Netty相关的文章。


第一篇就从「简单入门」开始吧！！！


# Netty简介


> Netty是由JBOSS提供的一个java开源框架，现为 Github上的独立项目。Netty提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发高性能、高可靠性的网络服务器和客户端程序。



提取句子主干，首先，Netty是一个网络应用程序框架，也可以理解为网络IO框架。利用Netty，开发者可以快速开发出一个高性能、高可靠的网络服务器或客户端程序。


例如，你要开发一个RPC框架，生产者需要暴露服务，消费者需要调用服务。生产者和消费者之间如何通信呢？使用什么协议通信呢？双方通信的IO模型如何定义呢？通过Netty就可以快速实现。


Netty的特点就是异步的、事件驱动的、高性能的，下面分别说下。


## 异步


在Netty中，所有的IO操作都是异步的，这意味着如：接收请求，Channel数据的读写等操作都不会阻塞，Netty会返回一个ChannelFuture，它是一个异步操作的结果占位符。如果开发者就是想同步调用怎么办？通过调用`ChannelFuture.sync()`可以异步转同步，但是非常不建议这么做，它会阻塞当前线程，这和高性能是相悖的。


ChannelFuture的接口定义：


```java
public interface ChannelFuture extends Future<Void>
```


泛型是Void，这意味着你并不能通过ChannelFuture获取到操作操作的结果，但是你可以通过`addListener()`来注册回调，Netty会在异步操作完成时触发回调，这时你可以知道操作是否成功，以决定后续的操作。


Netty官方推荐使用`addListener()`注册监听来获取结果，而不是调用`await()`，`await()`会阻塞当前线程，这不仅浪费了当前线程资源，而且线程间的切换和数据同步需要较大的开销。
另外需要特别注意的是：不要在ChannelHandler中调用`await()`，Channel整个生命周期事件都由一个唯一绑定的EventLoop线程处理，执行ChannelHandler逻辑的也是EventLoop，调用`await()`相当于线程本身在等待自己操作完成的一个结果，这会导致死锁。


相比之下，`addListener()`是完全非阻塞的，它会注册一个监听到Channel，当异步操作完成时，EventLoop会负责触发回调，性能是最优的。


## 事件驱动


Netty程序是靠事件来驱动执行的。


Netty使用不同的事件来通知我们，我们可以根据已经发生的事件来执行对应的动作逻辑。


Netty是一个网络IO框架，所以事件可以按照出站和入站进行分类：


-  入站 
   1. 连接已激活/失活。
   1. 有数据可以读取。
   1. 用户自定义事件。
   1. 异常事件。

 

-  出站 
   1. 打开/关闭到远程节点的连接。
   1. 将数据write/flush到Socket。

 
当Channel被注册到EventLoop后，该EventLoop会开启线程不断轮询，直到Channel有事件发生。有事件发生时，EventLoop会触发相应的回调，通过ChannelPipeline进行事件的传播。
![](https://img-blog.csdnimg.cn/20210525221413460.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyMDk5ODMz,size_16,color_FFFFFF,t_70#id=lzw86&originHeight=296&originWidth=1126&originalType=binary&status=done&style=none)


## 高性能


Netty开发服务端程序时，面对海量客户端连接，还必须保证高性能。


Netty为了高性能做了很多努力和优化，这里简单列下，后面会详细说明，包括但不仅限于：


1. 非阻塞的Nio编程，主从Reactor线程模型，只需少量线程即可应对海量连接。
1. 基于引用计数算法的内存池化技术，避免ByteBuf的频繁创建和销毁。
1. 更少的内存复制： 
   - Socket读写数据使用堆外内存，避免内存拷贝。
   - CompositeByteBuf组合ByteBuf，实现数据零拷贝。
   - 文件传输FileRegion避免内存拷贝。

 

4. 局部无锁化，EventLoop串行执行事件和任务，避免了线程竞争和数据同步。
4. Netty实现的MpscQueue高性能无锁队列。
4. 反射替换SelectorImpl的selectedKeys，将HashSet替换为数组，避免哈希冲突。
4. FastThreadLocal使用数组代替Hash表，带来更好的访问性能。
4. ... ...想到再补充。



# Netty的组件


## Channel


Channel译为「通道」，它代表一个到实体(文件、硬件、Socket等)的开放连接，例如针对网络有SocketChannl，针对文件有FileChannel等。既然是通道，就代表它可以被打开，也可以被关闭，


Netty没有使用JDK原生的Channel，而是自己封装了一个，这样可以为客户端和服务端Channel提供一个统一的视图，使用起来更加方便。


Channel分为两大类：


1. 服务端ServerSocketChannel，负责绑定本地端口，监听客户端的连接请求。
1. 客户端SocketChannel，负责和远程节点建立连接。



在网络编程模型中, 服务端和客户端进行IO数据交互的媒介就是Channel，Channel被打开的目的就是与对端进行数据交换，你可以通过Channel来给对端发送数据，和从对端读取数据。


常用的Channel实现如下：
![](https://img-blog.csdnimg.cn/img_convert/06228f2f5e6a3161cf653d1825279884.png)

## EventLoopGroup和EventLoop


EventLoopGroup本身并不干活，它负责管理一组EventLoop的启动和停止，它提供一个`next()`方法从一组EventLoop线程中挑选出一个来执行任务。
​

EventLoopGroup的`next()`方法依赖一个`EventExecutorChooser`选择器，通过选择器来从一组EventLoop中进行选择，Netty默认的策略就是简单轮询，源码如下：
```java
/*
创建一个选择器，从一组EventExecutor中挑选出一个。
Netty默认的选择策略就是:简单轮询。
*/
@Override
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    // 两种Chooser实现都有一个AtomicLong计数器，每次next()先自增再取余

    // 如果数量是2的幂次方数，则采用位运算
    if (isPowerOfTwo(executors.length)) {
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        // 否则，对长度进行取余
        return new GenericEventExecutorChooser(executors);
    }
}

// 2的幂次方数的选择器，位运算
private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
    private final AtomicInteger idx = new AtomicInteger();
    private final EventExecutor[] executors;

    PowerOfTwoEventExecutorChooser(EventExecutor[] executors) {
        this.executors = executors;
    }

    @Override
    public EventExecutor next() {
        // 计数器自增 & 长度-1，和HashMap一样
        return executors[idx.getAndIncrement() & executors.length - 1];
    }
}

// 普通的选择器，取余
private static final class GenericEventExecutorChooser implements EventExecutorChooser {
    private final AtomicLong idx = new AtomicLong();
    private final EventExecutor[] executors;

    GenericEventExecutorChooser(EventExecutor[] executors) {
        this.executors = executors;
    }

    @Override
    public EventExecutor next() {
        return executors[(int) Math.abs(idx.getAndIncrement() % executors.length)];
    }
}
```


EventLoopGroup管理的一组EventLoop应该趋向于处理同一类任务和事件，例如开发服务端程序，Netty官方推荐的Reactor主从线程模型需要两个EventLoopGroup：Boss和Worker，Boss专门负责接收客户端的连接，连接建立后，Boss会将客户端Channel注册到Worker中，由Worker来负责后续的数据读写事件。


EventLoopGroup可以理解为是一个多线程的线程池，而EventLoop则是一个单线程的线程池，也是真正干活的角色。
​

EventLoop不仅可以处理Channel的IO事件，还可以执行用户提交的系统任务，因为它本身就是个线程池。此外，它还实现了`ScheduledExecutorService`接口，因此它还可以执行定时任务。最常见的应用场景就是：你可以每隔一段时间检测一下客户端连接是否断开！
​

**EventLoop是如何工作的呢？**
​

拿最常用的NioEventLoop来说，它内部会持有一个`Selector`多路复用器，初始化时，`Selector`也会被一同创建，然后当有任务被提交到NioEventLoop时，它会利用`ThreadPerTaskExecutor`创建一个线程执行`run()`方法。核心就在`run()`方法里，它会不断的轮询，检查`Selector`上是否有准备就绪的Channel需要处理，如果有则根据`SelectionKey`的事件类型触发相应的事件回调，并通过`ChannelPipeline`将事件传播出去。
如果没有准备就绪的Channel，则去检查`taskQueue`中是否有待处理的系统任务、或定时任务，如果有则执行，否则就阻塞在`Selector.select()`上，等待准备就绪的Channel。
​

这里就简单过一下吧，后面会有源码解析的文章，敬请期待！


## ChannelFuture
前面已经说过，Netty是完全异步的IO框架，它所有的操作都会立即返回，不会阻塞在那里，这对于习惯了同步编程的同学可能要适应一下。你不能再调用一个方法，得到一个结果，根据结果判断再去执行后面的操作。因为此时异步操作可能还没有执行完，ChannelFuture还没有结果。
​

ChannelFuture只是一个异步操作的结果占位符，它代表未来可能会发生的一个结果，这个结果可能是执行成功，或是执行失败得到一个异常信息。
​

你可以通过调用`await()`阻塞等待这个操作完成，但是Netty不建议这么去做，这样会阻塞当前线程，浪费线程资源，而且线程间的切换和数据同步都是一个较大的开销。
​

Netty推荐使用`addListener()`来注册一个回调，当操作执行完成/异常时，ChannelFuture会向EventLoop提交任务来触发回调，你可以在回调方法里根据操作结果来执行后面的业务逻辑。回调和任务是由同一个线程驱动的，这样就避免了线程间数据同步的问题，性能是最好的。


**ChannelPromise和ChannelFuture的区别？**
ChannelPromise是ChannelFuture的子类，是一个特殊的可写的ChannelFuture。前面说过ChannelFuture代表未来操作的一个结果占位符，使用ChannelFuture你只能乖乖等待结果完成然后触发回调，这个结果是由Netty来设置，它没有提供可写操作。
而ChannelPromise就不同了，它提供了手动设置结果的API：`setSuccess()`和`setFailure()`，结果只能设置一次，设置完后会自动触发回调。


入站事件处理器`ChannelInboundHandler`所有操作都不需要提供ChannelPromise，因为这些回调是由Netty来主动触发的。而出站事件处理器`ChannelOutboundHandler`很多操作都需要提供一个ChannelPromise，当出站数据处理完成时，你需要往ChannelPromise设置结果来通知回调。




## ChannelHandler
ChannelHandler是Netty的事件处理器，根据数据的流向，分为入站、出站两大类。
​


- ChannelInboundHandler：入站事件处理器
- ChannelOutboundHandler：出站事件处理器

​
当一个ChannelHandler被添加到Channel的Pipeline后，只要EventLoop轮询到Channel有事件发生时，就会根据事件类型触发相应的回调。例如：收到对端发送的数据，Channel有数据可读时，会触发`channelRead()`方法。你要做的就是实现ChannelHandler类，重写`channelRead()`方法，Netty会将读取到的数据包装成`ByteBuf`，至于拿到数据要做哪些事，那就是你的业务了。
​
对于Netty开发者来说，你的主要工作，就是开发ChannelHandler，实现ChannelHandler类，重写你感兴趣的事件即可。
​

常用的类有：ChannelInboundHandlerAdapter和ChannelOutboundHandlerAdapter，分别处理入站和出站事件的，默认全部通过`ctx.fireXXX()`无脑向后传播，你只需要重写你感兴趣的事件，不用被迫重写所有方法了。
​

继承`SimpleChannelInboundHandler`类，你只需要重写`channelRead0()`方法，它会在该方法执行完毕后自动释放内存，防止内存泄漏，这一块后面会详细说明。
​

其他的就是Netty内置的一些开箱即用的编解码器，可以针对公有协议（如HTTP）进行编解码，处理读/写半包的问题，SslHandler针对读写数据进行加解密等等。
​

**需要注意的点：**

1. Netty使用池化技术来复用ByteBuf对象，使用完毕后切记及时释放资源。
1. 如果你需要将事件传播下去，必须手动触发`fireXXX()`方法，Pipeline可不会自动帮你传递。
1. 数据读取需要注意：粘包/拆包 问题。
1. 注意`write()`消息积压问题。



## ChannelPipeline
Pipeline译为「管道」，如果把 网络数据 比作水，那它就像 水管 一样，读入的数据从管道的头部流入，经过一系列ChannelInboundHandler处理，写出的数据从管道的尾部流入，经过一系列ChannelOutboundHandler处理，最终通过Socket发送给对端。
​

ChannelPipeline是ChannelHandler的容器，默认实现`DefaultChannelPipeline`是一个双向链表，头节点始终是HeadContext，尾节点始终是TailContext(头尾节点有它们自己的职责所在)，你可以往中间添加你自定义的ChannelHandler。
​

HeadContext头节点的职责：对于入站事件，它会无脑向后传播，确保你定义的ChannelHandler事件会被触发，对于出站事件，它会转交给Channel.Unsafe执行，例如`bind`、`write`等，因为这些操作是偏底层的，需要和底层类打交道，Netty不希望开发者去调用这些方法。


TailContext尾节点的职责：对于出站事件，当然是无脑向后传递了，但是对于入站事件，如果前面的Handler没有释放读取的数据资源，TailContext会自动释放，避免内存泄漏。对于异常，如果前面的Handler没有处理，TailContext会打印日志记录下来，提醒开发者需要处理异常。


## Bootstrap和ServerBootstrap
Netty的引导类，Bootstrap是客户端的引导类，ServerBootstrap是服务端的引导类。


一个Netty服务的运行需要多个组件互相配合，使用Bootstrap可以快速组装这些组件，让它们协同运行。当然，你也可以脱离Bootstrap，自己去引导服务，只是完全没有必要而已。
​

Bootstrap的创建非常简单，默认的构造器不需要你传任何参数，这是因为它需要的参数很多，可能以后的版本还会发生改变，因此Netty使用建造者Builder模式来构建Bootstrap。
​

Bootstrap本身逻辑很简单，它只是负责组装组件，核心的逻辑都在各个组件里。下面是一个ServerBootstrap的标准启动代码示例，这篇文章先简单带过，后面会有详细的源码解析。
```java
public class Server {
    public static void main(String[] args) {
        /*
        NioEventLoopGroup创建流程:
            1.创建ThreadPerTaskExecutor，利用FastThreadLocalThread来提升FastThreadLocal的性能
            2.初始化children，创建一组NioEventLoop(此时未创建线程)
            3.创建EventExecutorChooser，需要时从Group中轮询出一个EventLoop来执行任务
         */
        final NioEventLoopGroup boss = new NioEventLoopGroup(1);
        final NioEventLoopGroup worker = new NioEventLoopGroup();
        /*
        NioEventLoop流程:
            1.创建两个任务队列:taskQueue、tailTaskQueue
            2.openSelector()创建多路复用器Selector
            3.run()轮询Selector、taskQueue，串行处理IO事件和Task
        懒启动，只有在第一次execute()提交任务时才会利用executor创建线程
        对于Boss来说，线程启动是在调用bind()时，提交一个register()任务
        对于Worker，线程启动是在Boss接收到客户端连接时，提交一个register()任务
         */
        new ServerBootstrap()
                .group(boss, worker)
                .option(ChannelOption.SO_BACKLOG, 100)
                //.attr(null,null)
                .childOption(ChannelOption.SO_TIMEOUT, 1000)
                //.childAttr(null,null)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                            @Override
                            public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
                                ByteBuf buf = Unpooled.wrappedBuffer("hello".getBytes());
                                ctx.writeAndFlush(buf);
                            }
                        });
                    }
                }).bind(9999);
    }
}
```


# 第一个Netty程序
OK，前面介绍完Netty的组件，现在我们就基于这些组件，来写第一个Netty程序。
​

下面是一个Echo服务实例，如果有新的客户端接入，服务端会打印一句话，如果客户端向服务端发送数据，服务端会打印数据内容，并将数据原样写回给客户端，一个非常简单的程序。
​

`EchoServer`服务端标准实现：
```java
public class EchoServer {
    // 绑定的端口
    private final int port;

    public EchoServer(int port) {
        this.port = port;
    }

    public static void main(String[] args) {
        // 启动Echo服务
        new EchoServer(9999).start();
    }

    public void start() {
        /*
        bossGroup负责客户端的接入
        workerGroup负责IO数据的读写
         */
        NioEventLoopGroup boss = new NioEventLoopGroup(1);
        NioEventLoopGroup worker = new NioEventLoopGroup();
        new ServerBootstrap()
                .group(boss, worker)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel sc) throws Exception {
                        sc.pipeline().addLast(new ChannelInboundHandlerAdapter(){

                            @Override
                            public void channelActive(ChannelHandlerContext ctx) throws Exception {
                                super.channelActive(ctx);
                                System.out.println("有新的客户端连接...");
                            }

                            @Override
                            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                /*
                                原样写回给客户端，因为OutBoundHandler还要使用，因此不能释放msg。
                                底层数据写完后会自动释放。
                                 */
                                byte[] bytes = ByteBufUtil.getBytes(((ByteBuf) msg));
                                System.out.println("接受到数据:" + new String(bytes));
                                ctx.writeAndFlush(msg);
                            }

                            @Override
                            public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
                                // 出现异常了
                                cause.printStackTrace();
                                ctx.channel().close();
                            }
                        });
                    }
                })
                .bind(port);
    }
}
```
`EchoClient`标准实现：
```java
public class EchoClient {
    private final String host;//远程IP
    private final int port;//远程端口

    public EchoClient(String host, int port) {
        this.host = host;
        this.port = port;
    }

    public static void main(String[] args) {
        new EchoClient("127.0.0.1", 9999).start();
    }

    public void start() {
        // 客户端只需要一个WorkerGroup
        NioEventLoopGroup worker = new NioEventLoopGroup();
        new Bootstrap()
                .group(worker)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel sc) throws Exception {
                        sc.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                            @Override
                            public void channelActive(ChannelHandlerContext ctx) throws Exception {
                                super.channelActive(ctx);
                                System.out.println("连接建立,开始发送【hello】...");
                                ctx.writeAndFlush(Unpooled.wrappedBuffer("hello".getBytes()));
                            }

                            @Override
                            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                String data = ((ByteBuf) msg).toString(Charset.defaultCharset());
                                System.out.println("收到服务端数据:" + data);
                            }
                        });
                    }
                }).connect(host, port);//连接服务端
    }
}
```
如上，只需少量代码就可以快速开发出一个Echo服务，Netty向开发者屏蔽了底层实现，你甚至都不需要知道`Selector`多路复用器，Channel是何时注册到`Selector`上的？Netty是如何处理IO事件的？网络数据是如何被读入的？又是如何被写出的？你只需要开发ChannelHandler，写好回调逻辑，等待Netty调用即可。
​

如果使用JDK原生类网络编程，Bio和Nio两种不同的模式代码风格差异非常大，如果需要在两者之间做切换，工作量非常巨大。而Netty就显得非常灵活，只需要将`NioEventLoopGroup`和`NioSocketChannel`换成`OioEventLoopGroup`和`OioSocketChannel`即可快速切换，这是Netty易用性和灵活性的极好体现。


# 总结
Netty作为事件驱动的异步IO框架，在保证高性能的同时，还拥有非常好的灵活性和可扩展性。使用Netty你可以快速构建你的网络服务，或者开发一个框架，需要进行节点间的通信和数据传输，使用Netty来帮助你完成底层的通信是非常方便和高效的。例如阿里的Dubbo、Facebook的Thrift等RPC框架都使用Netty来完成底层的通信，你甚至可以使用Netty来定制一套你们公司内部的私有协议，非常酷！
​

入门就先写到这里吧，后面会有Netty服务端启动全流程的源码分析，一步一步看看Netty到底干了什么，后面针对Netty对高性能所做出的努力也会单独写篇文章，包括`FastThreadLocal`也会分析源码，看看Netty是如何提升`ThreadLocal`的性能的。敬请期待吧！！！