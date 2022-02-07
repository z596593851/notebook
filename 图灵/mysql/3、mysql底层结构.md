![[Pasted image 20220127165939.png]]

缓存：server层的缓存会用sql为id，查询结果为value进行缓存，但是每当表有更新，就会删除所有缓存。

binlog： binlog在server层，有Statement level和Row level两种模式，前者会记录所有执行过的sql，后者则不关心sql，而是记录数据的前后变化。