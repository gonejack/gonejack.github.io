---
title: golang 逃逸分析与栈、堆分配分析
categories:
  - 开发
tags:
  - golang
  - 内存管理
date: 2020-12-01 13:19:18
---
我们在写 golang 代码时候定义变量，那么一个很常见的问题，申请的变量保存在哪里呢？栈？还是堆？会不会有一些特殊例子？这篇文章我们就来探索下具体的case以及如何做分析。

还是从实际使用场景出发：

## Question

```go
package main

type User struct {
	ID     int64
	Name   string
	Avatar string
}

func GetUserInfo() *User {
	return &User{
		ID: 666666,
		Name: "sim lou",
		Avatar: "https://www.baidu.com/avatar/666666",
	}
}

func main()  {
	u := GetUserInfo()
	println(u.Name)
}
```

这里GetUserInfo 函数里面的 User 对象是存储在函数栈上还是堆上？

## 什么是堆？什么是栈？

简单说：

* 堆：一般来讲是人为手动进行管理，手动申请、分配、释放。一般所涉及的内存大小并不定，一般会存放较大的对象。另外其分配相对慢，涉及到的指令动作也相对多
* 栈：由编译器进行管理，自动申请、分配、释放。一般不会太大，我们常见的函数参数（不同平台允许存放的数量不同），局部变量等等都会存放在栈上

今天我们介绍的 Go 语言，它的堆栈分配是通过 Compiler 进行分析，GC 去管理的，而对其的分析选择动作就是今天探讨的重点

## 逃逸分析

逃逸分析是一种确定指针动态范围的方法，简单来说就是分析在程序的哪些地方可以访问到该指针。

通俗地讲，逃逸分析就是确定一个变量要放堆上还是栈上，规则如下：

* 是否有在其他地方（非局部）被引用。只要有可能被引用了，那么它一定分配到堆上。否则分配到栈上
* 即使没有被外部引用，但对象过大，无法存放在栈区上。依然有可能分配到堆上

对此你可以理解为，逃逸分析是编译器用于决定变量分配到堆上还是栈上的一种行为。

## 在什么阶段确立逃逸

go 在编译阶段确立逃逸，注意并不是在运行时

## 为什么需要逃逸

其实就是为了尽可能在栈上分配内存，我们可以反过来想，如果变量都分配到堆上了会出现什么事情？例如：

* 垃圾回收（GC）的压力不断增大
* 申请、分配、回收内存的系统开销增大（相对于栈）
* 动态分配产生一定量的内存碎片

其实总的来说，就是频繁申请、分配堆内存是有一定 “代价” 的。会影响应用程序运行的效率，间接影响到整体系统。因此 “按需分配” 最大限度的灵活利用资源，才是正确的治理之道。这就是为什么需要逃逸分析的原因，你觉得呢？

## go怎么确定是否逃逸

可以看到详细的逃逸分析过程。而指令集 -gcflags 用于将标识参数传递给 Go 编译器，涉及如下：

* -m 会打印出逃逸分析的优化策略，实际上最多总共可以用 4 个 -m，但是信息量较大，一般用 1 个就可以了
* -l 会禁用函数内联，在这里禁用掉 inline 能更好的观察逃逸情况，减少干扰

`$ go build -gcflags '-m -l' main.go`


## 第二：反编译命令查看

`$ go tool compile -S main.go`

注：可以通过 go tool compile -help 查看所有允许传递给编译器的标识参数

### 实际案例

```go
package main

type User struct {
	ID     int64
	Name   string
	Avatar string
}

func GetUserInfo() *User {
	return &User{
		ID: 666666,
		Name: "sim lou",
		Avatar: "https://www.baidu.com/avatar/666666",
	}
}

func main()  {
	u := GetUserInfo()
	println(u.Name)
}
```

看编译器命令执行结果：

```bash
$go build -gcflags '-m -l' escape_analysis.go 
# command-line-arguments
./escape_analysis.go:13:11: &User literal escapes to heap
```

