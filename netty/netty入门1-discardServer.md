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

1.DiscardServerHandler继承自ChannelInboundHandlerAdapter，这个类实现了ChannelInboundHandler接口，ChannelInboundHandler提供了许多事件处理的接口方法，然后你可以覆盖这些方法。现在仅仅只需要继承 ChannelInboundHandlerAdapter类而不是你自己去实现接口方法。<br>
2.这里覆盖了chanelRead()事件处理方法。每当从客户端收到新的数据时，这个方法会在收到消息时被调用，这个例子中，收到的消息的类型是ByteBuf。<br>
3.为了实现DISCARD协议，处理器不得不忽略所有接受到的消息。ByteBuf是一个引用计数对象，这个对象必须显示地调用release()方法来释放。请记住处理器的职责是释放所有传递到处理器的引用计数对象。通常channelRead()方法的实现就像下面的这段代码：<br>
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
4.exceptionCaught()事件处理方法是当出现Throwable对象才会被调用，即当Netty由于IO错误或者处理器在处理事件时抛出的异常时。在大部分情况下，捕获的异常应该被记录下来并且把关联的channel 给关闭掉。然而这个方法的处理方式会在遇到不同异常的情况下有不同的实现，比如你可能想在关闭连接之前发送一个错误码的响应消息<br>
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
  // 在这个例子中，可以关闭服务器。
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
1.NioEventLoopGroup是用来处理I/O操作的多线程事件循环器，Netty提供了许多不同的EventLoopGroup的实现用来处理不同的传输。在这个例子中实现了一个服务端的应用，因此会有2个NioEventLoopGroup被使用。第一个经常被叫做‘boss’，用来接收进来的连接。第二个经常被叫做‘worker’，用来处理已经被接收的连接，一旦‘boss’接收到连接，就会把连接信息注册到‘worker’上。如何知道多少个线程已经被使用，如何映射到已经创建的Channel上都需要依赖于EventLoopGroup的实现，并且可以通过构造函数来配置他们的关系。<br>
2.ServerBootstrap是一个启动 NIO 服务的辅助启动类。可以在这个服务中直接使用Channel，但是这会是一个复杂的处理过程，在很多情况下并不需要这样做。
3.这里指定使用 NioServerSocketChannel类来举例说明一个新的Channel如何接收进来的连接。<br>
4.这里的事件处理类经常会被用来处理一个最近的已经接收的Channel。ChannelInitializer是一个特殊的处理类，他的目的是帮助使用者配置一个新的Channel。也许你想通过增加一些处理类比如DiscardServerHandler来配置一个新的Channel或者其对应的ChannelPipeline来实现网络程序。当程序变的复杂时，可能会增加更多的处理类到pipline上，然后提取这些匿名类到最顶层的类上。<br>
5.可以设置这里指定的Channel实现的配置参数。允许设置socket的参数选项比如tcpNoDelay和keepAlive。请参考ChannelOption和详细的ChannelConfig实现的接口文档，以此可以对ChannelOption的有一个大概的认识。<br>
6.option()是提供给NioServerSocketChannel用来接收进来的连接。childOption()是提供给由父管道 ServerChannel 接收到的连接，在这个例子中也是 NioServerSocketChannel。<br>
7.剩下的就是绑定端口然后启动服务。在机器上绑定了机器所有网卡上的8080端口。当然可以多次调用bind()方法(基于不同绑定地址)<br>
