# translator_loop()

```c
void translator_loop(const TranslatorOps *ops, DisasContextBase *db,
                     CPUState *cpu, TranslationBlock *tb, int max_insns)
{
    int bp_insn = 0;
    bool plugin_enabled;
#if defined(CONFIG_LLVM)
    CPUArchState *env = cpu->env_ptr;
#endif
    /* Initialize DisasContext */
    db->tb = tb;
    db->pc_first = tb->pc;
    db->pc_next = db->pc_first;
    db->is_jmp = DISAS_NEXT;
    db->num_insns = 0;
    db->max_insns = max_insns;
    db->singlestep_enabled = cpu->singlestep_enabled;

    ops->init_disas_context(db, cpu);
    tcg_debug_assert(db->is_jmp == DISAS_NEXT);  /* no early exit */

    /* Reset the temp count so that we can identify leaks */
    tcg_clear_temp_count();

    /* Start translating.  */
#if defined(CONFIG_LLVM)
    ops->gen_hotpatch(db, cpu, tb);//生成中间码hotpatch
#else
    gen_tb_start(tb);//gen_hotpatch中也有gen_tb_start，没有替代
#endif
    ops->tb_start(db, cpu);//i386 is empty
    tcg_debug_assert(db->is_jmp == DISAS_NEXT);  /* no early exit */

    plugin_enabled = plugin_gen_tb_start(cpu, tb,
                                         tb_cflags(db->tb) & CF_MEMI_ONLY);

    while (true) {
        db->num_insns++;
        ops->insn_start(db, cpu);
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
            gen_io_start();
            ops->translate_insn(db, cpu);
        } else {
            /* we should only see CF_MEMI_ONLY for io_recompile */
            tcg_debug_assert(!(tb_cflags(db->tb) & CF_MEMI_ONLY));
            ops->translate_insn(db, cpu);
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
#if defined(CONFIG_LLVM)
#if defined(CONFIG_USER_ONLY)
        if (llvm_has_annotation(tb->pc, ANNOTATION_LOOP))//是否有环
           break;
#endif
#if defined(TARGET_X86_64)  || defined(TARGET_I386)
	if (build_llvm(env) && db->num_insns == tb->icount) {
           db->is_jmp = DISAS_TOO_MANY;//已经把该tb的所有targetcode翻译完了
	   break;
        }
#endif
#if defined(TARGET_AARCH64)
        if (build_llvm(env) && db->num_insns == tb->icount)
            break;
#endif
#endif
        /* Stop translation if the output buffer is full,
           or we have executed all of the allowed instructions.  */
        if (tcg_op_buf_full() || db->num_insns >= db->max_insns) {
            db->is_jmp = DISAS_TOO_MANY;
#if defined(CONFIG_LLVM) && (defined(TARGET_X86_64) || defined(TARGET_I386))
	    // set dc->fallthrough, dc->gen_ibtc
            ops->set_fallthrough(db, 1);
            ops->set_gen_ibtc(db, 1);
            tb->jmp_pc[0] = db->pc_next;//buffer满了，分成两个tb
#endif
            break;
        }
    }//end while

#if defined(CONFIG_LLVM)
    ops->warn_exit(db, cpu, tb);
#endif

    /* Emit code to exit the TB, as indicated by db->is_jmp.  */
    ops->tb_stop(db, cpu);

#if defined(CONFIG_LLVM)
    if (!build_llvm(env)) {
        /* Terminate the linked list.  */
        gen_tb_end(db->tb, db->num_insns - bp_insn);//?一模一样
    }
#else
    gen_tb_end(db->tb, db->num_insns - bp_insn);
#endif

    if (plugin_enabled) {
        plugin_gen_tb_end(cpu);
    }

    /* The disas_log hook may use these values rather than recompute.  */
#if defined(CONFIG_LLVM)
    if (!build_llvm(env)) {
      tb->size = db->pc_next - db->pc_first;
      tb->icount = db->num_insns;//计算tb大小和target指令数量
    }
#else
    tb->size = db->pc_next - db->pc_first;
    tb->icount = db->num_insns;
#endif

#ifdef DEBUG_DISAS
    #if defined(CONFIG_LLVM)
    if (qemu_loglevel_mask(CPU_LOG_TB_IN_ASM)
        && qemu_log_in_addr_range(db->pc_first) &&  !build_llvm(env)) {
    #else
    if (qemu_loglevel_mask(CPU_LOG_TB_IN_ASM)
        && qemu_log_in_addr_range(db->pc_first)) {
    #endif
        FILE *logfile = qemu_log_lock();
        qemu_log("----------------\n");
        ops->disas_log(db, cpu);
        qemu_log("\n");
        qemu_log_unlock(logfile);
    }
#endif
}

```

# TranslatorOps

