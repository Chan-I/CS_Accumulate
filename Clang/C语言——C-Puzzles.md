# C Puzzles 01—— sizeof

下述程序需要打印数字中所有的元素，但是事实上程序运行的结果并非如此，原因是什么呢？

```c
 #include<stdio.h>

  #define TOTAL_ELEMENTS (sizeof(array) / sizeof(array[0]))
  int array[] = {23,34,12,17,204,99,16};

  int main()
  {
      int d;

      for(d=-1;d <= (TOTAL_ELEMENTS-2);d++)
          printf("%d\n",array[d+1]);

      return 0;
  }
```

- **sizeof**

  sizeof是一个C语言的关键字，需要注意的是**sizeof的结果是在编译阶段就由编译器得出。**sizeof不是函数，没有元素；**sizeof编译为size_t类型的值**，size_t并不是unsigned int，在32位系统中，size_t为无符号四字节数；64位系统中sizeo_t为无符号八字节数；

  ```c
  #define my_sizeof(x) \
   ({ \
       typeof(x) _x;  \
    	 (size_t)((char *)(&_x + 1) - (char *)(&_x)); \
  })
      
  或
      
  size_t my_sizeof(x)
  {
  	typeof(x) _x;
  	return (size_t)((char *)(&_x + 1) -(char *)(&_x));
  }
  
  或
      
  #define sizeof(var) (size_t)((typeof(var) *)0 + 1)
  ```

(typeof(var) *)0表示的是一个基地址，还有两个类似用法。

```c
#define FIELD_OFFSET(_struct_name_, _field_name_) \
				((size_t) &(((_struct_name_ *)0)->_field_name_))

#define FIELD_SIZE(_struct_name_, _field_name_) \
				(sizeof(((_struct_name_ *)0)->_field_name_))
```

这里的理解就是首先将基地址0作为指针读取类型的地址，然后按照`_struct_name_`读取这个基地址，`-> _field_name_`表示指针从基地址0开始，按照`_struct_name_`的方式读取，同时读取`_field_name_`成员的地址。这样一来它返回的地址就是这个成员在结构体中的便宜offset数值。

当取到了成员地址后，sizeof会返回这个成员的类型大小。

具体代码如下：

```c
#include<stdio.h>
#include<stdlib.h>
#define FIELD_OFFSET(_struct_name_, _field_name_) \
				((size_t) &(((_struct_name_ *)0)->_field_name_))

#define FIELD_SIZE(_struct_name_, _field_name_) \
				(sizeof(((_struct_name_ *)0)->_field_name_))
typedef struct {
        int a;
        long b;
        char c;
} TEST;

int main()
{
        TEST test;

        printf("%d\n",FIELD_OFFSET(TEST, a));	// a offset = 0
        printf("%d\n",FIELD_SIZE(TEST, a));		// a size = 4

        printf("%d\n",FIELD_OFFSET(TEST, b));	// b offset = 8
        printf("%d\n",FIELD_SIZE(TEST, b));		// b size = 8

        printf("%d\n",FIELD_OFFSET(TEST, c));	// c offset = 16
        printf("%d\n",FIELD_SIZE(TEST, c));		// c size = 1
}
```

## 易错误点讲解

for后面有这样一条判断语句，我们来看一下它是如何进行自动类型转换的。

```c
d <= (TOTAL_ELEMENTS-2)
```

1. 各参数类型如下：

   d 	int

   TOTAL_ELEMENTS	size_t

   2	int

2. 转换过程：

   2由int转换为size_t，(TOTAL_ELEMENT - 2)值为5，类型为size_t；d由int转换为size_t时，负载在内存中以补码的形式存在，当d = -1的时候。d在内存中的存值为ffffffff，当d转换为size_t类型时，值为ffffffff。所以d <= (TOTAL_ELEMENT - 2)为false，不会进入for循环，也就不会打印数组信息。

```c
d <= (TOTAL_ELEMENTS-2) 
```
改为
```c
d <= (int)(TOTAL_ELEMENTS-2)
```

# C Puzzles 03——循环中的continue

```
enum {false,true};

int main()
{
        int i=1;
        do
        {
                printf("%d\n",i);
                i++;
                if(i < 15)
                    continue;
        }while(false);
        return 0;
}
```

## 分析

对于循环来说，break是立即跳出循环，continue是直接结束本次循环。这知识通俗意义上的理解。

官方对于continue的解释如下：

> it causes the next iteration of the enclosing for, while, or do loop to begin. In the while and do, this means that the test part is executed immediately; in the for, control passes to the increment step.

当程序运行到continue时，会直接跳转至while(false),并进行条件的判断，所以最终结果只有一个1。

# C Puzzles 04 —— fprintf(stderr)

