## 数组(array)
#### 声明数组和声明切片的不同
```
// 创建有3 个元素的整型数组
array := [3]int{10, 20, 30}
// 创建长度和容量都是3 的整型切片
slice := []int{10, 20, 30}
```
***
## 切片(slice)
#### 如何计算长度和容量
```
对底层数组容量是k 的切片slice[i:j]来说
长度: j - i
容量: k - i
```
#### 修改切片内容可能导致的结果
```
// 创建一个整型切片
// 其长度和容量都是5 个元素
slice := []int{10, 20, 30, 40, 50}
// 创建一个新切片
// 其长度是2 个元素，容量是4 个元素
newSlice := slice[1:3]
// 修改newSlice 索引为1 的元素
// 同时也修改了原来的slice 的索引为2 的元素
newSlice[1] = 35
```
#### 使用3 个索引创建切片
```
// 将第三个元素切片，并限制容量
// 其长度为1 个元素，容量为2 个元素
slice := source[2:3:4]
```
#### 如何计算长度和容量
```
对于 slice[i:j:k] 或 [2:3:4]
长度: j – i 或3 - 2 = 1
容量: k – i 或4 - 2 = 2
```
***
## 映射(map)
#### 对nil 映射赋值时的语言运行时错误
```
// 通过声明映射创建一个nil 映射
var colors map[string]string
// 将Red 的代码加入到映射
colors["Red"] = "#da1337"
Runtime Error:
panic: runtime error: assignment to entry in nil map
```
***
## 类型
#### golang.org/src/os/file_unix.go：第15行到第29行
```
15 // File 表示一个打开的文件描述符
16 type File struct {
17 *file
18 }
19
20 // file 是*File 的实际表示
21 // 额外的一层结构保证没有哪个os 的客户端
22 // 能够覆盖这些数据。如果覆盖这些数据，
23 // 可能在变量终结时关闭错误的文件描述符
24 type file struct {
25 fd int
26 name string
27 dirinfo *dirInfo // 除了目录结构，此字段为nil
28 nepipe int32 // Write 操作时遇到连续EPIPE 的次数
29 }
```
> 可以在代码清单5-31 里看到标准库中声明的File 类型。这个类型的本质是非原始的。这个类型的值实际上不能安全复制。对内部未公开的类型的注释，解释了不安全的原因。因为没有方法阻止程序员进行复制，所以File 类型的实现使用了一个嵌入的指针，指向一个未公开的类型。本章后面会继续探讨内嵌类型。正是这层额外的内嵌类型阻止了复制。不是所有的结构类型都需要或者应该实现类似的额外保护。程序员需要能识别出每个类型的本质，并使用这个本质来决定如何组织类型。
#### 规范里描述的方法集
Values|Methods Receivers
:--:|---
T  |(t T)
*T |(t T) and (t *T)
> T 类型的值的方法集只包含值接收者声明的方法。而指向T 类型的指针的方法集既包含值接收者声明的方法，也包含指针接收者声明的方法。
#### 从接收者类型的角度来看方法集
Methods Receivers|Values
:--:|---
(t T)  |  T and *T
(t *T)  |  *T
> 如果使用指针接收者来实现一个接口，那么只有指向那个类型的指针才能够实现对应的接口。如果使用值接收者来实现一个接口，那么那个类型的值和指针都能够实现对应的接口。
#### 公开或未公开的标识符
> 当一个标识符的名字以小写字母开头时，这个标识符就是未公开的，即包外的代码不可见。如果一个标识符以大写字母开头，这个标识符就是公开的，即被包外的代码可见。
>
> 要让这个行为可行，需要两个理由。第一，公开或者未公开的标识符，不是一个值。第二，短变量声明操作符，有能力捕获引用的类型，并创建一个未公开的类型的变量。永远不能显式创建一个未公开的类型的变量，不过短变量声明操作符可以这么做。
***
## 并发
#### 如何修改逻辑处理器的数量
```
import "runtime"
// 给每个可用的核心分配一个逻辑处理器
runtime.GOMAXPROCS(runtime.NumCPU())
```
> 调度器对可以创建的逻辑处理器的数量没有限制，但语言运行时默认限制每个程序最多创建10 000 个线程。这个限制值可以通过调用runtime/debug 包的SetMaxThreads 方法来更改。如果程序试图使用更多的线程，就会崩溃。
#### 用竞争检测器来编译并执行代码
```
go build -race // 用竞争检测器标志来编译程序
./example // 运行程序
==================
WARNING: DATA RACE
Write by goroutine 5:
main.incCounter()
/example/main.go:49 +0x96
Previous read by goroutine 6:
main.incCounter()
/example/main.go:40 +0x66
Goroutine 5 (running) created at:
main.main()
/example/main.go:25 +0x5c
Goroutine 6 (running) created at:
main.main()
/example/main.go:26 +0x73
==================
Final Counter: 2
Found 1 data race(s)
```
#### 锁住共享资源
1. 原子函数
> atmoic 包的AddInt64 函数

