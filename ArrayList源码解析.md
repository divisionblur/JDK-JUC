## ArrayList源码解析

### 1 ArratList 集合介绍

List接口可调节大小的数组实现

数组：一旦更改长度就不可以发生改变

#### 1.1 数组结构介绍

- 增删慢：每次删除元素，都需要更改数组的长度，拷贝及移动元素位置
- 查询快：由于数组在内存内部是一块连续的空间，因此可以根据地址+索引的方式快速获取对应位置上的元素

### 2 ArrayList继承关系

#### 2.1 Serializable标记性接口

1 类的序列化由实现java.io.Serializable接口类启用，不实现该接口不会使任何接口序列化或者反序列化。可序列化的类其子类都是可以序列化的。序列化接口没有方法和字段，仅用于标记串行化的语意。

序列化：将对象数据写入到文件

反序列化：将文件中的数据读取出来

2 Serializable 源码介绍

```java
public interface Serializable {
}
```

3 为什么冲写toString  toString方法会产生许多变量在内存中占用一定的空间，直接创建一个StringBuilder 可以极大的减小内存空间的使用。 

####  2.2 Cloneable标记性接口

1 **介绍**  一个类通过实现Cloneable接口来实现Object.clone()方法，该方法对于该类的实例进行字段复制是合法的。在不实现Cloneable方法调用对象的克隆方法会导致异常CloneNotSupportException。 **简言之** 克隆就是根据已有的数据，创建出一份新的完全一致的数据拷贝。

2 Cloneable源码介绍

```java
public interface Cloneable {
}
```

3 克隆的前提条件

- 被克隆的对象需要实现Cloneable接口
- 需要重新clone方法

##### 浅拷贝和深拷贝

###### 1. 浅拷贝介绍

浅拷贝是按位拷贝对象，它会创建一个新对象，这个对象有着原始对象属性值的一份精确拷贝。如果属性是基本类型，拷贝的就是基本类型的值；如果属性是内存地址（引用类型），拷贝的就是内存地址 ，因此如果其中一个对象改变了这个地址，就会影响到另一个对象。即默认拷贝构造函数只是对对象进行浅拷贝复制(逐个成员依次拷贝)，即只复制对象空间而不复制资源。

###### 2. 深拷贝介绍

因为浅拷贝会带来数据安全方面的隐患，而深拷贝，在拷贝引用类型的变量时，为引用类型的数据开辟了一个独立的内存空间，实现真正意义上的拷贝。

#### 2.3 RandomAcess标记接口

**介绍** 标记接口由List实现使用，以表明他们支持快速（通常为恒定时间）随机访问。

次接口主要目的是允许算法更改其行为，一遍运用于随机访问列表或顺序访问列表时提供良好的性能。

### 3 ArrayLIst源码分析

#### 3.1 构造方法

|              Constructor               |                       Construstor描述                        |
| :------------------------------------: | :----------------------------------------------------------: |
|              ArrayList()               |                构建一个初始容量为10 的空列表                 |
|         ArrayList(int Initial)         |                 构建一个具有指定容量的空列表                 |
| ArrayList(Collection<? extends E > c ) | 构建一个包含指定元素的列表，按照他们由集合的迭代器返回的顺序 |

**1.无参初始化并不是在无参构造方法的位置执行的，而是在第一次执行add方法的时候执行了容器大小的设置**

#### 3.2 添加方法

|                          方法名                           |                             描述                             |
| :-------------------------------------------------------: | :----------------------------------------------------------: |
|                  public boolean add(E e)                  |                 将指定元素追加到次列表的末尾                 |
|           public void add(int index, E element)           |                   在此列表指定位置添加元素                   |
|      public boolean add(Collection<? Exends e> c  )       | 按指定集合的iterator返回顺序将指定集合中的所有元素追加打此列表的末尾 |
| public boolean add(int index, Collection<? Exends e> c  ) |      将只等集合中的所有元素插入列表中，从指定的位置开始      |

调用add方法会初始化List容量默认为10，在需要扩容的时候关键命令为

