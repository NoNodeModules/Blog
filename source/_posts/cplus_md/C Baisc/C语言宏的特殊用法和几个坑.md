---
title: C语言宏的特殊用法和几个坑
date: '2020/12/31 23:59'
tags:
 - baisc
categories:
 - 博客搭建
abbrlink: 13000
cover: 
---

# C语言宏的特殊用法和几个坑

总结一下C语言中宏的一些特殊用法和几个容易踩的坑。由于本文主要参考GCC文档，某些细节（如宏参数中的空格是否处理之类）在别的编译器可能有细微差别，请参考相应文档。

## 宏基础

宏仅仅是在C预处理阶段的一种文本替换工具，编译完之后对二进制代码不可见。基本用法如下：

### 1. 标示符别名

```c
#define BUFFER_SIZE 1024
```

预处理阶段，`foo = (char *) malloc (BUFFER_SIZE);`会被替换成`foo = (char *) malloc (1024);`

宏体换行需要在行末加反斜杠\

```c
#define NUMBERS 1, \
                2, \
                3
```



预处理阶段`int x[] = { NUMBERS };`会被扩展成`int x[] = { 1, 2, 3 };`

### 2. 宏函数

宏名之后带括号的宏被认为是宏函数。用法和普通函数一样，只不过在预处理阶段，宏函数会被展开。优点是没有普通函数保存寄存器和参数传递的开销，展开后的代码有利于CPU cache的利用和指令预测，速度快。缺点是可执行代码体积大。

```c
#define min(X, Y)  ((X) < (Y) ? (X) : (Y))
```

y = min(1, 2);`会被扩展成`y = ((1) < (2) ? (1) : (2));

------

## 宏特殊用法

### 1. 字符串化(Stringification)

在宏体中，如果宏参数前加个`#`，那么在宏体扩展的时候，宏参数会被扩展成字符串的形式。如：

```c
#define WARN_IF(EXP) \
     do { if (EXP) \
             fprintf (stderr, "Warning: " #EXP "\n"); } \
     while (0)
```

`WARN_IF (x == 0);`会被扩展成：

```c
do { if (x == 0)
    fprintf (stderr, "Warning: " "x == 0" "\n"); }
while (0);
```

这种用法可以用在assert中，如果断言失败，可以将失败的语句输出到反馈信息中

### 2. 连接(Concatenation)

在宏体中，如果宏体所在标示符中有`##`，那么在宏体扩展的时候，宏参数会被直接替换到标示符中。如：

```c
#define COMMAND(NAME)  { #NAME, NAME ## _command }

struct command
{
    char *name;
    void (*function) (void);
};
```

在宏扩展的时候

```c
struct command commands[] =
{
    COMMAND (quit),
    COMMAND (help),
    ...
};
```

会被扩展成：

```c
struct command commands[] =
{
    { "quit", quit_command },
    { "help", help_command },
    ...
};
```

这样就节省了大量时间，提高效率。

------

## 几个坑

### 1. 语法问题

由于是纯文本替换，C预处理器不对宏体做任何语法检查，像缺个括号、少个分号神马的预处理器是不管的。这里要格外小心，由此可能引出各种奇葩的问题，一下还很难找到根源。

### 2. 算符优先级问题

不仅宏体是纯文本替换，宏参数也是纯文本替换。有以下一段简单的宏，实现乘法：

```c
#define MULTIPLY(x, y) x * y
```

`MULTIPLY(1, 2)`没问题，会正常展开成`1 * 2`。有问题的是这种表达式`MULTIPLY(1+2, 3)`，展开后成了`1+2 * 3`，显然优先级错了。

在宏体中，给引用的参数加个括号就能避免这问题。

```c
#define MULTIPLY(x, y) (x) * (y)
```

`MULTIPLY(1+2, 3)`就会被展开成`(1+2) * (3)`，优先级正常了。

其实这个问题和下面要说到的某些问题都属于由于纯文本替换而导致的语义破坏问题，要格外小心。

### 3. 分号吞噬问题

有如下宏定义：

```c
#define SKIP_SPACES(p, limit)  \
     { char *lim = (limit);         \
       while (p < lim) {            \
         if (*p++ != ' ') {         \
           p--; break; }}}
```

假设有如下一段代码：

```c
if (*p != 0)
   SKIP_SPACES (p, lim);
else ...
```

一编译，GCC报`error: ‘else’ without a previous ‘if’`。原来这个看似是一个函数的宏被展开后是一段大括号括起来的代码块，加上分号之后这个if逻辑块就结束了，所以编译器发现这个else没有对应的if。

这个问题一般用`do ... while(0)`的形式来解决：

```c
#define SKIP_SPACES(p, limit)     \
     do { char *lim = (limit);         \
          while (p < lim) {            \
            if (*p++ != ' ') {         \
              p--; break; }}}          \
     while (0)
```

展开后就成了

```c
if (*p != 0)
    do ... while(0);
else ...
```

这样就消除了分号吞噬问题。

这个技巧在Linux内核源码里很常见，比如这个置位宏`#define SET_REG_BIT(reg, bit) do { (reg |= (1 << (bit))); } while (0)`(位于arch/mips/include/asm/mach-pnx833x/gpio.h)

### 4. 宏参数重复调用

有如下宏定义：

```c
#define min(X, Y)  ((X) < (Y) ? (X) : (Y))
```

当有如下调用时`next = min (x + y, foo (z));`，宏体被展开成`next = ((x + y) < (foo (z)) ? (x + y) : (foo (z)));`，可以看到，`foo(z)`被重复调用了两次，做了重复计算。更严重的是，如果foo是不可重入的(foo内修改了全局或静态变量)，程序会产生逻辑错误。

所以，尽量不要在宏参数中传入函数调用。

### 5. 对自身的递归引用

有如下宏定义：

```c
#define foo (4 + foo)
```

按前面的理解，`(4 + foo)`会展开成`(4 + (4 + foo))`，然后一直展开下去，直至内存耗尽。但是，预处理器采取的策略是***只展开一次***。也就是说，`foo`只会展开成`(4 + foo)`，而展开之后`foo`的含义就要根据上下文来确定了。

对于以下的交叉引用，宏体也只会展开一次。

```c
#define x (4 + y)
#define y (2 * x)
```

`x`展开成`(4 + y) -> (4 + (2 * x))`，`y`展开成`(2 * x) -> (2 * (4 + y))`。

**注意，这是极不推荐的写法，程序可读性极差。**

### 6. 宏参数预处理

宏参数中若包含另外的宏，那么宏参数在被代入到宏体之前会做一次完全的展开，除非宏体中含有`#`或`##`。

有如下宏定义：

```c
#define AFTERX(x) X_ ## x
#define XAFTERX(x) AFTERX(x)
#define TABLESIZE 1024
#define BUFSIZE TABLESIZE
```

- `AFTERX(BUFSIZE)`会被展开成`X_BUFSIZE`。因为宏体中含有`##`，宏参数直接代入宏体。
- `XAFTERX(BUFSIZE)`会被展开成`X_1024`。因为`XAFTERX(x)`的宏体是`AFTERX(x)`，并没有`#`或`##`，所以`BUFSIZE`在代入前会被完全展开成`1024`，然后才代入宏体，变成`X_1024`。

-EOF-

------

参考资料：

http://gcc.gnu.org/onlinedocs/cpp/Macros.html