---
layout: post
title: Netty 4.1 - Writing multiple messages to a single channel
---
## Introduction 
In networking , sending push messages or notifications to a peer can be necessary whereby a peer needs to be notified of partial messages or stream of messages that cannot be received at once. An example of this is where by a websocket Server needs to notify a JS Client to acknowledge that a connection is still active by responding to the heartbeats from the server inorder to keep the channel operational , This can be likened to an echo client server concept.

This is only possible if and only if a channel connection is established and active for receiving inbound and outbound messages , with Netty a client and server has to maintain duplex communication using the NIO channel hence sending messages back and forth is done typically on one thread allocated to a channel in an EventLoop.

## [Event Loop](https://netty.io/4.1/api/io/netty/channel/EventLoop.html)
The event loop is what allows Netty to perform non-blocking I/O operations

![Event Loop]({{site.baseurl}}/assets/images/1681501236711.png)

By looping, this is just a concept that revolves around checking the channels for events and dispatch the channel to the different threads for processing. So this is the key idea of Netty on how it dispatches messages to channels using  different threads downstream for processing .



## Writing multiple messages to an established Channel

To accomplish this, its as simple as using an executor service provided in [ChannelHandlerContext](https://netty.io/4.1/api/io/netty/channel/ChannelHandlerContext.html) for an outbound , using a provided executor service produces the same effect but leads to more complexity where a developer has to manage the additional resources like multi threading on a JVM .


## Server Bootstrap

```java

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

/**
 * Heartbeat server that sends push notification to a peer
 */
public final class HeartBeatServer {

    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));

    public static void main(String[] args) throws Exception {
        // Configure SSL.
        final SslContext sslCtx = ServerUtil.buildSslContext();

        // Configure the server.
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        final HeartBeatServer serverHandler = new HeartBeatServer();
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
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
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

## Server Business Logic 

```java
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandler.Sharable;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.nio.charset.Charset;
import java.util.concurrent.TimeUnit;
import java.util.logging.Logger;

/**
 * Handler implementation for the HeartBeat server.
 */
@Sharable
public class HeartBeatServerHandler extends ChannelInboundHandlerAdapter {

    /**
     * @param ctx  channel handler context 
     * @param msg - raw byte buffer
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg){
       // Todo - logic to process Object msg
       // Next steps is to obtain the executor service from the 
       // channel handler context

       ctx.channel()
        .eventLoop()
        .scheduleAtFixedRate(() -> {

                ctx.writeAndFlush( Unpooled.copiedBuffer("testing scheduled message\n", Charset.forName("utf-8")))
                        .addListener((ChannelFutureListener) channelFuture -> {
                            // log complete operation evenet
                        });

        }, 5, 5, TimeUnit.SECONDS);
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


## Client

To test this quickly we can use a telnet program to connect and send a dummpy or empty space  string to the server and we can observe that a message is sent every 5seconds.

```
➜  heart-beat-app git:(develop) ✗ telnet localhost 8007

Trying ::1...
Connected to localhost.
Escape character is '^]'.

testing scheduled message
testing scheduled message
testing scheduled message
testing scheduled message
testing scheduled message
```

## Pitfall - Sending Multiple HttpContent

Http protocol as we know it naturally uses TCP as its underlining protocol or backbone for communication . If we attempt to send `HttpContent` usings Nettys combined channel duplex handler `HttpServerCodec` it will provide us with an encoder `HttpContentEncoder` which cannot abide to the multiple messages concept which is explained above. Below is a simple http response wrapper buffer message to be transformed by an encoder during a write operation . 

```java
tx.channel().eventLoop()
        .scheduleAtFixedRate(() -> {

            try {
                FullHttpResponse rsp = new DefaultFullHttpResponse(req.protocolVersion(), OK,
                        Unpooled.wrappedBuffer(CONTENT));
                rsp.headers()
                        .set(CONTENT_TYPE, TEXT_PLAIN)
                        .setInt(CONTENT_LENGTH, rsp.content().readableBytes());
                
              ctx.writeAndFlush(rsp).sync();
            }catch (Exception e) {
                e.printStackTrace();
            }
        }, 3, 3, TimeUnit.SECONDS);

```

It comes out with an error as this 

```
io.netty.handler.codec.EncoderException: java.lang.IllegalStateException: cannot send more responses than requests
	at io.netty.handler.codec.MessageToMessageEncoder.write(MessageToMessageEncoder.java:104)
	at io.netty.handler.codec.MessageToMessageCodec.write(MessageToMessageCodec.java:116)
	at io.netty.channel.AbstractChannelHandlerContext.invokeWrite0(AbstractChannelHandlerContext.java:717)
	at io.netty.channel.AbstractChannelHandlerContext.invokeWrite(AbstractChannelHandlerContext.java:709)
	at io.netty.channel.AbstractChannelHandlerContext.write(AbstractChannelHandlerContext.java:792)
	at io.netty.channel.AbstractChannelHandlerContext.write(AbstractChannelHandlerContext.java:702)
	at io.netty.channel.ChannelDuplexHandler.write(ChannelDuplexHandler.java:115)

```


```java

@Override
protected void encode(ChannelHandlerContext ctx, HttpObject msg, List<Object> out) throws Exception {

    final boolean isFull = msg instanceof HttpResponse && msg instanceof LastHttpContent;
    switch (state) {
        case AWAIT_HEADERS: {
            ensureHeaders(msg);

            assert encoder == null;

            final HttpResponse res = (HttpResponse) msg;
            final int code = res.status().code();
            final CharSequence acceptEncoding;

            if (code == CONTINUE_CODE) {
                // We need to not poll the encoding when response with CONTINUE as another response will follow

                // for the issued request. See https://github.com/netty/netty/issues/4079

                acceptEncoding = null;
            } else {
                // Get the list of encodings accepted by the peer.
                acceptEncoding = acceptEncodingQueue.poll();
                if (acceptEncoding == null) {
                    throw new IllegalStateException("cannot send more responses than requests");
                }
            }
```

This is as a result of this validation in the content encoder which maintains a corresponding queue of encoding char sequences, in order words there can only be one read one write at a time in an active channel

## Conclusion
It was nice writing this article and I hope it helps demystify issues around multiple writes or push messages communicated to a peer , If any queries about Netty in general I can be reached on my [mail](mailto:johnsoneyo@gmail.com) . Peace and Love .

