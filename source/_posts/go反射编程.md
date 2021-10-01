---
title: Go反射编程
tags: []
id: '65'
categories:
  - - uncategorized
date: 2020-09-23 07:05:18
---

reflect.TypeOf vs. reflect.ValueOf：

*   reflflect.TypeOf 返回类型 (reflflect.Type)
*   reflflect.ValueOf 返回值 (reflflect.Value)
*   可以从 reflflect.Value 获得类型
*   通过 kind 的来判断类型

```
func CheckType(v interface{}) {
    t := reflect.TypeOf(v)
    switch t.Kind() {
    case reflect.Float32, reflect.Float64:
        fmt.Println("Float")
    case reflect.Int, reflect.Int32, reflect.Int64:
        fmt.Println("Integer")
    default:
        fmt.Println("Unknown", t)
    }
}

func TestBasicType(t *testing.T) {
    var f float64 = 12
    CheckType(f)
    /** 运行结果：
    === RUN   TestBasicType
    Float
    --- PASS: TestBasicType (0.00s)
    */
}
```

利用反射编写灵活的代码：

*   按名字访问结构的成员
    
    `reflect.ValueOf(*e).FieldByName("Name")`
    
*   按名字访问结构的方法
    
    `reflect.ValueOf(*e).MethodByName("UpdateAge").Call([]reflect.Value{reflect.ValueOf(1)})`
    

```
type Employee struct {
    EmployeeId string
    Name       string `format:"normal"`
    Age        int
}

func (e *Employee) UpdateAge(newVal int) {
    e.Age = newVal
}

func TestInvokeByName(t *testing.T) {
    e := &Employee{"1", "Mike", 30}
    // 按名字获取成员
    t.Logf("Name：value(%[1]v)，Type(%[1]T)", reflect.ValueOf(*e).FieldByName("Name"))
    if nameField, ok := reflect.TypeOf(*e).FieldByName("Name"); !ok {
        t.Error("Failed to get 'Name' field.")
    } else {
        t.Log("Tag:format", nameField.Tag.Get("format"))
    }
    reflect.ValueOf(e).MethodByName("UpdateAge").Call([]reflect.Value{reflect.ValueOf(1)})
    t.Log("Updated Age:", e)
    /** 运行结果：
    === RUN   TestInvokeByName
        TestInvokeByName: reflect_test.go:28: Name：value(Mike)，Type(reflect.Value)
        TestInvokeByName: reflect_test.go:32: Tag:format normal
        TestInvokeByName: reflect_test.go:35: Updated Age: &{1 Mike 1}
    --- PASS: TestInvokeByName (0.00s)
    */
}
```

Struct Tag：

```
type BasicInfo struct {
  Name string `json:"name"`
  Age int `json:"age"`
}
```

访问Struct：

```
if nameField, ok := reflect.TypeOf(*e).FieldByName("Name"); !ok {
t.Error("Failed to get 'Name' field.")
} else {
t.Log("Tag:format", nameField.Tag.Get("format")) }
```

Reflect.Type 和 Reflflect.Value 都有 FieldByName ⽅法，注意他们的区别。

DeepEqual：

比较切片和map

```
type Customer struct {
    CookieID string
    Name     string
    Age      int
}

func TestDeepEqual(t *testing.T) {
    a := map[int]string{1: "one", 2: "two", 3: "three"}
    b := map[int]string{1: "one", 2: "two", 4: "three"}
    fmt.Println(reflect.DeepEqual(a, b))

    s1 := []int{1, 2, 3}
    s2 := []int{1, 2, 3}
    s3 := []int{2, 3, 1}
    t.Log("s1 == s2?", reflect.DeepEqual(s1, s2))
    t.Log("s1 == s3?", reflect.DeepEqual(s1, s3))

    c1 := Customer{"1", "Mike", 40}
    c2 := Customer{"1", "Mike", 40}

    fmt.Println(reflect.DeepEqual(c1, c2))
    /** 运行结果：
    === RUN   TestDeepEqual
    false
        TestDeepEqual: fiexible_reflect_test.go:23: s1 == s2? true
        TestDeepEqual: fiexible_reflect_test.go:24: s1 == s3? false
    true
    --- PASS: TestDeepEqual (0.00s)
    */
}
```

