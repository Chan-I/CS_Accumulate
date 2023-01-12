# 第一章	规则

## 1.1规则

### 具体规则

​	具体规则指的是用特定的文件为工作目标的必要条件，每个规则都可以有多个工作目标。形式如：

```makefile
vpath.o	variable.o	:	make.h config.h dep.h
```

这代表vpath和variable使用的相同的文件。

​	其实**不必将所有的规则一次性全部定义**，每当make看到一个工作目标，就会将工作目标和必要条件加入依存图。但是对于较复杂的应用来说，必要条件可以组成字看似毫无关联的文件：

```makefile
# 确定vpath.c被编译之前lexer.c就已经创建好了
vpath.o:lexer.c
...

# 用比较特殊的标记进行编译
vpath.o:vpath.c
	$(CC) $(RULE_FLAGS) $(OUTPUT_OPTION) $<
...

#引入一个应用程序所产生的依存关系
include auto-generated-dependencies.d
```

### 通配符

​	当有一长串文件需要执行，而有不方便一一指定时，为了简化make的编写，可以采用通配符的方式。make的通配符如同shell的～，*，？，[...]，[/^...]。

```shell
*.* 	: 	文件名中包含.号的所有文件；
?	：	任何但一字符；
[...]	:	代表一个字符集；
[^...]	：	代表某个字符集的补集；
```

### 假想工作目标

​	假想工作目标之所以较假想，是因为它不需要实际的工作目标。任何不代表文件的工作目标就叫做假想工作目标。

```makefile
clean:
	rm -rf *.o lexer.c
```

通常，make会执行假想工作目标，因为它并不会创建爱你医改工作目标为名词的文件。

**需要注意的是：make无法区分文件形式的工作目标和假想工作目标**。如果当前目录中有和假想名字相同的文件，make就会混淆无法区分。为了避免这个问题，GNU make体提供了一个特殊的工作目标 —— .PHONY，用来告诉make，该目标不是一个真正的文件而是一个假想工作目标。

```makefile
.PHONY:clean
clean:
	rm -rf *.o lex.c
```

假想工作目标可以作为另一个工作目标的必要条件。可以让make在进行世纪工作目标之前调用假想工作目标所代表的脚本。

```makefile
# 示例一：
.PHONY:make-documentation
make-documentation:
	df -k . | awk 'NR == 2 ( printf("%d available\n", $$4) )'
	....
	
# 示例二
.PHONY:make-documentation
make-documentation:df
	....

.PHONY:df
df:
	df -k . | awk 'NR == 2 ( printf("%d available\n", $$4) )'
```

上述示例1的问题在于：make-documentation中的命令在不同的环境运行需要做出不同的对应修改。如果全文后续省略部分有多处此命令，维护起来比较困难；如果将这句命令放在df的假想命令中，可以省去很多维护的麻烦。

假想工作目标还有很多其他的好处。

make的编写采用从上而下，执行采用从下而上。这么多层次的执行往往不好判断当前make运行的位置。采用假想工作目标可避免此问题。

```makefile
$(Program):build_msg $(OBJETCS) $(BUILDINS_DEP)
	$(RM) $@
	$(CC) $(LDFLAGS) -o $(Program) $(OBJETCS) $(LIBS)
	ls -al $(Program)
	
.PHONY:build_msg
build_msg:
	@printf "#\n# Building $(Program)\n #\n"
```

​	由于printf在假想工作目标中，在任何必要条件更新时都会输出信息。如果以build_msg作为$(Program)命令脚本的第一个命令，那么所有编译结果和依存关系都产生后才会执行这个命令。**由于假想工作目标总是尚未更新，所以假想目标build_msg会导致$(Program)被重建——即使已经被更新**。这么做似乎很明智，所有的计算工作在编译文件时候已经基本完成了，因此只有最后的链接一定会被执行。

### 空工作目标

​	空工作目标可以用来发挥一些潜在的能力，前文指出：假想工作目标总是尚未更新，所以假想工作目标总是会被执行，并且总是会是的他们的依存对象被重建。当有些规则不会输出任何文件，只是偶尔被执行以下，并且不想让依存对象被更新。就可以建立空工作目标

```makefile
prog:size prog.o
	$(CC) $(LDFLAGS) -o $@ $^
	
size:prog.o
	size $^
	touch size
```

### 变量

​	最简单的变量形如：```$(variable-name)```。

​	这代表着任何文字都可以被包含在变量中，而且大多数字符都可以用在变量名称上，但是必须**用$() 或 ${}括住**，这样make才会认得。**注意的是：但一字母的变量不要用$()括号的方式括住**。

### **自动变量**

| 符号   | 说明                                                         |
| ------ | ------------------------------------------------------------ |
| **$@** | **工作目标**的名字                                           |
| $%     | 档案文件成员结构中的文件名元素。                             |
| **$<** | **第一个**必要的文件名                                       |
| $?     | 时间戳在工作目标之后的所有必要条件，并以空格隔开这些条件。   |
| **$^** | **所有必要条件**的文件名，并用空格分隔，**去重**。           |
| **$+** | 形式如$^:代表**所有必要文件**的名称，并用空格分隔，**不去重**。 |
| $*     | 工作目标的主文件名，一个文件名称由两个部分形成：**主文件名**和**扩展文件名**。[**请勿在模式规则以外使用此变量**] |

具体形式如下：

```makefile
count_words:count_words.o counter.o lexer.o -lfl
	gcc $^ -o $@

count_words.o:count_words.c
	gcc $< -o $@
	
counter.o:counter.c
	gcc $<
	
lexer.o:lexer.l
	flex -t $< > $@
```

###  **用VPATH和vpath来查找文件**

​	有时候的真实情况比较复杂，所有的文件不是都存在同一个目录下的。接下来以重构refactor为例子，进行实际的文件布局。

```c
#include <lexer.h>
#include <counter.h>

void counter(int counts[])
{
	while ( yylex() )
		;
	counts[0] = fee_count;
	counts[1] = fie _count;
	counts[2] = foe_count;
	counts[3] = fum_count;
}
```

​	如果将其写成可重复使用的库函数，在头文件中应该要有一个声明，所以我们可以创建counter.h来包含这个声明：

- counter.h

```c
#ifndef COUNTER_H_
#define COUNTER_H_

extern void
	counter( int counts[4] );

#endif
```

- ​	lexer.h


```c
#ifndef LEXER_H_
#define LEXER_H_

extern int fee_count, fie_count, foe_count, fum_count;
extern int yylex( void );

#endif
```

​	

假设现在形成了如下的目录结构：