下面的程序看似会输出两句话，但是实际上只会输出一句话。

```c
  #include <stdio.h>
  #include <unistd.h>
  int main()
  {
          while(1)
          {
                  fprintf(stdout,"hello-out");
                  fprintf(stderr,"hello-err");
                  sleep(1);
          }
          return 0;
  }
```

## 分析

只会输出```fprintf(stderr,"hello-err");```

这是因为stdout选项会将结果暂存在缓冲区，stderr则不会将输出结果放在缓冲区。

如果想输出stdout的内容有如下几种方法：

1.手动刷新stdout。

​	```fflush(stdout);```

2.main函数推出：

​	程序交回给操作系统之前C运行库需要进行清理工作，其中一部分是刷新stdout缓冲区。

3.scanf执行之前会清理stdout缓冲区。

**4.fork与缓冲区**

​	fork()创建子进程，子进程会继承父进程的缓冲区。fork调用的时候，整个父进程会将缓冲区复制进子进程，包括指令、变量值、程序调用栈、环境变量和缓冲区。

## 例如：

下面会给一个经典的例子：下面程序一共会输出多少个"-"

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
 
int main(void)
{
   int i;
   for(i=0; i<2; i++){
      fork();
      printf("-");
      //printf("-\n");
   }
 
   return 0;
}
```

这个题涉及很多知识点：

- 首先printf(“-”)输出再遇到\n之前是会将buffer放在流中，而printf("-\n")会直接输出。
- fork会继承指令、变量值、程序调用栈、环境变量和缓冲区等。

---

1. 在父进程执行第一个fork时，还没有缓冲区。接下来父进程执行到print，此时父进程的缓冲区有一个“-”，派生的第一个子进程缓冲区也有一个“-”。如下：

   ```
    [-] -----> [-]
   ```

2. 接下来父进程还会再派生一个子进程，第一个被派生的子进程会在派生一个子进程。先分析第一个子进程的情况。第一个子进程在派生进程的时候会将自己的缓冲区传递给它的子进程。

   ```
    [-] -----> [-]
    			 |---------> [-]  
    			
    			↑ //这个进程是第一个自己成，它先派生子进程（此时缓冲区已经有一个“-”了）
               	// 那么它派生的子进程的缓冲区就自带一个"-"
    
   ```

   然后接着会继续向缓冲区添加“-”

   ```
    [-] -----> [--]
    			 |---------> [--]
    			 
                ↑ // 这个进程派生后会再往缓冲区添加一个“-”，那么此进程的缓冲区就有两个“-”
                
                			  ↑ //这个进程继承了i = 1，不会再派生，但是会向自己的缓冲区
                			  	// 续写一个“-”。
                
   ```

   以上由父进程派生出来的子进程就折腾完毕了。

3. 在分析父进程的第二次循环，第二次循环又会派生一个新的子进程（二儿子）,二儿子进程继承了父进程的缓冲区，此时状态如下：

   ```
   [-] -----> [--]
    |   		 |---------> [--]
    |
    |------> [-]
   ```

   接下来父进程会再向自己的缓冲区添加一个“-”。同时二儿子进程的i = 1。不会再派生，只会向自己的缓冲区添加一个“-”

   ```
   [--] -----> [--]
    |           |
    |   		 |---------> [--]
    |
    |------> [--]
   ```

   以上所有进程的缓冲区一共有8个"-"。

如果上述的printf中有换行符，则子进程集成父进程的信息时，缓冲区已经被清空了。所以会比上述情况少两个。

# C Puzzles 05 —— define#和##混用的限制

```c
  #include <stdio.h>
  #define f(a,b) a##b
  #define g(a)   #a
  #define h(a) g(a)

  int main()
  {
          printf("%s\n",h(f(1,2)));
          printf("%s\n",g(f(1,2)));
          return 0;
  }
```

## 讲解

- h(f(1,2))

  ```
  h(f(1,2))
  	->h(12)
  		->g(12)
  			->”12“
  ```

- g(f(1,2)) —— **需要考虑到 # 和 ## 混用时候的限制。**

  ```
  g(f(1,2))
  	->"f(1,2)"
  ```

## 扩展

- **宏定义中的两个符号：“#”，“##”**

**#**: 将其后面的宏参数进行**字符串化操作(stringfication)**，即对它所引用的宏变量通过替换后在其左右各加上一个双引号。

**##**：**连接符(concatenator)**，用来将两个token连接为一个token。连接的对象是token就行，不一定是宏的变量。



- **当宏参数为也是宏的时候，如果宏定义使用了#或者##，宏参数不会被展开**

```c
#define PARAM(x) x
#define ADDPARAM(x) INT_##x
```

面对```PARAM(ADDRPARAM(1))```时候，展开顺序如下，由外向内展开，先解析#或##：

```
PARAM(ADDRPARAM(1))
	->PARAM(INT_1)
		->“INT_1”
