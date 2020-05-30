#### 手撕ThreadLocal源码

```java
//线程获取threadLocal.get()时 如果是第一次在某个 threadLocal对象上get时，会给当前线程分配一个value
//这个value 和 当前的threadLocal对象 被包装成为一个 entry 其中 key是 threadLocal对象，value是threadLocal对象给当前线程生成的value
//这个entry存放到 当前线程 threadLocals 这个map的哪个桶位？ 与当前 threadLocal对象的threadLocalHashCode 有关系。
// 使用 threadLocalHashCode & (table.length - 1) 的到的位置 就是当前 entry需要存放的位置。
  private final int threadLocalHashCode = nextHashCode();
//创建ThreadLocal对象时 会使用到，每创建一个threadLocal对象 就会使用nextHashCode 分配一个hash值给这个对象。
  private static AtomicInteger nextHashCode =
        new AtomicInteger();
 //每创建一个ThreadLocal对象，这个ThreadLocal.nextHashCode 这个值就会增长 0x61c88647 。
 //这个值 很特殊，它是 斐波那契数  也叫 黄金分割数。hash增量为 这个数字，带来的好处就是 hash分布非常均匀。
  private static final int HASH_INCREMENT = 0x61c88647;

  //创建新的ThreadLocal对象时  会给当前对象分配一个hash，使用这个方法。
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
//默认返回为null，一般都是要重写这个方法的
protected T initialValue() {
        return null;
    }

```

##### get方法

```java
public T get() {
    //获取当前线程
    Thread t = Thread.currentThread();
    ////获取到当前线程Thread对象的 threadLocals map引用
    ThreadLocalMap map = getMap(t);
    //条件成立：说明当前线程已经拥有自己的 ThreadLocalMap 对象了
    if (map != null) {
        //key：当前threadLocal对象
        //调用map.getEntry() 方法 获取threadLocalMap 中该threadLocal关联的 entry
        ThreadLocalMap.Entry e = map.getEntry(this);
         //条件成立：说明当前线程 初始化过 与当前threadLocal对象相关联的 线程局部变量
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //执行到这里有几种情况？
        //1.当前线程对应的threadLocalMap是空
        //2.当前线程与当前threadLocal对象没有生成过相关联的 线程局部变量..

        //setInitialValue方法初始化当前线程与当前threadLocal对象 相关联的value。
        //且 当前线程如果没有threadLocalMap的话，还会初始化创建map。
    
    return setInitialValue();
}
```

```java
private T setInitialValue() {
    //调用的当前ThreadLocal对象的InitialValue方法，这个方法大部分情况下咱们都会重写
    //value就是当前ThreadLocal对象与当前线程关联的 线程局部变量
    T value = initialValue();
    //获取当前线程对象
    Thread t = Thread.currentThread();
    //获取线程内部的threadlocals	ThreadLoaclMap对象
    ThreadLocalMap map = getMap(t);
    //条件成立：当前线程内部已经初始化过ThreadLoaclMap对象了，(线程的ThreadLoaclMap对象只初始化一次)
    if (map != null)
        //保存当前threadlocal与当前线程生成的线程局部变量
        //key: 当前threadLocal对象 value 线程与ThreadLocal相关的局部变量
        map.set(this, value);
    else
        //执行到这里，当前线程还未初始化ThreadLocalMap，这里调用createMap给当前线程创建map
        
        //参数1:当前线程 参数2：线程与当前threadlocal相关的局部变量
        createMap(t, value);
    return value;
}
```

```java
void createMap(Thread t, T firstValue) {
        //传递t 的意义就是 要访问 当前这个线程 t.threadLocals 字段，给这个字段初始化。


        //new ThreadLocalMap(this, firstValue)
        //创建一个ThreadLocalMap对象 初始 k-v 为 ： this <当前threadLocal对象> ，线程与当前threadLocal相关的局部变量
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

```java
public void remove() {
         //获取当前线程的 threadLocalMap对象
         ThreadLocalMap m = getMap(Thread.currentThread());
         //条件成立：说明当前线程已经初始化过 threadLocalMap对象了
         if (m != null)
             //调用threadLocalMap.remove( key = 当前threadLocal)
             m.remove(this);
     }
