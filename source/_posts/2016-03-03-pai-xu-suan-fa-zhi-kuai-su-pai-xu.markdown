---
layout: post
title: "排序算法之快速排序"
date: 2015-10-19 09:24:16 +0800
comments: true
categories: algorithm
---
1.思想  

快速排序是C.R.A.Hoare于1962年提出的一种划分交换排序。它采用了一种分治的策略，通常称其为分治法。它是一种对冒泡排序法的改进。该方法的基本思想是：  

+ 先从序列中取出一个数作为基数(这个基数可任意选择)
+ 分区过程，将比这个数大的数全放到它的右边，小于或等于它的数全放到它的左边
+ 再对左右区间重复第二步，直到各区间只有一个数

示意图如下所示<!--more-->：  

![QuickSort2](http://7xn1yt.com1.z0.glb.clouddn.com/QuickSort2.png)

2.算法实现  

	import java.util.Random;

	public class QuickSort {

		public static void quickSort(int[]array,int left,int right)
		{
			//这个判断条件非常重要，否则会Stackoverflow
			if(left<right)
			{
				int base=array[left],l=left,r=right;
				while(l<r)
				{
					while(r>=l)
					{
						if(array[r]<base)
						{
							array[l]=array[r];
							break;
						}
						r--;
					}
					
					while(l<r)
					{
						if(array[l]>base)
						{
							array[r]=array[l];
							break;
						}
						l++;
					}
				}
				
				array[l]=base;
				
				quickSort(array,left,l-1);
				quickSort(array,l+1,right);
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
			quickSort(a,0,19);
			System.out.println("after sort,array is as below:");
			PrintUtils.print(a);
		}
		
		
	}

某一次的输出结果如下:  

	before sort,array is as below:
	76 1 19 34 6 26 16 11 10 44 92 97 89 1 54 38 87 31 69 45  
	after sort,array is as below:
	1 1 6 10 11 16 19 26 31 34 38 44 45 54 69 76 87 89 92 97  

3.算法分析  

在快速排序方法中，元素之间的比较和交换从两端向中间进行，值较大的元素一次就能交换到后面的某一个位置上，而值较小的元素也能一次交换到前面的某一个位置中，元素移动的间隔距离较大，因而总的比较次数较少。  

在最坏情况下，快速排序也需要O(n^2)的时间复杂度，但是它的平均时间复杂度较少，为O(nlog2n);  



