# Issues

C语言中，调用成员变量用点还是用箭头，取决于当前的ID是结构体指针还是结构体变量。

结构体变量/对象用点，结构体指针用箭头

struct与typedef struct的区别

为什么一般用结构体指针？，因为作为函数参数需要修改的话需要传地址

运算符优先级，取成员运算符优先级高于取地址运算符优先级

int \*a[10]  :   是指针数组,本质上就是数组元素是是个int型指针的一维数组,

int (\*a)[10] :  是数组指针， a是指针，指向一个数组。此数组有10个int型元素

inline内联函数

```
#define FALSE 0  
#define TRUE 1  
#define NULL 0
```

- 普通函数，在定义时需要指明返回类型及返回值
- 宏函数，在定义时**不需要**指明返回类型及返回值。
- 那么宏函数的返回值是什么？
- **答：宏函数中最后一个表达式的值，即为宏函数的返回值。该值的类型，即为宏函数的返回类型。**因此，可以说宏函数隐式地指名了其返回值与返回类型。


 [C语言*和&运算符说明，以及*p++、（*p）++、*++p](https://www.cnblogs.com/pengwangguoyh/articles/3170151.html)
 
# include

[include搜索路径](https://blog.csdn.net/farmwang/article/details/72819370)