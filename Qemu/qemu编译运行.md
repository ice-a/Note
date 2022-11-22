# 编译方法

## 步骤

1、git clone下载源代码。Readme.rst：公版自带，qemu的编译方法，README：自己写的，有编译参数、测试命令

（编译之前改文件拥有者sudo chown -R fei:fei /home/fei/qemu-6.2.0-work1/？），configure中搜Standard options中可以看到编译选项（./configure --help）搜default_feature为默认参数。

```shell
Usage: configure [options]
Options: [defaults in brackets after descriptions]

Standard options:
  --help                   print this message # 打印选项信息
  --prefix=PREFIX          install in PREFIX [$prefix] # 安装路径，一般写/usr就会安装在/usr/bin/目录下。
  --interp-prefix=PREFIX   where to find shared libraries, etc.
                           use %M for cpu name [$interp_prefix] # %M是cpu name，即arch
                 # 动态库路径，执行动态编译文件需要用到。公版默认（/usr/gnemul/qemu-%M），自己的一般写成/etc/qemu-binfmt/%M
  --target-list=LIST       set target list (default: build all non-deprecated) # 设定目标架构
Advanced options (experts only):
  --cross-prefix=PREFIX    use PREFIX for compile tools, PREFIX can be blank [$cross_prefix] # 使用指定编译工具，交叉编译器
  --extra-cflags=CFLAGS    append extra C compiler flags QEMU_CFLAGS # gcc编译参数
  --cross-cc-ARCH=CC       use compiler when building ARCH guest test cases
  --libdir=PATH            install libraries in PATH
  --libexecdir=PATH        install helper binaries in PATH
  --sysconfdir=PATH        install config in PATH/$qemu_suffix # 
  --disable-werror         disable compilation abort on warning
  --disable-blobs          disable installing provided firmware blobs

Optional features, enabled with --enable-FEATURE and
disabled with --disable-FEATURE, default is enabled if available
(unless built with --without-default-features):
  linux-user      all linux usermode emulation targets
  docs            build documentation
```

3、新建build文件夹，新建config.sh脚本，添加configure脚本的参数，运行的话参数设置O2，调试程序参数设置O0，测试性能需要把-g、debug等调试相关选项都去掉。更改脚本文件权限，增加执行权限chmod +x。

4、./config.sh配置环境（编译日志config.log），make -j40编译出可执行文件，make install安装可执行文件到指定路径下。

## 命令

```shell
mkdir build # 新建build文件夹
cd build
vim config.sh # 新建config.sh脚本，添加configure脚本的参数
chomd +x config.sh
./config.sh
make -j40
(sudo) make install
```

# Issue

## 缺少动态库

执行动态编译的程序需要拷贝对应架构的动态库到qemu编译参数的指定路径下。
路径：--interp-prefix=/etc/qemu-binfmt/%M，或通过qemu-xx -h/-help/--help查看。

以下报错均由此原因导致

> 报错信息：
> 情况一：
> 
> qemu-x86_64:Could not open '/lib64/ld-linux-x86-64.so.2':No such file or directory
> 情况二：
> 
> qemu-sw64:/lib/ld-linux.so.2:Invalid ELF image for this architecture
> 情况三：
> 
> /home/gao/spec2006_x86_ubuntu20.04/bin/specperl: error while loading shared libraries: libnsl.so.1: cannot open shared object file: No such file or directory.
> spec的命令要用qemu-x86_64执行，需要动态库。

## 编译与安装的md5校验

```shell
# md5校验
which qemu-xx # /usr/bin/qemu-xx
md5sum qemu-xx
md5sum /usr/bin/qemu-xx
# 两个md5知不一样是因为make install中有一个stripped操作，这一步并不是每一回都有。
```

## 工具链ld有问题

> 报错信息：

错误原因：系统自带的binutil，gcc8.3平时用没问题，但编译qemu会有问题，因此要重新编译。

解决方案：重新编译安装binutil工具链

1.编译安装binutil工具链

获取binutils-2.31.1源码包，configure配置环境参数：

```shell
./configure --target=sw_64-linux-gnu --prefix=${PREFIX}/usr --with-sysroot=${PREFIX} --disable-werror --with-cpu=${CPU}
```

(./configure --target=sw_64-linux-gnu --prefix=/home/deepin/util/usr --with-sysroot=/home/deepin/util --disable-werror --with-cpu=sw6a)
其中TARGET=sw_64-linux-gnu，PREFIX=/.. CPU=sw6a
PREFIX 表示安装路径，自己选择

```shell
make && make install 
```

编译，安装，会在PREFIX路径下生成可执行文件

2.添加环境变量

```shell
# 执行脚本
# 方法一：
source env.sh   
# 方法二：
. ./env.sh
```

> env.sh

```shell
unset PATH
unset LD_LIBRARY_PATH # 链接器库文件路径
unset LIBRARY_PATH    # 动态库文件路径
export PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games:/sbin:/usr/sbin
export LD_LIBRARY_PATH=/lib/sw_64-linux-gnu:/usr/lib/sw64-linux-gnu:$LD_LIBRARY_PATH
export PATH=/home/deepin/util/usr/bin:$PATH # binutils安装的目录
```

注意，也可以把这些命令写进.bashrc中，这样每次登录都会执行。

## 循环打印configure信息

> 报错信息：
> 
> [0/1]Regenerating build files
> .......

本机时间太过超前，在未来。修改本机时间后，重新克隆qemu源码。

# SWREACH编译器优化选项

申威编译器目前支持O0、O1、O2、Os、O3选项

-O或-O0生成没有优化的可执行代码

-O1或-O2生成优化的可执行代码

-O3生成高级优化的可执行代码

-Os在支持-O2所有选项的同时不会显著增加代码规模，其可以有更多优化代码空间

# 运行

系统级启动 先把core3-hmcode拷贝进/build/pc-bios/文件夹中，运行启动脚本 ./run.sh
 ./run.sh
/home/fei/qemu-6-6.2/qemu6/build/qemu-system-sw64 -machine core3 -m 512 -smp 1 -serial stdio -drive file=./sw_qemu/test6b.raw,if=virtio -kernel ./sw_qemu/vmlinux -append "root=/dev/vda2 rw console=ttyS0,115200n8 ignore_loglevel systemd.unit=multi-user.target numa=off" -vnc :7

file pc-bios写成的raw文件，vmlinux是内核

用户级启动  
qemu-6-6.2/qemu6/build/qemu-sw64 -cpu core3 自己写个Hello

启动虚拟机
图形化界面，看README里有写
