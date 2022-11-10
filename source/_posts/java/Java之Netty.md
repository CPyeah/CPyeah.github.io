---
title: Java之Netty.md
date: 2021-12-25 10:18:43
tags: 
- netty
- java
categories: java
---
## 介绍
Netty是一套支持NIO的客户端-服务器框架。
高性能、高并发。
支持异步通信。
他是使用Java编写的。
在Java领域中，他是IO界的老大，特别是网络IO。
很多项目都用它，比如Dubbo，RocketMQ，Cassandra等等。

在这片文章中，我们会学习到：
1. 常见的IO模型
2. Netty项目
3. Netty实战
4. Netty在我司生产中的应用

那我们开始吧！

<!--more-->

## 常见的IO模型

### BIO，阻塞IO
> 排队

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/PGtSwc.png)

当一个IO请求过来，客户端进入等待，服务端处理数据，并保持阻塞（其他的客户端访问不了）。

等到服务端处理完数据，返回给等待中的客户端。

很像我们排队买鸭子的过程（没错，我在南京）。一个一个来。

他的优点是，模型很简单，容易实现，且不容易出问题。
确定也很明显，在客户多了的时候，体验会很不好。

### NIO，非阻塞IO
> 轮询

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/F7rzu9.png)

客户端在执行了IO请求之后，会立刻返回结果，要不返回结果，要不还需等待。
如果还需要等待的话，客户会发起轮询，不断的请求服务端，直到返回结果。

比如我去买一个手抓饼，我下单了之后，需要等待，并且过一段时间，我会问一下老板，
我的手抓饼好了没，如果没有，再过一段时间，再问下老板，我的手抓饼好了没。
直到老板把手抓饼给我。

他的优势是它不会再阻塞了，不用排队等在那里。我可以干一些别的事情。

缺点也是有的，我需要不断的询问，这样也很消耗性能。
而且如果我需要的结果（手抓饼）已经准备好，但是我没有及时询问，
这种情况，客户端拿到数据的时间会晚于真实的数据时间。

### IO多路复用
> 事件处理

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/BqbARO.png)

IO多路复用，它是接受各种事件（比如连接、读取或写入、错误发生等)。
接受到各种事件消息后，记录下来，并返回一个描述符。

等到数据准备完成，再根据描述符找到是哪个客户端，并通知客户端来取数据。

打个比方，就像我们排队买奶茶，下单之后，拿到一个号码。
等到奶茶做好来之后，奶茶店会叫号，让我们去取奶茶。

它的优点是可以释放客户端，不需要阻塞。
因为仅仅是接受事件，一个线程就够了，不要很多线程来处理请求。

像Redis，Nginx都是使用的IO多路复用模型。

### AIO、异步IO模型
> 回调

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/FZRyq2.png)

这种IO模型很像JS中的AJAX，发起一个请求之后，继续往下之后后续逻辑。
等到数据返回之后，走到回调方法，并发数据结果带过来。

这种就像点外卖，下单之后，等到外卖准备好，直接把外卖送到家。

优点就是，非常高效，客户端有着最佳的性能。
缺点就是，模型结构复杂，不是所有操作系统都能支持。

## Netty项目结构
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/7qmm3O.png)

Netty是一个大而全的IO框架，它支持各种各样的协议。
对各种IO模型都有封装。
更友好的操作API。
更高效的性能。

接下来，我们自己看一下它。

### Netty IO 模型
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/CqK8RW.png)

Netty使用的是反应模型的一种，有主线程组和工作线程组。
主线程组负责处理各种连接，并分发任务给工作线程组。
工作线程组就负责处理各种输入输出请求。并执行定义好的pipeline。

### 三大核心

#### 1、ChannelBuffer 缓冲区
可以自定义的，多类型，API友好的缓冲区类型。

我们在使用IO的时候，都会使用到buffer，为解决速率不一致。
但是在操作系统层面和Java原生层面，只提供了ByteBuffer。字节缓冲区。
非常的单一，而且操作起来很不方便。

