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

### 批量重偏向
### 批量撤销

### 偏向锁变更
由于jvm的线程是操作系统级别的线程，开销较大，所以当一个线程获取到偏向锁时，锁对象的MarkWord中会记录线程id，如果该线程释放锁，并且短时间内另一个线程又获取到了这个偏向锁，那么锁对象中的线程id还是上一个线程的id。

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