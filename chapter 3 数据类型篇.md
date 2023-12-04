#  👀Data types 数据类型篇

## 🤔17.创建容易混淆的八进制字面量
错误示例：
```
sum := 100 + 010
fmt.Println(sum)
```
go中0开头的整型数字被认为是8进制数字，8进制的010是十进制的8，所以100+010是108
所以需要使用8进制的时候尽量用0o，0和0o效果是一样的，但是会使代码更易懂

## 🤔18.忽视整型溢出

## 🤔19.没理解浮点

## 🤔20.没理解切片slice的length和capacity
很多人容易把切片的length和capacity混淆
`s:=make([]int,3,6)`
第一个参数是length，第二个参数是capacity

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/73ed07f0-c9e2-4f1f-bbdd-cbd0f5527008)

切片通过append插入元素，如果超过capacity大小，会创建一个新的array，将之前的容量翻倍，当元素超过1024时，容量每次增长25%，初始array不再被引用，如果在堆上分配就会被gc回收
s1,s2引用同一个array

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/680bc066-9177-48e5-98ef-aff78e9ed620)

修改某一个值，s1和s2中的值都会改变

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/67a67b4b-dee0-4b40-8494-5798d4cc534b)

对s2 append新的值，s1不会被修改

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/50825e61-b590-4a82-8c6a-dd3e042beb7a)

如果对s2 append超过容量的值，会创建新的数组，s1和s2指向不同的array

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/ab01a38c-684a-45b8-92ec-c38818c7ca19)


## 🤔21.低效的切片初始化
用make初始化切片的时候尽量提供长度和容量，不然append值的时候会一直开辟新的空间并复制原来的array到新的array中，gc还需要努力回收，如果切片元素太大会降低性能
2种方式，一种分配cap不分配length，另一种分配length
```
func convert(foos []Foo) []Bar {
  n := len(foos)
  bars := make([]Bar, 0, n)
  for _, foo := range foos {
    bars = append(bars, fooToBar(foo))
  }
  return bars
}
```
```
func convert(foos []Foo) []Bar {
  n := len(foos)
  bars := make([]Bar, n)
  for i, foo := range foos {
    bars[i] = fooToBar(foo)
  }
  return bars
}
```

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/82c89425-5d4e-4bc1-a0f4-51fb65ea1704)

虽然第三种更快，但是和第二种性能相差不大，代码却更复杂，推荐第二种用append
```
func collectAllUserKeys(cmp Compare,tombstones []tombstoneWithLevel) [][]byte {
  keys := make([][]byte, 0, len(tombstones)*2)
  for _, t := range tombstones {
    keys = append(keys, t.Start.UserKey)
    keys = append(keys, t.End)
  }
  // ...
}
```
```
func collectAllUserKeys(cmp Compare,
tombstones []tombstoneWithLevel) [][]byte {
  keys := make([][]byte, len(tombstones)*2)
  for i, t := range tombstones {
    keys[i*2] = t.Start.UserKey  //这种方式代码可读性差
    keys[i*2+1] = t.End
  }
  // ...
}
```

## 🤔22.对nil和empty的切片混淆不清
nil切片是空的，但空切片不一定是nil
nil slice没有任何内存分配
empty slice长度为0
```
func main() {
  var s []string
  log(1, s)
  s = []string(nil)
  log(2, s)
  s = []string{}
  log(3, s)
  s = make([]string, 0)
  log(4, s)
}
func log(i int, s []string) {
  fmt.Printf("%d: empty=%t\tnil=%t\n", i, len(s) == 0, s == nil)
}
```

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/88c17e01-73db-4155-8553-6997f23fc515)

使用场景：
1.如果不确定切片长度以及切片是否能为空，使用 `var s []string`初始化
2.如果切片长度是已知的，就用`s=make([]string,length)`初始化
3.`s=[]string{}`在有初始化元素的时候使用


## 🤔23.检查slice为空的不恰当使用
不要用是否为nil判断切片是否为空，应该**取切片长度**来判断
错误示例：
```
func handleOperations(id string) {
  operations := getOperations(id)
  if operations != nil {
    handle(operations)
  }
}
func getOperations(id string) []float32 {
  operations := make([]float32, 0)
  if id == "" {
    return operations
  }
  // Add elements to operations
  return operations
}
```
检查长度
```
func handleOperations(id string) {
  operations := getOperations(id)
  if len(operations) != 0 {
    handle(operations)
  }
}
```


## 🤔24.错误使用切片copy函数
错误示范：

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/0d9724b5-6b47-48e7-b856-6ac5f194d760)

解决：
1. 创建一个给定长度的dst切片
```
src := []int{0, 1, 2}
var dst []int
copy(dst, src)
fmt.Println(dst)
```
2. 第二种方式
```
src := []int{0, 1, 2}
dst := append([]int(nil), src...)
```

## 🤔25.使用切片append产生的副作用
**错误示例：**
s[2]被修改为10
```
s1 := []int{1, 2, 3} 
s2 := s1[1:2]
s3 := append(s2, 10)
```
s1,s2,s3共享内存

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/0c543cf2-c623-4cf5-b111-6e4d4420e7c0)

