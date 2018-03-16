## 简单实例-aTimerServer

实现TimeServer，需要覆盖 channelActive() 方法，下面的就是实现的内容：<br>
<pre>
public class TimeServerHandler extends ChannelInboundHandlerAdapter {
  @Override
  public void channelActive(final ChannelHandlerContext ctx) { // (1)
  final ByteBuf time = ctx.alloc().buffer(4); // (2)
  time.writeInt((int) (System.currentTimeMillis() / 1000L + 2208988800L));
  final ChannelFuture f = ctx.writeAndFlush(time); // (3)
  f.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture future) {
      assert f == future;
      ctx.close();
      }
    }); // (4)
  } 
  @Override
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    cause.printStackTrace();
    ctx.close();
   }
}
</pre>
1.channelActive()方法将会在连接被建立并且准备进行通信时被调用。因此在这个方法里完成一个代表当前时间的32位整数消息的构建工作。<br>
2.为了发送一个新的消息，需要分配一个包含这个消息的新的缓冲。因为需要写入一个32位的整数，因此需要一个至少有4个字节的 ByteBuf。通过
ChannelHandlerContext.alloc() 得到一个当前的ByteBufAllocator，然后分配一个新的缓冲。<br>
3.编写一个构建好的消息。使用 NIO 发送消息时不需要调用 java.nio.ByteBuffer.flip()。另外一个点需要注意的是 ChannelHandlerContext.write()和 writeAndFlush()方法会返回一个ChannelFuture对象，一个 ChannelFuture代表了一个还没有发生I/O操作。这意味着任何一个请求操作都不会马上被执行，因为在 Netty里所有的操作都是异步的。举个例子下面的代码中在消息被发送之前可能会先关闭连接。<br>
Channel ch = ...;<br>
ch.writeAndFlush(message);<br>
ch.close();<br>
因此需要在write()方法返回的ChannelFuture完成后调用close()方法，然后当写操作已经完成，会通知他的监听者。请注意,close()方法也可能不会立马关闭，他也会返回一个ChannelFuture。<br>
4.请求已经完成通知,这个只需要简单地在返回的ChannelFuture上增加一个ChannelFutureListener。这里构建了一个匿名的 ChannelFutureListener类用来
在操作完成时关闭 Channel。或者以使用简单的预定义监听器代码:f.addListener(ChannelFutureListener.CLOSE);<br>

