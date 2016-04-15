---
layout: post
title: "排序算法之归并排序"
date: 2015-10-20 12:30:48 +0800
comments: true
categories: algorithm
---

1.思想  

归并是指将两个或者多个按值有序序列合并成为一个按值排列的有序序列的过程。若将两个按值有序序列合并成一个按值有序序列，则称之为二路归并。同理，有三路归并、四路归并等。其中二路归并最为简单，也最常用<!--more-->。

2.算法实现  

	import java.util.Random;

	public class MergeSort {

		private static void mergeArray(int array[],int first,int mid,int last,int[]temp)
		{
			int i=first,j=mid+1;
			int m=mid,n=last;
			int k=0;
			
			while(i<=m&&j<=n)
			{
				if(array[i]<=array[j])
				{
					temp[k++]=array[i++];
				}
				else
				{
					temp[k++]=array[j++];
				}
			}
			
			while(i<=m)
			{
				temp[k++]=array[i++];
			}
			
			while(j<=n)
			{
				temp[k++]=array[j++];
			}
			
			for(i=0;i<k;++i)
			{
				array[first+i]=temp[i];
			}
		}
		
		private static void merSort(int array[],int first,int last,int temp[])
		{
			if(first<last)
			{
				int mid=(first+last)/2;
				merSort(array,first,mid,temp);
				merSort(array,mid+1,last,temp);
				mergeArray(array,first,mid,last,temp);
			}
		}
		
		public static void mergeSort(int array[])
		{
			if(null==array||array.length<1)
			{
				return;
			}
			int[]temp=new int[array.length];
			merSort(array,0,array.length-1,temp);
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
			mergeSort(a);
			System.out.println("after sort,array is as below:");
			PrintUtils.print(a);
		}
		
		
	}

某一次的输出结果如下:  

	before sort,array is as below:
	20 64 57 29 52 90 90 35 84 12 48 95 22 45 28 51 48 41 80 9  
	after sort,array is as below:
	9 12 20 22 28 29 35 41 45 48 48 51 52 57 64 80 84 90 90 95  

3.算法分析  

二路归并排序算法的时间复杂度等于归并趟数和每一趟归并的时间复杂度的乘积，由于每一趟归并的时间复杂度为O(n),故总的时间复杂度为O(nlog2n);  

另外，由于二路归并算法中每次都是在相邻的数据中进行操作，故其是稳定的排序算法。  





