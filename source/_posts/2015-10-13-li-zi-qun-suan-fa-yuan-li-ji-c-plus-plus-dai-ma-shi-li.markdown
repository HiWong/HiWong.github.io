---
layout: post
title: "粒子群算法原理及C++代码实例"
date: 2015-10-13 03:38:56 +0800
comments: true
categories: Algorithm
---
粒子群优化算法(PSO)是一种进化计算技术(evolutionarycomputation)，1995年由Eberhart博士和kennedy博士提出，源于对鸟群捕食的行为研究。该算法最初是受到飞鸟集群活动的规律性启发，进而利用群体智能建立的一个简化模型。粒子群算法在对动物集群活动行为观察基础上，利用群体中的个体对信息的共享使整个群体的运动在问题求解空间中产生从无序到有序<!--more-->的演化过程，从而获得最优解。

PSO同遗传算法类似，是一种基于迭代的优化算法。系统初始化为一组随机解，通过迭代搜寻最优值。但是它没有遗传算法用的交叉(crossover)以及变异(mutation)，而是粒子在解空间追随最优的粒子进行搜索。同遗传算法比较，PSO的优势在于简单容易实现并且没有许多参数需要调整。粒子群算法较多地用于求解连续空间问题，离散空间问题的应用较少。  

在粒子群算法中，每个个体称为一个粒子，粒子是待求解问题的潜在可能的解，粒子种群由若干个粒子组成。粒子群算法的基本原理是粒子种群在搜索空间以一定的速度飞行, 每个粒子在搜索时，考虑自己搜索到的历史最优位置和种群内其他粒子的历史最优位置, 在此基础上进行位置的变化。  

下图为粒子群飞跃示意图。   

