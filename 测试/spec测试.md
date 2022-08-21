file bin/specperl 可以看出spec是在哪里编译的
file benchspec/CPU2006/400.perlbench/build/build_base_sw6.0000 可以看出题目是用的什么编译器
gcc.cfg是配置文件，编译运行配置

这次测试是在x86上用sw/alpha编译器编译题目，然后在sw上测，运行spec时ps -ef|grep spec可以看出会调用qemu-x86_64运行，然后在调用相应qemu跑题

# 环境部署

## 1.编译binutil工具链

gcc8.3编译qemu会有问题，因此要重新编译

获取binutils-2.31.1源码包，配置参数：

```shell
./configure --target=sw_64-linux-gnu --prefix=${PREFIX}/usr --with-sysroot=${PREFIX} --disable-werror --with-cpu=${CPU}
```

(./configure --target=sw_64-linux-gnu --prefix=/home/deepin/util --with-sysroot=/home/deepin/util --disable-werror --with-cpu=sw6a)
其中TARGET=sw_64-linux-gnu，PREFIX=/.. CPU=sw6a
PREFIX 表示安装路径，自己选择

```shell
make && make install 
```

编译，安装，会在PREFIX路径下生成可执行文件

## 2.设置LD环境变量

```shell
# 方法一：
source env.sh  
# 方法二：
. ./env.sh
```

> env.sh

```shell
unset PATH
unset LD_LIBRARY_PATH
unset LIBRARY_PATH
export PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games:/sbin:/usr/sbin
export LD_LIBRARY_PATH=/lib/sw_64-linux-gnu:/usr/lib/sw64-linux-gnu:$LD_LIBRARY_PATH
export PATH=/home/deepin/util/usr/bin:$PATH
```

注意，也可以把这些命令写进.bashrc中，这样每次登录都会识别到。

> 问题
> 
> spec 出现
> home/deepin/cpu2006/bin/runspec: line 15: ::runspec_time: command not found
> /home/deepin/cpu2006/bin/runspec: line 17: syntax error near unexpected token `{'
> /home/deepin/cpu2006/bin/runspec: line 17: `if (exists ENV{'SPECDB_PWD'}) {'
> 
> 说明binfmt没有配置。本地从头编spec比较复杂，都是用的别人在x86上编译好的spec，所以先要在sw上跑spec的话还是需要配置binfmt，用qemu-x86_64去编译运行spec命令，只是需要保证题目是sw本地的就行。ps -ef|grep qemu可以看到执行spec命令的是qemu-x86_64。file bin/specinvoke 可以看到这个是x86-64上编译的。

## 3.配置binfmt

主要是用qemu启动，改文件不知道是不是一个效果

按照qemu中的READEME文件配置，

1、模块已加载

2、binfmt-support已经安装

3、注册配置，或者修改配置。/usr/share/binfmts/qemu-86_64需要从别的地方拷过来，并且修改指定的qemu路径，update-binfmts --display 查看是否已经配置好（或者cat /proc/sys/fs/binfmt_misc/<name>）。不建议乱改，建议直接用/usr/bin/  目录下默认的qemu。

```shell
/usr/share/binfmts/<name>
/proc/sys/fs/binfmt_misc/<name>
```

4、qemu-binfmt需要动态库文件x86_64，需要拷贝到qemu配置参数--interp-prefix=/etc/qemu-binfmt/%M的路径下。

> 报错：qemu-x86_64:Could not open '/lib64/ld-linux-x86-64.so.2':No such file or directory

5、更新binfmt

```shell
update-binfmts --unimport qemu-xxx # 注意，当可执行程序更新时，最好也更新下binfmt配置
update-binfmts --import qemu-xxx
update-binfmts --display
```

## 4.Qemu编译

注意：跑测试时，配置参数里，编译器选项把-g相关去掉，用O2选项，后面的debug相关参数也都去掉。

sudo make install 

# spec安装，编译题目

spec原始代码获取并安装，编译题目
源代码在11.127/mnt/testpackage下spec2006.iso，mount挂载到选定目录下，然后安装，然后编译题目（gcc.cfg），需要g++和gfortran

## 编译题目需要用到的编译器

gcc
g++
gfortran

需要安装，并重新编译题目

### 安装g++和gfortran时发现仓库有问题，依赖包版本冲突，使用手动安装

手动下载deb包

mkdir temp && cd temp

sudo apt-get download [package] 下载安装包

dpkg -i --force-all            [package]    安装

/var/lib/dpkg/status 所有安装包的依赖包信息都在这里

备份cp status.bak，修改status，修改对应依赖包的版本号，以欺骗系统。

## 编译浮点类型题目时

> 报错：
> /usr/lib/ld:cannot find -lgfortran 
> collect2:error:ld returned 1 exit status

解决方法：

ld -lgfortran --verbose 查看内部链接脚本，及其链接过程

发现没找到静态库文件libgfortran.a，会去/usr/local/lib中找

先安装libgfortran-dev-8和libgfortran5，再在/usr/local/lib 中做个软链接链接/usr/lib中的libgfortran.a

> 问题：
> Notice:Unusable path detected in build directory list file.
> 
> 描述：找不到build directory，开始自动编译题目.

原因：tar包的时候加了--touch，导致它以为是新的。不能加--touch，时间戳不对就改时间戳。

# Spec配置，开始测试

# Stream测试

一个线程，export OMP_NUM_THREADS=1

.f是什么文件？fortran语言
