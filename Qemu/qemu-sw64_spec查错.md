# 456.hmmer

错误类型：返回值非0

参数写错了，文件打不开，现在执行到tb=2911卡住

ctrl+c停在了/fpu/softfloat.c 的第697行，switch这上面，继续c还能看到代码在往下跑。

include/qemu/bitops.h第358行，这个是符号扩展。等不了了，

最近的tb=2906有in_asm显示，

2895、会停一会儿，2900、2903、2910停一大会儿

运行结果：

> FATAL: tcg_qemu_tb_exec=2964
> 
> tcg_qemu_tb_exec=2984
> 
> fit failed; --num may be set too small
> 
> tcg_qemu_tb_exec=2985
> 
> tcg_qemu_tb_exec=3146
> 
> Inferior 1 (process 46205)exited with code 01

先从2906开始看起

/lib /usr/lib /usr/sw/sw....../usr/

libthread_db.so->libthread_db.so.1->libthread_db-1.0.so

插打印之后错误又变了，变为结果错误类型

fy上成功运行

fei上还是cpoy return 0

两者区别fy上编译题目用的O0，且qemu用的O0

现在不插打印是copy return 0，插打印后错误地方可以执行下去，不报这个错误了，可以出结果，但是结果上mu出错。

VCOND VCONS

# 471.omntpp

是可以执行完毕的，就是结果出错了

tb=26384

tb=27969 运行结束，正常退出

tb=24820开始输出event

报错信息是从event #5000000 T=0.0838662807(83mm)开始的此时tb=26357，后面都有错误。

第一个问题题目编译不出来？

第二个gdb对照有用吗，没法对照上，可以对照上吗。值差一点是有错还是对应的上？qemu就有一两次正常执行有很多个。
