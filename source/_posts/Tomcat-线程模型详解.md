---
title: Tomcat 线程模型详解
date: 2020-08-30 12:26:10
toc: true
tags:
 - Reactor
 - NIO
categories: Tomcat
---

Tomcat 作为最常见的 Servlet 容器，在 6.x 版本就支持了 NIO 模式的 Connector 和 Reactor 模式相比有什么特殊之处吗？首先来看下面的表格：

|                                 | Java Nio Connector NIO | Java Nio2 Connector NIO2 | APR/native Connector APR |
| :------------------------------ | :--------------------: | :----------------------: | ------------------------ |
| Classname                       |  `Http11NioProtocol`   |   `Http11Nio2Protocol`   | `Http11AprProtocol`      |
| Tomcat Version                  |      since 6.0.x       |       since 8.0.x        | since 5.5.x              |
| Support Polling                 |          YES           |           YES            | YES                      |
| Polling Size                    |    `maxConnections`    |     `maxConnections`     | `maxConnections`         |
| Read Request Headers            |      Non Blocking      |       Non Blocking       | Non Blocking             |
| Read Request Body               |        Blocking        |         Blocking         | Blocking                 |
| Write Response Headers and Body |        Blocking        |         Blocking         | Blocking                 |
| Wait for next Request           |      Non Blocking      |       Non Blocking       | Non Blocking             |
| SSL Support                     |  Java SSL or OpenSSL   |   Java SSL or OpenSSL    | OpenSSL                  |
| SSL Handshake                   |      Non blocking      |       Non blocking       | Blocking                 |
| Max Connections                 |    `maxConnections`    |     `maxConnections`     | `maxConnections`         |

<!-- more -->