```

---

```c
#define PARAM(x) #x
#define ADDPARAM(x) INT_##x
```

面对```PARAM(ADDPARAM(1))```的时候，展开顺序为，也是由外向内展开：

```c
PARAM(ADDPARAM(1))
	->“ADDRPARAM(1)”
```

- **当宏定义中对宏参数使用了#或者##，可以对这个宏在添加一层宏，实现宏的展开**

```c
#define _PARAM(x) #x
#define PARAM(x) _PARAM(x)
#define ADDPARAM(x) INT_##x
```

面对```PARAM(ADDPARAM(1))```的时候，展开顺序如下，由外向内，但是先解析#或##：

```c
PARAM(ADDPARAM(1))
	->PARAM(INT_1)
		->_PARAM(INT_1)
			->“INT_1”
```

- **总结就是，由外到内，有#（或##）先#（或##）**

# C Puzzles 06 —— switch可以没有default

```c
#include<stdio.h>
  int main()
  {
          int a=10;
          switch(a)
          {
                  case '1':
                      printf("ONE\n");
                      break;
                  case '2':
                      printf("TWO\n");
                      break;
                  defa1ut: 
                      printf("NONE\n");
          }
          return 0;
  }
```

## 解答

需要注意的是，C语言中的switch中不会检查你其中是否有case、default意外的条件。上面的default拼写错误但是不影响编译，所以上述的switch相当于没有default。



# C Puzzles 09 —— float精度低

```c
#include <stdio.h>

int main()
{
        float f=0.0f;
        int i;

        for(i=0;i<10;i++)
                f = f + 0.1f;
		printf("%f\n",f);
        if(f == 1.0f)
                printf("f is 1.0 \n");
        else
                printf("f is NOT 1.0\n");

        return 0;
}
```

需要注意的是float类型数据，精度不高。通过`<` ,`>`,`<=`, `>=`, `==` ,`!=` 比较是不对的。



# C Puzzles 10 —— 逗号表达式

```c
#include <stdio.h>

int main()
{
        int a = 1,2;
    	// int a = (1,2);
        printf("a : %d\n",a);
        return 0;
}
```

表达式的常见形式是：```(expr 1, expr 2, expr3, ... , expr n)```

(1) 逗号表达式的计算过程为从左往右逐个计算；

(2) 逗号表达式作为一个整体，它的值为最后一个表达式的值；

(3) 逗号运算符的优先级在所有运算符中最低。



# C Puzzles 11 —— printf()返回值

```c
#include <stdio.h>
int main()
{
        int i=43;
        printf("%d\n",printf("%d",printf("%d",i)));
        return 0;
}

# 4321
```

```printf()```的返回值为打印的字符数。

# C Puzzles 13 —— 位运算

```c
  int CountBits(unsigned int x)
  {
      int count=0;
      while(x)
      {
          count++;
          x = x&(x-1);
      }
      return count;
  }
```

## 解析

`x & ( x - 1 )`常见的两种应用：

- 计算二进制**x**中，数字1的个数；
- 判断**x**是否为2的幂。

# C Puzzles 14 —— 空和void作为函数输入

Are the following two function prototypes same? 

 int foobar(void);
 int foobar();

The following programs should be of some help in finding the answer: (Compile and run both the programs and see what happens) 

```c
#include <stdio.h>
  void foobar1(void)
  {
   printf("In foobar1\n");
  }

  void foobar2()
  {
   printf("In foobar2\n");
  }

  int main()
  {
     char ch = 'a';
     foobar1();
     foobar2(33, ch);
     return 0;
  }
```

```c
#include <stdio.h>
  void foobar1(void)
  {
   printf("In foobar1\n");
  }

  void foobar2()
  {
   printf("In foobar2\n");
  }

  int main()
  {
     char ch = 'a';
     foobar1(33, ch);
     foobar2();
     return 0;
  }
```

## 知识点

`foo()`和`foo(void)`是不同的

调用foo()时，可以传入任意个数的参数，也可以是任意类型的参数，编译器不关心传给foo的参数，传入参数对foo函数的执行没有影响；

foo(void)时，不可传入参数，否则编译不通过；

**日常写代码的时候，如果函数没有参数，最好写成foo(void)的类型。**

# C Puzzles 15

```c
  #include <stdio.h>
  int main()
  {
   float a = 12.5;
   printf("%d\n", a);
   printf("%d\n", *(int *)&a);
   return 0;
  }
