---
title: Netty源码分析-(伪)-(4)-ByteBuf
date: 2018-06-21 12:00:48
tags:
toc: true
categories: ["java", "网络编程", 'Netty']
---

{% raw %}

<script src="https://lib.baomitu.com/webfont/1.6.28/webfontloader.js"></script>
<script src="https://lib.baomitu.com/snap.svg/0.5.1/snap.svg-min.js"></script>
<script src="https://lib.baomitu.com/underscore.js/1.9.0/underscore-min.js"></script>
<script src="https://lib.baomitu.com/raphael/2.2.7/raphael.min.js"></script>
<script src="https://lib.baomitu.com/js-sequence-diagrams/1.0.6/sequence-diagram-min.js"></script>

{% endraw %}

## ByteBuf
ByteBuf 是 Netty 底层支撑。缓冲区是不同的通道之间传递数据的中介，JDK中的ByteBuffer操作复杂，而且没有经过优化，所以在netty中实现了一个更加强大的缓冲区 ByteBuf 用于表示字节序列。ByteBuf在netty中是通过Channel传输数据的，新的设计解决了JDK中ByteBuffer中的一些问题。 

- 可扩展到用户定义的buffer类型中
- 通过内置的复合buffer类型实现透明的零拷贝(zero-copy)
- 容量可以根据需要扩展
- 切换读写模式不需要调用ByteBuffer.flip()方法
- 读写采用不同的索引
- 支持方法链接调用
- 支持引用计数
- 支持池技术(比如：线程池、数据库连接池)

<!-- more -->

## 原理
ByteBuf维护两个不同的索引：读索引和写索引。当你从ByteBuf中读，它的readerIndex增加了读取的字节数；同理，当你向ByteBuf中写，writerIndex增加。下图显示了一个空的ByteBuf的布局和状态:
![empty](https://s1.ax1x.com/2018/06/21/PSN5TI.png)

## 自动扩容
在 `Java Nio` 的 `java.nio.HeapByteBuffer#put(byte)` 中我们可以看java 的 `ByteBuffer`
```java
public ByteBuffer put(byte x) {
    hb[ix(nextPutIndex())] = x;
    return this;
}

final int nextPutIndex() {                         
    if (position >= limit)
        throw new BufferOverflowException(); ➊
    return position++;
}
```
Java `ByteBuffer` 达到上限的时候需要自己手动创建一个新的 `ByteBuffer` 再存入。而对比 `ByteBuf`

`io.netty.buffer.AbstractByteBuf#writeByte`
```java
@Override
public ByteBuf writeByte(int value) {
    ensureWritable0(1);
    _setByte(writerIndex++, value);
    return this;
}

final void ensureWritable0(int minWritableBytes) {
    ensureAccessible();
    if (minWritableBytes <= writableBytes()) {
        return;
    }
    // Normalize the current capacity to the power of 2.
    int newCapacity = alloc().calculateNewCapacity(writerIndex + minWritableBytes, maxCapacity);
➋
    // Adjust to the new capacity.
    capacity(newCapacity);
}
```
在 ➋ Netty的ByteBuf 会进行自动的扩容。拓展的倍数是 2

## 池化技术
传统的 `ByteBuffer` 的生命周期是委托给 `JVM` 管理的，Netty分为2块

- 基于对象池的ByteBuf: PooledByteBuf和它的子类PoolDirectByteBuf、PoolUnsafeDirectByteBuf、PooledHeapByteBuf 
- 普通的ByteBuf: UnPoolDirectByteBuf、UnPoolUnsafeDirectByteBuf、UnPoolHeapByteBuf

基于对象池的ByteBuf 的核心逻辑就在 `PoolArena` 对象，关于 `PoolArena` 的原理可以参看
- [深入浅出Netty内存管理](https://www.jianshu.com/p/c4bd37a3555b)
- [Netty权威指南](https://book.douban.com/subject/26373138/)

## Netty的零拷贝
- DirectByteBuf: ByteBuf可以分为HeapByteBuf和DirectByteBuf，当使用DirectByteBuf可以实现零拷贝
- DefaultFileRegion: DefaultFileRegion是Netty的文件传输类，它通过transferTo方法将文件直接发送到目标Channel，而不需要循环拷贝的方式，提升了传输性能

这块都设计到java的 `native` 代码故不展开讲。

## Netty的内存回收管理
Netty会通过 引用计数法 及时申请释放不再被引用的对象 ，实现上是通过 AbstractReferenceCountedByteBuf来实现的

```java
// increment
private ByteBuf retain0(final int increment) {
    int oldRef = refCntUpdater.getAndAdd(this, increment);
    if (oldRef <= 0 || oldRef + increment < oldRef) {
        // Ensure we don't resurrect (which means the refCnt was 0) and also that we encountered an overflow.
        refCntUpdater.getAndAdd(this, -increment);
        throw new IllegalReferenceCountException(oldRef, increment);
    }
    return this;
}

// decrement
private boolean release0(int decrement) {
    int oldRef = refCntUpdater.getAndAdd(this, -decrement);
    if (oldRef == decrement) {
        deallocate();
        return true;
    } else if (oldRef < decrement || oldRef - decrement > oldRef) {
        // Ensure we don't over-release, and avoid underflow.
        refCntUpdater.getAndAdd(this, decrement);
        throw new IllegalReferenceCountException(oldRef, -decrement);
    }
    return false;
}
```
有同学会问，为什么需要`计数引用`，因为Netty采用了池化的技术，GC是无法回收这些数据的，需要自己实现一个简单的GC，技术引用是最简单的方式。