执行模块：
流程：
cpu_exec()开始执行，调用tb_find()去查找下一个翻译好的host codeTB，找到后调用cpu_loop_exec_tb()执行TB。
cpu_exec()执行。调用tb_find()和cpu_loop_exec_tb()。

tb_find()：查找下一个TB。
具体流程：
1)首先根据pc值，通过tb_lookup() 查找是否存在已有TB；
2)如果当前位置找不到已有TB，就调用tb_gen_code() 生成一个新的TB（tb_gen_code()作为TCG翻译的入口函数），并将翻译后的新TB加入cpu->tb_jmp_cache中；
3)若last_tb存在，则调用tb_add_jump() 将上面获得的TB链接到last_tb后面。

tb_lookup()：具体查找TB
1)获取TB的hash：tb_jmp_cache_hash_func() 根据pc值获取存储TB的hash表索引
一级缓存（fast path）；
2)从cpu->tb_jmp_cache获取TB，如果取到，且判断正确，则返回TB；tb_jmp_cache[]是hash table
3)如果一级缓存没能命中有效TB，则进入二级缓存（slow path）查找 -- tb_htable_lookup()，若找到就将结果放入cpu->tb_jmp_cache（即二级缓存中命中的TB都会被直接更新到一级缓存中，以增加一级缓存的TB命中率，从而提高TB查找效率）。

cpu_loop_exec_tb()：执行TB，有一些判断条件，判断下一个TB
cpu_tb_exec()：执行TB
具体流程
4)获取已翻译代码的地址：itb->tc.ptr
5)TCG目标会使用不同的tcg_qemu_tb_exec定义(调用tcg_target_qemu_prologue()生成的prologue.code)
返回Last_tb

前端模块：翻译guest code
tb_gen_code():翻译guest code
get_page_addr_code()：获取phys_pc，即Guest OS PC (Program counter) 的物理内存地址
tcg_tb_alloc()：在TCGContext中给TB分配空间
gen_intermediate_code()：生成中间代码存放在TCGContext中
tcg_gen_code()：将中间代码转换成host code
tb_link_page()：添加新的TB，并将其与物理页表链接

gen_intermediate_code()：调用translator_loop()进行块的中间代码翻译
tanslator_loop()：是架构通用的，依赖目标特定的翻译器操作符，此处是alpha架构对应的alpha_tr_ops。
将alpha架构的操作封装在TranslatorOps ops中，并用ops->func调用。
函数主要有
gen_tb_start()
translate_insn()--translate_one()--tcg_gen_<op>()--tcg_gen_op<n>_i<32/64>()--tcg_gen_op<n>()--tcg_emit_op()

gen_tb_end()--tcg_gen_exit_tb()

TCG后端的主要功能是把中间代码（TCG Operations）转化成Host Code。
tcg_reg_alloc_op():tcg_reg_alloc_op()中根据输入参数、输出参数的有效性进行寄存器分配，之后调用tcg_out_op()进行tcg-target代码的转换。
tcg_out_op()主要功能就是依据TCG操作码，调用对应的转换函数将操作转换成host机器码。

> 问题一：
> 
> cpu_tb_exec()->tcg_qemu_tb_exec 进不去，找不到定义
> 
> 问题二：
> 
> tb_lookup()这个函数sourceinsight找不到啊

- [ ] 问题一：
  
  cpu_tb_exec()->tcg_qemu_tb_exec 进不去，找不到定义，函数指针
  
  问题二：
  
  tb_lookup()这个函数sourceinsight找不到啊，很多函数都不好找
  
  问题三：
  
  i386的translator_loop在accel/tcg/translator.c
  
  gen_intermediate_code,i386_tr_translate_insn在target/i386/tcg/translate.c

# 源码分析 Version 6.0

用户模式user mode

linux-user/main.c:main()->

linux-user/sw64/cpu_loop.c: cpu_loop()->

accel/tcg/cpu-exec.c：cpu_exec()

 ![](qemu源码分析.assets/2022-07-14-10-30-30-image.png)

# 1.设备创建

### type_init

> include/qemu/module.h:type_init,module_init 构造函数，用于构造东西

```c
#define type_init(function) module_init(function, MODULE_INIT_QOM) //（gdb）macro expand type_init(function)
#define type_init(function, type) 
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \     //__attribute__gcc特性，main函数之前执行
{                                                                           \
    register_module_init(function, type);                                   \
}
```

> util/module.c:register_module_init

```c
void register_module_init(void (*fn)(void), module_init_type type)
{
    ModuleEntry *e;
    ModuleTypeList *l;

    e = g_malloc0(sizeof(*e));
    e->init = fn;
    e->type = type;

    l = find_type(type);

    QTAILQ_INSERT_TAIL(l, e, node);
}
```

> target/sw64/cpu.c:type_init(sw64_cpu_register_types) 注册sw64_cpu

```c
static void sw64_cpu_register_types(void)
{
    const SW64CPUInfo *info = sw64_cpus;

    type_register_static(&sw64_cpu_type_info);

    while (info->name) {
        cpu_register(info);
        info++;
    }
}
type_init(sw64_cpu_register_types) 
```

综上x86_cpu_register_types功能是：

将各种类型的cpu的信息存入一个hash表中。表中的每个元素是一种cpu类型，每个元素指定了cpu的name，parent等信息，以及instanc_init,  class_init 与具体的函数进行挂接。

后面通过查找这个hash表，就可以找到某个类型cpu的所有描述信息，并且可以找到这类cpu的初始化函数并执行

# 2.main

> linux-user/main.c:main

```c
int main(int argc, char **argv)//usermode入口函数main
{
    TaskState *ts;
    CPUArchState *env;//保存cpu相关信息的数据结构，这里就是CPUSW64State
    CPUState *cpu;//cpu表示一个cpu核心或者说线程
    int optind;
    char **target_environ, **wrk;
    char **target_argv;
    int target_argc;
    int i;
    int ret;
    int execfd;
    int log_mask;
    unsigned long max_reserved_va;
    bool preserve_argv0;

    module_call_init(MODULE_INIT_TRACE);//遍历MODULE_INIT_TRACE类型的链表每一个节点（对应一个设备）的init函数，将typeimpl添加到hash表中。
    qemu_init_cpu_list();
    module_call_init(MODULE_INIT_QOM);//模型调用，调用注册好的类型设备

    envlist = envlist_create();

   /* add current environment into the list */
    for (wrk = environ; *wrk != NULL; wrk++) {
        (void) envlist_setenv(envlist, *wrk);
    }

    cpu_model = NULL;
    optind = parse_args(argc, argv);//这一步获取-cpu option参数(core3)并赋值给cpu_model

    /* Scan interp_prefix dir for replacement files. */
    init_paths(interp_prefix);//库文件路径

    init_qemu_uname_release();//检查内核版本
    ...
    if (cpu_model == NULL) {
        cpu_model = cpu_get_model(get_elf_eflags(execfd));//获取elf文件的cpu_model
    }
    cpu_type = parse_cpu_option(cpu_model);//记录可选参数作为全局变量，返回cpu_type=core3-sw64-cpu

    /* init tcg before creating CPUs and to get qemu_host_page_size */
    {
        AccelClass *ac = ACCEL_GET_CLASS(current_accel());

        ac->init_machine(NULL);//初始化TCG,tcg_init
        accel_init_interfaces(ac);
    }
    cpu = cpu_create(cpu_type);//创建CPUState对象，实例化cpu并实现它，返回cpu，即CPUState
    env = cpu->env_ptr;//将cpu->env_ptr赋值给env
    cpu_reset(cpu);
    thread_cpu = cpu;//cpu就是thread_cpu
    ...
    /* build Task State */
    ts->info = info;
    ts->bprm = &bprm;
    cpu->opaque = ts;
    task_settid(ts);
    //装载ELF文件
    ret = loader_exec(execfd, exec_path, target_argv, target_environ, regs,
        info, &bprm);//加载应用程序，除此之外还有环境变量，参数
    if (ret != 0) {
        printf("Error while loading %s: %s\n", exec_path, strerror(-ret));
        _exit(EXIT_FAILURE);
    }
    ...
    if (log_mask) {
        int mask;

        mask = qemu_str_to_log_mask(log_mask);
        if (!mask) {
            qemu_print_log_usage(stdout);//-d 参数在哪儿读取,util/log.c 中的qemu_log_items qemu_print_log_usage
            exit(1);
        }
        qemu_set_log(mask);
    } }    
    ...
    cpu_loop(env);//翻译执行循环函数
    ...
}
```

## 设备初始化

```c
module_call_init(MODULE_INIT_QOM);


void module_call_init(module_init_type type)
{
    ModuleTypeList *l;
    ModuleEntry *e;

    if (modules_init_done[type]) {
        return;
    }

    l = find_type(type);

    QTAILQ_FOREACH(e, l, node) {
        e->init();//初始化
    }

    modules_init_done[type] = true;
}
```

> target/sw64/cpu.c:type_init(sw64_cpu_register_types) 注册sw64_cpu

```c
static void sw64_cpu_register_types(void)
{
    const SW64CPUInfo *info = sw64_cpus;

    type_register_static(&sw64_cpu_type_info);

    while (info->name) {
        cpu_register(info);
        info++;
    }
}
type_init(sw64_cpu_register_types) 
```

## ELF装载

> linux-user/linuxload.c:loader_exec

```c
//filename：要加载的ELF文件的名称
//target_argv：qemu运行的参数，在这里即hello（hello是生成的可执行文件名， $qemu hello）
//target_environ：执行qemu的shell的环境变量
//regs,info,bprm是ELF文件加载过程中涉及的三个重要数据结构。
loader_exec(int fdexec, const char *filename, char **argv, char **envp,
             struct target_pt_regs * regs, struct image_info *infop,
             struct linux_binprm *bprm)
{
    int retval;

    bprm->fd = fdexec;//bprm和内核中相同，一个对象程序的所有信息都有了
    bprm->filename = (char *)filename;
    bprm->argc = count(argv);
    bprm->argv = argv;
    bprm->envc = count(envp);
    bprm->envp = envp;
    /*1. 要加载文件的属性判断：是否常规文件，是否可执行文件，是否ELF文件； 2. 读取ELF文件的前1024个字节*/
    retval = prepare_binprm(bprm);

    if(retval>=0) {/*prepare_binrpm函数已经读出了目标文件的前1024个字节，先判断下这个文件是否是ELF文件，即前4个字节*/
        if (bprm->buf[0] == 0x7f //ASCII的DEL控制符 魔数
                && bprm->buf[1] == 'E'
                && bprm->buf[2] == 'L'
                && bprm->buf[3] == 'F') {//elf文件处理函数
            retval = load_elf_binary(bprm, infop);//根据segment的地址将各个段映射到对应的位置，如果不幸和qemu中已有地址冲突了，直接退出了
#if defined(TARGET_HAS_BFLT)
        } else if (bprm->buf[0] == 'b'
                && bprm->buf[1] == 'F'
                && bprm->buf[2] == 'L'
                && bprm->buf[3] == 'T') {
            retval = load_flt_binary(bprm, infop);
#endif
        } else {
            return -ENOEXEC;
        }
    }

    if(retval>=0) {
        /* success.  Initialize important registers */
        do_init_thread(regs, infop);
        return retval;
    }

    return(retval);
}
```

### load_elf_binary

