# SW整数寄存器定义

>/usr/include/sw_64/regdef.h

```c
/* HUANGLM20161111 alpha --> sw_64 */
#ifndef __alpha_regdef_h__
#define __alpha_regdef_h__

#define v0	$0	/* function return value */ //保存函数返回值

#define t0	$1	/* temporary registers (caller-saved) */
#define t1	$2  //临时寄存器，不需要save/restore，直接使用即可
#define t2	$3
#define t3	$4
#define t4	$5
#define t5	$6
#define t6	$7
#define t7	$8

#define	s0	$9	/* saved-registers (callee-saved registers) */
#define	s1	$10 //寄存器，需要save/restore
#define	s2	$11
#define	s3	$12
#define	s4	$13
#define	s5	$14
#define	s6	$15
#define	fp	s6	/* frame-pointer (s6 in frame-less procedures) */

#define a0	$16	/* argument registers (caller-saved) */
#define a1	$17
#define a2	$18
#define a3	$19
#define a4	$20
#define a5	$21

#define t8	$22	/* more temps (caller-saved) */
#define t9	$23
#define t10	$24
#define t11	$25
#define ra	$26	/* return address register */
#define t12	$27

#define pv	t12	/* procedure-variable register */
#define AT	$at	/* assembler temporary */
#define gp	$29	/* global pointer */
#define sp	$30	/* stack pointer */
#define zero	$31	/* reads as zero, writes are noops */

#endif /* __sw_64_regdef_h__ */
```


# sw整数寄存器（ 32个）

| 寄存器名称 | 软件名称  | 使用                                                                                               |
| ---------- | --------- | -------------------------------------------------------------------------------------------------- |
| $0         | v0        | 用于表达式求值并保存整数函数结果，跨程序调用不保留。                                               |
| $1-8       | t0-t7     | 用于表达式求值的临时寄存器，跨程序调用不保留                                                       |
| $9-14      | s0-s5     | 存储寄存器，跨过程调用保留                                                                         |
| $15 or $fp | s6 or fp  | 必要时存储帧指针（帧指针指向栈的最开始），否则是存储寄存器。指向帧内固定地址的指针叫做帧指针（fp） |
| $16-21     | a0-a5     | 用于传递函数前6个整数类型的实际参数，跨程序调用不保留。                                            |
| $22-25     | t8-t11    | 用于表达式求值的临时寄存器，跨程序调用不保留。                                                     |
| $26        | ra        | 包含返回地址，跨过程调用保留。                                                                     |
| $27        | pv or t12 | 包含过程值并用于表达式求值，跨程序调用不保留。                                                     |
| $28 or $at | at        | 为汇编器预留的寄存器，跨程序调用不保留。                                                           |
| $29 or $gp | gp        | 保存全局指针，跨程序调用不保留。                                                                   |
| $30 or $sp | sp        | 保存堆栈指针，跨过程调用保留。堆栈指针（sp），指向堆栈的顶部                                       |
| $31           |  zero          |  总是0                                                                                                   |

# sw浮点寄存器（32个）

| 浮点寄存器 | 用法                                                                                                     |
| ---------- | -------------------------------------------------------------------------------------------------------- |
| \$f0-f1    | 用于保存浮点类型函数返回值（$f0）和实数类型的函数返回值（$f0保存实部，$f1保存虚部。）,跨程序调用不保留。 |
| $f2-f9     | 存储寄存器，跨过程调用保留。                                                                             |
| $f10-f15   | 用于表达式求值的临时寄存器，跨程序调用不保留。                                                           |
| $f16-f21   | 用于传递前6个单双精度浮点类型的实际参数，跨程序调用不保留                                                |
| $f22-f30   | 用于表达式求值的临时寄存器，跨程序调用不保留。                                                           |
| $f31       | 总是0f                                                                                                   |