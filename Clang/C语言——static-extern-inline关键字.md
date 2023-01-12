# C语言的关键字——static

## 变量

### 1.局部变量

编译器一般不对普通局部变量进行初始化，也就是说它的值在初始时是不确定的，除非对其显式赋值。

>普通局部变量存储于进程栈空间，使用完毕会立即释放。

静态局部变量使用static修饰符定义，即使在声明时未赋初值，编译器会将其初始化为0。且静态局部变量存储于进程的全局数据区，即使函数返回，它的值会保持不变。

```c
#include <stdio.h>

void fn(void)
{
    int n = 10;

    printf("n=%d\n", n);
    n++;
    printf("n++=%d\n", n);
}

void fn_static(void)
{
    static int n = 10;

    printf("static n=%d\n", n);
    n++;
    printf("n++=%d\n", n);
}

int main(void)
{
    fn();
    printf("--------------------\n");
    fn_static();
    printf("--------------------\n");
    fn();
    printf("--------------------\n");
    fn_static();

    return 0;
}
```

结果如下：

```shell
-> % ./a.out 
n=10
n++=11
--------------------
static	 n=10
		n++=11
--------------------
n=10
n++=11
--------------------
static 	n=11
		n++=12
```

### 2.全局变量

全局变量定义在函数体外部，在全局数据区分配存储空间，编译器会自动初始化。

**普通全局变量对整个工程都可见，其他文件可以使用extern外部声明后直接使用。**也就是说其他文件不能再使用一个相同名字的变量。（否则编译器会认为是同一个变量，会报错重复定义）

**静态全局变量仅对当前文件可见！**

## 函数

函数的使用方式和全局变量很相似，在函数的返回值前加上**static**，就是静态函数。

静态函数：

- **静态函数**只能在声明他的文件中可见，其他文件不可见。
- 不同的文件可以使用相同的静态函数，互不影响。

非静态函数：

- 可以在另一个文件中直接引用。

 file1.c

```c
/* file1.c */
#include <stdio.h>

static void fun1(void)
{
    printf("hello from fun.\n");
}

int main(void)
{
    fun1();
    fun2();

    return 0;
}
```

 file2.c

```c
#include <stdio.h>

static void fun2(void)
{
    printf("hello from static fun1.\n");
}
```

```shell
-> $gcc file1.c file2.c
/tmp/cc2VMzGR.o：在函数‘main’中：
static_fun.c:(.text+0x20)：对‘fun2’未定义的引用
collect2: error: ld returned 1 exit status
```



# C语言的关键字——extern

> 对变量而言，如果你想在本源文件中使用另一个源文件的变量，就需要在使用前用extern声明该变量，或者在头文件中用extern声明该变量；

> 对函数而言，如果你想在本源文件中使用另一个源文件的函数，就需要在使用前用声明该函数，声明函数加不加extern都没关系，所以在头文件中函数可以不用加extern。

C程序采用模块化的编程思想，需合理地将一个很大的软件划分为一系列功能独立的部分合作完成系统的需求，在模块的划分上主要依据功能。模块由头文件和实现文件组成，对头文件和实现文件的正确使用方法是：

- 规则1：头文件(.h)中是对于该模块接口的声明，接口包括该模块提供给其它模块调用的外部函数及外部全局变量，对这些变量和函数都需在.h中文件中冠以extern关键字声明；

- 规则2：模块内的函数和全局变量需在.c文件开头冠以static关键字声明；
- 规则3：永远不要在.h文件中定义变量；
- 规则4：如果要用其它模块定义的变量和函数，直接包含其头文件即可。

extern关键字可以在一个文件中引用另一个文件中的变量或者函数。就表示此变量/函数是在别处定义的，要在这里引用。

extern不是定义，也就是说不分配空间。

```c
#ifndef  TEST_H
#define TEST_H

int a = 5;
int test(void);

#endif
```

```c
#include "test.h"
int test()
{
	return a;
}
```

```c
#include "test.h"

int main()
{
	a= 2;
	printf("%d\n",test());
}
```

编译运行会报错：

```shell
$ make
gcc -g -O0    -c -o test.o test.c
gcc -g -O0 main.c test.o -o main
test.o:(.data+0x0): a 的多重定义
/tmp/ccwX5VIh.o:(.data+0x0)：第一次在此定义
collect2: 错误：ld 返回 1
make: *** [main] 错误 1
```

