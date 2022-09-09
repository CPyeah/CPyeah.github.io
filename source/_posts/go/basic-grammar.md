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
我们使用`var`来声明一个变量，yong`const`来声明一个常量。
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

### 异常与错误

### 函数与接口

## 复杂类型

### array

### slice

### map

### struct

### 自定义类型

## 方法与协程

### goroutine

### channel

### wait group

## 总结

## 参考资料
- https://go.dev/
- https://www.bilibili.com/read/readlist/rl496566