```shell
# 目录结构示意
├── include
│   ├── counter.h
│   └── lexer.h
├── Makefile
└── src
    ├── counter.c
    ├── count_words.c
    └── lexer.l
```
在更改了目录结构以后，代码的元文件都被安置在src目录，头文件都被安置在include目录。
```makefile
# 上述目录结构的makefile例子
CPPFLAGS = -I include
VPATH = src include

count_words:count_words.o counter.o lexer.o -lfl
	gcc  $^ -o $@

count_words.o:count_words.c include/couter.h
	gcc -c $< -o $@
	
counter.o:counter.c include/counter.h include/lexer.h
	gcc -c $< -o $@
	
lexer.o:lexer.c include/lexer.h
	gcc -c $< -o $@
	
lexer.c:lexer.l
	flex -t $< $@
```

​	上述makefile规定了VPATH 和 CPPFLAGS，用于告诉make编译时候需要寻找的路径。但是VPATH有个很大的问题 —— ~~无法处理不同目录下的同名文件~~。make在找到多个不同目录下的同名文件时，会默认寻找第一个文件。可以采用如下指令语法：

​										```VAPTH pattern directory-list```

现在可以将之前的path进行修改：相当于告诉了make应该在src中搜寻.c .l 文件，在include目录中搜寻 .h文件。(这样以来就可以在头文件中移除 include/xxx.h 字样)。

```makefile
# 上述目录结构的makefile例子
CPPFLAGS = -I include

# VPATH pattern dir
VPATH *.h include
VAPTH *.c *.l src     

count_words:count_words.o counter.o lexer.o -lfl
	gcc  $^ -o $@

count_words.o:count_words.c include/couter.h
	gcc -c $< -o $@
	
counter.o:counter.c include/counter.h include/lexer.h
	gcc -c $< -o $@
	
lexer.o:lexer.c include/lexer.h
	gcc -c $< -o $@
	
lexer.c:lexer.l
	flex -t $< $@
```

​	总的来说使用VPATH解决**源文件散落在多个目录中**的问题，不过也可能出现仅用vpath无法解决的问题，稍后会接着讨论。

### **模式规则**

 上述例子中的makefile已经有点冗余了，如果整个工程非常庞大。千百个文件的对应关系都要手动一一指定，不仅麻烦也会占用大量的时间修正makefile。

​	makefile可以使用文件名匹配的方式简化规则的建立，以及提供内置规则来处理。意思就是C语言的编译器会默认的将.c 文件编译成同名的.o文件；将.l文件编译成同名的.c文件。这就使得在书写makefile时有些必要条件的文件可以省略。举个例子上述makefile如果使用模式规则的方式进行简化：

```makefile
# 上述目录结构的makefile例子
vpath = src include 
CPPFLAGS = -I include

count_words:counter.o lexer.o -lfl
count_words.o:counter.h
counter.o:counter.h lexer.h
lexer.c:lexer.h
```

​	这里面所有的内置规则都是模式规则的实例。一个模式规则看起来就像一般的规则，只是主文件名被表示成%字符，正面这个makefile之所以可行是因为内部隐藏了三个内置规则：

1. *.c -> *.o

   ```makefile
   %.o:%.c
   	$(COMPILE.c) $(OUTPUT_OPTION) $<
   ```

2. *.l -> *.c

   ```makefile
   %.c:%.l
   	@$(RM) $@
   	$(LEX.l) $< > $@
   ```
   
3. *.c -> xxx

   ```makefile
   %:%.c
   	$(LINK.c) $^ $(LOADFILES) $(LDLIBS) -o $@
   ```

##### 模式

模式规则中的%符号，等效于unix中的*，可以代替任意多个字符。可以用在任何地方但是只能出现一次。文件名中%以外的模式匹配会按照字符一一匹配。

##### 特定工作目标的静态规则

静态模式规则只能用在特定的工作目标上，且与其他规则的不同点就是前面多了一个$(OBJECTS)：规范。这就会使得该规则只能应用在$(OBJECTS)变量中所列举的文件上。

```makefile
$(OBJECTS):%.o:%.c
	$(CC) -c $(CPPFLAGS) $< -o $@
```

也就是说$(OBJECTS)中涉及到的所有的文件都适用这个规则。

##### 后缀规则

后缀规则是用来定义隐含规则的最初方案，虽然不常用但是依旧会在某些时候见到这种用法。

后缀规则中的工作目标，可以是一个或两个被衔接到一块的扩展名：

```makefile
.c.o:
	$(COMPILE.c) $(OUTPUT_OPTION) $<
	
# 区别于之前介绍的模式规则

%.o:%.c
	$(COMPILE.c) $(OUTPUT_OPTION) $<
```

后缀规则要求make中只会在这两个扩展名都在已知扩展名列表时，才将之视为后缀规则。

### 隐含规则的结构

​	GNU make有90多个隐含规则，隐含规则并不是模式规则形式就是后缀规则的形式。这些内容的模式规则可应用于C，C++，Pascal，FORTRAN，ratfor，Modula等。如果使用这些工具的时候会发现make的内置规则中已经有了所需要的东西了。如果用到了未受支持的语言，比如java或者XML，就可能需要编写自己的规则了。当make检查一个工作目标时，如果找不到更新它的具体规则，就会使用内置的隐含规则。

​	内置规则有标准的结构，方便用户进行自定义。下面的规则是从C源文件来更新目标文件的规则：

```makefile
%.o:%.c
	$(COMPILE.c) $(OUTPUT_OPTION) $<
# 这个规则自定义取决于用到的变量。我们在此处看到了两个变量，其中一个COMPILE.c是由多个其他变量所定义的：

COMPILE.c = $(CC) $(CPPFLAGS) $(TARGET_ARCH) -c
CC = gcc
OUTPUT_OPTION = -o $@
```

只要修改了CC变量的设定值，就可以更换C的编译器。此外还包括用来设定编译选项的变量CFLAGS...

​	在内置的规则中使用变量的目的，就是让规则的自定义尽可能简单。因此当需要在文件中自定义这些规则的时候，务必谨慎。

```makefile
CPPFLAGS = -I project/includes
```

如果用户想在命令行上加入CPP的定义，一般会这样调用：```make CPPFLAGS=-DEBUG```

如果真的这么做了，会以外的删除编译时所需要的-I选项。命令行上所设定的变量会覆盖掉该变量原来的数值。所以不当的在makefile中设定CPPFLAGS会破坏用户预设的自定义结果。为了避免此类问题出现：

```makefile
COMPILE.c = $(CC) $(CFLAGS) $(INCLUDES) $(CPPFLAGS) $(TARGET_ARCH) -c
INCLUDES = -I project/include
```

### 特殊工作目标

特殊工作目标是一个内置的假象工作目标，用来变更make的默认行为。特殊工作目标的语法和一般工作目标的语法没有不同点，也就是```target : prerequisite```,但是target并非文件，而是一个假象工作目标。它们实际上是比较像用来修改make内部算法的指令。

* .INTERMEDIATE:

  ​		这个特殊工作目标的必要条件会被视为中间文件。如果make在更新另一个工作目标期间创建了该文件，则该文件将会在make运行结束时被自动删除。如果在make想要更新该文件之际该文件已经存在了，则该文件不会被删除。
  
  
  
