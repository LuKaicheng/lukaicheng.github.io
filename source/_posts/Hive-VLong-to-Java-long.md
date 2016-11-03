---
title: RCFile文件格式下的Hive表bigint类型列值读取问题
date: 2016-10-25 10:25:39
tags: 
- RCFile
- Hive
---
## 背景缘起
目前在将hive部分列数据采集到hbase时，由于平台将原来hive表的文件格式从SequenceFile调整到RCFile，因此需要对原来的离线数据采集程序进行修改。然而在实际修改开发过程中，却碰到了程序读取hive列字段，值为乱码的问题。

## 初步诊断
由于之前的文件格式是SequenceFile，不管列在hive中数据类型是什么，程序都可以以统一的读取String方式来读取。因此在变更为RCFile方式时，仅仅调整了输入部分，转换依然采用了Bytes.toString方式。

{% codeblock lang:java %}
public static String convertHiveBigint(BytesRefWritable brw) throws Exception{
    return Bytes.toString(brw.getData(),brw.getStart(),brw.getLength());
}
{% endcodeblock %}

于是针对乱码的字段，查看了hive表对应列的数据类型，发现是Bigint，那么我想可能不仅仅需要调整输入，最终转换的地方也需要调整为Bytes.toLong。于是将代码修改如下:

{% codeblock lang:java %}
public static Long convertHiveBigint(BytesRefWritable brw) throws Exception{
    return Bytes.toLong(brw.getData(),brw.getStart(),brw.getLength());
}
{% endcodeblock %}

## 奇怪的-118
就在我以为上面的修改可以奏效之时，实际运行时却抛出了鲜红的异常

{% img [异常信息] /images/exception.png %}

从异常信息里不难看出，对于Java long来说，已经规定是需要8个字节，然而在上面代码里，最终转换时确变成了7个字节。为此，对程序进行了调试，将原始的brw.getData打印了出来

{% img [原始字节数据] /images/bytesarray.png %}

结合调试得到的offset和length值，获知到程序真实运行时取值为 **-118 1 58 92 103 58 -127**，而我直接通过hive sql查询列原始值，并转换成字节数组为 **0 0 1 58 92 103 58 -127**。观察这两组数据不难发现，不同之处在于前者开头是-118，而后者是0 0。而再仔细观察上面的字节数组，会很惊讶的发现，好像每隔6位，就会出现-118。这个时候，我就猜测假如能够弄明白-118的来源，那么我们的问题有很大概率就可以解决了。

## 无心插柳
正当我对这个问题陷入困顿的时候，无意间发现了下图的信息(莫非这个是hive序列化类(⊙o⊙)？)
{% img [hive信息] /images/hive.png %}

于是，我马上翻开了这个类的源码，根据类上面的注释确认了该类确实能将hive column序列化为BytesRefArrayWritable。该类只有initialize和serialize这两个方法，针对序列化过程，不难猜到入口肯定是serialize这个方法，由于需要序列化的列类型是Bigint，因此判定进入如下分支:

{% img [序列化] /images/serialize.png %}

进入此方法后，是一个switch case分支选择，根据bigint和long对应关系，判定进入如下case：

{% img [Long] /images/caselong.png %}

最终代码导航下去，你会发现实际的转换过程在方法**LazyBinaryUtils.writeVLongToByteArray**,这也解开了上面-118的问题,不过限于水平目前还不太理解这段代码的含义，(⊙﹏⊙)b

{% codeblock LazyBinaryUtils.writeVLongToByteArray lang:java %}
public static int writeVLongToByteArray(byte[] bytes, int offset, long l) {
  if (l >= -112 && l <= 127) {
    bytes[offset] = (byte) l;
    return 1;
  }

  int len = -112;
  if (l < 0) {
    l ^= -1L; // take one's complement'
    len = -120;
  }

  long tmp = l;
  while (tmp != 0) {
    tmp = tmp >> 8;
    len--;
  }

  bytes[offset] = (byte) len;

  len = (len < -120) ? -(len + 120) : -(len + 112);

  for (int idx = len; idx != 0; idx--) {
    int shiftbits = (idx - 1) * 8;
    long mask = 0xFFL << shiftbits;
    bytes[offset+1-(idx - len)] = (byte) ((l & mask) >> shiftbits);
  }
  return 1 + len;
}
{% endcodeblock %}

## 解决问题
最后，LazyBinaryUtils不仅提供了VLong序列化成字节数组的过程，同时也提供反序列化的过程，实际程序中只需调用LazyBinaryUtils.readVLongFromByteArray方法即可。
{% codeblock lang:java %}
public static long readVLongFromByteArray(final byte[] bytes, int offset) {
  byte firstByte = bytes[offset++];
  int len = WritableUtils.decodeVIntSize(firstByte);
  if (len == 1) {
    return firstByte;
  }
  long i = 0;
  for (int idx = 0; idx < len-1; idx++) {
    byte b = bytes[offset++];
    i = i << 8;
    i = i | (b & 0xFF);
  }
  return (WritableUtils.isNegativeVInt(firstByte) ? ~i : i);
}
{% endcodeblock %}

## 总结扩展
序列化和反序列化是相对的，在不了解具体序列化规则的情况下，去进行反序列化，有时候可能碰到问题。而在这个场景里，hive在序列化bigint时，是按照不定长的VLONG形式进行转换，如果我们依然按照原先读取long类型方式，那么就会碰到问题，因此一定要选择对应的反序列化方式才能获取到正确的值。在以后碰到hive不同列类型转换成Java中的类型时，可以参考LazyBinarySerDe.serialize对应类型的序列化方式，来寻找API中对应的反序列化方式，抑或自己动手造轮子。