# SHELL中的指令梳理(sed+awk+...)

## 1.sed 选取、替换、删除、新增数据

​	sed 是一种几乎可以应用在所有 UNIX 平台（包括 Linux）上的轻量级流编辑器。sed  有许多很好的特性。首先，它相当小巧，通常要比你所喜爱的脚本语言小多倍。其次，因为 sed  是一种流编辑器，所以，它可以对从如管道这样的标准输入中接收的数据进行编辑。因此，无须将要编辑的数据存储在磁盘上的文件中。因为可以轻易将数据管道输出到 sed，所以，将 sed 用作强大的 Shell 脚本中长而复杂的管道很容易。

```shell
# root @ 192-168-1-10 in ~ [8:56:43] 
$ sed
用法: sed [选项]... {脚本(如果没有其他脚本)} [输入文件]...

  -n, --quiet, --silent
                 取消自动打印模式空间，只打印经过sed处理的行。
  -e 脚本, --expression=脚本
                 添加“脚本”到程序的运行列表，允许对输入数据应用多个sed命令
  -f 脚本文件, --file=脚本文件
                 添加“脚本文件”到程序的运行列表
  --follow-symlinks
                 直接修改文件时跟随软链接
  -i[SUFFIX], --in-place[=SUFFIX]
                 edit files in place (makes backup if SUFFIX supplied)
                 如果不加-i选项，将不会修改文件具体内容，如果添加了-i选项，将修改文件内容
  -c, --copy
                 use copy instead of rename when shuffling files in -i mode
  -b, --binary
                 does nothing; for compatibility with WIN32/CYGWIN/MSDOS/EMX (
                 open files in binary mode (CR+LFs are not treated specially))
  -l N, --line-length=N
                 指定“l”命令的换行期望长度
  --posix
                 关闭所有 GNU 扩展
  -r, --regexp-extended
                 在脚本中使用扩展正则表达式
  -s, --separate
                 将输入文件视为各个独立的文件而不是一个长的连续输入
  -u, --unbuffered
                 从输入文件读取最少的数据，更频繁的刷新输出
  -z, --null-data
                 separate lines by NUL characters
  --help
                 display this help and exit
  --version
                 output version information and exit
                 
  a \：追加，在当前行后添加一行或多行。当添加多行时，除最后一行外，每行末尾需要用“\”代表数据未完结；
  c \：行替换，用c后面的字符串替换原数据行。当替换多行时，除最后一行外，每行末尾需用“\”代表数据未完结；
  i \：插入，在当前行前插入一行或多行。当插入多行时，除最后一行外，每行末尾需要用“\”代表数据未完结；
  d：删除，删除指定的行；
  P：打印，输出指定的行；
  s：字符串替换，用一个字符串替换另一个字符串。格式为“行范围s/旧字串/新字串/g”（和Vim中的替换格式类似）；

如果没有 -e, --expression, -f 或 --file 选项，那么第一个非选项参数被视为
sed脚本。其他非选项参数被视为输入文件，如果没有输入文件，那么程序将从标准
输入读取数据。
GNU sed home page: <http://www.gnu.org/software/sed/>.
General help using GNU software: <http://www.gnu.org/gethelp/>.
```



假设现在存在一个文件`student.txt`。

```txt
ID Name PHP Linux MySQL Average
1 Liming 82 95 86 87.66
2 Sc 74 96 87 85.66
3 Gao 99 83 93 91.66
```

### 1.1 输出

假设现在想要输出第二行数据`sed '2p' stu.txt`，可以看到下述效果：

```shell
# root @ 192-168-1-10 in ~/shell_test [9:28:14] 
$ sed '2p' stu.txt
ID Name PHP Linux MySQL Average
1 Liming 82 95 86 87.66
1 Liming 82 95 86 87.66
2 Sc 74 96 87 85.66
3 Gao 99 83 93 91.66

# root @ 192-168-1-10 in ~/shell_test [9:28:36] 
$ sed '3p' stu.txt
ID Name PHP Linux MySQL Average
1 Liming 82 95 86 87.66
2 Sc 74 96 87 85.66
2 Sc 74 96 87 85.66
3 Gao 99 83 93 91.66
```

由于没加`-n`选项，输出的数据内容既有要输出的文件内容，又有文本的内容：`sed -n '3p' stu.txt`。

```shell
# root @ 192-168-1-10 in ~/shell_test [9:28:43] 
$ sed -n '3p' stu.txt
2 Sc 74 96 87 85.66
```