> linux-user/elfload.c:load_elf_binary

```c
load_elf_binary() {
     load_elf_image(bprm->filename, bprm->fd, info, &elf_interpreter, bprm->buf);   //加载程序镜像
     setup_arg_pages(bprm, info);        //准备参数
     bprm->p = copy_elf_strings(1, &bprm->filename, scratch, bprm->p, info->stack_limit); 
     info->file_string = bprm->p;      //文件名
     bprm->p = copy_elf_strings(bprm->envc, bprm->envp, scratch, bprm->p, info->stack_limit);
     info->env_strings = bprm->p;       //环境变量
     bprm->p = copy_elf_strings(bprm->argc, bprm->argv, scratch, bprm->p, info->stack_limit);
     info->arg_strings = bprm->p;      //参数
}
```

### load_elf_image

```c
static void load_elf_image(const char *image_name, int image_fd,
                           struct image_info *info, char **pinterp_name,
                           char bprm_buf[BPRM_BUF_SIZE])
{
    struct elfhdr *ehdr = (struct elfhdr *)bprm_buf;
    struct elf_phdr *phdr;
    abi_ulong load_addr, load_bias, loaddr, hiaddr, error;
    int i, retval, prot_exec;
    Error *err = NULL;

    /* First of all, some simple consistency checks */
    if (!elf_check_ident(ehdr)) {
        error_setg(&err, "Invalid ELF image for this architecture");
        goto exit_errmsg;
    }
    bswap_ehdr(ehdr);
    if (!elf_check_ehdr(ehdr)) {
        error_setg(&err, "Invalid ELF image for this architecture");
        goto exit_errmsg;
    }

    i = ehdr->e_phnum * sizeof(struct elf_phdr);
    if (ehdr->e_phoff + i <= BPRM_BUF_SIZE) {
        phdr = (struct elf_phdr *)(bprm_buf + ehdr->e_phoff);
    } else {
        phdr = (struct elf_phdr *) alloca(i);
        retval = pread(image_fd, phdr, i, ehdr->e_phoff);
        if (retval != i) {
            goto exit_read;
        }
    }
    bswap_phdr(phdr, ehdr->e_phnum);

    info->nsegs = 0;
    info->pt_dynamic_addr = 0;

    mmap_lock();

    /*
     * Find the maximum size of the image and allocate an appropriate
     * amount of memory to handle that.  Locate the interpreter, if any.
     */
    loaddr = -1, hiaddr = 0;
    info->alignment = 0;
    for (i = 0; i < ehdr->e_phnum; ++i) {
        struct elf_phdr *eppnt = phdr + i;
        if (eppnt->p_type == PT_LOAD) {
            abi_ulong a = eppnt->p_vaddr - eppnt->p_offset;
            if (a < loaddr) {
                loaddr = a;
            }
            a = eppnt->p_vaddr + eppnt->p_memsz;
            if (a > hiaddr) {
                hiaddr = a;
            }
            ++info->nsegs;
            info->alignment |= eppnt->p_align;
        } else if (eppnt->p_type == PT_INTERP && pinterp_name) {
            g_autofree char *interp_name = NULL;

            if (*pinterp_name) {
                error_setg(&err, "Multiple PT_INTERP entries");
                goto exit_errmsg;
            }

            interp_name = g_malloc(eppnt->p_filesz);

            if (eppnt->p_offset + eppnt->p_filesz <= BPRM_BUF_SIZE) {
                memcpy(interp_name, bprm_buf + eppnt->p_offset,
                       eppnt->p_filesz);
            } else {
                retval = pread(image_fd, interp_name, eppnt->p_filesz,
                               eppnt->p_offset);
                if (retval != eppnt->p_filesz) {
                    goto exit_read;
                }
            }
            if (interp_name[eppnt->p_filesz - 1] != 0) {
                error_setg(&err, "Invalid PT_INTERP entry");
                goto exit_errmsg;
            }
            *pinterp_name = g_steal_pointer(&interp_name);
        } else if (eppnt->p_type == PT_GNU_PROPERTY) {
            if (!parse_elf_properties(image_fd, info, eppnt, bprm_buf, &err)) {
                goto exit_errmsg;
            }
        }
    }

    if (pinterp_name != NULL) {
        /*
         * This is the main executable.
         *
         * Reserve extra space for brk.
         * We hold on to this space while placing the interpreter
         * and the stack, lest they be placed immediately after
         * the data segment and block allocation from the brk.
         *
         * 16MB is chosen as "large enough" without being so large
         * as to allow the result to not fit with a 32-bit guest on
         * a 32-bit host.
         */
        info->reserve_brk = 16 * MiB;
        hiaddr += info->reserve_brk;

        if (ehdr->e_type == ET_EXEC) {
            /*
             * Make sure that the low address does not conflict with
             * MMAP_MIN_ADDR or the QEMU application itself.
             */
            probe_guest_base(image_name, loaddr, hiaddr);
        } else {
            /*
             * The binary is dynamic, but we still need to
             * select guest_base.  In this case we pass a size.
             */
            probe_guest_base(image_name, 0, hiaddr - loaddr);
        }
    }
```

## CPU创建 ，cpu初始化

init machine     accel_commmom_init

cpu_create        object_new   cpu     sw64_cpu_initfn    core3_init

## SW：异常类型

```c
enum {
    EXCP_NONE,
    EXCP_HALT,
    EXCP_IIMAIL,
    EXCP_OPCDEC,
    EXCP_CALL_SYS,
    EXCP_ARITH,
    EXCP_UNALIGN,
#ifdef SOFTMMU
    EXCP_MMFAULT,
#else
    EXCP_DTBD,
    EXCP_DTBS_U,
    EXCP_DTBS_K,
    EXCP_ITB_U,
    EXCP_ITB_K,
#endif
    EXCP_CLK_INTERRUPT,
    EXCP_DEV_INTERRUPT,
    EXCP_SLAVE,
};
```

## SW：cpu_loop

> linux-user/sw64/cpu_loop.c:cpu_loop

```c
void cpu_loop(CPUSW64State *env)
{
    CPUState *cs = CPU(sw64_env_get_cpu(env));
    int trapnr;
    target_siginfo_t info;
    abi_long sysret;

    while (1) {
        cpu_exec_start(cs);//设置进入翻译执行状态时的相关参数，比如cpu->running == true
        trapnr = cpu_exec(cs);//翻译执行的主流程，返回一个int值，代表处理过程中遇到的异常
        cpu_exec_end(cs);//设置退出翻译执行时的相关参数
        process_queued_cpu_work(cs);//多线程状态下，处理翻译执行过程中其他线程插入的任务
      switch (trapnr) {//处理系统调用产生的异常
    case EXCP_OPCDEC://非法的操作码
            cpu_abort(cs, "ILLEGAL SW64 insn at line %d!", __LINE__);
    case EXCP_CALL_SYS://正常处理
        switch (env->error_code) {
            case 0x83:
                /* CALLSYS */
                trapnr = env->ir[IDX_V0];
                sysret = do_syscall(env, trapnr,
                                    env->ir[IDX_A0], env->ir[IDX_A1],
                                    env->ir[IDX_A2], env->ir[IDX_A3],
                                    env->ir[IDX_A4], env->ir[IDX_A5],
                                    0, 0);
                if (sysret == -TARGET_ERESTARTSYS) {
                    env->pc -= 4;
                    break;
                }
                if (sysret == -TARGET_QEMU_ESIGRETURN) {
                    break;
                }
                /* Syscall writes 0 to V0 to bypass error check, similar
                   to how this is handled internal to Linux kernel.
                   (Ab)use trapnr temporarily as boolean indicating error. */
                trapnr = (env->ir[IDX_V0] != 0 && sysret < 0);
                env->ir[IDX_V0] = (trapnr ? -sysret : sysret);
                env->ir[IDX_A3] = trapnr;
                break;
            default:
                printf("UNDO sys_call %lx\n", env->error_code);
                exit(-1);
            }
            break;
        case EXCP_MMFAULT:
            info.si_signo = TARGET_SIGSEGV;
            info.si_errno = 0;
            info.si_code = (page_get_flags(env->trap_arg0) & PAGE_VALID
                            ? TARGET_SEGV_ACCERR : TARGET_SEGV_MAPERR);
            info._sifields._sigfault._addr = env->trap_arg0;
            queue_signal(env, info.si_signo, QEMU_SI_FAULT, &info);
            break;
        case EXCP_ARITH:
            info.si_signo = TARGET_SIGFPE;
            info.si_errno = 0;
            info.si_code = TARGET_FPE_FLTINV;
            info._sifields._sigfault._addr = env->pc;
            queue_signal(env, info.si_signo, QEMU_SI_FAULT, &info);
            break;
        case EXCP_INTERRUPT:
            /* just indicate that signals should be handled asap */
            break;
        default:
            cpu_abort(cs, "UNDO");
        }
        process_pending_signals (env);

        /* Most of the traps imply a transition through HMcode, which
           implies an REI instruction has been executed.  Which means
           that RX and LOCK_ADDR should be cleared.  But there are a
           few exceptions for traps internal to QEMU.  */
        }
}
```

## cpu_exec

> accel/tcg/cpu-exec.c:cpu_exec

```c
int cpu_exec(CPUState *cpu)
{
    ...
    while (!cpu_handle_exception(cpu, &ret)) {//处理异常
        TranslationBlock *last_tb = NULL;
        int tb_exit = 0;

        while (!cpu_handle_interrupt(cpu, &last_tb)) {//处理中断
            uint32_t cflags = cpu->cflags_next_tb;
            TranslationBlock *tb;

            /* When requested, use an exact setting for cflags for the next
               execution.  This is used for icount, precise smc, and stop-
               after-access watchpoints.  Since this request should never
               have CF_INVALID set, -1 is a convenient invalid value that
               does not require tcg headers for cpu_common_reset.  */
            if (cflags == -1) {
                cflags = curr_cflags(cpu);
            } else {
                cpu->cflags_next_tb = -1;
            }

            tb = tb_find(cpu, last_tb, tb_exit, cflags);//在缓存中查找TB
            cpu_loop_exec_tb(cpu, tb, &last_tb, &tb_exit);//执行TB
            /* Try to align the host and virtual clocks
               if the guest is in advance */
            align_clocks(&sc, cpu);
        }
    ...
}
```

### tb_find

> accel/tcg/cpu-exec.c:tb_find

```c
static inline TranslationBlock *tb_find(CPUState *cpu,
                                        TranslationBlock *last_tb,
                                        int tb_exit, uint32_t cflags)
{
    CPUArchState *env = (CPUArchState *)cpu->env_ptr;
    TranslationBlock *tb;
    target_ulong cs_base, pc;
    uint32_t flags;

    cpu_get_tb_cpu_state(env, &pc, &cs_base, &flags);//根据env，初始化pc，cs_base和flags

    tb = tb_lookup(cpu, pc, cs_base, flags, cflags);//在cache中根据target PC查找TB，/include/exec/tb-lookup.h
    if (tb == NULL) {
        mmap_lock();
        tb = tb_gen_code(cpu, pc, cs_base, flags, cflags);//生成TB
        mmap_unlock();
        /* We add the TB in the virtual pc hash table for the fast lookup */
        qatomic_set(&cpu->tb_jmp_cache[tb_jmp_cache_hash_func(pc)], tb);
    }
    #ifndef CONFIG_USER_ONLY
    /* We don't take care of direct jumps when address mapping changes in
     * system emulation. So it's not safe to make a direct jump to a TB
     * spanning two pages because the mapping for the second page can change.
     */
    if (tb->page_addr[1] != -1) {
        last_tb = NULL;
    }
#endif
    /* See if we can patch the calling TB. */
    if (last_tb) {// 链接tb块
        tb_add_jump(last_tb, tb_exit, tb);
    }
    return tb;
}
```

