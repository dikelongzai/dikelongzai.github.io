---
title: HBase-1.2.4LruBlockCache实现分析
categories:
  - 大数据存储
  - Hbase
  
tags:
  - 大数据存储
  - Hbase
abbrlink: 33760
date: 2019-09-05 01:29:56
---
# 一、简介
      BlockCache是HBase中的一个重要特性，相比于写数据时缓存为Memstore，读数据时的缓存则为BlockCache。
      LruBlockCache是HBase中BlockCache的默认实现，它采用严格的LRU算法来淘汰Block。
# 二、缓存级别
      目前有三种缓存级别，定义在BlockPriority中，如下：
```java 
public enum BlockPriority {  
  /** 
   * Accessed a single time (used for scan-resistance) 
   */  
  SINGLE,  
  /** 
   * Accessed multiple times 
   */  
  MULTI,  
  /** 
   * Block from in-memory store 
   */  
  MEMORY  
}  
```
 ## 1、SINGLE：主要用于scan等，避免大量的这种一次的访问导致缓存替换；
 ## 2、MULTI：多次缓存；
 ## 3、MEMORY：常驻缓存的，比如meta信息等。
# 三、缓存实现分析
      LruBlockCache缓存的实现在方法cacheBlock()中，实现逻辑如下：
      1、首先需要判断需要缓存的数据大小是否超过最大块大小，按照2%的频率记录warn类型log并返回；
      2、从缓存map中根据cacheKey尝试获取已缓存数据块cb；
      3、如果已经缓存过，比对下内容，如果内容不一样，抛出异常，否则记录warn类型log并返回；
      4、获取当前缓存大小currentSize，获取可以接受的缓存大小currentAcceptableSize，计算硬性限制大小hardLimitSize；
      5、如果当前大小超过硬性限制，当回收没在执行时，执行回收并返回，否则直接返回；
      6、利用cacheKey、数据buf等构造Lru缓存数据块实例cb；
      7、将cb放置入map缓存中；
      8、元素个数原子性增1；
      9、如果新大小超过当前可以接受的大小，且未执行回收过程中，执行内存回收。
      详细代码如下，可自行阅读分析：
