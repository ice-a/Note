## 目录系统

cd directory
cd ..
cd /
pwd 查看路径

lsblk 查看所有硬件
df -h 查看硬件使用率
/home 用户主目录 （服务器+客户端每个人一个用户）
/dev  所有设备信息
/media/user_name/ 下面有u盘文件
/bin 放各种系统自带的命令，放在里面就可以用tab提示
/opt 存放第三方软件
/lib 放各种库文件

## 用户管理

### 添加用户

```shell
su root
方法一：
useradd -m
passwd  [username]
会不显示用户名，需要vim /etc/passwd 
/bin/sh 改为/bin/bash
方法二：
adduser [username]更傻瓜，适合新手
使用adduser会执行从/etc/skel目录下拷贝所有文件到主目录，所以不出问题的话所有.文件应该都会有
```

### 删除用户

```shell
userdel -r [username]
```

### 用户权限设置

```shell
visudo
修改sudo权限
```

/etc/sudoers sudo权限
/etc/passwd  用户信息

### 切换用户

```shell
su username 普通切换
su - username 完全切换，切换用户后会加载该用户的环境变量。
```

```shell
sudo [-d] [-u username]
```

```shell
id -u username 查看是否存在该用户
```

## 环境变量配置

> 问题：终端字体一个颜色，文件夹和文件颜色不区分？,且不显示当前你路径

拷贝其他用户的.bashrc文件到自己目录下，（其中有很多环境变量配置）然后手动执行source .bashrc，生效

但是重登后失效，需要一个.profile文件添加自动运行.bashrc的配置信息

.profile .bashrc .bash_profile的区别，见百度收藏

.profile和.bash_profile是一个作用的，注意同时都有的话，貌似默认会执行.bash_profile

bash_history bash_logout是干什么的？

## tar命令

大文件传还是先tar包，再传输
压缩 tar -czf 新建文件名.tar.gz 要被压缩的文件
解压 tar -xf  要解压的文件 -C 目录

解压时出现is in the future时间戳不对
加上--touch

发送scp fei@xxxx：~

尝试压缩后缀写成tar，但这样压缩包解压出来不是原来的文件。

## grep、cut命令

cat /home/fei/cpu2006-alpha/result/CPU2006.037.log | grep "base ref ratio"  | cut -d " " -f3-6
cat /home/fei/cpu2006-alpha/result/CPU2006.037.log | grep "base ref ratio"  | cut -d " " -f6
cat /home/fei/cpu2006-alpha/result/CPU2006.037.log | grep "base ref ratio"  | cut -d " " -f6 | cut -d "," -f1
cat /home/fei/cpu2006-alpha/result/CPU2006.037.log | grep "base ref ratio"  | cut -d " " -f6 | cut -d "," -f1 | cut -d "=" -f2

## apt命令

```shell
apt update package 更新软件源中的所有软件列表

apt list | grep package 显示软件包
apt list --upgradeable 显示可升级的软件包
apt list --installed 显示已安装的软件包

apt-get install package 安装软件
apt install -f package 修复依赖关系

apt upgrade package 升级软件包

apt show package 显示包的具体信息

apt remove package 卸载指定软件
apt automove  

apt purge package 

apt clean 
```

## 制作软连接

```shell
sudo ln -sf 被链接文件  软链接 

sudo ln -sf /usr/include/linux/stddef.h stddef.h

file stddef.h 

symbolic link to /usr/include/linux/stddef.h
```

## locate、find、which命令

全局查找

find / -name filename

查找文件信息

locate filename

查找安装路径

which 软件

## 进程管理命令

ps aux |grep

ps -ef |grep
