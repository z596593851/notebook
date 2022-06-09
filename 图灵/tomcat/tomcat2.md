# 五种io模型
![[Pasted image 20220601233543.png]]
阻塞与否指的是第一阶段，同步与否指的是第二阶段

bio是同步阻塞io，nio是同步非阻塞io，aio是异步非阻塞io
 
## tomcat支持的io模型
![[Pasted image 20220606211419.png]]

![[Pasted image 20220606211615.png]]
异步io（aio）在linux的实现是epoll，但是需要写回调函数，性能跟java nio的selector差不多且容易出错；在windows的实现是ARP，性能较好。proactor模型

单reactor模型
![[Pasted image 20220606213028.png]]

单reactor+线程池
![[Pasted image 20220606213228.png]]

多reactor（主从）
![[Pasted image 20220606213328.png]]

tomcat对线程池的扩展
核心池可以预先启动
![[Pasted image 20220606215836.png]]
![[Pasted image 20220606215717.png]]
![[Pasted image 20220606215651.png]]

![[Pasted image 20220606220105.png]]
![[Pasted image 20220606220141.png]]

![[Pasted image 20220609213912.png]]
socketChannel这种大对象可以重复利用

tomcat在处理aio模型时使用的是proactor模型，即不像nio那样使用线程池，而是不断的调用accpet来递归调用处理新连接，模仿bio和nio的while(true)
为了防止达到连接上限，使用LimitLatch（和countdowmlatch一样使用aqs，只不过是反着的）限流

即使tomcat达到1000个连接，再有连接会被限流，但是还会和linux内核建立连接，此时linux还会多缓存100个连接
![[Pasted image 20220608235255.png]]

nio的请求连接过程
![[Pasted image 20220608235523.png]]

aio的请求连接过程
![[Pasted image 20220608235547.png]]