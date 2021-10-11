---
title: Go基本程序结构
tags: []
id: '45'
categories:
  - - Go
date: 2020-05-11 23:28:12
---

## 编写测试程序

测试程序：

- 源码文件以`_test`结尾：`x x x_test.go`

- 测试方法名以Test开头：`func TestXXX(t *testing.T) {...}`



```go
package test

import "testing"

func TestFirstTry(t *testing.T) {
	t.Log("My first try!")
}
```

实现Fibonacci数列

```go
package test

import (
	"testing"
)

func TestFibonacciList(t *testing.T) {
	//var a int = 1 // 定义变量
	//var b int = 1
	var (
		a int = 1
		b int = 1
	) // 这样也可以定义变量
	// a := 1 直接赋值
	// b := 1
	t.Log(a)
	for i := 0; i < 5; i++ {
		t.Log(" ", b)
		tmp := a
		a = b
		b = tmp + a
	}
}
```

## 变量及常量

变量

- 赋值可以进行自动类型推断
- 在一个赋值语句中可以对多个变量进行同时赋值

```go
func TestExchange(t *testing.T) {
	a := 1
	b := 2
	//tmp := a
	//a = b
	//b = tmp
	a, b = b, a // 变量交换
	t.Log(a, b)
}
```

常量 进行快速 设置连续值

```go
const (
	Monday = iota + 1
	Tuesday // 2
	Wednesday // 3
	Thursday // 4
	Friday // 5
	Saturday // 6
	Sunday // 7
)

const (
	Readable = 1 << iota
	Writable
	Executable
)

func TestConstantTry(t *testing.T) {
	t.Log(Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday) // 1 2 3 4 5 6 7
}

func TestConstantTry1(t *testing.T) {
	a := 7 // 1111
	t.Log(a&Readable == Readable, a&Writable == Writable, a&Executable == Executable) // true true true
}
```

## 数据类型

所有类型：

- `bool`

- `string`

- `int` `int8` `int16` `int32` `int64`

- `uint` `uint8`  `uint16` `uint32` `uint64` `uintptr`

- `byte`  // alias for uint8

- `rune` // alias for int32,represents a Unicode code point

- `float32` `float64`
- `Complex64` `complex128`



与其它主要编程语言的差异：

- Go语言不允许隐式类型转换

- 别名和原有类型也不能进行隐式类型转换



类型的预定义值：

- `math.MaxInt64`
- `math.MaxFloat64`
- `math.MaxUint32`



指针类型：

- 不支持指针运算
- `string`是值类型，其默认的初始化值为空字符串，而不是`nil`

## 运算符

### 算术运算符

| 运算符 | 描述 |
| :----: | :--: |
|   +    | 相加 |
|   -    | 相减 |
|   *    | 相乘 |
|   /    | 相除 |
|   %    | 求余 |
|   ++   | 自增 |
|   --   | 自减 |

Go语言没有前置的 ++和--

### 比较运算符

| 运算符 |             描述             |
| :----: | :--------------------------: |
|   ==   |      检查两个值是否相等      |
|   !=   |     检查两个值是否不相等     |
|   >    |   检查左边值是否大于右边值   |
|   <    |   检查左边值是否小于右边值   |
|   >=   | 检查左边值是否大于等于右边值 |
|   <=   | 检查左边值是否小于等于右边值 |

用 == 比较数组：

- 相同维数且含有相同个数元素的数组才可以比较
- 每个元素都相同的才想等

```go
func TestCompareArray(t *testing.T) {
	a := [...]int{1, 2, 3, 4}
	b := [...]int{1, 2, 3, 5}
	c := [...]int{1, 2, 3, 4}
	t.Log(a == b) // false
	t.Log(a == c) // true
}
```



### 逻辑运算符

| 运算符 |     描述      |
| :----: | :-----------: |
|   &&   | 逻辑AND运算符 |
|  \|\|  | 逻辑OR运算符  |
|   !    | 逻辑NOT运算符 |

### 位运算符

| 运算符 |               描述               |
| :----: | :------------------------------: |
|   &    |  参与运算两数各对应的二进位相与  |
|   \|   |  参与运算两数各对应的二进位相或  |
|   ^    | 参与运算两数各对应的二进位相异或 |
|   <<   |            左移运算符            |
|   >>   |            右移运算符            |

&^按位置零

1 &^ 0 -- 1

1 &^ 1 -- 0

0 &^ 1 -- 0

0 &^ 0 -- 0

## 条件和循环

### 循环

Go语言仅支持循环关键字 `for`

循环：`for (j := 7; j<=9; i++)`

条件循环 while(n<5)

```go
n := 0
	for n < 5 {
		t.Log(n)
		n++
	}
```

无限循环 while(true)

```go
n := 0
	for {
		t.Log(n)
		n++
	}
```

### 条件

if条件：

- condition 表达式结果必须为布尔值
- 支持变量赋值：

```go
func TestIfMultiSec(t *testing.T) {
	if a := 1 == 1; a {
		t.Log(a)
	}
}
```



switch条件：

- 条件表达式不限制为常量或者整数；
- 单个case中，可以出现多个结果选项，用逗号分隔；
- 与C语言等规则相反，Go语言不需要用break来明确退出一个case；
- 可以不设定switch之后的条件表达式，在此种情况下，整个switch结构与多个if...else..的逻辑作用等同；

```go
func TestSwitchMultiCase(t *testing.T) {
	for i := 0; i < 5; i++ {
		switch i {
		case 0, 2:
			t.Log("even")
		case 1, 3:
			t.Log("odd")
		default:
			t.Log("it is not 0-3")
		}
	}
}

func TestSwitchCaseCondition(t *testing.T) {
	for i := 0; i < 5; i++ {
		switch {
		case i%2 == 0:
			t.Log("even")
		case i%2 == 1:
			t.Log("odd")
		default:
			t.Log("it is not 0-3")
		}
	}
}
```


