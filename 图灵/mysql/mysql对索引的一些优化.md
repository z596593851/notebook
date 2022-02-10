# MRR
即多范围读取优化，当对二级索引进行范围查找时，如 name>'0' and name <'z'，虽然二级索引是有序的，但是它关联的聚簇索引却是无序的，需要进行多次随机io的回表操作，增加了io开销。而MRR会先查一部分二级索引，在内存中将这部分二级索引关联的聚簇索引进行排序后，再集中进行一次顺序io的回表，大大增加了效率。
![[Pasted image 20220129222255.png]]

# 索引合并
一般情况下mysql在一次查询中只会用到单个二级索引，特殊情况下会用到多个，称为索引合并。

## 交集合并
前提：
1. 二级索引必须是等值查询
2. 主键索引可以是范围查询
```shell
select * from table where name='a' and address='b'

select * from table where id<10 and name='a'
```
当二级索引的多个等值查询会分别查出过多结果时，比起先通过一个条件的结果回表查出结果集，再从结果集中过滤剩余条件，mysql会先在二级索引树中求出所有等值查询结果的id（主键索引）的交集，从而得出更小的结果集，再去回表。

## 并集合并
前提：
1. 二级索引必须是等值查询
2. 主键索引可以是范围查询
3. 搜索条件的部分使用交集合并得到的集合，与其他方式得到的集合，取并集
```shell
select * from table where sex=1 or (name='a' and address='b')
```
(name='a' and address='b')使用交集合并的结果，与sex=1进行并集合并，最后进行回表。

## 排序并集合并
并集合并要求是等值查询，当不是等值查询时，如：
```shell
select * from table where name>'a' or address<'b'
```
mysql会将 name>'a' 的id进行排序，然后将 address<'b' 的id排序，最后将二者合并进行回表。

当满足索引合并的条件时也不一定会执行，仍然取决于优化器的判断。但当进行了索引合并，会在执行计划的extra中体现出来：Using intersect(...)、Using union(...)、Using sort_union

其实完全没有必要进行索引合并，只需建立联合索引就可以更高效的进行查询。

# 索引下推
![[Pasted image 20220127173920.png]]
按照常识，这条sql用到的索引会止步在name，但是执行计划却显示用到了所有索引。索引下推是5.6引入的，在这之前执行这条sql时，会先拿到所有 LiLei% 的结果的主键id，再去回表拿到完整的数据，最后再去比对age和position；而5.6以后会直接在 LiLei% 结果集的基础上直接比对age和position。