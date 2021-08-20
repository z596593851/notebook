# 数据库的隔离级别

| 隔离级别 | 脏读（Dirty Read） | 不可重复读（NonRepeatable Read） | 幻读（Phantom Read） |
| --- | --- | --- | --- |
| 未提交读（Read uncommitted） | 可能 | 可能 | 可能 |
| 已提交读（Read committed） | 不可能 | 可能 | 可能 |
| 可重复读（Repeatable read） | 不可能 | 不可能 | 可能 |
| 可串行化（Serializable ） | 不可能 | 不可能 | 不可能 |


* 不可重复读：是指在数据库访问中，一个事务范围内两个相同的查询却返回了不同数据。这是由于查询时系统中其他事务修改的提交而引起的。比如事务T1读取某一数据，事务T2读取并修改了该数据，T1为了对读取值进行检验而再次读取该数据，便得到了不同的结果。  
* 幻读：是指事务A读取与搜索条件相匹配的若干行。事务B以插入或删除行等方式来修改事务A的结果集，事务A再次以相同条件搜索时发现结果集数量发生了变化。  

大致的区别在于不可重复读是由于另一个事务对数据的更改所造成的，而幻读是由于另一个事务插入或删除引起的。
幻读：
![[Pasted image 20210518181836.png]]

# MVCC
MVCC其实是通过undo log来实现的：每行记录的后面保存了两个隐藏的列,DB_TRX_ID(数据行的版本号)和DB_ROLL_PT(删除版本号)。

* select：

  1. 只查找版本早于当前事务版本的数据行，这样可以确保事务读取的行，要么是在事务开始前已经存在的，要么是事务自身插入或者修改过的；
  2. 行的删除版本要么未定义，要么大于当前事务版本号。这可以确保事务读取到的行，在事务开始之前未被删除。

* Insert：

    InnoDB为新插入的每一行保存当前系统版本号作为数据行版本号。

* delete：

    InnoDB为删除的每一行保存当前系统版本号作为行删除版本号。

* update：

    InnoDB插入一条新记录，保存当前系统版本号作为数据行版本号，同时保存当前系统版本号到原来的行作为删除版本号。


| Id   | name   | DB_TRX_ID(数据行的版本号) | DB_ROLL_PT(删除版本号) |
| ---- | ------ | ------------------------- | ---------------------- |
| 1    | lisi   | 1                         | 4                      |
| 2    | yunzhi | 1                         | 4                      |
| 1    | yunzhi | 4                         | null                   |
| 2    | yunzhi | 4                         | null                   |
| 3    | wangwu | 5                         | 4                      |
| 3    | yunzhi | 4                         | null                   |



1：insert into user values(NULL,'lisi');

​	insert into user values(NULL,'yunzhi');

4：select * from user; 

​	update user set name='yunzhi'; # 更新所有数据

​	select * from user;

5：insert into user values(NULL,'wangwu');

MySQL可重复读的隔离级别中并不是完全解决了幻读的问题，而是解决了读数据情况下的幻读问题。而对于修改的操作依旧存在幻读问题，就是说MVCC对于幻读的解决是不彻底的。

解决幻读有两个办法：

- 使用串行化读的隔离级别
- MVCC+next-key locks：next-key locks由record locks(索引加锁，即行锁) 和 gap locks(间隙锁，每次锁住的不光是需要使用的数据，还会锁住这些数据附近的数据)

mvcc：https://segmentfault.com/a/1190000022858460

gap锁：https://blog.csdn.net/u010841296/article/details/87907713