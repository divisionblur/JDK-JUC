### 手撕ConcurrentHashMap源码

#### ConcurrentHashMap常量

```java
    //散列表数组最大限制
    private static final int MAXIMUM_CAPACITY = 1 << 30;
    //散列表默认值
    private static final int DEFAULT_CAPACITY = 16;
    //并发等级 jdk1.7留下的。1.8的时候只用了一用，不代表并发级别
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;	
    //负载因子，在concurrenthashmap这边是个固定值
    private static final float LOAD_FACTOR = 0.75f;
	//树化阈值
    static final int TREEIFY_THRESHOLD = 8;
	//红黑树转化为链表的阈值
    static final int UNTREEIFY_THRESHOLD = 6;
	//联合TREEIFY_THRESHOLD控制桶位是否树化，只有当table数组长度达到64且 某个桶位 中的链表长度达到8，才会真正树化
    static final int MIN_TREEIFY_CAPACITY = 64;
	//线程迁移数据最小步长，控制线程迁移任务最小区间一个值
    private static final int MIN_TRANSFER_STRIDE = 16;
	//扩容相关，计算扩容时生成一个标识戳
    private static int RESIZE_STAMP_BITS = 16;
	//65535 标识并发编程允许的最大线程数
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
	//扩容相关
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
	//当node节点的hash值为-1时，表示当前节点为FWD节点
    static final int MOVED     = -1; // hash for forwarding nodes
	//当node节点的hash值为-2时，表示节点已经树化，且当前节点为treeBin对象，TreeBin对象代理操作红黑树
    static final int TREEBIN   = -2; // hash for roots of trees
    static final int RESERVED  = -3; // hash for transient reservations
	//0xfffffff 二进制为 0111 1111 1111 1111 1111 1111 1111 1111 可以将一个负数与运算得到正数，而不是绝对值
    static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
	//当前cpu核数
    static final int NCPU = Runtime.getRuntime().availableProcessors();
	//jdk1.8 序列化 为了兼容1.7的concurrenthashmap
    /** For serialization compatibility. */
    private static final ObjectStreamField[] serialPersistentFields = {
        new ObjectStreamField("segments", Segment[].class),
        new ObjectStreamField("segmentMask", Integer.TYPE),
        new ObjectStreamField("segmentShift", Integer.TYPE)
    };
```

ConcurrentHashMap静态代码块

```java
private static final sun.misc.Unsafe U;
    /**表示sizeCtl属性在ConcurrentHashMap中内存偏移地址*/
    private static final long SIZECTL;
    /**表示transferIndex属性在ConcurrentHashMap中内存偏移地址*/
    private static final long TRANSFERINDEX;
    /**表示baseCount属性在ConcurrentHashMap中内存偏移地址*/
    private static final long BASECOUNT;
    /**表示cellsBusy属性在ConcurrentHashMap中内存偏移地址*/
    private static final long CELLSBUSY;
    /**表示cellValue属性在CounterCell中内存偏移地址*/
    private static final long CELLVALUE;
    /**表示数组第一个元素的偏移地址*/
    private static final long ABASE;
    private static final int ASHIFT;

    static {
        try {
            U = sun.misc.Unsafe.getUnsafe();
            Class<?> k = ConcurrentHashMap.class;
            SIZECTL = U.objectFieldOffset
                (k.getDeclaredField("sizeCtl"));
            TRANSFERINDEX = U.objectFieldOffset
                (k.getDeclaredField("transferIndex"));
            BASECOUNT = U.objectFieldOffset
                (k.getDeclaredField("baseCount"));
            CELLSBUSY = U.objectFieldOffset
                (k.getDeclaredField("cellsBusy"));
            Class<?> ck = CounterCell.class;
            CELLVALUE = U.objectFieldOffset
                (ck.getDeclaredField("value"));
            Class<?> ak = Node[].class;
            ABASE = U.arrayBaseOffset(ak);
            //表示数组单元所占用空间大小,scale 表示Node[]数组中每一个单元所占用空间大小
            int scale = U.arrayIndexScale(ak);
            //1 0000 & 0 1111 = 0
            if ((scale & (scale - 1)) != 0)
                throw new Error("data type scale not a power of two");
            //numberOfLeadingZeros() 这个方法是返回当前数值转换为二进制后，从高位到低位开始统计，看有多少个0连续在一块。
            //8 => 1000 numberOfLeadingZeros(8) = 28
            //4 => 100 numberOfLeadingZeros(4) = 29
            //ASHIFT = 31 - 29 = 2 ？？
            //ABASE + （5 << ASHIFT）
            ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
        } catch (Exception e) {
            throw new Error(e);
        }
  }
```

