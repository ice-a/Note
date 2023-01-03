# 启动方式

```shell
gdb
gdb [program]
gdb --args [program] [arg1]
gdb -x gdbinit
```

# 常用命令

| 命令                                                                                               | 作用                         |
| ------------------------------------------------------------------------------------------------ | -------------------------- |
| **运行**                                                                                           |                            |
| `(gdb) file [program]`                                                                           | 设置调试程序                     |
| `(gdb) run [program]`                                                                            | 启动程序                       |
| `(gdb) set args []`                                                                              | 设置参数                       |
| `(gdb) show args`                                                                                | 显示参数                       |
| `start`                                                                                          | 相当于在main打断点并运行             |
| `starti`                                                                                         | 第一条汇编指令                    |
| `list(l)`                                                                                        | 显示源代码                      |
| `next(n)`                                                                                        | 单步跳过-源码级                   |
| `step(s)`                                                                                        | 单步进入-源码级                   |
| `nexti(ni)`                                                                                      | 单步跳过-指令级                   |
| `stepi(si)`                                                                                      | 单步进入-指令级                   |
| `finish`                                                                                         | 跳出当前函数                     |
| `until(u)`                                                                                       | 运行到函数某一行                   |
| `continue(c)`                                                                                    | 执行到下一个断点或程序结束              |
| **断点**                                                                                           |                            |
| `break(b) 行号`<br/>             `函数名`<br/>             `文件:行号`<br/>             `*地址`<br/>`if 条件` | 设置断点（函数名或者行号）<br/>设置断点触发条件 |
| `info breakpoint(b)`                                                                             | 查看断点                       |
| `delete(d，del) 断点编号`                                                                             | 删除断点                       |
| `disable/enable(dis,ena) 1`                                                                      | 禁用/启用断点                    |
| `ignore [断点编号]  [次数]`                                                                            | 设置断点忽略次数                   |
| `condition [断点编号] [变量==值 ]`                                                                      | 修改断点触发条件                   |
| **查看变量**                                                                                         |                            |
| `print(p)/x` <br/>             `/o`<br/>             `/t`                                        | 十六进制<br/>八进制<br/>二进制       |
| `frame(f)`                                                                                       | 查看函数信息                     |
| `info args`                                                                                      | 查看函数参数                     |
| `info locals`                                                                                    | 查看函数变量                     |
| `info(i) r`<br/>`info(i) r a`                                                                    | 查看整数寄存器<br/>查看所有寄存器        |
| `info source`                                                                                    | 查看当前源码信息，所在文件完整路径          |
|                                                                                                  |                            |
| `display(disp) /x,o,t`                                                                           | 监控值                        |
| `display /i $pc`                                                                                 | 打印下多少条汇编代码，左边箭头是下一条汇编      |
| `examine(x) /<n/f/u> <addr>`                                                                     | 查看内存地址中的值                       |
| `ctrl+alt+X`                                                                                     | 打开TUI界面                    |
| `focus(fs) cmd` <br/>`fs src fs next`                                                            | 聚焦到命令行                     |
| `set 变量名=值`                                                                                      | 赋值                         |
| `ctrl + L`                                                                                       | 刷新                         |
| `backtrace(bt)`                                                                                  | 显示函数调用栈                    |
| `show environment`                                                                               | 查看环境变量                     |
| `quit(q)`                                                                                        | 退出                         |
| `watchpoint`                                                                                     | 观察点                        |
| `checkpoint`                                                                                     | 检查点                        |

- 十进制（Decimal）
  由数字0-9组成，不能以0开头。

- 二进制（Binary）
  由数字0-1组成，为了区分于其他进制，都是以0b开头。

- 十六进制（Hexadecimal）
  
  由数字0-9和字母a-f(或者A-F)组成，为了区分于其他进制，都是以0x开头。

- 八进制（Octal）
  
  由数字0-7组成，为了区分于其他进制，都是以0开头。

> 左侧(gdb)不显示，然后输入一个命令直接执行，不需要按enter，怎么实现?

## examine命令

```shell
examine(x) /<n/f/u>  <addr>

n:是正整数，表示需要显示的内存单元的个数，即从当前地址向后显示n个内存单元的内容，
一个内存单元的大小由第三个参数u定义。

f:表示addr指向的内存内容的输出格式，s对应输出字符串，此处需特别注意输出整型数据的格式：
  x 按十六进制格式显示变量.只显示八位
  d 按十进制格式显示变量。
  u 按十进制格式显示无符号整型。
  o 按八进制格式显示变量。
  t 按二进制格式显示变量。
  a 按十六进制格式显示变量。可以显示全
  c 按字符格式显示变量。
  f 按浮点数格式显示变量。

u:就是指以多少个字节作为一个内存单元-unit,默认为4。u还可以用被一些字符表示:
  如b=1 byte, h=2 bytes,w=4 bytes,g=8 bytes.

<addr>:表示内存地址
```

# 初始化文件(.gdbinit)

gdb will read it when starting

## 作用：

执行命令脚本，将用的多的写进入，不用每次都手打一遍

## 运行顺序

1. $HOME/.gdbinit

2. 运行命令行选项

3. ./.gdbinit

## 常用配置

在GDB命令行中也可以使用，也可以使用tab键

```shell
# 常用配置
set breakpoint pending on      # 挂起断点，未来生效
set confirm off                # 退出时不显示提示信息
set print pretty on            # 打印缩进对齐的结构体
set pagination off             # 不要出现 Type <return> to continue 的提示信息
set print thread-events off    # 关闭线程产生和退出信息
set disassemblenext line auto  # 显示下一行反汇编
set print array-indexes on      # 打印数组索引下标

# 保存历史命令
set history filename ~/.gdb_history
set history save on       # 保存历史命令

# 保存日志
set logging on            # 开启日志

# 调试相关配置
directory 源代码目录             # 源代码路径
file [调试程序]                  # 调试程序
set args [设置调试程序执行的参数] # 设置参数
b    [函数名/文件:行号]           # 设置断点
```

数组名是地址 ，p数组名只能打印出地址，打印出所有数组元素？p*（数组名+数字）能打印出对应数组元素的内容。

# Issue

## 1 Unable to find libthread_db

> 报错：
> 
> warning: Unable to find libthread_db matching inferior's thread library

解决方案：[关于gdb “Unable to find libthread_db matching inferior's thread library”的解决办法-CSDN社区](https://bbs.csdn.net/topics/391832947)

## 2 使用gdb 9.1

> 6A机器原版gdb为gdb 7.12，想使用gdb-v9.1
> 执行gdb-v9.1/bin/gdb 报错
> ./gdb: error while loading shared libraries: libpython3.7m.so.1.0: cannot open shared object file: No such file or directory
> 
> ./gdb: error while loading shared libraries: libbabeltrace.so.1: cannot open shared object file: No such file or directory
> 
> ./gdb: error while loading shared libraries: libsource-highlight.so.4: cannot open shared object file: No such file or directory

问题原因：在6A机器上缺包

解决方案：下载包

```shell
libpython3.7/stable 3.7.3-2+0eagle7 sw_64
 Shared Python runtime library (version 3.7)
libbabeltrace1/stable 1.5.6-2+deb10u1 sw_64
 Babeltrace conversion libraries
libsource-highlight4v5/stable 3.1.8-1.2 sw_64
 source highlighting library
```

## 3 断点位置错乱

问题原因：修改了源代码没有编译，gdb调试用的仍旧是旧的二进制，显示的是新的源代码，行数不同导致断点位置错乱。

解决方案：不修改源代码，或者重新编译使用新的二进制。
