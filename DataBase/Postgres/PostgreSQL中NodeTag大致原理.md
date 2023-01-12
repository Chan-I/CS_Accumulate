## NodeTag引入的必要性

设想一下在比较大的工程中，往往不同的信息需要分门别类的保存在对应的结构体中，面对数量巨大的结构体，有时候需要将逻辑类似的函数拆分成专职处理不同类型数据结构的函数，这样一来无疑会增加代码的编写难度。能不能考虑这样一个功能，使用同样的函数，通过一种特有的方式实现传入不同类型的结构体，进而实现有选择性的进行处理。

然后在函数内部有针对性的进行不同类型的处理。

## 局部小实验

现在不妨进行一个局部小实验，使用的就是C语言的一些特性。

同样都是func函数，但是传入的数据类型不同，导致处理的流程有所不同。

也就是说可以把整个结构体的第一个成员作为标记位，由于整个结构体的内存是连续的，但当使用不同类型的结构体进行访问时，就会获得不同的信息。

```c
#include <stdio.h>
#include <string.h>

typedef struct {
        int head;
}Node;

typedef struct {
        Node node;
        char body[32];
}NBode;


void func(void *in)
{
        switch(((Node *)in)->head)
        {
                default:
                        sprintf(((NBode *)in) -> body,"head is %d",((Node *)in)->head);
                        break;
                case -1:
                        sprintf(((NBode *)in) -> body,"Go Fuxx URself!");
                        break;
        }
}

int main()
{
        NBode node1,node2,node3;

        node1.node.head = 2;
        node2.node.head = 1;
        node3.node.head = -1;

        func(&node1);
        func(&node2);
        func(&node3);

        printf("%s\n",node1.body);
        printf("%s\n",node2.body);
        printf("%s\n",node3.body);
}
```

以上，就一定程度上通过一个标记位，实现了代码逻辑的不重复。

## 数据结构

每一种处理不同内容的数据结构都有着类似的结构，并且会经历大致相同的代码逻辑。

一下列举了两种不同类型的数据结构：T_Value和T_Stmt。其中Value主要用于存储数值，Stmt主要存储text信息。作为数据结构头部信息的NodeTag则为已经定义好的枚举类型。

```c
typedef enum NodeTag { T_Stmt，T_Value } NodeTag; 

+-------------------------------+-------------------------------+
|	Value						|	Stmt						|
|   		 NodeTag type;		|			NodeTag type;		|
|			 long val;			|			char * text;		|
+-------------------------------+-------------------------------+
```

这里定义一个新的Node类型，就将枚举类型NodeTag作为一个结构体，方便直接获取内容。

```c
// 直接将NodeTag作为一个结构体，方便直接进行类型的赋值和转换。
typedef struct Node{
    NodeTag type;
}Node;
```

### 数据结构初始化

当需要对上述两种不同的结构体进行初始化操作时需要明确其类型。这里用到的技巧就是，首先声明一个仅指向Node类型的指针，在实际进行malloc的时候传入的size数值为整个Stmt或者Value的大小。

先用一个小规模的测试进行一下。

```c
#include<stdio.h>
#include<stdlib.h>

typedef struct Small{
        int a;
}Small;

typedef struct Big{
        Small head;
        char  text[32];
}Big;

int main()
{
        Small * sml;
        sml = (Small *) malloc (sizeof(Big));
        sml->a = 1;
    
        sprintf(((Big *)sml)->text,"lakjlfkjsdlfkjasdf");
        printf("%d\t%s\n",sml->a, ((Big *) sml) -> text);
}
```

现在将这个代码中比较重要的额逻辑部分单独拎出来。顺道梳理一下NodeTag怎么定义这个通用宏。

```c
	Small * sml;
    sml = (Small *) malloc (sizeof(Big));
    sml->a = 1;
```

1. 先初始化NodeTag类型的指针。
2. 为这个指针malloc一个Stmt或者Value类型大小的空间。
3. 根据Stmt还是Value的类型设置NodeTag头部的类型信息。

具体的代码如下：

```c
#define newNode(size, tag) \         /* 传入参数为“需要设置的类型” + “该类型对应的空间大小” */
({                                              \
         Node *_result;    \
         assert((size) >= sizeof(Node));        /* 检测申请的内存大小，>>=sizeof(Node) */ \
         _result = (Node *) malloc(size);       /* 申请内存 */ \
         _result->type = (tag);                 /*设置TypeTag */ \
         _result;                               /*返回值*/\
})


// 一方面获取具体类型的size大小，另一方面要拼接成enum限定的T_Value或者T_Stmt的形式。
#define makeNode(_type_) ((_type_ *)newNode(sizeof(_type_),T_##_type_))
```

### 获取数据结构类型

当函数被传入一个数据结构的地址以后，需要对其进行NodeTag类型的获取。

使用的方式是，void *类型指针的转换。对于同一个地址，使用不同类型的指针访问时，获取到的内容也有所不同。

这里定义一个新的Node类型，就将枚举类型NodeTag作为一个结构体，方便直接获取内容。

