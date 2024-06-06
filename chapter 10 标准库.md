## 🤔75.提供了错误的time.Duration
标准库提供了接收time.Duration的通用函数和方法，time.Duration是int64类型的别名，对初学者可能会感到迷惑，并提供错误的time.Duration，这和java会有差异
错误示例：
下面创建了一个新的周期性定时器，每秒嘀嗒一次
```go
ticker := time.NewTicker(1000)
for {
	select {
	case <-ticker.C:
		// Do something
	}
}
```
运行代码会发现并不是每秒一次而是每微秒一次，那是因为time.Duration代表两个瞬间之间经过的时间，这个单位是纳秒，所有上面代码实质是1000纳秒=1微秒
另外，如果想创建一个间隔1微秒的time.Ticker，不应该直接传递一个int64，应该使用以下方式
```go
ticker = time.NewTicker(time.Microsecond)
// Or
ticker = time.NewTicker(1000 * time.Nanosecond)
```

## 🤔75.time.After和内存泄漏
对于将消息发送到channel前需要等待一段时间的场景，time.After是个比较方便的函数，返回channel，通常将它用在并发的代码里。
下面的示例，函数consumer重复使用来自通道ch的消息，如果超过1个小时没收到任何消息，日志打印warning
在每次迭代中time.After求值，如果超时就会重置。
```
func consumer(ch <-chan Event) {
	for {
		select {
		case event := <-ch:
			handle(event)
		case <-time.After(time.Hour):
			log.Println("warning: no messages received")
		}
	}
}
```
乍一看这段代码挺好的，但其实会产生内存泄露的问题。
time.After返回的是channel，我们可能期望通道在每次循环中都是关闭的，但并不是，time.After创建的资源会在超时的时候才会释放，如果收到大量的消息，比如每小时五百万条，调用time.After应用将会消耗1GB的内存来存储资源。
**解决方式：**
1. 使用context代替time.After
```
func consumer(ch <-chan Event) {
	for {
		ctx, cancel := context.WithTimeout(context.Background(), time.Hour)
		select {
		case event := <-ch:
			cancel()     //如果收到消息，取消上下文
			handle(event)
		case <-ctx.Done():
			log.Println("warning: no messages received")
		}
	}
}
```
2. 使用time.NewTimer
time.NewTimer，返回的结构包含通道C，Reset重置duration，和Stop()方法停止计时器
每次循环调用Reset方法比创建一个新的上下文简单得多，速度快，对gc压力小因为不用分配新的堆空间
```go
func consumer(ch <-chan Event) {
	timerDuration := 1 * time.Hour
	timer := time.NewTimer(timerDuration) //创建一个新计时器
	for {
		timer.Reset(timerDuration)  //重置duration
		select {
		case event := <-ch:
			handle(event)
		case <-timer.C:       //计时器到期
			log.Println("warning: no messages received")
		}
	}
}
```
补充：
time.After其实也依赖于 time.Timer，但time.After只返回C字段，无法访问Reset方法

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/56c412e3-c456-40a3-bcb2-a1a46f8cc4fc)

所以用time.After的时候需要小心，其创建的资源只有在计时器到期时才会被释放，可能对导致内存消耗过多，一般还是倾向于使用


## 🤔76.常见的json处理错误
encoding/json是常用库，这里总结了3种json处理时可能会出现的错误
**1. 嵌入类型引起的问题**
```
type Event struct {
	ID int
	time.Time     //嵌入类型，Event可以直接使用time中的方法
}
```
那这对json序列化有什么影响？
```
event := Event{
	ID: 1234,
	Time: time.Now(),  //Event实例化期间匿名域的名字是结构体Time
}
b, err := json.Marshal(event)
if err != nil {
	return err
}
fmt.Println(string(b))
```
期望得到的结果是
{"ID":1234,"Time":"2021-05-18T21:15:08.381652+02:00"}
最后实际打印的结果是
"2021-05-18T21:15:08.381652+02:00"
有一点需要知道，如果嵌入字段实现了一个接口，那么包含该嵌入字段的结构也将实现这个接口。
上面代码中`Time`实现了`json.Marshaler`接口，因为`time.Time`是`Event`的嵌入字段，所以编译器会提升它的方法，这样`Event`也实现了`json.Marshaler`方法，这就导致给`json.Marshal`传递的event会使用time的那个方法，ID字段就被忽略了
**解决方式：**
- 1.给time.Time添加字段名
```
type Event struct {
	ID int
	Time time.Time
}
```
- 2.如果要保留嵌入类型，可以实现Event的json.Marshaler接口，该接口包含MarshalJSON函数，但这个解决方案更麻烦，而且要确保MarshalJSON方法总是最新的
```
type Marshaler interface {
	MarshalJSON() ([]byte, error)
}

//实现定制的MarshalJSON方法
func (e Event) MarshalJSON() ([]byte, error) {
    return json.Marshal(
        struct {       //创建一个匿名结构体
            ID   int
            Time time.Time
        }{
            ID:   e.ID,
            Time: e.Time,
        },
    )
}

```
**2.json和单调时钟**

下面代码比较time序列化再反序列化后和原来是否相同
```
t := time.Now()
event1 := Event{
	Time: t,
}
b, err := json.Marshal(event1)
if err != nil {
	return err
}
var event2 Event
err = json.Unmarshal(b, &event2)
if err != nil {
	return err
}
fmt.Println(event1 == event2)
```
结果为false，打印event1和event2如下：
event1：`2024-06-04 16:47:08.7698749 +0800 CST m=+0.001621101`
event2：`2024-06-04 16:47:08.7698749 +0800 CST`
操作系统会处理两种时钟类型：墙上时钟和单调时钟，墙上时钟指当前时间，系统的这个时间可以被修改，单调时钟保证时间总是向前移动，是相对机器的，不受时间跳跃影响
所以在比较time.Time字段时可以使用Equal方法
```
fmt.Println(event1.Time.Equal(event2.Time))
true
```
或者在序列化时间的时候去掉单调时间
```
t := time.Now()
event1 := Event{
	Time: t.Truncate(0),
}
b, err := json.Marshal(event1)
if err != nil {
	return err
}
var event2 Event
err = json.Unmarshal(b, &event2)
if err != nil {
	return err
}
fmt.Println(event1 == event2)
```
**3.any类型的map**

在反序列化数据的时候，我们经常会用map而不是结构体，因为这样比较灵活，但也有需要注意的地方
```
b := getMessage()
var m map[string]any
err := json.Unmarshal(b, &m)
if err != nil {
	return err
}
```
对下面json反序列化，如果使用value为any类型的map，id的类型是float64，因为对使用any的map任何数值都被转换为float64。