```c
#if defined(CONFIG_LLVM)
static void i386_tr_gen_hotpatch(DisasContextBase *dcbase, CPUState *cpu, TranslationBlock *tb)
{
    CPUArchState *env = cpu->env_ptr;
    if (!build_llvm(env)) {
        gen_tb_start(tb);//有了，所以代码逻辑可以那么写
        if (tracer_mode != TRANS_MODE_NONE)
            tcg_gen_hotpatch(IS_USER(dc), tracer_mode == TRANS_MODE_HYBRIDS ||
                                          tracer_mode == TRANS_MODE_HYBRIDM);
    }//IS_USER(dc)中系统模式i386和arm均有变量代表系统模式（cpl，user），sw没有定义。
}

static void i386_tr_set_fallthrough(DisasContextBase *dcbase, int n)
{
    DisasContext *dc = container_of(dcbase, DisasContext, base);
    dc->fallthrough = n;
}

static void i386_tr_set_gen_ibtc(DisasContextBase *dcbase, int n)
{
    DisasContext *dc = container_of(dcbase, DisasContext, base);
    dc->gen_ibtc = n;
}

static void i386_tr_warn_exit(DisasContextBase *dcbase, CPUState *cpu, TranslationBlock *tb)
{
   DisasContext *dc = container_of(dcbase, DisasContext, base);
   CPUArchState *env = cpu->env_ptr;

   if (build_llvm(env) && tb->size != dc->pc - tb->pc) {
        //pc_start=tb->pc
        /* consistency check with tb info. we must make sure
         * guest basic blocks are the same. skip this trace if inconsistent */
        fprintf(stderr, "inconsistent block with pc 0x"TARGET_FMT_lx" size=%d"
                " icount=%d (error size="TARGET_FMT_ld")\n",
                tb->pc, tb->size, tb->icount, dc->pc - tb->pc);
        exit(0);
    }
}
#endif
```

# tcg_out_op

```c
static void tcg_out_op(TCGContext *s, TCGOpcode opc, const TCGArg args[TCG_MAX_OP_ARGS],const int const_args[TCG_MAX_OP_ARGS])
{
    /* 99% of the time, we can signal the use of extension registers
       by looking to see if the opcode handles 64-bit data.  */
    TCGType ext = (tcg_op_defs[opc].flags & TCG_OPF_64BIT) != 0; 
    /* Hoist the loads of the most common arguments.  */
    TCGArg a0 = args[0];
    TCGArg a1 = args[1];
    TCGArg a2 = args[2];
    int c2 = const_args[2];

    /* Some operands are defined with "rZ" constraint, a register or
       the zero register.  These need not actually test args[I] == 0.  */
    #define REG0(I)  (const_args[I] ? TCG_REG_ZERO : (TCGReg)args[I])

    switch (opc) {
#if defined(CONFIG_LLVM)
    case INDEX_op_hotpatch:
       tcg_out_hotpatch(s, args[0], args[1]);
       break;
    case INDEX_op_jmp:
       if (const_args[0]) {
           tcg_out_goto(s, (tcg_insn_unit *)args[0]);
       } else {
	   tcg_out_insn_jump(s, OPC_JMP, TCG_REG_ZERO, args[0], noPara);
       }    
       break;
#endif
```

# tcg_out_hotpatch

