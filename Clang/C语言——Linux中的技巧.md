# **Linux 内核中的 C 语言语法扩展**

在 Linux 内核源码中，有大量的 C 程序看起来“怪怪的”。说它是C语言吧，貌似又跟教材中的写法不太一样；说它不是 C 语言呢，但是这些程序确确实实是在一个 C 文件中。此时，你肯定怀疑你看到的是一个“假的 C 语言”！

比如，下面的宏定义：

```c
#define mult_frac(x, numer, denom)(            \
	{                            \    
		typeof(x) quot = (x) / (denom);         \    
		typeof(x) rem  = (x) % (denom);         \    
		(quot * (numer)) + ((rem * (numer)) / (denom));    \
	}                            \
)
        
#define ftrace_vprintk(fmt, vargs)                    \
do {                                    \    
	if (__builtin_constant_p(fmt)) {                \        
		static const char *trace_printk_fmt __used      \      
		__attribute__((section("__trace_printk_fmt"))) =  \            
		__builtin_constant_p(fmt) ? fmt : NULL;     \
											\        
		__ftrace_vbprintk(_THIS_IP_, trace_printk_fmt, vargs);  \    
	} else                              \        
		__ftrace_vprintk(_THIS_IP_, fmt, vargs);        \
} while (0)
```

字符驱动的填充：

```c
static const struct file_operations lowpan_control_fops = 
{    
    .open        = lowpan_control_open,    
    .read        = seq_read,    
    .write        = lowpan_control_write,    
    .llseek        = seq_lseek,    
    .release    = single_release,    
};
```

内核中实现打印功能的宏定义：

```c
#define pr_info(fmt, ...)    pr(pr_info, fmt, ##VA_ARGS)
#define pr_debug(fmt, ...)    pr(pr_debug, fmt, ##VA_ARGS)
```


你没有看错，这些其实也是 C 语言，但并不是标准的 C 语言语法，而是我们 Linux 内核使用的 GNU C 编译器扩展的一些 C 语言语法。这些语法在 C 语言教材或资料中一般不会提及，所以你才会似曾相识而又感到陌生，看起来感觉“怪怪的”。我们在做 Linux 驱动开发，或者阅读 Linux 内核源码过程中，会经常遇到这些“稀奇古怪”的用法，如果不去了解这些特殊语法的具体含义，可能就对代码的理解造成一定障碍。

# 指定初始化结构体成员变量

 在 GNU C 中我们也可以通过结构域来初始化指定某个成员。 

```c
struct student
{    char name[20];    
	 int age;
};

int main(void)
{    
	struct student stu1={ "wit",20 };    
	printf("%s:%d\n",stu1.name,stu1.age);    
	struct student stu2=    
	{        
		.name = "wanglitao",        
		.age  = 28    
	};    
	printf("%s:%d\n",stu2.name,stu2.age);    
	return 0;
}
```

## Linux 内核驱动注册

 在 Linux 内核驱动中，大量使用 GNU C 的这种指定初始化方式，通过结构体成员来初始化结构体变量。比如在字符驱动程序中，我们经常见到这样的初始化： 

```c
static const struct file_operations ab3100_otp_operations = 
{
		.open        = ab3100_otp_open,
		.read        = seq_read,
		.llseek        = seq_lseek,
		.release    = single_release,
		
		... ...
};
```

 在驱动程序中，我们经常使用 `file_operations` 这个结构体变量来注册我们开发的驱动，然后以回调的方式来执行我们驱动实现的相关功能。结构体 `file_operations` 在 Linux 内核中的定义如下： 

```c
struct file_operations 
{        
	struct module *owner;        
	loff_t (*llseek) (struct file *, loff_t, int);        
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, 
                      			size_t, loff_t *); 
    
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);        
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);        
	int (*iterate) (struct file *, struct dir_context *);        
	unsigned int (*poll) (struct file *, struct poll_table_struct *);        
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);        
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);        
	int (*mmap) (struct file *, struct vm_area_struct *);        
	int (*open) (struct inode *, struct file *);        
	int (*flush) (struct file *, fl_owner_t id);        
	int (*release) (struct inode *, struct file *);        
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);        
	int (*aio_fsync) (struct kiocb *, int datasync);        
	int (*fasync) (int, struct file *, int);        
	int (*lock) (struct file *, int, struct file_lock *);        
	ssize_t (*sendpage) (struct file *, struct page *, 
                         		int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *,              
						unsigned long, unsigned long, unsigned long, 
												unsigned long);   
											
	int (*check_flags)(int);        
	int (*flock) (struct file *, int, struct file_lock *);   
	
	ssize_t (*splice_write)(struct pipe_inode_info *,             
					struct file *, loff_t *, size_t, unsigned int);  
							
	ssize_t (*splice_read)(struct file *, loff_t *, 
					struct pipe_inode_info *, size_t, unsigned int); 
						
	int (*setlease)(struct file *, long, struct file_lock **, void **);  
	
	long (*fallocate)(struct file *file, int mode, loff_t offset,                					loff_t len);   
    
	void (*show_fdinfo)(struct seq_file *m, struct file *f);   
	
#ifndef CONFIG_MMU        
	unsigned (*mmap_capabilities)(struct file *);        
#endif    
};
```