### 1.2 删除

```shell
[root@localhost ~]#sed '2,4d' student.txt
#删除从第二行到第四行的数据
ID Name PHP Linux MySQL Average

[root@localhost ~]# cat student.txt
#文件本身并没有被修改
ID Name PHP Linux MySQL Average
1 Liming 82 95 86 87.66
2 Sc 74 96 87 85.66
3 Gao 99 83 93 91.66
```

### 1.3 插入

再来看看如何追加和插入行数据：

```shell
[root@localhost ~]# sed '2a hello' student.txt
#在第二行后加入hello
ID Name PHP Linux MySQL Average
1 Liming 82 95 86 87.66
hello
2 Sc 74 96 87 85.66
3 Gao 99 83 93 91.66
```
**"a"**动作会在指定行后追加数据。如果想要在指定行前插入数据，则需要使用**"i"**动作。

```shell
[root@localhost ~]# sed '2i hello > world' student.txt
#在第二行前插入两行数据
ID Name PHP Linux MySQL Average
hello > world
1 Liming 82 95 86 87.66
2 Sc 74 96 87 85.66
3 Gao 99 83 93 91.66
```

如果想追加或插入多行数据，则除最后一行外，每行的末尾都要加入"\\"代表数据未完结。

### 1.4 替换

```shell
[root@localhost ~]# cat student.txt | sed '2c No such person'
ID Name PHP Linux MySQL Average
No such person
2 Sc 74 96 87 85.66
3 Gao 99 83 93 91.66
```

"c"动作是进行整行替换的，如果仅仅想替换行中的部分数据，就要使用"s"动作了。"s"动作的格式如下：

```shell
[root@localhost ~]# sed 's/旧字符串/新字符串/g' 文件名
```
替换的格式和 Vim 非常类似，例如：
```shell
[root@localhost ~]# sed '3s/74/99/g' student.txt
 #在第三行中，把74换成99
 ID Name PHP Linux MySQL Average
 1 Liming 82 95 86 87.66
 2 Sc 99 96 87 85.66
 3 Gao 99 83 93 91.66
```

 如果想把某行的成绩注释掉，让它不再生效，则可以这样做：
```shell
[root@localhost ~]#sed '4s/^/#/g' student.txt
 #在这里使用正则表达式，"^"代表行首
 ID Name PHP Linux MySQL Average
 1 Liming 82 95 86 87.66
 2 Sc 74 96 87 85.66
 \#3 Gao 99 83 93 91.66
```

 不仅如此，我们还可以这样做：
```shell
[root@localhost ~]# sed -e 's/Liming//g; s/Gao//g' student.txt
 #同时把"Liming"和"Gao"替换为空
 ID Name PHP Linux MySQL Average
 1 82 95 86 87.66
 2 Sc 74 96 87 85.66
 3 99 83 93 91.66
```

 "-e"选项可以同时执行多个 sed 动作，当然，如果只执行一个动作，则也可以使用"-e"选项，但是这时没有什么意义。还要注意，多个动作之间要用";"或回车分隔，例如，上一条命令也可以这样写：
```shell
[root@localhost ~]# sed -e 's/Liming//g;s/Gao//g' student.txt
 ID Name PHP Linux MySQL Average
 1 82 95 86 87.66
 2 Sc 74 96 87 85.66
 3 99 83 93 91.66
```


## 2. awk 文本和数据处理

 awk命令是一种编程语言，用于在linux/unix下对文本和数据进行处理。

 而且它支持用户自定义函数和动态正则表达式等先进功能，是linux/unix下的一个强大编程工具。

### 2.1 工作原理

逐行的读取文本，默认空格或者tab为分隔符进行分割，将分隔所得的各个字段保存到内建变量中，并按模式或者条件执行编辑命令。

sed命令常用于一整行的处理，而awk比较倾向于将一行分成多个“字段”然后再进行处理。awk信息的读入也是逐行读取的，执行结果可以通过print的功能将字段数据打印显示。在使用awk命令的过程中,可以使用逻辑操作符“&&”表示“与”、“||”表示“或”、“！”表示“非”；还可以进行简单的数学运算，如+、-、*、/、%、^分别表示加、减、乘、除、取余和乘方。


### 2.2 命令格式

