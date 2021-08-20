## 交换机
![[111.png.png]]
访问 localhost:15672 可以访问管理界面，账号密码guest
交换机的类型有：direct、fanout、topic、headers。direct和headers几乎不用。
![[222.png]]
![[333.png]]
![[444.png]]
![[555.png]]

## API
接收消息:

![[666.png]]
![[777.png]]
![[888.png]]
消费端ack：
默认是自动确认，收到后broker会删除消息，即消息一旦抵达消费端就会ack，即使业务没有处理完。如果处理过程中宕机，那么剩下的没有处理的消息也会ack，发生丢消息。

## 延时队列

![[Pasted image 20210702154002.png]]
业务场景：

![[Pasted image 20210702154117.png]]

稍加改进：

![[Pasted image 20210702154618.png]]

整体流程：

![[Pasted image 20210702161420.png]]
![[Pasted image 20210702161456.png]]

## 解锁库存
![[Pasted image 20210702163213.png]]
![[Pasted image 20210703152040.png]]

## 消息丢失
![[Pasted image 20210703153634.png]]

## 消息重复
![[Pasted image 20210703153949.png]]

## 消息积压
![[Pasted image 20210703154314.png]]