```c
#if defined(CONFIG_LLVM)
/*
 * Emit trace profiling/prediction stubs. The code sequence is as following:
 *   S1: direct jump (the reloc part requires 4-byte alignment)
 *   S2: trace profiling stub 
 *   S3: trace prediction stub
 *   S4: beginning of QEMU emulation code
 *
 * The jump inst of S1 is initially set to jump to S3 (i.e. skipping S2).
 * Remember the offset of S3 (patch_next) which is used to turn the
 * trace profiling off. Also remember the offset of S4 (patch_skip)
 * so that the trace stubs can be skipped quickly while searching pc.
 */
 //占位、桩代码
static void tcg_out_hotpatch(TCGContext *s, uint32_t is_user, uint32_t emit_helper)
{//IS_USER(dc) tracer_mode == TRANS_MODE_HYBRIDS ||tracer_mode == TRANS_MODE_HYBRIDM
    tcg_insn_unit *label_ptr[2];
    TranslationBlock *tb = s->tb;
    uint32_t old;
    //tcg_out_mov(s, TCG_TYPE_PTR, tcg_target_call_iarg_regs[0], TCG_AREG0); //luo for debug
    //tcg_out_movi(s, TCG_TYPE_I32, tcg_target_call_iarg_regs[1], tb->id); //luo for debug
    //tcg_out_call(s, (tcg_insn_unit *)helper_printf_tb); //luo for debug
 
    tb->patch_jmp = (uint16_t)((intptr_t)s->code_ptr - (intptr_t)s->code_buf);//patch_jmp=offset of S1 

    /* S1: Direct Jump  */
    if (is_user == 0 || emit_helper == 0) {//系统模式？
        tcg_out_goto(s, s->code_ptr + 1); //br s->code_ptr + 1
	tb->patch_next = (uint16_t)(s->code_ptr - s->code_buf);//patch_next=offset of 下一条
        return;
    } 

    label_ptr[0] = s->code_ptr;//占位0
    tcg_out_insn_br(s, OPC_BR, TCG_REG_ZERO, 0);//br $31,0
    for(int i = 0; i < 8-1; i++) {//7条NOP
        tcg_out32(s, OPC_NOP);
    }

/*    old = *(uint32_t *)s->code_ptr;
    if (old == sextract64(old, 0, 21)) {
        tcg_out_insn_br(s, OPC_BR, TCG_REG_ZERO, old);
        for(int i = 0; i < 8-1; i++) {
            tcg_out32(s, OPC_NOP); 
        //gaoqing for:start_trace_profiling
        //why 8: tb_target_set_target1 max need 8 instructions; -1 because OPC_BR included in 8;
        }
    }else {
        num_insn = bigaddr_jmp(s, TCG_TYPE_I64, TCG_REG_TMP, old);
	tcg_out_insn_jump(s, OPC_RET, TCG_REG_ZERO, TCG_REG_TMP, noPara);
        for(int i = 0; i < (8-num_insn-1); i++) {
            tcg_out32(s, OPC_NOP); 
        }
        num_insn = 0;
    }
*/

    /* S2: Trace Profiling Stub  */
    tcg_out_mov(s, TCG_TYPE_PTR, tcg_target_call_iarg_regs[0], TCG_AREG0);//mov $9, $0 | BIS $0,$9,$0
    tcg_out_movi(s, TCG_TYPE_I32, tcg_target_call_iarg_regs[1], tb->id);//movi tb->id $1
    tcg_out_call(s, (tcg_insn_unit *)helper_NET_profile);//call helper_NET_profile

    /* S3: Trace Prediction stub */
    tb->patch_next = (uint16_t)((intptr_t)s->code_ptr - (intptr_t)s->code_buf);//patch_next=offset of S3

    tcg_out_ld(s, TCG_TYPE_I32, tcg_target_reg_alloc_order[0], 
	    TCG_AREG0, offsetof(CPUArchState, start_trace_prediction));//ld $0,&(env->start_trace_prediction)
    tcg_out_cond_cmp(s, 0, TCG_COND_EQ, TCG_REG_TMP, tcg_target_reg_alloc_order[0], 0, true);//CMPEQ $0,0,$27
    

    label_ptr[1] = s->code_ptr;//占位1
    tcg_out_insn_br(s, OPC_BGT, TCG_REG_TMP, 0);//br $27,0
    for(int i = 0; i < 8-1; i++) {
        tcg_out32(s, OPC_NOP); 
    }
						       
    /*old = *(uint32_t *)s->code_ptr;
    if (old == sextract64(old, 0, 21)) {
        tcg_out_insn_br(s, OPC_BGT, TCG_REG_TMP, old);
        for(int i = 0; i < 8-1; i++) {
            tcg_out32(s, OPC_NOP); 
        }
    }else {
        num_insn = bigaddr_jmp(s, TCG_TYPE_I64, TCG_REG_TMP2, old);
        tcg_out_insn_br(s, OPC_BR, TCG_REG_TMP3, 0);
        tcg_out_insn_complexReg(s, OPC_SELEQ, TCG_REG_TMP, TCG_REG_TMP2, TCG_REG_TMP3, TCG_REG_TMP2);
	tcg_out_insn_jump(s, OPC_RET, TCG_REG_ZERO, TCG_REG_TMP2, noPara);
        if (num_insn+3>8)
            exit("sw hotpatch");                                  
        for(int i = 0; i < (8-num_insn-3); i++) {
            tcg_out32(s, OPC_NOP); 
        }
    }*/

    tcg_out_mov(s, TCG_TYPE_PTR, tcg_target_call_iarg_regs[0], TCG_AREG0);//mov $9, $0 | BIS $0,$9,$0
    tcg_out_movi(s, TCG_TYPE_I32, tcg_target_call_iarg_regs[1], tb->id);//movi tb->id $1
    tcg_out_call(s, (tcg_insn_unit *)helper_NET_predict);//call helper_NET_predict
    reloc_pc21(label_ptr[0], s->code_ptr);
    reloc_pc21(label_ptr[1], s->code_ptr);
    }
#endif

#if defined(CONFIG_LLVM)
tcg_insn_unit *ibtc_ret_addr;
tcg_insn_unit *tb_ret_addr;
#else
static const tcg_insn_unit *tb_ret_addr;
#endif
```

# helper_function

```c
/* Helper function to perform trace profiling. */
void helper_NET_profile(CPUArchState *env, int id)
{
//    TranslationBlock *tb = tbs_find_id(id);
    //printf("helper_NET_profile 0x%x, exec_count = %d\n",tbs[id]->pc,tbs[id]->exec_count+1);
    auto &Tracer = getNETTracer(env);
    Tracer.Profile(tbs[id]);
//    Tracer.Profile(tb);
}

/* Helper function to perform trace prediction. */
void helper_NET_predict(CPUArchState *env, int id)
{
//    TranslationBlock *tb = tbs_find_id(id);
    //printf("helper_NET_predict 0x%x\n",tbs[id]->pc);
    auto &Tracer = getNETTracer(env);
    Tracer.Predict(tbs[id]);
//    Tracer.Predict(tb);
}
```

# LLVM_function

