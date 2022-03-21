![[Pasted image 20220316214938.png]]
要想整合mybatis，就必须把mybatis针对mapper生成的代理对象注册到spring容器中为它所用

![[Pasted image 20220316221337.png]]
![[Pasted image 20220316221756.png]]

![[Pasted image 20220316233706.png]]
这么做的目的是，原生的sqlsession是线程不安全的，但mybatis-spring的sqlsessionTemplate是线程安全的，原理是利用了threadLocal让每个线程有自己的sqlsession。当不开启spring的事务时，不会存入threadlocal，每次执行sql都会创建新的sqlsession，导致mybatis的一级缓存失效。

![[Pasted image 20220317000339.png]]
新版的mybatis-spring可以不使用@MapperScan 而是注入一个MapperScanConfiguration


![[Pasted image 20220317002208.png]]
![[Pasted image 20220317002220.png]]
spring整合mybatis后，用的tx就不再是mybatis自己的tx对象了，而是整合了spring事务的，spring-mybatis自己实现的tx事务管理器。spring-mybatis利用自己的factoryBean，向容器中注入各个mapper的代理对象（不是mybatis的代理对象），这个代理对象在执行sql时又会使用spring-mybatis实现的sqlsessionTemplate，再结合spring的事务管理器等等spring自己的对象，最终去执行mybatis的原生逻辑（DefultSqlsession.selectone等）。