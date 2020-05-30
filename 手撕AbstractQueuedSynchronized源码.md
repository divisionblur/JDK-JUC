### 手撕AbstractQueuedSynchronized源码

#### 手写ReentrantLock

```java
package com.xiaoliu.niubility;

import sun.misc.Unsafe;

import java.lang.reflect.Field;
import java.util.concurrent.locks.LockSupport;

public class MiniReentrantLock implements Lock{
    /**
     * 锁的是什么？
     * 资源 -> state
     * 0 表示未加锁状态
     * >0 表示当前lock是加锁状态..
     */
    private volatile int state;

    /**
     * 独占模式？
     * 同一时刻只有一个线程可以持有锁，其它的线程，在未获取到锁时，会被阻塞..
     */
    //当前独占锁的线程（占用锁线程）
    private Thread exclusiveOwnerThread;


    /**
     * 需要有两个引用是维护 阻塞队列
     * 1.Head 指向队列的头节点
     * 2.Tail 指向队列的尾节点
     */
    //比较特殊：head节点对应的线程 就是当前占用锁的线程
    private Node head;
    private Node tail;





    /**
     * 阻塞的线程被封装成什么？？
     * Node节点，然后放入到FIFO队列
     */
    static final class Node {
        //前置节点引用
        Node prev;
        //后置节点引用
        Node next;
        //封装的线程本尊
        Thread thread;

        public Node(Thread thread) {
            this.thread = thread;
        }

        public Node() {
        }
    }


    /**
     * 获取锁
     * 假设当前锁被占用，则会阻塞调用者线程,直到抢占到锁为止
     * 模拟公平锁
     * 什么是公平锁？ 讲究一个先来后到！
     *
     * lock的过程是怎样的？
     * 情景1：线程进来后发现，当前state == 0 ，这个时候就很幸运 直接抢锁.
     * 情景2：线程进来后发现，当前state > 0 , 这个时候就需要将当前线程入队。
     */
    @Override
    public void lock() {
        //第一次获取到锁时：将state == 1
        //第n次重入时：将state == n
        acquire(1);
    }

    /**
     * 竞争资源
     * 1.尝试获取锁，成功则 占用锁 且返回..
     * 2.抢占锁失败，阻塞当前线程..
     */
    private void acquire(int arg) {
        if(!tryAcquire(arg)) {
            Node node = addWaiter();
            acquireQueued(node, arg);
        }
    }

    /**
     * 尝试抢占锁失败，需要做什么呢？
     * 1.需要将当前线程封装成 node，加入到 阻塞队列
     * 2.需要将当前线程park掉，使线程处于 挂起 状态
     *
     * 唤醒后呢？
     * 1.检查当前node节点 是否为 head.next 节点
     *   （head.next 节点是拥有抢占权限的线程.其他的node都没有抢占的权限。）
     * 2.抢占
     *   成功：1.将当前node设置为head，将老的head出队操作，返回到业务层面
     *   失败：2.继续park，等待被唤醒..
     *
     * ====>
     * 1.添加到 阻塞队列的逻辑  addWaiter()
     * 2.竞争资源的逻辑        acquireQueued()
     */


    private void acquireQueued(Node node, int arg) {
        //只有当前node成功获取到锁 以后 才会跳出自旋。
        for(;;) {
            //什么情况下，当前node被唤醒之后可以尝试去获取锁呢？
            //只有一种情况，当前node是 head 的后继节点。才有这个权限。


            //head 节点 就是 当前持锁节点..
            Node pred = node.prev;
            if(pred == head/*条件成立：说明当前node拥有抢占权限..*/ && tryAcquire(arg)) {
                //这里面，说明当前线程 竞争锁成功啦！
                //需要做点什么？
                //1.设置当前head 为当前线程的node
                //2.协助 原始 head 出队
                setHead(node);
                pred.next = null; // help GC
                return;
            }

            System.out.println("线程：" + Thread.currentThread().getName() + "，挂起！");
            //将当前线程挂起！
            LockSupport.park();
            System.out.println("线程：" + Thread.currentThread().getName() + "，唤醒！");

            //什么时候唤醒被park的线程呢？
            //unlock 过程了！
        }
    }

    /**
     * 当前线程入队
     * 返回当前线程对应的Node节点
     *
     * addWaiter方法执行完毕后，保证当前线程已经入队成功！
     */
    private Node addWaiter() {
        Node newNode = new Node(Thread.currentThread());
        //如何入队呢？
        //1.找到newNode的前置节点 pred
        //2.更新newNode.prev = pred
        //3.CAS 更新tail 为 newNode
        //4.更新 pred.next = newNode

        //前置条件：队列已经有等待者node了，当前node 不是第一个入队的node
        Node pred = tail;
        if(pred != null) {
            newNode.prev = pred;
            //条件成立：说明当前线程成功入队！
            if(compareAndSetTail(pred, newNode)) {
                pred.next = newNode;
                return newNode;
            }
        }

        //执行到这里的有几种情况？
        //1.tail == null 队列是空队列
        //2.cas 设置当前newNode 为 tail 时 失败了...被其它线程抢先一步了...
        enq(newNode);
        return newNode;
    }

    /**
     * 自旋入队，只有成功后才返回.
     *
     * 1.tail == null 队列是空队列
     * 2.cas 设置当前newNode 为 tail 时 失败了...被其它线程抢先一步了...
     */
    private void enq(Node node) {
        for(;;) {
            //第一种情况：队列是空队列
            //==> 当前线程是第一个抢占锁失败的线程..
            //当前持有锁的线程，并没有设置过任何 node,所以作为该线程的第一个后驱，需要给它擦屁股、
            //给当前持有锁的线程 补充一个 node 作为head节点。  head节点 任何时候，都代表当前占用锁的线程。
            if(tail == null) {
                //条件成立：说明当前线程 给 当前持有锁的线程 补充 head操作成功了..
                if(compareAndSetHead(new Node())) {
                    tail = head;
                    //注意：并没有直接返回，还会继续自旋...
                }
            } else {
                //说明：当前队列中已经有node了，这里是一个追加node的过程。

                //如何入队呢？
                //1.找到newNode的前置节点 pred
                //2.更新newNode.prev = pred
                //3.CAS 更新tail 为 newNode
                //4.更新 pred.next = newNode

                //前置条件：队列已经有等待者node了，当前node 不是第一个入队的node
                Node pred = tail;
                if(pred != null) {
                    node.prev = pred;
                    //条件成立：说明当前线程成功入队！
                    if(compareAndSetTail(pred, node)) {
                        pred.next = node;
                        //注意：入队成功之后，一定要return。。
                        return;
                    }
                }
            }
        }
    }




    /**
     * 尝试获取锁，不会阻塞线程
     * true -> 抢占成功
     * false -> 抢占失败
     */
    private boolean tryAcquire(int arg) {

        if(state == 0) {
            //当前state == 0 时，是否可以直接抢锁呢？
            //不可以，因为咱们模拟的是  公平锁，先来后到..
            //条件一：!hasQueuedPredecessor() 取反之后值为true 表示当前线程前面没有等待者线程。
            //条件二：compareAndSetState(0, arg)  为什么使用CAS ? 因为lock方法可能有多线程调用的情况..
            //      成立：说明当前线程抢锁成功
            if(!hasQueuedPredecessor() && compareAndSetState(0, arg)) {
                //抢锁成功了，需要干点啥？
                //1.需要将exclusiveOwnerThread 设置为 当前进入if块中的线程
                this.exclusiveOwnerThread = Thread.currentThread();
                return true;
            }
            //什么时候会执行 else if 条件？
            //当前锁是被占用的时候会来到这个条件这里..

            //条件成立：Thread.currentThread() == this.exclusiveOwnerThread
            //说明当前线程即为持锁线程，是需要返回true的！
            //并且更新state值。
        } else if(Thread.currentThread() == this.exclusiveOwnerThread) {
            //这里面存在并发么？ 不存在的 。 只有当前加锁的线程，才有权限修改state。
            //锁重入的流程
            //说明当前线程即为持锁线程，是需要返回true的！
            int c = getState();
            c = c + arg;
            //越界判断..
            this.state = c;
            return true;
        }

        //什么时候会返回false?
        //1.CAS加锁失败
        //2.state > 0 且 当前线程不是占用者线程。
        return false;
    }


    /**
     * true -> 表示当前线程前面有等待者线程
     * false -> 当前线程前面没有其它等待者线程。
     *
     * 调用链
     * lock -> acquire -> tryAcquire  -> hasQueuedPredecessor (ps：state值为0 时，即当前Lock属于无主状态..)
     *
     * 什么时候返回false呢？
     * 1.当前队列是空..
     * 2.当前线程为head.next节点 线程.. head.next在任何时候都有权利去争取一下lock
     */
    private boolean hasQueuedPredecessor() {
        Node h = head;
        Node t = tail;
        Node s;

        //条件一：h != t
        //成立：说明当前队列已经有node了...
        //不成立：1. h == t == null   2. h == t == head 第一个获取锁失败的线程，会为当前持有锁的线程 补充创建一个 head 节点。


        //条件二：前置条件：条件一成立 ((s = h.next) == null || s.thread != Thread.currentThread())
        //排除几种情况
        //条件2.1：(s = h.next) == null
        //极端情况：第一个获取锁失败的线程，会为 持锁线程 补充创建 head ，然后再自旋入队，  1. cas tail() 成功了，2. pred【head】.next = node;
        //其实想表达的就是：已经有head.next节点了，其它线程再来这时  需要返回 true。

        //条件2.2：前置条件,h.next 不是空。 s.thread != Thread.currentThread()
        //条件成立：说明当前线程，就不是h.next节点对应的线程...返回true。
        //条件不成立：说明当前线程，就是h.next节点对应的线程，需要返回false，回头线程会去竞争锁了。
        return h != t && ((s = h.next) == null || s.thread != Thread.currentThread());
    }




    /**
     * 释放锁
     */
    @Override
    public void unlock() {
        release(1);
    }

    private void release(int arg) {
        //条件成立：说明线程已经完全释放锁了
        //需要干点啥呢？
        //阻塞队列里面还有好几个睡觉的线程呢？ 是不是 应该喊醒一个线程呢？
        if(tryRelease(arg)) {
            Node head = this.head;

            //你得知道，有没有等待者？   就是判断 head.next == null  说明没有等待者， head.next != null 说明有等待者..
            if(head.next != null) {
                //公平锁，就是唤醒head.next节点
                unparkSuccessor(head);
            }
        }
    }

    private void unparkSuccessor(Node node) {
        Node s = node.next;
        if(s != null && s.thread != null) {
            LockSupport.unpark(s.thread);
        }
    }

    /**
     * 完全释放锁成功，则返回true
     * 否则，说明当前state > 0 ，返回false.
     */
    private boolean tryRelease(int arg) {
        int c = getState() - arg;

        if(getExclusiveOwnerThread() != Thread.currentThread()) {
            throw new RuntimeException("fuck you! must getLock!");
        }
        //如果执行到这里？存在并发么？ 只有一个线程 ExclusiveOwnerThread 会来到这里。

        //条件成立：说明当前线程持有的lock锁 已经完全释放了..
        if(c == 0) {
            //需要做什么呢？
            //1.ExclusiveOwnerThread 置为null
            //2.设置 state == 0
            this.exclusiveOwnerThread = null;
            this.state = c;
            return true;
        }

        this.state = c;
        return false;
    }


    private void setHead(Node node) {
        this.head = node;
        //为什么？ 因为当前node已经是获取锁成功的线程了...
        node.thread = null;
        node.prev = null;
    }

    public int getState() {
        return state;
    }

    public Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }

    public Node getHead() {
        return head;
    }

    public Node getTail() {
        return tail;
    }

    private static final Unsafe unsafe;
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;

    static {	
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);

            stateOffset = unsafe.objectFieldOffset
                    (MiniReentrantLock.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset
                    (MiniReentrantLock.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset
                    (MiniReentrantLock.class.getDeclaredField("tail"));

        } catch (Exception ex) { throw new Error(ex); }
    }

    private final boolean compareAndSetHead(Node update) {
        return unsafe.compareAndSwapObject(this, headOffset, null, update);
    }

    private final boolean compareAndSetTail(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }

    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
}

```