## 指定初始化的好处
这种指定初始化方式，不仅使用灵活，而且还有一个好处就是：代码易于维护。尤其是在 Linux 内核这种大型项目中，几万个文件，几千万的代码量，当成百上千个文件都使用 file_operations 这个结构体类型来定义变量并初始化时，那么一个很大的问题就来了：如果采用标准 C 那种按照固定顺序赋值，当我们的 file_operations 结构体类型发生改变时，如添加成员、减少成员、调整成员顺序，那么使用该结构体类型定义变量的大量 C 文件都需要重新调整初始化顺序，牵一发而动全身，想想这是多么可怕！

我们通过指定初始化方式，就可以避免这个问题。无论file_operations 结构体类型如何变化，添加成员也好、减少成员也好、调整成员顺序也好，都不会影响其它文件的使用。




# 宏——语句表达式

## 语句表达式

 GNU C 对 C 标准作了扩展，允许在一个表达式里内嵌语句，允许在表达式内部使用局部变量、for 循环和 goto 跳转语句。这样的表达式，我们称之为**语句表达式**。语句表达式的格式如下： 

```c
({ 表达式1; 表达式2; 表达式3; })
```

 语句表达式最外面使用小括号()括起来，里面一对大括号{}包起来的是代码块，代码块里允许内嵌各种语句。语句的格式可以是 “表达式;”这种一般格式的语句，也可以是循环、跳转等语句。 

```c
int main(void)
{    
	int sum = 0;    
	sum =     
	(
		{        
			int s = 0;        
			for( int i = 0; i < 10; i++)            
				s = s + i;            
			s;    
		}
	);    
	printf("sum = %d\n",sum);    
	return 0;
}

```

在上面的程序中，通过语句表达式实现了从1到10的累加求和，因为**语句表达式的值等于最后一个表达式的值**，所以在 for 循环的后面，我们要添加一个 s; 语句表示整个语句表达式的值。如果不加这一句，你会发现 sum=0。或者你将这一行语句改为100; 你会发现最后 sum 的值就变成了100，这是因为语句表达式的值总等于最后一个表达式的值。

### 语句表达式使用goto跳转

 在上面的程序中，我们在语句表达式内定义了局部变量，使用了 for 循环语句。在语句表达式内，我们同样也可以使用 goto 进行跳转。 

```c
int main(void)
{    
	int sum = 0;    
	sum =     
	(
		{        
			int s = 0;        
			for( int i = 0; i < 10; i++)            
				s = s + i;            
			goto here;            
			s;      
		}
	);    
	printf("sum = %d\n",sum);
here:    printf("here:\n");    
	printf("sum = %d\n",sum);    
	return 0;
}
```

## 在宏中使用语句表达式

比较两个数字的大小：

```c
#define MAX(type,x,y) 		\
({     						\    
		type _x = x;        \    
		type _y = y;        \    
		_x > _y ? _x : _y; 	\
})
#include<stdio.h>
int main()
{    
	int i = 2;    
	int j = 6;    
	printf("max=%d\n",MAX(int,i++,j++));    
	printf("max=%f\n",MAX(float,3.14,3.15));    
	return 0;
}
```

在这个宏中，我们添加一个参数：type，用来指定临时变量 _x 和 _y 的类型。这样，我们在比较两个数的大小时，只要将2个数据的类型作为参数传给宏，就可以比较任意类型的数据了。语句表达式，作为 GNU C 对 C 标准的一个扩展，在内核中，尤其是在内核的宏定义中，被大量的使用。使用语句表达式定义宏，不仅可以实现复杂的功能，还可以避免宏定义带来的一些歧义和漏洞。比如在 Linux 内核中，`max_t` 和 `min_t` 的宏定义，就使用了语句表达式： 