```java
int newCapacitor = oldCapacitor + (oldCapacitor>>1)
```

​	**原来的长度加上原来长度的一半，也就是每次扩容为原来的1.5倍**

#### 3.3 修改方法

```java
 public E set(int index, E element) {
     //校验索引
        rangeCheck(index);
        E oldValue = elementData(index);
        elementData[index] = element;
     //返回被替换的元素
        return oldValue;
    }
```

#### 3.3 获取方法

```java
public E get(int index) {
    rangeCheck(index);

    return elementData(index);
}
```

#### 3.4 转换方法

ArrayList本身没有toString方法

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> 
```

所用的toString方法是AbstractCollection下的toString方法

```java
public abstract class AbstractCollection<E>{
     public String toString() {
         //获取一个迭代器
        Iterator<E> it = iterator();
         //判断迭代器是否为空
        if (! it.hasNext())
            return "[]";
        StringBuilder sb = new StringBuilder();
        sb.append('[');
         //无线循环
        for (;;) {
            E e = it.next();
            //一个三元表达式
            sb.append(e == this ? "(this Collection)" : e);
            if (! it.hasNext())
                return sb.append(']').toString();
            sb.append(',').append(' ');
        }
    }
}

```

![image-20200502141326171](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200502141326171.png)

#### 3.5迭代器

```java
//获取迭代器
public Iterator<E> iterator() {
    //创建一个对象
    return new Itr();
}
private class Itr implements Iterator<E> {
        int cursor;       // 光标 默认值为0
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount; //将集合实际修改次数赋值给预期修改次数
    //判断是否有元素
        public boolean hasNext() {
            return cursor != size;
        }
    public E next() {
            checkForComodification();
        //将光标赋值给i
            int i = cursor;
        //当大于集合元素长度时，说明集合中没有元素了
            if (i >= size)
                throw new NoSuchElementException();
        //将集合储存数据的地址赋值给该方法的局部变量
            Object[] elementData = ArrayList.this.elementData;
        //并发修改异常
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
        //光标zizeng
            cursor = i + 1;
        //从数组中取出元素并返回
            return (E) elementData[lastRet = i];
        }
    //校验预期修改集合次数是否和实际修改集合次数一致
     final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
}
```

```java
//iterator.hasNext 当前光标值不等于ArrayList值时返回true
public boolean hasNext() {
    return cursor != size;
}
```

##### 3.5.1 迭代器删除一个典型的案例

```java
public class ArrayListDemo2 {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("hello");
        list.add("java");
        list.add("php");

        Iterator<String> it = list.iterator();
        while (it.hasNext()){
            String s = it.next();
            if(s.equals("php")){
                list.remove("php");
            }
        }
    }
}
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:901)
	at java.util.ArrayList$Itr.next(ArrayList.java:851)
	at com.gewei.Thread.ArrayListDemo2.main(ArrayListDemo2.java:16)
```

那么ConcurrentModificationException这个错误出现在哪那

首先需要查看下add方法

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
 private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```

此时modCount为3，接着获取迭代器

```
public Iterator<E> iterator() {
    return new Itr();
}
private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        //获取迭代器的时候，expectedModCount的值也是3 在迭代的时候赋值
        int expectedModCount = modCount;
}
```

迭代器在调用next的时候有一次判断

```
public E next() {
    checkForComodification();
final void checkForComodification() {
			//此时expectedModCount 与modCount也是相等的
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
}
```

执行删除方法

```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
//真正的删除方法
private void fastRemove(int index) {
		//在实际删除过程中实际修改次数会自增
		//此时集合修改次数为4 但是预期修改次数为3
        modCount++;
        //计算需要移动的元素个数
        int numMoved = size - index - 1;
        //移动的核心代码
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
    	//让删除的元素置位null 等待下次垃圾回收
        elementData[--size] = null; // clear to let GC do its work
    }
```

再次调用`while(it.hasNext())`

```
public boolean hasNext() {
	//此时光标位置为3 但是size为2 还要继续往下面执行
    return cursor != size;
}
```

