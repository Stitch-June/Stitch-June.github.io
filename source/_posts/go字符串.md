---
title: Go字符串
tags: []
id: '43'
categories:
  - - uncategorized
date: 2020-09-20 18:41:54
---

字符串：

*   string是数据类型，不是引用或指针类型
*   string是只读的byte slice，len函数可以获取它所包含的byte数
*   string的byte数组可以存放任何数据

```
func TestStringInit(t *testing.T) {
    var s string
    t.Log(s) // 初始化为默认零值"" 空字符串

    s = "hello"
    t.Log(len(s)) // 5 5个byte
    //s[1] = 3 // string是不可变的byte slice 不可以赋值

    s = "\xE4\xB8\xA5" // 可以存储任何二进制数据

    t.Log(s)      // 严
    t.Log(len(s)) // 3 为3个byte
}
```

Unicode UTF8：

*   Unicode是一种字符集（code point）
*   UTF8是unicode的存储实现（转换为字节序列的规则）

编码和存储：

字符

Unicode

UTF-8

string/\[\]byte

"中"

0x4E2D

0xE4B8AD

\[0xE4,0xB8,0xAD\]

```
func TestUnicode(t *testing.T) {
    s := "中"
    t.Log(len(s)) // 3 为3个byte

    // 新的数据类型 rune 能够取出字符串的unicode
    c := []rune(s)
    t.Log(len(c)) // 1

    // %x 输出十六进制
    t.Logf("中 unicode %x", c[0]) // 中 unicode 4e2d
    t.Logf("中 utf8 %x", s)       // 中 utf8 e4b8ad
}
```

字符串遍历：

```
func TestStringToRange(t *testing.T) {
    s := "PHP是世界上最好的语言"
    for _, c := range s {
        // 都用第一个参数
        t.Logf("%[1]c %[1]x", c)
    }
    /** 运行结果：
    === RUN   TestStringToRange
        TestStringToRange: string_test.go:37: P 50
        TestStringToRange: string_test.go:37: H 48
        TestStringToRange: string_test.go:37: P 50
        TestStringToRange: string_test.go:37: 是 662f
        TestStringToRange: string_test.go:37: 世 4e16
        TestStringToRange: string_test.go:37: 界 754c
        TestStringToRange: string_test.go:37: 上 4e0a
        TestStringToRange: string_test.go:37: 最 6700
        TestStringToRange: string_test.go:37: 好 597d
        TestStringToRange: string_test.go:37: 的 7684
        TestStringToRange: string_test.go:37: 语 8bed
        TestStringToRange: string_test.go:37: 言 8a00
    --- PASS: TestStringToRange (0.00s)
    */
}
```

常用字符串函数：

```
func TestStringFn(t *testing.T) {
    s := "A,B,C"
    // 字符串切割
    parts := strings.Split(s, ",")
    for _, part := range parts {
        t.Log(part)
    }
    // 字符串拼接
    t.Log(strings.Join(parts, "-"))
    /** 运行结果：
    === RUN   TestStringFn
        TestStringFn: string_fun_test.go:12: A
        TestStringFn: string_fun_test.go:12: B
        TestStringFn: string_fun_test.go:12: C
        TestStringFn: string_fun_test.go:14: A-B-C
    --- PASS: TestStringFn (0.00s)
    */
}
```

数据类型转换：

```
func TestStrConv(t *testing.T) {
    // 整型转字符串
    s := strconv.Itoa(10)
    t.Log("str" + s)

    // 字符串转整型对错误值需要做一个判断
    if i, err := strconv.Atoi("10"); err == nil {
        t.Log(10 + i)
    }
    /** 运行结果：
    === RUN   TestStrConv
        TestStrConv: string_fun_test.go:29: str10
        TestStrConv: string_fun_test.go:33: 20
    --- PASS: TestStrConv (0.00s)
     */
}
```