```

输出：

​		**0**
​		**1095237632**

## 题目讲解

`printf("%d\n", a);`，a转换为double型再入栈，%d读取double型数据的低4个字节；

`printf("%d\n", *(int *)&a);` float占4个字节，*(int *)&a将a在内存中存放值转换为整型数后再传入printf。

浮点数在内存中如何表示参考第九题讲解。

## 知识点

**printf中的%f读取sizeof(double)个字节**

```c
#include <stdio.h>

int main()
{
        int a = 10, b = 20, c = 30;
        printf("%f, %d\n", a, b, c);
        return 0;
}
```

输出：0.000000, 30

调用printf时，双引号后面的所有参数从右往左依次入栈，然后将栈中的数据根据双引号中的格式从低地址向高地址依次读取并打印出来。

上述代码调用printf时，c,b,a依次入栈，%f读取sizeof(double)=8个字节，

即a,b所占的区域，转换成浮点数为0.000000，%d继续读取下面4个字节，即c所占的区域，转换为整型数为30。

**printf中传入的float型参数自动转换成double型**

```c
#include <stdio.h>

int main()
{
        float a = 1.2;
        int b = 10;
        printf("%x, %x, %d\n", a, b);
        return 0;
}
```

输出：40000000, 3ff33333, 10

调用printf时，参数a被转为double型后再传入printf，printf的前两个%x读取的是a所占的8个字节，%d读取的是b所占的4个字节。

gcc –S test.c命令可以生成汇编代码，上例的汇编代码主体如下：

```mov
main:
        pushl   %ebp
        movl    %esp, %ebp
        andl    $-16, %esp
        subl    $32, %esp
        movl    $0x3f99999a, %eax
        movl    %eax, 28(%esp)
        movl    $10, 24(%esp)
        flds    28(%esp)
        movl    $.LC1, %eax
        movl    24(%esp), %edx
        movl    %edx, 12(%esp)
        fstpl   4(%esp)
        movl    %eax, (%esp)
        call    printf
        movl    $0, %eax
        leave
        ret
```

“flds   28(%esp)”，flds将28(%esp)地址处的单精度浮点数即0x3f99999a加载到浮点寄存器，“fstpl  4(%esp)”即将浮点寄存器中的浮点数转换成double型后放到4(%esp)处。

**printf中的%c读取4个字节，printf的char型参数先转换为int再入栈**

```c
#include <stdio.h>

int main()
{
        char a = 97;
        int b = 20;
        printf("%c, %d\n", a, b);
        return 0;
}
```

输出：a, 20

上述c代码的汇编代码主体为：

```mov
main:
        pushl   %ebp
        movl    %esp, %ebp
        andl    $-16, %esp
        subl    $32, %esp
        movb    $97, 31(%esp)
        movl    $20, 24(%esp)
        movsbl  31(%esp), %edx
        movl    $.LC0, %eax
        movl    24(%esp), %ecx
        movl    %ecx, 8(%esp)
        movl    %edx, 4(%esp)
        movl    %eax, (%esp)
        call    printf
        movl    $0, %eax
        leave
        ret
```

“movl   %edx, 4(%esp)”在栈中压了4个字节。

**sizeof('a') = 4**

```c
#include <stdio.h>

int main()
{
        char a = 'a';
        printf("%d, %d\n", sizeof(a), sizeof('a'));
        return 0;
}
```

**sizeof求类型的大小，sizeof(a)即sizeof(char)，sizeof(‘a’)即sizeof(int),‘a’自动转换为int型。**



# C Puzzles 17——switch的跳转

```c
  #include<stdio.h>
  int main()
  {
      int a=1;
      switch(a)
      {   int b=20;
          case 1: printf("b is %d\n",b);
                  break;
          default:printf("b is %d\n",b);
                  break;
      }
      return 0;
  }
```

## 题目讲解

输出：b is 0

switch判断后直接跳转到相应的case/default处，不会执行之前的赋值语句。

# C Puzzles 18——数组参数传递

```c
#define SIZE 10
  void size(int arr[SIZE])
  {
          printf("size of array is:%d\n",sizeof(arr));
  }

  int main()
  {
          int arr[SIZE];
          size(arr);
          return 0;
  }
```

题目讲解：

数组做参数传递时退化为指针，“void size(int arr[SIZE]) ”等价于“void size(int *arr) ”。

size(arr)给size函数传入的参数是指针，所以sizeof(arr)是指针的大小。



# C Puzzles 20——scanf的输入格式

```c
#include <stdio.h>
  int main()
  {
      char c;
      scanf("%c",&c);
      printf("%c\n",c);

      scanf(" %c",&c);
      printf("%c\n",c);

      return 0;
  }
