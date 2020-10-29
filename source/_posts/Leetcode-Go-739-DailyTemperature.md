---
title: Leetcode-Go-739-DailyTemperature
date: 2020-10-29 14:07:30
tags:
- 算法
- Golang
---
# 题目描述
> 请根据每日 气温 列表，重新生成一个列表。对应位置的输出为：要想观测到更高的气温，至少需要等待的天数。如果气温在这之后都不会升高，请在该位置用 0 来代替。
> 
> 例如，给定一个列表 temperatures = [73, 74, 75, 71, 69, 72, 76, 73]()，你的输出应该是 [1, 1, 4, 2, 1, 1, 0, 0]()。
> 
> 提示：气温 列表长度的范围是 [1, 30000]()。每个气温的值的均为华氏度，都是在 [30, 100]() 范围内的整数。
> 
> 来源：力扣（LeetCode）
> 链接：https://leetcode-cn.com/problems/daily-temperatures
> 著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。
# 解法
题目可归纳为如下问题：
对于i,j(I\<j)，寻找min(j-i)，且满足T[I]\<T[j]
## 思路一
既然温度满足30-100这个条件，我们可以构造一个大小为100的数组，索引温度下存储该温度对应的最小天数（原数组的最小索引）。遍历数组时，寻找大于当前温度的第一个存在的索引取差值即为所求。
时间复杂度：O(mn) m在此题中为100。
空间复杂度：O(m) 额外的数组开销。

## 思路二
维护一个存储数组下标的单调递减栈，遍历数组时，当新元素大于栈顶元素时，说明栈顶元素找到了离它最近的温度更高的下标，弹出栈顶元素并对结果数组赋值；若新元素小于等于栈顶元素，直接将新元素入栈即可。
时间复杂度：O(n)
空间复杂度：O(n)
代码：
```js
func dailyTemperatures(T []int) []int {
	ret := make([]int, len(T))
	var stack []int
	for i, v := range T {
		for len(stack) > 0 && v > T[stack[len(stack)-1]] {
			stackPopIndices := stack[len(stack)-1]
			ret[stackPopIndices] = i - stackPopIndices
			stack = stack[:len(stack)-1]
		}
		stack = append(stack, i)
	}
	return ret
}
```
# 思考
这是一道很典型的利用单调栈特性的题目，感觉自己每次遇到单调栈题目时总是会卡住没有思路，留一个todo：总结一下单调栈和它的应用场景。

