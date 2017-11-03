---
title: 关于Netty的OutOfDirectMemoryError
date: 2017-11-03 15:15:19
tags: [Netty]
categories: 技术
---

最近帮一个业务排查问题，发现业务日志里面好多下面的错误：

```
io.netty.util.internal.OutOfDirectMemoryError: failed to allocate 16777216 byte(s) of direct memory (used: 520093983, max: 536870912)
```

看日志就知道是Direct Buffer不够用了（512M），业务发送的数据有点大（16M）。看完日志的时候，想看看这个错误到底是哪儿抛出来的，然后就发现一点比较有意思的事情。

首先，这个类：***io.netty.util.internal.OutOfDirectMemoryError***,是在Netty4.1之后才有的。详情可以看看 ***PlatformDependent*** 这个类。

### 4.1之前
在4.1之前，这个类的主要方法是：

```
PlatformDependent.java
public static long allocateMemory(long size) {
    return PlatformDependent0.allocateMemory(size);
}
public static void freeDirectBuffer(ByteBuffer buffer) {
....
}

```

看看4.0.24里面的UnpooledDirectByteBuf：
```
UnpooledDirectByteBuf.java
protected ByteBuffer allocateDirect(int initialCapacity) {
    return ByteBuffer.allocateDirect(initialCapacity);
}
```

可以看到，4.0.24里面，分配DirectBuffer是使用Java里面的方法，这个方法最终会调用到:
```
DirectByteBuffer.java
DirectByteBuffer(int cap) {  
	....
	cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
}
```
Java DirectByteBuffer的这个构造方法里面，会构造一个带Cleaner机制的DirectBuffer。

那4.1之前，PlatformDependent.java里面的allocateMemory是谁在用呢？跟一下代码，发现只有**IovArray**在用，而这个类基本上被NativeDatagramPacket.java在使用，也就是说，Netty在4.1之前，基本上只有底层的通信模块使用了PlatformDependent.java里面的allocateMemory。 这有什么问题呢？

通过代码很容易看到，这块内存是没有限制的，直接调用底层的Unsafe来分配Direct内存，好在这块内存只在NativeDatagramPacket里面被用到，不会扩散乱用，看NativeDatagramPacket的注释也是这么说的:

```
static final class NativeDatagramPacket {
// Each NativeDatagramPackets holds a IovArray which is used for gathering writes.
// This is ok as NativeDatagramPacketArray is always obtained via a FastThreadLocal and
// so the memory needed is quite small anyway.
private final IovArray array = new IovArray();
```

### 4.1之后

4.1之后，还保留了之前的函数，但是增加了一个：

```
public static ByteBuffer allocateDirectNoCleaner(int capacity) {
	assert USE_DIRECT_BUFFER_NO_CLEANER;

	incrementMemoryCounter(capacity);
	try {
	    return PlatformDependent0.allocateDirectNoCleaner(capacity);
	} catch (Throwable e) {
	    decrementMemoryCounter(capacity);
	    throwException(e);
	    return null;
	}
}

private static void incrementMemoryCounter(int capacity) {
    if (DIRECT_MEMORY_COUNTER != null) {
        for (;;) {
            long usedMemory = DIRECT_MEMORY_COUNTER.get();
            long newUsedMemory = usedMemory + capacity;
            if (newUsedMemory > DIRECT_MEMORY_LIMIT) {
                throw new OutOfDirectMemoryError("failed to allocate " + capacity
                        + " byte(s) of direct memory (used: " + usedMemory + ", max: " + DIRECT_MEMORY_LIMIT + ')');
            }
            if (DIRECT_MEMORY_COUNTER.compareAndSet(usedMemory, newUsedMemory)) {
                break;
            }
        }
    }
}
```

4.1之后，Netty增加了一个方法，可以分配NoCleaner的DirectBuffer，Cleaner前面说了，就是Java DirectBuffer默认的构造函数里面，会给每个DirectBuffer附加一个Cleaner，该Cleaner是一个**PhantomReference**, 当DirectBuffer没有强引用后，会出发clean机制。

Netty4.1之后的DirectBuffer很明显，绕开了Cleaner（当然，有一些前提的判断），给开发造成的影响就是：

* 新的错误：**io.netty.util.internal.OutOfDirectMemoryError** 而不是 **OutOfMemoryError**.
* 当directBuffer内存不够的时候，不会出发System.gc了。这一点还是比较重要的，不要老是盯着gc日志看问题了。具体可以在incrementMemoryCounter这个方法看到，内存达到限制了，直接抛出错误
* 既然没有了Cleanner来抄底负责回收内存，那么，上层一定要自己记着收回DirectBuffer

翻代码还是有收获的，当然，这些只是解决问题之后看到的，至于业务的问题，就需要他们在write的时候，添加流控措施，避免到达内存限制。