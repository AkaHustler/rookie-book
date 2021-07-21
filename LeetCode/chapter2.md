## 第二章 手把手刷数据结构

### 手把手刷二叉树

#### 快速排序、归并排序与二叉树的关系

- 快速排序

快速排序逻辑是，对 **num[lo..hi]** 进行排序，先寻找一个分界点 **p** ，通过交换元素使得 **num[lo..p-1]** 小于 **num[p]**，且 **num[p+1..hi]** 大于 **num[p]** ，然后递归去 **num[lo..p-1]** 于 **num[p+1..hi]** 中寻找新的分界点，使得整个数组有序

先构造分界点，然后再去左右子数组构造分界点，即二叉树的前序遍历逻辑

``` java
 void sort(int[] nums, int lo, int hi) {
   /******  前序遍历位置 ******/
   // 通过交换元素构建分界点 p
   int p = partition(nums, lo, hi);
   /**************************/
   
   sort(nums, lo, p - 1);
   sort(nums, p + 1, hi)
 }
```



- 归并排序

对 **num[lo..hi]** 排序，先对 **num[lo..mid]** 排序，再对 **num[mid+1..hi]** 排序，最后将有序的子数组合并，整个数组就排好序了

先对左右子数组排序，然后合并，即二叉树的后序遍历处理，还包括了分治算法

``` java
void sort(int[] nums, int lo, int hi) {
  int mid = (lo + hi) / 2;
  sort(nums, lo, mid);
  sort(nums, mid + 1, hi);
  
  /****** 后序遍历位置 ******/
  //合并两个排好序的子数组
  merge(nums, lo, mid, hi);
  /*************************/
}
```

#### 递归算法的tips

举个例子：计算一颗二叉树共有几个节点

``` java
// 定义：count(root) 返回以 root 为根的树有多少节点
int count(TreeNode root) {
  //base case
  if (root == null) {return 0};
  // 自己加上左子树节点加上右子树节点 就是整棵树的节点数
  return 1 + count(root.left) + count(root.right);
}
```

**写树相关的算法，简单说就是，先搞清楚当前 root 节点该做什么，然后根据函数定义递归调用子节点** ，递归调用会让子节点做相同的事。

**写递归算法的关键是，明确函数的「定义」是什么，然后利用这个定义推到最终结果，不要跳入递归的细节**

#### 算法实践

- 翻转二叉树

输入一个二叉树跟节点 **root**，把整棵树镜像翻转，比如输入二叉树如下：

``` java
     4
   /   \
  2     7
 / \   / \
1   3 6   9
```

算法原地翻转二叉树，使得以 **root**为根的树变成：

``` java
     4
   /   \
  7     2
 / \   / \
9   6 3   1
```

**只要我们把二叉树上的每个节点的左右子节点进行交换，我们得到的就是完全翻转后的二叉树**

``` java
//将整棵树的节点翻转
TreeNode invertTree(TreeNode root) {
  //base case
  if (root == null) { return null;}
  //前序遍历位置
  //root 节点需要交换它的左右子节点
  TreeNode temp = root.left;
  root.left = root.right;
  root.right = temp;
  //让左右子节点继续翻转他们的子节点
  invertTree(root.left);
  invertTree(root.right);
}
```

**二叉树的一个难点是，如何把题目细化成每个节点需要做的事**

- 填充二叉树节点的右侧指针