![particle01](http://7xn1yt.com1.z0.glb.clouddn.com/particle01.gif)
![particle02](http://7xn1yt.com1.z0.glb.clouddn.com/particle02.gif)  
群最好位置之间的距离来更新速度。然后粒子根据式子(2-3)飞向新的位置。粒子群算法流程图如下图所示：  
![particle03](http://7xn1yt.com1.z0.glb.clouddn.com/particle03.gif)
目前国内外很多学者从基本粒子群优化算法的各项参数和拓扑结构]着手进行研究，尝试通过参数的设置和结构的改变来改进算法的性能。惯性因子w起着平衡算法的全局搜索能力和局部搜索能力的作用。较小的惯性因子w(w< 0.8)可以使算法的局部搜索能力较强；较大的惯性因子w(w>1.2)使算法有较强的全局搜索能力。所以通常惯性因子w不是常数值，而是一个随着迭代渐变的数值。迭代初始惯性因子w的值较大，使算法有较强的全局搜索能力，在整个解空间内搜索可行解；惯性因子w随着迭代代数增加逐步减小，使算法在迭代后期局部搜索能力增强全局搜索能力减弱以加速收敛到已搜索到的最优解。惯性因子w的渐变公式通常如下：  

![particle04](http://7xn1yt.com1.z0.glb.clouddn.com/particle04.gif)
![particle05](http://7xn1yt.com1.z0.glb.clouddn.com/particle05.gif)  

Ratnaweera 等人研究了可变学习因子对PSO 算法性能的影响，定义c1和c2如
下：  
![particle06](http://7xn1yt.com1.z0.glb.clouddn.com/particle06.gif)

下面以一个在40*40的平面直角坐标区域内寻找距离点(10,20)最近的点为例进行说明。
首先是新建一个类。源文件为Coordinate.h  

	#ifndef COORDINATE_H
	#define COORDINATE_H
	class Coordinate
	{
	public:
		float x;
		float y;
		Coordinate();
		Coordinate(float x,float y);
	};
	#endif

下面是相应的Coordinate.cpp文件。  

	#include "stdafx.h"
	#include "Coordinate.h"
	using namespace std;
	Coordinate::Coordinate()
	{
		x=0.0f;
		y=.0f;
	}

	Coordinate::Coordinate(float x,float y)
	{
		this->x=x;
		this->y=y;
	}

然后是粒子类的定义。源文件为Particle.h。  

	#ifndef PARTICLE_H
	#define PARTICLE_H
	#include "Coordinate.h"
	#include <math.h>
	#include <iostream>
	using namespace std;
	class Particle
	{
		friend ostream& operator<<(ostream &output,const Particle &right);

	public:
		Particle(float x,float y);
	        void setP();
	       float getP()const;

		//void setPBest();  //pBest的设置在setP()中就完成了。
		Coordinate getPBest()const;
		
	   ///////////////////////这是第一种方法，即采用恒定的学习因子。但是实际上可变的学习因子c1,c2可以使种群更快地收敛。此处，将两个维度的速度设置放在同一个函数中。
	   void setV(Coordinate gBest,float w); //w为惯性因子。
	   float getVx()const;
	   float getVy()const;
	   
	   void setCoordinate();
	   float getX()const;
	   float getY()const;

	   void outputFile(char Dir[])const;
	private:
		Coordinate c;
		float p;  //p为适应度。
		Coordinate pBest;
		////////////////////二维的话就要有两个速度。
		float Vx;
		float Vy;
		static float Xmax,Xmin;
		static float Ymax,Ymin;
		static float Vxmax,Vxmin; //它们是用来对坐标和速度进行限制的，限制它只能在一定的范围内。
		static float Vymax,Vymin;
		static float c1,c2; //c1,c2是学习因子。
		//由于要对所有的对象进行比较之后才能得到群体最优，所以它还是不
		//static Coordinate gBest; //这个是群体最优.整个群体共享一份就行，所以将它设置成static，但是注意千万不要以为static都是在初始化后就不能修改的，static const才是那样。
		
	};

	#endif

下面是相应的Particle.cpp文件。  

	#include "stdafx.h"
	#include "Particle.h"
	#include <fstream>
	#include <iostream>
	using namespace std;
	float Particle::Xmax=30.0f;   //自己通过测试证明，通过限制速度和位置坐标取得了良好的效果，可以很轻易地达到1e-006的精度以上，甚至于能达到1e-009的精度。原因就在于整个群体中的粒子都能更快地集中到适应度高的位置，自然群体最优解就更好。
	float Particle::Xmin=0.0f;
	float Particle::Ymax=30.0f;
	float Particle::Ymin=0.0f;
	float Particle::Vxmax=Xmax-Xmin;  //通常设置Vmax=Xmax-Xmin;  
	float Particle::Vxmin=0-Vxmax;
	float Particle::Vymax=Ymax-Ymin;
	float Particle::Vymin=0-Vymax;

	float Particle::c1=2.0f;
	float Particle::c2=2.0f;


	Particle::Particle(float x,float y)
	{
		c.x=x;
		c.y=y;
		//p=100.0f; //先给它一个较大的适应度值（这里我们要得到的是一个较小的适应值）。
		p=pow(c.x-10.0f,2)+pow(c.y-20.0f,2);
		Vx=(Xmax-Xmin)/8.0f;  //////////////////////这里先采用第一种初始化方法，即给所有粒子一个相同的初始速度,为Vmax/8.0f,而Vmax=Xmax-Xmin=30-0=30;注意这个初始速度千万不能太大，自己测试发现它的大小对最终结果的精度影响也很大，有几个数量级。当然，也不能太小，自己测试发现在(Xmax-Xmin)/8.0f时可以得到比较高的精度。
		Vy=(Xmax-Xmin)/8.0f; 
		
		///初始时的pBest
		
		pBest.x=x;
		pBest.y=y;
		

	}

	void Particle::setP()   //这里既完成了适应度值p的设置，也完成了局部最优pBest的设置，所以不需要另外加一个setPBest()函数 。
	{
		float temp=pow(c.x-10.0f,2)+pow(c.y-20.0f,2);
		if(temp<p)
		{
			p=temp;
			//pBest.x=c.x;
			//pBest.y=c.y;
			pBest=c;
		}
	}

	float Particle::getP()const
	{
		return p;
	}

	Coordinate Particle::getPBest()const
	{
		return pBest;
	}

	void Particle::setV(Coordinate gBest,float w)  //w为惯性因子，
	{
		float r1,r2;
		r1=rand()/(float)RAND_MAX;
		r2=rand()/(float)RAND_MAX;
		Vx=w*Vx+c1*r1*(pBest.x-c.x)+c2*r2*(gBest.x-c.x);
		if(Vx>Vxmax)
			Vx=Vxmax;
		else if(Vx<Vxmin)
			Vx=Vxmin;
		Vy=w*Vy+c1*r1*(pBest.y-c.y)+c2*r2*(gBest.y-c.y);
		if(Vy>Vxmax)
			Vy=Vxmax;
		else if(Vy<Vxmin)
			Vy=Vxmin;
	}

	float Particle::getVx()const
	{
		return Vx;
	}
	float Particle::getVy()const
	{
		return Vy;
	}


	//////////////////////必须先进行setV()的操作然后才能进行这步，否则坐标加的就不是第k+1次的速度了。
	void Particle::setCoordinate()
	{
		c.x=c.x+Vx;
		if(c.x>Xmax)
			c.x=Xmax;
		else if(c.x<Xmin)
			c.x=Xmin;
		c.y=c.y+Vy;
		if(c.y>Ymax)
			c.y=Ymax;
		else if(c.y<Ymin)
			c.y=Ymin;


	}

	float Particle::getX()const
	{
		return c.x;
	}
	float Particle::getY()const
	{
		return c.y;
	}

	void Particle::outputFile(char Dir[])const
	{
	    ofstream out(Dir,ios::app);  //这是添加吧？
		
		out<<this->getX()<<" "<<this->getY()<<" "<<pBest.x<<" "<<pBest.y<<endl;
		out.close();

	}

	ostream& operator<<(ostream &output,const Particle &right)
	{
		output<<"Now the current coordinates is X:"<<right.getX()<<" Y:"<<right.getY()<<endl;
		output<<"And the pBest is X:"<<right.getPBest().x<<" Y:"<<right.getPBest().y<<endl;
		return output;
	}

下面是main文件。  

	#include "stdafx.h"
	#include "Particle.h"
	#include <stdio.h> //因为要用到sprintf函数。
	#include <time.h>
	#include <string>
	#include <math.h>
	#include <fstream>
	#include <iostream>
	using namespace std;

	int _tmain(int argc, _TCHAR* argv[])
	{
		Particle*p[40];
		float w;//实际上是wmax=1.2f;此处设置wmin=0.5;

		Particle*temp;
		float randX,randY;
		srand((int)time(NULL));
		for(int i=0;i<40;++i)
		{
			randX=(rand()/(float)RAND_MAX)*30.0f;
			randY=(rand()/(float)RAND_MAX)*30.0f;
			cout<<"randX="<<randX<<endl;
			cout<<"randY="<<randY<<endl;

			p[i]=new Particle(randX,randY);
			cout<<"The temp info is X:"<<p[i]->getX()<<" Y:"<<p[i]->getY()<<endl;
		
		}
		//至此，就完成了粒子群的初始化。
	  //////////////////////////////////////
		Coordinate gBest;  //全局最优解。
		int bestIndex=0;
		float bestP; //最好的适应度。
	    bestP=p[0]->getP();
		gBest=p[0]->getPBest();
		for(int i=1;i<40;++i)
		{
			if(p[i]->getP()<bestP)
			{
				bestP=p[i]->getP();
				gBest=p[i]->getPBest();
				bestIndex=i;
			}
		}

		/////////////////////////////////// 
		cout<<"Now the initial gBest is X:"<<gBest.x<<" Y:"<<gBest.y<<endl;
		cout<<"And the p[0] is X:"<<p[0]->getX()<<" Y:"<<p[0]->getY()<<endl;
		cout<<"And the p[39] is X:"<<p[39]->getX()<<" Y:"<<p[39]->getY()<<endl;
		cout<<"Now p[0].p="<<p[0]->getP()<<endl;
		////////////////////////////////至此，已经寻找到初始时的种群最优。
		char buf[20];
		for(int i=0;i<40;++i)
		{
			sprintf(buf,"coordinate%d.dat",i);
			ofstream out(buf,ios::out);
			out.close();
		}
		//////////////////////////这样做是为了每运行一次都重复添加。
	  for(int k=0;k<100;++k)   //k为迭代次数。
	  {
		 w=0.9f-(0.9f-0.4f)*k/99.0f;  //这个因子很重要，既不能太大也不能太小。一开始自己就是设置得太大了导致出错。自己通过计算发现采用可变的惯性因子可以使得到的结果的精确度高一个数量级(达到1e-006)，而如果采用恒定的惯性因子，则只能得到1e-005的精度。
		 //////////////////////////一开始wmax=1.0,wmin=0.6,可以达到1e-006的精度，以为很高了，但是实际上wmax=0.9f,wmin=0.4f进可以很轻易地达到1e-11的水平，而这已经接近float的精度极限。

		// w=0.85f;

		  for(int i=0;i<40;++i)
		  {
			  temp=p[i];
			  temp->setV(gBest,w);
			  temp->setCoordinate();
			  temp->setP();
			  sprintf(buf,"coordinate%d.dat",i);
			  temp->outputFile(buf);	 
		  }
		  bestP=p[0]->getP();
		  gBest=p[0]->getPBest();
		  for(int i=1;i<40;++i)
		  {
			  temp=p[i];
			  if(temp->getP()<bestP)
			  {
				  bestP=temp->getP();
				  gBest=temp->getPBest();
				  bestIndex=i;
			  }		  
			  /*
			  if((pow(gBest.x-10,2)+pow(gBest.y-20,2))<0.00001f)
			  {
				  cout<<"The gBest which is good enough has found!"<<endl;
				  cout<<"The index is "<<i<<endl;
				  cout<<"gBest is X:"<<gBest.x<<" Y:"<<gBest.y<<endl;
				   exit(0);
			  }
			  */
		  }
		  cout<<"Now gBest is X:"<<gBest.x<<" Y:"<<gBest.y<<" and the minP="<<p[bestIndex]->getP()<<endl;
		  cout<<"bestIndex="<<bestIndex<<endl;

	  }
		system("pause");
		return 0;
	}

运行后的结果为（由于运行结果较长，所以只是选取了其中几张有代表性的结果图片，但已经可以看出优化的过程。读者可自行运行查看结果)：
运行结果01：  

![particle_result01](http://7xn1yt.com1.z0.glb.clouddn.com/particle05.jpg)

运行结果02：  

![particle_result02](http://7xn1yt.com1.z0.glb.clouddn.com/particle06.jpg)

运行结果03：  

![particle_result03](http://7xn1yt.com1.z0.glb.clouddn.com/particle07.jpg)

运行结果04：  

![particle_result04](http://7xn1yt.com1.z0.glb.clouddn.com/particle08.jpg)

运行结果05：  

![particle_result05](http://7xn1yt.com1.z0.glb.clouddn.com/particle09.jpg)

**结论：由以上结果可明显地看出随着优化的进行，所找到的种群最优点的误差在迅速减小。从而证明了粒子群算法的有效性。**  

读者可尝试改变其中的一些参数，如惯性因子、学习因子等，观察其对寻优过程的影响。







