# binfmt介绍

# 配置方法

## 基于Redhat的发行版

如：kylin系统

### 配置文件

```shell
/proc/sys/fs/binfmt_misc/<name>
    status # 配置状态
    register # 注册配置
```

### 配置binfmt命令

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

如：UOS系统

### 配置步骤

使用binfmt-support软件（按照qemu中的README文件配置）

1、加载模块

```shell
insmod binfmt_misc.ko # 默认模块已加载
```

2、安装binfmt-support

```shell
sudo apt-get install binfmt-support # 默认binfmt-support已经安装
```

3、注册配置，或者修改配置。配置文件/usr/share/binfmts/qemu-86_64需要从别的地方拷过来，并且修改指定的qemu路径。建议不要乱改，直接用/usr/bin/ 目录下默认安装位置的qemu-xxx。

```shell
# binfmt 配置相关的目录
/usr/share/binfmts/<name>
```

5、更新binfmt

```shell
# 注意，当可执行程序更新时，最好也重新启用下binfmt配置
update-binfmts --display                # 查看配置
sudo update-binfmts --unimport qemu-xxx # 禁用配置
sudo update-binfmts --import qemu-xxx   # 启用配置
```

> 参考文章：