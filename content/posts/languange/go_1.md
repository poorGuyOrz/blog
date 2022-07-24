---
title: "A Tour of Go速通"
date: 2022-05-17T22:34:11+08:00
draft: false
---

* 2022-05-17  基础语法
* 2022-05-24  复合类型，goroutine，channel

## 基础语法

### Packages

go使用Packages维护模块，使用import导入模块，import最后一个元素才是需要导入的模块
```go
import (
	"fmt"
	"math/rand"
)
```
import 可以单独一个导入一个模块，也可以批量导入，
与之对应的是export，go不显示声明export，首字母大写的变量或方法自动export，外部只能使用导出的变量或者方法，类似c++中类的私有和共有的概念。

### Functions
与c++或者Java或者其他语言不同的是，go的函数签名格式为`func func_name(parm1 [type], parm2 [type]....) retype {}`，先声明名字，再声明变量的类型，参数列表中有多个参数且类型一致的时候，前面的参数类型可以省略，只需要保留之后一个
```go
func add(x, y int) int {
	return x + y
}
```
且go可以很轻易的实现多返回值的功能，如下
```go
func swap(x, y string) (string, string) {
	return y, x
}

func main() {
	a, b := swap("hello", "world")
	fmt.Println(a, b)
}
```
上面的功能在c++中需要使用结构体或者tuple或者Paris才能实现。

go的return还可以使用不带参数的 "naked" return，此时要求函数签名中的return参数必须有名字，且在函数体中必须为参数赋值，此时使用return直接返回，参数可以直接传递到外部，但是需要注意的时候，如果函数体过于庞大且有多个出口，不建议使用，难于阅读
```go
func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return
}
```

### 变量

使用var声明变量，声明多个变量的时候可以类似参数列表中的参数，前面参数不需要声明类型。初始化的时候按顺序初始化，且初始化的参数个数必须前后一致
```go
var i, j, k int = 1,2,0 // true
var i, j, k int = 1,2   // false
```
初始化的时候还可以省略类型，程序会进行参数推导。
函数内部还可以使用 `:=`替换var声明变量，此时必须初始化，但是在函数体外部，由于go规定，每条语句必须由特定的关键字开头，所以外部不可使用此语法。

go有以下基础类型，除了额外的complex类型之外其他的和c++类似，零值也和c++类似，complex的零值为`(0+0i)`
```go
bool

string

int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr

byte // alias for uint8

rune // alias for int32
     // represents a Unicode code point

float32 float64

complex64 complex128
```
go可以使用c的语法进行类型转换，但是c允许隐式转换，而go必须声明转换的类型，否则语法错误
go中使用nil作为null

### 常量
直接使用`const xx = y`的语法声明，不允许使用`:=`，因为他隐含有变量声明的意思，


## 语句

### for

go只有for一种循环结构，与c++相比，go的()是省略掉的，{}是必须的，其中init statement和post statement是可以省略的， condition expression不可省略，这和c++一样，且在init statement和post statement省略的时候，;也可以省略，此时for循环类似c++中的while循环
```go
// the init statement: executed before the first iteration    可省略
// the condition expression: evaluated before every iteration 不可省略
// the post statement: executed at the end of every iteration 可省略
	for i := 0; i < 10; i++ {
		sum += i
	}

  i := 0
	for ;i < 10; {
		sum += i
		i++
	}

  i := 0
	for i < 10 {
		sum += i
		i++
	}
```

如果条件语句去掉，则此时是个死循环，下面的代码表现是一样的
```go

for true {
}

for {
}
```

### if

类似for，表达是的括号是可省略的，{}是必须的，前面的for的()其实也是可选的，但是必须是只有条件表达式的时候，_所以猜测是否是只有一个语句表达式的时候才可以使用括号_，if在条件语句之前也可以使用init语句，此时初始化的参数只能在if范围内使用
```go
func pow(x, n, lim float64) float64 {
	if v := math.Pow(x, n); v < lim {
		return v
	}
	return lim
}
```

### Switch

和前面类似，表达是的括号是可省略的，{}是必须的，Switch后面包含init 语句和需要匹配的表达式，init语句可以在外部实现，case语句和其他语言的实现不一样，没有默认fallthroh，他的每一个case不需要写break，默认行为就是break的，的且case的表达式要求是constant，不要求是整形
```go
func main() {
	fmt.Print("Go runs on ")
	switch os := runtime.GOOS; os {
	case "darwin":
		fmt.Println("OS X.")
	case "linux":
		fmt.Println("Linux.")
	default:
		fmt.Printf("%s.\n", os)
	}
}
```
还可以是一个表达式，case匹配表达式成功的分支
```go
	today := time.Now().Weekday()
	switch time.Saturday {
	case today + 0:
		fmt.Println("Today.")
	case today + 1:
		fmt.Println("Tomorrow.")
	case today + 2:
		fmt.Println("In two days.")
	default:
		fmt.Println("Too far away.")
	}
```
Switch是一个等值匹配过程，所以Switch后面的语句是case计算的结果，如果此时Switch之后的结果是true，则可以省略，此时就是一堆case在判断是否为true，相当于大量的连体if-else
```go
	t := time.Now()
	switch {
	case t.Hour() < 12:
		fmt.Println("Good morning!")
	case t.Hour() < 17:
		fmt.Println("Good afternoon.")
	default:
		fmt.Println("Good evening.")
	}
```