### cpu_loop_exec_tb

> accel/tcg/cpu-exec.c:cpu_loop_exec_tb

```c
static inline void cpu_loop_exec_tb(CPUState *cpu, TranslationBlock *tb,
                                    TranslationBlock **last_tb, int *tb_exit)
{
    int32_t insns_left;

    trace_exec_tb(tb, tb->pc);//追踪tb起始地址，可以打印
    tb = cpu_tb_exec(cpu, tb, tb_exit);
    if (*tb_exit != TB_EXIT_REQUESTED) {
        *last_tb = tb;
        return;
    }

    *last_tb = NULL;
    insns_left = qatomic_read(&cpu_neg(cpu)->icount_decr.u32);
    if (insns_left < 0) {
        /* Something asked us to stop executing chained TBs; just
         * continue round the main loop. Whatever requested the exit
         * will also have set something else (eg exit_request or
         * interrupt_request) which will be handled by
         * cpu_handle_interrupt.  cpu_handle_interrupt will also
         * clear cpu->icount_decr.u16.high.
         */
        return;
    }

    /* Instruction counter expired.  */
    assert(icount_enabled());
#ifndef CONFIG_USER_ONLY
    /* Ensure global icount has gone forward */
    icount_update(cpu);
    /* Refill decrementer and continue execution.  */
    insns_left = MIN(CF_COUNT_MASK, cpu->icount_budget);
    cpu_neg(cpu)->icount_decr.u16.low = insns_left;
    cpu->icount_extra = cpu->icount_budget - insns_left;

    /*
     * If the next tb has more instructions than we have left to
     * execute we need to ensure we find/generate a TB with exactly
     * insns_left instructions in it.
     */
    if (!cpu->icount_extra && insns_left > 0 && insns_left < tb->icount)  {
        cpu->cflags_next_tb = (tb->cflags & ~CF
```

### cpu_tb_exec

> accel/tcg/cpu-exec.c:cpu_tb_exec

```c
static inline TranslationBlock * QEMU_DISABLE_CFI
cpu_tb_exec(CPUState *cpu, TranslationBlock *itb, int *tb_exit)
{
    CPUArchState *env = cpu->env_ptr;
    uintptr_t ret;
    TranslationBlock *last_tb;
    const void *tb_ptr = itb->tc.ptr;

    qemu_log_mask_and_addr(CPU_LOG_EXEC, itb->pc,
                           "Trace %d: %p ["
                           TARGET_FMT_lx "/" TARGET_FMT_lx "/%#x] %s\n",
                           cpu->cpu_index, itb->tc.ptr,
                           itb->cs_base, itb->pc, itb->flags,
                           lookup_symbol(itb->pc));

#if defined(DEBUG_DISAS)
    if (qemu_loglevel_mask(CPU_LOG_TB_CPU)
        && qemu_log_in_addr_range(itb->pc)) {
        FILE *logfile = qemu_log_lock();
        int flags = 0;
        if (qemu_loglevel_mask(CPU_LOG_TB_FPU)) {
            flags |= CPU_DUMP_FPU;
        }
#if defined(TARGET_I386)
        flags |= CPU_DUMP_CCOP;
#endif
        log_cpu_state(cpu, flags);
        qemu_log_unlock(logfile);
    }
#endif /* DEBUG_DISAS */

    qemu_thread_jit_execute();
    //tcg_qemu_tb_exec_num++; 
    //printf("tcg_qemu_tb_exec=%d\n",tcg_qemu_tb_exec_num); 打印即将要执行的tb块号
    ret = tcg_qemu_tb_exec(env, tb_ptr);
    cpu->can_do_io = 1;
    /*
     * TODO: Delay swapping back to the read-write region of the TB
     * until we actually need to modify the TB.  The read-only copy,
     * coming from the rx region, shares the same host TLB entry as
     * the code that executed the exit_tb opcode that arrived here.
     * If we insist on touching both the RX and the RW pages, we
     * double the host TLB pressure.
     */
    last_tb = tcg_splitwx_to_rw((void *)(ret & ~TB_EXIT_MASK));
    *tb_exit = ret & TB_EXIT_MASK;

    trace_exec_tb_exit(last_tb, *tb_exit);

    if (*tb_exit > TB_EXIT_IDX1) {
        /* We didn't start executing this TB (eg because the instruction
         * counter hit zero); we must restore the guest PC to the address
         * of the start of the TB.
         */
        CPUClass *cc = CPU_GET_CLASS(cpu);
        qemu_log_mask_and_addr(CPU_LOG_EXEC, last_tb->pc,
                               "Stopped execution of TB chain before %p ["
                               TARGET_FMT_lx "] %s\n",
                               last_tb->tc.ptr, last_tb->pc,
                               lookup_symbol(last_tb->pc));
        if (cc->tcg_ops->synchronize_from_tb) {
            cc->tcg_ops->synchronize_from_tb(cpu, last_tb);
        } else {
            assert(cc->set_pc);
            cc->set_pc(cpu, last_tb->pc);
        }
    }
    return last_tb;
}
```

## 函数调用关系

main->cpu_loop->cpu_exec

cpu_exec->①tb_find②cpu_loop_exec_tb

tb_find->①tb_lookup②tb_gen_code

cpu_loop_exec_tb->cpu_tb_exec

如果tb_lookup找的到tb，直接执行cpu_loop_exec_tb执行tb

如果tb_lookup找不到tb，则调用tb_gen_code生成tb

tb_lookup()查找可执行的TranslationBlock（翻译好的host code机器代码），如果找到，执行用cpu_loop_exec_tb()执行TB；如果没有找到开始生成TB，则调用tb_gen_code()生成一个可执行TB。

tcg_tb_alloc()会在TCGContext中分配空间,并执行gen_intermediate_code()翻译guest code成中间代码放在TCGContext中，tcg_gen_code()会将TCGContext中的中间代码翻译成host code。

> 发现生成tb后执行时tb.pc=0x120000510和log上第一个一致。生成日志记录：qemu-sw64 -d in_asm,op,out_asm -cpu core3 hello > log 2>&1
> 
> OP:前两行为gen_tb_start内容，后两行为gen_tb_end内容

## 全局变量tcg_ctx，初始化

用户级tcg_ctx = &tcg_init_ctx;

系统级有多个线程，不同的处理方式

全局的TCGContext初始化
翻译代码时目前是单线程的，共享同一份的上下文，这样可以确保 同一份target 代码只会生成一份对应的 code cache， 在 tcg_context_init(&tcg_init_ctx); 初始化过程中，初始化 IR指令和helper指令和 TCGContext 的一些内部成员，为后续的指令翻译做准备。

> tcg/tcg.c:tcg_init_ctx

```c
/* code generation context */
TCGContext tcg_init_ctx;
__thread TCGContext *tcg_ctx;
TBContext tb_ctx;
/include/exec/gen-icount.h
void tcg_register_thread(void)//用户模式初始化tcg_ctx，经过层级调用，最后源头函数发现是type_init貌似是用于确定cpu类型的初始化的，也就是说
{                             //时初始化之后code cache就确定了
    tcg_ctx = &tcg_init_ctx;
}
```

用户模式下是单线程

## CPU,CPUState,CPUSW64State

要定义所有CPU的基类，需要定义CPU的类的数据结构和CPU的对象的数据结构，然后给对应的TypeInfo中的函数指针赋值即可。其中CPU类的数据结构名为CPUClass、CPU对象的数据结构名为CPUState，它们被定义在include/qom/cpu.h中，而对应的TypeInfo的赋值工作则在qom/cpu.c中进行。这里只说明CPUClass、CPUState数据结构。

> target/sw64/cpu.h:SW64CPU

```c
struct SW64CPU {
    /*< private >*/
    CPUState parent_obj;
    /*< public >*/
    CPUNegativeOffsetState neg;
    CPUSW64State env;

    uint64_t k_regs[158];
    uint64_t k_vcb[36];
    QEMUTimer *alarm_timer;
    target_ulong irq;
    uint32_t cid;
};
```

CPUState是CPU对象的数据结构，一个CPUState就表示一个虚拟机的CPU（一个cpu核心或者线程）。在QEMU中，任何CPU的操作的大部分都是对以CPUState形式出现的CPU来进行的。CPUSW64State，*env_ptr？这个数据结构保存该CPU的所有寄存器的状态，也包括段寄存器、通用寄存器、标志寄存器等，也包括FPU等浮点寄存器，以及与KVM状态相关的信息。sw64对应的数据结构是CPUSW64State，可在target/sw64/cpu.h中看到。

> target/sw64/cpu.h:CPUSW64State,target机器寄存器信息

```c
typedef CPUSW64State CPUArchState;
typedef SW64CPU ArchCPU;

struct CPUSW64State {//对应tcg_init_ctx.temps[]
    uint64_t ir[32];//SW整数寄存器R0~R31,一共32个，即/usr/include/sw_64/regdef.h中定义
    uint64_t fr[128];//SW浮点寄存器F0~F31，一共32个
    uint64_t pc;//程序计数器
    bool is_slave;

    uint64_t csr[0x100];//控制与状态寄存器
    uint64_t fpcr;//浮点控制寄存器
    uint64_t fpcr_exc_enable;
    uint8_t fpcr_round_mode;
    uint8_t fpcr_flush_to_zero;

    float_status fp_status;

    uint64_t hm_entry;

#if !defined(CONFIG_USER_ONLY)
    uint64_t sr[10]; /* shadow regs 1,2,4-7,20-23 */
#endif

    uint32_t flags;
    uint64_t error_code;
    uint64_t unique;
    uint64_t lock_addr;//
    uint64_t lock_valid;//
    uint64_t lock_flag;//
    uint64_t lock_success;//
#ifdef SW64_FIXLOCK
    uint64_t lock_value;//
#endif

    uint64_t trap_arg0;
    uint64_t trap_arg1;
    uint64_t trap_arg2;

    uint64_t features;
    uint64_t insn_count[537];

    /* reserve for slave */
    uint64_t ca[4];
    uint64_t scala_gpr[64];
    uint64_t vec_gpr[224];
    uint64_t fpcr_base;
    uint64_t fpcr_ext;
    uint64_t pendding_flag;
    uint64_t pendding_status;
    uint64_t synr_pendding_status;
    uint64_t sync_pendding_status;
    uint8_t vlenma_idxa;
    uint8_t stable;
};
```

> include/hw/core/cpu.h:CPUState

```c
struct CPUState {
    /*< private >*/
    DeviceState parent_obj;//父类是device
    /*< public >*/

    int nr_cores;
    int nr_threads;

    struct QemuThread *thread;
#ifdef _WIN32
    HANDLE hThread;
#endif
    int thread_id;
    bool running, has_waiter;
    struct QemuCond *halt_cond;
    bool thread_kicked;
    bool created;
    bool stop;
    bool stopped;


    void *env_ptr; /* CPUSW64State */ //在main中赋值给cpu

    /* Accessed in parallel; all accesses must be atomic */
    TranslationBlock *tb_jmp_cache[TB_JMP_CACHE_SIZE];

    void *opaque;//TaskState

    QTAILQ_ENTRY(CPUState) node;
};//thread_cpu
```

