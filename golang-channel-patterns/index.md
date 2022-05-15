# Golang Channel通道应用模式


## 通道设计

通道允许goroutine通过发送接收信号相互通信，通过**发送/接收数据**或**识别特定通道上的状态变化**实现收发信号。不要以通道是队列的想法来使用它，应该聚焦通道**收发信号的行为**来简化goroutine的编排。

### 通道语义

* 使用通道去编排和协调goroutine	
	* 聚焦通道收发信号的语义而不是数据共享。
	* 收发的信号可以是数据也可以不是数据（通过关闭通道）。
* 不带缓冲区的channel
  * **接收**行为发生在**发送**之前。
  * 优点：可以保证发送的信号被接收。（因为没有缓冲区，发送接收操作都会阻塞直到完成）
  * 缺点：无法确定信号被接收的延迟。
* 带缓存区的channel
	* **发送**行为发生在**接收**之前。
  * 优点：减少收发信号之间的阻塞延迟。
  * 缺点：无法保证发送的信号被接收。
    * 缓冲区越大，保证越小。
    * 值为1的缓冲区可以给你一个延迟发送的保证。（很多时候我们只是为了尽量不阻塞发送端而设置这样一个缓冲区）
* 关闭通道
  * 关闭行为发生在接收之前
  * 不通过数据收发信号（通常使用chan struct{}类型的通道）
  * 非常适用于收发**取消**和**截止时间**信号的场景
* NIL 通道（未通过make调用分配内存的通道，例如：var ch chan int）
  * 对NIL通道的发送和接收都阻塞
  * 非常适用于实现**限速**和**暂停**的场景

### 设计理念

根据要解决的问题，可能需要不同的通道语义。根据需要的语义，必须选择不同的体系结构。

* 如果通道上的任何信号发送操作都**会**导致发送端的goroutine阻塞
  * 需要格外注意通道的缓冲值是否大于1，缓冲值大于1需要明确的理由
  * 必须明确当发送端groutine阻塞时发生了什么
* 如果通道上的任何信号发送操作都**不会**导致发送端的goroutine阻塞
  * 当将要发送的信号有准确的数量时
    * 选择**Fan Out**模式
  * 当知道缓冲区的最大容量（根据系统的压测结果）
    * 选择**Drop**模式
* 缓冲区越少越好
  * 在考虑缓冲区时不要考虑性能。
  * 缓冲区可以帮助减少收发信号之间的阻塞延迟。
    * 将阻塞延迟降至零并不一定意味着更高的吞吐量。
    * 如果一个缓冲区能给你足够好的吞吐量，那就保留它。
    * 质疑那些缓冲区大于1的代码设计，并测试他确实需要的大小。
    * 找到能提供足够高吞吐量的最小缓冲区。

## WaitForResult

```go
func waitForResult() {
	// 使用不带缓存的通道
	ch := make(chan string)

	go func() {
		time.Sleep(time.Duration(rand.Intn(300)) * time.Millisecond)
		fmt.Println("child: send signal~")
		ch <- "result"
	}()

	res := <-ch
	fmt.Println("parent: get signal: ", res)
}

/* 输出：
child: send signal~
parent: get signal:  result
*/
```



## WaitForTask

```go
func waitForTask() {
	const taskNum = 3 // 任务数量

	// 带缓冲区的channel
	ch := make(chan string, taskNum)

	wg := sync.WaitGroup{}
	wg.Add(taskNum)

	go func() {
		for {
			task := <-ch
			fmt.Println("child: get task: ", task)
			time.Sleep(time.Duration(rand.Intn(300)) * time.Millisecond)
			// 完成任务
			wg.Done()
		}
	}()

	for i := 0; i < taskNum; i++ {
		fmt.Println("parent: send task")
		ch <- fmt.Sprint("task", i)
	}

	wg.Wait()
	fmt.Println("all done")
}

/*
parent: send task
parent: send task
parent: send task
child: get task:  task0
child: get task:  task1
child: get task:  task2
all done
*/
```



## WaitForFinish

> 方法二用WaitGroup来实现更好，方法一只用于理解

```go
// 方法一：使用channel实现
func waitForFinish() {
	ch := make(chan struct{}) // 通道类型是空结构体，意味着收发信号是不带数据的，只是为了用来关闭

	go func() {
		time.Sleep(time.Duration(rand.Intn(500)) * time.Microsecond)
		fmt.Println("child: is finished")
		close(ch)
	}()

	_, withData := <-ch
	fmt.Println("parent: get signal: with data: ", withData)
}
/* 
child: is finished
parent: get signal: with data:  false
*/

// 方法二：使用WaitGroup实现
func waitForFinishByWG() {
	wg := sync.WaitGroup{}
	wg.Add(1)

	go func() {
		time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
		fmt.Println("child: is finished")
		wg.Done()
	}()

	wg.Wait()
	fmt.Println("parent: finish")
}

/* 
child: is finished
parent: finish
*/
```



## Pooling