```

## 题目讲解

当第二个scanf没有空白符时，运行代码，输入a,回车后，会打印a,换行，换行，然后程序结束，字符’a’被第一个scanf读取，字符’\n’被第二个scanf读取；

当第二个scanf有空白符时，运行代码，输入a,回车，输出a，若继续敲回车，程序不结束，直到输入了字符(非换行符)后，打印该字符，程序结束。

C99的7.19.6.2第五条说，包含空白符的指令读取输入中的第一个非空白字符。

# C Puzzles 21——sacnf预防越界

```c
#include <stdio.h>
  int main()
  {
      char str[80];
      printf("Enter the string:");
      scanf("%s",str);
      printf("You entered:%s\n",str);

      return 0;
  }
```

**题目讲解：**

易造成数组越界，scanf改成如下形式比较保险



不管输入多少个字节，最多只读取79个字节。



# C Puzzles 22——sizeof(i++)

```c
  #include <stdio.h>
  int main()
  {
      int i;
      i = 10;
      printf("i : %d\n",i);
      printf("sizeof(i++) is: %d\n",sizeof(i++));
      printf("i : %d\n",i);
      return 0;
  }
```

## 题目讲解

输出为：

i: 10

sizeof(i++) is:4

i: 10

sizeof(i++)，等效于sizeof(int)。sizeof的值在编译时决定，不会执行i++。

sizeof的更多讲解见第一题。

# C Puzzles 23——const关键字

```c
  #include <stdio.h>
  void foo(const char **p) { }
  int main(int argc, char **argv)
  {
          foo(argv);
          return 0;
  }
```

## 题目讲解

- 关键字const，会将变量指定为只读。

```c
const int a = 10;//变量a只读
const int *p;//p指向的int型数只读，p可以被修改
int *const p;//p为只读，p指向的int型数可以被修改
```

- 带着const的一级指针和二级指针赋值

```c
int a = 10;
const int *p;
p = &a;//一级const指针赋值时，可以不将右边的指针转换为const型

-----------------------------------------------------------------
int *p;
const int **pp;
pp = (const int **)&p;//二级const指针赋值时，必须将右侧二级指针转换为const型
```

- 带const的一级指针和二级指针传参

```c
void foo(const char *p) {}
int main()
{
	char *p;
	foo(p);//p不用转化为const型
	return 0;
}
```

```c
void foo(const char **pp) {}
int main()
{
	char **pp;
	foo((const char **)pp);//pp必须转化为const型
	return 0;
}
```

# C Puzzles 24——逗号运算符的优先级

```c
  #include <stdio.h>
  int main()
  {
          int i;
          i = 1,2,3;
          printf("i:%d\n",i);
          return 0;
  }
```

## 题目讲解

输出：i=1

逗号运算符在所有的运算符中优先级最低，i = 1,2,3等效于(i = 1),2,3

若将”i = 1,2,3”改成”i = (1,2,3)”，i的值为3。



# C Puzzles 27——进制数

```c
  #include <stdio.h>
  #include <stdlib.h>

  #define SIZEOF(arr) (sizeof(arr)/sizeof(arr[0]))

  #define PrintInt(expr) printf("%s:%d\n",#expr,(expr))
  int main()
  {
      /* The powers of 10 */
      int pot[] = {
          0001,
          0010,
          0100,
          1000
      };
      int i;

      for(i=0;i<SIZEOF(pot);i++)
          PrintInt(pot[i]);
      return 0;
  }
```

## 题目讲解

输出为：

pot[i]:1

pot[i]:8

pot[i]:64

pot[i]:1000

PrintInt(pot[i])被替换为printf("%s:%d\n",“pot[i]”,pot[i]);

0001,0010,0100表示八进制数，1000表示十进制数。

---

- define 中的 #

“#”将其后面的宏参数进行字符串化操作(stringfication)，即对它所引用的宏变量通过替换后在其左右各加上一个双引号。更多关于宏中”#””##”的讲解见第五题。

- C语言中的二进制、八进制、十进制

以十进制数100为例：

二进制：C语言中无二进制数的表示方法；

八进制：以0开头，如0144；

十进制：以非0开头，如100；

十六进制：以0x开头，如0x64；



# C Puzzles 30——scanf的格式

```c
#include <stdio.h>
int main()
{
    int day,month,year;
    printf("Enter the date (dd-mm-yyyy) format including -'s:");
    scanf("%d-%d-%d",&day,&month,&year);
    printf("The date you have entered is %d-%d-%d\n",day,month,year);
    return 0;
}
```

## 题目讲解

scanf函数的定义为：

```c
int scanf(const char *format,…);
```

scanf会按照format指定的形式从标准输入读入数据到变量中。

```c
scanf("%d-%d-%d",&day,&month,&year);
```

当标准输入为“1-2-3”时，day=1,month=2,year=3；

当标准输入为“1,2,3”时，day=1,month和year为随机值。



# C Puzzles 32——位移<<的优先级

```c
#include <stdio.h>
#define PrintInt(expr) printf("%s : %d\n",#expr,(expr))
int FiveTimes(int a)
{
    int t;
    t = a<<2 + a;
    return t;
}

