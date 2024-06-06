**本章概要：**

1、防止一些goroutine和channel的常见错误

2、理解使用标准数据结构对并发代码的影响 

3、标准库的使用和一些扩展 

4、避免数据竞争和死锁 


## 🤔61.传递了不合适的context
HTTP handler处理一些任务然后返回response，在返回response之前我们想把它发送到kafka topic，但不想增加消费者的延时，所以起了一个新的协程并行处理publish（假设我们有一个可以接收上下文的publish函数，如果上下文被取消，发布消息的操作可以被中断）
错误示例：
```
func handler(w http.ResponseWriter, r *http.Request){
	response, err := doSomeTask(r.Context(), r)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	go func() {
		err := publish(r.Context(), response)
		// Do something with err
	}()
	writeResponse(response)
}
```
存在的问题：
我们必须知道附加在HTTP请求的上下文可以在下面几种情况下取消：
- 当客户端关闭时
- 在HTTP/2请求的情况下，请求被取消时
- 当response已经被写回客户端时

前两种情况下，这段代码可以正确的处理，例如从doSomeTask函数拿到response，这时客户端关闭了连接，那调用publish传过去的是取消的上下文，消息不会被发布
但针对最后一种情况，响应被写入客户端，与请求相关联的上下文将被取消，这就会面临一种竞争状态：
- 如果响应是在kafka发布之后写的，返回response并成功发布消息
- 如果是在发布前或者发布期间写的，消息则不应该被发布

最后一种场景下，调用publish将返回错误，因为我们快速返回了http响应
那如何解决这个问题？
（1）一种方式是不传原始上下文，使用空的上下文调用publish，这样不管写回http响应需要多长时间，都可以调用publish
```
err := publish(context.Background(), response)
```
但如果上下文包含有用的值呢？例如上下文包含用于分布式跟踪的关联ID，我们可以将HTTP请求和Kafka发布关联起来，理想情况下是希望有一个新的context与原来的context分离但依然能传递有用值，标准库没有这种方式但可以自定义上下文，区别是不携带取消信号
context.Context是个含4个函数的interface
```
type Context interface {
	Deadline() (deadline time.Time, ok bool)   //管理上下文的截止时间
	Done() <-chan struct{}   //通过done和err方法管理取消信号，截止时间已过或者上下文被取消时，Done应该返回一个关闭的channel
	Err() error           //err返回错误
	Value(key any) any     //通过value方法传这些值
}

type detach struct {            //自定义结构充当初始上下文的包装
	ctx context.Context     
}
func (d detach) Deadline() (time.Time, bool) {
	return time.Time{}, false
}
func (d detach) Done() <-chan struct{} {
	return nil
}
func (d detach) Err() error {
	return nil
}
func (d detach) Value(key any) any {
	return d.ctx.Value(key)    //将获取值的调用委托给初始上下文
}
```
除了调用原始上下文获取值的value方法外，其他的方法都返回默认值，所以上下文永远不会过期或取消
```
err := publish(detach{ctx: r.Context()}, response)
```
传context要谨慎，一旦返回响应，上下文呢就会被取消，所以异步操作也可能会意外停止，如果有必要，可以为特定的操作自定义上下文


## 🤔62.启动了一个协程却不知道何时停止它
起一个协程很容易开销也很小，一般没有必要专门做计划去停止一个goroutine，但容易导致内存泄露
一个协程的最小栈大小是2KB，可以根据需要增加或减少，64位最大栈大小是1GB。一个goroutine可以保存HTTP或数据库连接、打开的文件和套接字等资源，这些资源应该被正常关闭，如果goroutine被泄露，这些资源也会被泄露
```
ch := foo()
go func() {
	for v := range ch {
		// ...
	}
}()
```
这段代码中，创建的go协程会在ch关闭的时候退出，但这个channel无法知道什么时候关闭，因为ch是在foo函数中创建的，如果这个channel不会关闭，就会造成内存泄漏
**错误示例：**
设计了一个需要监视外部配置的应用（例如使用数据库连接）
```
func main() {
	newWatcher()
	// Run the application
}
type watcher struct { /* Some resources */ }

func newWatcher() {
	w := watcher{}
	go w.watch()     //创建了一个监视外部配置的协程
}
```
存在的问题：当主协程退出的时候（可能是因为os信号或者其工作负载有限），应用就会停止，这样由watcher创建的资源没有优雅的关闭
一种选择是，当main返回时给newWatcher传一个被取消的上下文
当上下文取消时，watcher结构应该关闭它的资源，但这有个设计缺陷，我们无法保证watch有时间这样做，如果我们要使用信号来传达就必须停止goroutine，直到资源关闭后才阻塞父协程
```
func main(){
	ctx,cancel:=context.WithCancel(context.Background())
	defer cancel()
	
	newWatcher(ctx)
	//Run the application
}

func newWatcher(ctx context.Context){
	w:=watcher{}
	go w.watch(ctx)
}
```
**解决方式：**
延迟调用close方法，使用defer来保证在应用退出之前关闭资源，而不是用信号通知watcher该关闭资源了
```
func main(){
	w:=newWatcher()
	defer w.close()
}

func newWatcher() watcher{
	w:=watcher{}
	go w.watch()
	return w
}

func (w watcher) close(){
	//close resources
}
```
goroutine和别的资源一样，最终都必须被关闭以释放内存或其他资源，所以一定要明确何时停止，同时要注意如果一个goroutine创建资源，其生命周期和应用的生命周期绑定在一起，那退出应用前等这个goroutine结束会更安全，这样可以确保释放资源。

