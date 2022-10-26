# GDB调试对比

sw翻译sw

qemu6调试-d in_asm,exec -singlestep 单步执行

断点打在accel/tcg/cpu-exec.c:192 ret=tcg_qemu_tb_exec这个地方

到这里前端后端已经翻译完了，里面就是执行后端翻译完的tb，是一块地址，代码级调试s进去了不会显示代码，只能用汇编级调试si进去，进去后会取tb执行然后退出，这期间很多变量打印不了。code_gen_buffer里可以通过$r9加上偏移量来查看各个寄存器的值。或者在外面通过打印env->ir[]查看虚拟内存单元的值，这里不能用info r rx查看了，因为此时是执行模式，gcc要使用寄存器来执行代码，而翻译模式的寄存器用虚拟内存单元来模拟。单步执行中一条汇编对应一个tb，有些tb对应的汇编没有显示？

hello调试

starti进入第一条汇编，调试的就是动态链接器链接的动态库，si执行下一条汇编，一个si对应上面一次c。可以打断点函数，然后si一步步往下，info r rx，display $rx查看寄存器的值。

qemu翻译的地址比正常执行的地址正常情况下大802000

# 插打印

qemu查错会涉及一些底层的动态库文件，以glibc包为例：
1、编译glibc

```shell
mkdir build
cd build
../glibc-2.31/configure --prefix=/home/lin/install --disable-werror --enable-debug=yes
make -j
```

编译出的libc.so.6文件在build/目录下，ld.so在build/elf/目录下。

编译脚本config.sh

```shell
CPU=sw6b
PREFIX=
# PREFIX_NATIVE=/home/luo/swgcc830_native_tools # 指定binutil
SRCPATH=$(shell pwd)                            # 当前路径
TARGET=sw_64-linux-gnu
# default:none

ac_cv_sizeof_int=4
ac_cv_sizeof_long=8
ac_cv_type_ssize_t=yes
ac_cv_type_size_t=yes
ac_cv_type_intptr_t=yes
ac_cv_type_uintptr_t=yes
ac_cv_type_uintmax_t=yes
ac_cv_type_pid_t=yes
ac_cv_type_off_t=yes
ac_cv_type_intmax_t=yes
ac_cv_type_uchar=no
ac_cv_type_caddr_t=yes
ac_cv_type_int_fast32_t=yes

libgcc_cv_fixed_point=no
ac_cv_type_u_int64_t=yes
ac_cv_type_uint64_t=yes
ac_cv_type_uint32_t=yes
ac_cv_type_u_int32_t=yes
ac_cv_func_epoll_create1=no

glibc:
    echo 'make glibc'
    # rm -rf glibc_build_native
    # mkdir -v glibc_build_native
    echo "libc_cv_forced_unwind=yes" > config.cache
    echo "libc_cv_c_cleanup=yes" >> config.cache
    echo "libc_cv_alpha_tls=yes" >> config.cache
    echo "libc_cv_gnu89_inline=yes" >> config.cache
    echo  "install_root=${PREFIX_NATIVE}" >  configparms
    CC="${TARGET}-gcc -g" AR="${TARGET}-ar" RANLIB="${TARGET}-ranlib" ../configure --prefix=/usr --host=${TARGET} --build=${BUILD} --disable-profile --enable-add-ons --with-tls --with-__thread --enable-obsolete-rpc --cache-file=config.cache --with-binutils=/usr/bin --with-headers=/usr/include --disable-werror --with-cpu=${CPU}
    # --with-binutils=${PREFIX_NATIVE}/usr/bin --with-headers=/usr/include --disable-werror --with-cpu=${CPU}
    make -j40
    # cd glibc_build_native; make install
```

2、插打印
libc.so常用断点：

