# synchronized和Locks

# 1.什么情况才有线程安全问题
## 线程安全 - 方法内部变量
如果变量是定义在方法内部，则不存在"非线程安全"问题，所得的结果也就是"线程安全"的了
"非线程安全"问题存在于"实例变量"中
## 非线程安全 - 实例变量
1. 如果多个线程共同访问1个对象中的实例变量，运行结果可能出现交叉的情况，则出现"非线程安全"问题
2. 多个线程访问同一个对象中的同步方法时一定是线程安全的
## 多个对象多个锁
1.关键字synchronized取得的锁都是对象锁，而不是把一段代码或方法当作锁
2.多个线程访问同一个对象，哪个线程先执行带synchronized关键字的方法，哪个线程就持有该方法所属对象的锁, 那么其他线程只能呈现等待状态
3.如果多个线程访问多个对象，则JVM会创建多个锁
## synchronized方法与锁对象
1.只有共享资源的读写访问才需要同步化，如果不是共享资源，那么就根本没有同步的必要。
2.调用带关键字synchronized声明的方法一定是排队运行的。
3.A线程先持有一个对象的锁，B线程可以以异步的方式调用该对象中的非synchronized类型的方法
当A线程调用一个对象加入synchronized关键字的X方法时，A线程就获得了X方法锁，更准确的讲，是获得了该对象的锁，所以其他线程必须等待A线程执行完毕才可以调用X方法，也就是释放对象锁后才可以调用，但可以调用该对象中其他的非synchronized同步方法
4.A线程先持有一个对象的Lock锁，B线程如果在这时调用该对象中的synchronized类型的方法，则需要等待，也就是同步
# 2.synchronize
 Synchronized 是Java 中很重要的关键字，解决的是多个线程之间访问资源的同步性，可以保证被它修饰的方法或者代码块在任意时刻只能有一个线程执行，从而避免多线程同时修改同一数据。
 在 Java 早期版本中，synchronized属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的 Mutex Lock 来实现的，Java 的线程是映射到操作系统的原生线程之上的。如果要挂起或者唤醒一个线程，都需要操作系统帮忙完成，而操作系统实现线程之间的切换时需要从用户态转换到内核态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的 synchronized 效率低的原因。
 在 Java 6 之后 Java 官方对从 JVM 层面对synchronized 较大优化，所以现在的 synchronized 锁效率也优化得很不错了。JDK1.6对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销。
当一个线程执行的代码出现异常时，其所持有的锁会自动释放
## 2.1synchronize原理
synchronized 关键字底层原理属于 JVM 层面。
### synchronized 同步语句块的情况
```java
public void method(){     
    synchronized (this){         
        System.out.println("代码块");     
    } 
}
```
 通过 JDK 自带的 javap 命令查看 SynchronizedDemo 类的相关字节码信息：首先切换到类的对应目录执行 javac SynchronizedDemo.java 命令生成编译后的 .class 文件，然后执行`javap -c -s -v -l `SynchronizedDemo.class。
![image.png](https://www.hounk.world/upload/2020/12/image-745d173ebbea4a24a26ecd5f7267ea04.png)
从上面我们可以看出：
  synchronized 同步语句块的实现使用的是 monitorenter 和 monitorexit 指令，其中 monitorenter 指令指向同步代码块的开始位置，monitorexit 指令则指明同步代码块的结束位置。 当执行 monitorenter 指令时，线程试图获取锁也就是获取 monitor（monitor对象存在于每个Java对象的对象头中，synchronized 锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因) 的持有权。当计数器为0则可以成功获取，获取后将锁计数器设为1也就是加1。相应的在执行 monitorexit 指令后，将锁计数器设为0，表明锁被释放。如果获取对象锁失败，那当前线程就要阻塞等待，直到锁被另外一个线程释放为止。
