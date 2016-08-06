---
layout: post
title: "红黑树及Java实现"
date: 2016-07-19 01:19:17 +0800
comments: true
categories: data_structure
---
###引言
红黑树（Red–black tree）是一种自平衡二叉查找树，是在计算机科学中用到的一种数据结构，典型的用途是实现关联数组。它是在1972年由鲁道夫·贝尔发明的，他称之为"对称二叉B树"，它现代的名字是在Leo J. Guibas和Robert Sedgewick于1978年写的一篇论文中获得的。它是复杂的，但它的操作有着良好的最坏情况运行时间，并且在实践中是高效的：它可以在O(logN)时间内做查找，插入和删除.  

有人可能觉得奇怪，已经有了AVL这种查找，插入和删除都严格为O(logN)复杂度的平衡二叉树，为什么还需要红黑树呢？<!--more-->  

其实是由于AVL为了保持严格的平衡，在插入和删除节点时需要进行大量的旋转（旋转较耗时)，从而使它不适合用于需要频繁插入和删除的场合。当然，如果建立后基本只有查找，而没有插入和删除的话，则AVL的性能是最好的。  

红黑树和AVL树一样都对插入时间、删除时间和查找时间提供了最好可能的最坏情况担保。由于红黑树实际上只是保持局部平衡，从而在插入和删除时不需要或者只需要进行少量的旋转即可，从而使它在实际工程中得到了大量的使用，从Linux的任务调度，到Android的Binder机制，都有红黑树的身影。本文我们就一起来揭开它的神秘面纱吧!

###1.基本性质
红黑树是每个节点都带有颜色的二叉查找树，颜色为红色或黑色。红黑树需要满足以下性质:  
1).节点是红色或黑色  
2).根是黑色  
3).所有叶子都是黑色（叶子是NIL节点）  
4)每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
5)从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。  

如下是一棵典型的红黑树：  

{% img /images/btree/red_black/red_black_sample.png %}

这些约束确保了红黑树的关键特性:从根到叶子的最长的可能路径不多于最短路径的两倍。结果是这个树大致上是平衡的。因为操作比如插入、删除和查找某个值的最坏情况时间都要求与树的高度成比例，这个在高度上的理论上限允许红黑树在最坏情况下都是高效的，而不同于普通的二叉查找树。  

要知道为什么这些性质确保了这个结果，注意到性质4导致了路径不能有两个毗连的红色节点就足够了。最短的可能路径都是黑色节点，最长的可能路径有交替的红色和黑色节点。因为根据性质5所有最长的路径都有相同数目的黑色节点，这就表明了没有路径能多于任何其他路径的两倍长。  

在很多树数据结构的表示中，一个节点有可能只有一个子节点，而叶子节点包含数据。用这种范例表示红黑树是可能的，但是这会改变一些性质并使算法复杂。为此，本文中我们使用"null叶子"或"空（null）叶子"，如上图所示.  

###2.定义 
为了保持它的基本性质，红黑树在插入时需要考虑它和父节点的颜色，故节点定义如下:  

```java
public class RBNode<T extends Comparable<T>> {

    public static final boolean RED=false;
    public static final boolean BLACK=true;
    
    public boolean color;
    public T key;
    public RBNode<T>left;
    public RBNode<T>right;
    public RBNode<T>parent;

    public RBNode(){

    }

    public RBNode(T key,boolean color,RBNode<T>left,RBNode<T>right,RBNode<T>parent){
        this.key=key;
        this.color=color;
        this.left=left;
        this.right=right;
        this.parent=parent;
    }

}

```
红黑树的定义如下:  

```java

public class RBTree<T extends Comparable<T>> {

   private RBNode<T>root; 
    
   private RBNode<T>getGrandParent(RBNode<T>node){
       return node.parent.parent;
   }

   private RBNode<T>getUncle(RBNode<T>node){
       if(node.parent==getGrandParent(node).left){
           return getGrandParent(node).right;
       }else{
           return getGrandParent(node).left;
       }
   }

}

```
其中getGrandParent()和getUncle()会在后面用到，故这里先给出。  

