# chapter 2 编码以及项目架构
## 🤔1.Unintended variable shadowing 意想不到的变量阴影
variable shadowing：在go中，一个变量在一块区域声明之后还能在内层模块中重新声明
下面的代码，外层内层都定义了client，因为内层用了:=符号声明，所以外层的client是空的，这段代码要不是因为log里面使用了client，一般情况会报错，指示有client变量已声明或者未使用
```go
var client *http.Client
if tracing {
  client, err := createClientWithTracing()
  if err != nil {
    return err
  }
  log.Println(client)
} else {
  client, err := createDefaultClient()
  if err != nil {
    return err
  }
  log.Println(client)
}
// Use client
```
那如何保证给原始client变量赋了值呢？两种方式
第一种：在内层模块里面使用临时变量，然后再把临时变量赋给client
```go
var client *http.Client
if tracing {
  c, err := createClientWithTracing()
  if err != nil {
    return err
  }
  client = c
} else {
  // Same logic
}
```
第二种：在内层模块直接用=赋值


## 🤔2.Unnecessary nested code 没必要的嵌套代码
错误示范：
```go
func join(s1, s2 string, max int) (string, error) {
  if s1 == "" {
    return "", errors.New("s1 is empty")
  } else {
    if s2 == "" {
      return "", errors.New("s2 is empty")
    } else {
      concat, err := concatenate(s1, s2)
      if err != nil {
        return "", err
      } else {
        if len(concat) > max {
          return concat[:max], nil
        } else {
          return concat, nil
          }
        }
      }
  }
}
func concatenate(s1 string, s2 string) (string, error) {
  // ...
}
```
正确示范：
```go
func join(s1, s2 string, max int) (string, error) {
  if s1 == "" {
    return "", errors.New("s1 is empty")
  }
  if s2 == "" {
    return "", errors.New("s2 is empty")
  }
  concat, err := concatenate(s1, s2)
  if err != nil {
    return "", err
  }
  if len(concat) > max {
    return concat[:max], nil
  }
  return concat, nil
}
func concatenate(s1 string, s2 string) (string, error) {
  // ...
}
```
函数嵌套层数越多可读性越差越难理解
if语句如果有返回值，后面就不要用else了
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/4420f0df-584c-43f6-952c-fcb4d76581a0)

也可以像下面这样
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/81016f08-33d1-468f-9a36-ed27536abab5)



## 🤔3.Misusing init functions 滥用init函数
1.main包和redis包，因为main引用了redis中的函数，所以先执行redis包里的init函数，然后执行main中的init函数，最后执行main函数
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/7ed15096-ae1b-4db4-afc8-d8af53c5a06d)

2.在同一个包里或者同一个go文件中可以定义多个init函数，一个go文件里init按顺序执行，同一个包不同go，按照go文件命名的字母顺序执行，比如a包里 有a.go,和b.go，先执行a.go再执行b.go中的init
危险在于源文件能够重命名，这会影响执行顺序
```go
package main
import (
"fmt"
_ "foo"
)
func main() {
  // ...
}
```
**错误示例：**
```go
var db *sql.DB
func init() {
  dataSourceName :=os.Getenv("MYSQL_DATA_SOURCE_NAME")
  d, err := sql.Open("mysql", dataSourceName)
  if err != nil {
    log.Panic(err)
  }
  err = d.Ping()
  if err != nil {
    log.Panic(err)
  }
  db = d
}
```
- 首先，init函数里面的容错管理比较局限，因为init没有入参也不返回，唯一能捕获error的方式就是panic，但如果数据库连接出现问题，这样整个程序就会挂掉
- 另一个缺点就是，要写单元测试的时候，init会在测试用例之前执行，如果只是测试一个工具函数，不需要数据库连接，这样init就把单元测试搞复杂了
- 第三个缺点，示例里面数据库连接必须要定义成全局变量，但是全局变量有严重的缺点，包里的任何函数都能修改全局变量，其实更应该封装一个变量，而不是保持它的全局性
应该就把前面的初始化当简单函数处理，错误处理应该交给函数调用者，而且数据库连接池也封装起来了
```go
func createClient(dsn string) (*sql.DB, error) {
  db, err := sql.Open("mysql", dsn)
  if err != nil {
    return nil, err
  }
  if err = db.Ping(); err != nil {
    return nil, err
  }
  return db, nil
}
```
**正确示例：**
下面init函数不会报错，因为http.HandleFunc会panic，而且也是有在handle为nil的时候会panic，示例不会有这种情况，同时示例里面也不用创建全局变量，函数也不会影响单元测试，
```go
func init() {
  redirect := func(w http.ResponseWriter, r *http.Request) {
    http.Redirect(w, r, "/", http.StatusFound)
  }
  http.HandleFunc("/blog", redirect)
  http.HandleFunc("/blog/", redirect)
  static := http.FileServer(http.Dir("static"))
  http.Handle("/favicon.ico", static)
  http.Handle("/fonts.css", static)
  http.Handle("/fonts/", static)
  http.Handle("/lib/godoc/", http.StripPrefix("/lib/godoc/",
  http.HandlerFunc(staticHandler)))
}
```
init一般在定义静态配置的时候有用，大多数情况还是用特定函数来初始化