#### AQS参数

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    static final class Node {
        //枚举：共享模式
        static final Node SHARED = new Node();
	    //枚举：独占模式
        static final Node EXCLUSIVE = null;

        //表示当前节点处于 取消 状态
        static final int CANCELLED =  1;
		//注释：表示当前节点需要唤醒他的后继节点。（SIGNAL 表示其实是 后继节点的状态，需要当前节点去喊它...）
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
   
        static final int PROPAGATE = -3;
		//node状态，可选值（0 ， SIGNAl（-1）, CANCELLED（1）, CONDITION, PROPAGATE）
        // waitStatus == 0  默认状态
        // waitStatus > 0 取消状态
        // waitStatus == -1 表示当前node如果是head节点时，释放锁之后，需要唤醒它的后继节点。
        volatile int waitStatus;

		//因为node需要构建成  fifo 队列， 所以 prev 指向 前继节点
        volatile Node prev;
		//因为node需要构建成  fifo 队列， 所以 next 指向 后继节点
        volatile Node next;
		//当前node封装的 线程本尊..
        volatile Thread thread;
        Node nextWaiter;
    }
       //头结点 任何时刻 头结点对应的线程都是当前持锁线程。
        private transient volatile Node head;

 
      //阻塞队列的尾节点   (阻塞队列不包含 头结点  head.next --->  tail 认为是阻塞队列)
        private transient volatile Node tail;
        //表示资源
        //独占模式：0 表示未加锁状态   >0 表示已经加锁状态
        private volatile int state;
}
```

#### 再读ReentrantLock

```java
package java.util.concurrent.locks;
import java.util.concurrent.TimeUnit;
import java.util.Collection;