关于“反射”你应该知道的：

*   提⾼了程序的灵活性
*   降低了程序的可读性
*   降低了程序的性能

```
type Employee struct {
    EmployeeID string
    Name       string `format:"normal"`
    Age        int
}

func (e *Employee) UpdateAge(newVal int) {
    e.Age = newVal
}

type Customer struct {
    CookieID string
    Name     string
    Age      int
}

func fillBySettings(st interface{}, settings map[string]interface{}) error {

    // func (v Value) Elem() Value
    // Elem returns the value that the interface v contains or that the pointer v points to.
    // It panics if v's Kind is not Interface or Ptr.
    // It returns the zero Value if v is nil.

    if reflect.TypeOf(st).Kind() != reflect.Ptr {
        return errors.New("the first param should be a pointer to the struct type.")
    }
    // Elem() 获取指针指向的值
    if reflect.TypeOf(st).Elem().Kind() != reflect.Struct {
        return errors.New("the first param should be a pointer to the struct type.")
    }

    if settings == nil {
        return errors.New("settings is nil.")
    }

    var (
        field reflect.StructField
        ok    bool
    )

    for k, v := range settings {
        if field, ok = (reflect.ValueOf(st)).Elem().Type().FieldByName(k); !ok {
            continue
        }
        if field.Type == reflect.TypeOf(v) {
            vstr := reflect.ValueOf(st)
            vstr = vstr.Elem()
            vstr.FieldByName(k).Set(reflect.ValueOf(v))
        }

    }
    return nil
}

func TestFillNameAndAge(t *testing.T) {
    settings := map[string]interface{}{"Name": "Mike", "Age": 30}
    e := Employee{}
    if err := fillBySettings(&e, settings); err != nil {
        t.Fatal(err)
    }
    t.Log(e)
    c := new(Customer)
    if err := fillBySettings(c, settings); err != nil {
        t.Fatal(err)
    }
    t.Log(*c)
    /** 运行结果：
    === RUN   TestFillNameAndAge
        TestFillNameAndAge: fiexible_reflect_test.go:69: { Mike 30}
        TestFillNameAndAge: fiexible_reflect_test.go:74: { Mike 30}
    --- PASS: TestFillNameAndAge (0.00s)
    */
}
```

”不安全“行为的危险性：

```
func TestUnsafe(t *testing.T) {
    i := 10
    f := *(*float64)(unsafe.Pointer(&i))
    t.Log(unsafe.Pointer(&i))
    t.Log(f)
    /** 运行结果：
    === RUN   TestUnsafe
        TestUnsafe: unsafe_test.go:11: 0xc000016268
        TestUnsafe: unsafe_test.go:12: 5e-323
    --- PASS: TestUnsafe (0.00s)
    */
}

// The cases is suitable for unsafe
type MyInt int

// 合理的类型转换
func TestConvert(t *testing.T) {
    a := []int{1, 2, 3, 4}
    b := *(*[]MyInt)(unsafe.Pointer(&a))
    t.Log(b)
    /** 运行结果：
    === RUN   TestConvert
        TestConvert: unsafe_test.go:26: [1 2 3 4]
    --- PASS: TestConvert (0.00s)
    */
}

// 原子类型操作
func TestAtomic(t *testing.T) {
    var shareBuffer unsafe.Pointer
    writeDataFn := func() {
        data := []int{}
        for i := 0; i < 100; i++ {
            data = append(data, i)
        }
        atomic.StorePointer(&shareBuffer, unsafe.Pointer(&data))
    }
    readDataFn := func() {
        data := atomic.LoadPointer(&shareBuffer)
        fmt.Println(data, *(*[]int)(data))
    }
    var wg sync.WaitGroup
    writeDataFn()
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            for i := 0; i < 10; i++ {
                writeDataFn()
                time.Sleep(time.Microsecond * 100)
            }
            wg.Done()
        }()
        wg.Add(1)
        go func() {
            for i := 0; i < 10; i++ {
                readDataFn()
                time.Sleep(time.Microsecond * 100)
            }
            wg.Done()
        }()
    }
    wg.Wait()
}
```