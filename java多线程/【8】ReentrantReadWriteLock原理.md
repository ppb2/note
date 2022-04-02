



![image-20220309224656220](https://raw.githubusercontent.com/ppb2/note/main/imgs/image-20220309224656220.png)

### 读写锁

```java
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing
     */
    Lock writeLock();
}
```

## 读写锁的设计

![img](https://raw.githubusercontent.com/ppb2/note/main/imgs/23353704-81afdd40c9908f0c.png)

在读写锁的设计中同样是通过一个state状态来表示读写锁，将32位的int值拆分成两部分：

- 高16位  代表读状态
- 低16位 代表写状态

### 读写锁的同步器

```java
 	    static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** Returns the number of shared holds represented in count  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** Returns the number of exclusive holds represented in count  */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```



### 读锁状态获取

```java
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
```

- 将state的值从高往低右移`SHARED_SHIFT` 即16位，即能获取读锁的状态
- 锁的最大数量`MAX_COUNT      = (1 << SHARED_SHIFT) - 1`

### 写锁状态获取

```java
 static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

- `EXCLUSIVE_MASK` 写锁的掩码 即 0000 0000 0000 0000 1111 1111 1111 1111
- 将state的值做&操作最后高位就被清除，留下低16位的值即为写锁的状态

## 判断读线程是否需要阻塞

### 公平锁实现

```java
 static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }
```

```java
public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

- 公平锁下，写线程竞争会通过`hasQueuedPredecessors`判断同步队列中是否存在节点，如果存在则阻塞等待。
- 公平锁下，读线程竞争会通过`hasQueuedPredecessors`判断同步队列中是否存在节点，如果存在则阻塞等待。

### 非公平锁的实现

```java
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
        final boolean readerShouldBlock() {
            /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
            return apparentlyFirstQueuedIsExclusive();
        }
    }
```

```java
final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
    //1.如果同步队列内头节点不为空，
    //2.头节点的下一个节点s不为空
    //3.s不是共享状态且s的线程不为空则 阻塞
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
```



- 如果是非公平锁，只要有写锁被当前线程竞争永远无需block，`writerShouldBlock` 返回false。谁抢到就是谁的
- 如果是非公平锁，读线程要竞争首先需要` apparentlyFirstQueuedIsExclusive` 判断是否被写锁占用，实现读写互斥

## 读锁实现ReadLock

### 读锁的获取

```java
public void lock() { sync.acquireShared(1);}
```

- ReadLock 调用同步器的acquireShared方法尝试获取读锁

```java
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

- tryAcquireShared尝试获取读锁，获取成功结束
- 获取失败执行doAcquireShared

```java
protected final int tryAcquireShared(int unused) {
           
            Thread current = Thread.currentThread();
            int c = getState();
     	    //1. 读锁高16位代表读锁，判断是否存在写锁，如果存在写锁且独占线程不是当前线程则获取失败
    		//如果写锁存在且当前线程即为独占线程则可以获取读锁-----锁降级
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
    		//  高16位代读锁获取读锁数量
            int r = sharedCount(c);
    		//2. 如果读线程无需阻塞通过cas尝试获取读锁
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                //单独处理第一个读线程的重入计数
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    //读线程的计数器每个线程都存在一个存放到threadLocal中，此计数器记录着每个线程的重入次数
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
    		//3. 尝试自旋获取
            return fullTryAcquireShared(current);
        }
```

- 1.注意上文获取读锁的判断条件
  - 如果写锁是当前线程则可以继续获取读锁 ---锁降级，同一个线程获取独占锁后允许锁降级，后文介绍
  - 如果独占锁非当前线程则说明存在写锁，读写互斥
- 2.读锁是否需要阻塞`readerShouldBlock` 在公平锁与非公平锁已经介绍此情况，此处不赘述
- 3.读锁的数量小于MAX_COUNT，则进行cas尝试增加读锁的获取数量
- 4.cas操作成功则获取读锁，如果是第一个读线程获取读锁，直接记录当前线程，与firstReaderHoldCount重入次数
- 5.如果同时存在多个线程持有读锁则需要 HoldCounter计数器来实现各线程的重入次数

思考：为什么单个线程只需要独立的两个字段记录，而多个读锁则需要计数器呢？



```java
        final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }

