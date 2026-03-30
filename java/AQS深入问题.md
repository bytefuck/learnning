# AQS 深入问题

## 1. AQS 中的公平锁和非公平锁在实现上有什么区别？

### 公平锁与非公平锁对比

| 特性 | 公平锁 | 非公平锁 |
|------|--------|----------|
| 获取顺序 | 严格按照 FIFO 队列顺序 | 允许插队 |
| 吞吐量 | 较低 | 较高 |
| 公平性 | 高 | 低 |
| 饥饿问题 | 无 | 可能存在 |

### 源码对比

**公平锁 tryAcquire 实现**：
```java
static final class FairSync extends Sync {
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 关键区别：检查是否有前驱节点
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        } else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

**非公平锁 tryAcquire 实现**：
```java
static final class NonfairSync extends Sync {
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c ==  0) {
        // 直接 CAS 抢占，不管队列
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### 关键方法：hasQueuedPredecessors

```java
public final boolean hasQueuedPredecessors() {
    Node t = tail;
    Node h = head;
    Node s;
    // 队列不为空，且头节点的后继不是当前线程
    return h != t &&
           ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

### 流程对比图

**公平锁获取流程**：
```
线程尝试获取锁
    |
    v
检查 state == 0?
    |
    +---是---> 检查队列是否有等待节点?
    |                |
    |                +---有---> 加入队列等待
    |                |
    |                +---无---> CAS 获取锁
    |                              |
    v                              v
检查是否可重入            获取成功/失败
    |
    +---是---> state++ 重入
    |
    v
返回结果
```

**非公平锁获取流程**：
```
线程尝试获取锁
    |
    v
检查 state == 0?
    |
    +---是---> 直接 CAS 抢占（不检查队列）
    |
    v
检查是否可重入
    |
    v
返回结果
```

### 非公平锁的优势

1. **减少线程切换**：刚释放锁的线程可能再次获取，避免唤醒开销
2. **提高吞吐量**：减少队列操作和线程上下文切换
3. **实际场景**：大多数情况下，锁持有时间短，非公平锁性能更好

---

## 2. ConditionObject 是如何实现 await/signal 的？与 Object.wait/notify 有什么区别？

### ConditionObject 基本结构

```java
public class ConditionObject implements Condition {
    // 条件队列的头节点
    private transient Node firstWaiter;
    
    // 条件队列的尾节点
    private transient Node lastWaiter;
}
```

### await() 实现

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    
    // 1. 将当前线程加入条件队列
    Node node = addConditionWaiter();
    
    // 2. 释放锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    
    // 3. 检查是否在同步队列中（不在则阻塞）
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    
    // 4. 被唤醒后，重新获取锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

### signal() 实现

```java
public final void signal() {
    // 1. 检查是否持有锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    
    Node first = firstWaiter;
    if (first != null)
        // 2. 唤醒条件队列中的第一个节点
        doSignal(first);
}

private void doSignal(Node first) {
    do {
        if ((firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}

final boolean transferForSignal(Node node) {
    // 1. 修改 waitStatus
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    
    // 2. 将节点从条件队列移到同步队列
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    
    return true;
}
```

### 双队列机制

```
                    ConditionObject 条件队列
                    +--------+    +--------+
                    | Node1  |--->| Node2  |
                    | waitSt.|    | waitSt.|
                    | =-2    |    | =-2    |
                    +--------+    +--------+
                         |              |
    signal() 后移动      |              |
         v              v              v
    
    AQS 同步队列
    +--------+    +--------+    +--------+
    | Head   |--->| Node   |--->| Tail   |
    | waitSt.|    | waitSt.|    | waitSt.|
    | =-1    |    | =0     |    | =0     |
    +--------+    +--------+    +--------+
```

### 与 Object.wait/notify 的区别

| 特性 | Condition.await/signal | Object.wait/notify |
|------|------------------------|---------------------|
| 所属 | Condition 接口 | Object 类 |
| 配合 | Lock 锁 | synchronized |
| 队列数 | 一个 Lock 可创建多个 Condition | 一个对象只有一个等待队列 |
| 精确唤醒 | 可以唤醒指定条件的线程 | 随机唤醒或全部唤醒 |
| 灵活性 | 更灵活 | 较简单 |
| 中断响应 | 支持可中断和不可中断 | 只支持可中断 |
| 超时 | 支持多种超时方式 | 只支持一种超时 |

### 使用示例

```java
class BoundedBuffer<T> {
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    private final Object[] items;
    private int putIdx, takeIdx, count;
    
    public void put(T x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await();  // 队列满，等待
            items[putIdx] = x;
            if (++putIdx == items.length) putIdx = 0;
            count++;
            notEmpty.signal();  // 唤醒消费者
        } finally {
            lock.unlock();
        }
    }
    
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await();  // 队列空，等待
            Object x = items[takeIdx];
            if (++takeIdx == items.length) takeIdx = 0;
            count--;
            notFull.signal();  // 唤醒生产者
            return (T) x;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 3. AQS 如何处理可中断的锁获取？

### 中断相关的方法

| 方法 | 说明 |
|------|------|
| acquire() | 不响应中断 |
| acquireInterruptibly() | 响应中断，抛出异常 |
| tryAcquireNanos() | 响应中断 + 超时 |

### acquire() - 不响应中断

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && 
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();  // 只是记录中断状态，不抛异常
}

final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null;
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;  // 记录中断，但继续等待
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();  // 清除中断标志并返回
}
```

### acquireInterruptibly() - 响应中断

```java
public final void acquireInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();  // 入口检查中断
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}

private void doAcquireInterruptibly(int arg) throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null;
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();  // 直接抛出异常
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### tryAcquireNanos() - 响应中断 + 超时

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}

private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null;
                failed = false;
                return true;
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;  // 超时返回 false
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);  // 带超时的阻塞
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### 三种方式对比

| 方式 | 中断处理 | 超时 | 返回值 |
|------|----------|------|--------|
| acquire() | 记录中断状态，继续等待 | 无 | void |
| acquireInterruptibly() | 抛出 InterruptedException | 无 | void |
| tryAcquireNanos() | 抛出 InterruptedException | 有 | boolean |

### 中断后的处理流程

```
线程被中断
    |
    v
parkAndCheckInterrupt() 返回 true
    |
    +-- acquire() -- 记录中断标志，继续等待，获取锁后 selfInterrupt()
    |
    +-- acquireInterruptibly() -- 抛出 InterruptedException
    |
    +-- tryAcquireNanos() -- 抛出 InterruptedException
```

### 节点取消处理

当线程被中断或超时，需要将节点从队列中移除：

```java
private void cancelAcquire(Node node) {
    if (node == null)
        return;
    
    node.thread = null;
    
    // 跳过已取消的前驱节点
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    
    Node predNext = pred.next;
    
    node.waitStatus = Node.CANCELLED;  // 标记为取消
    
    // 如果是尾节点，直接移除
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // 否则，唤醒后继节点
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }
        node.next = node;  // help GC
    }
}
```