项目使用jpa+hibernate，在进行单元测试的时候，对于已经在数据库层面设置了default字段，java方法中没有进行设值，会报错
```
`cdate` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
```
报错日志信息显示如下
![image.png](https://www.hounk.world/upload/2020/12/image-d9c8894fc5354240af9f602cb3a40ee8.png)

可见，对于未设置value的字段，jpa依然在insert语句中set了该字段。

因为该字段类型为timestamp，且没有设置值，导致报错cannot be null。

解决：

在org.hibernate.annotations;包下提供了DynamicInsert 注解
```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface DynamicInsert {
    boolean value() default true;
}
```
设置为true，表示insert对象的时候,生成动态的insert语句，如果这个字段的值是null就不会加入到insert语句当中。例：一般数据的创建时间字段一般在数据库层面已经设置了default，在对象字段为空的情况下，表字段能自动填写当前的系统时间。

类似注解@DynamicUpdate在更新时实现相同功能
```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface DynamicUpdate {
    boolean value() default true;
}
```
设置为true，表示update对象的时候,生成动态的update语句,如果这个字段的值是null就不会被加入到update语句中。有时候只想更新某个属性，但是不加这个注解就会把整个对象的属性都更新了，但是只更新修改的字段就够了就需要加上这个注解。

改好之后的entity如下图
![image.png](https://www.hounk.world/upload/2020/12/image-e604ae725eb74a28bf3e59d061da345c.png)
再次测试发现没有设置cdate字段，在插入语句中也不会出现了
