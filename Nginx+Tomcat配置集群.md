# Nginx配置Tomcat集群

参考：https://yq.aliyun.com/articles/654858

## Nginx

```
yum -y install nginx
```



### Nginx的启动、停止、重启

```
service nginx start | stop | restart
```



### Nginx常用代理

Http代理、反向代理：作为web服务器最常用的功能之一，尤其是反向代理。

![img](https://yqfile.alicdn.com/img_9dbeb7b8a8f046576257baecd2eba1d1.jpeg)

Nginx反向代理说明图：

Nginx在做反向代理时，提供性能稳定，并且能够提供配置灵活的转发功能。**Nginx可以根据不同的正则匹配，采取不同的转发策略，比如图片文件结尾的走文件服务器，动态页面走web服务器**，只要正则写的没问题，又有相对应的服务器解决方案，就可以随心所欲的玩。并且Nginx对返回结果进行错误页跳转，异常判断等。如果被分发的服务器存在异常，它可以将请求重新转发给另一台服务器，然后自动去除异常服务器。

![img](https://yqfile.alicdn.com/img_bdec9f249d82233fb0eca76552957702.jpeg)

负载均衡说明图1

![img](https://yqfile.alicdn.com/img_7faf18e46c466ccf5ff4431a3c1d1174.jpeg)

负载均衡说明图2

Nginx提供的负载均衡策略有2种：**内置策略和扩展策略**。内置策略为轮训，加权轮训，IP Hash。扩展策略就天马行空，只有想不到没有做不到。

### Nginx配置文件结构

```
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

#user nginx;
user nobody;
worker_processes auto; #工作进程的个数，一般与计算机的cpu核数一致
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024; #单个进程最大连接数（最大连接数=连接数*进程数）
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

	#开启高效文件传输模式，sendfile指令指定nginx是否调用sendfile函数来输出文件，对于普通应用设为 on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络I/O处理速度，降低系统的负载。注意：如果图片显示不正常把这个改成off。
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65; #长连接超时时间，单位是秒
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types; #文件扩展名与文件类型映射表
    default_type        application/octet-stream; #默认文件类型

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

	# Nginx的配置
    server { #每一个server相当于一个代理服务器
        listen       80 default_server; #监听80端口
        listen       [::]:80 default_server;
        server_name  _; #当前服务的域名，可以有多个，用空格分隔
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {#表示匹配的路径，这时配置了/表示所有请求都被匹配到这里
        	#root   html;
            #index  index.html index.htm;#当没有指定主页时，默认会选择这个指定的文件，可多个，空格分隔
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

# Settings for a TLS enabled server.
#
#    server {
#        listen       443 ssl http2 default_server;
#        listen       [::]:443 ssl http2 default_server;
#        server_name  _;
#        root         /usr/share/nginx/html;
#
#        ssl_certificate "/etc/pki/nginx/server.crt";
#        ssl_certificate_key "/etc/pki/nginx/private/server.key";
#        ssl_session_cache shared:SSL:1m;
#        ssl_session_timeout  10m;
#        ssl_ciphers PROFILE=SYSTEM;
#        ssl_prefer_server_ciphers on;
#
#        # Load configuration files for the default server block.
#        include /etc/nginx/default.d/*.conf;
#
#        location / {
#        }
#
#        error_page 404 /404.html;
#            location = /40x.html {
#        }
#
#        error_page 500 502 503 504 /50x.html;
#            location = /50x.html {
#        }
#    }
}

```

#### Nginx worker 进程运行的用户及用户组

默认 user nginx(user nobody nobody);

user用于设置master进程启动之后，fork出的worker进程运行在哪个用户和用户组下

#### Nginx worker进程个数

在master/worker运行方式下指定worker进程的个数；

worker进程的数量会直接影响性能，每个worker进程都是单线程的进程，它们会调用各模块以实现多种多样的功能，如果这些模块确定不会出现阻塞时的调用，那么有多少CPU内核就应该配置多少个进程；反之，如果可能存在多个阻塞的调用，则需要多为worker分配一些进程；

例如磁盘IO一直是后端服务器的一个瓶颈，如果分配到worker进程都是在执行阻塞式的IO服务，那么这个时候Nginx的效率可能会大大降低，这个时候就需要为Nginx多分配一点工作进程；

多worker进程可以充分利用多核的系统架构，但若worker进程的数量多于CPU内核数的话，这样同样会增大进程之间来回切换造成的消耗（Linux是抢占式内核），一般情况下雨内核相等的worker进程，就可以绑定到CPU内核上；

#### 虚拟主机与请求分发

由于IP地址数量有限，一次你经常存在多个主机域名对应同一个IP地址情况，这时在nginx.conf中就可以按照server_name（对应用户请求中的主机域名）并通过server块来定义虚拟主机，每个server块就是个虚拟主机，它只处理与之相对应的主机域名请求

##### 主机名称

server_name后可以跟多个主机名称，在开始处理一个HTTP请求时，Nginx会取出header中的host，与每个server中的server_name进行匹配，决定到哪个server块来处理这个请求，匹配规则也是有优先级的

1. 首先选择所有字符串都完全匹配的server_name
2. 其次选择通配符在前的
3. 选择通配符在后的
4. 最后选择正则表达式匹配的

##### location

location会尝试根据用户请求中的URI来匹配/uri表达式，如果可以匹配，就会选择这个location块中的配置来处理用户请求

##### 文件路径的定义

以root方式设置资源路径比如说一条HTTP请求时一条请求资源的请求，这个时候就可以使用server和location匹配到这一条资源请求，并且使用root指定这条资源请求的目录，实现资源访问，同样实现资源文件的访问还可以在location匹配到url请求之后使用alias，这两者间的配置有一些小的差别，一个用户如果通过一条HTTP请求需要访问到/user/local/nginx/conf/nginx.conf文件可使用如下两种配置：

```
location /conf {
	alias /usr/local/nginx/conf/;
}
location /conf {
	root /usr/local/nginx/;
}
```

使用alias时，在URI向实际文件路径的映射过程中，已经把location后的部分字符串丢弃掉了，因此，/conf/nginx.conf请求将根据alias path映射为path/nginx.conf；

root则不一样，它会根据完整的URI请求进行映射，因此/conf/nginx.conf将会被映射为root path/conf/nginx.conf，这也是为什么root可以放置到http、server、location中，而alias只能存在于location中的原因

#### 使用HTTP proxy module配置一个反向代理服务器

反向代理（reverse proxy）方式是指用代理服务器来接收Internet上的连接请求，然后根据请求转发给内网中的上游服务器，并将从上游服务器上得到的结果返回Internet上请求连接的客户端，此时代理服务器对外的表现就是一个web服务器，充当反向代理服务器也是Nginx的通常用法。

一般来说如果Nginx作为反向代理服务器，接收到的请求若是针对静态文件，则直接通过Nginx将请求映射到本地的静态资源上，如果需要动态的结果，即发出的是一条动态的请求，则需要继续反向代理的真正的web服务器。

当客户端发来的HTTP请求的时候，Nginx并不会立刻转发到上游服务器，而是先把用户的请求（包括HTTP包体）完整的接收到Nginx所在的服务器的硬盘或者内存中，然后再向上游服务器发起连接，把缓存的客户端请求请求转发到上游服务器，而其余的一些反向代理服务器采用的策略是边接收客户端的请求，边进行转发到上游服务器。

Nginx这种工作方式存在一个明显的缺点，很明显，这种缓存策略相当于是延长了一个HTTP请求的时间，并增加了用于缓存请求内容的内存和磁盘空间，而优点是降低了上游服务器的负担，将负担更多的放在了自己身上。

但是Nginx的这个策略同时也大大提高了效率更降低了上游服务器的负担，比如一个场景是客户端想要上传1G的文件，这个时候如果是别的一些反向代理服务器，则在刚与客户端连接还没有开始接受包体的时候，就开始向上游服务器进行发起连接，因为客户端和反向代理服务器之间基本都是走外网，这个时间加上自身的网络情况，上传1G大小的文件通常是一个很耗时的操作，这个时候，这些反向代理服务器就要一直保持着和客户端的关系，又要保持着和上游服务器的关系，这样一个上传文件操作就成了一个既耗时又耗资源的一个操作，而这个时候Nginx和客户端建立连接之后完整的接收包体之后，通过内网与上游服务器之间建立连接，这个速度和资源损耗就远远小于其余一些反向代理服务器了。

在介绍Nginx反向代理，搭建了一个小型的Tomcat集群，这样的话就可以通过Nginx做反向代理服务器，并且使用负载均衡策略，向上游的两台服务器按照指定的权重策略进行分发请求。

## 搭建Tomcat服务器集群

首先，熟悉基于J2EE规范以及使用Spring看框架开发的开发者应该都接触过Tomcat这种并发量适中的小型web应用服务器，可是，如果有一天，项目访问量，甚至说并发量达到了一定的级别，该如何处理这种情况，原始的架构只是单Tomcat服务器，但是这种结构肯定不足以支撑对整个后端服务器的运营，这个时候怎么办，首先要想到的就是在有限的一台服务器上鼓捣一个Tomcat集群，让多台Tomcat同时工作，并且使用Nginx作为反向代理服务器，配置负载均衡策略，这样去解决单台Tomcat负担过大访问量的单台服务器瓶颈。

### 简易搭建Tomcat集群

首先需要确保电脑或者服务器上有安装过两台Tomcat服务器，分别起名为main以及secoundary，并且修改了webapps中ROOT文件下的index.jsp，这样的话一会在测试环节上就能区分出Nginx分发请求到哪一台上游服务器上。

```
apache-tomcat-main
apache-tomcat-secondary
```

解压后两台Tomcat服务器，

#### 修改/etc/profile

添加如下环境变量

```
export CATALINA_BASE=/root/environment/apache-tomcat-main
export CATALINA_HOME=/root/environment/apache-tomcat-main
export TOMCAT_HOME=/root/environment/apache-tomcat-main

export CATALINA_SECONDARY_BASE=/root/environment/apache-tomcat-secondary
export CATALINA_SECONDARY_HOME=/root/environment/apache-tomcat-secondary
export TOMCAT_SECONDARY_HOME=/root/environment/apache-tomcat-secondary
```

#### 修改secondary中的bin/catalina.sh文件

在OS节点下添加如下信息，使secondary开启的时候使用/etc/profile中配置的环境变量

```
export CATALINA_BASE=$CATALINA_SECONDARY_BASE
export CATALINA_HOME=$CATALINA_SECONDARY_HOME
```

#### 修改main中conf/server.xml

修改启动端口

```
<Server port="18005" shutdown="SHUTDOWN">
```

修改Connector节点，并且为了防止乱码增加URIEncoding属性

```
<Connector port="18080" protocol="HTTP/1.1" 
connectionTimeout="20000" redirectPort="8443" URIEncoding="UTF-8" />
```

修改AJP端口

```
<Connector protocol="AJP/1.3" address="::1"
 port="18009" redirectPort="8443" />
```



#### 修改secondary中的conf/server.xml

修改启动端口

```
<Server port="28005" shutdown="SHUTDOWN">
```

修改Connector节点，并且为了防止乱码增加URIEncoding属性

```
<Connector port="28080" protocol="HTTP/1.1" 
connectionTimeout="20000" redirectPort="8443" URIEncoding="UTF-8" />
```

修改AJP端口

```
<Connector protocol="AJP/1.3" address="::1"
 port="28009" redirectPort="8443" />
```

#### 开启Tomcat服务

```
catalina.sh start | stop
```



#### AJP连接器可以通过AJP协议和一个web容器进行交互

当你想让Apache和Tomcat结合并且你想让Apache处理静态内容的时候用AJP，或者你想利用Apache的SSL处理能力使特属于HTTP Connector，AJP还可以与engine元素上的jvmRoute结合使用来实现负载均衡功能

#### 配置nginx.conf

1. 屏蔽nginx.conf中server

2. 添加配置

   ```
   include /etc/nginx/conf.d/*.conf
   ```

   

3. 在/etc/nginx/conf.d/目录下新建tomcat.cluster.conf配置文件

   ```
       upstream www.wft.com {
           server 127.0.0.1:18080;
           server 127.0.0.1:28080;
       }   
       server {
           listen       80 default_server;
           listen       [::]:80 default_server;
           server_name  127.0.0.1;
           root         /usr/share/nginx/html;
   
           location / { 
               proxy_pass http://www.wft.com;
           }   
   
           error_page 404 /404.html;
               location = /40x.html {
           }   
   
           error_page 500 502 503 504 /50x.html;
               location = /50x.html {
           }   
       }
   ```

   

4. 修改/etc/hosts文件，添加如下：

   ```
   127.0.0.1 www.wft.com
   ```

#### 配置说明

**upstream节点**

upstream节点用来配置上游服务器IP以及端口，用于Nginx转发使用，在这里这个www.wft.com域名是在本地hosts中配置指向127.0.0.1的域名，所以就相当于指向本地服务器IP。

**server节点中的server_name**

简单地说，location块需要结合server_name和location后的uri进行匹配，并且根据location块中国的proxy_pass配置确定转发的upstream节点。

这一段Nginx的配置主要就是当用户通过www.wft.com这个域名作为url访问服务器的时候，Nginx匹配到了这个请求，并且向上游两个开启的18080和28080服务器进行转发，因为没有指定权重，所以两台服务器权重weight默认都为1。

访问www.wft.com:18080,www.wft.28080,访问对应的Tomcat服务器，访问www.wft.com作为URL进行请求，可以看到Nginx将请求转发到了两台开机在不同端口的Tomcat。

#### QA

Q:解决Nginx的connect() to 127.0.0.1:8080 failed (13: Permission denied) while connect

A:SeLinux导致，检查命令

```
getenforce
```

1. 临时关闭

   ```
   setenforce 0
   ```

2. 永久关闭，修改/etc/selinux/config文件

   ```
   SELINUX=enforcing # 开启
   SELINUX=disabled  # 关闭
   ```

3. 重启机器

4. SELinux

   ```
   SELinux(Security-Enhanced Linux) 是美国国家安全局（NAS）对于强制访问控 制的实现，在这种访问控制体系的限制下，进程只能访问那些在他的任务中所需要文件。大部分使用 SELinux 的人使用的都是SELinux就绪的发行版，例如 Fedora、Red Hat Enterprise Linux (RHEL)、Debian 或 Gentoo。它们都是在内核中启用SELinux 的，并且提供一个可定制的安全策略，还提供很多用户层的库和工具，它们都可以使用 SELinux 的功能。 
   ```
```


```