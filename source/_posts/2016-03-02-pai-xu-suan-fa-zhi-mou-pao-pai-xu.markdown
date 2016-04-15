---
layout: post
title: "排序算法之冒泡排序"
date: 2015-10-16 22:50:59 +0800
comments: true
categories: algorithm
---

1.思想  
冒泡排序的思想：先将序列中的第1个元素与第2个元素进行比较，若前者大于后者，则两者交换位置，否则不交换。然后将第2个元素与第3个元素比较，若前者大于后者，两者交换位置，否则不交换;依此类推，真到第n-1个元素与第n个元素进行比较为止。经过如此一趟排序，使得n个元素中值最大的元素被安置到了最后。此后，再对前面(n-1)个元素进行同样的操作;之后对是第(n-2)个元素...一直到某一趟排序过程中不出现元素交换位置的动作，排序结束。  

2.流程图  

流程图如下所示<!--more-->：  

![BubbleSort](http://7xn1yt.com1.z0.glb.clouddn.com/BubbleSort.png)

3.算法实现  


	import java.util.Random;

	public class BubbleSort {

		public static void bubbleSort(int[]array)
		{
			if(null==array||array.length<1)
			{
				return;
			}
			
			int j,temp;
			boolean swapFlag=false;
			for(int i=0;i<array.length;++i)
			{
				swapFlag=false;
				for(j=0;j<array.length-i-1;++j)
				{
					if(array[j]>array[j+1])
					{
						temp=array[j+1];
						array[j+1]=array[j];
						array[j]=temp;
						swapFlag=true;
					}
				}
				
				if(!swapFlag)
				{
					break;
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
			bubbleSort(a);
			System.out.println("after sort,array is as below:");
			PrintUtils.print(a);
		}
		
	}


4.算法分析  

最好的情况下，初始序列已经是递增序列，则只需经过n-1次元素比较，此时的时间复杂度为O(n);最坏情况是当参加排序的初始序列为逆序，或者最小的元素在最后时，则需要进行n(n-1)/2次元素之间的比较，此时时间复杂度为O(n^2);  

所以冒泡排序算法的时间复杂度为O(n^2);  

另外，由于元素交换是在相邻元素之间进行的，不会改变相同值的元素间的相对位置，因此，冒泡排序算法是一种稳定排序算法。  


