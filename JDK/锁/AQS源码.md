# 一、AQS的数据结构

AQS是一个双向链表

```java
//表头
private transient volatile Node head;
//表尾
private transient volatile Node tail;
//加锁状态（0为未锁定）
private volatile int state;

protected final int getState() {
	return state;
}

protected final void setState(int newState) {
	state = newState;
}

protected final boolean compareAndSetState(int expect, int update) {
	// See below for intrinsics setup to support this
	return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
Node的结构：
```java
static final class Node {
	//节点加锁状态
	volatile int waitStatus;
	volatile Node prev;
	volatile Node next;
	//锁定着的线程
	volatile Thread thread;
	Node nextWaiter;
	
}
```
其中waitStatus的状态有：
- 0：初始化状态，表示节点在同步队列中等待获取锁

- CANCELLED=1：表示线程因为中断或者等待超时，需要从等待队列中取消等待；

- SIGNAL=-1：当前线程thread1占有锁，队列中的head(仅仅代表头结点，里面没有存放线程引用)的后继结点node1处于等待状态，如果已占有锁的线程thread1释放锁或被CANCEL之后就会通知这个结点node1去获取锁执行。

 - CONDITION=-2：表示结点在等待队列中(这里指的是等待在某个lock的condition上，关于Condition的原理下面会写到)，当持有锁的线程调用了Condition的signal()方法之后，结点会从该condition的等待队列转移到该lock的同步队列上，去竞争lock。(注意：这里的同步队列就是我们说的AQS维护的FIFO队列，等待队列则是每个condition关联的队列)

4. PROPAGTE=-3：表示下一次共享状态获取将会传递给后继结点获取这个共享同步状态。

AQS类继承AbstractOwnableSynchronizer：

```java
    private transient Thread exclusiveOwnerThread;

    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }

    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
```

当中的这两个方法可以分别记录或获取得到锁的线程。

# 二、acquire(int)——以独占方式加锁

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

tryAcquire()尝试直接去获取资源，如果成功则直接返回；(tryAcquire是抽象方法，暴露给子类实现)

addWaiter()方法负责把当前无法获得锁的线程包装为一个Node添加到队尾，SHARED：共享锁，EXCLUSIVE：独占锁。

```java
    static final Node SHARED = new Node();

    static final Node EXCLUSIVE = null;

    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
		//以CAS的方式将当前节点插入表尾
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
		//如果head为空，或者多个节点竞争插入表尾失败，则执行enq
        enq(node);
        return node;
    }
