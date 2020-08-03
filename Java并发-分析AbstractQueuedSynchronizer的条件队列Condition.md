前一篇文章以ReentrantLock的非公平锁为例，讲了AQS的原理，今天先来讲讲公平锁吧。

对非公平锁不太清楚的可以看看上一篇文章。

## ReentrantLock的公平锁

上篇文章我们讲了，非公平锁会试图两次去抢占锁，然后才会去排队，那么可想而知，对于公平锁而言，必须要看看有没有线程在排队，有的话就只能乖乖排在阻塞队列的末尾，没有线程在排队，才能去抢占锁。

下面我们来对比一下源码

公平锁：

```java
final void lock() {
    acquire(1);
  }

/**尝试获取锁资源**/
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
      if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
      }
    }
    else if (current == getExclusiveOwnerThread()) {
      int nextc = c + acquires;
      if (nextc < 0) // overflow
        throw new Error("Maximum lock count exceeded");
      setState(nextc);
      return true;
    }
    return false;
  }
```

非公平锁：

```java
final void lock() {
    if (compareAndSetState(0, 1))
      setExclusiveOwnerThread(Thread.currentThread());
    else
      acquire(1);
  }

/**尝试获取锁**/
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
      /**在尝试获取锁时，公平锁会通过hasQueuedPredecessors()方法检测有没有线程在排队，有的话直接返回失败**/
      if (!hasQueuedPredecessors() &&
          compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
      }
    }
    else if (current == getExclusiveOwnerThread()) {
      int nextc = c + acquires;
      if (nextc < 0)
        throw new Error("Maximum lock count exceeded");
      setState(nextc);
      return true;
    }
    return false;
  }
```

可以很明显地看出来，加锁时的区别，非公平锁会首先尝试抢占锁资源。其次在尝试获取锁资源时，公平锁会首先检测有没有线程在排队，有的话直接返回失败。

## 从一道面试题说起

大家在面试的时候，有个问题应该经常会被问到，实现一个生产者消费者模型。

我认为这道题的关键在于要实现一个阻塞队列，当队列为空时，消费者线程要挂起，当队列满了时，生产者线程要被挂起。

1. 用wait和notify实现一个生产者-消费者模型

   ```java
   public class ProducerAndConsumer {
       List<String> apple = new ArrayList<>();
       public synchronized void produce() {
           while (apple.size() ==5) {
               try {
                   System.out.println("队列已满，生产者等待");
                   wait();
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
   
           }
           String aa = UUID.randomUUID().toString().split("-")[0];
           apple.add(aa);
           System. out .println(Thread.currentThread().getName()+"生成苹果成功！"+aa);
           notify();
       }
       public synchronized void consumer() {
           while (apple.size() ==0) {
               try {
                   System.out.println("队列为空，消费者等待");
                   wait();
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
           }
           String aa = apple.get(apple.size()-1);
           apple.remove(aa);
           System. out.println(Thread.currentThread().getName()+"消费苹果成功！"+aa);
           notify();
       }
   }
   ```

   这里我们用一个List来保存生成的数据，对外提供produce()和consume()接口，这两个接口要保证是线程安全的。

2. 用ReentrantLock的Condition来实现生产者-消费者模型

   ```java
   public class ProducerAndConsumer_Lock {
       private List<String> apples = new ArrayList<>();
       private Lock lock = new ReentrantLock();
       private Condition fullCondition = lock.newCondition();
       private Condition emptyCondition = lock.newCondition();
       //线程安全的队列添加方法
       public void produce() {
           lock.lock();
           try {
               while (apples.size() == 5) {
                   /**当队列满了时，将生产者线程加入到一个等待队列中，等待队列不满时被唤醒**/
                   System.out.println("队列已满，生产者等待，开始通知消费者消费");
                   //队列已满，生产者等待
                   fullCondition.await();
               }
               //生产
               String aa = UUID.randomUUID().toString().split("-")[0];
               apples.add(aa);
               System.out.println(Thread.currentThread().getName()+"生成苹果成功！"+aa);
               //消费者条件满足，唤醒消费线程
               emptyCondition.signalAll();
           }catch (Exception e){
               e.printStackTrace();
           }finally {
               lock.unlock();
           }
       }
       //线程安全的消费方法
       public void consume(){
           lock.lock();
           try {
               while(apples.size() == 0){
                  /**当队列为空时，将消费者者线程加入到一个等待队列中，等待队列有数据时时被唤醒**/
                   System.out.println("队列已空，消费者等待，开始通知生产者生产");
                   emptyCondition.await();
               }
               //开始消费
               String aa = apples.get(apples.size()-1);
               apples.remove(aa);
               System. out.println("消费苹果成功！"+aa);
               //生产者线程条件满足，唤醒生产者线程
               fullCondition.signalAll();
           }catch (Exception e){
   						e.printStackTrace();
           }finally {
               lock.unlock();
           }
       }
   }
   ```

