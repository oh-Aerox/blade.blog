> 先说问题：如果启用了Async，而在项目中没有自定义线程池，在并发量大的时候容易出现OOM。


Spring中用@Async注解标记的方法，称为异步方法，它会在调用方的当前线程之外的独立的线程中执行。

# @Async注解使用

* @Async注解一般用在类的方法上，如果用在类上，那么这个类所有的方法都是异步执行的；
* 所使用的@Async注解方法的类对象应该是Spring容器管理的bean对象；
* 调用异步方法类上需要配置上注解@EnableAsync

使用注意：
1. 默认情况下（即@EnableAsync注解的mode=AdviceMode.PROXY），同一个类内部没有使用@Async注解修饰的方法调用@Async注解修饰的方法，是不会异步执行的，这点跟 @Transitional 注解类似，底层都是通过动态代理实现的。如果想实现类内部自调用也可以异步，则需要切换@EnableAsync注解的mode=AdviceMode.ASPECTJ，详见[@EnableAsync注解](https://www.apiref.com/spring5/org/springframework/scheduling/annotation/EnableAsync.html)。
2. 任意参数类型都是支持的，但是方法返回值必须是void或者Future类型。当使用Future时，可以使用 实现了Future接口的ListenableFuture接口或者CompletableFuture类与异步任务做更好的交互。如果异步方法有返回值，没有使用Future<V>类型的话，调用方获取不到返回值。

 # @Async应用默认线程池
Spring应用默认的线程池，指在@Async注解在使用时，不指定线程池的名称。查看源码，@Async的默认线程池为SimpleAsyncTaskExecutor。
![image.png](https://www.hounk.world/upload/2021/06/image-fd7968af96be4b788b9acd6a162010b3.png)

![image.png](https://www.hounk.world/upload/2021/06/image-8cd3f726f2174d7092d8814ae5442a3e.png)
## SimpleAsyncTaskExecutor
异步执行用户任务的SimpleAsyncTaskExecutor。每次执行客户提交给它的任务时，它会启动新的线程，并允许开发者控制并发线程的上限（concurrencyLimit），从而起到一定的资源节流作用。默认时，concurrencyLimit取值为-1，即不启用资源节流。参考SimpleAsyncTaskExecutor注释
![image.png](https://www.hounk.world/upload/2021/06/image-c56f1505abf84db899caa5832db5bda2.png)

# @Async应用自定义线程池
自定义线程池，可对系统中线程池更加细粒度的控制，方便调整线程池大小配置，线程执行异常控制和处理。在设置系统自定义线程池代替默认线程池时，虽可通过多种模式设置，但替换默认线程池最终产生的线程池有且只能设置一个（不能设置多个类继承AsyncConfigurer）。自定义线程池有如下模式：
1. 重新实现接口AsyncConfigurer
2. 继承AsyncConfigurerSupport
3. 配置由自定义的TaskExecutor替代内置的任务执行器
   
通过查看Spring源码关于@Async的默认调用规则，会优先查询源码中实现AsyncConfigurer这个接口的类，实现这个接口的类为AsyncConfigurerSupport。但默认配置的线程池和异步处理方法均为空，所以，无论是继承或者重新实现接口，都需指定一个线程池。且重新实现 public Executor getAsyncExecutor()方法。

## 实现接口AsyncConfigurer
```java
1 @Configuration
 2 public class AsyncConfiguration implements AsyncConfigurer {
 3     @Bean("kingAsyncExecutor")
 4     public ThreadPoolTaskExecutor executor() {
 5         ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
 6         int corePoolSize = 10;
 7         executor.setCorePoolSize(corePoolSize);
 8         int maxPoolSize = 50;
 9         executor.setMaxPoolSize(maxPoolSize);
10         int queueCapacity = 10;
11         executor.setQueueCapacity(queueCapacity);
12         executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
13         String threadNamePrefix = "kingDeeAsyncExecutor-";
14         executor.setThreadNamePrefix(threadNamePrefix);
15         executor.setWaitForTasksToCompleteOnShutdown(true);
16         // 使用自定义的跨线程的请求级别线程工厂类19         int awaitTerminationSeconds = 5;
20         executor.setAwaitTerminationSeconds(awaitTerminationSeconds);
21         executor.initialize();
22         return executor;
23     }
24 
25     @Override
26     public Executor getAsyncExecutor() {
27         return executor();
28     }
29 
30     @Override
31     public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
32         return (ex, method, params) -> ErrorLogger.getInstance().log(String.format("执行异步任务'%s'", method), ex);
33     }
34 }
```
## 继承AsyncConfigurerSupport
```java
@Configuration  
@EnableAsync  
class SpringAsyncConfigurer extends AsyncConfigurerSupport {  
  
    @Bean  
    public ThreadPoolTaskExecutor asyncExecutor() {  
        ThreadPoolTaskExecutor threadPool = new ThreadPoolTaskExecutor();  
        threadPool.setCorePoolSize(3);  
        threadPool.setMaxPoolSize(3);  
        threadPool.setWaitForTasksToCompleteOnShutdown(true);  
        threadPool.setAwaitTerminationSeconds(60 * 15);  
        return threadPool;  
    }  
  
    @Override  
    public Executor getAsyncExecutor() {  
        return asyncExecutor;  
}  

  @Override  
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
    return (ex, method, params) -> ErrorLogger.getInstance().log(String.format("执行异步任务'%s'", method), ex);
}
}
```
## 配置自定义的TaskExecutor
由于AsyncConfigurer的默认线程池在源码中为空，Spring通过beanFactory.getBean(TaskExecutor.class)先查看是否有线程池，未配置时，通过beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class)，又查询是否存在默认名称为TaskExecutor的线程池。所以可在项目中，定义名称为TaskExecutor的bean生成一个默认线程池。也可不指定线程池的名称，申明一个线程池，本身底层是基于TaskExecutor.class便可。

比如：
 * Executor.class:ThreadPoolExecutorAdapter->ThreadPoolExecutor->AbstractExecutorService->ExecutorService->Executor（这样的模式，最终底层为Executor.class，在替换默认的线程池时，需设置默认的线程池名称为TaskExecutor）
 * TaskExecutor.class:ThreadPoolTaskExecutor->SchedulingTaskExecutor->AsyncTaskExecutor->TaskExecutor（这样的模式，最终底层为TaskExecutor.class，在替换默认的线程池时，可不指定线程池名称。）
```java
1 @EnableAsync
 2 @Configuration
 3 public class TaskPoolConfig {
 4     @Bean(name = AsyncExecutionAspectSupport.DEFAULT_TASK_EXECUTOR_BEAN_NAME)
 5     public Executor taskExecutor() {
 6         ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
 7          //核心线程池大小
 8         executor.setCorePoolSize(10);
 9         //最大线程数
10         executor.setMaxPoolSize(20);
11         //队列容量
12         executor.setQueueCapacity(200);
13         //活跃时间
14         executor.setKeepAliveSeconds(60);
15         //线程名字前缀
16         executor.setThreadNamePrefix("taskExecutor-");
17         executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
18         return executor;
19     }
      @Bean(name = "new_task")
 5     public Executor taskExecutor() {
 6         ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
 7          //核心线程池大小
 8         executor.setCorePoolSize(10);
 9         //最大线程数
10         executor.setMaxPoolSize(20);
11         //队列容量
12         executor.setQueueCapacity(200);
13         //活跃时间
14         executor.setKeepAliveSeconds(60);
15         //线程名字前缀
16         executor.setThreadNamePrefix("taskExecutor-");
17         executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
18         return executor;
19     }
20 }
```

## 多个线程池
   @Async注解，使用系统默认或者自定义的线程池（代替默认线程池）。可在项目中设置多个线程池，在异步调用时，指明需要调用的线程池名称，如@Async("new_task")。
