---
title: Android对象池设计原理概述
author: lidu
date: 2018-04-03 21：35
tags: Android
---

# 对象池是什么

>“一个对象池包含一组已经初始化过且可以使用的对象。池的用户可以从池子中取得对象，对其进行操作处理，并在不需要时归还给池子而非直接销毁它。这是一种特殊的工厂对象。” 		--维基百科

对象池（Object Pool）是编程中常见的一种设计模式，常用于以下两类场景：（1）需要频繁创建某类对象，如字节流中的byte数组；（2）当创建某类对象的开销很大，如线程。

我们知道，一个Java对象典型的生命周期包含三个历程，分别是对象的创建（T1）、对象的使用（T2）、对象的销毁（T3）。其中T2是对象实际的有效使用时间，而T1和T3则是对象本身的开销。如果我们利用对象池技术把某些常用对象缓存在池子中，当需要使用这类对象时，不是通过new关键字新创建而是通过池子来获取，可以有效避免创建对象的时间开销。同样的道理，当某个对象不再使用后，我们将其放回对象池中，而不是即刻销毁，可以避免销毁对象的时间开销。通过有效避免T1和T3的时间开销，我们获取对象的速度可以获得显著提高，程序性能也因而得以提高。

# 从Message.obatin()说起

下面从一个最基础的Android面试题开始讲起对象池技术：
> 我们在使用Handler发送消息时，为什么不new一个Message，而是使用Message.obtain()方法来获取Message呢？

相信每一位做过Android开发的童鞋都能异口同声答出：
> 因为Handler机制中的Message利用了对象池技术，使用过的Message可以被暂存到Message Pool中。利用Message.obtain()方法就是从这个Message Pool中取可以被复用的Message，避免重新创建Message对象，提高获取Message对象的速度。

这么答当然没毛病，算是很正规的标准答案。但是我们能否更加深入思考一下，为什么Message要用对象池技术去实现呢？

按照我的理解，Message作为Android主线程消息循环中承载信息的重要媒介，在界面刷新、事件分发、Activity跳转等事务处理中需要被频繁创建、消耗、销毁的，这类对象符合对象池应用场景中的***对象被频繁创建***。将Message对象缓存进Message Pool中，可以提高获取Message对象的响应速度，从而显著提高主线程的响应速度，极大提高了用户操作的流畅度。

查看Message类的源码，我们可以知道Message Pool中Message是以单链表的方式存储的，其中***sPool***指向当前单链表的头结点，如下图：

