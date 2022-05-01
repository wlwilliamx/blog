---
layout: post
excerpt_separator: 
title: "Go Timer 详解以及 Reset 和 Stop 的详细用法"
description: 
thumb_image: 
tags: [go]
---

Go 的 Timer 计时器由于机制的原因导致在并发编程中容易出现错误或者 bug，但是只要了解清楚了其工作原理，就知道如何避免产生错误了。

## time.Timer

Timer 是 Go 中 `time` 包里的一种一次性计时器，它的作用是定时触发事件，在触发之后这个 Timer 就会失效，需要调用 `Reset()` 来让这个 Timer 重新生效。

{% highlight go %}
type Timer struct {
	C <-chan Time
	// contains filtered or unexported fields
}
{% endhighlight %}

这个是 Timer 类型的结构体，其中只有一个 channel 可供外部访问，这个 channel 的作用就是在定时结束结束之后，会发送当前时间到这个 channel 里面，所以在 channel 收到值的时候，就等于计时器超时了，可以执行定时的事件了。所以一般是和 `select` 语句搭配使用。

### Timer 的底层原理

在一个 Go 程序中，其中的所有计时器都是由一个运行着 `timerproc()` 函数的 goroutine 来维护。它采用了时间堆的算法来维护所有的 Timer，其底层的数据结构是基于数组的小根堆，堆顶的元素是距离超时最近的 Timer，这个 goroutine 会定期 wake up，读取堆顶的 Timer，执行对应的 `f` 函数或者 send time，然后将其从堆顶移除。

### time.NewTimer() 创建 Timer

{% highlight go %}
func NewTimer(d Duration) *Timer
{% endhighlight %}

`time.NewTimer()` 是创建 Timer 的其中一种方式，通过传入一个定时时间 `d`，`time.NewTimer()` 就会返回创建的 Timer 的指针，这个 Timer 会在经过 `d` 这么长的时间时间之后，向 Timer 中的 channel 发送当前时间。

{% highlight go %}
timer := time.NewTimer(5 * time.Minute)
select {
    case <-timer.C:
       fmt.Println("timed out")
    default:
}
{% endhighlight %}

在 Timer 超时之后，`select` 就会收到 channel 里发送的值，这样就可以往下执行定时事件了。

### Stop() 中止 Timer

{% highlight go %}
func (t *Timer) Stop() bool
{% endhighlight %}

`Stop()` 是 Timer 的成员函数，调用 `Stop()` 方法，会中止这个 Timer 的计时，使其失效，之后是不会触发定时事件的。

调用 `Stop()` 方法之后，会将这个 Timer 从时间堆里移除，如果这个 Timer 还没超时，依然在时间堆中，那么就会被成功移除并且返回 `true`；如果这个 Timer 不在时间堆里，说明已经超时了或者已经被 stop 了，这个时候就会返回 `false`。

### Reset() 重置 Timer

{% highlight go %}
func (t *Timer) Reset(d Duration) bool
{% endhighlight %}

`Reset()` 是 Timer 里的另一个成员函数，它的作用是重置这个 Timer。如果这个 Timer 已经超时失效了，那么 `Reset()` 会令其重新生效；如果这个 Timer 还没超时，那么 `Reset()` 会让其重新计时，并将超时时间设置为 `d`。

这里有一个需要**注意**的地方，在官方的 package 文档中，有这么一句话：

*For a Timer created with NewTimer, Reset should be invoked only on stopped or expired timers with drained channels.*

意思是调用 `Reset()` 之前，一定要保证这个 Timer 已经被 stop 了，或者这个 Timer 已经超时了，并且里面 channel 已经被排空了。

因为，如果这个 Timer 还没超时，但是不去保证这个 Timer 已经被 stop 了，那么旧的 Timer 依然存在时间堆里，并且依然会触发，就会产生意料之外的事。而如果这个 Timer 已经超时了，不在时间堆里了，但是可能是刚刚超时，并且往 channel 里发送了时间，如果不显式排空 channel 的话，那么也会触发超时事件，所以需要显式地排空 channel。

所以正常情况下，`Reset()` 要和 `Stop()` 一起搭配使用。官方文档里给出了示例：

{% highlight go %}
if !t.Stop() {
	<-t.C
}
t.Reset(d)
{% endhighlight %}

这样可以同时保证这个 Timer 已经被 stop 了，或者这个 Timer 已经超时了，但是对 channel 进行了显式排空。

但是这里**存在一个问题**，在正常情况下，如果之前的 Timer 还生效，那么 `Stop()` 会返回 `true`，不会产生问题；但是如果 Timer 已经超时了，`Stop()` 就会返回 `false`，而如果 channel 里面没有没有值，那么就会发生**阻塞**，导致程序卡在这里。

所以更好的做法是采用 `select`：

{% highlight go %}
if !t.Stop() {
    select {
    case <-t.C: // try to drain the channel
    default:
    }
}
t.Reset(d)
{% endhighlight %}

这样即使 channel 里面没有值，也不会发生阻塞，有值的话也可以成功排空 channel。

但是，显式排空 channel 并不是绝对的，如果 channel 里面存在值，但是对你想要的结果不会产生任何影响的话，那么不显式排空 channel 也是可以的，直接在 `Reset()` 之前调用一次 `Stop()` 就行，也不需要对 `Stop()` 的返回值进行判断。

### time.AfterFunc() 创建 Timer

对于 `time.Timer`，还有另一种创建方式：`time.AfterFunc()`。（time 包里还有其他几种计时器，这篇文章只讨论 time.Timer 这种计时器）

{% highlight go %}
func AfterFunc(d Duration, f func()) *Timer
{% endhighlight %}

`time.AfterFunc()` 与 `time.NewTimer()` 相比，参数上多了一个 `f`。这是因为 `time.AfterFunc()` 创建的 Timer 在超时之后会在一个新的 goroutine 中执行这个 `f` 函数，不会向 channel 里面发送值。

之前所讨论的需要 `Stop()` 之后显式排空 channel 的情况都是对于 `time.NewTimer()` 创建的 Timer 来说，对于 `time.AfterFunc()` 来说，由于不会向 channel 里发送值，所以不需要显式排空 channel 的额外操作，但是在 `Reset()` 之前还是需要调用 `Stop()` 的。

此外，`Stop()` 和 `Reset()` 的返回值对于 `time.AfterFunc()` 创建的 Timer 来说含义与**之前提到的**是不一样的。需要自己视情况而定，看 `f` 函数的**执行与否**对结果来说有没有影响，如果有影响，那么就需要额外判断返回值，如果没有影响，直接调用即可。

***

## 参考资料

[1] [论golang Timer Reset方法使用的正确姿势](https://tonybai.com/2016/12/21/how-to-use-timer-reset-in-golang-correctly/)

[2] [pkg/time](https://pkg.go.dev/time#Timer)

[3] [How Do They Do It: Timers in Go](https://blog.gopheracademy.com/advent-2016/go-timers/)

[4] [Golang Timer 源码探索及常见问题](https://lifan.tech/2019/10/30/golang/high-performance-timer/)
