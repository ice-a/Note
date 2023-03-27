# 1.VMware安装

# 2.镜像选择与下载

镜像站

# 3.创建虚拟机

方法一：简易安装，直接启动

方法二：不想使用简易安装，选择稍后安装操作系统，硬件CD/DVD(SATA)的连接使用ISO映像文件，再启动。

# 4.VMware Tools安装

安装完成后界面自适应，可以复制，

# 5.设置root密码

# 6.vim下载

# 7.共享文件夹

已经设置了共享文件夹，

在/mnt/hgfs下却看不到共享文件夹

```shell
vmware-hgfsclient //查看是否设置好共享目录
df -h //看有没有挂载上
```

重启后共享文件夹不见了？

手动挂载

```shell
sudo vmhgfs-fuse .host:/ /mnt/hgfs/ -o allow_other -o nonempty
```

# 8.配置qemu的环境

ubuntu 16.04

编译qemu

linux glibc版本太低，编译需要高的版本

glibc更新到新版

strings /lib/x86_linux_gnu/libc.so.6 |grep GLIBC查看支当前的版本

wget + 地址  下载安装包

glibc编译的时候出现问题

python预装版本是2.7太低，至少需要3.6要再装一个版本

qemu gdb调试问题

gdb:no such file or directory

解决办法：把gdbinit拷贝到build/中，再在该目录下调试

[CSDN链接](https://blog.csdn.net/ReturningProdigal/article/details/100700313?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-100700313-blog-118693382.pc_relevant_sortByStrongTime&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-100700313-blog-118693382.pc_relevant_sortByStrongTime&utm_relevant_index=1)

> WARN: disabling coroutine pool for stack usage debugging
> warn: ignoring non-existent submodule ui/keycodemapdb
> warn: ignoring non-existent submodule tests/fp/berkeley-testfloat-3
> warn: ignoring non-existent submodule tests/fp/berkeley-softfloat-3
> warn: ignoring non-existent submodule meson
> warn: ignoring non-existent submodule dtc
> warn: ignoring non-existent submodule capstone
> warn: ignoring non-existent submodule slirp
> /home/fy/qemu6-system/scripts/git-submodule.sh: 96: git: not found
> /home/fy/qemu6-system/scripts/git-submodule.sh: failed to update modules
> Unable to automatically checkout GIT submodules ''.
> If you require use of an alternative GIT binary (for example to
> enable use of a transparent proxy), then please specify it by
> running configure by with the '--with-git' argument. e.g.
> 
> $ ./configure --with-git='tsocks git'
> 
> Alternatively you may disable automatic GIT submodule checkout
> with:
> 
> $ ./configure --with-git-submodules=validate
> 
> and then manually update submodules prior to running make, with:
> 
> $ scripts/git-submodule.sh update
> 
> 原因：没有下载git

# Issues

## 1、U盘连接

硬件USB控制器的连接USB兼容性选择USB3.0

选择您希望将U盘连接到的位置 ，选择虚拟机

## 2、关闭ubuntu中终端使用TAB键时BELL提示音

```shell
sudo vim /etc/inputrc
找到# set bell-style none 去掉注释
```

## 3、软件仓库官方源更换为国内源

> [ubuntu更换国内源_EmbededCoder的博客-CSDN博客_ubuntu换源](https://blog.csdn.net/u012308586/article/details/102953882)

## 4、虚拟机使用代理

### 方法一、共享主机VPN

> [Ubuntu虚拟机共享主机VPN（适用于NAT或桥接） - 简书](https://www.jianshu.com/p/6c7abd4adc9b)

### 方法二：直接在虚拟机中使用VPN

## 5、时间同步

虚拟机设置开启时间同步，系统时区改为上海

setting->Date&Time

## 6、屏幕锁定时间

setting->Privacy->Screen Lock

## 7、终端分屏

> [Ubuntu终端多窗口分屏Terminator_一只积极向上的小咸鱼的博客-CSDN博客_ubuntu终端分屏](https://blog.csdn.net/m0_49448331/article/details/121909760)

## 8、install报错

> fy@ubuntu:~/perfdbt$ sudo apt-get install gcc-aarch64-linux-gnu
> E: dpkg was interrupted, you must manually run 'sudo dpkg --configure -a' to correct the problem. 

## 9、ubuntu中文输入

安装中文输入法setting->Region & Language->Manage Installed Languages->Install/Remove Languages->勾选Chinese(simplified)，然后重启系统，Input Sources添加Chinese(Intelligent Pinyin)

## 10、扩展磁盘空间

[vmware虚拟机扩展磁盘空间](https://blog.csdn.net/qq_43212582/article/details/123193404)