## Condition结构分析

当我们通过 `lock.newCondition()` 生成一个Condition队列时，实际上是新建了一个ConditionObject对象。

```java
public Condition newCondition() {
        return sync.newCondition();
    }

/**ConditionObject是AQS中的一个静态内部类**/
final ConditionObject newCondition() {
        return new ConditionObject();
   }  

public class ConditionObject implements Condition, java.io.Serializable {
        /**构建条件队列的哨兵节点**/
        private transient Node firstWaiter;
        private transient Node lastWaiter;
}
```

我们在上一篇文章中分析Node节点的时候，我们说过有个属性是Node nextWaiter，这个就是用来链接条件队列的。

```java
static final class Node {
  Node nextWaiter;
}
```

所以我们可以看出，条件队列实际是一个单向链表。

1. 当我们调用某个Condition的 `await()` 时，会将当前线程加入到这个条件队列的单向链表中。
2. 当另一个线程调用对应Condition的 `singal()` 时，会将对应Condition的单向链表中节点转移至阻塞队列，即我们上篇文章中讲到的，等待获取锁资源的双向链表中，参与到锁资源的竞争中。

## 源码分析

Condition是需要依靠Lock来进行创建的，而且一个锁可以创建多个Condition队列。

当我们调用await()时。

```java
public final void await() throws InterruptedException {
    // 检测线程中断
    if (Thread.interrupted())
        throw new InterruptedException();
    //当前线程加入等待队列
    Node node = addConditionWaiter();
    //释放锁
    long savedState = fullyRelease(node);
    int interruptMode = 0;
    /**
         * 检测此节点的线程是否在同步队上，如果不在，则说明该线程还不具备竞争锁的资格，则继续等待
         * 直到检测到此节点在同步队列上
         */
    while (!isOnSyncQueue(node)) {
        //线程挂起
        LockSupport.park(this);
        //如果已经中断了，则退出
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //竞争同步状态
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    //清理下条件队列中的不是在等待条件的节点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

//将线程加入到条件队列的链表中
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 如果条件队列最后一个节点取消了排队，则出发一次清理工作，将取消了排队的节点清理出去
    if (t != null && t.waitStatus != Node.CONDITION) {
      unlinkCancelledWaiters();
      t = lastWaiter;
    }
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
      firstWaiter = node;
    else
      t.nextWaiter = node;
    lastWaiter = node;
    return node;
  }

/**检测当前节点是否已经在阻塞队列中**/
final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
      
        return findNodeFromTail(node);
    }
    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }
```



调用Condition的signal()方法，将会唤醒在等待队列中等待最长时间的节点（条件队列里的首节点），在唤醒节点前，会将节点移到CLH同步队列中。

调用`condition1.signal()` 触发一次唤醒，此时唤醒的是队头，会将condition1 对应的**条件队列**的 firstWaiter（队头)移到**阻塞队列的队尾**，等待获取锁，获取锁后 await 方法才能返回，继续往下执行。

```java
public final void signal() {
    //检测当前线程是否为拥有锁的独
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //头节点，唤醒条件队列中的第一个节点
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);    //唤醒
}

/** 1. 先检测当前线程是否持有锁，这是前置条件。
		2. 唤醒等待队列中的第一个节点。**/
private void doSignal(Node first) {
    do {
        //修改头结点，完成旧头结点的移出工作
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
/**将节点转移到阻塞队列中，并唤醒节点**/
final boolean transferForSignal(Node node) {
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
      return false;
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
      LockSupport.unpark(node.thread);
    return true;
    }
```

当线程被转移到了阻塞队列双向链表中，被唤醒后，会从下面开始运行。这个时候节点已经在同步队列中的，所以会从while循环中跳出，执行acquireQueued()方法，尝试去获取锁资源，未获取到锁资源将会被挂起。具体的大家可以去查看上一篇文章。

```java
public final void await() throws InterruptedException {
    // 检测线程中断
    if (Thread.interrupted())
        throw new InterruptedException();
    //当前线程加入等待队列
    Node node = addConditionWaiter();
    //释放锁
    long savedState = fullyRelease(node);
    int interruptMode = 0;
    /**
         * 检测此节点的线程是否在同步队上，如果不在，则说明该线程还不具备竞争锁的资格，则继续等待
         * 直到检测到此节点在同步队列上
         */
    while (!isOnSyncQueue(node)) {
        //线程挂起
        LockSupport.park(this);
        //如果已经中断了，则退出
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //竞争同步状态
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    //清理下条件队列中的不是在等待条件的节点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

