# 一、Redolog
redo log是InnoDB特有的，即引擎层的。为了减少数据落盘时的io次数以及随机读写的成本，引入了redo log。redo log是顺序写的，即WAL技术，Write-Ahead Logging，它的关键点是先写日志，再写磁盘。具体来说就是，当需要更新一条数据时，InnoDB引擎会先把记录写进redo log中，并更新内存，这个时候就算完成。同时InnoDB会在空闲的时候将这个记录写进磁盘。
redo log是固定大小，从头开始写入，写到尾部就又回到开头循环写。指针write pos记录当前位置，checkpoint是要擦除的位置，当两者之间有空余位置，即可继续更新，当满了后，就需要checkpoint擦除数据后继续更新。有了redo log，InnoDB就可以保证数据库即使发生异常重启，之前提交的记录也不会丢失（可以通过redolog恢复）。

### 参数设置（默认1）
- **innodb\_flush\_log\_at\_trx\_commit=0**
表示每隔一秒把redolog buffer刷到文件系统中(os buffer)去，并且调用文件系统的“flush”操作将缓存刷新到磁盘上去。也就是说一秒之前的日志都保存在日志缓冲区，也就是内存上，如果机器宕掉，可能丢失1秒的事务数据。

- **innodb\_flush\_log\_at\_trx\_commit=1**
表示在每次事务提交的时候，都把redolog buffer刷到文件系统中(os buffer)去，并且调用文件系统的“flush”操作将缓存刷新到磁盘上去。这样的话，数据库对IO的要求就非常高了，如果底层的硬件提供的IOPS比较差，那么MySQL数据库的并发很快就会由于硬件IO的问题而无法提升。

- **innodb\_flush\_log\_at\_trx\_commit=2**
表示在每次事务提交的时候会把log buffer刷到文件系统(os buffer)中去，但并不会立即刷写到磁盘。如果只是MySQL数据库挂掉了，由于文件系统没有问题，那么对应的事务数据并没有丢失。只有在数据库所在的主机操作系统损坏或者突然掉电的情况下，数据库的事务数据可能丢失1秒之类的事务数据。这样的好处，减少了事务数据丢失的概率，而对底层硬件的IO要求也没有那么高(log buffer写到文件系统中，一般只是从log buffer的内存转移的文件系统的内存缓存中，对底层IO没有压力)。

0是最不安全的，如果断电会丢失mysql缓存文件系统缓存中将近一秒的数据。  
1是最安全的，怎么都不会丢失数据，但是也是性能最差的。  
2是居于中间。


# 二、BinLog
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

# 三、Undo log
主要用来回滚已经提交的事务。

# 四、Redo log和Binlog的2pc
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
### 参数设置
sync\_binlog：是MySQL 的二进制日志（binary log）同步到磁盘的频率
- **sync\_binlog=0**，当事务提交之后，MySQL不做fsync之类的磁盘同步指令将binlog\_cache中的信息到磁盘，而让Filesystem自行决定什么时候来做同步，或者cache满了之后才同步到磁盘。这个是性能最好的。
- **sync\_binlog=1**，当每进行1次事务提交之后，MySQL将进行一次fsync之类的磁盘同步指令来将binlog\_cache中的数据强制写入磁盘。
- **sync\_binlog=n**，当每进行n次事务提交之后，MySQL将进行一次fsync之类的磁盘同步指令来将binlog\_cache中的数据强制写入磁盘。