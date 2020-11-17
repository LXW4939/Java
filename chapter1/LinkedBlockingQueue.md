# Array**BlockingQueue**

![](/assets/import.png)

LinkedBlockingQueue内部有一个嵌套类Node，表示链表的节点

```java
static class Node<E> {
    E item; // 节点元素

    Node<E> next; // 后继节点

    Node(E x) { item = x; }
}
```

主要成员变量

```
// 链表的容量（若不指定则为 Integer.MAX_VALUE）
private final int capacity;

// 当前元素的数量（即链表中元素的数量）
private final AtomicInteger count = new AtomicInteger();

// 链表的头节点（节点元素为空）
transient Node<E> head;

// 链表的尾结点（节点元素为空）
private transient Node<E> last;

// take、poll 等出队操作持有的锁
private final ReentrantLock takeLock = new ReentrantLock();
/** Wait queue for waiting takes */
// 出队锁的条件队列
private final Condition notEmpty = takeLock.newCondition();

// put、offer 等入队操作的锁
private final ReentrantLock putLock = new ReentrantLock();

/** Wait queue for waiting puts */
// 入队锁的条件队列
private final Condition notFull = putLock.newCondition();
```

构造器

LinkedBlockingQueue有三个构造器，分别如下：

```
// 构造器 1：无参构造器，初始容量为 Integer.MAX_VALUE，即 2^31-1
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}

// 构造器 2：指定容量的构造器
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    // 初始化链表的头尾节点
    last = head = new Node<E>(null);
}

// 构造器 3：用给定集合初始化的构造器
public LinkedBlockingQueue(Collection<? extends E> c) {
    // 调用构造器 2 进行初始化
    this(Integer.MAX_VALUE);
    final ReentrantLock putLock = this.putLock;
    putLock.lock(); // Never contended, but necessary for visibility
    try {
        int n = 0;
        for (E e : c) {
            if (e == null)
                throw new NullPointerException();
            if (n == capacity)
                throw new IllegalStateException("Queue full");
            // 将集合中的元素封装成 Node 对象，并添加到链表末尾
            enqueue(new Node<E>(e));
            ++n;
        }
        count.set(n);
    } finally {
        putLock.unlock();
    }
}
```

enqueue方法如下：

```
// 将 node 节点添加到链表末尾
private void enqueue(Node<E> node) {
    last = last.next = node;
}
```

#### PUT入队方法

```
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    // Note: convention in all put/take/etc is to preset local var
    // holding count negative to indicate failure unless set.
    int c = -1;
    // 把 E 封装成 Node 节点
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        /*
         * Note that count is used in wait guard even though it is
         * not protected by lock. This works because count can
         * only decrease at this point (all other puts are shut
         * out by lock), and we (or some other waiting put) are
         * signalled if it ever changes from capacity. Similarly
         * for all other uses of count in other wait guards.
         */
        // 若队列已满，notFull 等待（类比生产者）
        while (count.get() == capacity) {
            notFull.await();
        }
        // node 入队
        enqueue(node);
        c = count.getAndIncrement();
        // 若该元素添加后，队列仍未满，唤醒一个其他生产者线程
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    // c==0 说明之前队列为空，出队线程处于等待状态，
    // 添加一个元素后，将出队线程唤醒（消费者）
    if (c == 0)
        signalNotEmpty();
}
```

signalNotEmpty方法：

```
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        // 唤醒 notEmpty 条件下的一个线程（消费者）
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
```

#### offer入队列操作

与put操作类似，不过该方法有超时时间，超时后返回false

```
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (e == null) throw new NullPointerException();
    long nanos = unit.toNanos(timeout);
    int c = -1;
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        while (count.get() == capacity) {
            // 等待超时，返回 false
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        // 入队
        enqueue(new Node<E>(e));
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return true;
}
```

```
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    // 若队列已满，直接返回 false
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        // 队列未满，入队
        if (count.get() < capacity) {
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                // 队列未满，唤醒 notFull 下的线程，继续入队
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}
```

take出队方法

```
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        // 队列为空，notEmpty 条件下的线程等待（消费者）
        while (count.get() == 0) {
            notEmpty.await();
        }
        // 从队列头部删除节点
        x = dequeue();
        c = count.getAndDecrement();
        // 若队列不为空，唤醒一个 notEmpty 条件下的线程（消费者）
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    // 队列已经不满了，唤醒 notFull 条件下的线程（生产者）
    if (c == capacity)
        signalNotFull();
    return x;
}
```

dequeue方法

```
private E dequeue() {
    Node<E> h = head; // 头节点
    Node<E> first = h.next; // 头节点的后继节点
    h.next = h; // help GC // 后继节点指向自己（从链表中删除）
    head = first; // 更新头节点
    E x = first.item; // 获取要删除节点的数据
    first.item = null; // 清空数据（新的头节点）
    return x;
}
```

signalNotFull方法

```
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        // 唤醒生产者
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
```

#### poll出队方法

```
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null;
    int c = -1;
    long nanos = unit.toNanos(timeout);
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        // 队列已空
        while (count.get() == 0) {
            // 超时返回 null
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
       x = dequeue();
        // 若队列不空，唤醒一个 notEmpty 条件下的线程（消费者）
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    // 队列不满，唤醒 notFull 条件下的线程（生产者）
    if (c == capacity)
        signalNotFull();
    return x;
}
```

```
public E poll() {
    final AtomicInteger count = this.count;
    // 队列为空，返回 null
    if (count.get() == 0)
        return null;
    E x = null;
    int c = -1;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        // 队列不为空，出队
        if (count.get() > 0) {
            x = dequeue();
            c = count.getAndDecrement();
            // 该元素出队后，队列仍不为空，唤醒其他消费者
            if (c > 1)
                notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    // 队列已经不满，唤醒生产者
    if (c == capacity)
        signalNotFull();
    return x;
}
```

#### peek方法

只是返回头结点，并不删除

```
public E peek() {
    if (count.get() == 0)
        return null;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        // 头节点的后继节点
        Node<E> first = head.next;
        if (first == null)
            return null;
        else
            return first.item;
    } finally {
        takeLock.unlock();
    }
}
```

总结：

1. LinkedBlockingQueue是基于单链表的阻塞队列实现，在初始化时可以指定容量，若未指定，则默认容量为Integer.MAX\_VALUE
2. 内部使用Reentrantlock保证线程安全；
3. 常用方法，入队，put, offer; 出队，take, poll, peek

LinkedBlockingQueue与ArrayBlockingQueue的区别：

1. ArrayBlockingQueue使用单个锁，可以指定是否公平；而LinkedBlockingQueue内部使用两个锁：putLock和takeLock，都是非公平锁
2. 入队和出队区别
   * 入队时，LinkedBlockingQueue会判断当前元素入队后，队列是否已满，若未满，则唤醒其他生产者线程；如队列后，当队列之前为空时才唤醒其他消费者线程；ArrayBlockingQueue在每次入队都会唤醒消费者线程；
   * 出队时，LinkedBlockingQueue会判断当前元素出队后，队列是否为空，若未空，则唤醒其他消费者线程；而出队后，当队列之前为满时才唤醒其他生产者线程；ArrayBlockingQueue则是每次出队都会唤醒生产者线程



