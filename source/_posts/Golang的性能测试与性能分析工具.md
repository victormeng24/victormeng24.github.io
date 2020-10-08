---
title: Golang的性能测试与性能分析工具（一）
date: 2020-06-01 13:24:44
tags:
- Golang
---
# Benchmark
```js
func BenchmarkStrAdd(b *testing.B)
```
被称为golang的基准测试，定义了以Benchmark开头的function后，通过”go test -bench FLAG”命令
多个基准测试将按照顺序进行执行。
testing.B中封装了Benchmark中常用的方法调用。
## ResetTimer
```js
// ResetTimer zeroes the elapsed benchmark time and memory allocation counters
// and deletes user-reported metrics.
// It does not affect whether the timer is running.
func (b *B) ResetTimer()
```
## StopTimer
```js
// StopTimer stops timing a test. This can be used to pause the timer
// while performing complex initialization that you don't
// want to measure.
func (b *B) StopTimer()
```
## ReportAllocs
```js
// ReportAllocs enables malloc statistics for this benchmark.
// It is equivalent to setting -test.benchmem, but it only affects the
// benchmark function that calls ReportAllocs.
func (b *B) ReportAllocs()
```
## Example
以字符串拼接为例，我们对比StringConcat、BytesBuffer和StringBuilder字符串拼接的性能，代码：
```js
import (
	"bytes"
	"strconv"
	"strings"
	"testing"
)

func BenchmarkStrAdd(b *testing.B) {
	b.ResetTimer()
	for j := 0; j < b.N; j++ {
		ret := ""
		for i := 0; i < 100; i++ {
			ret += strconv.Itoa(i)
		}
	}
	b.ReportAllocs()
	b.StopTimer()
}

func BenchmarkByteBuffer(b *testing.B) {
	b.ResetTimer()
	for j := 0; j < b.N; j++ {
		ret := bytes.Buffer{}
		for i := 0; i < 100; i++ {
			ret.WriteString(strconv.Itoa(i))
		}
		_ = ret.String()
	}
	b.StopTimer()
}

func BenchmarkStringBuilder(b *testing.B) {
	b.ResetTimer()
	for j := 0; j < b.N; j++ {
		ret := strings.Builder{}
		for i := 0; i < 100; i++ {
			ret.WriteString(strconv.Itoa(i))
		}
		_ = ret.String()
	}
	b.StopTimer()
}
```
运行代码 go test -bench . 结果如下
![][image-1]
可以看到StringBuilder的性能略好于BytesBuffer，两者远好于StringConcat，我们还可以通过-benchmem查看内存的Benchmark
![][image-2]

[image-1]:	https://myblog-1259548259.cos.ap-beijing.myqcloud.com/go_benchmark.png
[image-2]:	https://myblog-1259548259.cos.ap-beijing.myqcloud.com/go_benchmark_mem.png