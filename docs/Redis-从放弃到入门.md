# 简介
 略。参考《[redis简介](https://www.runoob.com/redis/redis-intro.html )》
# 数据类型
[《站内跳转》](https://www.hounk.world/archives/r-e-d-i-s---zhi-bu-zhi-shu-ju-lei-xing)

# redis为什么快
## 纯内存操作

## 单线程操作，避免了频繁的上下文切换
 Redis的单线程模型，指的是执行Redis命令的核心模块是单线程的，而不是整个 Redis实例就一个线程，Redis 其他模块还有各自模块的线程的。
![image.png](https://www.hounk.world/upload/2021/01/image-6586d5b8b7314e4194a36add55218f72.png)

## 采用非阻塞I/O多路复用机制
《[从IO到多路复用](https://www.hounk.world/archives/cong-i-o-dao-duo-lu-fu-yong)》

# 过期策略 & 淘汰机制
[《站内跳转》](https://www.hounk.world/archives/r-e-d-i-s--zhi--guo-qi-ce-lve)

# redis事务
[《站内跳转》](https://www.hounk.world/archives/r-e-d-i-s--zhi--shi-wu)

# 持久化
[《站内跳转》](https://www.hounk.world/archives/r-e-d-i-s--zhi--chi-jiu-hua)
# 遇到的问题
## 1.安装时make错误
错误如下：
![截图.png](https://www.hounk.world/upload/2021/01/%E6%88%AA%E5%9B%BE-dda096710a634d6cb60807ec07ed787e.png)

解决办法： 使用以下命令
```shell
make MALLOC=libc
```
原因分析：
在README 有这个一段话。
```
Allocator
———
Selecting a non-default memory allocator when building Redis is done by setting
the `MALLOC` environment variable. Redis is compiled and linked against libc
malloc by default, with the exception of jemalloc being the default on Linux
systems. This default was picked because jemalloc has proven to have fewer
fragmentation problems than libc malloc.
To force compiling against libc malloc, use:
% make MALLOC=libc
To compile against jemalloc on Mac OS X systems, use:
% make MALLOC=jemalloc
```
说关于分配器allocator， 如果有MALLOC这个环境变量，会有用这个环境变量去建立Redis。而且libc 并不是默认的分配器，默认的是jemalloc, 因为jemalloc 被证明比libc有更少的 fragmentation problems。但是如果你又没有jemalloc 而只有 libc 当然 make出错。 所以加这么一个参数

参考：
https://www.runoob.com/redis/redis-tutorial.html
https://mp.weixin.qq.com/s/X3e68ci6O9YPXbCAVaQc_w
https://www.cnblogs.com/ysocean/p/9114268.html
