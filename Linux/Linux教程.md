> 鸟哥的Linux私房菜基础学习篇

# 第一章 Linux的规则与安装

## 1.正确关机方法

window单人+多任务，关机不会对他人造成影响，Linux，程序和服务都是后台运行，所以需要正常关机

### 观察系统使用状态

```shell
# 查看目前有谁在线
who 
# 查看网络联机状态
netstat -a
# 查看后台执行的程序
ps aux
```

### 将数据同步写入硬盘中的命令

```shell
sync 
```

### 关机命令

```shell
/sbin/shutdown [-krhc] [时间] [警告信息]
```

## 2. 主机规划与磁盘划分

在Linux系统中，每个设备都被当成一个文件来对待。

# 第五章 Linux的文件权限与目录配置

# 第六章 Linux文件与目录

## 6.1 目录与路径

### 6.1.1 绝对路径与相对路径

绝对路径 根目录开始

相对路径 当前目录开始

### 6.1.2 目录相关的命令

cd （change directory，切换目录）

在路径后面加/表示该目录下的

```shell
[相对路径/绝对路径]   # 目录
.   ./               # 当前目录
..  ../              # 上一层目录
/                    # 根目录
~  ~/                # 当前用户的主文件目录
cd 空气              # 也是回到当前用户的主文件目录
~username            # user用户的主文件目录
-                    # 刚刚的目录
```

pwd（查看当前/文件所在的目录）

```shell
pwd [file] # 查看当前/文件所在路径
realpath file # 查看文件的完整路径
```

mkdir

```shell
mkdir
```

lsblk 查看所有硬件
df -h 查看硬件使用率
/home 用户主目录 （服务器+客户端每个人一个用户）
/dev  所有设备信息
/media/user_name/ 下面有u盘文件
/bin 放各种系统自带的命令，放在里面就可以用tab提示
/opt 存放第三方软件
/lib 放各种库文件 

# 第七章 Linux磁盘与文件系统管理

## 符号链接

Hard Link (实体链接, 硬式链接或实际链接)

Symbolic Link (符号链接)

```shell
fy@ubuntu:~$ ln [-sf] 来源文件 目标文件
选项与参数：
-s, --symbolic：不加此参数默认hard link，加就是symbolic link
-f ：如果目标文件存在时，就主动的将目标文件直接移除后再创建！
```

修改符号链接拥有者

chown -h [owner]:[group] Filename

