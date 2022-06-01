tomcat=http服务器+servlet容器

![[Pasted image 20220531220145.png]]
![[Pasted image 20220531220224.png]]
![[Pasted image 20220531220319.png]]

所以tomcat要实现两个功能：
- 处理socket连接，负责网络字节流与request和response对象的转化
- 加载和管理servlet，以及具体处理request请求

分别由connector和container负责

connector要实现的功能：
- 网络通信——EndPoint
- 序列化反序列化协议解析——Processor
- 适配servlet的request和response——Adapter

tomcat支持的IO模型有：BIO、NIO、AIO、APR
tomcat支持的网络协议有：HTTP1.1、AJP、HTTP2.0


![[Pasted image 20220531222208.png]]
![[Pasted image 20220531222551.png]]
tomcat的reactor模型
![[Pasted image 20220531223056.png]]

容器的管理用组合模式
![[Pasted image 20220531223713.png]]

请求定位servlet的过程：
![[Pasted image 20220531223857.png]]
Adaptor将协议转换成HttpRequest  ：
![[Pasted image 20220531230655.png]]
然后connector向上获取service，再向下获取container（即engine），将request传下去：
![[Pasted image 20220531230900.png]]
![[Pasted image 20220531225410.png]]



![[Pasted image 20220531233031.png]]
service extends lifecycle
![[Pasted image 20220531233121.png]] 
![[Pasted image 20220531233318.png]]
![[Pasted image 20220531232024.png]]
![[Pasted image 20220531232659.png]]
![[Pasted image 20220531232427.png]]

启动
![[Pasted image 20220601221651.png]]
![[Pasted image 20220601221703.png]]
![[Pasted image 20220601221742.png]]
![[Pasted image 20220601221822.png]]
![[Pasted image 20220601222050.png]]