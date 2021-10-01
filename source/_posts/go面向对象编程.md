---
title: Go面向对象编程
tags: []
id: '29'
categories:
  - - uncategorized
date: 2020-09-20 17:52:10
---

**Is Go an object-oriented language?**

> **Yes and no**. Although Go has types and methods and allows an object
> 
> oriented style of programming, there is **no type hierarchy**. **The concept**
> 
> **of “interface” in Go provides a different approach that we believe is**
> 
> **easy to use and in some ways more general.**
> 
> Also, the lack of a type hierarchy makes “objects” in Go feel much more
> 
> lightweight than in languages such as C++ or Java.

## 行为的定义和实现

结构体定义：

```
type Employee struct {
    Id   string
    Name string
    Age  int
}
```

实例创建及初始化：

```
func TestCreateEmployee(t *testing.T) {
    e := Employee{"0", "Bob", 20}         // 分别把值放进去
    e1 := Employee{Name: "Mike", Age: 30} // 指定某个field的值
    e2 := new(Employee)                   // new关键字 去创建指向实例的指针 这里返回的引用/指针 相当于 e:=Employee{}
    e2.Id = "2"                           // 通过 example.filed 去赋值
    e2.Name = "Rose"
    e2.Age = 22
    t.Log(e)
    t.Log(e1)
    t.Log(e1.Id)
    t.Log(e2)
    t.Logf("e is %T", e)
    t.Logf("e2 is %T", e2)
    /** 运行结果：
    === RUN   TestCreateEmployee
        TestCreateEmployee: encap_test.go:18: {0 Bob 20}
        TestCreateEmployee: encap_test.go:19: { Mike 30}
        TestCreateEmployee: encap_test.go:20:
        TestCreateEmployee: encap_test.go:21: &{2 Rose 22}
        TestCreateEmployee: encap_test.go:22: e is test.Employee
        TestCreateEmployee: encap_test.go:23: e2 is *test.Employee
    --- PASS: TestCreateEmployee (0.00s)
     */
}
```

行为定义：

```
// 第一种定义方式在实例对应方法被调用时，实例的成员会进行值复制
func (e Employee) String() string {
    return fmt.Sprintf("ID:%s-Name:%s-Age:%d", e.Id, e.Name, e.Age)
}

func TestStructOperations(t *testing.T) {
    e := Employee{"0", "Bob", 20}
    t.Log(e.String())
    /** 运行结果：
    === RUN   TestStructOperations
        TestStructOperations: encap_test.go:46: ID:0-Name:Bob-Age:20
    --- PASS: TestStructOperations (0.00s)
     */
}
```

```
// 通常情况下为了避免内存拷贝我们使用第二种定义方式
func (e *Employee) String() string {
    return fmt.Sprintf("ID:%s/Name:%s/Age:%d", e.Id, e.Name, e.Age)
}

func TestStructOperations(t *testing.T) {
    e := Employee{"0", "Bob", 20}
    t.Log(e.String())
    /** 运行结果：
    === RUN   TestStructOperations
        TestStructOperations: encap_test.go:51: ID:0/Name:Bob/Age:20
    --- PASS: TestStructOperations (0.00s)
    */
}
```

> 在Go语言中不管通过指针访问还是通过实例访问，都是一样的
> 
> 那么这两种定义没有区别吗？？

```
func (e *Employee) String() string {
    fmt.Printf("Address is %x \n", unsafe.Pointer(&e.Name))
    return fmt.Sprintf("ID:%s/Name:%s/Age:%d", e.Id, e.Name, e.Age)
}

func TestStructOperations(t *testing.T) {
    e := Employee{"0", "Bob", 20}
    fmt.Printf("Address is %x \n", unsafe.Pointer(&e.Name))
    t.Log(e.String())
    /** 运行结果：
    === RUN   TestStructOperations
    Address is c000060370
    Address is c000060370
        TestStructOperations: encap_test.go:54: ID:0/Name:Bob/Age:20
    --- PASS: TestStructOperations (0.00s)
    */
}
```

可以发现两个地址一致

```
func (e Employee) String() string {
    fmt.Printf("Address is %x \n", unsafe.Pointer(&e.Name))
    return fmt.Sprintf("ID:%s-Name:%s-Age:%d", e.Id, e.Name, e.Age)
}

func TestStructOperations(t *testing.T) {
    e := Employee{"0", "Bob", 20}
    fmt.Printf("Address is %x \n", unsafe.Pointer(&e.Name))
    t.Log(e.String())
    /** 运行结果：
    === RUN   TestStructOperations
    Address is c000092370
    Address is c0000923a0
        TestStructOperations: encap_test.go:55: ID:0-Name:Bob-Age:20
    --- PASS: TestStructOperations (0.00s)
    */
}
```

