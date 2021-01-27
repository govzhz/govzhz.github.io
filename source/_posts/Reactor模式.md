---
title: Reactor模式
date: 2020-08-23 16:41:25
toc: true
tags:
 - Reactor
 - NIO
categories: Netty
---

标准 I/O 会存在两个阶段：（以 TCP Socket 举例）

- 数据在内核态和用户态的复制：TCP 协议栈维护着 Send Buffer（发送缓冲区）和 Recv Buffer（接收缓冲区），因此 read/write 都只是将用户态数据复制到内核态的 Socket Buffer(Send/Recv Buffer)。而 Socket Buffer 和网卡之间会通过 DMA 进行数据传输（不占用 CPU）
- 等待数据就绪：对于 read 操作来说，就绪是指 Recv Buffer 没有可读的数据；而对于 write 操作来说，就绪是指 Send Buffer 已满无法写入

BIO 称为同步阻塞 I/O，它在上述的两个阶段均会阻塞。因此 **BIO 必须为每个 TCP 连接创建新线程**并阻塞等待其可读或可写。服务端若想支持大量客户端连接，在 BIO 的前提下使用多线程来解决是必然的事情，下面用一个简单的例子展示：

<!-- more -->

```java
ExecutorService executor = Executors.newFixedThreadPool(100);
try(ServerSocket serverSocket = new ServerSocket(8080)) {
    while (true) {
        Socket socket = serverSocket.accept();
        executor.execute(() -> {
            try(BufferedReader reader = new BufferedReader(new InputStreamReader(socket.getInputStream()))) {
                decode();
                process();
                encode();
                write();
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }
} catch (IOException e) {
    e.printStackTrace();
}
```

在连接数不多的情况下，上述例子并无不妥。但是在高并发下呢？关键点在于线程

- 每个线程占用 256K～1M 的空间，高并发下存在大量的连接会占用大量的内存
- 过多的线程会使得 CPU 调度成本很高，CPU 将会疲于调度导致 CPU sy 使用率特别高

因此我们回过头来考虑一个很重要的问题，**有必要为每个连接创建一个线程吗，这些阻塞等待I/O 的线程有意义吗**？假如大部分连接都不会同时处于活跃状态（即此时连接并无读写事件发生），那么线程实际上在做无用的等待；而假如并发活跃连接数非常大，那么会存在大量的线程执行 write/read，此时会存在大量用户态数据和内核态的 Socket Buffer 互相复制的操作，该操作需要消耗 CPU。既然是依赖于 CPU，那么创建大量线程并没有正向作用，反而应该选择和 CPU 核心数类似的线程数。因此**不管从哪个角度来说，为每个新连接都创建一个线程去阻塞等待读写都不是一个好选择**。

那么理想的处理模型是怎么样的呢？关键的点是将本该由线程去阻塞等待可读可写变成异步以及回调处理，这就依赖于两个技术点：

- `NIO`：作为同步非阻塞 I/O，它在等待就绪阶段是非阻塞的，在内核态和用户态的复制阶段是同步阻塞的。通过 NIO 能保证线程不会做无用的等待

- `I/O 多路复用`：只要应用通过系统调用通知内核所需监听的连接以及事件，内核能够在监听连接可读写时回调通知应用程序

  > 注意，这里提到的 I/O 多路复用和 HTTP2 中的 Multiplexing 并不是一个概念，HTTP2 中的 Multiplexing 代表同个 TCP 连接可以复用来支持多个 HTTP 请求并发请求，而 I/O 多路复用(也可以理解为 I/O 多路传输) 代表使用单循环来处理多个连接，比如 select, epoll

## I/O多路复用

Linux 为 I/O 多路复用提供了多个系统调用，如 select，poll，epoll。以 Linux 提供的 select 系统调用举例：

