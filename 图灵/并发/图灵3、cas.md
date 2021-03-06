## cas
cas操作需要三个参数，V是被修改值的地址，E是V的旧值，U是要更新的值。当且仅当 V 的值等于E时，才用U更新E，否则不会执行任何操作。比较和替换是一个原子操作，对应一条操作系统的指令（原语，不可中断），并且在指令前加lock来保证可见性。
 
Hotspot 虚拟机对compareAndSwapInt 方法的实现如下:
![[Pasted image 20211230225448.png]]


核心逻辑在Atomic::cmpxchg方法中，这个根据不同操作系统和不同CPU会有不同的 实现。这里我们以linux_x86的为例，查看Atomic::cmpxchg的实现：
![[Pasted image 20211230225607.png]]
如果成功返回旧值，失败则返回新值。

cas缺陷：
- 高并发时cpu自旋严重
- 只能保证一个变量的原子操作
- ABA

## @sun.misc.Contended 介绍

@sun.misc.Contended 是 Java 8 新增的一个注解，对某字段加上该注解则表示该字段会单独占用一个**缓存行**（Cache Line）。

这里的缓存行是指 CPU 缓存（L1、L2、L3）的存储单元，常见的缓存行大小为 64 字节。

（注：JVM 添加 -XX:-RestrictContended 参数后 @sun.misc.Contended 注解才有效）

## 单独使用一个缓存行有什么作用——避免伪共享

为了提高读取速度，每个 CPU 有自己的缓存，CPU 读取数据后会存到自己的缓存里。而且为了节省空间，一个缓存行可能存储着多个变量，即**伪共享**。但是这对于共享变量，会造成性能问题：

当一个 CPU 要修改某共享变量 A 时会先**锁定**自己缓存里 A 所在的缓存行，并且把其他 CPU 缓存上相关的缓存行设置为**无效**。但如果被锁定或失效的缓存行里，还存储了其他不相干的变量 B，其他线程此时就访问不了 B，或者由于缓存行失效需要重新从内存中读取加载到缓存里，这就造成了**开销**。所以让共享变量 A 单独使用一个缓存行就不会影响到其他线程的访问。

## Java 8 之前的方案
在 Java 8 之前，是通过代码里手动添加属性的方式解决的，如：

`class LongWithPadding {
	long value;
	long p0, p1, p2, p3, p4, p5, p6;
}` 

一个 long 占 8 个字节，再添加 7 个 long 属性就会变成 64 个字节，刚好是一个缓存行大小。

但是注意，Java 7 开始 JVM 做优化时可能会把不用的变量给去掉，所以这种方法并不推荐使用。

## 适用场景
主要适用于**频繁写**的**共享数据**上。如果不是频繁写的数据，那么 CPU 缓存行被锁的几率就不多，所以没必要使用了，否则不仅占空间还会浪费 CPU 访问操作数据的时间。

## LongAdder流程