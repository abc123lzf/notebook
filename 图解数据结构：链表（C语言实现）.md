---
title: 图解数据结构：链表（C语言实现）
date: 2017-10-20 20:11:53
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/78298991]( https://blog.csdn.net/abc123lzf/article/details/78298991)   
   1、链表的定义

链表是由一连串的结构（称为结点）组成的，其中每个节点包含指向链表中下一个结点的指针（除最后一个结点为空指针），其中每个结点可以含有若干个存储数据的数据域。

2、优缺点

相比较于数组，使用链表结构可以克服数组需要预先知道数据大小的缺点，可以充分利用计算机内存空间，实现灵活的内存动态管理。但是链表失去了数组随机读取的优点，同时链表由于增加了结点的指针域，空间开销比较大。

单链表的结构图：

![](https://timgsa.baidu.com/timg?image&amp;quality=80&amp;size=b9999_10000&amp;sec=1508512235656&amp;di=64eadc27dd23dd1f191938bdec84bd2e&amp;imgtype=0&amp;src=http%3A%2F%2Fimages.cnitblog.com%2Fblog%2F311549%2F201307%2F23211716-f91bf0299ef94e6aaf388bc2ac82966c.jpg)

3、单链表的C语言实现

（1）类型声明




```
#include <stdlib.h>
#include <stdbool.h>

#define DATA_TYPE int
struct node;
typedef struct node* List;
typedef struct node* Position;

List New_List(List head,DATA_TYPE X);       //新建链表
bool is_empty(List head);					//判定该链表是否为空
bool is_last(Position P);		            //判定位置P是否为链表的末尾
Position Find(DATA_TYPE X, List head);		//查找元素，返回指向该元素的指针
void Delete(DATA_TYPE X, List head);		//删除元素
void Insert(DATA_TYPE X, List head);		//插入元素
List Delete_list(List head);                //删除链表

struct node
{
	DATA_TYPE data;
	Position next;
};

```
  
  


（2）新建链表操作




```
List New_List(List head,DATA_TYPE X)
{
	Position new_node;

	if (!is_empty)
	{
		printf("链表已存在\n");
		return NULL;
	}
	
	new_node = malloc(sizeof(struct node));
	if (new_node == NULL)
	{
		printf("分配失败\n");
		return NULL;
	}
	head = NULL;
	new_node->data = X;
	new_node->next = head;
	head = new_node;
	return head;
}
```
  
图示：

![](https://img-blog.csdn.net/20171021153825678?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  




  
（3）判断链表是否为空以及判断目标结点是否为该链表最后一个结点




```
bool is_empty(List head)
{
	return head->next == NULL;
}

bool is_last(Position P)
{
	return P->next == NULL;
}
```
  
  


  
（4）查找目标结点X




```
Position Find(DATA_TYPE X, List head)
{
	Position TmpPt;
	for (TmpPt = head; TmpPt->next != NULL; TmpPt = TmpPt->next)
		if (TmpPt->data == X)
			return TmpPt;
	return NULL;
}
```
  
  


  
（5）删除目标结点X




```
void Delete(DATA_TYPE X, List head)
{
	Position cur, prev,TmpPt;       //prev指向cur的前一节点

	TmpPt = head;
	for (cur = head, prev = NULL; cur != NULL&&cur->data != X; prev = cur, cur = cur->next);
	
	if (cur == NULL)
		return head;              //数据X未找到
	if (prev == NULL)
		TmpPt = TmpPt->next;      //数据X在头结点
	else
		prev->next = cur->next;   //在其他结点
	free(cur);
	return TmpPt;
		
}
```
  
在下列链表中，我们假设需要删除的结点为40。

![](https://img-blog.csdn.net/20171021153930009?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  




  
（6）插入结点X




```
void Insert(DATA_TYPE X, List head)
{
	Position TmpPt, new_node;
	new_node = malloc(sizeof(struct node));
	if (new_node == NULL)
	{
		printf("无法分配新结点\n");
		return NULL;
	}

	new_node->data = X;
	new_node->next = head;
	head = new_node;
}
```
  
图示：

![](https://img-blog.csdn.net/20171021153959143?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  




（7）删除链表




```
List Delete_list(List head)
{
	Position P, TmpPt;

	P = head->next;
	head->next = NULL;
	while (P != NULL)
	{
		TmpPt = P->next;
		free(P);
		P = TmpPt;
	}
	free(head);
	return NULL;
}
```
![](https://img-blog.csdn.net/20171021155734507?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  
  


（4）数据结构算法题



题目描述：假设一个算术表达式中可以包含三种括号：圆括号“(”和“)”，方括号“[”和“]”和花括号“{”和“}”，且这三种括号可按任意的次序嵌套使用（如：…[…{… …[…]…]…[…]…(…)…)。编写判别给定表达式中所含括号是否正确配对出现的算法。输出结果YES 或者 NO。

输入：5+{[2X5]+2}

输出：NO




```
#include<stdio.h>
#include<stdlib.h>
#define LEN 10000
void push_stack(const char par);
int pop_stack(const char par);
void clear_stack(void);

struct node
{
	char par;
	struct node *next;
};

struct node *top = NULL;
struct node *new_node;

char bracket[LEN + 1];

int main()
{
	int i, j;
	scanf("%s", bracket);
	for (j = 0;; j++)
	{
		if (bracket[j] == '(' || bracket[j] == '[' || bracket[j] == '{')
			push_stack(bracket[j]);
		else if (bracket[j] == ')' || bracket[j] == ']' || bracket[j] == '}')
		{
			if (pop_stack(bracket[j]) == -1)
				break;
		}
		else if (bracket[j] == '\0')
		{
			if (top != NULL)
			{
				printf("NO\n");
				break;
			}
				printf("YES\n");
				break;
			}
			else continue;
	}
	clear_stack();
	return 0;
}

void push_stack(const char par)
{
	new_node = (struct node *)malloc(sizeof(struct node));
	if (new_node == NULL) exit(EXIT_FAILURE);
	new_node->par = par;
	new_node->next = top;
	top = new_node;
}

int pop_stack(const char par)
{
	struct node *TmpPt;
	if (par == ')')
	{
		if (top == NULL)
		{
			printf("NO\n");
			return -1;
		}
		if (top->par != '(')
		{
			printf("NO\n");
			return -1;
		}
		else
		{
			TmpPt = top;
			top = top->next;
			free(TmpPt);
		}
	}
	else if (par == ']')
	{
		if (top == NULL)
		{
			printf("NO\n");
			return -1;
		}
		if (top->par != '[')
		{
			printf("NO\n");
			return -1;
		}
		else
		{
			TmpPt = top;
			top = top->next;
			free(TmpPt);
		}
	}
	else if (par == '}')
	{
		if (top == NULL)
		{
			printf("NO\n");
			return -1;
		}
		if (top->par != '{')
		{
			printf("NO\n");
			return -1;
		}
		else
		{
			TmpPt = top;
			top = top->next;
			free(TmpPt);
		}
	}
	return 0;
}

void clear_stack(void)
{
	struct node *TmpPt;
	while (top != NULL)
	{
		TmpPt = top;
		top = top->next;
		free(TmpPt);
	}
}

```
  
  




  
  


   
 