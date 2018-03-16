##基于流的处理

*The First Solution*
最简单的方案是构造一个内部的可积累的缓冲，直到4个字节全部接收到了内部缓冲。下面的代码修改了 TimeClientHandler 的实现类修复了这个问题
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
1.ChannelHandler 有2个生命周期的监听方法：handlerAdded()和 handlerRemoved()。你可以完成任意初始化任务只要他不会被阻塞很长的时间。<br>
2.首先，所有接收的数据都应该被累积在 buf 变量里。<br>
3.然后，处理器必须检查 buf 变量是否有足够的数据，在这个例子中是4个字节，然后处理实际的业务逻辑。否则，Netty 会重复调用channelRead() 当有更多数据到达直到4个字节的数据被积累。<br>
