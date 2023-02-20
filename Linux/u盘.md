# U盘、镜像的挂载

## 查看设备有没有识别到，有没有挂载，挂载点

```shell
lsblk   # 默认挂载位置在 /media/当前用户/分区UUID
df -h
fdisk -l
```

注意：电脑休眠时，识别不到插入的U盘
disk硬盘，device分区

## 给硬盘分区

```shell
fdisk  /dev/sda
```

## 格式化硬盘

```shell
mkfs -t ext3/4 /dev/sda1
```

## 挂载u盘

```shell
mount -t vfat /dev/sda1  /mnt/usb # mount 设备分区 挂载点
```

## 解除挂载

```shell
umount /dev/sda1   # 设备分区或者挂载点均可
       /mnt/usb
```

## md5检验

```shell
md5sum xxx.iso
```

## 格式问题

ntfs格式linux和windows通用的文件系统格式

ext格式是linux的文件系统格式

exfat是闪存文件系统

> mount:/home/fei/spec2006: WARNING: device write-protected, mounted read-only.

解决办法：

```shell
mkfs.ext4 spec2006.sio # 没用
```

这个镜像应该本身就是只读的。就是只读的，安装没问题。

# 开机自动挂载

```shell
# 查看分区uuid
# 方式一：by-uuid
ls -l /dev/disk/by-uuid
# 方式二：blkid命令
sudo blkid 
```

配置开机挂载

```shell
sudo vim /etc/fstab  
# 新增
UUID=e965e17a-b1f8-4eeb-a23d-7b369df66f45 /data                    xfs    defaults  0 0
```

# ISSUE

>Error unmounting block device 8:20: GDBus.Error:org.freedesktop.UDisks2.Error.DeviceBusy: Error unmounting /dev/sdb4: target is busy