* .SECONDARY:
  ​		这个特殊工作目标的必要条件会被视为中间文件，但是不会被自动删除。.SECONDARY最常用来标示存储在程序库里的目标文件(object files)。按照惯例这些文件一旦被加入到档案库以后，就会被删除。在项目的开发期间保存这些文件，但是仍然使用make进行程序的更新，又是会比较方便。
  
  
  
* .PRECIOUS:

  ​		当make期间中断时，如果自make启动以来该文件被修改过，make将会删除它正在更新的工作目标文件。因此不会在编译树中留下上位编译完成的文件，但是有时候却不希望make这么做。特别是在该文件很大而且编译代价也很大的时候。如果该文件几位珍贵，你就该用.PRECIOUS来标示它，这样的话make只有在自己信号被中断的时候才会这么做。



* .DELETE_ON_ERROR:

  ​		.DELETE_ON_ERROR的作用和.PRECIOUS相反。将工作目标视为.DELETE_ON_ERROR，如果与规则响应的任何命令在运行时发生错误的话，就应该删除该工作目标文件。make通常只有在自己被信号中断时才会删除工作目标文件。

### 自动生成依存关系

​	在实际的工程环境中，会有个棘手的情况：大多数文件的头文件都会包含其他头文件从而生成结构复杂的树形结构。通过手动方式解析这些关系是一个令人绝望的事情。但是如果这些文件重新编译失败，可能会导致数小时的调试，此时该怎么处理呢？

​	可以在make中加入一个include命令。如今大多数的make版本都支持include命令，GNU make当然一定支持这种方式。因此，诀窍就是编写一个makefile的工作目录，本工作目录的工作就是以-M选项对所有的源文件执行gcc命令，并将结果存入一个文件中(dependency file)，然后重新执行make以便于吧刚才产生的依存文件引入makefile，这样就可以触发所需要的更新：

```makefile
depend：count_word.c lexer.c counter.c
	$(CC) -M $(CPPFLAGS) $^ > $@
include depend
```

运行以前就应该执行make depend以产生依存关系。这么做虽然不错，但是当人们对源文件加入或者移除依存关系的时候，一般不会重新产生depend文件，这才会造成无法编译源文件。

​	在GNU make中，可以简单地解决此问题：

```makefile
%.d:%c
	$(CC) -M $(CPPFLAGS) $< > $@.$$$$;						\
	sed 's,\($*\)\.o[ :]*,\l.o $@ : ,g' < $@.$$$$ > $@;		\
	rm -f $@.$$$$
```

> 这是一个令人印象深刻的小型命令脚本。首先会以 -M 的选项使用C编译器，以便创建一个包含此工作目标的依存关系的临时文件。该临时文件的名称由工作目标$@以及具唯一性的数字扩展名$$$$所组成。在shell中，变量$$会返回当前索引性的shell的进程编号。因为进程编号具有唯一性，所以这么做将可以产生一个独一无二的文件名。然后我们会使用sed以.d文件为工作目标加入此规划。
>
> sed表达式有搜索部分```\\($\*\)\.o[ :]\*```以及替换部分```\l.o $@ ```：(以逗号为分隔符)组成。搜索表达式的开头是工作目标的主文件吗```$*```，被扩在正则表达式的分组```\(\)```中，后面跟着扩展名```\.o```。工作目标的文件名之后，将会出现0个或多个空格或冒号，即```[ :]*```。替换部分会通过引用第一个RE分组来做恢复最初的工作目标并且附加上扩展名，即```\l.o```，然后加入依存文件工作目标$@.

### 双冒号规则

TO DO



## 1.2变量与宏

截止目前已经看到了makefile的变量及其应用。但是上述所有的例子太过粗浅，变量和宏越多越复杂，GNU make的功能就越强大。

在我们的任何讨论之前，最好能够了解make的两种语言：**描述工作目标和必要条件组成的依存图**；**用于进行文字替换的宏语言**。make允许为较长的字符定义简写并在程序中使用这个简写，宏处理器会认出这个简写并将它们替换为展开后的形式。

一个变量的名称集合可以由任何长度的字符组成，包括大部分的标点符号。变量名称是**区分大小写**的，所以cc和CC所指的是不同的变量，要取得某个变量的值可以使用```$()```将变量包含在内部。(**如果变量为单一字符，则可以省略括号**)

> 当变量用来表示**用户在命令行上或者环境中自定义的常数时，习惯上全部大写字母，下划线符号```_```分隔**;
>
> 当变量用来表示**只出现在makefile中的变量则会全部用小写字母来表示名字。单词之间用破折号```-```分隔**。

```makefile
# 常数
CC		：= gcc
MKDIR	:=	mkdir -p

# 内部变量
source 	=	*.c
objects	=	$(subst .c,.o,$(source))

# 函数
maybe-make-dir	=	$(if $(wildcard $1),,$(MKDIR) $1)
assert-not-null	=	$(if $1,,$(error illegal null value.))
```

需要注意的是，在对变量进行赋值的时候。从作为赋值符号的```等号（=）```右侧第一个非空字符开始直至最后，都被赋值给这个变量，包括行末的空格符号。

```makefile
LIB = libio.a # LIB的赋值末尾包含了一个空格符
missing_file:
	touch $(LIB)
		ls -al | grep '$(LIB)'
	
# 上述makefile运行结果会出现问题，因为LIB被赋值为 "libio.a "末尾有空格符号。
# 在进行grep操作时，理所当然的出现问题。
```

### 变量的用途

变量可以用来保存简单的常数，也可以用来保存用户自定义的命令序列。例如：下面的设定可以用来汇报尚未使用的磁盘空间：

```makefile
DF = df
AWK = awk
free-space	:=	$(DF) . | $(AWK) 'NR == 2 {print $$4}'
```

make的变量有两种类型

**1.简单扩展的变量**：可以使用```:=```赋值运算定义一个简单扩展的变量：```MAKE_DEPEND := $(CC) -M```

**2.递归扩展的变量**：可以使用```=```赋值运算定义一个经过递归扩展的变量：```MAKE_DEPEND = $(CC) -M```

#### 其他的赋值类型

make除了提供```:= ```, ```= ```两种赋值类型意外，还提供了```?=```,```+=```两种赋值类型。

```?=```类型为附带条件的变量赋值运算符号——条件赋值。此条件只会存在于变量的值尚不存在的状况下进行变量要求的赋值动作。

```+=```运算符通常被称为附加运算值，此运算符会将文本附加到变量里面。当递归变量被使用的时候，这就是个特别重要的特征了。赋值运算符右边的值会在“不影响变量中原有值的情况下”被附加到变量里。

### 宏

变量适合存储单一行的值，如果涉及到的值为多行，例如命令脚本，就需要宏。

