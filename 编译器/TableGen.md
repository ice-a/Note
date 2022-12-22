# 一、TableGen语法

参考文档：[TableGen Programmer’s Reference — LLVM 15.0.0git documentation](https://llvm.org/docs/TableGen/ProgRef.html?highlight=tablegen#let-override-fields-in-classes-or-records "TableGen Programmer’s Reference — LLVM 15.0.0git documentation")

## 本文中描述格式的标记符号含义

自己理解的，不一定准确。

| 符号      | 说明              |
| ------- | --------------- |
| `::=`   | 等价于             |
| 双引号`""` | 字符串。            |
| 中括号`[]` | 可选项。            |
| 竖线 \|   | 互斥项的分隔符。 只能选择一项 |
| 省略号 `…` | 表示范围。           |
| 星号`*`   | 表示0个或多个         |
| 加号`+`   | 表示1或者多个         |

## 1. 标识符

标识符格式：

```shell
ualpha        ::=  "a"..."z" | "A"..."Z" | "_"               # 大小写字母或下划线    
TokIdentifier ::=  ("0"..."9")* ualpha (ualpha | "0"..."9")* # 标识符可以以整数开头
TokVarName    ::=  "$" ualpha (ualpha |  "0"..."9")*         # dag中的name
```

注意，与大多数语言不同，TableGen允许`TokIdentifier`以整数开头。在出现歧义的情况下，标记被解释为数字字面量，而不是标识符。

### 保留关键字

```cpp
assert     bit           bits          class         code
dag        def           else          false         foreach
defm       defset        defvar        field         if
in         include       int           let           list
multiclass string        then          true
```

警告：`field`保留字已弃用，只有在`CodeEmitterGen`后端, 它被用来区分普通记录字段和编码字段。

## 2. include关键字

TableGen支持文件包含。包含的文件在包含的位置直接展开，然后被解析。

```shell
IncludeDirective ::=  "include" TokString
```

主文件和被包含文件的部分可以使用预处理指令进行条件化。

```shell
PreprocessorDirective ::=  "#define" | "#ifdef" | "#ifndef"
```

## 3. bang运算符

```shell
BangOperator ::=  one of
                  !add        !and         !cast        !con         !dag
                  !empty      !eq          !filter      !find        !foldl
                  !foreach    !ge          !getdagop    !gt          !head
                  !if         !interleave  !isa         !le          !listconcat
                  !listsplat  !lt          !mul         !ne          !not
                  !or         !setdagop    !shl         !size        !sra
                  !srl        !strconcat   !sub         !subst       !substr
                  !tail       !xor
```

`!cond`与其他bang操作符语法略有不同，因此它是单独定义的:

```shell
CondOperator ::=  !cond
```

各bang操作符的信息请参阅 [Appendix A: Bang Operators](https://llvm.org/docs/TableGen/ProgRef.html?highlight=tablegen#appendix-a-bang-operators "Appendix A: Bang Operators")

## 4. 数据类型

```shell
Type    ::=  "bit" | "int" | "string" | "dag"  
            | "bits" "<" TokInteger ">"
            | "list" "<" Type ">"
            | ClassID
ClassID ::=  TokIdentifier
```

- `bit`：位类型。表示一个0或1的布尔值。

- `int`：整型。表示64位整数。

- `string`：字符串类型。表示任意长度的有序字符序列。

- `bits<n>`：位类型。表示位宽固定为`n`的整数，`n`可以为任意值。这`n`个位可以单独访问。这种类型的字段用于表示指令操作代码、寄存器号或地址模式/寄存器/位移。字段的位可以单独设置，也可以作为子字段设置。例如，在指令地址中，寻址模式、基址寄存器号和位移可以分别设置。

- `list<type>`：列表。表示一个列表，其元素为`<>`中指定的类型`type`。元素类型是任意的；它甚至可以是另一种列表类型。列表元素索引下标从0开始。 

- `dag`：有向无环图。表示节点组成的嵌套的**有向无环图(DAG)**。每个节点有一个操作符和零个或多个参数(或操作数)。参数可以是另一个dag对象，允许任意的节点和边树。例如，DAG用于表示代码生成器指令选择算法使用的代码模式。详见[有向无环图(dag)](https://llvm.org/docs/TableGen/ProgRef.html?highlight=tablegen#directed-acyclic-graphs-dags "有向无环图(dag)")

- `ClassID`：类。在类型上下文中指定类名表示定义值的类型必须是指定类的子类。这与列表类型结合在一起很有用;例如，将列表的元素约束为一个公共基类(例如，a list只能包含从Register类派生的定义)。ClassID必须为以前声明或定义过的类名。

### 4.1 有向无环图（DAG）

有向无环图可以在TableGen中使用`dag`数据类型直接表示。DAG节点由一个操作符和零个或多个参数(或操作数)组成。每个参数可以是任何想要的类型。通过使用另一个DAG节点作为参数，可以构建任意的DAG节点图。

dag实例的语法如下:

                              (操作符 参数1，参数2，…)

操作符必须呈现并且必须有记录。可以有零个或多个参数，用逗号分隔。操作符和参数可以有三种格式。

| 格式         | 说明       |
| ---------- | -------- |
| value      | 参数值      |
| value:name | 参数值和关联名称 |
| name       | 未初始化的参数名 |

其中`value`可以是任何TableGen值。如果`name`存在，则必须是一个以美元符号$开头的`TokVarName`。`name`的作用是在DAG中标记具有特定含义的操作符或参数，或者将一个DAG中的参数与另一个DAG中的同名参数关联起来。

## 5. 字面量，值

### 5.1 字面量

#### 5.1.1 整数字面量

```shell
TokInteger     ::=  DecimalInteger | HexInteger | BinInteger  # 整型
DecimalInteger ::=  ["+" | "-"] ("0"..."9")+                  # 十进制
HexInteger     ::=  "0x" ("0"..."9" | "a"..."f" | "A"..."F")+ # 十六进制
BinInteger     ::=  "0b" ("0" | "1")+                         # 二进制
```

注意， `DecimalInteger`标记包含可选的`+`或`-`符号，这与大多数语言不同，在这些语言中，符号被视为一元操作符。

#### 5.1.2 字符串字面量

```shell
TokString ::=  '"' (non-'"' characters and escapes) '"'      # 字符串中不能有双引号
TokCode   ::=  "[{" (shortest text not containing "}]") "}]" #  字符串中不能有括号
```

`TokCode`是由`[{`和`}]`分隔的多行字符串文字。它可以跨行换行，换行符保留在字符串中。

以下转义字符:

```cpp
\\ \' \" \t \n
```

### 5.2 值

#### 5.2.1 简单值

```shell
Value       ::=  SimpleValue ValueSuffix* # 值='简单值'+'后缀''
                | Value "#" [Value]
ValueSuffix ::=  "{" RangeList "}"
                | "[" RangeList "]"
                | "." TokIdentifier
RangeList   ::=  RangePiece ("," RangePiece)*   # 范围列表
RangePiece  ::=  TokInteger                     # 范围块
                | TokInteger "..." TokInteger
                | TokInteger "-" TokInteger
                | TokInteger TokInteger   # ?
```

`SimpleValue`有很多形式：

- 形式一：
  
  ```shell
  SimpleValue ::=  TokInteger | TokString+ | TokCode
  ```
  
  值可以是整数字面量、字符串字面量或编码字面量。

- 形式二：
  
  ```shell
  SimpleValue2 ::=  "true" | "false"
  ```
  
   true和false字面量本质上是整数1和0的语法糖。解析时，这些字面量将转换为整数。

- 形式三：
  
  ```shell
  SimpleValue3 ::=  "?"
  ```
  
  问号表示未初始化

- 形式四：
  
  ```shell
  SimpleValue4 ::=  "{" [ValueList] "}"
  ValueList    ::=  ValueListNE
  ValueListNE  ::=  Value ("," Value)*
  ```
  
  表示一个位序列，可用于初始化bits<n>类型字段。

- 形式五：
  
  ```shell
  SimpleValue5 ::=  "[" ValueList "]" ["<" Type ">"]
  ```
  
  表示一个列表初始化器(注意括号)。括号中的值是列表的元素。可选的Type可用于指示特定的元素类型;否则从给定值推断元素类型。TableGen通常可以推断类型，但值为空列表([ ])时不能。

- 形式六：
  
  ```shell
  SimpleValue6 ::=  "(" DagArg [DagArgList] ")"
  DagArgList   ::=  DagArg ("," DagArg)*
  DagArg       ::=  Value [":" TokVarName] | TokVarName
  ```
  
  表示一个DAG初始化器(注意括号)。第一个DagArg被称为DAG的“操作符”，而且必须是一条记录。详见有[向无环图(DAG)](https://llvm.org/docs/TableGen/ProgRef.html?highlight=tablegen#directed-acyclic-graphs-dags "向无环图(DAG)")。

- 形式七：
  
  ```shell
  SimpleValue7 ::=  TokIdentifier
  ```
  
  得到的值是由标识符命名的实体的值。

- 形式八：
  
  ```shell
  SimpleValue8 ::=  ClassID "<" ValueListNE ">"
  ```
  
  这个形式创建了一个新的匿名类(就像使用未命名def从给定的类及模板参数创建记录那样;参见def），值就是那条记录。记录的一个字段可以使用后缀获得。参见[Suffixed Values](https://llvm.org/docs/TableGen/ProgRef.html?highlight=tablegen#suffixed-values "Suffixed Values")。
  
  以这种方式调用类可以提供简单的子例程功能。有关更多信息，请参见[将类用作子例程](https://llvm.org/docs/TableGen/ProgRef.html?highlight=tablegen#using-classes-as-subroutines "将类用作子例程")。 

- 形式九：
  
  ```shell
  SimpleValue9 ::=  BangOperator ["<" Type ">"] "(" ValueListNE ")"
                   | CondOperator "(" CondClause ("," CondClause)* ")"
  CondClause   ::=  Value ":" Value
  ```
  
  bang运算符提供其他简单值不可用的函数。除了!cond的情况，bang操作符接受圆括号中包含的参数列表，并使用这些参数执行一些函数，为bang操作符生成一个值。cond操作符接受以冒号分隔的参数对列表。参见[附录A:bang操作符](https://llvm.org/docs/TableGen/ProgRef.html?highlight=tablegen#appendix-a-bang-operators "附录A:bang操作符")以了解每个bang操作符的描述。

#### 5.2.2 值的后缀

- `value{17}`：表示整数value的第17位

- `value{8...15}`：整数value的第8到第15位的值

- `value[4]`：表示列表value的第4个元素。方括号充当列表的下标操作符。只有在指定单个元素时才会出现这种情况。

- `value[4...7,17,2...3,4]`：表示一个新列表，它是列表value的一个切片。新列表包含元素4、5、6、7、17、2、3和4。元素可以以任意顺序被包含多次。这是指定多个元素时的结果。

- `value.field`：指定记录value中指定字段field的值。

#### 5.2.3 粘贴操作符

粘贴操作符`#`是TableGen表达式中唯一可用的中缀操作符。它允许连接字符串或列表，但有一些特别的特性。

粘贴操作符可以在`def`或`defm`语句中指定记录名称时使用，在该场景下，它必须构造一个字符串。如果操作数是未定义的名称`TokIdentifier`或全局`Defvar`或`Defset`的名称，则将其视为字符串逐字处理。不使用全局名称的值。

粘贴操作符可以在所有其他值表达式中使用，在这些情况下，它可以构造字符串或列表。很奇怪，但与前面的情况一致，如果右边的操作数是一个未定义的名称或全局名称，则将其视为字符串逐字处理。左边的操作数被正常处理。

值可以有一个尾随的粘贴操作符，在这种情况下，左边的操作数连接到一个空字符串。

## 6. 声明

以下语句可能出现在TableGen源文件的顶层。

```shell
TableGenFile ::=  (Statement | IncludeDirective
                 | PreprocessorDirective)*
Statement    ::=  Assert | Class | Def | Defm | Defset | Defvar
                 | Foreach | If | Let | MultiClass
```

### 6.1 class -- 定义一个抽象记录类

`class`语句定义了一个抽象的记录类，其他的类和记录可以**继承**这个类。

```shell
Class           ::=  "class" ClassID [TemplateArgList] RecordBody   # 模板类
TemplateArgList ::=  "<" TemplateArgDecl ("," TemplateArgDecl)* ">" # 模板参数列表
TemplateArgDecl ::=  Type TokIdentifier ["=" Value]                 # 模板参数声明及[初始化]
```

类可以通过一组**模板参数**来初始化，这些参数的值可以在类的记录体中使用。每次类被另一个类或记录继承时，都会指定这些模板参数。

如果一个模板参数没有用`=`赋予一个默认值 ，那么它就是未初始化的，必须在继承该类时在模板参数列表中指定(必需的参数)。如果为参数分配了默认值，则不需要在参数列表中指定(可选参数)。在声明中，所有必需的模板参数必须位于任何可选参数之前。模板参数默认值从左到右计算。

下面定义了RecordBody。它可以包括当前类继承的父类列表，以及字段定义和其他语句。当一个类C从另一个类D继承时，D的字段被有效地合并到C的字段中。

一个给定的类只能定义一次。如果下列任何一个为真(RecordBody元素在下面描述)，则认为`class`语句定义了类。

- TemplateArgList已经存在，或者
- RecordBody中的ParentClassList存在，或者
- RecordBody中的主体存在并且不是空的。

你可以通过指定一个空的TemplateArgList和一个空的RecordBody来声明一个空的类。这可以作为一种受限制的前向声明形式。注意，从向前声明的类派生的记录不会从它继承字段，因为这些记录是在解析它们的声明时构建的，并且因此是在最终定义类之前构建的。

每个类都有一个名为NAME(大写)的隐式模板参数，该参数绑定到继承该类的`Def`或`Defm`的名称。如果类被匿名记录继承，则名称不指定，但全局唯一。

#### 6.1.1 记录体

类和记录定义中都包含记录体。记录体可以包括**父类列表**，该列表指定当前类或记录从哪些类**继承**字段。这样的类被称为类或记录的**父类**。记录体还包括**定义的主体**，其中包含类或记录的字段的说明。

```shell
RecordBody        ::=  ParentClassList Body      # 记录=父类列表+定义体
ParentClassList   ::=  [":" ParentClassListNE]   # 父类列表
ParentClassListNE ::=  ClassRef ("," ClassRef)*  
ClassRef          ::=  (ClassID | MultiClassID) ["<" [ValueList] ">"] 
```

包含MultiClassID的ParentClassList只在defm语句的类列表中才有效。在这种情况下，ID必须是一个多类的名称。

```shell
Body     ::=  ";" | "{" BodyItem* "}"
BodyItem ::=  (Type | "code") TokIdentifier ["=" Value] ";"
             | "let" TokIdentifier ["{" RangeList "}"] "=" Value ";"
             | "defvar" TokIdentifier "=" Value ";"
             | Assert
```

主体中的字段定义指定了包含在类或记录中的字段。如果没有指定初始值，则该字段的值是未初始化的。字段类型必须指定；TableGen不会从值推断类型。关键字`code`可以用来强调字段有一个字符串类型的值，这个字符串值就是编码（code）。

let用于将字段重置为新值。它可以用于直接在主体中定义的字段或从父类继承的字段。一个`RangeList`可以被指定重置bit<n>字段。

defvar形式定义了一个变量，它的值可以用在记录体中的其他值表达式中。变量不是字段:它不会成为正在定义的类或记录的字段。在处理记录体时，提供了用于保存临时值的变量。详情请参阅[记录体中的Defvar](https://llvm.org/docs/TableGen/ProgRef.html?highlight=tablegen#defvar-in-a-record-body "记录体中的Defvar")。

当类C2继承类C1时，它获得了C1的所有字段定义。当这些定义合并到类C2中时，C2传递给C1的任何模板参数都被替换到定义中。换句话说，C1定义的抽象记录字段在合并到C2之前用模板参数展开。

### 6.2 def -- 定义一个具体记录

`def`语句定义了一个新的具体记录。

```shell
Def       ::=  "def" [NameValue] RecordBody
NameValue ::=  Value (parsed in a special mode)
```

NameValue是可选的。如果指定，将以特殊模式解析它，其中未定义(无法识别)标识符将被解释为文字字符串。特别是，全局标识符被认为是不可识别的。其中包括defvar和defset定义的全局变量。记录名称可以是空字符串。

如果没有给出名称值，则记录是匿名的。匿名记录的最终名称未指定，但全局惟一。

如果def出现在多类语句中，就会做特殊处理。有关详细信息，请参阅下面的多类部分。

通过在记录主体的开头指定ParentClassList子句，一条记录可以继承一个或多个类。父类中的所有字段都被添加到这条记录中。如果两个或多个父类提供相同的字段，记录使用最后一个父类的该字段值。

### 6.3 let -- 重写类或者记录的字段

`let`语句收集一组字段值(有时称为绑定)，并将它们应用于`let`语句范围内定义的所有类和记录。

```shell
Let     ::=   "let" LetList "in" "{" Statement* "}"
            | "let" LetList "in" Statement
LetList ::=  LetItem ("," LetItem)*
LetItem ::=  TokIdentifier ["<" RangeList ">"] "=" Value
```

`let`语句建立一个作用域，它是**带大括号的语句序列**或**不带大括号的单个语句**。`LetList`中的绑定应用于该作用域中的语句。

`LetList`中的字段名称必须为语句中定义的类和记录继承的类中的字段命名。在记录从它们的父类继承所有字段之后，字段值被应用到类和记录。因此let的作用是重写继承的字段值。`let`不能重写模板参数的值。

注意，顶层`let`不会覆盖在类或记录本身定义的字段

### 6.4 multiclass --定义多个记录

虽然带有模板参数的类是提取多个记录之间的共性的好方法，但多类可以方便的一次定义多个记录。例如，考虑一个3地址指令体系结构，它的指令有两种格式:reg = reg op reg和reg = reg op imm(例如SPARC)。我们希望在一个地方指定这两种常见格式存在，然后在另一个地方指定所有操作是怎样的。`multiclass`和`defm`语句实现了这一目标。您可以将多类看作是展开为多个记录的宏或模板。

```shell
MultiClass          ::=  "multiclass" TokIdentifier [TemplateArgList]
                         ParentClassList
                         "{" MultiClassStatement+ "}"
MultiClassID        ::=  TokIdentifier
MultiClassStatement ::=  Assert | Def | Defm | Defvar | Foreach | If | Let
```

与类一样，多类有一个名称，并且可以有模板参数。一个多类可以继承其他多类，这导致其他多类被扩展，并贡献记录定义。多类的主体包含一系列使用`def`和`defm`定义记录的语句。此外，defvar、foreach和let语句可以用来分解出更多的公共元素。还可以使用If和Assert语句。

与常规类一样，多类也有隐式模板参数NAME(参见NAME)。当在多类中定义一个命名(非匿名)记录并且记录的名称不包含模板参数NAME时，模板参数NAME会自动添加到名称的前面。也就是说，以下代码在一个多类中是等价的:

```shell
def Foo ...
def NAME # Foo ...
```

### 6.5 defm --定义多个记录

一旦定义了多类，您就可以使用`defm`语句来“调用”它们，处理这些多类中的多个记录定义。这些记录定义由多类中的def语句指定，并由defm语句间接指定。

```shell
Defm ::=  "defm" [NameValue] ParentClassList ";"
```

可选的`NameValue`的形成方式与`def`语句相同。`ParentClassList`是一个冒号，后面是一个至少包含一个多类和任意个常规类的列表。多类必须在常规类之前。注意，`defm`没有主体。

该语句实例化所有指定的多类中定义的所有记录，可以直接使用def语句，也可以间接使用defm语句。这些记录还接收父类列表中包含的任何常规类中定义的字段。这对于向defm创建的所有记录添加一组公共字段非常有用。

该名称与 def 使用相同特殊模式下解析。如果不包含该名称，则提供未指定但全局唯一的名称。也就是说，以下示例以不同的名称结束:

```shell
defm    : SomeMultiClass<...>;   // A globally unique name.
defm "" : SomeMultiClass<...>;   // An empty name.
```

defm语句可以在多类主体中使用。当发生这种情况时，第二种变体相当于:

```shell
defm NAME : SomeMultiClass<...>;
```

更一般的情况是，当defm出现在一个多类中，并且它的名称不包含隐式模板参数NAME的使用时，NAME将自动被添加在前面。也就是说，以下代码在一个多类中是等价的:

```shell
defm Foo        : SomeMultiClass<...>;
defm NAME # Foo : SomeMultiClass<...>;
```

#### 6.5.1 示例：multiclass和defm

下面是一个使用multiclass和defm的简单示例。考虑一个3地址指令体系结构，它的指令有两种格式:reg = reg op reg和reg = reg op imm (immediate)。SPARC就是这种架构的一个例子。 

```shell
def ops;
def GPR;
def Imm;
class inst <int opc, string asmstr, dag operandlist>;

multiclass ri_inst <int opc, string asmstr> {
  def _rr : inst<opc, !strconcat(asmstr, " $dst, $src1, $src2"),
                   (ops GPR:$dst, GPR:$src1, GPR:$src2)>;
  def _ri : inst<opc, !strconcat(asmstr, " $dst, $src1, $src2"),
                   (ops GPR:$dst, GPR:$src1, Imm:$src2)>;
}

// Define records for each instruction in the RR and RI formats.
defm ADD : ri_inst<0b111, "add">;
defm SUB : ri_inst<0b101, "sub">;
defm MUL : ri_inst<0b100, "mul">;
```

 每次使用ri_inst多类都会定义两条记录，一条带有_rr后缀，另一条带有_ri。回想一下，使用多类的defm的名称是在该多类中定义的记录的名称的前缀。因此产生的定义被命名为:

```shell
ADD_rr, ADD_ri
SUB_rr, SUB_ri
MUL_rr, MUL_ri
```

如果没有多类特性，指令就必须定义如下:

```shell
def ops;
def GPR;
def Imm;
class inst <int opc, string asmstr, dag operandlist>;

class rrinst <int opc, string asmstr>
  : inst<opc, !strconcat(asmstr, " $dst, $src1, $src2"),
           (ops GPR:$dst, GPR:$src1, GPR:$src2)>;

class riinst <int opc, string asmstr>
  : inst<opc, !strconcat(asmstr, " $dst, $src1, $src2"),
           (ops GPR:$dst, GPR:$src1, Imm:$src2)>;

// Define records for each instruction in the RR and RI formats.
def ADD_rr : rrinst<0b111, "add">;
def ADD_ri : riinst<0b111, "add">;
def SUB_rr : rrinst<0b101, "sub">;
def SUB_ri : riinst<0b101, "sub">;
def MUL_rr : rrinst<0b100, "mul">;
def MUL_ri : riinst<0b100, "mul">;
```

可以在一个多类中使用defm来“调用”其他多类，除了创建当前多类中定义的记录之外，还创建了在这些多类中定义的记录。在下面的例子中，basic_s和basic_p多类包含引用basic_r多类的defm语句。basic_r多类只包含def语句。

```shell
class Instruction <bits<4> opc, string Name> {
  bits<4> opcode = opc;
  string name = Name;
}

multiclass basic_r <bits<4> opc> {
  def rr : Instruction<opc, "rr">;
  def rm : Instruction<opc, "rm">;
}

multiclass basic_s <bits<4> opc> {
  defm SS : basic_r<opc>;
  defm SD : basic_r<opc>;
  def X : Instruction<opc, "x">;
}

multiclass basic_p <bits<4> opc> {
  defm PS : basic_r<opc>;
  defm PD : basic_r<opc>;
  def Y : Instruction<opc, "y">;
}

defm ADD : basic_s<0xf>, basic_p<0xf>;
```

最后的defm创建以下记录，5条来自basic_s多类，5条来自basic_p多类:

```shell
ADDSSrr, ADDSSrm
ADDSDrr, ADDSDrm
ADDX
ADDPSrr, ADDPSrm
ADDPDrr, ADDPDrm
ADDY
```

defm语句，无论是在顶层还是在多类中，除了从多类继承外，还可以从常规类继承。规则是常规类必须列在多类之后，并且必须至少有一个多类。

```shell
class XD {
  bits<4> Prefix = 11;
}
class XS {
  bits<4> Prefix = 12;
}
class I <bits<4> op> {
  bits<4> opcode = op;
}

multiclass R {
  def rr : I<4>;
  def rm : I<2>;
}

multiclass Y {
  defm SS : R, XD;    // First multiclass R, then regular class XD.
  defm SD : R, XS;
}

defm Instr : Y;
```

这个示例将创建四个记录，按字段的字母顺序显示在这里。

```shell
def InstrSDrm {
  bits<4> opcode = { 0, 0, 1, 0 };
  bits<4> Prefix = { 1, 1, 0, 0 };
}

def InstrSDrr {
  bits<4> opcode = { 0, 1, 0, 0 };
  bits<4> Prefix = { 1, 1, 0, 0 };
}

def InstrSSrm {
  bits<4> opcode = { 0, 0, 1, 0 };
  bits<4> Prefix = { 1, 0, 1, 1 };
}

def InstrSSrr {
  bits<4> opcode = { 0, 1, 0, 0 };
  bits<4> Prefix = { 1, 0, 1, 1 };
}
```

还可以在多类中使用let语句，提供了另一种从记录中提取共性的方法，特别是在使用多个级别的多类实例化时。

```shell
multiclass basic_r <bits<4> opc> {
  let Predicates = [HasSSE2] in {
    def rr : Instruction<opc, "rr">;
    def rm : Instruction<opc, "rm">;
  }
  let Predicates = [HasSSE3] in
    def rx : Instruction<opc, "rx">;
}

multiclass basic_ss <bits<4> opc> {
  let IsDouble = false in
    defm SS : basic_r<opc>;

  let IsDouble = true in
    defm SD : basic_r<opc>;
}

defm ADD : basic_ss<0xf>;
```

### 6.6 defset--创建一个定义集合

`defset`语句用于将一组记录收集到一个全局记录列表中。

```shell
Defset ::=  "defset" Type TokIdentifier "=" "{" Statement* "}"
```

大括号内使用def和defm定义的所有记录都被正常定义，它们也被收集到给定名称的全局列表`TokIdentifier`中。

指定的类型必须是list<class>，其中class是某个记录类。`defset`语句为其语句建立一个作用域。在defset范围内定义非类类型的记录是错误的。

defset语句可以嵌套。内部的defset将记录添加到自己的集合中，所有这些记录也添加到外部集合中。

匿名记录在初始化表达式中使用ClassID<...> 语法创建，该记录不会收集到集合中。

### 6.7 defvar--定义一个变量

defvar语句定义了一个全局变量。它的值可以在定义之后的语句中使用。

```shell
Defvar ::=  "defvar" TokIdentifier "=" Value ";"
```

`=`左侧的标识符被定义为一个全局变量，其值由`=`右侧的值表达式给出。变量的类型是**自动推断**的。

一旦定义了变量，就不能将其设置为其他值。

### 6.8 foreach--遍历一个语句序列

```shell
Foreach         ::=  "foreach" ForeachIterator "in" "{" Statement* "}"
                    | "foreach" ForeachIterator "in" Statement
ForeachIterator ::=  TokIdentifier "=" ("{" RangeList "}" | RangePiece | Value)
```

`foreach`语句的主体是**带大括号的一系列语句**或**不带大括号的单个语句**。对于范围列表、范围块或单个值中的每个值，语句都要重新计算一次。在每次迭代中，`TokIdentifier`变量被设置为该值，可以在语句中使用。

语句列表建立一个内部作用域。foreach的局部变量会在每次循环迭代结束时超出作用域，因此它们的值不会从一次迭代延续到下一次迭代。Foreach循环可以嵌套。

### 6.9 if--基于一个测试的选择语句

`if`语句允许根据表达式的值选择两个语句组中的一个。

```shell
If     ::=  "if" Value "then" IfBody
           | "if" Value "then" IfBody "else" IfBody
IfBody ::=  "{" Statement* "}" | Statement
```

对值表达式进行计算。如果它的计算结果为true(与bang操作符使用的意义相同)，则处理then保留字后面的语句。否则，如果有else保留字，则处理else后面的语句。如果值为false且没有其他分支，则不处理任何语句。

因为then语句周围的大括号是可选的，该语法规则与“悬空else”子句具有歧义性，并且它以通常的方式解决:在类似于if v1 then if v2 then{…else}{…}， else与内部的if相关联，而不是外部的if。

if的then和else分支的IfBody建立一个内部作用域。当主体完成时，任何在主体中定义的defvar变量都将超出作用域(更多细节请参见记录主体中的defvar)。

if语句也可以用于记录体中。

### 6.10 assert--检查一个条件为true

`assert`语句检查一个布尔条件以确保其为真，如果不是，则打印一条错误消息。

```sql
Assert ::=  "assert" condition "," message ";"
```

如果布尔条件为真，则语句不执行任何操作。如果条件为假，则输出非致命错误消息。消息可以是任意字符串表达式，它作为一个提示包含在错误消息中。assert语句的确切行为取决于它的位置。

- 在顶层，断言被立即检查。
- 在记录定义中，将保存语句，在记录完全构建后检查所有断言。
- 在类定义中，断言由继承自该类的所有子类和记录保存和继承。然后在记录完全被构建时检查断言。
- 在多类定义中，断言与多类的其他组件一起保存，然后在每次用defm实例化多类时检查断言。

## 7. 如何构建记录

在构建记录时，TableGen会执行以下步骤。类仅仅是抽象的记录，因此要经历相同的步骤。

1. 构建记录名称`NameValue`并创建一个空记录。

2. 从左到右解析`ParentClassList`中的父类，从上到下访问每个父类的祖先类。
   
   - 将父类中的字段添加到记录中。
   - 将模板参数替换到这些字段中。
   - 将父类添加到记录的继承类列表中。

3. 对记录应用所有的顶层let绑定。回想一下，顶层绑定只应用于继承的字段。

4. 解析记录主体。
   
   - 向记录中添加所有的字段。
   - 根据本地let语句修改字段的值。
   - 定义所有的defvar变量。

5. 遍历所有字段以解析所有的字段间引用。

6. 将记录添加到最终记录列表。

# 二、.td文件与.inc文件内容的对应关系

## 指令相关类的定义

### 1、指令格式

#### InstructionEncoding、Instruction

所有指令共有的信息

> include/llvm/Target/Target.td

```cpp
class InstructionEncoding {
  // Size of encoded instruction.
  int Size;

  // The "namespace" in which this instruction exists, on targets like ARM
  // which multiple ISA namespaces exist.
  string DecoderNamespace = "";

  // List of predicates which will be turned into isel matching code.
  list<Predicate> Predicates = [];

  string DecoderMethod = "";

  // Is the instruction decoder method able to completely determine if the
  // given instruction is valid or not. If the TableGen definition of the
  // instruction specifies bitpattern A??B where A and B are static bits, the
  // hasCompleteDecoder flag says whether the decoder method fully handles the
  // ?? space, i.e. if it is a final arbiter for the instruction validity.
  // If not then the decoder attempts to continue decoding when the decoder
  // method fails.
  //
  // This allows to handle situations where the encoding is not fully
  // orthogonal. Example:
  // * InstA with bitpattern 0b0000????,
  // * InstB with bitpattern 0b000000?? but the associated decoder method
  //   DecodeInstB() returns Fail when ?? is 0b00 or 0b11.
  //
  // The decoder tries to decode a bitpattern that matches both InstA and
  // InstB bitpatterns first as InstB (because it is the most specific
  // encoding). In the default case (hasCompleteDecoder = 1), when
  // DecodeInstB() returns Fail the bitpattern gets rejected. By setting
  // hasCompleteDecoder = 0 in InstB, the decoder is informed that
  // DecodeInstB() is not able to determine if all possible values of ?? are
  // valid or not. If DecodeInstB() returns Fail the decoder will attempt to
  // decode the bitpattern as InstA too.
  bit hasCompleteDecoder = 1;
}
class Instruction : InstructionEncoding {
  string Namespace = "";

  dag OutOperandList;       // An dag containing the MI def operand list.
  dag InOperandList;        // An dag containing the MI use operand list.
  string AsmString = "";    // The .s format to print the instruction with.

  // Pattern - Set to the DAG pattern for this instruction, if we know of one,
  // otherwise, uninitialized.
  list<dag> Pattern;

  // The follow state will eventually be inferred automatically from the
  // instruction pattern.

  list<Register> Uses = []; // Default to using no non-operand registers
  list<Register> Defs = []; // Default to modifying no non-operand registers

  // Predicates - List of predicates which will be turned into isel matching
  // code.
  list<Predicate> Predicates = [];

  // Size - Size of encoded instruction, or zero if the size cannot be determined
  // from the opcode.
  int Size = 0;

  // Code size, for instruction selection.
  // FIXME: What does this actually mean?
  int CodeSize = 0;

  // Added complexity passed onto matching pattern.
  int AddedComplexity  = 0;

  // These bits capture information about the high-level semantics of the
  // instruction.
  bit isReturn     = 0;     // Is this instruction a return instruction?
  bit isBranch     = 0;     // Is this instruction a branch instruction?
  bit isEHScopeReturn = 0;  // Does this instruction end an EH scope?
  bit isIndirectBranch = 0; // Is this instruction an indirect branch?
  bit isCompare    = 0;     // Is this instruction a comparison instruction?
  bit isMoveImm    = 0;     // Is this instruction a move immediate instruction?
  bit isMoveReg    = 0;     // Is this instruction a move register instruction?
  bit isBitcast    = 0;     // Is this instruction a bitcast instruction?
  bit isSelect     = 0;     // Is this instruction a select instruction?
  bit isBarrier    = 0;     // Can control flow fall through this instruction?
  bit isCall       = 0;     // Is this instruction a call instruction?
  bit isAdd        = 0;     // Is this instruction an add instruction?
  bit isTrap       = 0;     // Is this instruction a trap instruction?
  bit canFoldAsLoad = 0;    // Can this be folded as a simple memory operand?
  bit mayLoad      = ?;     // Is it possible for this inst to read memory?
  bit mayStore     = ?;     // Is it possible for this inst to write memory?
  bit mayRaiseFPException = 0; // Can this raise a floating-point exception?
  bit isConvertibleToThreeAddress = 0;  // Can this 2-addr instruction promote?
  bit isCommutable = 0;     // Is this 3 operand instruction commutable?
  bit isTerminator = 0;     // Is this part of the terminator for a basic block?
  bit isReMaterializable = 0; // Is this instruction re-materializable?
  bit isPredicable = 0;     // 1 means this instruction is predicable
                            // even if it does not have any operand
                            // tablegen can identify as a predicate
  bit isUnpredicable = 0;   // 1 means this instruction is not predicable
                            // even if it _does_ have a predicate operand
  bit hasDelaySlot = 0;     // Does this instruction have an delay slot?
  bit usesCustomInserter = 0; // Pseudo instr needing special help.
  bit hasPostISelHook = 0;  // To be *adjusted* after isel by target hook.
  bit hasCtrlDep   = 0;     // Does this instruction r/w ctrl-flow chains?
  bit isNotDuplicable = 0;  // Is it unsafe to duplicate this instruction?
  bit isConvergent = 0;     // Is this instruction convergent?
  bit isAsCheapAsAMove = 0; // As cheap (or cheaper) than a move instruction.
  bit hasExtraSrcRegAllocReq = 0; // Sources have special regalloc requirement?
  bit hasExtraDefRegAllocReq = 0; // Defs have special regalloc requirement?
  bit isRegSequence = 0;    // Is this instruction a kind of reg sequence?
                            // If so, make sure to override
                            // TargetInstrInfo::getRegSequenceLikeInputs.
  bit isPseudo     = 0;     // Is this instruction a pseudo-instruction?
                            // If so, won't have encoding information for
                            // the [MC]CodeEmitter stuff.
  bit isExtractSubreg = 0;  // Is this instruction a kind of extract subreg?
                             // If so, make sure to override
                             // TargetInstrInfo::getExtractSubregLikeInputs.
  bit isInsertSubreg = 0;   // Is this instruction a kind of insert subreg?
                            // If so, make sure to override
                            // TargetInstrInfo::getInsertSubregLikeInputs.
  bit variadicOpsAreDefs = 0; // Are variadic operands definitions?

  // Does the instruction have side effects that are not captured by any
  // operands of the instruction or other flags?
  bit hasSideEffects = ?;

  // Is this instruction a "real" instruction (with a distinct machine
  // encoding), or is it a pseudo instruction used for codegen modeling
  // purposes.
  // FIXME: For now this is distinct from isPseudo, above, as code-gen-only
  // instructions can (and often do) still have encoding information
  // associated with them. Once we've migrated all of them over to true
  // pseudo-instructions that are lowered to real instructions prior to
  // the printer/emitter, we can remove this attribute and just use isPseudo.
  //
  // The intended use is:
  // isPseudo: Does not have encoding information and should be expanded,
  //   at the latest, during lowering to MCInst.
  //
  // isCodeGenOnly: Does have encoding information and can go through to the
  //   CodeEmitter unchanged, but duplicates a canonical instruction
  //   definition's encoding and should be ignored when constructing the
  //   assembler match tables.
  bit isCodeGenOnly = 0;

  // Is this instruction a pseudo instruction for use by the assembler parser.
  bit isAsmParserOnly = 0;

  // This instruction is not expected to be queried for scheduling latencies
  // and therefore needs no scheduling information even for a complete
  // scheduling model.
  bit hasNoSchedulingInfo = 0;

  InstrItinClass Itinerary = NoItinerary;// Execution steps used for scheduling.

  // Scheduling information from TargetSchedule.td.
  list<SchedReadWrite> SchedRW;

  string Constraints = "";  // OperandConstraint, e.g. $src = $dst.

  /// DisableEncoding - List of operand names (e.g. "$op1,$op2") that should not
  /// be encoded into the output machineinstr.
  string DisableEncoding = "";

  string PostEncoderMethod = "";

  /// Target-specific flags. This becomes the TSFlags field in TargetInstrDesc.
  bits<64> TSFlags = 0;

  ///@name Assembler Parser Support
  ///@{

  string AsmMatchConverter = "";

  /// TwoOperandAliasConstraint - Enable TableGen to auto-generate a
  /// two-operand matcher inst-alias for a three operand instruction.
  /// For example, the arm instruction "add r3, r3, r5" can be written
  /// as "add r3, r5". The constraint is of the same form as a tied-operand
  /// constraint. For example, "$Rn = $Rd".
  string TwoOperandAliasConstraint = "";

  /// Assembler variant name to use for this instruction. If specified then
  /// instruction will be presented only in MatchTable for this variant. If
  /// not specified then assembler variants will be determined based on
  /// AsmString
  string AsmVariantName = "";

  ///@}

  /// UseNamedOperandTable - If set, the operand indices of this instruction
  /// can be queried via the getNamedOperandIdx() function which is generated
  /// by TableGen.
  bit UseNamedOperandTable = 0;

  /// Should FastISel ignore this instruction. For certain ISAs, they have
  /// instructions which map to the same ISD Opcode, value type operands and
  /// instruction selection predicates. FastISel cannot handle such cases, but
  /// SelectionDAG can.
  bit FastISelShouldIgnore = 0;
}

/// PseudoInstExpansion - Expansion information for a pseudo-instruction.
/// Which instruction it expands to and how the operands map from the
/// pseudo.
class PseudoInstExpansion<dag Result> {
  dag ResultInst = Result;     // The instruction to generate. 生成的指令
  bit isPseudo = 1;
}


/// ops definition - This is just a simple marker used to identify the operand
/// list for an instruction. outs and ins are identical both syntactically and
/// semantically; they are used to define def operands and use operands to
/// improve readibility. This should be used like this:
///     (outs R32:$dst), (ins R32:$src1, R32:$src2) or something similar.
def ops;
def outs;
def ins;
```

#### InstSw64 基本指令格式

sw指令格式，五种基本指令格式

参考：sw指令手册 2.6 指令格式 

> lib/Target/Sw64/Sw64InstrFormats.td

```cpp
// Sw64 instruction baseline
class InstSw64<bits<6> op, string opstr, string operands> : Instruction {
  field bits<32> Inst; //32位指令
  let Namespace = "Sw64";
  let Inst{31-26} = op;

  let AsmString = opstr # " " # operands;//汇编指令字符串=操作码 操作数
  // ZHAIYH20181122_For_CodeEmit ; Add Size: Number of bytes in encoding
  let Size = 4;
  // SoftFail is a field the disassembler can use to provide a way for
  // instructions to not match without killing the whole decode process. It is
  // mainly used for ARM, but Tablegen expects this field to exist or it fails
  // to build the decode table.
  field bits<32> SoftFail = 0;
}
// Pseudo instructions. 伪指令格式父类
class PseudoInstSw64<dag oops, dag iops, string opstr="", list<dag> pattern> 
    : InstSw64<0, opstr, "">  {//无操作码
  let OutOperandList = oops;//def输出操作数列表
  let InOperandList = iops;//use输出操作数列表
  let Pattern = pattern;//模式匹配
  let isCodeGenOnly = 1;
}

// LDL/LDW     Chapter2.6.3
// Memory  |31     26|25      21|20      16|15               0|
//         |  Opcode |   RA/Fa  |    RB    |        disp      |
class MForm<bits<6> opcode, dag iops, dag oops,
            string opstr, string operands="", list<dag> pattern=[]> 
    : InstSw64<opcode, opstr, operands> {
  let Pattern = pattern;
  let OutOperandList = oops;
  let InOperandList = iops;

  bits<5> RA;
  bits<16> DISP;
  bits<5> RB;

  let Inst{25-21} = RA;
  let Inst{20-16} = RB;
  let Inst{15-0} = DISP;
}
```

#### 具体指令格式、指令记录

参考：sw指令手册 4 基本指令系统

> lib/Target/Sw64/Sw64InstrFormats.td

```cpp
//4.3 load and store instruction
//4.3.1 load integer

let hasSideEffects = 0, mayLoad = 1, mayStore = 0 in
class load_ri<string opstr, bits<6> opcode, RegisterClass regtype,
              SDPatternOperator loadop>
    : MForm<opcode, (ins s64imm:$DISP, GPRC:$RB), (outs regtype:$RA),
            opstr, "$RA,${DISP}(${RB})",
            [(set regtype:$RA,
                (loadop (add GPRC:$RB, immSExt16:$DISP)))]>;

let hasSideEffects = 0, mayLoad = 0, mayStore = 1 in
class store_ri<string opstr, bits<6> opcode, RegisterClass regtype,
               SDPatternOperator storeop>
    : MForm<opcode, (ins regtype:$RA, s64imm:$DISP, GPRC:$RB), (outs),
            opstr, "$RA,${DISP}(${RB})",
            [(storeop regtype:$RA,
               (add GPRC:$RB, immSExt16:$DISP))]>;
// integer load
def LDL  : load_ri<"ldl",  0x23, GPRC, load>;
def LDW  : load_ri<"ldw",  0x22, GPRC, sextloadi32>;
def LDHU : load_ri<"ldhu", 0x21, GPRC, zextloadi16>;
def LDBU : load_ri<"ldbu", 0x20, GPRC, zextloadi8>;
...
```

### 2、操作数

#### DAGOperand、Operand

> include/llvm/Target/Target.td

```cpp
// DAGOperand - An empty base class that unifies RegisterClass's and other forms
// of Operand's that are legal as type qualifiers in DAG patterns.  This should
// only ever be used for defining multiclasses that are polymorphic over both
// RegisterClass's and other Operand's.
class DAGOperand {
  string OperandNamespace = "MCOI";
  string DecoderMethod = "";
}
/// Operand Types - These provide the built-in operand types that may be used
/// by a target.  Targets can optionally provide their own operand types as
/// needed, though this should not be needed for RISC targets.
class Operand<ValueType ty> : DAGOperand {
  ValueType Type = ty;
  string PrintMethod = "printOperand";
  string EncoderMethod = "";
  bit hasCompleteDecoder = 1;
  string OperandType = "OPERAND_UNKNOWN";
  dag MIOperandInfo = (ops);

  // MCOperandPredicate - Optionally, a code fragment operating on
  // const MCOperand &MCOp, and returning a bool, to indicate if
  // the value of MCOp is valid for the specific subclass of Operand
  code MCOperandPredicate;

  // ParserMatchClass - The "match class" that operands of this type fit
  // in. Match classes are used to define the order in which instructions are
  // match, to ensure that which instructions gets matched is deterministic.
  //
  // The target specific parser must be able to classify an parsed operand into
  // a unique class, which does not partially overlap with any other classes. It
  // can match a subset of some other class, in which case the AsmOperandClass
  // should declare the other operand as one of its super classes.
  AsmOperandClass ParserMatchClass = ImmAsmOperand;
}
```

#### 具体操作数

> lib/Target/Sw64/Sw64InstrFormats.td

```cpp
def u8imm   : Operand<i64>{
  let DecoderMethod = "decodeUImmOperand<8>";
}
def s8imm   : Operand<i64>{
  let DecoderMethod = "decodeSImmOperand<8>";
}
def s14imm  : Operand<i64>{
  let DecoderMethod = "decodeSImmOperand<14>";
}
def s16imm  : Operand<i64>{
  let DecoderMethod = "decodeSImmOperand<16>";
  let OperandType = "OPERAND_PCREL";
}
def s21imm  : Operand<i64>{
  let DecoderMethod = "decodeSImmOperand<21>";
  let OperandType = "OPERAND_PCREL";
}
def s64imm  : Operand<i64>{//64位立即数 操作数
  let DecoderMethod = "decodeSImmOperand<64>";
  let PrintMethod = "printMemoryArg";
}
def u64imm  : Operand<i64>{
  let DecoderMethod = "decodeSImmOperand<64>";
}
```

### 3、寄存器

#### RegisterClass、Register

> include/llvm/Target/Target.td

```cpp
// Register file description - These classes are used to fill in the target
// description classes.

class RegisterClass; // Forward def 

// Register - You should define one instance of this class for each register
// in the target machine.  String n will become the "name" of the register.
class Register<string n, list<string> altNames = []> {
  string Namespace = "";
  string AsmName = n;
  list<string> AltNames = altNames;

  // Aliases - A list of registers that this register overlaps with.  A read or
  // modification of this register can potentially read or modify the aliased
  // registers.
  list<Register> Aliases = [];

  // SubRegs - A list of registers that are parts of this register. Note these
  // are "immediate" sub-registers and the registers within the list do not
  // themselves overlap. e.g. For X86, EAX's SubRegs list contains only [AX],
  // not [AX, AH, AL].
  list<Register> SubRegs = [];

  // SubRegIndices - For each register in SubRegs, specify the SubRegIndex used
  // to address it. Sub-sub-register indices are automatically inherited from
  // SubRegs.
  list<SubRegIndex> SubRegIndices = [];

  // RegAltNameIndices - The alternate name indices which are valid for this
  // register.
  list<RegAltNameIndex> RegAltNameIndices = [];

  // DwarfNumbers - Numbers used internally by gcc/gdb to identify the register.
  // These values can be determined by locating the <target>.h file in the
  // directory llvmgcc/gcc/config/<target>/ and looking for REGISTER_NAMES.  The
  // order of these names correspond to the enumeration used by gcc.  A value of
  // -1 indicates that the gcc number is undefined and -2 that register number
  // is invalid for this mode/flavour.
  list<int> DwarfNumbers = [];

  // CostPerUse - Additional cost of instructions using this register compared
  // to other registers in its class. The register allocator will try to
  // minimize the number of instructions using a register with a CostPerUse.
  // This is used by the x86-64 and ARM Thumb targets where some registers
  // require larger instruction encodings.
  int CostPerUse = 0;

  // CoveredBySubRegs - When this bit is set, the value of this register is
  // completely determined by the value of its sub-registers.  For example, the
  // x86 register AX is covered by its sub-registers AL and AH, but EAX is not
  // covered by its sub-register AX.
  bit CoveredBySubRegs = 0;

  // HWEncoding - The target specific hardware encoding for this register.
  bits<16> HWEncoding = 0;

  bit isArtificial = 0;
}

// DAGOperand - An empty base class that unifies RegisterClass's and other forms
// of Operand's that are legal as type qualifiers in DAG patterns.  This should
// only ever be used for defining multiclasses that are polymorphic over both
// RegisterClass's and other Operand's.
class DAGOperand {
  string OperandNamespace = "MCOI";
  string DecoderMethod = "";
}

// RegisterClass - Now that all of the registers are defined, and aliases
// between registers are defined, specify which registers belong to which
// register classes.  This also defines the default allocation order of
// registers by register allocators.
//
class RegisterClass<string namespace, list<ValueType> regTypes, int alignment,
                    dag regList, RegAltNameIndex idx = NoRegAltName>
  : DAGOperand {
  string Namespace = namespace;

  // The register size/alignment information, parameterized by a HW mode.
  RegInfoByHwMode RegInfos;

  // RegType - Specify the list ValueType of the registers in this register
  // class.  Note that all registers in a register class must have the same
  // ValueTypes.  This is a list because some targets permit storing different
  // types in same register, for example vector values with 128-bit total size,
  // but different count/size of items, like SSE on x86.
  //
  list<ValueType> RegTypes = regTypes;

  // Size - Specify the spill size in bits of the registers.  A default value of
  // zero lets tablgen pick an appropriate size.
  int Size = 0;

  // Alignment - Specify the alignment required of the registers when they are
  // stored or loaded to memory.
  //
  int Alignment = alignment;

  // CopyCost - This value is used to specify the cost of copying a value
  // between two registers in this register class. The default value is one
  // meaning it takes a single instruction to perform the copying. A negative
  // value means copying is extremely expensive or impossible.
  int CopyCost = 1;

  // MemberList - Specify which registers are in this class.  If the
  // allocation_order_* method are not specified, this also defines the order of
  // allocation used by the register allocator.
  //
  dag MemberList = regList;

  // AltNameIndex - The alternate register name to use when printing operands
  // of this register class. Every register in the register class must have
  // a valid alternate name for the given index.
  RegAltNameIndex altNameIndex = idx;

  // isAllocatable - Specify that the register class can be used for virtual
  // registers and register allocation.  Some register classes are only used to
  // model instruction operand constraints, and should have isAllocatable = 0.
  bit isAllocatable = 1;

  // AltOrders - List of alternative allocation orders. The default order is
  // MemberList itself, and that is good enough for most targets since the
  // register allocators automatically remove reserved registers and move
  // callee-saved registers to the end.
  list<dag> AltOrders = [];

  // AltOrderSelect - The body of a function that selects the allocation order
  // to use in a given machine function. The code will be inserted in a
  // function like this:
  //
  //   static inline unsigned f(const MachineFunction &MF) { ... }
  //
  // The function should return 0 to select the default order defined by
  // MemberList, 1 to select the first AltOrders entry and so on.
  code AltOrderSelect = [{}];

  // Specify allocation priority for register allocators using a greedy
  // heuristic. Classes with higher priority values are assigned first. This is
  // useful as it is sometimes beneficial to assign registers to highly
  // constrained classes first. The value has to be in the range [0,63].
  int AllocationPriority = 0;

  // The diagnostic type to present when referencing this operand in a match
  // failure error message. If this is empty, the default Match_InvalidOperand
  // diagnostic type will be used. If this is "<name>", a Match_<name> enum
  // value will be generated and used for this operand type. The target
  // assembly parser is responsible for converting this into a user-facing
  // diagnostic message.
  string DiagnosticType = "";

  // A diagnostic message to emit when an invalid value is provided for this
  // register class when it is being used an an assembly operand. If this is
  // non-empty, an anonymous diagnostic type enum value will be generated, and
  // the assembly matcher will provide a function to map from diagnostic types
  // to message strings.
  string DiagnosticString = "";
}

// The memberList in a RegisterClass is a dag of set operations. TableGen
// evaluates these set operations and expand them into register lists. These
// are the most common operation, see test/TableGen/SetTheory.td for more
// examples of what is possible:
//
// (add R0, R1, R2) - Set Union. Each argument can be an individual register, a
// register class, or a sub-expression. This is also the way to simply list
// registers.
//
// (sub GPR, SP) - Set difference. Subtract the last arguments from the first.
//
// (and GPR, CSR) - Set intersection. All registers from the first set that are
// also in the second set.
//
// (sequence "R%u", 0, 15) -> [R0, R1, ..., R15]. Generate a sequence of
// numbered registers.  Takes an optional 4th operand which is a stride to use
// when generating the sequence.
//
// (shl GPR, 4) - Remove the first N elements.
//
// (trunc GPR, 4) - Truncate after the first N elements.
//
// (rotl GPR, 1) - Rotate N places to the left.
//
// (rotr GPR, 1) - Rotate N places to the right.
//
// (decimate GPR, 2) - Pick every N'th element, starting with the first.
//
// (interleave A, B, ...) - Interleave the elements from each argument list.
//
// All of these operators work on ordered sets, not lists. That means
// duplicates are removed from sub-expressions.
```

#### Sw64寄存器 具体寄存器 寄存器类

> lib/Target/Sw64/Sw64RegisterInfo.td

```shell
let Namespace = "Sw64" in {
// ZHAIYH20181123_For_CodeEmit : For register encoding
class Sw64Reg<bits<16> Enc, string n, list<string> alt= []> : Register<n> {
   let HWEncoding = Enc;
   let AltNames = alt;
}

// GPR - One of the 32 32-bit general-purpose registers
class Sw64GPR<bits<16> Enc, string n, list<string> alt= []> : Sw64Reg<Enc, n, alt>;
// FPR - One of the 32 64-bit floating-point registers
class Sw64FPR<bits<16> Enc, string n, list<string> alt= []> : Sw64Reg<Enc, n, alt>;

class Sw64VEC<bits<16> Enc, string n, list<string> alt= []> : Sw64Reg<Enc, n, alt>;

} //Namespace Sw64  


// General-purpose registers
def R0 : Sw64GPR< 0, "$0">, DwarfRegNum<[0]>; 
...


/// Register classes
def GPRC : RegisterClass<"Sw64", [i64], 64, (add
     // Volatile
     R0, R1, R2, R3, R4, R5, R6, R7, R8, R16, R17, R18, R19, R20, R21, R22,
     R23, R24, R25, R28,
     //Special meaning, but volatile
     R27, //procedure address
     R26, //return address
     R29, //global offset table address
     // Non-volatile
     R9, R10, R11, R12, R13, R14,
// Don't allocate 15, 30, 31
     R15, R30, R31)>; //zero
```

### 4、SelectionDAG

#### SDNodeProperty、SDPatternOperator

> include/llvm/CodeGen/SDNodeProperties.td

```cpp
class SDNodeProperty;

// Selection DAG Pattern Operations
class SDPatternOperator {
  list<SDNodeProperty> Properties = [];//具有的属性
}

//===----------------------------------------------------------------------===//
// Selection DAG Node Properties.
//
// Note: These are hard coded into tblgen.
//
def SDNPCommutative : SDNodeProperty;   // X op Y == Y op X
def SDNPAssociative : SDNodeProperty;   // (X op Y) op Z == X op (Y op Z)
def SDNPHasChain    : SDNodeProperty;   // R/W chain operand and result
def SDNPOutGlue     : SDNodeProperty;   // Write a flag result
def SDNPInGlue      : SDNodeProperty;   // Read a flag operand
def SDNPOptInGlue   : SDNodeProperty;   // Optionally read a flag operand
def SDNPMayStore    : SDNodeProperty;   // May write to memory, sets 'mayStore'.
def SDNPMayLoad     : SDNodeProperty;   // May read memory, sets 'mayLoad'.
def SDNPSideEffect  : SDNodeProperty;   // Sets 'HasUnmodelledSideEffects'.
def SDNPMemOperand  : SDNodeProperty;   // Touches memory, has assoc MemOperand
def SDNPVariadic    : SDNodeProperty;   // Node has variable arguments.
def SDNPWantRoot    : SDNodeProperty;   // ComplexPattern gets the root of match
def SDNPWantParent  : SDNodeProperty;   // ComplexPattern gets the parent
```

#### SDNode

> include/llvm/Target/TargetSelectionDAG.td

```cpp
// Selection DAG Node definitions.
class SDNode<string opcode, SDTypeProfile typeprof,
             list<SDNodeProperty> props = [], string sdclass = "SDNode">
             : SDPatternOperator {
  string Opcode  = opcode;//操作码
  string SDClass = sdclass;
  let Properties = props;//具有的属性
  SDTypeProfile TypeProfile = typeprof;
}

// Special TableGen-recognized dag nodes
def set;
def add        : SDNode<"ISD::ADD"       , SDTIntBinOp   ,
                        [SDNPCommutative, SDNPAssociative]>;// 加法操作
def brind      : SDNode<"ISD::BRIND"      , SDTBrind,  [SDNPHasChain]>; //指令、微操作？
```

### 5、模式匹配片段

#### PatFrags

> include/llvm/Target/TargetSelectionDAG.td

```cpp
/// PatFrags - Represents a set of pattern fragments.  Each single fragment
/// can match something on the DAG, from a single node to multiple nested other
/// fragments.   The whole set of fragments matches if any of the single
/// fragemnts match.  This allows e.g. matching and "add with overflow" and
/// a regular "add" with the same fragment set.
///
class PatFrags<dag ops, list<dag> frags, code pred = [{}],
               SDNodeXForm xform = NOOP_SDNodeXForm> : SDPatternOperator {
  dag Operands = ops;
  list<dag> Fragments = frags;
  code PredicateCode = pred;
  code GISelPredicateCode = [{}];
  code ImmediateCode = [{}];
  SDNodeXForm OperandTransform = xform;

  // When this is set, the PredicateCode may refer to a constant Operands
  // vector which contains the captured nodes of the DAG, in the order listed
  // by the Operands field above.
  //
  // This is useful when Fragments involves associative / commutative
  // operators: a single piece of code can easily refer to all operands even
  // when re-associated / commuted variants of the fragment are matched.
  bit PredicateCodeUsesOperands = 0;

  // Define a few pre-packaged predicates. This helps GlobalISel import
  // existing rules from SelectionDAG for many common cases.
  // They will be tested prior to the code in pred and must not be used in
  // ImmLeaf and its subclasses.

  // Is the desired pre-packaged predicate for a load?
  bit IsLoad = ?;
  // Is the desired pre-packaged predicate for a store?
  bit IsStore = ?;
  // Is the desired pre-packaged predicate for an atomic?
  bit IsAtomic = ?;

  // cast<LoadSDNode>(N)->getAddressingMode() == ISD::UNINDEXED;
  // cast<StoreSDNode>(N)->getAddressingMode() == ISD::UNINDEXED;
  bit IsUnindexed = ?;

  // cast<LoadSDNode>(N)->getExtensionType() != ISD::NON_EXTLOAD
  bit IsNonExtLoad = ?;
  // cast<LoadSDNode>(N)->getExtensionType() == ISD::EXTLOAD;
  bit IsAnyExtLoad = ?;
  // cast<LoadSDNode>(N)->getExtensionType() == ISD::SEXTLOAD;
  bit IsSignExtLoad = ?;
  // cast<LoadSDNode>(N)->getExtensionType() == ISD::ZEXTLOAD;
  bit IsZeroExtLoad = ?;
  // !cast<StoreSDNode>(N)->isTruncatingStore();
  // cast<StoreSDNode>(N)->isTruncatingStore();
  bit IsTruncStore = ?;

  // cast<MemSDNode>(N)->getAddressSpace() ==
  // If this empty, accept any address space.
  list<int> AddressSpaces = ?;

  // cast<AtomicSDNode>(N)->getOrdering() == AtomicOrdering::Monotonic
  bit IsAtomicOrderingMonotonic = ?;
  // cast<AtomicSDNode>(N)->getOrdering() == AtomicOrdering::Acquire
  bit IsAtomicOrderingAcquire = ?;
  // cast<AtomicSDNode>(N)->getOrdering() == AtomicOrdering::Release
  bit IsAtomicOrderingRelease = ?;
  // cast<AtomicSDNode>(N)->getOrdering() == AtomicOrdering::AcquireRelease
  bit IsAtomicOrderingAcquireRelease = ?;
  // cast<AtomicSDNode>(N)->getOrdering() == AtomicOrdering::SequentiallyConsistent
  bit IsAtomicOrderingSequentiallyConsistent = ?;

  // isAcquireOrStronger(cast<AtomicSDNode>(N)->getOrdering())
  // !isAcquireOrStronger(cast<AtomicSDNode>(N)->getOrdering())
  bit IsAtomicOrderingAcquireOrStronger = ?;

  // isReleaseOrStronger(cast<AtomicSDNode>(N)->getOrdering())
  // !isReleaseOrStronger(cast<AtomicSDNode>(N)->getOrdering())
  bit IsAtomicOrderingReleaseOrStronger = ?;

  // cast<LoadSDNode>(N)->getMemoryVT() == MVT::<VT>;
  // cast<StoreSDNode>(N)->getMemoryVT() == MVT::<VT>;
  ValueType MemoryVT = ?;
  // cast<LoadSDNode>(N)->getMemoryVT().getScalarType() == MVT::<VT>;
  // cast<StoreSDNode>(N)->getMemoryVT().getScalarType() == MVT::<VT>;
  ValueType ScalarMemoryVT = ?;

  // TODO: Add alignment
}

// PatFrag - A version of PatFrags matching only a single fragment.
class PatFrag<dag ops, dag frag, code pred = [{}],
              SDNodeXForm xform = NOOP_SDNodeXForm>
  : PatFrags<ops, [frag], pred, xform>;
```

PatFrags-表示一组模式片段。每一个片段都可以匹配DAG上的某些内容，从单个节点到多个嵌套的其他片段。如果任何一个片段匹配，则整个片段集匹配。这允许例如匹配和“带溢出的加法”，以及具有相同片段集的常规“加法”

#### 片段实例

> lib/Target/Sw64/Sw64InstrInfo.td

```cpp
def immSExt16  : PatLeaf<(imm), [{ //imm fits in 16 bit sign extended field
  return ((int64_t)N->getZExtValue() << 48) >> 48 ==
         (int64_t)N->getZExtValue();
}]>;//扩展为64位
```

> include/llvm/Target/TargetSelectionDAG.td

```cpp
// load fragments.
def load : PatFrag<(ops node:$ptr), (unindexedload node:$ptr)> {
  let IsLoad = 1;
  let IsNonExtLoad = 1;
}
```

### 6、调度类型

> include/llvm/Target/TargetSchedule.td

```cpp
// A target architecture may define SchedReadWrite types and associate
// them with instruction operands.
class SchedReadWrite;  
// List the per-operand types that map to the machine model of an
// instruction. One SchedWrite type must be listed for each explicit
// def operand in order. Additional SchedWrite types may optionally be
// listed for implicit def operands.  SchedRead types may optionally
// be listed for use operands in order. The order of defs relative to
// uses is insignificant. This way, the same SchedReadWrite list may
// be used for multiple forms of an operation. For example, a
// two-address instruction could have two tied operands or single
// operand that both reads and writes a reg. In both cases we have a
// single SchedWrite and single SchedRead in any order. 
class Sched<list<SchedReadWrite> schedrw> {
  list<SchedReadWrite> SchedRW = schedrw;
}
// Define a scheduler resource associated with a def operand.
class SchedWrite : SchedReadWrite; 
// Map a set of opcodes to a list of SchedReadWrite types. This allows
// the subtarget to easily override specific operations.
//
// SchedModel ties this opcode mapping to a processor.
class InstRW<list<SchedReadWrite> rw, dag instrlist> {
  list<SchedReadWrite> OperandReadWrites = rw;
  dag Instrs = instrlist;
  SchedMachineModel SchedModel = ?;
  // Allow a subtarget to mark some instructions as unsupported.
  bit Unsupported = 0;
}
```

列出映射到一条指令的机器模型的每个操作数类型。必须按顺序为每个显式def操作数列出一个SchedWrite类型。可以选择为隐式def操作数列出其他SchedWrite类型。可以选择按顺序列出SchedRead类型为use操作数。def的顺序相对于use的顺序来说无关紧要。这样，同一个ScheduleReadWrite列表可以用于一条操作的多种形式。例如，双地址指令可以有两个绑定操作数或一个同时读取和写入寄存器reg的操作数。在这两种情况下，我们都有一个任意顺序的SchedWrite和一个SchedRead。

### 指令示例

> lib/Target/Sw64/Sw64InstrInfo.td

```cpp
def PseudoBrind : PseudoInstSw64<(outs), (ins GPRC:$RB), "",
                                 [(brind GPRC:$RB)]>,
                  PseudoInstExpansion<(JMP R31, GPRC:$RB, 0)>, 
                  Sched<[WriteJmp]>;
// 生成的伪指令JMP R31, GPRC:$RB, 0 为什么是R31？寄存器有限制
def LDL  : load_ri<"ldl",  0x23, GPRC, load>;
```

以--gen-instr-info后端为例：

## 1. 指令排序，输出指令枚举值

返回目标定义的所有指令，按其枚举值排序。
还保证以下指令顺序：

a.include/llvm/Support/TargetOpcodes.def中声明的固定/通用指令。也是伪指令？

b.按名称排序的伪指令。

c.按名称排序的其他指令。

### TableGen源码

```cpp
  OS << "#ifdef GET_INSTRINFO_ENUM\n";
  OS << "#undef GET_INSTRINFO_ENUM\n";

  OS << "namespace llvm {\n\n";

  CodeGenTarget Target(Records);

  // We must emit the PHI opcode first...
  StringRef Namespace = Target.getInstNamespace();

  if (Namespace.empty())
    PrintFatalError("No instructions defined!");

  OS << "namespace " << Namespace << " {\n";
  OS << "  enum {\n";
  unsigned Num = 0;
  for (const CodeGenInstruction *Inst : Target.getInstructionsByEnumValue())//获取
    OS << "    " << Inst->TheDef->getName() << "\t= " << Num++ << ",\n";
  OS << "    INSTRUCTION_LIST_END = " << Num << "\n";
  OS << "  };\n\n";
  OS << "} // end " << Namespace << " namespace\n";
  OS << "} // end llvm namespace\n";
  OS << "#endif // GET_INSTRINFO_ENUM\n\n";
```

```cpp
void CodeGenTarget::ComputeInstrsByEnum() const {
  const auto &Insts = getInstructions();
  for (const char *const *p = FixedInstrs; *p; ++p) {//从TargetOpcodes.def中依次取出指令名并去Insts中查找对应的指令记录
    const CodeGenInstruction *Instr = GetInstByName(*p, Insts, Records);
    assert(Instr && "Missing target independent instruction");
    assert(Instr->Namespace == "TargetOpcode" && "Bad namespace");//应有Namespace = TargetOpcode
    InstrsByEnum.push_back(Instr);//放入InstrsByEnum中
  }
  unsigned EndOfPredefines = InstrsByEnum.size();//记录固定指令数量
  assert(EndOfPredefines == getNumFixedInstructions() &&
         "Missing generic opcode");

  for (const auto &I : Insts) {
    const CodeGenInstruction *CGI = I.second.get();
    if (CGI->Namespace != "TargetOpcode") {//Namespace != TargetOpcode即取出除固定指令外的所有指令
      InstrsByEnum.push_back(CGI);//伪指令+一般指令
      if (CGI->TheDef->getValueAsBit("isPseudo"))
        ++NumPseudoInstructions;
    }
  }

  assert(InstrsByEnum.size() == Insts.size() && "Missing predefined instr");

  // All of the instructions are now in random order based on the map iteration.
  llvm::sort(//排序函数：伪指令在前，一般指令在后，各按首字母排序
      InstrsByEnum.begin() + EndOfPredefines, InstrsByEnum.end(),   // 容器从伪指令开始到结束
      [](const CodeGenInstruction *Rec1, const CodeGenInstruction *Rec2) {
        const auto &D1 = *Rec1->TheDef;
        const auto &D2 = *Rec2->TheDef;
        return std::make_tuple(!D1.getValueAsBit("isPseudo"), D1.getName()) <
               std::make_tuple(!D2.getValueAsBit("isPseudo"), D2.getName());//比较条件{!isPseudo，name}
      });
}
```

### *.td

> include/llvm/Target/Target.td

```shell
# Target.td中声明了架构无关的固定指令，标准伪指令
// Standard Pseudo Instructions.
// This list must match TargetOpcodes.def.
// Only these instructions are allowed in the TargetOpcode namespace.
// Ensure mayLoad and mayStore have a default value, so as not to break
// targets that set guessInstructionProperties=0. Any local definition of
// mayLoad/mayStore takes precedence over these default values.
class StandardPseudoInstruction : Instruction {
  let mayLoad = 0;
  let mayStore = 0;
  let isCodeGenOnly = 1;
  let isPseudo = 1;
  let hasNoSchedulingInfo = 1;
  let Namespace = "TargetOpcode";
}
def PHI : StandardPseudoInstruction { 
  let OutOperandList = (outs unknown:$dst);
  let InOperandList = (ins variable_ops);
  let AsmString = "PHINODE";
  let hasSideEffects = 0;
}
...
```

标准伪指令。此列表必须与TargetOpcodes.def匹配。TargetOpcode命名空间中只允许这些指令。确保mayLoad和mayStore具有默认值，以便不破坏将guessInstructionProperties设置为0的目标。mayLoad/mayStore的任何本地定义都优先于这些默认值。

> lib/Target/Sw64/Sw64InstrInfo.td

```shell
# Sw64InstrInfo.td中所有继承PseudoInstExpansion的记录才是伪指令
def PseudoBrind : PseudoInstSw64<(outs), (ins GPRC:$RB), "",
                                 [(brind GPRC:$RB)]>,
                  PseudoInstExpansion<(JMP R31, GPRC:$RB, 0)>,
                  Sched<[WriteJmp]>; 
```

```shell
# 一般指令
def LDL  : load_ri<"ldl",  0x23, GPRC, load>;
def LOADgprel : PseudoInstSw64<(outs GPRC:$dst), (ins s64imm:$addr), "",
    [(set GPRC:$dst, (Sw64_gprel tglobaladdr:$addr))]>, Sched<[WriteLD]>;
```

### *.inc

> include/llvm/Support/TargetOpcodes.def

```shell
HANDLE_TARGET_OPCODE(PHI) # 架构无关的固定指令，Target.td中有对应的声明
```

> build/lib/Target/Sw64/Sw64GenInstrInfo.inc

```shell
PHI = 0, # 架构无关的固定指令
PseudoBrind = 165, # 伪指令
LDL = 367, # 一般指令
```

## 2. 将指令按调度分类，输出枚举值

CodeGenSchedModels（即引用SchedModels）的容器SchedClasses保存了已知的所有调度类型，CodeGenSchedClass的来源有两种，第一种来自指令定义，包括createInstRWClass()方法从InstRW定义直接得到的类型，它们优先保存在SchedClasses容器，其他推导自ItinRW，InstRW及指令定义中的SchedVariant定义。

每个指令描述都将映射到调度类。有四种类型的类：
1） 设置了ItinClassDef的显式定义的行程类。Writes和ReadDefs为空。对任意处理器ProcIndices都是0（表示适用于所有的处理器）。
2） 一条指令定义中的一组SchedWrites和SchedReads所描述的隐含类型，它们在所有子目标中都是通用的。对任意处理器ProcIndices都是0（表示适用于所有的处理器）。
3）带有一组将指令映射到每个处理器的SchedWrites和SchedReads的InstRW隐含类。InstrClassMap应将相同的指令映射到此类。ProcIndices包含为该类提供InstrRW记录的所有处理器。对于没有InstRW条目的处理器，仍可以定义ItinClassDef或写入/读取。
4） 推断的类表示可以在运行时解析的另一类的变体。ProcIndices包含可能需要该类的一组处理器。随着变量的扩展，ProcIndex通过SchedClasses传播。可以从一个行程类别推断出多个SchedClasses。每个都从ItinRW记录中继承处理器索引，该记录将行程类映射到变量Writes或Reads

InstRW类将一组操作码映射到SchedReadWrite类型列表。

继承InstRW类的匿名记录（调度类型）的第二个参数就是指令正则表达式。

### TableGen源码

```cpp
  OS << "#ifdef GET_INSTRINFO_SCHED_ENUM\n";
  OS << "#undef GET_INSTRINFO_SCHED_ENUM\n";
  OS << "namespace llvm {\n\n";
  OS << "namespace " << Namespace << " {\n";
  OS << "namespace Sched {\n";
  OS << "  enum {\n";
  Num = 0;
  for (const auto &Class : SchedModels.explicit_classes())
    OS << "    " << Class.Name << "\t= " << Num++ << ",\n";
  OS << "    SCHED_LIST_END = " << Num << "\n";
  OS << "  };\n";
  OS << "} // end Sched namespace\n";
  OS << "} // end " << Namespace << " namespace\n";
  OS << "} // end llvm namespace\n";

  OS << "#endif // GET_INSTRINFO_SCHED_ENUM\n\n";
```

### *.td

第一类：指令定义中的调度类

第二类：继承InstRW的匿名记录。

> include/llvm/Target/TargetSchedule.td

```shell
// DAG operator that interprets each DAG arg as a regex pattern for
// matching Instruction opcode names.
// The regex must match the beginning of the opcode (as in Python re.match).
// To avoid matching prefixes, append '$' to the pattern.
def instregex;//正则表达式
```

> lib/Target/Sw64/Sw64SchedCore3.td

```shell
def : InstRW<[WriteLD], (instregex "^LD(L|W|HU|BU)$")>;
```

### *.inc

> build/lib/Target/Sw64/Sw64GenInstrInfo.inc

```shell
WriteJmp    = 1, # 通过指令定义得到的调度类型，这一指令使用执行步骤Jmp，资源使用情况由Write描述
LDBU_LDHU_LDL_LDW    = 10, # InstRW定义的调度类型
```

## 3. 输出执行指令时隐式使用（Uses）和定义（Defs）的寄存器列表。

作用：遍历所有指令记录中的Uses字段与Defs字段，如果不为空的话输出。

### TableGen源码

```cpp
  for (const CodeGenInstruction *II : Target.getInstructionsByEnumValue()) {
    Record *Inst = II->TheDef;
    std::vector<Record*> Uses = Inst->getValueAsListOfDefs("Uses");
    if (!Uses.empty()) {
      unsigned &IL = EmittedLists[Uses];
      if (!IL) PrintDefList(Uses, IL = ++ListNumber, OS);
    }
    std::vector<Record*> Defs = Inst->getValueAsListOfDefs("Defs");
    if (!Defs.empty()) {
      unsigned &IL = EmittedLists[Defs];
      if (!IL) PrintDefList(Defs, IL = ++ListNumber, OS);
    }
  }
```

### *.td

> lib/Target/Sw64/Sw64InstrInfo.td

```shell
let isBarrier = 1, isCall = 1, Defs = [R26], Uses = [R27, R29] in
def PseudoCall : PseudoInstSw64<(outs), (ins GPRC:$DISP), "",
                                [(Sw64JmpLink GPRC:$DISP)]>,
                 PseudoInstExpansion<(JSR R26, GPRC:$DISP, 0)>,
                 Sched<[WriteJmp]>;
```

### *.inc

> build/lib/Target/Sw64/Sw64GenInstrInfo.inc

```shell
static const MCPhysReg ImplicitList1[] = { Sw64::R27, Sw64::R29, 0 };
static const MCPhysReg ImplicitList2[] = { Sw64::R26, 0 };
```

## 4. 输出操作数信息

### TableGen源码

```cpp
void InstrInfoEmitter::EmitOperandInfo(raw_ostream &OS,
                                       OperandInfoMapTy &OperandInfoIDs) {
  // ID #0 is for no operand info.
  unsigned OperandListNum = 0;
  OperandInfoIDs[std::vector<std::string>()] = ++OperandListNum;//容器的第一项不使用

  OS << "\n";
  const CodeGenTarget &Target = CDP.getTargetInfo();
  for (const CodeGenInstruction *Inst : Target.getInstructionsByEnumValue()) {
    std::vector<std::string> OperandInfo = GetOperandInfo(*Inst);
    unsigned &N = OperandInfoIDs[OperandInfo];
    if (N != 0) continue;

    N = ++OperandListNum;
    OS << "static const MCOperandInfo OperandInfo" << N << "[] = { ";
    for (const std::string &Info : OperandInfo)
      OS << "{ " << Info << " }, ";
    OS << "};\n";
  }
}
```

```cpp
std::vector<std::string>
InstrInfoEmitter::GetOperandInfo(const CodeGenInstruction &Inst) {
  std::vector<std::string> Result;

  for (auto &Op : Inst.Operands) {
    // Handle aggregate operands and normal operands the same way by expanding
    // either case into a list of operands for this op.
    std::vector<CGIOperandList::OperandInfo> OperandList;

    // This might be a multiple operand thing.  Targets like X86 have
    // registers in their multi-operand operands.  It may also be an anonymous
    // operand, which has a single operand, but no declared class for the
    // operand.
    //这可能是一个多操作数的问题。像X86这样的目标在其多操作数操作数中有寄存器。
    //它也可以是一个匿名操作数，它有一个操作数，但没有为该操作数声明类。
    DagInit *MIOI = Op.MIOperandInfo;//微操作数。一个操作数可能由多个微操作数组成。

    if (!MIOI || MIOI->getNumArgs() == 0) {//无微操作数即单个操作数
      // Single, anonymous, operand.  
      OperandList.push_back(Op);
    } else {//具有子操作数
      for (unsigned j = 0, e = Op.MINumOperands; j != e; ++j) {
        OperandList.push_back(Op);

        auto *OpR = cast<DefInit>(MIOI->getArg(j))->getDef();
        OperandList.back().Rec = OpR;
      }
    }

    for (unsigned j = 0, e = OperandList.size(); j != e; ++j) {
      Record *OpR = OperandList[j].Rec;
      std::string Res;

      if (OpR->isSubClassOf("RegisterOperand"))
        OpR = OpR->getValueAsDef("RegClass");//返回RegClass字段的值
      if (OpR->isSubClassOf("RegisterClass"))
        Res += getQualifiedName(OpR) + "RegClassID, ";
      else if (OpR->isSubClassOf("PointerLikeRegClass"))
        Res += utostr(OpR->getValueAsInt("RegClassKind")) + ", ";
      else
        // -1 means the operand does not have a fixed register class.
        //-1表示操作数没有固定寄存器类。
        Res += "-1, ";

      // Fill in applicable flags.
      // 填写适用的标志。
      Res += "0";

      // Ptr value whose register class is resolved via callback.
      //通过回调解析其寄存器类的Ptr值。
      if (OpR->isSubClassOf("PointerLikeRegClass"))
        Res += "|(1<<MCOI::LookupPtrRegClass)";

      // Predicate operands.  Check to see if the original unexpanded operand
      // was of type PredicateOp.
      //谓词操作数。检查原始未展开操作数的类型是否为PredicateOp。
      if (Op.Rec->isSubClassOf("PredicateOp"))
        Res += "|(1<<MCOI::Predicate)";

      // Optional def operands.  Check to see if the original unexpanded operand
      // was of type OptionalDefOperand.
      // 可选的def操作数。检查原始未展开操作数的类型是否为OptionalDefOperand。
      if (Op.Rec->isSubClassOf("OptionalDefOperand"))
        Res += "|(1<<MCOI::OptionalDef)";

      // Fill in operand type.
      //填写操作数类型。
      Res += ", ";
      assert(!Op.OperandType.empty() && "Invalid operand type.");
      Res += Op.OperandType;

      // Fill in constraint info.
      //填写约束信息。
      Res += ", ";

      const CGIOperandList::ConstraintInfo &Constraint =
        Op.Constraints[j];
      if (Constraint.isNone())
        Res += "0";
      else if (Constraint.isEarlyClobber())
        Res += "(1 << MCOI::EARLY_CLOBBER)";
      else {
        assert(Constraint.isTied());
        Res += "((" + utostr(Constraint.getTiedOperand()) +
                    " << 16) | (1 << MCOI::TIED_TO))";
      }

      Result.push_back(Res);
    }
  }

  return Result;
}
```

```cpp
//===----------------------------------------------------------------------===//
// Machine Operand Flags and Description
//===----------------------------------------------------------------------===//

namespace MCOI {
// Operand constraints
enum OperandConstraint {
  TIED_TO = 0,  // Must be allocated the same register as.
  EARLY_CLOBBER // Operand is an early clobber register operand
};

/// These are flags set on operands, but should be considered
/// private, all access should go through the MCOperandInfo accessors.
/// See the accessors for a description of what these are.
enum OperandFlags { LookupPtrRegClass = 0, Predicate, OptionalDef };

/// Operands are tagged with one of the values of this enum.
enum OperandType {
  OPERAND_UNKNOWN = 0,
  OPERAND_IMMEDIATE = 1,
  OPERAND_REGISTER = 2,
  OPERAND_MEMORY = 3,
  OPERAND_PCREL = 4,

  OPERAND_FIRST_GENERIC = 6,
  OPERAND_GENERIC_0 = 6,
  OPERAND_GENERIC_1 = 7,
  OPERAND_GENERIC_2 = 8,
  OPERAND_GENERIC_3 = 9,
  OPERAND_GENERIC_4 = 10,
  OPERAND_GENERIC_5 = 11,
  OPERAND_LAST_GENERIC = 11,

  OPERAND_FIRST_TARGET = 12,
};
}
```

### *.inc

> build/lib/Target/Sw64/Sw64GenInstrInfo.inc

## 5.
