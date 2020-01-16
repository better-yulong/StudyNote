> 段落引用参考Netty官方指南 http://ifeve.com/netty-home/、https://netty.io/wiki/index.html，于是先从Echo ‐ 基础的客户端和服务器入手，尝试运行起来

### Server端代码
#### EchoServerHandler 
```
/*
 * Copyright 2012 The Netty Project
 *
 * The Netty Project licenses this file to you under the Apache License,
 * version 2.0 (the "License"); you may not use this file except in compliance
 * with the License. You may obtain a copy of the License at:
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations
 * under the License.
 */
package io.netty.example.echo;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandler.Sharable;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.CharsetUtil;

/**
 * Handler implementation for the echo server.
 */
@Sharable
public class EchoServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
//        ctx.write(msg);
    	
    	//fix001-start-byron: receive client send string Hello, Netty 
        ByteBuf in = (ByteBuf) msg;
        //将接受到的消息输出到客户端
        System.out.println("Server received：" + in.toString(CharsetUtil.UTF_8));
        //将接收到的消息写给发送者，而不冲刷出站消息
        ctx.write(in);
      //fix001-end-byron
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // Close the connection when an exception is raised.
        cause.printStackTrace();
        ctx.close();
    }
}

```
#### EchoServerHandler 
```language
/*
 * Copyright 2012 The Netty Project
 *
 * The Netty Project licenses this file to you under the Apache License,
 * version 2.0 (the "License"); you may not use this file except in compliance
 * with the License. You may obtain a copy of the License at:
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations
 * under the License.
 */
package io.netty.example.echo;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;
import io.netty.handler.ssl.SslContext;
import io.netty.handler.ssl.SslContextBuilder;
import io.netty.handler.ssl.util.SelfSignedCertificate;

/**
 * Echoes back any received data from a client.
 */
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
                     p.addLast(new LoggingHandler(LogLevel.INFO));
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

### Client端代码
#### EchoClientHandler
```language
/*
 * Copyright 2012 The Netty Project
 *
 * The Netty Project licenses this file to you under the Apache License,
 * version 2.0 (the "License"); you may not use this file except in compliance
 * with the License. You may obtain a copy of the License at:
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations
 * under the License.
 */
package io.netty.example.echo;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.CharsetUtil;

/**
 * Handler implementation for the echo client.  It initiates the ping-pong
 * traffic between the echo client and server by sending the first message to
 * the server.
 */
public class EchoClientHandler extends ChannelInboundHandlerAdapter {

    private final ByteBuf firstMessage;

    /**
     * Creates a client-side handler.
     */
    public EchoClientHandler() {
    	//fix001-start-byron: receive client send string Hello, Netty 
		/*
		 * 原示例代码：
		 * firstMessage = Unpooled.buffer(EchoClient.SIZE); 
		 * for (int i = 0; i <firstMessage.capacity(); i ++) 
		 * { 
		 * 		firstMessage.writeByte((byte) i); 
		 * }
		 */

        firstMessage = Unpooled.copiedBuffer("Hello, Netty", CharsetUtil.UTF_8);
      //fix001-end-byron:
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
      ctx.writeAndFlush(firstMessage);
      ctx.close();
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ctx.write(msg);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
       ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // Close the connection when an exception is raised.
        cause.printStackTrace();
        ctx.close();
    }
}

```
#### EchoClient 
```language
 * Copyright 2012 The Netty Project
package io.netty.example.echo;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;
import io.netty.handler.ssl.SslContext;
import io.netty.handler.ssl.SslContextBuilder;
import io.netty.handler.ssl.util.InsecureTrustManagerFactory;

/**
 * Sends one message when a connection is open and echoes back any received
 * data to the server.  Simply put, the echo client initiates the ping-pong
 * traffic between the echo client and server by sending the first message to
 * the server.
 */
public final class EchoClient {

