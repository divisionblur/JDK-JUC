## Java并发编程

#### 1  进程与线程

##### 1.1 进程与线程

**进程：**

- 程序由数据和指令构成，但这些指令要运行，数据要读写，就必须要将指令加载到CPU，数据加载到内存。在指令的运行过程中还需要用到磁盘、网络等设备，进程就是用来加载指令、管理内存、IO的。
- 当一个程序被运行，从磁盘加载这个程序的代码进内存，这时就开启了一个进程。
- 进程就可以视为程序的一个实例。大部分程序可以同时运行多个实例进程（例如记事本、画图、计算器等），也有的程序只能启动一个实例进程（例如网易云音乐、360安全卫士等）

**线程：**

- 一个进程之内可以分为一到多个线程。
- 一个线程就是一个指令流，将指令流中的一条条指令以一定的顺序交给 CPU 执行
- Java 中，线程作为最小调度单位，进程作为资源分配的最小单位。 在 windows 中进程是不活动的，只是作
  为线程的容器

**二者对比**

- 进程基本上相互独立的，而线程存在于进程内，是进程的一个子集
- 进程拥有共享的资源，如内存空间等，供其内部的线程共享
- 进程间通信较为复杂同一台计算机的进程通信称为 IPC（Inter-process communication）
  不同计算机之间的进程通信，需要通过网络，并遵守共同的协议，例如 HTTP
- 线程通信相对简单，因为它们共享进程内的内存，一个例子是多个线程可以访问同一个共享变量
  线程更轻量，线程上下文切换成本一般上要比进程上下文切换低

##### 1.2 并行与并发

并行：

​		单核 cpu 下，线程实际还是 串行执行 的。操作系统中有一个组件叫做任务调度器，将 cpu 的时间片（windows
下时间片最小约为 15 毫秒）分给不同的程序使用，只是由于 cpu 在线程间（时间片很短）的切换非常快，人类感
觉是  同时运行的 。总结为一句话就是： 微观串行，宏观并行 ，
**一般会将这种  线程轮流使用 CPU 的做法称为并发， concurrent**

并发：

​		**多核 cpu下，每个  核（core） 都可以调度运行线程，这时候线程可以是并行的**

- 并发（concurrent）是同一时间应对（dealing with）多件事情的能力

- 并行（parallel）是同一时间动手做（doing）多件事情的能力

##### 1.3 异步

- 需要等待结果返回，才能继续运行就是同步
- 不需要等待结果返回，就能继续运行就是异步

1) 设计

​		多线程可以让方法执行变为异步的（即不要巴巴干等着）比如说读取磁盘文件时，假设读取操作花费了 5 秒钟，如
果没有线程调度机制，这 5 秒 cpu 什么都做不了，其它代码都得暂停...
**2) 结论**

- 比如在项目中，视频文件需要转换格式等操作比较费时，这时开一个新线程处理视频转换，避免阻塞主线程
  - tomcat 的异步 servlet 也是类似的目的，让用户线程处理耗时较长的操作，避免阻塞 tomcat 的工作线程
    ui 程序中，开线程进行其他操作，避免阻塞 ui 线程

#### 2 Java线程

##### 2.4 常见方法

