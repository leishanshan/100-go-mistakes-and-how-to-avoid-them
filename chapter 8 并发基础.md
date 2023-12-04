## 🤔55.混淆并发和并行
并行是同时做很多事，并发是同时管理很多事情，提供了一套结构去解决问题，将一个任务拆分成多个任务，同时并发也支持并行


## 🤔56.认为并发就一定快
go调度
线程执行的两种方式
并发 — 两个或多个线程可以在重叠的时间段内启动、运行和完成
并行 — 相同的任务可以同时执行多次

GMP模型
goroutine只有下面几种状态
- executing 在m上执行
- runnable 可运行的，准备好进入执行状态
- waiting 运行程序停止并等待完成任务，如系统调用或同步操作获取互斥锁
- 还有一个阶段，goroutine被创建但是不能执行，其他M已经在执行G，go runtime会将这个G放入队列中，有两种队列，每个P一个本地队列，以及在所有P之间共享的全局队列

场景：归并排序
```
func sequentialMergesort(s []int) {
	if len(s) <= 1 {
		return
	}
	middle := len(s) / 2
	sequentialMergesort(s[:middle])
	sequentialMergesort(s[middle:])
	merge(s, middle)
}
func merge(s []int, middle int) {
	// ...
}
```
并行实现方式：
```
func parallelMergesortV1(s []int) {
	if len(s) <= 1 {
		return
	}
	middle := len(s) / 2
	var wg sync.WaitGroup
	wg.Add(2)
	go func() {
		defer wg.Done()
		parallelMergesortV1(s[:middle])
	}()
	go func() {
		defer wg.Done()
		parallelMergesortV1(s[middle:])
	}()
	wg.Wait()
	merge(s, middle)
}
```
测试两种方式运行时间，并行的慢了一个数量级

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/d846bf82-980f-4615-8271-9396016f380f)

假如切片有1024个元素，父协程起两个协程分别处理512个元素，两个协程又分别再起两个协程处理256个元素，直到起的协程只用处理单个元素，因为并行处理的工作负载太小了，归并元素的开销小于创建携程和调度执行的开销
解决方式：设置一个分治的阈值，低于阈值的直接就按原始方式处理，高于阈值使用协程并行处理
```
const max = 2048
func parallelMergesortV2(s []int) {
	if len(s) <= 1 {
		return
	}
	if len(s) <= max {
		sequentialMergesort(s)
	} else {
		middle := len(s) / 2
		var wg sync.WaitGroup
		wg.Add(2)
		go func() {
			defer wg.Done()
			parallelMergesortV2(s[:middle])
		}()
		go func() {
			defer wg.Done(
			parallelMergesortV2(s[middle:])
		}()
		wg.Wait()
		merge(s, middle)
	}
}
```
![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/780b316e-ec4c-4219-be81-8b2c97c95b19)

如果不确定是否用并发，可以用简单顺序的版本分析，同并发的方式做基准测试

## 🤔57.不清楚什么何时用channels还是mutexs

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/ab16e0df-eee4-4ecf-9d39-373ae7965993)

G1、G2并行，可能都在执行从channel接收消息的功能，或者都在处理http handler G3和G1,G2并发
一般来说并行的协程需要同步，例如，它们需要访问或者修改共享资源时，如切片，同步就必须用mutex而不是channel来实现。
但实际上相反，并发的协程必须协调，例如G3需要聚合G1和G2的结果，G1和G2就需要向G3发出信号表明有一个新的中间结果是可用的，这种协调需要channel完成

mutexs和channels具有不同语义，当我们想共享一个状态或者访问一个共享资源的时候，mutex可以确保对该资源的独占访问，而channel是一种信号机制，协调或资源转让权需要用channel完成，一般来说，并行程序使用mutex，并发程序使用channell


