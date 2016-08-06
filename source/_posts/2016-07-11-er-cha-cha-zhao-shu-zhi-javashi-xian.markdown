---
layout: post
title: "二叉查找树之Java实现"
date: 2016-07-11 12:21:55 +0800
comments: true
categories: data_structure
---

###引言  
二叉查找树的定义:左孩子比父节点小,右孩子比父节点大的二叉树。  
显然,通过中序遍历可得到从小到大的序列,因而二叉查找树常用于快速查找,排序等场合。  
下图就是一棵典型的二叉查找树<!--more-->：  

{% img /images/btree/searchtree/search_tree_sample.png %}

###1.二叉查找树的创建  

二叉查找树的定义如下:  

``` java
public class SearchTree<T extends Comparable<? super T>>{

	private BinaryNode<T>root;
	public SearchTree(){
		root=null;
	}

	public void clear(){
		root=null;
	}

	public boolean isEmpty(){
		return root==null;
	}

	public BinaryNode<T> getRoot(){
		return root;
	}

}


```

利用二叉查找树的定义即可得到其节点插入方法.代码如下:  

```java 

public BinaryNode<T> insert(T t,BinaryNode<T>rootNode,boolean recursionFlag){
	if(recursionFlag){
		if(null==rootNode){
			return new BinaryNode<>(t);
		}
		int result=t.compareTo(rootNode.data);
		if(result<0){
			rootNode.left=insert(t,rootNode.left,true);
		}else if(result>0){
			rootNode.right=insert(t,rootNode.right,true);
		}else{
			//do nothing
		}
		return rootNode;
	}else{

       if(null==rootNode){
       	  return new BinaryNode<>(t);
       }
       int result;
       BinaryNode<T>currentNode=rootNode;
       while(true){
       	  result=t.compareTo(currentNode.data);
       	  if(result<0){
       	  	 if(currentNode.left==null){
       	  	 	currentNode.left=new BinaryNode<>(t);
       	  	 	return rootNode;
       	  	 }else{
       	  	 	currentNode=currentNode.left;
       	  	 }
       	  }else if(result>0){
       	  	 if(currentNode.right==null){
       	  	 	 currentNode.right=new BinaryNode<>(t);
       	  	 	 return rootNode;
       	  	 }else{
       	  	 	  currentNode=currentNode.right;
       	  	 }
       	  }else{
       	  	 return rootNode;
       	  }
       }

	}
}

```
插入方法有了之后,二叉树的创建方法就简单了：  

```java

public void create(T[]array,boolean recursionFlag){
	for(T t:array){
		insert(t,recursionFlag);
	}
}

public void insert(T t,boolean recursionFlag){
	root=insert(t,root,recursionFlag);
}


```
###2.包含 

判断是否包含的方法很简单:如果小于当前节点,则跟其左子树比较;如果大于当前节点,则与其右子树比较;若等于,则返回true;

```java
public boolean contains(T t,boolean recursionFlag){
	return contains(t,root,recursionFlag);
}

private boolean contains(T t,BinaryNode<T>node,boolean recursionFlag){
    if(recursionFlag){
    	if(null==node){
    		return false;
    	}
    	int result=t.compareTo(t.data);
    	if(result<0){
    		return contains(t,node.left,true);
    	}else if(result>0){
    		return contains(t,node.right,true);
    	}else{
    		return true;
    	}
    }else{
    	if(node==null){
    		return false;
    	}
    	int result;
    	BinaryNode<T>currentNode=node;
    	while(true){
    		result=t.compareTo(currentNode.data);
    		if(result<0){
    			if(currentNode.left==null){
    				return false;
    			}else{
    				currentNode=currentNode.left;
    			}
    		}else if(result>0){
    			if(currentNode.right==null){
    				return false;
    			}else{
    				currentNode=currentNode.right;
    			}
    		}else{
    			return true;
    		}
    	}
    }


}


```
###3.查找最小和最大节点  