> 另外两个有用的原子函数是LoadInt64 和StoreInt64
```
01 // 这个示例程序展示如何使用atomic 包来提供
02 // 对数值类型的安全访问
03 package main
04
05 import (
06 "fmt"
07 "runtime"
08 "sync"
09 "sync/atomic"
10 )
11
12 var (
13 // counter 是所有goroutine 都要增加其值的变量
14 counter int64
15
16 // wg 用来等待程序结束
17 wg sync.WaitGroup
18 )
19
20 // main 是所有Go 程序的入口
21 func main() {
22 // 计数加 2，表示要等待两个goroutine
23 wg.Add(2)
24
25 // 创建两个goroutine
26 go incCounter(1)
27 go incCounter(2)
28
29 // 等待 goroutine 结束
30 wg.Wait()
31
32 // 显示最终的值
33 fmt.Println("Final Counter:", counter)
34 }
35
36 // incCounter 增加包里counter 变量的值
37 func incCounter(id int) {
38 // 在函数退出时调用Done 来通知main 函数工作已经完成
39 defer wg.Done()
40
41 for count := 0; count < 2; count++ {
42 // 安全地对counter 加1
43 atomic.AddInt64(&counter, 1)
44
45 // 当前 goroutine 从线程退出，并放回到队列
46 runtime.Gosched()
47 }
48 }
```
2. 互斥锁(mutex)
```
01 // 这个示例程序展示如何使用互斥锁来
02 // 定义一段需要同步访问的代码临界区
03 // 资源的同步访问
04 package main
05
06 import (
07 "fmt"
08 "runtime"
09 "sync"
10 )
11
12 var (
13 // counter 是所有goroutine 都要增加其值的变量
14 counter int
15
16 // wg 用来等待程序结束
17 wg sync.WaitGroup
18
19 // mutex 用来定义一段代码临界区
20 mutex sync.Mutex
21 )
22
23 // main 是所有Go 程序的入口
24 func main() {
25 // 计数加 2，表示要等待两个goroutine
26 wg.Add(2)
27
28 // 创建两个goroutine
29 go incCounter(1)
30 go incCounter(2)
31
32 // 等待 goroutine 结束
33 wg.Wait()
34 fmt.Printf("Final Counter: %d\\n", counter)
35 }
36
37 // incCounter 使用互斥锁来同步并保证安全访问，
38 // 增加包里counter 变量的值
39 func incCounter(id int) {
40 // 在函数退出时调用Done 来通知main 函数工作已经完成
41 defer wg.Done()
42
43 for count := 0; count < 2; count++ {
44 // 同一时刻只允许一个goroutine 进入
45 // 这个临界区
46 mutex.Lock()
47 {
48 // 捕获 counter 的值
49 value := counter
50
51 // 当前 goroutine 从线程退出，并放回到队列
52 runtime.Gosched()
53
54 // 增加本地 value 变量的值
55 value++
56
57 // 将该值保存回counter
58 counter = value
59 }
60 mutex.Unlock()
61 // 释放锁，允许其他正在等待的goroutine
62 // 进入临界区
63 }
64 }
```
