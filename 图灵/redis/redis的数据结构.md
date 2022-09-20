# redis的数据结构
redis db：一共16个：
![[Pasted image 20220811215114.png]]
每个dict就是一个HashTable，redis里所有的k-v都通过hash算法散列后存在HashTabel中。

![[Pasted image 20220811215518.png]]

dicht就是hashtable，有两个，rehash的时候会用到第二个
![[Pasted image 20220814173603.png]]

table是数组的指针，size是数组长度，初始容量4，扩容时x2，渐进式rehash，当数据量和数组长度为1:1时触发扩容

dictEntry：hashtable数组的存储元素。key指向key对象，即sds对象；val指向value对象，可能是string、list、hash、set等，这些对象由redisObject表示。
![[Pasted image 20220814172747.png]]


redis的五大数据结构：string、list、hash、set、zset，每种数据结构都对应一个redisObject：
```c
typedef struct redisObject{
     //类型
     unsigned type:4;

     //编码
     unsigned encoding:4;

     //指向底层数据结构的指针
     void *ptr;

     //引用计数
     int refcount;

     //记录最后一次被程序访问的时间`
     unsigned lru:22;
}robj
```
type就是上述5种

每种数据结构都有不同的编码方式，由encoding表示：
![[Pasted image 20220720204837.png]]

prt 指针指向对象底层的数据结构

# string
int 编码：保存的是<2^64-1的整数，并且value会直接存在ptr里，这样ptr就不用再指向别的地方。

embstr 编码：保存长度小于44字节的字符串（redis3.2版本之前是39字节，之后是44字节）
![[Pasted image 20220720205402.png]]
64位系统的一个缓存行大小是64byte，一个redisObject的大小是16byte，剩下48byte可以放下sdshdr8类型的sds（sdshdr5只能放下长度是31的string），而一个sdshdr8对象本身要占4byte，所以一个长度为44的string正好可以占用一个缓存行。embstr开辟了一个连续的内存空间，小于等于这个长度的string只需要一次cpu io就可以读出来。
![[Pasted image 20220814180159.png]]
![[Pasted image 20220814180600.png]]

raw 编码：保存长度大于44字节的字符串（redis3.2版本之前是39字节，之后是44字节）
![[Pasted image 20220720205412.png]]

sds在分配内存时会提前申请额外的空间，这样当使用append进行字符串追加时，如果free充足，直接用即可，不用频繁的进行内存分配；如果free不足，会进行扩容，扩容后的长度是 新长度（原有长度+需要分配的长度）x 2

由于如果用int来表示len，那么int的范围最大会到2^32-1，而并不是所有string都会存这么大的数据，而redis里string类型的数据占比又很大，会造成内存浪费，所以redis又定义了不同的sds来存放不同大小的string
![[Pasted image 20220803215642.png]]
![[Pasted image 20220803215625.png]]
![[Pasted image 20220803215953.png]]
# list
如果list中元素较小，就会出现next、pre指针比元素大的情况（指针8byte），导致内存的浪费，这时redis会使用连续的内存空间来存放list元素
ziplist编码：
![[Pasted image 20220720205756.png]]

linkedlist编码
![[Pasted image 20220720205821.png]]
在最新的redis中，数据量大时会使用分组的连续空间存储，不同组之间使用链表连接：
![[Pasted image 20220814192232.png]]
以下参数来控制分裂条件和压缩情况：
![[Pasted image 20220814192501.png]]

# hash
ziplist编码：
![[Pasted image 20220720210017.png]]

hashtable 编码：
![[Pasted image 20220720210032.png]]

![[Pasted image 20220814192823.png]]

当hash是ziplist编码时，元素是有序的；是hashtable时，是无序的。

# set
intset 编码：
![[Pasted image 20220720210237.png]]

hashtable 编码：
![[Pasted image 20220720210254.png]]

# zset
ziplist编码：
![[Pasted image 20220720210347.png]]

skiplist 编码：使用 zet 结构作为底层实现，一个 zset 结构同时包含一个字典和一个跳跃表
```c
typedef struct zset{
     //跳跃表`
     zskiplist *zsl;

     //字典
     dict *dice;

} zset
```