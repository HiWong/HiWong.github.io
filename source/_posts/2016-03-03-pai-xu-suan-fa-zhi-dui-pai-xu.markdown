---
layout: post
title: "排序算法之堆排序"
date: 2015-10-21 16:02:39 +0800
comments: true
categories: algorithm
---

1.思想  
1.1 二叉堆  
二叉堆满足两个特性：  
1)父结点的键值总是大于或等于(小于或等于)任何一个子节点的键值;  
2)每个结点的左子树和右子树都是一个二叉堆  

当父结点的键值总是大于或等于任何一个子结点的键值时为最大堆;反之为最小堆。  
下图是一个最小堆的示意图<!--more-->：  

![HeapSort](http://7xn1yt.com1.z0.glb.clouddn.com/HeapSort.png)

1.2 堆的调整  
以最小堆为例，从根结点开始，先在左右孩子结点中找最小的，如果父结点比这个最小的还小说明不需要调整了，反之将父结点和它交换后再对后面的结点进行相同的操作。  

1.3 堆的建立  
将堆中所有数据重新排序，使其成为最小堆。在实现堆调整的基础上，很容易就能实现推的创建。  

1.4 堆排序  

移除位于第一个数据的根结点，并做最小堆的递归运算。  


2.算法实现  

2.1 最小堆的实现  


	import java.util.Random;

	public class HeapSort {

		private static void swap(int[]array,int i,int j)
		{
			int temp=array[i];
			array[i]=array[j];
			array[j]=temp;
		}	
		
		public static void minHeapFixup(int array[],int i)
		{
			int j,temp;
			temp=array[i];
			j=(i-1)/2;
			while(j>=0&&i!=0)
			{
				if(array[j]<=temp)
				{
					break;
				}
				array[i]=array[j];
				i=j;
				j=(i-1)/2;
			}
			array[i]=temp;
			
		}
		
		public static void minHeapAdjust(int[]array,int i,int n)
		{
			int j,temp;
			temp=array[i];
			j=2*i+1;
			while(j<n)
			{
				//在左右孩子中找最小的
				if(j+1<n&&array[j+1]<array[j])
				{
					j++;
				}
				
				if(array[j]>=temp)
				{
					break;
				}
				
				//将较小的子节点往上移动，替换它的父节点
				array[i]=array[j];
				i=j;
				j=2*i+1;
			}
			array[i]=temp;
		}
		
		private static void buildMinHeap(int[]array)
		{
			for(int i=array.length/2-1;i>=0;--i)
			{
				minHeapAdjust(array,i,array.length);
			}
		}
		
		public static void minHeapSort(int[]array)
		{
			
			for(int i=array.length-1;i>=1;--i)
			{
				swap(array,i,0);
				minHeapAdjust(array,0,i);	
				//minHeapFixup(array,i);
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
			//attention!we need to build heap first! And then we can start to sort!
			buildMinHeap(a);
			minHeapSort(a);
			System.out.println("after sort,array is as below:");
			PrintUtils.print(a);
		}	
	}

某一次运行的结果如下：  

	before sort,array is as below:
	71 40 36 71 29 93 75 95 20 89 44 16 55 31 32 88 41 7 17 25  
	after sort,array is as below:
	95 93 89 88 75 71 71 55 44 41 40 36 32 31 29 25 20 17 16 7 

2.2 最大堆的实现  


	import java.util.Random;

	public class HeapSort {

		private static void swap(int[]array,int i,int j)
		{
			int temp=array[i];
			array[i]=array[j];
			array[j]=temp;
		}	
		
		
		public static void maxHeapAdjust(int[]array,int index,int heapSize)
		{
			int i=index;
			int left=2*index+1,right=2*(index+1);
			if(left<heapSize&&array[i]<array[left])
			{
				i=left;
			}
			if(right<heapSize&&array[i]<array[right])
			{
				i=right;
			}
			if(i!=index)
			{
				swap(array,i,index);
				maxHeapAdjust(array,i,heapSize);
			}
			
		}
		
		public static void buildMaxHeap(int[]array,int heapSize)
		{
			//int parent=(int)Math.floor(array.length-1)/2;
			//int parent=heapSize/2-1;
			int parent=(int)Math.floor((heapSize-1)/2);
			for(int i=parent;i>=0;--i)
			{
				maxHeapAdjust(array,i,heapSize);
			}
		}
		
		public static void maxHeapSort(int[]array)
		{
			buildMaxHeap(array,array.length);
			for(int i=array.length-1;i>0;--i)
			{
				swap(array,0,i);
				maxHeapAdjust(array,0,i);
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

			maxHeapSort(a);
			
			System.out.println("after sort,array is as below:");
			PrintUtils.print(a);
		}	
	}

某一次的输出结果如下:  

	before sort,array is as below:
	60 47 23 74 87 98 76 36 66 84 77 99 78 14 22 60 28 68 71 3  
	after sort,array is as below:
	3 14 22 23 28 36 47 60 60 66 68 71 74 76 77 78 84 87 98 99  

3.算法分析  

由于每次重新恢复堆的时间复杂度为O(log2n),共(n-1)次重新恢复堆操作，再加上前面建立堆时n/2次向下调整，故堆排序的时间复杂度为O(nlog2n);  



