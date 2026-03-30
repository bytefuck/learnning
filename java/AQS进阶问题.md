# AQS 进阶问题

## 1. AQS 的同步队列是如何实现的？为什么用双向链表？

### 同步队列实现原理

AQS 的同步队列是一个 **FIFO 双向链表**，主要包含以下部分：

```java
public abstract class AbstractQueuedSynchronizer {
    // 队列头节点（延迟初始化）
    private transient volatile Node head;
    
    // 队列尾节点（延迟初始化）
    private transient volatile Node tail;
    
    // 同步状态
    private volatile int state;
}
```

### 节点结构

```java
static final class Node {
    // 共享模式标记
    static final Node SHARED = new Node();
    
    // 独占模式标记
    static final Node EXCLUSIVE = null;
    
    // waitStatus 状态值
    static final int CANCELLED =  1;  // 取消
    static final int SIGNAL    = -1;  // 需要唤醒后继
    static final int CONDITION = -2;  // 条件队列
    static final int PROPAGATE = -3;  // 传播唤醒
    
    volatile int waitStatus;
    volatile Node prev;
    volatile Node next;
    volatile Thread thread;
    Node nextWaiter;
}
```

### 队列结构图

```
head (dummy node)          tail
  |                           |
  v                           v
+------+------+    +------+------+    +------+
| prev | next |--->| prev | next |--->| prev |
| null |    |<---|      |    |<---|      |
| ws=0 |    |    | ws=-1|    |    | ws=0 |
|thread=null|    |thread=t1 |    |thread=t2|
+------+------    +------+------+    +------+
  头节点(空)         等待节点1         等待节点2
```

### 为什么用双向链表？

| 原因 | 说明 |
|------|------|
| **快速找到前驱** | 判断是否阻塞需要检查前驱节点的 waitStatus |
| **支持取消操作** | 节点取消时，需要修改前驱的 next 指向其后继 |
| **提高效率** | O(1) 时间复杂度找到前驱和后继 |
| **简化唤醒逻辑** | 唤醒后继时可以快速找到前驱 |

### 入队操作

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) {
            // 队列为空，初始化头节点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            // CAS 设置尾节点
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

### 出队操作

```java
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

---

## 2. 独占模式和共享模式的区别是什么？分别适用于什么场景？

### 独占模式（Exclusive Mode）

**特点**：同一时刻只能有一个线程获取同步状态

**适用场景**：
- ReentrantLock
- ReentrantReadWriteLock 的写锁

**核心方法**：
```java
// 尝试获取独占锁
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}

// 尝试释放独占锁
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
```

**获取流程**：
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && 
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

### 共享模式（Shared Mode）

**特点**：同一时刻可以有多个线程获取同步状态

**适用场景**：
- Semaphore
- CountDownLatch
- ReentrantReadWriteLock 的读锁

**核心方法**：
```java
// 尝试获取共享锁
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}

// 尝试释放共享锁
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}
```

**返回值含义**：
| 返回值 | 含义 |
|--------|------|
| 负数 | 获取失败 |
| 0 | 获取成功，但后续共享获取不会成功 |
| 正数 | 获取成功，后续共享获取可能成功 |

**获取流程**：
```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null;
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

### 对比总结

| 特性 | 独占模式 | 共享模式 |
|------|----------|----------|
| 同时获取线程数 | 1个 | 多个 |
| tryAcquire 返回值 | boolean | int |
| 典型应用 | ReentrantLock | Semaphore, CountDownLatch |
| 唤醒方式 | 只唤醒一个后继 | 可能唤醒多个后继（传播） |
| 节点标记 | EXCLUSIVE (null) | SHARED |

---

## 3. tryAcquire、tryRelease 方法为什么设计成需要子类重写？

### 设计原因

**1. 模板方法模式**

AQS 作为模板类，定义了锁获取/释放的骨架流程，具体如何获取/释放由子类决定：

```java
// AQS 中的模板方法
public final void acquire(int arg) {
    // 1. 尝试获取（子类实现）
    if (!tryAcquire(arg) && 
        // 2. 入队等待
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 3. 处理中断
        selfInterrupt();
}
```

**2. 默认抛出异常**

```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}

protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}

protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}

protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}
```

**3. 需要子类实现的方法**

| 方法 | 说明 | 适用模式 |
|------|------|----------|
| tryAcquire | 尝试获取独占锁 | 独占 |
| tryRelease | 尝试释放独占锁 | 独占 |
| tryAcquireShared | 尝试获取共享锁 | 共享 |
| tryReleaseShared | 尝试释放共享锁 | 共享 |
| isHeldExclusively | 是否被当前线程独占 | 条件变量 |

### 不同子类的实现示例

**ReentrantLock 的实现**：
```java
static final class NonfairSync extends Sync {
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // CAS 获取锁
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        } else if (current == getExclusiveOwnerThread()) {
            // 重入
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

**Semaphore 的实现**：
```java
static final class NonfairSync extends Sync {
    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
    
    final int nonfairTryAcquireShared(int acquires) {
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```

**CountDownLatch 的实现**：
```java
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

protected boolean tryReleaseShared(int releases) {
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c - 1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

### 这样设计的好处

1. **灵活性**：不同的同步器可以定义自己的获取/释放逻辑
2. **可扩展性**：新的同步器只需继承 AQS 实现几个方法
3. **代码复用**：队列管理、线程阻塞等通用逻辑由 AQS 提供
4. **安全性**：防止用户误调用，明确抛出异常提示需要重写