#### ConcurrentHashMap中一些小函数

```java
     * 1100 0011 1010 0101 0001 1100 0001 1110
     * 0000 0000 0000 0000 1100 0011 1010 0101
     * 1100 0011 1010 0101 1101 1111 1011 1011
     * ---------------------------------------
     * 1100 0011 1010 0101 1101 1111 1011 1011
     * 0111 1111 1111 1111 1111 1111 1111 1111
     * 0100 0011 1010 0101 1101 1111 1011 1011
	//让高十六位也参与进来，增加散列性，然后和HASH_BITS进行与运算改成一个正数
    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
            return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
        }

    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
	//有参数初始化concurrenthashmap容量
     public ConcurrentHashMap(int initialCapacity) {
            if (initialCapacity < 0)
                throw new IllegalArgumentException();
            int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                       MAXIMUM_CAPACITY :
                       //这边与hashmap不同 如果给定初始化容量为15 那么最后的容量为32
                       tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
         //当目前table未初始化时，size表示初始化容量
            this.sizeCtl = cap;
        }
```

#### ConcurrentHashMap的put方法

```java
 public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        //控制k 和 v 不能为null
        if (key == null || value == null) throw new NullPointerException();

        //通过spread方法，可以让高位也能参与进寻址运算。
        int hash = spread(key.hashCode());
        //binCount表示当前k-v 封装成node后插入到指定桶位后，在桶位中的所属链表的下标位置
        //0 表示当前桶位为null，node可以直接放着
        //2 表示当前桶位已经可能是红黑树
        int binCount = 0;

        //tab 引用map对象的table
        //自旋
        for (Node<K,V>[] tab = table;;) {
            //f 表示桶位的头结点
            //n 表示散列表数组的长度
            //i 表示key通过寻址计算后，得到的桶位下标
            //fh 表示桶位头结点的hash值
            Node<K,V> f; int n, i, fh;

            //CASE1：成立，表示当前map中的table尚未初始化..
            if (tab == null || (n = tab.length) == 0)
                //最终当前线程都会获取到最新的map.table引用。
                tab = initTable();
            //CASE2：i 表示key使用路由寻址算法得到 key对应 table数组的下标位置，tabAt 获取指定桶位的头结点 f
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //进入到CASE2代码块 前置条件 当前table数组i桶位是Null时。
                //使用CAS方式 设置 指定数组i桶位 为 new Node<K,V>(hash, key, value, null),并且期望值是null
                //cas操作成功 表示ok，直接break for循环即可
                //cas操作失败，表示在当前线程之前，有其它线程先你一步向指定i桶位设置值了。
                //当前线程只能再次自旋，去走其它逻辑。
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }

            //CASE3：前置条件，桶位的头结点一定不是null。
            //条件成立表示当前桶位的头结点 为 FWD结点，表示目前map正处于扩容过程中..
            else if ((fh = f.hash) == MOVED)
                //看到fwd节点后，当前节点有义务帮助当前map对象完成迁移数据的工作
                //学完扩容后再来看。
                tab = helpTransfer(tab, f);

            //CASE4：当前桶位 可能是 链表 也可能是 红黑树代理结点TreeBin
            else {
                //当插入key存在时，会将旧值赋值给oldVal，返回给put方法调用处..
                V oldVal = null;

                //使用sync 加锁“头节点”，理论上是“头结点”
                synchronized (f) {
                    //为什么又要对比一下，看看当前桶位的头节点 是否为 之前获取的头结点？
                    //为了避免其它线程将该桶位的头结点修改掉，导致当前线程从sync 加锁 就有问题了。之后所有操作都不用在做了。
                    if (tabAt(tab, i) == f) {//条件成立，说明咱们 加锁 的对象没有问题，可以进来造了！

                        //条件成立，说明当前桶位就是普通链表桶位。
                        if (fh >= 0) {
                            //1.当前插入key与链表当中所有元素的key都不一致时，当前的插入操作是追加到链表的末尾，binCount表示链表长度
                            //2.当前插入key与链表当中的某个元素的key一致时，当前插入操作可能就是替换了。binCount表示冲突位置（binCount - 1）
                            binCount = 1;

                            //迭代循环当前桶位的链表，e是每次循环处理节点。
                            for (Node<K,V> e = f;; ++binCount) {
                                //当前循环节点 key
                                K ek;
                                //条件一：e.hash == hash 成立 表示循环的当前元素的hash值与插入节点的hash值一致，需要进一步判断
                                //条件二：((ek = e.key) == key ||(ek != null && key.equals(ek)))
                                //       成立：说明循环的当前节点与插入节点的key一致，发生冲突了
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    //将当前循环的元素的 值 赋值给oldVal
                                    oldVal = e.val;

                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                //当前元素 与 插入元素的key不一致 时，会走下面程序。
                                //1.更新循环处理节点为 当前节点的下一个节点
                                //2.判断下一个节点是否为null，如果是null，说明当前节点已经是队尾了，插入数据需要追加到队尾节点的后面。

                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        //前置条件，该桶位一定不是链表
                        //条件成立，表示当前桶位是 红黑树代理结点TreeBin
                        else if (f instanceof TreeBin) {
                            //p 表示红黑树中如果与你插入节点的key 有冲突节点的话 ，则putTreeVal 方法 会返回冲突节点的引用。
                            Node<K,V> p;
                            //强制设置binCount为2，因为binCount <= 1 时有其它含义，所以这里设置为了2 回头讲 addCount。
                            binCount = 2;

                            //条件一：成立，说明当前插入节点的key与红黑树中的某个节点的key一致，冲突了
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                //将冲突节点的值 赋值给 oldVal
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }

                //说明当前桶位不为null，可能是红黑树 也可能是链表
                if (binCount != 0) {
                    //如果binCount>=8 表示处理的桶位一定是链表
                    if (binCount >= TREEIFY_THRESHOLD)
                        //调用转化链表为红黑树的方法
                        treeifyBin(tab, i);
                    //说明当前线程插入的数据key，与原有k-v发生冲突，需要将原数据v返回给调用者。
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }

        //1.统计当前table一共有多少数据
        //2.判断是否达到扩容阈值标准，触发扩容。
        addCount(1L, binCount);

        return null;
    }
```