| 函数                     | 所在文件             |
| ---------------------- | ---------------- |
| __libc_start_main      | csu/libc-start.c |
| _libc_init_firs        | csu/init-first.c |
| _dl_non_dynamic_init   | elf/dl-support.c |
| _dl_init_paths         | elf/dl-load.c    |
| _dl_important_hwcaps.c | elf/dl-hwcaps.c  |
| _dl_start_final        | elf/rtld.h       |

ld.so常用断点：

| 函数             | 所在文件                        |
| -------------- | --------------------------- |
| _dl_start      | elf/rtld.c                  |
| _dl_start_user | sysdeps/x86_64/dl-machine.h |
| _dl_init       | elf/dl-init.c               |

3、用于打印的汇编码

嵌入式汇编、内联汇编——在C语言中插入汇编语言

内嵌汇编——在高级语言中插入汇编语言

glibc

```asm6502
asm("汇编语句"
    :输入部分
    :输出部分
    :会被修改的部分);
```

```asm6502
arm：
    const char *str = "dl_init_paths:005\n";
    asm("mov x8, 64\n\t"
        "mov x0, 0\n\t"
        "mov x1, %0\n\t"
        "mov x2, 18\n\t"
        "svc #0\n\t"
        :
        :"r"(str)
        : "x8","x0","x1","x2");

x86_64:
    const char *str = "socket/bits/recv01\n";
    asm("movq $1, %%rax\n\t"
        "movq $1, %%rdi\n\t"
        "movq %0, %%rsi\n\t"
        "movq $24, %%rdx\n\t"
        "syscall\n\t"
        :
        :"r"(str)
        :"rax","rdi","rsi","rdx");


sw_64:
```

ld

ld.so中可以用glibc自带的_dl_debug_printf函数

```c
_dl_debug_printf("_dl_debug_printf\n");
_dl_debug_printf("%x\n",fd); //%d 好像识别不了
```

插入之后重新编译glibc，并重新编译hello程序，注意编译时要指定动态库和动态链接器。

这是插打印动态链接的程序需要执行的命令

```shell
gcc -g test/hello.c -o hello-sw-dynamic -Wl,-rpath=/home/fei/glibc/glibc_build_native/ -Wl,-dynamic-linker=/home/fei/glibc/glibc_build_native/elf/ld.so
```

```shell
gcc -g  -O0 test456.c -o test456 -L /home/fei/glibc/glibc_build_native/ -lm -static
```

4、查错命令

```shell
display /10i $pc
info r r9
set var $r9=0x1
info symbol FUN    //查函数与哪个可执行文件有关
```

插打印，正常能打印出来，qemu翻译没打印出来的地方定位错误，用gdb调试，进入函数找到定义位置，继续插打印······

对于qmeu来说，glilbc和ld就是机器码，因此用gdb调试是没办法打上断点或者进入函数的

可以调试a.out进入glibc或者ld，同时在错误位置前后插打印，以此调试qemu定位错误的位置。

# 日志

qemu-sw64 -d in_asm,op,out_asm -cpu core3 hello > log 2>&1

in_asm,op,out_asm,exec

单步调试 -singlestep，每一个tb中就只有一条汇编代码，同时有很多tb没显示有汇编代码，可能是没有翻译，直接执行的？c 多少，在gdb ./hello-sw-dynamic中对应si 多少

exec打印出每个tb的首地址，方便对应。

# 查错小结：

报错信息还比较重要，通过报错信息能知道是什么地方出问题了，read library？create share object
descriptor？通过这个去qemu的日志信息中找（或者单步调试从后往前），从后往前，由捕获异常的地方向上找就能找到出错函数是哪一个，然后去glibc中搜索定位（或者了解glibc源码的话能快速定位到？或者看starce能看到是哪个系统调用出问题？）。然后可以通过插打印缩小范围。差不多到某一个具体的函数了，就用qemu开单步调试，找到对应的位置（或者也能由第一步直接定位？，但范围太大），gdb打上函数断点，一步步到对应位置，开始比对两者汇编码的地址、汇编。，最终出错原因可能是前端翻译错误、qemu相应分支没写。
