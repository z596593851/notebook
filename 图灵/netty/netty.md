为什么端口号有65535个？因为报文头里有两个字节存储端口号，2^16=65536，但是端口号0表示所有端口，所以有65535个

![[Pasted image 20220816215719.png]]

![[Pasted image 20220816221501.png]]

为什么需要三次握手？因为客户端和服务端都需要确认对方收到了自己发出的序列号，至少需要三次通信。
为什么需要四次挥手？因为客户端断开连接后，服务端处理完最后的工作，需要通知客户端。
time_wait状态：当四次挥手时，客户端最后一次发送ack后，会处于time_wait状态，等待2msl之后才会关闭连接。这是因为，如果立刻就关闭了连接，如果这个ack服务器没有收到，会再次发送关闭连接的请求，这时客户端已经关闭，无法回应这个请求，导致服务端无法关闭；客户端关闭连接后，原有端口可能会被分配给新的客户端，当新客户端收到老服务端的请求后，会造成报文混乱。2msl是互联网中一个tcp请求能存活的最大时间，linux中为60s。
![[Pasted image 20220816222842.png]]
mysql服务器出现大量time_wait：四次挥手是客户端和服务端都可以发起的。当客户端没有主动关闭mysql连接，超出一定时间后，由于服务端一直没收到客户端的请求，会主动发起关闭请求，所以服务端会出现大量time_wait状态。

所以如果mysql客户端在使用完mysql连接后没有调用con.close()，就会因为长时间没有活动导致mysql服务端主动关闭连接，并处于time_wait阶段。处于time_wait阶段的连接不会被mysql回收，在服务高峰期大量的time_wait连接容易造成连接池的枯竭，导致不能再处理新的连接请求。

SYN洪泛攻击
![[Pasted image 20220816221718.png]]

UDP：DNS、视频服务等

UDT：TCP提出的较早，其流量控制、拥塞控制算法不太适应现在的高速互联网，对传输小数据包友好，传输海量数据不友好。UDT基于UDP，在应用层实现了握手、流量控制、拥塞控制等。
QUIC：谷歌提出，结合了TCP的可靠性和UDP的高效性。可以多路服用（一次连接同时传递多个请求）、一次RTT就能建立安全的连接、丢包时采用前向纠错而不是重传（比如要发送5个包，还会根据这5个包计算第6个包出来，这样这5个包里如果丢了一个，会跟根据第6个算出来）

![[2#Tomcat的io模型#五种io模型]]

一般服务端用nio较好，客户端也可以用bio

# 直接内存（堆外内存）
![[Pasted image 20220824204056.png]]
在进行网络通信时，程序会将数据先从应用内存拷贝到socket缓冲区（用户态->内核态），再将数据发送出去。如果数据一开始放在了堆中，那么虚拟机会先将数据从堆拷贝到堆外内存，进而再拷贝到socket缓冲区。因为堆上会发生GC，如果直接将数据从堆发送到socket缓冲区，那么发送过程中由于GC，可能会导致原数据地址发生变化，从而引起不必要的麻烦。
所以nio框架会直接在堆外内存中分配数据，这样在发送数据时减少了一次从堆到堆外内存的拷贝，提高了通信速率。

# 零拷贝
linux2.1里，如果DMA支持，可以连最后一次cpu拷贝都能省略，直接由DMA完成从一个内核缓冲区到另一个内核缓冲区的拷贝。
![[Pasted image 20220824211541.png]]
linux2.6.17
一次上下文切换会耗费5000-10000时钟周期

![[Pasted image 20220824212014.png]]

kafka的零拷贝：
broker将生产者的消息持久化到磁盘时用的是mmap，broker将磁盘上的消息发给消费者时用的sendfile

nio不一定就比bio好，服务器适用nio（可以用很少的线程数服务更多的用户），客户端适用bio。
![[Pasted image 20220824214914.png]]

# select、poll、epoll的异同
相同点：都是操作系统用来实现io多路复用的机制，通过监听多个文件描述符，当某个描述符就绪（通常是读或写就绪）就通知操作系统进行相应的读写操作。

不同点-最大连接数：
- select：数组实现，最大为32\*系统位数；因为每次调用都会遍历连接数组，性能呈线性下降
- poll：链表实现，无限制；每次调用也会遍历链表
- epoll：受限于系统内存大小；只有活跃的socket才会回调callback函数，所以当活跃socket较少时效率较高。
不同点-消息传递方式：
- select、poll：需要将消息从内核态拷贝到用户态
- epoll：通过内核空间和用户空间共享一块内存实现。


# 操作系统通过中断来感知网络数据的触达
DMA将网卡上的数据拷贝到内存后，会向cpu发送中断，cpu相应中断去处理这部分数据。
![[Pasted image 20220925180309.png]]

# select
当select监听多个socket时，每当其中一个socket有事件发生，就会通知select，这时select就会拿出所有socket进行遍历，找到有事件的那个进行处理。这是最早的nio实现，几乎所有操作系统都支持。
![[Pasted image 20220925182138.png]]

当调用select时，进程会从工作队列移除，进入阻塞状态；当数据从网卡写入内存并发起中断后，cpu在响应中断时就会唤醒进程去执行读写操作。
![[Pasted image 20220925175005.png]]

# epoll
![[Pasted image 20220925182212.png]]
epoll将一次调用拆成了三次，并用eventpoll监听所有socket。当调用wait时，进程进入阻塞状态，系统将有事件发生的socket放入eventpoll中的rdlist里，进程只需要遍历rdlist中的活跃socket即可。
![[Pasted image 20220925180116.png]]
![[Pasted image 20220925180149.png]]
![[Pasted image 20220925181103.png]]

linux进程间通信方式：信号、管道、共享内存、消息队列（系统内部的）、socket（一台机器上的不同进程间，与网络通信不太一样）

linux的aio是伪aio，效果还不如nio
