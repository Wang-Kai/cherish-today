# Fatal vs Panic

### Fatal

What is `log.Fatal()` ?

```golang
// Fatal is equivalent to Print() followed by a call to os.Exit(1).
func Fatal(v ...interface{}) {
	std.Output(2, fmt.Sprint(v...))
	os.Exit(1)
}
```

Fatal 函数会先做一个参数打印，然后执行程序退出操作。

What is `os.Exit(1)` ?

```golang
func Exit(code int)
```

> Exit causes the current program to exit with the given status code. Conventionally, code zero indicates success, non-zero an error. The program terminates immediately; **deferred functions are not run**.

执行 `Exit()` 将会使程序直接退出，`Defer` 栈函数将不会执行。

### Panic

What is `panic()` ?

> **Panic** is a built-in function that stops the ordinary flow of control and begins panicking. When the function F calls panic, execution of F stops, any deferred functions in F are executed normally, and then F returns to its caller. To the caller, F then behaves like a call to panic. The process continues up the stack until all functions in the current goroutine have returned, at which point the program crashes. Panics can be initiated by invoking panic directly. They can also be caused by runtime errors, such as out-of-bounds array accesses.

panic 被调用后函数终止正常的运行流程，开始调用 Defer 栈上的函数，然后返回到其 Caller ，Caller 也像执行了 panic() 一样停止正常执行顺序，调用 Defer 栈上的函数，直到所在的 goroutine 结束。

提到 panic() 就不得不提它的搭档 recover():

> **Recover** is a built-in function that regains control of a panicking goroutine. Recover is only useful inside deferred functions. During normal execution, a call to recover will return nil and have no other effect. If the current goroutine is panicking, a call to recover will capture the value given to panic and resume normal execution.

`recover()` 函数只能在 `defer` 函数中调用，其将捕获 `panic(val)` 的参数然后做处理，可以使程序继续正常执行。

### Summary

**Fatal** 直接把程序终结了，`defer` 机制不会执行。**panic** 配合 `defer & recover` 更像是一种 `try & catch` 异常处理机制，来优雅的处理异常。


