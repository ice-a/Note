# UOS系统安装

6A镜像版本：uniontechos-server-20-enterprise-1021-sw_64-20210119-1119-B5

6B镜像版本：uniontechos-server-20-enterprise-1030-sw_64-20210810-1018-B5

## 1.U盘启动盘制作，U盘作为系统盘启动

使用UltraISO格式化，写入镜像

## 2.设置启动项

1、插上u盘。

2、开机，等待加载界面进入bios。

进bios方法：下面出现进度条时按delete。

3、进入bios后设置第一启动项为u盘，第二启动项为硬盘，第三启动项为uefi。

4、查看有没有识别到u盘，若没有则重启，若识别到则保存并退出。

## 3.进入安装界面，开始初始化系统

1、选择语言：中文、英文

2、选择时区，设置时间（可以手动对下时间）。

3、选择语言，键盘布局。

4、设置计算机用户密码，root用户密码。密码必须包含大小写以及特殊符号，统一先设置为User@123。

5、网络进系统再配置。

6、选择组件

左边图形化界面服务器 右边不选

常规?

7、硬盘分区

选择全盘安装

## 4.重启

完成安装后，快速拔出安装介质，插上硬盘。

## 5.修改用户密码与root密码

```shell
# 修改密码
passwd
passwd root
su root & passwd 
sudo passwd root（第一次设置root密码时这样操作）
CURRENT PASSWORD：
NEW PASSWORD：
RETYPE NEW PASSWORD：
# 注意输入密码时，想用小键盘输入的话必须先按下小键盘锁定键，否则输入的密码不是123456
```

## 6.配置网络

注意有没有插上网线，不同网口对应不同的网卡。

ip：172.16.xxx.xxx 1~254,自己选择，注意不能和别的机器冲突，ping一下看看有没有空闲的ip再设置。
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

## 7.配置软件仓库源

```shell
# 仓库源的配置文件
/etc/apt/sources.list 
# 修改前先备份一下
cp -p sources.list sources.bak
# 这里不使用在线仓库，使用本地离线仓库。（在知识库中）
vim /etc/apt/sources.list
# 更新apt
apt update
```

## 8.更换内核(可选)

1、下载内核包并解压

```shell
dpkg-deb -x linux-image-4.19.0-sw64-server-225-gdb+_4.19.0-sw64-server-225-gdb+-18_sw_64.deb .
# deb包内容
/boot #内核
/etc
/lib #模块
/usr
```

2、拷贝内核及内核模块

```shell
cp boot/vmlinux.bin-4.19.0-sw64-server-225-gdb+ /boot
cp -r lib/modules/4.19.0-sw64-server-225-gdb+ /lib/modules/
```

3、生成对应的启动镜像

```shell
sudo update-initramfs -c -k 4.19.0-sw64-server-225-gdb+
# 生成的启动镜像在/boot/initrd.img-4.19.0-sw64-server-225-gdb+
```

4、修改开机引导：修改打印信息，修改启动选项（linux.boot，linux.vmlinux），复制并添加到submunu。

```shell
vim /boot/grub/grub.cfg


menuentry 'UnionTech OS Server 20 Enterprise GNU/Linux for gdb' --class uniontech --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-f95390a7-e836-4cc0-af2f-4aef16250e90' {
    set boot=(${root})/boot/
    echo    'Loading initial ramdisk ...'
    linux.boot  ${boot}/initrd.img-4.19.0-sw64-server-225-gdb+
    echo    'Loading Linux 4.19.0-sw64-server-225-gdb+ ...'
    linux.vmlinux   ${boot}/vmlinux.bin-4.19.0-sw64-server-225-gdb+ root=UUID=f95390a7-e836-4cc0-af2f-4aef16250e90 ro  quiet splash DEEPIN_GFXMODE=$DEEPIN_GFXMODE
    boot
}
submenu 'Advanced options for UnionTech OS Server 20 Enterprise GNU/Linux' $menuentry_id_option 'gnulinux-advanced-f95390a7-e836-4cc0-af2f-4aef16250e90' {
    menuentry 'UnionTech OS Server 20 Enterprise GNU/Linux, with Linux 4.19.0-sw64-server' --class uniontech --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-4.19.0-sw64-server-advanced-f95390a7-e836-4cc0-af2f-4aef16250e90' {
        set boot=(${root})/boot/
        echo    'Loading initial ramdisk ...'
        linux.boot  ${boot}/initrd.img-4.19.0-sw64-server
        echo    'Loading Linux 4.19.0-sw64-server ...'
        linux.vmlinux   ${boot}/vmlinux.bin-4.19.0-sw64-server root=UUID=f95390a7-e836-4cc0-af2f-4aef16250e90 ro  quiet splash DEEPIN_GFXMODE=$DEEPIN_GFXMODE
        boot
    }   
        menuentry 'UnionTech OS Server 20 Enterprise GNU/Linux for gdb' --class uniontech --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-f95390a7-e836-4cc0-af2f-4aef16250e90' {
        set     boot=(${root})/boot/
        echo    'Loading initial ramdisk ...'
        linux.boot      ${boot}/initrd.img-4.19.0-sw64-server-225-gdb+
        echo    'Loading Linux 4.19.0-sw64-server-225-gdb+ ...'
        linux.vmlinux   ${boot}/vmlinux.bin-4.19.0-sw64-server-225-gdb+ root=UUID=f95390a7-e836-4cc0-af2f-4aef16250e90 ro  quiet splash DEEPIN_GFXMODE=$DEEPIN_GFXMODE
        boot
        }
    menuentry 'UnionTech OS Server 20 Enterprise GNU/Linux, with Linux 4.19.0-sw64-server (recovery mode)' --class uniontech --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-4.19.0-sw64-server-recovery-f95390a7-e836-4cc0-af2f-4aef16250e90' {
        set boot=(${root})/boot/
        echo    'Loading initial ramdisk ...'
        linux.boot  ${boot}/initrd.img-4.19.0-sw64-server
        echo    'Loading Linux 4.19.0-sw64-server ...'
        linux.vmlinux   ${boot}/vmlinux.bin-4.19.0-sw64-server root=UUID=f95390a7-e836-4cc0-af2f-4aef16250e90 ro single 
        boot
    }
}
```

