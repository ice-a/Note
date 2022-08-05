# u盘、镜像的挂载

## 查看设备有没有识别到，有没有挂载

```shell
lsblk   //默认在/media/系统用户/
df -h
fdisk -l
```

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
mount -t vfat /dev/sda1  /mnt/usb
```

## 解除挂载

```shell
umount /dev/sda1
       /mnt/usb
```

## md5检验

```shell
md5sum .iso
```

## 格式问题

ntfs格式linux和windows通用的文件系统格式

ext格式是linux的文件系统格式

exfat是闪存文件系统
