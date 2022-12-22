# TranslationBlock

```c
struct TranslationBlock {

    struct tb_tc tc;//host翻译块

    /* jmp_lock placed here to fill a 4-byte hole. Its documentation is below */
    QemuSpin jmp_lock;//锁，包含一个value，0表示没有持有者，1表示有

    /*
    以下数据用于从该TB的代码直接调用另一TB。这可以通过发出直接或间接的本机跳转指令来实现。
    这些跳转被重置，以便TB继续执行。通过设置一个跳转目标（或修补跳转指令），TB可以链接到另一个TB。
    仅支持其中两个跳转
    */
    uint16_t jmp_reset_offset[2]; /* offset of original jump target */ //跳转指令的原始跳转目标偏移量 
    //为jmp_reset_offset设置一个0xffff作为没有生成跳转的标志
#define TB_JMP_RESET_OFFSET_INVALID 0xffff /* indicates no jump generated */ 
    uintptr_t jmp_target_arg[2];  /* target address or offset */ //跳转指令相对于当前TB的偏移量或跳转目标地址

    /*
    每个tb有一个包含即将到来的跳转的空结束链表（jmp_list_head）。每个tb可以有两个向外的跳转，因此可以加入两个链表。
    链表实体存放在jmp_list_next[2]。这些链表指针的最低有效位（LSB）用于编码TB指向两个列表实体中的哪一个。

    列表遍历受jmp_lock保护。每个传出跳转的目标TB保存在jmp_dest[]中，以便可以从任何源TB获取适当的jmp_lock。

    jmp_dest[]也是标记指针。LSB是在TB失效时设置的，因此不能再设置它的传出跳转。

    jmp_lock还保护CF_INVALID cflag；跳跃不能用链子锁住到设置了CF_INVALID的目标TB。
    */
    uintptr_t jmp_list_head;//已执行的上一个TB
    uintptr_t jmp_list_next[2];//前驱TB链表上的下一个结点
    uintptr_t jmp_dest[2];//下一个要执行的TB
};
```

# TCGContext

```c
struct TCGContext {
    /* goto_tb support */ //goto_tb支持
    tcg_insn_unit *code_buf;//TB块翻译代码的开始位置,tb->tb_tc.ptr
    uint16_t *tb_jmp_reset_offset; /* tb->jmp_reset_offset */   //指针
    uintptr_t *tb_jmp_insn_offset; /* tb->jmp_target_arg if direct_jump */ //跳转指令相对于当前TB的偏移量，支持直接跳转
    uintptr_t *tb_jmp_target_addr; /* tb->jmp_target_arg if !direct_jump */ //跳转目标地址，不支持直接跳转

    tcg_insn_unit *code_ptr;//下一条翻译代码的保存位置
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
TranslationBlock *tb_gen_code(CPUState *cpu,
                              target_ulong pc, target_ulong cs_base,
                              uint32_t flags, int cflags)
{
    ...
    gen_intermediate(cpu,tb,max_insns);//前端
    ...
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
    ...
    gen_code_size=tcg_gen_code(tcg_ctx,tb);
    ...
    /* init jump list */ //初始化跳转链表，全置NULL
    qemu_spin_init(&tb->jmp_lock);//初始化自旋锁，将tb->jmp_lock->value置为0
    tb->jmp_list_head = (uintptr_t)NULL;
    tb->jmp_list_next[0] = (uintptr_t)NULL;
    tb->jmp_list_next[1] = (uintptr_t)NULL;
    tb->jmp_dest[0] = (uintptr_t)NULL;
    tb->jmp_dest[1] = (uintptr_t)NULL;
    ...
    /* init original jump addresses which have been set during tcg_gen_code() */
    //初始化在tcg_gen_code()期间设置的原始跳转地址
    if (tb->jmp_reset_offset[0] != TB_JMP_RESET_OFFSET_INVALID) {//有跳转
        tb_reset_jump(tb, 0);//tb_reset_jump重置一个TB的跳转实体n，使它没有被链接到其他TB
    }
    if (tb->jmp_reset_offset[1] != TB_JMP_RESET_OFFSET_INVALID) {
        tb_reset_jump(tb, 1);
    }
}
```

