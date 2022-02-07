### Semaphore
原理：共享锁

加锁时从总资源中扣取，小于0就阻塞：
```java
public void acquire() throws InterruptedException {  
    sync.acquireSharedInterruptibly(1);  
}

public final void acquireSharedInterruptibly(int arg)  
        throws InterruptedException {  
	if (Thread.interrupted())  
		throw new InterruptedException();  
	if (tryAcquireShared(arg) < 0)  
		doAcquireSharedInterruptibly(arg);  
}

protected int tryAcquireShared(int acquires) {  
	for (;;) {  
		int available = getState();  
		int remaining = available - acquires;  
		if (remaining < 0 || compareAndSetState(available, remaining))  
			return remaining;  
	} 
}

private void doAcquireSharedInterruptibly(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
					//当被唤醒后，将当前节点设置为头节点
					//如果资源充足，可以继续唤醒后续节点
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

private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            //继续唤醒后续节点
            doReleaseShared();
    }
}
```
释放共享锁：
```java
public void release() {  
    sync.releaseShared(1);  
}

public final boolean releaseShared(int arg) {  
	if (tryReleaseShared(arg)) {  
		doReleaseShared();  
		return true; 
	}
	return false;  
}

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

private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    // loop to recheck cases
                    continue;         
                unparkSuccessor(h);
            }
            else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                // loop on failed CAS
                continue;                
        }
        if (h == head)  
            // loop if head changed                 
            break;
    }
}
```
### CountDownLatch
原理：共享锁

CountDownLatch在初始化时会将state置为指定值，主线程在调await()时，在state减为0前会无脑返回-1使得线程阻塞：
```java
public void await() throws InterruptedException {  
	sync.acquireSharedInterruptibly(1);  
}

public final void acquireSharedInterruptibly(int arg)  
        throws InterruptedException {  
    if (Thread.interrupted())  
        throw new InterruptedException();  
 	if (tryAcquireShared(arg) < 0)  
        doAcquireSharedInterruptibly(arg);  
}

protected int tryAcquireShared(int acquires) {  
    return (getState() == 0) ? 1 : -1;  
}
```
当其他线程调用countDown()时，会固定将state-1，直到减为0时，唤醒阻塞的主线程：
```java
public void countDown() {  
    sync.releaseShared(1);  
}

public final boolean releaseShared(int arg) {  
    if (tryReleaseShared(arg)) {  
        doReleaseShared();  
		return true; 
	}  
	return false;
}

protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```


### CyclicBarrier
原理：Rentrenlock和Condition

初始化CyclicBarrier时会初始化count值和thread，并且记录count的初始值以便之后重置count；
当其他线程调用CyclicBarrier.await()时，会count--并且调用Condition.await()阻塞线程；
直到count减为0后，会先执行thread中的任务，然后调用Condition.signalAll()唤醒所有线程去竞争锁；
之后用count的副本重置count，以便复用CyclicBarrier。

### ReentrantReadWriteLock
![](https://oscimg.oschina.net/oscnet/2af06cd6459218c5b771ce5fd477a014f87.jpg)

锁降级中读锁的获取是否必要呢？答案是必要的。主要是为了保证数据的可见性，如果当
前线程不获取读锁而是直接释放写锁，假设此刻另一个线程（记作线程T）获取了写锁并修改了数据，那么当前线程无法感知线程T的数据更新。如果当前线程获取读锁，即遵循锁降级的步骤，则线程T将会被阻塞，直到当前线程使用数据并释放读锁之后，线程T才能获取写锁进行数据更新。

RentrantReadWriteLock不支持锁升级（把持读锁、获取写锁，最后释放读锁的过程。目的也是保证数据可见性，如果读锁已被多个线程获取，其中任意线程成功获取了写锁并更新了数据，则其更新对其他获取到读锁的线程是不可见的。

获取读锁：

```java
public void lock() {
	sync.acquireShared(1);
}

public final void acquireShared(int arg) {
	if (tryAcquireShared(arg) < 0)
		doAcquireShared(arg);
}

protected final int tryAcquireShared(int unused) {

	Thread current = Thread.currentThread();
	int c = getState();

	//写锁存在且不是当前线程
	if (exclusiveCount(c) != 0 &&
		getExclusiveOwnerThread() != current)
		return -1;
	//获取读锁
	int r = sharedCount(c);
	//读锁是否该阻塞，对于非公平模式下写锁获取优先级会高，如果存在要获取写锁的线程则读锁需要让步，公平模式下则先来先到
	if (!readerShouldBlock() &&
		r < MAX_COUNT && //MAX_COUNT为获取读锁的最大数量，为16位的最大值
		compareAndSetState(c, c + SHARED_UNIT)) {
		if (r == 0) {
			//记住第一次获取读锁的重入数
			firstReader = current;
			firstReaderHoldCount = 1;
		} else if (firstReader == current) {
			//累加第一次获取读锁的线程的重入次数
			firstReaderHoldCount++;
		} else {
			//从ThreadLocal中获取当前线程重入读锁的次数，然后自增
			HoldCounter rh = cachedHoldCounter;
			if (rh == null || rh.tid != getThreadId(current))
				cachedHoldCounter = rh = readHolds.get();
			else if (rh.count == 0)
				readHolds.set(rh);
			rh.count++;
		}
		return 1;
	}
	return fullTryAcquireShared(current);
}
```

获取写锁：就相当于获取一把独占锁

```java
public void lock() {
    sync.acquire(1);
}

public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        //有写锁且不是当前线程
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        //重入写锁
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```