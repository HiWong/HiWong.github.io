---
layout: post
title: "排序算法之选择排序"
date: 2015-10-13 10:24:34 +0800
comments: true
categories: Algorithm
---
1.算法基本思想  

选择排序方法的基本思想是:第i趟排序是从线性表后面的n-i+1个数据元素中选择一个值最小的数据元素,并将其与它n-i+1个数据元素中的第1个数据元素交换位置.经过这样的n-1趟排序以后,初始的线性表成为了一个按值从小到大排列的线性表。  

2.流程图  

流程图如下所示<!--more-->：  

![select_sort01](http://7xn1yt.com1.z0.glb.clouddn.com/select_sort01.png)  

3.算法实现  

	public class SelectSort {

		private static void selectSort(int[]array)
		{
			int index,temp;
			for(int i=0;i<array.length-1;++i)
			{
				index=i;
				//find the minimum element from the (n-i+1) elements
				for(int j=i+1;j<array.length;++j)
				{
					if(array[j]<array[index])
					{
						index=j;
					}
				}
				if(index!=i)
				{
					temp=array[index];
					array[index]=array[i];
					array[i]=temp;
				}
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
	        selectSort(a);
	        System.out.println("\nafter sort,array is as below:");
	        PrintUtils.print(a);
		}
		
	}
	

输出结果如下：  

	before sort,array is as below:
	13 3 43 64 63 82 25 46 13 27 14 49 77 49 58 93 64 33 68 78 
	after sort,array is as below:
	3 13 13 14 25 27 33 43 46 49 49 58 63 64 64 68 77 78 82 93 	