再次调用`it.next`时

```java
public E next() {
    checkForComodification();
final void checkForComodification() {
			//此时expectedModCount 与modCount不相等的 将报错
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
}
```

**结论**

- 1 集合每次在调用add方法的时候，实际修改次数的变量值会自增一次。
- 2 在获取迭代器的时候，集合会执行一次将实际修改次数赋值给预期修改次数的操作。
- 3 集合在调用删除元素的时候也会针对实际修改次数进行自增的操作。
- 4 在迭代器调用next方法时会进行实际修改次数和预期修改次数的判断，如果二者不相等，那么就会报ConcurrentModificationException();

**如果修改下删除元素所在数组的位置那**

```java
List<String> list = new ArrayList<>();
list.add("hello");
list.add("php");
list.add("java");
```

**结论：**
		当要删除的元素在集合的倒数第二个位置时，不会发生并发修改异常。

**原因：**
		因为在调用hasNext方法的时候，光标的值和集合长度一样，那么就会返回false，此时就不会再去调用next方法获取集合的元素，那么底层就不会再抛出异常。

##### 3.5.2 迭代器默认的删除方法

```java
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
       //同样也调用了一个检测并发修改异常的方法
    checkForComodification();

    try {
        //这边调用的是根据索引删除元素的方法
        ArrayList.this.remove(lastRet);
        //在调用it.next方法时 lastRet++ 此时lastRet为0
        cursor = lastRet;
        //其实也是保证每次删除都是从头开始删
        lastRet = -1;
        //将集合实际修改次数赋值给预期修改次数
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```

```java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

**结论**

- 迭代器调用remove方法删除元素，其实底层真正还是调用自己的删除方法来删除元素。
- 在调用remove方法中，每次都回给预期删除次数的变量赋值。

#### 3.6 清空方法

```java
public void clear() {
    //实际删除次数自增
    modCount++;
	//将每一个位置置为空 让垃圾回收机制尽快回收
    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}
```

#### 3.7 包含方法

`public boolean contains(Object o)` 判断集合是否包含指定元素

```java
public boolean contains(Object o) {
    //根据返回值判断是否包含
    return indexOf(o) >= 0;
}
```

```java
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

#### 3.8 判断集合是否为空

`public boolean isEmpty()` 判断集合是否为空

```java
public boolean isEmpty() {
    return size == 0;
}
```

### 4 面试题

#### 4.1 ArrayList是如何扩容的？

​		ArrayList在创建的时候默认容量是0，在第一次调用add方法时会进行第一次扩容大小为10，以后每次扩容为原容量的1.5倍。

#### 4.2 ArrayList频繁扩容导致性能急剧下降，应该如何处理?

​		可以调用初始化数组容量的构造方法

```java
List<String> list = new ArrayList<>(100000);
```

#### 4.3 ArrayList插入或者删除元素一定比LinkedList慢吗？

分别查看下ArrayList和LinkedList删除操作的源码

`ArrayList`：

