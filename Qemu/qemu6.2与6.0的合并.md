命令：
diff -ruNa  > qemu-6-6.2.patch 
patch -p0 -Et < qemu-6-6.2.patch

diff: qemu6/roms/edk2/EmulatorPkg/Unix/Host/X11IncludeHack: No such file or directory
diff: qemu-6.2.0-work1/roms/edk2/EmulatorPkg/Unix/Host/X11IncludeHack: No such file or directory
diff: qemu6/roms/skiboot/ccan/heap/LICENSE: No such file or directory

区别

前端中
6.0没有\target\sw sw架构，6.2有了sw相关架构,这一块补丁信息保留
\tcg\tcg-op.c 
\include\tcg\tcg-op.h 
\tcg\tcg.c
后端中
6.0有了\tcg\sw_64架构，6.2貌似也做了sw64架构，决定删除，不更新这一块
\tcg\tcg.c tcg_reg_alloc_op有一处改动

其他
\util 多个文件
\ui   多个文件
\.    yml文件
\trace
\tools
\tests

我的思路：

后端相关文件不动，把patch中相关文件的改动删除，其他更新，然后测试

\tcg\sw_64\，这个目录下的补丁信息都要删除，同时\tcg\sw64补丁信息也要删除

1版本
diff -ruNa qemu6 qemu-6.2.0-work1 > qemu-6-6.2.patch

\tcg\sw_64\
开始行号 5805733
结束行号 5808682

\tcg\sw64
开始行号 5808683
结束行号 5810942

\log 日志信息，这部分补丁信息也删除
开始行号 2497570
结束行号 2497913

1版本 可行
diff -ruNa qemu6/ qemu-6.2.0-work1/ > qemu-6-6.2.patch 
2版本 lot 可行
diff -ruNa -x capstone -x .git* -x git* -x llvm -x log -x Makefile.bak -x README -x qemu-options-wrapper.h -x VERSION qemu6/ qemu-6.2.0-work1/ > qemu-2.patch
patch -p1 -Et < ../qemu-2.patch
capstone default-configs \.git .gitattributes gitdm.config .github .gitignore .gitlab-ci.d .gitlab-ci.yml .gitmodules .gitpublish git_version.sh log Makefile.bak README qemu-options.h qemu-options-wrapper.h replication.h VERSION
开始行号 105831
结束行号 2348370

capstone diff信息
35268
54191

会报错：原因6.2修改了预处理命令中头文件的地址，导致找不到，6.2路径下也没有，编译不会出现这种问题？
找不到头文件 
../migration/colo.c 31 #include "block/replication.h"
/os-posix.c 35 #include "qemu/qemu-options.h"

3版本 little 

mian.c 增删头文件
6.0 default-configs 6.2 configs 6.0 tests\acceptance    6.2 tests\avocado    
6.2 多了个ebpf ，run_kvm_config.sh 
少了git_version,log，Makefile.bak,qemu-options.h,qemu-options-wrapper.h,README,replication.h

正确做法：
git log 查看提交记录
git diff 当前版本 原始版本 > diff  比较当前版本与原始版本
cat diff | grep "diff --git" 
cat diff | grep "diff --git" | wc -l
cat diff | grep "diff --git" | grep -v sw64 > diff-vsw64
cat diff | grep "diff --git" | grep sw64 > diffsw64

diff --git a/configure b/configure
diff --git a/disas.c b/disas.c
diff --git a/disas/meson.build b/disas/meson.build
diff --git a/hw/Kconfig b/hw/Kconfig
diff --git a/hw/meson.build b/hw/meson.build
diff --git a/hw/rtc/sun4v-rtc.c b/hw/rtc/sun4v-rtc.c
diff --git a/include/disas/dis-asm.h b/include/disas/dis-asm.h
diff --git a/include/elf.h b/include/elf.h
diff --git a/include/qemu/atomic.h b/include/qemu/atomic.h
diff --git a/include/qemu/timer.h b/include/qemu/timer.h
diff --git a/include/sysemu/arch_init.h b/include/sysemu/arch_init.h
diff --git a/linux-user/elfload.c b/linux-user/elfload.c
diff --git a/linux-user/meson.build b/linux-user/meson.build
diff --git a/linux-user/syscall_defs.h b/linux-user/syscall_defs.h
diff --git a/meson.build b/meson.build
diff --git a/meson/mesonbuild/envconfig.py b/meson/mesonbuild/envconfig.py
diff --git a/pc-bios/core3-hmcode b/pc-bios/core3-hmcode
diff --git a/pc-bios/core3-reset b/pc-bios/core3-reset
diff --git a/pc-bios/core4-hmcode b/pc-bios/core4-hmcode
diff --git a/pc-bios/meson.build b/pc-bios/meson.build
diff --git a/pc-bios/uefi-bios-sw b/pc-bios/uefi-bios-sw
diff --git a/qapi/machine.json b/qapi/machine.json
diff --git a/run_kvm_config.sh b/run_kvm_config.sh
diff --git a/softmmu/qdev-monitor.c b/softmmu/qdev-monitor.c
diff --git a/target/Kconfig b/target/Kconfig
diff --git a/target/meson.build b/target/meson.build

