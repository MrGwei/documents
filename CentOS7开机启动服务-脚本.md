# CentOS7 添加开机启动服务/脚本

## 1. 添加开机自启服务

在CentOS7中添加开机自己服务非常方便

```
systemctl enable firewalld.service
```

## 2. 添加开机自启脚本

在CentOS7增加脚本有两种方法，在/etc/init.d/目录下新建start.sh

```
#/bin/sh
#chkconfig: 2345 10 90
#description: httc-mo-appcheck-middleware

RUN_NAME="httc-mo-appcheck-middleware"
/opt/httc-mo-appcheck.sh start
```

### 方法一：

1. 赋予脚本可执行权限

   ```
   chmod +x /etc/rc.d/init.d/start.sh
   ```

   

2. 打开/etc/rc.d/rc.local文件，在末尾添加如下：

   ```
   /etc/rc.d/init.d/start.sh
   ```

   

3. rc.local文件权限被降低了，所以要赋予可执行权限

   ```
   chmod +x /etc/rc.d/init.d/rc.local
   ```

   

### 方法二：

1. 将脚本移动到/etc/rc.d/init.d目录下

   ```
   cp /opt/start.sh /etc/rc.d/init.d/start.sh
   ```

   

2. 增加脚本可执行权限

   ```
   chmod +x /etc/rc.d/init.d/start.sh
   ```

   

3. 添加脚本到开机自启动项目中

   ```
   cd /etc/rc.d/init.d/
   chkconfig --add start.sh
   chkconfig start.sh on
   ```

   

4. 删除脚本自启动

   ```
   chkconfig --del start.sh
   ```

   