## code_gen_buffer

> include/tcg/tcg.h:TCGContext

```c
struct TCGContext {
    /* Code generation.  Note that we specifically do not use tcg_insn_unit
       here, because there's too much arithmetic throughout that relies
       on addition and subtraction working on bytes.  Rely on the GCC
       extension that allows arithmetic on void*.  */
    //翻译缓存相关，一块连续的内存单元
    void *code_gen_buffer;//翻译缓存的首地址，翻译后保存host代码起始位置？
    size_t code_gen_buffer_size;//缓存大小
    void *code_gen_ptr;//指示当前未使用的缓存地址,当前翻译host代码保存位置？
    void *data_gen_ptr;

}
```

> accel/tcg/translate-all.c:code_gen_buffer如何分配空间

```c
#if TCG_TARGET_REG_BITS == 32  
#define DEFAULT_CODE_GEN_BUFFER_SIZE_1 (32 * MiB)
#ifdef CONFIG_USER_ONLY
/*
 * For user mode on smaller 32 bit systems we may run into trouble
 * allocating big chunks of data in the right place. On these systems
 * we utilise a static code generation buffer directly in the binary.
 */
#define USE_STATIC_CODE_GEN_BUFFER
#endif
#else /* TCG_TARGET_REG_BITS == 64 */    //sw64，所以是64位
#ifdef CONFIG_USER_ONLY                  //是usermode，有定义
/*
 * As user-mode emulation typically means running multiple instances
 * of the translator don't go too nuts with our default code gen
 * buffer lest we make things too hard for the OS.
 */
#define DEFAULT_CODE_GEN_BUFFER_SIZE_1 (128 * MiB)//控制buffer大小
#else
/*
 * We expect most system emulation to run one or two guests per host.
 * Users running large scale system emulation may want to tweak their
 * runtime setup via the tb-size control on the command line.
 */
#define DEFAULT_CODE_GEN_BUFFER_SIZE_1 (1 * GiB)
#endif


#ifdef USE_STATIC_CODE_GEN_BUFFER    //静态分配，宏没有定义，不会执行下去，
static uint8_t static_code_gen_buffer[DEFAULT_CODE_GEN_BUFFER_SIZE]
    __attribute__((aligned(CODE_GEN_ALIGN)));


#else //采用的是动态分配
static bool alloc_code_gen_buffer_anon(size_t size, int prot,
                                       int flags, Error **errp)
{
    void *buf;

    buf = mmap(NULL, size, prot, flags, -1, 0);
    if (buf == MAP_FAILED) {
        error_setg_errno(errp, errno,
                         "allocate %zu bytes for jit buffer", size);
        return false;
    }
    tcg_ctx->code_gen_buffer_size = size;



    /* Request large pages for the buffer.  */
    qemu_madvise(buf, size, QEMU_MADV_HUGEPAGE);

    tcg_ctx->code_gen_buffer = buf;
    return true;
}
```

这片内存可以采用静态分配方式，也可以采用动态分配方式，前者将code_gen_buffer指向静态分配的空间，后者将code_gen_buffer指向动态分配的空间。编译时由宏USE_STATIC_CODE_GEN_BUFFER控制选用那种方式。

## TranslationBlock

> include/exec/exec-all.h:TranslationBlock
> 
> 翻译块tb用于存储翻译好的代码以及地址。code cache用于存储很多个tb，查找tb用二叉查找树，相关变量存储在tb_tc中

```c
struct TranslationBlock {
    target_ulong pc;   /* simulated PC corresponding to this block (EIP + CS base) */
    target_ulong cs_base; /* CS base for this block */
    uint32_t flags; /* flags defining in which context the code was generated */
    uint32_t cflags;    /* compile flags */
#define CF_COUNT_MASK  0x00007fff
#define CF_LAST_IO     0x00008000 /* Last insn may be an IO access.  */
#define CF_MEMI_ONLY   0x00010000 /* Only instrument memory ops */
#define CF_USE_ICOUNT  0x00020000
#define CF_INVALID     0x00040000 /* TB is stale. Set with @jmp_lock held */
#define CF_PARALLEL    0x00080000 /* Generate code for a parallel context */
#define CF_CLUSTER_MASK 0xff000000 /* Top 8 bits are cluster ID */
#define CF_CLUSTER_SHIFT 24

    /* Per-vCPU dynamic tracing state used to generate this TB */
    uint32_t trace_vcpu_dstate;

    /*
     * Above fields used for comparing
     */

    /* size of target code for this block (1 <= size <= TARGET_PAGE_SIZE) */
    uint16_t size;
    uint16_t icount;

    struct tb_tc tc;

    /* first and second physical page containing code. The lower bit
       of the pointer tells the index in page_next[].
       The list is protected by the TB's page('s) lock(s) */
    uintptr_t page_next[2];
    tb_page_addr_t page_addr[2];

    /* jmp_lock placed here to fill a 4-byte hole. Its documentation is below */
    QemuSpin jmp_lock;

    /* The following data are used to directly call another TB from
     * the code of this one. This can be done either by emitting direct or
     * indirect native jump instructions. These jumps are reset so that the TB
     * just continues its execution. The TB can be linked to another one by
     * setting one of the jump targets (or patching the jump instruction). Only
     * two of such jumps are supported.
     */
    uint16_t jmp_reset_offset[2]; /* offset of original jump target */
#define TB_JMP_RESET_OFFSET_INVALID 0xffff /* indicates no jump generated */
    uintptr_t jmp_target_arg[2];  /* target address or offset */

    /*
     * Each TB has a NULL-terminated list (jmp_list_head) of incoming jumps.
     * Each TB can have two outgoing jumps, and therefore can participate
     * in two lists. The list entries are kept in jmp_list_next[2]. The least
     * significant bit (LSB) of the pointers in these lists is used to encode
     * which of the two list entries is to be used in the pointed TB.
     *
     * List traversals are protected by jmp_lock. The destination TB of each
     * outgoing jump is kept in jmp_dest[] so that the appropriate jmp_lock
     * can be acquired from any origin TB.
     *
     * jmp_dest[] are tagged pointers as well. The LSB is set when the TB is
     * being invalidated, so that no further outgoing jumps from it can be set.
     *
     * jmp_lock also protects the CF_INVALID cflag; a jump must not be chained
     * to a destination TB that has CF_INVALID set.
     */
    uintptr_t jmp_list_head;//tb之间jump相关
    uintptr_t jmp_list_next[2];
    uintptr_t jmp_dest[2];
};
```

### tb_tc

> include/exec/exec-all.h:tb_tc
> 
> tc是中用于存储翻译缓存相关的变量，指针ptr指向翻译好的代码，翻译块对应存放在翻译缓存的代码
> 
> 二叉查找树，查找数据时ptr+size

```c
/*
 * Translation Cache-related fields of a TB.
 * This struct exists just for convenience; we keep track of TB's in a binary
 * search tree, and the only fields needed to compare TB's in the tree are
 * @ptr and @size.
 * Note: the address of search data can be obtained by adding @size to @ptr.
 */
struct tb_tc {
    const void *ptr;    /* pointer to the translated code */  //tb块翻译后保存的位置
    size_t size;//tb块大小
};
```

# 3.SW前端

## DisasContext ，DisasContextBase

> target/sw64/translate.h:DisasContext

```c
typedef struct DisasContext DisasContext;
struct DisasContext {
    DisasContextBase base;//上下文头，是架构通用的。

    uint32_t tbflags;

    /* The set of registers active in the current context.  */
    TCGv *ir;//当前活跃的寄存器集合，虚拟内存单元模拟寄存器

    /* Accel: Temporaries for $31 and $f31 as source and destination.  */
    TCGv zero;
    int mem_idx;
    CPUSW64State *env;//CPUARCHState状态信息
    DisasJumpType (*translate_one)(DisasContextBase *dcbase, uint32_t insn,
            CPUState *cpu);
};
```

```c
typedef struct DisasContextBase {
    const TranslationBlock *tb;
    target_ulong pc_first;
    target_ulong pc_next;
    DisasJumpType is_jmp;
    int num_insns;
    int max_insns;
    bool singlestep_enabled;
} DisasContextBase;
```

## tb_gen_code

> accel/tcg/cpu-exec.c

