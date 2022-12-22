# 1、Elf格式不支持

场景：qemu-sw64执行ELF MACHINE=0x9906的hello程序会报 Invalid ELF image for this architecture。ELF MACHINE=0x9916的不会，不论是动态编译的还是静态编译的。

报错信息：

172.16.129.166：

> hello-sw-static-9916:
> hello
> 
> hello-sw-dynamic-9916:
> qemu-sw64：/lib/ld-linux.so.2：Invalid ELF image for this architecture
> 
> hello-sw-static-9906:
> qemu-sw64：test/hello-sw-static-9906：Invalid ELF image for this architecture
> 
> hello-sw-dynamic-9906:
> qemu-sw64：test/hello-sw-dynamic-9906：Invalid ELF image for this architecture

暂时解决方案：linux-user/elfload.c：2626 注释掉，这样就可以正常读取了

> hello-sw-static-9916:
> hello
> 
> hello-sw-dynamic-9916:
> hello-sw-dynamic-9916: error while loading shared libraries: cannot create cache for search path：Cannot allocate memory
> 
> hello-sw-static-9906:
> hello
> 
> hello-sw-dynamic-9906:
> hello-sw-dynamic-9906: error while loading shared libraries: cannot create cache for search
> path：Cannot allocate memory

全部修改后

> hello-sw-static-9916:
> hello
> 
> hello-sw-dynamic-9916:
> hello
> 
> hello-sw-static-9906:
> hello
> 
> hello-sw-dynamic-9906:
> hello

# 2、读取动态库出错

测试用例：动态编译的hello程序

报错信息：

执行动态编译程序时

172.16.129.177：

> hello-sw-dynamic-9916: error while loading shared libraries: /lib/libc.so.6.1: cannot read file data: Error 9

172.16.129.166：

> hello-sw-dynamic-9916: error while loading shared libraries: cannot create cache for search path: Cannot allocate memory

## 做的无用功

第799个tb块执行时出现问题，看它翻译时

mov指令手册里没有，代码里也没有注释

ldl $r16 ,1136(fp) 从存储器装入长字到寄存器

pc错了