> [Linux修改软连接拥有者及所属群_wjc_grace的博客-CSDN博客_修改软连接的所属用户](https://blog.csdn.net/qq_39557270/article/details/121673420)

# 用户管理

## 添加用户useradd,adduser

```shell
fy@ubuntu:~$ su root
方法一：
fy@ubuntu:~$ useradd -m [username] 需要加参数，是不会自动创建
fy@ubuntu:~$ passwd  [username]
方法二：
adduser [username]更傻瓜，适合新手

特殊情况：
# 不知道root密码，但可以登其他人账户，sudo adduser fei没有设密码这一步（116之前有过该账户只是删除了）
sudo adduser fei
sudo passwd fei
```

> [使用useradd创建新用户无法创建用户家目录的问题_会飞的鱼aaaa的博客-CSDN博客_useradd无法创建目录](https://blog.csdn.net/qq_40259620/article/details/124970751)

### useradd参考档

```shell
fy@ubuntu:~$ useradd -D
GROUP=100    # 默认的群组
HOME=/home   # 默认的主文件夹所在目录
INACTIVE=-1  # 密码失效日，在 shadow 内的第 7 栏
EXPIRE=      # 帐号失效日，在 shadow 内的第 8 栏
SHELL=/bin/sh # 默认的 shell
SKEL=/etc/skel # 使用者主文件夹的内容数据参考目录
CREATE_MAIL_SPOOL=no # 是否主动帮使用者创建邮件信箱（mailbox）
```

## 删除用户

```shell
userdel -r [username] # 删除主目录
```

## 用户权限设置

```shell
# 修改sudo权限
方法一：
sudo visudo
方法二:
sudo vim /etc/sudoers
```

/etc/sudoers sudo权限
/etc/passwd  用户信息

## 切换用户

```shell
su username # 普通切换
su - username # 完全切换，切换用户后会加载该用户的环境变量。
```

```shell
sudo [-d] [-u username] # 
```

```shell
id -u username # 查看是否存在该用户
```

# 环境变量配置

```shell
env # 显示所有环境变量
```

> 问题：终端字体一个颜色，各类型文件颜色不区分？且不显示当前所在的完整路径

会不显示用户名，需要vim /etc/passwd ，将/bin/sh 改为/bin/bash

拷贝其他用户的.bashrc文件到自己目录下，（其中有很多环境变量配置）然后手动执行

```shell
source .bashrc
./.bashrc      # 权限不够？找不到命令？因为不是脚本？
. .bashrc
```

，生效，但是重登后失效，需要一个.profile文件添加自动运行.bashrc的配置信息。

.profile .bashrc .bash_profile的区别，见百度收藏

.profile和.bash_profile是一个作用的，注意同时都有的话，貌似默认会执行.bash_profile

bash_history bash_logout是干什么的？

# ls（list directory contents）

自用alias设置

```shell
#alias ll='ls -l'
#alias la='ls -A'
#alias l='ls -CF'
alias l='ls -1F'                          # 行排列显示list 
alias la='ls -1A'                         # 行排列显示全部文件list all
alias ll='ls -hlF --time-style=long-iso'  # 显示详细信息list using a long listing format
alias lal='ls -AhlF --time-style=long-iso'# 显示全部文件，隐藏文件,不包含.和..
alias lh='ls -1d .!(|.)'                  # 只显示隐藏文件list hidden files 
alias lhl='ls -dhlF --time-style=long-iso .!(|.)' # 只显示隐藏文件详细信息
alias gdb='gdb -q'                         # gdb安静模式
```

# cp命令

```shell
cp -p # 连同文件的属性、时间一起复制，备份常用
# 想让文件复制过来自己是拥有者，就不切换用户直接去相关目录下复制就行。
```

# tar命令

大文件传还是先压缩，再传输

```shell
# 压缩 
tar -czf 压缩包名.tar.gz 要压缩的文件
# 解压 
tar -xf  要解压的文件 [-C 路径] # 默认解压在当前目录下，-C可以指定解压目录
```

解压时打印is in the future。这是因为tar包的时间戳来自未来，机器认为时间不对。

解决方法：一、加上--touch 二、调整本机时间戳。

文件由三种时间戳。

```shell
date # 查看本机时间
date -s "xxxx-xx-xx xx:xx:xx" # 调整本地时间戳
```

尝试压缩后缀写成tar，但这样压缩包解压出来不是原来的文件。

# unzip命令

```shell
unzip xxx.zip
```

# grep、cut、wc命令

```shell
grep -rn 'xxx'  [path] # 默认为. 选取包含xxx的行，n显示行
grep -v  'xxx'          # 去掉包含xxx的行
grep --binary-files=without-match # 不匹配二进制文件
grep |wc -l   # 统计行数 
```

```shell
cut -d '分隔符' -f 数字 # 以x为分隔符取第x段 
```

# apt命令

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

# 制作符号链接

```shell
sudo ln -sf 被链接文件 符号链接 
# 示例
sudo ln -sf /usr/include/linux/stddef.h stddef.h

file stddef.h 

symbolic link to /usr/include/linux/stddef.h
```

修改符号链接（软连接）拥有者和所属用户组

> [Linux修改软连接拥有者及所属群_wjc_grace的博客-CSDN博客_修改软连接的所属用户](https://blog.csdn.net/qq_39557270/article/details/121673420)

- 给ln命令加上-s选项，则建立软链接。
  
  格式：ln -s [真正的文件或者目录] [链接名]
  
  [链接名]可以是任何一个文件名或者目录名，并且允许它与原文件不在同一个文件系统中。
  
  如果[链接名]是一个已经存在的文件，将不做链接。
  
  如果[链接名]是一个已经存在的目录，linux系统会分两种情况自行进行处理：
  
  若链接指向的是一个文件名，系统将在已经存在的目录下建立一个与源文件名同名的符号链接文件
  
  若链接指向的是一个目录名，系统将在已经存在的目录下建立一个与源目录名同名的符号链接文件
  
  总之，建立软链接就是建立了一个新文件。当访问链接文件时，系统就会发现它是个链接文件，系统读取链接文件找到真正要访问的文件然后打开。

# locate、find、which命令

find 查找文件名

```shell
find [path ..] -name [filename] # 在指定目录下查找文件
# 注意：find 在查找时，把目录也当成文件处理，会查找并处理目录名，并不是只处理文件名。
```

查找文件信息

```shell
locate filename
```

查找命令的路径

```shell
which 命令
```

# 进程管理命令

ps

```shell
ps aux   # 简单显示当前系统进程
ps -elf  # 详细
```

top

```shell
top     # 查看所有用户进程
top -u  # 查看指定用户进程
```

nohup

```shell
nohup command &
```

kill & pkill

```shell
kill -9 进程号
pkill -u username
```

# 环境变量

```shell
/etc/profile
```

为什么linux执行脚本要加./

指在当前目录下寻找该文件，不加则会在PATH变量默认路径下寻找

三种方法

```shell
./test.sh
source test.sh # 不需要脚本具有执行权限
```

> https://segmentfault.com/a/1190000037797344?utm_source=tag-newest

# 如何在linux终端一次性执行多个命令

```shell
# 分号(;)运算符
# 分号(;)运算符允许你连续执行多个命令，而不管前面的每个命令是否成功

# 逻辑与运算符(&&)
# 如果希望第二个命令仅在第一个命令成功后运行，请使用逻辑与运算符分隔这些命令
# 建议在大多数情况下使用逻辑与运算符，而不是分号运算符(;)

# 逻辑or运算符(||)
# 有时你可能希望在第一个命令不成功时执行第二个命令，为此，使用逻辑or运算符

# 组合多个运算符
# 可以在命令行上组合多个运算符
```

# 深度系统终端（deepin-terminal）远程管理功能

添加服务器

该功能实际就是执行自动登录脚本，自行写一个类似的脚本后

在.bashrc中将自动登录脚本的路径添加到环境变量PATH中。

# 命令行语法

下表描述用于指示命令行语法的表示法。

| 表示法          | 说明              |
| ------------ | --------------- |
| 不含方括号或大括号的文本 | 必须按所显示键入的项。     |
| 尖括号`<>`      | 必须为其提供值的占位符。    |
| 中括号`[]`      | 可选项。            |
| 大括号`{}`      | 一组必需的项。         |
| 数显 `\|`      | 互斥项的分隔符。 只能选择一个 |
| 省略号 `…`      | 可重复使用多次的项。      |
| 小括号`( )`     | 选项默认值，只用于{}中    |
| 星号`*`        | 任意              |

# Linux命令行符号总结

左边的内容重定向到右边的文件会覆盖右边的文件内容
示例1：echo hello>test.txt#输出hello到标准输出并重定向到test.x文件
如果test.tx不存在会自动创建如果存在会覆盖内容
示例2：Is test.txt 2>/dev/null 查看test.tx文件明细，并将标准错误输出到黑洞（不输出错误）注意文件描述符必须和s号连在一起，中间没空格
s>左边的内容重定向到右边的文件会追加右边的文件
示例：echo hello>>test.txt#输出hello到标准输出并重定向到test.x文件
如果test.tx不存在会自动创建如果存在会追加到文件后面
<右边的内容重定向到左边
示例1：mysql-uroot-proot b2b< source.sql 将source.sq文件放入到mysq执行示例2：cat<testtxt 将test.tx文件内容重定向到ca命令
多行注释she/脚本xxx代表任意行she/脚本，EOF可以是任意字符，但是结束必须也是该字符，且结束字符在开头独占一行
：<<EOF
xxxx
Xx
#匿名文件xxxx代表任意行字符EOF代表结束符，可以是任意字符
<<EOF xXXX EOF示例：cat <<EOF>testtxt
>hello>world
>EOF
#cat一个匿名文件，然后将匿名文件重定向到test.x文件