### defer

把defer修饰的语言延迟到函数返回之后才返回，所有的defer修饰的语句使用栈维护，最后调用的defer语句最先执行
```go
func main() {
	fmt.Println("counting")

	for i := 0; i < 10; i++ {
		defer fmt.Println(i)
	}

	fmt.Println("done")
}
```


## 指针，复合类型

### 指针

和C/C++的指针类似，使用*声明指针，使用&取地址。
```go
  i, j := 42, 2701

	p := &i         // point to i
	fmt.Println(*p) // read i through the pointer
	*p = 21         // set i through the pointer
	fmt.Println(i)  // see the new value of i
```

### Structs

结构体，和C/C++类似，可以使用.访问变量，结构体是自定义复合类型，参数传递的时候需要区分值传递和引用传递，所以建议结构体使用指针操作，结构体使用指针的时候指针的*可以省略，可以直接使用`pointer.filed`的语法，一个语法糖
```go
type Vertex struct {
	X int
	Y int
}

func main() {
	v := Vertex{1, 2}
	p := &v
	p.X = 1e9
	fmt.Println(v)
}
```
结构体初始化的时候，可以按照结构体字段声明的顺序初始化，也可以使用字段名指定初始化，此时其他没有初始化的字段为零值。
```go
v2 = Vertex{Y: 1}  // Y:0 is implicit
```

另外，go里面初始化变量的时候语法是 `x := 1`，初始化基础类型的时候类型可以省略，会进行类型推导，但是对于非基础类型，需要显示声明类型，所以可以使用下面的语法
```go
	s := struct {
		i int
		b bool
	}{2, true}
```

### Arrays

类似C/C++中的普通数组，初始化之后无法直接改变大小，c++提供vector来扩展数组的功能，而go使用新类型slice与数组结合使用
```go
primes := [6]int{2, 3, 5, 7, 11, 13}
```
和结构体一起使用，可以使用下面的语法
```go
	s := []struct {
		i int
		b bool
	}{
		{2, true},
		{3, false},
		{5, true},
		{7, true},
		{11, false},
		{13, true},
	}
```

### slice

切片类型，可以理解为数组的窗口操作，在数组上使用语法`arr[low : high]`可以构造一个数组上`[low : high)`的片段，内部数据还是在原来的地址上，切片的数据的变动就是直接在更改元数据，在数组的所有的切片或者数组本身的数据变动，都会印象到其他对象，类似C++的string_view，使用的时候需要注意。范围需要是在arr的合理范围，不允许越界，不允许使用负数，两个元素，可以省略其中一个或者两个都省略，此时省略的元素就是0或者是arr.size()
```go
func main() {
	names := [4]string{
		"John",
		"Paul",
		"George",
		"Ringo",
	}
	fmt.Println(names)

	a := names[0:2]
	b := names[1:3]
	fmt.Println(a, b)

	b[0] = "XXX"
	fmt.Println(a, b)
	fmt.Println(names)
}

-----------
[John Paul George Ringo]
[John Paul] [Paul George]
[John XXX] [XXX George]
[John XXX George Ringo]
```

切片还可以使用切片初始化。切片可以使用类似数组的方式声明， 没有声明长度的时候是切片类型，下面是几个切片初始化的例子
```go
var a [10]int
a[0:10]
a[:10]
a[0:]
a[:]
```

切片可以使用`len`和`cap`查看大小和容量，len是当前切片的元素数量，cap是切片当前start到数组end的大小，切片调整的时候，只能在当前切片start的位置到cap之间调整，无法直接调整到切片之前的位置
```go
// len=6 cap=6 [2 3 5 7 11 13]
s := []int{2, 3, 5, 7, 11, 13}
// len=0 cap=6 []
s = s[:0]
// len=4 cap=6 [2 3 5 7]
s = s[:4]
// len=2 cap=4 [5 7]
s = s[2:]
// len=4 cap=4 [5 7 11 13]
s = s[:4]
// error
s = s[:5]
```

* 可以使用make构造切片，第二个参数为切片的len，第三个参数为切片的cap，cap默认为len
```go
a := make([]int, 5)  // len(a)=5
b := make([]int, 0, 5) // len(b)=0, cap(b)=5
```

