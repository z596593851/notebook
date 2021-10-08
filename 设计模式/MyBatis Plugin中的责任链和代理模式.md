假设我们需要统计应用中每个 SQL 的执行耗时，如果使用 MyBatis Plugin 来实现的话，我们只需要定义一个 SqlCostTimeInterceptor 类，让它实现 MyBatis 的 Interceptor 接口，并且，在 MyBatis 的全局配置文件中，简单声明一下这个插件就可以了：
```java
@Intercepts({
        @Signature(type = StatementHandler.class, method = "query", args = {Statement.class, ResultHandler.class}),
        @Signature(type = StatementHandler.class, method = "update", args = {Statement.class}),
        @Signature(type = StatementHandler.class, method = "batch", args = {Statement.class})})
public class SqlCostTimeInterceptor implements Interceptor {
  private static Logger logger = LoggerFactory.getLogger(SqlCostTimeInterceptor.class);

  @Override
  public Object intercept(Invocation invocation) throws Throwable {
    Object target = invocation.getTarget();
    long startTime = System.currentTimeMillis();
    StatementHandler statementHandler = (StatementHandler) target;
    try {
      return invocation.proceed();
    } finally {
      long costTime = System.currentTimeMillis() - startTime;
      BoundSql boundSql = statementHandler.getBoundSql();
      String sql = boundSql.getSql();
      logger.info("执行 SQL：[ {} ]执行耗时[ {} ms]", sql, costTime);
    }
  }

  @Override
  public Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }

  @Override
  public void setProperties(Properties properties) {
    System.out.println("插件配置的信息："+properties);
  }
}

<!-- MyBatis全局配置文件：mybatis-config.xml -->
<plugins>
  <plugin interceptor="com.xzg.cd.a88.SqlCostTimeInterceptor">
    <property name="someProperty" value="100"/>
  </plugin>
</plugins>
```
`Interceptor`接口以及`InterceptorChain`类：
```java

public class Invocation {
  private final Object target;
  private final Method method;
  private final Object[] args;
  // 省略构造函数和getter方法...
  public Object proceed() throws InvocationTargetException, IllegalAccessException {
    return method.invoke(target, args);
  }
}
public interface Interceptor {
  Object intercept(Invocation invocation) throws Throwable;
  Object plugin(Object target);
  void setProperties(Properties properties);
}

public class InterceptorChain {
  private final List<Interceptor> interceptors = new ArrayList<Interceptor>();

  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }

  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }
  
  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }
}
```
解析完配置文件之后，所有的 Interceptor 都加载到了 InterceptorChain 中。接下来，我们再来看下，这些拦截器是在什么时候被触发执行的？又是如何被触发执行的呢？

MyBatis 底层是通过 Executor 类来执行 SQL 的。Executor 类会创建 StatementHandler、ParameterHandler、ResultSetHandler 三个对象，并且，首先使用 ParameterHandler 设置 SQL 中的占位符参数，然后使用 StatementHandler 执行 SQL 语句，最后使用 ResultSetHandler 封装执行结果。所以，我们只需要拦截 Executor、ParameterHandler、ResultSetHandler、StatementHandler 这几个类的方法，基本上就能满足我们对整个 SQL 执行流程的拦截了。

在以上三个handler创建时，都会调用 InteceptorChain 的 pluginAll() 方法，SQLTimeCostInterceptor 的 plugin() 方法通过直接调用 Plugin 的 wrap() 方法来实现。wrap() 方法的代码实现如下所示：
```java

// 借助Java InvocationHandler实现的动态代理模式
public class Plugin implements InvocationHandler {
  private final Object target;
  private final Interceptor interceptor;
  private final Map<Class<?>, Set<Method>> signatureMap;

  private Plugin(Object target, Interceptor interceptor, Map<Class<?>, Set<Method>> signatureMap) {
    this.target = target;
    this.interceptor = interceptor;
    this.signatureMap = signatureMap;
  }

  // wrap()静态方法，用来生成target的动态代理，
  // 动态代理对象=target对象+interceptor对象。
  public static Object wrap(Object target, Interceptor interceptor) {
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }

  // 调用target上的f()方法，会触发执行下面这个方法。
  // 这个方法包含：执行interceptor的intecept()方法 + 执行target上f()方法。
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
      }
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }

  private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
    Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
    // issue #251
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());      
    }
    Signature[] sigs = interceptsAnnotation.value();
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<Class<?>, Set<Method>>();
    for (Signature sig : sigs) {
      Set<Method> methods = signatureMap.get(sig.type());
      if (methods == null) {
        methods = new HashSet<Method>();
        signatureMap.put(sig.type(), methods);
      }
      try {
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
      }
    }
    return signatureMap;
  }

  private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<Class<?>>();
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[interfaces.size()]);
  }
}
```
当执行 Executor、StatementHandler、ParameterHandler、ResultSetHandler 这四个类上的某个方法的时候，MyBatis 会嵌套执行每层代理对象（Plugin 对象）上的 invoke() 方法。而 invoke() 方法会先执行代理对象中的 interceptor 的 intecept() 函数，然后再执行被代理对象上的方法。就这样，一层一层地把代理对象上的 intercept() 函数执行完之后，MyBatis 才最终执行那 4 个原始类对象上的方法。