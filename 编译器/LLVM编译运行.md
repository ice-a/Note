# 编译运行方法

```context
sudo apt install cmake gcc g++ ninja-build git
10. qemu6+llvm9
    10.1 first build llvm9
         on sw: git clone luo@172.16.11.127:/mnt/swllvm9.git
         on other platform: luo@172.16.11.127:/mnt/llvm9.git 
         build llvm-debug: cmake -G Ninja -DLLVM_ENABLE_PROJECTS="clang;compiler-rt" -DCMAKE_INSTALL_PREFIX=/usr/bin/llvm9-Debug -DCMAKE_BUILD_TYPE=Debug ../llvm
         build llvm-release: cmake -G Ninja -DLLVM_ENABLE_PROJECTS="clang;compiler-rt" -DCMAKE_INSTALL_PREFIX=/usr/bin/llvm9-Release -DCMAKE_BUILD_TYPE=Release ../llvm
         ninja -j40 install
    10.2 set llvm environment
         10.2.1 modify env.sh 
         10.2.2 add "source env.sh" to ~/.bashrc
    10.3 build qemu+llvm9
         10.3.1 if you want to build hqemu-2.5.2+llvm9, git clone luo@172.16.11.127:/mnt/hqemu-2.5.2.git
         10.3.2 config qemu6+llvm9: make sure build on "~/qemu6-project/build" directory
         export CFLAGS="-gdwarf-2 -g3 -O0" && ../qemu6/configure --target-list=x86_64-linux-user --prefix=/usr --sysconfdir=/etc --disable-docs --disable-werror --disable-blobs --enable-debug --enable-debug-stack-usage --disable-debug-tcg --enable-debug-info --enable-qom-cast-debug --interp-prefix=/etc/qemu-binfmt/%M --enable-llvm
         10.3.3 config hqemu-2.5.2+llvm9: make sure build on "~/qemu6-project/build2" directory
         export CFLAGS="-gdwarf-2 -g3 -O0" && ../hqemu-2.5.2/configure --target-list=x86_64-linux-user --prefix=/usr --sysconfdir=/etc --disable-docs --disable-werror --disable-blobs --enable-debug --disable-debug-tcg --enable-debug-info --enable-qom-cast-debug --interp-prefix=/etc/qemu-binfmt/%M --enable-llvm
        make -j40
    10.4 get your qemu-x86_64
        qemu6+llvm9: ~/qemu6-project/build/x86_64-linux-user/qemu-x86_64
        hqemu-2.5.2+llvm9: ~/qemu6-project/build2/x86_64-linux-user/qemu-x86_64
```

> env.sh

```shell
unset C_INCLUDE_PATH
unset CPLUS_INCLUDE_PATH
unset LD_LIBRARY_PATH
unset LIBRARY_PATH
unset LLVM_CMD
unset LLVM_MODE

#C_INCLUDE_PATH=/usr/lib/sw_64-linux-gnu/glib-2.0/include   #only need on sw
#CPLUS_INCLUDE_PATH=/usr/lib/sw_64-linux-gnu/glib-2.0/include #only need on sw


export LLVM_PATH=/home/luo/qemu6-project/llvm9-Debug # llvm路径

export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games
export PATH=$LLVM_PATH/bin:$PATH # 把llvm/bin添加到环境变量PATH
export C_INCLUDE_PATH=$LLVM_PATH/include/:$LLVM_PATH/include/llvm:$LLVM_PATH/include/llvm-c:$C_INCLUDE_PATH
export CPLUS_INCLUDE_PATH=$LLVM_PATH/include/:$LLVM_PATH/include/llvm:$LLVM_PATH/include/llvm-c:$CPLUS_INCLUDE_PATH
export LD_LIBRARY_PATH=$LLVM_PATH/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$LLVM_PATH/lib:$LIBRARY_PATH # 把llvm/lib添加到环境变量LIBRARY_PATH

#export LLVM_MODE="hybrids" #single llvm thread   # 单线程 OMP_NUM_THREAD环境变量貌似也是线程数量相关
#export LLVM_CMD='-debuglv' #print GenTrace information
```

# Issues

## 1、磁盘空间满了

> 编译llvm时（ninja -j40）
> 
> 报错：
> 
> no space left on device 

错误原因：硬盘满了

> 安装llvm时（ninja install）
> 
> 报错：
> 
> CMake Error at tools/clang/lib/Frontend/cmake_install.cmake:41 (file):
>   file INSTALL cannot copy file
>   "/home/luo/qemu6-project/llvm9/build-Debug/lib/libclangFrontend.a" to
>   "/home/luo/qemu6-project/llvm9-Debug/lib/libclangFrontend.a".
> Call Stack (most recent call first):
>   tools/clang/lib/cmake_install.cmake:57 (include)
>   tools/clang/cmake_install.cmake:59 (include)
>   tools/cmake_install.cmake:48 (include)
>   cmake_install.cmake:68 (include)
> FAILED: CMakeFiles/install.util
> 
> cd /home/luo/qemu5-project/llvm9/build-Debug && /usr/bin/cmake -P cmake_install.cmake
> 
> ninja: build stopped: subcommand failed.

错误原因：首先猜测是权限原因，加sudo，试过没用。尝试手动cp，报错`cp:error writing '':No space left on space`，发现是硬盘满了。

解决方案：去/目录下执行`du -h --max-depth=1`可以查看当前目录下各个文件夹的大小，依次排查，知道找出占用空间很大的文件。此次是发现/home目录下各用户主目录占用空间大，可以直接询问本人是否仍在使用。

> 出错：
> 
> 用qemu6+ llvm9-release执行spec出错。
> 
> qemu代码中assertion failed每次运行都是这一类型错误，但是位置不一样。

解决方案：要有解决问题的意识，不能别人说一步动一步，自己思考解决方法的同时，都是同事，都可以向她们请教，说明错误情况和自己想采取的解决方法。高级别优化的出错就换低级别优化的，结果换了debug版本仍然不行。再试试hello能不能跑的通，动态静态的都试一试。qemu6不行，试试qemu6-system。提示是不是缺包可以下载一下？spec是怎么跑不过，是跑到哪一道题有错还是小工具不行？可以试试直接用qemu执行跑这些工具试试？都不行。

```shell
cmake -G Ninja -DLLVM_ENABLE_PROJECTS="clang;compiler-rt" -DCMAKE_INSTALL_PREFIX=/usr/bin/llvm9-Debug -DCMAKE_BUILD_TYPE=Debug ../llvm
```

https://blog.csdn.net/zhongyunde/article/details/109013865

1、utils/gdb-scripts/prettyprinters.py gdb数据结构llvm编写了自己的一些实用的类stl库，没有配置过的gdb没法直观的显示这些数据结构的内容
   在llvm仓下 ./utils/gdb-scripts/prettyprinters.py， 在~/.gdbinit中添加source ${path}/prettyprinters.py,就可以较为直观的显示了

2、在utils/vim/syntax/目录下有2个vim语法高亮文件llvm.vim tablegen.vim, 分别对应这llvm的IR(*.ll)以及tablegen(*.td)文件语法高亮, 将2个文件移入~/.vim/syntax/中并在~/.vimrc中添加utils/vim/vimrc

3、utils/TableGen/tdtags