```shell
$ awk
Usage: awk [POSIX or GNU style options] -f progfile [--] file ...
Usage: awk [POSIX or GNU style options] [--] 'program' file ...
Usage: awk '{pattern + action}' {filenames}
POSIX options:		GNU long options: (standard)
	-f progfile		--file=progfile
		从脚本文件读取awk命令
	-F fs			--field-separator=fs
		指定输入文件折分隔符，fs是一个字符串或者是一个正则表达式，如-F:。
	-v var=val		--assign=var=val
		赋值一个用户定义变量,也可以用来打印系统中的环境变量
Short options:		GNU long options: (extensions)
	-b			--characters-as-bytes
	-c			--traditional
	-C			--copyright
	-d[file]		--dump-variables[=file]
	-e 'program-text'	--source='program-text'
	-E file			--exec=file
	-g			--gen-pot
	-h			--help
	-L [fatal]		--lint[=fatal]
	-n			--non-decimal-data
	-N			--use-lc-numeric
	-O			--optimize
	-p[file]		--profile[=file]
	-P			--posix
	-r			--re-interval
	-S			--sandbox
	-t			--lint-old
	-V			--version

To report bugs, see node `Bugs' in `gawk.info', which is
section `Reporting Problems and Bugs' in the printed version.

gawk is a pattern scanning and processing language.
By default it reads standard input and writes standard output.

Examples:
	gawk '{ sum += $1 }; END { print sum }' file
	gawk -F: '{ print $1 }' /etc/passwd

```
### 2.3 awk常见的内建变量
| 变量        | 说明                                       |
| :---------- | :----------------------------------------- |
| ARGC        | 命令行参数的数目                           |
| ARGIND      | 命令行中当前文件的位置（从0开始算）        |
| ARGV        | 包含命令行参数的数组                       |
| CONVFMT     | 数字转换格式（默认值为%.6g）               |
| ENVIRON     | 环境变量关联数组                           |
| ERRNO       | 最后一个系统错误的描述                     |
| FIELDWIDTHS | 字段宽度列表（用空格键分隔）               |
| FILENAME    | 当前输入文件的名                           |
| FNR         | 同NR，但相对于当前文件                     |
| FS          | 列分隔符（默认是任何空格）                 |
| IGNORECASE  | 如果为真，则进行忽略大小写的匹配           |
| NF          | 表示字段数，在执行过程中对应于当前的字段数 |
| NR          | 表示记录数，在执行过程中对应于当前的行号   |
| OFMT        | 数字的输出格式（默认值是%.6g）             |
| OFS         | 输出字段分隔符（默认值是一个空格）         |
| ORS         | 输出记录分隔符（默认值是一个换行符）       |
| RS          | 记录分隔符（默认是一个换行符）             |
| RSTART      | 由match函数所匹配的字符串的第一个位置      |
| RLENGTH     | 由match函数所匹配的字符串的长度            |
| SUBSEP      | 数组下标分隔符（默认值是34）               |

`awk` 常用的变量有 `OFS`、`NF` 和 `NR`。`OFS` 和 `-F` 选项有类似的功能，也是用来定义分隔符的，但是它是在输出的时候定义的。`NF` 表示用分隔符分隔后一共有多少段。`NR` 表示行号。

`OFS` 的用法示例如下：

```
# head -5 /etc/passwd |awk -F ':' '{OFS="#"} {print $1,$3,$4}'root#0#0bin#1#1daemon#2#2adm#3#4lp#4#7
```

还有更高级一些的用法： 

```
# awk -F ':' '{OFS="#"} {if ($3>=1000) {print $1,$2,$3,$4}}' /etc/passwdnobody#x#65534#65534aminglinux#x#1000#1000
```

变量 `NF` 的具体用法如下：

```
# head -n3 /etc/passwd | awk -F ':' '{print NF}'777 
# head -n3 /etc/passwd | awk -F ':' '{print $NF}'/bin/bash/sbin/nologin/sbin/nologin
```

这里 `NF` 是多少段，`$NF` 是最后一段的值。变量 `NR` 的具体用法如下：

```
# head -n3 /etc/passwd | awk -F ':' '{print NR}'123
```

我们还可以使用 `NR` 作为判断条件，如下所示：

```
# awk 'NR>40' /etc/passwdinsights:x:978:976:Red Hat Insights:/var/lib/insights:/sbin/nologinsshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologinavahi:x:70:70:Avahi mDNS/DNS-SD Stack:/var/run/avahi-daemon:/sbin/nologintcpdump:x:72:72::/:/sbin/nologinaminglinux:x:1000:1000:aminglinux:/home/aminglinux:/bin/bash
```

`NR` 也可以配合段匹配一起使用，如下所示：

