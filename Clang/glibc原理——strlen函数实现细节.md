## 写在前面

strlen函数式用来获取某个字符串长度的函数，在日常开发的过程中使用的频率极高，有时候需要考虑一下，函数内部究竟是如何实现的。首先看K&R版本的实现：

## strlen函数实现细节——K&R

```c
int strlen(char *s)
{
		char *p = s;
  	while (*p != '\0')
    		p++;
  	return p - s;
}
```

K&R版本的实现比较贴近日常的开发思路，通过一个指针按位移动，直到查找到```\0```返回指针偏移量。

## strlen函数实现细节——glibc

后感觉深层次的strlen应该不至于和K&R版本一样平淡，特地来研究一下glibc作者们的杰作，这其中有不少技巧需要学习。

```c
size_t strlen(const char *str)
{
    const char *char_ptr;
    const unsigned long int *longword_ptr;
    unsigned long int longword, magic_bits, himagic, lomagic;

    for (char_ptr = str; ((unsigned long int) char_ptr 
             & (sizeof (longword) - 1)) != 0; ++char_ptr)
       if (*char_ptr == '\0')
           return char_ptr - str;

    longword_ptr = (unsigned long int *) char_ptr;

    himagic = 0x80808080L;
    lomagic = 0x01010101L;

    for (;;)
    { 
        longword = *longword_ptr++;

        if (((longword - lomagic) & ~longword & himagic) != 0)
        {
            const char *cp = (const char *) (longword_ptr - 1);

            if (cp[0] == 0)
                return cp - str;
            if (cp[1] == 0)
                return cp - str + 1;
            if (cp[2] == 0)
                return cp - str + 2;
            if (cp[3] == 0)
                return cp - str + 3;
        }
    }
}
```

### 实现思路

>考虑到计算机按照不同类型获取内存数据会有不同的结果，这些不同结果的本质内存数据样式是不会变的。所以如果每次用char类型的“模式”获取数据，每次取得内存中的数据较少。但是如果按照unsigned long类型的“模式”获取，每次获取的数据一下子就变多了。
>
>相较于传统单字节获取判断，一次性获取多个字节判断效率必然有所提升。另一层面，如果数据的获取能够和数据地址能够对齐，那么CPU存取内存数据的效率也会相应提高。
>
>也就是说，整个strlen标准库围绕的核心就是2个：
>
>1.每次获取的数据变多。
>
>2.每次的获取争取数据对齐。

标准库的大致流程如下：

1.按照字符依次判断，直到内存对齐。如果内存对齐之前就遇到了至关重要的```\0```，则直接返回。否则来到步骤2。

2.每次读入并判断一个WORD，如果这个WORD没有为```\0```的字节，则继续下一个WORD，否则来到步骤3。

3.此WORD中至少有一个字节为```\0```，找到第一个```\0```的字节的位置。

### 核心点1——先找到数据对齐的位置(data alignment)

[内存对齐](../Unix/数据对齐.md)是为了提高处理器访问内存的效率。如果一个变量的内存地址刚好位于它本身长度的整数倍（也就是说获取这个变量不跨“区”，至少是头部附近不跨区）就叫做自然对齐。

当传入判断的字符串长度很长时，每次按照基本类型（unsigned long int）获取数据时，希望每次获取的数据都能在一个获取周期内完成。而不是每个获取的数据（按照unsigned long int类型）都因为数据不对齐而获取多次。

```c
    for (char_ptr = str; ((unsigned long int) char_ptr 
             & (sizeof (longword) - 1)) != 0; ++char_ptr)
       if (*char_ptr == '\0')
           return char_ptr - str;
```

上述代码的主要目的就是帮助str字符串这块地址找到首个按照unsigned long int类型内存对齐的位置，也就是说不能保证这个str字符串这块内存地址的位置不一定能够刚好落在内存对齐的地址上。

需要先通过和N-1进行**与**计算，找到str字符串中首个内存位置对齐的位置。然后再按照每次获取一个unsigned long int的方式进行获取。

>原理：
>
>​		判断某个数能否被N（N = 2^n）整除，则查看这个数余除N(%)是否为0。如果为0，则一定可以整除；如果不为零则一定有余数。
>
>​		假设N为8，二进制表示为0x1000，很巧合的是，如果最后三位出现1，则这个数一定无法被8整除。所以上述代码中需要和N-1进行与计算。

在上述寻找内存对齐的位置中，如果直接找到了```\0```，就直接返回。

### 核心点2——判断每个数据中是否有```\0```

一切比较复杂的问题，都可以从复杂的情况转变为简单的情况。现在不妨分析简单的情况：

```c
/**
 * himagic      : 1000 0000
 * lomagic      : 0000 0001
 * longword     : XXXX XXXX
 * /
unsigned long himagic = 0x80L;
unsigned long lomagic = 0x01L;

unsigned long longword ;
```

 随后我们仔细分析下面公式

```c
((longword - lomagic) & ~longword & himagic)
```

( & himagic ) = ( & 1000 0000) 表明最终只在乎最高位. 

longword 分三种情况讨论

```c
longword     : 1XXX XXXX  128 =< x <= 255
longword     : 0XXX XXXX  0 < x < 128
longword     : 0000 0000  x = 0
```

**第一种** longword = 1XXX XXXX 

​       那么 ~longword = 0YYY YYYY 显然 ~ longword & himagic = 0000 0000 不用继续了.

**第二种** longword = 0XXX XXXX 且不为 0, 及不小于 1

​       显然 (longword - lomagic) = 0ZZZ ZZZ >= 0 且 < 127, 因为 lomagic = 1; 

​       此刻 (longword - lomagic) & himagic = 0ZZZ ZZZZ & 1000 0000 = 0 , 所以也不需要继续了.

**第三种** longword = 0000 0000

​       那么 ~longword & himagic = 1111 1111 & 1000 0000 = 1000 000;

​       再看 (longword - lomagic) = (0000 0000 - 0000 0001) , 由于无符号数减法是按照

​       (补码(0000 0000) + 补码(-000 0001)) = (补码(0000 0000) + 补码(~000 0001 + 1))

​       = (补码(0000 0000) + 补码(1111 1111)) = 1111 1111 (快捷的可以查公式得到最终结果),

​       因而 此刻最终结果为 1111 1111 & 1000 0000 = 1000 0000 > 0.

综合讨论, 可以根据上面公式巧妙的筛选出值是否为 ```\0```. 对于 2字节, 4 字节, 8 字节, 思路完全相似. 

