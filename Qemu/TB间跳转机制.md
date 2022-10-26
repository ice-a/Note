# TranslationBlock

```c
struct TranslationBlock {
    target_ulong pc;   /* simulated PC corresponding to this block (EIP + CS base) */ //对应该TB块的模拟PC值
    target_ulong cs_base; /* CS base for this block */
    uint32_t flags; /* flags defining in which context the code was generated */
    uint32_t cflags;    /* compile flags */
#define CF_COUNT_MASK  0x00007fff
#define CF_LAST_IO     0x00008000 /* Last insn may be an IO access.  */ //最后一条指令也许是IO访问
#define CF_MEMI_ONLY   0x00010000 /* Only instrument memory ops */ //仅仪器内存操作
#define CF_USE_ICOUNT  0x00020000
#define CF_INVALID     0x00040000 /* TB is stale. Set with @jmp_lock held */ //TB已过时，保持@jmp_lock时设置
#define CF_PARALLEL    0x00080000 /* Generate code for a parallel context */ //为并行上下文生成代码
#define CF_CLUSTER_MASK 0xff000000 /* Top 8 bits are cluster ID */ //前8位是群集ID
#define CF_CLUSTER_SHIFT 24

    /* Per-vCPU dynamic tracing state used to generate this TB */
    uint32_t trace_vcpu_dstate;

    /*
     * Above fields used for comparing
     */

    /* size of target code for this block (1 <= size <= TARGET_PAGE_SIZE) */
    uint16_t size;//target代码的大小
    uint16_t icount;//target代码的数量

    struct tb_tc tc;

    /* first and second physical page containing code. The lower bit
       of the pointer tells the index in page_next[].
       The list is protected by the TB's page('s) lock(s) */
    uintptr_t page_next[2];
    tb_page_addr_t page_addr[2];

    /* jmp_lock placed here to fill a 4-byte hole. Its documentation is below */
    QemuSpin jmp_lock;//锁，包含一个value，0表示没有持有者，1表示有

//1
    uint16_t jmp_reset_offset[2]; /* offset of original jump target */ //原本跳转指令的跳转目标偏移量 数组
    //为jmp_reset_offset设置一个0xffff作为没有生成跳转的标志
#define TB_JMP_RESET_OFFSET_INVALID 0xffff /* indicates no jump generated */ 
    uintptr_t jmp_target_arg[2];  /* target address or offset */ //跳转指令的跳转地址或偏移量

//2
    uintptr_t jmp_list_head;//空结束链表
    uintptr_t jmp_list_next[2];//链表下一个
    uintptr_t jmp_dest[2];//unsigned long类型变量保存向外跳转两个tb的地址，两个outgoing jumps。相当于指针？
};
```

1、

以下数据用于从该TB的代码直接调用另一TB。这可以通过发出直接或间接的本机跳转指令来实现。这些跳转被重置，以便TB继续执行。通过设置一个跳转目标（或修补跳转指令），TB可以链接到另一个TB。仅支持其中两个跳转。

2、

每个tb有一个包含即将到来的跳转的空结束链表（jmp_list_head）。每个tb可以有两个向外的跳转，因此可以加入两个链表。链表实体存放在jmp_list_next[2]。这些链表指针的最低有效位（LSB）用于编码TB指向两个列表实体中的哪一个。

列表遍历受jmp_lock保护。每个传出跳转的目标TB保存在jmp_dest[]中，以便可以从任何源TB获取适当的jmp_lock。

jmp_dest[]也是标记指针。LSB是在TB失效时设置的，因此不能再设置它的传出跳转。

jmp_lock还保护CF_INVALID cflag；跳跃不能用链子锁住到设置了CF_INVALID的目标TB。

# TCGContext

