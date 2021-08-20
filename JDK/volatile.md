**volatile的作用：**
-   保证可见性
-   禁止指令重排
-   不保证原子性

### 1.内存可见性
由于JVM运行程序的实体是线程，而每个线程创建时JVM都会为其创建一个工作内存（栈空间），工作内存是每个线程的私有数据区域，而java内存模型中规定所有变量都存储在主内存，主内存是共享内存区域，所有线程都可以访问，但线程对变量的操作（读取赋值等）必须在工作内存中进行，首先要将变量从主内存拷贝到自己的工作内存空间，然后对变量进行操作，操作完后再将变量写回主内存，不能直接操作主内存中的变量，各个线程中的工作内存中存储着主内存中的变量副本拷贝，因此不同的线程间无法访问对方的工作内存，线程间的通信（传值）必须通过主内存来完成。

volatile的内存可见性是通过内存屏障实现的：
-   Load：让缓存中的数据失效，重新从主内存加载数据
-   Store：数据对其他处理器可见（即：刷新到内存）

即当线程读取volatile修饰的数据时，不是直接读取工作内存中的数据，而是使缓存失效，直接从主存中读取；当线程重写volatile修饰的数据时，会将值直接刷新到主存中。

### 2.指令重排
单线程环境下，程序最终执行结果和代码顺序执行的结果一致。多线程环境下，为了提高性能，编译器和处理器常常会对指令做重排（重排时会考虑指令之间的数据依赖性），执行结果往往难以预料。而被volatile修饰的字段会避免指令重排。

真正线程安全的双重锁校验单例模式：加上volitale关键字禁止指令重排

```java
public class Singleton{
    private static volitale instance= null;
    private Singleton(){};
    public static Singleton getInstance(){
        if(instance!=null){
            Synchronized(Singleton.class){
                if(instance!=null){
                     instance=new Singleton();
                }
            }
        }
        return instance;
    }
}
```
