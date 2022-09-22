---
title: Go语言学习-快速入门
date: 2021-11-27 15:29:29
tags:
- go
- basic-grammar
categories: go
---

## 简介
最近觉得Go语言很有前途，语法灵活，容器化部署方便，资源消耗小。
而且还有强力的并发能力：Goroutines
所以决定从今天开始，学习Go语言。

这一次我们将学习到：
- Golang基本概念
- 基本数据类型
- 基本语法
- 复杂类型
- 方法与协程

我们现在开始吧！
<!--more-->

## Golang基本概念
Go， 他是一款Google开发的语言。
他是静态的
他是编译形语言
他拥有垃圾回收器
他有强大的并发机制

在容器越来越流行的今天，Go语言拥有着非常大的潜力。

## 基本数据类型
我们使用`var`来声明一个变量，用`const`来声明一个常量。
当然还有一个简便的写法，使用`:`

### int类型
```go
const (
		i1       = 1
		i2       = 1000
		i3 int8  = 127
		i4 int16 = 128
		i5 uint  = 257
		i6 int   = 0b101
		i7       = 0o77
		i8       = 0xAF
	)
	fmt.Println(i1, i2, i3, i4, i5, i6, i7, i8)
```
上面的代码，展示了不同类型的int类型。大家酌情使用。

### float类型
```go
	var (
		f1         = 0.1
		f2 float64 = 0.0001
	)
	fmt.Println(f1, f2)
```
float类型有两种，对应Java中的Float和Double。

### byte类型
```go
	var (
		c1 byte
		c2 = 'c'
		c3 = '鹏'
	)
	fmt.Printf("\"%v, %T; %v, %T; %v, %T \n", c1, c1, c2, c2, c3, c3)

	c4 := 'a'
	c5 := 'A'
	c6 := 'x' + c5 - c4
	fmt.Println(c6, string(c6))
```
为了节约内存，go使用了byte来做string的底层，所以中文可能会被截断。
Java在9之后，String的底层也从char变成了byte。

### boolean
```go
var b1 bool = true
var b2 = false
b3 := b1 && b2
fmt.Println(b1, b2, b3)
```

### string
```go
	s1 := "hello"
	s2 := "世界"
	fmt.Println(s1 + s2)
	fmt.Println(len(s1), len(s2))
```

## 基本语法

### 变量与常量
#### 变量声明的几种方式
```go
	var v1 int
	v1 = 1
	var v2 int = 2
	var v3 = 3
	v4 := 4
	fmt.Println(v1, v2, v3, v4)

	var (
		v5 = 5
		v6 = 6
	)
	fmt.Println(v5, v6)
	fmt.Println(globalVariable)
```
变量不会定义成特定的类型，他会进行类型推断。
还有比较有特色的声明方式是使用`:=`
#### 常量声明
```go
	const c1 = 1 // 1
	const (
		c2 = 2    //2
		c3 = iota //1 当前行数（从0开始）
		c4 = iota //2
		c5        //3 默认值为上一行
		c6        //4
		c7 = 7    //7
		c8        //7 默认值为上一行
		c9        //7
	)
	fmt.Println(c1, c2, c3, c4, c5, c6, c7, c8, c9)
	fmt.Println(globalConstant)
```
常量充当的是Java中final关键字的功能，还有枚举的功能。

### 执行顺序
我们通过`if else`， `for loop` 等来控制程序的执行流程。

#### 条件
```go
	var s = rand.Int31n(100)
	fmt.Println(s)
	if s > 60 {
		fmt.Println("pass")
	} else {
		fmt.Println("fall")
	}
```
```go
	var s int32 = rand.Int31n(100)
	fmt.Println(s)
	switch {
	case s < 10:
		fmt.Println("太差了")
	case s < 60:
		fmt.Println("不及格")
	case s < 80:
		fallthrough
	case s < 100:
		fmt.Println("good")

	}
```
两种条件的方式，更多的是使用if。

#### 循环
```go
	i := 0
	for {
		if i >= 10 {
			break
		}
		fmt.Print(i)
		i++
	}

	fmt.Println()

	i = 0
	for i < 10 {
		fmt.Print(i)
		i++
	}

	fmt.Println()

	for i := 0; i < 10; i++ {
		fmt.Print(i)
	}
```
不想Java有三种循环方式`while`、`do while`、`for`，
Go只有`for`这一种方式，当时功能依旧强大，有类似python的`range`关键字。
当然，依旧有`break`和`continue`的支持

### init函数
init()函数会在当前go文件加载前加载，
他的加载顺序是：
被依赖包的全局变量 -> 被依赖包的init函数 -> 当前包的全局变量 -> 当前包的init函数 -> 当前包的函数

这个方法可以在程序一开始的时候，创建一些数据连接什么的。

使用方法如下：
```go
func init() {
	fmt.Println("this main init function")
}
```

### 异常与错误
每一种程序都会有一场，在go语言种，使用panic机制来表示异常。
恐慌，很形象。

