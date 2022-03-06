+++
title="wait() 和 sleep() 简单对比"
tags=["java"]
categories=["Java"]
date="2022-03-06T22:33:07+08:00"

+++

## sleep()

不会放弃任何持有的锁

> Causes the currently executing thread to sleep (temporarily cease execution) for the specified number of milliseconds, subject to the precision and accuracy of system timers and schedulers. The thread does not lose ownership of any monitors.


## wait() 

只有持有对象锁的那个线程才能调用wait() 方法

> The current thread must own this object's monitor.


调用wait() 方法执行的动作

> This method causes the current thread (call it T) to place itself in the wait set for this object and then to relinquish any and all synchronization claims on this object.


调用wait() 的线程有四种方法继续执行

> 1. Some other thread invokes the notify method for this object and thread T happens to be arbitrarily chosen as the thread to be awakened.
> 2. Some other thread invokes the notifyAll method for this object.
> 3. Some other thread interrupts thread T.
> 4. The specified amount of real time has elapsed, more or less. If timeout is zero, however, then real time is not taken into consideration and the thread simply waits until notified.


当wait() 线程恢复执行后

> Once it has gained control of the object, all its synchronization claims on the object are restored to the status quo ante - that is, to the situation as of the time that the wait method was invoked.


当wait() 后的线程被Interrupt时

> If the current thread is interrupted by any thread before or while it is waiting, then an InterruptedException is thrown. This exception is not thrown until the lock status of this object has been restored as described above.


调用wait() 的线程只会释放被调用对象的锁，不会释放持有的其它对象的锁

> Note that the wait method, as it places the current thread into the wait set for this object, unlocks only this object; any other objects on which the current thread may be synchronized remain locked while the thread waits.


## 总结

| |是否放弃持有锁|是否可中断|唤醒方式|
|---|---|---|---|
|sleep()|不放弃任何持有的锁|是|timeout,interrupt|
|wait()|放弃当前对象的锁,不放弃持有的其它对象的锁|是|timeout,interrupt,notify,notifyAll|



## 测试

```Java
package multithread.fundation;

import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class SleepVsNotify {

    @AllArgsConstructor
    static class SleepWaitThread extends Thread {
        private final String name;
        private final long sleepSec;
        private final long waitSec;
        private final Thread target;
        private final String lock;

        @Override
        public void run() {
            log.info("{} started...", name);
            synchronized (lock) {
                try {
                    log.info("{} get lock on {}, start sleep for {}s...", name, lock, sleepSec);
                    Thread.sleep(sleepSec * 1000);
                    log.info("{} wakeup, start wait for {}s", name, waitSec);
                    lock.wait(waitSec * 1000);
                } catch (InterruptedException e) {
                    log.error("{} interrupted while waiting", name);
                    return;
                }

                if (target != null) {
                    target.interrupt();
                }

                log.info("{} finished", name);
            }
        }
    }

    public void testSleepWait() {
        String lock1 = "Lock-1";
        String lock2 = "Lock-2";

        SleepWaitThread thread1 = new SleepWaitThread("SleepWaitThread-1", 5, 5, null, lock1);
        SleepWaitThread thread2 = new SleepWaitThread("SleepWaitThread-2", 2, 2, thread1, lock2);
        SleepWaitThread thread3 = new SleepWaitThread("SleepWaitThread-3", 5, 5,null, lock2);

        thread1.start();
        thread2.start();
        thread3.start();
    }

    public static void main(String[] args) {
        SleepVsNotify sleepVsNotify = new SleepVsNotify();
        sleepVsNotify.testSleepWait();
    }
}
```


输出

```纯文本
22:06:06.617 [Thread-0] INFO multithread.fundation.SleepVsNotify - SleepWaitThread-1 started...
22:06:06.617 [Thread-2] INFO multithread.fundation.SleepVsNotify - SleepWaitThread-3 started...
22:06:06.621 [Thread-0] INFO multithread.fundation.SleepVsNotify - SleepWaitThread-1 get lock on Lock-1, start sleep for 5s...
22:06:06.621 [Thread-2] INFO multithread.fundation.SleepVsNotify - SleepWaitThread-3 get lock on Lock-2, start sleep for 5s...
22:06:06.617 [Thread-1] INFO multithread.fundation.SleepVsNotify - SleepWaitThread-2 started...
22:06:11.626 [Thread-2] INFO multithread.fundation.SleepVsNotify - SleepWaitThread-3 wakeup, start wait for 5s
22:06:11.626 [Thread-0] INFO multithread.fundation.SleepVsNotify - SleepWaitThread-1 wakeup, start wait for 5s
22:06:11.627 [Thread-1] INFO multithread.fundation.SleepVsNotify - SleepWaitThread-2 get lock on Lock-2, start sleep for 2s...
22:06:13.629 [Thread-1] INFO multithread.fundation.SleepVsNotify - SleepWaitThread-2 wakeup, start wait for 2s
22:06:15.632 [Thread-0] ERROR multithread.fundation.SleepVsNotify - SleepWaitThread-1 interrupted while waiting
22:06:15.632 [Thread-1] INFO multithread.fundation.SleepVsNotify - SleepWaitThread-2 finished
22:06:16.631 [Thread-2] INFO multithread.fundation.SleepVsNotify - SleepWaitThread-3 finished

Process finished with exit code 0
```