```c
TranslationBlock *tb_gen_code(CPUState *cpu,
                              target_ulong pc, target_ulong cs_base,
                              uint32_t flags, int cflags)
{
    CPUArchState *env = cpu->env_ptr;//env == env_ptr
    TranslationBlock *tb, *existing_tb;
    tb_page_addr_t phys_pc, phys_page2;
    target_ulong virt_page2;
    tcg_insn_unit *gen_code_buf;
    int gen_code_size, search_size, max_insns;
#ifdef CONFIG_PROFILER
    TCGProfile *prof = &tcg_ctx->prof;
    int64_t ti;
#endif

    assert_memory_lock();
    qemu_thread_jit_write();

    phys_pc = get_page_addr_code(env, pc);

    if (phys_pc == -1) {
        /* Generate a one-shot TB with 1 insn in it */
        cflags = (cflags & ~CF_COUNT_MASK) | CF_LAST_IO | 1;
    }

    max_insns = cflags & CF_COUNT_MASK;
    if (max_insns == 0) {
        max_insns = CF_COUNT_MASK;
    }
    if (max_insns > TCG_MAX_INSNS) {
        max_insns = TCG_MAX_INSNS;
    }
    if (cpu->singlestep_enabled || singlestep) {
        max_insns = 1;
    }

 buffer_overflow:
    tb = tcg_tb_alloc(tcg_ctx);//分配tb空间
    if (unlikely(!tb)) {
        /* flush must be done */
        tb_flush(cpu);
        mmap_unlock();
        /* Make the execution loop process the flush as soon as possible.  */
        cpu->exception_index = EXCP_INTERRUPT;
        cpu_loop_exit(cpu);
    }

    gen_code_buf = tcg_ctx->code_gen_ptr;
    tb->tc.ptr = tcg_splitwx_to_rx(gen_code_buf);
    tb->pc = pc;
    tb->cs_base = cs_base;
    tb->flags = flags;
    tb->cflags = cflags;
    tb->trace_vcpu_dstate = *cpu->trace_dstate;
    tcg_ctx->tb_cflags = cflags;
 tb_overflow:

#ifdef CONFIG_PROFILER
    /* includes aborted translations because of exceptions */
    qatomic_set(&prof->tb_count1, prof->tb_count1 + 1);
    ti = profile_getclock();
#endif

    gen_code_size = sigsetjmp(tcg_ctx->jmp_trans, 0);
    if (unlikely(gen_code_size != 0)) {
        goto error_return;
    }

    tcg_func_start(tcg_ctx);

    tcg_ctx->cpu = env_cpu(env);
    gen_intermediate_code(cpu, tb, max_insns);//翻译主函数
    tcg_ctx->cpu = NULL;
    max_insns = tb->icount;

    trace_translate_block(tb, tb->pc, tb->tc.ptr);

    /* generate machine code */
    tb->jmp_reset_offset[0] = TB_JMP_RESET_OFFSET_INVALID;
    tb->jmp_reset_offset[1] = TB_JMP_RESET_OFFSET_INVALID;
    tcg_ctx->tb_jmp_reset_offset = tb->jmp_reset_offset;
    if (TCG_TARGET_HAS_direct_jump) {
        tcg_ctx->tb_jmp_insn_offset = tb->jmp_target_arg;
        tcg_ctx->tb_jmp_target_addr = NULL;
    } else {
        tcg_ctx->tb_jmp_insn_offset = NULL;
        tcg_ctx->tb_jmp_target_addr = tb->jmp_target_arg;
    }

#ifdef CONFIG_PROFILER
    qatomic_set(&prof->tb_count, prof->tb_count + 1);
    qatomic_set(&prof->interm_time,
                prof->interm_time + profile_getclock() - ti);
    ti = profile_getclock();
#endif

    gen_code_size = tcg_gen_code(tcg_ctx, tb);//http://blog.chinaunix.net/uid-22954220-id-4978345.html
    if (unlikely(gen_code_size < 0)) {
 error_return:
        switch (gen_code_size) {
        case -1:
            /*
             * Overflow of code_gen_buffer, or the current slice of it.
             *
             * TODO: We don't need to re-do gen_intermediate_code, nor
             * should we re-do the tcg optimization currently hidden
             * inside tcg_gen_code.  All that should be required is to
             * flush the TBs, allocate a new TB, re-initialize it per
             * above, and re-do the actual code generation.
             */
            qemu_log_mask(CPU_LOG_TB_OP | CPU_LOG_TB_OP_OPT,
                          "Restarting code generation for "
                          "code_gen_buffer overflow\n");
            goto buffer_overflow;

        case -2:
            /*
             * The code generated for the TranslationBlock is too large.
             * The maximum size allowed by the unwind info is 64k.
             * There may be stricter constraints from relocations
             * in the tcg backend.
             *
             * Try again with half as many insns as we attempted this time.
             * If a single insn overflows, there's a bug somewhere...
             */
            assert(max_insns > 1);
            max_insns /= 2;
            qemu_log_mask(CPU_LOG_TB_OP | CPU_LOG_TB_OP_OPT,
                          "Restarting code generation with "
                          "smaller translation block (max %d insns)\n",
                          max_insns);
            goto tb_overflow;

        default:
            g_assert_not_reached();
        }
    }
    search_size = encode_search(tb, (void *)gen_code_buf + gen_code_size);
    if (unlikely(search_size < 0)) {
        goto buffer_overflow;
    }
    tb->tc.size = gen_code_size;

#ifdef CONFIG_PROFILER
    qatomic_set(&prof->code_time, prof->code_time + profile_getclock() - ti);
    qatomic_set(&prof->code_in_len, prof->code_in_len + tb->size);
    qatomic_set(&prof->code_out_len, prof->code_out_len + gen_code_size);
    qatomic_set(&prof->search_out_len, prof->search_out_len + search_size);
#endif

#ifdef DEBUG_DISAS
    if (qemu_loglevel_mask(CPU_LOG_TB_OUT_ASM) &&
        qemu_log_in_addr_range(tb->pc)) {
        FILE *logfile = qemu_log_lock();
        int code_size, data_size;
        const tcg_target_ulong *rx_data_gen_ptr;
        size_t chunk_start;
        int insn = 0;

        if (tcg_ctx->data_gen_ptr) {
            rx_data_gen_ptr = tcg_splitwx_to_rx(tcg_ctx->data_gen_ptr);
            code_size = (const void *)rx_data_gen_ptr - tb->tc.ptr;
            data_size = gen_code_size - code_size;
        } else {
            rx_data_gen_ptr = 0;
            code_size = gen_code_size;
            data_size = 0;
        }

        /* Dump header and the first instruction */
        qemu_log("OUT: [size=%d]\n", gen_code_size);//-d out
        qemu_log("  -- guest addr 0x" TARGET_FMT_lx " + tb prologue\n",
                 tcg_ctx->gen_insn_data[insn][0]);
        chunk_start = tcg_ctx->gen_insn_end_off[insn];
        log_disas(tb->tc.ptr, chunk_start);

        /*
         * Dump each instruction chunk, wrapping up empty chunks into
         * the next instruction. The whole array is offset so the
         * first entry is the beginning of the 2nd instruction.
         */
        while (insn < tb->icount) {
            size_t chunk_end = tcg_ctx->gen_insn_end_off[insn];
            if (chunk_end > chunk_start) {
                qemu_log("  -- guest addr 0x" TARGET_FMT_lx "\n",
                         tcg_ctx->gen_insn_data[insn][0]);
                log_disas(tb->tc.ptr + chunk_start, chunk_end - chunk_start);
                chunk_start = chunk_end;
            }
            insn++;
        }

        if (chunk_start < code_size) {
            qemu_log("  -- tb slow paths + alignment\n");
            log_disas(tb->tc.ptr + chunk_start, code_size - chunk_start);
        }

        /* Finally dump any data we may have after the block */
        if (data_size) {
            int i;
            qemu_log("  data: [size=%d]\n", data_size);
            for (i = 0; i < data_size / sizeof(tcg_target_ulong); i++) {
                qemu_log("0x%08" PRIxPTR ":  .quad  0x%" TCG_PRIlx "\n",
                         (uintptr_t)&rx_data_gen_ptr[i], rx_data_gen_ptr[i]);
            }
        }
        qemu_log("\n");
        qemu_log_flush();
        qemu_log_unlock(logfile);
    }
#endif

    qatomic_set(&tcg_ctx->code_gen_ptr, (void *)
        ROUND_UP((uintptr_t)gen_code_buf + gen_code_size + search_size,
                 CODE_GEN_ALIGN));

    /* init jump list */
    qemu_spin_init(&tb->jmp_lock);
    tb->jmp_list_head = (uintptr_t)NULL;
    tb->jmp_list_next[0] = (uintptr_t)NULL;
    tb->jmp_list_next[1] = (uintptr_t)NULL;
    tb->jmp_dest[0] = (uintptr_t)NULL;
    tb->jmp_dest[1] = (uintptr_t)NULL;

    /* init original jump addresses which have been set during tcg_gen_code() */
    if (tb->jmp_reset_offset[0] != TB_JMP_RESET_OFFSET_INVALID) {
        tb_reset_jump(tb, 0);
    }
    if (tb->jmp_reset_offset[1] != TB_JMP_RESET_OFFSET_INVALID) {
        tb_reset_jump(tb, 1);
    }

    /*
     * If the TB is not associated with a physical RAM page then
     * it must be a temporary one-insn TB, and we have nothing to do
     * except fill in the page_addr[] fields. Return early before
     * attempting to link to other TBs or add to the lookup table.
     */
    if (phys_pc == -1) {
        tb->page_addr[0] = tb->page_addr[1] = -1;
        return tb;
    }

    /* check next page if needed */
    virt_page2 = (pc + tb->size - 1) & TARGET_PAGE_MASK;
    phys_page2 = -1;
    if ((pc & TARGET_PAGE_MASK) != virt_page2) {
        phys_page2 = get_page_addr_code(env, virt_page2);
    }
    /*
     * No explicit memory barrier is required -- tb_link_page() makes the
     * TB visible in a consistent state.
     */
    existing_tb = tb_link_page(tb, phys_pc, phys_page2);
    /* if the TB already exists, discard what we just translated */
    if (unlikely(existing_tb != tb)) {
        uintptr_t orig_aligned = (uintptr_t)gen_code_buf;

        orig_aligned -= ROUND_UP(sizeof(*tb), qemu_icache_linesize);
        qatomic_set(&tcg_ctx->code_gen_ptr, (void *)orig_aligned);
        tb_destroy(tb);
        return existing_tb;
    }
    tcg_tb_insert(tb);
    return tb;
}
```

## gen_intermediate_code()

声明DisasContext结构体变量dc，调用transloop_loop()把guest code翻译为中间代码。

执行函数

> target/sw64/translate.c

```c
void gen_intermediate_code(CPUState* cpu, TranslationBlock* tb, int max_insns)
{
    DisasContext dc;
    init_transops(cpu, &dc);
    translator_loop(&sw64_trans_ops, &dc.base, cpu, tb, max_insns);
}
```

程序计数器pc =代码段寄存器cs+指令指针寄存器EIP（段地址：偏移地址）

## TranslatorOps

> target/sw64/translate.c

```c
static const TranslatorOps sw64_trans_ops = {
    .init_disas_context = sw64_tr_init_disas_context,
    .tb_start = sw64_tr_tb_start,
    .insn_start = sw64_tr_insn_start,
    .translate_insn = sw64_tr_translate_insn,
    .tb_stop = sw64_tr_tb_stop,
    .disas_log = sw64_tr_disas_log,
};
```

## translator_loop

void translator_loop(const TranslatorOps *ops, DisasContextBase *db,CPUState *cpu, TranslationBlock *tb, int max_insns)

ops：架构特定的翻译操作集合，sw64中

db：反汇编上下文，需要初始化

cpu：目标cpu信息，执行模块中已经获取，已知

tb：翻译块，需要生成的

max_insns：最大翻译指令数量

translator_loop

架构通用，依赖于特定的目标架构的翻译操作ops集合（此处为sw64_trans_ops），由 ./include/exec/translator.h中声明结构体TranslatorOps，声明结构体成员函数时，使用函数指针。在/targe/sw64/translate.c中声明每个架构自己的ops结构体变量（将自己定义的架构相关的函数地址赋值给成员的函数指针），translator_loop通过这些TranslatorOps ops结构体变量调用guest平台的对应操作。

(1)、Initialize DisasContext

初始化通用的DisasContextBase变量db，再通过调用ops->init_disas_context()初始化目标特定的DisasContext 变量dc，即调用container_of(dcbase, DisasContext, base)将dc起始地址赋值给Dis  asContext* ctx，并对*ctx成员进行初始化的过程（tbflags）。

static void sw64 _tr_init_disas_context(DisasContextBase*dcbase, CPUState *cpu)

宏函数container_of通过db的地址计算dc的起始地址，赋值给局部变量DisasContext* ctx，并对*ctx成员进行初始化。

(2)  、Start translating

DisasContext*tcg_ctx 反汇编上下文，全局变量

发现没有定义TARGET_INSN_START_WORDS，x86有定义

> accel/tcg/translator.c