## 🤔4.Overusing getters and setters / getters和setters过度使用
go里面没有强制的要使用getters和setters获取数据，一般情况下也不需要用，只有在比如后面要添加新功能，要隐藏内部实现更灵活的时候用，用也要按下面的方式用

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/4b115add-a163-4e23-b9f0-811f6a7a3152)

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/9f4d93b3-df93-4eca-9141-2fac5753052c)

## 🤔5.Interface污染
设计接口的时候注意，interface越大抽象能力越弱

用interface的场景：
- 1.Common behavior 通用功能
比如集合排序，可以抽象为3种方法（Len、Less、Swap），
- 2.Decoupling 解耦
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/274a7165-7bed-46b8-a74d-b993dc3cf46c)

如果要进行单元测试，需要创建一个新的customer并存储，
如果用上面的方式，因为customerService依赖具体的实现来存储customer，测试的时候不得不需要集成测试，还得启动一个mysql实例
为了更灵活一点，将customerService和具体的实现解耦，通过接口来存储customer，这样测试改方法的时候更灵活，既可以用集成测试，也可以通过模拟使用单元测试
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/f97f2531-8259-4c5f-b3bf-d7257215f860)

- 3.Restricting behavior  限制行为
示例：
实现一个custom配置包处理动态配置
```go
type IntConfig struct {
  // ...
}
func (c *IntConfig) Get() int {
  // Retrieve configuration
}
func (c *IntConfig) Set(value int) {
  // Update configuration
}
```
现在有一个threshold的配置，只能读不能对它更新，如果不想更改配置包怎么实现？
通过创建一个接口来约束对配置的只读功能
```go
type intConfigGetter interface {
Get() int
}

type Foo struct {
  threshold intConfigGetter
}
func NewFoo(threshold intConfigGetter) Foo {
  return Foo{threshold: threshold}
}
func (f Foo) Bar() {
  threshold := f.threshold.Get()
  // ...
}
```
Don’t design with interfaces, discover them.
建议接口不要过度设计，需要用的时候再创建，接口过度设计会导致代码很复杂可读性差
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/0f15c821-45c0-4fbc-b2d6-bfbd2790be1c)


## 🤔6.生产端的interface
接口应该放在哪？
第一种放在生产侧，指接口的定义和接口功能具体实现放在一个包
第二种放在消费侧，指接口的定义和外部调用接口的客户端放一起

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/558485b8-415c-461d-b393-4a4ba118eaad)

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/92cc92f6-92b3-4cf0-8ee2-b5b645c984b9)

在其他语言里使用第一种，但是在go里面一般用第二种方式，前面说过接口不要过度设计，需要用的时候再创建，所以更多的是根据消费侧的需求决定合适的抽象方式
因为客户端可能只需要用接口里的一个方法，这种情况就可以直接在客户端创建一个interface
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/efe56c3e-9df5-4ae5-a7c3-f4e5aac9966e)

也有一些场景例外，例如标准库里面，encoding包定义了interface，在子包encoding/json，encoding/binary中实现，这种一般是开发者很清楚这些接口是有用的


## 🤔7.Returning interfaces 返回接口
设计函数的时候，可以返回接口，也可以返回具体实现，如果返回接口会产生循环依赖的问题
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/dc8f023b-dfae-4111-ae13-daf3a93f73db)

store包中InMemoryStore struct实现了在client包中定义的store接口，NewInMemoryStore又返回client的store接口，这样client包就无法调用NewInMemoryStore函数，不然就会产生循环依赖
返回接口会限制灵活性，所以应该
- 返回structs而不是interfaces
- 入参尽可能接收interfaces
在有一种情况下也可以返回interface，就是很明确知道接口对client包有用时
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/c1c71959-5426-4c27-b383-96b09154c6b9)

