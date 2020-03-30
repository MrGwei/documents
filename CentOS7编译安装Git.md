# CentOS7 编译安装Git
[TOC]
## 下载地址：

```
https://github.com/git/git/releases
```

## 进入到指定路径

```
cd /usr/local/src/
```

## 下载文件

```
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.23.0.tar.xz
```



## 解压

```
tar -xvf git-2.23.0.tar.xz
```



## 进入源目录

```
cd git-2.23.0/ 
```



## 指定编译生成路径

```
make prefix=/usr/local/git all 
```



## 指定安装路径

```
make prefix=/usr/local/git install 
```



## 设置环境变量

```
echo "export PATH=$PATH:/usr/local/git/bin" >> /etc/profile
```



## 配置文件生效

```
source /etc/profile
```