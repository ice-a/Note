uos系统安装，镜像版本：uniontechos-server-20-enterprise-1030-sw_64-20210810-1018-B5

一般步骤

# 1.U盘启动盘制作，U盘作为系统盘启动

使用ultraISO格式化，写入镜像

# 2.启动项设置

插u盘

进bios 方法：下面出现进度条时按delete

进bios设置u盘第一启动，硬盘第二，uefi第三，

# 3.进入安装界面，初始化系统

1选择语言 中文、英文，

2选择时区，设置时间

3选择语言，键盘布局

4设置计算机用户密码，root用户密码

网络进系统再配置

5.选择组件

左边图形化界面服务器 右边不选

常规?

6.硬盘分区

全盘安装

# 4.重启，快速拔出安装介质

# 5.修改密码

```shell
passwd
passwd root
su root & passwd 
sudo passwd root（第一次设置root密码时这样操作）
```

# 6.配置网络

ip：172.16.xxx.xxx 1~254,自己选择，注意不能和别的机器冲突，ping一下看看有没有空闲的ip，
我们机房是129，周枫他们机房是131。
子网掩码：255.255.255.0 /24
网关：172.16.xxx.254 254

查看网关

```shell
ifconfig # windows下是ipconfig
ip addr （show）
```

手动修改 ip addr add

后续操作

# 7.配置软件仓库

修改前先备份一下

```shell
cp -p sources.list sources.bak
vim /etc/apt/sources.list
apt update
```

# 8.uos的问题

> 问题：deepin未授权
> 
> You system is not activated .Please activate as soon as possible for normal use . 

```shell
dpkg -l|grep license 
dpkg -r 上条命令搜到的包名
apt-get remove 上条命令搜到的包名
强制删除
dkpg --force-all -r 包名
完整删除命令
dkpg -P 包名
```

删除完后重启服务器，生效

> 终端下中文乱码

```shell
apt install deepin-terminal
```

修改默认程序终端为新的终端

# 深度终端远程服务器管理

# win10升级

window10升级window11，查看本机没有打开tpm，进入bios没有打开tpm，

怀疑是否要升级bios，或者就不支持tpm2.0