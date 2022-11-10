# 一、TableGen使用

```shell
# TableGen工具路径
build/bin/llvm-tblgen

# 示例一：
# 打印寄存器名
./build/bin/llvm-tblgen lib/Target/Sw64/Sw64.td -print-enums -class=Register -I include/ -I lib/Target/Sw64/

TableGen的后端：
    -print-enums 打印类的所有记录名
        -class=Register 指定类名
    -print-records 打印.td文件中所有类和记录
```

# 二、TableGen作用

TableGen的目的是根据源文件(.td)中的信息生成复杂的输出文件(.inc)，源文件比输出文件更容易编码，也更容易维护和修改。输入文件中供TableGen处理的信息以声明式风格编码，信息包括类和记录。内部化的记录被传递到各种后端，后端从记录的子集中提取信息，并生成一个或多个输出文件。这些输出文件通常是C++的.inc文件，但也可以是后端开发人员需要的任何其他类型的文件。

不同的TableGen后端会生成不同的.inc文件，RegisterInfo是一个后端的例子，它为特定的目标机器生成寄存器文件信息(TargetGenRegisterInfo.inc)，供LLVM目标无关代码生成器使用。

下面是后端可以做的一些事情：

- 为特定目标机器生成寄存器文件信息；
- 为目标生成指令定义；
- 生成代码生成器匹配指令与中间表示(IR)节点的模式；
- 生成Clang语义属性标识符；
- 生成Clang抽象语法树(AST)声明节点定义；
- 生成ClangAST语句节点定义；

LLVM工程中会包含TableGen生成的.inc文件。

应用场景：当前TableGen生成的代码主要在前端clang(target independent code, 在[llvm build path]/tools/clang/include/clang/下)以及后端llvm(target dependent code, 在[llvm build path]/lib/Target/[arch]下)。

# 三、TableGen语法

参考[TableGen语法](./TableGen语法.md)

# 四、TableGen原理

TableGen代码包含两块: 对.td文件的处理, 在lib/TableGen/目录下, 包含lexer与parser, 负责解析TableGen的语法并转换为内部数据结构；输出cpp代码, 在utils/TableGen/目录下, 用于生成我们需要的cpp代码, 这块与llvm代码逻辑强相关, 基本上一个cpp文件对应一类信息。

源码分析参考[TableGen源码](TableGen源码.md)