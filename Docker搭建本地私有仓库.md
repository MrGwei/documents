# Docker搭建本地私有仓库

## 1. 使用registry镜像创建私有仓库

安装Docker后，可以通过官方提供的registry镜像来简单搭建一套本地私有仓库环境：

```
docker run -p 5000:5000 registry
```

这将自动下载并启动一个registry容器，创建本地的私有仓库服务。

默认情况下，会将仓库创建在容器的/tmp/registry目录下。可以**通过-v参数**来将镜像文件存放在**本地的指定路径**。

例如下面的例子将上传的镜像放到/root/docker/registry目录：

```
docker run --restart=always -d --name registry -p 5000:5000 -v /root/docker/registry:/var/lib/registry --privileged=true registry
```



此时，在本地将启动一个私有仓库服务，通过命令查看监听端口5000。

```
ps -ef | grep 5000
```



## 2. 管理私有仓库

首先查看其地址为127.0.0.1:5000，然后在其它Docker环境里测试上传和下载镜像。

查看本地已有镜像：


```
~ # docker images                                                       
REPOSITORY  TAG  	IMAGE ID  		CREATED  	SIZE
ubuntu  	latest  4e5021d210f6  11 days ago  64.2MB
```



将已存在的镜像打标签：192.168.6.180:5000为本地私有仓库地址和端口

```
docker tag 4e5021d210f6 192.168.6.180:5000/ubuntu
```



使用docker push上传标记的镜像到本地私有仓库：

```
docker push 192.168.6.180:5000/ubuntu
```



第一次执行push命令可能会报如下异常：

```
The push refers to repository [192.168.6.180:5000/ubuntu]
Get https://192.168.6.180:5000/v2/: http: server gave HTTP response to HTTPS client
```



解决方案：

在/etc/docker目录下新建daemon.json，文件中写入：

```
{
    "registry-mirrors":["https://registry.docker-cn.com"],
    "insecure-registries":["192.168.1.160:5000"]
}
```



然后重启docker：

```
systemctl daemon-reload
systemctl restart docker
```



再次运行push命令，上传镜像到私有仓库

```
~ # docker push 192.168.6.180:5000/ubuntu
The push refers to repository [192.168.6.180:5000/ubuntu]
16542a8fc3be: Pushed 
6597da2e2e52: Pushed 
977183d4e999: Pushed 
c8be1b8f4d60: Pushed 
latest: digest: sha256:e5dd9dbb37df5b731a6688fa49f4003359f6f126958c9c928f937bec69836320 size: 1152
```



## 3. 管理私有仓库镜像

```
curl http://192.168.6.180:5000/v2/_catalog
```

下载仓库镜像：

```
docker pull 192.168.6.180:5000/ubuntu
```

删除仓库镜像：

```
rm -fr /root/docker/registry/docker/registry/v2/repositories/[name]
```