```

enq()以懒加载的方式初始化链表，以及以CAS的方式将node节点插入表尾：
```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        //如果tail为null，表示当前同步链表为null，就必须初始化这个同步队列的head和tail（建立一个空的哨兵结点）
        if (t == null) {
			//以CAS的方式初始化表头节点
            if (compareAndSetHead(new Node()))
                tail = head;
        }
        else {
			//以CAS的方式将当前节点插入表尾，失败则重新进入循环尝试
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
![](https://oscimg.oschina.net/oscnet/up-ce7ae064230057e678166a38de2409d1c98.png)
![](https://oscimg.oschina.net/oscnet/up-cc5c5c304a0bd3729a52e60800adf4560c1.png)

acquireQueued()的主要作用是把已经追加到队列的线程节点通过调用parkAndCheckInterrupt()进行阻塞，等被唤醒后再去竞争资源：
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //获取前驱结点
            final Node p = node.predecessor();
            //如果前驱结点是头节点，尝试加锁，因为此时头节点可能已经释放锁
            if (p == head && tryAcquire(arg)) {
                //如果前驱结点是头节点并且tryAcquire返回true，那么就重新设置头节点为node
                setHead(node);
                p.next = null; //将原来的头节点的next设置为null，交由GC去回收它
                failed = false;
                return interrupted;
            }
            //如果不是头节点,或者虽然前驱结点是头节点但是尝试加锁失败，就会将node结点的waitStatus设置为-1，并且park自己，等待前驱结点的唤醒
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private void setHead(Node node) {  
	head = node;  
	node.thread = null;  
	node.prev = null;  
}
```


shouldParkAfterFailedAcquire:

-   如果前继的节点状态为SIGNAL，表明当前节点需要阻塞，则返回true，此时parkAndCheckInterrupt将导致线程阻塞
-   如果前继节点状态为CANCELLED(ws>0)，说明前置节点已经被放弃，则回溯到一个非取消的前继节点，返回false，重新循环acquireQueued
-   如果前继节点状态为非SIGNAL、非CANCELLED，则设置前继的状态为SIGNAL，重新循环acquireQueued

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //获取前驱结点的waitStatus
    int ws = pred.waitStatus;
    //如果前驱结点的waitStatus为SINGNAL，就直接返回true（阻塞）
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        //将所有的前驱结点状态为CANCELLED的都移除
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        //CAS操作将这个前驱节点设置成SIGHNAL。
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

# 三、release(int)——以独占方式解锁

release：如果可以释放锁，则唤醒队列head节点的下一个节点：

```java
public final boolean release(int arg) {
	if (tryRelease(arg)) {
		Node h = head;
		if (h != null && h.waitStatus != 0)
			//唤醒同步队列head结点的后继结点中的线程
			unparkSuccessor(h);
		return true;
	}
	return false;
}

```


```java
private void unparkSuccessor(Node node) {
	int ws = node.waitStatus;
	if (ws < 0)
		//如果waitStatus小于0需要将其以CAS的方式设置为0
		compareAndSetWaitStatus(node, ws, 0);
	//获取head的后继节点
	Node s = node.next;
	if (s == null || s.waitStatus > 0) {
		s = null;
		//从尾部结点开始寻找，找到离head最近的不为null并且waitStatus<0的结点
		for (Node t = tail; t != null && t != node; t = t.prev)
			if (t.waitStatus <= 0)
				s = t;
	}
	if (s != null)
		//唤醒这个结点中的线程
		LockSupport.unpark(s.thread);
}
```

线程被唤醒后，回到之前被阻塞的地方，再次执行循环中的tryAcquire()获取锁：
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //获取前驱结点
            final Node p = node.predecessor();
            //如果前驱结点是头节点，尝试加锁
            if (p == head && tryAcquire(arg)) {
                //如果前驱结点是头节点并且tryAcquire返回true，那么就重新设置头节点为node
                setHead(node);
                p.next = null; //将原来的头节点的next设置为null，交由GC去回收它
                failed = false;
                return interrupted;
            }
            //如果不是头节点,或者虽然前驱结点是头节点但是尝试加锁失败，就会将node结点的waitStatus设置为-1，并且park自己，等待前驱结点的唤醒
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
acquireQueued返回false，acquire中会直接从if条件退出，最后执行自己锁住的代码块中的程序。

总结一下：

加锁时，如果链表中没有头节点，或者只有一个头节点，那么当前线程执行tryAcquire尝试获取锁。否则直接加入到链表末端并且阻塞当前线程。

解锁时，肯定是头节点解锁，会唤醒头节点后最近的可以唤醒的节点的线程，被唤醒的线程执行tryAcquire尝试获取锁。

# 四、ReentrantLock()的实现

ReentrantLock()的重入性，是由重入度state实现的。

加锁时：如果c==0说明没有线程正在竞争该锁，则通过CAS设置该状态值为acquires，acquires的初始调用值为1；如果c !=0说明有线程正拥有了该锁，但如发现是自己已经拥有锁，只是简单地++acquires，并修改status值。

```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }

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

释放锁时：如果线程多次锁定，则进行多次释放，直至status==0则真正释放锁，所谓释放锁即status为0。因为无竞争所以没有使用CAS。 

```java
    protected final boolean tryRelease(int releases) {
		//state-1
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
			//将当前锁持者设为null
            setExclusiveOwnerThread(null);
        }
        setState(c);
		//只有state==0，才会返回true
        return free;
    }
```

ReentrantLock()有公平锁和非公平锁两种实现，非公平锁如上述，公平锁如下：

```java
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

    public void lock() {
        sync.lock();
    }

```

公平锁：

```java
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
				//判断AQS队列中是否还有等待获取锁的节点
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
    }
```

hasQueuedPredecessors()就是判断锁是否公平的关键，公平锁在加锁时，如果在当前线程之前还有排队的线程就返回false，这时候当前线程就不会去竞争锁，从而保证了锁的公平性。

而非公平锁在加锁时，如果此时AQS队列的头节点正好释放锁，在唤醒下一个节点去获取锁时，两者就会产生加锁竞争。



# 六、condition源码
调用await()的线程1会先释放锁，进而被放入condition队列中，然后阻塞自己：
```java
public final void await() throws InterruptedException {
	if (Thread.interrupted())
		throw new InterruptedException();
	//将当前线程包装下后，添加到Condition自己维护的一个链表中。
	Node node = addConditionWaiter();
	//释放当前线程占有的锁
	int savedState = fullyRelease(node);
	int interruptMode = 0;
	//condition节点会进入循环
	while (!isOnSyncQueue(node)) {
		//释放完毕后，遍历AQS的队列，看当前节点是否在队列中，不在则阻塞自己
		LockSupport.park(this);
		if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
			break;
	}
 	//被唤醒后，重新开始正式竞争锁，同样，如果竞争不到还是会将自己沉睡，等待唤醒重新开始竞争。

	if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
		interruptMode = REINTERRUPT;
	if (node.nextWaiter != null) // clean up if cancelled
		unlinkCancelledWaiters();
	if (interruptMode != 0)
		reportInterruptAfterWait(interruptMode);
}
```

调用signal()方法的线程2会先把condition队列中的头节点拿出，放入AQS队列中，此时该节点还未被唤醒，而是等线程2释放锁后被唤醒，然后去竞争锁：
```java
//condition队列中的头节点
private transient Node firstWaiter;
//condition队列中的尾节点
private transient Node lastWaiter;

public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

private void doSignal(Node first) {
    do {
		//取出condition的头节点进行唤醒
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
		first.nextWaiter = null;
    } while (!transferForSignal(first) &&(first = firstWaiter) != null);
}

final boolean transferForSignal(Node node) {
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
	//将condition头节点放入AQS链表队列中
    Node p = enq(node);
    int ws = p.waitStatus;
    //如果该结点的状态为cancel 或者修改waitStatus失败，则直接唤醒。
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

