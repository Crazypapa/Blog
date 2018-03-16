## 简单实例-aTimeClient

在Netty中,编写服务端和客户端最大的并且唯一不同的使用了不同的BootStrap和Channel的实现。<br>
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
