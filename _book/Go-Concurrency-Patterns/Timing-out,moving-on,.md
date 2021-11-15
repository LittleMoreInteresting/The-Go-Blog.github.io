# Go Concurrency Patterns: Timing out, moving on

Andrew Gerrand
23 September 2010



并发编程有它自己的套路。 比如说超时处理。 虽然Go的Channel 本身不支持支持超时处理，但容易实现。 假设我们希望从`channel` `ch`接收数据，但希望最多等待一秒。 我们首先创建一个信号`channel`，然后启动一个`goroutine`在一秒后向这个channel上发送信号

Concurrent programming has its own idioms. A good example is timeouts. Although Go’s channels do not support them directly, they are easy to implement. Say we want to receive from the channel ch, but want to wait at most one second for the value to arrive. We would start by creating a signalling channel and launching a goroutine that sleeps before sending on the channel:

```
timeout := make(chan bool, 1)
go func() {
    time.Sleep(1 * time.Second)
    timeout <- true
}()
```

然后，我们可以使用`select`来接收ch或timeout。 如果在一秒钟后没有任何东西到达ch，则选择timeout分支，并放弃从ch读取数据。  

We can then use a select statement to receive from either ch or timeout. If nothing arrives on ch after one second, the timeout case is selected and the attempt to read from ch is abandoned.

```
select {
case <-ch:
// a read from ch has occurred
case <-timeout:
// the read from ch has timed out
}
```

timeout channel 的容量为1，允许超时goroutine发送到channel然后退出。 goroutine不知道(或关心)是否收到了该值。 这意味着如果ch 在超时之前接收到数据，超时 goroutine 不会永远挂起。 超时通道最终将由垃圾回收器回收。 

```
time.After(1 * time.Second)
```

The timeout channel is buffered with space for 1 value, allowing the timeout goroutine to send to the channel and then exit. The goroutine doesn’t know (or care) whether the value is received. This means the goroutine won’t hang around forever if the ch receive happens before the timeout is reached. The timeout channel will eventually be deallocated by the garbage collector.

(在这个例子中，我们使用了`time.Sleep`来演示goroutines 和channels 之间的运行方式。 在真正的程序中，你应该使用 `time.After` 返回一个channel并在相应类型的channle上发送。)  

(In this example we used time.Sleep to demonstrate the mechanics of goroutines and channels. In real programs you should use time.After, a function that returns a channel and sends on that channel after the specified duration.)

让我们看看这个模式的另一种姿势。 在这个例子中，我们有一个程序，它可以同时从多个复制数据库中读取数据。 程序只需要其中一个查询结果，并且它应该接受最先到达的查询结果。  

Let’s look at another variation of this pattern. In this example we have a program that reads from multiple replicated databases simultaneously. The program needs only one of the answers, and it should accept the answer that arrives first.

Query函数接受一个数据库连接切片和一个查询字符串。 它并行地查询每个数据库，并返回它收到的第一个响应:  

The function Query takes a slice of database connections and a query string. It queries each of the databases in parallel and returns the first response it receives:

```
func Query(conns []Conn, query string) Result {
    ch := make(chan Result)
    for _, conn := range conns {
        go func(c Conn) {
            select {
            case ch <- c.DoQuery(query):
            default:
            }
        }(conn)
    }
    return <-ch
}
```

在本例中，闭包通过select语句返回查询结果到ch 然后使用default 分支来避免goroutine阻塞。 如果发送不能立即进行，将执行default分支。 确保在循环中启动的goroutine不会阻塞。 但是，如果结果在主函数执行接收前返回了，则发送可能会失败，因为channel 的接收端没有准备好读取。  

这个问题是所谓的竞态条件的典型例子，但是解决方法很简单。 我们可以使用缓冲通道(`ch := make(chan Result,1)`)，确保第一次发送有放置值的位置。 这将确保发送始终成功，并且无论执行顺序如何，都会检索到第一个到达的值。  

In this example, the closure does a non-blocking send, which it achieves by using the send operation in select statement with a default case. If the send cannot go through immediately the default case will be selected. Making the send non-blocking guarantees that none of the goroutines launched in the loop will hang around. However, if the result arrives before the main function has made it to the receive, the send could fail since no one is ready.

This problem is a textbook example of what is known as a race condition, but the fix is trivial. We just make sure to buffer the channel ch (by adding the buffer length as the second argument to make), guaranteeing that the first send has a place to put the value. This ensures the send will always succeed, and the first value to arrive will be retrieved regardless of the order of execution.

这两个示例演示了Go实现goroutine之间复杂交互通信是多么简单。  

These two examples demonstrate the simplicity with which Go can express complex interactions between goroutines.