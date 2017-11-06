---
layout: post
comments: false
categories: "dubbo"
date:   2017-10-02 00:00:54
title: Dubbo源码学习（四）之理论准备
---

<div id="toc"></div>

在调试源代码之前，我们需要掌握Dubbo中的一些理论和设计理念。Dubbo给开发者提供的文档[Dubbo开发者文档](https://dubbo.gitbooks.io/dubbo-dev-book/implementation.html) 讲的已经很细致，这篇文章只是记录下自己看完部分文档的理解，做一下复述。

## 一次简单的远程服务调用
参考[Dubbo源码学习（一）之Dubbo Demo](/2017/09/30/dubbo-demo/)中的代码，其实就是实现了Consumer对Provider代码的调用，但是从代码的角度是看不到RPC调用的逻辑的，因为底层框架给你做了这些。

那么当Consumer要调用Provider的服务时(如果只是点对点的方式)，需要做什么呢，简单的理解是:

- Consumer拿到定义的接口demoService，底层框架判断需要发送RPC请求到Provider，即发送请求给指定的URL

- Provider接收到请求后，需要路由到相应DemoServiceImpl进行处理

- Provider处理结束后，将请求返回给Consumer

虽然点对点调用的步骤看起来简单，但对于底层Dubbo的框架来说，要做的事情还是很多的。我们来看一看Dubbo里的粗略的代码结构:

{% include image.html url="/static/img/java/dubbo_rpc_invoke.jpg" description="Dubbo Provider Consumer" width="600px" inline="true" %}

这里有一个Dubbo 领域模型中非常重要的一个概念Invoker。在Consumer端，Invoker实现了真正的远程服务调用。而在服务端，收到到一个请求后，会找到对应的Exporter实例，并调用它所对应的AbstractProxyInvoker实例，从而真正调用了服务提供者的代码。

## SPI
SPI全称为(Service Provider Interface)，是JDK内置的一种服务提供发现机制。假设有一个接口HelloInterface:
```
package spi;

public interface HelloInterface {
    public void sayHello();
}
```

有两个实现TextHello与ImageHello:
```
package spi.impl;

import spi.HelloInterface;

public class ImageHello implements HelloInterface {
    @Override
    public void sayHello() {
        System.out.println("Image Hello");
    }
}
```

以及
```
package spi.impl;

import spi.HelloInterface;

public class TextHello implements HelloInterface{

    @Override
    public void sayHello() {
        System.out.println("Text Hello");
    }
}
```

在代码中我们具体要使用哪个实现类呢，是否可以动态决定某个类呢，有了SPI我们就可以，在META-INF/services/spi.HelloInterface添加实现类`spi.impl.ImageHello`。而后在我们的代码中利用ServiceLoader就可以在不知道ImageHello的情况下实例化ImageHello了:


```
ServiceLoader<HelloInterface> loaders = ServiceLoader.load(HelloInterface.class);

for(HelloInterface in : loaders) {
    in.sayHello();
}
```

在Dubbo中，没有使用ServiceLoader，而是自定义了dubbo-common/src/main/java/com/alibaba/dubbo/common/extension/ExtensionLoader.java来加载实现类，其与@SPI注解(dubbo-common/src/main/java/com/alibaba/dubbo/common/extension/SPI.java)配合使用。

Dubbo中的扩展实现加载可以在以下的目录中找到：

- META-INF/dubbo/internal/

- META-INF/dubbo/

- META-INF/services/

如果一个接口又多个实现，如何选择合适的实现呢，Dubbo中也提供@Adaptive注解(dubbo-common/src/main/java/com/alibaba/dubbo/common/extension/Adaptive.java)，duboo的使用者可以自定义Adaptive类来定义选择接口实现的规则。

## Netty

百度百科的定义： Netty提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发高性能、高可靠性的网络服务器和客户端程序。

Netty框架的底层是Java NIO。而Java NIO是怎么来做事件驱动的呢，通常是使用I/O多路复用。I/O多路复用的目的是为了使得一个线程能够操作多个文件描述符I/O操作，从而减少或避免I/O阻塞的时间的同时减少线程的数量。

关于Netty的使用，主要是要掌握其一个概念是其ChannelPipeline，其可以看做一个ChannelHandler的链表。而每个ChannelHandler处理各种读写事件，进而进行读写数据/编解码等操作。下面来看一个简单的EchoServer的实现:

```
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import java.net.InetSocketAddress;

public class EchoServer {

    private final int port;

    public EchoServer(int port) {
        this.port = port;
    }

    public void start() throws Exception {
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(group)
                    .channel(NioServerSocketChannel.class)
                    .localAddress(new InetSocketAddress(port))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new EchoServerHandler());
                        }
                    });
            ChannelFuture future = bootstrap.bind().sync();
            System.out.println(EchoServer.class.getName() + " start and listen on " + future.channel().localAddress());
            future.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully().sync();
        }
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            System.out.println("Usage: " + EchoServer.class.getSimpleName() + " <port>");
            System.exit(1);
        }
        int port = Integer.parseInt(args[0]);
        new EchoServer(port).start();
    }
}
```

而EchoServerHandler为

```
public class EchoServerHandler extends ChannelInboundHandlerAdapter{

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println("Server received: " + msg);
        ctx.write(msg);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER)
                .addListener(ChannelFutureListener.CLOSE);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

这个实现看起来可能也不是特别简单，但是如果了解Java NIO的同学看到这段代码，应该是与简单的评价的。

我们将EchoServer启动在8080端口，通过`telnet 127.0.0.1 8080`就连接上服务器了。输入`nihao`回车，就可以收到服务器的回文了。

当然Netty的强大不仅仅是其能够较简便的编写Java NIO的代码，其还做了很多其他的工作，包括但不限于：

- 方便的解决TCP的粘包/拆包问题

- 方便的使用Protobuf等进行序列化传输

- 提供了大量的使用的ChannelHander实现

- 优化了NIO的ByteBuffer

- Netty的线程模型被精心的设计，即提升了框架的并发性能，又能在很大程度上避免锁，局部实现了无锁化设计

- 解决了臭名昭著的epoll问题

- 优雅退出


## 参考资料

- [跟我学Dubbo系列之Java SPI机制简介](http://www.jianshu.com/p/46aa69643c97)

- [Netty系列之Netty可靠性分析](http://www.infoq.com/cn/articles/netty-reliability)


<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