    static final boolean SSL = System.getProperty("ssl") != null;
    static final String HOST = System.getProperty("host", "127.0.0.1");
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));
    static final int SIZE = Integer.parseInt(System.getProperty("size", "256"));

    public static void main(String[] args) throws Exception {
        // Configure SSL.git
        final SslContext sslCtx;
        if (SSL) {
            sslCtx = SslContextBuilder.forClient()
                .trustManager(InsecureTrustManagerFactory.INSTANCE).build();
        } else {
            sslCtx = null;
        }

        // Configure the client.
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
             .channel(NioSocketChannel.class)
             .option(ChannelOption.TCP_NODELAY, true)
             .handler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc(), HOST, PORT));
                     }
                     p.addLast(new LoggingHandler(LogLevel.INFO)); //fix001-start-byron:print netty client detail log(see last log)
                     p.addLast(new EchoClientHandler());
                 }
             });

            // Start the client.
            ChannelFuture f = b.connect(HOST, PORT).sync();

            // Wait until the connection is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down the event loop to terminate all threads.
            group.shutdownGracefully();
        }
    }
}
```

### 运行结果（Run as Application)
#### Client端日志
 * 上面代码添加 p.addLast(new LoggingHandler(LogLevel.INFO));发起请求方可打印如下详细日志：
 *
15:53:34.586 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0x603c24e3] REGISTERED
15:53:34.588 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0x603c24e3] CONNECT: /127.0.0.1:8007
15:53:34.591 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0x603c24e3, L:/127.0.0.1:58166 - R:/127.0.0.1:8007] ACTIVE
15:53:34.592 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0x603c24e3, L:/127.0.0.1:58166 - R:/127.0.0.1:8007] WRITE: 12B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 48 65 6c 6c 6f 2c 20 4e 65 74 74 79             |Hello, Netty    |
+--------+-------------------------------------------------+----------------+
15:53:34.597 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0x603c24e3, L:/127.0.0.1:58166 - R:/127.0.0.1:8007] FLUSH
15:53:34.598 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0x603c24e3, L:/127.0.0.1:58166 - R:/127.0.0.1:8007] CLOSE
15:53:34.598 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0x603c24e3, L:/127.0.0.1:58166 ! R:/127.0.0.1:8007] INACTIVE
15:53:34.598 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0x603c24e3, L:/127.0.0.1:58166 ! R:/127.0.0.1:8007] UNREGISTERED
#### Server端日志
```
客户端请求后服务接收到“Hello, Netty”日志如下：
18:17:02.052 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xcf89a579] REGISTERED
18:17:02.055 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xcf89a579] BIND: 0.0.0.0/0.0.0.0:8007
18:17:02.057 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xcf89a579, L:/0:0:0:0:0:0:0:0:8007] ACTIVE
18:17:13.213 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xcf89a579, L:/0:0:0:0:0:0:0:0:8007] READ: [id: 0xd149e4f5, L:/127.0.0.1:8007 - R:/127.0.0.1:58220]
18:17:13.214 [nioEventLoopGroup-2-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xcf89a579, L:/0:0:0:0:0:0:0:0:8007] READ COMPLETE
18:17:13.247 [nioEventLoopGroup-3-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xd149e4f5, L:/127.0.0.1:8007 - R:/127.0.0.1:58220] REGISTERED
18:17:13.247 [nioEventLoopGroup-3-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xd149e4f5, L:/127.0.0.1:8007 - R:/127.0.0.1:58220] ACTIVE
18:17:13.257 [nioEventLoopGroup-3-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xd149e4f5, L:/127.0.0.1:8007 - R:/127.0.0.1:58220] READ: 12B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 48 65 6c 6c 6f 2c 20 4e 65 74 74 79             |Hello, Netty    |
+--------+-------------------------------------------------+----------------+
Server received：Hello, Netty
18:17:13.257 [nioEventLoopGroup-3-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xd149e4f5, L:/127.0.0.1:8007 - R:/127.0.0.1:58220] WRITE: 12B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 48 65 6c 6c 6f 2c 20 4e 65 74 74 79             |Hello, Netty    |
+--------+-------------------------------------------------+----------------+
18:17:13.258 [nioEventLoopGroup-3-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xd149e4f5, L:/127.0.0.1:8007 - R:/127.0.0.1:58220] READ COMPLETE
18:17:13.258 [nioEventLoopGroup-3-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xd149e4f5, L:/127.0.0.1:8007 - R:/127.0.0.1:58220] FLUSH
18:17:13.259 [nioEventLoopGroup-3-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xd149e4f5, L:/127.0.0.1:8007 - R:/127.0.0.1:58220] READ COMPLETE
18:17:13.259 [nioEventLoopGroup-3-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xd149e4f5, L:/127.0.0.1:8007 - R:/127.0.0.1:58220] FLUSH
18:17:13.259 [nioEventLoopGroup-3-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xd149e4f5, L:/127.0.0.1:8007 ! R:/127.0.0.1:58220] INACTIVE
18:17:13.259 [nioEventLoopGroup-3-1] INFO  i.n.handler.logging.LoggingHandler - [id: 0xd149e4f5, L:/127.0.0.1:8007 ! R:/127.0.0.1:58220] UNREGISTERED

```