```makefile
define create-jar
	@echo Creating $@...
	$(RM) $(TMP_JAR_DIR)
	$(MKDIR) $(TMP_JAR_DIR)
	$(CP) -r $^ $(TMP_JAR_DIR)
	cd $(TMP_JAR_DIR) && $(JAR) $(JARFLAGS) $@
	$(JAR) -ufm $@ $(MAINFEST)
	$(RM) $(TMP_JAR_DIR)
endef



############	line	################



$(UI_JAR):$(UI_CLASS)
	@$(create-jar)
```

通过将多个执行序列封装进一个宏中，减少了不必要的维护成本。需要注意的是```@```符号可以减少当前行的输出，如果把```@```放在宏的前面，就可以避免宏内部所有指令的输出。

### 何时扩展变量

扩展变量的规则是什么，这些规则有什么用处？当make运行的时候，会以两个阶段完成它的工作，首先会读取makefile以及被引入的任何其他makefile。第二个阶段make会分析易存图并判断需要更新的工作目标，然后执行脚本以完成需要更新的动作。

当make处理递归变量或者define的时候，会将变量里的每一行或者宏的主体存储起来，包括行符号，但是不会进行扩展。

当宏扩展的时候，make会立即扫描被扩展的文本中是否存在宏的变量的引用，如果存在就予以扩展，如此递归下去。如果宏是在命令脚本语境中被扩展的，则宏主体的每一行都会插入一个前导的跳格符。

下面是如何用处理"__makefile中的元素何时被扩展__"的准则：

* 对于变量赋值，make会在第一阶段读取改行的时候，立即扩展赋值运算符左边的部分；
* =和?=的右侧部分被延后到它们被使用的时候才扩展，并在第二阶段进行；
* :=的右边部分会立即扩展；
* 如果+=的左边部分原本被定义为简单变量，+=右边部分会被立即扩展，否则，求值动作会延后；
* 对于define定义的宏命令，宏的名称会被立即扩展。宏的主体会被延伸后到被使用的时候扩展；
* 对于规则工作目标和必要条件总是会立即扩展，命令总是延后扩展；

举个例子，第一阶段中进行变量的读取：

```makefile
BIN	:=	/usr/bin
PRINTF	:=	$(BIN)/printf
DF	:=	$(BIN)/df
AWK	:=	$(BIN)/awk

# 我们定义了三个变量，都是简单变量。当make读取这几个变量的定义时候，它们的右边会被立即扩展。
# BIN变量会被定义在其他三个变量之前，所以它们的值会被塞进其他三个变量里面
```

```makefile
# 接下来，定义了free-space宏
define free-space
	$(PRINTF) "Free disk space "
	$(DF) . | $(AWK) 'NR == 2 (print $$4 )'
endef

# 紧跟在define之后的变量名称会被立即扩展，但是就此例而言，并不需要扩展的东走。
```

```makefile
# 最后使用free-space这个宏
OUTPUT_DIR	:=	/tmp
$(OUTPUT_DIR)/very_big_file:
	$(free-space)
	
# 当$(OUTPUT_DIR)/very_big_file被读取的时候，工作目标何必要条件中所有变量都会立即扩展。
# 接着make会读取这个工作目标的命令脚本，会将前置跳格符的文本行视为命令行，不会进行扩展的动作。
```

第二阶段的进行中，也就是make读取进makefile中之后，make会对每个规则寻找工作目标。上述代码中只找到了```$(OUTPUT_DIR)/very_big_file```这个目标，因为这个工作目标没有存在于任何必要条件，make会将之扩展。

> 实际上，makefile文件中有两处的次序很重要：如果BIN放在AWK后面，另外三个变量会依次被扩展为```/print,/df,/awk```，因为:=会使得make对赋值运算符右边部分立即进行求值的动作。
>
> 如果目标OUTPUT_DIR的定义放在$(OUTPUT_DIR)/very_big_file之后，工作目标会被扩展为```/very_big_file```。

### 工作目标域模式的专属变量

假设我们现在要便以一个需要额外命令行选项```-DUSE_NEW_MALLOC=1```的文件，但是其他的编译项目不需此选项。

```makefile
gui.o:gui.h
	$(COMPILE.c) -DUSE_NEW_MALLOC=1 $(OUTPUT_OPTION) $<
```

如上所示，只有一个文件需要此特殊操作的时候，可以单独将其列出来编译。但是如果涉及到很多的文件需要此类特殊的编译指令，整个makefile的维护就变得特别冗长。

为此make提出了工作目标的专属变量。这些变量的定义会附加在工作目标上，而且只有在该工作目标以及相应的任何必要条件被处理的时候，它们才会发生作用。通过这个特性，可以改写为如下形式：

```makefile
gui.o:CPPFLAGS += _DUSE_NEW_MALLOC=1
gui.o:gui.h
	$(COMPILE.c) $(OUTPUT_OPTION) $<
```

当make处理gui.o的时候，CPPFLAGS除了包含原有的内容，还会包含```-DUSE_NEW_MALLOC=1```。当make处理完gui.o这个目标以后，CPPFLAGS的值会恢复原有的内容。

```makefile
	target... : variable = value
	target... : variable := value
	target... : variable += value
# 上述语法只能来定义工作目标的专属变量。此类变量再被赋值之前，并不需要实现存在。
```

### 变量来自何处

make的变量可以来自如下几个来源：

* 文件

  ​		变量可以被定义在makefile中，或者是被makefile引入(include指令)。

  

* 命令行

  ​		可以直接在make命令行定义或者重新定义变量：```make CFLAGS = -g```

  ​		每个命令行参数包含的等号=，都是一个变量赋值运算符。在命令行上，每个变量赋值运算符的右边必须是一个单独的shell参数。如果变量本身包括空格，需要使用```()或者{}```，或者避免使用空格。命令行上的变量赋值结果会覆盖掉环境变量以及makefile中的赋值结果。可以使用```:=、=```将命令行参数。设定成简单或递归命令。

  ​	

* 环境

  ​		make启动的时候，所有环境变量都会自动定义为make的变量。这些变量具有非常低的优先级，所以make文件或者命令行参数的赋值结果会覆盖掉环境变量的值。不过可以使用```-e```选项让环境变量覆盖掉makefile变量。

  ​		当make被递归调用的时候，有若干来自上层make的变量会通过环境传递被下层的make。默认情况下，只有原先就来自环境变量的变量会被导出到下层的环境中。不过只要使用了```export```指令，就可以让任何变量被导出到环境之中。

  

* 自动创建

  ​		make会在执行一个规则的命令脚本之前创建自动变量。

### **条件指令的引入和引入指令的处理**

#### 条件指令

当make所读进的makefile使用了条件处理指令时，makefile文件中有些部分会被省略。有些部分的语法会被挑选出来。用来控制是否选择的条件具有各种形式，比如是否已经定义或是否等于。

