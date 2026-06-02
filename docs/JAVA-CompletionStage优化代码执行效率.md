![image.png](https://github.com/oh-huohou/huohou.blog/blob/main/image/image-completionstage01.png)

CompletableFuture通过实现接口CompletionStage让Future的功能和使用场景得到极大的完善和扩展,提供了函数式编程能力,使代码更加美观优雅,而且可以通过回调的方式计算处理结果,对异常处理也有了更好的处理手段。
# 静态方法
CompletableFuture源码中有四个静态方法用来执行异步任务:
```
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier){..}

public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,Executor executor){..}

public static CompletableFuture<Void> runAsync(Runnable runnable){..}

public static CompletableFuture<Void> runAsync(Runnable runnable,Executor executor){..} 
```
run开头的两个方法,用于执行没有返回值的任务,因为它的入参是Runnable对象
supply开头的方法是执行有返回值的任务了

至于方法的入参,如果没有传入Executor对象将会使用ForkJoinPool.commonPool() 作为它的线程池执行异步代码.
在实际使用中,一般我们使用自己创建的线程池对象来作为参数传入使用,这样速度会快些.
# thenAccept()
```
public CompletionStage<Void> thenAccept(Consumer<? super T> action);
public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action);
public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action,Executor executor);
```
功能:当前任务正常完成以后执行,当前任务的执行结果可以作为下一任务的输入参数,无返回值.
# thenRun(..)
```
public CompletionStage<Void> thenRun(Runnable action);
public CompletionStage<Void> thenRunAsync(Runnable action);
public CompletionStage<Void> thenRunAsync(Runnable action,Executor executor);
```
功能:对不关心上一步的计算结果，执行下一个操作
# thenApply(..)
```
public <U> CompletableFuture<U>     thenApply(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U>     thenApplyAsync(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U>     thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
```
功能:当前任务正常完成以后执行，当前任务的执行的结果会作为下一任务的输入参数,有返回值
# thenCombine(..)  thenAcceptBoth(..)  runAfterBoth(..)
```
public <U,V> CompletableFuture<V>     thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
public <U,V> CompletableFuture<V>     thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
public <U,V> CompletableFuture<V>     thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor)
```
功能:结合两个CompletionStage的结果，进行转化后返回
场景:需要根据商品id查询商品的当前价格,分两步,查询商品的原始价格和折扣,这两个查询相互独立,当都查出来的时候用原始价格乘折扣,算出当前价格. 使用方法:thenCombine(..)
```
CompletableFuture<Double> futurePrice = CompletableFuture.supplyAsync(() -> 100d);
 CompletableFuture<Double> futureDiscount = CompletableFuture.supplyAsync(() -> 0.8);
 CompletableFuture<Double> futureResult = futurePrice.thenCombine(futureDiscount, (price, discount) -> price * discount);
 System.out.println("最终价格为:" + futureResult.join()); //最终价格为:80.0
```
thenCombine(..)是结合两个任务的返回值进行转化后再返回,那如果不需要返回呢,那就需要thenAcceptBoth(..),同理,如果连两个任务的返回值也不关心呢,那就需要runAfterBoth了
# thenCompose(..)
```
public <U> CompletableFuture<U>     thenCompose(Function<? super T,? extends CompletionStage<U>> fn)
public <U> CompletableFuture<U>     thenComposeAsync(Function<? super T,? extends CompletionStage<U>> fn)
public <U> CompletableFuture<U>     thenComposeAsync(Function<? super T,? extends CompletionStage<U>> fn, Executor executor)
```
功能:这个方法接收的输入是当前的CompletableFuture的计算值，返回结果将是一个新的CompletableFuture

这个方法和thenApply非常像,都是接受上一个任务的结果作为入参,执行自己的操作,然后返回.那具体有什么区别呢?

thenApply():它的功能相当于将CompletableFuture<T>转换成CompletableFuture<U>,改变的是同一个CompletableFuture中的泛型类型

thenCompose():用来连接两个CompletableFuture，返回值是一个新的CompletableFuture
# applyToEither(..)  acceptEither(..)  runAfterEither(..)
```
public <U> CompletionStage<U> applyToEither(CompletionStage<? extends T> other,Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn,Executor executor);
```
功能:执行两个CompletionStage的结果,那个先执行完了,就是用哪个的返回值进行下一步


节选自：https://www.cnblogs.com/fingerboy/p/9948736.html



