### 管程
管理共享变量以及对共享变量操作的过程，让他们支持并发

管程的MESA模型：

![[Pasted image 20220104161716.png]]

类似于Lock的阻塞队列和Condition队列。java内置的管程（sychronized）参考了MESA，其中只有一个条件队列：

![[Pasted image 20220104162110.png]]

管程对象的结构：

```java
ObjectMonitor() { 
//对象头
_header = NULL;
_count = 0; 
_waiters = 0, 
// 锁的重入次数 
_recursions = 0;
//存储锁对象
_object = NULL; 
// 标识拥有该monitor的线程（当前获取锁的线程）
_owner = NULL; 
// 等待线程（调用wait）组成的双向循环链表，_WaitSet是第一个节点 
_WaitSet = NULL; 
_WaitSetLock = 0 ; 
_Responsible = NULL ; 
_succ = NULL ; 
//多线程竞争锁会先存到这个单向链表中 （FILO栈结构）
_cxq = NULL ; 
FreeNext = NULL ; 
//存放在进入或重新进入时被阻塞(blocked)的线程 (也是存竞争锁失 败的线程)
_EntryList = NULL ;  
_SpinFreq = 0 ; 
_SpinClock = 0 ; 
OwnerIsThread = 0 ; 
_previous_owner_tid = 0; 
}
```
![[Pasted image 20220104183544.png]]
因竞争锁而阻塞的线程会放入_cxq（栈，唤醒时会先进的后唤醒），调用wait()的线程会放入_WaitSet，从wait()状态唤醒的线程会放入_EntryList。jvm的默认唤醒策略是，先唤醒_EntryList，如果EntryList为空，则将_cxq中的元素按原有顺序插入到_EntryList并唤醒。

### 对象的内存布局

![[Pasted image 20220104172900.png]]
![[Pasted image 20220104173315.png]]

所以一个new Object()占16字节（对象头12字节+填充位6字节，jvm要求对象的地址是8的整数倍）

sychronized对象锁底层依赖monitor对象，即每个锁对象对应一个monitor对象，锁对象会把指向monitor对象的指针存放在mark word中。

mark word中存储的锁信息：

![[Pasted image 20220104190745.png]]

### 偏向锁延迟
```java
public void method(){
	new Thread(()->{
		sychronized(obj){
			//...
		}
	}).start();
}
```
此时obj上的是轻量级锁，而非偏向锁。因为HotSpot虚拟机在启动后有 4s 的延迟才会对每个新建的对象开启偏向锁模式，除非使用jvm启动参数禁止偏向锁延迟：
```java
//关闭偏向锁延迟
‐XX:BiasedLockingStartupDelay=0

//禁止偏向锁
‐XX:‐UseBiasedLocking

//启用偏向锁
‐XX:+UseBiasedLocking
```
但是当关闭偏向锁延迟后，会使得所有对象都变为偏向锁匿名状态（可偏向状态，没有线程id）。

### 偏向锁撤销
#### 偏向锁->无锁：

当对象可偏向时（开启了偏向锁，4s后锁标识为是101，但是没有指向任何线程），调用hashCode，MarkWord将变成未锁定状态，并只能升级成轻量锁：

```java
public void method(){
	Object obj=new Object();
	//调用后，obj变为无锁状态
	obj.hashCode();
	new Thread(()->{
		//升级为轻量锁
		sychronized(obj){
			//...
		}
	}).start();
}
```

#### 偏向锁->轻量级锁：

当对象处于偏向锁，且持有锁的线程处于同步代码块中时，另一个线程来竞争锁，此时竞争者向JVM提交一个时间停止的请求；在时间停止的时候（safe point），JVM线程伪造一个displaced mark word到持有锁的线程的栈上，object的mark work更新为轻量锁模式。然后唤醒持有锁的线程。这样持有锁的线程就会自以为自己持有的是轻量锁了。而竞争者也会按照轻量锁的模式去竞争锁。
（到底是先暂停再时停，还是先时停再暂停）
![[Pasted image 20220108001004.png]]

#### 偏向锁->重量级锁：

当对象正处于偏向锁时，调用HashCode将使偏向锁强制升级成重量锁

