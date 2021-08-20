# 一、MyISAM和InnoDB

### MyISAM:

MyISAM的叶节点的索引和数据是分开存储的，即数据的逻辑顺序和索引的逻辑顺序一致，数据的物理顺序没有规则：

![](https://img-blog.csdnimg.cn/20190307212728579.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psMXpsMnpsMw==,size_16,color_FFFFFF,t_70)

上图是以Col1为聚集索引（即主键索引），如果给Col2再加一个非聚集索引（二级索引）：

![](https://img-blog.csdnimg.cn/20190307212747364.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psMXpsMnpsMw==,size_16,color_FFFFFF,t_70)

MyISAM的聚集索引和非聚集索引除了对唯一性的约束以外没什么区别。

### InnoDB：

#### 聚集索引

InnoDB的叶节点索引是和数据存在一起的，即数据行的物理顺序与索引的逻辑顺序相同。由于表里的数据只能按照一个维度进行划分，所以所以一个表中只能拥有一个聚集索引。

![](https://img-blog.csdnimg.cn/20190307212854761.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psMXpsMnpsMw==,size_16,color_FFFFFF,t_70)

一个表就像是我们以前用的新华字典，聚集索引就像是拼音目录，每个字存放的页码就是我们的数据物理地址，我们如果要查询一个“哇”字，我们只需要查询“哇”字对应在新华字典拼音目录对应的页码，就可以查询到对应的“哇”字所在的位置，而拼音目录对应的A-Z的字顺序，和新华字典实际存储的字的顺序A-Z也是一样的，如果我们中文新出了一个字，拼音开头第一个是B，那么他插入的时候也要按照拼音目录顺序插入到A字的后面 ；而非聚集索引就像是笔画目录，笔画相近的字所在的位置可能就相距甚远了。

下表的聚集索引是id，可以看到数据的物理地址顺序和id的排列顺序相近。

表一：

![](https://oscimg.oschina.net/oscnet/up-4bd86af9d6de5cde4a2ecbf801274208742.png)

InnoDB在建表时要指定聚集索引，如果不指定，系统会自动创建一个隐含列作为表的聚集索引。

### 非聚集索引

非聚集索引中索引的逻辑顺序与磁盘上行的物理存储顺序不同，就类似于MyISAM。一个表中可以拥有多个非聚集索引， 普通索引，唯一索引，全文索引都属于非聚集索引。

不得不提的是，InnoDB的非聚集索引的叶子节点存放的是当前非聚集索引覆盖列的数据，以及聚集索引覆盖列的数据：

  ![](https://img-blog.csdnimg.cn/20190307212917784.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psMXpsMnpsMw==,size_16,color_FFFFFF,t_70)

所以当查询的列包含了指定的非聚集索引以及非聚集索引覆盖列以外的列时，会导致二次查找。如表一，id为聚集索引，username为非聚集索引，当使用以下sql时，直接可以从非聚集索引的节点中获取到查询列的数据：

```sql
select id, username from t1 where username = '小明'
select username from t1 where username = '小明'
```

但是使用以下语句进行查询，就需要二次的查询去获取原数据行的score：

```sql
select username, score from t1 where username = '小明'
```

# 二、复合主键、联合主键

## 复合主键（复合索引）

一张表只能有一个主键，但根据需要，我们可以设置多个字段同时为主键，这就叫做复合主键。

```sql
CREATE TABLE IF NOT EXIST student(
Name VARCHAR (30 ) COMMENT '姓名',
Age INT(30)  COMMENT '年龄',
PRIMARY KEY(Name,Age)
);
```

以上信息如果用姓名或年龄来确定都可能出现同名和同龄的情况，但是加入把他们都设置成主键的话，意味着既要同龄也要同名，这张情况就很少了，所以以这两个字段作为复合主键来使用。

复合索引的查询要符合最左匹配，就类似于多级的排序，只有当之前的字段有序时，之后的字段才能保证有序。

![](https://img-blog.csdnimg.cn/20190307212938450.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psMXpsMnpsMw==,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20190307212954417.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3psMXpsMnpsMw==,size_16,color_FFFFFF,t_70)

对于`select * from employees.titles where title='1'`是不能用到索引的，因为它不能用到上面的索引，和第一节点进行比较时，没有empno这个字段的值，不能确定到底该去左子树还是右子树继续进行查询。

## 联合主键

联合主键其实就是中间表，在多对多模型里，需要两个表中的主键组成联合主键，这样就可以连表查询查到两个表中的每个数据。

学生表：

```sql
create table student(
id mediumint  auto_increment comment '主键id',
name varchar(30) comment '姓名',
age smallint comment '年龄',
primary key(id)
)
engine = myisam,
charset = utf8,
comment = '学生'
```

课程表：

```sql
create table course(
id mediumint  auto_increment comment '主键id',
name varchar(30) comment '课程名称',
primary key(id)
)
engine = myisam,
charset = utf8,
comment = '课程'
```

学生课程表：

```sql
create table IF NOT EXISTS stu_cour(
id mediumint  auto_increment comment '主键id',
stu_id mediumint comment '学生表id',
cour_id mediumint comment '课程表id',
primary key(id)
)
engine = myisam,
charset = utf8,
comment = '学生课程表'
```

# 三、表锁、行锁

## 表锁

-   开销小，加锁快，不会出现死锁，锁定粒度大，发生锁冲突概率高，并发度低。
-   MyISAM只有表锁，表锁用的是读写锁（共享、独占锁），即读读共享，读写互斥。
-   Mysql认为写请求一般比读请求重要，即使读请求先到等待队列，写请求后到，也是写请求优先执行。因此，MyISAM表不适合于有大量更新操作和查询操作的应用，因为大量更新操作会造成查询操作长时间阻塞。（可以使用指令给予读请求优先的权利。）

## 行锁

-   开销大，加锁慢，会出现死锁，锁定粒度小，发生锁冲突概率低，并发度高
-   InnoDB行锁实现是通过索引上的索引项加锁实现的，意味着：只有通过索引条件检索数据，InnoDB才会使用行锁，否则使用表锁。
-   如果索引值不唯一，也会由行锁升级为表锁。
-   即时使用了索引，但是存在需要全盘扫描的列，也会升级为表锁。