### SW前端 指令翻译translate_one()

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
    uint64_t dest = ctx->base.pc_next + (disp << 2);//计算跳转地址
    TCGLabel* lab_true = gen_new_label();

    if (use_goto_tb(ctx, dest)) {//同一页
        tcg_gen_brcondi_i64(cond, cmp, 0, lab_true);
        //goto_tb作用就是预留空间给跳转指令,index是让执行完毕tb后要建链，修改br指令时知道要修改哪一个
        tcg_gen_goto_tb(0);//op->args[0]=0  goto_tb $0x0
        tcg_gen_movi_i64(cpu_pc, ctx->base.pc_next);//mov_i64 pc,下一条guest指令地址
        tcg_gen_exit_tb(ctx->base.tb, 0);//exit_tb 返回值零或tb+idx 

        gen_set_label(lab_true);
        tcg_gen_goto_tb(1);
        tcg_gen_movi_i64(cpu_pc, dest);//mov_i64 pc,guest指令跳转地址
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

```c
bool use_goto_tb(DisasContext* ctx, uint64_t dest)
{
    /* Suppress goto_tb in the case of single-steping and IO.  */
    if (unlikely(use_exit_tb(ctx))) {
        return false;
    }
    /* If the destination is in the superpage, the page perms can't change.  */
    if (in_superpage(ctx, dest)) {
        return true;
    }
/* Check for the dest on the same page as the start of the TB.  */
#ifndef CONFIG_USER_ONLY
    return ((ctx->base.tb->pc ^ dest) & TARGET_PAGE_MASK) == 0;
#else
    return true;
#endif
}
```

#### 生成goto_tb

```c
void tcg_gen_goto_tb(unsigned idx)
{
    /* We only support two chained exits.  */
    tcg_debug_assert(idx <= TB_EXIT_IDXMAX);
#ifdef CONFIG_DEBUG_TCG
    /* Verify that we havn't seen this numbered exit before.  */
    tcg_debug_assert((tcg_ctx->goto_tb_issue_mask & (1 << idx)) == 0);
    tcg_ctx->goto_tb_issue_mask |= 1 << idx;
#endif
    plugin_gen_disable_mem_helpers();
    /* When not chaining, we simply fall through to the "fallback" exit.  */
    if (!qemu_loglevel_mask(CPU_LOG_TB_NOCHAIN)) {//开了nochain就不生成goto_tb，直接exit_tb
        tcg_gen_op1i(INDEX_op_goto_tb, idx);//生成goto_tb
    }
}
```

#### 生成exit_tb

```c
void tcg_gen_exit_tb(const TranslationBlock *tb, unsigned idx)
{
    //tcg_splitwx_to_rx的逻辑？exit_tb返回值val=tb+idx，低两位附加信息，index用了一位
    uintptr_t val = (uintptr_t)tcg_splitwx_to_rx((void *)tb) + idx;

    if (tb == NULL) {
        //断言idx=0
        tcg_debug_assert(idx == 0);
    } else if (idx <= TB_EXIT_IDXMAX) {//idx=0、1
#ifdef CONFIG_DEBUG_TCG
        /* This is an exit following a goto_tb.  Verify that we have
           seen this numbered exit before, via tcg_gen_goto_tb.  */
        tcg_debug_assert(tcg_ctx->goto_tb_issue_mask & (1 << idx));
#endif
        /* When not chaining, exit without indicating a link.  */ //不链接时，退出且不指示链接。
        if (qemu_loglevel_mask(CPU_LOG_TB_NOCHAIN)) {//-d nochain 则不链接TB
            val = 0;//置为0
        }
    } else {//idx=2、3
        /* This is an exit via the exitreq label.  */
        //这是通过exitreq标签的出口。
        //断言idx=3
        tcg_debug_assert(idx == TB_EXIT_REQUESTED);
    }

    plugin_gen_disable_mem_helpers();
    tcg_gen_op1i(INDEX_op_exit_tb, val);//生成返回值val
}
```

### x86前端disas_insn()

```c
switch(b) {
    ...
    case 0x70 ... 0x7f: /* jcc Jb */
        tval = (int8_t)insn_get(env, s, MO_8);
        goto do_jcc;
    do_jcc:
        next_eip = s->pc - s->cs_base;
        tval += next_eip;
        if (dflag == MO_16) {
            tval &= 0xffff;
        }
        gen_bnd_jmp(s);
        gen_jcc(s, b, tval, next_eip);
        break;
    ...
}
```

```c
static inline void gen_jcc(DisasContext *s, int b,
                           target_ulong val, target_ulong next_eip)
{
    TCGLabel *l1, *l2;

    if (s->jmp_opt) {//支持还是不支持direct jump都是1，为啥？
        l1 = gen_new_label();
        gen_jcc1(s, b, l1);

        gen_goto_tb(s, 0, next_eip);//next_eip与val是要跳转的前端地址

        gen_set_label(l1);
        gen_goto_tb(s, 1, val);
    } else {
        l1 = gen_new_label();
        l2 = gen_new_label();
        gen_jcc1(s, b, l1);

        gen_jmp_im(s, next_eip);
        tcg_gen_br(l2);

        gen_set_label(l1);
        gen_jmp_im(s, val);
        gen_set_label(l2);
        gen_eob(s);
    }
}
```

#### 生成goto_tb、exit_tb

```c
static inline void gen_goto_tb(DisasContext *s, int tb_num, target_ulong eip)
{
    target_ulong pc = s->cs_base + eip;

    if (use_goto_tb(s, pc))  {//判断是否在同一页内
        /* jump to same page: we can use a direct jump */
        tcg_gen_goto_tb(tb_num);//goto_tb
        gen_jmp_im(s, eip);
        tcg_gen_exit_tb(s->base.tb, tb_num);
        s->base.is_jmp = DISAS_NORETURN;
    } else {
        /* jump to another page */
        gen_jmp_im(s, eip);
        gen_jr(s, s->tmp0);
    }
}
```

### SW后端 中间码翻译tcg_reg_alloc_op()

#### 传参

```c
static void tcg_reg_alloc_op(TCGContext *s, const TCGOp *op)
{
    TCGArg new_args[TCG_MAX_OP_ARGS];
    int const_args[TCG_MAX_OP_ARGS];

    /* copy constants */ //拷贝常量
    //对于goto_tb代码来说，只有一个常量参数index放在op->args[0]
    //对于exit_tb代码来说，只有一个常量参数tb+tb_num放在op->args[0]
    memcpy(new_args + nb_oargs + nb_iargs, 
           op->args + nb_oargs + nb_iargs,
           sizeof(TCGArg) * def->nb_cargs);
    /* emit instruction */
    if (def->flags & TCG_OPF_VECTOR) {
        tcg_out_vec_op(s, op->opc, TCGOP_VECL(op), TCGOP_VECE(op),
                       new_args, const_args);
    } else {
        tcg_out_op(s, op->opc, new_args, const_args);
    }
}
```

#### 翻译goto_tb、exit_tb

```c
static void tcg_out_op(TCGContext *s, TCGOpcode opc, 
                        const TCGArg args[TCG_MAX_OP_ARGS],const int const_args[TCG_MAX_OP_ARGS])
{
    TCGArg a0 = args[0];//常量
     switch (opc) {
     case INDEX_op_exit_tb: /* sw */
        /* Reuse the zeroing that exists for goto_ptr.  */
        //exit_tb标记着从TB中返回到qemu，这里有个小优化，如果exit_tb 0则返回生成的epilogue位置，如果是exit_tb 非0则返回tb_ret_addr。
        //而epiogue是tb_ret_addr的上一条指令位置，将返回值赋为0，这样每个TB中就不必赋0，相当于省了一条指令。
        if (a0 == 0) {//对于exit_tb来说a0是返回值val，即val=0，epilogue会有一条ldi把$r0置为0
            tcg_out_goto_long(s, tcg_code_gen_epilogue);//br tcg_code_gen_epilogue
        } else {
            tcg_out_movi(s, TCG_TYPE_I64, TCG_REG_X0, a0);//movi $r0,a0
            tcg_out_goto_long(s, tb_ret_addr);//br tb_ret_addr
        }
        break;

     case INDEX_op_goto_tb: /* sw */  
        if (s->tb_jmp_insn_offset != NULL) {//支持direct_jump
            /* TCG_TARGET_HAS_direct_jump */
            /* Ensure that ADRP+ADD are 8-byte aligned so that an atomic
               write can be used to patch the target address. */
            //要按8字节对齐。取低3位，不是000则多一个NOP
            //pc地址一般按照字节编址，pc寄存器按字编址。
            //32位指令按照字节对齐后，一定是4的倍数，0x00,0x04,0x08,0x0c,0x10低2位一定是0
            //注释说要8字节对齐？才可以用原子操作修改？
            if ((uintptr_t)s->code_ptr & 7) {
                tcg_out32(s, OPC_NOP);//添加NOP指令
            }
            //对于goto_tb来说a0为index索引。记录的是第1/2条NOP指令在TB块中偏移量
            //s->tb_jmp_insn_offset[a0]=s->code_ptr - s->code_buf，偏移量赋给数组指针指向的数组元素
            s->tb_jmp_insn_offset[a0] = tcg_current_code_size(s);
            //*s->code_ptr=v (0x43ff075f),s->code_ptr++ 生成指令的函数tcg_out32，预留四条指令，全部填NOP指令
            tcg_out32(s, OPC_NOP);
            tcg_out32(s, OPC_NOP);
            tcg_out32(s, OPC_NOP);
            tcg_out32(s, OPC_NOP);//total 2 nop for tb_target
        } else {//不支持direct_jump
            /* !TCG_TARGET_HAS_direct_jump */
            tcg_debug_assert(s->tb_jmp_target_addr != NULL);
        //s->tb_jmp_target_addr是一个指针，指向数组首地址，+1则是指向1号元素。后续会把跳转地址放在这里面。
        //TCG_REG_TMP是$r27，TCG_REG_ZERO是$r31
        //ldl $27,(s->tb_jmp_target_addr + a0)$r31  把跳转地址放入$r27
        tcg_out_ld(s, TCG_TYPE_PTR, TCG_REG_TMP, TCG_REG_ZERO, (uintptr_t)(s->tb_jmp_target_addr + a0));
        }
        tcg_out_insn_jump(s, OPC_JMP, TCG_REG_ZERO, TCG_REG_TMP, noPara);//jmp $r31,($r27) 根据$r27跳转
        //此时host已经生成5/6条指令，4/5*nop+jmp，记录的是jmp指令的下一条指令在TB块中偏移量。
        set_jmp_reset_offset(s, a0);//s->tb_jmp_reset_offset[a0]=tcg_current_code_size(s)
        break;
     }
 }
```

### 初始化/重置跳转地址tb_reset_jump()

```c
/* reset the jump entry 'n' of a TB so that it is not chained to
   another TB */static inline void tb_reset_jump(TranslationBlock *tb, int n)
{
    //原始跳转地址（下一指令地址）=当前TB起始地址+jmp下一指令偏移量
    uintptr_t addr = (uintptr_t)(tb->tc.ptr + tb->jmp_reset_offset[n]);
    tb_set_jmp_target(tb, n, addr);//修改后端翻译中的NOP为BR指令，跳转地址为原始跳转地址
}
```

#### 设置跳转地址tb_set_jmp_target()

```c
void tb_set_jmp_target(TranslationBlock *tb, int n, uintptr_t addr)
{
    if (TCG_TARGET_HAS_direct_jump) {//后端是SW，支持直接跳转
        uintptr_t offset = tb->jmp_target_arg[n];//找到需要修改的NOP指令在TB中的偏移量
        uintptr_t tc_ptr = (uintptr_t)tb->tc.ptr;//当前TB的起始地址
        uintptr_t jmp_rx = tc_ptr + offset;//当前TB中NOP指令的地址
        uintptr_t jmp_rw = jmp_rx - tcg_splitwx_diff;//tcg_splitwx_diff SW中为0
        tb_target_set_jmp_target(tc_ptr, jmp_rx, jmp_rw, addr);//架构相关的修改函数，后端为SW
    } else {//不支持直接跳转，则下一指令地址给tb->jmp_target_arg[n]
        tb->jmp_target_arg[n] = addr;
    }
}
```

```c
/* TCG_TARGET_HAS_direct_jump */
void tb_target_set_jmp_target(uintptr_t tc_ptr, uintptr_t jmp_rx, uintptr_t jmp_rw, uintptr_t addr)
{
    tcg_insn_unit i1, i2;//tcg_insn_unit大小与host机器指令长度一致，表示一条指令
    uint64_t pair;//64位value

    //原始跳转地址偏移量=原始跳转地址（下一条指令地址）-当前TB中NOP指令的地址-1
    //为什么要-1，因为BR指令是先更新PC再PC+offset,所以偏移量要少一个，这里-1一会儿右移就能少
    ptrdiff_t offset = addr - jmp_rx -1;

    if (offset == sextract64(offset, 0, 21)) {//偏移量在21位内，在BR跳转范围内
        //disp->Vaddr要左移两位再符号扩展，这里右移两位再截取低21位
        //所有跳转指令的目标地址，编译存储的时候都会/4，实际运行的时候*4还原。
        i1 = OPC_BR | (TCG_REG_ZERO & 0x1f) << 21| ((offset >> 2) & 0x1fffff);//直接转移BR指令
    i2 = OPC_NOP;//NOP指令
        //把NOP指令和BR指令拼接起来，SW支持小端存储，因此先BR后NOP
        pair = (uint64_t)i2 << 32 | i1;
        qatomic_set((uint64_t *)jmp_rw, pair);//*jmp_rw=pair，0x42ff075f13e00083 ，原子写操作要64位一放
        flush_idcache_range(jmp_rx, jmp_rw, 8); //not support no sw_64
    } else if(offset == sextract64(offset, 0, 32)){//偏移量超过21位，跳转范围不够
        modify_direct_addr(addr, jmp_rw, jmp_rx);//四条指令
    } else {
      tcg_debug_assert("tb_target");
    }
}
```

## TB建链tb_add_jump()

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
    //仅当跳转目标插槽（tb->jmp_dest[n]）为NULL时，才原子声明该插槽 ，代表还没设置跳转，没建链。
    //原子比较和交换操作。若tb->jmp_dest[n]=NULL则tb->jmp_dest[n]=tb_next，返回NULL。
    //若tb->jmp_dest[n]!=NULL则返回tb->jmp_dest[n]。
    old = qatomic_cmpxchg(&tb->jmp_dest[n], (uintptr_t)NULL,
                          (uintptr_t)tb_next);//tb->jmp_dest[n]=tb_next，建链
    if (old) {//old正常返回NULL值，tb->jmp_dest[n]声明过则会返回tb->jmp_dest[n]。
        goto out_unlock_next;//不进行修改建链操作
    }

    /* patch the native jump address */
    tb_set_jmp_target(tb, n, (uintptr_t)tb_next->tc.ptr);//修改上一个TB的BR指令的地址部分

    /* add in TB jmp list */
    tb->jmp_list_next[n] = tb_next->jmp_list_head;//这两个是关联关系是为了unlink时使用
    tb_next->jmp_list_head = (uintptr_t)tb | n;//保存上一个TB指针tb的地址，低2位用于编码TB指向两个列表实体中的哪一个。

    qemu_spin_unlock(&tb_next->jmp_lock);//释放锁，将tb_next->jmp_lock->value置为0。

    qemu_log_mask_and_addr(CPU_LOG_EXEC, tb->pc,
                           "Linking TBs %p [" TARGET_FMT_lx
                           "] index %d -> %p [" TARGET_FMT_lx "]\n",
                           tb->tc.ptr, tb->pc, n,
                           tb_next->tc.ptr, tb_next->pc);//打印日志信息，将last_tb->tb链接起来
    return;

 out_unlock_next:
    qemu_spin_unlock(&tb_next->jmp_lock);//释放锁，将tb_next->jmp_lock->value置为0。
    return;
}
```

### 设置跳转地址tb_set_jmp_target()

```c
void tb_set_jmp_target(TranslationBlock *tb, int n, uintptr_t addr)
{
    if (TCG_TARGET_HAS_direct_jump) {//后端是SW，支持直接跳转
        uintptr_t offset = tb->jmp_target_arg[n];//找到需要修改的BR指令在TB中的偏移量
        uintptr_t tc_ptr = (uintptr_t)tb->tc.ptr;//前一个TB的起始地址
        uintptr_t jmp_rx = tc_ptr + offset;//前一个TB中BR指令的地址
        uintptr_t jmp_rw = jmp_rx - tcg_splitwx_diff;//tcg_splitwx_diff SW中为0
        tb_target_set_jmp_target(tc_ptr, jmp_rx, jmp_rw, addr);//架构相关的修改函数，后端为SW
    } else {//不支持直接跳转，则将当前tb的起始地址给tb->jmp_target_arg[n]
        tb->jmp_target_arg[n] = addr;
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
    //为什么要-1，因为BR指令是先更新PC再用PC+offset,所以偏移量要少一个
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
        modify_direct_addr(addr, jmp_rw, jmp_rx);//四条指令，因此要留四个NOP，这种情况和间接跳转就一样了。
    } else {
      tcg_debug_assert("tb_target");
    }
}
```

# TB块执行函数cpu_tb_exec()

```c
static inline void cpu_loop_exec_tb(CPUState *cpu, TranslationBlock *tb,
                                    TranslationBlock **last_tb, int *tb_exit)
{
    int32_t insns_left;

    trace_exec_tb(tb, tb->pc);
    tb = cpu_tb_exec(cpu, tb, tb_exit);
    if (*tb_exit != TB_EXIT_REQUESTED) {
        *last_tb = tb;
        return;
    }
    *last_tb = NULL;
    ...
}
```

```c
static inline TranslationBlock * QEMU_DISABLE_CFI
    cpu_tb_exec(CPUState *cpu, TranslationBlock *itb, int *tb_exit)
{
    uintptr_t ret;
    TranslationBlock *last_tb;
    qemu_log_mask_and_addr(CPU_LOG_EXEC, itb->pc,
                           "Trace %d: %p ["
                           TARGET_FMT_lx "/" TARGET_FMT_lx "/%#x] %s\n",
                           cpu->cpu_index, itb->tc.ptr,
                           itb->cs_base, itb->pc, itb->flags,
                           lookup_symbol(itb->pc));//打印日志信息，当前TB块的起始地址对应的guest地址
/*
    从给定的翻译块开始执行代码。如果翻译块已链接，则执行可以从给定的TB继续到连续的TB。
    只有在需要顶级循环的某些操作时，控制才能最终返回：要么控制必须传递给尚未直接链接的TB，要么需要处理异步事件（如中断）。
    返回：返回值是在尝试执行的最后一个tb的翻译时传递给相应tcg_gen_exit_tb（）的值。
    该值为零或指向该TB的4字节对齐指针，并在其两个最低有效位中结合了附加信息。附加信息编码如下：
    0，1：此TB和下一个TB之间的链接通过指定的TB索引（0或1）。
    也就是说，我们通过（相当于）“goto_TB＜index＞”离开了TB。主循环使用它来确定如何将刚刚执行的TB链接到下一个TB。
    2： 我们正在使用指令计数代码生成，但我们没有开始执行此TB，因为指令计数器在执行过程中会达到零。
    在这种情况下，返回的指针是我们即将执行的TB，调用方必须安排执行剩余的指令计数。
    3： 我们停止了，因为CPU的exit_request标志已设置（通常意味着需要处理中断）。返回的指针是我们注意到挂起的退出请求时即将执行的TB。

    如果底部两位指示通过索引退出，则CPU状态被正确同步并准备好执行下一TB（特别是客户PC是下一个要执行的地址）。
    否则，我们在TB开始之前就放弃了对它的执行，调用者必须通过使用我们返回的TB指针调用CPU的synchronize_from_TB（）方法来修复CPU状态
    （如果不存在synchronise_from.TB（），则返回到使用TB->pb调用CPU的set_pc方法）。

    注意，TCG目标可能使用与此默认值不同的tcg_qemu_tb_exec定义（它只调用tcg_target_qemu _prologue（）发出的prologue.code）
*/
    ret = tcg_qemu_tb_exec(env, tb_ptr);//goto_ptr会返回0
    last_tb = tcg_splitwx_to_rw((void *)(ret & ~TB_EXIT_MASK));//取高位的tb地址。
    *tb_exit = ret & TB_EXIT_MASK;//取低2位的附加信息
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
#define TB_EXIT_MASK      3
#define TB_EXIT_IDX0      0
#define TB_EXIT_IDX1      1
#define TB_EXIT_IDXMAX    1
#define TB_EXIT_REQUESTED 3
```

# Issue

- [x] n=0、1 是什么意思？TB通常是以跳转指令结束的， 而跳转指令其实只有两种选择: 向左或者向右, 两个 jump slot 代表两个跳转的目标， 也就是 goto_tb 0x0/0x1。

- [x] goto_to 0x0理论上跳到下一条指令，不需要修改为跳转TB，与tb_exit有关，last_tb和tb_exit是怎么确定的？由返回值ret确定。

- [x] 不支持直接跳转，还有偏移量超出范围这两种情况没有看。

- [ ] splitwx有空可以看一下，sw中是0。

- [x] s到tb是怎么赋值的？传参

- [x] 链表相关的成员变量，与页有关，tb_jmp_unlink还没看。

- [x] 动态翻译，翻译到跳转指令，条件跳转或者间接跳转要读寄存器什么的（跳转地址会有个计算结果的）不知道会跳转到哪里。因此TB得结束，然后执行完毕才能知道跳转到哪里。静态翻译就会把可能需要翻译的东西全给你翻译了。或者动态翻译完生成tb文件，下次接着用。

- [x] 变量对应关系，函数调用关系得做表格，不然容易忘记。

- [x] 为什么修改指令时要添加一条NOP指令?goto_tb为什么还需要翻译成一条jmp指令，有什么用，不是不执行吗？是不支持直接跳转时和 ldi指令一起实现跳转。为什么不写到else分支里？是可以写进去的。

- [ ] 为什么原始跳转地址一定是下一条呢，因为goto_tb就是预留的，确实是下一条。假如翻译直接跳转指令，地址都有了能不能知道去哪儿了？tb为什么还要停呢？

- [x] 带符号的偏移量，为什么要先左移两位，再符号扩展
  
  pc寄存器的内容（即地址）是按照字节寻址的，PC相对寻址的偏移量（是立即数）是按字(32)寻址的（类似PC+1这种用法，pc+1是按字寻址，pc+4是按字节寻址），所以需要disp*4。为什么要这么存储disp呢？因为低两位一定是0，所以编译存储/4，用的时候再\*4

- [ ] unsigned long类型转换成long类型溢出，变成-1

- [x] 不同分支，退出tb的返回值不一样？是不一样
  一个tb执行完后正常会返回一个tb和jump slot index混合，在翻译下一个tb时根据索引修改上一个tb

- [x] 前端看sw还是x86？x86

- [x] 为什么要记录偏移量，用的时候再计算还原成地址？高清说减少空间开销

- [ ] 什么时候会返回值ret会是0？除了nochain
