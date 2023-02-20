```shell
# 静态编译 
gcc  源文件 -o 生成 --static
# 动态编译 
gcc  源文件 -o 生成

gcc -g # 生成调试信息

gcc -s # 删除符号表和重定向信息

gcc -l # 指定要链接的库
```

# Strace

```shell
strace ./qemu-sw64 -cpu core3 /home/fei/hello-sw-dynamic >log 2>&1 可以查看错误日志
找到access那一栏，可以看到去哪个目录下找动态库，哪个没找到
目前6.2没做动态这一块
```

# ld

```shell
ldd 可执行文件
# 可以它的看到依赖的库文件，从其他地方拷过去。
# 有两个，一个是glibc库文件，一个是ld库文件，ld链接器会链接这两个库文件的。

ldd 库
# 依赖的库
```

[ldd not a dynamic executable 问题_Chris_zhangrx的博客-CSDN博客](https://blog.csdn.net/Chris_zhangrx/article/details/114396042)

```shell
# 查看链接文件所在路径
ldconfig -p|grep fortran
```

```shell
# -lgfortran libgfortran这个库 --verbose 查看详细信息
ld -lgfortran --verbose  # 查看链接器内部链接脚本及其链接过程
ld --verbose|grep SEARCH # 查看ld搜索路径
```

```shell
反汇编：把机器码翻译成成汇编语言
objdump -D 可执行文件 > 文件
```

[linux缺少libz.so,Ubuntu15.04如何查找libz.so.1属于哪个包并安装](https://blog.csdn.net/weixin_42139302/article/details/116931847)

# Binutils

GNU二进制工具集

# strip

strip是可以对已经编译生成的目标文件进行删减的工具，它有各种命令选项，可以用来删除对应的信息，比如 -g 仅删除 gcc -g 添加的调试信息。

库和可执行文件内含符号表symbol tables，可以使用strip命令strip掉，体积会变小，同时无法调试。

库strip后会无法链接，不推荐这么做。可执行文件（a.out和elf）可以strip，执行速度会提升？就是无法调试了。同时make install会有strip步骤，是否与这里相同？

如何看库和可执行文件是否strip，`file 库/可执行文件`，看是否有stripped这一状态。