```c
void NETTracer::Profile(TranslationBlock *tb)
{
    if (Atomic<uint32_t>::inc_return(&tb->exec_count) != ProfileThreshold)//执行次数tb->exec_count没到50次，退出profile
        return;

#if 0
    /* If the execution is already in the prediction mode, process the
     * previously recorded trace. */
    if (Env->start_trace_prediction && !TBs.empty()) {
        OptimizeTrace(Env, TBs, -1);
        Reset();
    }
#endif

    /* We reach a profile threshold, stop trace profiling and start trace tail
     * prediction. The profiling is disabled by setting the jump directly to
     * trace prediction stub. */
    patch_jmp(tb_get_jmp_entry(tb), tb_get_jmp_next(tb));//执行次数到50次，重写patch_jmp位置跳转到patch_next位置，即S3。目的：此次之后该tb无需再进行profile而直接跳转到predict。
    Env->start_trace_prediction = 1;//开始predict
}

void NETTracer::Predict(TranslationBlock *tb)
{
    /* The trace prediction will terminate if a cyclic path is detected.
     * (i.e., current tb has existed in the tracing butter either in the
     * head or middle of the buffer.) */
    int LoopHeadIdx = -1;

#if defined(CONFIG_LLVM)
    /* Skip this trace if the next block is an annotated loop head and
     * is going to be included in the middle of a trace. */
    if (!TBs.empty() && TBs[0] != tb &&
        llvm_has_annotation(tb->pc, ANNOTATION_LOOP)) {
        goto trace_building;
    }
#endif

#if defined(USE_TRACETREE_ONLY)
    /* We would like to have a straight-line or O-shape trace.
     * (the 6-shape trace is excluded) */
    if (!TBs.empty() && tb == TBs[0]) {
        LoopHeadIdx = 0;
        goto trace_building;
    }
#elif defined(USE_RELAXED_NET)
    /* Find any cyclic path in recently recorded blocks. */
    for (int i = 0, e = TBs.size(); i != e; ++i) {
        if (tb == TBs[i]) {
            LoopHeadIdx = i;
            goto trace_building;
        }
    }
#else
    if (!TBs.empty()) {
        if (tb == TBs[0]) {
            /* Cyclic path. */
            LoopHeadIdx = 0;
            goto trace_building;
        }
        if (tb->pc <= TBs[TBs.size() - 1]->pc) {
            /* Backward branch. */
            goto trace_building;
        }
    }
#endif

    TBs.push_back(tb);

    /* Stop if the maximum prediction length is reached. */
    if (TBs.size() == PredictThreshold)
        goto trace_building;

    return;

trace_building:
    /* If the trace is a loop with a branch to the middle of the loop body,
     * we forms two sub-traces: (1) the loop starting from the loopback to
     * the end of the trace and (2) the original trace. */
    /* NOTE: We want to find more traces so the original trace is included. */

    if (LoopHeadIdx > 0) {
        /* Loopback at the middle. The sub-trace (1) is optimized first. */
        TBVec Loop(TBs.begin() + LoopHeadIdx, TBs.end());
        update_tb_mode(Loop[0], BLOCK_ACTIVE, BLOCK_TRACEHEAD);
        OptimizeTrace(Env, Loop, 0);
    }
    OptimizeTrace(Env, TBs, LoopHeadIdx);

    Reset();
}
```

# tb_target_set_jmp_target1

```c
#if defined(CONFIG_LLVM)
/* TCG_TARGET_HAS_direct_jump */
void tb_target_set_jmp_target1(uintptr_t tc_ptr, uintptr_t jmp_rx, uintptr_t jmp_rw, uintptr_t addr)
{
    tcg_insn_unit i1, i2;
    uint64_t pair;
    
    ptrdiff_t offset = addr - jmp_rx -1;
    
    if (offset == sextract64(offset, 0, 21)) {
        i1 = OPC_BR | (TCG_REG_ZERO & 0x1f) << 21| ((offset >> 2) & 0x1fffff);
	i2 = OPC_NOP;
        pair = (uint64_t)i2 << 32 | i1;
        qatomic_set((uint64_t *)jmp_rw, pair);
        flush_idcache_range(jmp_rx, jmp_rw, 8); //not support no sw_64
    } else {
        long l0=0, l1=0, l2=0, l3=0, extra=0;
        uintptr_t val = addr;
        TCGReg rs = TCG_REG_ZERO;
        TCGReg rd = TCG_REG_TMP;
        tcg_insn_unit insn[8];
        uint64_t index = 0;
        uint64_t i = 0;
        uintptr_t jmp = jmp_rw;

        l0 = (int16_t)val;
        val = (val - l0) >> 16;
        l1 = (int16_t)val;
        if ( addr >> 31 == -1 || addr >> 31 == 0 ) {
            if ( l1 < 0 && addr >= 0) {
                extra = 0x4000;
                l1 = (int16_t)(val - 0x4000);
            }
        } else {
            val = (val - l1) >> 16;
            l2 = (int16_t)val;
            val = (val - l2) >> 16;
            l3 = (int16_t)val;

            if (l3) {
                insn[index++] = OPC_LDIH | (rd & 0x1f) << 21 | (rs & 0x1f) << 16 | (l3 & 0xffff);
                rs = rd;
            }
            if (l2) {
                insn[index++] = OPC_LDI | (rd & 0x1f) << 21 | (rs & 0x1f) << 16 | (l2 & 0xffff);
                rs = rd;
            }
            if ( l3 || l2 )
                insn[index++] = OPC_SLL_I | (rd & 0x1f) << 21 | (32 & 0xff) << 13 | (rd & 0x1f);
       } 

       if (l1) {
           insn[index++] = OPC_LDIH | (rd & 0x1f) << 21 | (rs & 0x1f) << 16 | (l1 & 0xffff);
           rs = rd;
       } 

       if (extra) {
           insn[index++] = OPC_LDIH | (rd & 0x1f) << 21 | (rs & 0x1f) << 16 | (extra & 0xffff);
           rs = rd;
       }

       insn[index++] = OPC_LDI | (rd & 0x1f) << 21 | (rs & 0x1f) << 16 |(l0 & 0xffff);
       insn[index++] = OPC_RET | TCG_REG_ZERO << 21 | (rd & 0x1f) << 16;
    //At Now, index represent real number of instruction
     
       if ((index % 2) != 0) {
           insn[index++] = OPC_NOP;
       }
       while (i < index) {
           pair = (uint64_t)insn[i+1] << 32 | insn[i];
           qatomic_set((uint64_t *)jmp, pair);
           i = i + 2;
           jmp = jmp + 8;
       }
       flush_idcache_range(jmp_rx, jmp_rw, index*4); 
    }
}
#endif
```

