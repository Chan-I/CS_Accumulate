# autotool工具的使用方式（简单的例子）

> autotool是一种帮助用户自动管理项目生成Makefile的工具。有时候手动写Makefile可以满足自己的要求，但是随着项目增加，代码结构也变得非常复杂，这样一来手动维护每个Makefile就变得非常困难。
>
> autotool的存在帮助降低了项目维护的难度。

autotool不是某一个工具，而是一系列工具的混合体。

autoscan

aclocal

autoconf

autoheader

automake

这一系列最终目的就是生成makefile，进而帮助项目编译。

------


## Makefile.am的作用

**Makefile.am**文件是整个autotool自动生成makefile的灵魂，这其中不需要规定多么复杂的逻辑生成关系。这里对这个内容进行着重介绍。

- 终极目标

  automake通过Makefile.am来生成Makefile.in。

|                                   |                                                              |
| --------------------------------- | ------------------------------------------------------------ |
| `bin_PROGRAMS(*program-list*)`    | a program or programs build in the local directory that should be compiled, linked and installed. |
| `noinst_PROGRAMS(*program-list*)` | a program or programs build in the local directory that should be compiled, linked but not installed. |
| `bin_SCRIPTS(*script-list*)`      | a script or scripts build in the local directory that should be installed. |
| `man_MANS(*page-list*)`           | man pages that should be installed.                          |
| `lib_LTLIBRARIES(*lib-list*)`     | a library or libraries that should be built using `libtool`. |

- 命名方案

  automake使用统一的命名规则，此举可以使工具明确需要构建的内容。

  PROGRAMS：（```bin_PROGRAMS```  ```sbin_PROGRAMS```  ```noinst_PROGRAMS```  etc）

  - `PROGRAMS`—用来生成可执行二进制文件的参数，多数为C/C++，lex，yacc，或者需要依赖工具。
  - `LIBRARIES`—生成二进制文件/分发软件的中间形式文件。
  - `LISP` (yeah, right)
  - `PYTHON` 
  - `JAVA`
  - `SCRIPTS`—for distribution in the package; an artificial intermediate source is established, usually the resulting script name unsurprisingly suffixed with *.in*.
  - `DATA` (beats me so far)
  - `MANS`—生成使用手册

- 隐藏变量

  automake会有一些预留的参数，例如`AM_CFLAGS`

- 源文件、头文件、库文件

   为每个目标文件指定源文件。

   生成的目标文件名称 + _ + SOURCES  =  源文件列表

   ```
   	lockproj_SOURCES                  = main.c
   	lib_LTLIBRARIES                   = libpthread_rwlock_fcfs.la
   	libpthread_rwlock_fcfs_la_SOURCES = rwlock.c queue.c
   	libpthread_rwlock_fcfs_la_HEADERS = rwlock_fcfs.h
   ```

## Makefile.am的基本例子（简单的写法教程)

   ```makefile
# Notice the bin_ prefix.
bin_PROGRAMS = kdialog
 
kdialog_SOURCES = kdialog.cpp widgets.cpp
kdialog_LDADD = $(LIB_KIO)
kdialog_LDFLAGS = $(all_libraries) $(KDE_RPATH)
 
AM_CPPFLAGS = $(all_includes)
 
METASOURCES = AUTO
   ```

- bin

  `bin`指的是想要创建的文件，这些文件会被安装在`KDE`中的bin目录。

  `*_PROGRAMS`指的就是想要编译的内容。

  `_SCRIPTS`指的是需要安装的脚本文件。

- _SOURCES

  对于`PROGRAMS`中列举出来的所有需要编译的文件都要在这里列举。

  **注意不要列举头文件，和一些在构建过程中用不到的文件**

- _LDADD

  这个参数列举了构建过程中需要链接的库文件。

  - 传给gcc的参数：-lfoo
  - 某个库文件的路径  (../path/to/thelib.la)
  - KDE编译系统定义的某个路径宏 ($LIB_KIO)

  如果某个库A依赖于库B，这里不必列出库B。

- LDFLAGS

  定义了所有编译需要的flags选项。

- KDE_CXXFLAGS

  一般来说是在其他flags后面作为编译选项的补充
  
------

### Makefile.am和shared library

