有状态就是有数据存储功能。有状态对象(Stateful Bean)，就是有实例变量的对象 ，可以保存数据，是非线程安全的。在不同方法调用间不保留任何状态。其实就是有数据成员的对象。

　　无状态就是一次操作，不能保存数据。无状态对象(Stateless Bean)，就是没有实例变量的对象。不能保存数据，是不变类，是线程安全的。具体来说就是只有方法没有数据成员的对象，或者有数据成员但是数据成员是可读的对象。

有状态对象:
```java
public class B {  
      private String name;  
      public B(String arg) {  
      this.name = arg;  
      }  
      public String hello() {  
         return "Hello" + this.name;  
      }  
   }  
  
   public class Client {  
      public Client() {  
         B a = new B("中国");  
         System.out.println(a.hello());  
         B b = new B("世界");  
         System.out.println(b.hello());  
      }  
   }
```

无状态对象:
```java
public class A {  
   public A() {}  
   public String hello() {  
      return "Hello 谁？";  
   }  
}  
  
public class Client {  
   public Client() {  
      A a = new A();  
      System.out.println(a.hello());  
      A b = new A();  
      System.out.println(b.hello());  
   }  
}
```
