
# 并发
达到某个状态后，等待线程再继续往下执行。可以用闭锁（Latch）或者栅栏（CyclicBarrier）

## 闭锁 CountDownLatch
内部计数器原子操作(compareAndSet)减到0，唤醒等待线程。


### 数据结构
计数器 + 线程等待双向队列

### 使用场景
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
            // 阻塞线程，直到countDown = 0
            downLatch.await();
            System.out.println(Thread.currentThread() + " I am done!");
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

### 实现原理
调用CountDownLatch.await()的线程，并尝试获取共享锁，如果成功获取返回当前线程继续执行，否则将当前线程加入等待队列，检查等待队列头结点是否可以获取共享锁，如果可以则唤醒头结点对应的线程，否则阻塞当前线程LockSupport.park()，直到被唤醒。

调用CountDownLatch.countDown()的线程，递减CountDownLatch实例的计数器，尝试释放共享锁（检查是否countDown到0），如果尝试成功，则释放共享锁，从线程等待队列第一个线程开始依次唤醒LockSupport.unpark()所有线程，自身线程继续执行。如果尝试释放线程失败，自身线程继续执行。

说明：调用CountDownLatch.countDown()不会阻塞当前线程，countDown()结果为0的线程负责唤醒等待队列中的所有线程。await()等待线程在加入等待队列前，先检查是否可以获取共享锁，如果可以则继续执行后续代码。防止await()前，CountDownLatch.countDown()已经为0，而使await线程无限等待。




### 释放
```java
/**
 * await() 实际调用AQS中的acquireSharedInterruptibly方法
 */
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

/**
 * 尝试获取共享锁
 */
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();

    // 尝试获取共享锁，获取失败，调用doAcquireSharedInterruptibly方法。
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}


/**
 * Acquires in shared interruptible mode.
 * @param arg the acquire argument
 */
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 将调用当前线程（调用了await()）加入等待队列中。
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        //检查线程等待队列头结点是否可以获取共享锁
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}


/**
 * Sets head of queue, and checks if successor may be waiting
 * in shared mode, if so propagating if either propagate > 0 or
 * PROPAGATE status was set.
 *
 * @param node the node
 * @param propagate the return value from a tryAcquireShared
 */
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}


/**
 * Try to release shared lock.
 */
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}


/**
 * Release action for shared mode -- signals successor and ensures
 * propagation. (Note: For exclusive mode, release just amounts
 * to calling unparkSuccessor of head if it needs signal.)
 */
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```


## 栅栏 CyslicBarrier
内部计数器加独占锁（ReentrantLock）减到0，唤醒所有线程。
计数器大于0时，lock.newCondition().await()
计数器等于0时，lock.newCondition().signalAll()


### 使用场景
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