```makefile
# COMSPEC 只会在windows被定义
ifdef COMSPEC
	PATH_SEP := ;
	EXE_EXT  := .exe
else
	PATH_SEP := :
	EXE_EXT :=
Endif	
```

如果COMSPEC已经定义，则会选择条件指令的第一个分支。条件指令的基本语法如下所示：

```makefile
if-condition
	text if the condition is true
endif
```

或者：

```makefile
if-condition
	text if the condition is true
else
	text if the condition is true
endif
```

if-condition可以使如下之一：

```makefile
ifdef variable-name
ifndef variable-name
ifeq test
ifneq test
```

其中，条件处理命令可以用在宏定义和命令脚本中，还可以用在makefile的顶层：

```makefile
libGui.a:$(gui_objects)
	$(AR) $(ARFLAGS) $@ $<
	ifdef RANLIB
		$(RANLIB) $@
	endif
```

```makefile
ifeq (a,a)
	# these are equal
endif

ifeq ( b,b )
	# These are not equal : ' b' != 'b '
	# 值得注意的是，如果使用括号的模式，注意不要加空格。
endif
```

上述例子中不建议使用```(text,text)```的方式，会因为空格被混淆，建议采用“”方式

```makefile
ifeq “a” “a”
	# these are equal
endif

ifeq “b” “b” 
	# These are equal 
endif
```

#### include指令

一个makefile文件可以引入其他文件。此功能常用来引入make头文件中所存放的共同的make的定义，或者是用来自动生成依存关系的信息。include指令的用法如下：```include definitions.mk```。

当make看到include指令的时候，会事先对通配符以及变量引用进行扩展的动作，然后试着读进引入文件。(include file)。如果这个文件存在整个过程会继续；然而，如果这个文件不存在，则make会汇报此问题并且继续读取其余的makefile。所有的makefile都被读完以后，make会从规则数据库中找出任何可用来更新引入文件的规则。如果找到了一个相符合的规则，make就会按照正常的步骤来更新工作目标。如果任何一个引入文件被规则更新，make就会接着清除内部数据库并重新读取这个makefile。

makefile 范例：

```makefile
# 引入一个简单生成的文件
include foo.mk
$(warning Finished include)

foo.mk:bar.mk
	m4 --define=FILENAME=$@ bar.mk > $@
```

bar.mk是一个文件引入的来源：

```makefile
# bar.mk —— 读取时汇报信息
$(warning Reading FILENAME)
```



整个流程的运行结果如下：

```shell
$ make
Makefile:2: foo.mk: No such file or directory
Makefile:3: Finished include
m4 --define=FILENAME=foo.mk bar.mk > foo.mk
foo.mk:2: Reading foo.mk
Makefile:3: Finished include
make: 'foo.mk' is up to date.
```

首先执行了make命令以后，include foo.mk会尝试引入foo.mk文件。但是显示的是make并未找到文件，第二行还是会继续读取以及执行makefile。完成读取动作以后，make找到了一个创建foo.mk的规则，所以就会接着执行创建foo.mk的指令。然后整个make会重新开始整个过程，这次依次读取引入文件不会遇到任何困难。

make会如何找到引入文件？如果include引入的是一个绝对路径，make自然会按照路径去寻找。如果这是一个相对的文件引用，make对在当前目录中寻找文件；如果寻找失败了，会接着道--include-dir选项指定的目录继续查找。如果**希望include忽略不存在的文件，即可在include前面加一个破折号**。

```make
-include i-my-not-exists.mk
```

### 标准的make变量

除了自动变量，make还会为“自己的状态以及内置规则的定义”提供变量，以便对外提供相关信息：

* MAKE_VERSION

  使用此变量来查看当前所运行的make的版本编号

* CURDIR

  正在执行make进程的当前工作目录。这个变量和shell的pwd命令相同，除非make在运行时遇到了```--directory```选项。假如有一个项目是用java写的，并且在顶层目录使用了一个makefile。此时，如果使用了```--directory```选项，不管我是从源代码哪个目录来调用make，仍然能够访问这个makefile。在makefile文件中，所有路径都应该被设定成相对于makefile所在的目录。需要使用绝对路径的时候则可以通过CURDIR访问。

* MAKEFILE_LIST

  make所读进的各个makefile文件的名称所构成的列表，包括默认的makefile以及命令行或者include指令所指定的makefile。在每个makefile被读进make之前，其文件名会被附加到MAKEFILE_LIST变量里。所以任何一个makefile总是可以查看此列表的最后一项来查看自己的文件名。

* MAKECMDGOALS

  当前运行的make而言，make运行时命令上指定了哪些工作目标。此变量并非不包含命令行选项或者变量的赋值。当工作目标需要特别的处理时，通常会使用MAKECMDGOALS。常见的例子是“clean”工作目标。当用户以“clean”调用make时，make不应该像平常那样进行有include指令触发的一寸文件产生动作。要避免此事可以使用ifneq和MAKECMDGOALS

  ```makefile
  idneq  "$(MAKECMDGOALS)" "clean"
  	-include $(subst .xml,.d,$(xml_src))
  endif
  ```

* .VARIABLES

  到目前为止，make从各个文件所读取的变量的名称构成的列表，不含工作目标的专属变量。

## 1.3函数

###  用户自定义函数

用户自定义函数能够将命令序列存储在变量里面，让我们得以在makefile中使用各种应用程序

```makefile
AWK		:=	awk
KILL	:=	kill

#$(kill-acroread)
define kill-acroread
	@ ps -W |											\
	$(WAK) 'BEGIN		{FILEDWIDTHS = "9 47 100"}		\
			/AcroRd32/	{								\
                			print "Killing " $$3;		\
                			system("$(KILL) -f " $$1)	\
						}'
endef
```

对于常用的脚本，宏是一种很好用的措施。可以使用define避免重复。但是这么做并不完美：如果我们要终止Acrobat Reader以外的程序，该怎么处理呢？难道还需要重复的脚本来定义另一个宏？当然不。。。

可以传递参数给变量和宏。宏的参数在定义的时候一次可以用\$1,\$2等进行引用。如果想让kill-acroread函数使用参数，只需要加入一个搜索参数即可。

```makefile
AWK			:=	awk
KILL		:=	kill
KILL_FLAGS	:=	-f
PS			:=	ps
PS_FLAGS	:=	-W
PS_FIELDS	:=	"9 47 100"

#$(call kill-program,awk-pattern)
define kill-program
	@ $(PS) $(PS_FLAGS)									\
	$(WAK) 'BEGIN	{FILEDWIDTHS = "9 47 100"}			\
			/$1/		{								\
                print "Killing " $$3;					\
                system("$(KILL) $(KILL_FLAGS) " $$1)	\
						}'
endef
```

