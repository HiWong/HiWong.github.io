---
layout: post
title: "排序算法之希尔排序"
date: 2015-10-18 23:40:14 +0800
comments: true
categories: algorithm
---

1.思想  

希尔排序(Shell's sort)又称缩小增量排序法，是由Shell在1959年提出的、对直接插入排序法的一种改进。  

其核心思想是：首先确定一个元素间隔数gap,然后将参加排序的序列按此间隔数第1个元素开始分成若干个子序列，即分别将所有位置相隔为gap的元素视为一个子序列，在各个字序列中采用某种排序方法进行排序（例如，可用冒泡排序方法),然后减小间隔数，并重新将整个序列按新的间隔数分成若干个子序列，再分别对各个子序列进行排序，如此下去，直到间隔数gap=1.  

示意图如下所示<!--more-->：  

![ShellSort](http://7xn1yt.com1.z0.glb.clouddn.com/ShellSort.png)

2.算法实现  

	import java.util.Random;

	public class ShellSort {

		public static void shellSort(int[]array,int initGap)
		{
			if(null==array||array.length<1)
			{
				return;
			}
			int i,temp,gap=initGap;
			boolean swapFlag=false;
			while(gap>1)
			{
				gap=gap/2;
				do{
					swapFlag=false;
					for(i=0;i<array.length-gap;++i)
					{
						if(array[i]>array[i+gap])
						{
							temp=array[i];
							array[i]=array[i+gap];
							array[i+gap]=temp;
							swapFlag=true;
						}
					}
					
				}while(swapFlag);
			}
		}
		

		public static void main(String[]args)
		{
			
			int[]a=new int[20];
			for(int i=0;i<20;++i)
			{
				a[i]=new Random().nextInt(100);
			}
			
			System.out.println("before sort,array is as below:");
			PrintUtils.print(a);
			shellSort(a,5);
			System.out.println("after sort,array is as below:");
			PrintUtils.print(a);
		}
		
	}

3.算法分析  

显然，希尔排序算法的时间复杂度与gap有关。对于如何选择gap的值，目前也没有很好地解决，常用的做法是每次取上一次的一半。一般情况下，认为希尔排序算法的时间复杂度在O(nlog2n)和O(n^2)之间。有人做过测试，发现其时间复杂度比O(nlog2n)稍大，接近于O(n^1.23).  

另外，由于希尔排序算法中的元素交换并不是针对相邻元素，因而相同值的元素的相对位置可能会改变。因此，希尔排序算法是一种不稳定的排序算法。  