###3.插入

首先以普通二叉查找树的方法增加节点并标记它为红色(因为性质5的原因，如果设为黑色，就会导致根到叶子的路径上有一条路上，多一个额外的黑节点，这个是很难调整的。但是设为红色节点后，可能会导致出现两个连续红色节点的冲突，那么可以通过颜色调换（color flips）和树旋转来调整。)代码如下:  

```java

public void insert(T key){

	    RBNode<T>newNode;
        if(root==null){
            root=new RBNode<>(key,RBNode.RED,null,null,null);
            newNode=root;
            
        }else{
        	RBNode<T>currentNode=root;
	        int result;
	        while(true){
	            result=key.compareTo(currentNode.key);
	            if(result<0){
	                if(currentNode.left==null){
	                    currentNode.left=new RBNode<>(key,RBNode.RED,null,null,currentNode);
	                    newNode=currentNode.left;
	                    break;
	                }else{
	                    currentNode=currentNode.left;
	                }
	            }else if(result>0){
	                if(currentNode.right==null){
	                    currentNode.right=new RBNode<>(key,RBNode.RED,null,null,currentNode);
	                    newNode=currentNode.right;
	                    break;
	                }else{
	                    currentNode=currentNode.right;
	                }
	            }else{
	                return;
	            }
	        }
        }
        //重新修正为红黑树
        insertFix(newNode);

 }

```
插入节点后，我们会发现:  
1)性质1和性质3总是保持着;  
2)性质4只在增加红色节点、重绘黑色节点为红色，或做旋转时受到威胁。  
3)性质5只在增加黑色节点、重绘红色节点为黑色，或做旋转时受到威胁  

在下面的示意图中，将要插入的节点标为N，N的父节点标为P，N的祖父节点标为G，N的叔父节点标为U.  
综合起来，修正主要考虑以下情形:  

1)新节点N位于根上，无父节点。在这种情形下，我们把它重绘为黑色以满足性质2。因为它在每个路径上对黑节点数目增加一，性质5匹配。  

```java

private void insertCase1(RBNode<T>newNode){
	if(newNode.parent==null){
		newNode.color=RBNode.BLACK;
	}else{
		insertCase2(newNode);
	}
}

```
2)新节点的父节点P是黑色，所以性质4没有失效（新节点是红色的）。在这种情形下，树仍是有效的。性质5也未受到威胁，尽管新节点N有两个黑色叶子子节点,但由于新节点N是红色，通过它的每个子节点的路径中的黑色节点数不变。从而可直接返回。  

```java

private void insertCase2(RBNode<T>newNode){
	if(newNode.parent.color==RBNode.BLACK){
		return;
	}else{
		//it means parent.color==RBNode.RED
		insertCase3(newNode);
	}
}

```
3)如果父节点P和叔父节点U二者都是红色，（此时新插入节点N做为P的左子节点或右子节点都属于情形3，这里右图仅显示N做为P左子的情形）则我们可以将它们两个重绘为黑色并重绘祖父节点G为红色（用来保持性质5）。现在我们的新节点N有了一个黑色的父节点P。因为通过父节点P或叔父节点U的任何路径都必定通过祖父节点G，在这些路径上的黑节点数目没有改变。但是，红色的祖父节点G可能是根节点，这就违反了性质2，也有可能祖父节点G的父节点是红色的，这就违反了性质4。为了解决这个问题，我们在祖父节点G上递归地进行情形1的整个过程。（把G当成是新加入的节点进行各种情形的检查）.  

{% img /images/btree/red_black/insert_case3.png %}