```c
#define min_t(type, x, y) ({            \    
			type __min1 = (x);          \    
			type __min2 = (y);          \    
			__min1 < __min2 ? __min1 : __min2; })
#define max_t(type, x, y) ({            \    
			type __max1 = (x);          \    
			type __max2 = (y);          \    
			__max1 > __max2 ? __max1 : __max2; })
```

# Linux中很常见的一个宏contain_of

 container_of在Linux内核中是一个常用的宏，用于从包含在某个结构中的指针获得结构本身的指针，通俗地讲就是通过结构体变量中某个成员的首地址进而获得整个结构体变量的首地址。 



实现方式：
	```container_of(ptr, type, member) ;```

- ```ptr```：指向结构体内成员的指针。
- ```Type```：结构体的类型。
- ```Member```：结构体成员的名字，和ptr是同一个成员

其实它的语法很简单，只是一些指针的灵活应用，它分两步：

  第一步，首先定义一个临时的数据类型（通过```typeof( ((type *)0)->member)```获得与ptr相同的指针变量__mptr，然后用它来保存ptr的值。

  第二步，用(char *)__mptr减去member在结构体中的偏移量，得到的值就是整个结构体变量的首地址（整个宏的返回值就是这个首地址）。

```c
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
#define  container_of(ptr, type, member) ({                      \
                const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
               (type *)( (char *)__mptr - offsetof(type,member) );})
```

假设有这么一个结构体：

```c
Struct student{
	Type1 A;
	Type2 B;
	Type3 C;
};
```

```c
/* 
 *	
 */
define container_of(pointer, struct student, B) ({ \
		const typeof( (( struct student*)0)->B) *__mptr = (pointer);    \
		(struct student*)( (char *)__mptr - offsetof( struct student,B) );})
```

```const typeof( (( struct student*)0)->B) *__mptr = (pointer);```

这句话表示声明了一个指针```_mptr```。typeof获取了变量```pointer```的类型。这个```_mptr```就和pointer的类型直接对应了。也就是说```_mptr```就和pointer指向了同一块内存。

```c
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

接下来，offsetof定义了某个结构体类型中，成员相对于结构体首地址的偏移量。这里的原理如下：

```((TYPE *) 0) -> MEMBER```理解的关键：

> ​	首先，这个offset_of宏的意义就在于获取到那个指针的偏移量。有一个比较冷门的理解，那就是```->```相当于从结构体的首地址开始，```->```指向了哪一个成员，它就从结构体的首地址往后偏移多少。那就可以通过结构体成员和结构体首地址作差，进而得到这个偏移量。
>
> ​	问题是我们不清楚结构体的首地址，这里就得用一些小技巧。
>
> 由于编译器的特点，```->```会自动偏移到MEMBER成员的地址上来。```((TYPE *) 0) -> MEMBER```就直接把把结构体```TYPE```的首地址扔到0上，这样一来```MEMBER```的地址直接就是偏移量了。需要注意的是，这个```TYPE```必须得是结构体的类型，这样```->```的取偏移量才能得到正确的值。
>

```c
#include <stdio.h>
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
#define  container_of(ptr, type, member) ({                      \
                      const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
                       (type *)( (char *)__mptr - offsetof(type,member) );})
struct test_struct {
           int num;
          char ch;
          float f1;
  };
 int main(void)
  {
          struct test_struct *test_struct;
          struct test_struct init_struct =
			{
					.num = 12,
					.ch = 'a',
					.f1 = 12.3
			};
          char *ptr_ch = &init_struct.ch;
          test_struct = container_of(ptr_ch,struct test_struct,ch);
          printf("test_struct->num =%d\n",test_struct->num);
          printf("test_struct->ch =%c\n",test_struct->ch);
          printf("test_struct->ch =%f\n",test_struct->f1);
          return 0;
  }

