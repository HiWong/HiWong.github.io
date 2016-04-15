---
layout: post
title: "排序算法之插入排序"
date: 2015-10-15 21:54:44 +0800
comments: true
categories: algorithm
---

1.插入排序的思想  
插入排序的思想很简单：第i趟排序将序列中的第i+1个元素插入到前面(i-1)个元素中的合适位置，其中前面的(i-1)个元素是已经按值排好序的。  

2.流程图  

流程图如下所示<!--more-->：  

![InsertSort](http://7xn1yt.com1.z0.glb.clouddn.com/InsertSort.png)

3.算法实现  

    public class InsertSort{

        public static void simpleInsertSort(int[]array)
        {
            if(null==array||array.length<1)
            {
                return;
            }
            int j,temp;
            for(int i=1;i<array.length;++i)
            {
                temp=array[i];
                j=i-1;
                while(j>=0&&temp<array[j])
                {
                    array[j+1]=array[j--];
                }

                array[j+1]=temp;
            }
        }

        public static void main(String[]args)
        {
            int a=new int[20];
            for(int i=0;i<20;++i)
            {
                a[i]=new Random().nextInt(100);
            }

            System.out.println("before sort,array is as below:");
            PrintUtils.print(a);
            simpleInsertSort(a);
            System.out.println("after sort,array is as below:");
            PrintUtils.print(a);
        }


    }

4.改进  

上面的只是简单插入排序算法，当原始序列就是一个按值递增的序列时，对每个i值只进行一次元素之间的比较，故总次数最少，为(n-1)次;  
当原始序列是一个按值递减的序列时，对每个i值要进行(i-1)次比较，此时总次数为n(n-1)/2;  
综上，如果各个情况下的概率相等，则平均值是n*n/4,因而插入排序算法的时间复杂度是O(n^2);  

如果采用折半插入排序的话，虽然最坏情况下的时间复杂度也是O(n^2)，但是最好情况下的时间复杂度是O(log2n),而简单插入排序在最好情况下的时间复杂度是O(n),显然前者要好。  

简单插入排序算法的实现如下:  

    public static void binInsertSort(int[]array)
    {
        if(null==array||array.length<1)
        {
            return;
        }
        int j,low,mid,temp;
        for(int i=1;i<array.length;++i)
        {
            low=0;
            high=i-1;
            temp=array[i];
            while(low<=high)
            {
                mid=(low+high)/2;
                if(temp<array[mid])
                {
                    high=mid-1;
                }
                else
                {
                    low=mid+1;
                }
            }
            for(j=i-1;j>=low;--j)
            {
                array[j+1]=array[j];
            }
            array[low]=temp;
        }
    }