# 简介
简单区分：make是一条命令，makefile 是一个文件
+ makefile：普通文本文件，记录项目构建的流程规则
+ make：一个解释程序，make会到当前目录下寻找makefile文件，并且对makefile中记录的项目构建规则进行解释执行

# makefile编写规则
目标对象 : 依赖对象
[Tab] + 命令操作

# make执行规则（原理）
make会在当前目录下寻找makefile或Makefile文件，进入文件中：
寻找文件中第一个目标对象：
+ 如果未找到，或是目标对象所以来的依赖对象的文件修改时间比当前目标对象的修改时间新：会执行后面所定义的命令来生成目标对象 
  - 如果依赖对象不存在：则make会在当前文件中找目标对象的依赖性
    * 如果找到依赖性，则根据以上规则生成当前目标文件（类似堆栈过程）
    * 如果未找到，则报错返回
  - 如果依赖对象存在，则通过依赖对象生成当前目标对象
+ 如果找到，将这个最想作为最终的目标对象


# 预定义变量
$@ : 目标对象
$^ : 所有依赖对象
$< : 依赖对象中的第一个


举例：

```cpp
//hello.c
#include <stdio.h>

int main()
{
    printf("hello world\n");
    return 0;
}
```

> \# makefile
> 
> hello:hello.o
>     gcc hello.o -o hello
> 
> hello.o:hello.s
>     gcc -c hello.s -o hello.o
> 
> hello.s:hello.i
>     gcc -S hello.i -o hello.s
> 
> hello.i:hello.c
>     gcc -E hello.c -o hello.i