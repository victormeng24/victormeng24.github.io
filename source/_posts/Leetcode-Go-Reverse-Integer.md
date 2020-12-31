---
title: Leetcode-Go-7-Reverse-Integer
date: 2020-02-26 13:40:02
tags:
- 算法
- Golang
---
> 给出一个 32 位的有符号整数，你需要将这个整数中每位上的数字进行反转。
> 
> 示例 1:
> 
> 输入: 123
> 输出: 321
>  示例 2:
> 
> 输入: -123
> 输出: -321
> 示例 3:
> 
> 输入: 120
> 输出: 21
> 注意:
> 
> 假设我们的环境只能存储得下 32 位的有符号整数，则其数值范围为 [−231,  231 − 1]()。请根据这个假设，如果反转后整数溢出那么就返回 0。
> 
> 来源：力扣（LeetCode）
> 链接：https://leetcode-cn.com/problems/reverse-integer
> 著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

题目很简单，唯一需要注意的地方是边界值问题，int类型的数据在反转时可能溢出，借本道题了解下Go的testing包，上代码
```js
func reverse(x int) int {
	y := 0
	for x != 0 {
		y = y * 10 + x % 10
		x = x / 10
	}
	if y > 1 << 31 -1  || y < -(1 << 31) {
		return 0
	}
	return y
}
```

```js
import (
	"fmt"
	"testing"
)

type question7 struct {
	para7
	ans7
}

// para 是参数
// one 代表第一个参数
type para7 struct {
	one int
}

// ans 是答案
// one 代表第一个答案
type ans7 struct {
	one int
}

func Test_Problem7(t *testing.T) {

	qs := []question7{

		question7{
			para7{321},
			ans7{123},
		},

		question7{
			para7{-123},
			ans7{-321},
		},

		question7{
			para7{120},
			ans7{21},
		},

		question7{
			para7{1534236469},
			ans7{0},
		},
	}

	fmt.Printf("------------------------Leetcode Problem 7------------------------\n")

	for _, q := range qs {
		ans, p := q.ans7, q.para7
		fmt.Printf("【input】:%v    【output】:%v    [expectoutput]:%v\n", p.one, reverse(p.one), ans.one)
	}
	fmt.Printf("\n\n\n")
}
```
testing参考的git上这个项目：[https://github.com/halfrost/LeetCode-Go][2]，对Go和力扣有兴趣的也可以看这个项目，带解析和测试用例。
目前对Go的测试有些简单了解：
- 文件名以\_test.go结尾\_
- 引入testing包，是Go的基础测试工具
- 运行go test，编译执行当前目录下所有test.go测试文件
- 函数名以Test开头，如果是benchmark以Benchmark开头

[2]:	https://github.com/halfrost/LeetCode-Go