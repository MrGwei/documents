# CentOS7进入单用户模式，修复系统
## 1. 开机时进入如下界面，（按下方向键盘，阻止系统自动继续）

# 2. 按e键出现下面界面

## 2.1 按方向键下，定位到最后，找到“ro”一行，ro的意思是read only，将“ro”替换成 rw init=/sysroot/bin/sh，

## 3. 按Ctrl+x进行重启进入单用户模式

## 4. 执行chroot /sysroot。其中chroot命令时用来切换系统，/sysroot/目录就是原始系统

## 5. 按Ctrl+d退出，执行reboot