|     方法名      | static |                  功能说明                   | 注意                                                         |
| :-------------: | :----: | :-----------------------------------------: | ------------------------------------------------------------ |
|     start()     |        | 启动一个线程，在新的线程运行run方法中的代码 | start 方法只是让线程进入就绪，里面代码不一定立刻运行（CPU 的时间片还没分给它）。每个线程对象的start方法只能调用一次，如果调用了多次会出现IllegalThreadStateException |
|      run()      |        |          新线程启动后会调用的方法           | 如果在构造 Thread 对象时传递了 Runnable 参数，则线程启动后会调用 Runnable 中的 run 方法，否则默认不执行任何操作。但可以创建 Thread 的子类对象，来覆盖默认行为 |
|     join()      |        |              等待线程运行结束               |                                                              |
|  join(long n)   |        |       等待线程运行结束,最多等待 n毫秒       |                                                              |
|     getId()     |        |                 获取线程Id                  | Id唯一                                                       |
|    getName()    |        |                 获取线程名                  |                                                              |
| setName(String) |        |                 修改线程名                  |                                                              |
|  getPriority()  |        |                 获取优先级                  |                                                              |
|  setPrority()   |        |               修改线程优先级                | java中规定线程优先级是1~10 的整数，较大的优先级能提高该线程被 CPU 调度的机率 |
|   getState()    |        |                获取线程状态                 | Java 中线程状态是用 6 个 enum 表示，分别为：NEW, RUNNABLE, BLOCKED, WAITING,TIMED_WAITING, TERMINATED |
| isInterrupted() |        |               判断是否被打断                | 不会清楚（打断标记）                                         |
|    isAlive()    |        |       线程是否存活（有没有运行完毕）        |                                                              |
|   interrupt()   |        |                  打断线程                   | 如果被打断的线程正在sleep,wait,join会导致被打断的线程抛出InterruptionException,并清除打断标记；如果打断正在运行的线程也会设置打断标记；park的线程被打断也会出现打断标记 |
|  interrupted()  | static |           判断当前线程是否被打断            | 会清除（打断标记）                                           |
| currentThread() | static |             获取当前执行的线程              |                                                              |
|  sleep(long n)  | static |          让当前执行的线程休眠n毫秒          |                                                              |
|      yield      | static |    提示线程调度器让出当前线程对CPU的使用    | 主要是为了测设和调试                                         |

##### 2.5 start 与 run

```java
public static void main(String[] args) {
    Thread t1 = new Thread("t1") {
        @Override
        public void run() {
            log.debug("running...");
        }
    };

    System.out.println(t1.getState());
    t1.start();
    System.out.println(t1.getState());
}
NEW
RUNNABLE
```

- 直接调用run是在主线程中执行了run，没有启动新的线程
- 使用start是启动了新的线程，通过新的线程间接访问了run中的代码

##### 2.6  sleep 与 yield

**sleep**

- 调用 sleep 会让当前线程从 Running 进入 Timed Waiting 状态（阻塞）
- 其它线程可以使用 interrupt 方法打断正在睡眠的线程，这时 sleep 方法会抛出  InterruptedException
- 睡眠结束后的线程未必会立刻得到执行
- 建议用 TimeUnit 的 sleep 代替 Thread 的 sleep 来获得更好的可读性

**yield**

- 调用 yield 会让当前线程从 Running 进入 Runnable 就绪状态，然后调度执行其它线程
- 具体的实现依赖于操作系统的任务调度器

**区别：** Runnable还是有重新获取CPU时间片的可能，让这个线程执行起来的。阻塞状态会把时间片交给那些就绪状态的线程。



##### 2.7 线程优先级

```java
public final static int MIN_PRIORITY = 1; // 最低优先级
public final static int NORM_PRIORITY = 5; // 默认优先级
public final static int MAX_PRIORITY = 10; // 最高优先级
```

- 线程优先级会提示（hint）任务调度器优先执行该线程，但他仅仅是一个提示，调度器可以忽略他
- 如果CPU比较忙，那么优先级高的线程会获取更多的时间片，但CPU闲时，调度器可以忽略它

##### 2.8 Interrupt

打断sleep, wite, join 的线程

```java
private static void test1() throws InterruptedException {
	Thread t1 = new Thread(()->{
		sleep(1);
	}, "t1");
	t1.start();
	sleep(0.5);
	t1.interrupt();
	log.debug(" 打断状态: {}", t1.isInterrupted());
}
```

输出：

```java
java.lang.InterruptedException: sleep interrupted
at java.lang.Thread.sleep(Native Method)
at java.lang.Thread.sleep(Thread.java:340)
at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
at cn.itcast.n2.util.Sleeper.sleep(Sleeper.java:8)
at cn.itcast.n4.TestInterrupt.lambda$test1$3(TestInterrupt.java:59)
at java.lang.Thread.run(Thread.java:745)
21:18:10.374 [main] c.TestInterrupt - 打断状态: false
```

