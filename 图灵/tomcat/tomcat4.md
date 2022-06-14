WebAppClassLoader
每个context实例都维护一个自己的WebAppClassLoader加载器，由于同名的类被不同的类加载器加载，也是不同的类，由此做到不同实例内同名类的隔离

重写loadclass方法：
跳过app加载器，先用ext和bootstrapt加载器加载，以防核心类被破坏；ext和boot加载不了的，再回来用WebAppClassLoader加载，比如每个容器自己的类库；WebAppClassLoader加载不了的，最后再用app加载器去加载。
![[Pasted image 20220608214447.png]]
![[Pasted image 20220608214615.png]]
![[Pasted image 20220608214813.png]]
其他三个是空的
![[Pasted image 20220608215138.png]]
![[Pasted image 20220608215201.png]]
comment：
![[Pasted image 20220608215356.png]]
WebAppClassLoader：WEB-INF/lib和WEB-INF/classes

线程上下文加载器：
![[Pasted image 20220608220041.png]]
![[Pasted image 20220608215944.png]]

热加载：修改类
context对象先stop再start

热部署：添加应用
Host组件

热加载是靠后台线程通过递归调用实现的，达到一个while(true)的效果，在ContainerBase中实现。为了防止该线程挂掉，会有一个监视器线程去检查，如果挂了就重新启动一个线程。为了防止这个监视器线程也挂掉，会有定时任务去新建这个定时器线程。

热部署则是直接由定时任务去监控实例文件。