## 用实例分析extern的特点

上述的代码编译存在问题，报错信息是a的重复定义，原因如下：

```shell
     test.h
        |
   |         |   
test.c   main.c
```

```test.c```和```main.c```同时引用了```test.h```。由于头文件在编译的时候将内容“拷贝”到文件中。可以看到test.o生成的时候没有发生问题。接下来main.c和test.o同时生成可执行文件。这就相当于```test.h```中的int a在test.o和main.o中各有一份。当生成main可执行文件的时候，就会报冲突。近似效果如下：

```c
#include <stdio.h>

int a = 4;
int a = 4;

int main()
{
	a = 5;
	printf("%d\n",a);
}

```
编译后的错误和之前的报错一样:

```shell
$ gcc a.c 
a.c:4:5: 错误：‘a’重定义
 int a = 4;
     ^
a.c:3:5: 附注：‘a’的上一个定义在此
 int a = 4;
```

### 修改01

第一种修改方式是将test.h中对a的赋初值去掉。这样依赖整个main.c最后生成的main可执行文件近似于：

```c
#include <stdio.h>

int a;
int a;

int main()
{
    a = 5;
    printf("%d\n",a);
}
```

**全局变量被视为一个临时性的定义，可声明多次，最后被链接器折叠起来，只有一个实体**。

这里要注意的问题是：

>局部变量无论是否赋值，都成为定义。
>
>全局变量只有生命+赋值，才称之为定义。

### 修改02

第二种修改方式是符合上述规则的，就是在.h文件中将全局变量修改为extern。表示此处只是声明，任何引用了该头文件的.c文件都可以使用此变量。而且需要在其余的文件中对此变量进行定义才能说实现使用。

```c
#ifndef TEST_H
#define TEST_H
#include <stdio.h>

extern int a;

int test();
#endif
```

```c
#include "test.h"
int a = 100;
int test()
{
	return a;
}
```

```c
#include "test.h"

int main()
{
    int a= 2;
    printf("%d\n",test());
}
```

编译：

```shell
$ make
gcc -g -O0    -c -o test.o test.c
gcc -g -O0 main.c test.o -o main

$ ./main 
100
```

# C语言的关键字——inline

如果一些函数被频繁的调用，不断地有函数入栈，有可能会造成栈内存消耗大量。一般可以将这些函数加上inline关键字。

```c
#include <stdio.h>  
 
//函数定义为inline即:内联函数  
inline char* dbtest(int a) 
{  
	return (i % 2 > 0) ? "奇" : "偶";  
}   
  
int main()  
{  
	int i = 0;  
	for (i=1; i < 100; i++) 
	{  
		printf("i:%d    奇偶性:%s /n", i, dbtest(i));      
	}  
} 
```

上述dbtest函数由于要经常性的被调用，所以一般可以构建为inline函数。

>其实在内部的工作就是在每个for循环的内部任何调用dbtest(i)的地方都换成了（i%2)>0?奇:偶。这样就避免了频繁调用函数对栈内存重复开辟带来的消耗。

## inline的使用方式

inline关键字一定要和函数定义放在一起，仅放在函数声明前不起作用。而且inline一般适用于函数内部结构简单的函数，如果函数内部有循环等操作，则不能使用inline。

## inline的限制

内联能提高函数的执行效率，为什么不把所有的函数都定义成内联函数？如果所有的函数都是内联函数，还用得着“内联”这个关键字吗？

> 内联是以代码膨胀（复制）为代价，仅仅省去了函数调用的开销，从而提高函数的执行效率。如果执行函数体内代码的时间，相比于函数调用的开销较大，那么效率的收

获会很少。另一方面，每一处内联函数的调用都要复制代码，将使程序的总代码量增大，消耗更多的内存空间。

以下情况不宜使用内联：

（1）如果函数体内的代码比较长，使用内联将导致内存消耗代价较高。

（2）如果函数体内出现循环，那么执行函数体内代码的时间要比函数调用的开销大。

一个好的编译器将会根据函数的定义体，自动地取消不值得的内联（这进一步说明了inline 不应该出现在函数的声明中）。
