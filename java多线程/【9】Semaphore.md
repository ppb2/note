

## Semaphore是什么？

Semaphore(信号量)是java.util.concurrent下的一个工具类.用来控制可同时访问特定资源的线程数.内部是通过维护父类(AQS)的 int state值实现。

## Semaphore常用方法



```java
acquire()  
获取一个令牌，在获取到令牌、或者被其他线程调用中断之前线程一直处于阻塞状态。

acquire(int permits)  
获取一个令牌，在获取到令牌、或者被其他线程调用中断、或超时之前线程一直处于阻塞状态。
    
acquireUninterruptibly() 
获取一个令牌，在获取到令牌之前线程一直处于阻塞状态（忽略中断）。
    
tryAcquire()
尝试获得令牌，返回获取令牌成功或失败，不阻塞线程。

tryAcquire(long timeout, TimeUnit unit)
尝试获得令牌，在超时时间内循环尝试获取，直到尝试获取成功或超时返回，不阻塞线程。

release()
释放一个令牌，唤醒一个获取令牌不成功的阻塞线程。

hasQueuedThreads()
等待队列里是否还存在等待线程。

getQueueLength()
获取等待队列里阻塞的线程数。

drainPermits()
清空令牌把可用令牌数置为0，返回清空令牌的数量。

availablePermits()
返回可用的令牌数量。
```





## Semaphore代码实现

###  **Semaphore初始化**

```java
Semaphore semaphore=new Semaphore(2);
```

1、当调用new Semaphore(2) 方法时，默认会创建一个非公平的锁的同步阻塞队列。

2、把初始令牌数量赋值给同步队列的state状态，state的值就代表当前所剩余的令牌数量。

### 获取令牌

```java
semaphore.acquire();
```



1、当前线程会尝试去同步队列获取一个令牌，获取令牌的过程也就是使用原子的操作去修改同步队列的state ,获取一个令牌则修改为state=state-1。

2、 当计算出来的state<0，则代表令牌数量不足，此时会创建一个Node节点加入阻塞队列，挂起当前线程。

3、当计算出来的state>=0，则代表获取令牌成功。

```java
public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```



- 尝试获取一个令牌

```java
  public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
      //尝试获取共享锁
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

```

- tryAcquireShared尝试获取令牌 arg为令牌数
- 如果获取失败则调用doAcquireSharedInterruptibly方法

```java
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        //添加节点到同步队列
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
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
```

- 获取失败后添加当前节点到同步队列
- 如果当前node的前继节点为head则尝试获取共享锁如果获取成功则唤醒后继节点进行获取令牌
- 如果自旋获取失败则进入阻塞状态



#### tryAcquireShared公平锁实现

```java
    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }
          protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
```

-  super(permits)此方法将state设置为令牌数即permits数
- 调用tryAcquireShared实现公平模式下令牌的获取

```java
  protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

- 自旋获取令牌，同步队列中存在阻塞线程则获取失败，此时进入下一步阻塞或者自旋获取（取决于当前进入同步队列的线程的前继节点是否为头节点）
- 获取可用令牌数
- 计算剩余令牌数
- 如果剩余令牌数小于0阻断，返回值小于0，获取失败
- 如果remaining数大于或等于0则则进行cas操作设置剩余令牌数，设置成功则返回值大于或等于0表示获取成功

#### tryAcquireShared非公平锁实现

```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }
```

-  super(permits)此方法将state设置为令牌数即permits数
- 调用nonfairTryAcquireShared实现非公平模式下令牌的获取

```java
 final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                //获取当前可用令牌数
                int available = getState();
                //计算剩余令牌数
                int remaining = available - acquires;
                //如果剩余令牌数小于0阻断，返回（false）如果remaining数大于或等于0则则进行cas操作设置剩余令牌数
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

- 获取可用令牌数
- 计算剩余令牌数
- 如果剩余令牌数小于0阻断，返回值小于0，获取失败
- 如果remaining数大于或等于0则则进行cas操作设置剩余令牌数，设置成功则返回值大于或等于0表示获取成功

注意：非公平锁上来直接竞争不关心同步队列内是否存在阻塞线程

### 释放令牌

```java
 public void release() {
        sync.releaseShared(1);
    }
```

```java
   public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

- tryReleaseShared尝试释放锁
- 释放成功执行doReleaseShared

```java
protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```

- 释放令牌的过程与获取相反，往state中添加释放数量
- 成功后执行doReleaseShared

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

- 释放同步队列内的后继节点，线程被唤醒后如果同步队列内仍然存在则继续唤醒，直到牌数为0
- 若当令牌数为零，当前线程会继续进入阻塞状态

