# redis的数据结构
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
int 编码：保存的是可以用 long 类型表示的整数值

embstr 编码：保存长度小于44字节的字符串（redis3.2版本之前是39字节，之后是44字节）
![[Pasted image 20220720205402.png]]

raw 编码：保存长度大于44字节的字符串（redis3.2版本之前是39字节，之后是44字节）
![[Pasted image 20220720205412.png]]

sds在分配内存时会提前申请额外的空间，这样当使用append进行字符串追加时，如果free充足，直接用即可，不用频繁的进行内存分配；如果free不足，会进行扩容，扩容后的长度是 （原有长度+需要分配的长度）x 2

由于如果用int来表示len，那么int的范围最大会到2^32-1，而并不是所有string都会存这么大的数据，而redis里string类型的数据占比又很大，会造成内存浪费，所以redis又定义了不同的sds来存放不同大小的string
![[Pasted image 20220803215642.png]]
![[Pasted image 20220803215625.png]]
![[Pasted image 20220803215953.png]]
# list
ziplist编码：
![[Pasted image 20220720205756.png]]

linkedlist编码
![[Pasted image 20220720205821.png]]

# hash
ziplist编码：
![[Pasted image 20220720210017.png]]

hashtable 编码：
![[Pasted image 20220720210032.png]]

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