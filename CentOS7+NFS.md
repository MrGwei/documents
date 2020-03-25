# CentOS7 NFS 服务配置实战

## NFS 是什么

NFS，是Network File System的简写，即网络文件系统。网络文件系统是FreeBSD支持的文件系统中的一种，也被称为NFS，NFS允许一个系统在网络上与他人共享目录和文件。通过使用NFS，用户和程序可以像访问本地文件一样访问远端系统上的文件。 运行模式：C/S 模式 端口：CentOS7以NFSv4作为默认版本，NFSv4使用TCP协议（端口号是2049）和NFS服务器建立连接。

## 典型应用场景

有个单体应用现在需要对其进行横向扩展，但是由于这个应用比较老且在开发之初未考虑其扩展性，文件与应用数据都是存在一台服务器上。 这样在对应用扩容时就不能简单的直接将应用部署多台，会导致应用文件路径不正确。我们先需要搭建一套分布式文件服务器如FastDFS，然后对所有操作文件的接口进行修改调整。改动量还是相当大的，如果需要快速上线直接搭建一套NFS网络文件系统即可。

- 首先利用NFS搭建文件Server端
- 然后在应用上也安装NFS，并将应用文件目录/app/file挂载到Server端指定目录/app/file，这样在应用上上传文件后，文件会自动同步到Server端
- 将应用部署多台进行横向扩容，并全部按照步骤2进行文件挂载。这样文件也都会同步到所有的应用服务器上。

由于文件在所有应用服务器上都存在一份，应用服务器读取其他服务器上的文件就跟在本地读取一样，应用端代码不需要进行改造，这样就实现了应用的快速扩容。

接下来我们就来看一下使用Centos7部署NFS的详细过程。

## 部署过程

### Server端

#### 安装NFS

- 检查是否安装NFS

  ```
  rpm -qa nfs-utils rpcbind
  ```

  

- 关闭防火墙

  ```
  ## 查看防火墙状态
  systemctl status firewalld
  ## 关闭防火墙
  systemctl stop firewalld
  ```

  

- 安装NFS

  ```
  yum install nfs-utils rpcbind -y
  ```

  

- 检查安装结果

  ```
  rpm -qa nfs-utils rpcbind
  ```

  ```
  /opt # rpm -qa nfs-utils rpcbind                                   
  rpcbind-0.2.0-48.el7.x86_64
  nfs-utils-1.3.0-0.65.el7.x86_64
  ```

  出现上内容所示则表明安装成功

#### 配置NFS

- 创建配置文件

  ```
  vim /etc/exports
  ```

  

- 建立同步文件夹.

  ```
  mkdir -p /data/file/upload
  ```

  

- 对同步文件夹进行授权

  ```
  chown -R nfsnobody.nfsnobody /data/file/upload
  ```

  

- 在配置文件中加入如下配置

  ```
  /app/file *(rw,sync,no_root_squash)
  ```

  执行`exportfs –rv`让配置立即生效

  

- 将NFS和rpcbind加入开机启动

  ```
  systemctl enable nfs
  systemctl enable rpcbind
  ```

  

- 启动NFS和rpcbind

  ```
  systemctl start nfs
  systemctl start rpcbind
  ```

  

- 查看NFS启动状态

  ```
  systemctl status nfs
  ```

  

### Client端配置

- 关闭防火墙

  ```
  ## 查看防火墙状态
  systemctl status firewalld
  ## 关闭防火墙
  systemctl stop firewalld
  ```

  

- 安装NFS软件包，并把NFS服务设置为开机启动

  ```
  ## 安装NFS
  yum install nfs-utils rpcbind  -y
  ## 将NFS加入开启启动
  systemctl enable nfs
  ## 将rpcbind加入开启启动
  systemctl enable rpcbind
  ##启动NFS
  systemctl start nfs
  ## 启动RPCbind
  systemctl start rpcbind	
  ```

  

- 将应用文件夹挂在到服务器上

  ```
  mount –t nfs 172.31.63.132:/data/file/upload /app/file
  ```

  挂载完成后可以使用`mount | grep file`命令查看挂载情况

  

- 取消挂载

  ```
  sudo fuser -m -v -i -k /app/file
  sudo umount /app/file
  ```

  直接使用 umount /app/file 可能会报“Device is busy”错误。