打断正常运行的线程

不会清空打断状态

```java
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(() -> {
        while(true) {
            boolean interrupted = Thread.currentThread().isInterrupted();
            if(interrupted) {
                log.debug("被打断了, 退出循环");
                break;
            }
        }
    }, "t1");
    t1.start();

    Thread.sleep(1000);
    log.debug("interrupt");
    t1.interrupt();
}
```

输出

```
09:06:14.389 c.Test12 [main] - interrupt
09:06:14.397 c.Test12 [t1] - 被打断了, 退出循环
```

##### 2.9 两阶段终止模式

Two phase Termination

一个线程T1如何“优雅”的终止线程T2?这里的优雅指的是给T2一个料理后事的机会

​															*1.错误思路*

- 使用线程的stop方法停止线程
  - stop方法会真正的杀死线程，如果这个线程锁住了共享资源，那么当他被杀死后就再也没有机会释放锁，其他线程将永远无法获取锁。
- 使用System.exit(int) 方法杀死停止线程
  - 目的是停止线程，这个方法把整个程序都给停止了

正确做法：

使用Interrupt打断，但是如果线程在阻塞的状态被打断，isInterrupt会被置位false怎么办？

答：可以捕获异常并将Thread.isInterrupt置为True然后等待线程结束循环。

**注意点：**

在使用park方法时，线程调用Interrupt打断，之后的park会失效。**可以使用Interrupted方法使打断标记为false**

##### 2.10 不推荐使用的方法

还有些不推荐使用的方法，这些方法已经过时，容易破坏同步代码块，造成线程死锁。

|  方法名   | static |       功能说明       |
| :-------: | :----: | :------------------: |
|  stop()   |        |     线程同步运行     |
| suspend() |        | 挂起（暂停）线程运行 |
| resume()  |        |    恢复线程运行2.    |

##### 2.11 守护线程

默认情况下，Java 进程需要等待所有线程都运行结束，才会结束。有一种特殊的线程叫做守护线程，只要其它非守
护线程运行结束了，即使守护线程的代码没有执行完，也会强制结束。

方法thread.setDaemon(true)

```java
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(() -> {
        while (true) {
            if (Thread.currentThread().isInterrupted()) {
                break;
            }
        }
        log.debug("结束");
    }, "t1");
    t1.setDaemon(true);
    t1.start();

    Thread.sleep(1000);
    log.debug("结束");
}
```

```
09:42:10.043 c.Test15 [main] - 结束
```

##### 2.12 5种运行状态

这是从操作系统层面上描述的分别为：初始状态、可运行状态、运行状态、阻塞状态、终止状态

- 【初始状态】仅是在语言层面创建了线程对象，还未与操作系统线程关
- 【可运行状态】（就绪状态）指该线程已经被创建（与操作系统线程关联），可以由 CPU 调度执行
- 【运行状态】指获取了 CPU 时间片运行中的状态
  - 当 CPU 时间片用完，会从【运行状态】转换至【可运行状态】，会导致线程的上下文切换
- 【阻塞状态】
  - 如果调用了阻塞 API，如 BIO 读写文件，这时该线程实际不会用到 CPU，会导致线程上下文切换，进入【阻塞状态】
  - 等 BIO 操作完毕，会由操作系统唤醒阻塞的线程，转换至【可运行状态】
  - 与【可运行状态】的区别是，对【阻塞状态】的线程来说只要它们一直不唤醒，调度器就一直不会考虑
    调度它们

- 【终止状态】表示线程已经执行完毕，生命周期已经结束，不会再转换为其它状态

##### 2.13 六种状态

着是从Java API层面来描述的

根据Thread.State枚举，分为六种状态：NEW, RUNABLE, BLOCK, WAITING, TIMED_WAITING, TERMINATED

