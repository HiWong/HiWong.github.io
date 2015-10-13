---
layout: post
title: "C++实现二叉树的递归和非递归遍历方法"
date: 2015-10-13 20:38:39 +0800
comments: true
categories: Algorithm
---
下面是二叉树和各种遍历方法，其中每种遍历方法都给出了递归和非递归的算法.<!--more-->

	template<class T>
	void preOrderTraversal(BTNode<T>*root,bool recursionFlag)
	{

		if(recursionFlag)//recursionFlag为真时则采用递归方法进行遍历。
		{
			if(root!=NULL)
			{
				cout<<root->data<<' ';

				//if(root->leftChild!=NULL)//这个虽然可加可不加，但是自己觉得其实加上会更好。
				   preOrderTraversal(root->leftChild,true);
				//if(root->rightChild!=NULL) 
				   preOrderTraversal(root->rightChild,true);

			}

		}
		else
		{
			BTNode<T>*p,*stack[100];
			int top=-1;
			p=root;
			do 
			{
				while(p!=NULL)
				{
					cout<<p->data<<' ';//因为是先序遍历嘛。
					stack[++top]=p;
					p=p->leftChild;
				}

				p=stack[top--];
				p=p->rightChild;

			} while (top>=0||p!=NULL);//之所以是||而不是&&是因为比如p==NULL时可能只是到了叶结点那里，但是此时显然不能结束。又比如根结点出栈之后top=-1，但是此时也应该继续下去，因为还有根结点的右孩子还要遍历嘛。

		}

	}

	//////////////////////////////////下面是中序遍历
	template<typename T>
	void inOrderTraversal(BTNode<T>*root,bool recursionFlag)
	{
		if(recursionFlag)
		{
			if(root!=NULL)
			{
				inOrderTraversal(root->leftChild,true);
				cout<<root->data<<' ';
				inOrderTraversal(root->rightChild,true);
			}

		}
		else
		{
			stack<BTNode<T>*>nodeStack;
			BTNode<T>*p=root;
			do 
			{
				while(p!=NULL)
				{
					nodeStack.push(p);
					p=p->leftChild;
				}

				p=nodeStack.top();
				nodeStack.pop();
				cout<<p->data<<' ';
				p=p->rightChild;

			} while (!nodeStack.empty()||p!=NULL);

		}

	}
	///////////////////////////////下面是后序遍历序列。
	template<class T>
	void postOrderTraversal(BTNode<T>*root,bool recursionFlag)
	{
		if(recursionFlag)
		{
			if(root!=NULL)
			{
				postOrderTraversal(root->leftChild,true);
				postOrderTraversal(root->rightChild,true);
				cout<<root->data<<' ';
			}
		}
		else
		{
			stack<BTNode<T>*>nodeStack;
			BTNode<T>*p=root;
			stack<int>flagStack;
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
					cout<<p->data<<' ';
					p=NULL;//这一句很重要，没有这一句的话就会陷入无穷循环中。
				}
				
			} while (!nodeStack.empty()||p!=NULL);

		}

	}

	////////////////////////////////////////////下面是按层次遍历。也称广度优先遍历。
	template<class T>
	void layerOrderTraversal(BTNode<T>*root,bool recursionFlag)
	{
		if(recursionFlag)//注意：这里即使是递归，也要借助队列。
		{
			static deque<BTNode<T>*>nodeQueue;
			static BTNode<T>*p;
			if(root==NULL)//注意：这句很重要，它是递归执行的。
				return;
			cout<<root->data<<' ';
			if(root->leftChild!=NULL)
				nodeQueue.push_back(root->leftChild);
			if(root->rightChild!=NULL)
				nodeQueue.push_back(root->rightChild);

			if(!nodeQueue.empty())//加这个判断条件很重要，没有它的话就会出错。
			{
				p=nodeQueue.front();
				nodeQueue.pop_front();
				layerOrderTraversal(p,true);
			}
			else
				return;

		}
		else
		{
			if(root==NULL)
				return;

			deque<BTNode<T>*>nodeQueue;
		    nodeQueue.push_back(root);
			BTNode<T>*p;	
			while(!nodeQueue.empty())
			{
				p=nodeQueue.front();
	            nodeQueue.pop_front();//注意是pop_front()而不是pop_back()，一定要保证从后面进，从前面出的队列性质。如果从后面进又从后面出的话就变成堆栈了。
				cout<<p->data<<' ';
				if(p->leftChild!=NULL)
				    nodeQueue.push_back(p->leftChild);
				if(p->rightChild!=NULL)
				    nodeQueue.push_back(p->rightChild);
			}

		}

	}

	///////////////////////////////////////
	template<class T>
	void depthFirstTraversal(BTNode<T>*root,int flag,bool recursionFlag)//flag==1时为前序遍历，flag==2时为中序遍历。但是这里要写非递归算法比上面的更简洁。
	{
		if(recursionFlag)
		{
			if(flag==1)
			{
				if(root!=NULL)
				{
					cout<<root->data<<' ';
					depthFirstTraversal(root->leftChild,1,true);
					depthFirstTraversal(root->rightChild,1,true);
				}

			}
			else if(flag==2)
			{
				if(root!=NULL)
				{
					depthFirstTraversal(root->leftChild,2,true);
					cout<<root->data<<' ';
					depthFirstTraversal(root->rightChild,2,true);
				}

			}
			else
			{
				cout<<"flag error!"<<endl;
				return;
			}

		}
		else
		{
			if(flag==1)
			{
				if(root==NULL)
					return;
				stack<BTNode<T>*>nodeStack;
				BTNode<T>*p;
				nodeStack.push(root);
				while(!nodeStack.empty())
				{
					p=nodeStack.top();
					nodeStack.pop();
					cout<<p->data<<' ';
					if(p->rightChild!=NULL)//堆栈是先进后出，所以如果要先遍历左子树的话，就要让右子树先进堆栈。
						nodeStack.push(p->rightChild);
					if(p->leftChild!=NULL)
						nodeStack.push(p->leftChild);

				}

			}
			else if(flag==2)
			{
				if(root==NULL)
					return;
				stack<BTNode<T>*>nodeStack;
				stack<int>flagStack;
				BTNode<T>*p;
				nodeStack.push(root);
				flagStack.push(1);

				while(!nodeStack.empty())
				{
					p=nodeStack.top();
						
					if(p->leftChild!=NULL&&flagStack.top()==1)
					{
						flagStack.pop();
						flagStack.push(2);
						flagStack.push(1);
						nodeStack.push(p->leftChild);
						//continue; //这个continue是多余的。
					}
					else
					{
						cout<<p->data<<' ';
						nodeStack.pop();
						flagStack.pop();
						if(p->rightChild!=NULL)
						{
							nodeStack.push(p->rightChild);
							flagStack.push(1);
						}
					}
				}

			}
			else
			{
				cout<<"flag error"<<endl;
				return;
			}
		}

	}



