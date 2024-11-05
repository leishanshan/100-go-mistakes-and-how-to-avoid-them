# 👀control structures 控制结构篇

## 🤔30.忽视元素在range循环中是值拷贝
**示例：**
range循环遍历中，会将每个元素进行值拷贝，上面代码中遍历每个account struct会拷贝一个副本赋给a，所以`a.balance+=1000`改变的是临时变量a中的值，原切片中的元素是没有更新的

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/bfa52680-e0e7-4188-b70a-03b519708bc8)

**解决方式：**
用索引变量 i 访问切片元素，或者用传统循环不用range
```go
for i := range accounts {
  accounts[i].balance += 1000
}
for i := 0; i < len(accounts); i++ {
  accounts[i].balance += 1000
}
```

## 🤔31.忽视参数在range循环中的计算方式
讨论` for i, v := range exp`中，exp是怎么计算的，exp可以是数组、切片、map或者channel

**示例1：** 下面循环会不会终止？
```go
s := []int{0, 1, 2}
for range s {
  s = append(s, 10)
}
```
使用range循环，提供的表达式只在循环开始前被计算一次，然后将表达式拷贝到一个临时变量中，再对这个临时变量迭代访问，所以上面的循环迭代3次之后终止，每次迭代中原始切片s也在更新

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/0f3892de-dc28-4ef5-95bc-5edbf39351ba)

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/f46494bc-61b2-4497-82a1-e955f5e1f543)

用传统循环方式，因为在不断append元素，永远不会终止
```go
s := []int{0, 1, 2}
for i := 0; i < len(s); i++ {
  s = append(s, 10)
}
```
**示例2：** 对channel
创建两个goroutine，虽然下面ch2赋给了ch，但是前面先做了ch也就是ch1的拷贝，打印的值是0，1，2
```go
ch1 := make(chan int, 3)
go func() {
  ch1 <- 0
  ch1 <- 1
  ch1 <- 2
  close(ch1)
}()
ch2 := make(chan int, 3)
go func() {
  ch2 <- 10
  ch2 <- 11
  ch2 <- 12
  close(ch2)
}()
ch := ch1
for v := range ch {
  fmt.Println(v)
  ch = ch2
}
```
不过ch=ch2也不是没有影响，如果调用close(ch)关闭的是ch2而不是ch1

**示例3：** 对array
```go
a := [3]int{0, 1, 2}
for i, v := range a {
  a[2] = 10
  if i == 2 {
    fmt.Println(v)
  }
}
```
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/6b25e013-4342-405b-9536-5a8406ee4c76)

```go
a := [3]int{0, 1, 2}
for i := range a {
  a[2] = 10
  if i == 2 {
    fmt.Println(a[2])
  }
}
```
也可以使用切片的指针，这样就不用拷贝整个数组，如果数组非常大，用指针更好一点
```go
a := [3]int{0, 1, 2}
for i, v := range &a {
  a[2] = 10
  if i == 2 {
    fmt.Println(v)
  }
}
```

## 🤔32.忽视在range循环中使用指针元素的影响
range循环使用指针元素，如果不够谨慎可能会引用错误的元素
使用指针元素的切片或map的原理：
```go
type Store struct {
  m map[string]*Foo
}
func (s Store) Put(id string, foo *Foo) {
  s.m[id] = foo
  // ...
}
```
1.指针元素被Store结构体和put函数共享
2.有时候已经对指针进行操作了，所以直接在集合存储指针更方便
3.如果store结构体很大而且要频繁修改，用指针可以避免大量的拷贝修改
```go
func updateMapValue(mapValue map[string]LargeStruct, id string) {
  value := mapValue[id]
  value.foo = "bar"
  mapValue[id] = value
}
func updateMapPointer(mapPointer map[string]*LargeStruct, id string) {
  mapPointer[id].foo = "bar"
}
```
**错误示例：**
```go
type Customer struct {
  ID string
  Balance float64
}
type Store struct {
  m map[string]*Customer
}
```
定义了一个Customer结构体，以及Store结构体存储customer信息，storeCustomers对Customer切片用range遍历并将信息存到map中
```go
func (s *Store) storeCustomers(customers []Customer) {
  for _, customer := range customers {
    s.m[customer.ID] = &customer
  }
}
```
如果调用这个函数，打印map结果会是什么？
```go
s.storeCustomers([]Customer{
  {ID: "1", Balance: 10},
  {ID: "2", Balance: -10},
  {ID: "3", Balance: 0},
})

//下面是打印的值
key=1, value=&main.Customer{ID:"3", Balance:0}
key=2, value=&main.Customer{ID:"3", Balance:0}
key=3, value=&main.Customer{ID:"3", Balance:0}
```
错误原因：用range迭代，创建一个具有固定地址的customer变量
每一次迭代，都是同一个指针指向的不同customer，map中存储的是同一个指针，所以map中的值就是最后一次迭代更新的值

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/dc651f5c-dd56-48a1-a940-82ee1f152d22)

解决：
第一种方式：创建一个局部变量，不存储引用customer的指针，而是存储当前元素的指针，这样map中存储的是引用了不同customer的指针
```go
func (s *Store) storeCustomers(customers []Customer) {
  for _, customer := range customers {
    current := customer
    s.m[current.ID] = &current
  }
}
```
另一种方式：引用customer切片每一个元素的索引指针
```go
func (s *Store) storeCustomers(customers []Customer) {
  for i := range customers {
    customer := &customers[i]
    s.m[customer.ID] = customer
  }
}
```

## 🤔33.错误理解Map的迭代
**场景1：排序**

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/7dca2c44-183e-4311-903e-2c4ae643e84b)

map迭代不按照key排序，每次迭代的顺序也可能不一样

**场景2：map迭代时更新元素**
错误示范：
运行多次得到的结果会不一样
```go
m := map[int]bool{
  0: true,
  1: false,
  2: true,
}
for k, v := range m {
  if v {
    m[10+k] = true
  }
}
fmt.Println(m)
```
官方有说过，如果遍历map期间创建新的元素，后续的迭代中这个元素可能被修改也可能被跳过
如果需要对map m做更新，正确做法是copy一个副本m2，遍历m，但是对m2作更新

## 🤔34.忽视break的工作原理

## 🤔35.在循环内使用defer
