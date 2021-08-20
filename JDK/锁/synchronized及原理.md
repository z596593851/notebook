# 一、对象锁，类锁
当synchronized修饰普通方法、修饰某个对象、修饰this时，是对象锁；

当synchronized修饰静态方法、修饰.class时，是类锁。

对于代码段1：
```java
test1.put();
```
和代码段2:
```java
test2.put();
```
如果Test类的put加了对象锁，那么当不同的线程执行到代码段1处时都会产生锁竞争，而执行到1处和2处的线程之间不会产生锁竞争；

如果Test类的put加了类锁，那么不管执行到1还是2处的线程都会产生锁竞争。

现在的应用一般都以spring为主，而spring管理的类一般都是单例的，所以加锁时用对象锁和类锁效果都一样。

# 二、synchronized原理
synchronized是通过monitor来实现同步的。
-   管程 (英语：Monitors，也称为监视器) 是一种程序结构，结构内的多个子程序（对象或模块）形成的多个工作线程互斥访问共享资源。
-   这些共享资源一般是硬件设备或一群变量。管程实现了在一个时间点，最多只有一个线程在执行管程的某个子程序。
-   与那些通过修改数据结构实现互斥访问的并发程序设计相比，管程实现很大程度上简化了程序设计。
-   管程提供了一种机制，线程可以临时放弃互斥访问，等待某些条件得到满足后，重新获得执行权恢复它的互斥访问。
## 2.1 synchronized作用于代码块的原理
```java
public class SynchronizedTest { 
	public void doSth(){ 
		synchronized (SynchronizedTest.class){ 
			System.out.println("test Synchronized" ); 
		} 
	} 
}
```
反编译得：
![[Pasted image 20210722165148.png]]
**对象锁**的实现使用的是字节码**monitorenter**和**monitorexit**指令，其中monitorenter指令指向同步代码块的开始位置，monitorexit指令则指明同步代码块的结束位置，当执行monitorenter指令时，当前线程将试图获取 objectref(即对象锁) 所对应的 monitor 的持有权，当 objectref 的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。如果当前线程已经拥有 objectref 的 monitor 的持有权，那它可以重入这个 monitor (关于重入性稍后会分析)，重入时计数器的值也会加 1。倘若其他线程已经拥有 objectref 的 monitor 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即monitorexit指令被执行，执行线程将释放 monitor(锁)并设置计数器值为0 ，其他线程将有机会持有 monitor 。
而monitor的本质是操作系统的互斥锁(Mutex Lock)。
java中每个对象都可以成为锁，是因为java对象头的设计。

**对象的内存布局：**

在HotSpot虚拟机中,对象在内存中存储的布局可以分为3块区域：对象头（Header），实例数据（Instance Data）和对象填充（Padding）。
![[Pasted image 20210722162625.png]]
-   **实例数据**：对象真正存储的有效信息，存放类的属性数据信息，包括父类的属性信息；
-   **对齐填充**：由于虚拟机要求 对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐。
-   **对象头**：Hotspot虚拟机的对象头主要包括两部分数据：Mark Word（标记字段）、Class Pointer（类型指针）。

**对象头：**
**对象头**含有三部分：Mark Word（存储对象自身运行时数据）、Class Metadata Address（存储类元数据的指针）、Array length（数组长度，只有数组类型才有）
![[Pasted image 20210722162856.png]]

**Mark word：**
Mark Word 用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等。

在32位的HotSpot虚拟机中，如果对象处于未被锁定的状态下，那么Mark Word的32bit空间里的25位用于存储对象哈希码，4bit用于存储对象分代年龄，2bit用于存储锁标志位，1bit固定为0，表示非偏向锁。其他状态如下图所示：
![[Pasted image 20210722164038.png]]
所以对象与monitor怎么关联？
-   对象里有对象头
-   对象头里面有Mark Word
-   Mark Word指针指向了monitor

## 2.2 synchronized作用于方法的原理
```java
public synchronized void doSth(){ 
	System.out.println("test Synchronized method" ); 
}
```
反编译得：
![[Pasted image 20210722164439.png]]
由图可得，添加了synchronized关键字的方法，多了**ACC_SYNCHRONIZED**标记。即JVM通过在方法访问标识符(flags)中加入ACC_SYNCHRONIZED来实现同步功能。

**方法级的同步**是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中。JVM可以从方法常量池中的方法表结构(method_info Structure) 中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法。当方法调用时，调用指令将会 检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先持有monitor， 然后再执行方法，最后再方法完成(无论是正常完成还是非正常完成)时释放monitor。

# 三、synchronized锁升级过程
1.  检测Mark Word里面是不是当前线程的ID，如果是，表示当前线程处于偏向锁
2.  如果不是，则使用CAS将当前线程的ID替换Mard Word，如果成功则表示当前线程获得偏向锁，置偏向标志位1
3.  如果失败，则说明发生竞争，撤销偏向锁，进而升级为轻量级锁。
4.  当前线程使用CAS将对象头的Mark Word替换为锁记录指针，如果成功，当前线程获得锁
5.  如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。
6.  如果自旋成功则依然处于轻量级状态。
7.  如果自旋失败，则升级为重量级锁