```c
#if defined(CONFIG_LLVM)
/* helpers for LLVM */
void * const llvm_ld_helpers[16] = {
    [MO_UB]   = llvm_ret_ldub_mmu,
    [MO_LEUW] = llvm_le_lduw_mmu,
    [MO_LEUL] = llvm_le_ldul_mmu,
    [MO_LEQ]  = llvm_le_ldq_mmu,
    [MO_BEUW] = llvm_be_lduw_mmu,
    [MO_BEUL] = llvm_be_ldul_mmu,
    [MO_BEQ]  = llvm_be_ldq_mmu,
};
				    
void * const llvm_st_helpers[16] = {
    [MO_UB]   = llvm_ret_stb_mmu,
    [MO_LEUW] = llvm_le_stw_mmu,
    [MO_LEUL] = llvm_le_stl_mmu,
    [MO_LEQ]  = llvm_le_stq_mmu,
    [MO_BEUW] = llvm_be_stw_mmu,
    [MO_BEUL] = llvm_be_stl_mmu,
    [MO_BEQ]  = llvm_be_stq_mmu,
};

#endif
```

```c
#if defined(CONFIG_LLVM)
/*
 * Emit trace profiling/prediction stubs. The code sequence is as following:
 *   S1: direct jump (the reloc part requires 4-byte alignment)
 *   S2: trace profiling stub
 *   S3: trace prediction stub
 *   S4: beginning of QEMU emulation code
 *
 * The jump inst of S1 is initially set to jump to S3 (i.e. skipping S2).
 * Remember the offset of S3 (patch_next) which is used to turn the
 * trace profiling off. Also remember the offset of S4 (patch_skip)
 * so that the trace stubs can be skipped quickly while searching pc.
 */
static void tcg_out_hotpatch(TCGContext *s, uint32_t is_user, uint32_t emit_helper)
{
    tcg_insn_unit *label_ptr[2];
    TranslationBlock *tb = s->tb;
    uint32_t old;
    //tcg_out_mov(s, TCG_TYPE_PTR, tcg_target_call_iarg_regs[0], TCG_AREG0); //luo for debug
    //tcg_out_movi(s, TCG_TYPE_I32, tcg_target_call_iarg_regs[1], tb->id); //luo for debug
    //tcg_out_call(s, (tcg_insn_unit *)helper_printf_tb); //luo for debug
 
    tb->patch_jmp = (uint16_t)((intptr_t)s->code_ptr - (intptr_t)s->code_buf);

    /* S1: Direct Jump  */
    if (is_user == 0 || emit_helper == 0) { 
        tcg_out_goto(s, s->code_ptr + 1); 
	tb->patch_next = (uint16_t)(s->code_ptr - s->code_buf);
        return;
    } 

    label_ptr[0] = s->code_ptr;
    tcg_out_insn_br(s, OPC_BR, TCG_REG_ZERO, 0);
    for(int i = 0; i < 8-1; i++) {
        tcg_out32(s, OPC_NOP); 
    }

/*    old = *(uint32_t *)s->code_ptr;
    if (old == sextract64(old, 0, 21)) {
        tcg_out_insn_br(s, OPC_BR, TCG_REG_ZERO, old);
        for(int i = 0; i < 8-1; i++) {
            tcg_out32(s, OPC_NOP); 
        //gaoqing for:start_trace_profiling
        //why 8: tb_target_set_target1 max need 8 instructions; -1 because OPC_BR included in 8;
        }
    }else {
        num_insn = bigaddr_jmp(s, TCG_TYPE_I64, TCG_REG_TMP, old);
	tcg_out_insn_jump(s, OPC_RET, TCG_REG_ZERO, TCG_REG_TMP, noPara);
        for(int i = 0; i < (8-num_insn-1); i++) {
            tcg_out32(s, OPC_NOP); 
        }
        num_insn = 0;
    }
*/

    /* S2: Trace Profiling Stub  */
    tcg_out_mov(s, TCG_TYPE_PTR, tcg_target_call_iarg_regs[0], TCG_AREG0);
    tcg_out_movi(s, TCG_TYPE_I32, tcg_target_call_iarg_regs[1], tb->id);
    tcg_out_call(s, (tcg_insn_unit *)helper_NET_profile);

    /* S3: Trace Prediction stub */
    tb->patch_next = (uint16_t)((intptr_t)s->code_ptr - (intptr_t)s->code_buf);

    tcg_out_ld(s, TCG_TYPE_I32, tcg_target_reg_alloc_order[0], 
	    TCG_AREG0, offsetof(CPUArchState, start_trace_prediction));
    tcg_out_cond_cmp(s, 0, TCG_COND_EQ, TCG_REG_TMP, tcg_target_reg_alloc_order[0], 0, true);
    

    label_ptr[1] = s->code_ptr;
    tcg_out_insn_br(s, OPC_BGT, TCG_REG_TMP, 0);
    for(int i = 0; i < 8-1; i++) {
        tcg_out32(s, OPC_NOP); 
    }
						       
    /*old = *(uint32_t *)s->code_ptr;
    if (old == sextract64(old, 0, 21)) {
        tcg_out_insn_br(s, OPC_BGT, TCG_REG_TMP, old);
        for(int i = 0; i < 8-1; i++) {
            tcg_out32(s, OPC_NOP); 
        }
    }else {
        num_insn = bigaddr_jmp(s, TCG_TYPE_I64, TCG_REG_TMP2, old);
        tcg_out_insn_br(s, OPC_BR, TCG_REG_TMP3, 0);
        tcg_out_insn_complexReg(s, OPC_SELEQ, TCG_REG_TMP, TCG_REG_TMP2, TCG_REG_TMP3, TCG_REG_TMP2);
	tcg_out_insn_jump(s, OPC_RET, TCG_REG_ZERO, TCG_REG_TMP2, noPara);
        if (num_insn+3>8)
            exit("sw hotpatch");                                  
        for(int i = 0; i < (8-num_insn-3); i++) {
            tcg_out32(s, OPC_NOP); 
        }
    }*/

    tcg_out_mov(s, TCG_TYPE_PTR, tcg_target_call_iarg_regs[0], TCG_AREG0);
    tcg_out_movi(s, TCG_TYPE_I32, tcg_target_call_iarg_regs[1], tb->id);
    tcg_out_call(s, (tcg_insn_unit *)helper_NET_predict);
    reloc_pc21(label_ptr[0], s->code_ptr);
    reloc_pc21(label_ptr[1], s->code_ptr);
    }
#endif

#if defined(CONFIG_LLVM)
tcg_insn_unit *ibtc_ret_addr;
tcg_insn_unit *tb_ret_addr;
#else
static const tcg_insn_unit *tb_ret_addr;
#endif
```

