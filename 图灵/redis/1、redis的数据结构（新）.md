# redis的数据结构
![[Pasted image 20220819173953.png]]

dicht就是hashTable，有两个，rehash的时候会用到第二个。

dicht中的size是数组长度，初始容量4，扩容时x2，当数据量和数组长度为1:1时触发扩容。扩容时采用渐进式rehash，即为了保证可用性，在新老hashTable拷贝时不是一次性拷贝完成，而是逐步进行。在这个过程中取值时，先从旧hashTable中取，如果取不到，再从新hashTable中取。

redis的五大数据结构string、list、hash、set、zset，每种数据结构都对应一个redisObject，type就是上述5种。

每种数据结构都有不同的编码方式，由encoding表示：
![[Pasted image 20220720204837.png]]

ptr 指针指向对象底层的数据结构。

# string
int 编码：保存的是<2^64-1的整数，并且value会直接存在ptr里，这样ptr就不用再指向别的地方。

embstr 编码：保存长度小于44字节的字符串（redis3.2版本之前是39字节，之后是44字节）
![[Pasted image 20220720205402.png]]
64位系统的一个缓存行大小是64byte，一个redisObject的大小是16byte，剩下48byte可以放下sdshdr8类型的sds（sdshdr5只能放下长度是31的string，而sdshdr8能放下的string长度范围是0-255，包含48），而一个sdshdr8对象本身要占4byte，所以一个长度为44的string正好可以占用一个缓存行。embstr开辟了一个连续的内存空间，小于等于这个长度的string只需要一次cpu io就可以读出来。
![[Pasted image 20220819162046.png]]

raw 编码：保存长度大于44字节的字符串（redis3.2版本之前是39字节，之后是44字节）
![[Pasted image 20220720205412.png]]

sds在分配内存时会提前申请额外的空间，这样当使用append进行字符串追加时，如果free充足，直接用即可，不用频繁的进行内存分配；如果free不足，会进行扩容，扩容后的长度是 新长度（原有长度+需要分配的长度）x 2

由于如果用int来表示len，那么int的范围最大会到2^32-1，而并不是所有string都会存这么大的数据，而redis里string类型的数据占比又很大，会造成内存浪费，所以redis又定义了不同的sds来存放不同大小的string
![[Pasted image 20220803215642.png]]
![[Pasted image 20220803215625.png]]
![[Pasted image 20220803215953.png]]
# list
如果list中元素较小，就会出现next、pre指针比元素大的情况（指针8byte），导致内存的浪费，这时redis会使用连续的内存空间来存放list元素，即ziplist编码：
![[Pasted image 20220720205756.png]]

linkedlist编码
![[Pasted image 20220720205821.png]]
在最新的redis中，数据量大时会使用分组的连续空间存储，不同组之间使用链表连接：
![[Pasted image 20220819181342.png]]
以下参数来控制分裂条件和压缩情况：
![[Pasted image 20220819181515.png]]

# hash
ziplist编码：
![[Pasted image 20220720210017.png]]

hashtable 编码：
![[Pasted image 20220720210032.png]]

![[Pasted image 20220819180930.png]]

当hash是ziplist编码时，元素是有序的；是hashtable时，是无序的。

# set
intset 编码：
![[Pasted image 20220720210237.png]]

hashtable 编码：
![[Pasted image 20220720210254.png]]

# zset
ziplist编码：
![[Pasted image 20220720210347.png]]
![[Pasted image 20220815195430.png]]

skiplist 编码：使用 zet 结构作为底层实现，一个 zset 结构同时包含一个字典和一个跳跃表
```c
typedef struct zset{
     //跳跃表`
     zskiplist *zsl;

     //字典
     dict *dice;

} zset
```

跳表：
![[Pasted image 20220815200440.png]]
![[Pasted image 20220815204209.png]]
![[Pasted image 20220815212412.png]]
插入数据时，从最高层（level字段存储了最高层）开始找，找到合适的位置时，通过一种随机算法生成层高（生成高层的概率低，低层的概率高），在插入节点后维护好每一层的层级指针关系。