## 简单实例-aTimeClient

在Netty中,编写服务端和客户端最大的并且唯一不同的使用了不同的BootStrap和Channel的实现。请看一下下面的代码：<br>
<pre>
public class TimeClient {
  public static void main(String[] args) throws Exception {
    String host = args[0];
    int port = Integer.parseInt(args[1]);
    EventLoopGroup workerGroup = new NioEventLoopGroup();
    try {
      Bootstrap b = new Bootstrap(); // (1)
      b.group(workerGroup); // (2)
      b.channel(NioSocketChannel.class); // (3)
      b.option(ChannelOption.SO_KEEPALIVE, true); // (4)
      b.handler(new ChannelInitializer<SocketChannel>() {
        @Override
        public void initChannel(SocketChannel ch) throws Exception {
          ch.pipeline().addLast(new TimeClientHandler());
        }
      });
      // 启动客户端
      ChannelFuture f = b.connect(host, port).sync(); // (5)
      // 等待连接关闭
      f.channel().closeFuture().sync();
    } finally {
      workerGroup.shutdownGracefully();
    }
  }
 }
</pre>
1.BootStrap 和 ServerBootstrap 类似,不过他是对非服务端的 channel 而言，比如客户端或者无连接传输模式的 channel。<br>
2.如果你只指定了一个 EventLoopGroup，那他就会即作为一个 boss group ，也会作为一个workder group，尽管客户端不需要使用到 boss worker.<br>
3.代替NioServerSocketChannel的是NioSocketChannel,这个类在客户端channel 被创建时使用。<br>
4.不像在使用 ServerBootstrap 时需要用 childOption() 方法，因为客户端的 SocketChannel没有父亲。<br>
5.我们用 connect() 方法代替了 bind() 方法。<br>

TimeClientHandler实现：
<pre>
import java.util.Date;
public class TimeClientHandler extends ChannelInboundHandlerAdapter {
  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf m = (ByteBuf) msg; // (1)
    try {
      long currentTimeMillis = (m.readUnsignedInt() - 2208988800L) * 1000L;
      System.out.println(new Date(currentTimeMillis));
      ctx.close();
    } finally {
      m.release();
    }
  } 
  @Override
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    cause.printStackTrace();
    ctx.close();
  }
}
</pre>
1.在TCP/IP中，Netty 会把读到的数据放到 ByteBuf 的数据结构中。
