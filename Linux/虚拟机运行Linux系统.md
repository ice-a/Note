## VMware安装

## 镜像下载

## 创建虚拟机

## VMware Tools安装

安装完成后界面自适应，可以复制，

## 设置root密码

## vim下载

## 共享文件夹

已经设置了共享文件夹，

在/mnt/hgfs下却看不到共享文件夹

首先，vmware-hgfsclient 确认已经设置好，接着 vmhgfs-fuse .host:/ /mnt/hgfs/ ，问题就解决了，这么搞文件权限有问题，属于root用户的

## 配置qemu的环境

编译qemu

linuxglibc版本太低，编译需要高的版本

glibc更新新版

strings /lib/x86_linux_gnu/libc.so.6 |grep GLIBC查看支当前的版本

wget + 地址  下载安装包

glibc编译的时候出现问题

python预装版本是2.7太低，至少需要3.6要再装一个版本