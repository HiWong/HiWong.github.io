---
layout: post
title: "智能搜索算法--从A*算法开始说起"
date: 2015-10-13 03:13:07 +0800
comments: true
categories: Algorithm
---

首先解释一下状态空间搜索。状态空间搜索法就是将问题求解过程表现为从 初始状态到目标状态寻找这个路径的过程。    
通俗点说，就是在解一个问题时，找到一条解题的过程可以从 求解的开始到问题的结果。由于求解问题的过程中分枝有很多，主要是求解过程中求 解条件的不确定性，不完备性造成的，使得求解的路径很多这就构成了一个图，我们说这个图就是状态空 间。问题的求解实际上就是在这个图中找到一条路径可以从开始到结果。这个寻找的过程<!--more-->就是状态空间搜索。  
常用的状态空间搜索有深度优先和广度优先。广度优先是从初始状态一层一层向下找，直到找到目标 为止。深度优先是按照一定的顺序前查找完一个分支，再查找另一个分支，以至找到目标为止。  
显然，只要目标点存在，广度优先(BFS)或深度优先(DFS)搜索算法都一定能够搜索到目标点。但是，当结点数很多时，BFS和DFS的空间复杂度和时间复杂度都变得不可接受。而它们的效率不高的一个重要原因在于没有利用任何已有的信息。而A*算法由于利用了启发信息，因而获得了较高的搜索效率。

我们先下个定义，如果一个估价函数可以找出最短的路径，我们称之为可采纳性。A*算法是一个可采纳的最好优先算法。A*算法的估价函数可表示为：  

	f'(n) = g'(n) + h'(n)  

这里，f'(n)是估价函数，g'(n)是起点到节点n的最短路径值，h'(n)是n到目标的最短路经的启发值。由于这个f'(n)其实是无法预先知道的，所以我们用前面的估价函数f(n)做近似。g(n)代替g'(n)，但 g(n)>=g'(n)才可（大多数情况下都是满足的，可以不用考虑），h(n)代替h'(n)，但h(n)<=h'(n)才可（这一点特别的重要）。可以证明应用这样的估价函数是可以找到最短的，也就是可采纳的。我们说应用这种估价函数的最好优先算法就是A*算法。
因而实际在算法中采用的估价函数为：f(n)=g(n)+h(n)，其中g(n)为起始点到当前点的实际代价，而h(n)为当前点到目标点的启发信息，目前，在二维平面内常用的A*算法的启发函数h(n)有曼哈顿距离、对角线距离、欧几里德距离。也有学者采用所谓“折距”作为启发函数。但是，采用对角线距离的话会太保守，搜索效率不高，而采用曼哈顿距离或者“折距”，虽然可以提高效率，但是都不能保证满足可接纳性条件，因而不宜采用。目前用于二维平面内的A*算法普遍采用欧几里德距离。  

举一个例子，其实广度优先算法(BFS)就是A*算法的特例。其中g(n)是节点所在的层数，h(n)=0，这种h(n)肯定小于h'(n)，所以由前述可知广度优先算法是可采纳的。实际也是。当然它是一种最差的A*算法。  

通过上面的介绍，我们可以确定A*算法的流程如下(注：出自wikipedia)：  

	function A*(start,goal)
    closedset := the empty set    // The set of nodes already evaluated.
    openset := {start}    // The set of tentative nodes to be evaluated, initially containing the start node
    came_from := the empty map    // The map of navigated nodes.

    g_score[start] := 0    // Cost from start along best known path.
    // Estimated total cost from start to goal through y.
    f_score[start] := g_score[start] + heuristic_cost_estimate(start, goal)
     
    while openset is not empty
        current := the node in openset having the lowest f_score[] value
        if current = goal
            return reconstruct_path(came_from, goal)
         
        remove current from openset
        add current to closedset
        for each neighbor in neighbor_nodes(current)
            tentative_g_score := g_score[current] + dist_between(current,neighbor)
            tentative_f_score := tentative_g_score + heuristic_cost_estimate(neighbor, goal)
            if neighbor in closedset and tentative_f_score >= f_score[neighbor]
                    continue

            if neighbor not in openset or tentative_f_score < f_score[neighbor] 
                came_from[neighbor] := current
                g_score[neighbor] := tentative_g_score
                f_score[neighbor] := tentative_f_score
                if neighbor not in openset
                    add neighbor to openset

    return failure


    function reconstruct_path(came_from, current_node)
    if current_node in came_from
        p := reconstruct_path(came_from, came_from[current_node])
        return (p + current_node)
    else
        return current_node

