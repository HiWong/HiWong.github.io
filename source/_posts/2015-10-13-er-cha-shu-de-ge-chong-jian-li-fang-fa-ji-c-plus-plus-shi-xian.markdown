---
layout: post
title: "二叉树的各种建立方法及C++实现"
date: 2015-10-13 20:41:49 +0800
comments: true
categories: Algorithm
---
二叉树是一种重要的数据结构，在文件管理、数据转发等领域都会用到。下面将详细地讲述二叉树的有关知识。首先是二叉树的建立。为了有更好的通用性，下面所有的代码都用实现。二叉树一般用二叉链表来表示，下面是其定义<!--more-->：  

	template<class T>
	struct BTNode
	{
		T data;
		BTNode<T>*leftChild,*rightChild;
	};

首先是根据广义表形式的字符串建立二叉树，如A(B(D,E(G)),C(F(,H)))  

	BTNode<char>*createBinaryTreeByGeneralList(char*a)
	{
		BTNode<char>*currentNode,*root=NULL,*stack[100];
		int top=-1,flag;
		for(char*p=a;*p!='\0';p++)
		{
			switch(*p)
			{
			case '(':
				stack[++top]=currentNode;
				flag=1;
				break;
			case ')':
				--top;
				break;
			case ',':
				flag=2;
				break;
			default:
				currentNode=new BTNode<char>();
				currentNode->data=*p;
				currentNode->leftChild=NULL;
				currentNode->rightChild=NULL;
				if(root==NULL)
					root=currentNode;
				else if(flag==1)
					stack[top]->leftChild=currentNode;
				else
					stack[top]->rightChild=currentNode;
			}
		}
		return root;

	}

下面这个是按前序遍历序列顺序存储的数组建立二叉树  

	template<class T>
	BTNode<T>*createBinaryTreeByPreOrderSequence(T valueArray[],int n,T flagValue)//暂时还没想到非递归的方法，所以就不加参数bool recursionFlag了。其中的flagValue是表示遇到它就代表相应的结点应该为NULL。
	{
		static int num=-1;
		if(num<n)
		{
			T temp=valueArray[++num];
			BTNode<T>*p=NULL;

			if(temp!=flagValue)
			{
				p=new BTNode<T>();
				p->data=temp;
				p->leftChild=createBinaryTreeByPreOrderSequence(valueArray,n,flagValue);
				p->rightChild=createBinaryTreeByPreOrderSequence(valueArray,n,flagValue);
			}
			return p;
		}
		
	}
	///////////////////////////////////////////下面这个函数是根据按层次遍历序列顺序存储的数组建立二叉树。
	template<class T>
	BTNode<T>*createBinaryTreeByLayerOrderSequence(T valueArray[],int n,T flagValue,int sub)//sub为下标。
	{
		if(sub>=n)
			return NULL;
		if(valueArray[sub]==flagValue)
			return NULL;
		else
		{
			BTNode<T>*p=new BTNode<T>();
			p->data=valueArray[sub];
			p->leftChild=NULL;
			p->rightChild=NULL;

			p->leftChild=createBinaryTreeByLayerOrderSequence(valueArray,n,flagValue,2*sub+1);
			p->rightChild=createBinaryTreeByLayerOrderSequence(valueArray,n,flagValue,2*sub+2);

			return p;
		}
	}

下面这个函数是根据按层次遍历序列顺序存储的数组建立二叉树  

	template<class T>
	BTNode<T>*createBinaryTreeByLayerOrderSequence(T valueArray[],int n,T flagValue,int sub)//sub为下标。
	{
		if(sub>=n)
			return NULL;
		if(valueArray[sub]==flagValue)
			return NULL;
		else
		{
			BTNode<T>*p=new BTNode<T>();
			p->data=valueArray[sub];
			p->leftChild=NULL;
			p->rightChild=NULL;

			p->leftChild=createBinaryTreeByLayerOrderSequence(valueArray,n,flagValue,2*sub+1);
			p->rightChild=createBinaryTreeByLayerOrderSequence(valueArray,n,flagValue,2*sub+2);

			return p;
		}
	}

