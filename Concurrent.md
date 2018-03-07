
并发
====
当所有线程达到某个状态时，同时执行。可以用闭锁（Latch）或者栅栏（CyclicBarrier）

闭锁 CountDownLatch
===================
内部计数器原子操作(compareAndSet)减到0，唤醒所有线程。


```java

/**
 * 闭锁CountDownLatch在多线程同步并发执行中使用。
 * 内部计数器在原子操作中减到0，所有线程并发执行。
 * Created by janze on 3/7/18.
 */
public class TestCountDownLatch {

    public static void main(String[] args) throws Exception{
        CountDownLatch downLatch = new CountDownLatch(3);
        long count = downLatch.getCount();
        List<Thread> ts = new ArrayList<>();
        for(int i = 0; i < 3; i++){
            Thread t = new Thread(new Player(downLatch));
            ts.add(t);
            t.start();
        }

        for(Thread t : ts){
            t.join();
        }
    }
}

class Player implements Runnable{

    private CountDownLatch downLatch;

    public Player (CountDownLatch downLatch){
        this.downLatch = downLatch;
    }
    @Override
    public void run() {
        try {
            System.out.println(Thread.currentThread() + " I am ready!");
            downLatch.countDown();
            downLatch.await();
            System.out.println(Thread.currentThread() + " I am done!");
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}

```


栅栏 CyslicBarrier
==================
内部计数器加锁（ReentrantLock）减到0，唤醒所有线程。
计数器大于0时，lock.newCondition().await()
计数器等于0时，lock.newCondition().signalAll()


```java

/**
 * 栅栏CyclicBarrier在多线程同步中的使用，
 * 内部计数器加锁（ReentrantLock）减到0所有等待线程并发执行。
 *
 * Created by janze on 3/7/18.
 *
 */
public class TestCyclicBarrier {

    private static CyclicBarrier cyclicBarrier;


    static class Player implements Runnable{
        @Override
        public void run()  {
            try{
                Thread.sleep(1000);
                System.out.println(Thread.currentThread() + " : I am ready!");
                // 并发执行点 wait for all thread ready
                cyclicBarrier.await();
                System.out.println(Thread.currentThread() + " : I am done!");
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }

    public TestCyclicBarrier(CyclicBarrier cyclicBarrier){

        this.cyclicBarrier = cyclicBarrier;
    }

    public static void main(String[] args) {

        CyclicBarrier cyclicBarrier = new CyclicBarrier(3);
        TestCyclicBarrier mcb = new TestCyclicBarrier(cyclicBarrier);
        int count = cyclicBarrier.getParties();
        List<Thread> ts = new ArrayList<>();

        for(; count > 0; --count){
            Thread t = new Thread(new TestCyclicBarrier.Player());
            ts.add(t);
            t.start();
        }

        for(Thread t : ts){
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

}

```