```
func main() {
  s := []int{1, 2, 3}
  f(s[:2])
  fmt.Println(s) // [1 2 10]
}
func f(s []int) {
  _ = append(s, 10)
}
```
**解决方式：**
1.使用copy，缺点是代码可读性差，以及slice很大时使用copy开销大
```
func main() {
  s := []int{1, 2, 3}
  sCopy := make([]int, 2)
  copy(sCopy, s)
  f(sCopy)
  result := append(sCopy, s[2])
  // Use result
}
func f(s []int) {
  // Update s
}
```
2.使用全切片表达式，从一个已有切片中创建新的切片
`newSlice=oldSlice[low:high:max]`
```
func main() {
  s := []int{1, 2, 3}
  f(s[:2:2])
  // Use s
}
func f(s []int) {
  // Update s
}
```
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/7f8d6285-d0c6-4082-bd89-62c8407bc545)


## 🤔26.切片内存泄漏
**场景示例1：** 一条消息包含100万字节，前5字节表示消息类型，getMessageType计算消息类型，运行程序消耗1G内存，剩余空间因为切片的引用无法被gc掉
```
func consumeMessages() {
  for {
    msg := receiveMessage()
    // Do something with msg
    storeMessageType(getMessageType(msg))
  }
}
func getMessageType(msg []byte) []byte {
  return msg[:5]
}
```
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/cee086ae-cba7-4f26-aec9-5c356347393e)

**解决：**
```
func getMessageType(msg []byte) []byte {
  msgType := make([]byte, 5)
  copy(msgType, msg)
  return msgType
}
```
注意：这种情况最好也不要用全切片表达式，gc不会回收
**场景示例2：**
```
func main() {
  foos := make([]Foo, 1_000)
  printAlloc()
  for i := 0; i < len(foos); i++ {
    foos[i] = Foo{
      v: make([]byte, 1024*1024),
    }
  }
  printAlloc()
  two := keepFirstTwoElementsOnly(foos)
  runtime.GC()
  printAlloc()
  runtime.KeepAlive(two)
}
func keepFirstTwoElementsOnly(foos []Foo) []Foo {
  return foos[:2]
}
```
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/3c7226c8-f74c-40b0-b283-d30886ae4e86)

**解决：**
1.使用copy，因为返回的不是原切片，没有被引用，所以gc能回收
```
func keepFirstTwoElementsOnly(foos []Foo) []Foo {
res := make([]Foo, 2)
copy(res, foos)
return res
}
```
2.如果想保留1000个元素的底层容量，可以显式地将其余切片标记为nil，这里返回len为2，cap为1000的切片，GC可以回收剩下的998个
```
func keepFirstTwoElementsOnly(foos []Foo) []Foo {
for i := 2; i < len(foos); i++ {
foos[i].v = nil
}
return foos[:2]
}
```
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/069281d3-3f39-4a48-8d47-6c03f82db451)

使用哪一种取决于保留的i的比例，用第一种方式，就必须从0迭代到i，用第二种方式就必须从i迭代到n

## 🤔27.无效的Map初始化
map增长时，桶的数量翻倍，出现map增长的情况：
1、负载因子大于常量值（目前是6.5）
2、桶溢出了（超过一个桶的数量）
插入一个元素时，最坏的情况下复杂度是O(n)，因为内存不够map增长时，所有的keys会被重新分配到所有的桶里
正确的map初始化：
指定capacity，避免大量的复制操作
```
m := make(map[string]int, 1_000_000)
```


## 🤔28.Map的内存泄漏
下面代码：
1、分配一个空map
2、新增100万个元素
3、删除所有元素
```
n := 1_000_000
m := make(map[int][128]byte)
printAlloc()
for i := 0; i < n; i++ {
  m[i] = randBytes()
}
printAlloc()
for i := 0; i < n; i++ {
  delete(m, i)
}
runtime.GC()
printAlloc()
runtime.KeepAlive(m)
```
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/e096fb3d-9b83-4719-9af2-af9214159d8c)

```
type hmap struct {
  B uint8 // log_2 of # of buckets  (can hold up to loadFactor * 2^B items)
  // ...
}
```
添加100万元素，hmap中B的值是18，有2^18=262144个桶，移除元素之后桶的数量还是这么多

删除元素后，堆大小还是没有缩减，因为map中的桶的数量不能减少，所以从map中删除元素不会影响现有的桶的数量，它只是将桶里的东西归零  （map只能不断增长，桶的数量只能增加，不能减少）

解决方案：
案例 如果map存储客户数据，平时的数据不多，在出现活动的时候用户数量激增，之后用户减少，100万个用户的数据很大
用指针存储value，即便删除元素桶的数量还是那么多，但是每个桶条目为值保留一个指针大小，而不是128byte
map[int][128]byte    ---->  map[int]*[128]byte


## 🤔29.错误的值比较
可操作符直接比较的数据类型：comparable类型
bool
整型
浮点
strings
channels
interfaces
pointers
structs
arrays

不可比较类型：
比较方式1：使用reflect包 中的reflect.DeepEqual
接收的数据类型：arrays, structs, slices, maps, pointers, interfaces, and functions
```
var cust1 any = customer{id: "x", operations: []float64{1.}}
var cust2 any = customer{id: "x", operations: []float64{1.}}
fmt.Println(cust1 == cust2)
```
缺点：性能比操作符低，因为内部要花时间比较
比较方式2：自定义，如果对性能要求更高，这是个最好的解决方案
```
func (a customer) equal(b customer) bool {
  if a.id != b.id {
    return false
  }
  if len(a.operations) != len(b.operations) {
    return false
  }
  for i := 0; i < len(a.operations); i++ {
    if a.operations[i] != b.operations[i] {
      return false
    }
  }
  return true
}
```