```java

 /**
   * pre-conditions:parent.color==RBNode.RED
   * @param newNode
   */
private void insertCase3(RBNode<T>newNode){
	if(getUncle(newNode)!=null&&getUncle(newNode).color==RBNode.RED){
        newNode.parent.color=RBNode.BLACK;
        newNode.parent.color=RBNode.BLACK;
        //has uncle,so getGrandParent(newNode) will not be null
        getGrandParent(newNode).color=RBNode.RED;
        insertCase1(getGrandParent(newNode));
	}else{
		//it means parent.color==RBNode.RED and getUncle(newNode).color==RBNode.BLACK if getUncle(newNode)!=null
		insertCase4(newNode);
	}
}

```

4)父节点P是红色而叔父节点U是黑色或缺少，并且新节点N是其父节点P的右子节点而父节点P又是其父节点的左子节点。在这种情形下，我们进行一次左旋转调换新节点和其父节点的角色;接着，我们按情形5处理以前的父节点P以解决仍然失效的性质4。注意这个改变会导致某些路径通过它们以前不通过的新节点N（比如图中1号叶子节点）或不通过节点P（比如图中3号叶子节点），但由于这两个节点都是红色的，所以性质5仍有效.还有一种对称的情况，进行右旋转即可。  

{% img /images/btree/red_black/insert_case4.png %}

```java
/**
   * pre-conditions:parent.color==RBNode.RED and getUncle(newNode).color==RBNode.BLACK if getUncle(newNode)!=null
   * @param newNode
   */
private void insertCase4(RBNode<T>newNode){
	if(newNode==newNode.parent.right&&newNode.parent==getGrandParent(newNode).left){
        newNode.parent=rotateLeft(newNode.parent);
        newNode=newNode.parent.left; 
	}else if(newNode==newNode.parent.left&&newNode.parent==getGrandParent(newNode).right){
		newNode.parent=rotateRight(newNode.parent);
		newNode=newNode.parent.right;
	}
	insertCase5(newNode);
}

```
5)父节点P是红色而叔父节点U是黑色或缺少，新节点N是其父节点的左子节点，而父节点P又是其父节点G的左子节点。在这种情形下，我们进行针对祖父节点G的一次右旋转（还有一种对称的情形，进行左旋转即可).  

在旋转产生的树中，以前的父节点P现在是新节点N和以前的祖父节点G的父节点。我们知道以前的祖父节点G是黑色，否则父节点P就不可能是红色（如果P和G都是红色就违反了性质4，所以G必须是黑色）。我们切换以前的父节点P和祖父节点G的颜色，结果的树满足性质4。性质5也仍然保持满足，因为通过这三个节点中任何一个的所有路径以前都通过祖父节点G，现在它们都通过以前的父节点P。在各自的情形下，这都是三个节点中唯一的黑色节点。  

{% img /images/btree/red_black/insert_case5.png %}

```java

/**
   * pre-conditions:newNode.color==RBNode.RED,parent.color==RBNode.RED and getUncle(newNode).color==RBNode.BLACK if getUncle(newNode)!=null
   * @param newNode
   */
private void insertCase5(RBNode<T>newNode){
	newNode.parent.color=RBNode.BLACK;
	getGrandParent(newNode).color=RBNode.RED;
	if(newNode==newNode.left&&newNode.parent==getGrandParent(newNode).left){
		getGrandParent(newNode)=rotateRight(getGrandParent(newNode));
	}else{
		getGrandParent(newNode)=rotateLeft(getGrandParent(newNode));
	}
}

```
综上，insertFix()的代码为：

```java

private void insertFix(RBNode<T>newNode){
	insertCase1(newNode);
}

```
###4.删除
如果需要删除的节点有两个儿子，那么问题可以被转化成删除另一个只有一个儿子的节点的问题（为了表述方便，这里所指的儿子，为非叶子节点的儿子）。对于二叉查找树，在删除带有两个非叶子儿子的节点的时候，我们找到要么在它的左子树中的最大元素、要么在它的右子树中的最小元素，并把它的值转移到要删除的节点中。我们接着删除我们从中复制出值的那个节点，它必定有少于两个非叶子的儿子。因为只是复制了一个值，不违反任何性质，这就把问题简化为如何删除最多有一个儿子的节点的问题。它不关心这个节点是最初要删除的节点还是我们从中复制出值的那个节点。  

