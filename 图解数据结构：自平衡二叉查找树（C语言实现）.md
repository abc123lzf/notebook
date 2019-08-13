---
title: 图解数据结构：自平衡二叉查找树（C语言实现）
date: 2017-10-22 13:43:22
tags: CSDN迁移
---
 版权声明：尊重原创，转载请注明出处 [ https://blog.csdn.net/abc123lzf/article/details/78309618]( https://blog.csdn.net/abc123lzf/article/details/78309618)   
   （1）二叉查找树的定义



在计算机科学中，二叉树是每个节点最多有两个子树的树结构。通常子树被称作“左子树”（left subtree）和“右子树”（right subtree）。二叉树常被用于实现二叉查找树和二叉堆。  
二叉树的每个结点至多只有二棵子树(不存在度大于2的结点)，二叉树的子树有左右之分，次序不能颠倒。每一棵子树的根叫做根r的儿子，而r是每一棵子树的根的父亲。  
具有N个节点的每一棵二叉树都将需要N+1个NULL指针。对于树中的每个结点X，它的左子树中所有关键字值小于X的关键字值，而它的右子树中所有关键字值大于X的关键字值。  
二叉查找树的平均深度为O（log N）。  
（2）AVL树与非自平衡二叉查找树

相比较于非自平衡二叉查找树，AVL树是带有平衡条件的二叉查找树。这个平衡条件必须需要保持。对于一棵AVL树，每个结点的左子树和右子树的高度最多差1。在这里，我们将空树的高度定义为-1。所以一个问题在于，插入一个结点可能破坏AVL树的特性，所以我们可以通过对树的修正来完成，我们称其为旋转。

（3）AVL树的旋转

由于任意结点最多有两个儿子，因此高度不平衡时，a点的两个子树的高度差为2。容易看出，这种不平衡容易可能出现在下面4种情形。

1、对a的左儿子的左子树进行一次插入。

2、对a的左儿子的右子树进行一次插入。

3、对a的右儿子的左子树进行一次插入。

4、对a的右儿子的右子树进行一次插入。

第一种情况是插入发生在外边的情况（即左-左的情况或者右-右的情况），该情况可通过对树的一次单旋转而完成调整。第二种则是发生在内部的情形。（即左-右情况或右-左情况），该情况可通过一次双旋转完成。

（4）AVL树的操作例程。

1、声明




```
#include <stdio.h>
#include <stdlib.h>

#ifndef _AVLTREE_H
#define _AVLTREE_H
#define DATA_TYPE int

struct AvlNode;
typedef struct AvlNode *Position;
typedef struct AvlNode *Tree;

Tree Delete_tree(Tree T);							//删除树
Tree New_tree(DATA_TYPE X);							//新建一棵树
Position Find(Tree T, DATA_TYPE X);					//查找
Position Find_Min(Tree T);							//查找树的最小值
Position Find_Max(Tree T);							//查找树的最大值
Position Insert(Tree T, DATA_TYPE X);				//插入结点
Position Delete(Tree T, DATA_TYPE X);				//
Position Print_tree(Tree T);						//先序遍历二叉树
static int Height(Position P);						//计算当前结点的高度
static Position Single_Rotate_left(Position K2);	//左左单旋转
static Position Single_Rotate_right(Position K2);	//右右单旋转
static Position Double_Rotate_left(Position K3);	//左右双旋转
static Position Double_Rotate_right(Position K3);	//右左双旋转
static int Max(int a, int b);
static Position Fix(Position K2);
struct AvlNode
{
	DATA_TYPE num;
	Tree left;
	Tree right;
	int height;
};

#endif
```
  


  
2、结点的高度与判定最大值函数




```
Tree Delete_tree(Tree T)
{
	if (T != NULL)
	{
		Delete_tree(T->left);
		Delete_tree(T->right);
		free(T);
	}
	return NULL;
}

static int Height(Position P)
{
	if (P == NULL)	//如果树为空则高度为-1
		return -1;
	else
		return P->height;
}

```
  
3、新建查找树与删除查找树




```
Tree New_tree(DATA_TYPE X)
{
	Position first;
	first = malloc(sizeof(struct AvlNode));
	if (first == NULL)
		return NULL;
	first->num = X;
	first->right = NULL;
	first->left = NULL;
	return first;
}

Tree Delete_tree(Tree T)
{
	if (T != NULL)
	{
		Delete_tree(T->left);
		Delete_tree(T->right);
		free(T);
	}
	return NULL;
}
```
  
4、查找目标节点、查找最小结点和查找最大结点




```
Position Find(Tree T, DATA_TYPE X)
{
	while (T != NULL)
	{
		if (X < T->num)
			T = T->left;
		else if (X > T->num)
			T = T->left;
		else return T;
	}
	return NULL;
}

Position Find_Min(Tree T)
{
	if (T != NULL)
	{
		for (; T->left != NULL; T = T->left);
		return T;
	}
	return T;
}

Position Find_Max(Tree T)
{
	if (T != NULL)
	{
		for (; T->left != NULL; T = T->right);
		return T;
	}
	return NULL;
}

```
  
5、插入结点




```
Position Insert(Tree T, DATA_TYPE X)
{
	if (T == NULL)	//成功找出插入位置
	{
		T = malloc(sizeof(struct AvlNode));
		if (T == NULL)
			return NULL;
		else
		{
			T->num = X;
			T->height = 0;
			T->left = T->right = NULL;
		}
	}
	else if (X < T->num)			//如果待插入结点小于该节点，则移向左子树
	{
		T->left = Insert(T->left, X);
		if (Height(T->left) - Height(T->right) == 2)//判定左子树高度与右子树高度差为2
			if (X < T->left->num)		//判定情况属于L-L还是L-R
			T = Single_Rotate_left(T);
			else
			T = Double_Rotate_left(T);
	}
	else if (X > T->num)			//如果待插入结点大于该节点，则移向右子树
	{
		T->right = Insert(T->right, X);
		if (Height(T->right) - Height(T->left) == 2)//判定右子树高度与左子树高度差为2
			if (X > T->right->num)		//判定情况属于R-R还是R-L
				T = Single_Rotate_right(T);
			else
				T = Double_Rotate_right(T);
	}
	T->height = Max(Height(T->left), Height(T->right)) + 1;	//更新结点高度
	return T;
}
```
  
6、删除结点




```
Position Delete(Tree T, DATA_TYPE X)
{
	Position TmpCell;

	if (T == NULL)
	{
		printf("Element not found!!\n");
		return NULL;
	}
	else if (T->num > X)						//往左边查找元素X
		T->num = Delete(X, T->left);
	else if (T->num < X)						//往右边查找元素X
		T->num = Delete(X, T->right);
	else if (T->left && T->right)				//元素找到了并且左右子树都不为空
	{
		TmpCell = FindMin(T->right);		    //将要删除的节点用右子树中的最小节点代替
		T->num = TmpCell->num;
		T->right = Delete(T->num, T->right);
	}
	else                                      //元素找到了并且左右子树有一个为空
	{
		TmpCell = T;
		if (T->left == NULL)
			T = T->right; 
		else if (T->right == NULL)
			T = T->left;

		free(TmpCell);
	}

	if (T != NULL)
	{
		T->height = Max(Height(T->left), Height(T->right)) + 1;		//删除完节点之后要更新高度
		//判断是否在节点T处失去平衡
		if ((Height(T->left) - Height(T->right) >= 2) || (Height(T->right) - Height(T->left) >= 2))
		{
			T = Fix(T);
			T->height = Max(Height(T->left), Height(T->right)) + 1;
		}
	}
	return T; 
}
```



```
static Position Fix(Position K2)
{
	if (Height(K2->left) > Height(K2->right))
	{
		//K2左儿子的左儿子的高度大于K2的左儿子的右儿子的高度, 执行左单旋转, 否则执行左-右双旋转
		if (Height(K2->left->left) > Height(K2->left->right))
			K2 = Single_Rotate_Left(K2);
		else if (Height(K2->left->left) < Height(K2->left->right))
			K2 = Double_Rotate_Left(K2);
	}
	else if (Height(K2->left) < Height(K2->right))
	{
		//K2右儿子的右儿子的高度大于K2的右儿子的左儿子的高度, 执行右单旋转, 否则执行右-左双旋转
		if (Height(K2->right->right) > Height(K2->right->left))
			K2 = Single_Rotate_Right(K2);
		else if (Height(K2->right->right) < Height(K2->right->left))
			K2 = Double_Rotate_Right(K2);
	}

	return K2;
}
```
  
  




7、先序打印树中的结点




```
Position Print_tree(Tree T)
{
	if (T == NULL)
	{
		printf("The Tree is empty.\n");
		return NULL;
	}
	else if (T->left != NULL && T->right != NULL)
	{
		printf("%d ", T->num);
		T->left = Print_tree(T->left);
		T->right = Print_tree(T->right);
	}
	else if (T->left != NULL && T->right == NULL)
	{
		printf("%d ", T->num);
		T->left = Print_tree(T->left);
	}
	else if (T->left == NULL && T->right != NULL)
	{
		printf("%d ", T->num);
		T->right = Print_tree(T->right);
	}
	else if (T->left == NULL && T->right == NULL)
		printf("%d ", T->num);
	return T;
}
```
  
8、树的旋转

左左单旋转




```
static Position Single_Rotate_left(Position K2)
{
	Position K1;
	K1 = K2->left;
	K2->left = K1->right;
	K1->right = K2;

	K2->height = Max(Height(K2->left), Height(K2->right)) + 1;
	K1->height = Max(Height(K1->left), K2->right) + 1;
	return K1;
}
```
  
![](https://img-blog.csdn.net/20171022182727236?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  


右右单旋转


```
static Position Single_Rotate_right(Position K2) 
{
	Position K1;
	K1 = K2->right;
	K2->right = K1->left;
	K1->left = K2;

	K2->height = Max(Height(K2->right), Height(K2->left)) + 1;
	K1->height = Max(Height(K1->right), K2->height) + 1;
	return K1;
}
```
  
![](https://img-blog.csdn.net/20171022182807281?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  




左右双旋转




```
static Position Double_Rotate_left(Position K3)	
{
	K3->left = Single_Rotate_right(K3->left);
	return Single_Rotate_left(K3);
}
```
![](https://img-blog.csdn.net/20171022182844337?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  


右左双旋转




```
static Position Double_Rotate_right(Position K3)
{
	K3->right = Single_Rotate_left(K3->right);
	return Single_Rotate_right(K3);
}
```
  
![](https://img-blog.csdn.net/20171022182902178?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYWJjMTIzbHpm/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  
  
  
  


   
 