```makefile
AM_CPPFLAGS = $(all_includes)
   
lib_LTLIBRARIES = libkonq.la
   
libkonq_la_LIBADD = $(LIB_KPARTS)
libkonq_la_LDFLAGS = $(all_libraries) -version-info 6:0:2 -no-undefined
libkonq_la_SOURCES = popupmenu.cc knewmenu.cpp ...
   
METASOURCES = AUTO
```

- lib

  `lib_` prex指的是将会被安装到`lib/`目录下的lib库文件，接着就是lib的名称，后缀是.la。

- _LTLIBRARIES

  `LibTool libraries`。换句话说，这个参数使得autotool明确了lib库是通过libtool程序生成的。

- _LIBADD

  指的是前缀库所以来的库。需要注意的是LDADD适用于programs，而LIBADD适用于lib。

- _LDFLAGS

  指的是编译时候需要的参数。

------

###  与库共享代码

如果使用相同的编译参数、相同的代码编译成两个不同的共享库，或者是两个程序。可以将它们都附在`_SOURCES`选项后。

```makefile
# Just as bin_ means install to /bin and lib_ means
# install to lib/, noinst_ means not to install at all.
noinst_LTLIBRARIES = libcommon.la
libcommon_la_SOURCES = dirk.cpp coolo.cpp ...
 
# no need for LIBADD or LDFLAGS, strictly speaking, but it can help
# if e.g. this code needs $(LIBJPEG), all users of this convenience
# lib won't have to specify it.
 
# Then you can use the convenience lib:
mylib_la_LIBADD = libcommon.la
myprogram_LDADD = libcommon.la
```




### 安装头文件
如果想安装头文件：

```makefile
include_HEADERS = foo.h bar.h
```

如果类使用的是命名空间，比如KParts，那么头文件应该被安装到`kparts/foo.h`

```makefile
kpartsincludedir = $(includedir)/kparts
kpartsinclude_HEADERS = foo.h bar.h
```

第一行定义了新的路径，第二行定义了需要安装的文件。需要注意的是`dir`和`_HEADERS`必须一一对应。
### 安装数据文件

为了向某个文件安装文件，需要使用dirname_DATA。

```makefile
kde_services_DATA = foo.desktop
```

为了在自定文件安装文件，必须首先定义，然后才能使用。

```makefile
myappfoodir = $(kde_datadir)/kmyapp
myappfoo_DATA = bar.desktop
```

### 将子目录添加到构建

正常情况下，只需要列出所有的子目录

```makefile
SUBDIRS = foo bar
```

为了自动编译所有的子目录

```makefile
SUBDIRS = $(AUTODIRS)
```

如果想编译一个可选的目录，需要使用automake的选择性条件功能，

```makefile
if compile_KOPAINTER
KOPAINTERDIR = kopainter
endif
 
SUBDIRS = foo bar $(KOPAINTERDIR)
```



## 0.代码前期准备

代码结构：

```shell
├── configure.ac
├── configure.scan
├── include
│   ├── abc
│   │   └── abc.h
│   └── prt
│       └── prt.h
├── Makefile.am
└── src
    ├── abc
    │   └── abc.c
    ├── hello.c
    ├── Makefile.am
    └── prt
        └── prt.c


# 代码逻辑如上，src目录中有一个主体文件，include 有两个目录分别定义了不同的函数。
```

```src/hello.c```

```c
#include "prt/prt.h"
#include "abc/abc.h"
#include "max.h"
int main()
{
        prt();
        abc();
        printf("[100,101] which one is bigger?\n");
        printf("%d\n",max(100,101));
    	//这里max函数是一个外部引用的函数，具体的放在max.h中。 具体的实现过程并没有在本工程中体现，
    	//就将max作为一个直接引用的第三方库。
        return 0;
}
```

```src/abc/abc.c```

```c
#include "abc/abc.h"
#include <stdio.h>

void abc()
{
        printf("------abc------\n");
}
```

```src/prt/prt.c```

```c
#include<stdio.h>
#include"prt/prt.h"

void prt()
{
        printf("+++++prt+++++++\n");
}
```

```include/abc/abc.h```

```c
#ifndef __ABC_H__
#define __ABC_H__
#include<stdio.h>

extern void abc();

#endif
```

```include/hello/hello.h```

