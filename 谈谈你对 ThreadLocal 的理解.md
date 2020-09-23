ThreadLocal 对有些同学来说可能有点陌生，今天我们就一起来学习一下。

##  What

ThreadLocal 直译就是**线程本地变量**，意思是 ThreadLocal 中填充的变量属于**当前**线程，该变量对其他线程而言是隔离的。ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。

## How

ThreadLocal 最主要的作用就是线程隔离，在 Android 的消息机制中，很重要的一个角色 Looper，其中每个线程都要有自己的 Looper，这就是通过 ThreadLocal 实现的。直接看代码

```java
public final class Looper {
    // sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
  
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        // Setter
        sThreadLocal.set(new Looper(quitAllowed));
    }
  
    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */  
    public static @Nullable Looper myLooper() {
        // Getter 返回当前线程的 Looper
        return sThreadLocal.get();
    }
}
```

用法也是相当的简单，变量最重要的无非就是 Setter 和 Getter . 这样我们就能在任意线程准备好自己的 Looper，也能获取到准备好的 Looper （线程内的单例）。 

## Why

接下来我们看下这么简单的接口，背后是怎么实现变量在线程间隔离的。

#### Setter 

````java
public class ThreadLocal<T> {
    /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value. 
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread(); 
        // 获取当前线程关联的 ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
          // 取到的 Map 为空就去创建，同时把这个值作为第一个值存储到 map
            createMap(t, value);
    }
  
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    } 
}
````

接下来我们来看下 ThreadLocalMap 又是什么

````JAVA
static class ThreadLocalMap {
    // 用来关联 ThreadLocal 和 其对应的 value
    static class Entry extends WeakReference<ThreadLocal<?>> {
       /** The value associated with this ThreadLocal. */
       Object value;

       Entry(ThreadLocal<?> k, Object v) {
           // 注意这里的 key 就是 ThreadLocal<?> k，实际上 Entry 对它仅持有一个弱引用
           super(k); 
           value = v;
       }
    }
    
    private Entry[] table;
    
    private void set(ThreadLocal<?> key, Object value) {

        Entry[] tab = table;
        int len = tab.length;
        // 通过 TheadLocal 的 hash 值去确定 Entry 该存放的位置
        int i = key.threadLocalHashCode & (len-1);

        for (Entry e = tab[i];//目标位置
             e != null; // 循环至目标位置为空
             e = tab[i = nextIndex(i, len)]) { // 目标位置不可用继续找下一个位置，到结尾则 index 跳到 0
            ThreadLocal<?> k = e.get();

            if (k == key) { //同一个 key，直接替换值即可
                e.value = value;
                return;
            }

            if (k == null) { 
               // key 已经过时被回收了，替换这个 Entry，并且该方法也会去检查和清除其它过期的 Entry
                replaceStaleEntry(key, value, i);
                return;
            }
        }

        tab[i] = new Entry(key, value);
        int sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash(); // 整理 + 扩容
    }    
}
````

我们可以看到，这个 ThreadLocalMap 比 HashMap 的实现简单得多，只是简单的通过一个数组来保存 keyValue 的组合 entry，并且在 hash 值计算得出来的 Index 已经有数据时也仅需要简单的找下一个位置即可。另外这个 Hash 值的计算也是非常的简单：

```JAVA
public class ThreadLocal<T> {
    // 每次 new 一个 ThreadLocal ，它的 hash 值都会递增
    private final int threadLocalHashCode = nextHashCode();
    private static AtomicInteger nextHashCode = new AtomicInteger();
    // hash 值累加的常量
    private static final int HASH_INCREMENT = 0x61c88647; 
    /**
     * Returns the next hash code.
     * 这个 hash 值和它本身没多少关系，仅和它是第几个被初始化的有关系。感觉挺有意思的
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }    
}
```

小结一下:

- 每个 Thread 都有一个 ThreadLocalMap 里面存放了这个 Thread 的所有 ThreadLocal
- 这个 map 会在 ThreadLocal 设置值的时候去懒加载，再将 ThreadLocal 作为 key 要存的值作为 value 一起放入 map
- 通过 ThreadLocal 的 hash 值去确定需要 value 放在 Map 的哪个位置

#### Getter

有了前面的铺垫，我们再来看 Get 方法就轻松了很多
```JAVA
public class ThreadLocal<T> {

public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            // 从 map 取 entry，前面看完了 set 方法，这边不用看源码也知道
            // 通过 ThreadLocal 的 hash 值计算 index, index 取到的 key 值与 当前 ThreadLocal 一致则为目标值返回；否则取下一个 index，直到取到的 entry 为空则直接返回空。
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        // 还未设置过值的时候默认初始化方法的值
        return setInitialValue();
    }

    private T setInitialValue() {
        T value = initialValue();
        ... // 省略 map 为空时初始化方法
        map.set(this, value);
        ....
        return value;
    }
    
    // 默认值为空
    protected T initialValue() {
        return null;
    }
}
```
## 内存泄漏？
我们看到在 ThreadLocalMap.Entry 里，key 是 WeakReference<ThreadLocal<?>>，弱引用，也就是这个 ThreadLocal 有可能被回收，如果已经被回收了，那就无法在 map 中 getValue，也就无法清除这个 value 的引用会造成 value 的泄露。不过这种场景还是比较少的，我们需要注意：
1.  ThreadLocalMap 的 Setter / Getter 方法中就包含了一些清理已回收的 key 的逻辑
2.  该 key 被回收时，这个 Thread 的生命周期可能也结束了，map 也一起回收了就无所谓泄露了
3.  如果 Thread 生命周期较长，那么我们在使用完 ThreadLocal 时要主动调用 ThreadLocal.remove 方法，即可以在这个线程的 Map 移除对应的 Entry，从而避免泄露


## 总结 
1. ThreadLocal 的作用是变量的线程隔离，同一个 ThreadLocal 在不同的线程中调用 get 方法会返回不同的 value
2. Thread 里有一个 ThreadLocalMap，用来存储本线程用到的 ThreadLocal 和 Value，每个线程都有自己的 ThreadLocalMap 就实现了变量线程隔离 
3. ThreadLocalMap 本质上是用一个 Entry 数组来保存数据的，Entry 的 key 是 ThreadLocal 的弱引用，value 就是 ThreadLocal 对象 set 的值
4. ThreadLocalMap 里通过 ThreadLocal 的 hash 值来计算 Index, 当 index 冲突里继续找下一个可用的 index
5.  ThreadLocal 有可能引起内存泄漏，确认使用结束可以调用 remove 移除相关引用

本文源码基于**Android-28**

