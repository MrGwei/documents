# Ubuntu安装NFS

Server:

1、先更新

```
sudo apt update
```

2、安装NFS服务

```
sudo apt install nfs-server
```

3、修改配置文件，添加共享目录

```
echo "/data/file/upload *(rw,sync,no_root_squash)">>/etc/exports
```

3.1、共享给指定IP的客户端

```
echo "/data/file/upload 192.168.1.2(rw,sync,no_root_squash)">>/etc/exports
```

4、重启NFS服务

```
sudo service nfs-server restart
```



Client:

1、安装客户端

```
sudo apt-get install nfs-common
```

2、NFS客户端挂载

```
mount -t nfs 192.168.1.1:/data/file/upload /mnt/file/upload
```

