# LruCacheStudy
# LruCache 源码解析

标签（空格分隔）： 源码解读

---
###1、LRU简介
LRU 是 Least Recently Used 最近最少使用算法。

曾经，在各大缓存图片的框架没流行的时候。有一种很常用的内存缓存技术：SoftReference 和 WeakReference（软引用和弱引用）。但是走到了 Android 2.3（Level 9）时代，垃圾回收机制更倾向于回收 SoftReference 或 WeakReference 的对象。后来，又来到了 Android3.0，图片缓存在内容中，因为不知道要在是什么时候释放内存，没有策略，没用一种可以预见的场合去将其释放。这就造成了内存溢出。
###2、使用方法




###附带源码分析
```
package android.support.v4.util;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Static library version of {@link android.util.LruCache}. Used to write apps
 * that run on API levels prior to 12. When running on API level 12 or above,
 * this implementation is still used; it does not try to switch to the
 * framework's implementation. See the framework SDK documentation for a class
 * overview.
 */
public class LruCache<K, V> {
    private final LinkedHashMap<K, V> map;

    /** Size of this cache in units. Not necessarily the number of elements. */
    private int size;  //已经存储的数据大小
    private int maxSize;  //存储的最大数据大小

    private int putCount;  //调用put的次数
    private int createCount; //调用create的次数
    private int evictionCount;  //收回的次数
    private int hitCount;  //命中的次数（取出数据成功的次数）
    private int missCount;  //丢失的次数（取出数据丢失的次数）

    /**
     * @param maxSize for caches that do not override {@link #sizeOf}, this is
     *     the maximum number of entries in the cache. For all other caches,
     *     this is the maximum sum of the sizes of the entries in this cache.
     *     构造方法   传入的最大的容量
     */
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
       /*
         * 初始化LinkedHashMap
         * 第一个参数：initialCapacity，初始大小
         * 第二个参数：loadFactor，负载因子=0.75f
         * 第三个参数：accessOrder=true，基于访问顺序；accessOrder=false，基于插入顺序
         */
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }

    /**
     * Sets the size of the cache.
     *  设置缓存的大小
     * @param maxSize The new maximum size.   最大缓存大小
     */
    public void resize(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }

        synchronized (this) {    //线程安全
            this.maxSize = maxSize;
        }
        trimToSize(maxSize);  //重整数据
    }

    /**
     * Returns the value for {@code key} if it exists in the cache or can be
     * created by {@code #create}. If a value was returned, it is moved to the
     * head of the queue. This returns null if a value is not cached and cannot
     * be created.
     *
     * 
     *  根据key值取数据如果存在了或者被create了
     *  返回一个value  那么他会被移动到双向链表的尾部
     *  如果没有value 返回null 
     */
    public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;   //碰撞次数   （获取到数据的次数）
                return mapValue; //返回value
            }
            missCount++;  //如果没有找到    获取到数据  失败的次数
        }

        /*
         * Attempt to create a value. This may take a long time, and the map
         * may be different when create() returns. If a conflicting value was
         * added to the map while create() was working, we leave that value in
         * the map and release the created value.
         *    
         * 这个方法将会花费一个比较长的时间   并且这个map会被创建的value也是不同的  如果
         * 在创建的时候一个冲突的值被创建   我们map将会删除这个值  并且释放被创造的值
         */

        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

          /***************************
         * 不覆写create方法走不到下面 *
         ***************************/


        synchronized (this) {
            //记录create的次数
            createCount++;
            // 将自定义create创建的值，放入LinkedHashMap中，如果key已经存在，会返回 之前相同key 的值
            mapValue = map.put(key, createdValue);  

            if (mapValue != null) {
                // There was a conflict so undo that last put
                /*
                 * 有冲突
                 * 所以 撤销 刚才的 操作
                 * 将 之前相同key 的值 重新放回去
                 */
                map.put(key, mapValue);
            } else {
                //拿到键值   并且计算出该值得大小  然后size加上所计算出的值得大小
                size += safeSizeOf(key, createdValue);
            }
        }

        if (mapValue != null) {  //如果上面已经走通了  判断出了键值   并且也是有冲突了
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
            trimToSize(maxSize);  //重新调整size大小
            return createdValue;
        }
    }

    /**
     * Caches {@code value} for {@code key}. The value is moved to the head of
     * the queue.
     *
     * 给对应key缓存value，该value将被移动到队头。
     *
     * @return the previous value mapped by {@code key}.
     */
    public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;  //记录put的个数
            size += safeSizeOf(key, value);  //根据key拿到键值   计算出大小  
            previous = map.put(key, value);
            if (previous != null) {
            // 计算出 冲突键值 在容量中的相对长度，然后减去
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            /*
             * previous值被剔除了，此次添加的 value 已经作为key的 新值
             * 告诉 自定义 的 entryRemoved 方法
             */
            entryRemoved(false, key, previous, value);
        }

        trimToSize(maxSize);
        return previous;
    }

    /**
     * Remove the eldest entries until the total of remaining entries is at or
     * below the requested size.
     * 
     * 删除旧的数据 直到总的数据大小低于或者等于已经请求得到的size大小
     *
     * @param maxSize the maximum size of the cache before returning. May be -1
     *            to evict even 0-sized elements.
     */
    public void trimToSize(int maxSize) {
        /*
         * 这是一个死循环，
         * 1.只有 扩容 的情况下能立即跳出
         * 2.非扩容的情况下，map的数据会一个一个删除，直到map里没有值了，就会跳出
         */
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }
                // 如果是 扩容 或者 map为空了，就会中断，因为扩容不会涉及到丢弃数据的情况
                if (size <= maxSize || map.isEmpty()) {
                    break;
                }
                //
                Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                // 拿到键值对，计算出在容量中的相对长度，然后减去。
                size -= safeSizeOf(key, value);
                evictionCount++;  //添加一次收回次数
            }
            /*
             * 将最后一次删除的最少访问数据回调出去
             */
            entryRemoved(true, key, value, null);
        }
    }

    /**
     * Removes the entry for {@code key} if it exists.
     *
     * @return the previous value mapped by {@code key}.
     */
    public final V remove(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V previous;
        synchronized (this) {
            previous = map.remove(key);
            if (previous != null) {
                size -= safeSizeOf(key, previous);
            }
        }

        /**
         *   如果 Map 中存在 该key ，并且成功移除了
         */
        if (previous != null) {
            /*
             * 会通知 自定义的 entryRemoved
             * previous 已经被删除了
             */
            entryRemoved(false, key, previous, null);
        }

        return previous;
    }

    /**
     * Called for entries that have been evicted or removed. This method is
     * invoked when a value is evicted to make space, removed by a call to
     * {@link #remove}, or replaced by a call to {@link #put}. The default
     * implementation does nothing.
     *
     * <p>The method is called without synchronization: other threads may
     * access the cache while this method is executing.
     *
     * @param evicted true if the entry is being removed to make space, false
     *     if the removal was caused by a {@link #put} or {@link #remove}.
     * @param newValue the new value for {@code key}, if it exists. If non-null,
     *     this removal was caused by a {@link #put}. Otherwise it was caused by
     *     an eviction or a {@link #remove}.
     *
     *
     * 1.当被回收或者删掉时调用。该方法当value被回收释放存储空间时被remove调用
     * 或者替换条目值时put调用，默认实现什么都没做。
     * 2.该方法没用同步调用，如果其他线程访问缓存时，该方法也会执行。
     * 3.evicted=true：如果该条目被删除空间 （表示 进行了trimToSize or remove）  evicted=false：put冲突后 或 get里成功create后
     * 导致
     * 4.newValue!=null，那么则被put()或get()调用。
     *
     */
    protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {}

    /**
     * Called after a cache miss to compute a value for the corresponding key.
     * Returns the computed value or null if no value can be computed. The
     * default implementation returns null.
     *
     * <p>The method is called without synchronization: other threads may
     * access the cache while this method is executing.
     *
     * <p>If a value for {@code key} exists in the cache when this method
     * returns, the created value will be released with {@link #entryRemoved}
     * and discarded. This can occur when multiple threads request the same key
     * at the same time (causing multiple values to be created), or when one
     * thread calls {@link #put} while another is creating a value for the same
     * key.
     *
     *
     * 1.缓存丢失之后计算相应的key的value后调用。
     * 返回计算后的值，如果没有value可以计算返回null。
     * 默认的实现返回null。
     * 2.该方法没用同步调用，如果其他线程访问缓存时，该方法也会执行。
     * 3.当这个方法返回的时候，如果对应key的value存在缓存内，被创建的value将会被entryRemoved()释放或者丢弃。
     * 这情况可以发生在多线程在同一时间上请求相同key（导致多个value被创建了），或者单线程中调用了put()去创建一个
     * 相同key的value
     *
     */
    protected V create(K key) {
        return null;
    }

     /**
     * 计算 该 键值对 的相对长度
     * 如果不覆写 sizeOf 实现特殊逻辑的话，默认长度是1。
     */
    private int safeSizeOf(K key, V value) {
        int result = sizeOf(key, value);
        if (result < 0) {
            throw new IllegalStateException("Negative size: " + key + "=" + value);
        }
        return result;
    }

    /**
     * Returns the size of the entry for {@code key} and {@code value} in
     * user-defined units.  The default implementation returns 1 so that size
     * is the number of entries and max size is the maximum number of entries.
     *
     * <p>An entry's size must not change while it is in the cache.
     *
     *
     * 返回条目在用户定义单位的大小。默认实现返回1，这样的大小是条目的数量并且最大的大小是条目的最大数量。
     * 一个条目的大小必须不能在缓存中改变
     */
     */
    protected int sizeOf(K key, V value) {
        return 1;
    }

    /**
     * Clear the cache, calling {@link #entryRemoved} on each removed entry.
     */
    public final void evictAll() {
        trimToSize(-1); // -1 will evict 0-sized elements
    }

    /**
     * For caches that do not override {@link #sizeOf}, this returns the number
     * of entries in the cache. For all other caches, this returns the sum of
     * the sizes of the entries in this cache.
     *
     * 对于这个缓存，如果不覆写sizeOf()方法，这个方法返回的是条目的在缓存中的数量。但是对于其他缓存，返回的是
     * 条目在缓存中大小的总和。
     */
     */
    public synchronized final int size() {
        return size;
    }

    /**
     * For caches that do not override {@link #sizeOf}, this returns the maximum
     * number of entries in the cache. For all other caches, this returns the
     * maximum sum of the sizes of the entries in this cache.
     */
    public synchronized final int maxSize() {
        return maxSize;
    }

    /**
     * Returns the number of times {@link #get} returned a value that was
     * already present in the cache.
     */
    public synchronized final int hitCount() {
        return hitCount;
    }

    /**
     * Returns the number of times {@link #get} returned null or required a new
     * value to be created.
     */
    public synchronized final int missCount() {
        return missCount;
    }

    /**
     * Returns the number of times {@link #create(Object)} returned a value.
     */
    public synchronized final int createCount() {
        return createCount;
    }

    /**
     * Returns the number of times {@link #put} was called.
     */
    public synchronized final int putCount() {
        return putCount;
    }

    /**
     * Returns the number of values that have been evicted.
     */
    public synchronized final int evictionCount() {
        return evictionCount;
    }

    /**
     * Returns a copy of the current contents of the cache, ordered from least
     * recently accessed to most recently accessed.
     */
    public synchronized final Map<K, V> snapshot() {
        return new LinkedHashMap<K, V>(map);
    }

    @Override public synchronized final String toString() {
        int accesses = hitCount + missCount;
        int hitPercent = accesses != 0 ? (100 * hitCount / accesses) : 0;
        return String.format("LruCache[maxSize=%d,hits=%d,misses=%d,hitRate=%d%%]",
                maxSize, hitCount, missCount, hitPercent);
    }
}

```