```
# awk -F ':' 'NR<20 && $1 ~ /roo/' /etc/passwdroot:x:0:0:root:/root:/bin/bash
```

### 2.4 基本用法

 ```bash
 # 格式
 $ awk 动作 文件名
 
 # 示例
 $ awk '{print $0}' demo.txt
 ```

上面示例中，`demo.txt`是`awk`所要处理的文本文件。前面单引号内部有一个大括号，里面就是每一行的处理动作`print $0`。其中，`print`是打印命令，`$0`代表当前行，因此上面命令的执行结果，就是把每一行原样打印出来。

下面，我们先用标准输入（stdin）演示上面这个例子。

 ```bash
 $ echo 'this is a test' | awk '{print $0}'
 this is a test
 ```

上面代码中，`print $0`就是把标准输入`this is a test`，重新打印了一遍。

`awk`会根据空格和制表符，将每一行分成若干字段，依次用`$1`、`$2`、`$3`代表第一个字段、第二个字段、第三个字段等等，`$0`输出所有的字段。

 ```bash
 $ echo 'this is a test' | awk '{print $3}'
 a
 ```

### 2.5 使用示例

下面列出一个最常用的awk命令结构，借此分析原理

```js
awk 'BEGIN{ commands } pattern{ commands } END{ commands }'
```

- 首先执行 `BEGIN {commands}` 内的语句块，注意这只会执行一次，经常用于变量初始化，头行打印一些表头信息，只会执行一次，在通过stdin读入数据前就被执行；
- 从文件内容中读取一行，注意**awk是以行为单位处理的，每读取一行使用** **`pattern{commands}`** **循环处理** 可以理解成一个for循环，这也是最重要的部分；
- 最后执行 `END{ commands }` ,也是执行一次，在所有行处理完后执行，一帮用于打印一些统计结果。

第一个例子，获得/etc/passwd文件种每行的地1个和第7个数据，以逗号分隔，并再第一行和最后一行打印一串文字。

```js
shell> cat /etc/passwd |awk  -F ':'  'BEGIN {print "name,shell"}  {print $1","$7} END {print "blue,/bin/nosh"}'
name,shell
root,/bin/bash
daemon,/usr/sbin/nologin
bin,/usr/sbin/nologin
sys,/usr/sbin/nologin
...
blue,/bin/nosh
```

**书写注意事项**

- awk后的命令需要用 单引号括起来
- 最好用 ‘{}’ 括起来每个部分，便于阅读；
- 每个 ‘{}’ 可以有多个命令或者其它，之间用 ‘;’ 号分割。

#### 2.5.1  怎么清晰的输出想要的信息？

awk的输出主要靠 `print`,`printf` 指令，这两个指令的用法和c语言中的 `print，printf` 一毛一样。awk处理每行时是以列为每个域，例如 `print $1` 就是输出第一列，`print $1,$2` 就是输出第1、2列，`print $0` 输出全部。

awk怎么区分列呢，默认是以空格区分，但是你也可以通过 `-F` 参数指定，例如 `-F;` 指定分号为分隔符，`-F[;,]` 指定分号和逗号为分隔符。

假设有如下一个文件 *netstat.txt*

```js
   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
  2304 york      20   0 2244404 213596  84764 S   6.2  5.3   4:56.97 cinnamon
 12489 york      20   0   43668   3708   2984 R   6.2  0.1   0:00.02 top
     1 root      20   0  185352   5968   3972 S   0.0  0.1   0:04.35 systemd
     2 root      20   0       0      0      0 S   0.0  0.0   0:00.05 kthreadd
     4 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
     6 root      20   0       0      0      0 S   0.0  0.0   0:00.06 ksoftirqd/0
     7 root      20   0       0      0      0 S   0.0  0.0   0:14.82 rcu_sched
    34 root      20   0       0      0      0 S   0.0  0.0   0:00.04 khungtaskd
    35 root      20   0       0      0      0 S   0.0  0.0   0:00.00 oom_reaper
    36 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 writeback
    37 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kcompactd0
```

#### 2.5.3 我只想打印第一列和第四列的内容

```shell
shell> awk '{print $1,$2}' top.txt 
PID USER
2304 york
12489 york
1 root
2 root
4 root
6 root
7 root
34 root
35 root
36 root
37 root
```

从第三行开始打印

```shell
shell> awk 'NR>4 {print $1,$2}' top.txt 
1 root
2 root
4 root
6 root
7 root
34 root
35 root
36 root
37 root
```