```c
void translator_loop(const TranslatorOps *ops, DisasContextBase *db,
                     CPUState *cpu, TranslationBlock *tb, int max_insns)
{
    int bp_insn = 0;
    bool plugin_enabled;//插件

    /* Initialize DisasContext */
    db->tb = tb;
    db->pc_first = tb->pc;//tb起始地址，即第一条指令
    db->pc_next = db->pc_first;//下一条指令
    db->is_jmp = DISAS_NEXT;
    db->num_insns = 0;//已翻译的指令
    db->max_insns = max_insns;//需要翻译的指令
    db->singlestep_enabled = cpu->singlestep_enabled;

    ops->init_disas_context(db, cpu);
    tcg_debug_assert(db->is_jmp == DISAS_NEXT);  /* no early exit */

    /* Reset the temp count so that we can identify leaks */
    tcg_clear_temp_count();

    /* Start translating.  */
    gen_tb_start(db->tb);//注入指令用以检查指令计数和退出条件，创建标签exitreq_label，供gen_tb_end()使用
    ops->tb_start(db, cpu);//sw什么也没有，alpha也没有。arm有，i386没有
    tcg_debug_assert(db->is_jmp == DISAS_NEXT);  /* no early exit */
//宏函数，展开do { if (!(db->is_jmp == DISAS_NEXT)) { __builtin_unreachable(); } } while(0)
    plugin_enabled = plugin_gen_tb_start(cpu, tb,
                                         tb_cflags(db->tb) & CF_MEMI_ONLY);

    while (true) {
        db->num_insns++;//翻译指令先提前加1
        ops->insn_start(db, cpu);//INDEX_op_insn_start微操作
        tcg_debug_assert(db->is_jmp == DISAS_NEXT);  /* no early exit */

        if (plugin_enabled) {
            plugin_gen_insn_start(cpu, db);
        }

        /* Pass breakpoint hits to target for further processing */
        if (!db->singlestep_enabled
            && unlikely(!QTAILQ_EMPTY(&cpu->breakpoints))) {
            CPUBreakpoint *bp;
            QTAILQ_FOREACH(bp, &cpu->breakpoints, entry) {
                if (bp->pc == db->pc_next) {
                    if (ops->breakpoint_check(db, cpu, bp)) {
                        bp_insn = 1;
                        break;
                    }
                }
            }
            /* The breakpoint_check hook may use DISAS_TOO_MANY to indicate
               that only one more instruction is to be executed.  Otherwise
               it should use DISAS_NORETURN when generating an exception,
               but may use a DISAS_TARGET_* value for Something Else.  */
            if (db->is_jmp > DISAS_TOO_MANY) {
                break;
            }
        }

        /* Disassemble one instruction.  The translate_insn hook should
           update db->pc_next and db->is_jmp to indicate what should be
           done next -- either exiting this loop or locate the start of
           the next instruction.  */
        if (db->num_insns == db->max_insns
            && (tb_cflags(db->tb) & CF_LAST_IO)) {
            /* Accept I/O on the last instruction.  */
            gen_io_start();//翻译到最后一条指令时开始IO
            ops->translate_insn(db, cpu);
        } else {
            /* we should only see CF_MEMI_ONLY for io_recompile */
            tcg_debug_assert(!(tb_cflags(db->tb) & CF_MEMI_ONLY));
            ops->translate_insn(db, cpu);//反汇编一条指令
        }

        /* Stop translation if translate_insn so indicated.  */
        if (db->is_jmp != DISAS_NEXT) {
            break;
        }

        /*
         * We can't instrument after instructions that change control
         * flow although this only really affects post-load operations.
         */
        if (plugin_enabled) {
            plugin_gen_insn_end();
        }

        /* Stop translation if the output buffer is full,
           or we have executed all of the allowed instructions.  */
        if (tcg_op_buf_full() || db->num_insns >= db->max_insns) {
            db->is_jmp = DISAS_TOO_MANY;
            break;
        }
    }

    /* Emit code to exit the TB, as indicated by db->is_jmp.  */
    ops->tb_stop(db, cpu);
    gen_tb_end(db->tb, db->num_insns - bp_insn);

    if (plugin_enabled) {
        plugin_gen_tb_end(cpu);
    }

    /* The disas_log hook may use these values rather than recompute.  */
    tb->size = db->pc_next - db->pc_first;
    tb->icount = db->num_insns;

#ifdef DEBUG_DISAS
    if (qemu_loglevel_mask(CPU_LOG_TB_IN_ASM)
        && qemu_log_in_addr_range(db->pc_first)) {
        FILE *logfile = qemu_log_lock();
        qemu_log("----------------\n");
        ops->disas_log(db, cpu);
        qemu_log("\n");
        qemu_log_unlock(logfile);
    }
#endif
}
```

### sw64_tr_translate_insn

> target/sw64/translate.c

```c
static void sw64_tr_translate_insn(DisasContextBase *dcbase, CPUState *cpu)
{
    DisasContext *ctx = container_of(dcbase, DisasContext, base);
    CPUSW64State *env = cpu->env_ptr;
    uint32_t insn;

    insn = cpu_ldl_code(env, ctx->base.pc_next & (~3UL));//取指令
    ctx->env = env;
    ctx->base.pc_next += 4;
    ctx->base.is_jmp = ctx->translate_one(dcbase, insn, cpu);

    free_context_temps(ctx);
    translator_loop_temp_check(&ctx->base);
}
```

### cpu_ldl_code

> accel/tcg/user-exec.c

```c
uint32_t cpu_ldl_code(CPUArchState *env, abi_ptr ptr)
{
    uint32_t ret;

    set_helper_retaddr(1);
    ret = ldl_p(g2h_untagged(ptr));
    clear_helper_retaddr();
    return ret;
}
```

动态翻译基本思想把一条target指令切分成若干条微操作，每条微操作由一段简单的C代码来实现，运行时通过一个动态代码生成器把这些微操作组合成一个函数，最后执行这个函数。

## translate_one

> target/sw64/translate.c

反汇编

```c
DisasJumpType translate_one(DisasContextBase *dcbase, uint32_t insn,CPUState *cpu)
{
    int32_t disp5, disp8, disp12, disp13, disp16, disp21, disp26 __attribute__((unused));//偏移量disp
    uint8_t opc, ra, rb, rc, rd;//操作码opc,寄存器域rx（寄存器编号）
    uint16_t fn3, fn4, fn6, fn8, fn11;//功能域fn
    int32_t i;
    TCGv va, vb, vc, vd;
    TCGv_i32 tmp32;
    TCGv_i64 tmp64, tmp64_0, tmp64_1, shift;
    TCGv_i32 tmpa, tmpb, tmpc;//
    DisasJumpType ret;//跳转类型ret
    DisasContext* ctx = container_of(dcbase, DisasContext, base);

    opc = extract32(insn, 26, 6);
    ra = extract32(insn, 21, 5);
    ....

    fn3 = extract32(insn, 10, 3);
    ....

    disp5 = extract32(insn, 5, 5);
    ....

    switch (opc) {
    case 0x00:
        /* SYS_CALL */
        ret = gen_sys_call(ctx, insn & 0x1ffffff);
        break;
        ....
    case 0x04:
        /* BR */
    case 0x05:
        /* BSR */
        ret = gen_bdirect(ctx, ra, disp21);
        break;
        ....
    }
}
```

提取函数extract()；从32位的输入指令中提取特定的位域

sextract32：会进行符号扩展

```c
static inline uint32_t extract32(uint32_t value, int start, int length)
{//value:指令 start:起始位数 length:提取长度
    assert(start >= 0 && length > 0 && length <= 32 - start);//判断参数是否合法
    return (value >> start) & (~0U >> (32 - length));
    //取相应位数。value右移start位，0U取反右移32 - length，再按位与           
}
```

声明翻译指令所有可能用到的操作码opcode，偏移量disp，功能域fn，寄存器域rx等。并调用提取函数extract()初始化当前正在翻译的指令的操作码opcode，偏移量disp，功能域fn，寄存器域rx。

声明TCG微操作，跳转类型ret，获取dc首地址赋值给ctx，并在接下来使用ctx代替。

进入switch case结构，根据操作码opcode进入对应的翻译流程。

第一条指令:无条件转移指令

转移指令格式

31            26  25           21  20             0

| Opcode | Ra/Fa | disp |
| ------ | ----- | ---- |

目标指令的有效地址计算方法：Vaddr<-PC+4*SEXT(disp)

指令分析：

            opc             ra              disp21

insn=0001 00 | 11 101 | 0 0000 0000 0000 0000 0000  0x13a00000

opc=100                                                                                   0x04

ra=11101                                                                                  0x1d

disp21=                                                                                     0x0

## gen_bdirect

```c
static DisasJumpType gen_bdirect(DisasContext *ctx, int ra, int32_t disp)
{
    uint64_t dest = ctx->base.pc_next + ((int64_t)disp << 2);
    if (ra != 31) {
        tcg_gen_movi_i64(load_gir(ctx, ra), ctx->base.pc_next & (~0x3UL));
    }
    if (disp == 0) {
        return 0;
    } else if (use_goto_tb(ctx, dest)) {
        tcg_gen_goto_tb(0);
        tcg_gen_movi_i64(cpu_pc, dest);
        tcg_gen_exit_tb(ctx->base.tb, 0);
        return DISAS_NORETURN;
    } else {
        tcg_gen_movi_i64(cpu_pc, dest);
        return DISAS_PC_UPDATED;
    }
}
```

dest:目标指令有效地址

64位

tcg_gen_<op>i_<reg size>()

Qemu调用各种微操作来生成IR，所有的微操作都是把操作码和操作数放到全局变量tcg_ctx中，相当于把指令保存到tcg_ctx。

```c
 void tcg_gen_movi_i64(TCGv_i64 ret, int64_t arg)
{
    tcg_gen_mov_i64(ret, tcg_constant_i64(arg));
}
```

```c
static inline void tcg_gen_mov_i64(TCGv_i64 ret, TCGv_i64 arg)
{
    if (ret != arg) {
        tcg_gen_op2_i64(INDEX_op_mov_i64, ret, arg);
    }
}
include/tcg/tcg-op.h
```

```c
void tcg_gen_op2(TCGOpcode opc, TCGArg a1, TCGArg a2)
{
    TCGOp *op = tcg_emit_op(opc);//生成TCG微操作
    op->args[0] = a1;//操作数放进去
    op->args[1] = a2;
}
tcg/tcg-op.c
```

```c
TCGOp *tcg_emit_op(TCGOpcode opc)
{
    TCGOp *op = tcg_op_alloc(opc);//为微操作节点op分配空间
    QTAILQ_INSERT_TAIL(&tcg_ctx->ops, op, link);//插入操作队列尾
    return op;
}
tcg/tcg-op.c
```

将生成的微操作插入队列尾

```c
static inline void tcg_gen_op2_i64(TCGOpcode opc, TCGv_i64 a1, TCGv_i64 a2)
{
    tcg_gen_op2(opc, tcgv_i64_arg(a1), tcgv_i64_arg(a2));//生成TCG微操作
}
include/tcg/tcg-op.h
```

## 系统调用指令

```c
    switch (opc) {
    case 0x00://系统调用指令
        /* SYS_CALL */
        //[25:0]，第26位不管，insn&1ffffff结果是取指令低25位的功能码，与1与表示保留自己，与0与表示置为0       
        ret = gen_sys_call(ctx, insn & 0x1ffffff);# 
        break;
    }
```

```c
static DisasJumpType gen_sys_call(DisasContext *ctx, int syscode)
{
    if (syscode >= 0x80 && syscode <= 0xbf) {//128~191号
        switch (syscode) {
        case 0x86://__NR_shutdown  134
            /* IMB */
            /* No-op inside QEMU */
            break;
#ifdef CONFIG_USER_ONLY//?
        case 0x9E://__NR_osf_nfssvc 158
            /* RDUNIQUE */
            tcg_gen_ld_i64(ctx->ir[IDX_V0], cpu_env,
                           offsetof(CPUSW64State, unique));
            break;
        case 0x9F://__NR_osf_getdirentries 159
            /* WRUNIQUE */
            tcg_gen_st_i64(ctx->ir[IDX_A0], cpu_env,
                           offsetof(CPUSW64State, unique));
            break;
#endif
        default:
            goto do_sys_call;//执行系统调用
        }
        return DISAS_NEXT;
    }
do_sys_call:
#ifdef CONFIG_USER_ONLY
    return gen_excp(ctx, EXCP_CALL_SYS, syscode);//产生用于系统调用的异常类型EXCP_CALL_SYS
#else
    tcg_gen_movi_i64(cpu_hm_ir[23], ctx->base.pc_next);
    return gen_excp(ctx, EXCP_CALL_SYS, syscode);
#endif
}
```