```c
struct TCGContext {
    /* goto_tb support */ //跳转支持goto_tb
    tcg_insn_unit *code_buf;//TB块翻译代码的开始位置,tb->tb_tc.ptr
    uint16_t *tb_jmp_reset_offset; /* tb->jmp_reset_offset */   //指针
    uintptr_t *tb_jmp_insn_offset; /* tb->jmp_target_arg if direct_jump */ //跳转指令偏移，支持直接跳转 数组指针，数组元素为偏移量
    uintptr_t *tb_jmp_target_addr; /* tb->jmp_target_arg if !direct_jump */ //跳转目标地址，不直接跳转

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

# tb_find()

```c
static inline TranslationBlock *tb_find(CPUState *cpu,
                                        TranslationBlock *last_tb,
                                        int tb_exit, uint32_t cflags)
{
    CPUArchState *env = (CPUArchState *)cpu->env_ptr;//用CPUState *cpu中的env强制类型转换成CPUArchState（CPUSW64State）的env
    TranslationBlock *tb;//TB块指针tb
    target_ulong cs_base, pc;
    uint32_t flags;

    cpu_get_tb_cpu_state(env, &pc, &cs_base, &flags);//从env中取pc，cs_base=0和flags

    tb = tb_lookup(cpu, pc, cs_base, flags, cflags);//在cache中根据查找TB，/include/exec/tb-lookup.h
    if (tb == NULL) {//如果没有找到tb，则
        mmap_lock();
        tb = tb_gen_code(cpu, pc, cs_base, flags, cflags);//翻译生成一个TB块
        mmap_unlock();
        /* We add the TB in the virtual pc hash table for the fast lookup */
        //我们把TB添加到虚拟pc哈希表中以便快速查找
        //根据pc生成hash值，将tb写入到hash表中
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
    if (last_tb) {//若有上一个TB块，将上一个TB块与当前TB块进行链接
//last_tb是TB的指针，而tb_exit是jump slot index（就是索引0还是1），链接last_tb->tb
        tb_add_jump(last_tb, tb_exit, tb);//修改上一个TB块的BR的跳转地址为当前TB
    }
    return tb;
}
```

## tb_gen_code()

```c
    gen_intermediate(cpu,tb,max_insns);//前端
    /* generate machine code */  //生成机器码，初始化tb与tcg_ctx中的相关成员变量
    tb->jmp_reset_offset[0] = TB_JMP_RESET_OFFSET_INVALID;
    tb->jmp_reset_offset[1] = TB_JMP_RESET_OFFSET_INVALID;
    //tb->jmp_reset_offset[]这个数组首地址赋值给tcg_ctx->tb_jmp_reset_offset这个数组指针
    //即让tcg_ctx->tb_jmp_reset_offset也指向tb->jmp_reset_offset[]数组
    tcg_ctx->tb_jmp_reset_offset = tb->jmp_reset_offset;
    if (TCG_TARGET_HAS_direct_jump) {//SW中宏设定为1，代表SW支持直接跳转这个功能？
        tcg_ctx->tb_jmp_insn_offset = tb->jmp_target_arg;//如支持，则tcg_ctx->tb_jmp_insn_offset是
        tcg_ctx->tb_jmp_target_addr = NULL;
    } else {
        tcg_ctx->tb_jmp_insn_offset = NULL;
        tcg_ctx->tb_jmp_target_addr = tb->jmp_target_arg;//如不支持，则tcg_ctx->tb_jmp_target_addr是
    }
    /* init jump list */ //初始化跳转链表，全置NULL
    qemu_spin_init(&tb->jmp_lock);//初始化自旋锁，将tb->jmp_lock->value置为0
    tb->jmp_list_head = (uintptr_t)NULL;
    tb->jmp_list_next[0] = (uintptr_t)NULL;
    tb->jmp_list_next[1] = (uintptr_t)NULL;
    tb->jmp_dest[0] = (uintptr_t)NULL;
    tb->jmp_dest[1] = (uintptr_t)NULL;

    gen_code_size=tcg_gen_code(tcg_ctx,tb);

    /* init original jump addresses which have been set during tcg_gen_code() */
    //初始化在tcg_gen_code()期间设置的原始跳转地址
    if (tb->jmp_reset_offset[0] != TB_JMP_RESET_OFFSET_INVALID) {//有跳转
        tb_reset_jump(tb, 0);
    }
    if (tb->jmp_reset_offset[1] != TB_JMP_RESET_OFFSET_INVALID) {
        tb_reset_jump(tb, 1);
    }
```

### 前端翻译

```c
        case 0x30:
            /* BEQ */
            ret = gen_bcond(ctx, TCG_COND_EQ, ra, disp21, (uint64_t)-1);
            break;
```

```c
static DisasJumpType gen_bcond(DisasContext *ctx, TCGCond cond, uint32_t ra,
                               int32_t disp, uint64_t mask)
{
    tcg_gen_andi_i64(tmp, load_gir(ctx, ra), mask);// mov_i64 tmp2,s0
    ret = gen_bcond_internal(ctx, cond, tmp, disp);
}
```

```c
static DisasJumpType gen_bcond_internal(DisasContext *ctx, TCGCond cond,
                                        TCGv cmp, int disp)
{
    uint64_t dest = ctx->base.pc_next + (disp << 2);
    TCGLabel* lab_true = gen_new_label();

    if (use_goto_tb(ctx, dest)) {
        tcg_gen_brcondi_i64(cond, cmp, 0, lab_true);

        tcg_gen_goto_tb(0);
        tcg_gen_movi_i64(cpu_pc, ctx->base.pc_next);
        tcg_gen_exit_tb(ctx->base.tb, 0);

        gen_set_label(lab_true);
        tcg_gen_goto_tb(1);
        tcg_gen_movi_i64(cpu_pc, dest);
        tcg_gen_exit_tb(ctx->base.tb, 1);

        return DISAS_NORETURN;
    } else {
        TCGv_i64 t = tcg_const_i64(0);
        TCGv_i64 d = tcg_const_i64(dest);
        TCGv_i64 p = tcg_const_i64(ctx->base.pc_next);

        tcg_gen_movcond_i64(cond, cpu_pc, cmp, t, d, p);

        tcg_temp_free_i64(t);
        tcg_temp_free_i64(d);
        tcg_temp_free_i64(p);
        return DISAS_PC_UPDATED;
    }
} 
```

### 后端翻译tcg_out_op()

```c
static void tcg_out_op(TCGContext *s, TCGOpcode opc, const TCGArg args[TCG_MAX_OP_ARGS],const int const_args[TCG_MAX_OP_ARGS])
{
    switch (opc) {
     case INDEX_op_exit_tb: /* sw */
        /* Reuse the zeroing that exists for goto_ptr.  */
        if (a0 == 0) {
            tcg_out_goto_long(s, tcg_code_gen_epilogue);
        } else {
            tcg_out_movi(s, TCG_TYPE_I64, TCG_REG_X0, a0);
            tcg_out_goto_long(s, tb_ret_addr);
        }
        break;

     case INDEX_op_goto_tb: /* sw */
        if (s->tb_jmp_insn_offset != NULL) {
            /* TCG_TARGET_HAS_direct_jump */
            /* Ensure that ADRP+ADD are 8-byte aligned so that an atomic
               write can be used to patch the target address. */
            //要按字节对齐，才可以用原子操作修改。取低3位，不是000则多一个NOP
            //pc地址一般按照字节编址，pc寄存器按字编址。
            //32位指令按照字节对齐后，一定是4的倍数，0x00,0x04,0x08,0x0c,0x10低2位一定是0
            if ((uintptr_t)s->code_ptr & 7) {
                tcg_out32(s, OPC_NOP);//添加NOP指令
            }
            //记录当前jmp指令在TB块中的偏移量？记录的是第1/2条NOP指令在TB块中偏移量
            //s->tb_jmp_insn_offset[a0]=s->code_ptr - s->code_buf，偏移量赋给数组指针指向的数组元素
            s->tb_jmp_insn_offset[a0] = tcg_current_code_size(s);
            //*s->code_ptr=v (0x43ff075f),s->code_ptr++ 生成指令的函数tcg_out32，预留四条指令，全部填NOP指令
            tcg_out32(s, OPC_NOP);
            tcg_out32(s, OPC_NOP);
            tcg_out32(s, OPC_NOP);
            tcg_out32(s, OPC_NOP);//total 2 nop for tb_target
        } else {
            /* !TCG_TARGET_HAS_direct_jump */
            tcg_debug_assert(s->tb_jmp_target_addr != NULL);
        tcg_out_ld(s, TCG_TYPE_PTR, TCG_REG_TMP, TCG_REG_ZERO, (uintptr_t)(s->tb_jmp_target_addr + a0));
        }
        tcg_out_insn_jump(s, OPC_JMP, TCG_REG_ZERO, TCG_REG_TMP, noPara);
        //此时host已经生成5/6条指令，4/5*nop+jmp，记录的是jmp指令的下一条指令在TB块中偏移量
        set_jmp_reset_offset(s, a0);//s->tb_jmp_reset_offset[a0]=tcg_current_code_size(s)
        break;
     }
 }
```

## tb_reset_jump()

```c
static inline void tb_reset_jump(TranslationBlock *tb, int n)
{
    //当前TB起始地址+jmp下一指令偏移量=原始跳转地址（下一指令地址）
    uintptr_t addr = (uintptr_t)(tb->tc.ptr + tb->jmp_reset_offset[n]);
    tb_set_jmp_target(tb, n, addr);//修改后端翻译中的NOP为BR指令，跳转地址为原始跳转地址
}
```

### tb_set_jmp_target()

```c
void tb_set_jmp_target(TranslationBlock *tb, int n, uintptr_t addr)
{
    if (TCG_TARGET_HAS_direct_jump) {//后端是SW，支持直接跳转
        uintptr_t offset = tb->jmp_target_arg[n];//找到需要修改的NOP指令在TB中的偏移量
        uintptr_t tc_ptr = (uintptr_t)tb->tc.ptr;//当前TB的起始位置
        uintptr_t jmp_rx = tc_ptr + offset;//当前TB中NOP指令的地址
        uintptr_t jmp_rw = jmp_rx - tcg_splitwx_diff;//？
        tb_target_set_jmp_target(tc_ptr, jmp_rx, jmp_rw, addr);//arch定义的修改指令，后端为SW
    } else {
        tb->jmp_target_arg[n] = addr;//不支持直接跳转，则下一指令地址给tb->jmp_target_arg[n]
    }
}
```

```c
/* TCG_TARGET_HAS_direct_jump */
void tb_target_set_jmp_target(uintptr_t tc_ptr, uintptr_t jmp_rx, uintptr_t jmp_rw, uintptr_t addr)
{
    tcg_insn_unit i1, i2;//tcg_insn_unit大小与host机器指令长度一致，表示一条指令
    uint64_t pair;

    //原始跳转地址偏移量=原始跳转地址（下一条指令地址）-当前TB中NOP指令的地址-1
    ptrdiff_t offset = addr - jmp_rx -1;

    if (offset == sextract64(offset, 0, 21)) {//偏移量在21位内，在BR跳转范围内
        //disp->Vaddr要左移两位再符号扩展，这里右移两位再截取低21位
        //所有跳转指令的目标地址，编译存储的时候都会/4，实际运行的时候*4还原。
        i1 = OPC_BR | (TCG_REG_ZERO & 0x1f) << 21| ((offset >> 2) & 0x1fffff);//直接转移BR指令
    i2 = OPC_NOP;//NOP指令
        //把NOP指令和BR指令拼接起来，SW支持小端存储，因此先BR后NOP
        pair = (uint64_t)i2 << 32 | i1;
        qatomic_set((uint64_t *)jmp_rw, pair);//*jmp_rw=pair，0x42ff075f13e00083
        flush_idcache_range(jmp_rx, jmp_rw, 8); //not support no sw_64
    } else if(offset == sextract64(offset, 0, 32)){//偏移量超过21位，跳转范围不够
        modify_direct_addr(addr, jmp_rw, jmp_rx);
    } else {
      tcg_debug_assert("tb_target");
    }
}
```

## tb_add_jump()

```c
static inline void tb_add_jump(TranslationBlock *tb, int n,
                               TranslationBlock *tb_next)
{
    uintptr_t old;

    qemu_thread_jit_write();
    assert(n < ARRAY_SIZE(tb->jmp_list_next));//n<2
//持有自旋锁tb_next->jmp_lock->value置为true。若为false，则一直读取value并等待知道锁可用
    qemu_spin_lock(&tb_next->jmp_lock);

    //确保目标TB有效
    if (tb_next->cflags & CF_INVALID) {
        goto out_unlock_next;
    }
    /* Atomically claim the jump destination slot only if it was NULL */
    //仅当跳转目标插槽（tb->jmp_dest[n]）为NULL时，才原子声明该插槽 ，代表还没设置跳转？没建链？
    old = qatomic_cmpxchg(&tb->jmp_dest[n], (uintptr_t)NULL,
                          (uintptr_t)tb_next);//tb->jmp_dest[n]=tb_next，建链
    if (old) {//old正常返回NULL值，tb->jmp_dest[n]声明过则会返回tb->jmp_dest[n]。
        goto out_unlock_next;//不进行修改建链操作
    }

    /* patch the native jump address */
    tb_set_jmp_target(tb, n, (uintptr_t)tb_next->tc.ptr);//修改上一个TB的BR指令的地址部分

    /* add in TB jmp list */
    tb->jmp_list_next[n] = tb_next->jmp_list_head;//这两个是关联关系是为了unlink时使用
    tb_next->jmp_list_head = (uintptr_t)tb | n;//保存指针tb的地址，低2位用于编码TB指向两个列表实体中的哪一个。

    qemu_spin_unlock(&tb_next->jmp_lock);//释放锁，将tb_next->jmp_lock->value置为0。

    qemu_log_mask_and_addr(CPU_LOG_EXEC, tb->pc,
                           "Linking TBs %p [" TARGET_FMT_lx
                           "] index %d -> %p [" TARGET_FMT_lx "]\n",
                           tb->tc.ptr, tb->pc, n,
                           tb_next->tc.ptr, tb_next->pc);
    return;

 out_unlock_next:
    qemu_spin_unlock(&tb_next->jmp_lock);//释放锁，将tb_next->jmp_lock->value置为0。
    return;
}
```

### tb_set_jmp_target()

```c
void tb_set_jmp_target(TranslationBlock *tb, int n, uintptr_t addr)
{
    if (TCG_TARGET_HAS_direct_jump) {//后端是SW，支持直接跳转
        uintptr_t offset = tb->jmp_target_arg[n];//找到需要修改的BR指令在TB中的偏移量
        uintptr_t tc_ptr = (uintptr_t)tb->tc.ptr;//前一个TB的起始位置
        uintptr_t jmp_rx = tc_ptr + offset;//前一个TB中BR指令的地址
        uintptr_t jmp_rw = jmp_rx - tcg_splitwx_diff;//
        tb_target_set_jmp_target(tc_ptr, jmp_rx, jmp_rw, addr);//arch定义的修改指令，后端为SW
    } else {
        tb->jmp_target_arg[n] = addr;//不支持直接跳转，则将当前tb的起始地址给tb->jmp_target_arg[n]
    }
}
```

```c
/* TCG_TARGET_HAS_direct_jump */
void tb_target_set_jmp_target(uintptr_t tc_ptr, uintptr_t jmp_rx, uintptr_t jmp_rw, uintptr_t addr)
{
    tcg_insn_unit i1, i2;//tcg_insn_unit大小与host机器指令长度一致，表示一条指令
    uint64_t pair;
    //相对跳转地址偏移量=当前TB起始地址-前一个TB中BR指令的地址-1
    ptrdiff_t offset = addr - jmp_rx -1;

    if (offset == sextract64(offset, 0, 21)) {//偏移量在21位内，在BR跳转范围内
        //disp->Vaddr要左移两位再符号扩展，这里右移两位再截取低21位
        i1 = OPC_BR | (TCG_REG_ZERO & 0x1f) << 21| ((offset >> 2) & 0x1fffff);//直接转移BR指令
    i2 = OPC_NOP;//NOP指令
        //把NOP指令和BR指令拼接起来，SW支持小端存储，因此先BR后NOP
        pair = (uint64_t)i2 << 32 | i1;
        qatomic_set((uint64_t *)jmp_rw, pair);//*jmp_rw=pair，0x42ff075f13e00083
        flush_idcache_range(jmp_rx, jmp_rw, 8); //not support no sw_64
    } else if(offset == sextract64(offset, 0, 32)){//偏移量超过21位，跳转范围不够
        modify_direct_addr(addr, jmp_rw, jmp_rx);//四条指令，因此要留四个NOP
    } else {
      tcg_debug_assert("tb_target");
    }
}
```

# Issue

- [ ] n=0、1 TB通常是以 跳转指令结束的， 而跳转指令其实只有两种选择: 向左或者向右, 两个 jump slot 代表两个跳转的目标， 也就是 goto_tb 0x0/0x1。
  
  现在只看了条件跳转指令翻译成goto_tb再后端翻译相关的，别的跳转还没看。

- [x] s到tb是怎么赋值的。

- [ ] 把链表相关的看完，与页有关。tb_jmp_unlink

- [ ] 动态翻译，翻译到跳转指令，要读寄存器什么的（跳转地址会有个计算结果的）不知道会跳转到哪里。因此TB得结束，然后执行完毕才能知道跳转到哪里。静态翻译就会把可能需要翻译的东西全给你翻译了。或者动态翻译完生成tb文件，下次接着用。

- [ ] 变量对应关系，函数调用关系得做表格，不然容易忘记。

- [ ] 为什么修改指令时要添加一条NOP指令

- [ ] 为什么原始跳转地址一定是下一条呢，假如翻译直接跳转指令，地址都有了能不能知道去哪儿了？tb为什么还要停呢？

- [ ] spliwx有空可以看一下，sw中是一样的

- [ ] 带符号的偏移量，为什么要先左移两位，再符号扩展
  
  pc寄存器的内容（即地址）是按照字节寻址的，PC相对寻址的偏移量（是立即数）是按字(32)寻址的（类似PC+1这种用法，pc+1是按字寻址，pc+4是按字节寻址），所以需要disp*4。为什么要这么存储disp呢？因为低两位一定是0，所以编译存储/4，用的时候再\*4

- [ ] unsigned long类型转换成long类型溢出，变成-1