```



# 零长度数组/柔性数组/变长数组

按照旧版的规则，数组的长度必须是直接定义的。

```c
int a[10];
```

C99新标准规定，可以定义一个边长数组。

```c
int len;
int a[len];
```

更牛逼的是还可以定义一个0长度数组。（仅限GNU C编译器）。特点就是**不占用内存空间**。

零长度数组经常以变长结构体的形式，在某些特殊的应用场合，被程序员使用。在一个变长结构体中，零长度数组不占用结构体的存储空间，但是我们可以通过使用结构体的成员 a 去访问内存，非常方便。变长结构体的使用示例如下。

```c
struct buffer{
    int len;
    int a[0];
};
int main(void)
{
    struct buffer *buf;
    buf = (struct buffer *)malloc \
        (sizeof(struct buffer)+ 20);

    buf->len = 20;
    strcpy(buf->a, "hello wanglitao!\n");
    puts(buf->a);

    free(buf);  
    return 0;
}
```

在这个程序中，我们使用 malloc 申请一片内存，大小为 sizeof(buffer) + 20，即24个字节大小。其中4个字节用来存储结构体指针 buf 指向的结构体类型变量，另外20个字节空间，才是我们真正使用的内存空间。我们可以通过结构体成员 a，直接访问这片内存。

通过这种灵活的动态内存申请方式，这个 buffer 结构体表示的一片内存缓冲区，就可以随时调整，可大可小。这个特性，在一些场合非常有用。比如，现在很多在线视频网站，都支持多种格式的视频播放：普清、高清、超清、1080P、蓝光甚至4K。如果我们本地程序需要在内存中申请一个 buffer 用来缓存解码后的视频数据，那么，不同的播放格式，需要的 buffer 大小是不一样的。如果我们按照 4K 的标准去申请内存，那么当播放普清视频时，就用不了这么大的缓冲区，白白浪费内存。而使用变长结构体，我们就可以根据用户的播放格式设置，灵活地申请不同大小的 buffer，大大节省了内存空间。

## 零长度数组在内核中的使用

 在网卡驱动中，大家可能都比较熟悉一个名字：套接字缓冲区，即 socket buffer，用来传输网络数据包。同样，在 USB 驱动中，也有一个类似的东西，叫 URB，其全名为 USB request block，即 USB 请求块，用来传输 USB 数据包。

```c
struct urb {
    struct kref kref;
    void *hcpriv;
    atomic_t use_count;
    atomic_t reject;
    int unlinked;

    ... ...

