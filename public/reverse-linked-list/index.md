# 算法练习-Reverse-Linked-List


## 起因

今天在刷算法题的时候，碰到了https://leetcode-cn.com/problems/reverse-linked-list/, 看到这个题，我很自然会写出了如下代码：


```go
func reverseList(head *ListNode) *ListNode {
	var prev *ListNode
	curr := head
	for curr != nil {
		nextTmp := curr.Next
		curr.Next = prev
		prev = curr
		curr = nextTmp
	}
	return curr
}
```

也很容易通过了，但是看到题目要求中：是否可以通过递归地反转链表的方式实现，想了半天没有实现，看了题解才知道还可以这么实现

## 递归实现方式

先看一下递归实现的代码：


```go
func reverseList2(head *ListNode) *ListNode {
	if head == nil || head.Next == nil {
		return head
	}
	// p 就是原来链表的最后一个
	p := reverseList2(head.Next)
	head.Next.Next = head
	head.Next = nil
	return p
}
```

递归有时候会不容易理解，我刚看到这个解题方法的时候也是完全不知道怎么回事，所以这里我们可以通过下面代码进行测试，并梳理这个翻转的过程：


```go
package main

import "fmt"

type ListNode struct {
	Val  int
	Next *ListNode
}

// 这个思路非常有意思，需要好好理解
func reverseList2(head *ListNode) *ListNode {
	if head == nil || head.Next == nil {
		return head
	}
	// p 就是原来链表的最后一个
	p := reverseList(head.Next)
	fmt.Println(p.Val)
	fmt.Println(head.Val)
	head.Next.Next = head
	head.Next = nil
	fmt.Println(head.Val)
	return p
}


func main() {
	var curr *ListNode
	var origin *ListNode
	for i := 1; i <= 4; i++ {
		one := &ListNode{
			Val: i,
		}
		if i == 1 {
			origin = one
		}
		if curr != nil {
			curr.Next = one
		}
		curr = one
	}
	res := reverseList(origin)
	fmt.Println(res)
}
```

初始为: [1,2,3,4]
第一步:  从1-2的循环都被压倒栈中，到head=3的时候即reverseList(4)的时候开始弹出栈，这个时候p = 4

第二步:  head.Next.Next = head 相当于将  指定了 3.Next.Next = 4.Next = 3, head.Next = nil 相当于 3.Next = nil, 这个时候 p = [4,3]

第三步: 依次弹出栈，head = 2的时候 head.Next.Next = head 相当于 2.Next.Next = 3.Next = 2 , head.Next = nil 相当于 2.Next = nil 这个时候p = [4,3,2]; 弹出 head =1 的时候， head.Next.Next = head 相当于 1.Next.Next = 2.Next = 1 ,  head.Next = nil 相当于 1.Next = nil 这个时候p = [4,3,2,1]

## 小结
递归的解题思路有点没有那么容易理解，但是如果把每一步进行拆分，其实也能理解这个过程。但是当把过程一步步分析之后，递归的这个解法就会让你有种茅塞顿开的感觉。也是一种学习过程。












