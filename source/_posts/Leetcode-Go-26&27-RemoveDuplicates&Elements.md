---
title: Leetcode-Go-26&27-RemoveDuplicates&Elements
date: 2020-03-04 14:41:22
tags: 
- 算法
- Golang
---
# 26-Remove Duplicates in Sorted Array
给定一个排序数组，你需要在原地删除重复出现的元素，使得每个元素只出现一次，返回移除后数组的新长度。

不要使用额外的数组空间，你必须在原地修改输入数组并在使用 O(1) 额外空间的条件下完成。

示例 1:

给定数组 nums = [1,1,2](), 

函数应该返回新的长度 2, 并且原数组 nums 的前两个元素被修改为 1, 2。 

你不需要考虑数组中超出新长度后面的元素。
示例 2:

给定 nums = [0,0,1,1,1,2,2,3,3,4](),

函数应该返回新的长度 5, 并且原数组 nums 的前五个元素被修改为 0, 1, 2, 3, 4。

你不需要考虑数组中超出新长度后面的元素。

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/remove-duplicates-from-sorted-array
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

思路：快慢指针，慢指针用于保存新数组的长度，快指针遍历整个数组，当快慢指针相等时快指针跳过，表示遇到重复元素，不等时，交换快指针索引与慢指针索引+1的值，最后的前i+1项即为去重后的数组元素。
自己写了个实现，又看了下解题圈子里的实现，感觉自己的代码还是不够优雅，需要继续努力。
自己代码：
```js
func removeDuplicates(nums []int) int {
	if len(nums) == 0 {
		return 0
	}
	i, j := 0, 0
	for ; i<len(nums) -1; i++ {
		for j < len(nums) && nums[i] == nums[j] {
			j++
		}
		if j == len(nums) {
			break
		}
		nums[i+1] = nums[j]
	}
	return i + 1
}
```
题解代码：
```js
func removeDuplicates(nums []int) int {
	if len(nums) == 0 {
		return 0
	}
	i, j := 0, 1
	for ; j < len(nums); j++ {
		if nums[i] != nums[j] {
			i++
			nums[i] = nums[j]
		}
	}
	return i + 1
}
```
不过有趣的是上面我写的代码平均执行时间是8ms左右，下面这个代码要18ms左右，不太了解编译器对上面两种代码是如何进行优化的，后面有机会可以深入了解下。
# 27-Remove Elements
给定一个数组 nums 和一个值 val，你需要原地移除所有数值等于 val 的元素，返回移除后数组的新长度。

不要使用额外的数组空间，你必须在原地修改输入数组并在使用 O(1) 额外空间的条件下完成。

元素的顺序可以改变。你不需要考虑数组中超出新长度后面的元素。

示例 1:

给定 nums = [3,2,2,3](), val = 3,

函数应该返回新的长度 2, 并且 nums 中的前两个元素均为 2。

你不需要考虑数组中超出新长度后面的元素。
示例 2:

给定 nums = [0,1,2,2,3,0,4,2](), val = 2,

函数应该返回新的长度 5, 并且 nums 中的前五个元素为 0, 1, 3, 0, 4。

注意这五个元素可为任意顺序。

你不需要考虑数组中超出新长度后面的元素。

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/remove-element
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

类似地，第27题基本也是相同的思路，只不过快指针指向指向数组尾部，当头指针发现元素等于目标值时，将其与尾指针指向元素交换，代码：
```js
func removeElement(nums []int, val int) int {
	if len(nums) == 0 {
		return 0
	}
	i, j := 0, len(nums) - 1
	for i <= j {
		if nums[i] == val {
			nums[i], nums[j] = nums[j], nums[i]
			j--
		} else {
			i++
		}
	}
	return j + 1
}
```

