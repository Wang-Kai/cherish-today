### 1. runtime.GOMAXPROCS() 是做什么的？

> The GOMAXPROCS variable limits the number of operating system threads that can execute user-level Go code simultaneously.

GOMAXPROCS 变量限制了可以并行执行用户层 Go 代码的操作系统线程数量。

### 2. G、M、P 是如何调度的？

> When a new G is created or an existing G becomes runnable, it is pushed onto a list of runnable goroutines of current P. When P finishes executing G, it first tries to pop a G from own list of runnable goroutines; if the list is empty, P chooses a random victim (another P) and tries to steal a half of runnable goroutines from it.

上文来自 Dmitry Vyukov 在 [《Scalable Go Scheduler Design Doc》](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#heading=h.mmq8lm48qfcw) 中的描述。翻译如下：**当一个 G 被创建或者退出的 G 变得可运行的时候，他被加入到当前 P 的可运行 Goroutine 列表中。当 P 完成了 G 的执行，它把 G 从列表中推出；如果列表是空的，P 就随机选择其他的 P，偷一半的 G 过来。**

![](https://static.ucloud.cn/405c3e45df3d4f72ad3e0877bd26230a.png)


新加入的 P 概念，P 代表执行 Go Code 所需要的资源。

```go
struct P
{
    Lock;
    G *gfree; // freelist, moved from sched
    G *ghead; // runnable, moved from sched
    G *gtail;
    MCache *mcache; // moved from M
    FixAlloc *stackalloc; // moved from M
    uint64 ncgocall;
    GCStats gcstats;
    // etc
    ...
};
```

从结构体来看，至少可以看出 P 会存储 G，并标识哪些是可被执行的。

> There is a P-specific local and a global goroutine queue. Each M should be assigned to a P. Ps may have no Ms if they are blocked or in a system call. At any time, there are at most GOMAXPROCS number of P. At any time, only one M can run per P. More Ms can be created by the scheduler if required.

摘自 [rakyll](https://rakyll.org/scheduler/) 大神的博客，释意如下：**整个调度中存在 P 的本地 G 队列，以及全局的 G 队列。每个 M 必须赋予 P 才能被执行。P 有时会没有 M ，当 M 都被阻塞的时候。任何时候，调度系统里有最多 GOMAXPROCS 个 P。任何时候，每个 P 只能运行一个 M。如果需要可以创建更多的 M。**

![](https://static.ucloud.cn/02cfc34ab8404dd9b66c1defe3dae413.png)

runtime 对 goroutine 的调度和操作系统对 进程 的调度不同，runtime 是一种**合作调度**，对于一个 goroutine 其会让他完全执行完毕。但 OS 的调度是基于 time slice 的调度，每个进程在时间片结束后，就会被调出，即使任务还未执行完毕。goroutine 只有在如下情况下才会被切换：

- Channels send and receive operations, if those operations would block.
- The Go statement, although there is no guarantee that the new Goroutine will be scheduled immediately.
- Blocking syscalls like file and network operations.
- After being stopped for a garbage collection cycle.
