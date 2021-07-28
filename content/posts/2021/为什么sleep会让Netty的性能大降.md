---
title: "为什么sleep会让Netty的性能大降"
date: 2021-03-17T13:37:54+08:00
draft: false
categories: ["Netty"]
tags: ["Netty"]
---

## TL;DR

[即时通讯技术分享](https://www.zhihu.com/column/helloim) 在知乎上分享了一些列的高性能网络编程的文章，该系列文章从底层原理说起，提到了高性能网络的方方面面，特别的干货。但是在第七篇文章中，[高性能网络编程(七)：到底什么是高并发？一文即懂！](https://zhuanlan.zhihu.com/p/214330310#comment-1274226668?notificationId=1351107234450092032)，作者通过ab分别对Java(Netty)和PHP(Swoole)进行性能压测的部分让我产生了一些疑惑。

作者使用了`ab`命令分别进行了压测

ab命令：`docker run --rm jordi/ab -k -c 1000 -n 1000000 http://10.234.3.32:5555/`

在并发1000进行100万次Http请求的基准测试中的结果如下。

**Netty的压测结果:**

![img](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/v2-254ce06a11e42c9ba1c50418a81c7e00_720w.jpg)

**Swoole的压测结果:**

![img](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/v2-445f9e0ccedd446dd774a6ce14de8d53_720w.jpg)

附图直接使用了上述文章中的图片，从数据来看Netty的Requests per second为84042.11，Swoole的Requests per second的结果为87222.98，可见在默认情况下，两者的表现基本是一致的。(不过docker容器、1G内存+2核CPU，能跑出这样的QPS，还是挺意外的)

但是后来作者通过在Java和PHP代码中,分别加上 sleep(0.01) //秒 的代码，模拟0.01秒的系统调用阻塞，下面就产生了奇迹时刻：

**Netty的压测结果:**

![img](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/v2-5d23f2013dce03b30ae8ed4cd0cfcd07_720w.jpg)

**Swoole的压测结果:**

![img](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/v2-20236790e5626397b6f7f07f87853616_720w.jpg)

可以看到，Netty的QPS一下子降低到了1562.69，较原来的8万多，一下子降低了好几十倍。比起Swoole，也差了好几倍，作者也提到 “**从结果中可以看出：**基于协程的php+ swoole服务比 Java + netty服务的QPS高了6倍。” 。而且这还是0.01秒的结果，到了真实的业务系统上，实际的业务操作时间往往都超过0.01秒。

那么，是什么原因，让Netty的性能大降呢？



## Netty 线程模型

首先需要了解下Netty的线程模型，那上文中的示例来说，其实际上是官方的Http中的HelloWorld示例 - [HttpHelloWorldServer.java](https://github.com/netty/netty/blob/4.1/example/src/main/java/io/netty/example/http/helloworld/HttpHelloWorldServer.java)

```java
public final class HttpHelloWorldServer {

    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", SSL? "8443" : "8080"));

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
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.option(ChannelOption.SO_BACKLOG, 1024);
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new HttpHelloWorldServerInitializer(sslCtx));

            Channel ch = b.bind(PORT).sync().channel();

            System.err.println("Open your web browser and navigate to " +
                    (SSL? "https" : "http") + "://127.0.0.1:" + PORT + '/');

            ch.closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

这是一个比较典型的服务端示例程序，在启动类 `main` 方法中负责创建Netty服务端，分别实例化了 2 个 `EventLoopGroup`，`bossGroup `和`workerGroup`。一个` EventLoopGroup` 实际就是一个 `EventLoop` 线程组，负责管理 `EventLoop` 的申请和释放。

`EventLoopGroup` 管理的线程数可以通过构造函数设置，如果没有设置，默认取`io.netty.eventLoopThreads`系统变量的值，如果该系统变量也没有指定，则设置为可用的 CPU 内核数 × 2 (`NettyRuntime.availableProcessors() * 2`)。

`bossGroup` 线程组实际就是 Acceptor 线程池，负责处理客户端的 TCP 连接请求，如果系统只有一个服务端端口需要监听，则建议 `bossGroup` 线程组线程数设置为 1。

`workerGroup` 没有指定线程数，由于我的机器是6核12线程，所以最后 `workerGroup` 的线程数为24。

```java
io.netty.channel.MultithreadEventLoopGroup#MultithreadEventLoopGroup(int, java.util.concurrent.Executor, java.lang.Object...)

private static final int DEFAULT_EVENT_LOOP_THREADS;

static {
    DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
        "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

    if (logger.isDebugEnabled()) {
        logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
    }
}
......
/**
     * @see MultithreadEventExecutorGroup#MultithreadEventExecutorGroup(int, Executor, Object...)
     */
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
```

### Netty 请求流程

![image-20210317174214132](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/image-20210317174214132.png)

大概的流程就是:

1. bossGroup接受来自客户端的请求
2. 通过轮询的方式绑定一个worker线程
3. worker线程通过**Selectors（选择器）**的方式处理通道事件 (JDK NIO)

![image-20210317174614198](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/image-20210317174614198.png)

当Netty 的 NioEventLoop 读取到消息之后，一直都是由NioEventLoop 调用用户的 ChannelHandler，期间不进行线程切换。既所有I/O操作和事件处理都在EventLoop所在的线程上执行，这种串行化的方式避免了多线程情况下的上下文切换及锁的竞争，从性能角度来看是最优的。

![image-20210317180646854](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/image-20210317180646854.png)

![image-20210318093617001](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/image-20210318093617001.png)

## JDK NIO 线程模型

在JDK 1.5后，引入了NIO，其中`Selector`是`Java NIO`核心组件中的一个。提供了IO的`多路复用模型`。多路复用模型使得可以使用**一个线程**来管理成千上万的连接，避免了线程上下文切换带来的开销，使得性能得到极大的提升。

## 总结

通过上面的分析，总结来说Netty是基于JDK NIO来实现其事件循环的，在Linux中是通过epoll。当Netty 的 NioEventLoop 读取到消息之后，一直都是由NioEventLoop 调用用户的 ChannelHandler，期间不进行线程切换。既所有I/O操作和事件处理都在EventLoop所在的线程上执行。

在上文提到的例子中，通过`Thread.sleep`的调用，阻塞Handler的执行线程，而由于Handler和EventLoop其实是同一个线程，这就造成了EventLoop的线程也阻塞了。

但是在Swoole中，其[睡眠函数](https://wiki.swoole.com/wiki/page/992.html)通过替换底层的sleep函数，并不会阻塞PHP进程。

> 最新的`4.2.0`版本增加了对`sleep`函数的`Hook`，底层替换了`sleep`、`usleep`、`time_nanosleep`、`time_sleep_until`四个函数。
>
> 当调用这些睡眠函数时会自动切换为协程定时器调度。不会阻塞进程。

## 结论
**在EventLoop这样的Reactor模式下，一定不要使用阻塞的sleep方式。**