```java
public void method(){
	Object obj=new Object();
	new Thread(()->{
		sychronized(obj){
			//调用后升级为重量级锁
			obj.hashCode();
			//...
		}
	}).start();
}
```

因为对于一个对象，其HashCode只会生成一次并保存，偏向锁没有地方保存。

轻量级锁会在锁记录中记录 hashCode，重量级锁会在 Monitor 中记录 hashCode。

当对象正处于偏向锁时，调用obj.wait(timeout) 会升级为重量级锁，因为要拿到monitor对象；调用obj.notify() 会升级为轻量级锁。

### 批量重偏向 & 批量撤销
从偏向锁的加锁解锁过程中可看出，当只有一个线程反复进入同步块时，偏向锁带来的性能开销基本可以忽略，但是当有其他线程尝试获得锁时，就需要等到safe point时，再将偏向锁撤销为无锁状态或升级为轻量级，会消耗一定的性能，所以在多线程竞争频繁的情况下，偏向锁不仅不能提高性能，还会导致性能下降。于是，就有了批量重偏向与批量撤销的机制。

原理：以class为单位，为每个class维护一个偏向锁撤销计数器，每一次该class的对象发生偏向撤销操作时，该计数器+1，当这个值达到撤销向阈值（默认20）时，JVM就认为该class的偏向锁有问题，因此会进行批量重偏向；当撤销偏向锁阈值超过 40 次后，jvm 会认为不该偏向，于是整个类的所有对象都会变为不可偏向的，新建的对象也是不可偏向的。

测试：

```java
public void biasedLockingTest() {
	Thread.sleep(5000);
	// 创建一个list，来存放锁对象
	List<Object> list = new ArrayList<>();

	// 线程1
	new Thread(() ‐> {
		for (int i = 0; i < 50; i++) {
			// 新建锁对象
			Object lock = new Object();
			synchronized (lock) {
				list.add(lock);
			}
		} 
		try {
			//为了防止JVM线程复用，在创建完对象后，保持线程thead1状态为存活
			Thread.sleep(100000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}, "thead1").start();
	//睡眠3s钟保证线程thead1创建对象完成
	Thread.sleep(3000);

	// 线程2 
	new Thread(() ‐> {
		for (int i = 0; i < 40; i++) { 
			Object obj = list.get(i); 
			synchronized (obj) {
				//do something
			} 
		}
		try { 
			Thread.sleep(100000);
		} catch (InterruptedException e) { 
			e.printStackTrace(); 
		} 
	}, "thead2").start(); 
	Thread.sleep(3000);

	// 线程3
	new Thread(() ‐> {
		for (int i = 0; i < 40; i++) { 
			Object obj = list.get(i); 
			synchronized (obj) {
				//do something
			} 
		}
		try { 
			Thread.sleep(100000);
		} catch (InterruptedException e) { 
			e.printStackTrace(); 
		} 
	}, "thead3").start(); 
	Thread.sleep(3000);

	LockSupport.park(); 
} 
```
解释：
- 线程1结束时，所有锁变为偏向锁；

- 线程2前20次从偏向锁撤销为轻量级锁，释放后变为无锁，此时达到偏向锁撤销的第一个阈值(20次)，之后批量重偏向；后30次批量重偏向，结束后仍是偏向锁；

- 线程3前20次从无锁直接获取轻量锁，后20次(由于已经重偏向过，不能再次重偏向)从偏向锁撤销为轻量级锁，此时达到偏向锁撤销的阈值（40次），最后10次触发批量撤销，升级为轻量级锁；

- 此后新建的obj都会是无锁状态，并且直接升级为轻量级锁，不会再有偏向状态。

注意：
1. 批量重偏向和批量撤销是针对类的优化，和对象无关。 
2. 偏向锁重偏向一次之后不可再次重偏向，只会撤销或批量撤销。 
3. 当某个类已经触发批量撤销机制后，JVM会默认当前类产生了严重的问题，剥夺了该类 的新实例对象使用偏向锁的权利

### 偏向锁的线程id
由于jvm的线程是操作系统级别的线程，开销较大，所以当一个线程获取到偏向锁时，锁对象的MarkWord中会记录线程id，如果该线程释放锁，并且短时间内另一个线程又获取到了这个偏向锁，那么锁对象中的线程id还是上一个线程的id。这个id指的是操作系统级别的线程id，而非jvm的线程id，说明jvm的第二个线程复用了第一个线程在操作系统级别的线程。

