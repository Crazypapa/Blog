## 简单服务器实例：discardServer

参考来源于《netty-4-user-guide.pdf》<br>
<pre>
import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

/** 处理服务端channel. **/
public class DiscardServerHandler extends ChannelInboundHandlerAdapter { // (1)
  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) { // (2)
    // 默默地丢弃收到的数据
    ((ByteBuf) msg).release(); // (3)
  } 
  @Override
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) { // (4)
    // 当出现异常就关闭连接
    cause.printStackTrace();
    ctx.close();
  }
}
</pre>

1.DiscardServerHandler 继承自ChannelInboundHandlerAdapter，这个类实现了ChannelInboundHandler接口，ChannelInboundHandler 提供了许多事件处理的接口方法，然后你可以覆盖这些方法。现在仅仅只需要继承 ChannelInboundHandlerAdapter 类而不是你自己去实现接口方法。<br>
2.这里我们覆盖了 chanelRead() 事件处理方法。每当从客户端收到新的数据时，这个方法会在收到消息时被调用，这个例子中，收到的消息的类型是ByteBuf。<br>
3.为了实现 DISCARD 协议，处理器不得不忽略所有接受到的消息。ByteBuf 是一个引用计数对象，这个对象必须显示地调用 release() 方法来释放。请记住处理器的职责是释放所有传递到处理器的引用计数对象。通常，channelRead() 方法的实现就像下面的这段代码：<br>
<pre>
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
  try {
  // Do something with msg
  } finally {
    ReferenceCountUtil.release(msg);
  }
}
</pre>
4.exceptionCaught() 事件处理方法是当出现 Throwable 对象才会被调用，即当 Netty 由于IO错误或者处理器在处理事件时抛出的异常时。在大部分情况下，捕获的异常应该被记录下来并且把关联的 channel 给关闭掉。然而这个方法的处理方式会在遇到不同异常的情况下有不同的实现，比如你可能想在关闭连接之前发送一个错误码的响应消息<br>
<pre>
main() 方法：
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

/** 丢弃任何进入的数据 **/
public class DiscardServer {
  private int port;
  public DiscardServer(int port) {
    this.port = port;
  }
  public void run() throws Exception {
  EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
  EventLoopGroup workerGroup = new NioEventLoopGroup();
  try {
  ServerBootstrap b = new ServerBootstrap(); // (2)
  b.group(bossGroup, workerGroup)
  .channel(NioServerSocketChannel.class) // (3)
  .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
      @Override
      public void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline().addLast(new DiscardServerHandler());
      }
    })
  .option(ChannelOption.SO_BACKLOG, 128) // (5)
  .childOption(ChannelOption.SO_KEEPALIVE, true); // (6)

  // 绑定端口，开始接收进来的连接
  ChannelFuture f = b.bind(port).sync(); // (7)
  // 等待服务器 socket 关闭 。
  //。。。
  // 在这个例子中，这不会发生，但你可以优雅地关闭你的服务器。
  f.channel().closeFuture().sync();
  } finally {
  workerGroup.shutdownGracefully();
  bossGroup.shutdownGracefully();
  }
} 
public static void main(String[] args) throws Exception {
  int port;
  if (args.length > 0) {
    port = Integer.parseInt(args[0]);
  } else {
    port = 8080;
  } 
  new DiscardServer(port).run();
  }
}
</pre>
1.NioEventLoopGroup 是用来处理I/O操作的多线程事件循环器，Netty 提供了许多不同的EventLoopGroup 的实现用来处理不同的传输。在这个例子中我们实现了一个服务端的应用，因此会有2个 NioEventLoopGroup 会被使用。第一个经常被叫做‘boss’，用来接收进来的连接。第二个经常被叫做‘worker’，用来处理已经被接收的连接，一旦‘boss’接收到连接，就会把连接信息注册到‘worker’上。如何知道多少个线程已经被使用，如何映射到已经创建的Channel上都需要依赖于 EventLoopGroup 的实现，并且可以通过构造函数来配置他们的关系。<br>
2.ServerBootstrap 是一个启动 NIO 服务的辅助启动类。你可以在这个服务中直接使用Channel，但是这会是一个复杂的处理过程，在很多情况下你并不需要这样做。
3.这里我们指定使用 NioServerSocketChannel 类来举例说明一个新的 Channel 如何接收进来的连接。<br>
4.这里的事件处理类经常会被用来处理一个最近的已经接收的 Channel。ChannelInitializer是一个特殊的处理类，他的目的是帮助使用者配置一个新的 Channel。也许你想通过增加一些处理类比如DiscardServerHandler 来配置一个新的 Channel 或者其对应的ChannelPipeline来实现你的网络程序。当你的程序变的复杂时，可能你会增加更多的处理类到 pipline 上，然后提取这些匿名类到最顶层的类上。<br>
5.你可以设置这里指定的 Channel 实现的配置参数。我们正在写一个TCP/IP 的服务端，因此我们被允许设置 socket 的参数选项比如tcpNoDelay 和 keepAlive。请参考 ChannelOption 和详细的 ChannelConfig 实现的接口文档以此可以对ChannelOption 的有一个大概的认识。<br>
6.你关注过 option() 和 childOption() 吗？option() 是提供给NioServerSocketChannel 用来接收进来的连接。childOption() 是提供给由父管道 ServerChannel 接收到的连接，在这个例子中也是 NioServerSocketChannel。<br>
7.我们继续，剩下的就是绑定端口然后启动服务。这里我们在机器上绑定了机器所有网卡上的8080 端口。当然现在你可以多次调用 bind() 方法(基于不同绑定地址)<br>
