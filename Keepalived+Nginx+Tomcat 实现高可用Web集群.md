# Keepalived+Nginx+Tomcat 实现高可用Web集群

参考：https://www.jianshu.com/p/bc34f9101c5e

环境准备，集群规划清单：

| 虚拟机                    | IP            | 说明                |
| ------------------------- | ------------- | ------------------- |
| Keepalived+Nginx1[Master] | 192.168.6.174 | Nginx Server 01     |
| Keeepalived+Nginx[Backup] | 192.168.6.172 | Nginx Server 02     |
| Tomcat01                  | 192.168.6.170 | Tomcat Web Server01 |
| Tomcat02                  | 192.168.6.171 | Tomcat Web Server02 |
| VIP                       | 192.168.6.150 | 虚拟漂移IP          |



## 0.安装JDK

官网：

https://www.oracle.com/java/technologies/javase-downloads.html

下载后执行安装命令：

```
rpm -ivh jdk[version].rpm
```



## 1.安装Tomcat

官网：

http://tomcat.apache.org/

配置环境变量/etc/profile

```
export CATALINA_BASE=/root/environment/apache-tomcat-main
export CATALINA_HOME=/root/environment/apache-tomcat-main
export TOMCAT_HOME=/root/environment/apache-tomcat-main
```

### 1.1更改Tomcat欢迎页面，用于标识切换Web

```
<div id="asf-box">
<h1>${pageContext.servletContext.serverInfo}-(192.168.6.170)-<%=request.getHeader("X-NGINX")%>
</h1>
</div>
```

```
<div id="asf-box">
<h1>${pageContext.servletContext.serverInfo}-(192.168.6.171)-<%=request.getHeader("X-NGINX")%>
</h1>
</div>
```

### 1.2启动Tomcat

分别访问：

	1. http://192.168.6.170:8080/
 	2. http://192.168.6.171:8080/

## 2.安装Nginx

官网：

http://nginx.org/

### 方法1

默认情况下CentOS7中无Nginx的源，最近发现Nginx官网提供了CentOS的源地址：

```
rpm -ivh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
```

### 方法2

配置EPEL源

```
yum install -y epel-release
yum -y update
```

**安装**

```
yum install -y nginx
```

安装成功后:

默认的网站目录为：/usr/share/nginx/html

默认的配置文件为：/etc/nginx/nginx.conf

默认自定义配置文件目录为：/etc/nginx/conf.d/

### 配置Nginx代理

#### 2.1 配置Master节点

```
    upstream tomcat {
        server 192.168.6.170:8080;
        server 192.168.6.171:8080;
    }   
    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  tomcat;
        root /usr/share/nginx/html;

        location / { 
            proxy_pass http://tomcat;
            proxy_set_header X-NGINX "NGINX-Master";
        }   

        error_page 404 /404.html;
            location = /40x.html {
        }   

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }    
    }
```



#### 2.2 配置Backup节点

```
    upstream 192.168.6.172 {
        server 192.168.6.170:8080;
        server 192.168.6.171:8080;
    }   
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name 192.168.6.172;
        root /usr/share/nginx/html;

        location / { 
            proxy_pass http://192.168.6.172;
            proxy_set_header X-NGINX "NGINX-Backup";
        }   
    
        error_page 404 /404.html;
            location = /40x.html {
        }   

        error_page 500 502 503 504 /50x.html;
            location = /50x.html{
        }   
    }
```



#### 2.3 配置Nginx开机自启动

```
chkconfig nginx on
```



## 3.安装Keepalived

官网：

https://www.keepalived.org/

```
yum install -y keepalived
```

### 3.1添加脚本

#### 3.1.1在Master节点和Slave节点/etc/keepalived/目录下添加check_nginx.sh文件，用于检测Nginx的存活情况

check_nginx.sh内容如下：

```
#!/bin/bash
#时间变量，用于记录日志
d=`date --date today +%Y%m%d_%H:%M:%S`
#计算nginx进程数量
n=`ps -C nginx --no-heading|wc -l`
#如果进程为0，则启动nginx，并且再次检测nginx进程数量，
#如果还为0，说明nginx无法启动，此时需要关闭keepalived
if [ $n -eq "0" ]; then
        /etc/rc.d/init.d/nginx start
        n2=`ps -C nginx --no-heading|wc -l`
        if [ $n2 -eq "0"  ]; then
                echo "$d nginx down,keepalived will stop" >> /var/log/check_ng.log
                systemctl stop keepalived
        fi
fi
```

添加完成后，为check_nginx.sh文件授权，获得执行权限

```
chmod -R 777 /etc/keepalived/check_nginx.sh
```

#### 3.1.2.在Master节点/etc/keepalived/目录下，添加keepalived.conf文件

keepalived.conf内容如下:

```
vrrp_script chk_nginx {  
 script "/etc/keepalived/check_nginx.sh"   //检测nginx进程的脚本  
 interval 2  
 weight -20  
}  

global_defs {  
 notification_email {  
     //可以添加邮件提醒  
 }  
}  
vrrp_instance VI_1 {  
 state MASTER                  #标示状态为MASTER 备份机为BACKUP
 interface ens33               #设置实例绑定的网卡(ip addr查看，需要根据个人网卡绑定)
 virtual_router_id 51          #同一实例下virtual_router_id必须相同   
 mcast_src_ip 192.168.6.101   
 priority 250                  #MASTER权重要高于BACKUP 比如BACKUP为240  
 advert_int 1                  #MASTER与BACKUP负载均衡器之间同步检查的时间间隔，单位是秒
 nopreempt                     #非抢占模式
 authentication {              #设置认证
        auth_type PASS         #主从服务器验证方式
        auth_pass 123456  
 }  
 track_script {  
        check_nginx  
 }  
 virtual_ipaddress {           #设置vip
        192.168.6.150         #可以多个虚拟IP，换行即可
 } 
```

