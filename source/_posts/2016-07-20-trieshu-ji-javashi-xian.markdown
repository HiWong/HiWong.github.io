---
layout: post
title: "Trie树及Java实现"
date: 2016-07-20 10:14:18 +0800
comments: true
categories: data_structure
---
###引言
Trie树，又称为前缀树或字典树，是一种有序树，用于保存关联数组，其中的键通常是字符串。与二叉查找树不同，键不是直接保存在节点中，而是由节点在树中的位置决定。一个节点的所有子孙都有相同的前缀，也就是这个节点对应的字符串，而根节点对应空字符串<!--more-->。  

如下是一棵典型的Trie树:  

{% img /images/btree/trie/trie_tree_sample.png %}

Trie的来源是Retrieval,它常用于前缀匹配和词频统计。可能有人要说了，词频统计简单啊，一个hash或者一个堆就可以搞定，但问题来了，如果内存有限呢？还能这么
玩吗？所以这里我们就可以用trie树来压缩下空间，因为公共前缀都是用一个节点保存的。  

###1.定义

这里为了简化，只考虑了26个小写字母。  

首先是节点的定义:

```java

public class TrieNode {
    
    public TrieNode[] children;

    public char data;

    public int freq;

    public TrieNode() {
        //因为有26个字母
        children = new TrieNode[26];
        freq = 0;
    }

}

```
然后是Trie树的定义:  

```java
public class TrieTree {

    private TrieNode root;

    public TrieTree(){
        root=new TrieNode();
    }

    ...

}

```
###2.插入

由于是26叉树，故可通过charArray[index]-'a';来得知字符应该放在哪个孩子中。  

```java

  public void insert(String word){
        if(TextUtils.isEmpty(word)){
            return;
        }
        insertNode(root,word.toCharArray(),0);
    }

    private static void insertNode(TrieNode rootNode,char[]charArray,int index){

        int k=charArray[index]-'a';
        if(k<0||k>25){
            throw new RuntimeException("charArray[index] is not a alphabet!");
        }
        if(rootNode.children[k]==null){
            rootNode.children[k]=new TrieNode();
            rootNode.children[k].data=charArray[index];
        }

        if(index==charArray.length-1){
            rootNode.children[k].freq++;
            return;
        }else{
            insertNode(rootNode.children[k],charArray,index+1);
        }

    }

```
###3.移除节点

移除操作中，需要对词频进行减一操作。  

```java

 public void remove(String word){
        if(TextUtils.isEmpty(word)){
            return;
        }
        remove(root,word.toCharArray(),0);
    }

    private static void remove(TrieNode rootNode,char[]charArray,int index){
        int k=charArray[index]-'a';
        if(k<0||k>25){
            throw new RuntimeException("charArray[index] is not a alphabet!");
        }
        if(rootNode.children[k]==null){
            //it means we cannot find the word in this tree
            return;
        }

        if(index==charArray.length-1&&rootNode.children[k].freq >0){
            rootNode.children[k].freq--;
        }

        remove(rootNode.children[k],charArray,index+1);
    }

```

###4.查找频率

```java

   public int getFreq(String word){
        if(TextUtils.isEmpty(word)){
            return 0;
        }
        return getFreq(root,word.toCharArray(),0);
    }

    private static int getFreq(TrieNode rootNode,char[]charArray,int index){
        int k=charArray[index]-'a';
         if(k<0||k>25){
            throw new RuntimeException("charArray[index] is not a alphabet!");
        }
        //it means the word is not in the tree
        if(rootNode.children[k]==null){
            return 0;
        }
        if(index==charArray.length-1){
            return rootNode.children[k].freq;
        }
        return getFreq(rootNode.children[k],charArray,index+1);
    }

```
###5.测试

测试代码如下:  

```java

 public static void test(){
        TrieTree trieTree=new TrieTree();
        String sourceStr="Democratic presumptive nominee Hillary Clintons campaign posed pounced on Trumps assertion that British term monetary turmoil might benefit his business venture in Scotland";

        //String sourceStr="the that";
        sourceStr=sourceStr.toLowerCase();

        String[]strArray=sourceStr.split(" ");
        for(String str:strArray){
            trieTree.insert(str);
        }


        String sourceStr2="Every president is tested by world events But Donald Trump thinks about how is his golf resort can profit from that";
        sourceStr2=sourceStr2.toLowerCase();
        String[]strArray2=sourceStr2.split(" ");
        for(String str:strArray2){
            trieTree.insert(str);
        }



        BinaryTree.print("frequence of 'that':"+trieTree.getFreq("that"));
        BinaryTree.print("\nfrequence of 'donald':"+trieTree.getFreq("donald"));

        trieTree.remove("that");
        BinaryTree.print("\nafter remove 'that' once,freq of 'that':"+trieTree.getFreq("that"));
        trieTree.remove("that");
        BinaryTree.print("\nafter remove 'that' twice,freq of 'that':"+trieTree.getFreq("that"));

        trieTree.remove("donald");
        BinaryTree.print("\nafter remove 'donald' once,freq of 'donald':"+trieTree.getFreq("donald"));

        BinaryTree.reallyStartPrint();


    }

```
测试结果如下:  

{% img /images/btree/trie/test_result_trie_tree.png %}