下面是根据中序遍历和后序遍历序列建立二叉树，即此时数组中没有用以标示节点为NULL的flagValue。  

	template<class T>
	BTNode<T>*createBinaryTreeByInAndPostOrderTraversal(T inOrderArray[],T postOrderArray[],int inOrderStart,int inOrderEnd,int postOrderStart,int postOrderEnd)
	{
		BTNode<T>*p=new BTNode<T>();
		p->data=postOrderArray[postOrderEnd];
		
	    int s1,e1,b1,t1;//s代表start,e代表end; b代表begin,t代表terminal.
		int s2,e2,b2,t2;
		
		int sub;
		sub=inOrderStart;
		while(inOrderArray[sub]!=postOrderArray[postOrderEnd])
		     sub++;

		if(sub==inOrderStart)
			p->leftChild=NULL;
		else
		{
			s1=inOrderStart;
			e1=sub-1;
			b1=postOrderStart;
			t1=b1+(e1-s1);//因为t1-b1==e1-s1;
			p->leftChild=createBinaryTreeByInAndPostOrderTraversal(inOrderArray,postOrderArray,s1,e1,b1,t1);
		}
		///////////////////
		if(sub==inOrderEnd)
			p->rightChild=NULL;
		else
		{
			s2=sub+1;
			e2=inOrderEnd;
			t2=postOrderEnd-1;
			b2=t2-(e2-s2);//因为t2-b2==e2-s2;
			p->rightChild=createBinaryTreeByInAndPostOrderTraversal(inOrderArray,postOrderArray,s2,e2,b2,t2);

		}
		return p;

	}

下面是根据中序和前序遍历序列建立二叉树。  

	template<class T>
	BTNode<T>*createBinaryTreeByPreAndInOrderTraversal(T preOrderArray[],T inOrderArray[],int preStart,int preEnd,int inStart,int inEnd)
	{
		BTNode<T>*p=new BTNode<T>();
		p->data=preOrderArray[preStart];

		int sub=inStart;
		while(inOrderArray[sub]!=preOrderArray[preStart])
			sub++;


		if(sub==inStart)
			p->leftChild=NULL;
		else
		{
			int s1,e1,b1,t1;
			b1=inStart;
			t1=sub-1;
			s1=preStart+1;
			e1=s1+(t1-b1);
			p->leftChild=createBinaryTreeByPreAndInOrderTraversal(preOrderArray,inOrderArray,s1,e1,b1,t1);

		}
		/////////////////////
		if(sub==inEnd)
			p->rightChild=NULL;
		else
		{
			int s2,e2,b2,t2;//其实写成preStart02,preEnd02,inStart02,inEnd02会更好。
			
			b2=sub+1;
			t2=inEnd;
			e2=preEnd;
			s2=e2-(t2-b2);//注意：不能通过s2=e1+1来计算，因为按自己那种写法，e1可能都没赋值。如果不管是哪种情况都赋值的话当然就可以。但是显然现在这种写法是比较合理而又比较美观的。
			p->rightChild=createBinaryTreeByPreAndInOrderTraversal(preOrderArray,inOrderArray,s2,e2,b2,t2);

		}
		return p;
	}

下面要建立二叉查找树。  

	template<class T>
	BTNode<T>*createSortTree(T valueArray[],int length,bool recursionFlag)
	{
		BTNode<T>*root=NULL;

		for(int i=0;i<length;++i)
		{
			insertSortTree(root,valueArray[i],recursionFlag);
		}
		return root;
	}
	template<class T>
	void insertSortTree(BTNode<T>*&root,T item,bool recursionFlag)
	{
	 if(recursionFlag)
	 {
	  BTNode<T>*p=new BTNode<T>();
	  p->data=item;
	  p->leftChild=NULL;
	  p->rightChild=NULL;
	  if(root==NULL)
	  {
	   root=p;
	   return;
	  }
	  else if(item<root->data)
	   insertSortTree(root->leftChild,item,true);
	  else
	   insertSortTree(root->rightChild,item,true);</p><p> }
	 else
	 {
	  BTNode<T>*newNode=new BTNode<T>();
	  newNode->data=item;
	  newNode->leftChild=NULL;
	  newNode->rightChild=NULL;</p><p>  BTNode<T>*p=root;
	  if(root==NULL)
	  {
	   root=newNode;
	   return;
	  }
	  else
	  {
	   while(1)
	   {
	    if(item<p->data)
	    {
	     if(p->leftChild!=NULL)
	           p=p->leftChild;
	     else
	     {
	      p->leftChild=newNode;
	      break;
	     }
	    }
	    else
	    {
	     if(p->rightChild!=NULL)
	      p=p->rightChild;
	     else
	     {
	      p->rightChild=newNode;
	      break;
	     }    } }
	  
	  }}}

下面这个函数将用于创建完全二叉树。其实上面那个函数createBinaryTreeByLayerOrderSequence就可以创建完全二叉树，只要所提供的数组没有flagValue就可以。  

	template<class T>
	BTNode<T>*createCompleteBinaryTree(T valueArray[],int n,int sub,bool recursionFlag)
	{
		if(recursionFlag)
		{
			if(sub>=n)
				return NULL;
			BTNode<T>*p=new BTNode<T>();
			p->data=valueArray[sub];
			p->leftChild=createCompleteBinaryTree(valueArray,n,2*sub+1,true);
			p->rightChild=createCompleteBinaryTree(valueArray,n,2*sub+2,true);
			return p;

		}
	}


  