cat diff | grep "diff --git" | grep -v sw64 | wc -l 
cp -r qemu-6.2.0-work1/ qemu-6.2.0-work1.orig
cd qemu-6.2.0-work1.orig
git log
git -reset --hard 原始版本         回退到原始版本
vimdiff ../qemu-6.2.0-work1.orig/configure configure

问题diff-vsw64
1、/include/disas/dis-asm.h
bfd_arch_sw_64,      /* sw64  */
#define bfd_mach_sw_64 1
#define bfd_mach_sw_64_core3  0x10

int print_insn_sw_64            (bfd_vma, disassemble_info*);
int print_insn_sw64             (bfd_vma, disassemble_info*);
我都没有改动
2、/include/sysemu/arch_init.h
6.0 没有QEMU_ARCH_HEXAGON = (1 << 22),这一行，我先加上了
include/elf.h
加了210,1466这一块，与670重复了
6是9906 6.2是9916，且6有ifdefine

3、meson.build
上面新加了riscv相关的if，决定加上
同时cpu sw_64=sw64
4、pc-bios没有那些文件 /target/Kconfig run_kvm_config.sh 
5、softmmu/qdev-monitor.c
6.0一大块都没有，加上。证明不应该加

diffsw64
6.0 没有hw/sw64，linux-headers/asm-sw64,linux-user/sw64,target/sw64加上
6.0的default-configs对应 6.2的configs，需要添加.mak文件 
/disas/sw64.c相差很大，决定不改 还有/tcg/sw64,决定不改，用6.0的

用户级：
测试
1、../disas/meson.build:26:10: ERROR: File sw64.c does not exist.

2、../hw/meson.build:64:16: ERROR: Expecting rparen got number.
subdir('sparc64'0)
      ^_________^
删除0
3、../accel/tcg/cpu-exec.c:47:12: error: ‘tcg_qemu_tb_exec_num’ defined but not used [-Werror=unused-variable]
   47 | static int tcg_qemu_tb_exec_num=0;
注释了
4、../softmmu/qdev-monitor.c:57: error: "QEMU_ARCH_VIRTIO_PCI" redefined [-Werror]
   57 | #define QEMU_ARCH_VIRTIO_PCI (QEMU_ARCH_ALPHA | QEMU_ARCH_ARM | \
      | 
In file included from ../softmmu/qdev-monitor.c:25:
/mnt/fei/qemu-2/qemu6-system/include/sysemu/arch_init.h:40: note: this is the location of the previous definition
   40 | #define QEMU_ARCH_VIRTIO_PCI (QEMU_ARCH_ALPHA | QEMU_ARCH_ARM | \
不在第一个里改，在第二个里改
5、.../disas/sw_64.c: In function ‘print_insn_sw_64’:
../disas/sw_64.c:1449:10: error: ‘bfd_mach_sw_64_sw6’ undeclared (first use in this function); did you mean ‘bfd_mach_sw_64’?
 1449 |     case bfd_mach_sw_64_sw6:
      |          ^~~~~~~~~~~~~~~~~~
      |          bfd_mach_sw_64
删去sw6
../disas/sw_64.c:1449:10: note: each undeclared identifier is reported only once for each function it appears in
At top level:
../disas/sw_64.c:621:1: error: ‘extract_bdisp26’ defined but not used [-Werror=unused-function]
  621 | extract_bdisp26 (unsigned insn, int *invalid ATTRIBUTE_UNUSED)
      | ^~~~~~~~~~~~~~~
