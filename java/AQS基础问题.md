# AQS 基础问题

## 1. AQS 的核心设计理念是什么？

**AQS（AbstractQueuedSynchronizer）** 是 Java 并发包的核心基础类，用于构建锁和同步器。

### 核心设计理念

1. **模板方法模式**
   - AQS 定义了同步器的骨架，子类只需实现特定方法
   - 子类继承 AQS 并实现 tryAcquire、tryRelease 等方法

2. **状态同步机制**
   - 使用一个 int 类型的 state 变量表示同步状态
   - 通过 CAS 操作保证 state 的原子性更新

3. **队列等待机制**
   - 没有获取到锁的线程会进入一个 FIFO 的双向队列等待
   - 队列中的节点通过 park/unpark 实现线程的阻塞和唤醒

### AQS 的核心组件

```
                    +------------------+
                    |      state       |  同步状态
                    +------------------+
                           |
                           v
    +------------------+   +------------------+   +------------------+
    |     Node        |<->|      Node        |<->|      Node        |
    |   (等待线程)     |   |    (等待线程)     |   |    (等待线程)     |
    +------------------+   +------------------+   +------------------+
         同步队列（CLH队列变体）
```

### 基于AQS实现的同步器

| 同步器 | 获取锁方式 | 释放锁方式 |
|--------|-----------|-----------|
| ReentrantLock | lock() | unlock() |
| ReentrantReadWriteLock | readLock()/writeLock() | unlock() |
| Semaphore | acquire() | release() |
| CountDownLatch | await() | countDown() |
| CyclicBarrier | await() | - |

---

## 2. AQS 中的 state 变量有什么作用？

state 是 AQS 的核心状态变量，定义如下：

```java
private volatile int state;
```

### state 的作用

1. **表示同步状态**
   - 对于 ReentrantLock：state=0 表示锁空闲，state>0 表示锁被占用
   - 对于 ReentrantReadWriteLock：高16位存读锁计数，低16位存写锁计数
   - 对于 Semaphore：state 表示可用许可数
   - 对于 CountDownLatch：state 表示需要倒计数的次数

2. **通过 CAS 保证原子性**
```java
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

3. **volatile 保证可见性**
   - 一个线程修改 state 后，其他线程能立即看到最新值

### state 的操作方法

| 方法 | 说明 |
|------|------|
| getState() | 获取当前 state 值 |
| setState(int newState) | 设置 state 值 |
| compareAndSetState(int expect, int update) | CAS 更新 state |

### 不同同步器的 state 含义

```java
// ReentrantLock
// state = 0：锁空闲
// state = 1：锁被占用
// state > 1：锁被重入

// ReentrantReadWriteLock
// state 高16位：读锁数量
// state 低16位：写锁数量

// Semaphore
// state：可用信号量数量

// CountDownLatch
// state：倒计数剩余次数
```

---

## 3. AQS 如何实现线程的排队等待？

AQS 使用 **CLH 队列的变体** 实现线程排队等待。

### CLH 队列介绍

CLH（Craig, Landin, and Hagersten）队列是一种基于链表的自旋锁队列。

AQS 对 CLH 队列进行了改进：
- 使用双向链表
- 支持阻塞而非自旋等待
- 支持可中断的锁获取

### 队列节点结构

```java
static final class Node {
    // 等待状态
    volatile int waitStatus;
    
    // 前驱节点
    volatile Node prev;
    
    // 后继节点
    volatile Node next;
    
    // 等待的线程
    volatile Thread thread;
    
    // 条件队列中的后继节点
    Node nextWaiter;
}
```

### waitStatus 状态说明

| 状态值 | 名称 | 说明 |
|--------|------|------|
| 0 | INIT | 初始状态 |
| 1 | CANCELLED | 线程已取消等待 |
| -1 | SIGNAL | 后继节点需要被唤醒 |
| -2 | CONDITION | 节点在条件队列中 |
| -3 | PROPAGATE | 共享模式下传播唤醒 |

### 入队流程

```
线程尝试获取锁失败
        |
        v
创建 Node 节点
        |
        v
CAS 将节点加入队尾
        |
        v
检查前驱节点状态
        |
    +---+---+
    |       |
    v       v
前驱是头节点   前驱不是头节点
且获取锁成功      |
    |            v
    v         阻塞等待
成为新头节点      |
    |            v
  返回      被唤醒后重试获取锁
```

### 核心源码分析

```java
// 获取锁的入口
public final void acquire(int arg) {
    if (!tryAcquire(arg) && 
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

// 将节点加入队列
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

// 队列中获取锁
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 前驱是头节点且获取锁成功
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 检查是否应该阻塞
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

### 为什么用双向链表？

1. **方便找到前驱节点**：检查前驱状态决定是否阻塞
2. **支持取消操作**：节点取消时，可以快速将其从队列中移除
3. **提高效率**：不需要从头遍历查找前驱