```c
static TCGConstraintSetIndex tcg_target_op_def(TCGOpcode op)
{
    switch (op) {
#if defined(CONFIG_LLVM)
    case INDEX_op_jmp:
        return C_O0_I1(ri);
    case INDEX_op_hotpatch:
        return C_O1_I1(i, i);
#endif

#if defined(CONFIG_LLVM)
    #define DEF(name,a1,a2,a3,a4) { case INDEX_op_##name: return -1; }
    #include "tcg-opc-vector.h"
    #undef DEF
#endif
}
```

```c
#if defined(CONFIG_LLVM)
#define STACK_SIZE 0x800
#else
#define STACK_SIZE TCG_STATIC_CALL_ARGS_SIZE
#endif
```

```c
static void tcg_target_qemu_prologue(TCGContext *s) 
{
    TCGReg r;
    int ofs;
    
    /* allocate space for all saved registers */
    /* subl $sp,PUSH_SIZE,$sp */
    tcg_out_simple(s, OPC_SUBL_I, OPC_SUBL, TCG_REG_SP, TCG_REG_SP, PUSH_SIZE);
    
    /* Push (FP, LR)  */
    /* stl $fp,0($sp) */
    tcg_out_insn_ldst(s, OPC_STL, TCG_REG_FP, TCG_REG_SP, 0);
    /* stl $26,8($sp) */
    tcg_out_insn_ldst(s, OPC_STL, TCG_REG_RA, TCG_REG_SP, 8);


    /* Set up frame pointer for canonical unwinding.  */
    /* TCG_REG_FP=TCG_REG_SP */
    tcg_out_movr(s, TCG_TYPE_I64, TCG_REG_FP, TCG_REG_SP);

    /* Store callee-preserved regs x9..x14.  */
#ifdef CONFIG_REG_OPT
    tcg_out_insn_ldst(s, OPC_STL, TCG_REG_X9, TCG_REG_SP,
                                              (TCG_REG_X9 - TCG_REG_X9 + 2) * 8);
    tcg_out_insn_ldst(s, OPC_STL, TCG_REG_X14, TCG_REG_SP,
                                              (TCG_REG_X14 - TCG_REG_X9 + 2) * 8);
#else
    for (r = TCG_REG_X9; r <= TCG_REG_X14; r += 1){
        ofs = (r - TCG_REG_X9 + 2) * 8;
        tcg_out_insn_ldst(s, OPC_STL, r, TCG_REG_SP, ofs);
    }
#endif

    /* Make stack space for TCG locals.  */
    /* subl $sp,FRAME_SIZE-PUSH_SIZE,$sp */
    tcg_out_simple(s, OPC_SUBL_I, OPC_SUBL, TCG_REG_SP, TCG_REG_SP, FRAME_SIZE - PUSH_SIZE);

    /* Inform TCG about how to find TCG locals with register, offset, size.  */
    tcg_set_frame(s, TCG_REG_SP, TCG_STATIC_CALL_ARGS_SIZE,
                  CPU_TEMP_BUF_NLONGS * sizeof(long));

#if !defined(CONFIG_SOFTMMU)
    if (USE_GUEST_BASE) {
        tcg_out_movi(s, TCG_TYPE_PTR, TCG_REG_GUEST_BASE, guest_base);
        tcg_regset_set_reg(s->reserved_regs, TCG_REG_GUEST_BASE);
    }
#endif
    
    /* TCG_AREG0=tcg_target_call_iarg_regs[0], on sw, we mov $16 to $9 */
    tcg_out_mov(s, TCG_TYPE_I64, TCG_AREG0, tcg_target_call_iarg_regs[0]);
    tcg_out_insn_jump(s, OPC_JMP, TCG_REG_ZERO, tcg_target_call_iarg_regs[1], noPara);

    /*
     * Return path for goto_ptr. Set return value to 0, a-la exit_tb,
     * and fall through to the rest of the epilogue.
     */
#if defined(CONFIG_LLVM)
    ibtc_ret_addr = s->code_ptr;
#endif
    tcg_code_gen_epilogue = tcg_splitwx_to_rx(s->code_ptr);
    tcg_out_movi(s, TCG_TYPE_I64, TCG_REG_X0, 0);

    /* TB epilogue */
    tb_ret_addr = tcg_splitwx_to_rx(s->code_ptr);

    /* Remove TCG locals stack space.  */
    /* addl $sp,FRAME_SIZE-PUSH_SIZE,$sp */
    tcg_out_simple(s, OPC_ADDL_I, OPC_ADDL, TCG_REG_SP, TCG_REG_SP, FRAME_SIZE - PUSH_SIZE);

    /* Restore registers x9..x14.  */
#ifdef CONFIG_REG_OPT
    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_X9, TCG_REG_SP,
                                              (TCG_REG_X9 - TCG_REG_X9 + 2) * 8);
    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_X14, TCG_REG_SP,
                                              (TCG_REG_X14 - TCG_REG_X9 + 2) * 8);
#else
    for (r = TCG_REG_X9; r <= TCG_REG_X14; r += 1) {
        int ofs = (r - TCG_REG_X9 + 2) * 8;
        tcg_out_insn_ldst(s, OPC_LDL, r, TCG_REG_SP, ofs);
    }
#endif
    
    /* Pop (FP, LR) */
    /* ldl $fp,0($sp) */
    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_FP, TCG_REG_SP, 0);
    /* ldl $26,8($sp) */
    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_RA, TCG_REG_SP, 8);
    
    /* restore SP to previous frame. */
    /* addl $sp,PUSH_SIZE,$sp */
    tcg_out_simple(s, OPC_ADDL_I, OPC_ADDL, TCG_REG_SP, TCG_REG_SP, PUSH_SIZE);
    
    tcg_out_insn_jump(s, OPC_RET, TCG_REG_ZERO, TCG_REG_RA, noPara);
#ifdef CONFIG_LOOKUP_JUMP_CACHE
    /* Lookup tb_jmp_cache */
    tb_jmp_cache_addr = tcg_splitwx_to_rx(s->code_ptr);

    /* hash = tb_jmp_cache_hash_func(pc) */
    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_TMP, TCG_AREG0, offsetof(CPUX86State, segs[R_CS]) + offsetof(struct SegmentCache, base));//tmp=cs_base=env->segs[R_CS].base

    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_TMP2, TCG_AREG0, offsetof(CPUX86State, eip)); //tmp2=env->eip

    tcg_out_insn_bitReg(s, OPC_ADDL, TCG_REG_TMP, TCG_REG_TMP, TCG_REG_TMP2);//tmp=pc=cs_base+env->eip

    tcg_out_insn_simpleImm(s, OPC_SRL_I, TCG_REG_TMP2, TCG_REG_TMP, TB_JMP_CACHE_BITS); //tmp2=pc>>12

    tcg_out_insn_bitReg(s, OPC_XOR, TCG_REG_TMP3, TCG_REG_TMP, TCG_REG_TMP2); //tmp3 = pc^(pc>>12)

    tcg_out_movi(s, TCG_TYPE_I64, TCG_REG_TMP2, TB_JMP_CACHE_SIZE-1);  //tmp2=(0x1000-1)=0xfff=4095

    tcg_out_insn_bitReg(s, OPC_AND, TCG_REG_TMP2, TCG_REG_TMP2, TCG_REG_TMP3); //tmp2=(pc^(pc>>12)) & 0xfff

    /* tb = qatomic_rcu_read(&cpu->tb_jmp_cache[hash])  warning*/
    tcg_out_insn_simpleImm(s, OPC_MULL_I, TCG_REG_TMP2, TCG_REG_TMP2, sizeof(TranslationBlock *)); //tmp2=(&tb_jmp_cache[hash]-&tb_jmp_cache[0])

    tcg_out_movi(s, TCG_TYPE_I64,  TCG_REG_TMP3, offsetof(CPUState, tb_jmp_cache) - offsetof(ArchCPU, env));//tmp3=&tb_jmp_cache[0]-&env)

    tcg_out_insn_simpleReg(s, OPC_ADDL, TCG_REG_TMP2, TCG_REG_TMP2, TCG_REG_TMP3); //tmp2=&tb_jmp_cache[hash]-&env)

    tcg_out_insn_simpleReg(s, OPC_ADDL, TCG_REG_TMP2, TCG_AREG0, TCG_REG_TMP2); //tmp2=&cpu->tb_jmp_cache[hash]

    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_TMP3, TCG_REG_TMP2, 0);//tmp3=tb

    //tcg_out_mb(s);
    /* if tb == NULL */
    tcg_out_insn_br(s, OPC_BGT, TCG_REG_TMP3, 1);

    tcg_out_insn_jump(s, OPC_RET, TCG_REG_ZERO, TCG_REG_RA, noPara); //tb=null, return

    /* if tb->pc == pc */
    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_TMP2, TCG_REG_TMP3, offsetof(TranslationBlock, pc)); //tb!=null tmp2=tb->pc

    tcg_out_insn_simpleReg(s, OPC_CMPEQ, TCG_REG_TMP, TCG_REG_TMP2, TCG_REG_TMP);//tb->pc == pc,tmp=1,else tmp=0

    tcg_out_insn_br(s, OPC_BGT, TCG_REG_TMP, 1);

    tcg_out_insn_jump(s, OPC_RET, TCG_REG_ZERO, TCG_REG_RA, noPara); //tb->pc != pc, return
    
    /* tb->cs_base == cs_base */
    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_TMP, TCG_AREG0, offsetof(CPUX86State, segs[R_CS]) + offsetof(struct SegmentCache, base));//tmp=cs_base=env->segs[R_CS].base

    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_TMP2, TCG_REG_TMP3, offsetof(TranslationBlock,cs_base));//tmp2=tb->cs_base

    tcg_out_insn_simpleReg(s, OPC_CMPEQ, TCG_REG_TMP, TCG_REG_TMP2, TCG_REG_TMP);

    tcg_out_insn_br(s, OPC_BGT, TCG_REG_TMP, 1);

    tcg_out_insn_jump(s, OPC_RET, TCG_REG_ZERO, TCG_REG_RA, noPara);
    
    /* if tb->flags == flags */
    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_TMP, TCG_AREG0, offsetof(CPUX86State, eflags));//tmp=env->eflags

    tcg_out_movi(s, TCG_TYPE_I64, TCG_REG_TMP2, IOPL_MASK | TF_MASK | RF_MASK | VM_MASK | AC_MASK);//tmp2=(IOPL_MASK | TF_MASK | RF_MASK | VM_MASK | AC_MASK)

    tcg_out_insn_bitReg(s, OPC_AND, TCG_REG_TMP, TCG_REG_TMP, TCG_REG_TMP2);//tmp=env->eflags & (IOPL_MASK | TF_MASK | RF_MASK | VM_MASK | AC_MASK)

    tcg_out_insn_ldst(s, OPC_LDW, TCG_REG_TMP2, TCG_AREG0, offsetof(CPUX86State, hflags));//tmp2=env->hflags

    tcg_out_insn_bitReg(s, OPC_BIS, TCG_REG_TMP, TCG_REG_TMP, TCG_REG_TMP2);//tmp=flags=env->hflags|(env->eflags & (IOPL_MASK | TF_MASK | RF_MASK | VM_MASK | AC_MASK))

    tcg_out_insn_ldst(s, OPC_LDW, TCG_REG_TMP2, TCG_REG_TMP3, offsetof(TranslationBlock,flags));//tmp2=tb->flags

    tcg_out_insn_simpleReg(s, OPC_CMPEQ, TCG_REG_TMP, TCG_REG_TMP2, TCG_REG_TMP);

    tcg_out_insn_br(s, OPC_BGT, TCG_REG_TMP, 1);

    tcg_out_insn_jump(s, OPC_RET, TCG_REG_ZERO, TCG_REG_RA, noPara);
    
    /* tb->trace_vcpu_dstate == *cpu->trace_dstate */
/*    tcg_out_simple(s, OPC_SUBL_I, OPC_SUBL, TCG_REG_TMP, TCG_AREG0, offsetof(ArchCPU, env) - offsetof(CPUState, trace_dstate));

    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_TMP, TCG_REG_TMP, 0);//tmp=cpu->trace_dstate

    tcg_out_insn_ldst(s, OPC_LDW, TCG_REG_TMP2, TCG_REG_TMP3, offsetof(TranslationBlock, trace_vcpu_dstate));//tmp2=tb->trace_vcpu_dstate

    tcg_out_insn_simpleReg(s, OPC_CMPEQ, TCG_REG_TMP, TCG_REG_TMP2, TCG_REG_TMP);

    tcg_out_insn_br(s, OPC_BGT, TCG_REG_TMP, 1);

    tcg_out_insn_jump(s, OPC_RET, TCG_REG_ZERO, TCG_REG_RA, noPara);*/
    
    /* tb_cflags(tb) == cflags warning: qatomic_read(&tb->cflags) */
/*    tcg_out_simple(s, OPC_SUBL_I, OPC_SUBL, TCG_REG_TMP, TCG_AREG0, offsetof(ArchCPU, env) - offsetof(CPUState, tcg_cflags));

    tcg_out_insn_ldst(s, OPC_LDW, TCG_REG_TMP, TCG_REG_TMP, 0);//tmp=cflags=cpu->tcg_cflags

    tcg_out_insn_ldst(s, OPC_LDW, TCG_REG_TMP2, TCG_REG_TMP3, offsetof(TranslationBlock, cflags));

    tcg_out_insn_simpleReg(s, OPC_CMPEQ, TCG_REG_TMP, TCG_REG_TMP2, TCG_REG_TMP);

    tcg_out_insn_br(s, OPC_BGT, TCG_REG_TMP, 1);

    tcg_out_insn_jump(s, OPC_RET, TCG_REG_ZERO, TCG_REG_RA, noPara);*/
   
    /* tb->tc.ptr */
    tcg_out_insn_ldst(s, OPC_LDL, TCG_REG_TMP, TCG_REG_TMP3, offsetof(TranslationBlock, tc)+offsetof(struct tb_tc, ptr)); //tb->pc == pc, tmp=tb->tc.ptr

    tcg_out_insn_jump(s, OPC_JMP, TCG_REG_ZERO, TCG_REG_TMP, noPara); //jmp to tb->tc.ptr, tmp=$27

#endif
}

```

## llvm-state.h

```c
/*
 * The following data structure and routine are used to save/restore the states
 * of CPUArchState. Only the states that could affect decoding the guest binary by
 * the TCG front-end are saved/restored. Such states are saved when translating
 * the block at the first time because the states could change later and are
 * restored to the saved values when the block is decoded again during the
 * trace formation.
 */
/*以下数据结构和例程用于保存/恢复CPUArchState的状态。仅保存/恢复可能影响TCG前端解码guest二进制文件的状态。这样的状态在第一次翻译块时被保存，因为这些状态可能稍后改变。*/
#if defined(TARGET_I386) || defined(TARGET_X86_64)
typedef struct i386_env {
    int singlestep_enabled;
    uint32_t hflags;
    target_ulong eflags;
} cpustate;
```

# ISSUE

swllvm9/llvm/lib/Support/Triple.cpp sw_64