### 3.2在Backup节点/etc/keepalived/目录下，添加keepalived.conf文件

keepalived.conf内容如下:

```
vrrp_script chk_nginx {  
 script "/etc/keepalived/check_nginx.sh"   //检测nginx进程的脚本  
 interval 2  
 weight -20  
}  

global_defs {  
 notification_email {  
     //可以添加邮件提醒  
 }  
}  
vrrp_instance VI_1 {  
 state BACKUP                  #标示状态为MASTER 备份机为BACKUP
 interface ens33               #设置实例绑定的网卡(ip addr查看)
 virtual_router_id 51          #同一实例下virtual_router_id必须相同   
 mcast_src_ip 192.168.6.102   
 priority 240                  #MASTER权重要高于BACKUP 比如BACKUP为240  
 advert_int 1                  #MASTER与BACKUP负载均衡器之间同步检查的时间间隔，单位是秒
 nopreempt                     #非抢占模式
 authentication {              #设置认证
        auth_type PASS         #主从服务器验证方式
        auth_pass 123456  
 }  
 track_script {  
        check_nginx  
 }  
 virtual_ipaddress {           #设置vip
        192.168.6.150         #可以多个虚拟IP，换行即可
 }  
}
```

**Tips:关于配置信息的几点说明**

- state - 主服务器需配成MASTER，从服务器需配成BACKUP

- interface - 这个是网卡名，我使用的是VM12.0的版本，所以这里网卡名为ens33

- mcast_src_ip - 配置各自的实际IP地址

- priority - 主服务器的优先级必须比从服务器的高，这里主服务器配置成250，从服务器配置成240

- virtual_ipaddress - 配置虚拟IP（192.168.43.150）

- authentication - auth_pass主从服务器必须一致，keepalived靠这个来通信

- virtual_router_id - 主从服务器必须保持一致

### 3.3 设置Keepalived开机自启动

```
chkconfig keepalived  on
```



## 4.集群高可用（HA）验证

### 4.1启动Master机器的Keepalived和Nginx服务

```
keepalived -D -f /etc/keepalived/keepalived.conf
service nginx start
```

**查看Keepalived服务启动进程**

```
/etc/keepalived # ps -aux | grep keepalived 
root       9918  0.0  0.0  123008 1408 ?  Ss 16:55 0:00 keepalived -D -f /etc/keepalived/keepalived.conf
root       9920  0.0  0.1  125132 3148 ?  S 16:55 0:00 keepalived -D -f /etc/keepalived/keepalived.conf
root       9921  0.0  0.1  125132 2444 ?  S 16:55 0:00 keepalived -D -f /etc/keepalived/keepalived.conf
```

**查看Nginx服务启动进程**

```
/etc/keepalived # ps -aux | grep nginx
root       9960  0.0  0.0  46448   972 ?  Ss 16:55 0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
nginx      9961  0.0  0.1  46856  1928 ? S 16:55 0:00 nginx: worker process
```

**使用ip add查看虚拟IP绑定**

如出现192.168.6.150节点信息则绑定到Master节点：

```
/etc/keepalived # ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:2f:e0:09 brd ff:ff:ff:ff:ff:ff
    inet 192.168.6.174/24 brd 192.168.6.255 scope global noprefixroute dynamic ens33
       valid_lft 83777sec preferred_lft 83777sec
    inet 192.168.6.150/32 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::cff:fb0a:b700:72d6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever

```

### 4.2启动Backup机器的Keepalived和Nginx服务

如Backup节点出现了虚拟IP，则Keepalived配置文件有问题，此情况称为脑裂

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:14:df:79 brd ff:ff:ff:ff:ff:ff
    inet 192.168.43.102/24 brd 192.168.43.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::314f:5fe7:4e4b:64ed/64 scope link 
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN qlen 1000
    link/ether 52:54:00:2b:74:aa brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN qlen 1000
    link/ether 52:54:00:2b:74:aa brd ff:ff:ff:ff:ff:ff
```

### 4.3验证服务

浏览并多次强制刷新地址：http://192.168.6.150，可以看到170和171多次交替显示，并显示Master，则表明Master节点在进行web服务转发；

### 4.4关闭Master Keepalived服务和Nginx服务

```
service keepalived stop
service nginx stop
```

此时强制刷新192.168.6.150发现 页面交替显示170和171并显示Backup，VIP已转移到192.168.6.172上，已证明服务自动切换到备份节点上。

### 4.5启动Master Keepalived服务和Nginx服务

此时再次验证发现，VIP已被Master重新夺回，并页面交替显示 170和171，此时显示Master；

## 5.关闭防火墙

```
iptables -F
```

### 5.1查看防火墙状态：

```
systemctl status firewalld.service
```

active（running），说明防火墙启用

disavtive（dead），说明防火墙已经关闭

### 5.2临时关闭：

```
systemctl stop firewalld.service
```

### 5.3永久关闭：

```
systemctl disable firewalld.service
```



