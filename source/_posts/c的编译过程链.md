---
title: c的编译过程链
date: 2020-08-22 07:46:18
tags:
---
编译，是一个逐步的过程，编译的最终输出是在某个特定平台(x86_64或者armv7_64)上运行的可执行文件
（x86_64 armv7的区别）

编译过程链如下：

源代码:rightarrow:预处理:rightarrow:编译:rightarrow:汇编:rightarrow:链接:rightarrow:可执行文件

以`main.c`为例
```c
#include <stdio.h>

int main()
{
     printf("Hello, world!\n");
     return 0;
}
```

一般情况下，无论是直接在命令行还是在Makefile中，我们都用的是`gcc main.c`来进行编译工作

而该句话省略了大部分的编译过程链，其大致等价于以下几句命令的组合：

```shell
gcc -E main.c > main.i          //对源文件中使用到的宏进行扩展，生成原始的c文件
gcc -S main.i                   //将上一步得到的原始c文件编译生成汇编代码
gcc -c main.s                   //调用汇编器将汇编代码转换为目标代码
gcc main.s -o main              //链接器解析外部函数的地址，最后结合目标文件生成一个可执行文件
rm main.c main.s
```