所以我们要考虑的只剩下只有一个孩子的节点。有以下情形需要考虑:  

+ 如果我们删除一个红色节点（此时该节点的儿子将都为叶子节点），它的父亲和儿子一定是黑色的。所以我们可以简单的用它的黑色儿子替换它，并不会破坏性质3和性质4。通过被删除节点的所有路径只是少了一个红色节点，这样可以继续保证性质5;  

+ 被删除节点是黑色而它的儿子是红色,只需要在删除后将它的孩子节点绘成黑色即可;  

+ 需要进一步讨论的是在要删除的节点和它的儿子二者都是黑色的时候，这是一种复杂的情况。我们首先把要删除的节点替换为它的儿子。出于方便，称呼这个儿子为N（在新的位置上），称呼它的兄弟（它父亲的另一个儿子）为S。在下面的示意图中，我们还是使用P称呼N的父亲，SL称呼S的左儿子，SR称呼S的右儿子。我们将使用下述函数找到兄弟节点：  

```java

private RBNode<T> getSibling(RBNode<T>node){
	if(node==node.parent.left){
		return node.parent.right;
	}else{
		return node.parent.left;
	}
}

```
删除节点的代码如下：  

```java

public void remove(RBNode<T>node){
	//get the non-null child
	RBNode<T>child=node.left!=null?node.left:node.right;
    
    //replace node with child
    replaceNode(node,child);

    if(node.color==RBNode.BLACK){
    	if(child.color==RBNode.RED){
    		child.color=RBNode.BLACK;
    	}else{
    		removeCase1(child);
    	}
    }
}


```
1)N（即上面的child)是新的根。在这种情形下，我们就做完了。我们从所有路径去除了一个黑色节点，而新根是黑色的，所以性质都保持着。  

```java
/**
*pre-conditions:node.color==RBNode.BLACK
*/
private void removeCase1(RBNode<T>node){
	if(node.parent==null){
		return;
	}else{
		removeCase2(node);
	}
}

```
2)S是红色。在这种情形下我们在N的父亲上做左旋转(对称的情况则需要右旋)，把红色兄弟转换成N的祖父，我们接着对调N的父亲和祖父的颜色。完成这两个操作后，尽管所有路径上黑色节点的数目没有改变，但现在N有了一个黑色的兄弟和一个红色的父亲（它的新兄弟是黑色因为它是红色S的一个儿子），所以我们可以接下去按情形4、情形5或情形6来处理。如下图所示: 

{% img /images/btree/red_black/remove_case2.png %}

代码如下:  

```java
/**
*pre-conditions:node.color==RBNode.BLACK and node.parent!=null
*/
private void removeCase2(RBNode<T>node){
    RBNode<T>sibling=getSibling(node);
    if(sibling.color==RBNode.RED){
    	node.parent.color=RBNode.RED;
    	sibling.color=RBNode.BLACK;
    	if(node==node.parent.left){
    		node.parent=rotateLeft(node.parent);
    	}else{
    		node.parent=rotateRight(node.parent);
    	}
    }
    removeCase3(node);
}

```
3) N的父亲、S和S的儿子都是黑色的。在这种情形下，我们简单的重绘S为红色。结果是通过S的所有路径，它们就是以前不通过N的那些路径，都少了一个黑色节点。因为删除N的初始的父亲使通过N的所有路径少了一个黑色节点，这使事情都平衡了起来。但是，通过P的所有路径现在比不通过P的路径少了一个黑色节点，所以仍然违反性质5。要修正这个问题，我们要从情形1开始，在P上做重新平衡处理。  

```java

private void removeCase3(RBNode<T>node){
	RBNode<T>sibling=getSibling(node);
	if((node.parent.color==RBNode.BLACK)&&(sibling.color==BLACK)
	&&(sibling.left.color==RBNode.BLACK)&&(sibling.right.color==BLACK)){
		sibling.color=RBNode.RED;
		removeCase1(node.parent);
	}else{
		removeCase4(node);
	}
}

```
4) S和S的儿子都是黑色，但是N的父亲是红色。在这种情形下，我们简单的交换N的兄弟和父亲的颜色。这不影响不通过N的路径的黑色节点的数目，但是它在通过N的路径上对黑色节点数目增加了一，添补了在这些路径上删除的黑色节点。  