在Netty中，对缓冲区有了更好的封装。自动的装包和拆包。

而且ChannelBuffer支持zero-copy，这里的零拷贝，不是操作系统的零拷贝。
它的原理是在用户模式直接维护一个内核模式的buffer地址，直接进行操作。
不需要再进行内核模式切换，把buffer拷贝到用户模式里面来。

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/lkHtRv.png)


#### 2、通用API
对BIO和NIO的通用抽象。
支持不同的通讯协议（TCP/UDP）

如果我们使用Java原生的IO/NIO接口编写，
会相当复杂繁琐。

更糟糕的是，如果我们需要把系统从BIO升级到NIO，
基本上需要重新开发一边

而如果我们一开始使用Netty开发，
IO模型、通讯协议都可以的切换得更加顺滑

#### 3、事件模型 
> 基于拦截器链模式的事件模型

Netty使用pipeline，实现了一个结构清晰得事件模型。

每当一个Channel被创建的时候，就会同时创建一个ChannelPipeline，
并永久的绑定到这个Channel上。

一个Event事件被加入到，触发一个pipeline，其中很多的Handler执行操作。

我们还可以实现自己的Handler来处理自己的业务逻辑。
而不会破坏代码。

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/HgrHQm.png)

而对于我们开发Netty应用，我们只需要配置好Netty，
然后写好自己的业务逻辑Handler，并加入到Pipeline中。
其他的交给Netty就好了，非常的方便。

### 传输服务
> 处理基于流的传输

在基于TCP/IP的基于流的协议中，数据包都是基于字节流的。
所以在一个数据包中，可能不是一个完整的我们需要的数据包。

所以在数据包的处理上，就会有很多繁琐的工作内容。
拆包、装包

所以Netty提供了一个可以扩展的类`ByteToMessageDecoder`
里面提供很多开箱即用的编码、解码器。
初次之外，我们还可以编写自己的定制编码、解码器。

在TCP的世界中，所有的数据传输都是基于字节流的。
但是Netty提供了一种封装，
可以同时解决粘包、拆包的问题，还可以给转换成POJO对象。
这样解码器直接解出来的就是一个Java对象。更优雅。


### 协议支持

Netty实现了各种高级的传输协议。
比如：HTTP、SSL、WebSockets等等。
我们甚至可以编写自己的协议。

## Netty实战
接下来  我们简单写一个Netty的服务端和客户端的demo。
我们使用SpringBoot来快速架构项目。
使用Gradle构建
这里是使用的是Netty 4.1.77，比较稳定的版本。

```groovy
dependencies {
    implementation group: 'io.netty', name: 'netty-all', version: '4.1.77.Final'
    compileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.24'
    annotationProcessor 'org.projectlombok:lombok:1.18.24'
    implementation 'org.springframework.boot:spring-boot-starter'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

### 服务端
我们写一个Netty服务，主要是以下几步：
1. 构建一个启动器
2. 创建EventLoopGroup
3. 构建ChildHandlers和Pipeline
4. 编写自己的业务Handler并加入Pipeline
5. 启动器绑定端口

具体代码如下：
```java
package org.cp.easychat.server.server;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;
import io.netty.handler.stream.ChunkedWriteHandler;
import lombok.extern.slf4j.Slf4j;

/**
 * ws://localhost:8080/chat
 */
@Slf4j
public class WebSocketServer {

	private NioEventLoopGroup boss;
	private NioEventLoopGroup workers;
	private ServerBootstrap serverBootstrap;

	public void start() {
		this.init();
		try {
			ChannelFuture future = serverBootstrap.bind(8080);
			log.info("server start success.");
			future.channel().closeFuture().sync();
		} catch (Exception e) {
			log.error(e.getMessage(), e);
		}
	}