这里必须吐槽一下度娘的A*算法流程是不严谨的（如果想要理解A*算法的同学，建议还是看英文版维基上的。当然，最好就是借一本人工智能方面的书认真看一下，可以理解得更透彻。
当然，上面只是最原始的A*算法。如果要获得较高的效率，还需要对其进行进一步的改进，其中目前大多数学者优化的重点都放在启发信息的改进和从Open表中更快地获取代价最小的结点。  
其中，关于启发信息的讨论在前面已经提到过，而且由于它一般要与具体的学科问题相结合才有意义，此处不再赘述。
下面主要讲解如何能更快地从Open表中获取代价最小的节点。显然有两个途径：第一，根据具体的问题，预先将不符合某些约束（比如在航迹规划时，将不符合动力学约束的节点去除。或者说，我们只在符合动力学约束的空间内选择邻节点放入表中）的子节点剪除掉。
第二种方法就是在邻节点数无法改变的情况下，选择一种好的排序算法就至关重要了。可以选择快速排序等，目前较常用的一种高效排序算法是维持一个称为“最小堆”的数据结构，它是一种二叉树结构，其特征是对于任一结点，它的左孩子（如果有的话）和右孩子必定大于它。显然，这种排序算法的时间复杂度为O(log2N)。排好序后，每次只要将根结点移除即可。如下图所示。  

