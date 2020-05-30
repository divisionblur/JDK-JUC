### 手撕CyclicBrrier

```java
package java.util.concurrent;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;


public class CyclicBarrier {

    private static class Generation {
        //表示当前“代”是否被打破，如果被打破，那么再来到这一代的线程，就直接抛出BrokenException异常
        //且在这一代挂起的线程都会被唤醒，然后抛出BrokenException
        boolean broken = false;
    }

    //因为cyclicBarrier是依赖于Condition条件队列的，condition条件必须依赖lock才能使用
    private final ReentrantLock lock = new ReentrantLock();
    // 线程挂起实现使用的条件队列  条件:当前代所有的线程到位，这个条件队列的线程才会被唤醒
    private final Condition trip = lock.newCondition();
    // Barrier 需要参与进来的线程数量
    private final int parties;
  	//当前代最后一个到来的线程需要执行的时间
    private final Runnable barrierCommand;
    //表示barrier对象当前“代”
    private Generation generation = new Generation();

	//当前代还有多少个线程未到位 初始值为parties	
    private int count;

 	//开启新的一“代”，当这一代所有线程到位后，假设（barrierCommand 不为空，还需要线程执行完事件）会调用nextGeneration()开启下一代
    private void nextGeneration() {
       	//将条件队列挂起的线程全部唤醒
        trip.signalAll();
		//重置count为parties
        count = parties;
        //开启新的一代，新的一代和上一代没有任何关系
        generation = new Generation();
    }

   //打破当前线程的屏障，	在屏障内的线程都会抛出异常
    private void breakBarrier() {
        //将代中的broken设置为true，表示这一代是打破的，再来到这一代的线程直接抛出异常。
        generation.broken = true;
        //重置count为parties
        count = parties;
        //将条件队列等待的线程全部唤醒，唤醒后的线程会检查当前代是否是打破的，如果是打破的，接下来的逻辑和开启下一代唤醒的逻辑不一样
        trip.signalAll();
    }

    //timed：表示当前调用await方法的线程是否指定了 超时时长，如果true 表示 线程是响应超时的
    //nanos：线程等待超时时长纳秒，如果timed == false ,nanos == 0
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        //获取barrier全局锁对象
        final ReentrantLock lock = this.lock;
        //加锁
        //为什么要加锁呢？
        //因为 barrier的挂起 和 唤醒 依赖的组件是 condition。         
        lock.lock();
        try {
            //获取当前代
            final Generation g = generation;
		   //如果当前代是已经被打破状态，则当前调用await方法的线程，直接抛出Broken异常
            if (g.broken)
                throw new BrokenBarrierException();
		   //如果当前线程的中断标记位 为 true，则打破当前代，然后当前线程抛出 中断异常
            if (Thread.interrupted()) {
                //1.设置当前代的状态为broken状态  2.唤醒在trip 条件队列内的线程
                breakBarrier();
                throw new InterruptedException();
            }
            
            
		    //执行到这里，说明 当前线程中断状态是正常的 false， 当前代的broken为 false（未打破状态）
            //正常逻辑...

            //假设 parties 给的是 5，那么index对应的值为 4,3,2,1,0
            int index = --count;
            //条件成立：说明当前线程是最后一个到达barrier的线程，此时需要做什么呢？
            if (index == 0) {  // tripped
                //标记：true表示 最后一个线程 执行cmd时未抛异常。  false，表示最后一个线程执行cmd时抛出异常了.
                //cmd就是创建 barrier对象时 指定的第二个 Runnable接口实现，这个可以为null
                boolean ranAction = false;
                try {


                    final Runnable command = barrierCommand;
                    //条件成立：说明创建barrier对象时 指定 Runnable接口了，这个时候最后一个到达的线程 就需要执行这个接口
                    if (command != null)
                        command.run();

                    //command.run()未抛出异常的话，那么线程会执行到这里。
                    ranAction = true;

                    //开启新的一代
                    //1.唤醒trip条件队列内挂起的线程，被唤醒的线程 会依次 获取到lock，然后依次退出await方法。
                    //2.重置count 为 parties
                    //3.创建一个新的generation对象，表示新的一代
                    nextGeneration();
                    //返回0，因为当前线程是此 代 最后一个到达的线程，所以Index == 0
                    return 0;
                } finally {
                    if (!ranAction)
                        //如果command.run()执行抛出异常的话，会进入到这里。
                        breakBarrier();
                }
            }

            //执行到这里，说明当前线程 并不是最后一个到达Barrier的线程..此时需要进入一个自旋中.

            // loop until tripped, broken, interrupted, or timed out
            //自旋，一直到 条件满足、当前代被打破、线程被中断，等待超时
            for (;;) {
                try {
                    //条件成立：说明当前线程是不指定超时时间的
                    if (!timed)
                        //当前线程 会 释放掉lock，然后进入到trip条件队列的尾部，然后挂起自己，等待被唤醒。
                        trip.await();
                    else if (nanos > 0L)
                        //说明当前线程调用await方法时 是指定了 超时时间的！
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    //抛出中断异常，会进来这里。
                    //什么时候会抛出InterruptedException异常呢？
                    //Node节点在 条件队列内 时 收到中断信号时 会抛出中断异常！


                    //条件一：g == generation 成立，说明当前代并没有变化。
                    //条件二：! g.broken 当前代如果没有被打破，那么当前线程就去打破，并且抛出异常..
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        //执行到else有几种情况？
                        //1.代发生了变化，这个时候就不需要抛出中断异常了，因为 代已经更新了，这里唤醒后就走正常逻辑了..只不过设置下 中断标记。
                        //2.代没有发生变化，但是代被打破了，此时也不用返回中断异常，执行到下面的时候会抛出  brokenBarrier异常。也记录下中断标记位。
                        Thread.currentThread().interrupt();
                    }
                }

                //唤醒后，执行到这里，有几种情况？
                //1.正常情况，当前barrier开启了新的一代（trip.signalAll()）
                //2.当前Generation被打破，此时也会唤醒所有在trip上挂起的线程
                //3.当前线程trip中等待超时，然后主动转移到 阻塞队列 然后获取到锁 唤醒。

                //条件成立：当前代已经被打破
                if (g.broken)
                    //线程唤醒后依次抛出BrokenBarrier异常。
                    throw new BrokenBarrierException();

                //唤醒后，执行到这里，有几种情况？
                //1.正常情况，当前barrier开启了新的一代（trip.signalAll()）
                //3.当前线程trip中等待超时，然后主动转移到 阻塞队列 然后获取到锁 唤醒。

                //条件成立：说明当前线程挂起期间，最后一个线程到位了，然后触发了开启新的一代的逻辑，此时唤醒trip条件队列内的线程。
                if (g != generation)
                    //返回当前线程的index。
                    return index;

                //唤醒后，执行到这里，有几种情况？
                //3.当前线程trip中等待超时，然后主动转移到 阻塞队列 然后获取到锁 唤醒。

                if (timed && nanos <= 0L) {
                    //打破barrier
                    breakBarrier();
                    //抛出超时异常.
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }

    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

    public CyclicBarrier(int parties) {
        this(parties, null);
    }

    public int getParties() {
        return parties;
    }

    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }
    public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }
    public boolean isBroken() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return generation.broken;
        } finally {
            lock.unlock();
        }
    }

    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier();   // break the current generation
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }
    public int getNumberWaiting() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return parties - count;
        } finally {
            lock.unlock();
        }
    }
}
```

#### 构造方法

```java
 // parties：Barrier需要参与的线程数量，每次屏障需要参与的线程数
 // barrierAction：当前“代”最后一个到位的线程，需要执行的事件（可以为null）
public CyclicBarrier(int parties, Runnable barrierAction) {
        //因为小于等于0 的barrier没有任何意义..
        if (parties <= 0) throw new IllegalArgumentException();

        this.parties = parties;
        //count的初始值 就是parties，后面当前代每到位一个线程，count--
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
```