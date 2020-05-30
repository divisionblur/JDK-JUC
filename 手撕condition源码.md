### 手撕condition源码

##### 手写BrokingQueue

```java
public class MiniArrayBrokingQueue implements BrokingQueue {
    //线程并发控制
    private Lock lock = new ReentrantLock();
    /**
     * 当生产者线程生产数据时，它会先检查当前queues是否已经满了，如果已经满了，需要将当前生产者线程 调用notFull.await()
     * 进入到notFull条件队列挂起。等待消费者线程消费数据时唤醒。
     */
    private Condition notFull = lock.newCondition();

    /**
     * 当消费者线程消费数据时，它会先检查当前queues中是否有数据，如果没有数据,需要将当前消费者线程 调用notEmpty.await()
     * 进入到notEmpty条件队列挂起。等待生产者线程生产数据时唤醒。
     */
    private Condition notEmpty = lock.newCondition();


    //底层存储元素的数组
    private Object[] queues;
    //数组长度
    private int size;

    /**
     * count:当前队列中可以被消费的数据量
     * putptr:记录生产者存放数据的下一次位置。每个生产者生产完一个数据后，会将 putptr ++
     * takeptr:记录消费者消费数据的下一次的位置。每个消费者消费完一个数据后，将将takeptr ++
     */
    private int count,putptr, takeptr;


    public MiniArrayBrokingQueue(int size) {
        this.size = size;
        this.queues = new Object[size];
    }



    @Override
    public void put(Object element) throws InterruptedException {
        lock.lock();
        try {
            //第一件事？ 判断一下当前queues是否已经满了...
            if(count == this.size) {
                notFull.await();
            }

            //执行到这里，说明队列未满，可以向队列中存放数据了..
            this.queues[putptr] = element;

            putptr ++;

            if(putptr == this.size) putptr = 0;
            //生产数据 自增count
            count ++;

            //当向队列中成功放入一个元素之后，需要做什么呢？
            //需要给notEmpty一个唤醒信号
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    @Override
    public Object take() throws InterruptedException {
        lock.lock();
        try {
            //第一件事？判断一下当前队列是否有数据可以被消费...
            if(count == 0) {
                notEmpty.await();
            }

            //执行到这里，说明队列有数据可以被消费了..
            Object element = this.queues[takeptr];

            takeptr ++;
            if(takeptr == this.size) takeptr = 0;
            //生产数据 自减count
            count --;

            //当向队列中成功消费一个元素之后，需要做什么呢？
            //需要给notFull一个唤醒信号
            notFull.signal();

            return element;
        }finally {
            lock.unlock();
        }
    }


    public static void main(String[] args) {
        BrokingQueue<Integer> queue = new MiniArrayBrokingQueue(10);

        Thread producer = new Thread(() -> {
            int i = 0;
            while(true) {
                i ++;
                if(i == 10) i = 0;

                try {
                    System.out.println("生产数据：" + i);
                    queue.put(Integer.valueOf(i));
                    TimeUnit.MILLISECONDS.sleep(200);
                } catch (InterruptedException e) {
                }
            }
        });
        producer.start();


        Thread consumer = new Thread(() -> {
            while(true) {
                try {
                    Integer result = queue.take();
                    System.out.println("消费者消费：" + result);
                    TimeUnit.MILLISECONDS.sleep(200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        consumer.start();
    }
}

```

##### await方法