    int error_count;
    void *context;
    usb_complete_t complete;
    struct usb_iso_packet_descriptor iso_frame_desc[0];
};
```

在这个结构体内定义了 USB 数据包的传输方向、传输地址、传输大小、传输模式等。这些细节我们不深究，我们只看最后一个成员：

```c
struct usb_iso_packet_descriptor iso_frame_desc[0];
```

在 URB 结构体的最后，定义一个零长度数组，主要用于 USB 的同步传输。USB 有4种传输模式：中断传输、控制传输、批量传输和同步传输。不同的 USB 设备对传输速度、传输数据安全性的要求不同，所采用的传输模式是不同的。USB 摄像头对视频或图像的传输实时性要求较高，对数据的丢帧不是很在意，丢一帧无所谓 ，接着往下传。所以 USB 摄像头采用的是 USB 同步传输模式。

现在淘宝上的 USB 摄像头，打开它的说明书，一般会支持多种分辨率：从16*16到高清720P多种格式。不同分辨率的视频传输，对于一帧图像数据，对 USB 的传输数据包的大小和个数需求是不一样的。那USB到底该如何设计，去适配这种不同大小的数据传输要求，但又不影响 USB 的其它传输模式呢？答案就在结构体内的这个零长度数组上。

当用户设置不同的分辨率传输视频，USB 就需要使用不同大小和个数的数据包来传输一帧视频数据。通过零长度数组构成的这个变长结构体就可以满足这个要求。可以根据一帧图像数据的大小，灵活地去申请内存空间，满足不同大小的数据传输。但这个零长度数组又不占用结构体的存储空间，当 USB 使用其它模式传输时，不受任何影响，完全可以当这个零长度数组不存在。所以，不得不说，这样的设计真是妙！

## 定长数组的使用模式

定长数组使用方便, 但是却浪费空间, 指针形式只多使用了一个指针的空间, 不会造成大量空间分浪费, 但是使用起来需要多次分配, 多次释放, 那么有没有一种实现方式能够既不浪费空间, 又使用方便的呢?

GNU C 的0长度数组, 也叫变长数组, 柔性数组就是这样一个扩展. 对于0长数组的这个特点，很容易构造出变成结构体，如缓冲区，数据包等等：

数据结构定义


```c
//  0长度数组
struct zero_buffer
{
    int     len;
    char    data[0];
};
```

数据结构大小
这样的变长数组常用于网络通信中构造不定长数据包, 不会浪费空间浪费网络流量, 因为char data[0]; 只是个数组名, 是不占用存储空间的,

即``` sizeof(struct zero_buffer) = sizeof(int)```

**开辟空间**
那么我们使用的时候, 只需要开辟一次空间即可

```c
///  开辟
if ((zbuffer = (struct zero_buffer *)malloc(sizeof(struct zero_buffer) + sizeof(char) * CURR_LENGTH)) != NULL)
{
    zbuffer->len = CURR_LENGTH;
    memcpy(zbuffer->data, "Hello World", CURR_LENGTH);
```


```c
    printf("%d, %s\n", zbuffer->len, zbuffer->data);
}
```
**释放空间**
释放空间也是一样的, 一次释放即可

```c
///  销毁
free(zbuffer);
zbuffer = NULL;
```

# \__attribute__

```c
void __attribute__((format(printf,1,2))) my_printf(char *fmt,...)
{
    va_list args;
    va_start(args,fmt);
    vprintf(fmt,args);
    va_end(args);
}
int main(void)
{
    int num = 0;
    my_printf("I am litao, I have %d car\n", num);
    return 0;
}
```

 GNU C 的一大特色就是__attribute__ 机制。__attribute__ 可以设置函数属性（Function Attribute ）、变量属性（Variable Attribute ）和类型属性（Type Attribute ）。 

 __attribute__ 书写特征是：__attribute__ 前后都有两个下划线，并切后面会紧跟一对原括弧，括弧里面是相应的__attribute__ 参数。 

__attribute__ 语法格式为：__attribute__ ((attribute-list))

关键字__attribute__ 也可以对结构体（struct ）或共用体（union ）进行属性设置。大致有六个参数值可以被设定，即：aligned, packed, transparent_union, unused, deprecated 和 may_alias 。

在使用__attribute__ 参数时，你也可以在参数的前后都加上“__” （两个下划线），例如，使用__aligned__而不是aligned ，这样，你就可以在相应的头文件里使用它而不用关心头文件里是否有重名的宏定义。

## \__attribute__ 的参数介绍

### aligned

指定对象的对齐格式（以字节为单位），如：

```objectivec
struct S { short b[3]; } __attribute__ ((aligned (8)));  typedef int int32_t __attribute__ ((aligned (8)));
```

该声明将强制编译器确保（尽它所能）变量类 型为struct S 或者int32_t 的变量在分配空间时采用8字节对齐方式。

如上所述，你可以手动指定对齐的格式，同样，你也可以使用默认的对齐方式。如果aligned 后面不紧跟一个指定的数字值，那么编译器将依据你的目标机器情况使用最大最有益的对齐方式。例如：

```objectivec
struct S { short b[3]; } __attribute__ ((aligned));
```

这里，如果sizeof （short ）的大小为2byte，那么，S 的大小就为6 。取一个2 的次方值，使得该值大于等于6 ，则该值为8 ，所以编译器将设置S 类型的对齐方式为8 字节。

aligned 属性使被设置的对象占用更多的空间，相反的，使用packed 可以减小对象占用的空间。

需要注意的是，attribute 属性的效力与你的连接器也有关，如果你的连接器最大只支持16 字节对齐，那么你此时定义32 字节对齐也是无济于事的。

### packed

 使用该属性对struct 或者union 类型进行定义，设定其类型的每一个变量的内存约束。就是告诉编译器取消结构在编译过程中的优化对齐（使用1字节对齐）,按照实际占用字节数进行对齐，是GCC特有的语法。这个功能是跟操作系统没关系，跟编译器有关，gcc编译器不是紧凑模式的，我在windows下，用vc的编译器也不是紧凑的，用tc的编译器就是紧凑的。

​    下面的例子中，packed_struct 类型的变量数组中的值将会紧紧的靠在一起，但内部的成员变量s 不会被“pack” ，如果希望内部的成员变量也被packed 的话，unpacked-struct 也需要使用packed 进行相应的约束。

```objectivec
struct unpacked_struct{      char c;      int i;};         struct packed_struct{     char c;     int  i;     struct unpacked_struct s;}__attribute__ ((__packed__));
```



###  \__attribute__ (format(printf,a,b)) 

 _attribute__ format属性可以给被声明的函数加上类似printf或者scanf的特征，它可以使编译器检查函数声明和函数实际调用参数之间的格式化字符串是否匹配。format属性告诉编译器，按照printf, scanf等标准C函数参数格式规则对该函数的参数进行检查。这在我们自己封装调试信息的接口时非常的有用。 

format的语法格式为：

```format (archetype, string-index, first-to-check)```

具体的使用如下所示：

__attribute__((format(printf, a, b)))

__attribute__((format(scanf, a, b)))

其中参数m与n的含义为：

　　　　a：第几个参数为格式化字符串(format string);

　　　　b：参数集合中的第一个，即参数“…”里的第一个参数在函数参数总数排在第几。

举例如下：

```c
#include <stdio.h>
#include <stdarg.h>

