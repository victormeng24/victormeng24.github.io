---
title: 用Golang刷一遍LeetCode-Two Sum
date: 2020-01-21 17:36:35
tags:
- Golang
-  算法
---
心血来潮想学一下Go，从最基础的数据结构算法开始学起吧，之前Java刷LeetCode也刷了不少遍，想用Golang把这些题重新写一遍。第一题来看最经典的Two Sum。
PS：一年多不见，LeetCode已经变成了力扣，有了中文官网，可见中国的程序员们对这玩意儿也是刚需了。。

> 给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。
> 
> 你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。
> 
> 示例:
> 
> 给定 nums = [2, 7, 11, 15](), target = 9
> 
> 因为 nums[0]() + nums[1]() = 2 + 7 = 9
> 所以返回 [0, 1]()
> 
> 来源：力扣（LeetCode）
> 链接：https://leetcode-cn.com/problems/two-sum
> 著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

暴力遍历时间复杂度O(n^2)，想要O(n)的解需要一个Map来存数据的value和index，题很简单，借机了解Go的一些语法，上代码

```js
func twoSum(nums []int, target int) []int {
	m := make(map[int]int)
	for i, v := range nums {
		if k, ok := m[target - v]; ok {
			return []int{k, i}
		}
		m[v] = i
	}
	return nil
}
```

用Go第一个比较惊艳的是这个ok的断言方式，感觉非常优雅，用在Map里面做key值非空的判断可以让代码非常简洁。
另外用range遍历数组会产生数组的index和value两个值，这个用起来感觉也蛮方便。
持续学习中。。。