	private void init() {
		// 配置启动器
		serverBootstrap = new ServerBootstrap();
		serverBootstrap.option(ChannelOption.SO_KEEPALIVE, true);
		serverBootstrap.option(ChannelOption.TCP_NODELAY, true);
		serverBootstrap.option(ChannelOption.SO_BACKLOG, 1024);

		boss = new NioEventLoopGroup();
		workers = new NioEventLoopGroup(7);

		serverBootstrap.group(boss, workers)
				.channel(NioServerSocketChannel.class)
				.childHandler(getChildHandlers());


	}

	private ChannelHandler getChildHandlers() {
		return new ChannelInitializer<>() {
			@Override
			protected void initChannel(Channel ch) throws Exception {
				ChannelPipeline pipeline = ch.pipeline();
				pipeline.addLast(new HttpServerCodec());// http协议的编解码器
				pipeline.addLast(new ChunkedWriteHandler());// 大数据流支持， 切成小块传输
				pipeline.addLast(new HttpObjectAggregator(64*1024));// 聚合器，对应上面的切块
				pipeline.addLast(new WebSocketServerProtocolHandler("/chat"));// 握手 心跳处理
				pipeline.addLast(new MyWebSocketHandler());// 我的业务处理Handler
			}
		};
	}
	
	// 我的业务处理逻辑
	private static class MyWebSocketHandler extends SimpleChannelInboundHandler<Object> {

		// 上线
		@Override
		public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
			log.info("⬆ new connection from {}", ctx.channel().remoteAddress());
		}

		// 下线
		@Override
		public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
			log.info(" ⬇️️ up connection close from {}", ctx.channel().remoteAddress());
		}

		// 读取消息，并返回
		@Override
		protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
			log.info(" 🆕 New Message: {}, from {}", msg, ctx.channel().remoteAddress());
			if (! (msg instanceof TextWebSocketFrame)) {
				log.error("message is not text, {}", msg);
				return;
			}
			TextWebSocketFrame request = (TextWebSocketFrame) msg;
			log.info("received text message : {}", request);

			ctx.writeAndFlush(new TextWebSocketFrame("server send :" + request.text()));
		}
	}
}

```
编写完成之后，我们可以启动服务器，
并使用在线Websocket工具调用测试一下


### 客户端

客户端的编写和服务端类似，
但是有些地方不一样。
需要服务端主动发起连接。

具体步骤如下：
1. 构建一个启动器
2. 创建EventLoopGroup
3. 构建ChildHandlers和Pipeline
4. 在业务Handler中，需要添加发起握手操作
5. 启动器发起连接

代码如下：
```java
package org.cp.client.client;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.ChannelPromise;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.http.FullHttpResponse;
import io.netty.handler.codec.http.HttpClientCodec;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
import io.netty.handler.codec.http.websocketx.WebSocketClientHandshaker;
import io.netty.handler.codec.http.websocketx.WebSocketClientHandshakerFactory;
import io.netty.handler.codec.http.websocketx.WebSocketVersion;
import java.net.URI;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class WebsocketClient {

	private final URI uri;
	private Bootstrap bootstrap;
	private EventLoopGroup eventLoopGroup;
	private ChannelPromise channelPromise;
	private Channel channel;

	public WebsocketClient(URI uri) {
		this.uri = uri;
		this.init();
	}

	public void connect() {
		try {
			channel = bootstrap.connect(uri.getHost(), uri.getPort()).sync().channel();
			channelPromise.sync();
			log.info("connect success and handshake complete.");
		} catch (InterruptedException e) {
			log.error("connect error, " + e.getMessage(), e);
		}
	}

	private void init() {
		bootstrap = new Bootstrap();
		bootstrap.option(ChannelOption.SO_KEEPALIVE, true);
		bootstrap.option(ChannelOption.TCP_NODELAY, true);

		eventLoopGroup = new NioEventLoopGroup();

		bootstrap.group(eventLoopGroup)
				.channel(NioSocketChannel.class)
				.handler(getHandlers());
	}

	private ChannelHandler getHandlers() {
		return new ChannelInitializer<>() {
			@Override
			protected void initChannel(Channel ch) throws Exception {
				ChannelPipeline channelPipeline = ch.pipeline();
				channelPipeline.addLast(new HttpClientCodec());
				channelPipeline.addLast(new HttpObjectAggregator(1048576));
				channelPipeline.addLast(new MyWebSocketHandler(getHandShaker(uri)));
			}

			private WebSocketClientHandshaker getHandShaker(URI uri) {
				return WebSocketClientHandshakerFactory
						.newHandshaker(uri, WebSocketVersion.V13, null, false, null);
			}
		};
	}

	public class MyWebSocketHandler extends SimpleChannelInboundHandler<Object> {


		private final WebSocketClientHandshaker handShaker;

		public MyWebSocketHandler(WebSocketClientHandshaker handShaker) {
			this.handShaker = handShaker;
		}

		@Override
		public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
			channelPromise = ctx.newPromise();
		}

		@Override
		public void channelActive(ChannelHandlerContext ctx) throws Exception {
			log.info("handshake to : {}", ctx.channel().remoteAddress());
			this.handShaker.handshake(ctx.channel());
		}

		@Override
		protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
			log.info("receive data {} from {}", msg, ctx.channel().remoteAddress());
			if (handShaker.isHandshakeComplete()) {
				finishHandShaker(ctx, msg);
				return;
			}
			// handle business data
			if (!(msg instanceof TextWebSocketFrame)) {
				log.warn("{} is not a text message.", msg);
				return;
			}

			TextWebSocketFrame textMsg = (TextWebSocketFrame) msg;
			log.info("client receive a message: {}", textMsg.text());
		}

		private void finishHandShaker(ChannelHandlerContext ctx, Object msg) {
			try {
				handShaker.finishHandshake(ctx.channel(), (FullHttpResponse) msg);
				channelPromise.setSuccess();
				log.info("handShake success.");
			} catch (Exception e) {
				log.error(e.getMessage(), e);
			}
		}
	}
}