```c
#ifndef __PRT_H__
#define __PRT_H__
#include<stdio.h>

extern void prt();

#endif
```



## 编写Makefile.am

回到项目中，工程个目录的`src`保存着项目的主体文件。`include`保存着头文件。

```Makefile.am```

```makefile
AUTOMAKE_OPTIONS=foreign subdir-objects

SUBDIRS=src
	# 表示子目录是src，也就是说除了在根目录会生成一个makefile意外，还会在
	# src目录也生成一个makefile

conf_include_dir=$(top_srcdir)/include
	# 需要注意的是，top_srcdir top_builddir都是预设好的autotool参数
	# 表示的是工程的根目录，只不过这个根目录是用 . 表示的。
	
#####################################################################################

cbaincludedir = $(includedir)/abc
cbainclude_HEADERS = $(conf_include_dir)/abc/abc.h

prtincludedir = $(includedir)/prt
prtinclude_HEADERS = $(conf_include_dir)/prt/prt.h

	# nameincludedir
	# nameinclude_HEADERS
	# 	前文已经叙述过 _HEADERS 表示的是头文件，这里需要注意得使用相同名称的规则

####################################################################################
yiyi:
        echo $(top_srcdir) 				# .
        echo $(top_builddir)			# .
        echo $(prefix)					# path/prefix
        echo $(bindir)					# path/prefix/bin
        echo $(libdir)					# path/prefix/lib
        echo $(datadir)					# path/prefix/share
        echo $(sysconfdir)				# path/prefix/etc
        echo $(includedir)				# path/prefix/include
        echo $(srcdir)					# .
 # 这里打印出了预设好的一些很常用的变量
        
```

```src/Makefile.am```

```makefile
bin_PROGRAMS= hello
	# 前文叙述过，bin_PROGRAMS表示需要进行安装的二进制文件
	
OBJECTS = $(top_srcdir)/src/abc/abc.c \
        $(top_srcdir)/src/prt/prt.c

hello_SOURCES= hello.c $(OBJECTS)
	# 这里表示的是用于编译的文件

AM_CPPFLAGS=    -I $(top_srcdir)/include \
                -I /home/zyjiang/tinyexpr
	# 还需要指明哪个是include目录。
	# 需要注意的是，这里直接采用top_srcdir。
	# 这个变量不建议在根目录使用
	# 		因为top_srcdir是. 在根目录使用会导致include目录紊乱
```



## 执行autotool 操作

1.```autoscan```

​	第一步就是执行autoscan操作，主要目的是扫描工作目录，并且生成configure.scan文件。这个configure.scan需要改为configure.ac然后修改其中的配置。

2.```mv configure.scan configure.in```

3.```vim configure.in```

```makefile
#                                               -*- Autoconf -*-
# Process this file with autoconf to produce a configure script.

AC_PREREQ([2.69])
	# 当前autotool的版本为2.69
AC_INIT(configure-test2, 2.0, aaa)
	# 需要生成的工程名称configure-test2 版本为2.0  bug报告地址为aaa
AC_COPYRIGHT([Copyright (c) xxxx, ZhanyiJiang STUDIO])
AM_INIT_AUTOMAKE
AC_CONFIG_SRCDIR([src])
	# 用来侦测所指定的源码文件是否存在，来确定源码目录的有效性。此处为当前目录下的hello.c
AC_CONFIG_HEADERS([config.h])
	# 用于生成config.h文件，以便autoheader使用
AC_CONFIG_FILES([Makefile \
                src/Makefile])	
	# CONFIG_FILES主要标记了工程中需要使用autotool自动生成哪些Makefile文件                

# Checks for programs.
AC_PROG_CC
	# 用来指定编译器，如果不指定，选用默认gcc。
	
	
# Checks for libraries.
AC_CHECK_LIB(JZY, max,[],[AC_MSG_ERROR([libJZY not found !!!!!!!!!!!!!!!!])])
AC_CHECK_LIB(ssl, SSL_new,[],[AC_MSG_ERROR([libssl not found !!!!!!!!!!!!!!!!])])
	# 检查ssl库和JZY库
	# 需要注意的是这个JZY库是我自己整活，自己编的一个第三方库，主要作用就是一个max函数。
	# 在生成了这个JZY库以后，将JZY库的动态库文件.so放进系统的/usr/local/lib目录下。
	# 这样JZY就可以作为第三方的库直接使用，就跟yum或者apt安装的依赖库是一样的。
	# 上述做了两个库的检查，如果没查到。则直接中断当前configure的进程，返回错误信息。

# Checks for header files.
AC_CHECK_HEADERS([stdlib.h] [unistd.h])

# Checks for typedefs, structures, and compiler characteristics.

# Checks for library functions.

AC_OUTPUT
    # 需要输出的文件名，最后一个变量。
    # 由于涉及嵌套的makefile.am，这里会根据scan自动生成两个makefile
```