```c
// 假设传入函数的变量为    obj。

// 直接将NodeTag作为一个结构体，方便直接进行类型的赋值和转换。
typedef struct Node{
    NodeTag type;
}Node;

// 直接判断obj结构体中NodeTag的类型。
switch( ((const Node *) obj) -> type )
```

可以直接抽象成一个宏定义

```c
#define nodeTag(_nodeptr_) (((const Node *)(_nodeptr_))->type)
```

## 代码逻辑

为了实现各个步骤有大致相同的前期处理逻辑，然后在必要的位置进行分支操作。

```c
char * func(void *obj)
{
       /* TO DO */

        switch(nodeTag(obj)){  // 判断obj结构体中 NodeTag的类型
                				   // 并根据不同类型走不同的处理逻辑。

                case T_Stmt:
                        // T_Stmt 相关的处理逻辑
                        break;
                
                case T_Value:
                        // T_Value 相关的处理逻辑
                        break;
                
                default:
                        // T_Other 相关逻辑
                
        }
        return r;
}
```



## 代码展示

```c
#include <stdio.h>
#include <stdbool.h>
#include <string.h>
#include <assert.h>
#include <stdlib.h>
#include <stdio.h>

typedef enum NodeTag
{
        T_Stmt,
        T_Value
}NodeTag;

typedef struct Node
{
        NodeTag type;
}Node;

#define newNode(size, tag) \
({                                              \
         Node *_result;  \
         assert((size) >= sizeof(Node));        /* 检测申请的内存大小，>>=sizeof(Node) */ \
         _result = (Node *) malloc(size);       /* 申请内存 */ \
         _result->type = (tag);                         /*设置TypeTag */ \
         _result;                                       /*返回值*/\
})

#define makeNode(_type_) ((_type_ *)newNode(sizeof(_type_),T_##_type_))
#define nodeTag(nodeptr) (((const Node *)(nodeptr))->type)


typedef struct Stmt
{
        NodeTag type;
        char *text;
}Stmt;

typedef struct Value
{
        NodeTag type;
        long val;
}Value;

bool equal(void *a,void *b)
{
        if(a == b)
                return true;

        if (a == NULL || b == NULL)
                return false;

        if(nodeTag(a) != nodeTag(b))
                return false;

        switch(nodeTag(a)){
                case T_Stmt:
                        return strcmp(((const Stmt*)a)->text,((const Stmt *)b)->text)==0? true:false;

                case T_Value:
                        return ((const Value *)a)->val==((const Value *)b)->val;
                default:
                        printf("error:unknown type\n");

        }
        return false;
}

char * nodetoString(void *obj)
{
        char *r =(char *)malloc(1024);

        if (obj == NULL){
                strcpy(r,"<>");
        }

        switch(nodeTag(obj)){

                case T_Stmt:
                        sprintf(r,"<Stmt:%s>",((const Stmt *)obj)->text);
                        break;
                case T_Value:
                        sprintf(r,"<Value:%ld>",((const Value *)obj)->val);
                        break;
                default:
                        strcpy(r,"<unknown node type>");
        }
        return r;
}



int main(int argc,char *argv[])
{
        Stmt *s = makeNode(Stmt);
        if(s){
                char str[]="select * from a";
                s->text=str;
        }

        Stmt *t= makeNode(Stmt);

        if(t){
                char str[]="select * from b";
                t->text=str;
        }

        Value *v=makeNode(Value);

        if(v)
                v->val=100;
     

        printf("t->text:%s\n",t->text);
        printf("equal:%d\n",equal(s,t));
        printf("%s\n",nodetoString(t));

        free(s);
        free(t);
        free(v);
        return 0;

}

```

## List和ListCell的说明

前文已经叙述过NodeTag的作用，这里就需要叙述一下ListCell和List的大致原理。

```c
typedef struct ListCell ListCell;
 
typedef struct List
{
	NodeTag		type;		/* T_List, T_IntList, or T_OidList, ... */
	int			length;
	ListCell   *head;
	ListCell   *tail;
} List;
 
struct ListCell
{
	union
	{
		void	   *ptr_value;
		int			int_value;
		Oid			oid_value;
	}			data;
	ListCell   *next;
};
```

 这两个类型的关系是，ListCell是一个单独的个体，作为一个容器来存储内容以及下一个 ListCell的指针。 

这其中，如果是一个由init或者Oid构成的List，那么这个ListCell直接存储init或者Oid。若不是，则使用void *来存储，这样可以存储的类型就多了。一般用的时候直接使用强制类型转换为(Type *)即可使用。

也就是说多个ListCell之间组成链表，然后List分别指着链表的头和尾。并标记一些必要的信息，包括链表的长度、listCell的类型等。

### 构建List类型

```c
static List *
new_list(NodeTag type)
{
	List	   *new_list;
	ListCell   *new_head;
 
	new_head = (ListCell *) palloc(sizeof(*new_head));
	new_head->next = NULL;
	/* new_head->data is left undefined! */
 
	new_list = (List *) palloc(sizeof(*new_list));
	new_list->type = type;
	new_list->length = 1;
	new_list->head = new_head;
	new_list->tail = new_head;
 
	return new_list;
}
```



