# Centos7 安装MySQL 5.7

## 1. 下载并安装MySQL官方的 Yum Repository

```
wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
```

### 1.1 使用上面的命令就直接下载了安装用的Yum Repository，大概25KB的样子，然后就可以直接yum安装了。

```
yum -y install mysql57-community-release-el7-10.noarch.rpm
```

### 1.2 之后就开始安装MySQL服务器。

```
yum -y install mysql-community-server
```

这步可能会花些时间，安装完成后就会覆盖掉之前的mariadb。

至此MySQL就安装完成了，然后是对MySQL的一些设置。

## 2. MySQL数据库设置

### 2.1 首先启动MySQL

```
systemctl start  mysqld.service
```

### 2.2 查看MySQL运行状态，运行状态

```
systemctl status mysqld.service
```

```
mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since 五 2020-03-20 10:50:40 CST; 7s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 23577 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid $MYSQLD_OPTS (code=exited, status=0/SUCCESS)
  Process: 23527 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 23580 (mysqld)
   CGroup: /system.slice/mysqld.service
           └─23580 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid

3月 20 10:50:37 localhost.localdomain systemd[1]: Starting MySQL Server...
3月 20 10:50:40 localhost.localdomain systemd[1]: Started MySQL Server.
```

此时MySQL已经开始正常运行，不过要想进入MySQL还得先找出此时root用户的密码，通过如下命令可以在日志文件中找出密码：

```
grep "password" /var/log/mysqld.log
```

```
[Note] A temporary password is generated for root@localhost: <Ty=9cdOTc#N
```

如下命令进入数据库：

```
mysql -uroot -p
```

输入初始密码（是上面图片最后面的 <Ty=9cdOTc#N），此时不能做任何事情，因为MySQL默认必须修改密码之后才能操作数据库：

```
ALTER USER 'root'@'localhost' IDENTIFIED BY 'new password';
```

其中‘new password’替换成你要设置的密码，注意:密码设置必须要大小写字母数字和特殊符号（,/';:等）,不然不能配置成功

如果要修改为root这样的弱密码，需要进行以下配置：

查看密码策略:

```
show variables like '%password%';
```

```
+----------------------------------------+-----------------+
| Variable_name                          | Value           |
+----------------------------------------+-----------------+
| default_password_lifetime              | 0               |
| disconnect_on_expired_password         | ON              |
| log_builtin_as_identified_by_password  | OFF             |
| mysql_native_password_proxy_users      | OFF             |
| old_passwords                          | 0               |
| report_password                        |                 |
| sha256_password_auto_generate_rsa_keys | ON              |
| sha256_password_private_key_path       | private_key.pem |
| sha256_password_proxy_users            | OFF             |
| sha256_password_public_key_path        | public_key.pem  |
+----------------------------------------+-----------------+
```

修改密码策略

```
vim /etc/my.conf
```

添加validate_password_policy配置

选择0（LOW），1（MEDIUM），2（STRONG）其中一种，选择2需要提供密码字典文件

```
#添加validate_password_policy配置
validate_password_policy=0
#关闭密码策略
validate_password = off
```

重启mysql服务使配置生效

```
systemctl restart mysqld
```

然后就可以修改为弱密码啦

## 3. 开启MySQL的远程访问

执行以下命令开启远程访问限制（注意：下面命令开启的IP是 192.168.0.1，如要开启所有的，用%代替IP）：

```
grant all privileges on *.* to 'root'@'192.168.0.1' identified by 'password' with grant option;
```

然后再输入下面两行命令:

```
flush privileges; 
exit;
```

## 4. 为firewalld添加开放端口

添加MySQL端口3306

```
firewall-cmd --zone=public --add-port=3306/tcp --permanent
```

然后再重新载入

```
firewall-cmd --reload
```

## 5. 修改mysql的字符编码（不修改会产生中文乱码问题）

显示原来的编码：

```
show variables like '%character%';
```

```
mysql> show variables like '%character%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | latin1                     |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | latin1                     |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
```

修改/etc/my.cnf

```
[mysqld]
character_set_server=utf8
init_connect='SET NAMES utf8'
```

重启数据库服务再次查看

```
mysql> show variables like '%character%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | utf8                       |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | utf8                       |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
```