```

完整代码请点击[这里](https://github.com/CPyeah/easy-chat)

## Netty在开放平台中的应用

在我们的开放平台中，我们需要通过Websocket对客户端推送一下消息。
所以我们使用Netty搭建了一个Webscoket服务器。

具体搭建步骤跟上面差不多。

但是对于生产环境，权限的校验很重要。

所以在每一个链接建立之后，客户端会发起登录操作。
如果登录成功，服务端会把客户端的信息保存下来，维持会话。

在正常的业务逻辑中，我们首先监听业务中台的MQ消息。
如果消息有对应的客户端在线，我们把消息内容，
封装成`Message`对象，推送给客户端。

然后客户端接受到消息之后，会发起一个消息确认，以保证消息顺利被接收。

流程如下：

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/5cnd8k.png)


## 总结
至此，我们学习了Netty的相关知识。并实操练习了一下。
我们知道了Netty是Java世界中最重要的NIO框架。
它对java.nio的封装，它的事件，反应模型都对我们开发NIO服务器非常有用。
它的多路复用的模型也非常重要。

我们还一起搭建了一个简单的Netty服务，
并使用Netty编写了一个简单的Netty客户端。
帮助大家上手入门。

还有我司在生产环境中是如何使用Netty搭建一个开放平台的消息推送系统的。
供大家参考。

希望这片文章能让大家有所收获！

## 参考
- https://netty.io/
- http://www.kegel.com/c10k.html
- https://gee.cs.oswego.edu/dl/cpjslides/nio.pdf
- https://github.com/netty/netty
- https://github.com/netty/netty/wiki/User-guide-for-4.x
- https://netty.io/3.8/guide/#architecture
- https://www.youtube.com/watch?v=I8yy2Cy7dDI
- https://alibaba-cloud.medium.com/essential-technologies-for-java-developers-i-o-and-netty-ec765676fd21
- https://liakh-aliaksandr.medium.com/java-sockets-i-o-blocking-non-blocking-and-asynchronous-fb7f066e4ede