通过查看分析结果，可得知 &User 逃到了堆里，也就是分配到堆上了。这是不是有问题啊…再看看汇编代码确定一下，如下：

```bash
$go tool compile -S escape_analysis.go       

"".GetUserInfo STEXT size=190 args=0x8 locals=0x18
	0x0000 00000 (escape_analysis.go:9)	TEXT	"".GetUserInfo(SB), ABIInternal, $24-8
	......
	0x002c 00044 (escape_analysis.go:13)	CALL	runtime.newobject(SB)
	......
	0x0045 00069 (escape_analysis.go:12)	CMPL	runtime.writeBarrier(SB), $0
	0x004c 00076 (escape_analysis.go:12)	JNE	156
	0x004e 00078 (escape_analysis.go:12)	LEAQ	go.string."sim lou"(SB), CX
	......
	0x0061 00097 (escape_analysis.go:13)	CMPL	runtime.writeBarrier(SB), $0
	0x0068 00104 (escape_analysis.go:13)	JNE	132
	0x006a 00106 (escape_analysis.go:13)	LEAQ	go.string."https://www.baidu.com/avatar/666666"(SB), CX
	......
```

执行了 runtime.newobject 方法，也就是确实是分配到了堆上。这是为什么呢？这是因为 GetUserInfo() 返回的是指针对象，引用被返回到了方法之外了。因此编译器会把该对象分配到堆上，而不是栈上。否则方法结束之后，局部变量就被回收了，岂不是翻车。所以最终分配到堆上是理所当然的。

那么所有的指针都在堆上？也不是：

```go
func PrintStr()  {
	str := new(string)
	*str = "hello world"
}

func main()  {
	PrintStr()
}
```

看编译器逃逸分析的结果：

```bash
$go build -gcflags '-m -l' escape_analysis3.go             
# command-line-arguments
./escape_analysis3.go:4:12: PrintStr new(string) does not escape
```

看，该对象分配到栈上了。很核心的一点就是它有没有被作用域之外所引用，而这里作用域仍然保留在 main 中，因此它没有发生逃逸。

### 不确定类型

```go
func main()  {
	str := new(string)
	*str = "hello world"
	fmt.Println(*str)
}
```

执行命令观察一下，如下：

```
$go build -gcflags '-m -l' escape_analysis4.go
# command-line-arguments
./escape_analysis4.go:6:12: main new(string) does not escape
./escape_analysis4.go:8:13: main ... argument does not escape
./escape_analysis4.go:8:14: *str escapes to heap
```

通过查看分析结果，可得知 str 变量逃到了堆上，也就是该对象在堆上分配。但上个案例时它还在栈上，我们也就 fmt 输出了它而已。这…到底发生了什么事？

相对案例一，案例二只加了一行代码 fmt.Println(str)，问题肯定出在它身上。其原型：
`func Println(a ...interface{}) (n int, err error)`

通过对其分析，可得知当形参为 interface 类型时，在编译阶段编译器无法确定其具体的类型。因此会产生逃逸，最终分配到堆上。

如果你有兴趣追源码的话，可以看下内部的 reflect.TypeOf(arg).Kind() 语句，其会造成堆逃逸，而表象就是 interface 类型会导致该对象分配到堆上。

## 总结

* 静态分配到栈上，性能一定比动态分配到堆上好
* 底层分配到堆，还是栈。实际上对你来说是透明的，不需要过度关心
* 每个 Go 版本的逃逸分析都会有所不同（会改变，会优化）
* 直接通过 go build -gcflags ‘-m -l’ 就可以看到逃逸分析的过程和结果
* 到处都用指针传递并不一定是最好的，要用对。

[golang 逃逸分析与栈、堆分配分析_惜暮-CSDN博客_golang 堆和栈](https://blog.csdn.net/u010853261/article/details/102846449#_34)