比如抛出一个异常可以这样：
```go
panic(Error)
```

我们可以生成一个error，来让panic抛出。
```go
	var lovelyError = errors.New("this is a lovely error")
	var cuteError = fmt.Errorf("this is a cute error")
```

有了异常和抛出异常，当然会有捕获异常，
在go里面是使用`defer`（延迟）和 `recover`（恢复）来做的。
```go
func errorOperation() {
	defer func() {
		var e = recover()
		fmt.Println("recover:", e)
	}()
	var lovelyError = errors.New("this is a lovely error")
	var cuteError = fmt.Errorf("this is a cute error")
	fmt.Println(lovelyError, ";", cuteError)

	panic(cuteError)
}
```

### 函数与接口
#### 函数
go的函数声明类似Java的方法声明，但是又有所增强。

简单的函数声明
```go
func add(a int, b int) int {
	return a + b
}
```
返回两个变量
```go
func sumAndDiff(a int, b int) (int, int) {
	sum := a + b
	diff := a - b
	return sum, diff
}
```

匿名函数
```go
func anonymousFunc() {
	var f = func() {
		fmt.Println("I am anonymous function")
	}
	f()
}
```

闭包
```go
func incr() func() int {
    var x int
    return func() int {
        x++
        return x
    }
}
```

#### 接口
> 如果它看起来像鸭子、游泳像鸭子、叫声像鸭子，那么它可能就是只鸭子。

go的接口和接口的实现，参考的是这种思想。

我们先声明一个接口：
```go
type Message interface {
	getType() string
	send()
}
```

可以看到，这个接口，定义了两个函数。

再来声明一个struct来实现这个接口
```go
// TextMsg obj
type TextMsg struct {
	text string
}

func (tm *TextMsg) getType() string {
	return "text"
}

func (tm *TextMsg) send() {
	fmt.Println("sendText -> ", tm.text, "; type is ", tm.getType())
}

```
大家可以看到， TextMsg并没有直接声明 implement Message接口
但是 TextMsg 绑定了Message接口的两个函数，当然，还可以绑定自己的函数。

这样的话，程序就会认为，TextMsg就是Message的实现结构体。


## 复杂类型

### array
和Java中的数组一样，需要提前定义好数组的容量。
直接分配内存，效率高。

使用方式如下：
```go
	var array [3]int = [3]int{1, 2, 3}
	fmt.Println(array[1], len(array))

	var array1 = [...]int{4,
		5,
		6}
	fmt.Println(array1[0])

	for i, v := range array1 {
		fmt.Println(i, "的值是", v)
	}
```

二维数组：
```go
	var points = [3][2]int{{1, 1}, {2, 2}, {3, 3}}
	for _, point := range points {
		fmt.Println(point)
	}
```

### slice
切片是Go中使用的最多的列表，类似Java中的ArrayList。

很多种方法可以创建一个切片。
```go
func createSlice() {
	// from array
	array := [5]int{1, 2, 3, 4, 5}
	var s1 = array[0 : len(array)-1]
	fmt.Println(s1)

	// from slice
	var s2 = s1[1:3]
	fmt.Println(s2)

	// make
	s3 := make([]int, 0)
	fmt.Println(s3)
	s3 = append(s3, 3)
	fmt.Println(s3)

	// sugar
	s4 := []int{4}
	fmt.Println(s4)
}
```

他是一种动态数组，支持自动扩容，底层当然是array。
可以参考Java中ArrayList的实现和扩容方式。
他有着长度和容量两个数值。
```go
func lenAndCap() {
	var slice = make([]int, 8, 10)
	fmt.Println(slice, len(slice), cap(slice))
}
```

当然还支持着拼接和copy
```go
func appendSlice() {
	var s1 = []int{1, 2, 3}
	fmt.Println(s1, len(s1), cap(s1))
	s1 = append(s1, 4)
	fmt.Println(s1, len(s1), cap(s1))
	s1 = append(s1, []int{5, 6, 7}...)
	fmt.Println(s1, len(s1), cap(s1))
}

func copySlice() {
	var s1 = []int{1, 2, 3, 4, 5}
	var s2 = []int{6, 7}
	fmt.Println(s1, s2)
	copy(s1, s2)
	fmt.Println(s1, s2)
}
```

在go中没有栈和队列的数据结构，一般都是用slice来模拟的。

### map
go中的map类似与Java中的HashMap，基本操作如下：
```go
func createMap() {
	var m1 = make(map[string]string)
	m1["morning"] = "eat breakfast"
	m1["noon"] = "have lunch"

	var m2 = map[string]string{
		"evening": "get dinner",
	}

	fmt.Println(m1, m2)

	fmt.Println(m1["noom"], m2["evening"])

}

func findDeleteRange() {
	var m1 = make(map[string]string)
	m1["morning"] = "eat breakfast"
	m1["noon"] = "have lunch"

	var v, ok = m1["noon"]
	fmt.Println(v, ok)

	delete(m1, "noon")

	for key, value := range m1 {
		fmt.Println(key, value)
	}
}
```