```

- 如果前面尝试获取读锁都失败了，则进行自旋锁获取读锁的尝试此处逻辑基本与上文一致
- 如果自旋获取失败则进行下一步操作 `doAcquireShared` 

```java
private void doAcquireShared(int arg) {
    	//竞争失败添加到同步队列
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //如果当前加入同步队列的节点的前一个节点是头节点
                final Node p = node.predecessor();
                if (p == head) {
                    //再次尝试获取读锁
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        //获取成功则设置头节点并且传播
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        //如果当前线程被中断则处理中断异常
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

- 如果竞争失败则调用`addWaiter` 操作添加到同步队列的队尾
- 如果当前线程的前驱节点为头节点则尝试获取锁，如果获取成功则将当前节点设置为头节点，并传播下去，逐个唤醒共享锁节点，并设置为头节点
- 如果获取失败 `shouldParkAfterFailedAcquire` 是否阻塞住等待唤醒
- 如果中断后响应中断后则`cancelAcquire` 设置为Cancel状态

```java
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
            //如果当前node的下一个node为空或者为共享状态则释放共享锁
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```



```java
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
                    //唤醒后继节点
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

- `setHeadAndPropagate` 方法如果node的节点为共享节点则调用`doReleaseShared` 
- `doReleaseShared` 尝试不断释放后继节点，直到head没有发生变化，防止后续没有shared节点一直在循环，浪费cpu资源
- 当后继节点被唤醒后，有会进入`doAcquireShared` 方法尝试获取锁并唤醒下一个shared节点

### 共享锁的传播过程

![image-20220322110736483](https://raw.githubusercontent.com/ppb2/note/main/imgs/image-20220322110736483.png)

1. `setHeadAndPropagate` 与`doReleaseShared` 所做的事情就是以下流程的操作。
2. Thread01为当前线程如果加入后同步队列后，接着有三个读线程也加入了同步队列
3. Thread01发现自己的前继节点为头节点，那么尝试获取读锁，获取成功后将自己设置为头节点，且将此步骤传播下去



![image-20220322110908761](https://raw.githubusercontent.com/ppb2/note/main/imgs/image-20220322110908761.png)

4. 当thread02被唤醒后，发现自己的前继节点为thread01且为头节点，那么会重复前面的动作，继续传播下去，逐步唤醒thread03

![image-20220322110942218](https://raw.githubusercontent.com/ppb2/note/main/imgs/image-20220322110942218.png)

![image-20220322111001960](https://raw.githubusercontent.com/ppb2/note/main/imgs/image-20220322111001960.png)





### 读锁释放

```java
 public void unlock() {
            sync.releaseShared(1);
        }
```

- 调用ReadLock的unlock方法

```java
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

- 调用tryReleaseShared尝试释放锁
- 如果当所有读线程都释放完成则返回true，即不存在读锁，否则只变更重入次数，以及读锁的数量

```java
protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
    		//减少重入次数
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
    		//尝试自旋设置读锁
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
```

- tryReleaseShared首先进行重入次数的减少，

- 自旋尝试将读锁减少一次，如果读锁不为0，说明存在其他读锁，此时还不能唤醒后继节点，即写线程（每一个读线程被唤醒后都会唤醒后继读线程）

- 当所有读锁都释放后，可以唤醒后继节点，做到读写互斥

  

```java
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





## 写锁的实现WriteLock

### 写锁的获取

```java
 public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

- 与独占锁的获取方式相同，通过tryAcquire获取，获取失败则会添加到同步队列，但是稍微有些差别
- 如果前继节点是头节点则会尝试自旋获取，获取失败则阻塞

```java
protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            int c = getState();
    		//获取写锁数量
            int w = exclusiveCount(c);
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                //如果c！=0即存在读锁或者写锁 w == 0即存在读锁 或 当前线程不为独占线程(存在其他写线程)，则获取失败
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //防止写锁超过最大数量
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                //存在写锁且写锁线程为当前线程，支持可重入
                setState(c + acquires);
                return true;
            }
    		//如果c=0即不存在读锁以及写锁，此时如果写锁是非公平实现则尝试cas操作获取锁，否则
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

1. 获取锁时先判断是否存在锁（读锁或写锁）即state是否为0
2. 若存在读锁或者写锁，如果写锁为0即存在读锁，或者当前线程不为独占线程则获取失败（不支持锁的升级）
3. 如果存在写锁且当前线程为独占线程，进行可重入操作
4. 如果state=0即不存在读锁以及写锁。则分公平锁实现与非公平锁实现判断写线程是否需要阻塞，
   - 如果需要阻塞则获取失败。
   - 如果无需阻塞则尝试竞争锁，竞争成功返回true

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

1. 竞争失败后则会执行此方法`acquireQueued`。
2. 如果当前节点的前驱节点为头节点则尝试自旋获取锁
3. 如果前驱节点不为头节点，且竞争获取失败，则失败进入阻塞状态

### 写锁的释放

```java
public void unlock() {
            sync.release(1);
        }
```

- 通过调用unlock进行锁的释放

```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

- release操作调用tryRelease进行释放写锁操作，如果成功唤醒头节点的后继节点
- 失败的返回false

```java
   protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
```

- 写锁释放时，如果写锁数量释放后为0则返回true尝试释放下一个节点
- 写锁如果为多次重入则只做写锁数量的减少，返回false。不唤醒后继节点，因为当前还持有写锁，直到写锁数量为0.



## 锁降级

```java
if (exclusiveCount(c) != 0) {
    if (getExclusiveOwnerThread() != current)
        return -1;
```



锁降级即写锁降级为读锁。锁降级是被支持的，在获取读锁的时候，如果存在写锁，且写线程即当前线程是允许当前线程获取读锁。

原因：可见性，场景如果写线程获取独占锁后，释放写锁后，当前写锁为其他线程占用，对值进行了修改，那么如果当前线程不获取读锁而是直接释放写锁， 假设此刻另一个线程（记作线程T）获取了写锁并修改了数据，那么当前线程无法感知线程T的数据更新。如果当前线程获取读锁，即遵循锁降级的步骤，则线程T将会被阻塞，直到当前线程使用数据并释放读锁之后，线程T才能获取写锁进行数据更新。

## 锁升级

```java
 if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                //如果c！=0即存在读锁或者写锁 w == 0即存在读锁 或 当前线程不为独占线程(存在其他写线程)，则获取失败
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
```



锁升级即读锁升级为写锁。锁升级不被允许，在获取写锁的时候，如果存在读锁，则一定不允许获取写锁。

原因：可见性 

## 





