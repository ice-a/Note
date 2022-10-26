# 操作码

[INDEX_op_goto_tb](#jump)

INDEX_op_goto_ptr

# 指令参考

## 函数调用

### call \<ret\> \<params\> ptr

调用函数‘ptr’（指针类型）

\<ret> 可选32位或64位返回值

\<params>可选32位或64位参数

## 跳转/标签

### set_label $label

在当前程序点定义标签‘label’

### br $label

跳到标签

### brcond_i32/i64 t0, t1, cond, label

当t0和t1条件跳转

Conditional jump if t0 cond t1 is true. cond can be:
    TCG_COND_EQ
    TCG_COND_NE
    TCG_COND_LT / signed /
    TCG_COND_GE / signed /
    TCG_COND_LE / signed /
    TCG_COND_GT / signed /
    TCG_COND_LTU / unsigned /
    TCG_COND_GEU / unsigned /
    TCG_COND_LEU / unsigned /
    TCG_COND_GTU / unsigned /

## 算术

## QEMU 特殊指令

### exit_tb t0

退出当前TB并返回值t0（字类型）。

### goto_tb index<a id=‘jump’>goto_tb index</a>

退出当前TB，如果当前TB链接到了TB索引‘index’（常量），则跳转到该TB，否则执行下一条指令。只有索引0和1有效，每个tb的每个slot索引最多只能发出一次tcg_gen_goto_tb。

### lookup_and_goto_ptr tb_addr

查找TB地址（'TB_addr'），如果有效，跳转到它。如果无效，跳转到TCG epilogue尾声，返回exec循环。此操作是可选的。如果TCG后端未实现goto_ptr操作码，则发出此操作等同于发出exit_tb（0）。