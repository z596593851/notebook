# 一、Random

在创建Random类时会生成seed，seed用于生成随机数。在每次生成随机数后，都会使用CAS的方式更新seed，所以Random是线程安全的。但是在并发度较高的情况下，CAS的效率会变得很低。
```java
protected int next(int bits) {
    long oldseed, nextseed;
    AtomicLong seed = this.seed;
    do {
        oldseed = seed.get();
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed));
    return (int)(nextseed >>> (48 - bits));
}
```

# 二、ThreadLocalRandom

1. current()

```java
static final ThreadLocalRandom instance = new ThreadLocalRandom();

public static ThreadLocalRandom current() {
    if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
        localInit();
    return instance;
}
```
current()首先查看当前线程中的PROBE（探针）是否初始化，如果是，则返回一个ThreadLocalRandom的单例；如果否，则调用localInit()进行初始化。

2. localInit()

```java
static final void localInit() {
    int p = probeGenerator.addAndGet(PROBE_INCREMENT);
    int probe = (p == 0) ? 1 : p; // skip 0
    long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
    Thread t = Thread.currentThread();
    UNSAFE.putLong(t, SEED, seed);
    UNSAFE.putInt(t, PROBE, probe);
}
```
localInit()生成probe和seed，并存入当前线程。probe和seed在Thread中的定义如下：

```java
@sun.misc.Contended("tlr")
int threadLocalRandomProbe;

@sun.misc.Contended("tlr")
long threadLocalRandomSeed;
```

3. nextInt()

```java
public int nextInt() {
    return mix32(nextSeed());
}

final long nextSeed() {
    Thread t; long r; // read and update per-thread seed
    UNSAFE.putLong(t = Thread.currentThread(), SEED,
                   r = UNSAFE.getLong(t, SEED) + GAMMA);
    return r;
}
```

由此可见，虽然ThreadLocalRandom是单例，所有线程共用一个，但是生成随机数的nextInt()却是使用各自线程中的seed，线程之间是相互隔离的。所以ThreadLocalRandom在高并发场景下的性能要优于Random。

4. 使用

错误使用：
```java
public static void main(String[] args) throws Exception{
	ThreadLocalRandom threadLocalRandom=ThreadLocalRandom.current();
	for(int i=1; i<=10; i++){
		new Thread(()->{
			System.out.println(Thread.currentThread().getName()+":"+threadLocalRandom.nextInt());
		},String.valueOf(i)).start();
	}
}
```
结果：

![](https://oscimg.oschina.net/oscnet/up-9078daf08af3d662b0c3017d503ee0837a4.png)

由于current()是在主线程中调用的，seed和probe初始化在了主线程中，而其他线程在调用nextInt时取到的seed和probe都为0，由于随机数生成算法都是固定的，所以也就生成了相同的随机数。

正确使用：

```java
public static void main(String[] args) throws Exception{
	for(int i=1; i<=10; i++){
		new Thread(()->{
			System.out.println(Thread.currentThread().getName()+":"+ThreadLocalRandom.current().nextInt());
		},String.valueOf(i)).start();
	}
}
```
结果：

![](https://oscimg.oschina.net/oscnet/up-fa5c6a1a8eb787522f9ac940b68e4baf94f.png)

一定要在各自的线程中初始化。

