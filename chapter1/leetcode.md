#                            Array**BlockingQueue**

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable 
```

继承AbstractQueue接口并实现了BlockingQueue和Serializable接口

其其相关属性如下：

```
final Object[] items;
    int takeIndex;
    int putIndex;
    int count;

    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;
```

ArrayBlockingQueue是基于数组的，因此会有一个数组来保存元素，还有两个指针来指向头结点和尾结点

其 构造方法如下

```
public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }

    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```

构造方法就是创建一个指定大小的队列对象，

#### PUT方法

函数功能：插入一个元素到队列的末尾，如果队列已满则等待

思路如下：

1. 判断要添加的元素是否为null，如果为null，则抛异常
2. 加锁
3. 检查队列是否已满，如果满了，则等待并释放锁，如果没有满则将元素加入队列里
4. 释放锁

```
public void put(E e) throws InterruptedException {
        checkNotNull(e);//检查是否为空，如果为空，则抛NullPointerException
        //获取锁，
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            //检查是否已满，如果已满，则调用Condition的await方法等待并释放锁
            while (count == items.length)
                notFull.await();
            enqueue(e);//如果没满，则直接加入到队列中
        } finally {
            lock.unlock();//最后释放锁
        }
    }

    private static void checkNotNull(Object v) {
        if (v == null)
            throw new NullPointerException();
    }
    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length)//转换下指针
            putIndex = 0;
        count++;
        //当添加元素后，则唤醒一个消费者
        notEmpty.signal();
    }
```

#### Take方法

功能：取出队列中的队首元素，如果为空，则等待直至队列非空

思路：

1. 加锁
2. 检查队列是否为空，如果为空，则阻塞等待，否则取出队首的元素并返回
3. 释放锁

```
public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            //如果队列中存储的元素为空，则等待直至队列中非空
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }

    /**
     * Extracts element at current take position, advances, and signals.
     * Call only when holding lock.
     */
    private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;//置为null
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        //唤醒等待的生产者
        notFull.signal();
        return x;
    }
```

总结：

1.ArrayBlockingQueue队列是基于数组+Condition类来实现的；

2. 线程安全是基于ReentrantLock来保证的

3. 队列中不允许元素为空；