| main                                       | 5、CPUArchState *env;env = cpu->env_ptr;<br/>6、target_cpu_copy_regs(env, regs);env->pc = regs->pc;<br/>7、struct target_pt_regs regs1, *regs = &regs1;ret = loader_exec(execfd, exec_path, target_argv, target_environ, regs, info, &bprm);                                                                                                                                                                                                                                                                  |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| loader_exec do_init_thread<br/>init_thread | 8、do_init_thread(regs, infop);init_thread(regs, infop);regs->pc = infop->entry;一路都是info传进来<br/>9、retval = load_elf_binary(bprm, infop);                                                                                                                                                                                                                                                                                                                                                                    |
| load_elf_binary<br/>load_elf_image         | 10、3158load_elf_image(bprm->filename, bprm->fd, info, &elf_interpreter, bprm->buf);2776info->entry = ehdr->e_entry + load_bias;正确赋值<br/>11、3231if (elf_interpreter) { info->load_bias = interp_info.load_bias; info->entry = interp_info.entry;错误赋值<br/>12、struct image_info interp_info;3201if (elf_interpreter) { load_elf_interp(elf_interpreter, &interp_info, bprm->buf);<br/>13、load_elf_image(filename, fd, info, NULL, bprm_buf);info->entry = ehdr->e_entry + load_bias;错误赋值<br/>14、ehdr->e_entry |
| tb_find                                    | 4、target_ulong pc;cpu_get_tb_cpu_state(env, &pc, &cs_base, &flags); *pc = env->pc;<br/>                                                                                                                                                                                                                                                                                                                                                                                                                    |
| tb_gen_code                                | 3、TranslationBlock *tb,tb->pc = pc;                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| gen_intermediate_code                      | 2、tb->pc出错                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| translator_loop                            | 1、tb->pc出错                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |

ehdr->e_type=EXEC DYN

动态

第一回load_elf_image 加载elf镜像

经过for循环x86 loaddr变0 ，而sw变为0x120000，

经过target_mmap x86 load_addr变为0x40000，sw变为0x12000。

2775:

sw-exec edhr->e_entry=0x120000470 load_bias=0x0 info->entry=0x120000470

x86-exec edhr->e_entry=0x401050 load_bias=0x0 info->entry=0x401050

x86-dyn edhr->e_entry=0x1060 load_bias=0x4000000000 info->entry=0x4000001060

第二回load_elf_interp中的load_elf_image 加载解释器

2775:

sw-exec edhr->e_entry=0x22e0 load_bias=0x4000802000 info->entry=0x40008042e0

x86-exec edhr->e_entry=0x1090 load_bias=0x4000802000 info->entry=0x4000803090

x86-dyn edhr->e_entry=0x1090 load_bias=0x4001808000 info->entry=0x4001809090

## 最终查错过程

### 错误1

> hello-sw-dynamic: error while loading shared libraries:/lib/libc.so.6.1: cannot read file data：Error 9

#### 定位过程

4008042e0 
4008042e4

4008042f0

400805358

last_tb 1780

5229

单步tb=5229 实际tb=55

0x40008055a4: rtc    $r1

rtc指令有问题

bne指令涉及到rtc的寄存器

5378 clr 和 bis是一个东西吗

- 5230 时 两个r1值不同，像是随机的，上一条是rtc取的计时器，没问题

- 5234 时上一条subw，字减，正常情况下，像是溢出了是个很大的数，qemu情况非常规整的数

- 5246 变得不一样，r1值

- 5265 r1一样了

- 5268 不一样

sp赋给r10，sp可能不一样

5232 mov r10 r16 地址：55b0

r10的值不一样，导致r16不一样

5245 ldi $r1,8($r16)

r16装入r1,r1是随机的，问题出在r16

- 5266 mov r16 r2
  
  qemu好像没有mov成功，r16的值也不一样，并且往前看r1值也不一样

- 5359 ldl r1,8(r2) r2+8取地址的值装入r1
  
  此处r2不一样
  
  qemu r2=0x40008000b8
  
  正常 r2=0x11fffe0a8
  
  此时r1都是0

- 5360
  
  qemu r1=0x3
  
  正常 r1=0x21
  
  此时r1不一样

- 5400
  
  一直到这里才又对r1的值进行了修改，当然修改后也不一样qemu r1=0 正常 r1=0x1e

- 5406 addl r29,r1,$r1
  
  qemu情况r1=0xfffffffffffd9b20,r29=4000846030 r1=0x400081fb50
  
  正常情况r1=0xfffffffffffd99b0,r29=0x400000044030 r1=0x40000001d9e0
  
  r29正常，r1错误

- 5407 jmp $r31,($r1),0x400081f900
  
  两个跳转不同,qemu r1=0x400081fb50,正常r1=0x40000001d9e0

插打印

_dl_start elf/rtld.c

_dl_setup_hash

_dl_start_final

_dl_sysdep_start elf/dl-sysdep.c:253

dl_main elf/rtld.c:1731

_dl_map_object_deps elf/dl-deps.c:248

qemu

19539 进入_dl_map_object_deps

开单步tb=21050 不开单步tb=633，地址80b828，ble $r0,0x400080c0ec 跳转了，

正常没有跳转 ，qemu open_verify+216 跳转到2460

上面的一个系统调用出错

dl-load.c 1549行跳转

出问题是__open64_nocancel 有两次，是第二次

报错在__read_nocancel

qemu中处理系统调用，处理流程

do_syscall->do_syscall1->······，这个系统调用会进入到 linux-user/host/sw_64/safe-syscall.inc.S 中处理。

#### 解决方案

sw特有的架构/usr/include/asm/fcntl.h

通用的架构/usr/include/asm-generic/fcntl.h

qemu中sw分支用的是通用分支，要写定义它特有的分支，在qemu6-system/linux-user/sw64/target_fcntl.h中定义，照抄/usr/include/asm/fcntl.h就行。

### 错误2

> hello-sw-dynamic: error while loading shared libraries: libc.so.6.1: cannot cannot create shared object descriptor: Error 9

#### 定位过程

_dl_catch_execption上面不远处有_dl_new_object

tb=21994 mmap

函数调用关系

_start->_dl_start_final->_dl_sysdep_start->dl_main->_dl_map_object_deps->openaux->_dl_map_object->_dl_map_object_from_fd->_dl_new_object->malloc->calloc:73

```shell
page = __mmap(0,nup,PROT_READ|PROT_WRITE,MAP_ANON|MAP_PRIVATE,-1,0)
if(page == MAP_FAILED)//map失败导致分配空间失败，返回NULL
    return NULL;

__mmap64()
return (void)
```

一共有两次_dl_map_object，qemu第二次分配空间出错，最后一分配在malloc函数中判断if page==MAP_FAILED-1，直接return null，正常也会进上一个if，但这个if判断不会进。

两个calloc参数都是一样的，都是591，49c

dl_object.c

dl_mininal.c

tb=22006块qemu跳转了，正常没有跳转，__mmap64+48

bne $r19,0x40008233d0

两边之前都是从r19=0x12，之后qemu的r19=0x1，正常r19=0

22004出问题

#### 解决方案

sw特有的架构 /usr/include/asm/mmani.h

通用的架构 /usr/include/asm-generic/mmani.h

qemu中sw分支用的是通用分支，要写定义它特有的分支，在qemu6-system/linux-user/syscall_defs.h中定义，照抄/usr/include/asm/mmani.h就行。 

# 3、VCOND、VCONS指令翻译错误

测试环境：114 6B

报错用例：spec2006 456.hmmer

## 错误一：返回值非0

> 报错信息：
> 
> 456.hmmer: copy 0 non-zero return code(exit code=1, signal=0)
> Contents of bimbesin.err
> FATAL: fit failed; --num may be set too small? 
> 
> 错误类型：返回值非0

/lib /usr/lib /usr/sw/sw....../usr/

libthread_db.so->libthread_db.so.1->libthread_db-1.0.so

该错误不能稳定复现，插打印后有几率不报该错误，暂时放下，先解决第二个

第76个log

## 错误二：结果出错

> 报错信息：
> 
> Miscompare of bimbesin.out;
> 
> 0014: HMM    :   Bombesin
> 0015: mu        :   -5.959400
>           mu        :   -5.959401
>                                          ^
> 0016: lamba   :    0.576502
> 0017: max       :   7.707000
> 0018: //
> 
> 错误类型：第一行是基准，第二行是实际执行结果，箭头指向出错的地方。结果出错，应该是0，结果算成了1。

### 定位过程：

VCOND VCONS 

### 解决方案：

指令执行步骤对应的函数顺序错误，参数错误。

# 4、FADDD指令翻译错误

测试环境：114 6B

报错用例：spec2006 416.gamess

> 报错信息：
> 
> Contents of exam29.err
> 
> Note: The following floating-point exceptions are signalling: IEEE_INVALID_FLAG IEEE_DIVIDE_BY_ZERO IEEE_OVERFLOW_FLAG IEEE_UNDERFLOW_FLAG IEEE_DENORMAL
> STOP IN ABRT
> 
> DIMENSIONS OF THE SYMMETRY SUBSPACES ARE 
> ERROR IN -SYMCUP- NOT ENOUGH COUPLING VECTORS FOUND.
> EXECUTION OF GAMESS TERMINATED -ABNORMALLY- AT

### 定位过程：

NINT函数 四舍五入函数 faddd指令翻译helper函数

### 解决方案：

关闭simple-float选项

# 5、系统调用getxid不支持

测试环境：114 6B

报错用例：spec2006 400.perlbench

> 报错信息：
> 
> Contents of splitmail.1600.12.26.16.4500.err
> 
> ERROR: setuid/setgid execution not supported!
> BEGIN failed--compilation aborted at lib/mhamain.pl line 59.
> Compilation failed in require at splitmail.pl line 51.

## 定位过程：

系统调用出错：

相关C函数：

getpid() getppid()

getuid() geteuid()

getgid() getegid()

相关系统调用：

不同机器上这些函数翻译成的指令有所不同

6A:113 系统调用名 getppid geteuid getegid

6B:114 系统调用名 

6B:121 系统调用名 getpid getuid getgid

## 解决方案：

相关系统调用宏未定义，TARGET宏未定义，系统调用代码未实现

# 6、RTID指令翻译错误

测试环境：121，系统版本1050d

是因为core3比较新？

执行所有程序时，均会报段错误。

> 报错信息：
> 
> Segments Faults  段错误

测试用例：hello程序

定位：段错误停在code_gen_buffer即执行时，往上查是哪一个翻译块，在查是哪一条指令。

```asm6502
0x12003f52c <ptmalloc_init+60>: rtid $r0
```

本地执行r0会有值，qemu翻译执行r0为0。

同时本地执行经gdb调试`info r a`后观察发现r0的值是由unique寄存器的值传来的。

错误原因：rtid没有翻译，是调用helper函数返回csr->[unique]，而csr为空。说是老版内核不支持读写csr寄存器（因为是硬件相关），新版本支持了，而qemu的sw前后端并没有实现。

# 7、ELF 格式不支持 续

问题原因：历史遗留问题，申威没有申请ELF MACHINE号，同时有0x9906和0x9916两个号码。而Qemu只添加了对0x9916的支持。

解决方案一：注释Qemu对ELF的检查。

解决方案二：添加Qemu对0x9906的支持。
