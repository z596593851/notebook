# 手动实现aop
```java
public class Advice1 implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("拦截器1-前");
        Object proceed = invocation.proceed();
        System.out.println("拦截器1-后");
        return proceed;
    }
}

public class Advice3 implements MethodBeforeAdvice {
    @Override
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("advice3");
        //会自动执行invocation.proceed();
    }
}

public class UserService {

    public void test(){
        System.out.println("test");
    }

    public static void main(String[] args) {
        UserService target=new UserService();
        ProxyFactory proxyFactory=new ProxyFactory();
        proxyFactory.setTarget(target);
        proxyFactory.addAdvice(new Advice1());
        proxyFactory.addAdvice(new Advice2());
        proxyFactory.addAdvice(new Advice3());

        UserService proxy = (UserService)proxyFactory.getProxy();
        proxy.test();
    }
}
```

# 半手动实现aop

![[Pasted image 20220318190540.png]]

或

![[Pasted image 20220318191247.png]]

或

![[Pasted image 20220318191336.png]]

Advisor = Advice + Pointcut

