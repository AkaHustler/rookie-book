## 小白入门计划

### 单链表相关

单链表的解题思路：

1. 哑节点 dummy，在链表前面添加一个节点
2. 双指针，1)快慢指针 2)前驱指针和工作指针

#### 递归反转链表的一部分

``` java
//单链表节点的结构
public class ListNode {
  int val;
  ListNode next;
  
  ListNode(int x) {
    val = x;
  }
}
```

反转单链表的一部分，就是给定一个索引区间，让你把单链表这部分的元素反转，其他部分不变。

![](https://mmbiz.qpic.cn/mmbiz_png/map09icNxZ4lC6h97zG2q2kzKZMxfOeBibcZgxicjlib8yX8oNWRicYl56icLWMABxk6s53zQowCWmKFlwaDicgv6993Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这里的索引从 **1** 开始，迭代的思路大概是：先用一个for循环找到位置m，然后循环遍历位置 **m** 到 **n** 实现元素的反转。**递归解法** 不适用for循环，纯递归实现反转。

- 递归反转整个链表

``` java
ListNode reverse(ListNode head) {
  if (head.next == null) {return head;}
  ListNode last = reverse(head.next);
  head.next.next = head;
  head.next = null;
  return last;
}
```

递归的思想就是 **明确递归函数的定义**，reverse函数的定义就是 **输入一个节点head，将以「head为起点」的链表反转，并返回反转之后的头接点**。

有几个地方需要注意下：

要有base case，意思是只有一个节点的时候反转是它是自己；当链表反转之后，新的头节点是last，原来的head节点变成最后一个节点，末尾的节点要指向null

``` java
head->next = null;
```

- 反转列表前N个节点

定义递归函数，

``` java
//将链表的前N个节点反转
ListNode reverseN (ListNode head, int n)
```

``` java
private ListNode precursorNode;
//将链表的前N个节点反转
ListNode reverseN (ListNode head, int n) 
{
  //base case
  if (n == 1) {
    precursorNode = head->next;
    return head;
  }
  ListNode last = reverseN(head.next);
  head->next->next = head;
  head->next = precursorNode;
  return last;
}
```



- 反转链表的一部分

定义递归函数

``` java
//给一个索引区间[m,n]，仅反转区间中的链表元素
ListNode reverseBetween(ListNode head, int m, int n)
```

如果m = 1，就相当于反转列表前n个节点，如果不等于1，将head的索引视为1，是从第m个节点开始反转，对于head->next来说，就是从第m-1个节点开始反转

```java
ListNode reverseBetween(ListNode head, int m, int n) {
  if (m == 1) {
    return reverseN(head, n);
  }
  $head->next = reverseBetween(head->next, m - 1, n - 1);
  return head;
}
```

值得一提的是，**递归操作列表并不是高效的**，虽然时间复杂度都是O(n)，但是迭代解法的空间复杂度是O(1)，而递归解法需要堆栈，空间复杂度是O(n)，主要是理解递归思想。

#### 如何k个一组反转链表

「**k个一组反转链表**」

给定一个链表，每k个节点一组进行反转，返回反转后的链表，如果节点总数不是k的整数倍，请将最后剩余的节点保持原有顺序，示例：

> 给定链表：1->2->3->4->5
>
> 当k=2时，应返回 2->1->4->3->5
>
> 当k=3时，应返回 3->2->1->4->5

- 分析问题

**链表是一种兼具 递归和迭代 性质的数据结构**，这个问题具有递归性质。

递归函数定义

``` php
public function reverseGroup($head, $k)
```

k个节点为1组反转链表，子问题与原问题的结构完全相同，就是具有 **递归性质**。

1. 先反转以 **head** 开头的k个元素
2. 将第 **k+1** 个元素作为head，递归调用 **reverseGroup** 函数

base case就是最后的元素如果不足 **k**个，就保持不变。

``` java
ListNode reverseKGroup(ListNode head, int k) {
    if (head == null) return null;
    // 区间 [a, b) 包含 k 个待反转元素
    ListNode a, b;
    a = b = head;
    for (int i = 0; i < k; i++) {
        // 不足 k 个，不需要反转，base case
        if (b == null) return head;
        b = b.next;
    }
    // 反转前 k 个元素
    ListNode newHead = reverse(a, b);
    // 递归反转后续链表并连接起来
    a.next = reverseKGroup(b, k);
    return newHead;
}
```

reverse函数是反转[a,b)区间的节点，reverseKGroup函数是每k个节点反转为一组反转以head为头节点的链表

reverseKGroup是每k个节点反转，所以需要将 **a.next = reverseKGroup(head, k)** 链接起来

#### 如何判断回文链表

- 最长回文子串

**回文串就是正着读反着读都一样的字符串**

解决该类问题的核心是 **双指针**

给定一个字符串s，找到s中最长的回文子串

> 示例1:
>
> 输入："babad"
>
> 输出："bab"
>
> 注意："aba"也是一个有效答案

> 示例2:
>
> 输入："cbbd"
>
> 输出："bb"

``` java
string longestPalindrome(string s) {}
```

解题思路：

**寻找回文串的核心思想是：从中间开始向两边扩散来判断回文串**

从中间向两边扩散：

> for 0 <= i <= len(s)
>
> ​	找到以 **s[i]** 为中心的回文串
>
> ​	更新答案

但是字符串的长度可能是奇数或偶数，所以改下

> for 0 <= i <= len(s)
>
> ​	找到以 **s[i]** 为中心的回文串
>
> ​	找到以 **s[i]** 和 **s[i+1]** 为中心的回文串
>
> ​	更新答案

还有一些细节问题索引越界等问题，代码中解决