## 🤔58.不懂竟态问题
数据竞争：2个以上协程在同时访问同一块内存并至少有一个在写入
不产生数据竞争的几种方式
1.使用 atomic.Value使操作变成原子操作，原子操作使用sync/atomic包
```
var i int64
go func() {
	atomic.AddInt64(&i, 1)
}()
go func() {
	atomic.AddInt64(&i, 1)
}()
```
2.使用mutex
```
i := 0
mutex := sync.Mutex{}
go func() {
	mutex.Lock()
	i++
	mutex.Unlock()
}()
go func() {
	mutex.Lock()
	i++
	mutex.Unlock()
}()
```
3.使用channel （sync/atomic包只对特定数据类型有用，map,struct,slice就不能用）
```
i := 0
ch := make(chan int)
go func() {
	ch <- 1
}()
go func() {
	ch <- 1
}()
i += <-ch
i += <-ch
```
一个无数据竞争的程序是否就能产生一个确定的结果呢？
```
i := 0
mutex := sync.Mutex{}
go func() {
	mutex.Lock()
	defer mutex.Unlock()
	i=1
}()
go func() {
	mutex.Lock()
	defer mutex.Unlock()
	i=2
}()
```
虽然没有产生数据竞争，但是产生了竞争条件，当操作行为依赖于不可控制的执行顺序或者时间时，就会产生竞争条件。如果想保证goroutine按照指定顺序执行，channel是一种解决方式

**go内存模型**	
```
i := 0
ch := make(chan struct{})
go func() {
	<-ch
	fmt.Println(i)
}()
i++
ch <- struct{}{}
//执行顺序
variable increment < channel send < channel receive < variable read
```

## 🤔59.不了解不同工作负载类型的并发性影响
工作负载的执行时间受以下因素影响：
1.cpu速度-cpu绑定
2.I/O的速度-i/o绑定
3.可用内存量-内存绑定
```
func read(r io.Reader) (int, error) {
	count := 0
	for {
		b := make([]byte, 1024)
		_, err := r.Read(b)       //读1024字节
		if err != nil {
			if err == io.EOF {
				break
			}
			return 0, err
		}
		count += task(b)       //将task函数返回的结果相加
	}
	return count, nil
}
```
如果要并行实现task函数
一种方式是使用工作池，创建固定大小的协程池，通过读取公共的channel接收任务，然后原子的将更新counter计数器

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/39604650-aa61-4df0-9542-f5f3047c1f17)

```
func read(r io.Reader) (int, error) {
	var count int64    
	wg := sync.WaitGroup{}
	var n = 10
	ch := make(chan []byte, n)        //创建一个大小和工作池相同的channel
	wg.Add(n)
	for i := 0; i < n; i++ {        //创建工作池
		go func() {
			defer wg.Done()         //一旦从channel接收到任务就执行Done方法
			for b := range ch {     //每个协程从共享channel接收读取的内容
				v := task(b)
				atomic.AddInt64(&count, int64(v))       
			}
		}()
	}
	for {
		b := make([]byte, 1024)
		// Read from r to b
		ch <- b                 //每次读取后都会向channel发布新任务
	}
	close(ch)
	wg.Wait()
	return int(count), nil
}
```
关于这个问题，工作池的大小为多少最合适？
如果工作负载是I/O绑定，那取决于外部系统
如果工作负载是cpu绑定，最好是取决于GOMAXPROCS数量，一般设置为逻辑cpu数量

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/43edbd12-7b3e-4dca-a72e-eec23ab027eb)

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/b18e4774-55de-44d2-bcab-6e12efbde5b1)

![image](https://github.com/leishanshan/100-go-mistakes-and-how-to-avoid-them/assets/59813538/cf8a4125-790a-4c91-90d9-f8ccabce6cca)

以上的情况不是由开发者决定的，但是我们可以在CPU-绑定类型的工作负载下，基于GOMAXPROCS设置工作池的大小，如果内核数是4，但只有3个线程，应该起3个而不是4个协程，否则将会有两个协程共享一个线程的执行时间，这样会增加上下文切换的次数


## 🤔60.不理解go context