#### 2.5.4 对齐不整齐，打印的更漂亮些？能加上行号？

来看看格式化输出 `printf`

```js
shell> awk '{printf "%-8s %-8s %-8s %-18s\n",NR, $1,$2,$12}' top.txt
1        PID      USER     COMMAND           
2        2304     york     cinnamon          
3        12489    york     top               
4        1        root     systemd           
5        2        root     kthreadd          
6        4        root     kworker/0:0H      
7        6        root     ksoftirqd/0       
8        7        root     rcu_sched         
9        34       root     khungtaskd        
10       35       root     oom_reaper        
11       36       root     writeback         
12       37       root     kcompactd0
```

上面的对齐方式是不是更好看，命令里面的 `%-8s` 用过c语言输出的一定很眼熟，另外看到一个新的东西 `NR` 这是awk内部提供显示行号的变量，除了这些还有以下常用的,(下面这张表就是用awk处理的)



#### 2.5.5 输出结果筛选

例如上面的例子，我想统计出所有进程总共占了多少cpu，**awk变量和基本运算** 了解一下，先看例子

```js
shell> awk 'BEGIN {sum=0} {printf "%-8s %-8s %-18s\n", $1, $9, $11; sum+=$9} END {print "cpu sum:"sum}' top.txt
PID      %CPU     TIME+             
2304     6.2      4:56.97           
12489    6.2      0:00.02           
1        0.0      0:04.35           
2        0.0      0:00.05           
4        0.0      0:00.00           
6        0.0      0:00.06           
7        0.0      0:14.82           
34       0.0      0:00.04           
35       0.0      0:00.00           
36       0.0      0:00.00           
37       0.0      0:00.00           
cpu sum:12.4
```

又看到几个新的东西，变量初始化及awk的一些基本运算

- `sum=0` 一般都在 `BEGIN` 里面初始化一个变量，如果不需要初始化可以直接进行对变量的赋值，这很像脚本语言中的自动推断，除了提供基本的运算以外（有哪些？c有哪些这就基本有哪些），还提供了一些高级计算函数,例如数学运算`log、sqr、cos、sin`, 字符运算 `length、substr`
- 引入 **awk变量** 除了能在内部使用，还可以从外部引入，使用 `-v` 参数指定，例如下面是打印环境变量的例子，这个在编写脚本中很常见。 `awk -v var=$PATH 'BEGIN {print var}' top.txt`

上面的例子引入了 awk 的运算，知道可以定义变量运算，除此之外还支持很多运算符，算术运算符，逻辑运算符，甚至一些内部提供的函数。

##### 2.5.5.1 去掉第一行，只输出cpu消耗大于0的

```js
shell> awk 'NR>1 && $9>0 {printf "%-8s %-8s %-18s\n",$1,$9,$12}' top.txt 
2304     6.2      cinnamon          
12489    6.2      top 
```

首先按照上面所介绍的 awk 执行流程来介绍，单引号里面的 *模式+命令* 在每读到一行时就会执行，判断条件也是如此 `NR>1 && $9>0` 这种写法和c语言没有两样，只是少了判断 `if` 而已，每读到一行时都执行这个判断条件来确定是否过滤；下面转换成高级语言的代码。

```js
for l in lines {
	if ( NR>1 && $9 >0 ) {
		printf ("%-8s %-8s %-18s\n",$1,$9,$12);
	}
}
```

