#### The Second Solution
<pre>
public class TimeDecoder extends ByteToMessageDecoder{ //(1)
@Override
protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out){ //(2)
      if (in.readableBytes() < 4) {
        return; //(3)
      }
    out.add(in.readBytes(4)); //(4)
  }
}
</pre>

把一整个ChannelHandler拆分成多个模块以减少应用的复杂程度，比如把TimeClientHandler拆分成2个处理器：<br>
1,TimeDecoder处理数据拆分的问题<br>
2,TimeClientHandler原始版本的实现<br>
Netty提供了一个可扩展的类，可以完成TimeDecoder的开发。<br>

1.ByteToMessageDecoder是ChannelInboundHandler的一个实现类，他可以在处理数据拆分的问题上变得很简单。<br>
2.每当有新数据接收的时候ByteToMessageDecoder都会调用decode()方法来处理内部的那个累积缓冲。<br>
3.Decode()方法可以决定当累积缓冲里没有足够数据时,可以往out对象里放任意数据。<br>
当有更多的数据被接收了 ByteToMessageDecoder 会再一次调用 decode() 方法。<br>
4.如果在decode()方法里增加了一个对象到out对象里，这意味着解码器解码消息成功。ByteToMessageDecoder将会丢弃在累积缓冲里已经被读过的数据。<br>
请记得不需要对多条消息调用decode()，ByteToMessageDecoder会持续调用decode()直到不放任何数据到out里。<br>

现在将另外一个处理器插入到ChannelPipeline 里，应该在TimeClient里修改ChannelInitializer 的实现：<br>
<pre>
b.handler(new ChannelInitializer<SocketChannel>() {
  @Override
  public void initChannel(SocketChannel ch) throws Exception {
    ch.pipeline().addLast(new TimeDecoder(), new TimeClientHandler());
    }
});
</pre>

还可以尝试使用更简单的解码类ReplayingDecoder。需要参考一下API文档来获取更多的信息。<br>
<pre>
public class TimeDecoder extends ReplayingDecoder<Void> {
  @Override
  protected void decode(
    ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
      out.add(in.readBytes(4));
  }
}
</pre>