### 轻量级锁
轻量级锁的MarkWord会记录指向栈中锁记录的指针：

![[Pasted image 20220105204740.png]]
![[Pasted image 20220107232440.png]]

也就是说，加锁的线程会在自己的线程栈中存一份锁对象的MarkWord，用来解锁时恢复锁对象的hashcode等。当轻量级锁重入时，则会在栈中存一个null值，每次解锁就出栈一次，直到MarkWord也出栈。

轻量级锁在cas失败后就直接升级成重量级锁，这并不等同于轻量级锁是自旋锁，它没有自旋操作。

偏向锁释放后还是偏向锁，轻量级锁释放后是无锁。

偏向锁竞争时变为轻量级锁，轻量级锁解锁后变为无锁（因为轻量级锁在线程栈空间存了无锁状态的锁对象的markword）
轻量级锁竞争激烈变为重量级锁，重量级锁释放变为无锁（通过gc清掉monitor）
无锁也可直接变为重量级锁

### 重量级锁
#### 自旋优化
重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步块，释放了锁），这时当前线程就可以避免阻塞。自旋会占用 CPU 时间，单核 CPU 自旋就是浪费，多核 CPU 自旋才能发挥优势。

在 Java 6 之后自旋是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次；反之，就少自旋甚至不自旋，比较智能。Java 7 之后不能控制是否开启自旋功能

注意：自旋的目的是为了减少线程挂起的次数，尽量避免直接挂起线程（挂起操作涉及系统调用，存在用户态和内核态切换，这才是重量级锁最大的开销）

#### 锁粗化
假设一系列的连续操作都会对同一个对象反复加锁及解锁，甚至加锁操作是出现在循环体中的，即使没有出现线程竞争，频繁地进行互斥同步操作也会导致不必要的性能损耗。如果JVM检测到有一连串零碎的操作都是对同一对象的加锁，将会扩大加锁同步的范围（即锁粗化）到整个操作序列的外部。
```java
StringBuffer buffer = new StringBuffer();
public void append(){
	buffer.append("aaa").append("bbb").append("ccc");
}
```
上述代码每次调用 buffer.append 方法都需要加锁和解锁，如果JVM检测到有一连串的对同一个对象加锁和解锁的操作，就会将其合并成一次范围更大的加锁和解锁操作，即在第一次append方法时进行加锁，最后一次append方法结束后进行解锁。

#### 锁消除
锁消除即删除不必要的加锁操作。锁消除是Java虚拟机在JIT编译期间，通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁，通过锁消除，可以节省毫无意义的请求锁时间。
```java
public void append(String str1, String str2) {
	StringBuffer stringBuffer = new StringBuffer();
	stringBuffer.append(str1).append(str2);
}
```
StringBuffer的append是个同步方法，但是append方法中的 StringBuffer 属于一个局部变量，不可能从该方法中逃逸出去，因此其实这过程是线程安全的，可以将锁消除。

### 逃逸分析
- 方法逃逸(对象逃出当前方法)：
当一个对象在方法中被定义后，它可能被外部方法所引用，例如作为调用参数传递到其他地方中。
- 线程逃逸((对象逃出当前线程)：
这个对象甚至可能被其它线程访问到，例如赋值给类变量或可以在其它线程中访问的实例变量。

使用逃逸分析，编译器可以对代码做如下优化：

**锁消除**

**将堆分配转化为栈分配(Stack Allocation)**
如果一个对象在子程序中被分配，要使指向该对象的指针永远不会逃逸，对象可能是栈分配的候选，而不是堆分配。
```java
private static String alloc() {
	Point point = new Point();
	return point.toString();
}

public static void main(){
	int i=0;
	while(i<50000){
		alloc();
	}
}
```
这样一部分Point会分配到线程栈上，一部分分配到堆上。

**标量替换**
```java
private static void test2() {
	Point point = new Point(1,2);
	System.out.println("point.x="+point.getX()+"; point.y="+point.getY());
	// 上述代码等价于：
	// int x=1;
	// int y=2;
	// System.out.println("point.x="+x+"; point.y="+y);
}
```