五种io模型
![[Pasted image 20220601233543.png]]
bio是同步阻塞io，nio是同步非阻塞io，aio是异步非阻塞io

异步io（aio）在linux的实现是epoll，但是需要写回调函数，性能跟java nio的selector差不多且容易出错；在windows的实现是ARP，性能较好。
 
tomcat支持的io模型
![[Pasted image 20220606211419.png]]

![[Pasted image 20220606211615.png]]

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