这时候两个地址不一致，说明结构体的数据被复制了，会造成开销

## Go语言的相关接口

Java的接口与依赖：

![](http://qiniu.gaobinzhan.com/2020/05/17/234f907d27e94.png)

Go的 Duck Type 式接口实现：

*   接口为非入侵性，实现不依赖与接口定义
*   所以接口的定义可以包含在接口使用者包内

![](http://qiniu.gaobinzhan.com/2020/05/17/b2c005524c89d.png)

```
type Programmer interface {
    WriteHelloWorld() string
}

type GoProgrammer struct {
}

func (g *GoProgrammer) WriteHelloWorld() string {
    return "Hello World"
}

func TestClient(t *testing.T) {
    var p Programmer
    p = new(GoProgrammer)
    t.Log(p.WriteHelloWorld())
    /** 运行结果：
    === RUN   TestClient
        TestClient: interface_test.go:19: Hello World
    --- PASS: TestClient (0.00s)
     */
}
```

接口变量：

![](http://qiniu.gaobinzhan.com/2020/05/17/87e5943515174.png)

自定义类型：

*   type IntConvertionFn func(n int) int
*   type MyPoint int

## 扩展和复用

复合：

匿名类型嵌入：

它不是**继承**，如果我们把“内部 struct”看作父类，把“外部 struct” 看作子类，会发现如下问题：

*   不支持子类替换
*   子类并不是真正继承了父类的方法
    

```
type Pet struct {
}

func (p *Pet) Speak() {
    fmt.Print("...")
}

func (p *Pet) SpeakTo(string string) {
    p.Speak()
    fmt.Println(" ", string)
}

type Dog struct {
    p *Pet
}

func (d *Dog) Speak() {
    fmt.Print("Wang!")
}

func (d *Dog) SpeakTo(string string) {
    d.p.SpeakTo(string)
}

func TestDog(t *testing.T) {
    dog := new(Dog)
    dog.SpeakTo("Gao") // 没有打印 Wang! 需要改动 dog中SpeakTo方法
    /** 运行结果：
    === RUN   TestDog
    ...  Gao
    --- PASS: TestDog (0.00s)
     */
}
```

## 多态与空接口

多态：

```
type Code string // 自定义类型

type Programmer interface {
    WriteHelloWorld() Code
}

type GoProgrammer struct {
}

type PhpProgrammer struct {
}

func (g *GoProgrammer) WriteHelloWorld() Code {
    return "fmt.Println(\"Hello World\")"
}

func (p *PhpProgrammer) WriteHelloWorld() Code {
    return "echo \"Hello World\""
}

func writeFirstProgram(p Programmer) {
    fmt.Printf("%T %v\n", p, p.WriteHelloWorld())
}

func TestPolymorphism(t *testing.T) {
    goProg := new(GoProgrammer)
    phpProg := new(PhpProgrammer)
    writeFirstProgram(goProg)
    writeFirstProgram(phpProg)
    /** 运行结果
    === RUN   TestPolymorphism
    *test.GoProgrammer fmt.Println("Hello World")
    *test.PhpProgrammer echo "Hello World"
    --- PASS: TestPolymorphism (0.00s)
     */
}
```

空接口与断言：

*   空接口可以表示任何类型
*   通过断言来将空接口转换为制定类型
    
    `v, ok := p.(int) // ok=true 时为转换成功`
    

```
func DoSomething(p interface{}) {
    // 如果传入的参数能被断言成一个整型
    if i, ok := p.(int); ok {
        fmt.Println("Integer", i)
        return
    }

    // 如果传入的参数能被断言成一个字符型
    if s, ok := p.(string); ok {
        fmt.Println("String", s)
        return
    }

    fmt.Println("Unknow Type")

    // 也可以通过switch来判断
    /*switch v := p.(type) {
    case int:
        fmt.Println("Integer", v)
    case string:
        fmt.Println("String", v)
    default:
        fmt.Println("Unknow Type")
    }*/
}

func TestEmptyInterfaceAssertion(t *testing.T) {
    DoSomething(10)
    DoSomething("gaobinzhan")
    /** 运行结果
    === RUN   TestEmptyInterfaceAssertion
    Integer 10
    String gaobinzhan
    --- PASS: TestEmptyInterfaceAssertion (0.00s)
    */
}
```

Go接口最佳实践：

![](http://qiniu.gaobinzhan.com/2020/05/17/f8ba0c6a8e958.png)