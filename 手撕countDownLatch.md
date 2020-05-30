### 手撕countDownLatch源码

#### await方法

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    //条件成立：说明当前AQS.state > 0 ，此时将线程入队，然后等待唤醒..
    //条件不成立：AQS.state == 0，此时就不会阻塞线程了..
    //对应业务层面 执行任务的线程已经将latch打破了。然后其他再调用latch.await的线程，就不会在这里阻塞了
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

```
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

```java
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
    //将调用latch.await()方法的线程 包装成node加入到 AQS的阻塞队列当中。
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                 //获取当前线程节点的前驱节点
                final Node p = node.predecessor();
                if (p == head) {
                    //head.next节点就有权利获取 共享锁了..
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                //shouldParkAfterFailedAcquire  会给当前线程找一个好爸爸，最终给爸爸节点设置状态为 signal（-1），返回true
                //parkAndCheckInterrupt 挂起当前节点对应的线程...
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

```java
//设置当前节点为head节点，并且向后传播，依次唤醒
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    //将当前节点设置为新的head节点
    setHead(node);
   //propagate = 1一定成立
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        //获取当前节点的后继节点
        Node s = node.next;
        //条件一： true 当前节点已经是tail了，调用doReleaseShared处理这种情况
        // 条件二： 要求节点为共享模式，latch.await()->addWaiter(Node,SHARED)
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

#### countDown方法

```java
public void countDown() {
    sync.releaseShared(1);
}
```

```java
public final boolean releaseShared(int arg) {
    //条件成立，说明当前调用CountDown方法的线程正好是state-1 = 0 的线程，需要做触发唤醒await的线程
    if (tryReleaseShared(arg)) {
        //调用countDown方法的线程 只有一个线程会进入这个if块里面去调用doReleaseShared方法，唤醒阻塞状态的逻辑
        doReleaseShared();
        return true;
    }
    return false;
}
```

```java
//更新aqs state值，每调用一次state值减一，当 state - 1 = 0时返回true
protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            //条件成立，说明之前线程触发了唤醒操作，这里返回false
            if (c == 0)
                return false;
            int nextc = c-1;
            //cas成功，说明当前线程执行 tryReleaseShared 方法 c-1之前，没有其它线程 修改过 state。
            if (compareAndSetState(c, nextc))
                //nextc == 0 ：true ，说明当前调用 countDown() 方法的线程 就是需要触发 唤醒操作的线程.
                return nextc == 0;
        }
    }
}
```

```java
// 1.latch.countDown() -> AQS.state == 0 -> doReleaseShared() 唤醒当前阻塞队列内的 head.next 对应的线程。
// 2.被唤醒的线程 -> doAcquireSharedInterruptibly parkAndCheckInterrupt() 唤醒 -> setHeadAndPropagate() -> doReleaseShared()
//AQS.doReleaseShared
    private void doReleaseShared() {
        for (;;) {
            //获取当前AQS 内的 头结点
            Node h = head;
            //条件一：h != null 成立，说明阻塞队列不为空..
            //不成立：h == null 什么时候会是这样呢？
            //latch创建出来后，没有任何线程调用过 await() 方法之前，有线程调用latch.countDown()操作 且触发了 唤醒阻塞节点的逻辑..

            //条件二：h != tail 成立，说明当前阻塞队列内，除了head节点以外  还有其他节点。
            //h == tail  -> head 和 tail 指向的是同一个node对象。 什么时候会有这种情况呢？
            //1. 正常唤醒情况下，依次获取到 共享锁，当前线程执行到这里时 （这个线程就是 tail 节点。）
            //2. 第一个调用await()方法的线程 与 调用countDown()且触发唤醒阻塞节点的线程 出现并发了..
            //   因为await()线程是第一个调用 latch.await()的线程，此时队列内什么也没有，它需要补充创建一个Head节点，然后再次自旋时入队
            //   在await()线程入队完成之前，假设当前队列内 只有 刚刚补充创建的空元素 head 。
            //   同期，外部有一个调用countDown()的线程，将state 值从1，修改为0了，那么这个线程需要做 唤醒 阻塞队列内元素的逻辑..
            //   注意：调用await()的线程 因为完全入队完成之后，再次回到上层方法 doAcquireSharedInterruptibly 会进入到自旋中，
            //   获取当前元素的前驱，判断自己是head.next， 所以接下来该线程又会将自己设置为 head，然后该线程就从await()方法返回了...
            if (h != null && h != tail) {
                //执行到if里面，说明当前head 一定有 后继节点!


                int ws = h.waitStatus;
                //当前head状态 为 signal 说明 后继节点并没有被唤醒过呢...
                if (ws == Node.SIGNAL) {
                    //唤醒后继节点前 将head节点的状态改为 0
                    //这里为什么，使用CAS呢？ 回头说...
                    //当doReleaseShared方法 存在多个线程 唤醒 head.next 逻辑时，
                    //CAS 可能会失败...
                    //案例：
                    //t3 线程在if(h == head) 返回false时，t3 会继续自旋. 参与到 唤醒下一个head.next的逻辑..
                    //t3 此时执行到 CAS WaitStatus(h,Node.SIGNAL, 0) 成功.. t4 在t3修改成功之前，也进入到 if (ws == Node.SIGNAL) 里面了，
                    //但是t4 修改 CAS WaitStatus(h,Node.SIGNAL, 0) 会失败，因为 t3 改过了...
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    //唤醒后继节点
                    unparkSuccessor(h);
                }

                else if (ws == 0 &&
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }

            //条件成立：
            //1.说明刚刚唤醒的 后继节点，还没执行到 setHeadAndPropagate方法里面的 设置当前唤醒节点为head的逻辑。
            //这个时候，当前线程 直接跳出去...结束了..
            //此时用不用担心，唤醒逻辑 在这里断掉呢？、
            //不需要担心，因为被唤醒的线程 早晚会执行到doReleaseShared方法。

            //2.h == null  latch创建出来后，没有任何线程调用过 await() 方法之前，
            //有线程调用latch.countDown()操作 且触发了 唤醒阻塞节点的逻辑..
            //3.h == tail  -> head 和 tail 指向的是同一个node对象

            //条件不成立：
            //被唤醒的节点 非常积极，直接将自己设置为了新的head，此时 唤醒它的节点（前驱），执行h == head 条件会不成立..
            //此时 head节点的前驱，不会跳出 doReleaseShared 方法，会继续唤醒 新head 节点的后继...
            if (h == head)                   // loop if head changed
                break;
        }
    }
```