* 切片可以包含任何类型，例如切片的切片
```go
	board := [][]string{
		[]string{"_", "_", "_"},
		[]string{"_", "_", "_"},
		[]string{"_", "_", "_"},
	}
```
可以使用append动态扩展切片，类似vector，但是在go中，简单的实验之后，没有发现扩容之后内存地址发生变化，尚且不清楚内存超出容量之后是否重新分配空间了
```go
	s = append(s, 1)
	printSlice(s)
	p3 := &s
	fmt.Printf("%p\n", p3)

	// We can add more than one element at a time.
	s = append(s, 2, 3, 4)
	printSlice(s)
	p4 := &s
	fmt.Printf("%p\n", p4)
------
len=2 cap=2 [0 1]
0xc0000b4018
len=5 cap=6 [0 1 2 3 4]
0xc0000b4018
```

> TIPS: go中，优先使用切片类型

### Exercise: Slices
```go
func Pic(dx, dy int) [][]uint8 {
	s := make([][]uint8, dy)
	for x := range s {
		s[x] = make([]uint8, dx)
		for y := range s[x] {
			s[x][y] = uint8((x^y))
		}
	}
	return s
}

func main() {
	pic.Show(Pic)
}
```

### range

对于切片，在for循环中使用range可以获得切片的下标和对应的值，可以使用`_`省略不想使用的参数，或者直接使用一个参数接收index
```go
func main() {
	pow := make([]int, 10)
	for value := range pow {
		pow[value] = 1 << uint(value) // == 2**i
	}
	for _, value := range pow {
		fmt.Printf("%d\n", value)
	}
}
```

### map
键值对，语法为`map[ktype]vtype`，使用make构建，和C++的map使用类似，可以使用`[]`来访问或者设置一个值，其中可以使用语法`ele, ok = map[key]`来测试key是否存在，存在则ok为true，其ele获得值，否则ele为零值且ok为false。
```go
func main() {
	m := make(map[string]int)

	m["Answer"] = 42
	fmt.Println("The value:", m["Answer"])

	m["Answer"] = 48
	fmt.Println("The value:", m["Answer"])

	delete(m, "Answer")
	fmt.Println("The value:", m["Answer"])

	v, ok := m["Answer"]
	fmt.Println("The value:", v, "Present?", ok)
}
```


### func as value
类似函数指针，函数可以使用函数作为参数，但是C++中可以使用指针或者function表达式作为参数声明，go中则需要使完整的参数签名。
```go
func compute(fn func(float64, float64) float64) float64 {
	return fn(3, 4)
}

	hypot := func(x, y float64) float64 {
		return math.Sqrt(x*x + y*y)
	}
	fmt.Println(hypot(5, 12))

	fmt.Println(compute(hypot))

```
函数还可以使用闭包的特性，下面的例子中，adder的返回值是一个函数，他捕获了adder中的sum变量。之后对于每一个adder势力，他内部的sum都是独立存在的，且会被之前的调用影响到，直观的理解就是他实现了一个简单的类，打包了类属性和方法。
```go
func adder() func(int) int {
	sum := 0
	return func(x int) int {
		sum += x
		return sum
	}
}

func main() {
	pos, neg := adder(), adder()
	for i := 0; i < 10; i++ {
		fmt.Println(
			pos(i),
			neg(-2*i),
		)
	}
}
```

Exercise: Fibonacci closure
```go
func fibonacci() func() int {
	x, y := 0, 1
	return func() int {
		ret := x
		x, y = y, (x + y)
		return ret
	}
}

func main() {
	f := fibonacci()
	for i := 0; i < 10; i++ {
		fmt.Println(f())
	}
}
```

至此就是go的基础语法与结构，有其他语言基础几乎一遍过，接下来是比较高级点的知识，可能需要更多的思考
----------

## high level

### 方法
go没有类，不像C++可以定义自己的基础类型，但是可以使用其他机制实现类似C++的对象调用方法的实现，在函数名称之前声明receiver，调用的时候使用receiver 类型调用函数，  
语法为`func (v receivertype) func_name(args ...) ret.. {}`，具体例子为
```go
type Vertex struct {
	X, Y float64
}

func (v Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
	v := Vertex{3, 4}
	fmt.Println(v.Abs())
}
```

可以看见，语法和C++的类调用方法一样，虽然实现不一样，C++实现类方法的底层原理是方法的第一个隐藏的参数是this指针，调用的时候使用obj.func的语法调用，但是实际上的调用是`func(obj...)`，只是为了体现OOP的思想，语法按照现在的实现机制实现，其实都差不多。go的是实现是把类型和方法分离，最终实现的效果是一样的，。

