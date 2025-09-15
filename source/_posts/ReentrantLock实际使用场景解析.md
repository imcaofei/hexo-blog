---
title: ReentrantLock实际使用场景解析
date: 2025-09-15 13:41:28
tags:
  - ReentrantLock
  - 并发控制
  - 锁优化
  - 系统架构
---

首先，我们快速回顾一下 `ReentrantLock` 的核心优势，这有助于理解为什么会在某些场景下选择它而不是传统的 `synchronized` 关键字：

* **可中断的锁获取**：`lockInterruptibly()` 允许在等待锁时响应中断，为处理死锁或取消任务提供了更大的灵活性。
* **尝试非阻塞获取锁**：`tryLock()` 方法可以立即返回获取锁的结果，或者在一段指定时间内尝试，避免了无限期等待，非常适合解决活锁或进行优雅降级。
* **公平锁选项**：通过构造函数可以创建公平锁（`fair = true`），虽然会牺牲一些吞吐量，但能保证等待时间最长的线程优先获得锁，避免了线程饥饿。
* **支持多个条件变量**：一个 `ReentrantLock` 可以关联多个 `Condition`
  对象，这意味着我们可以对不同的等待线程组进行更精细的唤醒控制（例如，生产者-消费者模型中可以分别唤醒生产者或消费者）。

基于这些特点，`ReentrantLock` 在需要**更灵活、更复杂、更健壮**的并发控制的场景中大放异彩。

***

### 场景一：细粒度的资源池管理（如数据库连接池）

**背景**：
想象一个自研或需要高度定制的数据库连接池。连接池中的连接是有限且有状态的珍贵资源。分配和归还连接是一个典型的并发操作。

**问题**：
使用 `synchronized` 关键字来同步 `getConnection()` 和 `releaseConnection()`
方法虽然可行，但粒度太粗。所有线程在获取和归还任何连接时都会竞争同一把锁，即使池中有多个空闲连接，也会变成串行操作，并发性能低下。

**解决方案与 `ReentrantLock` 的应用**：
我们可以为**每个独立的资源（连接）** 分配一个独立的 `ReentrantLock`，实现更细粒度的锁定。

```java
import java.util.concurrent.locks.ReentrantLock;

public class FineGrainedConnectionPool {
    // 模拟连接池，实际可能是一个队列
    private final Connection[] connections;
    // 为每一个连接配备一个独立的锁
    private final ReentrantLock[] connectionLocks;
    // 一个状态标记数组，表示连接是否被占用
    private final boolean[] inUse;

    public FineGrainedConnectionPool(int size) {
        connections = new Connection[size];
        connectionLocks = new ReentrantLock[size];
        inUse = new boolean[size];
        for (int i = 0; i < size; i++) {
            connections[i] = createNewConnection(); // 创建新连接
            connectionLocks[i] = new ReentrantLock(); // 非公平锁，通常性能更好
            inUse[i] = false;
        }
    }

    public Connection getConnection() throws InterruptedException {
        // 遍历所有连接，尝试获取空闲的那个
        for (int i = 0; i < connections.length; i++) {
            // 使用 tryLock 非阻塞地尝试获取当前连接的锁
            if (connectionLocks[i].tryLock(50, TimeUnit.MILLISECONDS)) {
                try {
                    if (!inUse[i]) {
                        inUse[i] = true;
                        System.out.println(Thread.currentThread().getName() + " acquired connection " + i);
                        return connections[i];
                    }
                } finally {
                    // 无论是否成功获取到连接，都要释放当前连接的锁
                    connectionLocks[i].unlock();
                }
            }
        }
        // 如果所有连接都忙，可以抛出异常、等待重试或使用其他策略（如创建新连接）
        throw new RuntimeException("No available connection after retrying");
    }

    public void releaseConnection(Connection conn) {
        // 找到要归还的连接索引 (这里需要实现根据conn找到index的逻辑，简化处理)
        int index = findIndex(conn);
        connectionLocks[index].lock();
        try {
            inUse[index] = false;
            System.out.println(Thread.currentThread().getName() + " released connection " + index);
        } finally {
            connectionLocks[index].unlock();
        }
    }

    private int findIndex(Connection conn) { ...}

    private Connection createNewConnection() { ...}
}
```

**为什么用 `ReentrantLock`？**

1. **细粒度锁**：每个连接有自己的锁，多个线程可以同时尝试获取**不同**的空闲连接，极大地提高了并发度。
2. **`tryLock()` 的优势**：`getConnection()` 中使用 `tryLock(timeout)`，线程不会在一个繁忙的连接上傻等，而是快速尝试下一个，避免了不必要的阻塞。如果使用
   `synchronized`，很难实现这种“尝试失败后立即尝试下一个”的逻辑。
3. **超时控制**：`tryLock` 提供了超时机制，防止线程无限期等待，增加了系统的健壮性。

***

### 场景二：实现高性能的阻塞队列（如自定义生产者-消费者）

**背景**：
`LinkedBlockingQueue` 等JDK内置队列内部就使用了 `ReentrantLock`。假设我们需要实现一个有自己的特殊逻辑（例如，优先级、特定触发条件）的阻塞队列。

**问题**：
我们需要保证在队列空时消费者阻塞，队列满时生产者阻塞。并且需要在条件满足时精确地唤醒另一方的线程。