这里使用了一个参数引用$1来取代AWK的搜索模式/AcroRd32/。注意含参数\$1和awk字段参数\$\$1之间的细微差别，要时刻谨记“哪个程序是变量引用的接受者”是一个非常重要的事情。

如下例子：

```makefile
FOP			:=	org.apache.fop.apps.Fop
FOP_FLAGS	:=	-q
FOP_OUTPUT	:=	>	/dev/null
%.pdf:	%.fo
	$(call kill-program,AcroRd32)
	$(JAVA) $(FOP) $(FOP_FLAGS) $< $@ $(FOP_OUTPUT)
```

此模式规则首先会终止Acroba进程，如果正在运行，会使用Fop处理器吧fo文件转换成pdf文件。其中用来扩展变量或者宏的语法如下：

```makefile
$(call macro-name[, param1 ...])
```

call是一个内置的make函数，call会扩展它的第一个参数并将其余参数一次替换到出现\$1,\$2...的地方。macro-name可以指任何一个宏或者变量的名称。宏或者变量的值中不必包含任何的$n引用，如果是这样的话，使用call几乎没有什么意义。macro-name之后是宏的参数，并以逗号为分隔符。

请注意，call的第一个参数是一个非扩展式的变量名称，这种状况并不常见。origin是另一个可以接受非扩展式变量名称的内置函数。如果为call的第一个参数加上一个美元符号或者圆括号，该参数就会像变量一样被扩展，它的值就会被传给call。

call的参数检查机制非常简单。可以为call指定任意多个参数。如果一个宏引用了一个参数$n,但是调用的call实例并未指定相应的参数，呢么改变了就会变为空值。如果call实例所指定的参数比宏\$n还多，name在宏中并不会扩展额外的参数。

如果使用一个宏引用另一个宏，被调用者将可以在它的宏被扩展期间看到调用者的参数：

```makefile
define parent
	echo "parent has two parameters: $1,$2"
	$(call child,$1)
endef
```

```makefile
define child
	echo "child has one paramter : $1"
	echo "but child can also see parent's second parameter:$2!"
endef

scoping_issue:
	@$(call parent,one,two)
```

对此makefile运行make就可以看到这个宏实现具有可见性的问题

```makefile
$ make
parent has two parameters:one,two
child has one parameter:one
but child can also see parent's second parameter:two!
```

### **内置函数**

GNU提供了很多内置函数可以让我们来操作变量和他们的内容。make的函数可以分为：字符串操作、文件名操作、流程控制、用户自定义函数和若干杂项函数。

首先可以多了解一点函数的语法。所有的函数都会行程如下形式：

```makefile
$(function-name arg1[, argn])
```