此外，go的receiver是通过拷贝的方式调用的，所以函数内的操作无法直接影响到receiver，但是可以把receiver声明为指针，此时就是在使用指针调用函数了，可以实现和C++常规调用一样的效果，其由于GO的指针和值的调用一样，所以几乎在代码上没有差别
```go
type Vertex struct {
	X, Y float64
}

func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func (v *Vertex) Scale(f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func main() {
	v := Vertex{3, 4}
	v.Scale(10)
	fmt.Println(v.Abs())
}
```

但是当函数中的参数使用指针的时候，直接调用的时候必须使用指针类型， 此时编译器没有做直接替换，receiver调用的时候，是会自动转换的，
```go
var v Vertex
ScaleFunc(v, 5)  // Compile error!
ScaleFunc(&v, 5) // OK

------
var v Vertex
v.Scale(5)  // OK
p := &v
p.Scale(10) // OK
```

> 简而言之，使用receiver的时候，函数无论声明receiver为指针或者值，都可以自由调用，具体行为有receiver的声明方式决定，但是作为参数的时候，声明为指针，则必须使用指针参数

### Interfaces

接口，一组抽线方法的声明，所有实现对应的方法的类型都视为实现了接口，此时可以把相应的具体的对象赋值给接口对象
```go
type Abser interface {
	Abs() float64
}

type Vertex struct {
	X, Y float64
}

func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

var a Abser
v := Vertex{3, 4}
a = &v
```

接口具有具体的值和和类型，虽然在调用的时候是直接使用接口调用具体方法。但是应该和C++的多态类似，可以在运行时使用某信息进行函数调用，接口可以使用`%v`和`%T`打印值和类型

当实现接口的类型的receiver是指针类型且是nil，则此时还是可以继续正常调用函数，只是必须自己处理nil。类似C++中，null对象也可以正常调用类的非静态方法，只要没有使用到类的其他成员变量或者成员函数，就可以正常运行
```go
func (t *T) M() {
	if t == nil {
		fmt.Println("<nil>")
		return
	}
	fmt.Println(t.S)
}
```

没有被实现的接口称为nil interface，此时可以正常定义对象，但是值和type都是nil，不能调用接口函数
```go
type I interface {
	M()
}

func main() {
	var i I
	describe(i)
	i.M()
}

func describe(i I) {
	fmt.Printf("(%v, %T)\n", i, i)
}

--------------
(<nil>, <nil>)
panic: runtime error: invalid memory address or nil pointer dereference
```

empty interface可以接受任何参数，类似C++ 的void指针可以接受全指针类型。

此外，map中有个语法可以测试对应的key是否存在，interface也提供类似的功能，检验接口是否是指定类型，语法为`v, ok := i.(type)`，使用规则和map类似
```go
	var i interface{} = "hello"

	s := i.(string)
	fmt.Println(s)

	s, ok := i.(string)
	fmt.Println(s, ok)

	f, _ := i.(float64)
	fmt.Println(f)
```
考虑需要检索接口的类型的时候，可以使用循环语句一个一个的实验，但是最好的办法还是有接口可以主动的告诉我们。所以go提供`i.(type)`和switch连用的方式，让我们可以快速的编写出不同类型接口的代码，如下案例，其中type是固定语法。且只能在switch中使用，无法单独使用。
```go
func do(i interface{}) {
	switch v := i.(type) {
	case int:
		fmt.Printf("Twice %v is %v\n", v, v*2)
	case string:
		fmt.Printf("%q is %v bytes long\n", v, len(v))
	default:
		fmt.Printf("I don't know about type %T!\n", v)
	}
}

func main() {
	do(21)
	do("hello")
	do(true)
}
```

### std预设接口
std预定义的接口Stringer，只要自定义类型实现String方法，则可以自定义toString的方法。
```go
type Stringer interface {
    String() string
}
```
error 接口，定义Error方法，用于实现自定义错误类型，
```go
type error interface {
    Error() string
}
```

### Exercise: Stringers

```go
type IPAddr [4]byte

func (i IPAddr) String() string {
	return fmt.Sprintf("%v.%v.%v.%v", i[0], i[1], i[2], i[3])
}

func main() {
	hosts := map[string]IPAddr{
		"loopback":  {127, 0, 0, 1},
		"googleDNS": {8, 8, 8, 8},
	}
	for name, ip := range hosts {
		fmt.Printf("%v: %v\n", name, ip)
	}
}
```

### Exercise: Errors
```go
type ErrNegativeSqrt float64

func (e ErrNegativeSqrt) Error() string {
	return fmt.Sprintf("cannot Sqrt negative number: %v", float64(e))
}

func Sqrt(x float64) (float64, error) {
	if x < 0 {
		return 0, ErrNegativeSqrt(x)
	}
	return 0, nil
}

func main() {
	fmt.Println(Sqrt(2))
	fmt.Println(Sqrt(-2))
}
```

### goroutine 
### Channels
### Select