#if 1
#define CHECK_FMT(a, b)	__attribute__((format(printf, a, b)))
#else
#define CHECK_FMT(a, b)
#endif

void TRACE(const char *fmt, ...) CHECK_FMT(1, 2);
void TRACE(const char *fmt, ...)
{
	va_list ap;
	va_start(ap, fmt); 
	(void)printf(fmt, ap);
	va_end(ap);
}

int main(void)
{
	TRACE("iValue = %d\n", 6);
	TRACE("iValue = %d\n", "test");

	return 0;
}
```

# 内联函数

一般来说，调用一个函数的流程为：当前调用命令的地址被保存下来，程序跳转到所调用的函数并执行该函数，最后跳转回到所保存的命令地址。

```inline```关键字告诉编译器，任何地方主要调用内联函数，就直接把这个函数的机器码插入到调用的地方，这样程序执行更有效率。

 **内联函数其实就是普通函数，只不过它们在调用时采用机器码形式。**和普通函数一样，内联函数具有自己的地址。如果内联函数使用到宏，预处理器就会展开宏，展开时所用的宏值，取该内联函数在源代码中定义所在位置的宏值。然而，在没被声明为 static 的内联函数中，不应该以静态存储周期的方式来定义可修改的对象。 

# 可变参数宏

 在GNU C中，宏可以接受可变数目的参数，就象函数一样，例如:  

```c
#define pr_debug(fmt,arg...) \ 
printk(KERN_DEBUG fmt, ##arg)
```

 用可变参数宏(variadic macros)传递可变参数表 
你可能很熟悉在函数中使用可变参数表，如: 

```c
void  printf ( const  char * format, ...);
```

C99 可以定义可变参数宏(variadic macros)，这样你就可以使用拥有可以变化的参数表的宏。可变参数宏就像下面这个样子: 

```C
 #define debug(...) printf(VA_ARGS) 
```

 缺省号代表一个可以变化的参数表。使用保留名 __VA_ARGS__ 把参数传递给宏。当宏的调用展开时，实际的参数就传递给 printf()了。例如:  

```c
Debug( "Y = %d\n" , y);
```

 而处理器会把宏的调用替换成:  

```c
printf ( "Y = %d\n" , y);
```

 因为debug()是一个可变参数宏，你能在每一次调用中传递不同数目的参数:  

```c
debug( "test" );&nbsp; // 一个参数
```

新的C99规范支持了可变参数的宏 
具体使用如下:

以下内容为程序代码:


```c
#include <stdarg.h> 
#include <stdio.h> 
#define LOGSTRINGS(fm, ...) printf(fm,__VA_ARGS__) 
int  main()
{
     LOGSTRINGS( "hello, %d " , 10); 
     return  0;
}
```

```c
// variadic_macros.cpp
#include <stdio.h>
#define EMPTY

#define CHECK1(x, ...) if (!(x)) { printf(__VA_ARGS__); }
#define CHECK2(x, ...) if ((x)) { printf(__VA_ARGS__); }
#define CHECK3(...) { printf(__VA_ARGS__); }
#define MACRO(s, ...) printf(s, __VA_ARGS__)

int main() {
    CHECK1(0, "here %s %s %s", "are", "some", "varargs1(1)\n");
    CHECK1(1, "here %s %s %s", "are", "some", "varargs1(2)\n");   // won't print

    CHECK2(0, "here %s %s %s", "are", "some", "varargs2(3)\n");   // won't print
    CHECK2(1, "here %s %s %s", "are", "some", "varargs2(4)\n");

    // always invokes printf in the macro
    CHECK3("here %s %s %s", "are", "some", "varargs3(5)\n");

    MACRO("hello, world\n");

    MACRO("error\n", EMPTY); // would cause error C2059, except VC++
                             // suppresses the trailing comma
}
```