```

##### entry

```java
        /* 什么是弱引用呢？
         * A a = new A();     //强引用
         * WeakReference weakA = new WeakReference(a);  //弱引用
         *
         * a = null;
         * 下一次GC 时 对象a就被回收了，别管有没有 弱引用 是否在关联这个对象。
         *
         * key 使用的是弱引用保留，key保存的是threadLocal对象。
         * value 使用的是强引用，value保存的是 threadLocal对象与当前线程相关联的 value。
         *
         * entry#key 这样设计有什么好处呢？
         * 当threadLocal对象失去强引用且对象GC回收后，散列表中的与 threadLocal对象相关联的 entry#key 再次去key.get() 时，拿到的是null。
         * 站在map角度就可以区分出哪些entry是过期的，哪些entry是非过期的。
         *
         *
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

```java
/**
         * The initial capacity -- MUST be a power of two.
         * 初始化当前map内部 散列表数组的初始长度 16
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         * threadLocalMap 内部散列表数组引用，数组的长度 必须是 2的次方数
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         * 当前散列表数组 占用情况，存放多少个entry。
         */
        private int size = 0;

        /**
         * The next size value at which to resize.
         * 扩容触发阈值，初始值为： len * 2/3
         * 触发后调用 rehash() 方法。
         * rehash() 方法先做一次全量检查全局 过期数据，把散列表中所有过期的entry移除。
         * 如果移除之后 当前 散列表中的entry 个数仍然达到  threshold - threshold/4  就进行扩容。
         */
        private int threshold; // Default to 0

        /**
         * Set the resize threshold to maintain at worst a 2/3 load factor.
         * 将阈值设置为 （当前数组长度 * 2）/ 3。
         */
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }
/**
         * Increment i modulo len.
         * 参数1：当前下标
         * 参数2：当前散列表数组长度
         */
        private static int nextIndex(int i, int len) {
            //当前下标+1 小于散列表数组的话，返回 +1后的值
            //否则 情况就是 下标+1 == len ，返回0
            //实际形成一个环绕式的访问。
            return ((i + 1 < len) ? i + 1 : 0);
        }

        /**
         * Decrement i modulo len.
         * 参数1：当前下标
         * 参数2：当前散列表数组长度
         */
        private static int prevIndex(int i, int len) {
            //当前下标-1 大于等于0 返回 -1后的值就ok。
            //否则 说明 当前下标-1 == -1. 此时 返回散列表最大下标。
            //实际形成一个环绕式的访问。
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }
```

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            //创建entry数组长度为16，表示threadLocalMap内部的散列表。
            table = new Entry[INITIAL_CAPACITY];
            //寻址算法：key.threadLocalHashCode & (table.length - 1)
            //table数组的长度一定是 2 的次方数。
            //2的次方数-1 有什么特征呢？  转化为2进制后都是1.    16==> 1 0000 - 1 => 1111
            //1111 与任何数值进行&运算后 得到的数值 一定是 <= 1111

            //i 计算出来的结果 一定是 <= B1111
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);

            //创建entry对象 存放到 指定位置的slot中。
            table[i] = new Entry(firstKey, firstValue);
            //设置size=1
            size = 1;
            //设置扩容阈值 （当前数组长度 * 2）/ 3  => 16 * 2 / 3 => 10
            setThreshold(INITIAL_CAPACITY);
        }
```

```java
//ThreadLocal对象的get方法，实际上是ThreadLocalMap.getEntry()代理完成的
//key:某个 ThreadLocal对象，因为 散列表中存储的entry.key 类型是 ThreadLocal。
private Entry getEntry(ThreadLocal<?> key) {
    //路由规则： ThreadLocal.threadLocalHashCode & (table.length - 1) ==》 index
    int i = key.threadLocalHashCode & (table.length - 1);
    //访问散列表中 指定指定位置的 slot
    Entry e = table[i];
    //条件一：成立 说明slot有值
    //条件二：成立 说明 entry#key 与当前查询的key一致，返回当前entry 给上层就可以了。
    if (e != null && e.get() == key)
        return e;
    else
         //有几种情况会执行到这里？
                //1.e == null
                //2.e.key != key


            //getEntryAfterMiss 方法 会继续向当前桶位后面继续搜索 e.key == key 的entry.

            //为什么这样做呢？？
            //因为 存储时  发生hash冲突后，并没有在entry层面形成 链表.. 存储时的处理 就是线性的向后找到一个可以使用的slot，并且存放进去。
        return getEntryAfterMiss(key, i, e);
}
```

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            //获取当前threadLocalMap中的散列表 table
            Entry[] tab = table;
            //获取table长度
            int len = tab.length;

            //条件：e != null 说明 向后查找的范围是有限的，碰到 slot == null 的情况，搜索结束。
            //e:循环处理的当前元素
            while (e != null) {
                //获取当前slot 中entry对象的key
                ThreadLocal<?> k = e.get();
                //条件成立：说明向后查询过程中找到合适的entry了，返回entry就ok了。
                if (k == key)
                    //找到的情况下，就从这里返回了。
                    return e;
                //条件成立：说明当前slot中的entry#key 关联的 ThreadLocal对象已经被GC回收了.. 因为key 是弱引用， key = e.get() == null.
                if (k == null)
                    //做一次 探测式过期数据回收。
                    expungeStaleEntry(i);
                else
                    //更新index，继续向后搜索。
                    i = nextIndex(i, len);
                //获取下一个slot中的entry。
                e = tab[i];
            }

            //执行到这里，说明关联区段内都没找到相应数据。
            return null;
        }
