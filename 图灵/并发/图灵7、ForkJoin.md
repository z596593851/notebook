线程数=CPU核心数*(1+平均等待时间/平均工作时间)

![[Pasted image 20220120183031.png]]
提交第一个任务时，会初始化workQueue[]，并且new一个workQueue放在工作队列数组的偶数位；而后创建一个工作线程，再创建一个workQueue与线程绑定，并把workQueue放在工作队列数组的奇数位，再把工作线程放在workQueue中。

唤醒工作线程，执行线程的compute()方法。

工作线程继续fork子任务，向当前线程对应的workQueue头部放入任务。如果空闲线程太少，则创建新的线程和对应的workQueue，他们会自动窃取任务去执行；否则唤醒那些空闲线程。
![[Pasted image 20220120193332.png]]