# 4.TCG

## 参考资料

### TCG中间码参考

> tcg/README

### SW整数寄存器及别名

> /usr/include/sw_64/regdef.h

### TCG寄存器

```c
typedef enum {
    TCG_REG_X0, TCG_REG_X1, TCG_REG_X2, TCG_REG_X3,
    TCG_REG_X4, TCG_REG_X5, TCG_REG_X6, TCG_REG_X7,
    TCG_REG_X8, TCG_REG_X9, TCG_REG_X10, TCG_REG_X11,
    TCG_REG_X12, TCG_REG_X13, TCG_REG_X14, TCG_REG_X15,
    TCG_REG_X16, TCG_REG_X17, TCG_REG_X18, TCG_REG_X19,
    TCG_REG_X20, TCG_REG_X21, TCG_REG_X22, TCG_REG_X23,
    TCG_REG_X24, TCG_REG_X25, TCG_REG_X26, TCG_REG_X27,
    TCG_REG_X28, TCG_REG_X29, TCG_REG_X30, TCG_REG_X31, 
//32个整型
    TCG_REG_F0=32, TCG_REG_F1, TCG_REG_F2, TCG_REG_F3,
    TCG_REG_F4, TCG_REG_F5, TCG_REG_F6, TCG_REG_F7,
    TCG_REG_F8, TCG_REG_F9, TCG_REG_F10, TCG_REG_F11,
    TCG_REG_F12, TCG_REG_F13, TCG_REG_F14, TCG_REG_F15,
    TCG_REG_F16, TCG_REG_F17, TCG_REG_F18, TCG_REG_F19,
    TCG_REG_F20, TCG_REG_F21, TCG_REG_F22, TCG_REG_F23,
    TCG_REG_F24, TCG_REG_F25, TCG_REG_F26, TCG_REG_F27,
    TCG_REG_F28, TCG_REG_F29, TCG_REG_F30, TCG_REG_F31,
//32个浮点
    /* Aliases.  */
    TCG_REG_FP = TCG_REG_X15,//特殊的
    TCG_REG_RA = TCG_REG_X26,
    TCG_REG_GP = TCG_REG_X29,
    TCG_REG_SP = TCG_REG_X30,
    TCG_REG_ZERO = TCG_REG_X31,
    TCG_AREG0  = TCG_REG_X9,
} TCGReg;
```

### 系统调用

> /usr/include/asm/unistd.h：sw系统调用号表
> 
> linux-user/syscall.c：qemu处理系统相关
> 
> linux-user/host/sw_64/safe-syscall.inc.S qemu翻译到系统调用有时会调用这个函数，执行汇编

## TCGOpcode

> include/tcg/tcg.h

```c
typedef enum TCGOpcode {//枚举所有TCG操作
#define DEF(name, oargs, iargs, cargs, flags) INDEX_op_ ## name,//宏函数定义成名字，把每一个TCG操作对应到操作码
#include "tcg/tcg-opc.h"
#undef DEF
    NB_OPS,?什么玩意，中间码个数
} TCGOpcode;
```

## TCGOp

TCG数据结构，定义操作队列节点

> include/tcg/tcg.h:TCGOp,TCGContext

```c
typedef struct TCGOp {
    TCGOpcode opc   : 8;        /*  8 */

    /* Parameters for this opcode.  See below.  */
    unsigned param1 : 4;        /* 12 */
    unsigned param2 : 4;        /* 16 */

    /* Lifetime data of the operands.  */
    unsigned life   : 16;       /* 32 */

    /* Next and previous opcodes.  */
    QTAILQ_ENTRY(TCGOp) link;
#ifdef CONFIG_PLUGIN
    QSIMPLEQ_ENTRY(TCGOp) plugin_link;
#endif

    /* Arguments for the opcode.  */
    TCGArg args[MAX_OPC_PARAM];

    /* Register preferences for the output(s).  */
    TCGRegSet output_pref[2];
} TCGOp;
```

## TCGContext

```c
struct TCGContext {
    uint8_t *pool_cur, *pool_end;//pool池，存储label和Micro-op
    //pool_cur指向当前可以分配的pool池空间，pool_first和pool_end记录pool池的开始和结束位置
    TCGPool *pool_first, *pool_current, *pool_first_large;
    int nb_labels;//labels个数
    int nb_globals;//全局变量的个数
    int nb_temps;//临时变量的个数
    int nb_indirects;
    int nb_ops;//Mico-op个数    

    /* goto_tb support */
    tcg_insn_unit *code_buf;//TB块翻译代码的开始位置,tb->tb_tc->ptr
    uint16_t *tb_jmp_reset_offset; /* tb->jmp_reset_offset */
    uintptr_t *tb_jmp_insn_offset; /* tb->jmp_target_arg if direct_jump */
    uintptr_t *tb_jmp_target_addr; /* tb->jmp_target_arg if !direct_jump */

    TCGRegSet reserved_regs;//保留的寄存器
    uint32_t tb_cflags; /* cflags of the current TB */ 编译参数
    intptr_t current_frame_offset;
    intptr_t frame_start;
    intptr_t frame_end;
    TCGTemp *frame_temp;

    tcg_insn_unit *code_ptr;//下一条翻译代码的保存位置

    //翻译缓存相关，一块连续的内存单元
    void *code_gen_buffer;//翻译缓冲的首地址
    size_t code_gen_buffer_size;//缓冲大小
    void *code_gen_ptr;//指示当前未使用的缓冲地址，即下一条翻译代码保存的位置
    void *data_gen_ptr;

    GHashTable *const_table[TCG_TYPE_COUNT];
    TCGTempSet free_temps[TCG_TYPE_COUNT * 2];
    TCGTemp temps[TCG_MAX_TEMPS]; /* globals first, temps after */
    ....
    QTAILQ_HEAD(, TCGOp) ops, free_ops;//队列头结点ops,中间码
    QSIMPLEQ_HEAD(, TCGLabel) labels;//中间码label
}
```

## TCGTemp

```c
typedef struct TCGTemp {
    TCGReg reg:8;//tcg中的寄存器编号，
    TCGTempVal val_type:8;
    TCGType base_type:8;
    TCGType type:8;
    TCGTempKind kind:3;
    unsigned int indirect_reg:1;
    unsigned int indirect_base:1;
    unsigned int mem_coherent:1;
    unsigned int mem_allocated:1;
    unsigned int temp_allocated:1;

    int64_t val;
    struct TCGTemp *mem_base;//=tcg_init_ctx.temp[0];基地址
    intptr_t mem_offset;//=offsetof(CPUSW64State,xx);偏移量
    const char *name;//=xx，名称

    /* Pass-specific information that can be stored for a temporary.
       One word worth of integer data, and one pointer to data
       allocated separately.  */
    uintptr_t state;
    void *state_ptr;
} TCGTemp;
```

## TCGOpDef，tcg_op_defs

> include/tcg/tcg.h

```c
typedef struct TCGOpDef {//TCG操作的相关信息
    const char *name;//TCG操作
    uint8_t nb_oargs, nb_iargs, nb_cargs, nb_args;//输出，输入，常量、参数个数
    uint8_t flags;
    TCGArgConstraint *args_ct;//输入输出参数的约束条件，非通用
} TCGOpDef;
typedef struct TCGArgConstraint {
    unsigned ct : 16;
    unsigned alias_index : 4;
    unsigned sort_index : 4;
    bool oalias : 1;
    bool ialias : 1;
    bool newreg : 1;
    TCGRegSet regs;
} TCGArgConstraint;
```

> tcg/tcg-common.c:全局变量tcg_op_defs

```c
TCGOpDef tcg_op_defs[] = {//初始化TCG操作
#define DEF(s, oargs, iargs, cargs, flags) \
         { #s, oargs, iargs, cargs, iargs + oargs + cargs, flags },//宏函数定义成参数列表
#include "tcg/tcg-opc.h"  //include/tcg/tcg-opc.h中有大量DEF(···)宏函数，都可以展开成{#s，···}参数列表形式
#undef DEF                //再用参数列表初始化TCG操作
};
const size_t tcg_op_defs_max = ARRAY_SIZE(tcg_op_defs);//TCG操作个数
```

## TCG初始化

> accel/tcg/tcg-all.c

```c
static int tcg_init(MachineState *ms)
{
    TCGState *s = TCG_STATE(current_accel());

    tcg_exec_init(s->tb_size * 1024 * 1024, s->splitwx_enabled);
    mttcg_enabled = s->mttcg_enabled;

    /*
     * Initialize TCG regions only for softmmu.
     *
     * This needs to be done later for user mode, because the prologue
     * generation needs to be delayed so that GUEST_BASE is already set.
     */
#ifndef CONFIG_USER_ONLY
    tcg_region_init();
#endif /* !CONFIG_USER_ONLY */

    return 0;
}
void tcg_exec_init(unsigned long tb_size, int splitwx)
{
    bool ok;

    tcg_allowed = true;
    cpu_gen_init();//
    page_init();
    tb_htable_init();

    ok = alloc_code_gen_buffer(size_code_gen_buffer(tb_size),
                               splitwx, &error_fatal);//初始化code_gen_buffer
    assert(ok);

#if defined(CONFIG_SOFTMMU)
    /* There's no guest base to take into account, so go ahead and
       initialize the prologue now.  */
    tcg_prologue_init(tcg_ctx);
#endif
}
```

> accel/tcg/translate-all.c

```c
static void cpu_gen_init(void)
{
    tcg_context_init(&tcg_init_ctx);
}
```

### tcg_context_init

> tcg/tcg.c:tcg_context_init初始化TCGContext，

