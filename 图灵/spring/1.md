 

AOP大致流程 AOP就是进行动态代理，在创建一个Bean的过程中，Spring在最后一步会去判断当前正在 创建的这个Bean是不是需要进行AOP，如果需要则会进行动态代理。 如何判断当前Bean对象需不需要进行AOP:
1. 找出所有的切面Bean  
2. 遍历切面中的每个方法，看是否写了@Before、@After等注解  
3. 如果写了，则判断所对应的Pointcut是否和当前Bean对象的类是否匹配 
4. 如果匹配则表示当前Bean对象有匹配的的Pointcut，表示需要进行AOP

利用cglib进行AOP的大致流程:  
1. 生成代理类UserServiceProxy，代理类继承UserService  
2. 代理类中重写了父类的方法，比如UserService中的test()方法  
3. 代理类中还会有一个target属性，该属性的值为被代理对象(也就是通过 UserService类推断构造方法实例化出来的对象，进行了依赖注入、初始化等步骤的 对象)  
4. 代理类中的test()方法被执行时的逻辑如下:

a. 执行切面逻辑(@Before) b. 调用target.test()

事务也是由动态代理实现的。在执行真正的sql前后，加入事务相关的操作。所以方法调用当前类里的方法时，事务不会生效，因为不能触发动态代理。