../disas/sw_64.c:613:1: error: ‘insert_bdisp26’ defined but not used [-Werror=unused-function]
  613 | insert_bdisp26 (unsigned insn, int value, const char **errmsg)
注释掉
6、../softmmu/arch_init.c:97:28: error: ‘QEMU_ARCH’ undeclared here (not in a function); did you mean ‘QEMU_ARCH_RX’?
   97 | const uint32_t arch_type = QEMU_ARCH;
      |                            ^~~~~~~~~
      |                            QEMU_ARCH_RX
原因是编译sw相关时，QEMU_ARCH不能自动关联到QEMU_ARCH_SW,需要在/softmmu/arch_init.c中88行处添加
7、../target/sw64/cpu.c: In function ‘sw64_cpu_disas_set_info’:
../target/sw64/cpu.c: In function ‘sw64_cpu_disas_set_info’:
../target/sw64/cpu.c:126:18: error: ‘bfd_mach_sw64_core3’ undeclared (first use in this function); did you mean ‘bfd_mach_sw_64_core3’?
  126 |     info->mach = bfd_mach_sw64_core3;
      |                  ^~~~~~~~~~~~~~~~~~~
      |                  bfd_mach_sw_64_core3
../target/sw64/cpu.c:126:18: note: each undeclared identifier is reported only once for each function it appears in
../target/sw64/cpu.c:127:24: error: ‘print_insn_sw64’ undeclared (first use in this function); did you mean ‘print_insn_sw_64’?
  127 |     info->print_insn = print_insn_sw64;
      |                        ^~~~~~~~~~~~~~~
      |                        print_insn_sw_64
需要修改，因为在/include/disas/dis-asm.h我没有改动，仍然用的sw_64，所以这里需要改一下
8、
../target/sw64/cpu.c: At top level:
../target/sw64/cpu.c:230:10: fatal error: hw/core/sysemu-cpu-ops.h: No such file or directory
  230 | #include "hw/core/sysemu-cpu-ops.h"
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~
这是6.2版本本来就有的，添加
../hw/sw64/sw64_iommu.c:30:10: fatal error: hw/sw64/sw64_iommu.h: No such file or directory
   30 | #include "hw/sw64/sw64_iommu.h"
../hw/sw64/core3_board.c:21:10: fatal error: hw/sw64/sw64_iommu.h: No such file or directory
   21 | #include "hw/sw64/sw64_iommu.h"
这是自己当初看漏了，添加
9、
../target/sw64/cpu.c: In function ‘sw64_cpu_class_init’:
../target/sw64/cpu.c:265:7: error: ‘CPUClass’ {aka ‘struct CPUClass’} has no member named ‘sysemu_ops’
  265 |     cc->sysemu_ops = &sw64_sysemu_ops;
      |       ^~
../target/sw64/cpu.c:271:17: error: assignment discards ‘const’ qualifier from pointer target type [-Werror=discarded-qualifiers]
  271 |     cc->tcg_ops = &sw64_tcg_ops;
      |                 ^
6.2版本新增，/include/hw/core/cpu.h 192行cpuclass成员 197 tcg_ops加const
10、
In file included from ../target/riscv/cpu.h:23,
                 from ../softmmu/memory_mapping.c:17:
/mnt/fei/qemu-2/qemu6-system/include/hw/core/cpu.h:208:9: error: duplicate member ‘reset_dump_flags’
  208 |     int reset_dump_flags;
      |         ^~~~~~~~~~~~~~~~
/mnt/fei/qemu-2/qemu6-system/include/hw/core/cpu.h:209:9: error: duplicate member ‘gdb_num_core_regs’
  209 |     int gdb_num_core_regs;
      |         ^~~~~~~~~~~~~~~~~
/mnt/fei/qemu-2/qemu6-system/include/hw/core/cpu.h:210:10: error: duplicate member ‘gdb_stop_before_watchpoint’
  210 |     bool gdb_stop_before_watchpoint;
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~
貌似是重复定义，6.2中删除了，试一试
11、
../linux-user/syscall.c: In function ‘do_syscall1’:
../linux-user/syscall.c:9042:42: error: ‘struct target_old_sigaction’ has no member named ‘sa_restorer’
 9042 |                 act.sa_restorer = old_act->sa_restorer;
      |                                          ^~
