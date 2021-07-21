## 小白入门计划

### 单链表相关

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

#### 如何k个一组反转链表

#### 如何判断回文链表



