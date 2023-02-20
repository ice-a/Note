# gprof

--enable-gprof  
[超级方便的Linux自带性能分析工具！gprof介绍、安装、使用及实践](https://zhuanlan.zhihu.com/p/385842627)  

![](https://lists.nongnu.org/favicon.ico)[[Qemu-devel] profiling qemu](https://lists.nongnu.org/archive/html/qemu-devel/2012-02/msg01730.html)

gprof是一个linux应用程序性能分析工具。gprof可以给出函数的调用时间、调用次数和调用关系。

https://zhuanlan.zhihu.com/p/385842627
https://lists.nongnu.org/archive/html/qemu-devel/2012-02/msg01730.html

gprof可以给出函数调用时间、调用次数和调用关系。

## 1、配置

```shell
../configure --enable-gprof # O0不对，用O2?
make
```

## 2、使用

```shell
# 生成gmon.out，
./qemu-x86_64 ./nbench
# 读取该报告
gprof ./qemu-x86_64 gmon.out
```

## 3、输出分析

（1）第一张表是各个函数的执行和性能报告
The Flat Profile：

| 项目               | 含义                                                                     |
| ------------------ | ------------------------------------------------------------------------ |
| % time             | 此函数执行时间占程序总运行时间的百分比                                 |
| cumulative seconds | 此函数和上面列出的函数累计执行的时间                                 |
| self seconds       | 此函数本身执行的时间。                                                   |
| calls              | 此函数被调用的次数，如果此函数被概化，则为空，如helper函数。             |
| self ms/call       | 每次调用该函数所花费的平均毫秒数，如果该函数被概化，则为空。             |
| total ms/call      | 每次调用此函数及其子函数上花费的平均毫秒数，如果此函数被概化，则为空。 |
| name               | 函数名                                                                   |
 
（2）第二张表是程序运行时的函数调用关系，按顺序排序花费在每个函数及其子函数上的总时间。
The Call Graph：

| 项目     | 含义                                                                                                     |
| -------- | -------------------------------------------------------------------------------------------------------- |
| % time   | 花费在此函数及其子函数上总时间的百分比。                                                                 |
| self     | 花费在此函数上的总时间                                                                                   |
| children | 它的子函数传播到这个函数的总时间。                                                                       |
| called   | 这是函数被调用的次数。如果函数递归地调用自身，则数字只包括非递归调用，后面跟着一个' +'和递归调用的数量。 |
| name     | 函数名                                                                                                   |

（3）第三张表是函数与其在报告中序号的对应表 
The Annotated Source Listing

4、实例分析

以nbench为例，分析Qemu的性能瓶颈。

5、缺点

（1）函数执行时间是估计值。

（2）不能处理内联函数。

（3）不适合存在大量递归调用的程序。

# profiler

--enable-profiler

[QEMU 自带的简易计时器 — profiler 的简介及代码分析 - JciX ~](http://blog.jcix.top/2018-07-20/qemu_profiler/)

# plugins

--enable-plugins

[QEMU TCG Plugins使用简介](https://zhuanlan.zhihu.com/p/535618102)

[QEMU TCG Plugins详解_JaCenz的博客-CSDN博客](https://blog.csdn.net/JaCenz/article/details/125302647)

[QEMU TCG Plugins — QEMU documentation](https://qemu-project.gitlab.io/qemu/devel/tcg-plugins.html)

## 1、配置

`../configure --enable-plugins`

## 2、编译

`make && make plugins`

## 3、使用

`./qemu-x86_64 -plugin contrib/plugin/libhotblocks.so -d plugin ./hello`

![](Pasted%20image%2020221222112412.png)