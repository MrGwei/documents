# 使用docker安装consul服务发现

[TOC]

## 1.拉取consul镜像

```
docker pull consul:latest  
```

## 2.consul参数详解

- –net=host docker参数, 使得docker容器越过了net namespace的隔离，免去手动指定端口映射的步骤
- -server consul支持以server或client的模式运行, server是服务发现模块的核心, client主要用于转发请求
- -advertise 将本机私有IP传递到consul
- -retry-join 指定要加入的consul节点地址，失败后会重试, 可多次指定不同的地址
- -client 指定consul绑定在哪个client地址上，这个地址可提供HTTP、DNS、RPC等服务，默认是>127.0.0.1
- -bind 绑定服务器的ip地址；该地址用来在集群内部的通讯，集群内的所有节点到地址必须是可达的，>默认是0.0.0.0
  allow_stale 设置为true则表明可从consul集群的任一server节点获取dns信息, false则表明每次请求都会>经过consul的server leader
- -bootstrap-expect 数据中心中预期的服务器数。指定后，Consul将等待指定数量的服务器可用，然后>启动群集。允许自动选举leader，但不能与传统-bootstrap标志一起使用, 需要在server模式下运行。
- -data-dir 数据存放的位置，用于持久化保存集群状态
- -node 群集中此节点的名称，这在群集中必须是唯一的，默认情况下是节点的主机名。
- -config-dir 指定配置文件，当这个目录下有 .json 结尾的文件就会被加载，详细可参考https://www.consul.io/docs/agent/options.html#configuration_files
- -enable-script-checks 检查服务是否处于活动状态，类似开启心跳
- -datacenter 数据中心名称
- -ui 开启ui界面
- -join 指定ip, 加入到已有的集群中

## 3.简单使用
启动第一个节点，叫 consul1

```
docker run --name consul1 -d -p 8500:8500 -p 8300:8300 -p 8301:8301 -p 8302:8302 -p 8600:8600 consul agent -server -bootstrap-expect 2 -ui -bind=0.0.0.0 -client=0.0.0.0
```



### 端口详解

- 8500 : http 端口，用于 http 接口和 web ui访问；
- 8300 : server rpc 端口，同一数据中心 consul server 之间通过该端口通信；
- 8301 : serf lan 端口，同一数据中心 consul client 通过该端口通信; 用于处理当前datacenter中LAN的gossip通信；
- 8302 : serf wan 端口，不同数据中心 consul server 通过该端口通信; agent Server使用，处理与其他datacenter的gossip通信；
- 8600 : dns 端口，用于已注册的服务发现；
### 启动第二个节点，加入到 consul1
  查看consul1的ip地址

```
docker inspect --format='{{.NetworkSettings.IPAddress}}' consul1
```


开启第二个节点（端口8501），并加入到 consul1

```
docker run --name consul2 -d -p 8501:8500 consul agent -server -ui -bind=0.0.0.0 -client=0.0.0.0 -join 172.17.0.4
```



开启第三个节点（端口8502），并加入到consul1

```
docker run --name consul3 -d -p 8502:8500 consul agent -server -ui -bind=0.0.0.0 -client=0.0.0.0 -join 172.17.0.4
```



查看consul集群信息

```
docker exec -it consul1 consul members
```



我们看到集群里有三个节点

```
~ # docker exec -it consul1 consul members                                   
Node          Address          Status  Type    Build  Protocol  DC   Segment
238b54883cb4  172.17.0.7:8301  alive   server  1.7.3  2         dc1  <all>
6160f23ca27e  172.17.0.8:8301  alive   server  1.7.3  2         dc1  <all>
a246c025ae1c  172.17.0.6:8301  alive   server  1.7.3  2         dc1  <all>
```



## 4.查看集群信息
我们可以打开浏览器： http://localhost:8500 来查看整个集群的信息
