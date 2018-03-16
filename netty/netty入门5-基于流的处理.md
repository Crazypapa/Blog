##基于流的处理

**The First Solution**<br>
最简单的方案是构造一个内部的可积累的缓冲，直到4个字节全部接收到了内部缓冲。<br>
下面的代码修改了TimeClientHandler 的实现类，解决了这个问题：<br>
<pre>
ublic class TimeClientHandler extends ChannelInboundHandlerAdapter {
  private ByteBuf buf;
  @Override
  public void handlerAdded(ChannelHandlerContext ctx) {
    buf = ctx.alloc().buffer(4); // (1)
  }
  @Override
  public void handlerRemoved(ChannelHandlerContext ctx) {
    buf.release(); // (1)
    buf = null;
  }
  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf m = (ByteBuf) msg;
    buf.writeBytes(m); // (2)
    m.release();
    if (buf.readableBytes() >= 4) { // (3)
      long currentTimeMillis = (buf.readUnsignedInt() - 2208988800L) * 1000L;
      System.out.println(new Date(currentTimeMillis));
      ctx.close();
    }
  } 
  @Override
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    cause.printStackTrace();
    ctx.close();
  }
}
</pre>

1.ChannelHandler 有2个生命周期的监听方法：handlerAdded()和 handlerRemoved()。可以完成任意初始化任务，只要保证初始时不能被阻塞很长的时间。<br>
2.首先，所有接收的数据都应该被累积在buf变量里。<br>
3.然后，处理器必须检查buf变量是否有足够的数据，在这个例子中是4个字节，然后处理实际的业务逻辑。<br>
否则，Netty会重复调用channelRead()当有更多数据到达直到4个字节的数据被积累。<br>
