原子性：undo日志，记录事务内数据的变更，以便事务的回滚
一致性，
隔离性：锁+mvcc
持久性：redo日志

脏页为什么不直接落盘：
可能只改了几条，但会导致一整页的刷新；
随机io性能差

# BinLog
binlog是server层的，主要用来做主从同步，有以下两种方式：
- Statement level（基于SQL语句的复制，默认）
每一条会修改数据的sql语句会记录到binlog中。
优点：不需要记录每一条SQL语句与每行的数据变化，这样子binlog的日志也会比较少，减少了磁盘IO，提高性能。
缺点：在某些情况下会导致master-slave中的数据不一致(如sleep()函数， last\_insert\_id()，以及user-defined functions(udf)等会出现问题)

- Row level（基于行的复制）
不记录每一条SQL语句的上下文信息，仅需记录哪条数据被修改了，修改成了什么样子了。
优点：不会出现某些特定情况下的存储过程、或function、或trigger的调用和触 发无法被正确复制的问题。
缺点：会产生大量的日志，尤其是alter table的时候会让日志暴涨。

无论是增量备份还是主从复制，都是需要开启mysql-binlog日志，最好跟数据目录设置到不同的磁盘分区，可以降低io等待，提升性能；并且在磁盘故障的时候可以利用mysql-binlog恢复数据。

## 参数设置
sync\_binlog：是MySQL 的二进制日志（binary log）同步到磁盘的频率
- **sync\_binlog=0**，当事务提交之后，MySQL不做fsync之类的磁盘同步指令将binlog\_cache中的信息到磁盘，而让Filesystem自行决定什么时候来做同步，或者cache满了之后才同步到磁盘。这个是性能最好的。
- **sync\_binlog=1**，当每进行1次事务提交之后，MySQL将进行一次fsync之类的磁盘同步指令来将binlog\_cache中的数据强制写入磁盘。
- **sync\_binlog=n**，当每进行n次事务提交之后，MySQL将进行一次fsync之类的磁盘同步指令来将binlog\_cache中的数据强制写入磁盘。

# Undo log
记录了当前数据所处事务的事务id，以及当前数据对应的undo日志，所以可以根据undo日志在事务失败时回滚当前事务内的所有操作。
![[Pasted image 20220202224222.png]]

# Redo Log
Redo Log的作用：当Mysql宕机，BufferPool中没有刷盘的数据会从Redo Log中恢复。为什么BufferPool不直接同步刷盘？因为磁盘读取是随机操作，效率低，所以将刷盘动作设计成异步；同时同步向Redo Log中顺序写，防止数据丢失。顺序写的效率也远远高于随机写。

redo日志格式
![[Pasted image 20220202213017.png]]
type：有53种
space id：表空间id
page number：页号

## redo log buffer的刷盘时机
- **innodb\_flush\_log\_at\_trx\_commit=0**
表示每隔一秒把redolog buffer刷到文件系统中(os buffer)去，并且调用文件系统的“flush”操作将缓存刷新到磁盘上去。也就是说一秒之前的日志都保存在日志缓冲区，也就是内存上，如果机器宕掉，可能丢失1秒的事务数据。

- **innodb\_flush\_log\_at\_trx\_commit=1（默认）**
表示在每次事务提交的时候，都把redolog buffer刷到文件系统中(os buffer)去，并且调用文件系统的“flush”操作将缓存刷新到磁盘上去。这样的话，数据库对IO的要求就非常高了，如果底层的硬件提供的IOPS比较差，那么MySQL数据库的并发很快就会由于硬件IO的问题而无法提升。

- **innodb\_flush\_log\_at\_trx\_commit=2**
表示在每次事务提交的时候会把log buffer刷到文件系统(os buffer)中去，但并不会立即刷写到磁盘。如果只是MySQL数据库挂掉了，由于文件系统没有问题，那么对应的事务数据并没有丢失。只有在数据库所在的主机操作系统损坏或者突然掉电的情况下，数据库的事务数据可能丢失1秒之类的事务数据。这样的好处，减少了事务数据丢失的概率，而对底层硬件的IO要求也没有那么高(log buffer写到文件系统中，一般只是从log buffer的内存转移的文件系统的内存缓存中，对底层IO没有压力)。

- 另外，容量超过一半，或者服务器关闭时也会触发刷盘

0是最不安全的，如果断电会丢失mysql缓存文件系统缓存中将近一秒的数据。  
1是最安全的，怎么都不会丢失数据，但是也是性能最差的。  
2是居于中间。

redo log日志在磁盘上有两个文件，ib_logfile0和ib_logfile1，默认每个48M，都写满了就重新覆盖写。

Log Sequence Number：每条redo日志都有对应的lsn，mysql可以根据lsn得知当前redo日志的数量，以及哪些redo日志已经刷盘。
![[Pasted image 20220203003707.png]]

## insert操作
当插入一条数据时，会同时更新聚簇索引和二级索引，但是只用主键id产生一条undo日志就可以了，不需要再用二级索引产生undo日志，因为二级索引里也记录着主键id。

## delete操作
删除时，先将delete_mask置为1，等事务提交后，再将记录从链表删除，添加到page_free中。
![[Pasted image 20220202230244.png]]
![[Pasted image 20220202230258.png]]
![[Pasted image 20220202230313.png]]