{% img /images/btree/red_black/remove_case4.png %}

```java

private void removeCase4(RBNode<T>node){

   RBNode<T>sibling=getSibling(node);
   if((node.parent.color==RBNode.RED)&&(sibling.color==RBNode.BLACK)
       &&(sibling.right.color==RBNode.BLACK)&&(sibling.right.color==RBNode.BLACK)){
       	  sibling.color=RBNode.RED;
       	  node.parent.color=RBNode.BLACK;
    }else{
    	  removeCase5(node);
    }
}

```
5)S是黑色，S的左儿子是红色，S的右儿子是黑色，而N是它父亲的左儿子。在这种情形下我们在S上做右旋转，这样S的左儿子成为S的父亲和N的新兄弟。我们接着交换S和它的新父亲的颜色。所有路径仍有同样数目的黑色节点，但是现在N有了一个黑色兄弟，他的右儿子是红色的，所以我们进入了情形6。N和它的父亲都不受这个变换的影响。  

{% img /images/btree/red_black/remove_case5.png %}

代码如下:  

```java

private void removeCase5(RBNode<T>node){
	RBNode<T>sibling=getSibling(node);
	if(sibling.color==RBNode.BLACK){
		if((node==node.parent.left)&&(sibling.right.color==RBNode.BLACK)
		    &&(sibling.left.color==RBNode.RED)){
		    	sibling.color=RBNode.RED;
		    	sibling.left.color=RBNode.BLACK;
		    	sibling=rotateRight(sibling);
		    }else if((node==node.parent.right)&&(sibling.left.color==RBNode.BLACK)
		    &&(sibling.right.color==RBNode.RED)){
                sibling.color=RBNode.RED;
                sibling.right.color=RBNode.BLACK;
                sibling=rotateLeft(sibling);
		    }
	}
	removeCase6(node);
}

```
6)S是黑色，S的右儿子是红色，而N是它父亲的左儿子。在这种情形下我们在N的父亲上做左旋转，这样S成为N的父亲（P）和S的右儿子的父亲。我们接着交换N的父亲和S的颜色，并使S的右儿子为黑色。子树在它的根上的仍是同样的颜色，所以性质3没有被违反。但是，N现在增加了一个黑色祖先：要么N的父亲变成黑色，要么它是黑色而S被增加为一个黑色祖父。所以，通过N的路径都增加了一个黑色节点。  
此时，如果一个路径不通过N，则有两种可能性：  
+ 它通过N的新兄弟。那么它以前和现在都必定通过S和N的父亲，而它们只是交换了颜色。所以路径保持了同样数目的黑色节点。
+ 它通过N的新叔父，S的右儿子。那么它以前通过S、S的父亲和S的右儿子，但是现在只通过S，它被假定为它以前的父亲的颜色，和S的右儿子，它被从红色改变为黑色。合成效果是这个路径通过了同样数目的黑色节点。  
在任何情况下，在这些路径上的黑色节点数目都没有改变。所以我们恢复了性质4。在示意图中的白色节点可以是红色或黑色，但是在变换前后都必须指定相同的颜色。

{% img /images/btree/red_black/remove_case6.png %}

代码如下:  

```java

private void removeCase6(RBNode<T>node){
   RBNode<T>sibling=getSibling(node);

   sibling.color=node.parent.color;
   node.parent.color=RBNode.BLACK;

   if(node==node.parent.left){
   	  sibling.right.color=RBNode.BLACK;
   	  node.parent=rotateLeft(node.parent);
   }else{
      sibling.left.color=RBNode.BLACK;
      node.parent=rotateRight(node.parent);
   }

}

```
###5.测试

















