参考文档：[TableGen Programmer’s Reference — LLVM 15.0.0git documentation](https://llvm.org/docs/TableGen/ProgRef.html?highlight=tablegen#let-override-fields-in-classes-or-records "TableGen Programmer’s Reference — LLVM 15.0.0git documentation")

# 本文中描述格式的标记符号含义

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

# 1. 标识符

标识符格式：

```shell
ualpha        ::=  "a"..."z" | "A"..."Z" | "_"                
TokIdentifier ::=  ("0"..."9")* ualpha (ualpha | "0"..."9")*
TokVarName    ::=  "$" ualpha (ualpha |  "0"..."9")*        
```

注意，与大多数语言不同，TableGen允许`TokIdentifier`以整数开头。在出现歧义的情况下，标记被解释为数字字面量，而不是标识符。

## 保留关键字

```cpp
assert     bit           bits          class         code
dag        def           else          false         foreach
defm       defset        defvar        field         if
in         include       int           let           list
multiclass string        then          true
```

警告：`field`保留字已弃用，只有在`CodeEmitterGen`后端, 它被用来区分普通记录字段和编码字段。

# 2. include关键字

TableGen支持文件包含。包含的文件在包含的位置直接展开，然后被解析。

```shell
IncludeDirective ::=  "include" TokString
```

主文件和被包含文件的部分可以使用预处理指令进行条件化。

```shell
PreprocessorDirective ::=  "#define" | "#ifdef" | "#ifndef"
```

# 3. bang运算符

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

# 4. 数据类型

```shell
Type    ::=  "bit" | "int" | "string" | "dag"  
            | "bits" "<" TokInteger ">"
            | "list" "<" Type ">"
            | ClassID
ClassID ::=  TokIdentifier
```

- 位类型（`bit`）：表示一个值为0或者1的布尔值。

- 整型（`int`）：表示64位整型数。

- 字符串类型（`string`）：表示任意长度的有序字符序列。

- 位类型（`bits<n>`）：表示位宽固定为`n`的整数，`n`可以为任意值。这`n`个位可以单独访问。这种类型的字段用于表示指令操作代码、寄存器号或地址模式/寄存器/位移。字段的位可以单独设置，也可以作为子字段设置。例如，在指令地址中，寻址模式、基址寄存器号和位移可以分别设置。

- 列表（`list<type>`）：表示一个列表，其元素为`<>`中指定的类型`type`。元素类型是任意的；它甚至可以是另一种列表类型。列表元素索引下标从0开始。 

- `dag`：这种类型表示节点组成的嵌套的**有向无环图(DAG)**。每个节点有一个操作符和零个或多个参数(或操作数)。参数可以是另一个dag对象，允许任意的节点和边树。例如，DAG用于表示代码生成器指令选择算法使用的代码模式。详见[有向无环图(dag)](https://llvm.org/docs/TableGen/ProgRef.html?highlight=tablegen#directed-acyclic-graphs-dags "有向无环图(dag)")

- 类（`ClassID`）：在类型上下文中指定类名表示定义值的类型必须是指定类的子类。这与列表类型结合在一起很有用;例如，将列表的元素约束为一个公共基类(例如，a list<Register>只能包含从Register类派生的定义)。ClassID必须为以前声明或定义过的类名。

### 4.1 有向无环图（DAG）

有向无环图可以在TableGen中使用`dag`数据类型直接表示。DAG节点由一个操作符和零个或多个参数(或操作数)组成。每个参数可以是任何想要的类型。通过使用另一个DAG节点作为参数，可以构建任意的DAG节点图。

dag实例的语法如下:

                              (operator argument1, argument2，…)

操作符必须呈现并且必须有记录。可以有零个或多个参数，用逗号分隔。操作符和参数可以有三种格式。

| 格式         | 说明         |
| ---------- | ---------- |
| value      | 参数值        |
| value:name | 参数值和对应的名称  |
| name       | 未初始化的参数的名称 |

其中value可以是任何TableGen值。如果该名称name存在，则必须是一个以美元符号($)开头的TokVarName。名称的作用是将DAG中的操作符或参数以特定含义标记，或将一个DAG中的参数与另一个DAG中的同名参数关联起来。

# 5. 字面量，值

## 5.1 字面量

### 5.1.1 整数字面量

```shell
TokInteger     ::=  DecimalInteger | HexInteger | BinInteger  # 整型
DecimalInteger ::=  ["+" | "-"] ("0"..."9")+                  # 十进制
HexInteger     ::=  "0x" ("0"..."9" | "a"..."f" | "A"..."F")+ # 十六进制
BinInteger     ::=  "0b" ("0" | "1")+                         # 二进制
```

注意， `DecimalInteger`标记包含可选的`+`或`-`符号，这与大多数语言不同，在这些语言中，符号被视为一元操作符。

### 5.1.2 字符串字面量

```shell
TokString ::=  '"' (non-'"' characters and escapes) '"'     # 字符串中不能有双引号和空格
TokCode   ::=  "[{" (shortest text not containing "}]") "}]" #  字符串中不能有括号
```

`TokCode`是由`[{`和`}]`分隔的多行字符串文字。它可以跨行换行，换行符保留在字符串中。

以下转义字符:

```cpp
\\ \' \" \t \n
```

## 5.2 值

### 5.2.1 简单值

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
  
  表示一个位序列，可用于初始化bits<n>类型字段(注意括号)。当这样做时，这些值必须总共代表n位。

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

### 5.2.2 值的后缀

- `value{17}`：表示整数value的第17位

- `value{8...15}`：整数value的第8到第15bit位的值

- `value[4]`：表示列表value的第4个元素。方括号充当列表的下标操作符。只有在指定单个元素时才会出现这种情况。

- `value[4...7,17,2...3,4]`：表示一个新列表，它是列表value的一个切片。新列表包含元素4、5、6、7、17、2、3和4。元素可以以任意顺序被包含多次。这是指定多个元素时的结果。

- `value.field`：指定记录value中指定字段field的值。

# 6. 语句

以下语句可能出现在TableGen源文件的顶层。

```shell
TableGenFile ::=  (Statement | IncludeDirective
                 | PreprocessorDirective)*
Statement    ::=  Assert | Class | Def | Defm | Defset | Defvar
                 | Foreach | If | Let | MultiClass
```

## 6.1 class -- 定义一个抽象记录类

`class`语句定义了一个抽象的记录类，其他的类和记录可以**继承**这个类。

```shell
Class           ::=  "class" ClassID [TemplateArgList] RecordBody   # 模板类
TemplateArgList ::=  "<" TemplateArgDecl ("," TemplateArgDecl)* ">" # 模板参数列表
TemplateArgDecl ::=  Type TokIdentifier ["=" Value]                 # 模板参数声明及[初始化]
```

类可以通过一组**模板参数**来初始化，这些参数的值可以在类的记录体中使用。每次类被另一个类或记录继承时，都会指定这些模板参数。

如果一个模板参数没有用`=`赋予一个默认值 ，那么它就是未初始化的，必须在继承该类时在模板参数列表中指定(必需的参数)。如果为参数分配了默认值，则不需要在参数列表中指定(可选参数)。在声明中，所有必需的模板参数必须位于任何可选参数之前。模板参数默认值从左到右计算。

### 6.1.1 记录体

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

let用于将字段重置为新值。它可以用于直接在主体中定义的字段或从父类继承的字段。一个`RangeList`可以被指定重置bit\<n>字段。

defvar形式定义了一个变量，它的值可以用在记录体中的其他值表达式中。变量不是字段:它不会成为正在定义的类或记录的字段。在处理记录体时，提供了用于保存临时值的变量。详情请参阅[记录体中的Defvar](https://llvm.org/docs/TableGen/ProgRef.html?highlight=tablegen#defvar-in-a-record-body "记录体中的Defvar")。

当类C2继承类C1时，它获得了C1的所有字段定义。当这些定义合并到类C2中时，C2传递给C1的任何模板参数都被替换到定义中。换句话说，C1定义的抽象记录字段在合并到C2之前用模板参数展开。

## 6.2 def -- 定义一个具体记录

`def`语句定义了一个新的具体记录。

```shell
Def       ::=  "def" [NameValue] RecordBody
NameValue ::=  Value (parsed in a special mode)
```

NameValue是可选的。如果指定，将以特殊模式解析它，其中未定义(无法识别)标识符将被解释为文字字符串。特别是，全局标识符被认为是不可识别的。其中包括defvar和defset定义的全局变量。记录名称可以是空字符串。

如果没有给出名称值，则记录是匿名的。匿名记录的最终名称未指定，但全局惟一。

如果def出现在多类语句中，就会做特殊处理。有关详细信息，请参阅下面的多类部分。

通过在记录主体的开头指定ParentClassList子句，一条记录可以继承一个或多个类。父类中的所有字段都被添加到这条记录中。如果两个或多个父类提供相同的字段，记录使用最后一个父类的该字段值。

## 6.3 let -- 重写类或者记录的字段

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

## 6.4 multiclass --定义多个记录

虽然带有模板参数的类是提取多个记录之间的共性的好方法，但多类可以方便的一次定义多个记录。例如，考虑一个3地址指令体系结构，它的指令有两种格式:reg = reg op reg和reg = reg op imm(例如SPARC)。我们希望在一个地方指定这两种常见格式存在，然后在另一个地方指定所有操作是怎样的。`multiclass`和`defm`语句实现了这一目标。您可以将多类看作是展开为多个记录的宏或模板。

```shell
MultiClass          ::=  "multiclass" TokIdentifier [TemplateArgList]
                         ParentClassList
                         "{" MultiClassStatement+ "}"
MultiClassID        ::=  TokIdentifier
MultiClassStatement ::=  Assert | Def | Defm | Defvar | Foreach | If | Let
```

与类一样，多类有一个名称，并且可以有模板参数。一个多类可以继承其他多类，这导致其他多类被扩展，并贡献记录定义。多类的主体包含一系列使用`def`和`defm`定义记录的语句。此外，defvar、foreach和let语句可以用来分解出更多的公共元素。还可以使用If和Assert语句。

## 6.5 defm --定义多个记录

一旦定义了多类，您就可以使用`defm`语句来“调用”它们，处理这些多类中的多个记录定义。这些记录定义由多类中的def语句指定，并由defm语句间接指定。

```shell
Defm ::=  "defm" [NameValue] ParentClassList ";"
```

可选的`NameValue`的形成方式与`def`语句相同。`ParentClassList`是一个冒号，后面是一个至少包含一个多类和任意个常规类的列表。多类必须在常规类之前。注意，`defm`没有主体。

该语句实例化所有指定的多类中定义的所有记录，可以直接使用def语句，也可以间接使用defm语句。这些记录还接收父类列表中包含的任何常规类中定义的字段。这对于向defm创建的所有记录添加一组公共字段非常有用。

## 6.6 defset--创建一个定义集合

`defset`语句用于将一组记录收集到一个全局记录列表中。

```shell
Defset ::=  "defset" Type TokIdentifier "=" "{" Statement* "}"
```

大括号内使用def和defm定义的所有记录都被正常定义，它们也被收集到给定名称的全局列表`TokIdentifier`中。

指定的类型必须是list\<class>，其中class是某个记录类。`defset`语句为其语句建立一个作用域。在defset范围内定义非类类型的记录是错误的。

defset语句可以嵌套。内部的defset将记录添加到自己的集合中，所有这些记录也添加到外部集合中。

匿名记录在初始化表达式中使用ClassID<...> 语法创建，该记录不会收集到集合中。

## 6.7 defvar--定义一个变量

defvar语句定义了一个全局变量。它的值可以在定义之后的语句中使用。

```shell
Defvar ::=  "defvar" TokIdentifier "=" Value ";"
```

`=`左侧的标识符被定义为一个全局变量，其值由`=`右侧的值表达式给出。变量的类型是**自动推断**的。

一旦定义了变量，就不能将其设置为其他值。

## 6.8 foreach--遍历一个语句序列

```shell
Foreach         ::=  "foreach" ForeachIterator "in" "{" Statement* "}"
                    | "foreach" ForeachIterator "in" Statement
ForeachIterator ::=  TokIdentifier "=" ("{" RangeList "}" | RangePiece | Value)
```

`foreach`语句的主体是**带大括号的一系列语句**或**不带大括号的单个语句**。对于范围列表、范围块或单个值中的每个值，语句都要重新计算一次。在每次迭代中，`TokIdentifier`变量被设置为该值，可以在语句中使用。

语句列表建立一个内部作用域。foreach的局部变量会在每次循环迭代结束时超出作用域，因此它们的值不会从一次迭代延续到下一次迭代。Foreach循环可以嵌套。

## 6.9 if--基于一个测试的选择语句

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

## 6.10 assert--检查一个条件为true

`assert`语句检查一个布尔条件以确保其为真，如果不是，则打印一条错误消息。

```sql
Assert ::=  "assert" condition "," message ";"
```

如果布尔条件为真，则语句不执行任何操作。如果条件为假，则输出非致命错误消息。消息可以是任意字符串表达式，它作为一个提示包含在错误消息中。assert语句的确切行为取决于它的位置。

- 在顶层，断言被立即检查。
- 在记录定义中，将保存语句，在记录完全构建后检查所有断言。
- 在类定义中，断言由继承自该类的所有子类和记录保存和继承。然后在记录完全被构建时检查断言。
- 在多类定义中，断言与多类的其他组件一起保存，然后在每次用defm实例化多类时检查断言。

# 7. 如何构建记录

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