```java 
 // BlockCache implementation  
  
 /** 
  * Cache the block with the specified name and buffer. 
  * <p> 
  * It is assumed this will NOT be called on an already cached block. In rare cases (HBASE-8547) 
  * this can happen, for which we compare the buffer contents. 
  * @param cacheKey block's cache key 
  * @param buf block buffer 
  * @param inMemory if block is in-memory 
  * @param cacheDataInL1 
  */  
 @Override  
 public void cacheBlock(BlockCacheKey cacheKey, Cacheable buf, boolean inMemory,  
     final boolean cacheDataInL1) {  
  
// 首先需要判断需要缓存的数据大小是否超过最大块大小  
   if (buf.heapSize() > maxBlockSize) {  
     // If there are a lot of blocks that are too  
     // big this can make the logs way too noisy.  
     // So we log 2%  
     if (stats.failInsert() % 50 == 0) {  
       LOG.warn("Trying to cache too large a block "  
           + cacheKey.getHfileName() + " @ "  
           + cacheKey.getOffset()  
           + " is " + buf.heapSize()  
           + " which is larger than " + maxBlockSize);  
     }  
     return;  
   }  
  
   // 从缓存map中根据cacheKey尝试获取已缓存数据块  
   LruCachedBlock cb = map.get(cacheKey);  
   if (cb != null) {// 如果已经缓存过  
     // compare the contents, if they are not equal, we are in big trouble  
     if (compare(buf, cb.getBuffer()) != 0) {// 比对缓存内容，如果不相等，抛出异常，否则记录warn日志  
       throw new RuntimeException("Cached block contents differ, which should not have happened."  
         + "cacheKey:" + cacheKey);  
     }  
     String msg = "Cached an already cached block: " + cacheKey + " cb:" + cb.getCacheKey();  
     msg += ". This is harmless and can happen in rare cases (see HBASE-8547)";  
     LOG.warn(msg);  
     return;  
   }  
     
   // 获取当前缓存大小  
   long currentSize = size.get();  
     
   // 获取可以接受的缓存大小  
   long currentAcceptableSize = acceptableSize();  
     
   // 计算硬性限制大小  
   long hardLimitSize = (long) (hardCapacityLimitFactor * currentAcceptableSize);  
     
   if (currentSize >= hardLimitSize) {// 如果当前大小超过硬性限制，当回收没在执行时，执行回收并返回  
     stats.failInsert();  
     if (LOG.isTraceEnabled()) {  
       LOG.trace("LruBlockCache current size " + StringUtils.byteDesc(currentSize)  
         + " has exceeded acceptable size " + StringUtils.byteDesc(currentAcceptableSize) + "  too many."  
         + " the hard limit size is " + StringUtils.byteDesc(hardLimitSize) + ", failed to put cacheKey:"  
         + cacheKey + " into LruBlockCache.");  
     }  
     if (!evictionInProgress) {// 当回收没在执行时，执行回收并返回  
       runEviction();  
     }  
     return;  
   }  
     
   // 利用cacheKey、数据buf等构造Lru缓存数据块实例  
   cb = new LruCachedBlock(cacheKey, buf, count.incrementAndGet(), inMemory);  
   long newSize = updateSizeMetrics(cb, false);  
     
   // 放置入map缓存中  
   map.put(cacheKey, cb);  
     
   // 元素个数原子性增1  
   long val = elements.incrementAndGet();  
   if (LOG.isTraceEnabled()) {  
     long size = map.size();  
     assertCounterSanity(size, val);  
   }  
     
   // 如果新大小超过当前可以接受的大小，且未执行回收过程中  
   if (newSize > currentAcceptableSize && !evictionInProgress) {  
     runEviction();// 执行内存回收  
   }  
 }  
 ```
