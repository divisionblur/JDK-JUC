### ��˺CyclicBrrier

```java
package java.util.concurrent;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;


public class CyclicBarrier {

    private static class Generation {
        //��ʾ��ǰ�������Ƿ񱻴��ƣ���������ƣ���ô��������һ�����̣߳���ֱ���׳�BrokenException�쳣
        //������һ��������̶߳��ᱻ���ѣ�Ȼ���׳�BrokenException
        boolean broken = false;
    }

    //��ΪcyclicBarrier��������Condition�������еģ�condition������������lock����ʹ��
    private final ReentrantLock lock = new ReentrantLock();
    // �̹߳���ʵ��ʹ�õ���������  ����:��ǰ�����е��̵߳�λ������������е��̲߳Żᱻ����
    private final Condition trip = lock.newCondition();
    // Barrier ��Ҫ����������߳�����
    private final int parties;
  	//��ǰ�����һ���������߳���Ҫִ�е�ʱ��
    private final Runnable barrierCommand;
    //��ʾbarrier����ǰ������
    private Generation generation = new Generation();

	//��ǰ�����ж��ٸ��߳�δ��λ ��ʼֵΪparties	
    private int count;

 	//�����µ�һ������������һ�������̵߳�λ�󣬼��裨barrierCommand ��Ϊ�գ�����Ҫ�߳�ִ�����¼��������nextGeneration()������һ��
    private void nextGeneration() {
       	//���������й�����߳�ȫ������
        trip.signalAll();
		//����countΪparties
        count = parties;
        //�����µ�һ�����µ�һ������һ��û���κι�ϵ
        generation = new Generation();
    }

   //���Ƶ�ǰ�̵߳����ϣ�	�������ڵ��̶߳����׳��쳣
    private void breakBarrier() {
        //�����е�broken����Ϊtrue����ʾ��һ���Ǵ��Ƶģ���������һ�����߳�ֱ���׳��쳣��
        generation.broken = true;
        //����countΪparties
        count = parties;
        //���������еȴ����߳�ȫ�����ѣ����Ѻ���̻߳��鵱ǰ���Ƿ��Ǵ��Ƶģ�����Ǵ��Ƶģ����������߼��Ϳ�����һ�����ѵ��߼���һ��
        trip.signalAll();
    }

    //timed����ʾ��ǰ����await�������߳��Ƿ�ָ���� ��ʱʱ�������true ��ʾ �߳�����Ӧ��ʱ��
    //nanos���̵߳ȴ���ʱʱ�����룬���timed == false ,nanos == 0
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        //��ȡbarrierȫ��������
        final ReentrantLock lock = this.lock;
        //����
        //ΪʲôҪ�����أ�
        //��Ϊ barrier�Ĺ��� �� ���� ����������� condition��         
        lock.lock();
        try {
            //��ȡ��ǰ��
            final Generation g = generation;
		   //�����ǰ�����Ѿ�������״̬����ǰ����await�������̣߳�ֱ���׳�Broken�쳣
            if (g.broken)
                throw new BrokenBarrierException();
		   //�����ǰ�̵߳��жϱ��λ Ϊ true������Ƶ�ǰ����Ȼ��ǰ�߳��׳� �ж��쳣
            if (Thread.interrupted()) {
                //1.���õ�ǰ����״̬Ϊbroken״̬  2.������trip ���������ڵ��߳�
                breakBarrier();
                throw new InterruptedException();
            }
            
            
		    //ִ�е����˵�� ��ǰ�߳��ж�״̬�������� false�� ��ǰ����brokenΪ false��δ����״̬��
            //�����߼�...

            //���� parties ������ 5����ôindex��Ӧ��ֵΪ 4,3,2,1,0
            int index = --count;
            //����������˵����ǰ�߳������һ������barrier���̣߳���ʱ��Ҫ��ʲô�أ�
            if (index == 0) {  // tripped
                //��ǣ�true��ʾ ���һ���߳� ִ��cmdʱδ���쳣��  false����ʾ���һ���߳�ִ��cmdʱ�׳��쳣��.
                //cmd���Ǵ��� barrier����ʱ ָ���ĵڶ��� Runnable�ӿ�ʵ�֣��������Ϊnull
                boolean ranAction = false;
                try {


                    final Runnable command = barrierCommand;
                    //����������˵������barrier����ʱ ָ�� Runnable�ӿ��ˣ����ʱ�����һ��������߳� ����Ҫִ������ӿ�
                    if (command != null)
                        command.run();

                    //command.run()δ�׳��쳣�Ļ�����ô�̻߳�ִ�е����
                    ranAction = true;

                    //�����µ�һ��
                    //1.����trip���������ڹ�����̣߳������ѵ��߳� ������ ��ȡ��lock��Ȼ�������˳�await������
                    //2.����count Ϊ parties
                    //3.����һ���µ�generation���󣬱�ʾ�µ�һ��
                    nextGeneration();
                    //����0����Ϊ��ǰ�߳��Ǵ� �� ���һ��������̣߳�����Index == 0
                    return 0;
                } finally {
                    if (!ranAction)
                        //���command.run()ִ���׳��쳣�Ļ�������뵽���
                        breakBarrier();
                }
            }

            //ִ�е����˵����ǰ�߳� ���������һ������Barrier���߳�..��ʱ��Ҫ����һ��������.

            // loop until tripped, broken, interrupted, or timed out
            //������һֱ�� �������㡢��ǰ�������ơ��̱߳��жϣ��ȴ���ʱ
            for (;;) {
                try {
                    //����������˵����ǰ�߳��ǲ�ָ����ʱʱ���
                    if (!timed)
                        //��ǰ�߳� �� �ͷŵ�lock��Ȼ����뵽trip�������е�β����Ȼ������Լ����ȴ������ѡ�
                        trip.await();
                    else if (nanos > 0L)
                        //˵����ǰ�̵߳���await����ʱ ��ָ���� ��ʱʱ��ģ�
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    //�׳��ж��쳣����������
                    //ʲôʱ����׳�InterruptedException�쳣�أ�
                    //Node�ڵ��� ���������� ʱ �յ��ж��ź�ʱ ���׳��ж��쳣��


                    //����һ��g == generation ������˵����ǰ����û�б仯��
                    //��������! g.broken ��ǰ�����û�б����ƣ���ô��ǰ�߳̾�ȥ���ƣ������׳��쳣..
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        //ִ�е�else�м��������
                        //1.�������˱仯�����ʱ��Ͳ���Ҫ�׳��ж��쳣�ˣ���Ϊ ���Ѿ������ˣ����﻽�Ѻ���������߼���..ֻ���������� �жϱ�ǡ�
                        //2.��û�з����仯�����Ǵ��������ˣ���ʱҲ���÷����ж��쳣��ִ�е������ʱ����׳�  brokenBarrier�쳣��Ҳ��¼���жϱ��λ��
                        Thread.currentThread().interrupt();
                    }
                }

                //���Ѻ�ִ�е�����м��������
                //1.�����������ǰbarrier�������µ�һ����trip.signalAll()��
                //2.��ǰGeneration�����ƣ���ʱҲ�ỽ��������trip�Ϲ�����߳�
                //3.��ǰ�߳�trip�еȴ���ʱ��Ȼ������ת�Ƶ� �������� Ȼ���ȡ���� ���ѡ�

                //������������ǰ���Ѿ�������
                if (g.broken)
                    //�̻߳��Ѻ������׳�BrokenBarrier�쳣��
                    throw new BrokenBarrierException();

                //���Ѻ�ִ�е�����м��������
                //1.�����������ǰbarrier�������µ�һ����trip.signalAll()��
                //3.��ǰ�߳�trip�еȴ���ʱ��Ȼ������ת�Ƶ� �������� Ȼ���ȡ���� ���ѡ�

                //����������˵����ǰ�̹߳����ڼ䣬���һ���̵߳�λ�ˣ�Ȼ�󴥷��˿����µ�һ�����߼�����ʱ����trip���������ڵ��̡߳�
                if (g != generation)
                    //���ص�ǰ�̵߳�index��
                    return index;

                //���Ѻ�ִ�е�����м��������
                //3.��ǰ�߳�trip�еȴ���ʱ��Ȼ������ת�Ƶ� �������� Ȼ���ȡ���� ���ѡ�

                if (timed && nanos <= 0L) {
                    //����barrier
                    breakBarrier();
                    //�׳���ʱ�쳣.
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

#### ���췽��

```java
 // parties��Barrier��Ҫ������߳�������ÿ��������Ҫ������߳���
 // barrierAction����ǰ���������һ����λ���̣߳���Ҫִ�е��¼�������Ϊnull��
public CyclicBarrier(int parties, Runnable barrierAction) {
        //��ΪС�ڵ���0 ��barrierû���κ�����..
        if (parties <= 0) throw new IllegalArgumentException();

        this.parties = parties;
        //count�ĳ�ʼֵ ����parties�����浱ǰ��ÿ��λһ���̣߳�count--
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
```