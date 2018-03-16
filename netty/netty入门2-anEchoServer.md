## 简单实例-anEchoServer

参考来源于《netty-4-user-guide.pdf》<br>
和discard server唯一不同的是把在此之前我们实现的channelRead()方法，返回所有的数，<br>
据替代打印接收数据到控制台上的逻辑。因此，需要把channelRead()方法修改如下：<br>
<pre>
@Override<br>
public void channelRead(ChannelHandlerContext ctx, Object msg) {
  ctx.write(msg); // (1)
  ctx.flush(); // (2)
}
</pre>
1. ChannelHandlerContext对象提供了许多操作，使能够触发各种各样的I/O事件和操作。这里调用了write(Object)方法来逐字地把接受到的消息写入。请注意不同于
DISCARD的例子并没有释放接受到的消息，这是因为当写入的时候 Netty已经自己已经释放了。<br>
2. ctx.write(Object)方法不会使消息写入到通道上，他被缓冲在了内部，需要调用ctx.flush()方法来把缓冲区中数据强行输出。或者可以用更简洁的
cxt.writeAndFlush(msg)以达到同样的目的。<br>
