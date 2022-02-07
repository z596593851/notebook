# Explain
type
-   const：使用主键或唯一索引，返回记录一定是1行记录的等值where查询
-   eq_ref：多表查询时使用主键进行关联
-   ref：没用主键或唯一索引，而是用普通索引，或者联合主键的部分主键的等值查询，有可能查出多条结果
-   range：使用索引，但是是范围查找
-   index，ALL：相当与全表查询

key：用到的索引

key_len：使用联合索引时，可以通过这个字段判断使用了联合索引的哪个部分

extra
- using index：覆盖索引，即索引中包含了所有要查找的字段，不用回表
- using index condition：查找的列不完全被索引覆盖，比如只用到了联合索引的前一部分
- using where：查询结果未被索引覆盖，也就是没走索引
- using temporary：使用了临时表，需要优化。比如distinct时没加索引，加索引后就可以优化成using index
- using filesort：使用外部排序而非索引排序，还是没加索引
- select tables optimized away：使用聚合函数（min，max）来访问存在索引的某个字段
