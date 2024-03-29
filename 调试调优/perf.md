perf 是Linux的一款性能分析工具，能够进行函数级和指令级的热点查找，可以用来分析程序中热点函数的CPU占用率，从而定位性能瓶颈。

`sudo perf stat -d ./a.out`  统计指令数量
stat 执行某个命令，收集特定进程的性能概况

-e 指定event事件

-d detailed

## perf的使用

`perf --help`之后可以看到perf的二级命令。

| 命令          | 作用                                                                                                                  |
| ------------- | --------------------------------------------------------------------------------------------------------------------- |
| `annotate`      | 解析perf record生成的perf.data文件，显示被注释的代码。                                                                |
| `archive`       | 根据数据文件记录的build-id，将所有被采样到的elf文件打包。利用此压缩包，可以再任何机器上分析数据文件中记录的采样数据。 |
| `bench`         | perf中内置的benchmark，目前包括两套针对调度器和内存管理子系统的benchmark。                                            |
| `buildid-cache` | 管理perf的buildid缓存，每个elf文件都有一个独一无二的buildid。buildid被perf用来关联性能数据与elf文件。                 |
| `buildid-list`  | 列出数据文件中记录的所有buildid。                                                                                     |
| `diff`          | 对比两个数据文件的差异。能够给出每个符号（函数）在热点分析上的具体差异。                                              |
| `evlist`        | 列出数据文件perf.data中所有性能事件。                                                                                 |
| `inject`        | 该工具读取perf record工具记录的事件流，并将其定向到标准输出。在被分析代码中的任何一点，都可以向事件流中注入其它事件。 |
| `kmem`          | 针对内核内存（slab）子系统进行追踪测量的工具                                                                          |
| `kvm`           | 用来追踪测试运行在KVM虚拟机上的Guest OS。                                                                             |
| `list`          | 列出当前系统支持的所有性能事件。包括硬件性能事件、软件性能事件以及检查点。                                            |
| `lock`          | 分析内核中的锁信息，包括锁的争用情况，等待延迟等。                                                                    |
| `mem`           | 内存存取情况                                                                                                          |
| `record`        | 收集采样信息，并将其记录在数据文件中。随后可通过其它工具对数据文件进行分析。                                          |
| `report`        | 读取perf record创建的数据文件，并给出热点分析结果。                                                                   |
| `sched`         | 针对调度器子系统的分析工具。                                                                                          |
| `script`        | 执行perl或python写的功能扩展脚本、生成脚本框架、读取数据文件中的数据信息等。                                          |
| `stat`          | 执行某个命令，收集特定进程的性能概况，包括CPI、Cache丢失率等。                                                        |
| `test`          | perf对当前软硬件平台进行健全性测试，可用此工具测试当前的软硬件平台是否能支持perf的所有功能。                          |
| `timechart`     | 针对测试期间系统行为进行可视化的工具                                                                                  |
| `top`           | 类似于linux的top命令，对系统性能进行实时分析。                                                                        |
| `trace`         | 关于syscall的工具。                                                                                                   |
| `probe`         | 用于定义动态检查点。                                                                                                  |

## perf list

`perf list`不能完全显示所有支持的事件类型，需要`sudo perf list`。

# 统计qemu执行的指令数量：

思路一：一小题一小题统计，开日志-d out_asm，打印到log.txt中，手动筛选，统计行数。

筛选脚本

```shell
$file=
sed -i '/^$/d' $file        # 空白行
sed -i '/--/d' $file        # 包含--的行
sed -i '/^[a-zA-Z]/d' $file # 以字母开头的行
# 思考 缩减为一句，去除非数字开头的行？？？去除非0开头的行？？？
```

缺点：统计的是code_buffer中的指令条数，但实际执行中并不是顺序全部执行的，一有分支跳转，二有TB建链后热点TB的反复执行。

思路二：代码中添加功能+1（执行模式时每执行一条指令+1？不太现实）

思路三：perf统计，但也只能一小题一小题统计，可以每一大题写一个脚本一次性执行多条命令

> 脚本编写：

```shell
sudo perf stat -e instructions ..(specinvoke -n得到的小题linux命令).. 2>>log.txt &&
...
sudo perf stat -e instructions ..(specinvoke -n得到的小题linux命令).. 2>>log.txt 
```

为什么要单事件，因为硬件性能计数器有限，多事件共享硬件，每个事件占用硬件一定时间，最后的结果使用占用时间/总时间估算出来的，因此不准确，单事件占用100%则不会有这一问题。