$(之后是内置函数的名称，接着是函数的参数。第一个参数的前导空格会被删除，但是后续的任何参数若包含前导空格都会被保留下来。

函数的参数是以括号为分隔符，所以只有一个参数的函数并不需要使用逗号，具有两个参数的函数需要使用一个逗号，以此类推。许多只接受一个参数的函数会把它的参数视为一串以空格为分隔符的单词。对于这些函数而言，他们会以空格作为分隔符。

make有很多参数允许以模式为参数，此模式的语法如同模式规则中所使用的模式。模式中可以包含一个%符号，以及前导或者跟在后面的字符。%字符代表零个或者多个任何类型的字符。进行目标字符串匹配时，此模式必须匹配整个字符串，而不是字符串的子集。

#### **字符串函数**

make内置的函数大部分都可以操作两种形式的文本，不过有些函数具有特别强的字符串处理能力，这就是要探讨的内容。

在make中常见的字符串函数操作就是从一份文件列表中选出一组文件，shell脚本中之所以常会用到grep就是这个原因。在make中我们有filter、filter-out、findstring等函数可用。

* **$(filter pattern ...,text)**

  	filter函数会将text视为一系列被空格隔开的单词，与pattern比较以后，接着会返回相符者。

  ``` makefile
  $(ui_library) : $(filter ui/%.o,$(objects))
  	$(AR) $(ARFLAGS) $@ $^
  ```

  上述例子当中，是为了建立用户界面的程序库，我们可能指向从ui子目录中选出目标文件。接下来会文件名列表中去除开头为ui/结尾为.o的文件名。%字符会将其中任意多个字符匹配。

  filter还可以接受多个格式。前文说过格式必须是整个单词，才可以将相符的单词放到输出列表中。所有如下的例子：

  ```makefile
  words := he the hen other the%
  get-the:
  	@echo he matches:	$(filter he, $(words))
  	@echo %he matches:	$(filter %he, $(words))
  	@echo he% matches:	$(filter he%, $(words))
  	@echo %he% matches:	$(filter %he%, $(words))
  ```

  运行后会出现如下结果：

  ```shell
  $ make
  he matches: he
  %he matches: he the
  he% matches: he hen
  %he% matches: the%

  # 需要注意的是filter中的模式匹配，只能识别第一个%通配符，第二个%只能作为普通字符参与其中。
  ```

  

* **$(filter-out pattern ...,text)**

  filter-out函数所做的事刚好和filter相反，用来选出与模式不相符的每个单词。所以下面的例子可以从文件名列表中选出所有非C头文件的文件。

  ```makefile
  all_source	:=	count_words.c counter.c lexer.l counter.h lexer.h
  to_compile	:=	$(filter-out %.h, $(all_source))
  ```

* **$(findstring pattern ...,text)**

  这个函数会在text中搜索string，如果这个字符串被找到了，此函数会返回string。否则将会返回空值。
  
  需要注意的是，这个函数不同于grep。此函数只会返回“搜索字符串”，而非它所找到的包含该“搜索字符串”的单词。其次，“搜索字符串无法包含通配符。
  
* **$(subst search-string, replace-string, text)**
  
  这是一个不具备通配符能力的“搜索和替换”函数。它最常被用来在文件名列表中将一个扩展名替换为另一个扩展名。需要注意的是，这里采用的是全局替换策略。而且空格也会被识别为字符串的一部分。
  
  ```makefile
  sources	:=	count_words.c counter.c lexer.c
  objects	:=	$(subst .c,.o $(sources))
  ```
  
* **$(patsubst search-pattern,replace-pattern,text)**
  
  这是一个具有通配能力的“搜索和替换函数”。照例，此处的模式可以包含一个%符号。replace-pattern的百分比符号会被扩大成与模式相符的文字。切记search-pattern必须和text整个值进行匹配。例如，下面的例子就会删除text结尾的斜线符号，而不是每一个斜线符号。
  
  ```makefile
  strip-trailing-slash = $(patsubst %/,%,$(sirectory-path))
  ```
  
* **$(words text)**

  此函数会返回text中单词的数量。

  ```makefile
  CURRENT_PATH := $(subst /, ,$(HOME))
  words:
  	@echo MY HOME path has $(words $(CURRENT_PATH) directories)
  ```

* **$(words n,text)**

  此函数会返回text中的第n个单词，第一个单词的编号是1。如果n的值大雨text中单词的个数，则此函数将会返回空值。

  ```makefile
  version_list := $(sbust ., ,$(MAKE_VERSION))
  minor_version := $(word 2, $(version_list))
  ```

* **$(firstword text)**

  此函数会返回text中的第一个单词。此功能等效于$(word 1,text)。

  ```makefile
  version_list	:=	$(subst ., ,$(MAKEVERSION))
  major_version	:=	$(firstword $(version_list))
  ```

* **$(wordlist start,end,text)**

  此函数会返回text中范围从start(包含)到end(包含)的单词。如同word函数，第一个单词的编号是1。如果start的值大于单词的个数，则函数所返回的是空值；如果end的值大于单词的个数，则函数将会自start开始返回所有单词。

  ```makefile
  # $(call uid_gid, user-name)
  uid_gid	=	$(wordlist 3, 4, \
  			$(subst :, ,  \
  			$(shell grep "^$1:" /etc/passwd)))
  ```

#### **文件名函数**

* **$(wildcard pattern)**

  第二章曾经提到过，通配符可能被用于工作目标，必要条件以及命令脚本等语境中。但是如果我们想将此功能用在其他语境中，例如变量定义，该怎么办？使用shell函数，我们可以通过subshell函数扩展模式。此时我们可以使用wildcard函数：

  ```makefile
  sources	:= 	$(wildcard *.c *.h)
  ```

* **$(dir list)**

  dir函数会返回list中每个单词的目录部分。下面的用户自定义函数会返回包含C源文件的每个子目录：

  ```makefile
  source-dirs	:=	$(sort				\
  					$(dir			\
  						$(shell find $1 -name '*.c')))
  ```

  find命令会返回当前目录中所有C源文件的文件路径，接着dir函数删掉文件名保留目录的部分，最后由sort函数移除重复的部分。

* **$(notdir name) **

  notdir函数会返回文件路径的文件名部分。例如，下面的用户自定义函数会从搞一个Java源文件返回java的类名称：

  ```makefile
  # $(call get-java-class-name, file-name)
  get-java-class-name	=	$(notdir $(subst .java,,$1))
  ```

#### **流程控制函数**

* **$(if condition,then-part,else-part)**

  if函数(不要跟第三章所提交的条件指令ifeq，ifne，ifdef和ifndef搞混了)会根据条件表达式的求值结果，从两个宏中选出来一个进行扩展的动作。如果condition扩展之后包含任何字符，那么他的求值结果为“真”，于是会对then-part进行扩展的动作；否则，如果condition扩展之后会空无一物，那么他的求值结果为假，于是会对else-part进行扩展的动作。

* **$(error text)**

  error函数可用来输出“无可挽回的”错误信息。在此函数输出信息之后，make将会以这两个结果结束状态进行终止运行。输出中包含当前makefile的名称，当前的行号以及消息正文。

  ```makefile
  # $(call assert,condition,massage)
  define assert
  	$(if $1,,$(error Asserting fialed: $2))
  endef
  # $(call assert,$(wildcard $1),$1 does not exist)
  endef
  # $(call assert-not-null,make-variable)
  define assert-not-null
  	$(call assert,$($1)),The variable "$1" is null)
  endef
  error-exit:
  	$(call assert-not-null,NON_EXSITENT)
  ```

  第一个函数assert只会测试它的第一个参数，如果该参数是空的，则会输出用户的错误信息。第二个函数建立在第一个函数至上，他会以通配模式来测试文件是否存在。

  第三个函数是一个非常有用的断言，这是一个基于“经求值的变量”的函数。一个make变量可以包含任何内容，包括另一个make变量的名称。

* **$(foreach variable,list,body)**

  这个函数可以让你在反复扩展文本的时候，将不同的值替换进去。
  
  ```makefile
  letters	:=	$(foreach letter,a b c d,$(letter))
  show-words:
  	# letters has $(words $(letters)) words: '$(letters)'
  	
  $ make
  # letter has 4 words: 'a b c d'
  ```
  
  当这个foreach函数被执行的时候，他会反复的扩展循环主体$(letter)，并且将循环控制变量letter的值一次设定为a,b,c,d。每次扩展所得到的文本会被积累起来，并以空格为分隔符。
  
  ### 高级用户自定义函数
  
  在实际的使用过程中，可能会花费大量的时间在宏函数的编写上面，可惜的是，make不会提供多少可以协助我们调试的功能。可以从编写一个简单的调试追踪函数以协助我们摆脱这个困境。
  
  如前文，所示call函数会把它的每个参数依次绑定到$1，$2等编号变量。可以为call函数指定任意多个参数，还可以通过$0来访问当前所执行的函数的名称。
  
  
  
``` makefile
# $(debug_enter）
debug-enter	=	$(if $(debug_trace),	\
					$(warning Entering $0($(echo-args))))

# $(debug-level)
debug-leave	=	$(if $(ddebug_trace),$(warning Leaving $0))

comma	:= ,
echo-args	=	$(subst ' ','$(comma) ', \
				$(foreach a,1 2 3 4 5 6 7 8 9,'$($a)'))
```

如果想查看函数a和b是如何被调用的，可以这么做：

```makefile
debug_trace	=	1
define a
	$(debug-enter)
	$echo $1 $2 $3
	$(debug-leave)
endef

define b
	$(debug-enter)
	$(call a,$1,$2,hi)
	$(debug-leave)
enfef

trace-macro:
	$(call b,5,$(MAKE))
```

### eval与value函数

eval是个与所有内置函数都不同的函数，用途在于将文本直接放在make解析器。例如:

```makefile
$(eval source := foo.c bar.c)
```

首先make会扫描eval参数中是否有变量以便进行扩展的动作，然后make会解析文本以及进行求值的动作，就好像是来自输入文件一样。

假设有一个用来编译许多程序的makefile，而且想要为每个程序定义若干变量，例如source，headers，objects。此时，不必反复的用手动的方式为每个程序定义这些变量。

```makefile
# $(call program-variables, variable-prefix, file-list)
define program-variables
	$1_sources = $(filter %.c,$2)
	$2_sources = $(filter %.h,$2)
	$1_projects = $(subst .c,.o,$(filter %.c,$2))
endef

$(call program-variables, ls, ls.c ls.h glob.c glob.h)

show-variables:
	# $(ls_sources)
	# $(ls_headers)
	# $(ls_objects)
```

program-variables宏有两个自变量，一个是三个变量的前缀；另一个是一份文件列表，宏可以从中选出文件以便设定这三个变量。但是，当我们使用这个宏的时候，会报错。

这个make解析器的运作方式，一个宏被扩展为多行文本是不合法的，这会导致语法错误。此时，解析器会以为call这一行是一个规则或者是命令行的一部分，但是找不到分隔符几号。这是一个令人相当困惑的错误信息，eval函数可用来解决这个问题。如果对call做出如下修改。

```makefile
$(eval $(call program-variables, ls, ls, ls.c ls.h glob.c glob.h))
```





​	现在，可以非常简单的使用这个宏来定义三个变量。注意宏当中赋值表达式的变量名称，它是由一个可以变得前缀和一个固定的后缀所构成，例如$1_sources。准确的说，这些变量并非前文提到的“经求值的变量”，不过他们非常相似。

```makefile
# $(call program-variables, variable-prefix, file-list)
define program-variables
	$1_sources = $(filter %.c,$2)
	$2_sources = $(filter %.h,$2)
	$1_projects = $(subst .c,.o,$(filter %.c,$2))
	
	$($1_objects): $($1_headers)
endef

ls:	$(ls_objects)

$(eval $(call program-vari))
```

program-variables的两个版本，可以让我们看到函数参数中的空格所造成的困惑。在前一版本中，函数所使用的两个参数并不受签到空格的影响。也就是说，不管$1或$ 2是否有```前导空格```，行为都一样。然后新的版本使用了“经求值的变量”$($1_objects)和$($1_headers)。现在为函数的第一个参数ls前面加上一个前导的空格，好让“经过求值的变量”能够以一个前导空格开头，这样会被扩展为空无一物，因为没有一个变量被定义成为以前导空格开头的。这是一个相当难以诊断的问题。



我们要解决的最后一个问题的方法是延后“经求值的变量”的求值动作。处理这个问题的另一个方法就是以eval函数封装每个变量的赋值表达式，迫使其提早进行求值的动作。

```makefile
# $(call program-variables,variable-prefix,file-list)
define program-variables
	$(eval $1_sources = $(filter %.c,$2))
	$(eval $1_headers = $(filter %.h,$2))
	$(eval $1_objects = $(subst .c,.o,$(filter %.c,$2)))
	$($1_objects) : $($1_headers)
endef
ls: $(ls_objects)
$(eval $(call program-variables,ls,ls.c ls.h glob.c glob.h))
```

通过把变量赋值表达式封装在自己的eval调用中，可以让make在扩展program-variables宏的时候将它们放在内置数据库中。于是它们就可以立即被使用在宏中。

加强这个makfile的同时，我们觉得应该为宏加入另一个规则。任何一个我们要编译的程序都应该依存于他自己的目标文件。所以为了让我们的makefile能够参数化，我们会加入一个最顶层的all工作目标，以及使用一个变量来保存我们要编译的每个程序。

```makefile
#$(call program-variables,variable-prefix,file-list)
define program-variables
	$(eval $1_sources = $(filter %.c,$2))
	$(eval $1_headers = $(filter %.h,$2))
	$(eval $1_objects = $(subst .c,.o,$(filter %.c,$2)))
	
	program += $1
	
	$1: $($1_objects)
	
	$($1_objects): $($1_headers)
endef

# 将all工作目标放在此处，所以它就是默认目标
all:

$(eval $(call program-variables,ls,ls,c ls.h glob.c glob.h))
$(eval $(call program-variables,cp,...))
$(eval $(call program-variables,mv,...))
$(eval $(call program-variables,ln,...))
$(eval $(call program-variables,rm,...))

# 将program必要条件放在此处，所以all工作目标被定义在这里
all: $(programs)
```

注意all工作目标与其必要条件的摆放位置。programs变量必须等到那五个eval调用被扩展之后才会被有正确的定义，但是我们想让all工作目标成为默认目标。此时，我们可以先为makefile加入all工作目标，稍后再加入他的必要条件。

program-variables函数所发生的的问题，起因于有些变量的求值动作太早进行。实际上，make所提供的value函数可以协助我们解决此问题。value函数UI返回他的variable参数未扩展的值。这个未扩展的值可以传给eval函数做进一步的处理。通过函数未扩展的值，我们可以避免必须为宏中的变量引用加上引号的为题。

可惜program-variables宏无法使用此函数。因此value是一个“非全有即全无”（all-of-nothing）的函数，如果加以证明，value将不会扩展宏中的任何变量。此外，value不接受任何参数，所以我们的程序名称以及文件名列表参数都不会背扩展。



### 函数倒挂

用户自定义的函数只是一个用来存放文本的变量。如果变量名称中存在$1,$2等引用，call函数会予以扩展。如果函数中不存在任何的变量引用，call并不在意。事实上，如果变量中不存在任何文本，call也不会在意。所以你看不到的任何错误或者警告信息。

要想让函数具有很高的重用性，可以对它加入hook，hook是一种函数引用，用户可以重新定义，以便进行自己所定义的工作。

```makefile
# $(call build-library, object-files)
define build-library
	$(AR) $(ARFLAGS) $@ $1
	$(call build-library-hook,$@)
endef
```

为了使用此hook，我们会定义build-library-hook函数：

```makefile
$(foo_lib): build-library-hoook = $(RANLIB) $1
$(foo_lib): $(foo_objects)
	$(call build-library,$^)
	
$(bar_lib): build-library-hook = $(CHMOD) 444 $1
$(bar_lib): $(bar_objects)
	$(call build-library,$^)
```

### 传递参数

一个函数可以从四种来源取得他的数据：使用call传入的参数，全局变量，自动变量以及工作目标专属变量。其中，已通过参数为最模块化的选择，因为函数的使用可以函数中的变动与全局数据无关，但是有时候这并不是最重要的评断标准。

假设有若干项目共享一组make函数。将会通过变量前缀，例如PROJECT1_，来区分每个项目，而且项目里的重要便利都会使用具有“跨项目后缀”的前缀。之前范例中所用到的PROJECT_SEC等变量，将会变成PROJECT1_SRC、PROJECT1_BIN、和PROJECT_BIN。这样我么就不必编写用来读取这三个变量的函数，我们可以使用经求值的变量以及传递单一参数：

```makefile
# (call process-xsl,project-prefix,file-name)
define process-xml
	$($1_LIB)/xmlto -o $($1_BIN)/xml/$2 $($1_SRC)/xml/$2
endef
```

传递参数的另一个方式就是使用工作目标专属变量。当大部分的调用所使用的都是标准的值，仅有少数需要特别处理的时候，这个方式特别有用。当规则被定义在一个引入文件中，但是我们想从定义变量的makefile来调用该规则时，工作目标专属变量还可以为我们提供相当的灵活性。

```makefile
release : MAKING_RELEASE = 1
release : libraries executables
...
$(foo_lib):
	$(call build-library,$^)
...
# $(call build-library, file-list)
define build-library
	$(AR) $(ARFLAGS) $@				\
		$(if $(MAKING_RELEASE), 		\
			$(filter-out debug/%,$1),	\
				$1)
endef
```









## 1.4命令





# 第二章	高级与特别的用法

## 2.1大型项目的管理



## 2.2 可以移植性的makefile




```

```