```java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    //复制操作 需要复制的数组  索引 复制到的数组 索引 复制的数量
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

`LinkedList`：

```java
public E remove(int index) {
    checkElementIndex(index);
    //unlink 为解绑方法 node(index) 是一个寻找元素的方法
    return unlink(node(index));
}
private void checkElementIndex(int index) {
    if (!isElementIndex(index))
       throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
//校验下删除的位置
private boolean isPositionIndex(int index) {
        return index >= 0 && index <= size;
}
```

LinkedList寻找元素的方法

```java
Node<E> node(int index) {
    // assert isElementIndex(index);
	//判断index是否小于集合长度的一半
    if (index < (size >> 1)) {
        //如果小于一半 将第一个值赋值给x
        Node<E> x = first;
        //向后面找 直到找到index
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        //大于一半 就将最后一个值赋值给x 向前找
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

解绑的方法：

```java
E unlink(Node<E> x) {
    // assert x != null;
    //获取元素的值
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    //实际修改次数自增
    modCount++;
    return element;
}
```

​		由此看来LinkedList在删除元素时需要找到待删除元素的位置，这个操作是相当耗时的，所以LinkedList插入或者删除元素实际上也不会比ArrayList快多少。

#### 4.4  ArrayList是线程安全的吗

ArrayList的线程安全问题主要体现在add这个方法上，在add方法时，会进行两个大的步骤：

- 判断数组容量是否满足需求
- 在elementData上设置值

第一个步骤：在多个线程进行add操作时可能会导致elementData数组越界。

第二个步骤：  `elementData[size++] = e` 不是原子操作，覆盖会使下标为空。

针对这个问题可以采用两个办法：

- 1 使用线程安全的Vector
- 2 通过Collections的工具类把List变成一个线程安全的集合。（`Collection.synchronizedList(list)`）

#### 4.5 如何复制一个ArrayList到另一个ArrayList中去？

- 使用clone方法
- 使用ArrayList构造方法
- 使用addAll方法

#### 4.6 已知成员变量集合储存N多用户名称，在多线程的环境下，使用迭代器在去取数据的同时，如何保证还可以正常写入数据到集合。

将ArrayList集合改为CopyOnWriteArrayList集合

4.7 ArrayList 和LinkedLIst的区别

**ArrayList**

- 基于动态数据的数据结构
- 对于随机访问的get和set,ArrayList 要优于LinkedList
- 对于随机操作的add和remove,ArrayList不一定会比LinkedList（ArrayList的底层是动态数组，因此并不是每次的add和remove都需要创建新数组）

**LinkedList**

- 基于链表的数据结构
- 对于顺序操作，LinkedList 不一定比ArrayList慢
- 对于随机操作，LinkedList效率明显降低

### 5  自定义ArrayList

```java
package com.gewei.Thread;


import java.sql.SQLOutput;

public class MyArrayList<E> {
    //自定义数组，用于储存
    private Object[] elementData;
    // 储存数组长度
    private int size;
    // 定义空数组，用于ArrayList初始化
    private Object[] emptyArray = {};
    //定义一个默认初始化的数组长度
    private final int DEFAULT_SIZE = 10;
    //构造方法
    public MyArrayList(){
        elementData = emptyArray;
    }
    //添加方法
    public boolean add(E e){
        grow();
        elementData[size++] = e;
        return true;
    }
    //简单扩容
    private void grow(){
        //集合为空数组时
        if(elementData == emptyArray){
            elementData = new Object[DEFAULT_SIZE];
        }
        //当size等于集合元素的长度时，需要扩容为原来的1.5倍
        if(size == elementData.length){
            int oldCapacitor = elementData.length;
            int newCapacitor = oldCapacitor + (oldCapacitor>>1);
            Object[] obj = new Object[newCapacitor];
            //拷贝元素
            System.arraycopy(elementData,0,obj,0,elementData.length);
            elementData = obj;
        }
    }
    //修改方法
    public E set(int index, E element){
        Check(index);
        E value = (E)elementData[index];
        elementData[index] = element;
        return value;
    }
    //检查索引是否符合规则
    private void Check(int index){
        if(index < 0 || index >= size){
            throw new IndexOutOfBoundsException("索引越界");
        }
    }
    //删除方法
    public E remove(int index){
        Check(index);
        E value = (E) elementData[index];
        //计算出需要移动的元素长度
        int numMove = size - index - 1;
        //移动
        if(numMove > 0){
            System.arraycopy(elementData,index+1, elementData,index,numMove);
        }
        elementData[--size] = null;
        return value;
    }
    //根据索引获取元素
    public E get(int index){
        Check(index);
        return (E) elementData[index];
    }
    //获取集合长度
    public int getSize(){
        return size;
    }
    //toString方法
    public String toString() {
        if(size == 0){
            return "[]";
        }
        StringBuilder sb = new StringBuilder();
        sb.append("[");
        for (int i = 0; i < size; i++) {
            if(i == size-1){
                sb.append(elementData[i]).append("]");
            }else{
                sb.append(elementData[i]).append(",").append(" ");
            }
        }
        return sb.toString();
    }

}
```