### synchronized 修饰方法的的情况
```java
public synchronized void method(){     
    System.out.println("代码块");
}
```
![image.png](https://www.hounk.world/upload/2020/12/image-7d1582e5c1bc4e5cab92410c5052851a.png)
 synchronized 修饰的方法并没有 monitorenter 指令和 monitorexit 指令，取得代之的是 ACC_SYNCHRONIZED 标识，该标识指明了该方法是一个同步方法，JVM 通过该 ACC_SYNCHRONIZED 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。如果修饰的是静态方法，在flags为 flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
## 2.2synchronized关键字的作用域
### 修饰实例方法方法
`Public synchronized void methodAAA() {  //  .....}`
 修饰实例方法时它锁定的是调用这个同步方法对象。synchronized aMethod(){}可以防止多个线程同时访问这个对象的synchronized方法。
 如果**一个对象**有多个synchronized方法，只要一个线程访问了其中的一个synchronized方法，其它线程不能同时访问**这个对象**中任何一个synchronized方法）。这时，**不同的对象实例**的synchronized方法**是不相干扰**的。也就是说，其它线程照样可以同时访问相同类的另一个对象实例中的synchronized方法。例：当一个对象P1在不同的线程中执行这个同步方法时，它们之间会形成互斥，达到同步的效果；但是这个对象所属的Class所产生的另一对象P2却可以任意调用这个被加了synchronized关键字的方法。
上边的示例代码等同于如下代码：
``` java
public void methodAAA() {     synchronized (this) {  //   ......}                    // (1) }
```
 (1)处的this指的是什么呢？它指的就是调用这个方法的对象，如P1。可见同步方法实质是将synchronized作用于object reference。――那个拿到了P1对象锁的线程，才可以调用P1的同步方法，而对P2而言，P1这个锁与它毫不相干，程序也可能在这种情形下摆脱同步机制的控制，造成数据混乱
### 修饰static方法
 它可以对类的所有对象实例起作用，所有调用该方法的线程都会实现同步，示例代码如下：
``` java
class Foo {
	public synchronized static void methodAAA(){ // 同步的static 函数
		// ….
	}
	public void methodBBB() {
		synchronized (Foo.class) { // class literal(类名称字面常量)
		}
	}
}
```
 代码中的methodBBB()方法是把class literal作为锁的情况，它和同步的static函数产生的效果是一样的，取得的锁很特别，是当前调用这个方法的对象所属的类（Class，而不再是由这个Class产生的某个具体对象了）。
可以推断：**如果一个类中定义了一个synchronized的static函数A，也定义了一个synchronized 的instance函数B，那么这个类的同一对象Obj在多线程中分别访问A和B两个方法时，不会构成同步，因为它们的锁都不一样。A方法的锁是Obj这个对象，而B的锁是Obj所属的那个Class**。
### 修饰代码块
```java
public void method3(SomeObject so) {     
    synchronized(so);
}
```

 这时，锁就是so这个对象，谁拿到这个锁谁就可以运行它所控制的那段代码。当有一个明确的对象作为锁时，就可以这样写程序，但当没有明确的对象作为锁，只是想让一段代码同步时，可以创建一个特殊的instance变量（它得是一个对象）来充当锁：
```java
class Foo implements Runnable{ 	
    private byte[] lock = new byte[0];  
    // 特殊的instance变量 	
    public void methodA(){        
        synchronized(lock) {
            //...        
        }     
    } //….. 
}
```
注：零长度的byte数组对象创建起来将比任何对象都经济――查看编译后的字节码：生成零长度的byte[]对象只需3条操作码，而Object lock = new Object()则需要7行操作码。
## 2.3synchronized是重入锁
“可重入锁”：自己可以再次获取自己的内部锁，比如有1个线程获取了某个对象的锁，此时这个对象锁还没有释放，当其再次想要获取这个对象的锁的时候还是可以获取的，如果不可锁重入的话，就会造成死锁
关键字synchronized拥有锁重入功能，也就是在使用synchronized时，当一个线程得到对象锁后，再次请求此对象锁时是可以再次得到该对象锁的。也证明了在一个synchronized方法（块）的内部调用本类的其他synchronized方法（块）时，是永远可以得到锁的
## 2.4synchronized同步不具有继承性
 基类的方法synchronized f(){} 在继承类中并不自动是synchronized f(){}，而是变成了f(){}。继承类需要你显式的指定它的某个方法为synchronized方法； 
## 2.5synchronize总结

1. 无论synchronized关键字加在方法上还是对象上，它取得的锁都是对象，而不是把一段代码或函数当作锁――而且同步方法很可能还会被其他线程的对象访问。
1. 实现同步是要很大的系统开销作为代价的，甚至可能造成死锁，所以尽量避免无谓的同步控制。
1. 线程同步的目的是为了保护多个线程访问一个资源时对资源的破坏。
1. 线程同步方法是通过锁来实现，每个对象都有切仅有一个锁，这个锁与一个特定的对象关联，线程一旦获取了对象锁，其他访问该对象的线程就无法再访问该对象的其他非同步方法
1. 对于静态同步方法，锁是针对这个类的，锁对象是该类的Class对象。静态和非静态方法的锁互不干预。一个线程获得锁，当在一个同步方法中访问另外对象上的同步方法时，会获取这两个对象锁。
1. 对于同步，要时刻清醒在哪个对象上同步，这是关键。
1. 编写线程安全的类，需要时刻注意对多个线程竞争访问资源的逻辑和安全做出正确的判断，对“原子”操作做出分析，并保证原子操作期间别的线程无法访问竞争资源。
1. 当多个线程等待一个对象锁时，没有获取到锁的线程将发生阻塞。
1. 死锁是线程间相互等待锁锁造成的，在实际中发生的概率非常的小。但是，一旦程序发生死锁，程序将死掉。
# 3.Lock
``` java
public interface Lock
```
`Lock` 是 Java中很重要的一个接口，它要比 Synchronized 关键字更能直译"锁"的概念，Lock需要手动加锁和手动解锁，一般通过 lock.lock() 方法来进行加锁， 通过 lock.unlock() 方法进行解锁。与 Lock 关联密切的锁有 ReetrantLock 和 ReadWriteLock。
**3.1ReetrantLock**
``` java
public class ReentrantLock implements Lock, java.io.Serializable
```
`ReetrantLock` 实现了Lock接口，它是一个可重入锁，内部定义了公平锁与非公平锁。
**3.2ReadWriteLock**
``` java
public interface ReadWriteLock
```
`ReadWriteLock` 一个用来获取读锁，一个用来获取写锁。也就是说将文件的读写操作分开，分成2个锁来分配给线程，从而使得多个线程可以同时进行读操作。
**3.2.1ReentrantReadWirteLock**
``` java
public class ReentrantReadWriteLock implements ReadWriteLock, java.io.Serializable
```
实现了ReadWirteLock接口，并未实现Lock接口。
**3.3`Lock`主要方法**
``` java
public interface Lock {
     void lock();
     void lockInterruptibly() throws InterruptedException;
     boolean tryLock();
     boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
     void unlock(); 
    Condition newCondition(); 
}
```
**3.3.1 lock 方法**
是平常使用最多的一个方法，就是用来获取锁。如果锁被其他线程获取，则进行等待。 如果采用Lock，必须主动去释放锁，并且在发生异常时，不会自动释放锁。
``` java
Lock lock = ...; 
lock.lock(); 
try{
     //处理任务 
}catch(Exception ex){       
}finally{
     lock.unlock();   //释放锁 
}
```
**3.3.2 tryLock()**
有返回值的，它表示用来尝试获取锁，如果获取成功，则返回true，如果获取失败（即锁已被其他线程获取），则返回false，也就说这个方法无论如何都会立即返回。在拿不到锁时不会一直在那等待。
**3.3.3 tryLock(long time, TimeUnit unit)**
 和tryLock()方法是类似的，只不过区别在于这个方法在拿不到锁时会等待一定的时间，在时间期限之内如果还拿不到锁，就返回false。如果如果一开始拿到锁或者在等待期间内拿到了锁，则返回true。
``` java
Lock lock = ...;
if(lock.tryLock()) {
      try{
          //处理任务
      }catch(Exception ex){
      }finally{ 
         lock.unlock();   //释放锁 
     }
}else {
     //如果不能获取锁，则直接做其他事情 
}
```
**3.3.4 lockInterruptibly()**
此方法比较特殊，当通过这个方法去获取锁时，如果线程正在等待获取锁，则这个线程能够响应中断，即中断线程的等待状态。也就使说，当两个线程同时通过 lock.lockInterruptibly() 想获取某个锁时，假若此时线程A获取到了锁，而线程B只有在等待，那么对线程B调用 threadB.interrupt() 方法能够中断线程B的等待过程。 由于 lockInterruptibly() 的声明中抛出了异常，所以 lock.lockInterruptibly() 必须放在try块中或者在调用lockInterruptibly() 的方法外声明抛出 InterruptedException。一般形式如下：
public void method() throws InterruptedException {     lock.lockInterruptibly();     try {        //.....     }     finally {         lock.unlock();     }   }
一般来说，使用Lock必须在try{}catch{}块中进行，并且将释放锁的操作放在finally块中进行，以保证锁一定被被释放，防止死锁的发生。
# 4.Synchronzied 和 Lock 的主要区别如下
-**存在层面**： Syncronized 是Java 中的一个关键字，存在于 JVM 层面，Lock 是 Java 中的一个接口
-**锁的释放条件**：1. 获取锁的线程执行完同步代码后，自动释放；2. 线程发生异常时，JVM会让线程释放锁；Lock 必须在 finally 关键字中释放锁，不然容易造成线程死锁
-**锁的获取:** 在 Syncronized 中，假设线程 A 获得锁，B 线程等待。如果 A 发生阻塞，那么 B 会一直等待。在 Lock 中，会分情况而定，Lock 中有尝试获取锁的方法，如果尝试获取到锁，则不用一直等待
-**锁的状态**： Synchronized 无法判断锁的状态，Lock 则可以判断
-**锁的类型**： Synchronized 是可重入，不可中断，非公平锁；Lock 锁则是 可重入，可判断，可公平锁
-**锁的性能**： Synchronized 适用于少量同步的情况下，性能开销比较大。Lock 锁适用于大量同步阶段：
          Lock 锁可以提高多个线程进行读的效率(使用 readWriteLock) 
          在竞争不是很激烈的情况下，Synchronized的性能要优于ReetrantLock，但是在资源竞争很激烈的情况下，Synchronized的性能会下降，但是ReetrantLock的性能能维持常态； 
          ReetrantLock 提供了多样化的同步，比如有时间限制的同步，可以被Interrupt的同步（synchronized的同步是不能Interrupt的）等。