## update操作
不更新主键：
1.如果更新操作不会改变变长字段的大小，那么就地更新
2.如果改变了，则先删掉旧记录（直接从链表中转移到page_free中），再插入新记录

更新主键：
1.将旧记录进行delete操作
2.创建新纪录

# undo log、redo log的顺序
![[Pasted image 20220203001440.png]]

# mysql数据恢复
mysql异常关闭，如果事务还未提交，则使用undo log对事务进行回滚；如果事务已经提交，则使用redo log恢复数据。

为何不用binlog恢复？因为binlog属于逻辑日志，记录了某条sql对某行记录的影响，是偏业务角度的日志，主要用来人工恢复数据，无法根据binlog判断某些数据是否真的落盘了；而redo log是偏物理层面的日志，可以反映出某条数据是否落盘

# Redo log和Binlog的2pc
如果redo log和binlog不一致，会导致发生宕机后主从数据的不一致（主设备从redo log恢复，从设备从binlog恢复）。
![[Pasted image 20210526205328.png]]
2pc的过程：
- prepare阶段，binlog什么也不做，redolog落盘，并把回滚状态置为prepare状态
- commit阶段，binlog落盘，成功后事务管理器把redolog的回滚状态置为commit状态

如果发生宕机，若发现redo log处于commit状态，说明双方数据处于一致状态，则提交redolog；若发现redo log处于prepare状态，以redolog中的xid与binlog中的xid进行比较，如果xid在binlog中则提交，否则回滚。（xid是用来标记log中发生数据更改的第几个event）。

### 组提交
- 在没有开启binlog时：
Redo log的刷盘操作（指的是从redolog buffer中刷到redolog 磁盘上）将会是最终影响MySQL TPS的瓶颈所在。为了缓解这一问题，MySQL使用了组提交，将多个刷盘操作合并成一个，如果说10个事务依次排队刷盘的时间成本是10，那么将这10个事务一次性一起刷盘的时间成本则近似于1。
- 当开启binlog时：
为了保证Redo log和binlog的数据一致性，MySQL使用了二阶段提交，由binlog作为事务的协调者。而引入二阶段提交使得binlog又成为了性能瓶颈，先前的Redo log 组提交 也成了摆设。为了再次缓解这一问题，MySQL增加了binlog的组提交，目的同样是将binlog的多个刷盘操作合并成一个，结合Redo log本身已经实现的 组提交，分为三个阶段(Flush 阶段、Sync 阶段、Commit 阶段)完成binlog 组提交，最大化每次刷盘的收益，弱化磁盘瓶颈，提高性能。
![[Pasted image 20210526211843.png]]
![[Pasted image 20210526211859.png]]
![[Pasted image 20210526212116.png]]
![[Pasted image 20210526212134.png]]
![[Pasted image 20210526212147.png]]
![[Pasted image 20210526212156.png]]
![[Pasted image 20210526212206.png]]
![[Pasted image 20210526212216.png]]
![[Pasted image 20210526212226.png]]
![[Pasted image 20210526212234.png]]
![[Pasted image 20210526212243.png]]
![[Pasted image 20210526212252.png]]
![[Pasted image 20210526212300.png]]
![[Pasted image 20210526212308.png]]
在MySQL中每个阶段都有一个队列，每个队列都有一把锁保护，第一个进入队列的事务会成为leader，leader领导所在队列的所有事务，全权负责整队的操作，完成后通知队内其他事务操作结束。

**Flush 阶段 (图中第一个渡口)**
-   首先获取队列中的事务组
-   将Redo log中prepare阶段的数据刷盘(图中Flush Redo log)
-   将binlog数据写入文件系统缓冲区（os cache，而不是磁盘），并不能保证数据库崩溃时binlog不丢失 (图中Write binlog)
-   Flush阶段队列的作用是提供了Redo log的组提交
-   如果在这一步完成后数据库崩溃，由于协调者binlog中不保证有该组事务的记录，所以MySQL可能会在重启后回滚该组事务
    

**Sync 阶段 (图中第二个渡口)**
-   这里为了增加一组事务中的事务数量，提高刷盘收益，MySQL使用两个参数控制获取队列事务组的时机：
    binlog\_group\_commit\_sync\_delay=N：在等待N μs后，开始事务刷盘(图中Sync binlog)
	binlog\_group\_commit\_sync\_no\_delay\_count=N：如果队列中的事务数达到N个，就忽视binlog\_group\_commit\_sync\_delay的设置，直接开始刷盘(图中Sync binlog)
-   Sync阶段队列的作用是支持binlog的组提交 
-   如果在这一步完成后数据库崩溃，由于协调者binlog中已经有了事务记录，MySQL会在重启后通过Flush 阶段中Redo log刷盘的数据继续进行事务的提交

**Commit 阶段 (图中第三个渡口)**
-   首先获取队列中的事务组
-   依次将Redo log中已经prepare的事务在引擎层提交(图中InnoDB Commit)
-   Commit阶段不用刷盘，如上所述，Flush阶段中的Redo log刷盘已经足够保证数据库崩溃时的数据安全了
-   Commit阶段队列的作用是承接Sync阶段的事务，完成最后的引擎提交，使得Sync可以尽早的处理下一组事务，最大化组提交的效率

第一个渡口是prepare阶段，后两个渡口是commit阶段。