# 四、淘汰缓存实现分析
淘汰缓存的实现方式有两种：
## 1、第一种是在主线程中执行缓存淘汰；
## 2、第二种是在一个专门的淘汰线程中通过持有对外部类LruBlockCache的弱引用WeakReference来执行缓存淘汰。
应用那种方式，取决于构造函数中的boolean参数evictionThread，如下：
 ```java 
if(evictionThread) {  
  this.evictionThread = new EvictionThread(this);  
  this.evictionThread.start(); // FindBugs SC_START_IN_CTOR  
} else {  
  this.evictionThread = null;  
}  
      而在执行淘汰缓存的runEviction()方法中，有如下判断：
[java] view plain copy
/** 
 * Multi-threaded call to run the eviction process. 
 * 多线程调用以执行回收过程 
 */  
private void runEviction() {  
  if(evictionThread == null) {// 如果未指定回收线程  
    evict();  
  } else {// 如果执行了回收线程  
    evictionThread.evict();  
  }  
}  
        而EvictionThread的evict()实现如下：
[java] view plain copy
@edu.umd.cs.findbugs.annotations.SuppressWarnings(value="NN_NAKED_NOTIFY",  
    justification="This is what we want")  
public void evict() {  
  synchronized(this) {  
    this.notifyAll();  
  }  
}  
        通过synchronized获取EvictionThread线程的对象锁，然后主线程通过回收线程对象的notifyAll唤醒EvictionThread线程，那么这个线程是何时wait的呢？答案就在其run()方法中，notifyAll()之后，线程run()方法得以继续执行：
[java] view plain copy
@Override  
public void run() {  
  enteringRun = true;  
  while (this.go) {  
    synchronized(this) {  
      try {  
        this.wait(1000 * 10/*Don't wait for ever*/);  
      } catch(InterruptedException e) {  
        LOG.warn("Interrupted eviction thread ", e);  
        Thread.currentThread().interrupt();  
      }  
    }  
    LruBlockCache cache = this.cache.get();  
    if (cache == null) break;  
    cache.evict();  
  }  
}  
        线程会wait10s，放弃对象锁，在notifyAll()后，继续执行后面的淘汰流程，即：
[java] view plain copy
/** 
 * Eviction method. 
 */  
void evict() {  
  
  // Ensure only one eviction at a time  
/ 通过可重入互斥锁ReentrantLock确保同一时刻只有一个回收在执行  
  if(!evictionLock.tryLock()) return;  
  
  try {  
      
    // 标志位，是否正在进行回收过程  
    evictionInProgress = true;  
      
    // 当前缓存大小  
    long currentSize = this.size.get();  
    // 计算应该释放的缓冲大小bytesToFree  
    long bytesToFree = currentSize - minSize();  
  
    if (LOG.isTraceEnabled()) {  
      LOG.trace("Block cache LRU eviction started; Attempting to free " +  
        StringUtils.byteDesc(bytesToFree) + " of total=" +  
        StringUtils.byteDesc(currentSize));  
    }  
  
    // 如果需要回收的大小小于等于0，直接返回  
    if(bytesToFree <= 0) return;  
  
    // Instantiate priority buckets  
    // 实例化优先级队列：single、multi、memory  
    BlockBucket bucketSingle = new BlockBucket("single", bytesToFree, blockSize,  
        singleSize());  
    BlockBucket bucketMulti = new BlockBucket("multi", bytesToFree, blockSize,  
        multiSize());  
    BlockBucket bucketMemory = new BlockBucket("memory", bytesToFree, blockSize,  
        memorySize());  
  
    // Scan entire map putting into appropriate buckets  
    // 扫描缓存，分别加入上述三个优先级队列  
    for(LruCachedBlock cachedBlock : map.values()) {  
      switch(cachedBlock.getPriority()) {  
        case SINGLE: {  
          bucketSingle.add(cachedBlock);  
          break;  
        }  
        case MULTI: {  
          bucketMulti.add(cachedBlock);  
          break;  
        }  
        case MEMORY: {  
          bucketMemory.add(cachedBlock);  
          break;  
        }  
      }  
    }  
  
    long bytesFreed = 0;  
    if (forceInMemory || memoryFactor > 0.999f) {// 如果memoryFactor或者InMemory缓存超过99.9%，  
      long s = bucketSingle.totalSize();  
      long m = bucketMulti.totalSize();  
      if (bytesToFree > (s + m)) {// 如果需要回收的缓存超过则全部回收Single、Multi中的缓存大小和，则全部回收Single、Multi中的缓存，剩余的则从InMemory中回收  
        // this means we need to evict blocks in memory bucket to make room,  
        // so the single and multi buckets will be emptied  
        bytesFreed = bucketSingle.free(s);  
        bytesFreed += bucketMulti.free(m);  
        if (LOG.isTraceEnabled()) {  
          LOG.trace("freed " + StringUtils.byteDesc(bytesFreed) +  
            " from single and multi buckets");  
        }  
        // 剩余的则从InMemory中回收  
        bytesFreed += bucketMemory.free(bytesToFree - bytesFreed);  
        if (LOG.isTraceEnabled()) {  
          LOG.trace("freed " + StringUtils.byteDesc(bytesFreed) +  
            " total from all three buckets ");  
        }  
      } else {// 否则，不需要从InMemory中回收，按照如下策略回收Single、Multi中的缓存：尝试让single-bucket和multi-bucket的比例为1:2  
        // this means no need to evict block in memory bucket,  
        // and we try best to make the ratio between single-bucket and  
        // multi-bucket is 1:2  
        long bytesRemain = s + m - bytesToFree;  
        if (3 * s <= bytesRemain) {// single-bucket足够小，从multi-bucket中回收  
          // single-bucket is small enough that no eviction happens for it  
          // hence all eviction goes from multi-bucket  
          bytesFreed = bucketMulti.free(bytesToFree);  
        } else if (3 * m <= 2 * bytesRemain) {// multi-bucket足够下，从single-bucket中回收  
          // multi-bucket is small enough that no eviction happens for it  
          // hence all eviction goes from single-bucket  
          bytesFreed = bucketSingle.free(bytesToFree);  
        } else {  
        // single-bucket和multi-bucket中都回收，且尽量满足回收后比例为1:2  
          // both buckets need to evict some blocks  
          bytesFreed = bucketSingle.free(s - bytesRemain / 3);  
          if (bytesFreed < bytesToFree) {  
            bytesFreed += bucketMulti.free(bytesToFree - bytesFreed);  
          }  
        }  
      }  
    } else {// 否则，从三个队列中循环回收  
      PriorityQueue<BlockBucket> bucketQueue =  
        new PriorityQueue<BlockBucket>(3);  
  
      bucketQueue.add(bucketSingle);  
      bucketQueue.add(bucketMulti);  
      bucketQueue.add(bucketMemory);  
  
      int remainingBuckets = 3;  
  
      BlockBucket bucket;  
      while((bucket = bucketQueue.poll()) != null) {  
        long overflow = bucket.overflow();  
        if(overflow > 0) {  
          long bucketBytesToFree = Math.min(overflow,  
              (bytesToFree - bytesFreed) / remainingBuckets);  
          bytesFreed += bucket.free(bucketBytesToFree);  
        }  
        remainingBuckets--;  
      }  
    }  
  
    if (LOG.isTraceEnabled()) {  
      long single = bucketSingle.totalSize();  
      long multi = bucketMulti.totalSize();  
      long memory = bucketMemory.totalSize();  
      LOG.trace("Block cache LRU eviction completed; " +  
        "freed=" + StringUtils.byteDesc(bytesFreed) + ", " +  
        "total=" + StringUtils.byteDesc(this.size.get()) + ", " +  
        "single=" + StringUtils.byteDesc(single) + ", " +  
        "multi=" + StringUtils.byteDesc(multi) + ", " +  
        "memory=" + StringUtils.byteDesc(memory));  
    }  
  } finally {  
    // 重置标志位，释放锁等  
    stats.evict();  
    evictionInProgress = false;  
    evictionLock.unlock();  
  }  
}  
```
逻辑比较清晰，如下：
1、通过可重入互斥锁ReentrantLock确保同一时刻只有一个回收在执行；  
2、设置标志位evictionInProgress，是否正在进行回收过程为true；  
3、获取当前缓存大小currentSize；  
4、计算应该释放的缓冲大小bytesToFree：currentSize - minSize()； 
5、如果需要回收的大小小于等于0，直接返回；  
6、实例化优先级队列：single、multi、memory；  
7、扫描缓存，分别加入上述三个优先级队列； 
8、如果forceInMemory或者InMemory缓存超过99.9%：  
  &#8194; 8.1、如果需要回收的缓存超过则全部回收Single、Multi中的缓存大小和，则全部回收Single、Multi中的缓存，剩余的则从InMemory中回收（this means we need to evict blocks in memory bucket to make room,so the single and multi buckets will be emptied）：
  &#8194; 8.2、否则，不需要从InMemory中回收，按照如下策略回收Single、Multi中的缓存：尝试让single-bucket和multi-bucket的比例为1:2：
       &emsp;8.2.1、 single-bucket足够小，从multi-bucket中回收； 
       &emsp;8.2.2、 multi-bucket足够小，从single-bucket中回收；  
       &emsp;8.2.3、single-bucket和multi-bucket中都回收，且尽量满足回收后比例为1:2；  
9、否则，从三个队列中循环回收；  
10、最后，重置标志位，释放锁等。
