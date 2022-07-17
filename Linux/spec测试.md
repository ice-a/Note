file bin/specperl 可以看出spec是在哪里编译的
file benchspec/CPU2006/400.perlbench/build/build_base_sw6.0000 可以看出题目是用的什么编译器
gcc.cfg存储题目编译器设置

这次测试是在x86上用sw/alpha编译器编译题目，然后在sw上测，运行spec时ps -ef|grep spec可以看出会调用qemu-x86_64运行，然后在调用相应qemu跑题

# spec测试

## 编译binutil ld工具链,gcc8.3编译qemu会有问题，因此要重新编译

配置参数

```shell
./configure --target=sw_64-linux-gnu --prefix=${PREFIX}/usr --with-sysroot=${PREFIX} --disable-werror --with-cpu=${CPU}
```

(./configure --target=sw_64-linux-gnu --prefix=/home/deepin/util --with-sysroot=/home/deepin/util --disable-werror --with-cpu=sw6a)
PREFIX表示安装路径，自己选择
make &&make  install生成可执行文件
(环境参数source env.sh  注意. ./=source)
env.sh

```shell
unset PATH
unset LD_LIBRARY_PATH
unset LIBRARY_PATH
export PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games:/sbin:/usr/sbin
export LD_LIBRARY_PATH=/lib/sw_64-linux-gnu:/usr/lib/sw64-linux-gnu:$LD_LIBRARY_PATH
export PATH=/home/deepin/util/usr/bin:$PATH
```

spec 出现
home/deepin/cpu2006/bin/runspec: line 15: $::runspec_time: command not found
/home/deepin/cpu2006/bin/runspec: line 17: syntax error near unexpected token `{'
/home/deepin/cpu2006/bin/runspec: line 17: `if (exists $ENV{'SPECDB_PWD'}) {'

说明binfmt没有配置。

## 配置binfmt

按照READEME配置，模块已加载，binfmt-support已经安装，qemu-86_64需要从别的地方拷过来，并且修改路径，update-binfmts --display |grep x86_64查看是否已经配置好。
qemu-binfmt库文件，需要拷贝到qemu配置参数--interp-prefix=/etc/qemu-binfmt/%M的路径下。

spec原始代码安装，编译题目
代码在11.127/mnt/testpackage下，mount挂载到选定目录下，然后安装，然后编译题目（gcc.cfg），需要g++和gfortran

## 编译题目需要的编译器

gcc
g++
gfortran

需要安装

并重新编译

### 安装g++和gfortran时发现仓库有问题，依赖包版本冲突

手动下载deb包

mkdir temp && cd temp

sudo apt-get download [package] 下载安装包

dpkg -i --force-all [package]            安装

/var/lib/status 所有安装包的依赖包信息都在这里

备份cp status.bak，修改status，修改对应依赖包的版本号，以欺骗系统。

## 编译浮点类型题目时

> 报错：
> 
> /usr/lib/ld:cannot find -lgfortran 
> collect2:error:ld returned 1 exit status

解决方法：

ld -lgfortran --verbose 查看内部链接脚本，及其链接过程

发现没找到静态库文件libgfortran.a，会去/usr/local/lib中找

先安装libgfortran-dev-8和libgfortran5，再在/usr/local/lib 中做个软链接链接/usr/lib中的libgfortran.a

> 问题：
> 
> 找不到build directory

环境变量有问题，. ./shrc没有设置好环境变量，多搞几遍，考验人品

- 1
