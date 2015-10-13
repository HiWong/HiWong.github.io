---
layout: post
title: "二叉树的常用操作(C++实现)"
date: 2015-10-13 20:33:33 +0800
comments: true
categories: Algorithm
---
前面已经罗列了二叉树的建立、遍历的各种方法，但是二叉树作为一种重要的数据结构，它还有一些很常用的操作，比如寻找结点，删除二叉树中的结点，复制二叉树，判断二叉树是否相似、相等，等待。  

空谈误国，直接上代码。代码中有相应的说明<!--more-->。  

	#ifndef OTHEROPERATIONS_H
	#define OTHEROPERATIONS_H
	#include <stack>
	#include <deque>
	#include <queue>
	#include "BTNode.h"

	////////////////////////////////////这个函数用于在普通二叉树中寻找节点。
	template<class T>
	BTNode<T>*findNodeInBinaryTree(BTNode<T>*root,T item,bool recursionFlag)
	{
		if(recursionFlag) //当然，采用递归方法时，也可以利用某种遍历方法。比如这里的其实就是前序遍历方法。
		{
			if(root!=NULL)
			{
				if(root->data==item)
					return root;
				else
				{
					BTNode<T>*possibleLeft,*possibleRight;
					possibleLeft=findNodeInBinaryTree(root->leftChild,item,true);
					possibleRight=findNodeInBinaryTree(root->rightChild,item,true);
					if(possibleLeft!=NULL)
						return possibleLeft;
					else if(possibleRight!=NULL)
						return possibleRight;
					else
						return NULL;
				}
			}
			else   //其实这样写不好，这样写很容易忘记return NULL这一种情况。所以比较好的做法是先写简单的情形。
			    return NULL;

		}
		else //采用非递归的方法。其实也就是非递归的遍历方法。这里采用非递归的中序遍历方法。
		{
			BTNode<T>*p=root;
			stack<BTNode<T>*>nodeStack;
			do 
			{
				while(p!=NULL)
				{
					nodeStack.push(p);
					p=p->leftChild;
				}

				p=nodeStack.top();
				nodeStack.pop();
				if(p->data==item)
					return p;
				p=p->rightChild;//别忘了这一步，要不然遍历不到右子树。

			} while (!nodeStack.empty()||p!=NULL);

			//如果上面都结束了还没有返回的话，那要么是找不到，要么root==NULL，那么就应该返回NULL。
			return NULL;
		}
	}
	////////////////////////////////////能否写一个函数可以同时找到相应的节点及其双亲结点呢？其实是可以的，而且只要再增加一个变量即可。
	template<class T>
	void findNodeAndParentInBT(BTNode<T>*root,T item,BTNode<T>*&node,BTNode<T>*&parentNode,bool recursionFlag)
	{
		if(recursionFlag)
		{
			if(root==NULL)
			{
				node=NULL;
				parentNode=NULL;
				return;
			}
			else
			{
				if(item==root->data)
				{
					node=root;
					parentNode=NULL;
					return;
				}
				else
				{
					if(root->leftChild!=NULL)
					{
						if(root->leftChild->data==item)
						{
							node=root->leftChild;
							parentNode=root;
							return;
						}
						else
							//findNodeAndParentInBT(root->leftChild,item,possibleLeftNode,possibleLeftParent,true);
							findNodeAndParentInBT(root->leftChild,item,node,parentNode,true);

					}
				    ///////////////////////////////这其实就类似中序遍历，即先查找左子树，然后查找右子树。

					if(root->rightChild!=NULL)
					{
						if(root->rightChild->data==item)
						{
							node=root->rightChild;
							parentNode=root;
							return;
						}
						else
							//findNodeAndParentInBT(root->rightChild,item,possibleRightNode,possibleRightParent,true);
							findNodeAndParentInBT(root->rightChild,item,node,parentNode,true);
					}	
				}
			
			}

		}
		else//下面运用非递归方法。
		{
			if(root==NULL)
			{
				node=NULL;
				parentNode=NULL;
				return;
			}
			else
			{
				if(root->data==item)
				{
					node=root;
					parentNode=NULL;
					return;      
				}
				else
				{
					stack<BTNode<T>*>nodeStack;
					stack<BTNode<T>*>parentStack;
					nodeStack.push(root);
					parentStack.push(NULL);

					BTNode<T>*p1=root;
					BTNode<T>*p2=p1->leftChild;
					//BTNode<T>*p;
					do 
					{
						while(p2!=NULL)
						{
							nodeStack.push(p2);
							parentStack.push(p1);
							p1=p1->leftChild;
							p2=p2->leftChild;
						}

						p2=nodeStack.top();
						p1=parentStack.top();
						nodeStack.pop();
						parentStack.pop();
						//////////////////
						if(p2->data==item)
						{
							node=p2;
							parentNode=p1;
							return;
						}

						/////////////////如果不满足这个条件的话就要转向右子树。
						p1=p2; //加上这句这重要，一定要记得时候让p1是p2的双亲结点。这点要时刻保持不变。
						p2=p2->rightChild;

					} while (!nodeStack.empty()||p2!=NULL); //注意：此时这里的控制条件不能写成(p2!=NULL&&p1!=NULL)，记住，p1只是我们附加的变量，它变不起控制程序流程的作用。

				}

			}
		}

	}

	//////////////////////////////很简单，只要按照寻找父结点的方法来寻找就行了。
	template<class T>
	void findNodeAndParentInSortTree(BTNode<T>*root,T item,BTNode<T>*&node,BTNode<T>*&parentNode,bool recursionFlag)
	{
		if(recursionFlag)
		{
			if(root==NULL)
			{
				node=NULL;
				parentNode=NULL;
				return;
			}
			else
			{
				if(root->data==item) //注意这种情况并不多余，因为可能要找的那个节点就是根结点呢。
				{
					node=root;
					parentNode=NULL;
					return;
				}
				else if(item<root->data)
				{
					if(root->leftChild!=NULL)
					{
						if(item==root->leftChild->data)
						{
							node=root->leftChild;
							parentNode=root;
							return;
						}
						else
							findNodeAndParentInSortTree(root->leftChild,item,node,parentNode,true);
					}
					////其实这个else完全可以不要，因为实际上if(root==NULL)那里就把这种情况考虑进去了，等下测试一下就知道。
					/*
					else
					{
						node=NULL;
						parentNode=NULL;
						return;
					}
					*/

				}
				else
				{
					if(root->rightChild!=NULL)
					{
						if(item==root->rightChild->data)
						{
							node=root->rightChild;
							parentNode=root;
							return;
						}
						else
							findNodeAndParentInSortTree(root->rightChild,item,node,parentNode,true);

					}
					///////////这个else语句也可以不要的。
					/*
					else
					{
						node=NULL;
						parentNode=NULL;
					}
					*/

				}

			}

		}
		else
		{
			if(root==NULL)
			{
				node=NULL;
				parentNode=NULL;
				return;
			}
			else
			{
				if(item==root->data)
				{
					node=root;
					parentNode=NULL;
					return;
				}
				else
				{
					BTNode<T>*p=root;
					while(p!=NULL)
					{
						if(item<p->data)
						{
							if(p->leftChild!=NULL)
							{
								if(item==p->leftChild->data)
								{
									node=p->leftChild;
									parentNode=p;
									return;
								}
								else
								{
									p=p->leftChild;
									//continue;//这个continue好像是多余的。
								}
							} //如果while里面没加那个判断条件的话，这里就要加一个else条件。
							
						}
						else
						{
							if(p->rightChild!=NULL)
							{
								if(item==p->rightChild->data)
								{
									node=p->rightChild;
									parentNode=p;
									return;
								}
								else
								{
									p=p->rightChild;
								}
							}

						}

					}
					//////////////////////////如果没有找到的话，那就只能是返回NULL了。
					node=NULL;
					parentNode=NULL;
				    return;

				}

			}

		}

	}





	////////////////////////////////这个函数用于在二叉查找树中寻找节点。
	template<class T>
	BTNode<T>*findNodeInSortTree(BTNode<T>*root,T item,bool recursionFlag)
	{
		//////////////////////////////////当然可以采用某种遍历的方法来寻找节点，但那只是针对一般的二叉树而言，对于二叉排序树没有必要这样。
		if(recursionFlag)
		{
			if(root==NULL)
				return NULL;
			else
			{
				if(item==root->data)
					return root;
				else if(item<root->data)
					return findNodeInSortTree(root->leftChild,item,true);
				else
					return findNodeInSortTree(root->rightChild,item,true);
			}
		}
		else
		{
			BTNode<T>*p=root;
			while(1)
			{
				if(p==NULL)
					return NULL;
				if(item==p->data)
					return p;
				else if(item<p->data)
				{
					p=p->leftChild;
				}
				else
					p=p->rightChild;

			}
		}

	}

	////////////////////////////////这个函数用于在二叉查找树中寻找某个节点的双亲节点。
	template<typename T>
	BTNode<T>*findParentNodeInSortTree(BTNode<T>*root,T item,bool recursionFlag)//还要写一个findParentNodeInSortTree(BTNode<T>*root,BTNode<T>*node);的函数。要找的是item的父结点。
	{
		if(recursionFlag)
		{
			if(root==NULL)
				return NULL;
			else
			{
				if(item==root->data)
					return NULL;
				else if(item<root->data)
				{
					if(root->leftChild!=NULL)
					{
						if(item==root->leftChild->data)
							return root;
						else
							return findParentNodeInSortTree(root->leftChild,item,true);	
					}
					else
						return NULL;

				}
				else
				{
					if(root->rightChild!=NULL)
					{
						if(item==root->rightChild->data)
							return root;
						else
							return findParentNodeInSortTree(root->rightChild,item,true);
					}
					else
						return NULL;
				}

			}
			
		}
		else
		{
			BTNode<T>*p=root;
			while(1)
			{
				if(p==NULL||root->data==item)  //会分这几种情况很重要。特别注意的是虽然要跟p的孩子结点进行值的比较，但是分情况却还是要根据item与p->data的关系来分，因为这决定了要找的结点会在哪一边。
					return NULL;
				else if(item<p->data)
				{
					if(p->leftChild!=NULL)
					{
						if(item==p->leftChild->data)
							return p;
						else
							p=p->leftChild;
							
					}
					else 
						return NULL;
				}
				else
				{
					if(p->rightChild!=NULL)
					{
						if(item==p->rightChild->data)
							return p;
						else
							p=p->rightChild;
					}
					else
						return NULL;

				}

			}

		}

	}
	///////////////////////////////////////////////////////////////////////下面这个函数是给出结点的地址，要寻找它的双亲结点，其方法与上一个函数类似。可以利用函数重载的方法，而没有必要另起一个函数名。
	template<class T>
	BTNode<T>*findParentNodeInSortTree(BTNode<T>*root,BTNode<T>*node,bool recursionFlag)
	{
		if(recursionFlag)
		{
			if(root==NULL||root==node)
				return NULL;
			else if(node->data<root->data)
			{
				if(root->leftChild!=NULL)
				{
					if(root->leftChild==node)
						return root;
					else
						return findParentNodeInSortTree(root->leftChild,node,true);
				}
				else
					return NULL;

			}
			else
			{
				if(root->rightChild!=NULL)
				{
					if(node==root->rightChild)
						return root;
					else
						return findParentNodeInSortTree(root->rightChild,node,true);
				}
				else
					return NULL;
			}

		}
		else //采用非递归方法。
		{
			BTNode<T>*p=root;
			while(1)
			{
				if(p==NULL||root==node)
					return NULL;
				else if(node->data<p->data)
				{
					if(p->leftChild!=NULL)
					{
						if(p->leftChild==node)
							return p;
						else
							p=p->leftChild;
					}
					else
						return NULL; //其实这个分类情况跟if(p==NULL||root==node)重复了。但是提前一步返回也未尝不可。

				}
				else
				{
					if(p->rightChild!=NULL)
					{
						if(p->rightChild==node)
							return p;
						else
							p=p->rightChild;
					}
					else
						return NULL;

				}

			}
		}
	}

	/////////////////////////下面这个函数用于删除二叉查找树中的结点。
	template<class T>
	void deleteNodeInSortTree(BTNode<T>*&root,BTNode<T>*targetNode,BTNode<T>*targetParentNode)
	{
		BTNode<T>*r,*s;
		int flag=0;
		if(targetNode->leftChild==NULL)
		{
			if(targetNode==root)
				root=targetNode->rightChild;
			else
			{
				r=targetNode->rightChild;
				flag=1;
			}

		}
		else if(targetNode->rightChild==NULL)
		{
			if(targetNode==root)
				root=targetNode->leftChild;
			else
			{
				r=targetNode->leftChild;
				flag=1;
			}
		}
		else
		{
			////////////////////////首先要找到targetNode的右子树中最小的那个节点。
			s=targetNode;
			r=s->rightChild;
			while(r->leftChild!=NULL)
			{
				s=r;
				r=r->leftChild;
			}
			r->leftChild=targetNode->leftChild;//千万别忘了这句。
			/////////////////至此，就找到了targetNode的右子树中最小的那个节点。
			if(s!=targetNode)
			{
				s->leftChild=r->rightChild;
				r->rightChild=targetNode->rightChild;
			}

			/////////////////////////////下面这几句特别重要千万别忘了。
			if(targetNode==root)
				root=r;
			else
				flag=1;

		}
		if(flag==1)
		{
			if(targetParentNode->leftChild==targetNode)
				targetParentNode->leftChild=r;
			else
				targetParentNode->rightChild=r;

			delete targetNode;
			targetNode=NULL;

		}

	}


	//////////////////////////////////////////下面这个函数用于销毁一棵二叉树。
	template<class T>
	void destroyBinaryTree(BTNode<T>*&root,bool recursionFlag)//要销毁二叉树的话，显然是用后序遍历的方法最合适。
	{
		if(recursionFlag)
		{
			if(root==NULL)
				return;
			else
			{
				destroyBinaryTree(root->leftChild,true);//这里之所以不用加if(root->leftChild!=NULL)这个条件是因为前面的if(root==NULL)其实就已经把这种情况考虑进去了。
				destroyBinaryTree(root->rightChild,true);
				
				delete root;
				root=NULL;
			}

		}
		else
		{
			/////////////////////////////很简单，也用后序遍历的非递归方法就行嘛。
			if(root==NULL)
				return;
			else
			{
				stack<BTNode<T>*>nodeStack;
				stack<int>flagStack;
				int flag;
				BTNode<T>*p=root;
				do 
				{
					while(p!=NULL)
					{
						nodeStack.push(p);
						flagStack.push(1);
						p=p->leftChild;
					}

					p=nodeStack.top();
					nodeStack.pop();
					flag=flagStack.top();
					flagStack.pop();
					////////////////////
					if(flag==1)
					{
						nodeStack.push(p);
						flagStack.push(2);
						p=p->rightChild;//这句很关键，如果忘了的话就遍历不到右子树。
					}
					else
					{
						delete p;
						p=NULL;
					}

				} while (!nodeStack.empty()||p!=NULL);

			}

		}

	}

	///////////////////////////////////////////////////////////
	template<class T>
	void deleteNodeInSortTree(BTNode<T>*&root,T item)
	{
		BTNode<T>*targetNode=findNodeInSortTree(root,item,true);
		BTNode<T>*targetParentNode=findParentNodeInSortTree(root,item,true);
		deleteNodeInSortTree(root,targetNode,targetParentNode);
		cout<<"删除节点"<<item<<"成功!"<<endl;

	}
	//////////////////////////////////////////////////////////////////////下面这个函数用于删除普通二叉树中的节点。必须明确的一点是：对于二叉查找树，可以单独删除一个节点，因为删除后其它点可以按原来的规则进行整合；
	//但是，对于普通的二叉树，没有通用的整合规则，所以我们在删除普通二叉树的结点时，不可能单独删除它这一个，还必须删除它的左子树和右子树。
	//另外，要记得将它的双亲结点的左子树或者右子树置为NULL。
	template<class T>
	void deleteNodeInOrdinaryBinaryTree(BTNode<T>*&root,T item,bool recursionFlag)  
	{
		//////////////////////////首先要找到该节点。记得分类讨论。
		BTNode<T>*node,*parentNode;
		findNodeAndParentInSortTree(root,item,node,parentNode,recursionFlag);
		if(node==NULL)//即没有找到。
			return;
		else if(parentNode==NULL)//说明是根节点。因为除了根结点之外，其它所有结点都有双亲结点。
		{
			destroyBinaryTree(root,recursionFlag);
			root=NULL;
		}
		else
		{
		   if(parentNode->leftChild==node)
			   parentNode->leftChild=NULL;
		   else
			   parentNode->rightChild=NULL;
		   destroyBinaryTree(node,recursionFlag);

		}

	}

	/////////////////////////////////////////////下面这个函数用于判断两个二叉树是否相似。
	template<class T>
	bool isSimiliar(BTNode<T>*root01,BTNode<T>*root02,bool recursionFlag)
	{
		if(recursionFlag)
		{
			if(root01==NULL&&root02==NULL)
				return true;
			else if(root01==NULL||root02==NULL)
				return false;
			else
			{
				return (isSimiliar(root01->leftChild,root02->leftChild,true))&&(isSimiliar(root01->rightChild,root02->rightChild,true));
			}

		}
		else //采用非递归的方法该怎么判断呢?利用两个堆栈应该就可以判断吧!
		{
			BTNode<T>*p1=root01,*p2=root02;
			stack<BTNode<T>*>nodeStack1;
			stack<BTNode<T>*>nodeStack2;
			do 
			{
				while(p1!=NULL&&p2!=NULL)
				{
					nodeStack1.push(p1);
					nodeStack2.push(p2);
					p1=p1->leftChild;
					p2=p2->leftChild;
				}
				/////////////////////////如果只是因为其中的某一个为NULL而跳出，那么肯定是不相似。
				if(p1!=NULL||p2!=NULL)
					return false;
				
				p1=nodeStack1.top();
				nodeStack1.pop();
				p2=nodeStack2.top();
				nodeStack2.pop();
				/////////////////////////////
				p1=p1->rightChild;
				p2=p2->rightChild;

			} while ((!nodeStack1.empty()||p1!=NULL)&&(!nodeStack2.empty()||p2!=NULL));

			//////////////////////////////////////////如果中断循环是因为有一个stack为空或者只是一个指针为NULL的话，就要返回false.
			//if(nodeStack1.empty()||nodeStack2.empty()) //首先要判断导致循环中断的条件是什么。仔细想一下的话就会发现这个判断是多余的。因为它会中断循环至少有一个stack为空而且p为NULL。
				if(!nodeStack1.empty()||!nodeStack2.empty())
					return false;
			
			//if(p1==NULL||p2==NULL)  //同样地，这个判断条件也是多余的。
			    if(p1!=NULL||p2!=NULL)
					return false;
			/////////////////////////////////////如果到这里还没有返回的话，那就说明是相似了。
			return true;

		}

	}


	////////////////////////下面这个函数用于判断两个二叉树是否相等。
	template<class T>
	bool isEqual(BTNode<T>*root01,BTNode<T>*root02,bool recursionFlag)
	{
		if(recursionFlag)
		{
			if(root01==NULL&&root02==NULL)
				return true;
			else if(root01==NULL||root02==NULL)//这里其实既可以写成现在这样，也可以写成else if(root01!=NULL||root02!=NULL)
				return false;
			else
			{
				if(root01->data==root02->data)
					//return true; //此时不是return true,而是应该继续往下比较。
					return isEqual(root01->leftChild,root02->leftChild,true)&&isEqual(root01->rightChild,root02->rightChild,true);

				else
					return false;

			}

		}
		else //采用非递归算法该怎么比较呢？
		{
			stack<BTNode<T>*>nodeStack1,nodeStack2;
			BTNode<T>*p1=root01,*p2=root02;
			do 
			{
				while(p1!=NULL&&p2!=NULL)
				{
					nodeStack1.push(p1);
					nodeStack2.push(p2);
					p1=p1->leftChild;
					p2=p2->leftChild;
				}
				///////////////////////
				if(p1!=NULL||p2!=NULL)
					return false;
				////////////////////////
				p1=nodeStack1.top();
				nodeStack1.pop();
				p2=nodeStack2.top();
				nodeStack2.pop();
				////////////////////////
				if(p1->data!=p2->data)
					return false;
				else
				{
					p1=p1->rightChild;
					p2=p2->rightChild;
				}

			} while ((!nodeStack1.empty()||p1!=NULL)&&(!nodeStack2.empty()||p2!=NULL));

			////////////////////////////////////
			if(!nodeStack1.empty()||!nodeStack2.empty())
				return false;
			if(p1!=NULL||p2!=NULL)
				return false;
			///////////////////////////////////如果到这里还没有return false的话，就要return true了。
			return true;
		}

	}

	///////////////////////////////////////////////////////////下面这个函数用于获取二叉树的尝试。
	template<typename T>
	int getBinaryTreeDepth(BTNode<T>*root,bool recursionFlag)
	{
		if(recursionFlag)
		{
			if(root==NULL)
				return 0;
			else
				//return 1+getBinaryTreeDepth(root->leftChild)+getBinaryTreeDepth(root->rightChild);//1就是当前不为NULL的节点。
			{
				int maxDepth,leftDepth,rightDepth;
				leftDepth=getBinaryTreeDepth(root->leftChild,true);
				rightDepth=getBinaryTreeDepth(root->rightChild,true);
				maxDepth=leftDepth>rightDepth?leftDepth:rightDepth;
				return 1+maxDepth;
			}
		
		}
		else  //此处采用类似中序遍历的方法。这里是用堆栈实现的，那么能否利用队列实现呢？
		{
			if(root==NULL)
				return 0;


			BTNode<T>*nodeStack[100],*p=root;
			int depthStack[100];
			int currentDepth=1,maxDepth=0,top=-1;

			do 
			{
				while(p!=NULL)
				{
					nodeStack[++top]=p;
					depthStack[top]=currentDepth;
					p=p->leftChild;
					++currentDepth;

				}

				p=nodeStack[top];
				currentDepth=depthStack[top--];

				if(p->leftChild==NULL&&p->rightChild==NULL)
				{
					if(currentDepth>maxDepth)
						maxDepth=currentDepth;
				}
				
				p=p->rightChild;
				++currentDepth;
			} while (top>=0||p!=NULL);

			return maxDepth;

			//下面这个是利用了STL，经过测试发现正确。
			/*
			if(root==NULL)
				return 0;  //空树的深度为0.

			stack<BTNode<T>*>nodeStack;
			stack<int>depthStack;
			BTNode<T>*p=root;
			int currentDepth=1,maxDepth=0;
			do 
			{
				while(p!=NULL)
				{
					nodeStack.push(p);
					depthStack.push(currentDepth);
					++currentDepth;
					p=p->leftChild;

				}
				p=nodeStack.top();
				nodeStack.pop();
				currentDepth=depthStack.top();
				depthStack.pop();
				if(p->leftChild==NULL&&p->rightChild==NULL)
				{
					if(currentDepth>maxDepth)
						maxDepth=currentDepth;
				}

				p=p->rightChild;
				++currentDepth;//只要是进入下一层，它的深度就要加1.

			} while (!nodeStack.empty()||p!=NULL);
			return maxDepth;
			*/

		}

	}

	/////////////////////////////////////////////////////////下面这个函数用于求某个值在二叉树中所处的层次。
	template<class T>
	int getLayerNum(BTNode<T>*root,T item,bool recursionFlag,int methodNum=1)
	{
		if(recursionFlag)//这个递归算法其实是比非递归算法更难想到。
		{
			if(root==NULL)
				return 0;
			else if(root->data==item)
				return 1;
			int layerNum;
			layerNum=getLayerNum(root->leftChild,item,true);
			if(layerNum>0)
				return (layerNum+1);//因为要加上根结点嘛。
			/////////////////////////////////如果到这里还没有返回的话，就要遍历右子树了。
			layerNum=getLayerNum(root->rightChild,item,true);
			if(layerNum>0)
				return (layerNum+1);
			//////////////////////////如果遍历完右子树还没有返回，就表示还没有找到，这样就要返回0了。
			return 0;

		}
		else  //虽然都是求深度，但是这里有3种方法，分别对应前序、中序、后序遍历方法。
		{
			if(methodNum==1)
			{
				if(root==NULL)
					return 0;

				stack<BTNode<T>*>nodeStack;
				stack<int>depthStack;
				int currentDepth=1;
				BTNode<T>*p=root;

				do 
				{
					while(p!=NULL)
					{
						nodeStack.push(p);
						depthStack.push(currentDepth);
						p=p->leftChild; //只要是向下走，深度就要加1.
						currentDepth++;
					}

					p=nodeStack.top();
					nodeStack.pop();
					currentDepth=depthStack.top();
					depthStack.pop();

					if(p->data==item)
						return currentDepth;
					else  //这个else其实可写可不写。
					{
						p=p->rightChild;
						++currentDepth;
					}

				} while (!nodeStack.empty()||p!=NULL);

				return 0;//如果遍历所有结点还是没有找到的话，那么肯定是二叉树中不存在这样的结点，所以返回0.
			}
			else if(methodNum==2)//前面是采用类似中序遍历的方法来求深度，在这个过程中就可以求出节点所在的层次。当然也可以采用类似前序遍历的方法来求深度。
			{
				if(root==NULL)
					return 0;
				stack<BTNode<T>*>nodeStack;
				stack<int>depthStack;
				int currentDepth=1;
				BTNode<T>*p=root;
				do 
				{
					while(p!=NULL)
					{
						if(p->data==item)
							return currentDepth;
						//else
						nodeStack.push(p);
						depthStack.push(currentDepth);
						p=p->leftChild;
						++currentDepth;//只要是向下，深度就要加1.

					}

					p=nodeStack.top();
					nodeStack.pop();
					currentDepth=depthStack.top();
					depthStack.pop();

					p=p->rightChild;//只要是向下，深度就要加1.
					++currentDepth;

				} while (!nodeStack.empty()||p!=NULL);

				/////////////////////如果遍历所有节点还没有找到的话，就要返回0.
				return 0;

			}
			else  //用后序遍历的算法时由于currentDepth==top+2==stack.size()+1，所以就不必再加一个depthStack了。
			{
				if(root==NULL)
					return 0;

				stack<BTNode<T>*>nodeStack;
				stack<int>flagStack;
				BTNode<T>*p=root;
				int flag;
				do 
				{
					while(p!=NULL)
					{
						nodeStack.push(p);
						flagStack.push(1);
						p=p->leftChild;
					}

					p=nodeStack.top();
					nodeStack.pop();
					flag=flagStack.top();
					flagStack.pop();
					if(flag==1)
					{
						nodeStack.push(p);
						flagStack.push(2);
						p=p->rightChild;
					}
					else
					{
						if(p->data==item)
							return (nodeStack.size()+1);
						//else 
						   p=NULL;
					}
				} while (!nodeStack.empty()||p!=NULL);
				////////////////////////如果遍历完还没找到，就返回0.
				return 0;
			}

		}

	}

	///////////////////////////////////////////////////////////下面这个函数用于得到叶节点的数目。
	template<class T>
	int getLeafNodeCount(BTNode<T>*root,bool recursionFlag)
	{
		if(recursionFlag)
		{
			if(root==NULL)
				return 0;
			
			if(root->leftChild==NULL&&root->rightChild==NULL)  //这种分类方法极化。如果是按root->left!=NULL及root->rightChild!=NULL来分的话，情况就会很多很乱。
				return 1;
			else
				return getLeafNodeCount(root->leftChild,true)+getLeafNodeCount(root->rightChild,true);
			
		}
		else //对于非递归方法，思路很简单，就是进行遍历，对每个结点进行判断。此处采用按层次遍历的非递归算法。
		{
			if(root==NULL)
				return 0;

			BTNode<T>*p;
			deque<BTNode<T>*>nodeQueue;
			nodeQueue.push_back(root);
			int leafCount=0;
			while(!nodeQueue.empty())
			{
				p=nodeQueue.front();
				nodeQueue.pop_front();
				if(p->leftChild==NULL&&p->rightChild==NULL)
					++leafCount;

				if(p->leftChild!=NULL)
					nodeQueue.push_back(p->leftChild);
				if(p->rightChild!=NULL)
					nodeQueue.push_back(p->rightChild);

			}
			return leafCount; 

		}

	}

	/////////////////////////////////////////////////////////////下面这个函数用于求二叉树中结点的个数，其实好简单，就是遍历计数就行。
	template<typename T>
	int getNodeCount(BTNode<T>*root,bool recursionFlag,int methodNum=1)
	{
		if(recursionFlag) //前序遍历。
		{
			static int nodeCount=0;
			if(root==NULL)
				return 0;
			
			++nodeCount;
			getNodeCount(root->leftChild,true);
			getNodeCount(root->rightChild,true);
			return nodeCount;
			
		}
		else if(methodNum==1)//前序
		{
			if(root==NULL)
				return 0;
			int nodeCount=0;	
			BTNode<T>*nodeStack[100],*p=root;
			int top=-1;
			do 
			{
				while(p!=NULL)
				{
				    ++nodeCount;
					nodeStack[++top]=p;
					p=p->leftChild;
				}  
				p=nodeStack[top--];
				p=p->rightChild;

			} while (top>=0||p!=NULL);

			return nodeCount;

		}
		else if(methodNum==2)//中序遍历。
		{
			if(root==NULL)
				return 0;

			BTNode<T>*p=root;
			stack<BTNode<T>*>nodeStack;
			long nodeCount=0;
			do 
			{
				while(p!=NULL)
				{
					nodeStack.push(p);
					p=p->leftChild;
				}

				p=nodeStack.top();
				nodeStack.pop();
				++nodeCount; 
				p=p->rightChild;

			} while (!nodeStack.empty()||p!=NULL);

			return nodeCount;
		}
		else if(methodNum==3)//后序
		{
			if(root==NULL)
				return 0;

			BTNode<T>*p=root;
			stack<BTNode<T>*>nodeStack;
			stack<bool>flagStack; //其实用bool就行了。bool型变量只占一个字节。
			bool flag;
			long nodeCount=0;
			do 
			{
				while(p!=NULL)
				{
					nodeStack.push(p);
				    flagStack.push(false);
					p=p->leftChild;
				}

				p=nodeStack.top();
				nodeStack.pop();
				flag=flagStack.top();
				flagStack.pop();

				if(flag==false)
				{
					nodeStack.push(p);
					flagStack.push(true);
					p=p->rightChild;
				}
				else
				{
					++nodeCount;
					p=NULL; //这一步很关键，没有它的话就会陷入死循环。
				}

			} while (!nodeStack.empty()||p!=NULL);

			return nodeCount;

		}
		else //层次遍历。
		{
			if(root==NULL)
				return 0;
			long nodeCount=0;
			BTNode<T>*p;
			deque<BTNode<T>*>nodeQueue;
			nodeQueue.push_back(root);

			while(!nodeQueue.empty())
			{
				p=nodeQueue.front();
				nodeQueue.pop_front();
				++nodeCount;

				if(p->leftChild!=NULL)
					nodeQueue.push_back(p->leftChild);
				if(p->rightChild!=NULL)
					nodeQueue.push_back(p->rightChild);

			}

			return nodeCount;

		}
	}

	/////////////////////////////////////////////////////////下面这个函数用于求第k层的结点数。
	template<class T>
	int getNodeCountOfSpecificLayer(BTNode<T>*root,int k,bool recursionFlag,short methodNum=1)
	{
		static int depth=getBinaryTreeDepth(root,true);//首先要求出树的深度，k不能超过当前树的深度。
		if(recursionFlag)
		{
			if(root==NULL||k<=0||k>depth)
				return 0;
			if(k==1)
				return 1;	
			int leftNodeCount=getNodeCountOfSpecificLayer(root->leftChild,k-1,true); //注意：对于根结点为第k层的结点，对于根结点的子结点构成的二叉树来说，它就处于第k-1层，因而可以一直这样递归下去。
			int rightNodeCount=getNodeCountOfSpecificLayer(root->rightChild,k-1,true);
			return leftNodeCount+rightNodeCount;
		}
		else if(methodNum==1) //知道了怎么求节点所在的层次的话，这个非递归方法就不在话下了。这里把种方法都写一下。首先写后序的方法。
		{
			if(root==NULL||k<=0||k>depth)
				return 0;
			BTNode<T>*nodeStack[100],*p=root;
			bool flagStack[100]; //用bool比较节省内存。
			bool flag;
			int top=-1;
			int nodeCount=0;
			do 
			{
				while(p!=NULL)
				{
					nodeStack[++top]=p;
					flagStack[top]=false;
					p=p->leftChild;
				}

				p=nodeStack[top];
				flag=flagStack[top--];

				if(flag==false)
				{
					nodeStack[++top]=p;
					flagStack[top]=true;
					p=p->rightChild;//因为是后序，所以先不遍历根结点，而是转到右结点。
				}
				else
				{
					if(top+2==k) //即如果当前结点就在第k 层。
						nodeCount++;
					p=NULL; //这一步很关键。
				}


			} while (top>=0||p!=NULL);

			return nodeCount;	
		}
		else if(methodNum==2) //中序方法。
		{
			if(root==NULL||k<=0||k>depth)
				return 0;
			stack<BTNode<T>*>nodeStack;
			stack<int>depthStack;
			BTNode<T>*p=root;
			int currentDepth=1,nodeCount=0;

			do 
			{
				while(p!=NULL)
				{
					nodeStack.push(p);
					depthStack.push(currentDepth);
					p=p->leftChild;
					++currentDepth; 
				}

				p=nodeStack.top();
				nodeStack.pop();
				currentDepth=depthStack.top();
				depthStack.pop();

				if(currentDepth==k)
					++nodeCount;

				p=p->rightChild;
				currentDepth++;
			} while (!nodeStack.empty()||p!=NULL);

			return nodeCount;

		}
		else //前序方法。
		{
			if(root==NULL||k<=0||k>depth)
				return 0;
			BTNode<T>*p=root;
			stack<BTNode<T>*>nodeStack;
			stack<int>depthStack;
			int currentDepth=1;
			long nodeCount=0;
			do 
			{
				while(p!=NULL)
				{
					nodeStack.push(p);
					depthStack.push(currentDepth);
					if(currentDepth==k)
						++nodeCount;
					p=p->leftChild;
					++currentDepth;
				}
				p=nodeStack.top();
				nodeStack.pop();
				currentDepth=depthStack.top();
				depthStack.pop();
				p=p->rightChild;
				++currentDepth;//只要向下，深度就要加1.

			} while (!nodeStack.empty()||p!=NULL);

			return nodeCount;

		}

	}

	//////////////////////////////////////////////////////////////////////下面这个函数，用于交换其所有的左右结点。
	template<class T>
	void exchangeLeftAndRightChild(BTNode<T>*&root,bool recursionFlag,int methodNum=1)
	{
		if(recursionFlag)
		{
			if(methodNum==1)
			{
				if(root==NULL)
					return;
				//事实证明这个交换的部分可以放在exchangeLeftAndRightChild(root->leftChild,true)及exchangeLeftAndRightChild(root->rightChild,true)的前面，中间和后面。这其实就是一个前序、中序、后序遍历的问题。
				BTNode<T>*temp=root->leftChild;
				root->leftChild=root->rightChild;
				root->rightChild=temp;
				////////////////
				exchangeLeftAndRightChild(root->leftChild,true);
				exchangeLeftAndRightChild(root->rightChild,true);
			}
			else if(methodNum==2)
			{
				if(root==NULL)
					return;	
				exchangeLeftAndRightChild(root->leftChild,true);
				//事实证明这个交换的部分可以放在exchangeLeftAndRightChild(root->leftChild,true)及exchangeLeftAndRightChild(root->rightChild,true)的前面，中间和后面。这其实就是一个前序、中序、后序遍历的问题。
				BTNode<T>*temp=root->leftChild;
				root->leftChild=root->rightChild;
				root->rightChild=temp;
				////////////////
				exchangeLeftAndRightChild(root->rightChild,true);
			}
			else
			{
				if(root==NULL)
					return;	
				exchangeLeftAndRightChild(root->leftChild,true);
				exchangeLeftAndRightChild(root->rightChild,true);
				//事实证明这个交换的部分可以放在exchangeLeftAndRightChild(root->leftChild,true)及exchangeLeftAndRightChild(root->rightChild,true)的前面，中间和后面。这其实就是一个前序、中序、后序遍历的问题。
				BTNode<T>*temp=root->leftChild;
				root->leftChild=root->rightChild;
				root->rightChild=temp;
			}
			

		}
		else//该操作采用按层次遍历方法比较合适。遍历过程中访问一个结点时，就将该结点的左、右子树的位置进行交换。当然，其实我们也可以遍历每个结点，交换其左右子树即可。所以非递归算法其实也有三种，只是此处采用层遍历会快一点。
		{
			
			if(root==NULL)
				return;
			deque<BTNode<T>*>nodeQueue;
			BTNode<T>*p,*temp;
			nodeQueue.push_back(root);
			while(!nodeQueue.empty())
			{
				p=nodeQueue.front();
				nodeQueue.pop_front();
				temp=p->leftChild;
				p->leftChild=p->rightChild;
				p->rightChild=temp;
				if(p->leftChild!=NULL)
					nodeQueue.push_back(p->leftChild);
				if(p->rightChild!=NULL)
					nodeQueue.push_back(p->rightChild);
			}

		}

	}

	////////////////////////////////////////////////////////////下面这个函数用于复制二叉树。
	template<class T>
	BTNode<T>*copyBinayTree(BTNode<T>*root,bool recursionFlag)
	{
		if(recursionFlag)  //这里只能采用前序遍历的方式来建立。
		{
			if(root==NULL)
				return NULL;
			BTNode<T>*p=new BTNode<T>();
			p->data=root->data;
			p->leftChild=copyBinayTree(root->leftChild,true);
			p->rightChild=copyBinayTree(root->rightChild,true);
			return p;
		}
		else   //这里采用的是类似前序遍历的方式来复制二叉树，当然也可以采用中序遍历的方式，甚至是后序遍历的方式。
		{
			if(root==NULL)
				return NULL;
			BTNode<T>*p=root,*copyRoot,*p2;
			p2=new BTNode<T>();
			copyRoot=p2;
			stack<BTNode<T>*>nodeStack,nodeStack2;
			do 
			{
				while(p!=NULL)
				{
					p2->data=p->data;
					nodeStack.push(p);
					nodeStack2.push(p2);
					//////////////
					p=p->leftChild;
					if(p!=NULL)
					    p2->leftChild=new BTNode<T>();
					else
						p2->leftChild=NULL;

					p2=p2->leftChild;
				}
				///////////////////
				p=nodeStack.top();
				nodeStack.pop();
				p2=nodeStack2.top();
				nodeStack2.pop();

				p=p->rightChild;

				if(p!=NULL)//注意：此时p已经是原来的p的右孩子。
					p2->rightChild=new BTNode<T>();	
				else
					p2->rightChild=NULL;

				p2=p2->rightChild;//当时自己因为忘了这一步而出错。

			} while (!nodeStack.empty()||p!=NULL);

			return copyRoot;//当时犯了一个好低级的错误，就是忘记设置返回值了。

		}

	}

	/////////////////////////////////////////////////下面这个函数用于判断一棵二叉树是否为完全二叉树。关键是要弄清楚完全二叉树的定义：在一棵二叉树中，只有最下面两层的度可以小于2，并且最下面一层的结点都依次排列在该层最左边的位置上。
	template<class T>
	bool isCompleteBinaryTree(BTNode<T>*root,bool recursionFlag,short methodNum=1)
	{
		if(recursionFlag)//	我们可以采用类似前序遍历的方法。
		{
			if(root==NULL)
				return false;
			static bool mustHaveNoChild=false;//如果不想用static的话，就要将这个标志作为函数参数向下传递。
			BTNode<T>*p;
		    if(mustHaveNoChild)
			{
				if(root->leftChild!=NULL||root->rightChild!=NULL)
					return false;
			}
			else
			{
				if(root->leftChild!=NULL&&root!=NULL)
				{
					return isCompleteBinaryTree(root->leftChild,true)&&isCompleteBinaryTree(root->rightChild,true);
				}
				else if(root->leftChild!=NULL&root->rightChild==NULL)
				{
					mustHaveNoChild=true;
					return isCompleteBinaryTree(root->rightChild,true);
				}
				else if(root->leftChild==NULL&&root->rightChild!=NULL)
					return false;
				else //这个好像是多余的吧，在递归中不会出现这种情况吧？
				{
					mustHaveNoChild=true;
				}

			}

		}
		else  if(methodNum==1)
		{
			if(root==NULL)
				return false;

			BTNode<T>*p;
			queue<BTNode<T>*>nodeQueue;//用queue其实比deque更好。记住最小权限准则。
			nodeQueue.push(root);
			bool mustHaveNoChild=false;
			while(!nodeQueue.empty())
			{
				p=nodeQueue.front();
				nodeQueue.pop();

				if(mustHaveNoChild)
				{
					if(p->leftChild!=NULL||p->rightChild!=NULL)
						return false;
				}
				else
				{
					if(p->leftChild!=NULL&&p->rightChild!=NULL)
					{
						nodeQueue.push(p->leftChild);
						nodeQueue.push(p->rightChild);//这样就保证是从左到右搜索。
					}
					else if(p->leftChild!=NULL&&p->rightChild==NULL)
					{
						mustHaveNoChild=true;
						nodeQueue.push(p->leftChild);//别忘了这一步。
					}
					else if(p->leftChild==NULL&&p->rightChild!=NULL)
						return false;
					else
						mustHaveNoChild=true;//左右子树都为空的话，那么mustHaveNoChild当然就要为true了。但是也可能是最后一层的叶结点啊，如果是叶结点的话，不能这样吧？它是把叶结点的情况排除了吗？因为它是按层次遍历，到了叶结点这一层的话，所有的结点都不会有子树，所以就不要紧了。这就是他要采用按层次遍历的真实原因!
				}

			}
			/////////////////////////////////如果一直到结束循环都还没返回的话，就说明它是完全二叉树了。
			return true;

		}
		else if(methodNum==2)//其实非遍历的方法根本不止这一种，不用层次遍历的话，用前序遍历也是能搞定的。事实证明，用前序的方法是没法验证一棵树是否为完全二叉树的，至少用上面的那种思想不行，因为前序方法会有回退，即从更低层回到更上层，而mustHaveNoChild其实是针对同一层及其下层而言的，对于它的上层并没有约束力。
		{
			if(root==NULL)
				return false;
			BTNode<T>*p=root;
			stack<BTNode<T>*>nodeStack;
			bool mushHaveNoChild=false;

			int depth=getBinaryTreeDepth(root,true);
			stack<int>depthStack;
			int currentDepth=1;
			int lastNodeDepth=1;//初始值为1.

			do 
			{
				while(p!=NULL)
				{
					nodeStack.push(p);
					depthStack.push(currentDepth);
					p=p->leftChild;
					currentDepth++;
				}

				p=nodeStack.top();
				nodeStack.pop();

				currentDepth=depthStack.top();
				depthStack.pop();


				if(mushHaveNoChild)
				{
					if(p->leftChild!=NULL||p->rightChild!=NULL)
						return false;
				}
				else  //下面的程序写得还不够简洁。
				{
					if(p->leftChild!=NULL&&p->rightChild!=NULL)
					{
						//p=p->rightChild;
					}
					else if(p->leftChild!=NULL&&p->rightChild==NULL)
					{
						mushHaveNoChild=true;
						//p=p->rightChild;
						
					}
					else if(p->leftChild==NULL&&p->rightChild!=NULL)
						return false;
					else //要如何排除最后一层的叶结点的情况呢？要排除的话，肯定要加辅助变量，比如先求出其深度，然后加一个深度堆栈，如果是在最后一层的话，出现这种情况就不要置mustHaveNoChild为true。
					{
						if(currentDepth>=lastNodeDepth&¤tDepth<depth) //即使是加了一个lastNodeDepth进去，还是得不到正确结果，说明还是有什么地方没考虑周全。
						    mushHaveNoChild=true;
						//p=p->rightChild;
					}

					////////////////////////
					p=p->rightChild;
					lastNodeDepth=currentDepth++;
					//currentDepth++;

				}

			} while (!nodeStack.empty()||p!=NULL);


			return true;

		}


	}

	#endif

