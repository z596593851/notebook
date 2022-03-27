## 1.适配器模式
spring中有三种可以定义controller类的方法：
```java

// 方法一：通过@Controller、@RequestMapping来定义
@Controller
public class DemoController {
    @RequestMapping("/employname")
    public ModelAndView getEmployeeName() {
        ModelAndView model = new ModelAndView("Greeting");        
        model.addObject("message", "Dinesh");       
        return model; 
    }  
}

// 方法二：实现Controller接口 + xml配置文件:配置DemoController与URL的对应关系
public class DemoController implements Controller {
    @Override
    public ModelAndView handleRequest(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        ModelAndView model = new ModelAndView("Greeting");
        model.addObject("message", "Dinesh Madhwal");
        return model;
    }
}

// 方法三：实现Servlet接口 + xml配置文件:配置DemoController类与URL的对应关系
public class DemoServlet extends HttpServlet {
  @Override
  protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    this.doPost(req, resp);
  }
  
  @Override
  protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    resp.getWriter().write("Hello World.");
  }
}
```
Spring 定义了统一的接口 HandlerAdapter，并且对每种 Controller 定义了对应的适配器类。这些适配器类包括：AnnotationMethodHandlerAdapter、SimpleControllerHandlerAdapter、SimpleServletHandlerAdapter 等：
```java

public interface HandlerAdapter {
  boolean supports(Object var1);

  ModelAndView handle(HttpServletRequest var1, HttpServletResponse var2, Object var3) throws Exception;

  long getLastModified(HttpServletRequest var1, Object var2);
}

// 对应实现Controller接口的Controller
public class SimpleControllerHandlerAdapter implements HandlerAdapter {
  public SimpleControllerHandlerAdapter() {
  }

  public boolean supports(Object handler) {
    return handler instanceof Controller;
  }

  public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    return ((Controller)handler).handleRequest(request, response);
  }

  public long getLastModified(HttpServletRequest request, Object handler) {
    return handler instanceof LastModified ? ((LastModified)handler).getLastModified(request) : -1L;
  }
}

// 对应实现Servlet接口的Controller
public class SimpleServletHandlerAdapter implements HandlerAdapter {
  public SimpleServletHandlerAdapter() {
  }

  public boolean supports(Object handler) {
    return handler instanceof Servlet;
  }

  public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    ((Servlet)handler).service(request, response);
    return null;
  }

  public long getLastModified(HttpServletRequest request, Object handler) {
    return -1L;
  }
}
```
使用时统一调用`handlerAdapter.handle`即可

## 2.装饰器模式
Spring 使用装饰器模式，用 TransactionAwareCacheDecorator 增加了对事务的支持，在事务提交、回滚的时候分别对 Cache 的数据进行处理。TransactionAwareCacheDecorator 实现 Cache 接口，并且将所有的操作都委托给 targetCache 来实现，对其中的写操作添加了事务功能：
```java

public class TransactionAwareCacheDecorator implements Cache {
  private final Cache targetCache;

  public TransactionAwareCacheDecorator(Cache targetCache) {
    Assert.notNull(targetCache, "Target Cache must not be null");
    this.targetCache = targetCache;
  }

  public Cache getTargetCache() {
    return this.targetCache;
  }

  public String getName() {
    return this.targetCache.getName();
  }

  public Object getNativeCache() {
    return this.targetCache.getNativeCache();
  }

  public ValueWrapper get(Object key) {
    return this.targetCache.get(key);
  }

  public <T> T get(Object key, Class<T> type) {
    return this.targetCache.get(key, type);
  }

  public <T> T get(Object key, Callable<T> valueLoader) {
    return this.targetCache.get(key, valueLoader);
  }

  public void put(final Object key, final Object value) {
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
      TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
        public void afterCommit() {
          TransactionAwareCacheDecorator.this.targetCache.put(key, value);
        }
      });
    } else {
      this.targetCache.put(key, value);
    }
  }
  
  public ValueWrapper putIfAbsent(Object key, Object value) {
    return this.targetCache.putIfAbsent(key, value);
  }

  public void evict(final Object key) {
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
      TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
        public void afterCommit() {
          TransactionAwareCacheDecorator.this.targetCache.evict(key);
        }
      });
    } else {
      this.targetCache.evict(key);
    }

  }

  public void clear() {
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
      TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
        public void afterCommit() {
          TransactionAwareCacheDecorator.this.targetCache.clear();
        }
      });
    } else {
      this.targetCache.clear();
    }
  }
}
```
## 3.责任链模式
拦截器，spring aop的执行链

## 4.代理模式
AOP