```

```java
private int expungeStaleEntry(int staleSlot) {
            //获取散列表
            Entry[] tab = table;
            //获取散列表当前长度
            int len = tab.length;

            // expunge entry at staleSlot
            //help gc
            tab[staleSlot].value = null;
            //因为staleSlot位置的entry 是过期的 这里直接置为Null
            tab[staleSlot] = null;
            //因为上面干掉一个元素，所以 -1.
            size--;

            // Rehash until we encounter null
            //e：表示当前遍历节点
            Entry e;
            //i：表示当前遍历的index
            int i;

            //for循环从 staleSlot + 1的位置开始搜索过期数据，直到碰到 slot == null 结束。
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                //进入到for循环里面 当前entry一定不为null


                //获取当前遍历节点 entry 的key.
                ThreadLocal<?> k = e.get();

                //条件成立：说明k表示的threadLocal对象 已经被GC回收了... 当前entry属于脏数据了...
                if (k == null) {
                    //help gc
                    e.value = null;
                    //脏数据对应的slot置为null
                    tab[i] = null;
                    //因为上面干掉一个元素，所以 -1.
                    size--;
                } else {
                    //执行到这里，说明当前遍历的slot中对应的entry 是非过期数据
                    //因为前面有可能清理掉了几个过期数据。
                    //且当前entry 存储时有可能碰到hash冲突了，往后偏移存储了，这个时候 应该去优化位置，让这个位置更靠近 正确位置。
                    //这样的话，查询的时候 效率才会更高！

                    //重新计算当前entry对应的 index
                    int h = k.threadLocalHashCode & (len - 1);
                    //条件成立：说明当前entry存储时 就是发生过hash冲突，然后向后偏移过了...
                    if (h != i) {
                        //将entry当前位置 设置为null
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.


                        //h 是正确位置。

                        //以正确位置h 开始，向后查找第一个 可以存放entry的位置。
                        while (tab[h] != null)
                            h = nextIndex(h, len);

                        //将当前元素放入到 距离正确位置 更近的位置（有可能就是正确位置）。
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

##### set方法

```java
  private void set(ThreadLocal<?> key, Object value) {
            //获取散列表
            Entry[] tab = table;
            //获取散列表数组长度
            int len = tab.length;
            //计算当前key 在 散列表中的对应的位置
            int i = key.threadLocalHashCode & (len-1);


            //以当前key对应的slot位置 向后查询，找到可以使用的slot。
            //什么slot可以使用呢？？
            //1.k == key 说明是替换
            //2.碰到一个过期的 slot ，这个时候 咱们可以强行占用呗。
            //3.查找过程中 碰到 slot == null 了。
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {

                //获取当前元素key
                ThreadLocal<?> k = e.get();

                //条件成立：说明当前set操作是一个替换操作。
                if (k == key) {
                    //做替换逻辑。
                    e.value = value;
                    return;
                }

                //条件成立：说明 向下寻找过程中 碰到entry#key == null 的情况了，说明当前entry 是过期数据。
                if (k == null) {
                    //碰到一个过期的 slot ，这个时候 咱们可以强行占用呗。
                    //替换过期数据的逻辑。
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }



            //执行到这里，说明for循环碰到了 slot == null 的情况。
            //在合适的slot中 创建一个新的entry对象。
            tab[i] = new Entry(key, value);
            //因为是新添加 所以++size.
            int sz = ++size;

            //做一次启发式清理
            //条件一：!cleanSomeSlots(i, sz) 成立，说明启发式清理工作 未清理到任何数据..
            //条件二：sz >= threshold 成立，说明当前table内的entry已经达到扩容阈值了..会触发rehash操作。
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