```c++
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

构建 `select` 需要传入三个 fd(文件描述符) 数组分别指向内核需要对可读、可写和异常等事件进行监听的对应的描述符集合。调用 select 函数后，就会阻塞的等待事件到来。当 select 返回时应用需要遍历所有的 fd 来寻找发生事件的 socket。

select 有一些比较明显的缺点：

- 应用每次调用 select 都需要传入所有的 fd（**最大个数不能超过1024个**），这会导致这三个数组**频繁的从用户态复制到内核态**
- 内核每次都需要遍历所有的 fd 去检查就绪的事件
- **select 返回后应用程序需要遍历 fd 数组获取真正发生事件的 fd**

而后续出现的 `epoll` 解决了上面几个痛点，它会在**内核中维护一个数据表以避免每次都需要传入 fd 数组**（减少了数据复制），同时也**去除了 fd 个数的限制**，并且 **epoll 只会返回被触发的事件对应的 fd** 从而避免应用程序去做额外的扫描和过滤。

但是不管是哪种实现方式，I/O 多路复用的基本使用方式都是：**注册 -> 监听事件 -> 回调处理**，因为只有应用先向内核注册感兴趣的事件以及 socket，内核才会在事件发生时回调应用。

## Reactor 模式

Reactor 模式是基于事件驱动的网络编程模式，能用少量的线程支持大量的连接，它也可以理解为 NIO 和 I/O 多路复用的最佳实践，在诸如 Netty 等网络编程框架已支持了 Reactor 的多种模式。单线程 Reactor 基本原理如下：

![SingleThreadReactor](https://zzcoder.oss-cn-hangzhou.aliyuncs.com/SingleThreadReactor.png)

上图存在几个角色：

- Reactor：可以理解为事件分派器，维护多路复用器，监听并分派事件到对应的 Handler
- Acceptor：处理连接事件，负责向多路复用器注册感兴趣事件
- Handler：处理读写就绪事件，包含 read, decode，process，encode, send 等处理，也可以向多路复用器注册感兴趣事件（比如写就绪）

通过单线程版 Reactor 模式可以使用单个线程就完成 Server 端的所有处理逻辑，而不再需要为每个连接创建一个新线程。但在实际应用中还需要做更多的优化：

- Handler 中的 read/write 这些操作由于是在接收到就绪事件后调用的，所以 I/O 操作大部分时间会使用 CPU 的，因此实际上可以构建和 CPU 核心数相同的线程数来提高 CPU 利用率
- Handler 中的 process 业务处理操作可能存在其他阻塞 I/O 操作，比如 DB，RPC 等，因此可以创建 Worker 线程池进行处理，其线程数可大于 CPU 核心数

因此优化后的 Reactor 模式将 read/write 操作交由其线程数等于 CPU 核心数的线程池处理，而业务处理逻辑交由单独的 Worker 线程池处理。而这就是多线程 Reactor 模式（有些地方也称为主从 Reactor 模式）的原理

![MultipleReactor](https://zzcoder.oss-cn-hangzhou.aliyuncs.com/MultipleReactor模式.png)

- Main Reactor 仍然使用单线程处理，只不过这次它只需要处理新连接就绪事件
- Sub Reactor 使用和 CPU 核心数相等的线程池负责读写事件的监听以及分配
- Handler 的 I/O 部分（实际上 decode/encode 也可以放在这里）和 Sub Reactor 处于同一线程
- Handler 的业务处理部分交由 Worker 线程池处理（线程数可大于 CPU 核心数）

其实还可以考虑一下，Handler 的业务处理所在的 Worker 线程池实际上会为每个请求分配一个线程，那么它和最初的 BIO + 多线程的模式看起来结果是一样的，所以多线程 Reactor 模式还有意义吗？答案是肯定的，Reactor 模式能承载更高的并发连接，因为大部分情况下并不是所有的连接都会产生读写事件的，可能 10K 的连接只有 1K 的连接并发产生了读写，那么处理线程数就从 10K 降到了 1K，因此 Reactor 模式才能称得上高性能 I/O 模型

## Netty 中的线程模型

Netty 支持上述的两种 Reactor 模式。首先是单线程 Reactor 模式

```java
NioEventLoopGroup mainReactor = new NioEventLoopGroup(1);
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(mainReactor)
    .channel(NioServerSocketChannel.class)
    .localAddress(new InetSocketAddress(8080))
    .childHandler(new ServerHandlerInitializer());
```

NioEventLoopGroup 实现了 ExecutorService，本身是一个线程池，所以设置 NioEventLoopGroup 线程数为 1，当 `bootstrap.group` 仅传入一个 group 时 Main Reactor 和 Sub Reactor 就会使用同个线程

```java
@Override
public ServerBootstrap group(EventLoopGroup group) {
    return group(group, group);
}
```

在 Sub Reactor 收到请求后会通过 ChannelHandler 链处理，默认情况下整个责任链传递过程都是同步的。而在多线程 Reactor 中的 Main Reactor 和 Sub Reactor 将会使用不同的 group

```java
NioEventLoopGroup mainReactor = new NioEventLoopGroup(1);
NioEventLoopGroup subReactor = new NioEventLoopGroup(); # 默认创建一个 CPU * 2 线程数的线程池
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(mainReactor, subReactor)
    .channel(NioServerSocketChannel.class)
    .localAddress(new InetSocketAddress(8080))
    .childHandler(new ServerHandlerInitializer());
```

此时其实不算真正的多线程 Reactor，因为 ChannelHandler 此时是和 Sub Reactor 使用同个线程的。因此需要用户在实现 ChannelHandler 时使用业务线程池进行处理或使用 Netty 提供的 EventExecutorGroup 来支持

> Netty 提供的 EventExecutorGroup 会将不同的 Channel 绑定到固定的线程，后续该连接的所有请求都会在同一线程处理，因此若是为了提供并发连接的响应速度使用该线程池是可以的，但若是同个连接并发的请求（比如 HTTP2 的多路复用）是无法解决的，这时候应使用自定义线程池解决

## 拓展

从 Reactor 模式对线程资源的复用也给我们了启示，在连接数比较大时应利用非阻塞 API 以及事件回调的方式来实现高性能，比如爬虫程序会产生大量的请求，可以实现基于 NIO 的异步 HttpClient 来复用线程，再比如类似 Dubbo 的 RPC 框架会维护大量的 TCP 长连接，因此这也可以通过 NIO 来复用线程。在使用 NIO 作为客户端的实践中过程中需要考虑的问题是如何找到响应对应的请求，比如 Dubbo 的实现：

1. 每个请求都会生成一个唯一的请求 ID（RID），然后将 Channel 保存在 DefaultFuture 中，并放入 Map 维护

2. 通过 NIO 非阻塞接口发送请求后，根据调用模式（同步/异步）来确定是否需要阻塞当前线程

   > 同步调用模式下，在获取到 ResponseFuture 后由**框架会自动调用 get 阻塞当前线程**；异步调用模式下 Dubbo 会将 ResponseFuture 放入 RpcContext（线程上下文）交由**用户自行调用**

3. 请求响应后将响应结果放入 DefaultFuture，然后唤醒阻塞线程（同步调用时）