## 🤔63.对goroutine和循环变量使用不够小心
错误处理goroutine和循环变量可能是go开发常见的错误
**错误示例：**
下面代码初始化一个切片，然后在新的goroutine执行的闭包中访问这个元素
```
s := []int{1, 2, 3}
for _,i:=range s {
	go func() {
		fmt.Println(i)
	}()
}
```
我们可能期望这段代码不按特定的顺序打印123，但这个代码实际输出不确定，结果可能为233或者333。
这段代码中，从闭包中创建新的goroutines，所有的协程都引用相同的变量i，在每次迭代中都是有一个新的协程，但我们无法保证goroutine什么时候开始和完成，所以结果也会不同。

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/ea70490d-c6bd-4bb1-ba95-a2ca08115f0e)

**解决方式1：** 每次循环中创建一个局部变量val
```
for _, i := range s {
	val := i
	go func() {
		fmt.Println(val)
	}()
}
```
**解决方式2**：通过函数参数将 i 传进去
```
for _,i:=range s{
	go func(val int){
		fmt.Println(val)
	}(i)
}
```

## 🤔64.使用select和channels时预期的确定行为
开发人员在使用channel时犯的一个常见错误是对select如何使用多个channel做出错误的假设，这会导致bug且很难察觉出来
假设我们想要实现一个需要从两个channel接收数据的goroutine，我们要优先考虑messageCh，以确保返回之前我们收到了所有的消息
```
for {
	select {
		case v := <-messageCh:    //messageCh为待处理的新消息
			fmt.Println(v)
		case <-disconnectCh:     //disconnectCh接收断连的通知
			fmt.Println("disconnection, return")
		return
	}
}
```
我们使用select从多个channel接收数据，由于我们优先考虑messageCh，我们会假设应该先写messageCh再写disconnectCh
```
for i := 0; i < 10; i++ {
	messageCh<-i
}
disconnectCh <- struct{}{}
```
和switch不同，如果有多个channel，select会随机选择一个
这种机制是为了避免出现饥饿，假设选择的第一个通信基于源顺序，这可能陷入一种情况，发送消息者很快，这样只能从一个通道接收，为了防止这种情况就设计为随机选择

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/2d7b8b2d-4ac6-4a8f-82df-c639db17c24d)

所以对上面的示例，即使`case v := <-messageCh`在第一个，也无法保证哪一个channel会被选择，示例的结果是不确定的，可能收到0个，5个或10个消息
**解决方式：**
- 如果只有一个生产者goroutine，有两个选择：
1. 让messageCh成为非缓冲channel，因为发送方goroutine会阻塞直到接收goroutine准备好，这种方式可以保证在disconnectCh断开连接之前，接收到来自messageCh的所有消息；
2. 用单个channel替代两个channel，我们可以定义一个struct既传送新消息也传送断连消息，通道保证发送消息的顺序和接收消息的顺序相同，这样就能确保最后接收到断开连接
- 如果遇到多个生产者goroutine，可能无法保证哪一个先写入，以上解决方式无论如何都会出现竞争的场景，可以用以下解决方案：
1.从messageCh或disconnectCh接收
2.如果接收到断开连接，阅读messageCh中所有的信息，没有了就返回（嵌套for/select）
```
for {
	select{
	case v:=<-messageCh:
		fmt.Println(v)
	case <-disconnectCh:
		for {            
			select {
			case v:=<-messageCh:
				fmt.Println(v)
			}
			default:
				fmt.Println("disconnection,return")
		}
	}
}
```

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/4fd89d49-731d-40d9-a46c-029014f2c456)

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/b04337d1-f201-45ae-9e2c-9db130465b86)

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/61b5e1db-cf5d-445a-809b-457773fa7f8e)

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/4c4a607a-41f6-4948-8c57-5133e65d2a87)

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/f9b1ef05-f353-4f5c-833c-0a74e1c274a0)

所以使用多通道的时候要注意，go随机选择通道，不能保证哪个选项会被选中，如果是单个生产者，可以使用无缓冲通道或者单个通道，如果有多个生产者goroutine，可以使用嵌套select和default来选择。


## 🤔67.对通道大小困惑
使用make创建channel时，通道可以是有缓冲也可以是无缓冲的，常见的问题是不知道选哪一种，如果选择有缓冲channel，那size又需要多大
无缓冲通道创建方式
```
ch1 := make(chan int)
ch2 := make(chan int, 0)
```
使用无缓冲的channel，在接收方从channel中接收消息前发送方会被阻塞
有缓冲的创建方式
```
ch3 := make(chan int, 1)
ch3 <-1   //不被阻塞
ch3 <-2   //阻塞
```
- 无缓冲通道支持同步，保证两个goroutine在某个时刻处于已知状态：一个接收消息，另一个发送消息
- 缓冲通道不提供任何强同步，如果通道未满，生产者goroutine可以发消息并继续执行，唯一能保证的就是goroutine在消息发送之前不会收到消息
我们如果需要同步，就必须要用无缓冲通道，还有一种场景最好使用无缓冲通道：使用通知通道的情况下，notification是通过通道关闭（close(ch)）来处理的
```
var done =make(chan bool)
var msg string
func aGoroutine(){
	msg="hello,world"
	close(done)   //关闭通道
}

func main(){
	go aGoroutine()
	<-done
	fmt.Println(msg)
}
```
如果需要使用有缓冲通道，size需要多大？
一般使用的默认值是最小值1。需要用到其他size的场景有以下情况：
1. 使用工作池模式的场景，起了多个goroutine发送消息到共享通道。这种情况size会设成创建的goroutine数量
2. 使用通道实现限速问题时。例如我们需要限制请求数量来加强资源利用率，这时候应该根据限制来设置通道大小
如果不是上面的场景，设成其他的size需要谨慎点，决定一个队列大小并不是一个简单的事，这是cpu和内存之间的平衡，size越小面临的cpu竞争就越多，size越大需要分配的内存就更多
