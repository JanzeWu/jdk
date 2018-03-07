
并发
====
当所有线程达到某个状态时，同时执行。可以用闭锁（Latch）或者栅栏（CyclicBarrier）

闭锁 CountDownLatch
===================
内部计数器原子操作(compareAndSet)减到0，唤醒所有线程。


栅栏 CyslicBarrier
==================
内部计数器加锁（ReentrantLock）减到0，唤醒所有线程。
计数器大于0时，lock.newCondition().await()
计数器等于0时，lock.newCondition().signalAll()

