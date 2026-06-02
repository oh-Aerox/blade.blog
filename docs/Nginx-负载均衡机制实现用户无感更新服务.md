# 准备Nginx
```shell
# 安装 Nginx
sudo yum install -y nginx

# 启动 Nginx
sudo systemctl start nginx.service

# 设置开机自启 Nginx
sudo systemctl enable nginx.service
```
安装之后
![image.png](https://www.hounk.world/upload/2021/04/image-ebb87c134d1043019dbcd24cbac5d693.png)
## 准备项目
本例使用Springboot项目为例，新建项目略
添加测试接口
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.web.ServerProperties;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author naikuoh
 * @DATE 2021/4/11 14:12
 */
@RestController
@RequestMapping(value = "/test")
public class TestController {
    @Autowired
    private ServerProperties serverProperties;

    @GetMapping
    public String hello() {
        return "The current service port :" + serverProperties.getPort();
    }
}
```
# 启动项目
启动三个进程，端口号自定
```
java -jar spring-boot-gather-0.0.1-SNAPSHOT.jar --server.port=8080 &
java -jar spring-boot-gather-0.0.1-SNAPSHOT.jar --server.port=8081 &
java -jar spring-boot-gather-0.0.1-SNAPSHOT.jar --server.port=9090 &
```
# 配置nginx
进入nginx/conf.d/，新建boot.conf
![image.png](https://www.hounk.world/upload/2021/04/image-496ee5757aef4b63a99c53b9881deacb.png)
boot.conf内容如下
```shell
upstream boot {
     server 127.0.0.1:8080 weight=1;
     server 127.0.0.1:8081 weight=2;
     server 127.0.0.1:9090 backup;
 }
server {
    listen       80;
    server_name  127.0.0.1;
    access_log /var/log/nginx/user.log;
    error_log /var/log/nginx/user.error;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://boot/;
    }
}
```
保存boot.conf后重新加载nginx
```shell
# nginx -s reload
```
# 测试
## 测试权重weight
![image.png](https://www.hounk.world/upload/2021/04/image-2a2e85788ba9417b8ec37c8c8a5cc95f.png)
配置的权重生效
## 测试备用服务backup
杀掉8080和8081端口的服务
![image.png](https://www.hounk.world/upload/2021/04/image-30b4728702094ae2ada6375d27821079.png)
请求转发到备用服务9090
