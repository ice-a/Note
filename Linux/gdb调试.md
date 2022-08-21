```shell
gdb
gdb [program]
gdb --args program arg1
gdb -x gdbinit
(gdb) 
```

命令行输入gdb，进入调试

gdb会先读取~/.gdbinit，再执行命令行选项，再读取./.gdbinit

参数

n ni

s si

directory 源代码目录

ctags -R *

p/x 十六进制

p/o 八进制

p/t 二进制

b 行号 设置断点

info b 查看断点

info(i) r 查看整数寄存器

info(i) r a 查看所有寄存器

del 数字 删除断点

display /x 追踪值

display /i  $pc 打印下一条汇编值

focus cmd 聚焦到命令行

fs src fs next

set 

ctrl +l 刷新

bt        函数调用栈

左侧(gdb)不显示，然后输入一个命令直接执行，不需要按enter，怎么实现?



```shell
set breakpoint pending on # 挂起断点，未来生效
set confirm off           # 退出时不显示提示信息
set print pretty on       # 打印缩进对齐的结构体
set pagination off        # 不要出现 Type <return> to continue 的提示信息
set print thread-events off# 关闭线程产生和退出信息
set history filename ~/.gdb_history
set history save on       # 保存历史命令
set logging on            # 开启日志
set disassemblenext line auto # 显示下一行反汇编
set print arrayindexes on # 打印数组索引下标
```