public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;

    /**
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         */
        abstract void lock();

        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // Methods relayed from outer class

        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * Reconstitutes the instance from a stream (that is, deserializes it).
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }

    /**
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }

    /**
     * Sync object for fair locks
     * 公平锁
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        //AQS#cancelAcquire
        /**
         * 取消指定node参与竞争。
         */
        private void cancelAcquire(Node node) {
            //空判断..
            if (node == null)
                return;

            //因为已经取消排队了..所以node内部关联的当前线程，置为Null就好了。。
            node.thread = null;

            //获取当前取消排队node的前驱。
            Node pred = node.prev;

            while (pred.waitStatus > 0)
                node.prev = pred = pred.prev;

            //拿到前驱的后继节点。
            //1.当前node
            //2.可能也是 ws > 0 的节点。
            Node predNext = pred.next;

            //将当前node状态设置为 取消状态  1
            node.waitStatus = Node.CANCELLED;



            /**
             * 当前取消排队的node所在 队列的位置不同，执行的出队策略是不一样的，一共分为三种情况：
             * 1.当前node是队尾  tail -> node
             * 2.当前node 不是 head.next 节点，也不是 tail
             * 3.当前node 是 head.next节点。
             */


            //条件一：node == tail  成立：当前node是队尾  tail -> node
            //条件二：compareAndSetTail(node, pred) 成功的话，说明修改tail完成。
            if (node == tail && compareAndSetTail(node, pred)) {
                //修改pred.next -> null. 完成node出队。
                compareAndSetNext(pred, predNext, null);

            } else {


                //保存节点 状态..
                int ws;

                //第二种情况：当前node 不是 head.next 节点，也不是 tail
                //条件一：pred != head 成立， 说明当前node 不是 head.next 节点，也不是 tail
                //条件二： ((ws = pred.waitStatus) == Node.SIGNAL || (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL)))
                //条件2.1：(ws = pred.waitStatus) == Node.SIGNAL   成立：说明node的前驱状态是 Signal 状态   不成立：前驱状态可能是0 ，
                // 极端情况下：前驱也取消排队了..
                //条件2.2:(ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))
                // 假设前驱状态是 <= 0 则设置前驱状态为 Signal状态..表示要唤醒后继节点。
                //if里面做的事情，就是让pred.next -> node.next  ,所以需要保证pred节点状态为 Signal状态。
                if (pred != head &&
                        ((ws = pred.waitStatus) == Node.SIGNAL ||
                                (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                        pred.thread != null) {
                    //情况2：当前node 不是 head.next 节点，也不是 tail
                    //出队：pred.next -> node.next 节点后，当node.next节点 被唤醒后
                    //调用 shouldParkAfterFailedAcquire 会让node.next 节点越过取消状态的节点
                    //完成真正出队。
                    Node next = node.next;
                    if (next != null && next.waitStatus <= 0)
                        compareAndSetNext(pred, predNext, next);

                } else {
                    //当前node 是 head.next节点。  更迷了...
                    //类似情况2，后继节点唤醒后，会调用 shouldParkAfterFailedAcquire 会让node.next 节点越过取消状态的节点
                    //队列的第三个节点 会 直接 与 head 建立 双重指向的关系：
                    //head.next -> 第三个node  中间就是被出队的head.next 第三个node.prev -> head
                    unparkSuccessor(node);
                }

                node.next = node; // help GC
            }
        }

        //公平锁入口..
        //不响应中断的加锁..响应中断的 方法为：lockinterrupted。。
        final void lock() {
            acquire(1);
        }

        //AQS#release方法
        //ReentrantLock.unlock() -> sync.release()【AQS提供的release】
        public final boolean release(int arg) {
            //尝试释放锁，tryRelease 返回true 表示当前线程已经完全释放锁
            //返回false，说明当前线程尚未完全释放锁..
            if (tryRelease(arg)) {

                //head什么情况下会被创建出来？
                //当持锁线程未释放线程时，且持锁期间 有其它线程想要获取锁时，其它线程发现获取不了锁，而且队列是空队列，此时后续线程会为当前持锁中的
                //线程 构建出来一个head节点，然后后续线程  会追加到 head 节点后面。
                Node h = head;

                //条件一:成立，说明队列中的head节点已经初始化过了，ReentrantLock 在使用期间 发生过 多线程竞争了...
                //条件二：条件成立，说明当前head后面一定插入过node节点。
                if (h != null && h.waitStatus != 0)
                    //唤醒后继节点..
                    unparkSuccessor(h);
                return true;
            }

            return false;
        }

        //AQS#unparkSuccessor
        /**
         * 唤醒当前节点的下一个节点。
         */
        private void unparkSuccessor(Node node) {
            //获取当前节点的状态
            int ws = node.waitStatus;

            if (ws < 0)//-1 Signal  改成零的原因：因为当前节点已经完成喊后继节点的任务了..
                compareAndSetWaitStatus(node, ws, 0);

            //s是当前节点 的第一个后继节点。
            Node s = node.next;

            //条件一：
            //s 什么时候等于null？
            //1.当前节点就是tail节点时  s == null。
            //2.当新节点入队未完成时（1.设置新节点的prev 指向pred  2.cas设置新节点为tail   3.（未完成）pred.next -> 新节点 ）
            //需要找到可以被唤醒的节点..

            //条件二：s.waitStatus > 0    前提：s ！= null
            //成立：说明 当前node节点的后继节点是 取消状态... 需要找一个合适的可以被唤醒的节点..
            if (s == null || s.waitStatus > 0) {
                //查找可以被唤醒的节点...
                s = null;
                for (Node t = tail; t != null && t != node; t = t.prev)
                    if (t.waitStatus <= 0)
                        s = t;

              //上面循环，会找到一个离当前node最近的一个可以被唤醒的node。 node 可能找不到  node 有可能是null、、
            }


            //如果找到合适的可以被唤醒的node，则唤醒.. 找不到 啥也不做。
            if (s != null)
                LockSupport.unpark(s.thread);
        }


        //Sync#tryRelease()
        protected final boolean tryRelease(int releases) {
            //减去释放的值..
            int c = getState() - releases;
            //条件成立：说明当前线程并未持锁..直接异常.,.
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();

            //当前线程持有锁..


            //是否已经完全释放锁..默认false
            boolean free = false;
            //条件成立：说明当前线程已经达到完全释放锁的条件。 c == 0
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            //更新AQS.state值
            setState(c);
            return free;
        }


        //AQS#acquire
        public final void acquire(int arg) {
            //条件一：!tryAcquire 尝试获取锁 获取成功返回true  获取失败 返回false。
            //条件二：2.1：addWaiter 将当前线程封装成node入队
            //       2.2：acquireQueued 挂起当前线程   唤醒后相关的逻辑..
            //      acquireQueued 返回true 表示挂起过程中线程被中断唤醒过..  false 表示未被中断过..
            if (!tryAcquire(arg) &&
                    acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                //再次设置中断标记位 true
                selfInterrupt();
        }

        //acquireQueued 需要做什么呢？
        //1.当前节点有没有被park? 挂起？ 没有  ==> 挂起的操作
        //2.唤醒之后的逻辑在哪呢？   ==> 唤醒之后的逻辑。
        //AQS#acquireQueued
        //参数一：node 就是当前线程包装出来的node，且当前时刻 已经入队成功了..
        //参数二：当前线程抢占资源成功后，设置state值时 会用到。
        final boolean acquireQueued(final Node node, int arg) {
            //true 表示当前线程抢占锁成功，普通情况下【lock】 当前线程早晚会拿到锁..
            //false 表示失败，需要执行出队的逻辑... （回头讲 响应中断的lock方法时再讲。）
            boolean failed = true;
            try {
                //当前线程是否被中断
                boolean interrupted = false;
                //自旋..
                for (;;) {


                    //什么时候会执行这里？
                    //1.进入for循环时 在线程尚未park前会执行
                    //2.线程park之后 被唤醒后，也会执行这里...


                    //获取当前节点的前置节点..
                    final Node p = node.predecessor();
                    //条件一成立：p == head  说明当前节点为head.next节点，head.next节点在任何时候 都有权利去争夺锁.
                    //条件二：tryAcquire(arg)
                    //成立：说明head对应的线程 已经释放锁了，head.next节点对应的线程，正好获取到锁了..
                    //不成立：说明head对应的线程  还没释放锁呢...head.next仍然需要被park。。
                    if (p == head && tryAcquire(arg)) {
                        //拿到锁之后需要做什么？
                        //设置自己为head节点。
                        setHead(node);
                        //将上个线程对应的node的next引用置为null。协助老的head出队..
                        p.next = null; // help GC
                        //当前线程 获取锁 过程中..没有异常
                        failed = false;
                        //返回当前线程的中断标记..
                        return interrupted;
                    }

                    //shouldParkAfterFailedAcquire  这个方法是干嘛的？ 当前线程获取锁资源失败后，是否需要挂起呢？
                    //返回值：true -> 当前线程需要 挂起    false -> 不需要..
                    //parkAndCheckInterrupt()  这个方法什么作用？ 挂起当前线程，并且唤醒之后 返回 当前线程的 中断标记
                    // （唤醒：1.正常唤醒 其它线程 unpark 2.其它线程给当前挂起的线程 一个中断信号..）
                    if (shouldParkAfterFailedAcquire(p, node) &&
                            parkAndCheckInterrupt())
                        //interrupted == true 表示当前node对应的线程是被 中断信号唤醒的...
                        interrupted = true;
                }
            } finally {
                if (failed)
                    cancelAcquire(node);
            }
        }

        //AQS#parkAndCheckInterrupt
        //park当前线程 将当前线程 挂起，唤醒后返回当前线程 是否为 中断信号 唤醒。
        private final boolean parkAndCheckInterrupt() {
            LockSupport.park(this);
            return Thread.interrupted();
        }

        /**
         * 总结：
         * 1.当前节点的前置节点是 取消状态 ，第一次来到这个方法时 会越过 取消状态的节点， 第二次 会返回true 然后park当前线程
         * 2.当前节点的前置节点状态是0，当前线程会设置前置节点的状态为 -1 ，第二次自旋来到这个方法时  会返回true 然后park当前线程.
         *
         * 参数一：pred 当前线程node的前置节点
         * 参数二：node 当前线程对应node
         * 返回值：boolean  true 表示当前线程需要挂起..
         */
        private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
            //获取前置节点的状态
            //waitStatus：0 默认状态 new Node() ； -1 Signal状态，表示当前节点释放锁之后会唤醒它的第一个后继节点； >0 表示当前节点是CANCELED状态
            int ws = pred.waitStatus;
            //条件成立：表示前置节点是个可以唤醒当前节点的节点，所以返回true ==> parkAndCheckInterrupt() park当前线程了..
            //普通情况下，第一次来到shouldPark。。。 ws 不会是 -1
            if (ws == Node.SIGNAL)
                return true;

            //条件成立： >0 表示前置节点是CANCELED状态
            if (ws > 0) {
                //找爸爸的过程，条件是什么呢？ 前置节点的 waitStatus <= 0 的情况。
                do {
                    node.prev = pred = pred.prev;
                } while (pred.waitStatus > 0);
                //找到好爸爸后，退出循环
                //隐含着一种操作，CANCELED状态的节点会被出队。
                pred.next = node;

            } else {
                //当前node前置节点的状态就是 0 的这一种情况。
                //将当前线程node的前置node，状态强制设置为 SIGNAl，表示前置节点释放锁之后需要 喊醒我..
                compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
            }
            return false;
        }



        //AQS#addWaiter
        //最终返回当前线程包装出来的node
        private Node addWaiter(Node mode) {
            //Node.EXCLUSIVE
            //构建Node ，把当前线程封装到对象node中了
            Node node = new Node(Thread.currentThread(), mode);
            // Try the fast path of enq; backup to full enq on failure
            //快速入队
            //获取队尾节点 保存到pred变量中
            Node pred = tail;
            //条件成立：队列中已经有node了
            if (pred != null) {
                //当前节点的prev 指向 pred
                node.prev = pred;
                //cas成功，说明node入队成功
                if (compareAndSetTail(pred, node)) {
                    //前置节点指向当前node，完成 双向绑定。
                    pred.next = node;
                    return node;
                }
            }

            //什么时候会执行到这里呢？
            //1.当前队列是空队列  tail == null
            //2.CAS竞争入队失败..会来到这里..

            //完整入队..
            enq(node);

            return node;
        }

        //AQS#enq()
        //返回值：返回当前节点的 前置节点。
        private Node enq(final Node node) {
            //自旋入队，只有当前node入队成功后，才会跳出循环。
            for (;;) {
                Node t = tail;
                //1.当前队列是空队列  tail == null
                //说明当前 锁被占用，且当前线程 有可能是第一个获取锁失败的线程（当前时刻可能存在一批获取锁失败的线程...）
                if (t == null) { // Must initialize
                    //作为当前持锁线程的 第一个 后继线程，需要做什么事？
                    //1.因为当前持锁的线程，它获取锁时，直接tryAcquire成功了，没有向 阻塞队列 中添加任何node，所以作为后继需要为它擦屁股..
                    //2.为自己追加node

                    //CAS成功，说明当前线程 成为head.next节点。
                    //线程需要为当前持锁的线程 创建head。
                    if (compareAndSetHead(new Node()))
                        tail = head;

                    //注意：这里没有return,会继续for。。
                } else {
                    //普通入队方式，只不过在for中，会保证一定入队成功！
                    node.prev = t;
                    if (compareAndSetTail(t, node)) {
                        t.next = node;
                        return t;
                    }
                }
            }
        }


        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         * 抢占成功：返回true  包含重入..
         * 抢占失败：返回false
         */
        protected final boolean tryAcquire(int acquires) {
            //current 当前线程
            final Thread current = Thread.currentThread();
            //AQS state 值
            int c = getState();
            //条件成立：c == 0 表示当前AQS处于无锁状态..
            if (c == 0) {
                //条件一：
                //因为fairSync是公平锁，任何时候都需要检查一下 队列中是否在当前线程之前有等待者..
                //hasQueuedPredecessors() 方法返回 true 表示当前线程前面有等待者，当前线程需要入队等待
                //hasQueuedPredecessors() 方法返回 false 表示当前线程前面无等待者，直接尝试获取锁..

                //条件二：compareAndSetState(0, acquires)
                //成功：说明当前线程抢占锁成功
                //失败：说明存在竞争，且当前线程竞争失败..
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    //成功之后需要做什么？
                    //设置当前线程为 独占者 线程。
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //执行到这里，有几种情况？
            //c != 0 大于0 的情况，这种情况就需要检查一下 当前线程是不是 独占锁的线程，因为ReentrantLock是可以重入的.

            //条件成立：说明当前线程就是独占锁线程..
            else if (current == getExclusiveOwnerThread()) {
                //锁重入的逻辑..

                //nextc 更新值..
                int nextc = c + acquires;
                //越界判断，当重入的深度很深时，会导致 nextc < 0 ，int值达到最大之后 再 + 1 ...变负数..
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                //更新的操作
                setState(nextc);
                return true;
            }

            //执行到这里？
            //1.CAS失败  c == 0 时，CAS修改 state 时 未抢过其他线程...
            //2.c > 0 且 ownerThread != currentThread.
            return false;
        }
    }

    /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

 
    public void lock() {
        sync.lock();
    }

    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }


    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

 
    public void unlock() {
        sync.release(1);
    }

    public Condition newCondition() {
        return sync.newCondition();
    }


    public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }

    public boolean isLocked() {
        return sync.isLocked();
    }

    public final boolean isFair() {
        return sync instanceof FairSync;
    }

    protected Thread getOwner() {
        return sync.getOwner();
    }

    public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }


    public final boolean hasQueuedThread(Thread thread) {
        return sync.isQueued(thread);
    }


    public final int getQueueLength() {
        return sync.getQueueLength();
    }

    protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }


    public boolean hasWaiters(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
    }


    public int getWaitQueueLength(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
    }


    protected Collection<Thread> getWaitingThreads(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitingThreads((AbstractQueuedSynchronizer.ConditionObject)condition);
    }


    public String toString() {
        Thread o = sync.getOwner();
        return super.toString() + ((o == null) ?
                                   "[Unlocked]" :
                                   "[Locked by thread " + o.getName() + "]");
    }
}

```

