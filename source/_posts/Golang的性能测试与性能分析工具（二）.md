---
title: Golang的性能测试与性能分析工具（二）
date: 2020-06-01 15:56:23
tags:
- Golang
---
对pprof做个简单总结和演示
1. 可通过HTTP请求的方式，导入"net/http/pprof”，访问http://ip:port/debug/pprof/
2. 命令行方式，以cpu性能测试为例 go tool pprof http://ip:port/debug/pprof/profile?seconds=10，10s为单位进行采样。
3. 可以通过代码嵌入的方式对代码块进行详细分析。
```js
fi, err := os.Create("cpu.prof")
if err != nil {
// handle error
}
pprof.StartCPUProfile(fi)
// your code 
pprof.StopCPUProfile()
```
4. 分析.prof文件的命令 go tool pprof prof filename.prof
5. go test也支持profile输出性能分析文件 go test -bench=flag -cpuprofile=file.prof

# 示例
## HTTP请求采样
用简单的make slice方式来进行一下性能分析，代码如下
```js
import (
	"fmt"
	"github.com/julienschmidt/httprouter"
	"log"
	"net/http"
	_ "net/http/pprof"
)

func Index(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Welcome!\n")
}

func LoopSlice(w http.ResponseWriter, r *http.Request) {
	var fbs []int
	for i := 0; i < 1000000; i++ {
		fbs = MakeSlice(100)
	}
	w.Write([]byte(fmt.Sprintf("%v", fbs)))

}

func MakeSlice(n int) []int {
	var ret []int
	for i := 0; i < n; i++ {
		ret = append(ret, i)
	}
	return ret
}


func main() {
	http.HandleFunc("/", Index)
	http.HandleFunc("/loop", LoopSlice)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```
启动Server后，我们运行
```js
go tool pprof http://localhost:8080/debug/pprof/profile\?seconds\=10
```
进行采样，简单对localhost:8080/loop进行几次模拟请求，得到采样结果：
![][image-1]
可以看到cpu时间集中在append上（slice扩容）
## Benchmark输出输出prof结果
我们换一种方式，采用benchmark输出prof来进行性能分析，
```js
import "testing"

func BenchmarkRequest(b *testing.B) {
	b.ResetTimer()
	var _ []int
	for i := 0; i < b.N; i++ {
		_ = MakeSlice(100000)
	}
	b.StopTimer()
}

func MakeSlices(n int) []int {
	var slice []int
	for i := 0; i < n; i++ {
		slice = append(slice, i)
	}
	return slice
}
```
运行 go test -bench=. -memprofile=mem.prof 将结果输出到men.prof文件中，用 go tool pprof mem.prof 分析 mem.prof
￼![][image-2]
可以看到append对于内存的消耗
## 一个有趣的问题
我们对Go中的RWMutex（读写锁）的读锁进行性能测试，代码如下
```js
func init() {
	m = make(map[string]string)
	m["a"] = "a"
	m["b"] = "b"
}

func LockFreeAccess() {
	wg := sync.WaitGroup{}
	wg.Add(NumberReaders)
	for i := 0; i < NumberReaders; i++ {
		go func() {
			for j := 0; j < ReadTimes; j++ {
				_, ok := m["a"]
				if !ok {

				}
			}
			wg.Done()
		}()
	}
	wg.Wait()
}

func LockAccess() {
	wg := sync.WaitGroup{}
	wg.Add(NumberReaders)
	rwLock := sync.RWMutex{}
	for i := 0; i < NumberReaders; i++ {
		go func() {
			for j := 0; j < ReadTimes; j++ {
				rwLock.RLock()
				_, ok := m["a"]
				if !ok {

				}
				rwLock.RUnlock()
			}
			wg.Done()
		}()
	}
	wg.Wait()
}

func BenchmarkLockFree(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		LockFreeAccess()
	}
}

func BenchmarkLock(b *testing.B) {
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		LockAccess()
	}
}
```
首先进行Benchmark，得到的结果
![][image-3]
即使读锁之前并不存在互斥，加了读锁的多协程并发性能并不好，将bench的cpu测试输出到cpu.prof，分析结果如下
![][image-4]
RWMutex的Rlock和Runlock占用的cpu时间极长。因此，读锁的不互斥并不代表着加了读锁之后的并发读性能会不受影响。

[image-1]:	https://myblog-1259548259.cos.ap-beijing.myqcloud.com/pprof_cpu_sample.png
[image-2]:	https://myblog-1259548259.cos.ap-beijing.myqcloud.com/pprof_mem_sample.png
[image-3]:	https://myblog-1259548259.cos.ap-beijing.myqcloud.com/rwmutex_bench.png
[image-4]:	https://myblog-1259548259.cos.ap-beijing.myqcloud.com/rwmutex_cpuprof.png