int main()
{
    int a = 1, b = 2,c = 3;
    PrintInt(FiveTimes(a));
    PrintInt(FiveTimes(b));
    PrintInt(FiveTimes(c));
    return 0;
}
```

## 题目讲解

在函数FiveTimes中

```c
t = a<<2 + a;
```

`+`的优先级高于`<<`，应该改为：

```c
t = (a << 2) + a;
```

# C Puzzles 33——return不能写在选择式子中

```c
#include <stdio.h>
#define PrintInt(expr) printf("%s : %d\n",#expr,(expr))
int max(int x, int y)
{
    (x > y) ? return x : return y;
}

int main()
{
    int a = 10, b = 20;
    PrintInt(a);
    PrintInt(b);
    PrintInt(max(a,b));
}
```

## 题目讲解

上述代码会出现编译错误，应该改为：

```c
return (x > y) ? x : y;
```



# C Puzzles 34——只修改一个字符

只修改一个字符，使下面程序成功运行。有三种修改方式

```c
 #include <stdio.h>
 int main()
 {
    int i;
    int n = 20;
    for( i = 0; i < n; i-- )
        printf("-");
    return 0;
}
```

## 题目讲解

```c
for( i = 0; -i < n; i-- )
```

```c
for( i = 0; i < n; n-- )
```

```c
for( i = 0; i + n; i-- )
```



# C Puzzles 35——声明中的int* 

```c
#include <stdio.h>
int main()
{
    int* ptr1,ptr2;
    ptr1 = malloc(sizeof(int));
    ptr2 = ptr1;
    *ptr2 = 10;
    return 0;
}
```

## 题目讲解

```c
int* ptr1,ptr2;
```

在声明中，int* 不能表示为声明一个整形指针。

改成下述模式：

```c
int *ptr1,*ptr2;
```



# C Puzzles 36——&&与||条件的运算

```c
#include <stdio.h>
int main()
{
    int i = 6;
    if( ((++i < 7) && ( i++/6)) || (++i <= 9))
        ;
    printf("%d\n",i);
    return 0; 
}
```

## 题目解答

i的值为8。先执行(++i < 7)，此表达式的值为0，i=7，由于逻辑运算符的短路处理，(i++/6)跳过执行，((++i < 7) && ( i++/6))值为0，接着执行(++i <= 9)，i的值最终为8。

# C Puzzles 38

```c
#include <stdlib.h>
#include <stdio.h>
#define SIZE 15 
int main()
{
    int *a, i;

    a = malloc(SIZE*sizeof(int));

    for (i=0; i<SIZE; i++)
        *(a + i) = i * i;
    for (i=0; i<SIZE; i++)
        printf("%d\n", *a++);
    free(a);
    return 0;
}
```

## 题目讲解

printf打印的时候，a指针已经移动了。

需要注意的是：

**malloc和free必须使用的是相同的内存地址，模块。**

# C Puzzles 39——a[0]与0[a]

```c
 #include <stdio.h>
  int main()
  {
    int a=3, b = 5;

    printf(&a["Ya!Hello! how is this? %s\n"], &b["junk/super"]);
    printf(&a["WHAT%c%c%c  %c%c  %c !\n"], 1["this"],
       2["beauty"],0["tool"],0["is"],3["sensitive"],4["CCCCCC"]);
    return 0;
  }
