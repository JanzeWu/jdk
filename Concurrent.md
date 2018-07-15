
# 并发
达到某个状态后，等待线程再继续往下执行。可以用闭锁（Latch）或者栅栏（CyclicBarrier）

其中闭锁使用共享锁， 栅栏使用独占锁 递减计数器。

## 闭锁 CountDownLatch
使用共享锁机制对内部计数器原子操作(compareAndSet)减到0，唤醒等待线程。


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
调用CountDownLatch.await()的线程，尝试获取共享锁，如果成功获取返回当前线程继续执行，否则将当前线程加入等待队列，检查等待队列头结点是否可以获取共享锁，如果可以则唤醒头结点对应的线程，否则阻塞当前线程LockSupport.park()，直到被唤醒。

调用CountDownLatch.countDown()的线程，递减CountDownLatch实例的计数器，尝试释放共享锁（检查是否countDown到0），如果尝试成功，则释放共享锁，从线程等待队列第一个线程开始依次唤醒LockSupport.unpark()所有线程，自身线程继续执行。如果countDown()结果大于0，countDown线程继续执行。

说明：调用CountDownLatch.countDown()不会阻塞当前线程，countDown()结果为0的线程负责唤醒等待队列中的所有线程。await()等待线程在加入等待队列前，先检查是否可以获取共享锁，如果可以则继续执行后续代码。防止await()前，CountDownLatch.countDown()已经为0，而使await线程无限等待。


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
        for (;;) {
            final Node p = node.predecessor();
            //检查线程等待队列头结点是否可以获取共享锁
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            // 在必要的情况下阻塞当前线程
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

/**
 * Wakes up node's successor, if one exists.
 *
 * @param node the node
 */
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
/**
 * “共享锁”可用的条件，就是“锁计数器”的值为0！
 */
protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
}

```


## 栅栏 CyslicBarrier

内部计数器加独占锁（ReentrantLock）减到0，唤醒所有线程。

计数器大于0时，ReentrantLock.newCondition().await()

计数器等于0时，ReentrantLock.newCondition().signalAll()


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

### 数据结构

计数器 + 线程等待队列

### 实现原理

调用await 的线程获取独占锁后将计数器减1，减到0时唤醒等待队列中的线程（打开栅栏，等待队列中所有线程并发执行）。

```java

    /**
     * Main barrier code, covering the various policies.
     */
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        // 阻塞线程
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }


    /**
     * Sets current barrier generation as broken and wakes up everyone.
     * Called only while holding lock.
     */
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        // 唤醒所有线程
        trip.signalAll();
    }

```