这是从 [Apache Tomcat 9 Configuration Reference-Connector Comparison](https://tomcat.apache.org/tomcat-9.0-doc/config/http.html#Connector_Comparison) 中得到的多种模式下的 Connector 的对比：

- NIO 就是我们熟悉的同步非阻塞 I/O 模型

- NIO2 指的是 AIO，即异步 I/O 模型，其实 Netty 已经抛弃 AIO，至于原因根据 [ISSUE-NIO.2 support](https://github.com/netty/netty/issues/2515) 介绍主要还是 Linux 下技术不够成熟以及复杂度的考虑

- ARP 实际上还是 NIO 模式，只不过它是基于 ARP（Apache Portable Runtime Libraries，由 C 写的 Apache 可移植运行时库）实现以提高性能。我们在启动 Tomcat 经常能看到 *The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path:* 提示就是指需要我们安装指定的环境以便正常启动 ARP 模式，当然我们现在常见使用的还是 NIO 模式，所以可以忽略该条提示

  

我们主要关心的是 NIO 模式，而从表格中可以知道比较重要的点：

- Read Request Body/Write Response Headers and Body 是阻塞的
- Read Request Headers/Wait for next Request 是非阻塞的

既然使用 NIO 模式为什么还要阻塞的读写数据呢？因为 Tomcat 是 Servlet 容器，Read Request Body/Write Response Headers and Body 是由 ServletInputStream 和 ServletOutputStream 定义的，而在 Servlet3.1 之前的规范中 ServletInputStream 和 ServletOutputStream 是**阻塞**的。

## Tomcat 线程模型

下图是 Tomcat 的整体线程模型：

![Tomcat线程模型](https://zzcoder.oss-cn-hangzhou.aliyuncs.com/Tomcat线程模型.png)

- Acceptor：**单线程阻塞**监听连接事件，和 Multi-Reactor 模型的 Main Reactor + Acceptor 的作用相似

- Poller：Acceptor 接收到的连接事件将通过 PollEvent 事件传递到 Poller 线程池处理，本质上就是将该 Socket 后续的读写事件交由 Poller 线程中的 Selector 进行监听并分派。Poller 线程池的线程池数为 CPU * 2，和 Multi-Reactor 模型的 Sub Reactor 的作用类似

  > 在 Multi-Reactor 模型中，读写 I/O 也是在 Sub Reactor 中执行的，而在 Tomcat 中仅有 Http 请求头，请求行是在 Poller 中执行的

- Worker（Servlet）：Poller 在解析完 Http 请求行，请求头后会将请求交由 Worker 业务线程池进行处理，默认情况下会创建最大线程数为 200 的线程池。当用户通过 ServletInputStream.read/write 读取 Request Body 或写入时将会向 Block Poller 注册监听读写事件，和 Multi-Reactor 模型的 Handler 的作用类似

- Block Poller：这是 Multi-Reactor 模型没有的角色，单线程的监听向其注册的 Socket 的读写事件，当事件发生后将交由 Worker 线程阻塞读写。它主要是结合 Worker 线程实现基于 NIO 的模拟阻塞读写，在后文会通过源码进行解析

Servlet 规范导致 Tomcat 的线程模型和 Reactor 模式具有一定差距，在 Multi-Reactor 中，I/O 读写（Sub-Reactor 线程执行）和业务阻塞是分离的（Handler 所在业务线程执行），而 Tomcat 的 I/O 读写和业务阻塞均在同个 Worker 线程（也可以称为 Servlet 线程），这在业务阻塞时长比较大的情况下具有比较大的影响

## 源码解析

### 阻塞读取Request Body

Tomcat 的 Request Body 的读取是阻塞式的，调用 `ServletInputStream.read` 时会触发读取 Request Body，在 Tomcat 中的实现是 `CoyoteInputStream.read`。其内部会触发 ActiveFilter 链的调用

```java
# org.apache.coyote.http11.Http11InputBuffer#doRead
@Override
public int doRead(ApplicationBufferHandler handler) throws IOException {

    if (lastActiveFilter == -1)
        return inputStreamInputBuffer.doRead(handler);
    else
        return activeFilters[lastActiveFilter].doRead(handler);

}
```

默认处理非 chunked 请求的 Filter 是 IdentityInputFilter

> chunked 请求：HTTP1.1 长连接模式下，为了能正确复用 TCP 连接，需要通过 Content-Length 或 chunked 来标志响应体大小，但诸如网络文件等要想准确获取长度需要生成完成才能确认，所以通过 chunked 分块编码来提高性能

```java
# org.apache.coyote.http11.filters.IdentityInputFilter#doRead
@Override
public int doRead(ApplicationBufferHandler handler) throws IOException {

    int result = -1;

    if (contentLength >= 0) {
        if (remaining > 0) {
            int nRead = buffer.doRead(handler);
            if (nRead > remaining) {
                // The chunk is longer than the number of bytes remaining
                // in the body; changing the chunk length to the number
                // of bytes remaining
                handler.getByteBuffer().limit(handler.getByteBuffer().position() + (int) remaining);
                result = (int) remaining;
            } else {
                result = nRead;
            }
            if (nRead > 0) {
                remaining = remaining - nRead;
            }
        } else {
            // No more bytes left to be read : return -1 and clear the
            // buffer
            if (handler.getByteBuffer() != null) {
                handler.getByteBuffer().position(0).limit(0);
            }
            result = -1;
        }
    }

    return result;

}
```

> 这里插一句和线程模型无关的知识点，即如何判断 Http 请求已读取完成？根据上面的逻辑可以看出，Tomcat 在读取每个 Http 请求的时候是根据 Header 中的 Content-Length 来读取字节的，每次读取一定字节时会更新 `remaining = remaining - nRead`，当 remaining = 0 时代表当前 Http 请求的 Request Body 已读取完成，因此当 CoyoteInputStream.read 未返回 -1 时均代表当前 Http 请求未接收完整

其中 `buffer.doRead(handler)` 调用的实际上是 `Http11InputBuffer.SocketInputBuffer#doRead`

```java
private class SocketInputBuffer implements InputBuffer {

    @Override
    public int doRead(ApplicationBufferHandler handler) throws IOException {

        if (byteBuffer.position() >= byteBuffer.limit()) {
            // The application is reading the HTTP request body which is
            // always a blocking operation.
            if (!fill(true))
                return -1;
        }

        int length = byteBuffer.remaining();
        handler.setByteBuffer(byteBuffer.duplicate());
        byteBuffer.position(byteBuffer.limit());

        return length;
    }
}
```

这里注意在调用 `fill` 方法时会传入 true，代表**阻塞的获取 Http request body**。去除无关的调用链，最终读取的关键位置在 `NioSelectorPool#read`，该方法通过 NIO 模拟阻塞

```java
# org.apache.tomcat.util.net.NioBlockingSelector#read
public int read(ByteBuffer buf, NioChannel socket, long readTimeout) throws IOException {
    SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());
    if (key == null) {
        throw new IOException(sm.getString("nioBlockingSelector.keyNotRegistered"));
    }
    KeyReference reference = keyReferenceStack.pop();
    if (reference == null) {
        reference = new KeyReference();
    }
    NioSocketWrapper att = (NioSocketWrapper) key.attachment();
    int read = 0;
    boolean timedout = false;
    int keycount = 1; //assume we can read
    long time = System.currentTimeMillis(); //start the timeout timer
    try {
        while(!timedout) {
            if (keycount > 0) { //only read if we were registered for a read
                read = socket.read(buf);
                if (read != 0) {
                    break;
                }
            }
            try {
                if ( att.getReadLatch()==null || att.getReadLatch().getCount()==0) att.startReadLatch(1);
                poller.add(att,SelectionKey.OP_READ, reference);
                if (readTimeout < 0) {
                    att.awaitReadLatch(Long.MAX_VALUE, TimeUnit.MILLISECONDS);
                } else {
                    att.awaitReadLatch(readTimeout, TimeUnit.MILLISECONDS);
                }
            } catch (InterruptedException ignore) {
                // Ignore
            }
            if ( att.getReadLatch()!=null && att.getReadLatch().getCount()> 0) {
                //we got interrupted, but we haven't received notification from the poller.
                keycount = 0;
            }else {
                //latch countdown has happened
                keycount = 1;
                att.resetReadLatch();
            }
            if (readTimeout >= 0 && (keycount == 0))
                timedout = (System.currentTimeMillis() - time) >= readTimeout;
        } //while
        if (timedout)
            throw new SocketTimeoutException();
    } finally {
        poller.remove(att,SelectionKey.OP_READ);
        if (timedout && reference.key!=null) {
            poller.cancelKey(reference.key);
        }
        reference.key = null;
        keyReferenceStack.push(reference);
    }
    return read;
}
```

上述代码逻辑为：

- 通过 `socket.read(buf)` 尝试进行读取一次，若读取成功则返回

- 若当前不可读，则通过 `poller.add(att,SelectionKey.OP_READ, reference)` 向 BlockPoller 注册 OP_READ 事件，然后通过 CountDownLatch 将当前 Worker 线程阻塞

- 由 BlockPoller 单线程监听 OP_READ 事件，在可读时唤醒 Worker 线程，由 Worker 线程进行读取

  ```java
  # org.apache.tomcat.util.net.NioBlockingSelector.BlockPoller#run
  Iterator<SelectionKey> iterator = keyCount > 0 ? selector.selectedKeys().iterator() : null;
  
  // Walk through the collection of ready keys and dispatch
  // any active event.
  while (run && iterator != null && iterator.hasNext()) {
      SelectionKey sk = iterator.next();
      NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();
      try {
          iterator.remove();
          sk.interestOps(sk.interestOps() & (~sk.readyOps()));
          if ( sk.isReadable() ) {
              countDown(attachment.getReadLatch());
          }
          if (sk.isWritable()) {
              countDown(attachment.getWriteLatch());
          }
      }catch (CancelledKeyException ckx) {
          sk.cancel();
          countDown(attachment.getReadLatch());
          countDown(attachment.getWriteLatch());
      }
  }//while
  ```

所以实际上 Request Body 的读取是由 Worker 线程阻塞读取完成的，至于 BlockPoller 其实类似 Poller，只不过它的 Selector 单独处理 Request Body 的读写事件

### 阻塞写入数据

Tomcat 的写入同样是阻塞式的，由 `ServletOutputStream.write` 时会触发，在 Tomcat 的实现类为 `CoyoteOutputStream#write`，其内部同样有个 OutputFilter

```java
# org.apache.coyote.http11.Http11OutputBuffer#flush
@Override
public void flush() throws IOException {
    if (lastActiveFilter == -1) {
        outputStreamOutputBuffer.flush();
    } else {
        activeFilters[lastActiveFilter].flush();
    }
}
```

默认情况下调用的是 `IdentityOutputFilter.write`

```java
# org.apache.coyote.http11.filters.IdentityOutputFilter#doWrite
@Override
public int doWrite(ByteBuffer chunk) throws IOException {

    int result = -1;

    if (contentLength >= 0) {
        if (remaining > 0) {
            result = chunk.remaining();
            if (result > remaining) {
                // The chunk is longer than the number of bytes remaining
                // in the body; changing the chunk length to the number
                // of bytes remaining
                chunk.limit(chunk.position() + (int) remaining);
                result = (int) remaining;
                remaining = 0;
            } else {
                remaining = remaining - result;
            }
            buffer.doWrite(chunk);
        } else {
            // No more bytes left to be written : return -1 and clear the
            // buffer
            chunk.position(0);
            chunk.limit(0);
            result = -1;
        }
    } else {
        // If no content length was set, just write the bytes
        result = chunk.remaining();
        buffer.doWrite(chunk);
        result -= chunk.remaining();
    }

    return result;

}
```

> 这里同样插一句和线程模型无关的知识点，即如何判断 Http 请求已写入完成？根据上面的逻辑可以看出，Tomcat 在写入每个 Http 请求的时候是根据 Response Header 中的 Content-Length 来写入字节的，每次写入一定字节时会更新 `remaining = remaining - nRead`，当 remaining = 0 时代表当前 Http 请求的 Reponse 已写入完成

其中 `buffer.doWrite(chunk);` 调用的实际上是 `Http11OutputBuffer.SocketOutputBuffer#doWrite`

```java
# org.apache.coyote.http11.Http11OutputBuffer.SocketOutputBuffer#doWrite
@Override
public int doWrite(ByteBuffer chunk) throws IOException {
    try {
        int len = chunk.remaining();
        socketWrapper.write(isBlocking(), chunk);
        len -= chunk.remaining();
        byteCount += len;
        return len;
    } catch (IOException ioe) {
        response.action(ActionCode.CLOSE_NOW, ioe);
        // Re-throw
        throw ioe;
    }
}
```

这里的 `socketWrapper.write(isBlocking(), chunk);` 会使用**阻塞的方式进行写入**，最终写入的核心方法为 `NioSelectorPool#write` 

```java
# org.apache.tomcat.util.net.NioBlockingSelector#write
public int write(ByteBuffer buf, NioChannel socket, long writeTimeout)
        throws IOException {
    SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());
    if (key == null) {
        throw new IOException(sm.getString("nioBlockingSelector.keyNotRegistered"));
    }
    KeyReference reference = keyReferenceStack.pop();
    if (reference == null) {
        reference = new KeyReference();
    }
    NioSocketWrapper att = (NioSocketWrapper) key.attachment();
    int written = 0;
    boolean timedout = false;
    int keycount = 1; //assume we can write
    long time = System.currentTimeMillis(); //start the timeout timer
    try {
        while ( (!timedout) && buf.hasRemaining()) {
            if (keycount > 0) { //only write if we were registered for a write
                int cnt = socket.write(buf); //write the data
                if (cnt == -1)
                    throw new EOFException();
                written += cnt;
                if (cnt > 0) {
                    time = System.currentTimeMillis(); //reset our timeout timer
                    continue; //we successfully wrote, try again without a selector
                }
            }
            try {
                if ( att.getWriteLatch()==null || att.getWriteLatch().getCount()==0) att.startWriteLatch(1);
                poller.add(att,SelectionKey.OP_WRITE,reference);
                if (writeTimeout < 0) {
                    att.awaitWriteLatch(Long.MAX_VALUE,TimeUnit.MILLISECONDS);
                } else {
                    att.awaitWriteLatch(writeTimeout,TimeUnit.MILLISECONDS);
                }
            } catch (InterruptedException ignore) {
                // Ignore
            }
            if ( att.getWriteLatch()!=null && att.getWriteLatch().getCount()> 0) {
                //we got interrupted, but we haven't received notification from the poller.
                keycount = 0;
            }else {
                //latch countdown has happened
                keycount = 1;
                att.resetWriteLatch();
            }

            if (writeTimeout > 0 && (keycount == 0))
                timedout = (System.currentTimeMillis() - time) >= writeTimeout;
        } //while
        if (timedout)
            throw new SocketTimeoutException();
    } finally {
        poller.remove(att,SelectionKey.OP_WRITE);
        if (timedout && reference.key!=null) {
            poller.cancelKey(reference.key);
        }
        reference.key = null;
        keyReferenceStack.push(reference);
    }
    return written;
}
```

上述代码逻辑为：

- 通过 `socket.write(buf)` 尝试进行写入一次，若读取成功则返回

- 若当前不可写，则通过 `poller.add(att,SelectionKey.OP_WRITE, reference)` 向 BlockPoller 注册 OP_WRITE 写入事件，然后通过 CountDownLatch 将当前 Worker 线程阻塞

- 由 BlockPoller 单线程监听 OP_WRITE 事件，在可读时唤醒 Worker 线程，由 Worker 线程进行读取

  ```java
  # org.apache.tomcat.util.net.NioBlockingSelector.BlockPoller#run
  Iterator<SelectionKey> iterator = keyCount > 0 ? selector.selectedKeys().iterator() : null;
  
  // Walk through the collection of ready keys and dispatch
  // any active event.
  while (run && iterator != null && iterator.hasNext()) {
      SelectionKey sk = iterator.next();
      NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();
      try {
          iterator.remove();
          sk.interestOps(sk.interestOps() & (~sk.readyOps()));
          if ( sk.isReadable() ) {
              countDown(attachment.getReadLatch());
          }
          if (sk.isWritable()) {
              countDown(attachment.getWriteLatch());
          }
      }catch (CancelledKeyException ckx) {
          sk.cancel();
          countDown(attachment.getReadLatch());
          countDown(attachment.getWriteLatch());
      }
  }//while
  ```

所以写数据和读取 Request Body 是非常相似的过程

## Servlet 异步和非阻塞

上面提到了 Tomcat 的 I/O 读写和业务阻塞均在同个 Servlet 线程，这在业务阻塞时间较长时很容易出现 Tomcat 的 Worker 线程池任务堆积（默认最大 200 个线程），无法支撑高并发场景。而在 Servlet3.0 开始支持了 Servlet 异步接口：

```java
AsyncContext asyncContext = request.startAsync();

asyncContext.start(() -> {
    process();
    asyncContext.complete();
});
```

通过异步模式可以将 I/O 读写和业务阻塞分离以达到复用 Worker 线程的作用。但是此时 InputStream 和 OutputStream 的 I/O 操作仍然是阻塞的，因此无论将 I/O 操作放在 Servlet 线程还是自定义线程，这总是会导致对应线程无用的等待，因此在 Servlet3.1 开始支持非阻塞 I/O，通过非阻塞 I/O 才解决了读写时线程资源浪费的问题。

<img src="https://zzcoder.oss-cn-hangzhou.aliyuncs.com/image-20200924185239968.png" alt="servlet-api" style="zoom:50%;" />

结合异步模式和非阻塞 I/O，Tomcat 就真正**既解决了“一个连接一个请求”的问题，又解决高并发下业务阻塞会影响到读写线程的问题**

```java
AsyncContext asyncContext = request.startAsync();
ServletInputStream inputStream = request.getInputStream();
inputStream.setReadListener(new ReadListener() {
    @Override
    public void onDataAvailable() throws IOException {
    }

    @Override
    public void onAllDataRead() throws IOException {
        executor.execute(() -> {
            process();
            try {
                asyncContext.getResponse().getWriter().write("xxx");
            } catch (IOException e) {
                e.printStackTrace();
            }
            asyncContext.complete();
        });
    }

    @Override
    public void onError(Throwable t) {
        asyncContext.complete();
    }
});
```

根据 [Apache Tomcat Versions](http://tomcat.apache.org/whichversion.html) 的介绍，在 Tomcat8.x 中已经支持了 Servlet3.1

![tomcat-version](https://zzcoder.oss-cn-hangzhou.aliyuncs.com/image-20200830112515693.png)

## Tomcat or Netty

Tomcat 属于 Servlet 容器，Netty 属于异步网络编程框架，但它们都能作为 web 容器。上面介绍了 Tomcat 的线程模型，那么和 Netty 相比，谁更适合做 Web 服务器呢？如果针对于 Http 协议，那么 Tomcat 是毫无疑问的，它对于 Http 有着更完善的支持，比如 Chunk 分块传输。而 Netty 是基于传输层的，所以可以支持更多的应用层协议。

那么 Tomcat 和 Netty 的线程模型谁更有优势呢？其实没有本质上的区别，Tomcat 的 NIO 模式和 Netty 的 Multi-Reactor 模式都解决了“一个连接一个线程”的问题，且默认模式下业务处理和 I/O 读写都是在同个线程

- Netty 的默认模型并不是 Multi-Reactor 模型，业务处理默认情况下是和 Sub Reactor 处于同个线程
- Tomcat 的读写均在 Worker 线程

所以容器可优化业务阻塞时间较长场景的方案是：采用业务阻塞和 I/O 阻塞分离的方式实现（异步 Servlet 或 Multi-Reactor 模式）。但这已经是容器优化的极限了，它仍然无法解决大量业务线程阻塞时间过长时仍然会导致大量的线程产生的问题。因此要想进一步提高性能，不能只看容器，而需要将业务阻塞（比如缓存，数据库，RPC 等）操作变为非阻塞操作以提高线程复用程度

## 拓展

Http Request Body 是由用户在 Servlet 线程进行阻塞读取的，假如用户不主动读取上传的问题会出现什么情况呢，比如上传一个 100M 的文件 Tomcat 会自动接收吗？

- 从 TCP 层面来看，如果数据一直不被接收，那么 Recv Buffer 会很快被占满，根据流量控制机制发送方将不会再发送数据，所以是不会自动接收所有数据的

- 从 Tomcat 层面来看，我们可以观察 `IdentityInputFilter#end` 方法

  ```java
  # org.apache.coyote.http11.filters.IdentityInputFilter#end
  @Override
  public long end() throws IOException {
  
      final boolean maxSwallowSizeExceeded = (maxSwallowSize > -1 && remaining > maxSwallowSize);
      long swallowed = 0;
  
      // Consume extra bytes.
      while (remaining > 0) {
  
          int nread = buffer.doRead(this);
          tempRead = null;
          if (nread > 0 ) {
              swallowed += nread;
              remaining = remaining - nread;
              if (maxSwallowSizeExceeded && swallowed > maxSwallowSize) {
                  // Note: We do not fail early so the client has a chance to
                  // read the response before the connection is closed. See:
                  // https://httpd.apache.org/docs/2.0/misc/fin_wait_2.html#appendix
                  throw new IOException(sm.getString("inputFilter.maxSwallow"));
              }
          } else { // errors are handled higher up.
              remaining = 0;
          }
      }
  
      // If too many bytes were read, return the amount.
      return -remaining;
  
  }
  ```

  在请求结束后 Tomcat 会自动调用该方法，其代码如果缓存中存在剩余的字节，Tomcat 会自动读取，但是会有限制，默认只读取 2M(由 maxSwallowSize 确定)，如果请求体数据大于 2M，Tomcat 会抛出异常并关闭连接。

  > The maximum number of request body bytes (excluding transfer encoding overhead) that will be swallowed by Tomcat for an aborted upload. An aborted upload is when Tomcat knows that the request body is going to be ignored but the client still sends it. If Tomcat does not swallow the body the client is unlikely to see the response. If not specified the default of 2097152 (2 megabytes) will be used. A value of less than zero indicates that no limit should be enforced.
  >
  > ​                                                       [The HTTP Connector](https://tomcat.apache.org/tomcat-8.0-doc/config/http.html)

