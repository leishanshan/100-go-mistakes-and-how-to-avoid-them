## 🤔42. 不清楚接收器用什么类型（值还是指针？）
值接收器
```go
type customer struct {
	balance float64
}
func (c customer) add(v float64) {
	c.balance += v
}
func main() {
	c := customer{balance: 100.}
	c.add(50.)
	fmt.Printf("balance: %.2f\n", c.balance) 
}
```
指针接收器
```go
type customer struct {
	balance float64
}
func (c *customer) add(operation float64) {
	c.balance += operation
}
func main() {
	c := customer{balance: 100.0}
	c.add(50.0)
	fmt.Printf("balance: %.2f\n", c.balance)
}
```
使用哪一种接收器要看具体场景

1.必须使用指针接收器：
调用的方法需要修改接收器，如果接收器是一个切片且需要append值同样有效
```go
type slice []int
func (s *slice) add(element int) {
	*s = append(*s, element)
}
```
2.可以使用指针接收器：
接收器是一个比较大的对象，使用指针可以使调用更高效，因为这样做可以防止生成大量副本

3.必须使用值接收器
a.强制接收器的值不变
b.接收器是map、channel或function，不然会编译错误

4.可以使用值接收器
a.接收器是一个不需要修改的切片
b.接收器是一个很小的数组或结构体，比如time.Time
c.接收器是int、float之类的基本数据类型

有一种场景balance不直接属于customer,而是在一个由指针字段引用的结构体中，即便使用值接收器，调用add方法最终balance的值还是会改变
```go
type customer struct {
	data *data
}
type data struct {
	balance float64   
}
func (c customer) add(operation float64) {
	c.data.balance += operation
}
func main() {
	c := customer{data: &data{
		balance: 100,
	}
}
	c.add(50.)
	fmt.Printf("balance: %.2f\n", c.data.balance)
}
```


## 🤔43.从来不用已命名的输出参数
这节主要讲何时使用命名结果参数，使用api更方便

结果参数被命名时，方法或函数会将它初始化为零值，返回时也可以直接return不返回参数，结果参数的当前值就被用作返回的值
```go
func f(a int) (b int) {
	b = a
	return
}
```
使用命名结果参数的场景：

下面接口两个float32看不出是哪个是纬度哪个是经度，必须得看具体实现才能知道
```go
type locator interface {
	getCoordinates(address string) (float32, float32, error)
}
```
```go
type locator interface {
	getCoordinates(address string) (lat, lng float32, err error)  //这种情况最好还是用命名参数
}
```
不需要使用命名参数的场景
```go
func StoreCustomer(customer Customer) (err error) {
	// ...
}
```
下面场景也不需要使用命名参数，不仅不能提高可读性，反而代码看起来更困惑了，对于只return不返回参数，更适用于比较短的函数，否则会降低可读性，读者还得记住整个函数的输出是什么，而且还得保持一致，要么只return，要么return全部参数
```go
func ReadFull(r io.Reader, buf []byte) (n int, err error) {
	for len(buf) > 0 && err == nil {
		var nr int
		nr, err = r.Read(buf)
		n += nr
		buf = buf[nr:]
	}
	return
}
```
综合来讲，大多数情况下在接口的上下文中使用命名返回参数可以提高可读性，但是在具体的实现中，谨慎使用命名返回参数，如果是返回参数类型一样也可以提高可读性，其他情况谨慎

## 🤔44.使用命名结果参数的意外副作用
错误场景：

命名参数开始被初始化为零值，下面ctx.Err()不为空，返回的err永远都是nil
```go
func (l loc) getCoordinates(ctx context.Context, address string) (lat, lng float32, err error) {
	isValid := l.validateAddress(address)
	if !isValid {
		return 0, 0, errors.New("invalid address")
	}
	if ctx.Err() != nil {
		return 0, 0, err   //if err := ctx.Err(); err != nil {return 0, 0, err}
	}
	// Get and return coordinates
}
```