![A_star01](http://7xn1yt.com1.z0.glb.clouddn.com/A_star01.png)

A*算法的理论介绍基本就这些。下面是自己写的一个利用A*算法寻找二维平面内任意两点间的最短路径，希望能够抛砖引玉。  

由于在节点的扩展过程中需要实现较多的操作，而这些操作与节点密切相关。因而，为了实现更好的封装，此处没有使用C++中的STL，而是自己实现了链表结构。  

首先是LisNode的定义：


	#ifndef LISTNODE_H
	#define LISTNODE_H
	#include "Point.h"
	#include <vector>
	using namespace std;
	class List;  //前向引用声明。
	class ListNode
	{
		friend class List;
	public:
		ListNode(const Point&p);
		//////////////////只要有一个节点（比如起始节点）调用了下面这两个函数就行。
		//static void setStartPoint(const Point&p);
		static void setEndPoint(const Point&p);
		static void setPlanAreaWidth(float width);
		static void setPlanAreaHeight(float height);
		void setFather(ListNode*listNode);
		void setNext(ListNode*listNode);
		bool setg(); //这个是要知道它前一个节点的信息的。它是从起点到这个点所花费的实际代价。
		void setg(float g1);//这个是为第一个点而设计的函数，直接设定它的g值。
		void seth(); //这个是需要知道终点信息的。它是从现在这个点到目标点的代价。
		void setf();
		float getg()const;
		float getf()const;
		Point getPos()const;
		ListNode* getPrev()const;
		void getNeightborNodes(vector<ListNode*>&v); //得到它的邻近节点。	
		bool isCloseEnough(float r)const;//这个函数用于判断它是否离终点足够近。
	private:
		Point pos;
		float g,h,f;
		ListNode*prev;  //其实，为了在prev和next这两个方面为了更好地模仿结构体，prev和next应该设置为public的。
		ListNode*next;
		static float planAreaWidth;
		static float planAreaHeight;
		static Point endPoint;

	};

	#endif

下面是ListNode的实现部分，即listNode.cpp文件：  

	#include "stdafx.h"
	#include "ListNode.h"
	#include <math.h>
	#include <iostream>
	using namespace std;
	Point p(80.0f,80.0f);
	Point ListNode::endPoint=p;
	float ListNode::planAreaWidth=100.0f;
	float ListNode::planAreaHeight=100.0f;
	ListNode::ListNode(const Point&p)
	{
		pos.x=p.x;
		pos.y=p.y;
		prev=NULL;
		next=NULL;
	}
	void ListNode::setEndPoint(const Point&p)
	{
		endPoint.x=p.x;
		endPoint.y=p.y;
	}
	void ListNode::setPlanAreaWidth(float widhth)
	{
		planAreaWidth=widhth;
	}
	void ListNode::setPlanAreaHeight(float height)
	{
		planAreaHeight=height;
	}
	void ListNode::setFather(ListNode*listNode)
	{
		prev=listNode;
	}
	void ListNode::setNext(ListNode*listNode)
	{
		next=listNode;
	}
	bool ListNode::setg()
	{
		if(prev==NULL)   //其实可以把startPoint利用上，然后求解起始节点的g值的。
			return false;
		else
		{
			///////////////////////////////它与前一个节点的方向关系有5种可能。当然，如果不想写if语句，直接用距离公式也可以，只不过用距离公式会快一些。
			if(pos.x-prev->pos.x==0.0f) //这里包含两种情况。
			{
				g=prev->g+1.0f; //实际上应该是delta，只不过这里风格间距为1.0f;
				return true;
			}
			else if(pos.x-prev->pos.x>0.0f)
			{
				if(pos.y-prev->pos.y==0.0f)
				{
					g=prev->g+1.0f;
					return true;
				}
				else
				{
					//g=prev->g+1.4142136f;
					g=prev->g+1.414f;
					//cout<<"Now g="<<g<<endl; //This is just for test.
					return true;
				}
			}	
		}
		return false;

	}
	void ListNode::setg(float g1)
	{
		g=g1;
	}
	void ListNode::seth()
	{
		h=sqrt(pow(pos.x-endPoint.x,2)+pow(pos.y-endPoint.y,2));
	}
	void ListNode::setf()
	{
		f=g+h;
		cout<<"("<<pos.x<<","<<pos.y<<") "<<" f="<<f<<endl;
	}
	float ListNode::getg()const
	{
		return g;
	}

	float ListNode::getf()const
	{
		return f;
	}

	Point ListNode::getPos()const
	{
		return pos;
	}

	ListNode* ListNode::getPrev()const
	{
		return this->prev;
	}
	///////////////////////////////这个函数还不敢肯定写对了，要测试一下。另外，有一点自己还没有区分开来：那就是路线终点和规划区域终点是两回事，实际上这里要的是规划区域的终点而不是路线的终点!
	void ListNode::getNeightborNodes(vector<ListNode*>&v) //注意：要传递引用才能改变实参的值。
	{
	     ///////////////////////////////////注意：由于后面它算每个节点的估价值，而要求估值就要知道它的父节点，所以将所有邻节点的父节点都                  //设置成当前节点。
	     int countNum=0; //countNum is just for test.
	    if(pos.x+1.0f<=endPoint.x)
		{
			Point tempPoint(pos.x+1.0f,pos.y);
			ListNode*tempPtr=new ListNode(tempPoint); 
			tempPtr->prev=this; //This step is very important.

			v.push_back(tempPtr);//加入第三个方向的节点。
			countNum++;
			//if(pos.y+1.0f<=endPoint.y)
			if(pos.y+1.0f<=planAreaHeight)  //注意：由于不能后退，所以x的限制条件仍然是endPoint.x，但是
			{
				tempPoint.y=pos.y+1.0f;
				tempPtr=new ListNode(tempPoint);  
				tempPtr->prev=this;

				v.push_back(tempPtr);   //加入第2个方向的节点。
				countNum++;
			}
			if(pos.y-1.0f>=0.0f)
			{
				tempPoint.y=pos.y-1.0f;
				tempPtr=new ListNode(tempPoint);
				tempPtr->prev=this;

				v.push_back(tempPtr); //加入第四个方向的节点
				countNum++;
			}
		}
		////////////////////////////////////////
		//if(pos.y+1.0f<=endPoint.y)  
		if(pos.y+1.0f<=planAreaHeight)
		{
			Point tempPoint(pos.x,pos.y+1.0f);
			ListNode*tempPtr=new ListNode(tempPoint);
			tempPtr->prev=this;

			v.push_back(tempPtr);  //加入第一个方向（向上）的节点
			countNum++;
		}
		if(pos.y-1.0f>=0.0f)
		{
			Point tempPoint(pos.x,pos.y-1.0f);
			ListNode*tempPtr=new ListNode(tempPoint);
			tempPtr->prev=this;
			v.push_back(tempPtr);  //加入第五个方向(向下)的节点
			countNum++;		
		}
	    cout<<"The node whose coordinates is ("<<pos.x<<","<<pos.y<<") has "<<countNum<<" neighbor points."<<endl; //This is for test.

	}
	bool ListNode::isCloseEnough(float r)const  //这个函数用于判断它是否与终点足够接近。或者说是否在终点的大小为r的邻域内。
	{
		float xDistance=abs(pos.x-endPoint.x);
		float yDistance=abs(pos.y-endPoint.y);
		if((xDistance<=r)&&(yDistance<=r))
			return true;

		return false;
	}

下面是链表的定义：  

	#ifndef LIST_H
	#define LIST_H
	#include "ListNode.h"

	class List
	{
	public:
		List();
		~List();
		void insertAtBack(ListNode*pNode);
	    void insertProperly(ListNode*pNode);
		bool isEmpty()const;
		bool isInList(ListNode*pNode,float&g)const;
		ListNode*getFromList(const Point&p);//从链表中取出指定坐标的节点，然后再改变它的g和f值，注意不能改变它的prev，然后再插入Open链表中。

		ListNode*removeBestNode();//将f值最小的节点从链表中脱离出来，注意脱离只要解除与前后节点的关系就行，而不需要删除它，因为在后面还要用到它。	
		void exportList(char dir[])const; //This is for test.
	private:
		ListNode*firstPtr;
		ListNode*lastPtr;	
	};
	#endif

下面是其实现：  

	#include "stdafx.h"
	#include "List.h"
	#include <fstream>
	using namespace std;
	List::List()
	{
		firstPtr=NULL;
		lastPtr=NULL;
	}
	List::~List()
	{
		ListNode*tempNode01=firstPtr; //注意：头结点也存放数据。
		ListNode*tempNode02=tempNode01;
		while(tempNode01->next!=NULL)
		{
			tempNode02=tempNode01;
			tempNode01=tempNode02->next;
			delete tempNode02;
			tempNode02=NULL;
		}
	}
	void List::insertAtBack(ListNode*pNode)
	{
		if(isEmpty())
		{
			firstPtr=lastPtr=pNode;
			lastPtr->next=NULL;
		}
		else
		{
			lastPtr->next=pNode;  //让原来的最后一个节点的next指向pNode;
			lastPtr=pNode;       //新插入的节点成为新的最后一个节点。	
		}
	}
	void List::insertProperly(ListNode*pNode)
	{
		//////////////////////////只要找到第一个f值比它的f值大(>=)的节点，然后插入在那个节点的前面就行。但是由于现在自己打算不在List中对ListNode中的Prev进行设置，所以这种方法操作起来不容易。还是先用原来那种方法。
		/*
		ListNode*tempNode=firstPtr;
		while((tempNode->f<pNode->f)&&tempNode->next!=NULL)
		{
			tempNode=tempNode->next;
		}
		*/
		if(isEmpty())
			insertAtBack(pNode);
		else
		{
			if(pNode->f<=firstPtr->f)
			{
				pNode->next=firstPtr;
				firstPtr=pNode;
			}
			else if(pNode->f>=lastPtr->f)
			{
				lastPtr->next=pNode;
				lastPtr=pNode;
				lastPtr->next=NULL; 		}
			else
			{
				ListNode*tempPtr=firstPtr;
				while(tempPtr->next!=NULL)
				{
					if((tempPtr->f<=pNode->f)&&(tempPtr->next->f>=pNode->f))
					{
						pNode->next=tempPtr->next;
						tempPtr->next=pNode;
						break;
					}
					tempPtr=tempPtr->next;
				}
			}
		}
	}
	bool List::isEmpty()const
	{
		return firstPtr==0;
	}
	bool List::isInList(ListNode*pNode,float &g)const //如果这个点已经在OpenList中的话，将该点的g值赋给g（因为是同一点，h值必然相同，所以比较f和比较g是一样的。）
	{
		////////////////////////////////////只要判断这个点的坐标在OpenList或者ClosedList中能不能找到相同的就行。
		if(!isEmpty())//首先要不为空才可以继续下面的查找判断
		{
			ListNode*tempPtr=firstPtr;
			while(tempPtr->next!=NULL)   //这种查找方法用的是最古老的查找方法，其实对于排好了序的链表，可以用二分查找的，以后要改进这个。
			{
				if((tempPtr->pos.x==pNode->pos.x)&&(tempPtr->pos.y==pNode->pos.y))
				{
					g=tempPtr->g;
					return true;
				}
				tempPtr=tempPtr->next;  //千万别忘了这句。
			}
	   }
		return false;
	}
	ListNode* List::getFromList(const Point&p)  //取出坐标点与p相同的节点并从链表中移出。注意：一定要保证这个坐标点在链表中的基础上才能使用这个函数，否则会出错。
	{
		ListNode*tempPtr=firstPtr;
		int countNum=0;
		while(tempPtr->next!=NULL)
		{
			countNum++;
			if((tempPtr->pos.x==p.x)&&(tempPtr->pos.y==p.y))
			{
			///////////////////////要先解除链接关系。
				break;		
			}
			tempPtr=tempPtr->next;
		}
		ListNode*tempPtr02=firstPtr;
		for(int i=0;i<countNum-1;++i)//要找到那个等坐标节点的前一个节点以解除链接关系。
		{
			tempPtr02=tempPtr02->next;
		}
		tempPtr02->next=tempPtr->next;
		tempPtr->next=NULL; //为了彻底解除链接关系，将要移出的这个节点的next设置为NULL;
		return tempPtr; //不需要用break，因为函数遇到return就会返回。
	}
	ListNode* List::removeBestNode()
	{
		ListNode*tempPtr=firstPtr;
		firstPtr=tempPtr->next;
		tempPtr->next=NULL;
		return tempPtr;


	}
	void List::exportList(char dir[])const
	{
		if(firstPtr!=NULL)
		{
			ListNode*tempPtr=firstPtr;
			ofstream outData;
			outData.open(dir);
			while(tempPtr->next!=NULL)
			{
				outData<<"("<<tempPtr->pos.x<<","<<tempPtr->pos.y<<") "<<"f="<<tempPtr->f<<" g="<<tempPtr->g<<" h="<<tempPtr->h<<endl;
				tempPtr=tempPtr->next;
			}
			outData.close();


		}
		
	}

最后是main函数，它与上面介绍的A*算法几乎一样，只是自己作了一点小的改进：  

	#include "stdafx.h"
	#include "List.h"
	#include <vector>
	#include <iostream>
	#include <string>
	using namespace std;


	int _tmain(int argc, _TCHAR* argv[])
	{
		Point startPoint(10.0f,60.0f);
		Point endPoint(100.0f,59.0f);   //现在这个算法有一个缺陷就是只能计算终点横纵坐标相等的情形，这点必须改进。
		float width=200.0f;
		float height=200.0f;
		ListNode*startNode=new ListNode(startPoint);
		////////////////////////它是第一个节点，所以要它来调用static函数以设置类的static变量（起点和终点)
		//startNode->setStartPoint(startPoint);
		startNode->setEndPoint(endPoint);
		startNode->setPlanAreaWidth(width);
		startNode->setPlanAreaHeight(height);
		/////////////////////
		startNode->setg(0.0f);
		startNode->seth();
		startNode->setf();
		//////////////////////////////
		List OpenList;//注意：要写成List OpenList而不是List OpenList()
		List ClosedList;
		/////////////////////////////
		OpenList.insertAtBack(startNode);
		////////////////////////
		int countNum=0;
	       while(!OpenList.isEmpty())
		{
			countNum++; //输出文件号。
			ListNode*currentNode=OpenList.removeBestNode();
			if(currentNode->isCloseEnough(0.1f))  //说明它离终点足够近，可以停止规划了。
			{
				cout<<"Congratulations! A Star Algorithm has succeeded!"<<endl;
				cout<<"And the path info is as follows:"<<endl;
				ListNode*tempPtr=currentNode;
				while(tempPtr!=NULL) //注意要写成tempPtr!=NULL而不是tempPtr->getPrev()!否则无法遍历到路径的第一个节点。
				{
					float x=tempPtr->getPos().x;
					float y=tempPtr->getPos().y;
					cout<<"("<<x<<","<<y<<")"<<endl;
					tempPtr=tempPtr->getPrev();
				}

				//cout<<"("<<startPoint.x<<","<<startPoint.y<<")"<<endl;

				break;
			}
			////////////////////////////////////////////////////////////////////////////////////
			vector<ListNode*>v;
			currentNode->getNeightborNodes(v);
			vector<ListNode*>::iterator iter;

			/*
		    //////////////////////////////////为了测试工作情况，把OpenList中的所有元素输出到文件中。By the way,this is for tes.
			char dir[20];
			sprintf(dir,"OpenList%d.txt",countNum);
			OpenList.exportList(dir);*/

			int i=0;
			for(iter=v.begin();iter!=v.end();++iter)
			{
				v[i]->setg();
				v[i]->seth();
				v[i]->setf();
				/////////////////////////////////在求得g,h,f后，要取消父子连接吗？要!
				v[i]->setFather(NULL);
				float tempg=-1.0f;
				if(OpenList.isInList(v[i],tempg))
				{
					if(v[i]->getg()<tempg)
					{
						v[i]->setFather(currentNode);
						ListNode*tempPtr=OpenList.getFromList(v[i]->getPos());	
						//OpenList.insertProperly(tempPtr);
						OpenList.insertProperly(v[i]);
					}
					else
					{
						delete v[i];  //这样做会不会引起v的变化，还要先测试一下。
						v[i]=NULL;  //删除没有插入OpenList的邻节点，以防内存泄漏。

					}
					

				}
				///////////////////////////////////////////////////////////////////////
				//float tempg=-1.0f; //上面已经定义过了
				else if(ClosedList.isInList(v[i],tempg))   //初步怀疑是因为下面的这个操作而导致出错。
				{
					delete v[i];
					v[i]=NULL; 
					++i; //由于此时不会执行到下面，所以要在这里让i自加!
					continue;
				}
				//////////////////////////////////////////////////////////////////////
				//if((!OpenList.isInList(v[i],tempg))&&(!ClosedList.isInList(v[i],tempg)))
				else  //一个节点只有这三种可能：要么在OpenList中，要么在ClosedList中，要么都不在。
				{
					v[i]->setFather(currentNode);
					OpenList.insertProperly(v[i]);
				}

			
				++i;

			}

			v.clear();
			ClosedList.insertAtBack(currentNode);

		}

		cout<<"End of pathplaning!"<<endl;
		system("pause");
		return 0;
	}

以上。