## 9.硬盘挂载(可选)

1、单次挂载硬盘

```shell
# 硬盘会自动挂载到/media/deepin/xxx，对应设备为/dev/sxx
lsblk
# 解除挂载
umount /dev/sxx
# 将硬盘重新挂载到/mnt/
sudo mount /media/deepin/xxx /mnt/
```

2、硬盘自动挂载

```shell
blkid # 查看设备的UUID和磁盘格式
vim /etc/fstab # 写入硬盘信息和挂载点
```

## 10.创建用户，链接用户主目录(可选)

2、重新创建用户，并链接到/mnt相应文件夹

```shell
# 重新创建用户
adduser username
# 删除主目录
rm -rf /home/user
# 创建软链接
sudo ln -sf /mnt/user /home/usr
# 注意最好注意/mnt/user及其内容的权限要是自己的，不是的话更改，软链接权限最好也改下
sudo chown fei:fei /home/fei  # 改软链接
sudo chown fei:fei /home/fei/ # 改文件夹内容
```

## 10.重启ssh服务

> 安装完毕后首次通过ssh连接提示：
> 
> ssh: connect to host 172.16.129.114 port 22: Connection refused

解决方案：重启ssh服务

```shell
sudo service ssh restart 
```

自动重启服务？

# Issues

## 1、deepin系统未激活

> 终端提示：You system is not activated .Please activate as soon as possible for normal use . 

问题原因：deepin系统未激活，没有授权

解决方案：1、向deepin要激活

2、删除与该提示相关的软件（包）

```shell
dpkg -l|grep license 
dpkg -r 上条命令搜到的包名
apt-get remove 上条命令搜到的包名
强制删除
dkpg --force-all -r 包名
完整删除命令
dkpg -P 包名
```

删除完后重启服务器，即可生效

## 2、终端中文乱码

> 终端下中文乱码

问题原因：deepin系统自带的终端有问题。

解决方案：下载deepin-terminal并设置为默认应用。

```shell
apt install deepin-terminal
```

## 3、提示密码不正确

> su root 及 sudo 命令输入密码提示密码不正确

问题原因：大概率是修改密码时用小键盘输入123456且没有按下小键盘锁定键。

解决方案：再次修改密码，不按下小键盘锁定键输入123456。

## 4、密码安全检查

> 修改密码时提示：BAD PASSWORD：it is too simplistic/systematic

问题原因：有密码安全检查

解决方案：

```shell
/etc/pam.d/passwd # 密码检查
@include common-passwd
/etc/pam.d/common-passwd 
#pam_carcklib.so 是密码规则，让它用pam_unix.so这个最简单的规则。
```

## 5、ssh断连

> 长时间没有输入，自动断开，需要重新登录：
> 
> Connection to xxx closed by remote host.
> Connection to xxx closed

解决方案：

```shell
/etc/ssh/sshd_config
# 下面两个都注释掉
#ClientAliveInterval 900
#ClientAliveCountMax 1
ClientAliveInterval 60
ClientAliveCountMax 3
```

# win10

## 1、win10升级

window10升级window11，查看本机没有打开tpm，进入bios没有打开tpm，

怀疑是否要升级bios，或者就不支持tpm2.0

## 2、win10修改设备连接声音

右键扬声器，选择程序事件

## 3、win10 搜索

win10 搜索开启高级

## 4、win+L键锁屏