```

## 题目讲解

对于数组中的元素，最常见的表示方法是：地址[偏移]，如a[0]，a[1]，a[2]。还有一种不常见的表示方法是：偏移[地址]，如0[a]，1[a]，2[a]。

举例：

```c
#include <stdio.h>
int main()
{
	int a[] = {0, 1, 2};
	printf(“%d, %d, %d\n”, 0[a], 1[a], 2[a]);
	return 0;
}
```

运行结果为：0, 1, 2

```c
printf(“%d, %d, %d\n”, 0[a], 1[a], 2[a]);
```

等效于

```c
printf(“%d, %d, %d\n”, a[0], a[1], a[2]);
```

# C Puzzles 40

```c
#include <stdio.h>
int main()
{
    char dummy[80];
    printf("Enter a string:\n");
    scanf("%[^a]",dummy);
    printf("%s\n",dummy);
    return 0;
}
```

## 知识点讲解

scanf格式控制符：

%[...]：读取字符串，直到遇到不是[]中的字符；

%[\^...]：读取字符串，直到遇到[]中的字符。

如：

scanf("%[a-z]",dummy);输入为”abc123”时，dummy为”abc”；

scanf("%[\^a-z]",dummy);输入为”123abc”时，dummy为”123”；

## 题目讲解

“%[\^a]”表示读取字符串，直到遇到字符’a’。所以当输入为”Life is beautiful”时，dummy为”Life is be”。



# C Puzzles 42——计算结构体的偏移

```c
#define offsetof(a,b) ((int)(&(((a*)(0))->b)))
```

## 题目讲解

计算结构体成员变量在结构体中的偏移。a为结构体类型，b为成员变量。



# C Puzzles 43——异或^=

```c
#define SWAP(a,b) ((a) ^= (b) ^= (a) ^= (b))
```

## 知识点讲解

1）a和b不能是同一个变量，即如果执行SWAP(a, a)那么不管原来a值是多少，执行后a值被置为0； 

2）a和b不能是浮点数，异或操作对浮点数没有意义； 

3）a和b不能是结体体等复合数据类型，原因同上； 

4）a或b不能是表达式；



# C Puzzles 45——检测溢出

```c
int IAddOverFlow(int* result,int a,int b)
{
      /* ... */
}
```



# C Puzzles 46——内存对齐

```c
#define ROUNDUP(x,n) ((x+n-1)&(~(n-1)))
```

用于内存对齐，n为2的幂。

# C Puzzles 47——检测是否为大写字母

很多书籍中提及，检测是否为大写字母：

```c
#define isupper(c) (((c) >= 'A') && ((c) <= 'Z'))
```

但是一旦出现如下使用写法，会导致结果出现问题：

```c
 char c;
  /* ... */
  if(isupper(c++))
  {
      /* ... */
  }
```

## 知识点讲解

Linux内核中isupper的实现见lib/ctype.c和include/linux/ctype.h。

ctype.h代码如下：

```c
#ifndef _LINUX_CTYPE_H
#define _LINUX_CTYPE_H

/*
 * NOTE! This ctype does not handle EOF like the standard C
 * library is required to.
 */

#define _U	0x01	/* upper */
#define _L	0x02	/* lower */
#define _D	0x04	/* digit */
#define _C	0x08	/* cntrl */
#define _P	0x10	/* punct */
#define _S	0x20	/* white space (space/lf/tab) */
#define _X	0x40	/* hex digit */
#define _SP	0x80	/* hard space (0x20) */

extern const unsigned char _ctype[];

#define __ismask(x) (_ctype[(int)(unsigned char)(x)])

#define isalnum(c)	((__ismask(c)&(_U|_L|_D)) != 0)
#define isalpha(c)	((__ismask(c)&(_U|_L)) != 0)
#define iscntrl(c)	((__ismask(c)&(_C)) != 0)
#define isdigit(c)	((__ismask(c)&(_D)) != 0)
#define isgraph(c)	((__ismask(c)&(_P|_U|_L|_D)) != 0)
#define islower(c)	((__ismask(c)&(_L)) != 0)
#define isprint(c)	((__ismask(c)&(_P|_U|_L|_D|_SP)) != 0)
#define ispunct(c)	((__ismask(c)&(_P)) != 0)
/* Note: isspace() must return false for %NUL-terminator */
#define isspace(c)	((__ismask(c)&(_S)) != 0)
#define isupper(c)	((__ismask(c)&(_U)) != 0)
#define isxdigit(c)	((__ismask(c)&(_D|_X)) != 0)

#define isascii(c) (((unsigned char)(c))<=0x7f)
#define toascii(c) (((unsigned char)(c))&0x7f)

static inline unsigned char __tolower(unsigned char c)
{
	if (isupper(c))
		c -= 'A'-'a';
	return c;
}

static inline unsigned char __toupper(unsigned char c)
{
	if (islower(c))
		c -= 'a'-'A';
	return c;
}

#define tolower(c) __tolower(c)
#define toupper(c) __toupper(c)

#endif
```

ctype.c代码如下：

```c
/*
 *  linux/lib/ctype.c
 *
 *  Copyright (C) 1991, 1992  Linus Torvalds
 */

#include <linux/ctype.h>
#include <linux/module.h>

