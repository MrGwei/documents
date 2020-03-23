# CentOS7 安装 SFtp

## 1、添加用户组 

```
groupadd sftp
```



## 2、添加用户并设置为sftp组
```
useradd -g sftp -s /sbin/nologin -M sftp
```



## 3、修改sftp用户密码为123
```
passwd sftp ：sftp@httc.com
```



## 4、创建sftp用户的跟目录和属主，属组修改权限755
```
cd /usr
mkdir sftp
chown root:sftp sftp
chmod 755 sftp
```



## 5、在sftp的目录中创建可写如的目录
```
cd /usr/sftp/
mkdir file
chown sftp:sftp file
```



## 6、修改ssdh_config配置文件
vim /etc/ssh/sshd_config
修改Subsystem

```
Subsystem sftp internal-sftp
```



## 7、添加用户配置
```
Match User sftp
    X11Forwarding no
    AllowTcpForwarding no
    ForceCommand internal-sftp
    ChrootDirectory /usr/sftp
```



## 8、重启SSH
```
systemctl restart sshd.service
```