```go
func pooling() {
	ch := make(chan string)
	wg := sync.WaitGroup{}

	// 任务数
	const taskNum = 10
	wg.Add(taskNum)

	// 获取系统逻辑cpu数
	c := runtime.GOMAXPROCS(0)
	fmt.Println("逻辑cpu数: ", c)

	// 根据核心数打开对应数量的goroutine
	for i := 0; i < c; i++ {
		go func(child int) {
			for data := range ch {
				fmt.Printf("child %d: get signal: %s\n", child, data)
				time.Sleep(time.Duration(rand.Intn(300)) * time.Millisecond)
				wg.Done()
			}
			fmt.Printf("child %d: shut down\n", child)
		}(i)
	}

	// 发布任务
	for i := 0; i < taskNum; i++ {
		ch <- fmt.Sprint("data", i)
	}

	fmt.Println("close channel")
	close(ch)
	wg.Wait()
	fmt.Println("all done")
}

/*
逻辑cpu数:  8
child 1: get signal: data7
child 5: get signal: data4
child 6: get signal: data5
...
child 4: get signal: data9
close channel
child 2: shut down
...
child 7: shut down
all done
*/
```



## FanOut

不适用于类似WebService本身有很多goroutine的服务，会导致goroutine数量膨胀

较适用于crontab任务，cli任务的场景

```go
func fanOut() {
	// 需要处理的任务数量
	taskNum := 300
	// 使用带taskNum数量缓冲区的通道，这样完成任务的child可以马上退出
	ch := make(chan string, taskNum)

	for i := 0; i < taskNum; i++ {
		go func(child int) {
			time.Sleep(time.Duration(rand.Intn(200)) * time.Microsecond)
			ch <- fmt.Sprint("chile", child)
		}(i)
	}

	for taskNum > 0 {
		res := <-ch
		taskNum--
		fmt.Println("get result from: ", res)
	}

	fmt.Println("all done")
}

/*
get result from:  chile8
get result from:  chile7
get result from:  chile2
get result from:  chile14
get result from:  chile9
get result from:  chile6
get result from:  chile0
get result from:  chile4
get result from:  chile3
...
all done
*/
```



## FanOutSemaphore

基于FanOut模式，增加semaphore通道实现对同时进行中的goroutine数量的限制

> 所有goroutine都已经提前生成，限制同时进行的数量只是为了缓解被访问方的压力

```go
func fanOutSem() {
	// 需要处理的任务数量
	taskNum := 300
	// 使用带taskNum数量缓冲区的通道，这样完成任务的child可以马上退出
	ch := make(chan string, taskNum)

	// 限制同时运行的goroutine数量
	// 可以用于缓解被访问方的压力
	c := runtime.GOMAXPROCS(0)
	sem := make(chan bool, c)

	for i := 0; i < taskNum; i++ {
		go func(child int) {
			sem <- true
			{
				time.Sleep(time.Duration(rand.Intn(200)) * time.Microsecond)
				ch <- fmt.Sprint("chile", child)
			}
			<-sem
		}(i)
	}

	for taskNum > 0 {
		res := <-ch
		taskNum--
		fmt.Println("get result from: ", res)
	}

	fmt.Println("all done")
}

/*
get result from:  chile4
get result from:  chile6
get result from:  chile2
get result from:  chile1
get result from:  chile8
get result from:  chile5
get result from:  chile0
get result from:  chile3
get result from:  chile12
...
all done
*/
```



## Drop

当服务超过额定容量时，主动放弃任务。用于当子goroutine阻塞或者超负荷时，不至于导致主goroutine的阻塞

```go
func drop() {
	// 容量
	cap := 100
	ch := make(chan string, cap)

	go func() {
		for data := range ch {
			fmt.Println("child: get signal: ", data)
		}
	}()

	// 任务数
	const task = 1000
	for i := 0; i < task; i++ {
		select {
		case ch <- fmt.Sprint("data ", i):
			fmt.Println("parent: send signal: ", i)
		default:
			fmt.Println("parent: drop task: ", i)
		}
	}

	close(ch)
	fmt.Println("parent : sent shutdown signal")

	time.Sleep(time.Second)
}
```



## Cancellation

超时取消

```go
func cancellation() {
	ch := make(chan string, 1)
	defer close(ch)

	// 设置50毫秒后超时
	duration := 50 * time.Millisecond
	ctx, cancel := context.WithTimeout(context.Background(), duration)
	defer cancel()

	go func() {
		// 任务执行100毫秒
		time.Sleep(100 * time.Millisecond)
		ch <- "finished"
	}()

	// 一旦进入select块，ctx就开始计时
	select {
	case d := <-ch:
		fmt.Println("parent: get signal: ", d)
	case <-ctx.Done():
		fmt.Println("parent: timeout")
	}
}
```



## RetryTimeout

超时重试

```go
func main() {
  // 设置超时时间为5秒
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
  
  checkFun := func(ctx context.Context) error {
    // 函数总是返回失败
		return errors.New("always fail")
	}
	retryTimeout(ctx, time.Second, checkFun)
  fmt.Println("finished")
}

func retryTimeout(
	ctx context.Context,
	retryInterval time.Duration,
	check func(ctx context.Context) error) {

	for {
		fmt.Println("调用传入的函数")
		if err := check(ctx); err == nil {
			fmt.Println("任务成功")
			return
		}

		fmt.Println("检查是否已超时")
		if ctx.Err() != nil {
			fmt.Println("超时一: ", ctx.Err())
			return
		}

		fmt.Printf("等待 %s 后重试\n", retryInterval)
		t := time.NewTimer(retryInterval)

		select {
		case <-ctx.Done():
			fmt.Println("超时二: ", ctx.Err())
			t.Stop()
			return
		case <-t.C:
			fmt.Println("重试")
		}
	}
}
```