4.```aclocal```

​	扫描configure.ac文件生成aclocal.m4文件，该文件主要用于处理本地的宏定义。

5.```autoconf```

​	将configure.ac的宏展开，生成configure脚本。

6.```autoheader```

​	生成configure.h.in文件

7.```automake --add-missing```

​	生成Makefile.in文件，--add-missing选项可以让automake自动添加一些必要的脚本文件，

8.```configure```

​	可以运行configure命令，将Makefile.in生成Makefile了

# 参考附录

## configure.ac参数

| 标签名             | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| AC_PREREQ          | 声明autoconf要求的版本号                                     |
| AC_INIT            | 定义软件名称、版本号、联系方式                               |
| AM_INIT_AUTOMAKE   | 必须要的，指定编译参数                                       |
| AC_CONFIG_SRCDIR   | 用来侦测所指定的源码文件是否存在, 来确定源码目录的有效性     |
| AC_CONFIG_HEADER   | 指定产生的配置文件名称(一般是config.h),用于生成config.h文件，以便 autoheader 命令使用 |
| AC_PROG_CC         | 用以探测当前系统的C编译器                                    |
| AC_PROG_RANLIB     | 用于生成静态库                                               |
| AC_PROG_LIBTOOL    | 用于生成动态库                                               |
| AM_PROG_AR         | 生成静态库时使用，用于指定打包工具，一般指ar                 |
| AC_CONFIG_FILES    | 告知autoconf本工程生成哪些相应的Makefile文件，不同文件夹下的Makefile通过空格分隔 |
| AC_OUTPUT          | 最后一个必须的宏,用以输出需要产生的文件                      |
| AC_PROG_CXX        | 用于探测系统的c++编译器                                      |
| AC_CHECK_LIB       | 探测工程中出现的库文件及库文件中的方法                       |
| AC_SUBST           | 输出一个变量到由configure生成的文件中。                      |
| AC_DEFINE          | 定义预编译的宏 `AC_DEFINE(DEBUG)` or `AC_DEFINE(VERSION, "2.3")` |
| AC_DEFINE_UNQUOTED | AC_DEFINE_UNQUOTED(FOO, "${some_variable}")  使用shell的扩展变量来定义预编译宏 |
| AC_CHECK_LIB       | AC_CHECK_LIB(ssl, SSL_new, [], [AC_MSG_ERROR([lib ssl not found])])检查某个库中是否有某个symbol |

## Makefile.am参数

|                  | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| include_HEADERS  | 标明哪些头文件将在执行make install命令之后被安装到系统include目录下 |
| bin_PROGRAMS     | 生成的目标库文件名，如果有多个，用空格隔开，与configure.ac中AC_INIT对应库名对应 |
| XXX_SOURLDADDCES | 编译XXX库需要哪些源文件，使用相对路径                        |
| XXX_LDADD        | 指定要链接的静态库名称                                       |
| LIBS             | 指定要链接的动态库名称                                       |
| INCLUDE          | 一般指定要使用的头文件所在路径                               |
| AUTOMAKE_OPTIONS | 设置automake的选项，automake提供了三种软件等级：foreign、gnu和gnits，当当前库文件编译所需源文件不在当前目录时要设置参数subdir-objects |
| XXX_CPPFLAGS     | 预处理器选项，编译选项，一般用来指定所需要头文件目录         |
| noinst_LIBRARIES | 指定生成的静态库名称，当前目录下源码及头文件最终生成的目标文件名 |
| AM_V_AR          | 指定把目标打包成静态库，使用ar命令                           |
| RANLIB           | 指定为静态库创建索引，使用ranlib                             |
