# PostgreSQL的对齐宏——TYPEALIGN

## 分析

在理解TYPEALIGN之前现理解一下这个词语的意义，TYPEALIGN乍一看就是一个合成词，TYPE + ALGN。

> ## align
>
> 美 [ə'laɪn]
>
> 英 [ə'laɪn]
>
> - **v.**排列整齐；使对齐；（尤指）成一直线；使一致
> - **网络**对齐方式；调整；对准
>
> align
>
> 显示所有例句
>
> v.
>
> | 1.   | [i][t]~ (sth) (with sth)排列整齐；使对齐；（尤指）成一直线to arrange sth in the correct position, or to be in the correct position, in relation to sth else, especially in a straight line |
> | ---- | ------------------------------------------------------------ |
> |      |                                                              |
>
> | 2.   | [t]~ sth (with/to sth)使一致to change sth slightly so that it is in the correct relationship to sth else |
> | ---- | ------------------------------------------------------------ |
> |      |                                                              |

通过将词语拆分可以看出这个宏的主要某地就是将不同类型所占的长度一一对齐。

在```aset.c```文件```AllocSetContextCreateInternal```函数有这样一行函数

```c
block = (AllocBlock) (((char *)set) + MAXALIGN(sizeof(AllocSetContext)));
```



首先```MAXALIGN```这个宏定义在```c.h```中出现：

```c
/* Define as the maximum slignment requirement of any data type */
#define MAXALIGN(LEN)	TYPEALIGN(MAXIMUM_ALIGNOF, (LEN))
```

上述```MAXIMUM_ALIGNOF```在```pg_config.h```中定义，这个```pg_config.h```比较特殊，它是在```configure```时候自动生成的。这个头文件里面的定义都和编译环境与```configure```时候设置的参数息息相关。这个```MAXIMUM_ALIGNOF```就表示编译器中最大.位的基本类型，比如```long long```类型。不同的编译环境可能导致这个数值存在差异。

接下来分析一下```TYPEALIGN```这个宏定义：

```c
/* ----------------
 * Alignment macros: align a length or address appropriately for a given type.
 * The fooALIGN() macros round up to a multiple of the required alignment,
 * while the fooALIGN_DOWN() macros round down.  The latter are more useful
 * for problems like "how many X-sized structures will fit in a page?".
 *
 * NOTE: TYPEALIGN[_DOWN] will not work if ALIGNVAL is not a power of 2.
 * That case seems extremely unlikely to be needed in practice, however.
 *
 * NOTE: MAXIMUM_ALIGNOF, and hence MAXALIGN(), intentionally exclude any
 * larger-than-8-byte types the compiler might have.
 * ----------------
 */

#define TYPEALIGN(ALIGNVAL,LEN)  \
    (((uintptr_t) (LEN) + ((ALIGNVAL) - 1)) & ~((uintptr_t) ((ALIGNVAL) - 1)))

#define SHORTALIGN(LEN)         TYPEALIGN(ALIGNOF_SHORT, (LEN))
#define INTALIGN(LEN)           TYPEALIGN(ALIGNOF_INT, (LEN))
#define LONGALIGN(LEN)          TYPEALIGN(ALIGNOF_LONG, (LEN))
#define DOUBLEALIGN(LEN)        TYPEALIGN(ALIGNOF_DOUBLE, (LEN))
#define MAXALIGN(LEN)           TYPEALIGN(MAXIMUM_ALIGNOF, (LEN))
/* MAXALIGN covers only built-in types, not buffers */
#define BUFFERALIGN(LEN)        TYPEALIGN(ALIGNOF_BUFFER, (LEN))
#define CACHELINEALIGN(LEN)     TYPEALIGN(PG_CACHE_LINE_SIZE, (LEN))

```

接下来进行分析，64为系统ALIGNVAL的值是8，二进制值为```1000```。8-1的二进制值为```0111```；```AllocSetContext```的大小为216，216+8-1 = 223。随后将223与非7进行按位与。

```
1 1 0 1 | 1 1 1 1
1 1 1 1 | 1 0 0 0
——————————————————
1 1 0 1 | 1 0 0 0  =  216
```

## 为什么这么计算呢

1.为什么要对ALIGNVAL减一，并且进行按位与操作？

首先对于与操作来说，1意味着保持不变，0意味着全部制0。既然要想使每个数字都能成为8的倍数，就很自然的让后三位全部变为0。这样一来通过和非7进行与操作，就可以保证后三位直接被舍弃。

而只舍弃后三位还不够，还需要将原有的数字扩大后在进行舍弃。（就是所谓的余数进位原则）即只要余数不为0，就需要添加额外的位数补齐。这里之所以+7是因为7是整数被8整除以后的最大余数。无论原有的整数被8整除余数是几，加7以后一定会满足先前所谓补位的需求，此时再通过和非7的与运算直接舍弃后三位，就可以保证按8为单位进行补齐。



2.TYPEALIGN的意义是什么？

这个宏的意义就是将所有整数进行TYPEALIGN为基本单位的补齐，简便后续内存分配的优化算法。不同的运算环境中TYPEALIGN的值有所不同，此种灵活的实现方法保证了在所有的环境下都可以进行位数的对齐。