**解决方案与 `ReentrantLock` 的应用**：
利用一个 `ReentrantLock` 和它创建的两个 `Condition` 对象：`notEmpty` 和 `notFull`。

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class MyBlockingQueue<T> {
    private final Object[] items;
    private int putIndex;
    private int takeIndex;
    private int count;

    private final ReentrantLock lock;
    // 两个条件变量：”非空“条件和”非满“条件
    private final Condition notEmpty;
    private final Condition notFull;

    public MyBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(); // 默认非公平锁
        notEmpty = lock.newCondition();
        notFull = lock.newCondition();
    }

    public void put(T t) throws InterruptedException {
        lock.lockInterruptibly();
        try {
            // 1. 如果队列满了，就在“notFull”条件上等待
            while (count == items.length) {
                notFull.await(); // 释放锁并等待，被唤醒后会重新获取锁
            }
            // 2. 队列不满，执行入队操作
            items[putIndex] = t;
            if (++putIndex == items.length) putIndex = 0;
            count++;
            // 3. 入队后，队列肯定“非空”了，唤醒一个在“notEmpty”上等待的消费者线程
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            // 1. 如果队列空了，就在“notEmpty”条件上等待
            while (count == 0) {
                notEmpty.await();
            }
            // 2. 队列不空，执行出队操作
            T x = (T) items[takeIndex];
            items[takeIndex] = null;
            if (++takeIndex == items.length) takeIndex = 0;
            count--;
            // 3. 出队后，队列肯定“未满”了，唤醒一个在“notFull”上等待的生产者线程
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```

**为什么用 `ReentrantLock`？**

1. **多个条件变量（`Condition`）**：这是最核心的原因。`synchronized` 配合 `wait()/notifyAll()`
   只能有一个等待队列，所有生产者和消费者都在同一个队列里，唤醒时可能会唤醒更多的同类线程（“惊群效应”），不够高效。而
   `ReentrantLock` 配合 `Condition` 可以将生产者和消费者放到两个不同的等待集合中，`signal()` 可以精确地只唤醒一个同类线程，效率更高。
2. **可中断的锁**：`lockInterruptibly()` 保证了在等待锁或条件时，线程可以响应中断，使程序更具可控性。

***

### 场景三：交易订单处理中的并发控制（避免重复处理）

**背景**：
在电商系统中，一个订单可能同时来自不同渠道的重复回调（如支付成功通知），或者用户快速重复点击提交订单。我们需要一个机制来保证对
**同一个订单ID**的操作是串行的，并且避免重复处理。

**问题**：
使用全局锁 `synchronized(OrderService.class)` 会导致所有订单的处理都串行化，性能无法接受。我们需要的是对**同一订单ID**
的操作加锁，不同订单ID之间可以并行。

**解决方案与 `ReentrantLock` 的应用**：
使用一个 **`ConcurrentHashMap` 来存储每个订单ID对应的锁**。Java 8 的 `computeIfAbsent` 方法让这个模式变得非常简洁和线程安全。

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;

public class OrderService {
    // Key: 订单ID, Value: 保护该订单的专用锁
    private final ConcurrentHashMap<String, ReentrantLock> orderLocks = new ConcurrentHashMap<>();

    public void processOrder(String orderId) {
        // 为当前订单ID获取或创建一个专属锁
        ReentrantLock lock = orderLocks.computeIfAbsent(orderId, k -> new ReentrantLock());

        lock.lock(); // 锁定这个特定的订单
        try {
            // 核心业务逻辑：检查订单状态、更新库存、生成发货单等
            System.out.println("Processing order: " + orderId + " by " + Thread.currentThread().getName());

            // 模拟耗时操作
            Thread.sleep(1000);

            System.out.println("Finished processing order: " + orderId);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            lock.unlock();
            // 非常重要：处理完成后，移除锁对象以防止内存泄漏。
            // 但要注意，必须在确实没有其他线程在等待这个锁时才能移除。
            // 这里简单的做法可能有问题，更安全的做法是使用引用队列或定时清理。
            orderLocks.remove(orderId, lock); // 只有当map中的值仍然是当前lock时才移除
        }
    }
}
```

**为什么用 `ReentrantLock`？**

1. **细粒度锁（按Key锁）**：实现了完美的并发——不同订单的处理完全并行，只有同一订单的请求才会串行化。这是 `synchronized`
   很难优雅实现的。
2. **可重入性**：如果一个方法内需要多次对同一订单加锁（例如，`processOrder` 内部调用了另一个也需要同步的
   `updateOrderStatus` 方法），可重入特性保证了不会死锁。
3. **灵活性**：如果需要，可以轻松地引入 `tryLock()` 来实现快速失败（例如，“系统繁忙，请稍后再试”的提示）。

### 总结

| 场景          | `ReentrantLock` 的核心价值     | 对比 `synchronized` 的优势   |
|:------------|:--------------------------|:------------------------|
| **细粒度资源池**  | 非阻塞尝试获取锁 (`tryLock`)，超时机制 | 实现细粒度并发控制，避免粗粒度锁的性能瓶颈   |
| **高性能阻塞队列** | 多个条件变量 (`Condition`)      | 实现精确的线程唤醒，减少“惊群效应”，性能更高 |
| **交易订单处理**  | 动态创建细粒度锁（按Key锁）           | 实现完美的并发（不同Key并行），代码更灵活  |

总而言之，当你的并发场景超越了简单的“互斥”需求，开始涉及**超时控制、可中断性、多个等待条件、非阻塞尝试、或需要更细粒度的锁策略
**时，`ReentrantLock` 就是比 `synchronized` 更强大的工具。它在高级并发设计模式中扮演着至关重要的角色。