```c
void tcg_context_init(TCGContext *s)
{
    int op, total_args, n, i;
    TCGOpDef *def;
    TCGArgConstraint *args_ct;
    TCGTemp *ts;

    memset(s, 0, sizeof(*s));//memset是c/c++初始化函数，将s清0
    s->nb_globals = 0;

    /* Count total number of arguments and allocate the corresponding
       space */
    total_args = 0;
    for(op = 0; op < NB_OPS; op++) {//计算所有的中间码包含的输入和输出参数个数的和
        def = &tcg_op_defs[op];
        n = def->nb_iargs + def->nb_oargs;
        total_args += n;
    }
    //给每个输入输出参数的约束条件TCGArgConstraint申请空间
    args_ct = g_new0(TCGArgConstraint, total_args);
    //给每个tcg_op_defs[]->args_ct分配指定的空间
    for(op = 0; op < NB_OPS; op++) {
        def = &tcg_op_defs[op];
        def->args_ct = args_ct;
        n = def->nb_iargs + def->nb_oargs;
        args_ct += n;
    }
    //初始化helper函数
    /* Register helpers.  */
    /* Use g_direct_hash/equal for direct pointer comparisons on func.  */
    helper_table = g_hash_table_new(NULL, NULL);

    for (i = 0; i < ARRAY_SIZE(all_helpers); ++i) {
        g_hash_table_insert(helper_table, (gpointer)all_helpers[i].func,
                            (gpointer)&all_helpers[i]);
    }

    tcg_target_init(s);//初始化target寄存器相关信息 tcg/sw_64/tcg-target.c.inc
    process_op_defs(s);//给tcg_op_defs[]->args_ct赋初值

    /* Reverse the order of the saved registers, assuming they're all at
       the start of tcg_target_reg_alloc_order.  */
    for (n = 0; n < ARRAY_SIZE(tcg_target_reg_alloc_order); ++n) {
        int r = tcg_target_reg_alloc_order[n];
        if (tcg_regset_test_reg(tcg_target_call_clobber_regs, r)) {
            break;
        }
    }
    for (i = 0; i < n; ++i) {
        indirect_reg_alloc_order[i] = tcg_target_reg_alloc_order[n - 1 - i];
    }
    for (; i < ARRAY_SIZE(tcg_target_reg_alloc_order); ++i) {
        indirect_reg_alloc_order[i] = tcg_target_reg_alloc_order[i];
    }

    alloc_tcg_plugin_context(s);

    tcg_ctx = s;
    /*
     * In user-mode we simply share the init context among threads, since we
     * use a single region. See the documentation tcg_region_init() for the
     * reasoning behind this.
     * In softmmu we will have at most max_cpus TCG threads.
     */
#ifdef CONFIG_USER_ONLY
    tcg_ctxs = &tcg_ctx;
    n_tcg_ctxs = 1;
#else
    MachineState *ms = MACHINE(qdev_get_machine());
    unsigned int max_cpus = ms->smp.max_cpus;
    tcg_ctxs = g_new(TCGContext *, max_cpus);
#endif

    tcg_debug_assert(!tcg_regset_test_reg(s->reserved_regs, TCG_AREG0));
    ts = tcg_global_reg_new_internal(s, TCG_TYPE_PTR, TCG_AREG0, "env");//保存一个全局变量env
    cpu_env = temp_tcgv_ptr(ts);//初始化cpu_env
}
```

### sw64_translate_init

函数调用关系

```c
cpu = cpu_create(cpu_type)
CPUState *cpu_create(const char *typename)
{
    if (!qdev_realize(DEVICE(cpu), NULL, &err)) {
}

bool qdev_realize(DeviceState *dev, BusState *bus, Error **errp)
{
    return object_property_set_bool(OBJECT(dev), "realized", true, errp);
}

bool object_property_set_bool(Object *obj, const char *name,
                              bool value, Error **errp)
{
    bool ok = object_property_set_qobject(obj, name, QOBJECT(qbool), errp);
}

bool object_property_set_qobject(Object *obj,
                                 const char *name, QObject *value,
                                 Error **errp)
{
    ok = object_property_set(obj, name, v, errp);
}

bool object_property_set(Object *obj, const char *name, Visitor *v,
                         Error **errp)
{
    prop->set(obj, v, name, prop->opaque, &err);
}

static void property_set_bool(Object *obj, Visitor *v, const char *name,
                              void *opaque, Error **errp)
{
    prop->set(obj, value, errp);
}

static void device_set_realized(Object *obj, bool value, Error **errp)
{       
        dc->realize(dev, &local_err);
}

static void sw64_cpu_realizefn(DeviceState *dev, Error **errp)
{
    cpu_exec_realizefn(cs, &local_err);
} 

void cpu_exec_realizefn(CPUState *cpu, Error **errp)
{
        tcg_exec_realizefn(cpu, errp);
}

void tcg_exec_realizefn(CPUState *cpu, Error **errp)
{
        cc->tcg_ops->initialize();
}

void sw64_translate_init(void)
{}
```

> target/sw64/translate.c:初始化TCGContext成员temp

```c
void sw64_translate_init(void)
{
#define DEF_VAR(V) \
    { &cpu_##V, #V, offsetof(CPUSW64State, V) }

    typedef struct {
        TCGv* var;
        const char* name;
        int ofs;
    } GlobalVar;

    static const GlobalVar vars[] = {
        DEF_VAR(pc),         DEF_VAR(lock_addr),
        DEF_VAR(lock_flag),  DEF_VAR(lock_success),
#ifdef SW64_FIXLOCK
        DEF_VAR(lock_value),
#endif
    };
    cpu_pc = tcg_global_mem_new_i64(cpu_env,
                                    offsetof(CPUSW64State, pc), "PC");//pc

#undef DEF_VAR

    /* Use the symbolic register names that match the disassembler.  */
    static const char ireg_names[31][4] = {
        "v0", "t0", "t1",  "t2",  "t3", "t4",  "t5", "t6", "t7", "s0", "s1",
        "s2", "s3", "s4",  "s5",  "fp", "a0",  "a1", "a2", "a3", "a4", "a5",
        "t8", "t9", "t10", "t11", "ra", "t12", "at", "gp", "sp"};

    static const char freg_names[128][4] = {
        "f0",  "f1",  "f2",  "f3",  "f4",  "f5",  "f6",  "f7",  "f8",  "f9",
        "f10", "f11", "f12", "f13", "f14", "f15", "f16", "f17", "f18", "f19",
        "f20", "f21", "f22", "f23", "f24", "f25", "f26", "f27", "f28", "f29",
        "f30", "f31", "f0",  "f1",  "f2",  "f3",  "f4",  "f5",  "f6",  "f7",
        "f8",  "f9",  "f10", "f11", "f12", "f13", "f14", "f15", "f16", "f17",
        "f18", "f19", "f20", "f21", "f22", "f23", "f24", "f25", "f26", "f27",
        "f28", "f29", "f30", "f31", "f0",  "f1",  "f2",  "f3",  "f4",  "f5",
        "f6",  "f7",  "f8",  "f9",  "f10", "f11", "f12", "f13", "f14", "f15",
        "f16", "f17", "f18", "f19", "f20", "f21", "f22", "f23", "f24", "f25",
        "f26", "f27", "f28", "f29", "f30", "f31", "f0",  "f1",  "f2",  "f3",
        "f4",  "f5",  "f6",  "f7",  "f8",  "f9",  "f10", "f11", "f12", "f13",
        "f14", "f15", "f16", "f17", "f18", "f19", "f20", "f21", "f22", "f23",
        "f24", "f25", "f26", "f27", "f28", "f29", "f30", "f31"};

#ifndef CONFIG_USER_ONLY
    static const char shadow_names[10][8] = {
        "hm_p1", "hm_p2",  "hm_p4",  "hm_p5",  "hm_p6",
        "hm_p7", "hm_p20", "hm_p21", "hm_p22", "hm_p23"};
    static const int shadow_index[10] = {1, 2, 4, 5, 6, 7, 20, 21, 22, 23};
#endif

    int i;
    //ir
    for (i = 0; i < 31; i++) {
        cpu_std_ir[i] = tcg_global_mem_new_i64(
            cpu_env, offsetof(CPUSW64State, ir[i]), ireg_names[i]);
    }
    //fr
    for (i = 0; i < 128; i++) {
        cpu_fr[i] = tcg_global_mem_new_i64(
            cpu_env, offsetof(CPUSW64State, fr[i]), freg_names[i]);
    }
    for (i = 0; i < ARRAY_SIZE(vars); ++i) {//pc,lock_addr,lock_flag,lock_success,lock_value
        const GlobalVar* v = &vars[i];
        *v->var = tcg_global_mem_new_i64(cpu_env, v->ofs, v->name);
    }
#ifndef CONFIG_USER_ONLY
    memcpy(cpu_hm_ir, cpu_std_ir, sizeof(cpu_hm_ir));
    for (i = 0; i < 10; i++) {
        int r = shadow_index[i];
        cpu_hm_ir[r] = tcg_global_mem_new_i64(
            cpu_env, offsetof(CPUSW64State, sr[i]), shadow_names[i]);
    }
#endif
}
```

cpu_env 表示tcg_ctx->temps[0]相对于tcg_ctx的offset，tcg_ctx->temps[0].reg=TCG_AREG0,tcg_ctx->temps[0].kind=TEMP_FIXED, tcg_ctx->temps[0].name=env

TCG_AREG0是架构相关的临时寄存器，在host执行tcg_qemu_tb_exec时使用TCG_AREG0 来访问翻译模式下的env。（aarch64: TCG_AREG0 =TCG_REG_X19,sw_64:TCG_AREG0=TCG_REG_X9）

tcg_init_ctx.temps[0]

在执行模式中用来指向env(CPUX86State)

temps[1].mem_base指向temps[0]，

也就是CPUX86State,

temps[1].mem_offset表示cc_op在CPUX86State的偏移

r9的值加上偏移量得到的地址取值就是各个汇编语言中寄存器r几的值

## QTAILQ，QTailQLink

> include/qemu/queue.h：QTAILQ数据结构

```c
//队列头
#define QTAILQ_HEAD(name, type)                                         \
union name {                                                            \
        struct type *tqh_first;       /* first element */               \
        QTailQLink tqh_circ;          /* link for circular backwards list */ \
}
typedef struct QTailQLink {
    void *tql_next;
    struct QTailQLink *tql_prev;
} QTailQLink;
//队列实体
#define QTAILQ_ENTRY(type)                                              \
union {                                                                 \
        struct type *tqe_next;        /* next element */                \
        QTailQLink tqe_circ;          /* link for prev element */       \
}
//初始化队列头
#define QTAILQ_HEAD_INITIALIZER(head)                                   \
        { .tqh_circ = { NULL, &(head).tqh_circ } }

#define QTAILQ_INIT(head) do {                                          \
        (head)->tqh_first = NULL;                                       \
        (head)->tqh_circ.tql_prev = &(head)->tqh_circ;                  \
} while (/*CONSTCOND*/0)
//插入队列尾
#define QTAILQ_INSERT_TAIL(head, elm, field) do {                       \
        (elm)->field.tqe_next = NULL;                                   \
        (elm)->field.tqe_circ.tql_prev = (head)->tqh_circ.tql_prev;     \
        (head)->tqh_circ.tql_prev->tql_next = (elm);                    \
        (head)->tqh_circ.tql_prev = &(elm)->field.tqe_circ;             \
} while (/*CONSTCOND*/0)
QTAILQ_INSERT_TAIL(&tcg_ctx->ops, op, link);
/*tcg_ctx是TCGContext类型指针，结构体定义了一个队列头ops。队列节点op是TCGOp类型指针，结构体定义了一个队列实体link。都是宏定义。
双向链表，*tqe_next没用上，实际用只在队尾，相当于队列。*/
```

# PPT

做PPT重点

Qemu是什么得介绍下，qemu工作流程、结构得讲一下

Qemu重要的数据结构得讲一下

Elf文件存在哪儿，中间代码放在哪儿，翻译后的host代码放在哪儿

重点，指令提取的图来一张，过程说明一下

指令插入，操作链表，结构可以讲一下。

# 日志

qemu-sw64 -d in_asm,op,out_asm -cpu core3 hello > log 2>&1

in_asm,op,out_asm,exec

单步调试 -singlestep，每一个tb中就只有一条汇编代码，同时有很多tb没显示有汇编代码，可能是没有翻译，直接执行的？c 多少，在gdb ./hello-sw-dynamic中对应si 多少

exec打印出每个tb的首地址，方便对应。