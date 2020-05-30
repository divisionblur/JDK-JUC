### 源码之LongAdder

#### 简介

LongAdder 是并发大师 @author Doug Lea （大哥李）的作品，设计的非常精巧 LongAdder 类有几个关键域

```java
// 累加单元数组, 懒惰初始化
transient volatile Cell[] cells;
// 基础值, 如果没有竞争, 则用 cas 累加这个域
transient volatile long base;
// 在 cells 创建或扩容时, 置为 1, 表示加锁
transient volatile int cellsBusy;
```

#### 伪共享

其中 Cell 即为累加单元

```java
// 防止缓存行伪共享
    @sun.misc.Contended
    static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        // 最重要的方法, 用来 cas 方式进行累加, prev 表示旧值, next 表示新值
        final boolean cas(long prev, long next) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, prev, next);
        }
// 省略不重要代码
    }
```

得从缓存说起
缓存与内存的速度比较

![image-20200505090756291](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200505090756291.png)

![image-20200505090807387](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200505090807387.png)

因为 CPU 与 内存的速度差异很大，需要靠预读数据至缓存来提升效率。而缓存以缓存行为单位，每个缓存行对应着一块内存，一般是 64 byte（8 个 long）缓存的加入会造成数据副本的产生，即同一份数据会缓存在不同核心的缓存行中CPU 要保证数据的一致性，如果某个 CPU 核心更改了数据，其它 CPU 核心对应的整个缓存行必须失效

![image-20200505091556849](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200505091556849.png)

因为 Cell 是数组形式，在内存中是连续存储的，一个 Cell 为 24 字节（16 字节的对象头和 8 字节的 value），因
此缓存行可以存下 2 个的 Cell 对象。这样问题来了：

- Core-0 要修改 Cell[0]
- Core-1 要修改 Cell[1]

无论谁修改成功，都会导致对方 Core 的缓存行失效，比如 Core-0 中  Cell[0]=6000, Cell[1]=8000 要累加
Cell[0]=6001, Cell[1]=8000 ，这时会让 Core-1 的缓存行失效
@sun.misc.Contended 用来解决这个问题，它的原理是在使用此注解的对象或字段的前后各增加 128 字节大小的
padding，从而让 CPU 将对象预读至缓存时占用不同的缓存行，这样，不会造成对方缓存行的失效

#### 源码解读

```java
demo(
        () -> new LongAdder(),
        adder -> adder.increment()
);
public void increment() {
        add(1L);
    }
public void add(long x) {
    //as表示cells引用
    //b 表示获取的base值
    //v 表示期望值
    //m 表示cells数组长度-1
    //a 表示当前线程命中的cell单元格
   
        Cell[] as; long b, v; int m; Cell a;
    //cell懒加载，只有在用的时候才会去创建
    //条件一 ：true 表示cells已经被初始化过了，当前线程应该将数据写到对应的cell中去
    		//false 表示cells未初始化，当前线程将数据写到base中去
    //条件二： true 表示当前线程cas替换成功
    		//false 发生竞争了，可能需要重试，或者扩容
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            //什么时候可以进来
            //1 表示cells已经被初始化过了，当前线程应该将数据写到对应的cell中去
            //2 发生竞争了，可能需要重试，或者扩容
            //true 未竞争 false 竞争
            boolean uncontended = true;
            //条件一：true cells未初始化 也就是说base发生竞争了
            	//false cells初始化了，当前线程应该是找自己的cell写值
            //条件二：getProbe()获取当前线程的hash值 m为cells的长度减一 cell长度是2的次方数 15 = b1111
            	//true 说明当前线程对应cell的下标为null，需要创建
            	//false 说明当前线程对应cell不为空
            //条件三：true：表示cas失败，意味着当前线程对应的cell有竞争 uncontended==false
            	//false：表示cas成功
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                //都有哪些情况去调用
                //1 cells未初始化 也就是说base发生竞争了
                //2 前线程对应cell的下标为null，需要创建
                //3 表示cas失败，意味着当前线程对应的cell有竞争 
                longAccumulate(x, null, uncontended);
        }
    }
```

![image-20200505092655266](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200505092655266.png)

```java
//1 cells未初始化 也就是说base发生竞争了
//2 前线程对应cell的下标为null，需要创建
//3 表示cas失败，意味着当前线程对应的cell有竞争
// wasUncontended 只有cells初始化后，并且当前线程竞争修改失败 uncontended == false
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
    //h 表示获取当前线程的hash值，条件true 表示当前线程还未分配hash值
        if ((h = getProbe()) == 0) {
            //分配hash值
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            //为什么？ 在默认情况下 当前线程肯定是写到了cells[0]这个位置上，不能将他视为一次真正的竞争
            wasUncontended = true;
        }
    //扩容意向 false一定不会扩容 true可能会扩容
        boolean collide = false;                // True if last slot nonempty
    //自旋
        for (;;) {
            //as 表示cells引用
            //a 当前线程命中的cell
            //n 表示cells数组长度
            //v 表示期望值
            Cell[] as; Cell a; int n; long v;
            //case1： 表示cells已经被初始化了，当前线程将数据写入到对应cell中
            if ((as = cells) != null && (n = as.length) > 0) {
                //case1.1 当前线程对应cell为null 需要去创建
                if ((a = as[(n - 1) & h]) == null) {
                    //true 当前锁未被占用
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        //拿到当前x创建一个cell
                        Cell r = new Cell(x);   // Optimistically create
                        //true 获取锁成功
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                //条件一 条件二恒成立
                                //rs[j = (m - 1) & h] == null 防止其他线程初始化过该位置，然后该线程再次                                   //初始化，导致数据丢失
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    //扩容意向改为false
                    collide = false;
                }
                //case1.2 只有一种情况来到这里
                //只有cells初始化后，并且当前线程竞争修改失败 uncontended == false 
                //跳到 h = advanceProbe(h);
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                //case1.3 当前线程rehash过hash值 并且线程对应cell不为null
                //true: 写成功，退出循环
                //false: rehash之后命中的cell依然有竞争 重试一次
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                //case1.4 
                //条件一：true cells长度大于cpu核数 collide(扩容意向)改为false 表示不扩容了
                	   //false 可以扩容
                //条件二：cells ！= as true其他线程已经扩容过了，当前线程rehash即可
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                //case1.5 设置扩容意向为true 但不一定真的需要扩容
                else if (!collide)
                    collide = true;
                //case 1.6 真正的扩容逻辑 此时case1.3经过了两次重试 
                //条件一：true 当前线程可以去竞争这把锁
                //条件二：true 当前线程竞争锁成功，去进行扩容 
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            //扩容 容量为原来的2倍 
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                //重置当前线程hash值
                h = advanceProbe(h);
            }
            //case2: 前置条件cells还未初始化 此时拿到的as == null
            //条件一：true 未加锁
            //条件二：cells == as？ 因为其他线程可能在你拿到as==null之后修改了cells
            //条件三：表示获取锁成功，会把cellsBusy改为1，失败表示其他线程正在持有这把锁
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                          // Initialize table
                     //防止其他线程已经初始化了，当前线程再次初始化，导致数据丢失
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            //case3:
            //条件一：当前cellsBusy加锁状态，表示其他线程正在初始化cells，所以当前线程将值累加到base
            //条件二：cells被其他线程初始化后，所以当前线程将值累加到base
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
```

![image-20200505093052165](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200505093052165.png)