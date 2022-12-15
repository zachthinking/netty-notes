本文介绍Netty程序的基本结构。
我准备了一个服务端程序和一个客户端程序。
服务端的工作流程是：服务端启动后，监听端口，等待客户端连接，接收客户端发送的字符串消息，并打印。
客户端的工作流程是：客户端启动后，连接服务端，发送一条hello的字符串消息，然后断开连接，关闭事件循环，退出程序。

## 服务端程序

示例代码
```java
public class FirstServer {  
  
    static int port = 12345;  
    EventLoopGroup bossGroup = new NioEventLoopGroup(); 
    EventLoopGroup workerGroup = new NioEventLoopGroup();  
    ServerBootstrap b = new ServerBootstrap();
    Channel channel;  
  
    void start() throws InterruptedException {  
        try {  
            b.group(bossGroup, workerGroup)  
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {  
                        @Override  
                        public void initChannel(SocketChannel ch) throws Exception {  
						    ChannelPipeline p = ch.pipeline();  
						    p.addLast(new LoggingHandler(LogLevel.INFO));  
						    p.addLast(new StringDecoder());  
						    p.addLast(new FirstServerHandler());
                        }  
                    })  
                    .option(ChannelOption.SO_BACKLOG, 128)          
                    .childOption(ChannelOption.SO_KEEPALIVE, true);  
  
            ChannelFuture f = b.bind(port).sync(); 
            System.out.println("bind on port: " + port);  
              
            channel = f.channel();  
            System.out.println("channel: " + channel.toString());  
            channel.closeFuture().sync();  
        } finally {  
            System.out.println("stop ...");  
            workerGroup.shutdownGracefully();  
            bossGroup.shutdownGracefully();  
        }  
    }  
  
    public void stop() {  
        channel.close();  
    }  
  
    public static void main(String[] args) throws InterruptedException {  
        FirstServer server = new FirstServer();  
        server.start();  
    }  
}

public class FirstServerHandler extends ChannelInboundHandlerAdapter {  
  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) {  
        System.out.println((String) msg);  
    }  
  
    @Override  
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {  
        cause.printStackTrace();  
        ctx.close();  
    }  
}
```

### 组件

下面分别介绍Netty程序的几个组件。组件的详细介绍会在后面的文章一一展开。这里有个初步印象，知道有这个组件，大致是做什么用的，几个组件之间怎么配合的，就可以了。

#### EventLoopGroup

EventLoopGroup，事件循环组，相当于线程池。负责事件循环。它可以设置同时处理请求的线程数。
服务端有两个EventLoopGroup。一个是用于接收客户端请求，叫bossGroup。一个是用于处理客户端请求，叫工作线程，workGroup。

#### ServerBootstrap

ServerBootstrap，服务端启动器，把各个组件组合在一起。包括EventLoopGroup事件循环组、NioServerSocketChannel（Channel类型）、组装Pipline。

#### Channel

Channel，渠道，是对 SocketChannel 的封装，代表网络连接。可以通过它读取和写入消息。也可以关闭连接。
服务端监听时，有一个channel。每个请求过来时，单独分配一个channel。

#### ChannelPipeline

ChannelPipeline，渠道流水线，用于编排处理请求的一系列ChannelHandler。

#### ChannelHandler

ChannelHandler，渠道处理器，是处理请求的基本单元。
这里配置了两个ChannelHandler，一个是String解码器，另一个是自定义处理器，打印接收到的消息。

#### ChannelFuture

ChannelFuture，继承Future，异步操作接口。

### 启动服务

调用ServerBootstrap的bind()方法，绑定端口，监听。

### 停止服务

调用EventLoopGroup的shutdownGracefully()方法，停止workerGroup和bossGroup。

### 异步与同步

bind() 返回 ChannelFuture 是个异步操作。通过sync() 同步。当绑定结束后，完成同步。
closeFuture() 返回 ChannelFuture 是个异步操作。通过sync() 同步。当连接关闭后，完成同步。

## 客户端程序

示例代码
```java
public final class FirstClient {  
  
    String host = "127.0.0.1";  
    int port = 12345;  
    EventLoopGroup group = new NioEventLoopGroup();  
    Bootstrap b = new Bootstrap();  
    Channel channel;  
  
    void setup() {  
        b.group(group)  
                .channel(NioSocketChannel.class)  
                .option(ChannelOption.TCP_NODELAY, true)  
                .handler(new ChannelInitializer<SocketChannel>() {  
                    @Override  
                    public void initChannel(SocketChannel ch) throws Exception {  
                        ChannelPipeline p = ch.pipeline();  
                        p.addLast(new LoggingHandler(LogLevel.INFO));  
                        p.addLast(new StringEncoder());  
                    }  
                });  
    }  
  
    void connect() throws InterruptedException {  
        ChannelFuture f = b.connect(host, port).sync();  
        channel = f.channel();  
        System.out.println("channel: " + channel.toString());  
    }  
  
    void send() {  
        channel.writeAndFlush("hello");  
    }  
  
    void close() {  
        channel.close();  
    }  
      
    void destory() {  
        group.shutdownGracefully();          
    }  
  
    public static void main(String[] args) throws Exception {  
        FirstClient client = new FirstClient();  
        client.setup();  
        client.connect();  
        client.send();  
        client.close();  
        client.destory();  
    }  
}
```

### 初始化

#### EventLoopGroup

EventLoopGroup，事件循环组。与服务端不同。客户端只需要一个。

#### Bootstrap

Bootstrap，客户端启动器。和服务端类似，把各个组件组合在一起。包括EventLoopGroup事件循环组、NioServerSocketChannel（Channel类型）、组装Pipline。

#### ChannelHandler

这里配置了一个ChannelHandler，String编码器。

### 连接

调用Bootstrap的connect()方法，连接到服务端。获得Chanel实例。

### 发送

调用Channel的writeAndFlush()方法，发送消息。

### 断开连接

调用Channel的close()方法，关闭连接。

### 停止

调用EventLoopGroup的shutdownGracefully()方法，停止group。
