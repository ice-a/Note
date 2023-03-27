# binfmt介绍

参考文章：
[binfmt.d 中文手册](https://www.wenjiangs.com/doc/systemd-binfmt-d)
[linux下使用binfmt_misc设定不同二进制的打开程序](https://blog.csdn.net/whatday/article/details/88299482/)

# 配置方法

## 基于Redhat的发行版

如：centos、kylin系统

### 配置文件

```shell
/proc/sys/fs/binfmt_misc/<name>
    status   # 配置状态
    register # 注册配置
```

### 配置命令

```shell
# 注册qemu-x86_64配置
sudo sh -c 'echo ":qemu-x86_64:M:0:\x7f\x45\x4c\x46\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x3e\x00:\xff\xff\xff\xff\xff\xfe\xfe\xfc\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-x86_64:OCF" > /proc/sys/fs/binfmt_misc/register'

cat /proc/sys/fs/binfmt_misc/qemu-x86_64 # 查看配置文件
sudo sh -c 'echo -1 >/proc/sys/fs/binfmt_misc/qemu-x86_64' # 删除qemu-x86_64配置
sudo sh -c 'echo 1 >/proc/sys/fs/binfmt_misc/qemu-x86_64'  # 启用qemu-x86_64配置
sudo sh -c 'echo 0 >/proc/sys/fs/binfmt_misc/qemu-x86_64'  # 禁用qemu-x86_64配置
```

见百度收藏

用于qemu运行二进制文件，改文件不知道是不是一个效果

## 基于Debian的发行版

如：ubuntu、Uos、欧拉系统

方法一：沿用上述配置方法

方法二：使用binfmt-support

### binfmt-support

1、加载模块

```shell
insmod binfmt_misc.ko # 默认模块已加载
```

2、安装binfmt-support

```shell
sudo apt-get install binfmt-support # 默认binfmt-support已经安装
```

3、注册配置，或者修改配置。配置文件需要指定qemu路径。建议不要乱改，直接用/usr/bin/ 目录下默认安装位置的qemu-xxx。

```shell
# binfmt 配置文件所在目录
/usr/share/binfmts/<name>

# 配置文件
# qemu-x86_64
package qemu-user-static
interpreter /home/fei/qemu6/build/qemu-x86_64 # 解释器路径
magic \x7f\x45\x4c\x46\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x3e\x00 # 查看命令 od -x --read-bytes=20 可执行文件
offset 0  # 偏移
mask \xff\xff\xff\xff\xff\xfe\xfe\xfc\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff # 匹配规则，1代表必须完全匹配，0代表不必完全匹配
credentials yes
fix_binary yes

# qemu-sw64
package qemu-user-static
interpreter /home/fei/qemu6/build/qemu-sw64 # 解释器路径，想要给额外的参数就需要使用脚本
magic \x7f\x45\x4c\x46\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x16\x99 # 查看命令 od -x --read-bytes=20 可执行文件
offset 0  # 偏移
mask \xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xef\xff # 9906与9916均能够匹配
credentials yes
fix_binary yes
```

5、更新binfmt

```shell
# 注意，当可执行程序更新时，最好也重新启用下binfmt配置
update-binfmts --display                # 查看配置
sudo update-binfmts --unimport qemu-xxx # 禁用配置
sudo update-binfmts --import qemu-xxx   # 启用配置
```

> 参考文章：