[116. 填充每个节点的下一个右侧节点指针 - 力扣（LeetCode） (leetcode-cn.com)](https://leetcode-cn.com/problems/populating-next-right-pointers-in-each-node/)

![](https://labuladong.gitee.io/algo/images/%e4%ba%8c%e5%8f%89%e6%a0%91%e7%b3%bb%e5%88%97/1.png)

就是将二叉树每层的节点用next指针连起来，题目为完美二叉树，即除了最右侧的节点指针 **next** 会指向null，其他节点一定有相邻的节点

**递归算法思路：不要考虑太细化，需要考虑的是每个节点应该做的事，确定函数作用与含义，递归处理子节点时就会将完成要求**

**想法1**: **root节点** 做的是将自己的子节点关联上，自己的关联由自己的父亲节点去关联。明确这点后，将左右子节点关联又包含两种情况：

**1.左子节点关联自己的右子节点** ，**2.右子节点关联的是自己的兄弟节点(无则关联null)的左子节点**

第一种情况好处理，第二种情况如果单靠 **root节点** 按理说是完成不了的，这个时候可以用题中的定义，利用 **root节点** 的next节点找到兄弟节点，进而将兄弟节点的左子节点关联到 **root节点** 的右子节点。

**关注题目的定义，确定root节点的需要做的事**

- 将二叉树展开为链表

[114. 二叉树展开为链表 - 力扣（LeetCode） (leetcode-cn.com)](https://leetcode-cn.com/problems/flatten-binary-tree-to-linked-list/)

![](https://assets.leetcode.com/uploads/2021/01/14/flaten.jpg)

题目是将二叉树展开为一个单链表，展开后的单链表同样适用树节点**TreeNode**，其中 **right指针** 指向链表下一个节点， **left指针** 总是指向null。

```php
class Solution {
  public $tempNode;
  
  function flatten($root) {
    //base case
    if ($root == null) {
      retutn null;
    }
    //遍历子节点，后序遍历
    $this->flatten($root->right);
    $this->flatten($root->left);
    //root子节点做的事情
    $root->right = $this->tempNode;
    $root->left  = null;
    $this->tempNode = $root;
    return null;
  }
}
```

解题思路：

- 最大二叉树

[654. 最大二叉树 - 力扣（LeetCode） (leetcode-cn.com)](https://leetcode-cn.com/problems/maximum-binary-tree/)

![](https://assets.leetcode.com/uploads/2020/12/24/tree1.jpg)

解题思路：

- 从前序与中序遍历序列构造二叉树

[105. 从前序与中序遍历序列构造二叉树 - 力扣（LeetCode） (leetcode-cn.com)](https://leetcode-cn.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/)

**Input**: preorder = [3,9,20,15,7], inorder = [9,3,15,20,7]
**Output**: [3,9,20,null,null,15,7]

![](https://assets.leetcode.com/uploads/2021/02/19/tree.jpg)

解题思路：

- 寻找重复子树

[652. 寻找重复的子树 - 力扣（LeetCode） (leetcode-cn.com)](https://leetcode-cn.com/problems/find-duplicate-subtrees/)

```php
		 1
       / \
      2   3
     /   / \
    4   2   4
       /
      4

```

下面是重复的子树：

```php
      2
     /
    4
```

和

```php
 4
```

给定一棵二叉树，返回所有重复的子树。

**后序遍历获取节点的左子树与右子树**

将 **root节点** 的子树表示出来存放到hashmap中，从而得到重复子树



### 手把手刷二叉搜索树

二叉搜索树（Binary Search Tree）**BST**的特性

1. 对于BST的每一个节点 **node** ,左子树节点的值都比 **node**的值要小，右子树节点的值都要比 **node**的值大
2. 对于BST的每个节点 **node**，它的左子树和右子树都是BST

直接基于BST的数据结构有AVL树、红黑树等等，拥有了自平衡性质，可以提供 **logN** 级别的增删改查效率；还有B+树、线段树都是基于BST的思想来设计的。

**BST还有一个重要的性质：BST的中序遍历结果是有序的（升序）**

- 二叉搜索树中第k小的元素

[230. 二叉搜索树中第K小的元素 - 力扣（LeetCode） (leetcode-cn.com)](https://leetcode-cn.com/problems/kth-smallest-element-in-a-bst/)

![](https://assets.leetcode.com/uploads/2021/01/28/kthtree1.jpg)

```php
输入：root = [3,1,4,null,2], k = 1
输出：1
```

解题思路：

1. 该题利用了BST的特性：中序遍历结果是有序的（升序），所以获取中序遍历结果即可
2. 在中序遍历过程中，根据当前遍历到的rank与k相比，当遍历到想要的节点时，可直接返回

1，2其实都是要完成一次中序遍历，其效率为O(n)，但是基于BST的AVL、红黑树都能达到log(N)的查询效率，那BST应该也可以，BST操作高效的原因主要是，其左子树节点的值比右子树节点要小，每个节点可以对比自身的值去判断判断左子树还是右子树搜索，从而达到对数级的时间复杂度。

所以想要查找排名为k的元素，BST中每个节点应该知道自己排名第几，当前节点为m，m > k 接下来去左子树搜索， m < k 接下来去右子树搜索，m = k 即所查询，但是这种依赖节点需要记录当前排名信息。

- 把二叉搜索树转换为累加树

[538. 把二叉搜索树转换为累加树 - 力扣（LeetCode） (leetcode-cn.com)](https://leetcode-cn.com/problems/convert-bst-to-greater-tree/)

![](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2019/05/03/tree.png)

给出二叉 搜索 树的根节点，该树的节点值各不相同，请你将其转换为累加树（Greater Sum Tree），使每个节点 node 的新值等于原树中大于或等于 node.val 的值之和。

解题思路：利用BST特性，可以去获取 root节点右子树的val总和作为新值，但是root父节点也可能比root大，所以不能单纯的去获取右子树。

BST特性之一，中序遍历结果是有序的（升序）即

```java
void traverse(TreeNode root) {
    if (root == null) return;
    traverse(root.left);
    // 中序遍历代码位置
    print(root.val);
    traverse(root.right);
}
```

我们将遍历顺序颠倒下，即

``` java
void traverse(TreeNode root) {
    if (root == null) return;
    // 先递归遍历右子树
    traverse(root.right);
    // 中序遍历代码位置
    print(root.val);
    // 后递归遍历左子树
    traverse(root.left);
}
```

得到的是降序的遍历结果，这样我们就可以将 **root**节点操作累加的和作为 **root**的新值，因为此时的累加和是比root节点大的所有节点累加的和，嘻嘻嘻



### 手把手刷链表题目