```java
private final Node<K,V>[] initTable() {
        //tab 引用map.table
        //sc sizeCtl的临时值
        Node<K,V>[] tab; int sc;
        //自旋 条件：map.table 尚未初始化
        while ((tab = table) == null || tab.length == 0) {

            if ((sc = sizeCtl) < 0)
                //大概率就是-1，表示其它线程正在进行创建table的过程，当前线程没有竞争到初始化table的锁。
                Thread.yield(); // lost initialization race; just spin

            //1.sizeCtl = 0，表示创建table数组时 使用DEFAULT_CAPACITY为大小
            //2.如果table未初始化，表示初始化大小
            //3.如果table已经初始化，表示下次扩容时的 触发条件（阈值）
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    //这里为什么又要判断呢？ 防止其它线程已经初始化完毕了，然后当前线程再次初始化..导致丢失数据。
                    //条件成立，说明其它线程都没有进入过这个if块，当前线程就是具备初始化table权利了。
                    if ((tab = table) == null || tab.length == 0) {

                        //sc大于0 创建table时 使用 sc为指定大小，否则使用 16 默认值.
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;

                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        //最终赋值给 map.table
                        table = tab = nt;
                        //n >>> 2  => 等于 1/4 n     n - (1/4)n = 3/4 n => 0.75 * n
                        //sc 0.75 n 表示下一次扩容时的触发条件。
                        sc = n - (n >>> 2);
                    }
                } finally {
                    //1.如果当前线程是第一次创建map.table的线程话，sc表示的是 下一次扩容的阈值
                    //2.表示当前线程 并不是第一次创建map.table的线程，当前线程进入到else if 块 时，将
                    //sizeCtl 设置为了-1 ，那么这时需要将其修改为 进入时的值。
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

ConcruuentHashMap扩容方法

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    //n 扩容前数组长度 stride 表示分配给线程的步长
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```