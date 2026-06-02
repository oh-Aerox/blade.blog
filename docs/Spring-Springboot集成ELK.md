# 1、简介
 “ELK”是三个开源项目的首字母缩写，这三个项目分别是：Elasticsearch、Logstash 和 Kibana。Elasticsearch 是一个搜索和分析引擎。Logstash 是服务器端数据处理管道，能够同时从多个来源采集数据，转换数据，然后将数据发送到诸如 Elasticsearch 等“存储库”中。Kibana 则可以让用户在 Elasticsearch 中使用图形和图表对数据进行可视化。
# 2、使用docker compose 搭建ELK环境

系统：Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-142-generic x86_64)

## 2.1 创建工作目录及配置文件
### 2.1.1 工作目录
```
mkdir elk
cd elk
mkdir elasticsearch
mkdir logstash
mkdir elasticsearch/data
mkdir elasticsearch/plugins
```
### 2.1.2 配置文件logstash.conf
```
vim logstash/logstash.conf

input {
  tcp {
    mode => "server"
    host => "0.0.0.0"
    port => 4560
    codec => json
  }
}    

output {
  elasticsearch {
    hosts => "es:9200"
    index => "springboot-%{+YYYY.MM.dd}"
    codec => json
  }
}

```
### 2.1.3 文件docker-compose.yml
```
vim docker-compose.yml

version: '3.0'
services:
  elasticsearch:
    image: elasticsearch:6.4.0
    container_name: elasticsearch
    environment:
      - "cluster.name=elasticsearch" #设置集群名称为elasticsearch
      - "discovery.type=single-node" #以单一节点模式启动
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m" #设置使用jvm内存大小
    volumes:
      - /usr/local/elk/elasticsearch/plugins:/usr/share/elasticsearch/plugins #插件文件挂载
      - /usr/local/elk/elasticsearch/data:/usr/share/elasticsearch/data #数据文件挂载
    ports:
      - 9200:9200
      - 9300:9300
  kibana:
    image: kibana:6.4.0
    container_name: kibana
    links:
      - elasticsearch:es #可以用es这个域名访问elasticsearch服务
    depends_on:
      - elasticsearch #kibana在elasticsearch启动之后再启动
    environment:
      - "elasticsearch.hosts=http://es:9200" #设置访问elasticsearch的地址
    ports:
      - 5601:5601
  logstash:
    image: logstash:6.4.0
    container_name: logstash
    volumes:
      - /usr/local/elk/logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf #挂载logstash的配置文件
    depends_on:
      - elasticsearch #kibana在elasticsearch启动之后再启动
    links:
     - elasticsearch:es #可以用es这个域名访问elasticsearch服务
    ports:
      - 4560:4560
```
## 2.2 使用docker-compose.yml脚本启动ELK服务
```
 docker-compose up -d
```
启动之后通过docker ps查看是否成功，发现只有logstash和kibana，查看es的日志发现错误，没有权限
```
 docker logs -f -t --tail 200 elasticsearch
```
解决【[详情跳转](https://www.hounk.world/archives/d-o-c-k-e-r-qi-dong-bao-cuo)】

修改之后再次启动，报错：
```
ERROR: Version in "./docker-compose.yml" is unsupported. You might be seeing this error because you're using the wrong Compose file version. Either specify a version of "2" (or "2.0") and place your service definitions under the `services` key, or omit the `version` key and place your service definitions at the root of the file to use version 1.
For more on the Compose file format versions, see https://docs.docker.com/compose/compose-file/
```
之前是用apt安装的docker-compose，应该是版本有点低，根据提示，修改docker-compose.yml文件，version: '3.0'改为version: '2.0'，再次启动，OK

在logstash中安装json_lines插件
```
# 进入logstash容器(d12c6c0c9d36为logstash CONTAINER ID)
docker exec -it d12c6c0c9d36 /bin/bash
# 进入bin目录
cd /bin/
# 安装插件
logstash-plugin install logstash-codec-json_lines
# 退出容器
exit
# 重启logstash服务
docker restart logstash
```
## 2.4 查看页面
访问ip:9200和ip:5601
![image.png](https://www.hounk.world/upload/2021/03/image-e9683eae32464de6836d675e82bdf10d.png)
![image.png](https://www.hounk.world/upload/2021/03/image-462d89cab4d44ba9a9628122aa30b106.png)

# 3、Springboot集成
## 3.1 添加依赖
```
    compile 'net.logstash.logback:logstash-logback-encoder:5.3'
```
## 3.2 修改配置文件
首先是application.yml，如果没有application.name则添加
```
spring:
  application:
    name: after-sale
```

然后在配置文件logback-spring.xml中添加appender，让logback的日志输出到logstash
```
    <appender name="STASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>10.8.0.50:4560</destination>
        <!-- 日志输出编码 -->
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <pattern>
                    <pattern>
                        {
                        "severity": "%level",
                        "service": "${springAppName:-}",
                        "pid": "${PID:-}",
                        "thread": "%thread",
                        "class": "%logger{40}",
                        "rest": "%message",
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
    </appender>
```

## 3.3 在kibana中查看日志信息
创建index pattern
![image.png](https://www.hounk.world/upload/2021/03/image-ebceecdfe28f42f4adc7a38bf6efd8c9.png)
配置 pattern 输入springboot-*，匹配相关数据
![image.png](https://www.hounk.world/upload/2021/03/image-c58d35659f6c43d7a2ea6a522809aa67.png)
然后选择时间@timestamp，这样数据展示会以时间排序。

启动项目，就可以看到启动日志已经输出到elasticsearch中了:
点击discover：
![image.png](https://www.hounk.world/upload/2021/03/image-0a4af0e3c97546efbafbd1b66ab447f4.png)

看一下错误日志的展示情况，故意调个错误接口，接口中打印：
```
   log.error("handle ApiException=", e);
```
遗憾的是异常的堆栈信息并没有输出
![image.png](https://www.hounk.world/upload/2021/03/image-73353fc78f1e4374bd6ee02c68e348d0.png)
网上一通搜索，多是修改logstash.conf加上正则、过滤、合并等等，具体不再赘述，还有是通过filebeat来解决，尝试了一下，在其配置文件filebeat.yml中需要配置一下收集日志的路径,这就要在项目服务器安装一下filebeat，但是还有好多个项目要收集日志，分布在不同的服务器，有点麻烦。
```
filebeat.inputs:
- type: log
  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /var/log/*.log
```
放弃这种解决方式，回到logback-spring.xml,会不会是没有输出堆栈信息引起的，仔细看了一下输出的pattern，之前搜索的各种解决方式真的是南辕北辙了，输出里面没有ex，修改之后的pattern
```
                    <pattern>
                        {
                        "severity": "%level",
                        "service": "${springAppName:-}",
                        "pid": "${PID:-}",
                        "thread": "%thread",
                        "class": "%logger{40}",
                        "rest": "%m%n %ex"
                        }
                    </pattern>
```
再试一下错误
![image.png](https://www.hounk.world/upload/2021/03/image-02e0bc851f6c466da8a972a3177a0683.png)
没有问题，可以打开前面的三角，看起来更舒服一些
![image.png](https://www.hounk.world/upload/2021/03/image-c1e231bd97c64900b035b896f72060a0.png)

# 附
## 完整logback-spring.xml
```
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<configuration>
    <!-- 日志文件存储位置 -->
    <property name="log_dir" value="logs/erp/"/>
    <!-- 日志最大存活时间（天） -->
    <property name="maxHistory" value="30"/>
    <springProperty scope="context" name="springAppName" source="spring.application.name"/>

    <contextListener class="ch.qos.logback.classic.jul.LevelChangePropagator">
        <resetJUL>true</resetJUL>
    </contextListener>

    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d [%p %t %X{ip} %X{flag} %F:%L] - %m%n %ex{8}</pattern>
        </encoder>
    </appender>

    <appender name="trace" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <param name="file" value="${log_dir}/trace.log"/>
        <param name="Encoding" value="UTF-8"/>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>TRACE</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log_dir}/trace.%d{yyyy-MM-dd}.log
            </fileNamePattern>
            <!-- keep 30 days' worth of history -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d [%p %t %X{ip} %X{flag} %F:%L] - %m%n</pattern>
        </encoder>
    </appender>

    <appender name="debug" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <param name="file" value="${log_dir}/debug.log"/>
        <param name="Encoding" value="UTF-8"/>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>DEBUG</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log_dir}/debug.%d{yyyy-MM-dd}.log
            </fileNamePattern>
            <!-- keep 30 days' worth of history -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d [%p %t %X{ip} %X{flag} %F:%L] - %m%n</pattern>
        </encoder>
    </appender>

    <appender name="info" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <param name="file" value="${log_dir}/info.log"/>
        <param name="Encoding" value="UTF-8"/>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log_dir}/info.%d{yyyy-MM-dd}.log
            </fileNamePattern>
            <!-- keep 30 days' worth of history -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d [%p %t %X{ip} %X{flag} %F:%L] - %m%n</pattern>
        </encoder>
    </appender>

    <appender name="warn" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <param name="file" value="${log_dir}/warn.log"/>
        <param name="Encoding" value="UTF-8"/>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>WARN</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log_dir}/warn.%d{yyyy-MM-dd}.log
            </fileNamePattern>
            <!-- keep 30 days' worth of history -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d [%p %t %X{ip} %X{flag} %F:%L] - %m%n</pattern>
        </encoder>
    </appender>

    <appender name="error" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <param name="file" value="${log_dir}/error.log"/>
        <param name="Encoding" value="UTF-8"/>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log_dir}/error.%d{yyyy-MM-dd}.log
            </fileNamePattern>
            <!-- keep 30 days' worth of history -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d [%p %t %X{ip} %X{flag} %F:%L] - %m%n %ex</pattern>
        </encoder>
    </appender>

    <appender name="STASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>10.8.0.50:4560</destination>
        <!-- 日志输出编码 -->
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <pattern>
                    <pattern>
                        {
                        "severity": "%level",
                        "service": "${springAppName:-}",
                        "pid": "${PID:-}",
                        "thread": "%thread",
                        "class": "%logger{40}",
                        "rest": "%m%n %ex"
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
    </appender>

    <root level="info">
        <appender-ref ref="STASH"/>
        <appender-ref ref="info"/>
        <appender-ref ref="error"/>
        <appender-ref ref="warn"/>
        <appender-ref ref="debug"/>
        <appender-ref ref="console"/>
    </root>
</configuration>
```






参考：
https://www.elastic.co/cn/what-is/elk-stack
https://blog.csdn.net/MCmango/article/details/114022493
https://blog.csdn.net/K_Zombie/article/details/51156319
https://blog.csdn.net/oyc619491800/article/details/107804816
https://www.cnblogs.com/dyh004/p/9641377.html
https://www.cnblogs.com/z-x-p/p/11684477.html