```java
public final void await() throws InterruptedException {
    //判断当前线程是否中断，如果中断直接返回一个中断异常
    if (Thread.interrupted())
        throw new InterruptedException();
    //将调用await方法的当前线程封装成为一个node并且加入条件队列中，并返回当前线程的node
    Node node = addConditionWaiter();
    //释放锁
    int savedState = fullyRelease(node);
    //0 在condition队列挂起期间未收到过中断信号
    //-1 在condition队列挂起期间收到过中断信号
    //1 在condition队列挂起期间未收到中断信号，但是迁移到阻塞队列之后收到中断信号了
    int interruptMode = 0;
    //isOnSynQueue就是加入阻塞队列的一个过程
    //true 表示当前node已经加入阻塞队列中了
    //false 当前队列仍在条件队列中，需要park
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        
        
        //合适会唤醒：
        //1 正常逻辑下，外部线程获取锁之后，调用signal方法，转移条件队列头节点到阻塞队列，该节点获取锁后，会唤醒
        //2 转移至条件队列中，发现前驱结点为取消状态 会唤醒当前节点
        //3 当前线程挂起期间，被外部线程中断唤醒
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //执行到这里说明当前node已经加入阻塞队列了
    //条件一： true 表示在阻塞队列中，被外部线程中断唤醒过
    //interruptMode != THROW_IE 表示当前线程在条件队列中未发生过中断
    //设置interruptMode = REINTERRUPT 进入阻塞队列的过程中发生中断
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
     //考虑下 node.nextWaiter != null 条件什么时候成立呢？
     //其实是node在条件队列内时 如果被外部线程 中断唤醒时，会加入到阻塞队列，但是并未设置nextWaiter = null。
    if (node.nextWaiter != null) // clean up if cancelled
        //清理条件队列内取消状态的节点..
        unlinkCancelledWaiters();
    //条件成立：说明挂起期间 发生过中断（1.条件队列内的挂起 2.条件队列之外的挂起）
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

##### addConditionWaiter 方法

```java
//调用await方法的线程都是持琐转态，不存在并发
private Node addConditionWaiter() {
    //获取天剑队列尾结点引用，保存在局部变量t中
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    //条件一: true 当前条件队列中已经有元素了
    //条件二：node在条件队列中的状态是condition（-2）
    //t.waitStatus != Node.CONDITION 表示当前node发生中断了
    if (t != null && t.waitStatus != Node.CONDITION) {
        //清理条件队列中处于取消队列的节点
        unlinkCancelledWaiters();
        //更新局部变量t为最新的队尾引用，因为上面unlinkCancelledWaiters可能会更改lastWaiter引用。
        t = lastWaiter;
    }
    //将当前线程封装成node 状态为CONDITION(-2)
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    //条件成立，条件队列中没有任何元素，当前元素是第一个进入条件队列的元素
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    //更新队尾节点，指向当前node
    lastWaiter = node;
    //返回当前线程的node
    return node;
}
```

##### fullyRelease方法

```java
final int fullyRelease(Node node) {
    //完全释放锁成功标志 failed失败时，表示当前线程未持有锁，用调用了await方法（错误写法）
    //假设失败，在finally代码块中，会将刚刚插入条件队列的node状态修改为取消状态
    //后继节点会在执行清理操作的过程中，将取消状态的节点清理出去
    boolean failed = true;
    try {
        //获取当前线程索持有的state值
        int savedState = getState();
        //绝大多数是成功的
        if (release(savedState)) {
            failed = false;
            
           	//返回当前线程的savedState
            //为什么要返回savedState？
            //因为当你被迁移到阻塞队列后，再次被唤醒，且当前node在阻塞队列中是head.next时
            //此时lock状态是state = 0；当前node重新获得锁，此时需要将state设为savedState
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
     	//修改失败将当前线程设为取消状态
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

```java
final boolean isOnSyncQueue(Node node) {
    //条件一：true 当前node一定在条件队列中，因为signal方法在node迁移到阻塞队列前会将状态设为0
    //node.waitStatus == 0 (表示当前节点已经被signal了)
    //node.waitStatus == 1 （当前线程是未持有锁调用await方法..最终会将node的状态修改为 取消状态..）
    //为什么waitSignal == 0了还要判断 node.prev == null
    //因为signal是先修改状态再迁移
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    //执行到这里有哪些情况？
    //waitStatus = 0 && node.prev != null 因为waitStatus处于取消状态时，signal不会迁移node节点的
    //设置prev引用的逻辑 是 迁移 阻塞队列 逻辑的设置的（enq()）
    //入队的逻辑：1设置node.prev = tail 2.cas更改tail 成功了才算进入阻塞队列3 pred.next = node;
    //可以推算出，就算prev不是null，也不能说明当前node已经进入阻塞队列了
    
    //条件成立，当前节点后面都有值了  肯定进入阻塞队列了
    if (node.next != null) // If has successor, it must be on queue
        return true;
    //执行到这，waitStatus = 0 && node.prev != null 
    //findNodeFromTail从阻塞队列的尾巴向前遍历寻找node，如果查找到返回true，否则返回false
    //false原因可能是当前node在迁移过程中，还未完成
    return findNodeFromTail(node);
}
```

```java
private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }
```

##### signal方法

```java
public final void signal() {
    //判断当前线程是否持有锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //获取条件队列的第一个节点
    Node first = firstWaiter;
    ////第一个节点不为null，则将第一个节点 进行迁移到 阻塞队列的逻辑..
    if (first != null)
        doSignal(first);
}
```

```java
private void doSignal(Node first) {
    do {
    	//更新条件队列的第一个节点，如果条件队列只有一个元素，将尾结点也设为null
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        //断开与下一个节点的联系
        first.nextWaiter = null;
        //条件一：true 转移节点成功，false迁移失败
       	//条件二：迁移失败，将first节点改为first.next节点继续尝试迁移，直至成功，或者条件队列为null为止
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

```java
final boolean transferForSignal(Node node) {
	//cas修改当前节点状态，修改为0，表示当前节点要进入阻塞队列了
    //true：当前节点状态修改成功
    //false: 原因 1 消状态 （线程await时 未持有锁，最终线程对应的node会设置为 取消状态）
    //2node对应的线程 挂起期间被其它线程使用 中断信号 唤醒过...（就会主动进入到 阻塞队列，这时也会修改状态为0）
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

   //enq最终会将当前 node 入队到 阻塞队列，p 是当前节点在阻塞队列的 前驱节点.
    Node p = enq(node);
    int ws = p.waitStatus;
    //ws > 0 说明前驱节点状态为取消状态，直接唤醒当前节点
    //compareAndSetWaitStatus(p, ws, Node.SIGNAL) 返回true 表示设置前驱节点状态为 SIGNAl状态成功
    //compareAndSetWaitStatus(p, ws, Node.SIGNAL) 返回false  ===> 什么时候会false?
     //当前驱node对应的线程是lockInterrupt入队的node时，是会响应中断的，外部线程给前驱线程中断信号之后，前驱node会将状态修改为 取消状态，并且执行 出队逻辑..
    
    //前驱节点只要不是0或-1	就唤醒线程
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

##### 中断唤醒情况

```java
private int checkInterruptWhileWaiting(Node node) {
    	//Thread.interrupted() 返回当前线程中断标记位，并且重置当前标记位 为 false 。
            return Thread.interrupted() ?
                //判断中断位置是在条件队列中中断，还是signal过程中中断
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }
```

```java
final boolean transferAfterCancelledWait(Node node) {
    //条件成立，说明当前node一定在条件队列中被唤醒，因为迁移到阻塞队列会将节点状态改为0
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        //中断唤醒的node也会被加入阻塞队列中
        enq(node);
        //表示在条件队列内中断的
        return true;
    }
   //1.当前node已经被外部线程调用 signal 方法将其迁移到 阻塞队列内了。
   //2.当前node正在被外部线程调用 signal 方法将其迁移至 阻塞队列中 进行中状态..
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

```java
 private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            //条件成立：说明在条件队列内发生过中断，此时await方法抛出中断异常
            if (interruptMode == THROW_IE)
                throw new InterruptedException();

            //条件成立：说明在条件队列外发生的中断，此时设置当前线程的中断标记位 为true
            //中断处理 交给 你的业务处理。 如果你不处理，那什么事 也不会发生了...
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }
```