最近想把所有的接口保存到数据库，因为之前项目已经集成了swagger，所以就直接去读去的ApiOperation注解。
```
...
...
 Method[] methods = clazz.getDeclaredMethods();
            for (Method method : methods) {
                ApiOperation annotation = method.getAnnotation(ApiOperation.class);
                if (annotation != null) {
                    ...
                }
            }
...
...
```
执行时候发现并没有获取到ApiOperation的对象。

经大神指点，才注意到当前class对象是代理对象，
![image.png](https://github.com/oh-huohou/huohou.blog/blob/main/image/image-5241f52474c141e19cf5b28b97e3b38b.png)

按说我是通过重写的org.springframework.context.support.ApplicationObjectSupport#initApplicationContext(org.springframework.context.ApplicationContext)获取的实例不应该是代理对象。那这里为什么获取到了代理对象呢？

AOP的插足

之前为了统一拦截打印日志，所以在Controller方法上加了aop的处理，而spring aop的实现就是通过代理来实现的，导致现在获取的对象变成了代理对象。

那么如何通过代理对象获取到原对象呢。

```
package com.huohou;

import org.springframework.aop.framework.AdvisedSupport;
import org.springframework.aop.framework.AopProxy;
import org.springframework.aop.support.AopUtils;

import java.lang.reflect.Field;

/**
 * @author huohou
 */
public class AopTargetUtils {
    public static Object getTarget(Object obj) {
        if (!AopUtils.isAopProxy(obj)) {
            return obj;
        }
        try {
            //判断是jdk还是cglib代理
            if (AopUtils.isJdkDynamicProxy(obj)) {
                obj = getJdkDynamicProxyTargetObject(obj);
            } else {
                obj = getCglibDynamicProxyTargetObject(obj);
            }
        } catch (Exception e) {
            System.out.println(e);
        }
        return obj;
    }

    private static Object getCglibDynamicProxyTargetObject(Object obj) throws Exception {
        Field h = obj.getClass().getDeclaredField("CGLIB$CALLBACK_0");
        h.setAccessible(true);

        Object dynamicAdvisedInterceptor = h.get(obj);
        Field advised = dynamicAdvisedInterceptor.getClass().getDeclaredField("advised");
        advised.setAccessible(true);

        return ((AdvisedSupport) advised.get(dynamicAdvisedInterceptor)).getTargetSource().getTarget();
    }

    private static Object getJdkDynamicProxyTargetObject(Object obj) throws Exception {
        Field h = obj.getClass().getSuperclass().getDeclaredField("h");
        h.setAccessible(true);

        AopProxy aopProxy = (AopProxy) h.get(obj);
        Field advised = aopProxy.getClass().getDeclaredField("advised");
        advised.setAccessible(true);

        return ((AdvisedSupport) advised.get(aopProxy)).getTargetSource().getTarget();
    }
}
```
工具类来自：https://www.cnblogs.com/hu0529/p/15173003.html