../linux-user/syscall.c:9058:24: error: ‘struct target_old_sigaction’ has no member named ‘sa_restorer’
 9058 |                 old_act->sa_restorer = oact.sa_restorer;
      |                        ^~
解决办法：linux-user/syscall_defs.h 569   #if#endif都注释了，怀疑是宏没定义
../linux-user/syscall.c:9119:36: error: ‘restorer’ undeclared (first use in this function)
 9119 |                 act->ka_restorer = restorer;
      |                                    ^~~~~~~~
解决办法：9107加了一行
12、
/home/fei/qemu-2/qemu6-system/build/../linux-user/sw64/signal.c:262: undefined reference to `target_restore_altstack'
collect2: error: ld returned 1 exit status

linux-user/signal.c 300 6.2添加函数target_restore_altstack

系统级：
1、no core3-hmcode provided
把core3-hmcode拷贝进/build/pc-bios/文件夹中
2、qemu-system-sw64: -drive file=/home/fei/sw_qemu/test6b.raw,if=virtio: 'virtio-blk' is not a valid device model name,original_name=virtio-blk
报错在softmmu/qdev-monitor.c中
qom/object.c 93行
../softmmu/qdev-monitor.c 202 find_typename_by_alias函数为什么有问题，229插入了if
56行添加
自己添加的宏写错了

用自己编的内核测试

elf格式问题 linux-user/elfload.c ，这个格式问题是9906编译的hello的会报 Invalid ELF image for this architecture，9916的不会，不论是动态编译的还是静态编译的。目前166的这一版本注释了，暂时解决了。

执行动态编译程序时

177：

hello-sw-dynamic-9916:error while loading shared libraries：/lib/libc.so.6.1:cannot read file data:Error 9

166：

hello-sw-dynamic-9916:error while loading shared libraries: cannot create cache for search path:Cannot allocate memory

第799个tb块执行时出现问题，看它翻译时

mov指令手册里没有，代码里也没有注释

ldl  $r16 ,1136(fp)  从存储器装入长字到寄存器

pc错了

| main                                       | 5、CPUArchState *env;env = cpu->env_ptr;<br/>6、target_cpu_copy_regs(env, regs);env->pc = regs->pc;<br/>7、struct target_pt_regs regs1, *regs = &regs1;ret = loader_exec(execfd, exec_path, target_argv, target_environ, regs, info, &bprm);                                                                                                                                                                                                                                                                  |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| loader_exec do_init_thread<br/>init_thread | 8、do_init_thread(regs, infop);init_thread(regs, infop);regs->pc = infop->entry;一路都是info传进来<br/>9、retval = load_elf_binary(bprm, infop);                                                                                                                                                                                                                                                                                                                                                                    |
| load_elf_binary<br/>load_elf_image         | 10、3158load_elf_image(bprm->filename, bprm->fd, info, &elf_interpreter, bprm->buf);2776info->entry = ehdr->e_entry + load_bias;正确赋值<br/>11、3231if (elf_interpreter) { info->load_bias = interp_info.load_bias; info->entry = interp_info.entry;错误赋值<br/>12、struct image_info interp_info;3201if (elf_interpreter) { load_elf_interp(elf_interpreter, &interp_info, bprm->buf);<br/>13、load_elf_image(filename, fd, info, NULL, bprm_buf);info->entry = ehdr->e_entry + load_bias;错误赋值<br/>14、ehdr->e_entry |
| tb_find                                    | 4、target_ulong pc;cpu_get_tb_cpu_state(env, &pc, &cs_base, &flags); *pc = env->pc;<br/>                                                                                                                                                                                                                                                                                                                                                                                                                    |
| tb_gen_code                                | 3、TranslationBlock *tb,tb->pc = pc;                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| gen_intermediate_code                      | 2、tb->pc出错                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| translator_loop                            | 1、tb->pc出错                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |

动态

第一回load_elf_image

经过for循环x86 loaddr变0 ，而sw变为0x120000，

经过target_mmap x86 load_addr变为0x40000，sw变为0x12000

swentry022e0没有加 x86加了

第二回load_elf_interp中的load_elf_image

ehdr->e_type

sw entry没有加       x86加了