> 注意：这个 `NR>1` 经常在[数据库](https://cloud.tencent.com/solution/database?from=10680)查询后被使用，来剔除表头。

这个例子里面出现的就是 awk 的条件判断，条件判断运算符也是和c语言一样不多阐述，在比较时不仅可以比较数字还可以比较字符串，awk会自动识别，比较字符串时会按照ASCII码顺序比较。

##### 2.5.5.2 保留表头，只输出特定用户名的进程

```js
awk 'NR==1 || /york/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt 
PID      USER     %CPU     COMMAND           
2304     york     6.2      cinnamon          
12489    york     6.2      top               
```

上面 *pattern* 出现了一个新语法 `/york/` ,这个就是正则匹配，面对一些字符串匹配来进行过滤，通过运算符显的很无力，这在处理大量log时尤为突出，awk 也想到这点，支持正则匹配来精准筛选；正则过滤有好几种运用方法，但主要格式都是 **在双斜杠内写上你的正则表达式**；例如上面的例子就是 **该行只要出现 ‘york’ 字符即可满足过滤条件**

##### 2.5.5.3 上面例子不太准确，更好的做法是针对域匹配

上面例子表述的是 **改行只要出现** 即可匹配到，万一不是 *USER* 字段也有 ‘york’ 字符就会出现错误；

```js
shell> awk 'NR==1 || $2~/york/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt 
PID      USER     %CPU     COMMAND           
2304     york     6.2      cinnamon          
12489    york     6.2      top 
```

上面引入了 `~` 运算符，该运算符和 `!~` 成对立关系，类似 `=`和`！=`的关系，前者表示匹配到的输出，后者表示匹配到的过滤。假设过滤 `york`的输出，可以这样写：

```js
shell> awk 'NR==1 || $2!~/york/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt 
PID      USER     %CPU     COMMAND           
1        root     0.0      systemd           
2        root     0.0      kthreadd          
4        root     0.0      kworker/0:0H      
6        root     0.0      ksoftirqd/0       
7        root     0.0      rcu_sched         
34       root     0.0      khungtaskd        
35       root     0.0      oom_reaper        
36       root     0.0      writeback         
37       root     0.0      kcompactd0  
```

另外要注意的就是针对域匹配 `$2~/york/`, 前面有说过 `$0` 代表整个域，所以 `$0~/york/` 和 `/york/` 是等价了，说这个的原因就是当我们需要 **针对一行做正则过滤的时候可以这样写** **`$0!~/york/`**, 这个在过滤日志的时候非常重要。下面增加几个例子

```js
# 输出打印一行中出现 k 字符的行
awk 'NR==1 || /k/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt
awk 'NR==1 || $0~/k/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt

# 过滤掉一行中出现york字符的行
awk 'NR==1 || $0！~/york/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt

# 输出某个域字符以 k 开头的行
 awk 'NR==1 || $12~/^k/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt
```

### 2.6 数据统计

```js
awk '{if($9>0){printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}}' top.txt
2304     york     6.2      cinnamon          
12489    york     6.2      top    
```

上面的例子在扩展下，在结尾并统计下cpu的总和

```js
awk 'BEGIN {sum=0} {if($9>0){printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12; sum+=$9}} END {printf "cpu total usaged: %s\n", sum}' top.txt
2304     york     6.2      cinnamon          
12489    york     6.2      top               
cpu total usaged: 12.4
```

```js
shell> awk 'NR!=1{a[$2]++;} END {for (i in a) print i ", " a[i];}' top.txt
york, 2
root, 9
```

这个例子用一个数组统计不同用户的进程个数，并在最后用循环打印出来，这里有两个新的概念，一个是另外一种流程控制**循环**，另一个是数组的使用。关于循环的控制语法如下，和其它高级语言都类似。 

### 2.7 怎样使用数组

上面看到了数组的基本使用，其实 awk 给数组赋予了很多功能，和很多高级脚本语言一样，提供了相关的函数获取长度，排序等，另外存储是 *key-value* 结构，能像map一样判断key是否存在。

*获取长度 length* 

```js
shell> awk 'BEGIN{info="it is a test";lens=split(info,tA," ");print length(tA),lens;}'
4 4 
```

> 上面 split 函数用于字符串分割，和c语言的又是一毛一样

*循环输出*

```js
shell> awk 'BEGIN{info="it is a test";split(info,tA," ");for(k in tA){print k,tA[k];}}'
4 test
1 it
2 is
3 a 
```

> 由于是 *key-value* 的存储结构，所以使用这样的for循环输出结果是无序的，

*有序输出*

```js
shell> awk 'BEGIN{info="it is a test";tlen=split(info,tA," ");for(k=1;k<=tlen;k++){print k,tA[k];}}'
1 it
2 is
3 a
4 test
```

> 注意：数组下标从1开始的 注意：这种输出方法仅适用于把数组真正当作 ‘数组’ 使用，key值就是自然递增的数，而不是当map

*判断是否存在 key in array*

```js
shell> awk 'BEGIN{tB["a"]="a1";tB["b"]="b1";if( "c" in tB){print "ok";};for(k in tB){print k,tB[k];}}'
a a1
b b1
```

> 注意：判断语法是 `key in array` 不能直接写 `array[key] != value` 形式，如果这样写它会默认创建一个，使用过高级脚本语言的都知道；

*删除 deletekey*

```js
shell> awk 'BEGIN{tB["a"]="a1";tB["b"]="b1";delete tB["a"];for(k in tB){print k,tB[k];}}'                     
b b1
```







