# HashMap
## 一. put
```java
public V put(K key, V value) { 
	int hash = hash(key.hashCode());
	//如果key为null，则放在下标0处
	int i = indexFor(hash, table.length);
	for (Entry<K,V> e = table[i]; e != null; e = e.next) { 
		Object k; 
		//如果发生hash冲突，则根据==和equals判断是否覆盖
		if (e.hash == hash && 
		((k = e.key) == key || (key != null && key.equals(k)))) { 
			V oldValue = e.value; 
			e.value = value; 
			e.recordAccess(this); 
			return oldValue; 
		} 
	} 
	modCount++; 
	addEntry(hash, key, value, i); 
	return null; 
} 
```
## 二. get
```java
 public V get(Object key) { 
	 if (key == null) 
		 return getForNullKey(); 
	 int hash = hash(key.hashCode()); 
	 for (Entry<K,V> e = table[indexFor(hash, table.length)]; 
		 e != null; 
		 e = e.next
		 ){ 
			 Object k; 
			 if (e.hash == hash && 
			 ((k = e.key) == key || (key != null && key.equals(k)))) 
				 return e.value; 
	 } 
	 return null; 
 }
```
1、hash()方法
```java
static final int hash(Object key) {
	int h;
	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
**>>>**是无符号右移的意思，这里的作用是将hashCode和自己的高16位做^运算。由于在计算下标时要h & (length-1)，而length一般都小于2^16即小于65536。 h & (length-1)的结果始终是h的低16位与（length-1）进行&运算。所以为了让h的低16位更随机，故选择让其和高16位做^运算。
2、扩容机制
初始容量是16，扩容因子是0.75，扩容时大小变为原来2倍。
在jdk1.7中，触发扩容时，会先申请一个2倍于原Node数组大小的数组，然后依次遍历原数组，并重新计算每个元素在新数组中对应的下标。由于jdk1.7在往链表中插入数据时使用的是头插法，这样会导致扩容后链表中的元素变为逆序。
在jdk1.8中，为了防止逆序，头插法改为了尾插法，并且不再像1.7那样重新计算每个元素的新的下标 h&(length-1)，而是计算`key^oldSize`如果值为0则新下标不变，如果值为1，则`新下标=原下标+原容量`。原理如下：
![[Pasted image 20210720181628.png]]
3、红黑树
jdk1.8中，当节点的链表长度超过8以后，转化为红黑树（同时数组容量必须大于64时才转化为红黑树，否则触发扩容）。这么做是为了防止在数组容量较小的初期，多个值插入同一位置导致引起不必要的转化。
阈值设置为8的原因是，理想状态下哈希表的每个箱子中，元素的数量遵守泊松分布，元素个数超过8的概率极小。 而当链表中的元素个数小于6时，红黑树退化为链表。设置为6而不是7，是为了防止当链表容量为7时，频繁的插入和删除一个元素导致频繁的进行链表和红黑树之间的转化。

# ConcurrentHashMap
1、锁
jdk1.7中，ConcurrentHashMap会创建16个Segment，Segment继承ReentrantLock，每个Segment包含一个HashEntry[]数组。取值时，先根据hashCode找到Segment，再根据hashCode找到数组下标。
jdk1.8中，已经摒弃了Segment的概念，并发控制使用Synchronized和CAS来操作。在put时，如果没有hash冲突，则以CAS方式插入；如果有hash冲突，则用Synchronized加锁。
2、size()
JDK1.8的ConcurrentHashMap使用baseCount以及CounterCell[]数组统计容量的大小。本来可以使用CAS的方式更新一个表示容量的字段，但是为了保证高并发下的效率，作者使用CounterCell[]数组的方式，用多个CAS的方式去更新多个表示容量的字段，最后再将这些字段加和表示map的总容量。
size()方法调用sumCount()计算总容量：
```java
 final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
可以看出sumCount()就是将baseCount和CounterCell[]中各元素求和。在put()、remove()等方法中会调用addCount()去修改map的容量：
```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    //第一次执行到这里，CounterCell为空，是否执行if内的语句取决于baseCount是否能cas累加成功
    //如果永远没有并发，则永远只累加baseCount。一旦有并发产生，就会初始化CounterCell，不再累加baseCount
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        //cas失败，有竞争
        boolean uncontended = true;
        //第一次执行到这里，as为空，直接执行if内语句进行CounterCell的初始化
        if (as == null || (m = as.length - 1) < 0 ||
            //probe相当于线程的hash码，这里判断当前线程的CounterCell是否初始化过，否则初始化，是则累加对应的cellValue
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            //初始化CounterCell
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
}
```
可以看出，在没有并发的情况下，只会累加baseCount的值；当有冲突时，才会调用fullAddCount()初始化CounterCell[]：
```java
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    //初始化线程的探针
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        //第一次counterCells为空
        //之后counterCells不为空
        CounterCell[] as; CounterCell a; int n; long v;
        if ((as = counterCells) != null && (n = as.length) > 0) {
            //当前线程对应的CounterCell槽位为空，初始化槽位
            if ((a = as[(n - 1) & h]) == null) {
                //cas
                if (cellsBusy == 0) {            // Try to attach new Cell
                    CounterCell r = new CounterCell(x); // Optimistic create
                    //cas
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            //槽位不为空，再次cas累加cellValue。成功则退出，失败则继续往下执行
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            //CounterCell容量是否超过cpu核心数，否，则接更新collide标志，使下次再进行到此处时执行扩容
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
            else if (!collide)
                collide = true;
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    //扩容为原来的2倍
                    if (counterCells == as) {// Expand table unless stale
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            //更新probe
            h = ThreadLocalRandom.advanceProbe(h);
        }
        //CELLSBUSY是一个cas方式的自旋锁，旨在初始化CounterCell[]
        else if (cellsBusy == 0 && counterCells == as &&
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {
                if (counterCells == as) {
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        //获取cas锁失败
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```