![](https://ws1.sinaimg.cn/large/dc50da5fly1fpzq0s65kej20p908s74t.jpg)

从池子中取出Message对象，本质上就是取出***sPool***指向的Message，也就是单链表的头结点删除问题，如下图：

![](https://ws1.sinaimg.cn/large/dc50da5fly1fpzq3cphkaj20r2095dgi.jpg)

将使用完的Meesage存进池子中，也就是将当前Message添加到单链表的头结点，不赘述。

这里贴下相应代码：

```java
// 从Message Pool中取Meesgae
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}

// 向Message Pool中存Message
void recycleUnchecked() {
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    //…各种清除操作，保证Message复用后数据是干净的

    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```

其实这里还有一个问题值得我们思考一下：
>Message Pool中池子元素为什么以单链表形式组合，可以使用数组来实现吗？

答案当然是可以用数组实现。我们只需要在Message类中创建一个size为**MAX_POOL_SIZE**的static的Object数组，然后可以有两种方式来实现Message对象的存与取操作：

1. 时间复杂度为O(n)的实现
> 取元素-遍历数组，找到第一个不为null的元素，取出  
> 存元素-遍历数组，找到第一个为null的index值，存入

2. 时间复杂度为O(1)的实现
> 额外增加一个sPoolSize的int值，指示当前池子缓存元素的个数  
> 取元素-直接取数组中index为sPoolSize - 1的元素即可  
> 存元素-向数组后面存入，即存入数组index为sPoolSize的位置

对比一下单链表和数组的实现方式，我们发现单链表和优化后的数组实现方式在时间复杂度上都可以达到O(1)，但是单链表在空间使用上更有优势，可以动态拓展元素值，而数组则是在初始化时就分配了一个较大的对象数组。

# 对象池的优点和注意事项

对象池的优点可以概括为以下三点：

1. 空间换时间，提升了调用者获取对象的响应速度
2. 一定程度上可以减轻GC压力
3. 对于实时性要求较高的程序有很大帮助（如Message Pool）

同时在设计对象池时，我们需要注意以下几点：

1. 生命周期的问题，池子元素生命周期一般较长，如何合理维护池子对象占用内存
2. 池子容量和池子容器的选择，池子元素“饿汉”与“懒汉”加载
3. 如何实现多线程同步
4. 如何通过泛型机制设计更通用对象池系统

# 如何设计更通用的对象池

这里我们透过Fresco来看如何设计更加通用的对象池系统**（注意配合源码食用更佳）**：

![](https://ws1.sinaimg.cn/large/dc50da5fly1fpzqra1kecj22jo1yuq9x.jpg)

Fresco通过泛型和继承构建了易拓展的通用对象池。其中***Pool<V>***是对象池最基础的接口，有两个待实现的方法，一是指定size获得泛型对象V，二是将对象V归还至池子中。BasePool是实现了Pool接口的抽象类，它承载了对象池中最核心的工作-管理池子对象和实现get、release方法。再往下就是真正的实现类，比如实现Bitmap缓存的BitmapPool、实现Android native memory缓存的NativeMemoryChunkPool、实现字节数组缓存的GenericByteArrayPool等等。

Fresco对象池系统最强大的地方是它实现了一套基于多桶结构的通用对象池，如下图：

![](https://ws1.sinaimg.cn/large/dc50da5fly1fpzr1c6ud3j22jo1yutd6.jpg)

一个缓存泛型对象V的BasePool，其内部是一个多桶（Bucket）结构。BasePool持有一个SparseArray，map的key值是就是单个Bucket内的item size，map的value就是这个Bucket。每个Bucket内会有一个mFreeList存储可以被使用的缓存对象队列，然后有一个itemSize值指明队列中元素V的大小，比如说16K，32K等等，每个Bucket各不相同。这样设计的好处在于可以解决碎片化问题。比如说我们在网络字节流中经常需要byte数组，一般是分配16K的byte数组，如果没有多桶结构，那么缓存池中全部存放16K大小的byte数组就OK了。但是一个问题是，如果某些情况下需要20K大小的byte数组怎么办？如果没有多桶结构，我们只需要返回两个16K的byte数组就好了，但是这样造成的结果是32-20=12K的空间没有被利用起来，被我们白白浪费掉了。如果利用多桶结构的对象池，可以有效解决这个问题，我们设置多个Bucket，在这个case下，比如说我们就设置两个Bucket，Bucket0存放16K大小的byte数组，Bucket1存放20K大小的byte数组，调用者需要20K的byte数组，可以直接取Bucket1的20K的byte数组，而无须取两个16K的byte数组，能够有效提高内存使用的效率。这一点在Fresco中尤为常见，Fresco中有很大一部分是操作Android中的native memory，而不是JVM中的堆内存。由于native memory没有垃圾回收，且可分配的空间通常比heap memory要多很多，所以Fresco可以任性地使用native memory，只要手动管理好内存，在该free掉的时候及时释放即可。

这个多桶结构可以由使用者通过PoolParams来设置，我们来看一个典型的native memory池的PoolParams：

```java
// DefaultNativeMemoryChunkPoolParams.java
private static final int SMALL_BUCKET_LENGTH = 5;
private static final int LARGE_BUCKET_LENGTH = 2;

public static PoolParams get() {
  SparseIntArray DEFAULT_BUCKETS = new SparseIntArray();
  DEFAULT_BUCKETS.put(1 * ByteConstants.KB, SMALL_BUCKET_LENGTH);
  DEFAULT_BUCKETS.put(2 * ByteConstants.KB, SMALL_BUCKET_LENGTH);
  DEFAULT_BUCKETS.put(4 * ByteConstants.KB, SMALL_BUCKET_LENGTH);
  DEFAULT_BUCKETS.put(8 * ByteConstants.KB, SMALL_BUCKET_LENGTH);
  DEFAULT_BUCKETS.put(16 * ByteConstants.KB, SMALL_BUCKET_LENGTH);
  DEFAULT_BUCKETS.put(32 * ByteConstants.KB, SMALL_BUCKET_LENGTH);
  DEFAULT_BUCKETS.put(64 * ByteConstants.KB, SMALL_BUCKET_LENGTH);
  DEFAULT_BUCKETS.put(128 * ByteConstants.KB, SMALL_BUCKET_LENGTH);
  DEFAULT_BUCKETS.put(256 * ByteConstants.KB, LARGE_BUCKET_LENGTH);
  DEFAULT_BUCKETS.put(512 * ByteConstants.KB, LARGE_BUCKET_LENGTH);
  DEFAULT_BUCKETS.put(1024 * ByteConstants.KB, LARGE_BUCKET_LENGTH);
  return new PoolParams(
      getMaxSizeSoftCap(),
      getMaxSizeHardCap(),
      DEFAULT_BUCKETS);
}
```

可以看到这个池子中有N个Bucket，每个Bucket内的对象分别是1KB、2KB、4KB、...1024KB大小，这里的大小也就是上面我们提到的Bucket内的mItemSize值了。由于BasePool内使用SparseArray来Map这个itemSize和Bucket，所以取元素的时候很容易通过itemSize来找到Bucket。在SparseArray中，key值是递增的，所以通过二分查找可以在O(logN)时间内找到Bucket，而不是普通数组实现的O(n)。当然这里没有用HashMap主要是考虑到Android设备内存吃紧，用空间换时间来避免HashMap中Entry对象的内存创建和Key值的自动装箱。

![](https://ws1.sinaimg.cn/large/dc50da5fly1fpzrun7y0mj22jo1yuq7j.jpg)

我们看一下BasePool取元素的get方法，首先是通过itemSize找到对应Bucket，然后从Bucket从取value，如果不为null，返回该值即可，如果为null，会先考虑池子中内存占用情况判断是否可以新创建对象，如果可以，调用子类方法alloc()，然后返回之。

canAllocate()方法的源码如下，这里BasePool有两个阈值，一个是较大的阈值HardCap，另一个是较小的阈值SoftCap。如果要分配的对象加已使用的大小超出HardCap，会直接返回false。如果没有超过HardCap，会判断当前要分配的大小加池子中对外提供的对象（包含在外使用的和缓存在池子中的）大小总和是否超过SoftCap，如果超过，做一次trim操作，将池子中缓存对象free掉一部分，不管是通过GC还是native memory中的释放，或者是bitmap.recycle()，最终缓存对象不在池子中而是被系统回收，从而将内存占用降低下来。

```java
 synchronized boolean canAllocate(int sizeInBytes) {
    int hardCap = mPoolParams.maxSizeHardCap;

    // even with our best effort we cannot ensure hard cap limit.
    // Return immediately - no point in trimming any space
    if (sizeInBytes > hardCap - mUsed.mNumBytes) {
      mPoolStatsTracker.onHardCapReached();
      return false;
    }

    // trim if we need to
    int softCap = mPoolParams.maxSizeSoftCap;
    if (sizeInBytes > softCap - (mUsed.mNumBytes + mFree.mNumBytes)) {
      trimToSize(softCap - sizeInBytes);
    }

    // check again to see if we're below the hard cap
    if (sizeInBytes > hardCap - (mUsed.mNumBytes + mFree.mNumBytes)) {
      mPoolStatsTracker.onHardCapReached();
      return false;
    }

    return true;
  }
```

```java
synchronized void trimToSize(int targetSize) {
    // find how much we need to free
    int bytesToFree = Math.min(mUsed.mNumBytes + mFree.mNumBytes - targetSize, mFree.mNumBytes);
    if (bytesToFree <= 0) {
      return;
    }
    if (FLog.isLoggable(FLog.VERBOSE)) {
      FLog.v(
          TAG,
          "trimToSize: TargetSize = %d; Initial Size = %d; Bytes to free = %d",
          targetSize,
          mUsed.mNumBytes + mFree.mNumBytes,
          bytesToFree);
    }
    logStats();

    // now walk through the buckets from the smallest to the largest. Keep freeing things
    // until we've gotten to what we want
    for (int i = 0; i < mBuckets.size(); ++i) {
      if (bytesToFree <= 0) {
        break;
      }
      Bucket<V> bucket = mBuckets.valueAt(i);
      while (bytesToFree > 0) {
        V value = bucket.pop();
        if (value == null) {
          break;
        }
        free(value);
        bytesToFree -= bucket.mItemSize;
        mFree.decrement(bucket.mItemSize);
      }
    }

    // dump stats at the end
    logStats();
    if (FLog.isLoggable(FLog.VERBOSE)) {
      FLog.v(
          TAG,
          "trimToSize: TargetSize = %d; Final Size = %d",
          targetSize,
          mUsed.mNumBytes + mFree.mNumBytes);
    }
  }
```

```java
/**
   * (Try to) trim the pool until its total space falls below the max size (soft cap). This will
   * get rid of values on the free list, until the free lists are empty, or we fall below the
   * max size; whichever comes first.
   * NOTE: It is NOT an error if we have eliminated all the free values, but the pool is still
   * above its max size (soft cap)
   * <p>
   * The approach we take is to go from the smallest sized bucket down to the largest sized
   * bucket. This may seem a bit counter-intuitive, but the rationale is that allocating
   * larger-sized values is more expensive than the smaller-sized ones, so we want to keep them
   * around for a while.
   * @param targetSize target size to trim to
   */
synchronized void trimToSize(int targetSize) {
    // find how much we need to free
    int bytesToFree = Math.min(mUsed.mNumBytes + mFree.mNumBytes - targetSize, mFree.mNumBytes);
    if (bytesToFree <= 0) {
      return;
    }

    // now walk through the buckets from the smallest to the largest. Keep freeing things
    // until we've gotten to what we want
    for (int i = 0; i < mBuckets.size(); ++i) {
      if (bytesToFree <= 0) {
        break;
      }
      Bucket<V> bucket = mBuckets.valueAt(i);
      while (bytesToFree > 0) {
        V value = bucket.pop();
        if (value == null) {
          break;
        }
        free(value);
        bytesToFree -= bucket.mItemSize;
        mFree.decrement(bucket.mItemSize);
      }
    }
  }
```
我们最后看一下向BasePool中存元素的release方法，同样先是通过value的size找到对应的Bucket，然后会试着从mInUseValue这个Set中remove掉这个value，如果返回true，说明成功remove掉，这个元素是从池子中取出去的（get方法返回前会把元素存入mInUseValue），然后进行一系列判断，如果符合四个条件之一，说明不适合继续往池子存元素，就直接free让系统回收掉，否则存入对应Bucket中。如果remove方法返回false，说明该元素根本就是从池子从取走的，池子也不再接受它，直接让系统回收即可。

![](https://ws1.sinaimg.cn/large/dc50da5fly1fpzs98sgfuj22jo1yu433.jpg)

# Fresco通用对象池系统总结

1. 最核心、繁重的工作都在BasePool类完成，子类只需实现几个抽象方法即可，易于拓展
2. 通过PoolParams可以设置多桶结构对象池，调用者请求不同大小的对象时，会根据ItemSize从合适的桶中取出对象
3. 通过PoolParams可以设置SoftCap和HardCap，及时清理池子无用对象，避免长时间占用较高内存
4. 多次利用synchronized来实现多线程同步

