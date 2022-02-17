



# 一、概述

## 1、特点

![image-20210104104142881](/Users/apple/Library/Application Support/typora-user-images/image-20210104104142881.png)

## 2、文件结构

![image-20210104104311123](/Users/apple/Library/Application Support/typora-user-images/image-20210104104311123.png)



## 3、应用场景

### 3.1、统一命名服务

![image-20210104104457073](/Users/apple/Library/Application Support/typora-user-images/image-20210104104457073.png)

### 3.2、统一配置管理

![image-20210104104658875](/Users/apple/Library/Application Support/typora-user-images/image-20210104104658875.png)

### 3.3、统一集群管理

![image-20210104104925211](/Users/apple/Library/Application Support/typora-user-images/image-20210104104925211.png)

### 3.4、服务器动态上下线

![image-20210104105110387](/Users/apple/Library/Application Support/typora-user-images/image-20210104105110387.png)

### 3.5、软负载均衡

![image-20210104105155870](/Users/apple/Library/Application Support/typora-user-images/image-20210104105155870.png)

![image-20210104112112901](/Users/apple/Library/Application Support/typora-user-images/image-20210104112112901.png)

# 二、客户端命令行

![image-20210104174208412](/Users/apple/Library/Application Support/typora-user-images/image-20210104174208412.png)

create /sanguo "nn"

get /sanguo

创建短暂节点：create -e /sanguo/wuguo "zhouyu"

创建带序号节点：create -s /sanguo/weiguo "caocao"

修改节点值：set /sanguo/wuguo "me"

监听节点值：get -w /sanguo

监听路径变化：ls -w /sanguo

删除节点：delete /sanguo/wuguo

递归删除：deleteall /sanguo

# 三、部分原理

### 1、stat结构体

![image-20210104195647526](/Users/apple/Library/Application Support/typora-user-images/image-20210104195647526.png)

### 2、监听器原理

![image-20210104200012450](/Users/apple/Library/Application Support/typora-user-images/image-20210104200012450.png)

### 3、写数据流程

![image-20210105175402372](/Users/apple/Library/Application Support/typora-user-images/image-20210105175402372.png)

