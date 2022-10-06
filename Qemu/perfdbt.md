- gen_asm.py:生成test程序
  生成（提取）汇编指令。调用gc.load_asm('600.perlbench_s')函数,blk_count,file_count

- gen_mem_only.py:
  相比，少了许多for statistics，且gc.load_asm('600.perlbench_s')注释起来了

- batch-run.sh:批量处理。
  调用gen_asm.py脚本生成test程序输出到auto.c ，并编译、运行，同时用time命令测量编译和运行花费的时间。

- block-size.sh:块大小。
  file变量保存600.perlbench_s.asm，cur_size=0,打印$file并存入line，读取行数，如果小于2，则打印cur_size。

- coverge.sh:

- code-analiisys.sh:代码分析。
  调用gen_mem_only.py脚本生成xx输出到auto.c，调用split_c.sh脚本，继续调用compile-all.sh脚本生成run_auto_0。用交叉编译器aarch64反汇编run_auto_0，并进行分析。
  调用gen_asm.py脚本生成xx输出到auto.c，调用split_c.sh脚本，继续调用compile-all.sh脚本生成run_auto_0。用交叉编译器aarch64反汇编run_auto_0，并进行分析。

- compile-all.sh:编译
  删除所有已生成的二进制文件run_auto_*，grep选取c_auto/中的auto_0，循环输入给line变量，取名字，并且编译。