- NEW 线程刚被创建，但是还没有调用  start() 方法
- RUNNABLE 当调用了  start() 方法之后，注意，Java API 层面的  RUNNABLE 状态涵盖了 操作系统 层面的【可运行状态】、【运行状态】和【阻塞状态】（由于 BIO 导致的线程阻塞，在 Java 里无法区分，仍然认为
  是可运行）
- BLOCKED ， WAITING ， TIMED_WAITING 都是 Java API 层面对【阻塞状态】的细分。
- TERMINATED 当线程代码运行结束

#### 3 并发编程原理

##### Monitor原理

Monitor可以被翻译为监视器或管程，每个Java对象都可以关联一个Monitor对象，如果使用synchronized给对象加上锁以后，该对象头的MarkWord就被设置了指向Monitor的指针

Monitor结构如下

![image-20200503184223436](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200503184223436.png)

- 一开始Monitor的Owner为null
- 当线程执行到synchronize（object）就会将Monitor的所有者置位Thread-2，Monitor中只能有一个Owner。
- 在Thread-2上锁的过程中，如果Thread-3、Thread-1也来执行synchronized(object)，就会进入EntryList BLOCK状态。
- Thread-2执行完同步代码块的内容，释放锁，唤醒EntryList中等待的线程竞争锁，竞争这个过程是非公平的。
- 图中 WaitSet 中的 Thread-0，Thread-1 是之前获得过锁，但条件不满足进入 WAITING 状态的线程，后面
  wait-notify 时会分析。

**注意**

- synchronized 必须是进入同一个对象的 monitor 才有上述的效果
- 不加 synchronized 的对象不会关联监视器，不遵从以上规则

##### 3.5 synchronized原理

###### 3.5.1 轻量级锁

轻量级锁使用场景：如果一个对象有多个线程使用需要加锁，但是加锁时间是错开的（也就是说没有竞争），那么可以使用轻量级锁来优化。

轻量级锁对使用者来说是透明的，语法仍然是synchronized；

加锁过程：

- 创建锁记录（Lock Record）对象，对象线程的栈帧会包含一个锁记录的结构，内部可以储存锁定对象的MarkWord

![image-20200503185559470](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200503185559470.png)

- 让锁记录的Object reference 指向锁对象，并尝试使用CAS替换Object的MarkWord，将MarkWord的值存入锁对象。

  ![image-20200503185758839](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200503185758839.png)

- 如果替换成功，对象头中记录锁记录地址和锁状态00，表示该线程给该对象加锁，图示如下

  ![image-20200503190000801](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200503190000801.png)

- 如果cas失败，有两种情况
  - 如果是其他线程已经持有了该对象的轻量级锁，这说明之间存在竞争，首先先自旋等待获取锁，如果自旋超过规定的阈值，那么会进入锁膨胀。
  - 如果是自己持有了锁，那么再添加一条LockRecord作为冲入的计数

![image-20200503190422023](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200503190422023.png)

- 当退出synchronized代码块时（解锁时）如果有取值为null的锁记录，表示有重入，重置锁记录，表示重入技术减一。

- 当退出synchronized代码块（解锁时）所记录不为null，这时利用cas操作将MarkWord的值恢复给对象头
  - 成功，解锁成功
  - 失败，表明升级为重量级锁，进入重量级锁解锁流程

###### 3.5.2 锁膨胀

如果在尝试加轻量级锁的过程中，CAS 操作无法成功，这时一种情况就是有其它线程为此对象加上了轻量级锁（有
竞争），这时需要进行锁膨胀，将轻量级锁变为重量级锁。

**锁膨胀过程：**

- 当 Thread-1 进行轻量级加锁时，Thread-0 已经对该对象加了轻量级锁

![image-20200503191026842](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200503191026842.png)

- 这时 Thread-1 加轻量级锁失败，进入锁膨胀流程
  - 即为 Object 对象申请 Monitor 锁，让 Object 指向重量级锁地址
  - 然后自己进入 Monitor 的 EntryList BLOCKED

