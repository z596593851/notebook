![[Pasted image 20210720210058.png]]
![[Pasted image 20210720212418.png]]
![[Pasted image 20210720212751.png]]
DispatcherServletRegistrationConfiguration将DispatcherServlet和DispatcherServletRegistrationBean注入容器。
DispatcherServletRegistrationBean实现了ServletContextInitializer，并且把容器的DispatcherServlet包含进去。


包含关系：
MyTomcatWebServer->Tomcat->MyTomcatEmbeddedContext->MyTomcatStarter
->MyServletContextInitializer

调用关系亦是如此。当 MyTomcatWebServer 调用 tomcat.start() 时，tomcat调用MyTomcatStarter 的 onStartup方法，此时遍历 initializers 并调用其 onStartup ，