``` java

public T findMin(boolean recursionFlag){
	return findMin(root,recursionFlag).data;
}

public BinaryNode<T> findMin(BinaryNode<T>rootNode,boolean recursionFlag){
    if(recursionFlag){
    	if(null==rootNode){
    		return null;
    	}
    	if(rootNode.left!=null){
           return findMin(rootNode.left,true);
    	}else{
    		return rootNode;
    	}
    }else{
    	if(null==rootNode){
    		return null;
    	}
    	BinaryNode<T>currentNode=rootNode;
    	while(true){
    		if(currentNode.left!=null){
    			currentNode=currentNode.left;
    		}else{
    			return currentNode;
    		}
    	}
    }

}

public T findMax(boolean recursionFlag){
	return findMax(root,recursionFlag).data;
}

public BinaryNode<T> findMax(BinaryNode<T>rootNode,boolean recursionFlag){
	if(recursionFlag){
		if(null==rootNode){
			return null;
		}
		if(rootNode.right==null){
			return rootNode;
		}else{
			return findMax(rootNode.right,true);
		}
	}else{
		if(null==rootNode){
			return;
		}
		BinaryNode<T>currentNode=rootNode;
		while(true){
			if(currentNode.right==null){
				return currentNode.right;
			}else{
				currentNode=currentNode.right;
			}
		}
	}
}

```
###4.寻找前驱和后驱节点  
前驱节点:二叉树中左子树的最大节点;  
后驱节点:二叉树中右子树的最小节点;  
显然，中序遍历时它们分别排在根节点的前面和后面.  

由于上面已经给出了findMax()和findMin的方法,从而查找前驱和后驱节点的方法也非常简单：  

``` java
public BinaryNode<T>findPredecessor(BinaryNode<T>rootNode,boolean recursionFlag){
	return findMax(rootNode.left,recursionFlag);
}

public BinaryNode<T>findSuccessor(BinaryNode<T>rootNode,boolean recursionFlag){
	return findMin(rootNode.right,recursionFlag);
}

```
值得注意的是,前驱节点和后驱节点都最多只有一个孩子:前驱节点最多只有一个右孩子,后驱节点最多只有一个左孩子.  

后面删除节点时会用到这个性质.  

###5.移除节点  

删除较为复杂，主要考虑以下情况:  
1)无孩子  
直接移除  
2)单孩子  
这个比较简单，如果删除的节点有左孩子那就把左孩子顶上去，如果有右孩子就把右孩子顶上去，如下图所示:  

{% img /images/btree/searchtree/search_tree_remove_node.png %}

3)双孩子  

对于一个数组,如果我们要删除一个元素,那么可以让其前面或后面的元素来顶替它.如下图所示:  

{% img /images/btree/searchtree/remove_node_in_array.png %}

显然,这个要利用前面说的前驱和后驱节点,即既可以利用其前驱节点来替换它,也可以利用其后驱节点来替换它.如下图所示:  

{% img /images/btree/searchtree/remove_node_stree_case2.png %}