## 方法与协程

### goroutine
Go语言的核心之一，最最重要的特点，就是goroutine。
就是Go语言原生帮我实现的协程，又成用户线程。
可以非常快速和方便的实现异步操作。

使用方法非常简单：
```go
go function()
```
这样就会表示，新开一个协程来调用`function()`函数。

在Java中需要这样写：
```java
new Thread(() -> function()).start();
```
当然还可以使用协程池来开启一个新线程。

但是不管那种方式，不管是在使用方便的程度，还是程序的运行效率。
Go 的 goroutine都更加优异。

```go
func test() {
	go doWork1()
	go doWork2()
	fmt.Println("END")
}

func doWork2() {
	fmt.Println("WORK-2")
}

func doWork1() {
	fmt.Println("WORK-1")
}
```
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/2Ml3uc.png)

类似这个图，`END`,`WORK-1`,`WORK-2`的顺序是没办法保证的。

但是在实际的操作中，我们可能看不到`WORK-1`,`WORK-2`，
因为在协程执行完之前，整个程序的进程就结束了。

为了解决这个这个问题，我们需要`wait group`来同步结果。

### wait group
我们使用WG来保证所有协程都能够结束
```go
func test() {
	var wg sync.WaitGroup
	wg.Add(2)
	go doWork1(&wg)
	go doWork2(&wg)
	wg.Wait()
	fmt.Println("END")
}

func doWork2(wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Println("WORK-2")
}

func doWork1(wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Println("WORK-1")
}
```
使用`WaitGroup`，可以保证，在`END`之前，保证`WORK-1`,`WORK-2`执行完毕。
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/OxRa4Q.png)

### channel
channel类似一个管道，可以又缓存，可以往里面输入东西，可以从里面取出东西。
在Java中，有个很类似的东西，BlockingQueue。
两者都具备阻塞队列的功能。

但是他们核心不同点是，Go里面的channel使用的[CSP](http://www.usingcsp.com/cspbook.pdf)模型。
channel的使用也更简洁，还支持select语法，和定向channel。

使用方法如下：
```go
func test() {
	var wg sync.WaitGroup
	wg.Add(2)
	var c = make(chan string)
	go doWork1(&wg, &c)
	go doWork2(&wg, &c)
	wg.Wait()
	fmt.Println("END")
}

func doWork2(wg *sync.WaitGroup, c *chan string) {
	defer wg.Done()
	m := <-*c
	fmt.Println("WORK-2")
	fmt.Println(m)
}

func doWork1(wg *sync.WaitGroup, c *chan string) {
	defer wg.Done()
	fmt.Println("WORK-1")
	*c <- "test message"
}
```
输出的结果如下：
```
WORK-1
WORK-2
test message
END
```
我们可以使用channel来把信息跨协程传输，
我们还可以保证`WORK-1`在`WORK-2`之前被打印。
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/0peR0n.png)

### context
上下文，我们可以通过它，来控制协程的活动，比如中断一个协程。
我们来看段代码：
```go
func test() {

	// wait group
	var wg sync.WaitGroup
	wg.Add(2)

	// channel
	var c = make(chan string)

	// context
	ctx, cancel := context.WithCancel(context.Background())

	go doWork1(&wg, &c, ctx)
	go doWork2(&wg, &c, ctx)

	time.Sleep(5 * time.Second)
	// 中断
	cancel()

	wg.Wait()
	fmt.Println("END")
}

func doWork2(wg *sync.WaitGroup, c *chan string, ctx context.Context) {
	defer wg.Done()
	m := <-*c
	fmt.Println("WORK-2")
	fmt.Println(m)
}

func doWork1(wg *sync.WaitGroup, c *chan string, ctx context.Context) {
	defer wg.Done()
	fmt.Println("WORK-1")
	*c <- "test message"

	for i := 0; i < 100; i++ {
		time.Sleep(time.Second)
		fmt.Println("WORK-1", i)
		select {
		case <-ctx.Done():
			fmt.Println("WORK-1 DONE")
			return
		default:

		}
	}
}

```

在work1里面 我们有段逻辑，是打印 0～99，但是如果主线程通知我们中断，我们就会中断当前操作。

我们就会打印出这样的结果。
```
WORK-1
WORK-2
test message
WORK-1 0
WORK-1 1
WORK-1 2
WORK-1 3
WORK-1 4
WORK-1 DONE
END
```
我们的示意图，就会变成这样：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/rWNJKX.png)

我们除了`func WithCancel(parent Context) (ctx Context, cancel CancelFunc)`之外，
还有
```go
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

func WithValue(parent Context, key interface{}, val interface{}) (Context)

```

大家如果感兴趣，可以自己尝试一下。

## 总结
这次，我们快速入门了Go，这是一种非常新的语言。
我相信它会在不久的未来会大展宏图。在市场上有自己的一席之地。

希望这片文章对大家有所帮助～

Bye！

## 参考资料
- https://go.dev/
- https://github.com/CPyeah/hello-go/tree/master/grammar