###### ![image-20200503191149200](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200503191149200.png)



###### 3.5.3偏向锁

轻量级锁在没有锁竞争的时候（就自己一个过程），每次依然要执行cas操作

Java1.6中引入了偏向锁来做进一步的优化：只有第一次使用CAS将线程的ID存放在对象头的MarkWord头，之后发现这个线程ID是自己就表示没有竞争，不需要重新CAS，以后只要不发生竞争，这个对象就归该线程所有。

对象头格式：

![image-20200504071350420](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200504071350420.png)

一个对象创建时：

- 如果开启了偏向锁（默认开启），那么对象创建后，markword的值为0x05即最后三位为101，这是他的Thread，epoch，unused都为0
- 偏向锁是默认是延迟的，不会在程序启动时立即生效，如果想避免延迟，可以加 VM 参数  -
  XX:BiasedLockingStartupDelay=0 来禁用延迟
- 如果没有开启偏向锁，那么对象创建后，markword 值为 0x01 即最后 3 位为 001，这时它的 hashcode、
  age 都为 0，第一次用到 hashcode 时才会赋值

**撤销 - 调用对象 hashCode**

调用了对象的hashcode，但偏向锁的对象MarkWord中储存的是线程id，如果调用hashcode会导致偏向锁被撤销

- 轻量级锁会在锁记录中存放hashcode
- 重量级锁在monitor中存放hashcode

**批量重偏向**

- 如果对象虽然被多个线程访问，但没有竞争，这时偏向了线程 T1 的对象仍有机会重新偏向 T2，重偏向会重置对象
  的 Thread ID
- 当撤销偏向锁阈值超过 20 次后，jvm 会这样觉得，我是不是偏向错了呢，于是会在给这些对象加锁时重新偏向至
  加锁线程

**批量撤销**

当撤销偏向锁阈值超过 40 次后，jvm 会这样觉得，自己确实偏向错了，根本就不该偏向。于是整个类的所有对象
都会变为不可偏向的，新建的对象也是不可偏向的

##### 4 wait notify 原理

![image-20200504072352161](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200504072352161.png)

- Owner线程发现条件不满足时，调用wait方法，即可进入WaitSet变为WAITING状态
- BLOCK和WAITING都属于阻塞状态不占用CPU时间片
- BLOCK线程会在Owner线程释放锁的时唤醒
- WAITING

##### 3.5 final原理

### 函数式编程

supplier 提供者  无中生有 ()->结果

function 函数 一个参数一个结果 (参数)->结果  bifunction   (参数1,参数2)->结果

consumer 消费者  一个参数没有结果 (参数)->void biconsumer (参数1,参数2)->void

#### 5 共享模型之不可变

##### 5.1 日期转换的问题

下面的代码在运行时，由于 SimpleDateFormat 不是线程安全的

```java
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
for (int i = 0; i < 10; i++) {
    new Thread(() -> {
        try {
            log.debug("{}", sdf.parse("1951-04-21"));
        } catch (Exception e) {
            log.error("{}", e);
        }
    }).start();
}
```

有很大几率出现 java.lang.NumberFormatException 或者出现不正确的日期解析结果。

思路 - 同步锁
这样虽能解决问题，但带来的是性能上的损失，并不算很好：

```java
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
for (int i = 0; i < 50; i++) {
    new Thread(() -> {
        synchronized (sdf) {
            try {
                log.debug("{}", sdf.parse("1951-04-21"));
            } catch (Exception e) {
                log.error("{}", e);
            }
        }
    }).start();
}
```

思路 - 不可变
如果一个对象在不能够修改其内部状态（属性），那么它就是线程安全的，因为不存在并发修改啊！这样的对象在
Java 中有很多，例如在 Java 8 后，提供了一个新的日期格式化类：