const unsigned char _ctype[] = {
_C,_C,_C,_C,_C,_C,_C,_C,				/* 0-7 */
_C,_C|_S,_C|_S,_C|_S,_C|_S,_C|_S,_C,_C,			/* 8-15 */
_C,_C,_C,_C,_C,_C,_C,_C,				/* 16-23 */
_C,_C,_C,_C,_C,_C,_C,_C,				/* 24-31 */
_S|_SP,_P,_P,_P,_P,_P,_P,_P,				/* 32-39 */
_P,_P,_P,_P,_P,_P,_P,_P,				/* 40-47 */
_D,_D,_D,_D,_D,_D,_D,_D,				/* 48-55 */
_D,_D,_P,_P,_P,_P,_P,_P,				/* 56-63 */
_P,_U|_X,_U|_X,_U|_X,_U|_X,_U|_X,_U|_X,_U,		/* 64-71 */
_U,_U,_U,_U,_U,_U,_U,_U,				/* 72-79 */
_U,_U,_U,_U,_U,_U,_U,_U,				/* 80-87 */
_U,_U,_U,_P,_P,_P,_P,_P,				/* 88-95 */
_P,_L|_X,_L|_X,_L|_X,_L|_X,_L|_X,_L|_X,_L,		/* 96-103 */
_L,_L,_L,_L,_L,_L,_L,_L,				/* 104-111 */
_L,_L,_L,_L,_L,_L,_L,_L,				/* 112-119 */
_L,_L,_L,_P,_P,_P,_P,_C,				/* 120-127 */
0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,			/* 128-143 */
0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,			/* 144-159 */
_S|_SP,_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,	/* 160-175 */
_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,_P,	/* 176-191 */
_U,_U,_U,_U,_U,_U,_U,_U,_U,_U,_U,_U,_U,_U,_U,_U,	/* 192-207 */
_U,_U,_U,_U,_U,_U,_U,_P,_U,_U,_U,_U,_U,_U,_U,_L,	/* 208-223 */
_L,_L,_L,_L,_L,_L,_L,_L,_L,_L,_L,_L,_L,_L,_L,_L,	/* 224-239 */
_L,_L,_L,_L,_L,_L,_L,_P,_L,_L,_L,_L,_L,_L,_L,_L};	/* 240-255 */

EXPORT_SYMBOL(_ctype);
```

_ctype数组中的每个元素的值对应ASCII码为0-255的每个字符的类型，以字符的ASCII码为索引即可获取到相应字符的类型。

# C Puzzles 48——不定参数

## 知识点讲解

对于带有不定参数的函数，用va_start，va_arg，va_end对参数进行索引，内核代码中va_*系列函数的实现在include/acpi/platform/acenv.h中

```c
#ifndef va_arg

#ifndef _VALIST
#define _VALIST
typedef char *va_list;
#endif				/* _VALIST */

/*
 * Storage alignment properties
 */
#define  _AUPBND                (sizeof (acpi_native_int) - 1)
#define  _ADNBND                (sizeof (acpi_native_int) - 1)

/*
 * Variable argument list macro definitions
 */
#define _bnd(X, bnd)            (((sizeof (X)) + (bnd)) & (~(bnd)))
#define va_arg(ap, T)           (*(T *)(((ap) += (_bnd (T, _AUPBND))) - (_bnd (T,_ADNBND))))
#define va_end(ap)              (void) 0
#define va_start(ap, A)         (void) ((ap) = (((char *) &(A)) + (_bnd (A,_AUPBND))))

#endif				/* va_arg */
```

由如上定义可知，va_start返回第一个不定参数的地址，va_arg返回下一个不定参数的地址，va_end用来销毁ap。va_start和va_end总是成对使用。

va_start获取第一个不定参数的地址时，必须知道最后一个固定参数A的地址，即函数必须提供至少一个固定参数。故定义形如VarArguments(...)的函数是不正确的。

# C Puzzles 49——不适用对比符号，查找最小值

```c
int smallest(int x, int y, int z)
{
  int c = 0;
  while ( x && y && z )
  {
      x--;  y--; z--; c++;
  }
  return c;
}
```

```c
#define CHAR_BIT 8

/*Function to find minimum of x and y*/
int min(int x, int y)
{
  return  y + ((x - y) & ((x - y) >>
            (sizeof(int) * CHAR_BIT - 1)));
}

/* Function to find minimum of 3 numbers x, y and z*/
int smallest(int x, int y, int z)
{
    return min(x, min(y, z));
}
```

```c
// Using division operator to find minimum of three numbers
int smallest(int x, int y, int z)
{
    if (!(y/x))  // Same as "if (y < x)"
        return (!(y/z))? y : z;
    return (!(x/z))? x : z;
}
```

# C Puzzles 50——不使用+实现加法

```c
int Add(int x, int y)
{
    // Iterate till there is no carry  
    while (y != 0)
    {
        // carry now contains common set bits of x and y
        int carry = x & y;  
 
        // Sum of bits of x and y where at least one of the bits is not set
        x = x ^ y; 
 
        // Carry is shifted by one so that adding it to x gives the required sum
        y = carry << 1;
    }
    return x;
}
```

```c
int Add(int x, int y)
{
    if (y == 0)
        return x;
    else
        return Add( x ^ y, (x & y) << 1);
}
```

# C Puzzles 54——memcpy和memmove区别

## 题目讲解

 他们的作用是一样的，唯一的区别是，当内存发生局部重叠的时候，memmove保证拷贝的结果是正确的，memcpy不保证拷贝的结果的正确。 