```java

public BinaryNode<T> remove(T t ,boolean recursionFlag){
	return remove(t,root,recursionFlag);
}

private BinaryNode<T>remove(T t,BinaryNode<T>rootNode,boolean recursionFlag){
    if(recursionFlag){
    	if(null==rootNode){
    		return null;
    	}
    	int result=t.compareTo(rootNode.data);
    	if(result<0){
    		rootNode.left=remove(t,rootNode.left,true);
    	}else if(result>0){
    		rootNode.right=remove(t,rootNode.right,true);
    	}else{
    		if(rootNode.left!=null&&rootNode.right!=null){
                //利用后驱节点，其实也可以利用前驱节点.
                //先替换值
                rootNode.data=findMin(rootNode.right,true).data;
                //再移除后驱节点
                rootNode.right=remove(rootNode.data,rootNode.right,true);
    		}else{
                 rootNode=(rootNode.left!=null)?rootNode.left:rootNode.right;
    		}
    	}
    	return rootNode;
    }else{
    	if(null==)
    	BinaryNode<T>currentNode=rootNode;
    	BinaryNodde<T>parentNode=null;
    	int result;
    	while(true){
    		result=t.comparetTo(currentNode.data);
    		if(result<0){
    			if(null!=currentNode.left){
    				parentNode=currentNode;
    				currentNode=currentNodde.left;
    			}else{
    				return null;
    			}
    		}else if(result>0){
    			if(null!=currentNode.right){
    				parentNode=currentNode;
    				currentNode=currentNode.right;
    			}else{
    				return null;
    			}
    		}else{
                if(null!=currentNode.left&&null!=currentNode.right){
                	//////////////////////////////利用前驱节点
                	BinaryNode<T>currentNode2=currentNode.left;
                	BinaryNode<T>parentNode2=null;
                	while(currentNode2.right!=null){
                		parentNode2=currentNode2;
                		currentNode2=currentNode2.right;
                	}
                	if(parentNode2==null){
                		currentNode.data=currentNode2.data;
                		currentNode.left=null;
                	}else{
                		currentNode.data=currentNode2.data;
                		//前驱节点至多只有一个左孩子
                		parentNode2.right=currentNode2.left;
                	}
                	//////////////////////////////////
                }else if(currentNode.left!=null){
                	 //that means currentNode is just node,i.e the root
                	 if(parentNode==null){
                	 	rootNode=currentNode.left;
                	 	return currentNode;
                	 }
                	 if(parentNode.left==currentNode){
                	 	//this means currentNode is the left child of parentNode
                	 	parentNode.left=currentNode.left;
                	 }else{
                	 	//this means currentNode is the right child of parentNode
                	 	parentNode.right=currentNode.left;
                	 }
                }else{
                	if(parentNode==null){
                		rootNode=currentNode.right;
                		return rootNode;
                	}
                	if(parentNode.left==currentNode){
                		parentNode.left=currentNode.right;
                	}else{
                		parentNode.right=currentNode.right;
                	}
                }
                return currentNode;
    		}
    	}
    }

}

```
上面对于左右孩子都不为空的情况利用了前驱节点,其实也可以利用后驱节点，代码片段如下:  

```java

//////////////////////////////////利用后驱节点
BinaryNode<T>currentNode2=currentNode.right;
BinaryNode<T>parentNode2=null;
while(currentNode2.left!=null){
	parentNode2=currentNode2;
	currentNode2=currentNode2.left;
}
//意味着currentNode.right即为其后驱节点
if(parentNode2==null){
	currentNode.data=currentNode2.data;
	currentNode.right=null;
}else{
	currentNode.data=currentNode2.data;
	parentNode2.left=currentNode2.right;
}
/////////////////////////////////////////////

```
###6.测试  
测试源码如下:  

```java
Integer[]intArray2={30,20,50,10,28,16,900,110,320,1,-10,25,26,23};
SearchTree<Integer>searchTree3=new SearchTree<>();
searchTree3.create(intArray2,true);
BinaryTree.printAsHorizontalTreeShape(searchTree3.getRoot());
BinaryTree.print("\ninOrder of searchTree3:");
BinaryTree.inOrderTraversal(searchTree3.getRoot(),true);
BinaryTree.print("\nafter remove -10,inOrder:");
searchTree3.remove(-10,true);
BinaryTree.inOrderTraversal(searchTree3.getRoot(),true);
BinaryTree.printAsHorizontalTreeShape(searchTree3.getRoot());
BinaryTree.print("\nafter remove 20,inOrder:");
searchTree3.remove(20,true);
BinaryTree.inOrderTraversal(searchTree3.getRoot(),true);
BinaryTree.printAsHorizontalTreeShape(searchTree3.getRoot());
BinaryTree.print("\nafter remove 110,inOrder:");
searchTree3.remove(110,true);
BinaryTree.inOrderTraversal(searchTree3.getRoot(),true);
BinaryTree.printAsHorizontalTreeShape(searchTree3.getRoot());
BinaryTree.print("\n");

```

测试结果如下:  

{% img /images/btree/searchtree/searchtree_test01.png %}

{% img /images/btree/searchtree/search_tree_test02.png %}