```java
DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-dd");
for (int i = 0; i < 10; i++) {
    new Thread(() -> {
        LocalDate date = dtf.parse("2018-10-01", LocalDate::from);
        log.debug("{}", date);
    }).start();
}
```

可以看 DateTimeFormatter 的文档：

```java
@implSpec
This class is immutable and thread-safe.
```

不可变对象，实际是另一种避免竞争的方式。

##### 5.2 不可变设计

另一个大家更为熟悉的 String 类也是不可变的，以它为例，说明一下不可变设计的要素

```java
public final class String
                implements java.io.Serializable, Comparable<String>, CharSequence {
            /** The value is used for character storage. */
            private final char value[];
            /** Cache the hash code for the string */
            private int hash; // Default to 0
// ...
        }
```

**final 的使用**
发现该类、类中所有属性都是 final 的

- 属性用 final 修饰保证了该属性是只读的，不能修改
- 类用 final 修饰保证了该类中的方法不能被覆盖，防止子类无意间破坏不可变性

**保护性拷贝**
使用字符串时，也有一些跟修改相关的方法啊，比如 substring 等，那么下面就看一看这些方法是如何实现的，就以 substring 为例：

```java
public String substring(int beginIndex) {
    if (beginIndex < 0) {
        throw new StringIndexOutOfBoundsException(beginIndex);
    }
    int subLen = value.length - beginIndex;
    if (subLen < 0) {
        throw new StringIndexOutOfBoundsException(subLen);
    }
    return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
}
```

发现其内部是调用 String 的构造方法创建了一个新字符串，再进入这个构造看看，是否对 final char[] value 做出
了修改：

```java
public String(char value[], int offset, int count) {
      if (offset < 0) {
          throw new StringIndexOutOfBoundsException(offset);
      }
      if (count <= 0) {
          if (count < 0) {
              throw new StringIndexOutOfBoundsException(count);
          }
          if (offset <= value.length) {
              this.value = "".value;
              return;
          }
      }
      if (offset > value.length - count) {
          throw new StringIndexOutOfBoundsException(offset + count);
      }
      this.value = Arrays.copyOfRange(value, offset, offset+count);
  }
```

结果发现也没有，构造新字符串对象时，会生成新的 char[] value，对内容进行复制 。这种通过创建副本对象来避
免共享的手段称之为【保护性拷贝（defensive copy）】

##### 5.3 享元模式

1. 简介
- 定义 英文名称：Flyweight pattern. 当需要重用数量有限的同一类对象时
- wikipedia： A flyweight is an object that minimizes memory usage by sharing as much data as possible with other similar objects
- 出自 "Gang of Four" design patterns
- 归类 Structual patterns
2. 体现

在JDK中 Boolean，Byte，Short，Integer，Long，Character 等包装类提供了 valueOf 方法，例如 Long 的
valueOf 会缓存 -128~127 之间的 Long 对象，在这个范围之间会重用对象，大于这个范围，才会新建 Long 对
象：

```java
public static Long valueOf(long l) {
    final int offset = 128;
    if (l >= -128 && l <= 127) { // will cache
        return LongCache.cache[(int)l + offset];
    }
    return new Long(l);
}
```

**注意：**

- Byte, Short, Long 缓存的范围都是 -128~127

- Character 缓存的范围是 0~127

- Integer的默认范围是 -128~127

  - 最小值不能变
  - 但最大值可以通过调整虚拟机参数 `
    -Djava.lang.Integer.IntegerCache.high` 来改变

- Boolean 缓存了 TRUE 和 FALSE

  同样的String 串池  BigDecimal BigInteger 也是利用了享元模式作为一个不可变类

3. DIY

例如：一个线上商城应用，QPS 达到数千，如果每次都重新创建和关闭数据库连接，性能会受到极大影响。 这时
预先创建好一批连接，放入连接池。一次请求到达后，从连接池获取连接，使用完毕后再还回连接池，这样既节约
了连接的创建和关闭时间，也实现了连接的重用，不至于让庞大的连接数压垮数据库。