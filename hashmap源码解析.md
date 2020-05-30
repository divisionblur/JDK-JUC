### 手撕HashMap源码

什么是hash：Hash也成散列、哈希，对应的英文都是hash，基本原理就是把任意长度的输入，通过hash算法变成固定长度的输出，这个映射规则就是对应的hash算法，而原始数据映射后的二进制串就是哈希值。

#### Hash特点

- 从hash值不可以反向推导出原始数据
- 输入数据的微小变化会得到完全不同的hash值，相同的数据会得到相同的值
- hash算法的执行效率要高效，长的文本也可以很快计算出hash值
- hash算法的冲突概率要小

由于hash原理是将输入空间的值映射到hash空间内，而hash值的空间远小于输入的空间，根据抽屉原理，一定会存在不同的输入被映射成相同的输出的情况。

**抽屉原理：** 桌上有10个苹果放到9个抽屉里，那么必定至少有一个抽屉放两个苹果。这一现象就称为抽屉原理。

#### HashMap核心属性分析

threshold，localFactory，size，modCount

默认参数

```java
//缺省的数组大小
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
//数组最大大小
static final int MAXIMUM_CAPACITY = 1 << 30;
//缺省的负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//树化阈值
static final int TREEIFY_THRESHOLD = 8;
//树降级成链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;
//最小树化容量
static final int MIN_TREEIFY_CAPACITY = 64;
```

核心属性

```java
//hash表长度
transient int size;
//hash表修改次数
transient int modCount;
//hash表扩容阈值
int threshold;
//用来计算扩容阈值  threshold = capacity * loadFactior
final float loadFactor;
```

#### HashMap构造方法

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    //保证数组必须是2的次方数
    this.threshold = tableSizeFor(initialCapacity);
}
```

```java
//返回一个大于等于当前值的一个数字，这个数字必须是2的次方数
//原理 比如给你一个 0001 1000 1010 -> 0001 1111 1111 + 1 = 0010 0000 0000 保证是2的次方数
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

#### HashMap的put方法

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

首先先看下hash方法

```java
//扰动函数，让key的高16位也参与路由运算
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    //tab:引用当前hashMap的散列表
    //p: 表示当前散列表的元素
    //n: 表示数组的长度
    //i: 表示路由寻址结果
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //懒加载 第一次去插元素的时候才会初始化table
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //路由算法，n在上面赋值
    //最简单的一种情况：寻址找到的桶位正好为null，直接将当前(key,value)放进去即可	
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        //e: 临时节点 k： 临时key	
        Node<K,V> e; K k;
        //找到了一个与当前要插入(key,value)一致的key
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //树化
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //链表的情况，并且链表的头元素与我们要插入的(key,value)不一致
            for (int binCount = 0; ; ++binCount) {
                //迭代到最后，也没有找到与插入key相同的node
                if ((e = p.next) == null) {
                    //在末尾插入
                    p.next = newNode(hash, key, value, null);
                    //树化
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //替换操作
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //散列表结构被修改的次数，更改val不改变size和modcount
    ++modCount;
    //扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

#### HashMap的resize方法

```java
//为什么需要扩容
//为了解决hash冲突导致的链化影响查询效率的问题，扩容会缓解该问题
final Node<K,V>[] resize() {
    //引用前的hashMap
    Node<K,V>[] oldTab = table;
    //扩容前table数组的长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //扩容前的扩容阈值，也就是触发本次扩容的阈值
    int oldThr = threshold;
    //newcap：扩容之后table长度
    //newThr: 下次扩容的阈值
    int newCap, newThr = 0;
    //hashMap已经初始化过了，这是一次正常的扩容
    if (oldCap > 0) {
        //大于table的最大容量，没法再次扩容了，且设置扩容条件为int的最大值
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //扩容为原来的两倍，并且扩容阈值翻倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //oldCap == 0 构造方法中自己传入了初始化容量，oldThr就是数组容量	
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    //oldCap == 0 无参构造初始化数组容量和扩容阈值
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY; //16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY); //12
    }
    //两种情况执行下面的方案
    //1 oldCap >= DEFAULT_INITIAL_CAPACITY不满足
    //2 oldThr > 0
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    //创建一个更大的数组
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    //扩容之前table不为null
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            //当前节点
            Node<K,V> e;
            //当前桶位有数据，但是当前数据是单个数据，还是链表，还是红黑树，不知道
            if ((e = oldTab[j]) != null) {
                //置null，方便JVM GC时回收内存
                oldTab[j] = null;
                //桶位里是单个元素
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                //判断是否为树
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    //定义成两个链表
                    //因为扩容后 原来链表上的数据要么还是原来的下标，要么就是原来下标加上旧数组长度
                    
                    //保存与原来下标一致的链表
                    Node<K,V> loHead = null, loTail = null;
                    //保存与原来下标+旧数组长度的链表
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //e.hash 可能是 1 1010
                        //       还可能 0 1010
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        //将原来链表低位的最后一个节点置位null
                        loTail.next = null;
                        //将链表放到新的table中
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

HashMap的get方法

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

```java
final Node<K,V> getNode(int hash, Object key) {
    //tab 当前hashMap的引用
    //first 桶中的头元素
    //e 临时node元素
    //n 数组的长度
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    //定位元素
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //当前桶位只有一个元素
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //桶位升级成了红黑树
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //桶位是链表
            do { 
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
  
```

#### HashMap的remove方法

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
```

```java
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    //判断table是否为空，并且找到的当前桶位不能为null
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        //当前桶位的元素即为你要删除的元素
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            //判断当前桶位是否升级为红黑树了
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        //mathchValue在只传入一个key的情况下为false，remove(key,value)为true需要比较value的值
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            //删除操作
            //树的删除逻辑
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            //当前桶位第一个元素即为删除元素，直接删除元素
            else if (node == p)
                tab[index] = node.next;
            //删除的元素在链表中，node为待删除元素，直接将p.next = node.next即可
            else
                p.next = node.next;
            //操作次数加一
            ++modCount;
            //size减一
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

HashMap的replace方法

```java
@Override
public boolean replace(K key, V oldValue, V newValue) {
    Node<K,V> e; V v;
    if ((e = getNode(hash(key), key)) != null &&
        ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {
        e.value = newValue;
        afterNodeAccess(e);
        return true;
    }
    return false;
}
```

#### 面试题

