---
title: "Nginx+Tomcat集群"
tag:
    - Nginx
    - Tomcat
---
# Tomcat集群的优势

    1.提高服务的性能，==并发能力==，以及高可用性。

    2.提高项目架构的横向拓展能力。

# 一、多Tomcat服务器集群部署

### 1.端口配置

    在准备多台Tomcat服务器时，首先要对每台服务器上的server.xml文件进行相应的端口文件修改，确保各个服务器之间不会产生冲突。

##### （1）修改shutdown端口号

    默认情况下，Tomcat的shutdown端口号为8005。为了区分不同的Tomcat实例，可以将该端口设置为8006、8007等。0

```
<Connector port="8006" protocol="AJP/1.3" redirectPort="8433"/>
```

##### （2）修改访问端口号

    将8080端口改为8090、8091、8092等不同的值。

```
<Connector port="8090" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
```

##### （3）修改AJP端口号

    AJP默认使用8009端口，可根据实际情况调整为8010、8011等。

```
<Connector port="8010" protocol="AJP/1.3" redirectPort="8443" />
```

### 2.测试验证

    修改完成后，通过访问`http://localhost:8091`、`http://localhost:8092`等地址确认各Tomcat服务器运行正常。

### 3.项目部署

* 打包：先打包父项目为WAR包，再打包子项目为JAR包。
* 部署：将打包好的项目放置到各个Tomcat的webapps目录下，并删除ROOT文件夹，将主项目重命名名为ROOT.war，以便通过IP+端口直接访问。

# 二、Nginx负载均衡搭配策略

    Nginx提供了多重负载均衡算法，可以根据实际需求灵活配置。

|    负载均衡策略    | 方式            |
| :----------------: | --------------- |
|        轮询        | 默认方式        |
|       Weight       | 权重方式        |
|      IP_hash      | 根据ip分配方式  |
|     least_conn     | 最少链接方式    |
|   fair（第三方）   | 响应时间方式    |
| url_hash（第三方） | 依据URL分配方式 |

    在这里，只详细说明Nginx自带的负载均衡策略，第三方不多描述。

### 1.轮询

    默认是负载均衡策略，按顺序轮流将将请求分别发送到不同的后端服务器。

    参数如下：

| fail_timeout | 与max_fails结合使用。                                                                                                                            |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| max_fails    | 设置在fail_timeout参数设置的时间内最大失败次数，<br />如果在这个时间内，所有针对该服务器的请求都失败<br />了，那么认为该服务器会被认为是停机了。 |
| fail_time    | 服务器会被认为停机的时间长度,默认为10s。                                                                                                         |
| backup       | 标记该服务器为备用服务器。当主服务器停止时，请求<br />会被发送到它这里。                                                                         |
| down         | 标记服务器永久停机了。                                                                                                                           |

注意：

* 在轮询中，如果服务器down掉了，会自动剔除该服务器。
* 缺省配置就是轮询策略。
* 此策略适合服务器配置相当，无状态且短平快的服务使用。

```
upstream jt {
    server localhost:8091 max_fails=1 fail_timeout=60s;
    server localhost:8092;
}
 
server {
    listen 80;
    server_name localhost;
 
    location / {
        proxy_pass http://jt;
        proxy_read_timeout 3;
        proxy_connect_timeout 3;
        proxy_send_timeout 3;
    }
}
```

### 2.权重

    适用于后端服务器性能不均很的情况，通过设置权重值来控制流量分配。

```
upstream jt {
    server 192.168.0.14 weight=3;
    server 192.168.0.15 weight=7;
}
```

注意：

* 权重越高分配到需要处理的请求越多。
* 此策略可以与least_conn和ip_hash结合使用。
* 此策略比较适合服务器的硬件配置差别比较大的情况。

### 3.IP Hash

    当需要解决Session粘连问题时，可使用IP Hash算法。此方法将同一IP地址的请求始终定向到同一台服务器上。

```
upstream jt {
    ip_hash;
    server localhost:8091;
    server localhost:8092;
}
```

注意：

* 在nginx版本1.3.1之前，不能在ip_hash中使用权重（weight）。
* ip_hash不能与backup同时使用。
* 此策略适合有状态服务，比如session。
* 当有服务器需要剔除，必须手动down掉。

### 4.least_conn

    把请求转发给连接数较少的后端服务器。轮询算法是把请求平均的转发给各个后端，使它们的负载大致相同；但是，有些请求占用的时间很长，会导致其所在的后端负载较高。这种情况下，least_conn这种方式就可以达到更好的负载均衡效果。

```
#动态服务器组
    upstream dynamic_zuoyu {
        least_conn;    #把请求转发给连接数较少的后端服务器
        server localhost:8080   weight=2;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082 backup;  #tomcat 8.5
        server localhost:8083   max_fails=3 fail_timeout=20s;  #tomcat 9.0
    }
```

注意：

* 此负载均衡策略适合请求处理时间长短不一造成服务器过载的情况。

### 5.第三方策略

    第三方的负载均衡策略的实现需要安装第三方插件。

#### ①.fair

    按照服务器端的响应时间来分配请求，响应时间短的优先分配。

```
#动态服务器组
    upstream dynamic_zuoyu {
        server localhost:8080;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082;  #tomcat 8.5
        server localhost:8083;  #tomcat 9.0
        fair;    #实现响应时间短的优先分配
    }
```

#### ②.url_hash

    按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，要配合缓存命中来使用。同一个资源多次请求，可能会到达不同的服务器上，导致不必要的多次下载，缓存命中率不高，以及一些资源时间的浪费。而使用url_hash，可以使得同一个url（也就是同一个资源请求）会到达同一台服务器，一旦缓存住了资源，再此收到请求，就可以从缓存中读取。

```
#动态服务器组
    upstream dynamic_zuoyu {
        hash $request_uri;    #实现每个url定向到同一个后端服务器
        server localhost:8080;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082;  #tomcat 8.5
        server localhost:8083;  #tomcat 9.0
    }
```

### 总结

    以上便是6种负载均衡策略的实现方式，其中除了轮询和轮询权重外，都是Nginx根据不同的算法实现的。在实际运用中，需要根据不同的场景选择性运用，大都是多种策略结合使用以达到实际需求。

### 故障迁移

    当某台服务器出现故障时，可以通过设置`down`状态临时将其排除在外，并配置备用机以备不时之需。

```
upstream jt {
    server localhost:8091 down;
    server localhost:8092;
    server localhost:8093 backup;
}
```

### 三、公司服务器部署流程

1. **策略制定** ：根据实际需求确定集群部署方案。
2. **下线处理** ：对即将更新的服务器标记为 down状态。
3. **部署测试** ：上传WAR包至指定Tomcat服务器，并进行全面测试。
4. **上线操作** ：确认无误后，将服务器状态恢复为 up。
5. 通过以上步骤，我们不仅能有效提升系统的稳定性和响应速度，还能更好地应对高峰期流量激增所带来的挑战。希望本文能够帮助读者更好地理解和实践多Tomcat服务器集群与Nginx负载均衡技术。