## 🤔8. any 不传达任何信息
any是空接口的别名，可以用来代替interface{}
any缺乏表达能力，后面维护的人还得去看文档或者实现代码才能理解
```go
package store
type Customer struct{
  // Some fields
}
type Contract struct{
  // Some fields
}
type Store struct{}
func (s *Store) Get(id string) (any, error) {
  // ...
}
func (s *Store) Set(id string, v any) error {
  // ...
}
```
编译不会有问题，但是入参和返回值无法表达有用的信息，而且这种可能会有啥类型都调用的情况，比如int
```go
s := store.Store{}
s.Set("foo", 42)
```
还不如把每个结构体的get和set方法都写一遍，减少不易理解的风险，虽然这样方法会很多，但是一般client包会使用interface，只封装自己需要的方法
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/03975e73-0de7-4094-9d07-86b48dbeaa60)

**可以使用any的场景**
示例1：encoding/json包中的Marshal函数
示例2：database/sql包，如果query字段参数化，也可以用any参数
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/19590dbb-bba5-4170-a7b5-660a0ba4078b)


## 🤔9. 困惑何时使用泛型
示例：
需要获取`map[string][int]`中的string
```go
func getKeys(m map[string]int) []string {
  var keys []string
  for k := range m {
    keys = append(keys, k)
  }
  return keys
}
```
如果后面又新增获取`map[int][string]`的需求，写两个函数或者用switch case的方式都会有代码冗余，而且用switch case方式返回类型必须是any，检查类型是在运行的阶段进行的而不是编译的阶段，所以还得return error，同时调用方还要做类型检查或者额外的类型转换
```go
func getKeys(m any) ([]any, error) {
  switch t := m.(type) {
    default:
    return nil, fmt.Errorf("unknown type: %T", t)
    case map[string]int:
    var keys []any
    for k := range t {
      keys = append(keys, k)
    }
    return keys, nil
    case map[int]string:
    // Copy the extraction logic
  }
}
```
使用泛型
```go
func getKeys[K comparable, V any](m map[K]V) []K {
  var keys []K
  for k := range m {
    keys = append(keys, k)
  }
  return keys
}
```
注意：map的key必须是可以比较的类型，切片、map和函数都不能直接比较，所有泛型这个的key的类型是comparable而不是any
也可以自定义约束类型
```go
type customConstraint interface {
  ~int | ~string
}
func getKeys[K customConstraint,V any](m map[K]V) []K {   //Changes the type parameter K to be a customConstraint type
  // Same implementation
}
```
调用方式，使用自定义约束类型之后这个函数会强制要求key类型必须是int或者string
```go
m = map[string]int{
  "one": 1,
  "two": 2,
  "three": 3,
}
keys := getKeys(m)
```
**使用泛型的场景：**
1.数据结构：二叉树、堆和链表。。
2.使用any类型切片、map或channel的函数，例如函数要合并两个any类型的channel
```go
func merge[T any](ch1, ch2 <-chan T) <-chan T {
  // ...
}
```
3.提取公用功能而不是变量类型的时候，例如sort包
```go
type Interface interface {
  Len() int
  Less(i, j int) bool
  Swap(i, j int)
}
```
这个接口可以被不同的函数sort.Ints或sort.Float64s使用
```go
type SliceFn[T any] struct {
  S []T
  Compare func(T, T) bool
}
func (s SliceFn[T]) Len() int { return len(s.S) }
func (s SliceFn[T]) Less(i, j int) bool { return s.Compare(s.S[i], s.S[j]) }
func (s SliceFn[T]) Swap(i, j int) { s.S[i], s.S[j] = s.S[j], s.S[i] }
```

## 🤔10. 没有意识到内嵌类型可能出现的问题

## 🤔11. 没有使用 functional options pattern

## 🤔12. 项目不分层

## 🤔13. 创建一些无用的工具包
比如util包，内涵newStringSet和sortStringSet的函数，调用的时候util这个包名毫无意义，应该把包名改成stringset，函数名new和sort就行了，这样也很容易理解

## 🤔14.忽略包名冲突
申明变量的时候变量名和包名冲突，导致包无法再次使用
同时也要注意避免变量名和内置函数的冲突
copy := copyFile(src, dst)

## 🤔15. 没有代码注释

## 🤔16.不使用静态分析器
