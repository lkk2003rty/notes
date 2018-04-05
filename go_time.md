# time 包中定时器实现

在实际的开发中难免会用到定时器，而在 Go 的标准库中就提供了定时器相关的 API 供调用。那么在标准库中 Go 是怎么实现定时器的呢？扒拉代码看一下吧。

## 快速上手

在 time 包中与定时器相关的主要有这么几个函数。

- NewTicker 创建一个周期定时器，这个定时器每隔指定的时间往其自身的 channel 塞入数据 
- NewTimer 创建一个定时器，这个定时器到了指定时间后往其自身的 channel 塞入数据，然后就歇菜了
- AfterFunc 在指定的时间执行一个指定的函数，一次性操作，可以看成是 NewTimer 的简化版
- After 也是 NewTimer 的简化版
- Tick 实际上就是 NewTicker 的简化版

可以看到绕了半天主要就两种定时器，一种跑个不停的，一种是跑一次就完事的。那么这种两定时器的实现有啥区别呢？下面看一下这种两定时器的数据结构。

```go
 // A Ticker holds a channel that delivers `ticks' of a clock
 // at intervals.
 type Ticker struct {
     C <-chan Time // The channel on which the ticks are delivered.
     r runtimeTimer
 }
 
  // The Timer type represents a single event.
 // When the Timer expires, the current time will be sent on C,
 // unless the Timer was created by AfterFunc.
 // A Timer must be created with NewTimer or AfterFunc.
 type Timer struct {
     C <-chan Time
     r runtimeTimer
 }
 
  // Interface to timers implemented in package runtime.
 // Must be in sync with ../runtime/time.go:/^type timer
 type runtimeTimer struct {
     // 数组下标
     i      int
     when   int64
     period int64
     f      func(interface{}, uintptr) // NOTE: must not be closure
     arg    interface{}
     seq    uintptr
 }
```

其实长得都一样。再查看下 NewTimer 和 NewTicker 的实现，实际上都是把各自的结构体实例化，然后调用 `startTimer(&t.r)` 后返回实例。所以重点就是 startTimer 这个函数拉。再查看 startTimer 这个函数之前我们先考虑一个问题。既然是定时器有其启动的方法，那么必然有其停止的方法。很快就能发现其停止定时器的内部调用的是 `stopTimer(&t.r)` 这个方法。于是剩下的就是看一下 startTimer 和 stopTimer 这两个函数了。

这里需要注意下这两函数不在 time 这个文件夹中的任何文件中，它们在 runtime/time.go 这个文件里面哦。

通过这两个函数的注释可以看到 Go 对定时器的实现用到的数据结构是堆。

## startTimer

首先要注意的是 startTimer 的参数类型是 *timer，而这个名为 timer 的结构体实际上和上面说的 runtimeTimer 是一样的。看来是有某种机制将 runtime/time.go 中定义的 timer 和 time/sleep.go 中定义的 runtimeTimer 绑定在一起了。下面就统一用 timer 来指代 runtimeTimer 这个结构体。

在 runtime/time.go 中有一个全局变量，结构如下，startTimer 中做的事情主要也是针对这个结构进行的操作。

```go
 var timers struct {
     lock         mutex
     gp           *g
     created      bool
     sleeping     bool
     rescheduling bool
     sleepUntil   int64
     waitnote     note
     t            []*timer
 }
 
  // startTimer adds t to the timer heap.
 //go:linkname startTimer time.startTimer
 func startTimer(t *timer) {
     if raceenabled {
         racerelease(unsafe.Pointer(t))
     }
     addtimer(t)
 }
 
  func addtimer(t *timer) {
     lock(&timers.lock)
     addtimerLocked(t)
     unlock(&timers.lock)
 }

 // Add a timer to the heap and start or kick timerproc if the new timer is
 // earlier than any of the others.
 // Timers are locked.
 func addtimerLocked(t *timer) {
     // when must never be negative; otherwise timerproc will overflow
     // during its delta calculation and never expire other runtime timers.
     if t.when < 0 {
         t.when = 1<<63 - 1
     }
     t.i = len(timers.t)
     timers.t = append(timers.t, t)
     siftupTimer(t.i)
     if t.i == 0 {
         // siftup moved to top: new earliest deadline.
         if timers.sleeping {
             timers.sleeping = false
             notewakeup(&timers.waitnote)
         }
         if timers.rescheduling {
             timers.rescheduling = false
             goready(timers.gp, 0)
         }
     }
     if !timers.created {
         timers.created = true
         go timerproc()
     }
 }
```

通过以上代码可以看到
- 不管有多少个定时器，都只有有一个 goroutine 跑着，那就是 timerproc。它是干活的主力。
- 所有的定时器都保存在了 timers.t 这个 slice 里了。
- timers.t 虽然是个 slice，但实际上就是却实现了上面说的堆这一数据结构。

首先来看一下怎么用 slice 来实现堆这一数据结构的。这里的堆实际上是一个四叉树。每个元素有个 i 这个成员变量来表示这个元素在 slice 中的位置，实际上就是数组下标。在这个四叉树中，父节点的 when 这个成员变量都比子节点的要来得小。这个堆对应的操作如下：

- 插入新元素 t
  - 设置好 t.i
  - 通过 append 将 t 追加到 slice 中
  - 调用 siftupTimer(t.i) 方法将元素位置调整到正确的位置
 
- 删除元素 t
  - 将 t 与 timers.t 的最后一个元素互换位置
  - 缩短 timers.t
  - 调用 siftupTimer(t.i)
  - 调用 siftdownTimer(t.i)

这里 siftupTimer 和 siftdownTimer 的作用分别如下：

- siftupTimer 不断将该元素与其父节点元素的 when 比较，小就往上挪
- siftdownTimer 不断将该元素与其子节点元素中最小的 when 比较，大就往下挪

删除元素的时候为什么既要调用 siftupTimer 又要调用 siftdownTimer 呢？因为不知道这个位置上元素的 when 到底是比父节点小呢还是比子节点的都要大，所以要”上窜下跳“。

所以，总结下 startTimer 主要就做两件事

1. 把定时器添加到堆里
2. 如果没有启动 timerproc 那么启动 timerproc 这个 goroutine

接下来就需要看一下 timerproc 到底干了些啥

## timerproc

timerproc 干的事情很简单其实就是取出堆顶元素看看是不是到点了，如果到点了，那就执行定时器对应的函数(实际上就是往 channel 扔时间拉，或者 AfterFunc 当参数传进来的自定义函数)。如果没到点，那就计算下要睡多久，然后就睡觉去了。这里需要注意的是对于 Ticker 这种定时器可不能直接移除栈顶元素，需要根据 period 重新计算 when，然后再调用 siftdownTimer 来调整它在 slice 中的位置也就即在堆中的位置。

## stopTimer

stopTimer 干的事儿也很简单找到定时器，从堆里面删除就是了。

## 总结

以上，对于 Go 标准库中的定时器实现就基本分析完了。其中里面有不少细节略过了，比如 notewakeup(&timers.waitnote)、goready(timers.gp, 0) 等对执行 timerproc 函数的 goroutine 自身的控制等。一来看注释也能瞎蒙出这些函数大致实现了什么功能，二来不了解这些函数并不影响定时器实现的主要逻辑，这里就先暂时省略。

通过这次代码阅读倒是学到了怎么用 slice 去实现一个堆。这也算是一个收获吧。




